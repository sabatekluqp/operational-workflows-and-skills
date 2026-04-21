---
name: dynatrace-investigation
description: "Investigate a deployment rollout, PagerDuty-backed incident, service bug, telemetry gap, or GUID-based data validation question in Dynatrace. Use when the user wants rollout health checks, debugging, incident investigation, or tracing exact ids such as GUIDs, order ids, customer ids, or correlation ids through logs, spans, events, or business telemetry."
---

# Dynatrace Investigation

Read [../../workflows/dynatrace-investigation.md](../../workflows/dynatrace-investigation.md) before starting.
Then read exactly one branch playbook:

- [../../workflows/dynatrace-rollout-check.md](../../workflows/dynatrace-rollout-check.md)
- [../../workflows/dynatrace-incident-path-analysis.md](../../workflows/dynatrace-incident-path-analysis.md)
- [../../workflows/dynatrace-service-debugging.md](../../workflows/dynatrace-service-debugging.md)
- [../../workflows/dynatrace-guid-trace.md](../../workflows/dynatrace-guid-trace.md)

Read [../../references/dynatrace-query-patterns.md](../../references/dynatrace-query-patterns.md) when you need starter DQL shapes or a trace-friendly query pattern.
Read [../../explanations/dynatrace-evidence-interpretation.md](../../explanations/dynatrace-evidence-interpretation.md) when the evidence shows caller-side failure without matching callee-side degradation, when service-local logs are missing, when a low-load or custom-metric alert may be misleading, or when you need to distinguish rollout correlation from code causation or onset from late secondary events.
Read [../../templates/incident-analysis-page.md](../../templates/incident-analysis-page.md) when the user wants an incident-style write-up.
Read [../../templates/dynatrace-investigation-result.md](../../templates/dynatrace-investigation-result.md) when this is a bounded child investigation feeding a parent incident workflow.
Read [../../references/confluence-routing.md](../../references/confluence-routing.md) only when the user wants the result published or routed to Confluence by owning team.

When invoked as a child investigation, stay within the assigned scope and return a structured evidence package instead of trying to narrate the entire incident.
