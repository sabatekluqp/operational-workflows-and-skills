---
name: ship-ticket
description: "Take a Jira ticket end-to-end into a reviewed draft PR. Use when the user points at a ticket (URL or key like DQ-9031) and wants to start the work: reading the ticket and related context, scanning for in-flight branches that affect scope, proposing a plan, cutting a feature branch off the latest default branch, implementing + testing, opening a draft PR, spawning an independent reviewer agent, applying accepted feedback, and handing the PR back to the user to mark ready. Always opens the PR as a draft; never marks it ready for review autonomously."
---

# Ship Ticket

Read [../../workflows/ship-ticket.md](../../workflows/ship-ticket.md) before starting.

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
