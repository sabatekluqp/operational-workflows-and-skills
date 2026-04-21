---
name: service-metric-analysis
description: "Analyze a repo service or component to identify emitted metrics from code, query production telemetry in Dynatrace, and produce a local markdown report. Use when the user wants metric breakdowns, trend analysis, telemetry semantics, or caveats for a service such as credit-line-change or another repo-owned component."
---

# Service Metric Analysis

Read [../../workflows/service-metric-analysis.md](../../workflows/service-metric-analysis.md) before starting.

Default to production unless the user explicitly asks for another environment.
Use fixed calendar windows for long-range history instead of trusting one giant scalar rollup.
