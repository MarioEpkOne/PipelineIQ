# Re-Audit Verification Process

Read the fixer's working log from `[MASTER_REPO_PATH]/Working Logs/` to understand what was changed. Find the most recently modified file whose name starts with "fixer-".

## Steps

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
[only rows that changed status -- omit unchanged rows]

### Test suite
[pass/fail/count]

### Remaining Actionable Errors
[numbered list in the same format as the original Actionable Errors section, or "None" if all resolved]
