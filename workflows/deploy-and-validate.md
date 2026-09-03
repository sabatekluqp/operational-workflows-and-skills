# Deploy And Validate

Use this workflow to deploy a branch or PR to CI in Azure DevOps, wait for the run (including BDD) to complete, then check the deployed service and its topology-derived neighbors in Dynatrace for errors that look related to the change. Emit a merge-readiness verdict.

Read `../references/dynatrace-query-patterns.md` when you need starter DQL. This workflow ships its own filter shape for the "same service, same build version, post-deploy window" case, which is the primary lens here.

## Fixed Environment

- Dynatrace cluster: `zip-aks-cluster-dev-green`. Do not query prod or other non-prod clusters from this skill.
- Azure DevOps: `https://dev.azure.com/quadpay/quadpay-services`.
- Runtime shape of the pipelines: `.NET service` → `Build, test solution` → (`Check ephemeral` in parallel) → `Build, publish docker image` → `Deploy ci` → `BDD - <service> (CI)`. Downstream stages (`Deploy ut`, `Deploy ut_2`, `Deploy pd`, `GitHub release`) are always skipped here.

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

Also fetch the pipeline definition once so stage names are ground truth for step 5:

```bash
az rest --method GET \
  --uri "https://dev.azure.com/quadpay/quadpay-services/_apis/build/definitions/<definitionId>?api-version=7.0" \
  --query "process.yamlFilename" -o tsv
```

### 4. Baseline snapshot (pre-deploy)

Establish a comparison anchor before the pipeline runs. Dev traffic is usually sparse and BDD-driven, so this is often qualitative — say so explicitly if the pre-window has almost no data.

- Resolve the service entity: `find_entity_by_name` scoped to cluster `zip-aks-cluster-dev-green`. Record the `SERVICE-*` id.
- Grab the last-30-minutes counts for that service on the dev cluster: failure count, exception count, top error operations. Use `list_problems` for that entity and window as well.
- Send a deploy-start marker so the timeline is annotated for future investigations:
  - `send_event` with a distinctive title (e.g. `deploy-and-validate: <service> <branch> start`) and the service entity attached.

Write down the pre-deploy timestamp — you will use it as the anchor for both the post-deploy comparison window and the topology-neighbor scan.

### 5. Trigger the pipeline with the right stages

`az pipelines run` uses the older builds API and does not accept `stagesToSkip`. Use the Runs API through `az rest` so the exact stages from the screenshot are honored:

```bash
az rest --method POST \
  --uri "https://dev.azure.com/quadpay/quadpay-services/_apis/pipelines/<definitionId>/runs?api-version=7.0" \
  --headers "Content-Type=application/json" \
  --body '{
    "stagesToSkip": ["Deploy ut", "Deploy ut_2", "Deploy pd", "GitHub release"],
    "resources": {
      "repositories": {
        "self": { "refName": "refs/heads/<branch>" }
      }
    }
  }'
```

Notes:

- `stagesToSkip` names must match the pipeline exactly (case, spacing, punctuation). Pull them from the definition YAML or the last successful run rather than guessing.
- Some pipelines do not have a BDD stage. Do not add one — just proceed and record "no BDD stage for this pipeline" in the final report.
- Capture `id` (runId) and `_links.web.href` from the response. Show the user the run URL immediately so they can watch too.

### 6. Poll the run

Poll every 30–60 seconds until the run reaches a terminal state. Do not sleep less than 60 seconds — the deploy takes 20–40 minutes and short-sleep polling wastes context.

```bash
az rest --method GET \
  --uri "https://dev.azure.com/quadpay/quadpay-services/_apis/pipelines/<definitionId>/runs/<runId>?api-version=7.0" \
  --query "{state:state, result:result, finishedDate:finishedDate}" -o json
```

Prefer the Monitor tool with an `until` loop when you want the harness to wake you only on completion:

```bash
until [ "$(az rest --method GET --uri '.../runs/<runId>?api-version=7.0' --query state -o tsv)" = "completed" ]; do sleep 60; done
```

Report a short status update at each stage transition (`Build, test solution`, `Deploy ci`, `BDD - <service> (CI)`, `Check ephemeral`). Do not narrate every poll.

### 7. Pull stage-level results

When the run is terminal, retrieve per-stage timing and result:

```bash
az rest --method GET \
  --uri "https://dev.azure.com/quadpay/quadpay-services/_apis/build/builds/<runId>/timeline?api-version=7.0" \
  --query "records[?type=='Stage'].{name:name, result:result, start:startTime, end:finishTime}" -o json
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
- `dt.entity.service` == the entity id resolved in step 4
- object-dependent version filter:
  - **spans / exceptions**: filter by `service.version` (or `dt.service.version`) equal to the build number from step 7. This is the primary way to isolate _this deploy_'s telemetry from ambient noise on shared endpoints.
  - **logs**: `service.version` is not populated on log records at Zip. Time-window filtering is the default. When you need to prove a log line came from _this_ deploy specifically, correlate via `trace_id` — pull the trace ids from step 9.3's version-filtered spans, then `filter trace_id in [...]` (or the Zip-specific field `p.trace_id`) on the log query. Do not claim a log line is from this deploy solely because it fell in the post-window.

Run these in this order:

1. `list_problems` scoped to the service and the post-window. Any new problems opened after `deployCompletedAt` are highest signal.
2. `list_exceptions` scoped to the service and post-window, filtered by `service.version`.
3. Top failing operations DQL (see `dynatrace-query-patterns.md#top-failing-operations`), version-filtered on spans — this is where you find the concrete endpoint that broke, and the trace ids you collect here become the anchor for step 9.4's log correlation.
4. Error / warning log search for the service in the post-window. Skip Info / Debug (per user memory). Default: time-window only. If step 9.2 or 9.3 surfaced failing traces, re-run this filtered by those `trace_id`s (`p.trace_id` on Zip log records) to prove which log lines belong to this deploy.
5. `get_kubernetes_events` for the service's namespace and pods in the post-window — catches `CrashLoopBackOff`, `OOMKilled`, `ImagePullBackOff`, readiness-probe failures that never make it to a Dynatrace problem.

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
