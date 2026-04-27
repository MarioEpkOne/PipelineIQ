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

## Resolve COMMANDS_PATH

Determine the absolute path to this plugin's `commands/` directory.

Discovery order:
1. Read `~/.claude/plugins/installed_plugins.json`. Parse as JSON.
   Find any key matching `pipelineiq@*` (match plugin name prefix, ignore marketplace suffix).
   If multiple matches: collect all entries across all matching keys, sort by `lastUpdated`
   descending, take the first. Set COMMANDS_PATH = <installPath>/commands.
2. If the registry file does not exist, is unreadable, or contains no `pipelineiq@*` key:
   walk up from CWD looking for `.claude-plugin/plugin.json` where the adjacent `commands/`
   directory contains `pipeline.md`.
   If found: COMMANDS_PATH = <that commands/ directory>
3. If neither found: STOP with "Cannot locate PipelineIQ plugin. Ensure the plugin
   is installed via the plugin marketplace, or you are in the PipelineIQ repo."

Verify: check that COMMANDS_PATH/references/pipeline/ exists.
Store COMMANDS_PATH as an absolute path for use in all subsequent phases.

---

## Pre-flight -- Validate Reference Files

Before starting any phase, verify that all required reference files exist. Check each path and collect any missing files into a list. If any are missing, STOP immediately with the full list — do not start Phase 1.

Required files:
- `[COMMANDS_PATH]/references/pipeline/phase2-impl-plan.md`
- `[COMMANDS_PATH]/references/pipeline/phase3-impl.md`
- `[COMMANDS_PATH]/references/pipeline/phase4-audit.md`
- `[COMMANDS_PATH]/references/pipeline/phase5-fix.md`
- `[COMMANDS_PATH]/references/pipeline/phase5-reaudit.md`
- `[COMMANDS_PATH]/references/pipeline/phase6-merge.md`

On failure: "Missing reference file(s): [list]. These files are required for the pipeline subagent prompts. Check that `[COMMANDS_PATH]/references/pipeline/` exists and contains all 6 phase files."

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

## Model Routing

| Phase | Model | Rationale |
|---|---|---|
| 2 Impl-Plan | opus | Deep reasoning for architecture and planning |
| 3 Impl | sonnet | Mechanical execution of well-specified steps |
| 4 Audit | opus | Independent judgment and architectural review |
| 5 Fix | sonnet | Mechanical execution of targeted fixes |
| 5 Re-audit | sonnet | Verification of specific fix results |

## Phase 1 -- Spec (Interactive, Inline)

--- Phase 1/6: SPEC ---

Run the full spec workflow inline -- do NOT spawn a subagent for this phase, user interaction is required.

> **Existing spec shortcut**: If the remaining argument (after stripping SKIP_MERGE) is a path to an existing `.md` file in `specs/` or `specs/applied/`, skip the spec workflow and go directly to Phase 2 with that path.

### Phase 1a -- Run the Spec Workflow

Read `[COMMANDS_PATH]/spec.md` in full and follow every phase in it exactly. Note the exact filename written to `specs/`.

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

Read `[COMMANDS_PATH]/references/pipeline/phase2-impl-plan.md`.
Read `[COMMANDS_PATH]/impl-plan.md` and use its content to fill [COMMAND_FILE_CONTENT].
Fill in variables: [SPEC_FILENAME], [WORKTREE_PATH], [MASTER_REPO_PATH], [COMMANDS_PATH], [COMMAND_FILE_CONTENT].
Launch a general-purpose Agent with model=opus and the filled prompt.

Wait for completion. Note the impl plan filename (check `[MASTER_REPO_PATH]/Implementation Plans/` for the most recently modified file if not reported).

**Decision tree:**
- If subagent completes successfully -> proceed to Phase 3
- If subagent fails -> STOP. Report the error. Do not proceed.

---

## Phase 3 -- Impl (Subagent)

--- Phase 3/6: IMPL (<plan filename>) ---

Read `[COMMANDS_PATH]/references/pipeline/phase3-impl.md`.
Read `[COMMANDS_PATH]/impl.md` and use its content to fill [COMMAND_FILE_CONTENT].
Fill in variables: [IMPL_PLAN_FILENAME], [WORKTREE_PATH], [MASTER_REPO_PATH], [COMMANDS_PATH], [COMMAND_FILE_CONTENT].
Launch a general-purpose Agent with model=sonnet and the filled prompt.

Wait for completion. Note the working log filename (check `[MASTER_REPO_PATH]/Working Logs/` for the most recently modified file).

**Decision tree:**
- If subagent completes (even with some FAILED steps) -> proceed to Phase 4
- If subagent crashes or produces no output -> STOP. Report: "Impl subagent failed with no output. Last known artifact: [working log if any]." Do not proceed.

**Escalation advisory**: If the impl subagent's working log shows a step that failed 3 retries, log "Model escalation candidate: step N failed 3 retries on Sonnet" in the working log for manual review. No automatic re-dispatch.

---

## Phase 4 -- Audit (Subagent)

--- Phase 4/6: AUDIT (<working log filename>) ---

Read `[COMMANDS_PATH]/references/pipeline/phase4-audit.md`.
Read `[COMMANDS_PATH]/audit-implementation.md` and use its content to fill [COMMAND_FILE_CONTENT].
Fill in variables: [WORKING_LOG_FILENAME], [MASTER_REPO_PATH], [COMMANDS_PATH], [COMMAND_FILE_CONTENT].
Launch a general-purpose Agent with model=opus and the filled prompt.

Wait for completion. Parse the audit subagent's output for the saved audit path (the subagent outputs "Audit saved to `Working Logs/audit-impl--...md`"). Construct the full path as `[MASTER_REPO_PATH]/Working Logs/<parsed-filename>`. Store as `AUDIT_FILENAME`. If the subagent's output does not contain a parseable audit path, fall back to the most recently modified `audit-impl--*.md` file in `[MASTER_REPO_PATH]/Working Logs/`. Log warning: "Could not parse audit path from subagent output -- using fallback."

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

Read `[COMMANDS_PATH]/references/pipeline/phase5-fix.md`.
Read `[COMMANDS_PATH]/fix.md` and use its content to fill [COMMAND_FILE_CONTENT].
Fill in variables: [AUDIT_FILENAME], [WORKTREE_PATH], [MASTER_REPO_PATH], [COMMANDS_PATH], [COMMAND_FILE_CONTENT].
Launch a general-purpose Agent with model=sonnet and the filled prompt.

Wait for the fixer to complete.

**Before spawning re-audit:** Briefly scan the fixer's output or its working log (most recently modified file in `[MASTER_REPO_PATH]/Working Logs/` whose name starts with "fixer-"). If the fixer logged "0 errors fixed" or applied no file changes, skip the re-audit subagent and proceed to Phase 6 with remaining errors unchanged.

### Re-audit:

--- RE-AUDIT (after fix loop {loop_count + 1}) ---

Read `[COMMANDS_PATH]/references/pipeline/phase5-reaudit.md`.
Fill in variables: [WORKING_LOG_FILENAME], [AUDIT_FILENAME], [MASTER_REPO_PATH], [COMMANDS_PATH], {loop_count}.
Launch a general-purpose Agent with model=sonnet and the filled prompt.

Parse the "Remaining Actionable Errors" section from the appended content in `[AUDIT_FILENAME]`.

**Note for Phase 6:** The re-audit appended a "Remaining Actionable Errors" section to `AUDIT_FILENAME`. Phase 6 reads remaining errors from there.

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
- Audit(s): `Working Logs/<filename>` (and any re-audit files)
- Fixer log(s): `Working Logs/fixer-log--<filename>` (if fix loop ran)

### Report remaining issues
If the fix loop ran and errors remain after 2 cycles:
- Read `AUDIT_FILENAME` and parse the last "### Remaining Actionable Errors" section (from the most recent re-audit addendum). If the section says "None", treat as 0 remaining errors.
- List each remaining error (title + brief description)
- Say: "N errors remain after 2 fix cycles. Review the final audit at `Working Logs/<filename>` and fix manually or run `/fix <filename>` again."

### Update learnings.md

Reflect on the pipeline run across all phases. Identify only actionable improvements to the pipeline process itself -- not code patterns, not project-specific observations. A pipeline learning is something that, if addressed, would make future pipeline runs across ANY project work better.

**Step 1 -- Read existing learnings.md**

Read `learnings.md` at `[MASTER_REPO_PATH]/learnings.md`. If it does not exist, create it with the template:

```
# Pipeline Learnings

## HIGH

---

## MEDIUM

---

## LOW
```

Parse all existing entries, noting their titles, phases, occurrence counts, and severity tiers.

If the file exists but entries lack **Occurrences** / **Seen in** / **Suggested diff** fields (old format), migrate each entry before proceeding:
- Set Occurrences to 1
- Set Seen in to "pre-migration"
- Set severity to LOW
- Read the target command file referenced in the suggestion and generate a diff
- Rewrite the entire file in the new format (severity-tiered with all required fields)

**Step 2 -- Generate observations**

For this pipeline run, identify any friction, slowdowns, or failures attributable to the pipeline process (not the code being implemented). Consider:
- Did the impl plan miss something the spec required?
- Did the impl agent deviate from the plan? Why?
- Did the audit catch real issues? Did it flag false positives?
- Did the fix loop resolve errors, or did it churn?
- Was a phase unnecessarily slow or redundant?

For each observation, determine:
- Which command file in `commands/` would need to change to prevent this (diffs target `commands/` only -- no diffs for skills, CLAUDE.md, or files outside the pipeline command directory)
- What the specific change would be

If this run was smooth with nothing notable: skip to the end (do not write empty entries).

**Step 3 -- Deduplicate against existing entries**

For each new observation, read through existing entries in learnings.md and determine whether the observation describes the same class of problem as an existing entry. Use semantic judgment -- different phrasing of the same underlying issue counts as a match. For example, "Plan Scope misses sync targets" and "Impl plan scope didn't include commands/ files" are the same class of problem.

- **If match found**: This is a recurrence. Proceed to Step 4 (update existing entry).
- **If no match**: This is a new observation. Proceed to Step 5 (create new entry).

When uncertain whether two observations are the same problem, err on the side of creating a new entry. False separation is safer than false deduplication.

**Step 4 -- Update existing entry (recurrence)**

For the matched entry:
1. Increment the occurrence count
2. Add the spec/batch identifier to the **Seen in** list
3. Update **What happened** to incorporate the new observation if it adds useful detail (do not just repeat -- enrich)
4. Recalculate severity: 1 = LOW, 2-3 = MEDIUM, 4+ = HIGH
5. If severity changed: move the entry to the correct tier section
6. Re-read the target command file (the one in **Suggested diff**) and regenerate the diff against its current state.

**One pipeline run = one occurrence per learning.** Even if the same problem manifested multiple times within this single run, count it as one occurrence.

**Step 5 -- Create new entry**

1. Read the target command file to draft an exact diff
2. Run `date '+%Y-%m-%d %H:%M'` to get the current date-and-time.
3. Create a new entry under the **LOW** section with:
   - Title (short, descriptive)
   - Date: the date-and-time from the command above
   - Phase affected
   - Occurrences: 1
   - Seen in: spec/batch identifier (no date — the Date field covers that)
   - What happened (one sentence)
   - Suggestion (concrete change)
   - Suggested diff (file path in `commands/` + diff block). If the target file cannot be read (missing, wrong path), set Suggested diff to "Unable to generate -- target file not found: [path]". Still track the observation.

**Step 6 -- Write learnings.md**

Rewrite the full `learnings.md` with all entries properly sorted:
- HIGH tier first, then MEDIUM, then LOW
- Within each tier: highest occurrence count first, then most recent date
- Preserve the `---` separators between tiers

Each entry must follow this exact format:

```
### [Title]
**Date**: [YYYY-MM-DD HH:MM]
**Phase affected**: [phase]
**Occurrences**: [N]
**Seen in**: [spec-name, spec-name, ...]
**What happened**: [description]
**Suggestion**: [concrete change]
**Suggested diff**:
File: `commands/[filename]`
```diff
- old line
+ new line
```
```

**Important constraints:**
- Do NOT invoke /learnings-review or modify any command/skill files
- Do NOT delete or discard existing entries (even if they seem stale)
- Do NOT create entries about code quality, project architecture, or implementation patterns -- only pipeline process friction
- If an observation matches an entry already in `promotion-log.md` (a previously applied fix), this is a regression: create a new entry at LOW with a note: "Previously applied on [date] -- see promotion-log.md. Recurrence suggests the fix was insufficient."
- If the target file has changed so much a diff concept no longer applies, regenerate from scratch. If the problem appears fixed, note: "May be resolved -- verify before applying" but do NOT remove the entry.
- If the pipeline ran inside a project other than PipelineIQ, include the project name or path in each **Seen in** entry (e.g., "spec--widgets, project: /path/to/other") to distinguish cross-project observations.

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
  - Read `[COMMANDS_PATH]/references/pipeline/phase6-merge.md`.
  - Follow the merge workflow. Variables: WORKTREE_PATH, WORKTREE_BRANCH, MASTER_REPO_PATH.
  - Suggest next steps:
    - "Run manual tests for items marked 'requires runtime verification' in the audit"
    - "Review proposed skill changes in the audit -- reply 'apply [title]' or 'apply all'"
    - "Run `/learnings-review` to review and apply accumulated pipeline improvement suggestions"
    - Any other context-specific suggestions

### Cleanup (success only)

If remaining errors = 0 (all errors resolved or no errors were found):

Delete the following artifacts from this pipeline run:
- `[MASTER_REPO_PATH]/Working Logs/wlog--*` matching this run's description slug
- `[MASTER_REPO_PATH]/Working Logs/audit-impl--*` matching this run's description slug
- `[MASTER_REPO_PATH]/Working Logs/fixer-log--*` matching this run's description slug
- `[MASTER_REPO_PATH]/Implementation Plans/impl--*` matching this run's description slug

Match by description slug (the text portion of the timestamp-prefixed filename) to avoid deleting artifacts from concurrent runs.

If remaining errors > 0: skip cleanup. Say: "Artifacts preserved for manual review -- run `/fix <audit-filename>` to address remaining errors."
