# Service Endpoint Traffic Analysis

Inventory a service's HTTP endpoints from code, map them to Dynatrace traffic, and write the results to a local markdown report.

Read `../workflows/service-analysis-common.md` before starting.
Read `../templates/endpoint-traffic-analysis-page.md` when writing the report body.
Read `../references/subagent-usage.md` when deciding whether to split code inventory from Dynatrace mapping.
Read `../templates/analysis-child-result.md` when bounded child investigations are being used.

## Subagent Posture

- Optional, not default.
- Good split:
  - endpoint inventory from code
  - Dynatrace entity mapping and traffic rollups
- Keep the parent as the canonical writer for the report.
- Avoid splitting small services or trivial endpoint surfaces.
- When split, have each child return `../templates/analysis-child-result.md`.

## Workflow

1. Resolve the analysis target.
- Start from the service name the user gives, usually under `services/<service-name>`.
- Derive a default report filename as `<service-name>-endpoint-traffic-analysis.md`.
- Save the report in the current working directory unless the user explicitly asks for a different path.

2. Build the endpoint inventory from code.
- Inspect `Startup.cs`, `Program.cs`, and all `*Controller.cs` files under the service.
- Identify the service base path, controller routes, action routes, HTTP verbs, and route aliases.
- Normalize the inventory to canonical endpoint paths with a leading `/`.
- Treat infrastructure routes such as `/healthz` separately from controller-defined business endpoints.
- If a controller exposes route aliases, collapse them into a single inventory row and call out the alias in notes.

3. Resolve the Dynatrace entities for the requested environments.
- Default to the same eastus split used in prior analyses unless the user asks for something else:
  - `eastus dev`
  - `eastus prod`
- Use topology discovery first, not wide span scans.
- Start with `K8S_SERVICE` or `K8S_DEPLOYMENT` objects for the service name, then walk edges to the backing `SERVICE-*` entities.
- Confirm which `SERVICE-*` ids are the actual HTTP application services.
- Exclude sandbox or alternate dev clusters unless the user explicitly requests them.
- Exclude sidecars, `Unexpected service`, message receivers, and other non-HTTP or background services from the endpoint table.

4. Pull endpoint traffic.
- Use endpoint-level request counts grouped by `dt.entity.service` and `endpoint.name`.
- Prefer the last 30 days unless the user requests another timeframe.
- Keep the query scoped to the confirmed `SERVICE-*` ids for the requested environments.
- Avoid broad environment scans when topology or narrow metric queries can answer the question.

5. Map Dynatrace names back to the code inventory.
- Normalize Dynatrace endpoint names when they omit the base path or leading slash.
- Match canonical controller endpoints to observed `endpoint.name` values.
- Ignore malformed requests, Swagger paths, TRACE traffic, raw concrete-id paths, service-name endpoints, and other obvious noise unless the user explicitly wants them analyzed.
- Use these default tiers unless the user asks for something else:
  - `Very high`: more than 10M requests in 30 days
  - `High`: 1M to 10M in 30 days
  - `Medium`: 100k to 1M in 30 days
  - `Low`: 1k to 100k in 30 days
  - `Very low`: 1 to 999 in 30 days
  - `None observed`: no matching canonical endpoint metric series in the requested window

6. Write the markdown report.
- Save to `<service-name>-endpoint-traffic-analysis.md` in the current working directory (or the path the user specifies).
- Mirror the structure in `../templates/endpoint-traffic-analysis-page.md`.
- Link the final file path in the chat summary.

## Report Content Standard

Use concise, decision-oriented prose. Favor exact dates instead of relative dates.

For the endpoint table:

- include one row per canonical endpoint
- show `eastus dev` and `eastus prod` usage tiers with approximate counts
- add a short `Notes` column for aliases, dev-only or prod-only behavior, or cleanup risk

For recommendations, group findings into:

- `Safest Deprecation Candidates First`
- `Next Wave After Caller Validation`
- `Clearly Not Cleanup Targets`

State this caveat explicitly in some form:

- `None observed` means no matching canonical endpoint series was observed in Dynatrace, not proof that usage is impossible.

## Dynatrace Query Pattern

Use topology discovery before traffic aggregation.

Typical sequence:

- find `K8S_SERVICE` or `K8S_DEPLOYMENT` nodes for the service name
- use topology edges to identify the relevant `SERVICE-*` entities for the requested environments
- inspect those `SERVICE-*` nodes and discard sidecars or background services
- run the final 30-day request-count query only against the selected application `SERVICE-*` ids

Include the exact DQL used in the final report.

## Output Rules

- Use the service name to inspect the repo; do not rely on memory for endpoint inventory.
- Prefer primary evidence from code, topology, and Dynatrace metrics.
- Link the report file path in the final response.
- Summarize the high-signal outcomes for the user:
  - where the report lives
  - which endpoints are clearly active
  - which zero-traffic or low-traffic endpoints look like cleanup candidates
