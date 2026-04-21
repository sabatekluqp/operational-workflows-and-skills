# Dynatrace Investigation

Use this file as the top-level router for Dynatrace work.

Read `../references/dynatrace-query-patterns.md` when you need starter DQL shapes for metrics, logs, spans, events, or GUID tracing.
Read `../explanations/dynatrace-evidence-interpretation.md` when you need help interpreting caller-vs-callee mismatch, telemetry gaps, late secondary events, or rollout-vs-code ambiguity.
Read `../references/subagent-usage.md` when deciding whether to split the investigation into independent child scopes.
Read `../templates/dynatrace-investigation-result.md` when this workflow is being used as a bounded child investigation that must return a structured evidence package.
Read `../explanations/dynatrace-investigation-pattern.md` when you need the rationale for the router structure, narrow entity scope, branch selection, or bounded child investigation rules.

After reading this router, read exactly one branch playbook:

- deployment rollout check: `../workflows/dynatrace-rollout-check.md`
- incident path analysis: `../workflows/dynatrace-incident-path-analysis.md`
- service debugging: `../workflows/dynatrace-service-debugging.md`
- GUID or data validation trace: `../workflows/dynatrace-guid-trace.md`

## Branch Selection

Choose the branch that matches the user's starting artifact.

### Deployment Rollout Check

Use when the user wants to know whether a deployment, release, or rollout caused degradation.

Typical inputs:

- service name
- deployment or rollout time
- environment
- suspected symptom

### Incident Path Analysis

Use when the user wants to investigate a live or recent incident, especially one that starts from PagerDuty.

Typical inputs:

- PagerDuty incident id or URL
- service name
- incident time window
- symptom summary

### Service Debugging

Use when the user wants to debug a service, endpoint, job, consumer, dependency failure, or user-facing error.

Typical inputs:

- service name
- endpoint, operation, consumer, or job name
- error text
- status code
- exception type
- customer, merchant, order, or request identifiers

### GUID or Data Validation Trace

Use when the user wants to search for a GUID, request id, order id, customer id, merchant id, correlation id, BI event key, or similar value across telemetry.

Typical inputs:

- one or more exact identifiers
- an expected downstream event, warehouse record, or BI event
- a time window
- an optional owning service

## Shared Preflight

Run these steps before branching.

1. Normalize the investigation target.
- Convert relative dates into exact timestamps in the user's timezone.
- Normalize exact ids before searching:
  - original form
  - lowercase or uppercase variants when relevant
  - dashless GUID form only when the source system is known to emit it that way
- Write down the exact question being answered:
  - whether a rollout caused regression
  - what caused an incident
  - why a service failed
  - where an identifier trail stops

2. Resolve the narrowest useful entity scope.
- If the user gives a service name, start with `find_entity_by_name`.
- Prefer production application `SERVICE-*` entities over sidecars, projections, infra helpers, or `Unexpected service` unless the user explicitly wants those.
- When entity discovery is ambiguous, use narrow Smartscape or entity-scoped DQL to reduce the candidate set before running broader analysis.
- Keep prod and non-prod separate from the start.
- Record the exact entities examined so the final answer can state what was and was not searched.

3. Classify the runtime shape before choosing evidence.
- HTTP or API service
- async worker, consumer, projection, or queue processor
- scheduled job or batch process
- downstream dependency or shared platform issue
- telemetry or BI validation path

4. Choose the first evidence source based on the question.
- Rollout check:
  - `list_problems`
  - request or processing volume
  - failure rate
  - latency or duration
  - deployment or change events if available
- Incident path analysis:
  - `list_problems`
  - service-level failure, latency, and volume changes
  - top failing endpoints or operations
  - logs, exceptions, spans, or events
- Service debugging:
  - workload-specific coarse signal first
  - then logs, exceptions, spans, and endpoint or operation detail
- GUID or data validation trace:
  - discover the data object and field names first when unclear
  - then search logs, spans, events, and business telemetry fields in expected order

5. Keep queries narrow.
- Scope by exact entity ids whenever possible.
- Scope by a concrete time window before searching logs, spans, or events.
- Prefer exact-match filters over `contains` when the field is known.
- Use `generate_dql_from_natural_language` when it is safer or faster than hand-writing DQL.
- Use `fetch dt.semantic_dictionary.models` when the right data object or field is unclear before guessing at schema names.
- Prefer one concrete hypothesis per query.

## Subagent Posture

- Default to one thread when the question is already narrow and bounded.
- Use subagents only when the top-level question can be split into clearly independent tracks with separate scopes.
- If this workflow is being used as a bounded child investigation under a parent orchestrator, usually stay local and return one compact evidence package.
- When subagents are used, follow `../references/subagent-usage.md`.

## Cross-Branch Heuristics

- Run `list_problems` before expensive drill-downs when the question smells like an incident or rollout regression.
- Avoid broad environment scans when topology or entity ids can answer the question.
- For HTTP services, do endpoint drill-down only after confirming overall regression or narrowing to one failing path.
- For workers, consumers, or projections, prioritize exceptions, backlog symptoms, retries, processing failures, and logs over request-rate analysis.
- For GUID tracing, search exact identifiers first and only relax the search if exact filters miss and there is evidence the identifier may be transformed.
- For BI-event validation, identify the actual telemetry object and field names before concluding an event is missing.
- Distinguish direct evidence from inference in every branch.

## Child-Investigation Contract

Use this section when another workflow, such as PagerDuty incident analysis, invokes this workflow as a bounded sub-investigation.

1. Take one narrow question.
- Accept one exact question, not a broad incident brief.
- Good examples:
  - whether one service path failed because of one downstream dependency
  - whether one database call family regressed in the incident window
  - where one identifier trail stopped

2. Keep the scope explicit.
- Record the exact time window.
- Record the exact entities, dependencies, objects, and fields searched.
- Do not silently widen from one scope to another.

3. Gather evidence using the matching branch.
- Use `../workflows/dynatrace-incident-path-analysis.md` for a narrow service-path or dependency-path investigation.
- Use `../workflows/dynatrace-service-debugging.md` for a specific failure path.
- Use `../workflows/dynatrace-guid-trace.md` for propagation and missing-event questions.
- Use `../workflows/dynatrace-rollout-check.md` for deployment-correlation questions.

4. Return a structured result package.
- Use `../templates/dynatrace-investigation-result.md`.
- Include:
  - exact question answered
  - exact time window searched
  - exact scoped entities and objects
  - direct evidence
  - interpretation
  - confidence
  - strongest unresolved gap
  - next best narrow query if unresolved

5. Do not act like the parent incident orchestrator.
- Do not try to explain the whole incident from one narrow track.
- Do not overwrite a shared parent document unless explicitly assigned sole ownership of a subpage or scratch artifact.
- Optimize for defensible evidence.

## Output Rules

- Prefer exact timestamps, entity ids, and query snippets over vague summaries.
- Lead with:
  - exact question answered
  - exact time window searched
  - exact entities and objects searched
  - primary conclusion
- Separate:
  - direct evidence
  - interpretation or inference
  - blind spots or unresolved ambiguity
- Include the exact DQL or key filtered query snippets that materially support the conclusion.
- Say explicitly when no evidence was found in the searched window and scope.
- Do not claim end-to-end absence unless the search scope and data objects actually support that conclusion.
- If unresolved, state the next best narrow query instead of recommending a broad re-scan.
