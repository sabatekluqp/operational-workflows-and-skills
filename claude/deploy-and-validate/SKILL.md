---
name: deploy-and-validate
description: "Deploy a branch or PR to CI in Azure DevOps. First ensures the branch is current with base (runs gh pr update-branch when BEHIND, stops if DIRTY), then triggers Build → Docker → Deploy ci → BDD → Check ephemeral and polls to completion. Once green, validates in Dynatrace (dev cluster zip-aks-cluster-dev-green) — problems, exceptions, error traces, K8s events for the deployed service and its topology-derived neighbors, plus BDD failure triage. Judge which findings look related to the PR diff and emit a merge-readiness verdict. Use before merging a PR when you want dev-environment proof the change is safe."
---

# Deploy And Validate

Read [../../workflows/deploy-and-validate.md](../../workflows/deploy-and-validate.md) before starting.
Read [../../references/token-budget.md](../../references/token-budget.md) when you need the shared token-cost model.

Token budget summary:
- Claude: `20%` session history + artifacts, `15%` workflow/reference/template, `15%` external context, `15%` tool output, `12%` coding, `10%` runtime evidence, `8%` planning, `5%` wrapper
- Codex: `28%` coding, `24%` build/test/verification, `14%` workflow/reference/template, `14%` tool output, `12%` session history + artifacts, `10%` external context, `8%` planning, `4%` wrapper

Read [../../references/dynatrace-query-patterns.md](../../references/dynatrace-query-patterns.md) when you need starter DQL shapes for the post-deploy comparison window or exception-by-version queries.

Fixed inputs for this skill:
- Dynatrace cluster: `zip-aks-cluster-dev-green` (dev). Never target another environment.
- Azure DevOps org/project: `https://dev.azure.com/quadpay/quadpay-services`.
- Stages to run (per pipeline): `Build, test solution`, `Build, publish docker image`, `Check ephemeral`, `Deploy ci`, `BDD - <service> (CI)` when a BDD stage exists.
- Stages to skip: `Deploy ut`, `Deploy ut_2`, `Deploy pd`, `GitHub release` (and anything else downstream of these).

Never mark a PR ready for review, merge, or promote past `Deploy ci` from this skill. Hand the verdict back to the user.
