---
name: dynatrace-investigation
description: "Investigate a rollout regression, incident, service bug, telemetry gap, or exact-id trace in Dynatrace. Use for rollout checks, incident-path analysis, service debugging, and proving whether identifiers such as GUIDs, order ids, or correlation ids flowed through the expected telemetry path."
---

# Dynatrace Investigation

Read [../../workflows/dynatrace-investigation.md](../../workflows/dynatrace-investigation.md) before starting.
Read [../../references/token-budget.md](../../references/token-budget.md) when you need the shared token-cost model.

Token budget summary:
- Claude: `20%` session history + artifacts, `15%` workflow/reference/template, `15%` external context, `15%` tool output, `12%` coding, `10%` runtime evidence, `8%` planning, `5%` wrapper
- Codex: `28%` coding, `24%` build/test/verification, `14%` workflow/reference/template, `14%` tool output, `12%` session history + artifacts, `10%` external context, `8%` planning, `4%` wrapper
Then read exactly one branch playbook:

- [../../workflows/dynatrace-rollout-check.md](../../workflows/dynatrace-rollout-check.md)
- [../../workflows/dynatrace-incident-path-analysis.md](../../workflows/dynatrace-incident-path-analysis.md)
- [../../workflows/dynatrace-service-debugging.md](../../workflows/dynatrace-service-debugging.md)
- [../../workflows/dynatrace-guid-trace.md](../../workflows/dynatrace-guid-trace.md)

Read [../../references/dynatrace-query-patterns.md](../../references/dynatrace-query-patterns.md) when you need starter DQL shapes or a trace-friendly query pattern.
Read [../../templates/incident-analysis-page.md](../../templates/incident-analysis-page.md) when the user wants an incident-style write-up.
Read [../../templates/dynatrace-investigation-result.md](../../templates/dynatrace-investigation-result.md) when this is a bounded child investigation feeding a parent incident workflow.
Read [../../references/confluence-routing.md](../../references/confluence-routing.md) only when the user wants the result published or routed to Confluence by owning team.

When invoked as a child investigation, stay within the assigned scope and return a structured evidence package instead of trying to narrate the entire incident.
