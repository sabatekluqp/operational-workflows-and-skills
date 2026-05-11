---
name: slo-breach-investigation
description: "Find the asker's services that are breaching or close to breaching their SLOs in Dynatrace, drill into the endpoints driving the breach, and propose fixes. Use for SLO health sweeps, cross-service root-cause investigation, and reliability-budget summaries."
---

# SLO Breach Investigation

Read [../../workflows/slo-breach-investigation.md](../../workflows/slo-breach-investigation.md) before starting.
Read [../../references/token-budget.md](../../references/token-budget.md) when you need the shared token-cost model.

Token budget summary:
- Claude: `20%` session history + artifacts, `15%` workflow/reference/template, `15%` external context, `15%` tool output, `12%` coding, `10%` runtime evidence, `8%` planning, `5%` wrapper
- Codex: `28%` coding, `24%` build/test/verification, `14%` workflow/reference/template, `14%` tool output, `12%` session history + artifacts, `10%` external context, `8%` planning, `4%` wrapper

Read [../../workflows/service-analysis-common.md](../../workflows/service-analysis-common.md) for shared scope, evidence, and output rules.
Read [../../references/dynatrace-query-patterns.md](../../references/dynatrace-query-patterns.md) when you need starter DQL shapes.
Read [../../references/telemetry-measurability.md](../../references/telemetry-measurability.md) when interpreting `sre.slo_summary` rollups, especially the `range_X` performance buckets.

Default to production unless the user explicitly asks for another environment.
Default to a 7-day window aligned to the SLO Reporting dashboard.
Always restrict the candidate service list to services owned by the asker unless the user explicitly asks for a wider scope.
