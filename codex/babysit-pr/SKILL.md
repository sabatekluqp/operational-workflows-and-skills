---
name: babysit-pr
description: "Babysit a GitHub pull request you own after review feedback lands. Use for PR-author follow-up: triage review comments against the current diff, decide which comments need code changes versus explanation, make safe revisions, run focused verification, push updates, and reply to addressed threads."
---

# Babysit PR

Read [../../workflows/babysit-pr.md](../../workflows/babysit-pr.md) before starting.
Read [../../references/token-budget.md](../../references/token-budget.md) when you need the shared token-cost model.

Token budget summary:
- Claude: `20%` session history + artifacts, `15%` workflow/reference/template, `15%` external context, `15%` tool output, `12%` coding, `10%` runtime evidence, `8%` planning, `5%` wrapper
- Codex: `28%` coding, `24%` build/test/verification, `14%` workflow/reference/template, `14%` tool output, `12%` session history + artifacts, `10%` external context, `8%` planning, `4%` wrapper

Follow the shared workflow:

- identify the PR and current review state first
- triage review comments against the current code and diff
- make safe revisions, push updates, and reply to each addressed comment unless the user asked for draft-only handling
- review the current diff against base before taking a position
- sign every PR reply exactly as `saba's little friend`
