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
- **Strict fakes, no exceptions.** Every new `A.Fake<T>(...)` in a unit test must pass `o => o.Strict()`. This is the established pattern across the repo and `davidsgbang` enforces it in review. If a strict fake forces you to arrange a call you'd rather leave defaulted, that's the signal — arrange it explicitly or rethink the dependency. Do not ship loose fakes in new test files.
- **Keep raw research out of main context.** Prefer a subagent for ticket fetch, branch scan, and initial code discovery; keep only a compact brief in the main session unless the task is trivial.
- **Write session artifacts for multi-turn work.** When the task will continue later or switch tools, use `../references/session-artifacts.md` and refresh `checkpoint.md`, `handoff.md`, and `resume-prompt.md` in the working directory.

## Workflow

### 1. Gather context with a brief-first pass

- If the repo is unclear, ask the user which repo to use before proceeding.
- Prefer a subagent for the first pass when the ticket is non-trivial. Its job is to fetch the ticket, scan overlapping branches, and identify likely code paths.
- Ask the subagent to return only a compact brief:
  - ticket summary and acceptance criteria
  - blocking or linked issues that affect scope
  - overlapping in-flight branches with a recommendation to stack, ignore, or coordinate
  - files of interest with one-line reasons
  - open questions for the user
- Do not carry raw Jira payloads, full branch diffs, or full file contents back into the main session.
- After the brief lands, read only the files needed for planning or implementation.
- Skip delegation for truly trivial work.

### 2. Keep branch scan folded into the first pass

Branch discovery and related-code grep belong in the same brief-first pass. Do not pre-read everything locally just to duplicate that work.

### 3. Propose the plan before coding

Summarize:

- what the ticket requires in plain prose
- what can be reused from the current base branch
- overlapping in-flight branches and your recommendation
- production files you expect to change
- the concrete test plan mapped to the acceptance criteria
- design decisions that still need a user call
- a draft Mermaid diagram only when the changed flow is non-trivial

Treat the test plan as part of the approval, not an afterthought. Wait for confirmation before coding unless the task is genuinely trivial.

### 4. Create the feature branch

- Match the repo's existing remote branch naming convention.
- Use a short kebab-case deliverable suffix.
- Branch from the latest default branch, using `main` or `master` based on the repo's actual default.

### 5. Implement

- Follow the user's test conventions: Given/When/Should naming, `// Arrange` / `// Act` / `// Assert` section comments in each test body, **strict fakes** (`A.Fake<T>(o => o.Strict())` — never bare `A.Fake<T>()`) that throw on unexpected calls, SUT instantiated in the constructor.
- Follow the user's logging preference: no Debug/Info logs in new code; only `LogError` / `LogWarning` for real error branches. If the only logger use would be Debug/Info, drop the `ILogger` dependency entirely.
- Keep commits cohesive. Don't bundle unrelated changes.

### 6. Build and test

- Run the narrowest relevant build and test commands for the changed area.
- Do not commit until green. If tests fail, triage and fix before proceeding.
- Before staging test files, scan for loose fakes you authored: `rg 'A\.Fake<[^>]+>\(\)' <changed test files>`. Any hit on a fake you introduced must be converted to `A.Fake<T>(o => o.Strict())` — repeat until that grep returns nothing for your new code.

### 7. Commit

- Match the repo's commit style. For this user's DQ project the pattern is `TICKET-XXXX: short description` followed by a body that explains *why* (one paragraph is enough for most tickets).
- Stage specific files (`git add <file1> <file2>`), never `git add -A` or `git add .`.
- Do not amend. Create new commits.

### 8. Push and open the draft PR

- Push with `-u` on first push: `git push -u origin <branch-name>`.
- Open the PR as a draft and use the repo's PR template if one exists.
- Include what changed, why, the Jira link, verification performed, remaining gaps, and any coordination notes.
- Include a focused Mermaid diagram only when the PR changes an execution path, interaction sequence, or rollout shape.
- Return the PR URL to the user.

### 9. Spawn an independent reviewer agent

- Use an independent reviewer agent.
- Brief it with the repo path, branch, ticket summary, acceptance criteria, PR URL, changed files, notable design context, and specific risks to scrutinize.
- Ask for findings grouped as **Blocking / Should-fix / Nits / Looks good** with concrete file:line citations.
- The reviewer reports findings only; you decide which accepted fixes to apply.

### 10. Apply accepted feedback

- Present the reviewer's findings to the user and confirm which to act on. Be explicit about which you plan to skip and why (e.g. reviewer flagged a defensive pattern that's intentional).
- Apply fixes as a follow-up commit on the same branch. Keep the follow-up commit's message explicit: `TICKET-XXXX: address review feedback`.
- Re-run tests before pushing. Push.

### 11. Hand off

- Summarize what changed in the follow-up commit(s), mapped to the reviewer's categories.
- Confirm the PR is still a **draft**.
- Tell the user it's ready for them to mark ready-for-review when they're satisfied.
- Do not click "ready for review" yourself. Do not request reviewers yourself. Those are the user's call.
- If the work is pausing or moving to another session/tool, refresh local session artifacts before stopping.

## Guardrails

- Never force-push, reset --hard, or amend published commits without explicit user instruction.
- Never skip hooks (`--no-verify`, `--no-gpg-sign`) without explicit user instruction.
- Never commit files likely to contain secrets (`.env`, credentials, keys).
- Do not rebase onto an unmerged branch unless the user explicitly authorized stacking.
- If Sonar / CI flags issues after the first push, treat them as another review round — surface them to the user and confirm scope before fixing.
- If the ticket's acceptance criteria are ambiguous, ask the user rather than inventing a contract.

## Command Patterns

Prefer these commands:

- Context gathering: a subagent that returns a structured brief instead of raw Jira, branch, and file payloads
- Jira (inside the Explore subagent): `mcp__atlassian__getJiraIssue(cloudId=..., issueIdOrKey=..., responseContentFormat="markdown")`
- Branch discovery (inside the Explore subagent): `git fetch && git branch -r | grep -iE "<keyword>"`
- Cross-branch inspection without checkout (inside the Explore subagent): `git log --oneline origin/master..origin/<branch>` and `git show origin/<branch>:<path>`
- PR creation: `gh pr create --draft ...`
- Reviewer agent: a general-purpose reviewer seeded with ticket context, branch, file list, and specific risks
