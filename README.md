# PipelineIQ

Claude Code slash commands for a closed-loop **spec → plan → implement → audit → fix → merge** pipeline, plus a multi-spec parallel runner built on agent teams.

## Commands

| Command | Purpose |
|---|---|
| `pipeline-team` | Runs multiple specs in parallel waves via an agent team; serializes merges through the lead. |
| `pipeline` | Single-spec closed loop. |
| `spec` | Interviews the user and writes a spec file. |
| `impl-plan` | Produces an actionable implementation plan from a spec. |
| `impl` | Executes an implementation plan inside a worktree. |
| `audit-implementation` | Independent audit against the spec; emits structured Actionable Errors. |
| `fix` | Applies surgical fixes from an audit. |

## Install

Copy the files in `commands/` into your Claude Code commands directory:

- User-scope (global): `~/.claude/commands/`
- Project-scope: `<repo>/.claude/commands/`

## Requirements

- `pipeline-team` requires `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` and Claude Code ≥ 2.1.32.
- The commands reference each other by absolute path (e.g. `/mnt/c/Users/Epkone/.claude/commands/pipeline.md`). Adjust to match your own `~/.claude/commands/` location.
