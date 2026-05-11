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

- Never sweep all services unless explicitly asked.
- Resolve the asker's GitHub team(s). If multiple teams come back, ask which team(s) to scope to.
- Pull `CODEOWNERS` from the relevant repo.
- Build the owned-services list from `services/<name>/` paths owned by those team(s).
- Record what was searched and what was excluded.

### 2. Pull the SLO summary across owned services

- Use the `sre.slo_summary.*` metrics for a single 7-day bucket over the owned services.
- Pull availability from `error` and `total_requests`.
- Pull a performance proxy from the slow latency buckets, usually `range_6 + range_7` over `total_requests`.
- Keep the query at one bucket for the full window so each service returns as one row; avoid smaller intervals that turn the math into arrays.
- Use `../references/dynatrace-query-patterns.md` when you need a starter query shape.

### 3. Classify each service against breach thresholds

- Confirm the target from the SLO Reporting dashboard when it is unclear.
- Classify each service as:
  - already breaching
  - close to breaching
  - healthy
- For flagged availability services, pull a daily breakdown to separate chronic issues from recent regressions.

### 4. Drill into endpoints for the flagged services

- For each flagged service, use server spans to find the routes driving failure or latency.
- Cap drill-down span queries at 24h by default and exclude `/healthz`.
- When latency is the issue, inspect client spans for the slow route to find the downstream call dominating wall time.
- Narrow by `url.path` when `span.name` is too coarse.

### 5. Distinguish real bugs from SLO-config calibration issues

- Before recommending a code fix, rule out common false positives:
  - 4xx counted as errors
  - latency dominated by an external dependency
  - auth failures misclassified as 5xx
- State the calibration interpretation explicitly so the user can decide between a code fix and an SLO-definition fix.

### 6. Recommend fixes per service

- For each flagged service, record the symptom, likely root cause, recommended fix, owner, and confidence.
- When a nearby service already shipped a relevant fix, assess whether the pattern transfers, but verify from source instead of assuming by naming.

### 7. Output

Default output is a chat summary. When the user asks for a write-up, default to Confluence under their team's home page (use `references/confluence-routing.md` for the routing rules).

The report should include:

- Exact 7-day window and environment.
- Resolved list of owned services examined and excluded.
- Per-service finding (symptom, root cause, recommendation).
- The exact DQL or query shapes used for the core evidence.
- Caveats about SLO-metric calibration where they apply.
- Links to the SLO Reporting dashboard filtered by service.

## Output Rules

- Lead with the breach list, sorted by severity, before the per-service drill-down.
- Distinguish availability findings from performance findings — they have different fixes.
- Always state whether a finding came from raw error metrics, span sampling, or interpretation.
- Never recommend "cache the token" without specifying *which* token, *what TTL*, and *what the security tradeoff is* — caching M2M tokens is safe, caching user bearer tokens is not.
- Say explicitly when the SLO `error` metric appears to count 4xx so the user can decide whether to fix the metric or the code.
- Cap drill-down span queries at 24h unless the question demands otherwise — wider windows scan hundreds of GB of spans.
