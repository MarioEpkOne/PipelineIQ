# PipelineIQ

> **Work in progress** — actively developed and updated. Commands may change between versions.

Claude Code slash commands for a closed-loop **spec → plan → implement → audit → fix → merge** pipeline, plus a multi-spec parallel runner built on agent teams.

## How It Works

### `/pipeline` — Single-Spec Pipeline

Runs the full lifecycle for one feature or change. The spec phase is interactive (you answer questions); every subsequent phase runs as an isolated subagent with clean context.

```
/pipeline add a retry queue for failed webhook deliveries — store payloads in SQLite, exponential backoff with jitter, max 5 attempts, dead-letter after exhaustion with a summary email to the ops alias
```

**Phases:**

1. **Spec** — interactive interview that produces a spec file in `specs/`
2. **Worktree** — creates an isolated git worktree so work never touches your main branch
3. **Impl-Plan** — subagent reads the spec and writes a step-by-step implementation plan
4. **Impl** — subagent executes the plan inside the worktree
5. **Audit** — independent subagent evaluates the implementation against the spec, emits structured Actionable Errors
6. **Fix Loop** — if errors are found, a fix → re-audit cycle runs automatically (max 2 rounds)
7. **Summary** — lists artifacts, updates `learnings.md`, merges the worktree to master

If you already have a spec written, skip the interview:

```
/pipeline specs/webhook-retry-queue.md
```

**Artifacts produced:**

| Directory | Contents |
|---|---|
| `specs/` → `specs/applied/` | Spec file (moved to `applied/` after planning) |
| `Implementation Plans/` | Step-by-step implementation plan |
| `Working Logs/` | Execution log from the impl phase |
| `Retros/` | Audit document with Actionable Errors |
| `Working Logs/fixer-log--*` | Fix loop log (if the fix loop ran) |

### `/pipeline-team` — Parallel Multi-Spec Pipeline

Runs `/pipeline` across multiple pre-written specs simultaneously using Claude Code's agent teams. Specs execute in parallel waves where dependencies allow; merges are serialized through the lead to keep git history linear.

**Specs must be written beforehand** — the interactive spec interview doesn't compose with parallelism. Write your specs first (manually or via `/spec`), then run:

```
/pipeline-team
```

This picks up all `.md` files in `specs/`. To run a subset:

```
/pipeline-team dark-mode
```

**What happens:**

1. **Pre-flight** — checks feature flags, Claude Code version, clean worktree state
2. **Dependency graph** — analyzes spec titles/content for keyword overlap, proposes execution waves, asks you to confirm or edit
3. **Wave execution** — spawns up to 5 teammates in parallel, each running the full pipeline with `SKIP_MERGE` so they don't merge on their own
4. **Merge queue** — the lead merges completed specs one at a time (rebase + fast-forward only), broadcasts to active teammates to rebase
5. **Failure handling** — spawns a repair agent on failure, escalates to user if repair fails
6. **Batch summary** — writes a results table to `Retros/batch-summary--*.md`

### `/spec`, `/impl-plan`, `/impl`, `/audit-implementation`, `/fix`

Each phase of the pipeline is also available as a standalone command. Useful when you want to run or re-run a single phase:

```
/spec                              # interactive spec interview
/impl-plan specs/my-feature.md     # plan from an existing spec
/impl "Implementation Plans/..."   # execute a plan
/audit-implementation              # audit the current worktree
/fix Retros/audit--my-feature.md   # fix errors from an audit
```

## Learnings

Each pipeline run reflects on what went well and what didn't, appending actionable observations to `learnings.md` at the project root. These are process improvements (not code patterns) — things like "the plan prescribed content that exceeded its own line-count target."

Learnings accumulate across runs. Use `/learnings-review` to review them and decide which to apply.

## Install

Copy the entire `commands/` directory (including `references/`) into your Claude Code commands directory:

- User-scope (global): `~/.claude/commands/`
- Project-scope: `<repo>/.claude/commands/`

The `references/` subdirectory contains subagent prompt templates used by `pipeline` and `pipeline-team`. Both commands validate these files at startup and will stop with a clear error if any are missing.

## Requirements

- Claude Code (the CLI tool or IDE extension)
- `pipeline-team` requires `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` and Claude Code >= 2.1.32
- The commands reference each other by absolute path (e.g. `~/.claude/commands/pipeline.md`). After installing, check for hardcoded paths and adjust to match your setup.
