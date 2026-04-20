PLAN ONLY — reads a spec, orients in the codebase, creates a worktree, and writes a self-contained implementation plan. Manual handoff: you open a new session and run /impl yourself.

The user's request is: $ARGUMENTS

## Constraints

Do not modify any codebase files during this skill. Your only written output is the implementation plan document (and the worktree creation via EnterWorktree).

## Process

### Phase 1 — Find the Spec

Search the `specs/` directory for a file whose name contains the argument (case-insensitive match on the description slug).

- If multiple matches: list them and ask the user to pick one.
- If no match: tell the user to run `/spec` first and exit.
- If no argument was provided: list all specs in `specs/` and ask the user which one to plan.

Read the full spec document.

### Phase 2 — Orient in the Codebase

From the spec's **Technical Design** section, extract every file that will change.

Read every file in that list fully before writing anything. For Unity scene or prefab changes, also use MCP tools to confirm live state:

```
find_gameobjects name='...'
manage_prefabs action='get_hierarchy' prefab_path='...'
manage_scene action='get_info'
```

> **Worktree path rule**: When reading `.prefab` or `.unity` files to verify current property values, always read from the **main project path** (`C:\Users\Epkone\Humble beginnings\Assets\...`), never from a worktree-relative path. Worktrees are branch snapshots; MCP and Unity operate on the main project. This step happens before the worktree is created, so use the main project path unconditionally.

If the live state diverges from the spec's **Current State** section (e.g., a method was renamed, a file was deleted, a component is missing), flag the divergence explicitly and ask the user how to proceed before continuing.

### Phase 3 — Write the Implementation Plan

Before writing the file, run `date '+%Y-%m-%d--%H-%M'` in the project root to get the actual current timestamp. Save the plan to `Implementation Plans/impl--YYYY-MM-DD--HH-MM--<description>.md` in the **main project directory** (replacing `YYYY-MM-DD--HH-MM` with the output of that command). This is written before entering the worktree so the plan is visible from the main project and can be found by `/impl` without switching directories.

The plan must be self-contained — the implementing agent reads only the files listed in the reading list and executes only what is written in the steps. No free exploration.

> **Worktree path must match reality**: The `**Worktree**:` line in the plan header must point to a worktree that actually exists on disk when the implementing agent runs it. Complete Phase 4 (EnterWorktree) successfully before saving the final plan. If EnterWorktree fails, omit the Worktree line and document the failure — do not write a path that doesn't exist. An implementing agent that reads a declared-but-missing worktree path will waste turns trying to find it.

> **"Current value" snippets must come from a fresh Read in this session.** When you include a "Current value" code block in a step (typical for surgical Edit operations), you MUST have called `Read` on the target file in the same session as writing this plan. Do not copy from a previous plan, the spec, an older audit, or memory — source files drift between when a feature is specced and when it is planned, and a stale snippet will cause the implementer to either (a) waste a turn resolving the discrepancy, or (b) silently rewrite the wrong region if `apply_text_edits` (line-number-based) is used instead of `Edit` (string-anchor-based). Tag the snippet `**Current value (verified from <main-project absolute path>):**` so the implementer can trust it.

> **Prefab full-rebuild ordering rule**: If the plan includes both a targeted prefab property fix (e.g. a temp `FixX.cs` script that sets `anchoredPosition`) and a full-rebuild editor script for the same prefab (one that calls `PrefabUtility.SaveAsPrefabAsset` on a newly created `GameObject`), the rebuild will clobber the targeted fix. Either: (a) bake the fix values directly into the rebuild script and omit the separate fix step, or (b) apply the targeted fix **after** the rebuild step (place it later in the plan). Never put a targeted fix step before a full-rebuild step for the same prefab.

> **Verified baseline rule**: For every step that mutates a value from A → B (e.g. `sizeDelta`, `preferredHeight`, a script field), the step must state the **verified current value of A** and the source path it was read from (always the main project path). If you cannot confirm the current value during Phase 2, add an explicit read/verify sub-step before the mutation step in the plan. The implementing agent must be able to spot a mismatch between the plan's stated baseline and reality before touching anything.

> **Test infrastructure claims must be verified, not assumed.** When the plan references a test dependency (`pytest-asyncio`, `hypothesis`, a custom fixture, etc.), verify it is actually installed/configured before writing a plan step that relies on it. Check `pyproject.toml`, `pytest.ini`, or run `pip show <pkg>` in the target venv. Do NOT infer availability from the mere existence of a test file with the relevant import — the import may be dead, commented, or the file may be broken in CI and nobody noticed. Record the check in the plan's Reading List or a dedicated "Environment assumptions verified" section.

**Required structure:**

```markdown
# Implementation Plan: <Feature Name>

## Header
- **Spec**: specs/spec--<date>--<description>.md
- **Worktree**: .claude/worktrees/<slug>/
- **Scope — files in play** (agent must not touch files not listed here):
  - Assets/Scripts/Foo.cs
  - Assets/Scripts/Bar.cs
  - Assets/Resources/SomePrefab.prefab  ← via MCP only, never Edit/Write
- **Reading list** (read these in order before starting, nothing else):
  1. Assets/Scripts/Foo.cs
  2. Assets/Scripts/Bar.cs

## Steps

### Step N: <Short title>
**File**: `Assets/Scripts/Foo.cs`
**Location**: Class `FooClass`, after line ~42 (after `void Start()`)
**Action**: Add method

**Signature**:
\`\`\`csharp
private void DoThing(int param1, string param2)
\`\`\`

**What it does**: <1-2 sentence description of behavior>

**Before/after** (only for changes to existing methods or complex logic):
\`\`\`
// BEFORE
void OldMethod() { ... }

// AFTER
void OldMethod() { newBehavior(); ... }
\`\`\`

> **Docstring rule**: If this step changes an existing function's observable behavior, add an explicit sub-action: "Update docstring/comments to reflect new behavior." Do not rely on the auditor to catch stale docs.

> **Return type rule**: If this step changes the return type of a public function, add an explicit sub-action: "Update all existing tests that call this function to match the new return type." Do not leave test breakage to be discovered later.

**Verification**: Compile OK, no console errors, <specific observable behavior>

---

### Step N+1: Wire component in prefab
**Tool**: `manage_prefabs action='modify_contents'`
**Target**: `Resources/PlayerPrefab.prefab`
**Action**: Set property `componentType='FooClass'` `propertyName='barRef'` `value='Bar'`
**Current value** (verified from `C:\Users\Epkone\Humble beginnings\Assets\Resources\PlayerPrefab.prefab`): `barRef` is null
**Verification**: Prefab hierarchy shows FooClass.barRef is not null in play mode

---

## Post-Implementation Checklist
- [ ] All files compile (read_console logType='error' returns empty)
- [ ] Play mode entry: no errors
- [ ] <specific observable behavior 1>
- [ ] <specific observable behavior 2>

## Verification Approach
Describe how the implementing agent should verify each step succeeds. Examples:
- "Run `dotnet build` after each C# file change"
- "Run `npm run lint` after each TypeScript change"
- "Check Unity console for errors after each script edit"
- "Run `cargo check` after each Rust file change"

If no specific command is known, state: "Agent should determine the appropriate verification method based on the project type."

## Commit Message (draft)
feat: <short summary under 72 chars>

<longer body explaining what was done and why>
```

### Phase 3.5 — Self-Review: Coverage Check Before Saving

Before saving the plan, verify it covers every requirement in the spec. This is a quick mechanical pass — you already have the spec and the draft plan in context.

Go through each spec section and check coverage:

| Section | What to check |
|---|---|
| **Goal** | Every stated objective has at least one plan step that achieves it |
| **Decisions** | Every decision's *chosen* approach is reflected in the plan — not an alternative |
| **Technical Design** | Every file change, new class, method, or component has a corresponding step |
| **Edge Cases** | Every edge case has a step or explicit handling (not just mentioned). If any edge case contradicts the Technical Design code sketch, implement the edge case — code sketches are illustrative, the edge cases table is authoritative. When writing test steps for edge cases, cross-check each test input against **all upstream guard conditions** in the implementation path (e.g., a length threshold that fires before a classifier regex). If a guard would preempt the spec-required behavior for that input, surface the conflict explicitly — do not silently substitute a different test input that bypasses the guard. A test that passes because it avoids the guard is not evidence that the spec behavior is implemented. |
| **Constraints** | Every constraint is enforced somewhere in the steps or post-implementation checklist |
| **Success Criteria** | Cross-check every Success Criteria bullet against the logic/design sections of the spec to confirm each criterion has a corresponding implementation path. Flag any criterion with no matching code path as a spec inconsistency before the impl starts — do not leave it to the auditor to discover. |
| **Testing Strategy** | Every verification criterion appears in the post-implementation checklist **as a numbered Step** (not just a checklist entry). If the testing strategy lists multiple data paths (e.g. "includes X + Y entries"), each path must have a dedicated step or sub-step. If a testing strategy item references a function or behavior that cannot be tested in the project's test runner (e.g., a client-side-only inline helper with no exported module), the plan must either (a) include a step to extract it to a testable module, or (b) add an explicit checklist item documenting the test gap and why it cannot be covered. Silent omission of a testing strategy item is never acceptable — the checklist must account for every item, even if the account is "untestable without refactor." |
| **Shared / common types** | For every field the spec says the UI or API should display, verify that field actually exists on the shared type the component receives (not just on the full model). `ArticleSummary = Omit<Article, "body" \| "sources" \| "agentMetadata">` is a real example where a spec field (`agentCount`) was silently dropped because the list endpoint returns a subset type. If a field is absent from the shared type, either add it to the type (and the Lambda handler) as a plan step, or explicitly mark it as a known gap in the plan. Do not leave it to the implementer to discover. |

**Assertion quality rule**: When a plan step includes a verification assertion against a spec-stated expected value (e.g. a score, count, or computed result), write the assertion against the spec's *literal stated value* — not a re-derived formula. `assert round(result, 1) == 4.1` (which fails if the formula is wrong) is correct; `assert result == 5*0.4 + 3*0.35 + 4*0.25` (which cannot catch spec/float discrepancies) is not. Tautological assertions must be replaced before the plan is saved.

**Test count accuracy rule**: When embedding a full test file in the plan, count the `def test_` (or `it(`, `test(` depending on language) functions explicitly before writing the plan. Use that exact count in every reference to the expected pass count (verification command, checklist, commit message). Do not estimate or copy a count from a previous plan or spec — count the functions in the file you are about to embed.

For each gap found — silently add the missing step(s) or checklist item(s) to the plan draft. Do not produce a report or list the gaps; just fix them. The goal is a complete plan, not a paper trail of what was missing.

The only exception: if a gap requires a design decision you cannot make without user input (e.g. the spec says "use approach X or Y, TBD"), stop and ask before proceeding.

Once all gaps are filled, save the plan.

---

### Phase 3.6 — Move Spec to Applied

Now that the implementation plan is saved and verified, move the spec to `specs/applied/` so it is clearly marked as used.

```bash
# Create the applied directory if it doesn't exist
mkdir -p "C:\Users\Epkone\Humble beginnings\specs\applied"

# Move the spec (replace <spec-filename> with the actual filename)
mv "C:\Users\Epkone\Humble beginnings\specs\<spec-filename>" \
   "C:\Users\Epkone\Humble beginnings\specs\applied\<spec-filename>"
```

After moving, update the **Spec** line in the implementation plan's Header to reflect the new path:

```
- **Spec**: specs/applied/<spec-filename>
```

This keeps `specs/` as the inbox for new, unplanned specs and `specs/applied/` as the archive of specs that already have an implementation plan.

---

### Phase 4 — Create the Worktree

> **Fetch EnterWorktree schema first**: `EnterWorktree` is a deferred tool — its schema is not loaded by default. Before calling it, run `ToolSearch` with query `"select:EnterWorktree"` to load the schema. A planner that skips this step will get an `InputValidationError` and may falsely conclude the tool is unavailable. Do this as the very first action in this phase.

Call `EnterWorktree name='<slug>'` where `<slug>` is derived from the spec description — lowercase, hyphens, no spaces. Example: `declaration-bar-fix`.

This creates an isolated checkout at `.claude/worktrees/<slug>/` with its own branch. The current session is now working from inside the worktree.

If `EnterWorktree` fails (name collision, git error), report the error and ask the user to choose a different name or resolve the issue manually. Do not proceed without a successful worktree.

> **EnterWorktree failure ≠ no git repo**: If `EnterWorktree` returns an error or is unavailable, run `git rev-parse --show-toplevel` before concluding there is no git repository. A tool failure means the tool could not run — not that git is absent. If git is confirmed present, work proceeds in the main repo and the commit step must still run.

After the worktree is created, copy the plan into it so `/impl` can find it whether run from the main project or from inside the worktree:

```bash
cp "C:\Users\Epkone\Humble beginnings\Implementation Plans\impl--<filename>.md" \
   "C:\Users\Epkone\Humble beginnings\.claude\worktrees\<slug>\Implementation Plans\"
```

### Phase 5 — Confirm

Tell the user:
- Impl plan written to `Implementation Plans/impl--...md` (main project and worktree copy)
- Worktree created at `.claude/worktrees/<slug>/`
- **"Run `/impl <description>` to begin implementation."**

Remind the user: Unity MCP tools connect to the running Unity Editor regardless of working directory. Script file edits are written to the worktree and are not visible to the Unity Editor until the worktree is merged to main.
