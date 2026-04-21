---
name: review-pr
description: "Review a GitHub pull request from the outside when the user gives a PR number or link and wants an independent code review. Use for reviewer tasks: identifying the PR scope and rollout role, reading the PR body, Jira story, parent or epic when relevant, linked or discoverable Confluence design docs, current review threads, inspecting the current diff against base, and producing findings about bugs, requirement mismatches, regressions, test gaps, and gate-only noise. Default to discussing findings in session unless the user explicitly asks to comment on the PR."
---

# Review PR

Read [../../workflows/review-pr.md](../../workflows/review-pr.md) before starting.

Follow the shared workflow:

- identify the PR, classify its rollout role, and build the requirement context from Jira, epic, and Confluence before judging the code
- review the current diff against base and the current review state, treating existing comments as hypotheses to verify
- separate real behavior findings from gate-only or process-only observations
- produce findings in session by default
- do not comment on the PR or make code changes unless the user explicitly asks
