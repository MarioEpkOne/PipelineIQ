CLOSED-LOOP PIPELINE -- runs the complete spec -> impl-plan -> impl -> audit -> fix loop -> summary workflow.
The spec phase is interactive (inline). Each subsequent phase runs as an isolated subagent with clean context.
If errors are found during audit, a fix->re-audit loop runs automatically (max 2 cycles).

The user's request/feature description is: $ARGUMENTS

---

## Argument Parsing

Parse $ARGUMENTS for the `SKIP_MERGE=true` sentinel:
- Match `SKIP_MERGE=true` only as a **whitespace-delimited token** (not as a substring of a filename or path). In practice this means: split arguments on whitespace, check if any token is exactly `SKIP_MERGE=true`, remove that token, rejoin the rest.
- If found: set SKIP_MERGE = true; the remaining text is the user's request / spec path.
- If not found: SKIP_MERGE = false (default).

SKIP_MERGE is used in Phase 6 to control whether the merge workflow runs. When /pipeline is invoked standalone by a user, SKIP_MERGE is never present.

**SKIP_MERGE contract** (do not change without updating pipeline-team.md):
- When SKIP_MERGE=true, Phase 6 MUST output: `PIPELINE_COMPLETE spec=<path> worktree=<path> branch=<name> working_log=<path> audit=<path> remaining_errors=<N>`
- pipeline-team.md depends on this exact output format for teammate coordination.

---

## Pre-flight -- Validate Reference Files

Before starting any phase, verify that all required reference files exist. Check each path and collect any missing files into a list. If any are missing, STOP immediately with the full list — do not start Phase 1.

Required files:
- `commands/references/pipeline/phase2-impl-plan.md`
- `commands/references/pipeline/phase3-impl.md`
- `commands/references/pipeline/phase4-audit.md`
- `commands/references/pipeline/phase5-fix.md`
- `commands/references/pipeline/phase5-reaudit.md`
- `commands/references/pipeline/phase6-merge.md`

On failure: "Missing reference file(s): [list]. These files are required for the pipeline subagent prompts. Check that `commands/references/pipeline/` exists and contains all 6 phase files."

---

## Decision Tree Overview

```
Phase 1: SPEC (inline) -> 1.5: WORKTREE (inline)
  -> Phase 2: IMPL-PLAN (subagent) -> Phase 3: IMPL (subagent)
  -> Phase 4: AUDIT (subagent)
     0 errors -> Phase 6 | errors -> Phase 5
  -> Phase 5: FIX LOOP (max 2: fix -> re-audit -> check)
  -> Phase 6: SUMMARY + learnings + (SKIP_MERGE ? PIPELINE_COMPLETE : merge)
Each phase failure -> STOP + report (except Phase 3 partial -> continue)
```

---

## Phase 1 -- Spec (Interactive, Inline)

--- Phase 1/6: SPEC ---

Run the full spec workflow inline -- do NOT spawn a subagent for this phase, user interaction is required.

> **Existing spec shortcut**: If the remaining argument (after stripping SKIP_MERGE) is a path to an existing `.md` file in `specs/` or `specs/applied/`, skip the spec workflow and go directly to Phase 2 with that path.

### Phase 1a -- Run the Spec Workflow

Read `/home/epkone/.claude/commands/spec.md` in full and follow every phase in it exactly. Note the exact filename written to `specs/`.

---

## Phase 1.5 -- Create Worktree (Inline)

--- Phase 1.5: WORKTREE ---

**Capture the main repo path** via `git rev-parse --show-toplevel`. Store as `MASTER_REPO_PATH`.

**Derive a slug** from the spec filename: lowercase, hyphens only, max 30 chars. Check `git worktree list` for collisions; append `-2`/`-3` if needed. STOP after 3 collisions.

**Create the worktree**: Load the EnterWorktree schema via `ToolSearch query="select:EnterWorktree"`. Call `EnterWorktree name='<slug>'`. If it fails: STOP. Do not fall back to master.

**Verify**: Run `git worktree list` and confirm the new entry exists on disk. If not: STOP with "EnterWorktree returned success but the worktree does not exist on disk."

Store the worktree path as `WORKTREE_PATH` and branch as `WORKTREE_BRANCH`.

---

## Phase 2 -- Impl-Plan (Subagent)

--- Phase 2/6: IMPL-PLAN (specs/<filename>) ---

Read `commands/references/pipeline/phase2-impl-plan.md`. If the file does not exist or is empty, STOP with: "Reference file not found: commands/references/pipeline/phase2-impl-plan.md. Cannot proceed."

Fill in variables: [SPEC_FILENAME], [WORKTREE_PATH], [MASTER_REPO_PATH].
Launch a general-purpose Agent with the filled prompt.

Wait for completion. Note the impl plan filename (check `[MASTER_REPO_PATH]/Implementation Plans/` for the most recently modified file if not reported).

**Decision tree:**
- If subagent completes successfully -> proceed to Phase 3
- If subagent fails -> STOP. Report the error. Do not proceed.

---

## Phase 3 -- Impl (Subagent)

--- Phase 3/6: IMPL (<plan filename>) ---

Read `commands/references/pipeline/phase3-impl.md`. If the file does not exist or is empty, STOP with: "Reference file not found: commands/references/pipeline/phase3-impl.md. Cannot proceed."

Fill in variables: [IMPL_PLAN_FILENAME], [WORKTREE_PATH], [MASTER_REPO_PATH].
Launch a general-purpose Agent with the filled prompt.

Wait for completion. Note the working log filename (check `[MASTER_REPO_PATH]/Working Logs/` for the most recently modified file).

**Decision tree:**
- If subagent completes (even with some FAILED steps) -> proceed to Phase 4
- If subagent crashes or produces no output -> STOP. Report: "Impl subagent failed with no output. Last known artifact: [working log if any]." Do not proceed.

---

## Phase 4 -- Audit (Subagent)

--- Phase 4/6: AUDIT (<working log filename>) ---

Read `commands/references/pipeline/phase4-audit.md`. If the file does not exist or is empty, STOP with: "Reference file not found: commands/references/pipeline/phase4-audit.md. Cannot proceed."

Fill in variables: [WORKING_LOG_FILENAME], [MASTER_REPO_PATH].
Launch a general-purpose Agent with the filled prompt.

Wait for completion. Read the audit document from `[MASTER_REPO_PATH]/Retros/` (most recently modified file). Store its path as `AUDIT_FILENAME`.

**Parse the "Actionable Errors" section:**
- Count the numbered error entries (ignore the "Not actionable" list)

**Decision tree:**
- If 0 actionable errors -> skip Phase 5, proceed to Phase 6
- If >= 1 actionable errors -> proceed to Phase 5 with `AUDIT_FILENAME`
- If audit subagent fails -> report warning, proceed to Phase 6 with note: "Audit failed -- fix loop skipped. Review manually."

---

## Phase 5 -- Fix Loop (max 2 iterations)

Set `MAX_LOOPS = 2`. Initialize `loop_count = 0`.

### Loop start:

--- Phase 5/6: FIX (loop {loop_count + 1}/{MAX_LOOPS}, errors: {error_count}) ---

Read `commands/references/pipeline/phase5-fix.md`. If the file does not exist or is empty, STOP with: "Reference file not found: commands/references/pipeline/phase5-fix.md. Cannot proceed."

Fill in variables: [AUDIT_FILENAME], [WORKTREE_PATH], [MASTER_REPO_PATH].
Launch a general-purpose Agent with the filled prompt.

Wait for the fixer to complete.

**Before spawning re-audit:** Briefly scan the fixer's output or its working log (most recently modified file in `[MASTER_REPO_PATH]/Working Logs/` whose name starts with "fixer-"). If the fixer logged "0 errors fixed" or applied no file changes, skip the re-audit subagent and proceed to Phase 6 with remaining errors unchanged.

### Re-audit:

--- RE-AUDIT (after fix loop {loop_count + 1}) ---

Read `commands/references/pipeline/phase5-reaudit.md`. If the file does not exist or is empty, STOP with: "Reference file not found: commands/references/pipeline/phase5-reaudit.md. Cannot proceed."

Fill in variables: [WORKING_LOG_FILENAME], [AUDIT_FILENAME], [MASTER_REPO_PATH], {loop_count}.
Launch a general-purpose Agent with the filled prompt.

Parse the "Remaining Actionable Errors" section from the appended content in `[AUDIT_FILENAME]`.

**Persist remaining errors to disk:** Write the remaining error list (or "None") to `[MASTER_REPO_PATH]/Retros/pipeline-remaining-<timestamp>.md`. Phase 6 reads this file -- do not rely on session context alone.

**Decision tree after re-audit:**
- If 0 actionable errors -> exit loop, proceed to Phase 6
- If errors remain AND `loop_count < MAX_LOOPS - 1` -> increment `loop_count`, go to **Loop start**
- If errors remain AND `loop_count >= MAX_LOOPS - 1` -> exit loop, proceed to Phase 6 with remaining errors

---

## Phase 6 -- Summary

--- PIPELINE COMPLETE ---

### Artifact paths
List all artifacts created during this pipeline run:
- Spec: `specs/applied/<filename>` (moved from `specs/` after planning)
- Impl plan: `Implementation Plans/<filename>`
- Working log: `Working Logs/<filename>`
- Audit(s): `Retros/<filename>` (and any re-audit files)
- Fixer log(s): `Working Logs/fixer-log--<filename>` (if fix loop ran)

### Report remaining issues
If the fix loop ran and errors remain after 2 cycles:
- Read the remaining errors from `[MASTER_REPO_PATH]/Retros/pipeline-remaining-<timestamp>.md`
- List each remaining error (title + brief description)
- Say: "N errors remain after 2 fix cycles. Review the final audit at `Retros/<filename>` and fix manually or run `/fix <filename>` again."

### Update learnings.md

Reflect on the pipeline run. Write only actionable improvements to the pipeline process (not code patterns).

Format each entry as:
```
## [Short title]
**Phase affected**: [spec / impl-plan / impl / audit / fix]
**What happened**: [one sentence — the slowdown or friction observed]
**Suggestion**: [concrete change to the pipeline prompt or process]
```

If `learnings.md` does not exist: create it with a `# Pipeline Learnings` header. Append new entries; skip if nothing notable. Do NOT delete existing entries. Do NOT invoke learnings-review or modify any command/skill files.

### SKIP_MERGE branch:

**If SKIP_MERGE = true:**
  - Skip merge workflow, patch-notes, and next-steps suggestions.
  - Output structured completion report:
    ```
    PIPELINE_COMPLETE spec=<path> worktree=<path> branch=<name>
    working_log=<path> audit=<path> remaining_errors=<N>
    ```
  - Then go idle.

**If SKIP_MERGE = false (default, standalone /pipeline):**
  - Read `commands/references/pipeline/phase6-merge.md`. If the file does not exist or is empty, STOP with: "Reference file not found: commands/references/pipeline/phase6-merge.md. Cannot proceed."
  - Follow the merge workflow. Variables: WORKTREE_PATH, WORKTREE_BRANCH, MASTER_REPO_PATH.
  - Suggest next steps:
    - "Run manual tests for items marked 'requires runtime verification' in the audit"
    - "Review proposed skill changes in the audit -- reply 'apply [title]' or 'apply all'"
    - "Run `/learnings-review` to review and apply accumulated pipeline improvement suggestions"
    - Any other context-specific suggestions
