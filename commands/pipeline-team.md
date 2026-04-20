Orchestrates the full implementation pipeline (impl-plan → impl → audit → fix → merge) across **multiple pre-written specs simultaneously** using Claude Code's agent teams feature. Specs run in parallel where dependencies allow; merges are serialized through the lead to preserve the worktree invariant. Replaces the pattern of manually running `/pipeline` N times in sequence.

Invocation: `/pipeline-team [optional-filter]`
- No filter → all `.md` files in `specs/` (not `specs/applied/`)
- With filter → only files whose names contain the filter string (case-insensitive)

**Requires**: `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` in env or settings.json, Claude Code >= 2.1.32.

**Specs must be pre-written** — the interactive spec interview does not compose with parallelism. All specs must exist in `specs/` before invoking.

The argument (if any) is: $ARGUMENTS

---

## Phase 0 — Pre-flight Checks (Lead, Inline)

```
═══════════════════════════════════════════════════════════
  /pipeline-team — PRE-FLIGHT
═══════════════════════════════════════════════════════════
```

Run all checks in order. STOP means print the error message and exit immediately.

**Check 1 — Agent teams feature flag**

Check whether `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` is set to `1` (process env and settings.json). If not:

> STOP: "Agent teams are disabled. Add `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS: \"1\"` to settings.json and rerun."

**Check 2 — Claude Code version**

Run `claude --version`. If the version is below 2.1.32:

> STOP: "Claude Code >= 2.1.32 is required for agent teams. Current version: [X]. Run `npm install -g @anthropic-ai/claude-code` to upgrade."

**Check 3 — Active team detection**

Check `~/.claude/teams/` for any subdirectory containing a `config.json` with a non-empty `members` array. If found:

> STOP: "An active agent team exists at `~/.claude/teams/<name>/`. Clean it up before starting a new batch:
> 1. Tell the lead to run 'clean up the team' in the active session, OR
> 2. Manually delete `~/.claude/teams/<name>/` if the session is gone, then run `git worktree prune`.
> Then rerun `/pipeline-team`."

**Check 4 — Open worktrees**

Run `git worktree list`. If any non-main worktrees exist:

> STOP: "There are open worktrees: [list them]. Close them before running /pipeline-team to avoid merge conflicts."

**Check 5 — Discover specs**

List all `.md` files in `specs/` (not `specs/applied/`). Apply the filter argument if provided (case-insensitive filename match).

If 0 files found:
- With filter → STOP: "No specs found in `specs/` matching filter '$ARGUMENTS'."
- Without filter → STOP: "No specs found in `specs/`. Add spec files there and rerun."

Print: "Found N spec(s): [list filenames]"

**Check 6 — Permission mode warning**

Check the session's current permission mode. If not `bypassPermissions` or `auto`:

```
⚠ WARNING: The lead is not running in auto-approve mode.
  Permission prompts from teammates will surface here and may pause
  parallel pipelines waiting for your approval.
  To prevent interruptions, restart with --dangerously-skip-permissions
  or set permissionMode: bypassPermissions in settings.json.
  Continuing anyway...
```

Do NOT stop. Continue.

**Check 7 — Display mode note**

Print:

```
ℹ Display mode: in-process (use Shift+Down to cycle through teammates)
  Split-pane mode requires tmux or iTerm2 — not available in VS Code,
  Windows Terminal, or Ghostty. In-process mode works in any terminal.
```

---

## Phase 1 — Dependency Graph (Lead, Inline)

```
═══════════════════════════════════════════════════════════
  /pipeline-team — DEPENDENCY ANALYSIS
  Found N specs. Building dependency graph...
═══════════════════════════════════════════════════════════
```

**Detection algorithm:**

For each spec file `S`:
1. Extract the title (first `# ` heading) and the **Goal** + **Current State** body text.
2. For every other spec `T`, extract `T`'s title keywords (words > 4 chars, lowercase, excluding common stopwords like "with", "that", "this", "from", "into", "using", "their").
3. If 2 or more of `T`'s title keywords appear in `S`'s Goal or Current State text → `S` depends on `T`.
4. Ignore self-edges. If a spec's title has no keywords > 4 chars, it contributes no dependency edges.
5. Also check `specs/applied/` — if a dependency is already implemented there, treat it as satisfied (no pending task needed for it; its dependents are immediately unblocked).

**Cycle detection:** before rendering, verify the graph is a DAG. If a cycle is found, STOP and render the cycle:

> STOP: "Circular dependency detected: spec-A → spec-B → spec-C → spec-A. Remove one of these edges before proceeding."

**Render the proposed graph and wait for confirmation:**

```
Proposed dependency graph:
  spec-A.md (no deps)
  spec-B.md (no deps)
  spec-C.md → depends on: spec-A.md
  spec-D.md → depends on: spec-A.md, spec-B.md

Wave 1 (parallel): spec-A.md, spec-B.md
Wave 2 (after A): spec-C.md
Wave 3 (after A+B): spec-D.md

Confirm? (yes / edit)
```

If user says **edit**: accept free-text edits (e.g., "remove spec-C depends-on spec-A", "add spec-D depends-on spec-C"). Re-render and re-confirm. Re-check for cycles after edits.

If user says **yes**: proceed to Phase 2.

---

## Phase 2 — Task List Creation (Lead, Inline)

Create the agent team task list. Each spec becomes one task with dependency edges encoding the confirmed graph.

Task naming convention: `pipeline: <spec-filename-without-extension>`

Initialize the **merge queue**: write `~/.claude/teams/<team-name>/merge-queue.json` as an empty array `[]`. This file persists across teammate messages and is used for crash resilience.

---

## Phase 3 — Wave Execution (Lead + Teammates)

```
═══════════════════════════════════════════════════════════
  /pipeline-team — SPAWNING WAVE 1
  Unblocked specs: [list]
  Spawning N teammates...
═══════════════════════════════════════════════════════════
```

For each spec in the current unblocked wave (max 5 active teammates at any time), spawn a **fresh teammate** using the template below.

When a teammate slot opens (a teammate sends `PIPELINE_COMPLETE` or `PIPELINE_FAILURE`), spawn the next unblocked spec immediately.

### Teammate spawn prompt template

Fill in `[SPEC_FILENAME]` for each spawn. The pipeline command path is always `/mnt/c/Users/Epkone/.claude/commands/pipeline.md`.

```
You are a pipeline worker. Your job is to run the implementation pipeline
for exactly ONE spec file, then report back to the lead.

Spec file: [SPEC_FILENAME]

IMPORTANT: The commands you need are at /mnt/c/Users/Epkone/.claude/commands/.
If that path is unavailable in your environment, check ~/.claude/commands/ as a fallback.

Steps to execute:

1. WORKTREE: Load the EnterWorktree schema (ToolSearch query="select:EnterWorktree"),
   then call EnterWorktree to create a worktree. Derive the slug from the spec filename:
   lowercase, hyphens, max 30 chars. Note the worktree path returned.

2. IMPL-PLAN: Launch a general-purpose subagent with this prompt:
   ---
   A worktree already exists at: [to be filled with your worktree path]
   When you reach the "Create Worktree" phase in impl-plan.md: SKIP IT.
   In the plan's Header, write exactly: **Worktree**: [worktree path]

   You are a planner. Produce a thorough, actionable implementation plan. Do not implement anything.

   Read /mnt/c/Users/Epkone/.claude/commands/impl-plan.md in full and follow every phase in it.
   The spec to plan is: [SPEC_FILENAME]
   ---
   Wait for completion. Note the impl plan filename from Implementation Plans/.

3. IMPL: Launch a general-purpose subagent with this prompt:
   ---
   The worktree for this work is at: [your worktree path]
   All file edits and commits must happen inside this worktree, not on master.

   You are an implementer. Execute the plan precisely. If a step fails after 3 retries, report it and move on.

   Read /mnt/c/Users/Epkone/.claude/commands/impl.md in full and follow every phase in it.
   The implementation plan to execute is: [IMPL_PLAN_FILENAME]

   When you reach the commit phase: commit automatically without asking the user.
   SKIP the merge-into-main phase entirely — the lead handles all merges.
   ---
   Wait for completion. Note the working log filename from Working Logs/.

4. AUDIT: Launch a general-purpose subagent with this prompt:
   ---
   You are an independent auditor. Evaluate the implementation against the spec. Do not modify files.
   Your output must include a structured "Actionable Errors" section.

   Read /mnt/c/Users/Epkone/.claude/commands/audit-implementation.md in full and follow every phase in it.
   The working log to analyze is: [WORKING_LOG_FILENAME]
   ---
   Wait for completion. Note the audit filename and actionable error count.

5. FIX LOOP (if audit found errors): run up to 2 fix cycles per the pipeline.md Phase 5 logic.
   For each cycle, launch a fixer subagent reading /mnt/c/Users/Epkone/.claude/commands/fix.md,
   then re-audit. Stop when errors reach 0 or after 2 cycles.

6. REBASE: Before reporting completion, always rebase:
   git -C [WORKTREE_PATH] rebase master
   If rebase has conflicts, message the lead:
     REBASE_CONFLICT spec=[SPEC_FILENAME] worktree=[WORKTREE_PATH] files=[comma-separated conflict files]
   Then wait for lead instruction before proceeding.

7. COMPLETION REPORT: Message the lead:
   PIPELINE_COMPLETE spec=[SPEC_FILENAME] worktree=[WORKTREE_PATH] branch=[BRANCH_NAME]
   working_log=[WLOG_PATH] audit=[AUDIT_PATH] remaining_errors=[N]
   Then go idle. Do NOT merge.

If impl fails fatally (subagent crashes or produces no output):
   Message the lead:
   PIPELINE_FAILURE spec=[SPEC_FILENAME] worktree=[WORKTREE_PATH]
   branch=[BRANCH_NAME] working_log=[WLOG_PATH] reason=[brief description]
   Then go idle.

MASTER_ADVANCED handling: when the lead sends you MASTER_ADVANCED, note it. At your
next natural checkpoint (before starting a new pipeline phase, or before sending
PIPELINE_COMPLETE), run:
   git -C [WORKTREE_PATH] rebase master
Do not interrupt an in-progress tool call to do this — defer to the next checkpoint.
You must always rebase before sending PIPELINE_COMPLETE.
```

---

### Shutdown procedure

Applies any time the lead asks a teammate to shut down:
1. Lead sends: "Please shut down."
2. Lead waits up to **5 minutes** for confirmation or idle.
3. If confirmed or idle: proceed with cleanup.
4. If 5 minutes pass with no confirmation: log "Teammate [name] did not confirm shutdown — treating as force-abandoned." The stale session will exit when its current tool call completes. Proceed without waiting.

---

## Phase 4 — Merge Queue Processing (Lead)

The lead processes merge requests sequentially, one at a time.

**When lead receives `PIPELINE_COMPLETE` from a teammate:**

1. Append to merge queue: `{ spec, worktree, branch, working_log, audit, remaining_errors }` in `~/.claude/teams/<team-name>/merge-queue.json`.
2. If no merge is currently processing: begin merge immediately.
3. Otherwise: log "Queued merge for [spec] — processing after current merge."

**Merge processing (one at a time):**

```bash
# Step 1: Rebase worktree onto current master
git -C "<worktree_path>" rebase master
```

Rebase outcomes:
- **Success** → continue to step 2.
- **Conflicts in non-critical files only** (docs, changelogs, `learnings.md`, `PATCHNOTES.md`): auto-resolve by taking the worktree version (`git checkout --ours <file>` then `git add <file>`), then `git rebase --continue`. Continue to step 2.
- **Any source code or type definition conflict**: PAUSE. Tell user: "Merge conflict in source files for [spec]: [file list]. Resolve conflicts in [worktree_path] and reply 'continue' or 'skip [spec]'."

```bash
# Step 2: Fast-forward merge into master
git merge --ff-only <branch>
```

If `--ff-only` fails: tell user about the non-fast-forward situation. Do not auto-force. Wait for instructions.

```bash
# Step 3: Clean up worktree
git worktree remove "<worktree_path>"
git branch -d <branch>
git worktree prune
```

**Step 4:** Broadcast `MASTER_ADVANCED` to all active teammates.

**Step 5:** Lead marks the task `completed` in the task list immediately after the successful `git merge --ff-only`. Never wait for the teammate to do this.

**Task-lag fallback:** if dependent tasks do not appear unblocked within 3 minutes of the task being marked complete:
1. Check the task list state directly.
2. If the task still shows as `in_progress` or `pending`: message the now-idle teammate "Please confirm task `pipeline: <spec>` is marked complete."
3. If still stuck after 3 more minutes: mark it complete manually and log: "Task lag detected for `<spec>` — marked complete manually."

**Step 6:** For each newly unblocked task, spawn a fresh teammate (up to 5 total active). Return to Phase 3.

**Step 7:** Process the next item in the merge queue.

---

## Phase 5 — Failure Handling (Lead)

**When lead receives `PIPELINE_FAILURE` from a teammate:**

1. Mark the task as **failed** in the task list.
2. Find all direct and transitive dependents. Mark them **blocked-by-failure**.
3. Spawn a repair subagent via the `Agent` tool:

```
You are a repair agent. A pipeline implementation failed in a worktree.
Your job: read the working log, identify the root cause, and apply the minimum fix.

Working log: [WLOG_PATH]
Worktree path: [WORKTREE_PATH]
Spec file: [SPEC_FILENAME]
Failure reason reported: [REASON]

Steps:
1. Read the working log to find the last failed step and error.
2. Read the relevant source files in the worktree.
3. Apply the minimum fix. Follow the fix logic in /mnt/c/Users/Epkone/.claude/commands/fix.md.
   If that path is unavailable, check ~/.claude/commands/fix.md as a fallback.
4. Report: REPAIR_SUCCESS or REPAIR_FAILURE with a brief explanation.
   Do not audit. Do not merge. Just fix and report.
```

**If repair reports `REPAIR_SUCCESS`:**
- Spawn a fresh audit-phase teammate for this spec (pass existing worktree path).
- Audit teammate runs phases 4–5 only (audit + fix loop, max 2 cycles), then sends `PIPELINE_COMPLETE`.

**If repair reports `REPAIR_FAILURE`:**
- Surface to user:
  > "Spec [SPEC_FILENAME] failed and auto-repair failed. Blocked dependents: [list].
  > Reply 'retry [spec]' to spawn a fresh full pipeline, 'skip [spec]' to unblock dependents anyway, or 'abort' to stop the batch."
- Wait for user instruction.
  - `retry [spec]`: delete the stale worktree, spawn a fresh full-pipeline teammate.
  - `skip [spec]`: mark spec as **skipped** (not failed); unblock all direct dependents; note in batch summary.
  - `abort`: initiate shutdown for all active teammates (5-min wait each), write partial batch summary, exit.

**When lead receives `REBASE_CONFLICT` from a teammate:**
- Non-critical files only: message teammate with resolution instructions, let it continue.
- Source code files: surface to user with the conflict file list and worktree path.

---

## Resume Protocol — After Lead Session Crash

On any `/pipeline-team` invocation that detects a stale team config in `~/.claude/teams/`:

**Step 1 — Detect orphans**

Read `~/.claude/teams/` for a config from the crashed session. For each member in `config.json`, check whether the session ID is still alive. Discard orphaned entries — do not attempt to message dead session IDs.

**Step 2 — Re-evaluate pending specs**

Combine two signals:
- Specs NOT in `specs/applied/` → not yet completed.
- `git worktree list` → specs with an open worktree that started but didn't merge.

For each open worktree, read the most recent working log in `Working Logs/`:
- Working log exists + impl complete → resume from **audit phase** (spawn audit-only teammate).
- Working log exists + impl incomplete → restart from **impl-plan phase**.
- No working log → restart from **impl-plan phase**.

Also check `~/.claude/tasks/<team-name>/` — if the task list file survives, read it directly rather than re-deriving state.

Special case — crash while a merge was in progress:
- If the merge appears in `git log` (the commit exists): clean up the worktree, treat spec as completed.
- If no merge commit: worktree is still open, resume from audit phase.

**Step 3 — Rebuild and continue**

Re-run dependency analysis on the unfinished spec set. Rebuild the task list. Continue from Phase 2. Do not broadcast `MASTER_ADVANCED` — master state is already current for any new worktrees.

---

## Phase 6 — Batch Summary (Lead, After All Tasks Terminal)

All tasks are terminal when every task is `completed`, `failed`, `blocked-by-failure`, or `skipped`.

```
═══════════════════════════════════════════════════════════
  /pipeline-team — COMPLETE
═══════════════════════════════════════════════════════════
```

If `Retros/` does not exist, create it. Write `Retros/batch-summary--YYYY-MM-DD--HH-MM.md`:

```markdown
# Batch Pipeline Summary — YYYY-MM-DD HH:MM

## Overview
- Total specs: N
- Completed: N
- Failed: N
- Blocked by upstream failure: N
- Skipped (user decision): N

## Results

| Spec | Status | Impl Plan | Working Log | Audit | Notes |
|------|--------|-----------|-------------|-------|-------|
| spec-A.md | ✅ completed | Implementation Plans/... | Working Logs/... | Retros/... | |
| spec-B.md | ❌ failed | Implementation Plans/... | Working Logs/... | — | Repair failed: [reason] |
| spec-C.md | ⏸ blocked | — | — | — | Blocked by failure of spec-B.md |

## Remaining Issues
[List specs with unresolved audit errors after the fix loop]

## Next Steps
- Manual tests required for: [specs with 'requires runtime verification' audit items]
- Review proposed skill changes in audit docs
- Specs blocked by failure — resolve and re-run: /pipeline-team [filter]
```

After writing the summary:
- If `package.json` exists at the project root: invoke the `patch-notes` skill.
- Update `learnings.md` with any batch-level observations (patterns across specs, recurring failures, dependency surprises).

---

## Constraints & Invariants

1. **Master is read-only while any worktree is open.** Only the lead merges to master, one at a time.
2. **Only fast-forward merges** (`--ff-only`). No merge commits, no squash merges.
3. **Max 5 active teammates at any time.** Hard cap enforced by the lead.
4. **One repair attempt per failure.** If repair fails, escalate. No infinite repair loops.
5. **Fix loop max 2 cycles per spec** (same as `/pipeline`).
6. **All implementations stay in worktrees** until the lead explicitly merges. Teammates never run merge logic.
7. **Rebase before `PIPELINE_COMPLETE` is mandatory.** Teammates defer mid-tool-call rebases to the next checkpoint.
8. **Lead marks tasks complete** after its own successful merge. Never waits for the teammate.
9. **Teammates do not read the lead's conversation history.** All needed context is in the spawn prompt.
10. **A crashed lead triggers the resume protocol, not a full abort.**
11. **Teammates use the `Agent` tool for pipeline-phase subagents** — this is not a nested team and is explicitly permitted.
12. **Shutdown requests wait up to 5 minutes before force-abandon.**

---

## Edge Cases

| Scenario | Required behavior |
|---|---|
| `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` not set | STOP in Phase 0 with fix instructions |
| Claude Code version < 2.1.32 | STOP in Phase 0 with version mismatch message |
| Zero specs after filtering | STOP with filter value in message |
| Active team config exists at startup | STOP with cleanup instructions |
| Open worktrees at startup | STOP listing the open worktrees |
| Circular dependency detected | STOP: render the cycle, refuse until user removes the edge |
| Self-referential spec | Ignore; no self-edge |
| Spec title has no keywords > 4 chars | Skip keyword extraction; contributes no dependency edges |
| Dependency already in `specs/applied/` | Treat as satisfied; its dependents are immediately unblocked |
| Wave size > 5 unblocked specs | Cap at 5; remaining spawn as slots open |
| Two specs in same wave edit overlapping files | Both proceed; rebase at merge time surfaces conflicts per merge conflict rules |
| Teammate crashes with no message | Lead treats idle notification as `PIPELINE_FAILURE` with `reason=no_output` |
| Teammate ignores shutdown for 5 min | Force-abandon; log event; stale session exits on its own |
| Lead messages a dead teammate | Treat as orphan; remove from member list; inspect git state to resume |
| `git merge --ff-only` fails after clean rebase | Report non-fast-forward to user; do not auto-force |
| Merge queue has multiple items while one is processing | Process sequentially; each merge broadcasts `MASTER_ADVANCED` |
| All specs have no dependencies | All in wave 1; spawn up to 5; rest wait for slots |
| Audit-only replacement teammate receives `MASTER_ADVANCED` | Rebase existing worktree at next checkpoint as normal |
| `Retros/` directory does not exist | Create it before writing batch summary |
| Task lag after lead marks complete | Nudge idle teammate at 3 min; mark manually after second nudge; log "task lag detected" |
| Repair fails + user replies 'skip [spec]' | Mark as skipped; unblock direct dependents; note in summary |
| Repair fails + user replies 'abort' | Shut down active teammates (5-min wait each), write partial summary, exit |
| Lead session crashes mid-batch | Run resume protocol on next invocation |
| Permission prompts mid-run | Expected if not in auto-approve mode (warned in Phase 0); user approves each |
