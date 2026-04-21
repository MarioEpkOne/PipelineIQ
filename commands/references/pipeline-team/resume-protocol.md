On any `/pipeline-team` invocation that detects a stale team config in `~/.claude/teams/`:

**Step 1 -- Detect orphans**

Read `~/.claude/teams/` for a config from the crashed session. For each member in `config.json`, check whether the session ID is still alive. Discard orphaned entries -- do not attempt to message dead session IDs.

**Step 2 -- Re-evaluate pending specs**

Combine two signals:
- Specs NOT in `specs/applied/` -> not yet completed.
- `git worktree list` -> specs with an open worktree that started but didn't merge.

For each open worktree, read the most recent working log in `Working Logs/`:
- Working log exists + impl complete -> resume from **audit phase** (spawn audit-only teammate).
- Working log exists + impl incomplete -> restart from **impl-plan phase**.
- No working log -> restart from **impl-plan phase**.

Also check `~/.claude/tasks/<team-name>/` -- if the task list file survives, read it directly rather than re-deriving state.

Special case -- crash while a merge was in progress:
- If the merge appears in `git log` (the commit exists): clean up the worktree, treat spec as completed.
- If no merge commit: worktree is still open, resume from audit phase.

**Step 3 -- Rebuild and continue**

Re-run dependency analysis on the unfinished spec set. Rebuild the task list. Continue from Phase 2. Do not broadcast `MASTER_ADVANCED` -- master state is already current for any new worktrees.
