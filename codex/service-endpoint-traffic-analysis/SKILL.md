---
name: service-endpoint-traffic-analysis
description: "Analyze a repo service by name, inventory its HTTP endpoints from code, map those endpoints to Dynatrace traffic, and produce a local markdown report. Use for endpoint usage analysis, endpoint traffic tiers, or deprecation-candidate review."
---

# Service Endpoint Traffic Analysis

Read [../../workflows/service-endpoint-traffic-analysis.md](../../workflows/service-endpoint-traffic-analysis.md) before starting.
Read [../../references/token-budget.md](../../references/token-budget.md) when you need the shared token-cost model.

Token budget summary:
- Claude: `20%` session history + artifacts, `15%` workflow/reference/template, `15%` external context, `15%` tool output, `12%` coding, `10%` runtime evidence, `8%` planning, `5%` wrapper
- Codex: `28%` coding, `24%` build/test/verification, `14%` workflow/reference/template, `14%` tool output, `12%` session history + artifacts, `10%` external context, `8%` planning, `4%` wrapper

Follow the shared workflow and include exact absolute dates in both the report and the user-facing summary.
