# Review PR

Review a GitHub pull request as an external reviewer: gather the requirement context from Jira, epic, and Confluence, inspect the current diff, consider the current PR review state, and produce defensible findings about correctness, scope alignment, rollout sequencing, regressions, and testing gaps.

Read `../references/pr-review-context-gathering.md` before building context.

## Inputs

Accept any of:

- a PR number
- a GitHub PR URL
- a ticket-like phrase that can be resolved to a PR

## Defaults

Default to `session-only review`.

- do not modify code
- do not push commits
- do not comment on the PR
- do not resolve review threads

Only move into PR-writing or code-changing behavior if the user explicitly asks for that follow-up.

## Workflow

1. Identify the PR.
- Read the PR title, body, base branch, head branch, changed files, and linked issue references.
- Read enough of the PR description to understand claimed scope and rollout intent.
- Classify the rollout role before judging the code:
  - `producer-only`
  - `consumer-only`
  - `stacked companion`
  - `end-to-end`

2. Build the requirement context pack.
- Resolve and read the Jira story.
- Read the parent or epic when it changes scope interpretation, sequencing, or architectural intent.
- Read directly linked Confluence pages first.
- Search Confluence only when the intended behavior is still unclear after reading the story and PR body.
- Reduce the result to the effective contract:
  - required behavior
  - explicit non-goals
  - important design constraints
  - follow-up dependencies or companion PR assumptions
  - rollout constraints that affect whether a gap is a defect or intentional sequencing
  - customer-facing or legal output paths that must be checked end to end when relevant

3. Build the current review context.
- Read inline review comments, replies, unresolved threads, and top-level review summaries.
- Treat existing comments as signals, not as the source of truth.
- Verify whether earlier comments still apply to the current branch tip before repeating them.

4. Review the current code.
- Inspect the current diff against base.
- Read the touched code paths and any adjacent code needed to evaluate behavior.
- Compare the implementation against the requirement context pack and the current comments.
- Use narrow local verification when a bug, regression, or missing coverage claim needs proof.
- For customer-facing, compliance, or document-generation changes, inspect the final rendered-text or final document-selection path, not only the data plumbing.

5. Produce the review result.
- Findings come first, ordered by severity.
- Use file and line references.
- Call out requirement mismatches explicitly when they diverge from Jira, epic, or Confluence design.
- Separate real behavior findings from policy or gate observations such as quality-gate noise, style-only issues, or acceptable duplication.
- If no findings exist, say so plainly and mention any residual testing or context gaps.

## Command Patterns

Prefer these GitHub queries:

- `gh pr view <number> --json ...`
- `gh api repos/<owner>/<repo>/pulls/<number>/comments --paginate`
- `gh api repos/<owner>/<repo>/pulls/<number>/reviews --paginate`
- GraphQL review-thread queries when you need resolved versus unresolved state

Prefer these Atlassian reads:

- `searchAtlassian` to resolve a Jira key or Confluence page from a ticket-like reference
- `getJiraIssue` for story and epic fields that define scope and acceptance criteria
- `fetchAtlassian` when search returns an ARI you need to inspect directly
- `getConfluencePage` for directly linked design docs when the page id is known

Prefer these local checks:

- `git diff origin/<base>...HEAD -- <path>`
- `rg` for locating symbols and related code paths
- targeted `dotnet test` or other narrow test runs when verification matters

## Decision Standard

A review finding is real when at least one of these is true:

- the current code contradicts the Jira story, epic scope, or relevant Confluence design
- the change introduces a bug, regression, or brittle behavior
- the implementation leaves an important failure mode or edge case untested
- the PR description or existing review comments rely on a premise that the current code disproves

A failing gate or bot warning is not, by itself, a code-review finding unless:

- it reveals a real behavior risk
- it contradicts an explicit requirement
- repo policy makes the gate outcome part of merge correctness

When in doubt:

- verify first
- cite the requirement source
- keep the review grounded in the current code, not the earlier diff snapshot
- update or withdraw earlier assumptions when scope clarification changes the right judgment
