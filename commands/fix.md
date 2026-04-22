TARGETED FIX — reads an audit document's "Actionable Errors" section and fixes each error. Can be used standalone or invoked by the pipeline.

The user's request is: $ARGUMENTS

---

## Phase 1 — Find Inputs

Search `Working Logs/` for an audit document whose filename starts with `audit-impl--` and contains the argument (case-insensitive).

- If multiple matches: list them and ask the user to pick one.
- If no match and argument looks like a file path: try reading it directly.
- If no match: tell the user no audit found and exit.
- If no argument: use the most recently modified file in `Working Logs/` whose name starts with `audit-impl--`.

Read the audit document in full.

Parse the **"Actionable Errors"** section. If no such section exists: tell the user "This audit does not have a structured Actionable Errors section. Run `/audit-implementation` with the latest skill version to generate one." and exit.

From the audit header, read:
- The referenced impl plan from `Implementation Plans/`
- The referenced working log from `Working Logs/` (if it exists)

---

## Phase 2 — Read Context

For each actionable error entry, read every file listed in its **File(s)** field.

If the audit references a worktree in its header, resolve and use that worktree as the working root for file reads and edits.

---

## Phase 3 — Fix Each Error

For each actionable error (skip entries under "Not actionable"):

1. Read the relevant file(s) fresh (they may have changed since the audit)
2. Apply the suggested fix using Edit/Write
3. After each fix: verify no errors were introduced by running the project's build/lint/test command as appropriate
4. If a fix introduces new errors:
   - Revert the change
   - Try a different approach (max 2 attempts per error)
   - If both attempts fail: mark as "fix failed" with explanation
5. If the suggested fix doesn't apply (file changed, method renamed, etc.): mark as "deferred to user" with explanation

Log each fix result as you go — do not batch.

---

## Phase 3.5 — Handle Test Assertions Requiring Out-of-Scope Production Changes

Before applying a fix, check whether the suggested fix requires modifying production code that is outside the current spec's scope. This situation arises when:
- The audit flags a failing test assertion
- The only way to make the assertion pass is to change production behavior (not just the test)
- The production change was not part of the spec being implemented

When this condition is met:
1. Do **not** modify production code
2. Update the test assertion to match the actual (correct) production behavior, with a comment: `# Assertion updated to match actual behavior — production change is out of scope for this spec`
3. Log the entry under **Skipped (Product Decision)** in the fixer log (see Phase 4 format below), with: what the original assertion expected, what the production code actually does, and why changing production is out of scope
4. The re-audit **must** treat this as resolved (0 remaining errors for this item) — the fixer log entry is the paper trail

---

## Phase 4 — Write Fixer Log

Before writing the file, run `date '+%Y-%m-%d--%H-%M'` to get the actual current timestamp. Save to `Working Logs/fixer-log--YYYY-MM-DD--HH-MM--<description>.md` (replacing `YYYY-MM-DD--HH-MM` with the output of that command). Use the same description slug as the audit document.

**Required structure:**

```markdown
# Fixer Log
**Date**: YYYY-MM-DD
**Audit**: Working Logs/<audit-filename>
**Impl plan**: Implementation Plans/<impl-plan-filename>

## Fixes Applied
- `<file>`: <what was changed and why>

## Skipped (Not Actionable)
- <description and why>

## Skipped (Fix Failed)
- <description, what was tried, why it failed>

## Skipped (Product Decision)
- <error title>: Original assertion expected `<X>`. Production code actually does `<Y>` — this is correct behavior. Changing production is out of scope for this spec. Test assertion updated to match actual behavior.

## Deferred to User
- <description and what the user needs to decide>
```

---

## Phase 5 — Commit

After writing the fixer log, stage all modified files and commit. Stage each file explicitly by name — never use `git add -A` or `git add .`.

```bash
git add <fixed-file-1> <fixed-file-2> Working Logs/fixer-log--...md
git commit -m "$(cat <<'EOF'
fix: <N> audit errors resolved

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
EOF
)"
```

> **Worktree commit hook caveat**: If working in a worktree, bare `git commit` is blocked by a hook reading the main repo CWD. Use `git -C <worktree-path> add <files>` and `git -C <worktree-path> commit -m "..."` instead.

---

## After Writing

Tell the user:
- Fixer log saved to `Working Logs/fixer-log--...md`
- Summary: N fixed, N skipped, N failed, N deferred
- If any errors were deferred: list them with brief descriptions
- If all errors were fixed: "All actionable errors resolved. Run `/audit-implementation` again to verify."
