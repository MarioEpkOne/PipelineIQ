CLOSED-LOOP PIPELINE — runs the complete spec → impl-plan → impl → audit → fix loop → summary workflow.
The spec phase is interactive (inline). Each subsequent phase runs as an isolated subagent with clean context.
If errors are found during audit, a fix→re-audit loop runs automatically (max 2 cycles).

The user's request/feature description is: $ARGUMENTS

---

## Decision Tree Overview

```
Phase 1: SPEC (interactive, inline)
  |
  v
Phase 1.5: WORKTREE (inline)
  |
  | failure → STOP + report
  v
Phase 2: IMPL-PLAN (subagent)
  |
  | failure → STOP + report
  v
Phase 3: IMPL (subagent, with per-step retries)
  |
  | success (even partial) → continue
  | total failure → STOP + report
  v
Phase 4: AUDIT (subagent)
  |
  | 0 actionable errors → skip to Phase 6
  | >= 1 actionable errors → Phase 5
  | audit failure → Phase 6 with warning
  v
Phase 5: FIX LOOP (max 2 iterations)
  |
  | fix → re-audit → 0 errors → Phase 6
  | fix → re-audit → errors + loop < 2 → loop again
  | fix → re-audit → errors + loop >= 2 → Phase 6 + report remaining
  v
Phase 6: SUMMARY
  - Artifact paths
  - Update learnings.md
  - Remaining errors (if any)
  - Next steps
```

---

## Phase 1 — Spec (Interactive, Inline)

Print this banner:

```
══════════════════════════════════════════════════════════════
  PIPELINE PHASE 1 OF 6 — SPEC
  Writing spec interactively with you.
  Phases 2-6 will run automatically as isolated subagents.
══════════════════════════════════════════════════════════════
```

Run the full spec workflow inline — do NOT spawn a subagent for this phase, user interaction is required.

> **Existing spec shortcut**: If the argument is a path to an existing file in `specs/` or `specs/applied/` (e.g. it ends in `.md` and the file exists on disk at either location), skip Phase 1a and go directly to Phase 2. Resolve the absolute path and pass it directly to the Phase 2 subagent. Do not create a second spec document. The planner (impl-plan.md Phase 3.6) will move it to `specs/applied/` as normal.

### Phase 1a — Run the Spec Workflow

Read `/home/epkone/.claude/commands/spec.md` in full and follow every phase in it exactly (orient, interview, write spec to `specs/`).

Note the exact filename written to `specs/` — you will pass it to the Phase 2 subagent.

---

## Phase 1.5 — Create Worktree (Inline)

Print this banner:

```
══════════════════════════════════════════════════════════════
  PIPELINE PHASE 1.5 — WORKTREE
  Creating isolated branch for this pipeline run...
══════════════════════════════════════════════════════════════
```

**Capture the main repo path before entering the worktree:**
```bash
git rev-parse --show-toplevel
```
Store this as `MASTER_REPO_PATH`. You will inject it into every subagent prompt and use it in Phase 6 git commands.

Derive a slug from the spec filename: lowercase, hyphens only, max 30 chars.
Example: `spec--2026-04-16--18-00--perception-verification-and-dom-injection.md` → `perception-verification-dom`

**Check for slug collision before calling EnterWorktree:**
```bash
git worktree list
```
If a worktree with the same branch name already exists: append `-2` to the slug (or `-3` if `-2` also collides). If you cannot derive a unique slug after 3 attempts, STOP and tell the user to clean up stale worktrees first.

Load the EnterWorktree schema:
```
ToolSearch query="select:EnterWorktree"
```

Call: `EnterWorktree name='<slug>'`

**If EnterWorktree fails for any reason: STOP the pipeline. Report the error. Do not fall back to working on master. Tell the user to resolve the issue and rerun.**

**After EnterWorktree succeeds, verify the worktree actually exists on disk:**
```bash
git worktree list
```
Confirm a new entry appears with the expected branch name AND that its listed path exists. If `git worktree list` does not show the new entry, or the path is missing: STOP the pipeline with: "EnterWorktree returned success but the worktree does not exist on disk — check your WorktreeCreate hook."

Store the returned worktree path as `WORKTREE_PATH` and the branch name as `WORKTREE_BRANCH`. You will use these in every subsequent phase and in Phase 6 git commands.

---

## Phase 2 — Impl-Plan (Subagent)

Print this banner:

```
══════════════════════════════════════════════════════════════
  PIPELINE PHASE 2 OF 6 — IMPL-PLAN
  Spec: specs/<filename>
  Launching planner subagent...
══════════════════════════════════════════════════════════════
```

Launch a general-purpose Agent with this prompt (fill in [SPEC_FILENAME], [WORKTREE_PATH], [MASTER_REPO_PATH]):

---
*Subagent prompt:*

```
A worktree has already been created at: [WORKTREE_PATH]

When you reach Phase 4 (Create Worktree) in impl-plan.md: SKIP IT. The worktree already exists.
In the plan's Header section, write exactly: **Worktree**: [WORKTREE_PATH]

You are a planner. Your job is to produce a thorough, actionable implementation plan. Do not implement anything. Do not modify code files.

**Artifact directories** (specs/, Implementation Plans/, Retros/, Working Logs/, learnings.md) are gitignored and exist only in the main repo at [MASTER_REPO_PATH]. Always write/read these using their absolute path under [MASTER_REPO_PATH], not relative to the worktree.

Read the file at /home/epkone/.claude/commands/impl-plan.md for the full process, then execute it.

The spec to plan is: [SPEC_FILENAME]

After reading impl-plan.md, follow every phase in it exactly.
```
---

Wait for completion. Note the impl plan filename (check `[MASTER_REPO_PATH]/Implementation Plans/` for the most recently modified file if not reported).

**Decision tree:**
- If subagent completes successfully → proceed to Phase 3
- If subagent fails → STOP. Report the error. Do not proceed.

---

## Phase 3 — Impl (Subagent)

Print this banner:

```
══════════════════════════════════════════════════════════════
  PIPELINE PHASE 3 OF 6 — IMPL
  Plan: Implementation Plans/<filename>
  Launching implementer subagent...
  (This is the longest phase — per-step retries enabled)
══════════════════════════════════════════════════════════════
```

Launch a general-purpose Agent with this prompt (fill in [IMPL_PLAN_FILENAME], [WORKTREE_PATH], [MASTER_REPO_PATH]):

---
*Subagent prompt:*

```
The worktree for this work is at: [WORKTREE_PATH]
All file edits and commits must happen inside this worktree, not on master.

You are an implementer. Execute the plan precisely. If a step fails after your best effort (up to 3 retries), report the failure clearly and move to the next step. Do not skip steps silently.

**Artifact directories** (specs/, Implementation Plans/, Retros/, Working Logs/, learnings.md) are gitignored and exist only in the main repo at [MASTER_REPO_PATH]. Always write/read these using their absolute path under [MASTER_REPO_PATH], not relative to the worktree.

Read the file at /home/epkone/.claude/commands/impl.md for the full process, then execute it.

The implementation plan to execute is: [IMPL_PLAN_FILENAME]

After reading impl.md, follow every phase in it exactly. Do not skip the working log. When you reach Phase 7 (Commit Prompt): do NOT ask the user — commit automatically. Stage the files listed in the plan's Scope section and commit with the draft message from the plan. Skip Phase 8 (Merge into Main Project) — the pipeline handles the merge in Phase 6.
```
---

Wait for completion. Note the working log filename (check `[MASTER_REPO_PATH]/Working Logs/` for the most recently modified file).

**Decision tree:**
- If subagent completes (even with some FAILED steps) → proceed to Phase 4
- If subagent crashes or produces no output → STOP. Report: "Impl subagent failed with no output. Last known artifact: [working log if any]." Do not proceed.

---

## Phase 4 — Audit (Subagent)

Print this banner:

```
══════════════════════════════════════════════════════════════
  PIPELINE PHASE 4 OF 6 — AUDIT
  Working log: Working Logs/<filename>
  Launching independent auditor subagent...
══════════════════════════════════════════════════════════════
```

Launch a general-purpose Agent with this prompt (fill in [WORKING_LOG_FILENAME], [MASTER_REPO_PATH]):

---
*Subagent prompt:*

```
You are an independent auditor. Evaluate the implementation against the spec. Do not modify any files. Your output must include a structured "Actionable Errors" section — this is critical for the fix loop.

**Artifact directories** (specs/, Implementation Plans/, Retros/, Working Logs/, learnings.md) are gitignored and exist only in the main repo at [MASTER_REPO_PATH]. Always write/read these using their absolute path under [MASTER_REPO_PATH], not relative to the worktree.

Read the file at /home/epkone/.claude/commands/audit-implementation.md for the full process, then execute it.

The working log to analyze is: [WORKING_LOG_FILENAME]

After reading audit-implementation.md, follow every phase in it exactly.
```
---

Wait for completion. Read the audit document from `[MASTER_REPO_PATH]/Retros/` (most recently modified file). Store its path as `AUDIT_FILENAME`.

**Parse the "Actionable Errors" section:**
- Count the numbered error entries (ignore the "Not actionable" list)

**Decision tree:**
- If 0 actionable errors → skip Phase 5, proceed to Phase 6
- If >= 1 actionable errors → proceed to Phase 5 with `AUDIT_FILENAME`
- If audit subagent fails → report warning, proceed to Phase 6 with note: "Audit failed — fix loop skipped. Review manually."

---

## Phase 5 — Fix Loop (max 2 iterations)

Set `MAX_LOOPS = 2`. Initialize `loop_count = 0`.

### Loop start:

Print this banner:

```
══════════════════════════════════════════════════════════════
  PIPELINE PHASE 5 OF 6 — FIX (loop {loop_count + 1}/{MAX_LOOPS})
  Errors to fix: {error_count}
  Launching fixer subagent...
══════════════════════════════════════════════════════════════
```

Launch a general-purpose Agent with this prompt (fill in [AUDIT_FILENAME], [WORKTREE_PATH], [MASTER_REPO_PATH]):

---
*Subagent prompt:*

```
You are a fixer. Fix only the errors listed in the audit document. Do not add features, refactor, or make improvements beyond what is needed to fix each error.

**Artifact directories** (specs/, Implementation Plans/, Retros/, Working Logs/, learnings.md) are gitignored and exist only in the main repo at [MASTER_REPO_PATH]. Always write/read these using their absolute path under [MASTER_REPO_PATH], not relative to the worktree.

Read the file at /home/epkone/.claude/commands/fix.md for the full process, then execute it.

The audit document is: [AUDIT_FILENAME]

After reading fix.md, follow every phase in it exactly.
```
---

Wait for the fixer to complete.

**Before spawning re-audit:** Briefly scan the fixer's output or its working log (most recently modified file in `[MASTER_REPO_PATH]/Working Logs/` whose name starts with "fixer-"). If the fixer logged "0 errors fixed" or applied no file changes, skip the re-audit subagent and proceed to Phase 6 with remaining errors unchanged — there is nothing new to verify.

### Re-audit:

Print this banner:

```
══════════════════════════════════════════════════════════════
  PIPELINE — RE-AUDIT (after fix loop {loop_count + 1})
  Launching auditor subagent to verify fixes...
══════════════════════════════════════════════════════════════
```

Launch a general-purpose Agent with this prompt (fill in [WORKING_LOG_FILENAME], [AUDIT_FILENAME], [MASTER_REPO_PATH]):

---
*Subagent prompt:*

```
You are an independent auditor verifying that a fixer's changes resolved the errors from a prior audit.

**Artifact directories** (specs/, Implementation Plans/, Retros/, Working Logs/, learnings.md) are gitignored and exist only in the main repo at [MASTER_REPO_PATH]. Always write/read these using their absolute path under [MASTER_REPO_PATH], not relative to the worktree.

Do NOT read /home/epkone/.claude/commands/audit-implementation.md. Follow only the instructions below.

Your tasks:
1. Read the fixer's working log: the most recently modified file in [MASTER_REPO_PATH]/Working Logs/ whose name starts with "fixer-".
2. Read the original audit document: [AUDIT_FILENAME]
3. For each error listed in the "Actionable Errors" section of [AUDIT_FILENAME]: check whether the fixer's stated changes address it. Read the relevant source files to verify.
4. Run the test suite (e.g. pytest) and note the result.
5. Append a new section to [AUDIT_FILENAME] with this exact format:

## Re-Audit (after fix loop {loop_count + 1})
**Date**: YYYY-MM-DD

### What the fixer did
[summary of changes from fixer working log]

### Updated Goals
[only rows that changed status — omit unchanged rows]

### Test suite
[pass/fail/count]

### Remaining Actionable Errors
[numbered list in the same format as the original Actionable Errors section, or "None" if all resolved]
```
---

Parse the "Remaining Actionable Errors" section from the appended content in `[AUDIT_FILENAME]`.

**Persist remaining errors to disk:** Write the remaining error list (or "None") to `[MASTER_REPO_PATH]/Retros/pipeline-remaining-<timestamp>.md`. Phase 6 reads this file — do not rely on session context alone.

**Decision tree after re-audit:**
- If 0 actionable errors → exit loop, proceed to Phase 6
- If errors remain AND `loop_count < MAX_LOOPS - 1` → increment `loop_count`, go to **Loop start**
- If errors remain AND `loop_count >= MAX_LOOPS - 1` → exit loop, proceed to Phase 6 with remaining errors

---

## Phase 6 — Summary

Print this banner:

```
══════════════════════════════════════════════════════════════
  PIPELINE COMPLETE
══════════════════════════════════════════════════════════════
```

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

### Suggest next steps
- "Run manual tests for items marked 'requires play mode' in the audit"
- "Review proposed skill changes in the audit — reply 'apply [title]' or 'apply all'"
- "Run `/learnings-review` to review and apply accumulated pipeline improvement suggestions"
- Any other context-specific suggestions

### Merge worktree to master

Determine whether the work was done in a worktree by reading the `**Worktree:**` field from the impl plan, or by running `git worktree list` and checking if more than one entry exists.

**If in a worktree:**

Use AskUserQuestion to ask the user:

> "Pipeline complete — all changes committed to the worktree branch. Ready to merge to master and delete the worktree? (yes/no)"

Set `MERGE_HAPPENED = false`.

- If **yes**:

  1. Use `WORKTREE_PATH` and `WORKTREE_BRANCH` captured in Phase 1.5. If these are unavailable, re-derive with:
     ```bash
     git worktree list
     ```

  1a. **Check for uncommitted main-repo changes before merging:**
     ```bash
     git -C "MASTER_REPO_PATH" status
     ```
     If any staged or unstaged changes exist in files that overlap with the worktree branch, **do not attempt the merge**. Instead, warn the user and offer two options:
     - **(a)** Commit the main-repo changes as a separate commit first, then rebase the worktree branch onto the new master, then proceed with the merge.
     - **(b)** Skip the automated merge and print the manual steps below.

     Do **not** attempt stash/pop automatically — it creates conflicts that require human resolution.

  2. Rebase the worktree branch onto master (resolves any divergence before the merge):
     ```bash
     git -C "WORKTREE_PATH" rebase master
     ```
     If the rebase has conflicts: describe every conflicting file to the user and stop. Tell them to resolve the conflicts inside the worktree and then run the following manually when ready:
     ```
     git -C "MASTER_REPO_PATH" merge --ff-only WORKTREE_BRANCH
     git worktree remove "WORKTREE_PATH"
     git -C "MASTER_REPO_PATH" branch -d WORKTREE_BRANCH
     git -C "MASTER_REPO_PATH" worktree prune
     ```

  3. Merge into master (fast-forward only — guarantees linear history):
     ```bash
     git -C "MASTER_REPO_PATH" merge --ff-only WORKTREE_BRANCH
     ```
     If this fails (non-fast-forward): tell the user the rebase did not produce a clean fast-forward and they must resolve it manually.

  4. Clean up the worktree:
     ```bash
     git worktree remove "WORKTREE_PATH"
     git -C "MASTER_REPO_PATH" branch -d WORKTREE_BRANCH
     git -C "MASTER_REPO_PATH" worktree prune
     ```

  5. Set `MERGE_HAPPENED = true`. Report: "Merged to master and worktree deleted. Run `git push` when ready to push to remote."

- If **no**:
  Set `MERGE_HAPPENED = false`.
  Report the manual steps so the user can do it later:
  ```
  git -C "WORKTREE_PATH" rebase master
  git -C "MASTER_REPO_PATH" merge --ff-only WORKTREE_BRANCH
  git worktree remove "WORKTREE_PATH"
  git -C "MASTER_REPO_PATH" branch -d WORKTREE_BRANCH
  git -C "MASTER_REPO_PATH" worktree prune
  ```

**If not in a worktree:**

Check for uncommitted changes with `git status`. If any exist, ask the user if they want to commit them. If everything is already committed, skip silently.

Set `MERGE_HAPPENED = true`.

### Update patch notes

**Only run if `MERGE_HAPPENED = true`** (skip if the user declined the merge).

Check whether `package.json` exists at the project root. If it does, invoke the patch-notes skill:

```
Use the Skill tool with skill: "patch-notes"
```

This records every new commit since the last patch notes entry, generates AI-written summaries, bumps the version in `package.json`, and commits `PATCHNOTES.md` + `package.json` automatically. No user input required.

If `package.json` does not exist, skip this step.

### Update learnings.md

Reflect on the pipeline run across all phases. Write only entries that are actionable improvements to the pipeline process itself — not code patterns or project-specific observations.

Format each entry as:
```
## [Short title]
**Phase affected**: [spec / impl-plan / impl / audit / fix]
**What happened**: [one sentence — the slowdown or friction observed]
**Suggestion**: [concrete change to the pipeline prompt or process]
```

Then:
- If `learnings.md` does not exist at the project root: create it with a `# Pipeline Learnings` header
- Append any new entries
- If this run was smooth with nothing notable: skip (do not append empty sections)

**IMPORTANT — DO NOT apply these learnings:**
- Do NOT invoke the learnings-review skill
- Do NOT modify any files under `~/.claude/commands/` or `~/.claude/skills/`
- Do NOT clear or delete existing entries from `learnings.md`
- The sole purpose of this step is to record observations for future human review
