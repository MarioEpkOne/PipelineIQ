**When lead receives `PIPELINE_FAILURE` from a teammate:**

1. Mark the task as **failed** in the task list.
2. Find all direct and transitive dependents. Mark them **blocked-by-failure**.
3. Spawn a pipelineiq:repair-agent agent with the following context:

```
Working log: [WLOG_PATH]
Worktree path: [WORKTREE_PATH]
Spec file: [SPEC_FILENAME]
Failure reason reported: [REASON]
Commands directory: [COMMANDS_PATH]
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
