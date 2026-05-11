# Session Artifacts

Use these local files to preserve compact working context across turns, across tools, or before ending a session. The goal is to avoid reloading raw Jira, PR, branch, or telemetry context when a short state file will do.

Use the templates in `../templates/`:

- `checkpoint.md` — where the work currently stands
- `handoff.md` — what is left to do and what the next agent should watch for
- `resume-prompt.md` — the prompt you would give yourself to restart the work efficiently

## When To Write Them

Write or refresh these files when at least one of these is true:

- the task is likely to continue in another turn
- the task is moving from Claude to Codex or the reverse
- the task has expensive context-gathering that should not be repeated
- the task is paused after planning, research, or partial implementation

Skip them for trivial one-turn tasks.

## Storage Rules

- Save them in the current working directory unless the user asks for another location.
- Prefer stable names:
  - `checkpoint.md`
  - `handoff.md`
  - `resume-prompt.md`
- Overwrite them with the newest state instead of creating a trail of near-duplicates unless the user explicitly wants history.

## Content Rules

- Keep each file short and scannable.
- Store conclusions, scope, file paths, commands, open questions, and next steps.
- Do not paste full raw payloads from Jira, GitHub, or telemetry.
- Do not store secrets, tokens, cookies, or credentials.
- Use exact dates, branch names, PR numbers, ticket keys, entity ids, and file paths when known.

## Division Of Labor

Use each file for one job only:

### Checkpoint

Capture the current known-good state:

- exact objective
- current repo / branch / PR / ticket
- what has already been done
- key decisions already made
- files changed or files likely to matter
- latest verification state

### Handoff

Capture the unfinished work:

- next concrete steps
- blockers and unknowns
- risks or edge cases to verify
- what should not be redone
- what source of truth should be trusted if files and live systems disagree

### Resume Prompt

Capture the shortest good restart prompt:

- enough context to resume quickly
- the exact task to continue
- which local files to read first
- what not to spend tokens re-fetching unless stale

## Read Order

When resuming work from local artifacts:

1. Read `checkpoint.md`
2. Read `handoff.md`
3. Read `resume-prompt.md`
4. Re-fetch only the live state that may have changed since the checkpoint, such as PR comments, CI, branch tip, ticket status, or telemetry windows
