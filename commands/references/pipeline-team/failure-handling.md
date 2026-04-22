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
3. Apply the minimum fix. Follow the fix logic in [COMMANDS_PATH]/fix.md.
4. Report: REPAIR_SUCCESS or REPAIR_FAILURE with a brief explanation.
   Do not audit. Do not merge. Just fix and report.
```

**If repair reports `REPAIR_SUCCESS`:**
- Spawn a fresh audit-phase teammate for this spec (pass existing worktree path).
- Audit teammate runs phases 4-5 only (audit + fix loop, max 2 cycles), then sends `PIPELINE_COMPLETE`.

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
