# Service Analysis Common

Shared procedure for service or component analysis workflows that combine service understanding, telemetry evidence, and a user-facing artifact such as a local markdown report, health summary, or chat-only rollup.

Read `../references/telemetry-measurability.md` when the question depends on metric dimensions, historical rollups, or distinct-entity conclusions.
Read `../templates/analysis-child-result.md` when bounded child investigations are being used for a service-analysis workflow.
Read `../references/session-artifacts.md` when the investigation will continue across turns or switch tools.

## Common Workflow

1. Resolve the target, scope, and output surface.
- Start from the service, component, or code path the user gives.
- Read the local `AGENTS.md` guidance for that codebase and any relevant parent `AGENTS.md`.
- Write down the exact question being answered.
- Distinguish whether the user wants a chat answer, a local markdown artifact, or a publishable operational summary.
- If the user provides an explicit output target, treat it as the canonical output target unless there is a strong reason not to.

2. Set the defaults explicitly.
- Use exact absolute dates in all outputs.
- Default the environment to production unless the user explicitly asks for something else.
- Default to read-only investigation unless a local file or external publish is explicitly requested.
- When writing an artifact, the current workflow is the canonical writer for it unless a larger orchestrator owns it.

3. Build the code-backed understanding first.
- Inspect the local codebase before interpreting telemetry when local code inspection is relevant to the question.
- Use local code to disambiguate service identity, entrypoints, runtime variants, telemetry emitters, or important short-circuit behavior.
- When code inspection matters, record the files that establish the entrypoint, core execution path, signal emission, and any important short-circuit or no-op behavior.

4. Gather telemetry evidence second.
- Use the narrowest reliable telemetry surface first.
- Prefer topology discovery, exact metric series inspection, and exact filters over broad scans.
- Confirm that the dimensions or tags you intend to filter on are actually retained by the telemetry source.
- Treat fixed windows as the source of truth when long-range scalar rollups disagree with the sum of fixed windows.

5. Separate evidence from interpretation.
- Distinguish direct evidence from code, direct evidence from telemetry, and interpretation.
- Call out what the telemetry can answer directly and what it cannot answer from the available signal.

6. Write the user-facing artifact with caveats in the body.
- Include exact scope, exact dates, and exact filters.
- Put important caveats in the artifact itself, not only in the chat response.
- Include the exact query shapes used for the core evidence when the workflow depends on telemetry analysis.
- If the investigation will resume later, save the resolved scope, key queries, and next drill-downs in local session artifacts.

## Child Result Contract

Use this section when a service-analysis workflow splits into bounded child investigations.

- Keep the parent workflow as the canonical writer for the final artifact.
- Have each child return a compact evidence package using `../templates/analysis-child-result.md`.
- Each child package should preserve the exact question, exact scope, strongest direct evidence, interpretation, unresolved gap, and a parent-ready summary.

## Output Rules

- Use exact dates, not relative dates.
- Do not claim distinct-customer or distinct-entity results unless the telemetry preserves identity strongly enough to support that conclusion.
- If an important question cannot be answered from the current telemetry surface, say so explicitly.
- When writing a local markdown artifact, save it in the current working directory using a clear filename and link the path in the final response.
