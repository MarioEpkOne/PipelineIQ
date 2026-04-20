EXECUTE PLAN — reads an existing implementation plan and runs all steps autonomously, verifies each step, writes a working log, and prompts to commit.

The user's request is: $ARGUMENTS

> Can be run from the main repo or from inside the worktree directory. If the plan has a `**Worktree:**` header, the skill automatically resolves and uses that worktree as its working root.

## Process

### Phase 1 — Find the Implementation Plan

Search `Implementation Plans/` (relative to the current working directory) for a file whose name contains the argument (case-insensitive).

- If multiple matches: list them and ask the user to pick one.
- If no match: tell the user to run `/impl-plan` first and exit.
- If no argument was provided: list all impl plans in `Implementation Plans/` and ask the user which one to execute.

Read the full implementation plan.

### Phase 2 — Verify Worktree

Read the `**Worktree:**` field from the plan header.

- **No worktree field in the plan:** proceed from the current working directory.
- **Worktree field present, CWD already ends with that slug:** proceed normally.
- **Worktree field present, CWD is the main repo:** resolve the absolute path of the worktree (the field value is relative to the main repo root, e.g. `.claude/worktrees/my-feature/`). Check whether `<worktree_abs_path>/Implementation Plans/<same-filename>` exists — if it does, re-read the plan from there (the worktree copy may differ from the main repo copy, so prefer it). Whether or not a worktree copy exists, use `<worktree_abs_path>` as the working root for all subsequent file reads, writes, and relative path references in this session. Do **not** exit — continue with the next phase.

### Phase 3 — Read Reading List (and optionally learnings.md)

Read every file listed in the plan's **Reading list** section, in order.

Do not read any other files unless a specific step explicitly requires it and explains why.

**Optional — learnings.md**: If `learnings.md` exists in the project root and the task involves an area with known gotchas (prefabs, RectTransform, layout), skim it for relevant entries. Treat them as risks to watch for, not hard rules.

**Staleness check**: As you read each file, compare against any "Current value" fields in the plan's steps. If a plan step says "Current value: `barRef` is null" but you read the file and `barRef` is already set, or a method the plan references has been renamed/removed — stop and warn the user before proceeding. The plan may be outdated.

**Worktree baseline mismatch check**: If the worktree's files differ structurally from what the plan expects (e.g., the worktree was branched from an earlier state of master and is missing fields, rows, or features that were added to master after the branch was created), do not silently proceed. Enumerate every missing element you find and confirm with the user whether to (a) port the missing elements before executing the plan, or (b) treat the omissions as intentional. Implementing a new feature on top of a diverged baseline that silently drops pre-existing functionality is a regression even if the new feature works.

### Phase 4 — Execute Steps Autonomously

Execute each step in order. Do not skip steps.

After each step:
- Run `read_console logType='error'`
- If compile errors exist: diagnose and fix before proceeding to the next step. Do not accumulate errors across steps.
- For prefab and scene steps: use MCP tools only. Never use Edit or Write on `.prefab` or `.unity` files.
- Before any MCP scene or prefab step that follows a script edit: poll `mcpforunity://editor_state` until `isCompiling == false`, then run `read_console logType='error'` to confirm clean compile before proceeding.

**Per-step retry behavior:**
- After each step, verify no errors were introduced using whatever build/lint/test command is appropriate for the project (see the plan's "Verification Approach" section for guidance).
- If a step fails (compilation error, test failure, MCP error, unexpected state):
  1. Record the error message, which step failed, and what was tried
  2. If this is attempt 1 or 2 of 3: retry the step with a **fresh approach**. Do NOT repeat the exact same action. Think about *why* it failed and try differently. Include the error context from the previous attempt in your reasoning.
  3. If this is attempt 3 of 3: mark the step as **FAILED** in the working log under "Errors Encountered". Record the error, all 3 attempts, and why each failed. Then **continue to the next step** — do not stop the entire implementation.
- The working log must record for each retried step: what error occurred on each attempt, what was tried differently, and what finally worked (or that all 3 attempts failed).

**Build tool unavailable in this environment**: If the spec-required build command fails because the build tool itself cannot run (e.g., a native binary is missing, rollup fails on WSL with `@rollup/rollup-linux-x64-gnu` not found), do NOT silently accept a stale distributable as "built." Either: (a) find an alternative build path that achieves the same result, or (b) explicitly flag the distributable as stale in the working log with: what command failed, why the tool can't run, and what the user must do to produce a valid build. A spec-required build that fails = `INCOMPLETE_TASK` unless it is explicitly deferred with user-actionable instructions.

**Important**: The old behavior was to stop and ask the user on any failure. The new behavior is: retry up to 3 times, then mark as failed and continue. This gives a complete picture of what works and what doesn't. Only stop the entire implementation if you cannot make *any* progress (e.g., every remaining step depends on the failed step).

**Verify external library API signatures**: For any external library call shown in the spec or impl plan (Playwright, httpx, etc.), verify the target language's method signature before implementing. If the spec was written against a different language's API (e.g. JavaScript Playwright docs used in a Python project), adapt the call to the correct language equivalent — do not reproduce a spec code example verbatim without confirming the signature exists in the installed version.

**Correcting buggy test assertions in the plan**: If a plan step contains a test whose assertion directly contradicts the comment or description explaining the expected behavior, the comment/description is the authoritative intent. Fix the assertion to match the stated intent, document the correction in the working log's "Deviations from Plan" section, and continue. Do not faithfully reproduce a buggy assertion — a test that always passes (or always fails) because its assertion is wrong is worse than no test.

**Stale test names when spec prohibits test file changes**: If the spec says "no test file changes" and you therefore cannot rename or update test functions, you may leave behind test names that describe behavior that was removed or changed. Do not silently skip this. In the working log's "Deviations from Plan" section, list each stale test name, what behavior it originally described, and mark it: `documentation debt — test name describes removed behavior; rename when test file changes are permitted`.

**Correcting plan errors about the environment**: If a plan step assumes a dependency, fixture, or config that does not actually exist in the target environment, the correct response is:
1. Verify the gap (not just guess — e.g. `pip show <pkg>`, read `pyproject.toml`, run the failing test first so the error is on the record).
2. Make the MINIMAL correction needed to restore the step's intent — do not expand scope ("while I'm here, let me migrate all tests to asyncio").
3. Document the deviation in the working log's "Deviations from Plan" section with: (a) what the plan said, (b) what was actually true, (c) what you changed, (d) why the change preserves the plan's intent.
4. Do NOT silently skip the step. Do NOT fall back to "just make it work" without recording what was swapped out.

**Layout step verification**: For any step that modifies `sizeDelta`, `anchoredPosition`, or anchors on a Canvas element, `read_console logType='error'` is not sufficient verification. Run the editor-mode layout measurement script described in CLAUDE.md → *Layout Cannot Be Verified with MCP Inspection Alone* and confirm the logged corners are on-screen. Only mark the step verified when the measurement confirms it.

**Conditional steps that depend on visual play-mode observation**: If a step in the plan says "only do X if you see Y in play mode," you must not skip it based on static reasoning. Either: (a) execute the editor measurement script to check the condition programmatically, or (b) mark the step explicitly in the working log as "deferred to user — requires visual verification" AND include the exact remediation instruction (what menu item or script to run if the condition is true).

**Hand-offs to the user must include remediation**: When a checklist item cannot be verified automatically (e.g. requires navigating game flow), record in the working log: (1) exactly what to look for, and (2) what step to run if the check fails. A bare "handed off to user" with no next-step instruction is not acceptable.

If the plan says line ~42 and the actual line is 45: adjust and document the deviation. Do not fail over line number drift.

### Phase 5 — Post-Implementation Checklist

Run through every item in the plan's **Post-Implementation Checklist**.

Report pass or fail for each item. If any item fails, fix it before writing the working log.

**Pre-existing test failures must be disclosed**: If running the test suite reveals any failures — including timeouts, flaky tests, or failures unrelated to this feature — you must check whether the same failures exist on `main` before writing the working log. If they are pre-existing, record them in the working log's "Errors Encountered" section as: `pre-existing: also fails on main — <test name or description>`. Never write "0 failures" or report a clean test run if any tests are failing, even if they appear unrelated to the current change.

**Mandatory: tick every item you verify.** Replace `[ ]` with `[x]` in the checklist for each item that passes. Do not leave passing items unchecked — the pipeline's fix-loop reads unchecked boxes as incomplete work and will flag them as `INCOMPLETE_TASK` even if the work is done. Only items that genuinely fail or are deferred should remain `[ ]`.

### Phase 6 — Write Working Log

Before writing the file, run `date '+%Y-%m-%d--%H-%M'` to get the actual current timestamp. Save to `Working Logs/wlog--YYYY-MM-DD--HH-MM--<description>.md` in the **main repo** `Working Logs/` directory (replacing `YYYY-MM-DD--HH-MM` with the output of that command). Even when executing inside a worktree, use the absolute path to the main repo: `<main-repo-root>/Working Logs/wlog--...md`. The worktree's own copy of the directory is not the canonical location and the pipeline's Phase 4 glob will not find it there.

**Out-of-scope file disclosure**: Every file created or modified during implementation — including files outside the impl plan's Scope — must appear in "Changes Made". Any file not listed in the plan's "Scope — files in play" must also appear in "Deviations from Plan" with a one-sentence justification. Creating or modifying a file outside plan scope without disclosing it is a process violation.

**Required structure:**

```markdown
# Working Log: <Feature Name>
**Date**: YYYY-MM-DD
**Worktree**: .claude/worktrees/<slug>/
**Impl plan**: Implementation Plans/impl--...md

## Changes Made
- `Assets/Scripts/Foo.cs`: Added `DoThing(int, string)` method to FooClass
- `Assets/Scripts/Bar.cs`: Modified `HandleEvent()` — added call to DoThing
- `Assets/Resources/SomePrefab.prefab`: Set FooClass.barRef via MCP

## Errors Encountered
- <any error that occurred + how it was resolved, or "None">
- For retried steps: list each attempt (1/3, 2/3, 3/3) with what was tried and the result
- For failed steps: mark as FAILED with all 3 attempt details

## Deviations from Plan
- <any steps that differed from the plan + why, or "None">

## Verification
- Compile: OK
- Play mode: <describe what was observed>
```

Write the working log even if the user declines to commit.

**The working log is mandatory and must be written before Phase 7.** Do not proceed to the commit prompt without it. If you reach Phase 7 and no working log file exists in `Working Logs/`, stop and write it now before asking about committing. Committing directly without a working log is a process violation.

### Phase 7 — Commit Prompt

Ask: "Implementation complete — commit now?"

If yes: stage only the files listed in the plan's **Scope** section, then commit. Never use `git add -A`, `git add .`, or any wildcard staging — these pull in working logs, process artifacts, and other unintended files. Stage each path explicitly by name.

> **Worktree commit hook caveat**: When working in a worktree, bare `git commit` is intercepted by a harness hook that reads the main repo CWD (always on master) and blocks the commit. Do NOT retry the same command. Instead, tell the user to run the commit manually from a terminal outside the harness using: `git -C <worktree-path> commit -m "..."`. Stage files the same way: `git -C <worktree-path> add <file>`. Include this exact command in your response so the user can paste it directly.

```bash
git commit -m "$(cat <<'EOF'
<draft commit message from impl plan, edited if needed>

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
EOF
)"
```

If no: tell the user the working log has been written and they can commit manually with `/commit`.

### Phase 8 — Merge into Main Project

After committing (or if the user declined but changes are committed), merge the worktree branch into the main project so Unity reflects the final result.

1. Get the worktree's current branch name:
   ```bash
   git -C "C:\Users\Epkone\Humble beginnings\.claude\worktrees\<slug>" branch --show-current
   ```

2. Merge into the main project:
   ```bash
   git -C "C:\Users\Epkone\Humble beginnings" merge <worktree-branch>
   ```

3. If the merge succeeds: tell the user **"Unity will now reload with all changes from this session."**

4. If the merge has conflicts: report exactly which files conflict and tell the user to resolve them manually in `C:\Users\Epkone\Humble beginnings` before opening Unity.

5. If the user declined to commit in Phase 7 and there are no committed changes in the worktree: skip the merge and tell the user they can run `/commit` first, then merge manually with `git -C "C:\Users\Epkone\Humble beginnings" merge <worktree-branch>`.
