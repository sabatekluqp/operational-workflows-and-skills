---
name: pr-diagram
description: "Write or rewrite a GitHub pull request description using a short summary and a focused Mermaid flowchart or sequence diagram. Use when the user wants a PR body that explains the changed flow, interaction sequence, rollout shape, or verification clearly without turning into a file-by-file changelog."
---

# PR Diagram

Read [../../workflows/pr-diagram.md](../../workflows/pr-diagram.md) before starting.

Follow the shared workflow:

- identify the PR or local diff, then reduce it to the one changed flow reviewers need to understand
- write a short summary plus one focused Mermaid flowchart or sequence diagram instead of diagramming the whole service
- preserve issue links, checklists, and automation blocks unless the user explicitly asks to rewrite them
- update the live PR body only when the user explicitly asks for the mutation
