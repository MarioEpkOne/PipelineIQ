Context for this run:
- Worktree: [WORKTREE_PATH]
- Implementation plan: [IMPL_PLAN_FILENAME]
- Main repo: [MASTER_REPO_PATH]
- Commands directory: [COMMANDS_PATH]

Phase-specific overrides:
- When you reach Phase 7 (Commit Prompt): do NOT ask the user -- commit automatically. Stage the files listed in the plan's Scope section and commit with the draft message from the plan.
- Skip Phase 8 (Merge into Main Project) -- the pipeline handles the merge in Phase 6.
