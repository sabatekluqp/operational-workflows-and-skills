---
name: pr-diagram
description: "Write or rewrite a GitHub pull request description with a short summary and one focused Mermaid diagram. Use when reviewers need the changed flow, interaction order, or rollout shape explained clearly without a file-by-file changelog."
---

# PR Diagram

Read [../../workflows/pr-diagram.md](../../workflows/pr-diagram.md) before starting.
Read [../../references/token-budget.md](../../references/token-budget.md) when you need the shared token-cost model.

Token budget summary:
- Claude: `20%` session history + artifacts, `15%` workflow/reference/template, `15%` external context, `15%` tool output, `12%` coding, `10%` runtime evidence, `8%` planning, `5%` wrapper
- Codex: `28%` coding, `24%` build/test/verification, `14%` workflow/reference/template, `14%` tool output, `12%` session history + artifacts, `10%` external context, `8%` planning, `4%` wrapper

Follow the shared workflow:

- identify the PR or local diff, then reduce it to the one changed flow reviewers need to understand
- write a short summary plus one focused Mermaid flowchart or sequence diagram instead of diagramming the whole service
- preserve issue links, checklists, and automation blocks unless the user explicitly asks to rewrite them
- update the live PR body only when the user explicitly asks for the mutation
