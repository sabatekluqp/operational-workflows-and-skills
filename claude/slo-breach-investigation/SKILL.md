---
name: slo-breach-investigation
description: "Find services owned by the asker that are breaching or close to breaching their SLOs in Dynatrace, drill into the endpoints causing each breach, and propose fixes. Use when the user wants an SLO health sweep, root-cause hunt across their owned services, or a summary of where their team's reliability budget is being spent."
---

# SLO Breach Investigation

Read [../../workflows/slo-breach-investigation.md](../../workflows/slo-breach-investigation.md) before starting.

Read [../../workflows/service-analysis-common.md](../../workflows/service-analysis-common.md) for shared scope, evidence, and output rules.
Read [../../references/dynatrace-query-patterns.md](../../references/dynatrace-query-patterns.md) when you need starter DQL shapes.
Read [../../references/telemetry-measurability.md](../../references/telemetry-measurability.md) when interpreting `sre.slo_summary` rollups, especially the `range_X` performance buckets.

Default to production unless the user explicitly asks for another environment.
Default to a 7-day window aligned to the SLO Reporting dashboard.
Always restrict the candidate service list to services owned by the asker (resolved from `CODEOWNERS` + the asker's GitHub team membership) unless the user explicitly asks for a wider scope.
