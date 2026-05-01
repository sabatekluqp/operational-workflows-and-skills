# SLO Breach Investigation

Use this workflow to identify which services owned by the asker are breaching or close to breaching their SLOs, drill into the endpoints causing each breach, and propose concrete fixes.

Read `../workflows/service-analysis-common.md` before starting.
Read `../references/dynatrace-query-patterns.md` when you need starter DQL.
Read `../references/telemetry-measurability.md` when interpreting `sre.slo_summary` rollups (especially `range_1..7` performance buckets).
Read `../references/subagent-usage.md` when deciding whether to split the per-service drill-downs.

## Defaults

- Production environment.
- 7-day window aligned to the SLO Reporting dashboard.
- Read-only investigation.
- Service scope: services owned by the asker, derived from `CODEOWNERS` + the asker's GitHub team membership. Do not widen scope without explicit ask.

## Workflow

### 1. Resolve the asker's owned services

The skill is per-team — never sweep all services unless explicitly asked.

1. Resolve the asker's GitHub team(s):
   - Run `gh api user/teams --jq '.[] | select(.organization.login == "<org>") | .slug'`.
   - If multiple teams come back, ask the asker which team(s) to scope to.
2. Pull `CODEOWNERS` from the relevant repo.
3. Build the owned-services list: every `services/<name>/` path whose owners include the asker's team.
4. Record the resolved list explicitly in the report — what was searched and what was excluded matters when the user pushes back.

### 2. Pull the SLO summary across owned services

The Zip SLO metrics are emitted by the `custom:zip-slos` extension into `sre.slo_summary.*`:

- `sre.slo_summary.error` — failed requests (the SLI numerator for availability)
- `sre.slo_summary.total_requests` — total requests (the SLI denominator)
- `sre.slo_summary.range_1..7` — latency-bucket counters (the performance SLI uses these; the exact formula is service-specific, but `1 - (range_6 + range_7) / total_requests` is a useful upper-bound proxy)

Dimensions: `service`, `environment`, `dt.metrics.source = "custom:zip-slos"`.

Use a single 7-day bucket so the result is one row per service. Do not pass `interval` smaller than the window or you will get arrays and the per-row math will silently return `null`.

```dql
timeseries {
  errors = sum(sre.slo_summary.error),
  total = sum(sre.slo_summary.total_requests),
  r6 = sum(sre.slo_summary.range_6),
  r7 = sum(sre.slo_summary.range_7)
}, from: now()-7d, to: now(), by:{service, environment}
| filter environment == "production" and in(service, <owned-services>)
| fieldsAdd e = arraySum(errors), t = arraySum(total), slow = arraySum(r6) + arraySum(r7)
| filter t > 10000
| fieldsAdd availability_pct = (1.0 - toDouble(e)/toDouble(t)) * 100,
            perf_proxy_pct  = (1.0 - toDouble(slow)/toDouble(t)) * 100
| fields service, e, t, availability_pct, perf_proxy_pct
| sort availability_pct asc
```

### 3. Classify each service against breach thresholds

Common targets for Zip services (confirm against the SLO Reporting dashboard if uncertain):

- Availability: 99.95% (some services use 99.9% or 99%).
- Performance: 95% (some services use 99%).

Bucket each service:

- **Already breaching** — availability or performance below target.
- **Close to breaching** — within ~1.5x the error budget of the target (e.g., 99.93% on a 99.95% target with one full week consumed).
- **Healthy** — comfortably above target.

For services flagged as breaching availability, also pull a daily breakdown to distinguish a chronic pattern from a recent regression:

```dql
timeseries {
  errors = sum(sre.slo_summary.error),
  total = sum(sre.slo_summary.total_requests),
  r6 = sum(sre.slo_summary.range_6),
  r7 = sum(sre.slo_summary.range_7)
}, from: now()-7d, to: now(), by:{service, environment}, interval: 1d
| filter environment == "production" and in(service, <flagged-services>)
```

### 4. Drill into endpoints for the flagged services

For each flagged service, query spans to see which routes drive the slow or failing share. Watch the data scan — wide span queries scan hundreds of GB. Default to a 24h window for the drill-down and exclude `/healthz`.

```dql
fetch spans, from: now()-24h, to: now()
| filter in(service.name, <flagged-services>)
  and span.kind == "server"
  and http.route != "GET /healthz"
| summarize {
    req = count(),
    errs_5xx = countIf(toLong(http.status_code) >= 500),
    errs_4xx = countIf(toLong(http.status_code) >= 400 and toLong(http.status_code) < 500),
    p50_ms = percentile(toDouble(duration)/1000000, 50),
    p95_ms = percentile(toDouble(duration)/1000000, 95),
    p99_ms = percentile(toDouble(duration)/1000000, 99)
  }, by:{service.name, http.route}
| filter req > 100
| sort p95_ms desc
```

When latency is the issue, also pull the `client`-kind spans to see which downstream calls dominate wall-time. The route giving the slowest p95 is rarely the cause — the slowest *downstream* in its trace usually is:

```dql
fetch spans, from: now()-3h, to: now()
| filter service.name == "<service>" and span.kind == "client"
| summarize {
    req = count(),
    p50_ms = percentile(toDouble(duration)/1000000, 50),
    p95_ms = percentile(toDouble(duration)/1000000, 95),
    p99_ms = percentile(toDouble(duration)/1000000, 99),
    total_ms = sum(toDouble(duration)/1000000)
  }, by:{span.name}
| sort total_ms desc
```

For specific paths, narrow by `url.path` to expose the actual upstream endpoint (often more useful than `span.name`, which only carries the host).

### 5. Distinguish real bugs from SLO-config calibration issues

Before recommending a code fix, rule out these common false positives:

- **4xx counted as errors.** If `sre.slo_summary.error` for a service is non-zero but the corresponding spans show only 4xx (no 5xx), the SLO definition is including client errors. The fix is in the SLO config, not the service.
- **External-dependency latency.** If the slow downstream is a third party (Auth0, TransUnion, Stripe), the service has no internal lever to pull. The fix is to revisit the SLO target, add caching, or both.
- **401/403 misclassified as 5xx.** Some auth middleware throws inside a path that converts to 500. Confirm the actual response status before treating it as a server fault.

State the calibration interpretation explicitly in the report so the reader can decide whether to retune the SLO or open a code ticket.

### 6. Recommend fixes per service

For each flagged service, write down:

- **Symptom** — availability or performance number, breach severity, daily trend.
- **Root cause** — the specific endpoint(s), downstream, or misclassification.
- **Recommended fix** — code change, SDK config, SLO retune, or caching.
- **Owner** — the team from `CODEOWNERS`.
- **Confidence** — direct evidence vs. inference.

Where a fix already shipped against an adjacent service (look at recent merged PRs in the owning team's area), call it out and assess whether the same pattern transfers. Verify by reading the candidate service's source — do not assume from naming.

### 7. Output

Default output is a chat summary. When the user asks for a write-up, default to Confluence under their team's home page (use `references/confluence-routing.md` for the routing rules).

The report should include:

- Exact 7-day window and environment.
- Resolved list of owned services examined and excluded.
- Per-service finding (symptom, root cause, recommendation).
- The exact DQL used for the core evidence.
- Caveats about SLO-metric calibration where they apply.
- Links to the SLO Reporting dashboard filtered by service.

## Output Rules

- Lead with the breach list, sorted by severity, before the per-service drill-down.
- Distinguish availability findings from performance findings — they have different fixes.
- Always state whether a finding came from raw error metrics, span sampling, or interpretation.
- Never recommend "cache the token" without specifying *which* token, *what TTL*, and *what the security tradeoff is* — caching M2M tokens is safe, caching user bearer tokens is not.
- Say explicitly when the SLO `error` metric appears to count 4xx so the user can decide whether to fix the metric or the code.
- Cap drill-down span queries at 24h unless the question demands otherwise — wider windows scan hundreds of GB of spans.
