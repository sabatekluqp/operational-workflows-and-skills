# Ship Ticket

Take a Jira ticket from first-read to a reviewed draft PR: gather context, propose a plan, branch off the latest default branch, implement with tests, commit+push, open a draft PR, spawn an independent reviewer, apply accepted feedback, and hand the draft back to the user to mark ready.

## Inputs

Accept any of:

- a Jira ticket key (e.g. `DQ-9031`)
- a Jira ticket URL (e.g. `https://quadpay.atlassian.net/browse/DQ-9031`)
- a short description that can be resolved to a ticket via Atlassian search

## Invariants

- **Draft PR by default.** Always open the PR in draft mode. Do not mark it ready for review. That is the user's call.
- **Confirm plan before coding.** After reading the ticket and scanning related branches, summarize the proposed approach and surface unknowns. Wait for user approval before writing code, unless the task is a trivial one-liner.
- **Don't stack on unmerged PRs unless asked.** If related in-flight branches exist, report them and prefer to build on `master`/`main`. Only stack on another branch if the user explicitly authorizes it.
- **Never push red.** Tests must pass locally before every commit. If tests fail, stop and report.
- **Match the repo's conventions.** Branch naming, commit message style, and PR template come from the repo — inspect `git log`, `git branch -r`, and `.github/pull_request_template.md` before writing your own.
- **Honor user memories.** Test conventions (Given/When/Should, Arrange/Act/Assert, strict fakes, SUT in ctor), logging preferences (errors/warnings only for new code), and ticket/PR style notes live in `MEMORY.md`. Apply them.

## Workflow

### 1. Resolve the ticket and build context

- Fetch the ticket via Atlassian MCP (`mcp__atlassian__getJiraIssue`). Read summary, description, acceptance criteria (may live in a custom field), parent epic, and linked issues (blocks / blocked by / implements).
- Note blockers — if a blocking ticket is still open, flag it before starting.
- If the ticket references code types, endpoints, or interfaces, use `Grep` / `Glob` to locate them in the current working repo.
- If the working directory is not already the target repo, ask the user which repo to work in rather than guessing.

### 2. Scan for in-flight branches

- Run `git fetch` then `git branch -r | grep -iE "<TICKET>|<related-ticket>|<keyword>"`.
- For each potentially related branch, list new/modified files vs `origin/master` (or `origin/main`).
- Look for existing scaffolding you can reuse (already on master) vs. work that's still on other branches.
- If a related branch already implements the ticket's route/handler/class — even partially — surface the overlap. Do not silently duplicate.

### 3. Propose the plan

Summarize:

- what the ticket actually asks for (acceptance criteria in plain prose)
- what's already on master that you can reuse
- what's on in-flight branches that overlaps (and your recommendation: stack, ignore, or coordinate)
- the files you plan to add/modify
- any design decisions that need the user's call (e.g. response shape, default behavior for empty input, error semantics)
- when the change has a non-trivial flow (new execution path, interaction sequence, or rollout shape), include a draft Mermaid diagram per `pr-diagram.md` so the user can sanity-check the flow before coding. Skip for trivial diffs (rename, one-liner, config-only).

Wait for the user to confirm or redirect before touching code. For trivial tickets (one-line fix, obvious rename), you can skip this gate and proceed, but still narrate the plan briefly.

### 4. Create the feature branch

- Read recent branches with `git branch -r | head -20` and match the naming convention — common patterns: `feature/TICKET-XXXX/short-description`, `TICKET-XXXX-short-description`, `USER/TICKET-XXXX`.
- Short description should be kebab-case and describe the deliverable, not the problem.
- Always branch off the latest default branch:
  ```
  git checkout master && git pull && git checkout -b <branch-name>
  ```
- Use `master` or `main` based on what the repo actually uses (check `git branch --show-current` on a clean clone or `git symbolic-ref refs/remotes/origin/HEAD`).

### 5. Implement

- Use `TaskCreate`/`TaskUpdate` to track sub-steps when the work is non-trivial (3+ distinct steps).
- Follow the user's test conventions: Given/When/Should naming, `// Arrange` / `// Act` / `// Assert` section comments in each test body, strict fakes that throw on unexpected calls, SUT instantiated in the constructor.
- Follow the user's logging preference: no Debug/Info logs in new code; only `LogError` / `LogWarning` for real error branches. If the only logger use would be Debug/Info, drop the `ILogger` dependency entirely.
- Keep commits cohesive. Don't bundle unrelated changes.

### 6. Build and test

- Run the relevant build and test commands for the language:
  - .NET: `dotnet test --nologo` on the affected test project
  - Node: `pnpm test` or `npm test` scoped to the package
  - Python: `pytest <path>`
- Do not commit until green. If tests fail, triage and fix before proceeding.

### 7. Commit

- Match the repo's commit style. For this user's DQ project the pattern is `TICKET-XXXX: short description` followed by a body that explains *why* (one paragraph is enough for most tickets).
- Stage specific files (`git add <file1> <file2>`), never `git add -A` or `git add .`.
- Use a heredoc for the commit body and include the standard Claude co-author line.
- Do not amend. Create new commits.

### 8. Push and open the draft PR

- Push with `-u` on first push: `git push -u origin <branch-name>`.
- Open the PR as a draft: `gh pr create --draft --title "..." --body "$(cat <<'EOF' ... EOF)"`.
- Populate the body from the repo's PR template (`.github/pull_request_template.md`) if it exists. Fill the checklist items honestly. Include:
  - Description of what changed and why
  - Issue link back to the Jira ticket
  - Test plan (what's covered, what isn't)
  - Any coordination notes (e.g. route collisions with other in-flight branches)
  - A focused Mermaid diagram per `pr-diagram.md` when the PR changes an execution path, interaction sequence, or rollout shape. Reuse the diagram drafted in step 3 if there is one. Skip for trivial diffs.
- Return the PR URL to the user.

### 9. Spawn an independent reviewer agent

- Use the `Agent` tool with `general-purpose` (or `code-reviewer` if available).
- Brief the reviewer as a cold colleague. Include:
  - repo path, branch name, commit SHA
  - the ticket summary + acceptance criteria
  - the PR URL
  - specific risks to scrutinize (contract fidelity, route conflicts, edge cases, test gaps, DI ordering)
  - the list of files changed (so the agent doesn't have to spelunk)
  - any context the author relied on (e.g. "DTOs defined locally because the real project has heavy transitive deps")
  - the user's test/logging conventions so the reviewer can check adherence
- Ask for the report categorized as **Blocking / Should-fix / Nits / Looks good**, with concrete file:line citations, capped at ~600 words.
- Do **not** ask the reviewer to also fix the issues — reviewer produces findings; you apply them.

### 10. Apply accepted feedback

- Present the reviewer's findings to the user and confirm which to act on. Be explicit about which you plan to skip and why (e.g. reviewer flagged a defensive pattern that's intentional).
- Apply fixes as a follow-up commit on the same branch. Keep the follow-up commit's message explicit: `TICKET-XXXX: address review feedback`.
- Re-run tests before pushing. Push.

### 11. Hand off

- Summarize what changed in the follow-up commit(s), mapped to the reviewer's categories.
- Confirm the PR is still a **draft**.
- Tell the user it's ready for them to mark ready-for-review when they're satisfied.
- Do not click "ready for review" yourself. Do not request reviewers yourself. Those are the user's call.

## Guardrails

- Never force-push, reset --hard, or amend published commits without explicit user instruction.
- Never skip hooks (`--no-verify`, `--no-gpg-sign`) without explicit user instruction.
- Never commit files likely to contain secrets (`.env`, credentials, keys).
- Do not rebase onto an unmerged branch unless the user explicitly authorized stacking.
- If Sonar / CI flags issues after the first push, treat them as another review round — surface them to the user and confirm scope before fixing.
- If the ticket's acceptance criteria are ambiguous, ask the user rather than inventing a contract.

## Command Patterns

Prefer these commands:

- Jira: `mcp__atlassian__getJiraIssue(cloudId=..., issueIdOrKey=..., responseContentFormat="markdown")`
- Branch discovery: `git fetch && git branch -r | grep -iE "<keyword>"`
- Cross-branch inspection without checkout: `git log --oneline origin/master..origin/<branch>` and `git show origin/<branch>:<path>`
- PR creation: `gh pr create --draft --title "..." --body "$(cat <<'EOF' ... EOF)"`
- Reviewer agent: `Agent` tool with `subagent_type: general-purpose`, briefed with ticket + branch + file list + specific risks
