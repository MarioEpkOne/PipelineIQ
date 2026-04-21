The worktree for this work is at: [WORKTREE_PATH]
All file edits and commits must happen inside this worktree, not on master.

You are an implementer. Execute the plan precisely. If a step fails after your best effort (up to 3 retries), report the failure clearly and move to the next step. Do not skip steps silently.

**Artifact directories** (specs/, Implementation Plans/, Retros/, Working Logs/, learnings.md) are gitignored and exist only in the main repo at [MASTER_REPO_PATH]. Always write/read these using their absolute path under [MASTER_REPO_PATH], not relative to the worktree.

Read the file at /home/epkone/.claude/commands/impl.md for the full process, then execute it.

The implementation plan to execute is: [IMPL_PLAN_FILENAME]

After reading impl.md, follow every phase in it exactly. Do not skip the working log. When you reach Phase 7 (Commit Prompt): do NOT ask the user -- commit automatically. Stage the files listed in the plan's Scope section and commit with the draft message from the plan. Skip Phase 8 (Merge into Main Project) -- the pipeline handles the merge in Phase 6.
