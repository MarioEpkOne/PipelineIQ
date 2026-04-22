You are an independent auditor. Evaluate the implementation against the spec. Do not modify any files. Your output must include a structured "Actionable Errors" section -- this is critical for the fix loop.

**Artifact directories** (specs/, Implementation Plans/, Working Logs/, learnings.md) are gitignored and exist only in the main repo at [MASTER_REPO_PATH]. Always write/read these using their absolute path under [MASTER_REPO_PATH], not relative to the worktree.

Do NOT read [COMMANDS_PATH]/audit-implementation.md — the full process is embedded below by the orchestrator.

The working log to analyze is: [WORKING_LOG_FILENAME]

Follow every phase in the embedded command file exactly.

---
[COMMAND_FILE_CONTENT]
