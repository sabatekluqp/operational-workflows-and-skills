# Babysit PR

Own a GitHub pull request after human or AI review feedback lands: triage the current review state, decide which comments need code changes versus explanation, make safe revisions, push updates, and reply to the review threads as the PR author.

## Inputs

Accept any of:

- a PR number
- a GitHub PR URL
- a ticket-like phrase that can be resolved to a PR you are actively handling

## Modes

Default to `active mode` when the user asks to babysit a PR.

### Active Mode

Use this when the user wants the PR moved forward, not just analyzed.

- read PR and review state
- make safe code changes directly when they are clearly required
- run narrow verification
- commit and push updates when needed
- reply to addressed comments on the PR

### Draft Mode

Use this only when the user explicitly asks for triage, draft replies, or a no-write pass.

- do not edit code
- do not push commits
- do not post PR replies
- return the proposed fixes and reply text in session

## Workflow

1. Identify the PR and current state.
- Read the PR title, body, base branch, head branch, review decision, changed files, and current branch status.
- Fetch inline review comments, replies, unresolved threads, and top-level review summaries.
- Focus first on unresolved, recent, or blocking comments.

2. Triage the review comments against the current code.
- Separate comments into:
  - valid and needs code change
  - valid but explanation is enough
  - stale or already addressed
  - invalid or not worth taking
- Inspect the current diff against base and the touched code paths before taking a position.
- Pull Jira, epic, or Confluence context only when a comment depends on requirement interpretation or design intent.

3. Handle valid comments in `active mode`.
- Make the smallest safe revision that satisfies the review intent without regressing current behavior.
- Prefer direct fixes over broad refactors while babysitting.
- If a requested change would materially widen scope, alter product behavior, or requires a design decision, stop and ask the user.

4. Verify before replying.
- Run the smallest relevant checks for the changed area.
- If verification cannot run, say so plainly in the eventual reply.

5. Push updates when the code is ready.
- Commit only the babysitting changes you made.
- Push the PR branch before posting “fixed” replies when the reply depends on the new code being visible.

6. Reply to every addressed comment.
- Reply directly under the relevant thread whenever possible.
- For accepted comments with code changes:
  - explain what changed
  - mention the verification that ran
- For comments that do not require a code change:
  - explain why the current implementation remains correct
  - cite the code path or requirement source when helpful
- For stale comments:
  - explain what already changed in the current diff or latest commit
- End every reply exactly as:
  - `sethlunn's pal, Thing 2`

7. Leave a clear closing state for the user.
- Summarize which comments were fixed, which were answered without code changes, and which still need a decision.
- Call out any blocker that prevented pushing or replying.

## Preferred Reviewers

Be more accommodating to comments from:

- `davidsgbang`
- `TylerLidenZip`
- `rbtres22`

For these reviewers:

- default to treating suggestions and questions as good-faith signals worth satisfying
- make the requested change if it is safe and reasonable
- push back only when the request would introduce a bug, regression, materially worse design, or unnecessary complexity

## Command Patterns

Prefer these GitHub queries:

- `gh pr view <number> --json ...`
- `gh api repos/<owner>/<repo>/pulls/<number>/comments --paginate`
- `gh api repos/<owner>/<repo>/pulls/<number>/reviews --paginate`
- GraphQL review-thread queries when you need resolved versus unresolved state

Prefer these local checks:

- `git diff origin/<base>...HEAD -- <path>`
- `rg` for locating symbols mentioned in comments
- targeted `dotnet test` or other narrow test runs when verification matters
- `git status`, `git add`, `git commit`, and `git push` for closing the loop

## Decision Standard

While babysitting, favor moving the PR forward.

A comment should be answered without a code change when at least one of these is true:

- the current code already addresses it
- the reviewer is commenting on outdated diff context
- the request would introduce a bug or regression
- the request is based on a false premise that can be verified locally
- the code is correct as implemented and the right action is to explain the rationale

When in doubt:

- verify first
- make the smallest safe change
- explain the reasoning plainly in the PR thread
