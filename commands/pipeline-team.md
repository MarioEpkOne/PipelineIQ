Orchestrates the full implementation pipeline (impl-plan -> impl -> audit -> fix -> merge) across **multiple pre-written specs simultaneously** using Claude Code's agent teams feature. Specs run in parallel where dependencies allow; merges are serialized through the lead to preserve the worktree invariant. Replaces the pattern of manually running `/pipeline` N times in sequence.

Invocation: `/pipeline-team [path-or-filter]`
- No argument -> scans `specs/team/`
- Argument with `/` -> treated as directory path to scan
- Argument without `/` -> treated as filename filter in `specs/team/`

**Requires**: `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` in env or settings.json, Claude Code >= 2.1.32.

**Specs must be pre-written** -- the interactive spec interview does not compose with parallelism. Split large specs with `/spec-splitter` first, then run this command.

The argument (if any) is: $ARGUMENTS

---

## Phase 0 -- Pre-flight Checks (Lead, Inline)

--- /pipeline-team: PRE-FLIGHT ---

Run all checks in order. STOP means print the error message and exit immediately.

**Check 1 -- Agent teams feature flag**

Check whether `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` is set to `1` (process env and settings.json). If not:

> STOP: "Agent teams are disabled. Add `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS: \"1\"` to settings.json and rerun."

**Check 2 -- Claude Code version**

Run `claude --version`. If the version is below 2.1.32:

> STOP: "Claude Code >= 2.1.32 is required for agent teams. Current version: [X]. Run `npm install -g @anthropic-ai/claude-code` to upgrade."

**Check 3 -- Active team detection**

Check `~/.claude/teams/` for any subdirectory containing a `config.json` with a non-empty `members` array. If found:

> STOP: "An active agent team exists at `~/.claude/teams/<name>/`. Clean it up before starting a new batch:
> 1. Tell the lead to run 'clean up the team' in the active session, OR
> 2. Manually delete `~/.claude/teams/<name>/` if the session is gone, then run `git worktree prune`.
> Then rerun `/pipeline-team`."

**Check 4 -- Open worktrees**

Run `git worktree list`. If any non-main worktrees exist:

> STOP: "There are open worktrees: [list them]. Close them before running /pipeline-team to avoid merge conflicts."

**Check 5 -- Discover specs**

Parse the argument to determine the scan directory and optional filter:

1. If no argument: scan `specs/team/`.
2. If argument contains `/` or ends with `/`: treat the argument as a directory path. Scan that directory.
3. Otherwise: treat the argument as a filename filter. Scan `specs/team/` and apply the filter (case-insensitive filename match).

In all cases, never scan `specs/applied/` or `specs/split/`. List all `.md` files in the resolved directory (non-recursive -- do not descend into subdirectories).

If 0 files found:
- With filter -> STOP: "No specs found in `specs/team/` matching filter '$ARGUMENTS'."
- With explicit directory -> STOP: "No specs found in `$DIR`."
- Without argument -> STOP: "No specs found in `specs/team/`. Split a spec with `/spec-splitter` or add spec files there and rerun."

Print: "Found N spec(s): [list filenames]"

**Check 6 -- Permission mode warning**

If not in `bypassPermissions` or `auto` mode: warn that permission prompts may pause parallel pipelines; suggest `--dangerously-skip-permissions` or `permissionMode: bypassPermissions`. Do NOT stop.

**Check 7 -- Display mode note**

Print: "Display mode: in-process (Shift+Down to cycle teammates). Split-pane requires tmux/iTerm2."

---

## Phase 1 -- Dependency Graph (Lead, Inline)

--- /pipeline-team: DEPENDENCY ANALYSIS (N specs) ---

**Detection algorithm:**

For each spec file `S`:
1. Extract the title (first `# ` heading) and the **Goal** + **Current State** body text.
2. For every other spec `T`, extract `T`'s title keywords (words > 4 chars, lowercase, excluding common stopwords like "with", "that", "this", "from", "into", "using", "their").
3. If 2 or more of `T`'s title keywords appear in `S`'s Goal or Current State text -> `S` depends on `T`.
4. Ignore self-edges. If a spec's title has no keywords > 4 chars, it contributes no dependency edges.
5. Also check `specs/applied/` -- if a dependency is already implemented there, treat it as satisfied (no pending task needed for it; its dependents are immediately unblocked).

**Cycle detection:** before rendering, verify the graph is a DAG. If a cycle is found, STOP and render the cycle:

> STOP: "Circular dependency detected: spec-A -> spec-B -> spec-C -> spec-A. Remove one of these edges before proceeding."

**Render the proposed graph and wait for confirmation:**

```
Proposed dependency graph:
  spec-A.md (no deps)
  spec-B.md (no deps)
  spec-C.md -> depends on: spec-A.md
  spec-D.md -> depends on: spec-A.md, spec-B.md

Wave 1 (parallel): spec-A.md, spec-B.md
Wave 2 (after A): spec-C.md
Wave 3 (after A+B): spec-D.md

Confirm? (yes / edit)
```

If user says **edit**: accept free-text edits (e.g., "remove spec-C depends-on spec-A", "add spec-D depends-on spec-C"). Re-render and re-confirm. Re-check for cycles after edits.

If user says **yes**: proceed to Phase 2.

---

## Phase 2 -- Task List Creation (Lead, Inline)

Create the agent team task list. Each spec becomes one task with dependency edges encoding the confirmed graph.

Task naming convention: `pipeline: <spec-filename-without-extension>`

Initialize the **merge queue**: write `~/.claude/teams/<team-name>/merge-queue.json` as an empty array `[]`. This file persists across teammate messages and is used for crash resilience.

---

## Phase 3 -- Wave Execution (Lead + Teammates)

--- /pipeline-team: SPAWNING WAVE N (specs: [list]) ---

For each spec in the current unblocked wave (max 5 active teammates at any time), spawn a **fresh teammate** using the template below.

When a teammate slot opens (a teammate sends `PIPELINE_COMPLETE` or `PIPELINE_FAILURE`), spawn the next unblocked spec immediately.

### Teammate spawn prompt template

For each spec, spawn a teammate with this prompt (fill in [SPEC_FILENAME]):

---
You are a pipeline worker. Your job is to run the full implementation
pipeline for exactly ONE spec file, then report back to the lead.

Read `commands/pipeline.md` in full and follow every phase in it.

Your arguments are: [SPEC_FILENAME] SKIP_MERGE=true

(SKIP_MERGE=true MUST be a separate whitespace-delimited token, not
embedded in a filename. pipeline.md's argument parser expects this.)

The commands directory is at /mnt/c/Users/Epkone/.claude/commands/.
If that path is unavailable, check ~/.claude/commands/ as a fallback.

When pipeline.md reaches Phase 6 with SKIP_MERGE=true, it will output
a structured PIPELINE_COMPLETE report. Forward that exact output to the
lead by messaging:

PIPELINE_COMPLETE spec=[SPEC_FILENAME] worktree=[WORKTREE_PATH]
branch=[BRANCH_NAME] working_log=[WLOG_PATH] audit=[AUDIT_PATH]
remaining_errors=[N]

Then go idle. Do NOT merge.

If the pipeline fails fatally, message the lead:
PIPELINE_FAILURE spec=[SPEC_FILENAME] worktree=[WORKTREE_PATH]
branch=[BRANCH_NAME] working_log=[WLOG_PATH] reason=[brief description]
Then go idle.

When the lead sends MASTER_ADVANCED, note it. At your next natural
checkpoint (before starting a new pipeline phase), run:
  git -C [WORKTREE_PATH] rebase master
You must always rebase before the pipeline's Phase 5 fix loop exits.
---

### Shutdown procedure

Applies any time the lead asks a teammate to shut down:
1. Lead sends: "Please shut down."
2. Lead waits up to **5 minutes** for confirmation or idle.
3. If confirmed or idle: proceed with cleanup.
4. If 5 minutes pass with no confirmation: log "Teammate [name] did not confirm shutdown -- treating as force-abandoned." The stale session will exit when its current tool call completes. Proceed without waiting.

---

## Phase 4 -- Merge Queue Processing (Lead)

The lead processes merge requests sequentially, one at a time.

**When lead receives `PIPELINE_COMPLETE` from a teammate:**

1. Append to merge queue: `{ spec, worktree, branch, working_log, audit, remaining_errors }` in `~/.claude/teams/<team-name>/merge-queue.json`.
2. If no merge is currently processing: begin merge immediately.
3. Otherwise: log "Queued merge for [spec] -- processing after current merge."

**Merge processing (one at a time):**

1. **Rebase**: `git -C "<worktree_path>" rebase master`. On success -> continue. Non-critical file conflicts (docs, changelogs, learnings.md, PATCHNOTES.md): auto-resolve with `--ours`, then `git rebase --continue`. Source code conflicts: PAUSE and ask user to resolve or skip.
2. **Merge**: `git merge --ff-only <branch>`. If ff-only fails: report to user, do not force.
3. **Cleanup**: `git worktree remove "<worktree_path>" && git branch -d <branch> && git worktree prune`.
4. **Broadcast** `MASTER_ADVANCED` to all active teammates.
5. **Mark task complete** immediately after merge. Never wait for the teammate.
6. **Task-lag fallback**: if dependents are not unblocked after 3 min, nudge the idle teammate. After 3 more min, mark complete manually and log "task lag detected."
7. **Spawn** newly unblocked tasks (up to 5 active). Return to Phase 3.
8. **Next queue item**: process sequentially.

---

## Phase 5 -- Failure Handling (Lead)

Read `commands/references/pipeline-team/failure-handling.md` and follow the failure handling protocol. Variables: WLOG_PATH, WORKTREE_PATH, SPEC_FILENAME, REASON.

If the reference file does not exist or is empty, STOP with: "Reference file not found: commands/references/pipeline-team/failure-handling.md. Cannot proceed."

---

## Resume Protocol -- After Lead Session Crash

Read `commands/references/pipeline-team/resume-protocol.md` and follow the resume protocol.

If the reference file does not exist or is empty, STOP with: "Reference file not found: commands/references/pipeline-team/resume-protocol.md. Cannot proceed."

---

## Phase 6 -- Batch Summary (Lead, After All Tasks Terminal)

All tasks are terminal when every task is `completed`, `failed`, `blocked-by-failure`, or `skipped`.

--- /pipeline-team: COMPLETE ---

If `Retros/` does not exist, create it. Write `Retros/batch-summary--YYYY-MM-DD--HH-MM.md` with:
- **Overview**: total/completed/failed/blocked/skipped counts
- **Results table**: `| Spec | Status | Impl Plan | Working Log | Audit | Notes |` -- one row per spec
- **Remaining Issues**: specs with unresolved audit errors
- **Next Steps**: manual tests needed, proposed skill changes to review, blocked specs to re-run

### Post-merge patch-notes (after all merges complete)

After all merge queue items are processed and before writing the batch summary: if `package.json` exists at the project root, invoke the patch-notes skill. This captures all merged specs in a single pass.

### Update learnings

After writing the batch summary, update `learnings.md` using the same smart learnings process defined in `pipeline.md`'s Phase 6 "Update learnings.md" section. Follow all 6 steps (read existing, generate observations, deduplicate, update/create entries, write file).

**Batch-specific additions to the standard process:**

1. **Gather observations from all teammates**: Read all teammate working logs and audit documents to identify friction, slowdowns, or failures across the batch. Each teammate's pipeline run is an independent source of observations.

2. **Teammate occurrence counting**: If multiple teammates independently hit the same issue, count each teammate as a separate occurrence. For example, if 3 out of 5 teammates hit the same friction point, that is +3 occurrences for that entry.

3. **Batch-level observations**: In addition to per-teammate observations, look for batch-level patterns:
   - Did the dependency graph miss an edge? Did specs conflict at merge time?
   - Were there recurring failures across specs? Did the fix loop churn on the same class of error?
   - Did wave scheduling cause unnecessary delays?
   - Were there merge conflicts that a better spec dependency analysis could have prevented?

4. **Single batch = one occurrence per teammate per learning.** If the same teammate hits the same issue multiple times within their pipeline run, that counts as one occurrence from that teammate.

Follow all constraints from pipeline.md's learnings section: diffs target `commands/` only, never delete entries, never invoke /learnings-review, err toward false separation.

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
11. **Teammates read and follow `commands/pipeline.md` with SKIP_MERGE=true** -- they delegate all pipeline logic to pipeline.md, no inline re-implementation. **Contract dependency:** pipeline.md's SKIP_MERGE output format (`PIPELINE_COMPLETE spec=... worktree=... branch=... working_log=... audit=... remaining_errors=...`) is parsed by the lead. If pipeline.md changes this format, pipeline-team breaks. Both files carry matching contract markers.
12. **Shutdown requests wait up to 5 minutes before force-abandon.**
13. **Patch-notes runs once after all merges**, in the lead's batch summary phase. Not after each merge, not by teammates.

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
| Reference file not found | STOP with clear error message naming the missing file |
