# Deploy And Validate

Use this workflow to deploy a branch or PR to CI in Azure DevOps, wait for the run (including BDD) to complete, then check the deployed service and its topology-derived neighbors in Dynatrace for errors that look related to the change. Emit a merge-readiness verdict.

Read `../references/dynatrace-query-patterns.md` when you need starter DQL. This workflow ships its own filter shape for the "same service, same build version, post-deploy window" case, which is the primary lens here.

## Fixed Environment

- Dynatrace cluster: `zip-aks-cluster-dev-green`. Do not query prod or other non-prod clusters from this skill.
- Azure DevOps: `https://dev.azure.com/quadpay/quadpay-services`.
- Runtime shape of the pipelines: `.NET service` → `Build, test solution` → (`Check ephemeral` in parallel) → `Build, publish docker image` → `Deploy ci` → `BDD - <service> (CI)`. Downstream stages (display names `Deploy ut`, `Deploy ut_2`, `Deploy pd`, `GitHub release` — internal identifiers `ut`, `ut_2`, `pd`, `GitHubRelease_`) are always skipped here.
- **`az rest` for Azure DevOps requires `--resource "499b84ac-1321-427f-aa17-267ca6975798"`** (the Azure DevOps app registration id). Without it, `az rest` acquires an ARM token that Azure DevOps rejects with a sign-in HTML page instead of a real API error. Every `az rest` example below includes this flag. `az pipelines *` commands handle auth internally and do not need it.

## Inputs

Always ask for one of:

- a PR URL (preferred — carries branch, diff, checks, and inferred pipeline all together), or
- a branch name (require the user to also confirm the service if you cannot deduce it)

Do not assume the current working directory tells you the target. This skill runs against Azure DevOps, not the local repo.

## Workflow

### 1. Resolve the target

- If given a PR URL: `gh pr view <url> --json headRefName,baseRefName,title,url,files,statusCheckRollup,commits` — extract `headRefName` (branch), the list of changed files, and any existing pipeline runs already attached as checks.
- If given a branch: `gh pr list --head <branch> --state open --json url,number,headRefName,files` to find the PR. If none exists, ask the user for the service name explicitly.
- Deduce the service from the changed file paths (top-level directory or well-known project folder). If the PR touches multiple services, ask the user which one to deploy — this skill runs one pipeline at a time.
- Confirm the deduced service with the user before continuing. A wrong pipeline is a wasted 30 minutes.

### 2. Preflight auth

Run in order, stopping and asking the user to act if a check fails:

```bash
az account show --query '{name:name, user:user.name, id:id}' -o json
```

If the command errors or returns nothing, tell the user to run `az login` themselves (interactive login cannot be automated from inside the skill). Wait for confirmation before continuing.

Also verify GitHub access is present: `gh auth status`.

### 3. Discover the pipeline

Given the service name, find its Azure DevOps pipeline definition id.

```bash
az pipelines list \
  --organization https://dev.azure.com/quadpay \
  --project quadpay-services \
  --name "*<service>*" \
  --query "[].{id:id, name:name, path:path}" -o json
```

If more than one candidate returns, present the list and ask the user to pick. Cache the chosen `definitionId` for later steps. Known reference: `decision-engine = 34`.

Also fetch the stage **identifiers** from the most recent completed run — the Runs API `stagesToSkip` payload accepts identifiers, not display names (see step 5). This is the fastest way to get them without pulling the pipeline YAML:

```bash
recentBuild=$(az rest --method GET \
  --resource "499b84ac-1321-427f-aa17-267ca6975798" \
  --uri "https://dev.azure.com/quadpay/quadpay-services/_apis/build/builds?definitions=<definitionId>&\$top=1&api-version=7.0" \
  --query "value[0].id" -o tsv)

az rest --method GET \
  --resource "499b84ac-1321-427f-aa17-267ca6975798" \
  --uri "https://dev.azure.com/quadpay/quadpay-services/_apis/build/builds/${recentBuild}/timeline?api-version=7.0" \
  --query "records[?type=='Stage'].{name:name, identifier:identifier}" -o json
```

Observed identifiers on Zip `.NET service` pipelines (compliance.api, decision-engine, and siblings share this shape): `Build` (Build, test solution), `CheckEphemeral` (Check ephemeral), `BuildImage` (Build, publish docker image), `ci` (Deploy ci), `ci_bdd_<service>_CI` (BDD stage), `ut` / `ut_2` (Deploy ut / ut_2), `pd` (Deploy pd), `GitHubRelease_` (GitHub release). If the queried run does not match this shape, use the identifiers you just fetched and warn the user that the pipeline layout diverges from the defaults this skill was designed against.

### 4. Baseline snapshot (pre-deploy)

Establish a comparison anchor before the pipeline runs. Dev traffic is usually sparse and BDD-driven, so this is often qualitative — say so explicitly if the pre-window has almost no data.

- Resolve the service entity via `find_entity_by_name`. Zip services frequently surface **two entities for the same pod**:
  - a `.NET agent` view named like `<service> (Zip.<Service>)` — carries logs and .NET-agent spans, but **`service.version` is `null`** on this entity.
  - an `OTel` sibling named like `<service>-primary (OTel)` — carries OTel-instrumented spans with `service.version` populated (matches the Azure DevOps build number, e.g. `20260903.5`).

  Confirm which cluster each candidate lives in before proceeding — the same service name typically has one entity per k8s cluster (dev-green + prod-green + others). Use this one-shot DQL to disambiguate:

  ```text
  fetch spans, from: now()-2h
  | filter in(dt.entity.service, array("SERVICE-<candidate-1>", "SERVICE-<candidate-2>", ...))
  | summarize count(), by:{dt.entity.service, k8s.cluster.name}
  ```

  Record **both** the .NET-agent entity id and (when it exists) the OTel entity id. Discard entities named `... (Unexpected service)` and anything outside `zip-aks-cluster-dev-green`.

- Grab the last-30-minutes counts on both entities: failure count, exception count, top error operations. Use `list_problems` for both entities and the window as well.
- Send a deploy-start marker so the timeline is annotated for future investigations:
  - `send_event` with `eventType: CUSTOM_DEPLOYMENT`, a distinctive title (e.g. `deploy-and-validate: <service> <branch> run <runId> start`), and the .NET-agent service entity attached via `entitySelector`. Include the branch, commit SHA, PR URL, and build number as `properties`.

Write down the pre-deploy timestamp — you will use it as the anchor for both the post-deploy comparison window and the topology-neighbor scan.

### 5. Trigger the pipeline with the right stages

`az pipelines run` uses the older builds API and does not accept `stagesToSkip`. Use the Runs API through `az rest`. **`stagesToSkip` takes stage identifiers, not display names** — passing the display name `"Deploy ut"` returns `PipelineValidationException: 'Deploy ut' is not a valid stage to skip.` Use the identifiers from step 3.

Write the payload to a file first — inline JSON quoting via `--body` is fragile in zsh (backticks, `$` expansion, embedded quotes):

```bash
cat > /tmp/deploy-and-validate-run.json <<'EOF'
{
  "stagesToSkip": ["ut", "ut_2", "pd", "GitHubRelease_"],
  "resources": {
    "repositories": {
      "self": { "refName": "refs/heads/<branch>" }
    }
  }
}
EOF

az rest --method POST \
  --resource "499b84ac-1321-427f-aa17-267ca6975798" \
  --uri "https://dev.azure.com/quadpay/quadpay-services/_apis/pipelines/<definitionId>/runs?api-version=7.0" \
  --headers "Content-Type=application/json" \
  --body @/tmp/deploy-and-validate-run.json
```

Notes:

- Identifiers are case-sensitive. If step 3 surfaced a different set, substitute them here — the Zip default is `["ut", "ut_2", "pd", "GitHubRelease_"]`.
- Some pipelines do not have a BDD stage. Do not add one — just proceed and record "no BDD stage for this pipeline" in the final report.
- Capture `id` (runId) and `_links.web.href` from the response. Show the user the run URL immediately so they can watch too.

### 6. Poll the run

Poll every 30–60 seconds until the run reaches a terminal state. Do not sleep less than 60 seconds — the deploy takes 20–40 minutes and short-sleep polling wastes context.

```bash
az rest --method GET \
  --resource "499b84ac-1321-427f-aa17-267ca6975798" \
  --uri "https://dev.azure.com/quadpay/quadpay-services/_apis/pipelines/<definitionId>/runs/<runId>?api-version=7.0" \
  --query "{state:state, result:result, finishedDate:finishedDate}" -o json
```

Prefer the Monitor tool with a poll loop that emits one line per stage-transition and one line at completion — the harness wakes you on each notification without you having to re-poll:

```bash
prev=""
while true; do
  timeline=$(az rest --method GET \
    --resource "499b84ac-1321-427f-aa17-267ca6975798" \
    --uri "https://dev.azure.com/quadpay/quadpay-services/_apis/build/builds/<runId>/timeline?api-version=7.0" \
    --query "records[?type=='Stage'].{identifier:identifier, state:state, result:result}" -o json 2>/dev/null || echo "[]")
  runstate=$(az rest --method GET \
    --resource "499b84ac-1321-427f-aa17-267ca6975798" \
    --uri "https://dev.azure.com/quadpay/quadpay-services/_apis/pipelines/<definitionId>/runs/<runId>?api-version=7.0" \
    --query "{state:state, result:result}" -o json 2>/dev/null || echo "{}")
  now=$(echo "$timeline" | python3 -c "import json,sys; d=json.load(sys.stdin); print('|'.join(sorted([f\"{s['identifier']}:{s.get('state','?')}:{s.get('result','?')}\" for s in d if s.get('result') != 'skipped'])))" 2>/dev/null || echo "")
  if [ "$now" != "$prev" ]; then
    echo "[$(date -u +%H:%M:%S)] stages: $now"
    prev="$now"
  fi
  state=$(echo "$runstate" | python3 -c "import json,sys; d=json.load(sys.stdin); print(d.get('state',''))" 2>/dev/null || echo "")
  if [ "$state" = "completed" ]; then
    echo "[$(date -u +%H:%M:%S)] RUN COMPLETED"
    break
  fi
  sleep 60
done
```

Report a short status update at each stage transition (`Build, test solution`, `Deploy ci`, `BDD - <service> (CI)`, `Check ephemeral`). Do not narrate every poll.

### 7. Pull stage-level results

When the run is terminal, retrieve per-stage timing and result:

```bash
az rest --method GET \
  --resource "499b84ac-1321-427f-aa17-267ca6975798" \
  --uri "https://dev.azure.com/quadpay/quadpay-services/_apis/build/builds/<runId>/timeline?api-version=7.0" \
  --query "records[?type=='Stage'].{name:name, identifier:identifier, result:result, start:startTime, end:finishTime}" -o json
```

Record:

- `deployCompletedAt`: finish time of `Deploy ci`.
- `bddStartedAt` / `bddCompletedAt`: window for `BDD - <service> (CI)` when present.
- `bddResult`: succeeded / failed / partiallySucceeded / no-BDD-stage.
- Any failed stage other than BDD — capture and surface it, but the Dynatrace analysis still runs because there may already be usable telemetry.

### 8. BDD failure triage (skip if no BDD stage or BDD succeeded)

For each failed BDD task:

```bash
az pipelines runs show --id <runId> \
  --organization https://dev.azure.com/quadpay \
  --project quadpay-services --open   # or use the log-download endpoints if you prefer non-interactive
```

Or fetch logs directly:

```bash
az rest --method GET \
  --resource "499b84ac-1321-427f-aa17-267ca6975798" \
  --uri "https://dev.azure.com/quadpay/quadpay-services/_apis/build/builds/<runId>/logs?api-version=7.0"
```

Extract:

- failing scenario / feature file names
- top of each stack trace (assertion vs. exception vs. timeout)
- HTTP status of failing calls if the BDD is API-level

Cross-reference each failing scenario against the PR diff (files, changed classes, endpoints). Classify each failure as:

- `related`: failing step touches code or configuration changed by this PR
- `unrelated`: failure is in an unrelated feature or is a known-flaky scenario
- `unclear`: not enough signal without deeper investigation

Always cite the specific PR file(s) or diff hunk(s) that motivate a `related` classification.

### 9. Dynatrace post-deploy check for the deployed service

Anchor the post-window on `deployCompletedAt`. The observation window is `[bddStartedAt, bddCompletedAt + 2 minutes]` when BDD ran, otherwise `[deployCompletedAt, deployCompletedAt + 5 minutes]` — the user's guidance is that BDD produces immediate activity, so a short trailing buffer is enough.

Filter every query by:

- cluster: `zip-aks-cluster-dev-green` (via the environment binding for the MCP tool — this cluster is the dev tenant)
- `dt.entity.service` — use **both** entity ids from step 4 (`.NET-agent` + `OTel`) via `in(dt.entity.service, array(...))`. The same pod produces telemetry to both — dropping one means missing half the coverage.
- object-dependent version filter:
  - **OTel-view spans / exceptions**: filter by `service.version == "<build number>"` (e.g. `20260903.5`). This is the strongest signal that a failure belongs to _this deploy_ vs. ambient noise on shared endpoints.
  - **.NET-agent-view spans / exceptions**: `service.version` is **not populated** on this entity at Zip. Fall back to time-window + entity filter only. Same for logs (logs never carry `service.version`).
  - **logs**: time-window filtering is the default across both entity views. When you need to prove a log line came from _this_ deploy specifically, correlate via `trace_id` — pull the trace ids from step 9.3's version-filtered OTel spans, then `filter trace_id in [...]` (or the Zip-specific field `p.trace_id`) on the log query. Do not claim a log line is from this deploy solely because it fell in the post-window.

Run these in this order:

1. `list_problems` scoped to both entities and the post-window. Any new problems opened after `deployCompletedAt` are highest signal.
2. `list_exceptions` scoped to both entities and the post-window. Apply the `service.version` filter only when the exception rows come from the OTel entity.
3. Top failing operations DQL (see `dynatrace-query-patterns.md#top-failing-operations`) run twice: once version-filtered on the OTel entity (highest confidence), once time-window-only on the .NET-agent entity (broader coverage). Collect trace ids from the OTel side for step 9.4's log correlation.
4. Error / warning log search for both entities in the post-window. Skip Info / Debug (per user memory). Default: time-window only. If step 9.2 or 9.3 surfaced failing traces, re-run this filtered by those `trace_id`s (`p.trace_id` on Zip log records) to prove which log lines belong to this deploy.
5. Kubernetes events check for the service's workloads in the post-window. Prefer DQL over `get_kubernetes_events` for tight scoping — Zip runs multiple workloads per logical service (e.g. `<service>`, `<service>-primary`, `<service>-qp-master`). Query pattern:

   ```text
   fetch events, from: <post-window-start>, to: <post-window-end>
   | filter k8s.cluster.name == "zip-aks-cluster-dev-green"
   | filter contains(k8s.workload.name, "<service>") or contains(k8s.pod.name, "<service>")
   | filter event.kind == "K8S_EVENT"
   | fields timestamp, k8s.workload.name, k8s.pod.name, event.type, event.name, event.description
   ```

   Catches `CrashLoopBackOff`, `OOMKilled`, `ImagePullBackOff`, readiness-probe failures that never make it to a Dynatrace problem. Only `DAVIS_EVENT` / `CUSTOM_INFO` results = pods are healthy.

For each finding, capture: timestamp, entity, operation or span name, exception type / status code, and count.

### 10. Impacted services via topology

Use Dynatrace call topology (Smartscape) to identify neighbors:

- upstream (services that call the deployed service)
- downstream (services the deployed service calls)

For each neighbor in the post-window, re-run steps 9.1–9.3 (problems, exceptions, top failing operations) scoped to that neighbor. Do not filter neighbors by version — the change is not deployed there — but do keep the same time window.

Keep the neighbor list tight (typically 2–5 first-hop entities). If Smartscape returns dozens, cap to the ones with actual call volume to/from the deployed service in the pre-window baseline.

### 11. Relatedness judgment

For every error, exception, K8s event, or neighbor regression, classify it as:

- `related`: stack trace touches a file in the PR diff, exception type matches a code path changed, failing endpoint is one the PR modified, config value in the diff explains the failure, or the deployed version tag appears in the failing spans / logs.
- `unrelated / pre-existing`: the same error exists in the pre-window baseline at similar rates, or the failing path is untouched by the diff.
- `unclear`: signal is real but the connection to the diff is not provable from the evidence collected.

Cite the diff line, span, or exception message that supports each `related` call. Vague relatedness claims are worse than none.

### 12. Verdict

Emit one of:

- `safe to merge` — deploy + BDD green, no new problems or exceptions on the deployed service or its neighbors, K8s events clean.
- `safe to merge with caveats` — everything passed but there is at least one `unrelated` or `unclear` finding worth noting.
- `needs investigation` — at least one `related` finding but not a full block (e.g. an exception spike on a non-critical path).
- `blocked` — BDD failed with a `related` classification, `Deploy ci` failed, pods failing to start, or a `related` problem opened on the deployed service.

Never move from `blocked` or `needs investigation` to `safe` by re-running queries with wider filters. If a finding is real, it stays in the report.

## Output shape

Give the user a single structured report. Lead with the verdict, then the evidence.

```
Verdict: <one of the four>

Pipeline
- run: <URL>
- stages run / skipped: <list>
- per-stage results: <name → result → duration>
- Deploy ci completed at: <ISO timestamp>

BDD
- result: <pass / fail / partial / none>
- failing scenarios (if any):
  - <name>: <related | unrelated | unclear> — <one-line reason with file cite>

Dynatrace — deployed service (<SERVICE-ID>, version <build#>)
- new problems in window: <n> — <list>
- exceptions: <n> — <top types>
- top failing operations: <endpoint → count>
- k8s pod events: <clean | list>

Dynatrace — impacted neighbors
- upstream: <service → findings>
- downstream: <service → findings>

Diff context
- files touched: <n> across <top dirs>
- endpoints changed: <list if applicable>

Related findings (drive the verdict)
- <finding>: <cite from diff or telemetry>

Unrelated / unclear findings (informational)
- <finding>: <cite>

Windows searched
- baseline: <pre-window>
- post-deploy: <post-window>
- cluster: zip-aks-cluster-dev-green
```

Then stop. Do not offer to merge, mark ready-for-review, or trigger a follow-up deploy without the user asking.

## Guardrails

- Never trigger a pipeline against `main` or a release branch from this skill — this skill is for feature branches / PR branches only. If the resolved branch is a protected branch, stop and ask the user.
- Never skip `Check ephemeral` or `Deploy ci` — those are the stages that produce the telemetry the skill is checking. Only the downstream promotion stages are skippable.
- Never widen the Dynatrace cluster scope. `zip-aks-cluster-dev-green` is the entire allowed surface.
- Never re-run a pipeline automatically on failure. Report the failure and let the user decide.
- If `az account show` fails midway (token expired), stop and ask the user to re-login. Do not attempt device-code login from inside the skill.
- If polling has been running for more than 60 minutes, stop and check whether the pipeline is genuinely stuck (agent starvation, external dependency). Do not sit in a poll loop indefinitely.
