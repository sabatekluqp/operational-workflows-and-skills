---
name: service-metric-analysis
description: "Analyze a repo service or component to identify emitted metrics from code, query Dynatrace telemetry, and produce a local markdown report. Use for metric breakdowns, trend analysis, telemetry semantics, and caveat analysis."
---

# Service Metric Analysis

Read [../../workflows/service-metric-analysis.md](../../workflows/service-metric-analysis.md) before starting.
Read [../../references/token-budget.md](../../references/token-budget.md) when you need the shared token-cost model.

Token budget summary:
- Claude: `20%` session history + artifacts, `15%` workflow/reference/template, `15%` external context, `15%` tool output, `12%` coding, `10%` runtime evidence, `8%` planning, `5%` wrapper
- Codex: `28%` coding, `24%` build/test/verification, `14%` workflow/reference/template, `14%` tool output, `12%` session history + artifacts, `10%` external context, `8%` planning, `4%` wrapper

Default to production unless the user explicitly asks for another environment.
Use fixed calendar windows for long-range history instead of trusting one giant scalar rollup.
