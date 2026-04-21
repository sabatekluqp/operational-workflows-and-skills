# Operational Workflows and Skills

Portable skill wrappers and shared workflows for Claude Code and Codex.

Layout modeled on [sethlunn/operational-workflows-and-skills](https://github.com/sethlunn/operational-workflows-and-skills).

## Layout

- `claude/` — thin Claude Code skill wrappers. Each folder holds a `SKILL.md`.
- `codex/` — thin Codex skill wrappers. Each folder holds a `SKILL.md` and usually `agents/openai.yaml`.
- `workflows/` — shared reusable procedures referenced by skill wrappers.
- `references/` — stable facts, query patterns, routing rules.
- `templates/` — reusable output structures.
- `scripts/` — launchers and sync utilities.

## Using This Repo With Claude Code

Symlink the repo-managed skills into your Claude home so new sessions see repo updates automatically:

```bash
./scripts/link-claude-skills
```

That script:

- links every folder under `claude/` into `${CLAUDE_HOME:-$HOME/.claude}/skills/`
- links shared repo roots (`workflows`, `references`, `templates`, `reviews`, `scripts`) into `${CLAUDE_HOME:-$HOME/.claude}/` so relative references inside `SKILL.md` keep working
- safely backs up conflicting copied folders with a timestamped suffix

To have a new session automatically pull the newest available source, use:

```bash
./scripts/claude-session
```

That launcher:

- refreshes the skill links via `sync-claude-skills` before starting Claude Code
- prefers the current local checkout when it contains the newest edits or commits
- fast-forwards the local repo when the tracked upstream is newer and the pull is safe
- otherwise builds a temporary upstream snapshot and links from that without disturbing local work
- then `exec`s `claude --add-dir <repo>` so the skill repo is a writable workspace

To refresh links without starting Claude Code:

```bash
./scripts/sync-claude-skills
```

To force the current local checkout as the source (for iterating on uncommitted changes):

```bash
./scripts/sync-claude-skills --link-only
```

## Using This Repo With Codex

Mirror setup for Codex:

```bash
./scripts/link-codex-skills      # one-off link
./scripts/sync-codex-skills      # refresh links with upstream awareness
./scripts/codex-session          # refresh links then launch codex in this repo
```

These link under `${CODEX_HOME:-$HOME/.codex}/`.

## Adding a New Skill

1. Create `claude/<skill-name>/SKILL.md` (and/or `codex/<skill-name>/SKILL.md`).
2. Keep the wrapper thin — it routes the agent to a shared workflow.
3. Put the reusable procedure in `workflows/<name>.md`.
4. Put stable supporting facts in `references/`.
5. Put output shape in `templates/`.
6. Re-run the relevant `sync-*` script.

## Design Principles

- Keep `SKILL.md` files small and routing-oriented.
- Put reusable business logic in `workflows/`, not in skill wrappers.
- Put stable supporting knowledge in `references/`, not in workflows unless it is procedural.
- Put output shape in `templates/`, not process.
- Prefer exact dates, exact ids, and exact query shapes over vague summaries.
- Don't store secrets or credentials in the repo.
