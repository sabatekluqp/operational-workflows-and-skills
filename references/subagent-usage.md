# Subagent Usage

Use this reference when deciding whether to split a workflow into parent plus child investigations or keep the work in one thread.

## Core Rule

Subagents are worth using when they reduce context pressure and allow bounded evidence gathering to happen in parallel.

Subagents are not worth using when the task is already narrow, when one coherent reader is more important than parallelism, or when the parent is blocked on the delegated result immediately.

## When To Use Subagents

Use subagents when all of these are true:

- the work splits into independent sidecar questions
- each sidecar question has a bounded scope
- the parent can keep moving without waiting on every result immediately
- the child can return a compact evidence package instead of a full narrative

Good split patterns:

- code-path discovery vs telemetry-history analysis
- endpoint inventory vs Dynatrace entity mapping
- one service path vs one downstream dependency path
- one claim validation track vs another claim validation track

## When Not To Use Subagents

Do not use subagents by default for:

- one already-bounded Dynatrace question
- one small repo inspection task
- one PR review where a single coherent reader matters more than parallelism
- a critical-path step that the parent needs before it can continue

Bad split patterns:

- `investigate this whole incident`
- `read the repo and tell me everything`
- overlapping code and telemetry scopes
- child tasks that all need to keep rewriting the same parent artifact

## Parent And Child Responsibilities

Parent responsibilities:

- define the narrow question for each child
- define the exact time window and entity scope when relevant
- remain the canonical writer for the parent artifact
- synthesize the final answer across child results

Child responsibilities:

- stay within the assigned scope
- gather direct evidence
- keep interpretation clearly separated from evidence
- return a compact result package, not a sprawling narrative

Use a shared child-result contract when one exists for the workflow family:

- incident-style Dynatrace tracks: `../templates/dynatrace-investigation-result.md`
- non-incident bounded analysis tracks: `../templates/analysis-child-result.md`

## Context Management Rules

- Prefer 2 to 3 small bounded subagents over one vague large one.
- Do not delegate overlapping scopes.
- Do not copy large raw logs, diffs, or query outputs into the parent unless they materially support the conclusion.
- Have each child return:
  - exact question answered
  - exact scope searched
  - strongest direct evidence
  - interpretation
  - unresolved gap if any

## Workflow-Specific Guidance

### PagerDuty Incident Analysis

- Strong fit for subagents.
- Use a parent orchestrator plus bounded child investigations.
- Child investigations should return evidence packages and should not write the parent Confluence page directly.

### Dynatrace Investigation

- Usually stay local when the incoming question is already narrow.
- Use subagents only when the top-level question can be split into clearly independent tracks.
- If used as a child investigation under another workflow, usually stay local.

### Service Metric Analysis

- Optional fit.
- Good split:
  - code-path and metric-semantics discovery
  - Dynatrace history, breakdowns, and rollups
- Parent should remain the canonical writer for the Confluence page.
- When used, child results should follow `../templates/analysis-child-result.md`.

### Service Endpoint Traffic Analysis

- Optional fit.
- Good split:
  - endpoint inventory from code
  - Dynatrace entity mapping and traffic rollups
- Avoid splitting if the service is small or the endpoint surface is already trivial.
- When used, child results should follow `../templates/analysis-child-result.md`.

### Service System Documentation

- Strong fit.
- Good split:
  - code structure and runtime entrypoints
  - interfaces and schemas
  - Dynatrace topology, traffic, and health
  - operability and debugging guidance
- Keep the parent as the canonical writer for the final documentation set.
- Child outputs may live in markdown scratch or temporary notes, but they should still return compact evidence packages rather than final reference pages.
- When used, child results should follow `../templates/analysis-child-result.md`.

### Incident Follow-up Planning

- Optional fit.
- Use subagents only when there are many independent claims to validate or multiple bounded evidence tracks to re-check.
- Keep story synthesis and final Jira-writing decisions in the parent thread.
- When used, child results should follow `../templates/analysis-child-result.md`.

### PagerDuty Assigned Service Health

- Usually stay local.
- Consider subagents only when the owned service set is large enough that independent per-service health checks can run in parallel.
- Keep the parent responsible for the final rollup and any Slack publication.
- When used, child results should follow `../templates/analysis-child-result.md`.

### PR Review And Triage

- Usually stay local.
- Review quality often benefits from one coherent reader more than from parallel exploration.
