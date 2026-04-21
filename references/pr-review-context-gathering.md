# PR Review Context Gathering

Use this note at the start of a PR review before taking a position on code or review comments.

## Goal

Build one compact context pack that explains:

- what the PR is trying to deliver
- what requirement source defines that behavior
- what design constraints matter for review
- what reviewers are reacting to in the current diff

Do not start arguing about code before this pack exists.

## Context Order

1. PR metadata
- Read the PR title, body, base branch, head branch, changed files, and linked issue references.
- Treat the PR body as intent, not as the sole source of truth.

2. Jira story
- Resolve the Jira story from:
  - an explicit Jira link in the PR body
  - a ticket key in the PR title
  - a ticket key in the branch name
- Read at least:
  - summary
  - description
  - acceptance criteria
  - definition of done
  - status
  - issue links
  - remote links

3. Epic or parent
- Read the parent or epic only when it affects:
  - scope boundaries
  - acceptance interpretation
  - sequencing with related work
  - architectural intent that the story only summarizes
- Do not read the epic by default if the story is already specific enough.

4. Confluence design context
- Read directly linked Confluence pages from the PR, Jira story, or epic first.
- If no page is linked and the design intent is still unclear, search Confluence by Jira key and feature terms.
- Prefer one or two directly relevant pages over broad Confluence exploration.

5. Existing PR review state
- Read inline comments, replies, unresolved threads, and top-level review summaries.
- Review the current diff against base before deciding whether a comment is still valid.

## Required Output For Yourself

Before making a review judgment, be able to answer these in one or two sentences each:

- What exact behavior is this PR supposed to implement?
- Which requirement source is authoritative for that behavior?
- What is explicitly out of scope?
- Which review comments are contradicted by the current code or by the requirement source?
- Which review comments expose a real mismatch with the requirement source?

## Tool Hints

- Use `gh pr view` for PR metadata and body.
- Use `gh api repos/<owner>/<repo>/pulls/<number>/comments --paginate` for inline comments.
- Use `gh api repos/<owner>/<repo>/pulls/<number>/reviews --paginate` for top-level review summaries.
- Use Atlassian `searchAtlassian` first to resolve Jira and Confluence items.
- Use `getJiraIssue` for the story and epic once resolved.
- Use `getConfluencePage` when you have a page id and need the page body.

## Review Standard

If the current code matches the Jira story and relevant Confluence design, a reviewer request that pushes it away from that contract needs a strong reason to override the documented requirement.

If the code diverges from the Jira story or relevant design doc, treat that as a review issue even when no reviewer noticed it yet.
