# Token Budget

These percentages are rough operating targets, not exact measurements. Use them to reason about where token cost usually goes during a real task.

## Claude

Typical split:

- `20%` session history, local memory, and session artifacts
- `15%` shared workflows, references, and templates
- `15%` external context such as Jira, PR state, and design docs
- `15%` tool output such as command results and code-search hits
- `12%` coding context such as repo files, symbols, and local diffs
- `10%` telemetry and runtime evidence
- `8%` planning and coordination
- `5%` wrapper metadata and wrapper body

## Codex

Typical split:

- `28%` coding context such as repo files, symbols, and local diffs
- `24%` build, test, and verification output
- `14%` shared workflows, references, and templates
- `14%` tool output such as command results and code-search hits
- `12%` session history, local memory, and session artifacts
- `10%` external context such as Jira, PR state, and design docs
- `8%` planning and coordination
- `4%` wrapper metadata and wrapper body

## Practical Reading

- The wrappers are not the main cost center.
- Shared workflows matter, but live task context usually dominates.
- In coding-heavy Codex work, repo reads, diffs, and test output are usually the largest categories.
- In mixed operational Claude work, external systems and tool output usually outweigh pure code context.
- Session artifacts help by replacing expensive re-fetch and re-explanation with short local state.
- The biggest savings usually come from narrowing repo reads, tool payloads, telemetry windows, and PR or Jira fetches.
