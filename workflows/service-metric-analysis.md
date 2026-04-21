# Service Metric Analysis

Inspect a repo service or component to identify emitted metrics from code, analyze the metric telemetry in Dynatrace, and write the results to a local markdown report.

Read `../workflows/service-analysis-common.md` before starting.
Read `../templates/service-metric-analysis-page.md` when writing the report body.
Read `../references/dynatrace-query-patterns.md` when you need starter DQL shapes.
Read `../references/telemetry-measurability.md` when the question depends on metric dimensions, historical rollups, or distinct-entity claims.
Read `../references/subagent-usage.md` when deciding whether to split code discovery from telemetry analysis.
Read `../templates/analysis-child-result.md` when bounded child investigations are being used.

## Subagent Posture

- Optional, not default.
- Good split:
  - code-path and metric-semantics discovery
  - Dynatrace history, breakdowns, and rollups
- Keep the parent as the canonical writer for the report.
- Avoid splitting small or already-localized questions.
- When split, have each child return `../templates/analysis-child-result.md`.

## Workflow

1. Resolve the analysis-specific question and output target.
- Write down the exact question being answered:
  - metric inventory
  - bypass or outcome breakdown
  - daily or monthly trends
  - duration analysis
  - customer-level question that may or may not be supported
- Derive a default report filename as `<service-or-component>-metric-analysis.md`.
- Save the report in the current working directory unless the user explicitly asks for a different path.

2. Identify the metrics in code.
- Inspect the service entry path and follow the call path to the metric emitter.
- Prefer `rg` searches for likely emitters such as:
  - `StatsD`
  - `IStatsDPublisher`
  - `Meter`
  - `Counter`
  - `Histogram`
  - `Gauge`
  - `Telemetry`
  - `AddTelemetry`
  - `Prometheus`
  - metric name fragments
- Record:
  - exact metric names
  - exact dimensions or tags emitted
  - where timing starts and stops
  - what makes a result count as bypassed, failed, or changed
  - conditions where no metric is emitted at all
- Distinguish event-count telemetry from identity-preserving telemetry. If `customerId`, `orderId`, or another unique identifier is absent from the metric dimensions, say so explicitly.

3. Verify the metric surface in Dynatrace.
- Use `fetch metric.series` first to inspect the retained dimensions before doing rollups.
- Confirm the production filter dimensions actually exist on the metric, such as `env`, `service`, or similar retained tags.
- Prefer exact filters over fuzzy searches.
- If a metric is present in code but absent in Dynatrace, say so explicitly and narrow the likely reasons:
  - environment mismatch
  - metric not emitted in the searched window
  - dimension mismatch
  - ingestion or retention issue

4. Calculate the requested analysis.
- For long histories, prefer:
  - fixed calendar months for monthly reporting
  - fixed days for daily averages
- Do not rely on one giant scalar query when fixed-window sums disagree with it.
- Compute at least:
  - total event count in scope
  - average per day
  - monthly-equivalent average when the history is partial
  - requested dimension breakdowns such as `isbypassed`, `changetype`, `testcohort`, or other retained dimensions
- When the user asks for distinct-customer or distinct-entity questions:
  - first verify whether the metric retains a unique identifier dimension
  - if not, state clearly that the metric stream cannot answer the question
  - only pivot to logs, spans, or events if the user explicitly wants a broader non-metric analysis

5. Explain why the telemetry looks the way it does.
- Tie the observed dimensions and percentages back to the code path.
- Call out important distinctions such as:
  - bypassed metric result vs top-level kill switch with no metric
  - `changetype=None` vs true no-op vs bypass path
  - timing metric including model build or post-assessment work
  - partial dimension coverage in Dynatrace vs total raw event count
- Separate:
  - direct evidence from code
  - direct evidence from Dynatrace
  - interpretation or inference

6. Write the markdown report.
- Save to `<service-or-component>-metric-analysis.md` in the current working directory (or the path the user specifies).
- Mirror the structure in `../templates/service-metric-analysis-page.md`.
- Include exact absolute timestamps and exact filters used.
- Include the exact DQL used for the core rollups.
- State all important caveats in the report body, not only in the final chat response.

## Report Content Standard

Use concise, decision-oriented prose.

The report should usually include:

- executive summary
- exact scope and date range
- code-path explanation
- metric inventory with retained dimensions
- historical summary
- daily and monthly averages
- breakdowns by the requested dimensions
- explicit limitations
- exact DQL used

If the user asks for customer-level conclusions and the metric does not preserve identity, say this explicitly:

- this metric supports event counts, not distinct-customer counts

If fixed-window sums and one-shot long-range scalars disagree, say this explicitly:

- fixed-window sums are being treated as the source of truth for the report

## Output Rules

- Link the report file path in the final response.
- Distinguish:
  - event counts
  - unique-entity counts
  - what is measurable vs not measurable from the metric stream
- When a dimensioned breakdown does not sum to the full total, call out likely missing-dimension coverage instead of hiding the mismatch.
