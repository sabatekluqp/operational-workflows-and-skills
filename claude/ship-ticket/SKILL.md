---
name: ship-ticket
description: "Take a Jira ticket from first read to a reviewed draft PR. Use when the user gives a Jira key or URL and wants the work started end to end: gather ticket and repo context, check for overlapping in-flight branches, confirm the implementation and test plan, code and verify the change, open a draft PR, run an independent review pass, apply accepted feedback, and hand the draft back to the user. Never mark the PR ready for review autonomously."
---

# Ship Ticket

Read [../../workflows/ship-ticket.md](../../workflows/ship-ticket.md) before starting.
Read [../../references/token-budget.md](../../references/token-budget.md) when you need the shared token-cost model.

Token budget summary:
- Claude: `20%` session history + artifacts, `15%` workflow/reference/template, `15%` external context, `15%` tool output, `12%` coding, `10%` runtime evidence, `8%` planning, `5%` wrapper
- Codex: `28%` coding, `24%` build/test/verification, `14%` workflow/reference/template, `14%` tool output, `12%` session history + artifacts, `10%` external context, `8%` planning, `4%` wrapper

Follow the shared workflow:

- resolve the ticket, read linked/blocking issues, and classify the work before touching code
- scan the target repo for in-flight branches (DQ-XXXX-* / feature/TICKET-*) that overlap scope and surface collisions to the user
- propose a plan and confirm it with the user before coding — include unknowns and the "don't stack on unmerged PRs unless asked" default
- include a concrete test plan in that proposal — for each acceptance criterion, name the test(s) (Given/When/Should subject), call out new test files vs. extensions, and surface any criterion not covered by unit tests (and what covers it instead)
- match the repo's existing branch naming and commit message style from `git log` / `git branch -r`
- honor the user's memories (test conventions, logging, ticket style) when writing code, tests, and PR bodies
- run tests locally before every commit; never push red
- open the PR as a **draft** and spawn an independent reviewer agent (general-purpose or code-reviewer subagent) to critique it
- present review findings to the user, confirm scope of fixes, then iterate
- hand the draft PR back to the user — never click "ready for review" yourself
