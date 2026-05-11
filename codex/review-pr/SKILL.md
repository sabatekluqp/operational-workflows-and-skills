---
name: review-pr
description: "Review a GitHub pull request from the outside when the user wants an independent code review. Build requirement context from the PR, Jira, and relevant design docs, inspect the current diff and review state, and produce findings about bugs, requirement mismatches, regressions, rollout risk, and test gaps. Default to discussing findings in session unless the user explicitly asks to comment on the PR."
---

# Review PR

Read [../../workflows/review-pr.md](../../workflows/review-pr.md) before starting.
Read [../../references/token-budget.md](../../references/token-budget.md) when you need the shared token-cost model.

Token budget summary:
- Claude: `20%` session history + artifacts, `15%` workflow/reference/template, `15%` external context, `15%` tool output, `12%` coding, `10%` runtime evidence, `8%` planning, `5%` wrapper
- Codex: `28%` coding, `24%` build/test/verification, `14%` workflow/reference/template, `14%` tool output, `12%` session history + artifacts, `10%` external context, `8%` planning, `4%` wrapper

Follow the shared workflow:

- identify the PR, classify its rollout role, and build the requirement context before judging the code
- review the current diff against base and the current review state, treating existing comments as hypotheses to verify
- separate real behavior findings from gate-only or process-only observations
- produce findings in session by default
- do not comment on the PR or make code changes unless the user explicitly asks
