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

- resolve the ticket, read linked context, and classify the work before touching code
- scan for overlapping in-flight branches and surface collisions before coding
- confirm the implementation and test plan with the user before non-trivial coding
- match the repo's branch naming, commit style, and PR template
- run tests locally before every commit and never push red
- open the PR as a **draft**, run an independent review pass, apply accepted feedback, and hand the draft back to the user
- never mark the PR ready for review yourself
