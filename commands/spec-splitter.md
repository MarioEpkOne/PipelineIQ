SPEC SPLITTER -- analyzes a spec for context window pressure, proposes a decomposition, and generates self-contained child specs optimized for parallel `/pipeline-team` execution.

Invocation: `/spec-splitter $ARGUMENTS`

The argument is the path to a spec file. If no argument provided, STOP: "Usage: /spec-splitter <path-to-spec>".

---

## Phase 1 -- Read and Analyze

### Argument resolution

Resolve the argument as a file path relative to the project root.

1. If the path exists as given -> use it.
2. If not found -> check `specs/<filename>` as a fallback.
3. If still not found -> check `specs/team/<filename>` as a fallback.
4. If still not found -> STOP: "Spec file not found: [original-path]. Checked: [original-path], specs/[filename], specs/team/[filename]."

### Re-split detection

After reading the spec, check for a `**Split from**:` line. If found, store the parent path for a warning in Phase 4 but do not stop yet.

### Read and extract

Read the spec file in full. Extract these components:

1. **Title** -- first `# ` heading
2. **Goal** -- full text of Goal section
3. **Decisions** -- all D1..DN entries, including rationale
4. **Technical Design** -- full text, noting distinct subsections (each `###` heading within Technical Design = one subsection)
5. **Edge Cases** -- all rows from the table
6. **Constraints** -- all items
7. **Current State** -- full text
8. **Open Questions** -- if present

### Open Questions check

If the spec has Open Questions (non-empty, not "None"):

> "This spec has N open questions. Splitting with unresolved questions may distribute ambiguity across children. Continue? [yes/no]."

If no -> exit. If yes -> proceed.

### Missing Technical Design check

If the spec has no Technical Design section:

> STOP: "Spec is missing a Technical Design section -- cannot analyze for splitting."

### Assess context window pressure

Analyze these signals:

| Signal | Method |
|---|---|
| Files to modify | Count distinct file paths in Technical Design (create, modify, delete actions) |
| Decision clusters | Group decisions by which Technical Design subsection they affect. Independent clusters = split candidates. |
| Technical Design separability | Identify subsections that share no file paths and no decision references. Each independent subsection is a potential child. |
| Edge case span | Map each edge case row to the Technical Design subsection(s) it references. Edge cases spanning multiple subsections indicate coupling. |
| Reading list estimate | Count files referenced in Current State + Technical Design that an impl agent would need to read. |

Print the complexity assessment:

```
## Analysis: [Spec Title]

**Files to modify**: N files across M directories
**Decisions**: N total, clustering into K independent groups
**Technical Design**: N independent subsections identified
**Edge cases**: N total, M cross-cutting (spanning subsections)
**Estimated reading list**: N files

**Assessment**: [Right-sized / Should split]
**Reasoning**: [1-2 sentences explaining why]
```

### Assessment heuristics

Assess as **right-sized** (no split needed) when:
- The spec has < 3 decisions AND 1 Technical Design section, OR
- All decisions and Technical Design subsections are tightly coupled (no clean split boundary -- every subsection shares files or decisions with every other subsection), OR
- Files to modify are few (roughly 3 or fewer) and all concentrated in one subsystem

Assess as **should split** when:
- Technical Design has 2+ independent subsections (no shared files, no shared decisions), OR
- Decision clusters are clearly separable into groups that don't cross-reference each other, OR
- The reading list estimate exceeds roughly 8-10 files spanning multiple subsystems

These are guidelines for the agent's judgment, not rigid thresholds. Always explain the reasoning.

---

## Phase 2 -- Right-Sized Exit

If the assessment is "Right-sized", print the analysis and:

```
This spec is right-sized for a single /pipeline run -- no split needed.
Run `/pipeline specs/[filename]` to implement it.
```

Then exit. Do not offer to force a split.

---

## Phase 3 -- Propose Split Plan

If the spec should be split, group the content into child specs:

### Grouping algorithm

1. **Identify natural split boundaries** -- each independent Technical Design subsection + its associated decisions + its edge cases = one child spec candidate.
2. **Extract shared foundations** -- decisions or Technical Design elements that multiple children depend on become a foundation spec. The foundation spec contains only the shared concern, not the features that build on it. Even trivially small foundation specs (1 decision, 1 file) are acceptable -- they unblock parallelism.
3. **Assign edge cases** -- each edge case goes to the child spec whose Technical Design subsection it references. Cross-cutting edge cases go to every child they touch (duplicated, with a note: "Also applies to: [sibling spec name]").
4. **Assign constraints** -- project-wide constraints go into every child. Constraints specific to one area go only to that child.
5. **Duplicate shared decisions** -- decisions that apply to multiple children are copied in full (including rationale) into every relevant child. No cross-references to parent or siblings.
6. **Wire dependencies** -- determine which child specs depend on which others. For each dependency edge, intentionally embed 2+ title keywords (> 4 chars) from the dependency's title into the dependent's Goal or Current State text. This ensures pipeline-team's keyword-matching algorithm detects the edge.
7. **Maximize parallelism** -- if two possible groupings produce similar child sizes but different wave structures, prefer the one with more specs in Wave 1 (no dependencies).
8. **Handle shared files** -- if two Technical Design subsections both modify the same file, that file is a coupling point. Assign it to the child that modifies it most heavily. Other children list it as a read-only dependency in their Current State (which triggers a dependency edge via keyword wiring). If the file is modified equally by multiple subsections, it becomes part of the foundation spec.

### Verify keyword wiring

Before presenting, run pipeline-team's dependency detection algorithm against the proposed child titles and Goal/Current State text:

For each pair of children (A, B):
1. Extract B's title keywords (words > 4 chars, lowercase, excluding stopwords: "with", "that", "this", "from", "into", "using", "their").
2. Check if 2+ of B's keywords appear in A's Goal or Current State text.
3. If yes -> A depends on B.

Verify the detected graph matches the intended dependency graph. If not, adjust the keyword wiring in the Goal/Current State text.

Also check for cycles. If a cycle is detected: "Circular dependency: A -> B -> A. Adjusting grouping to remove cycle." Re-group and re-check.

### Present the split plan

```
## Proposed Split: [Parent Title]

**Child specs** (N total):

1. **[Child 1 title]** -- [1-line summary]
   - Decisions: D2, D5, D8 (from parent numbering)
   - Files: [list of files to modify]
   - Edge cases: [count]
   - Dependencies: none (Wave 1)

2. **[Child 2 title]** -- [1-line summary]
   - Decisions: D1, D3, D4
   - Files: [list of files to modify]
   - Edge cases: [count]
   - Dependencies: none (Wave 1)

3. **[Child 3 title]** -- [1-line summary]
   - Decisions: D6, D7, D9
   - Files: [list of files to modify]
   - Edge cases: [count]
   - Dependencies: Child 1

**Dependency graph:**
  01-child-slug.md (no deps)
  02-child-slug.md (no deps)
  03-child-slug.md -> depends on: 01-child-slug.md

**Wave schedule:**
  Wave 1 (parallel): 01-child-slug.md, 02-child-slug.md
  Wave 2 (after Child 1): 03-child-slug.md

**Estimated parallelism**: 2 specs in Wave 1, 1 in Wave 2
  (vs. 1 sequential spec without splitting)

Proceed with this split? [yes / edit / abort]
```

### User response handling

**abort**: Exit without generating any files. Print: "Split aborted. No files created."

**edit**: Accept free-text edits (e.g., "move D3 from child 1 to child 2", "merge children 1 and 2", "make child 3 independent"). Re-compute the split plan:
- If the edit merges all children back into one: "Merging all children back into one spec produces the original. No split needed." Exit.
- If the edit creates a circular dependency: "Circular dependency detected: A -> B -> A. Remove one edge." Re-present without the circular edit applied.
- Re-verify keyword wiring. Re-present the updated plan.

**yes**: Proceed to Phase 4.

---

## Phase 4 -- Re-split Check

If the spec had a `**Split from**:` line (detected in Phase 1):

```
Warning: This spec was already split from [parent path].
Re-splitting creates a deeper hierarchy. Consider re-splitting
the original parent instead.

Continue anyway? [yes / no]
```

If no -> exit. If yes -> proceed.

If no `Split from` header was found, skip this phase.

---

## Phase 5 -- Generate Child Specs

This is the only phase that writes files. All prior phases are read-only.

### Filename derivation

Extract from the parent spec filename:
- `{PARENT_DATE}` = the timestamp portion (e.g., `2026-04-22--01-54`)
- `{PARENT_SLUG}` = the description slug (e.g., `spec-splitter-command`)

Child filenames: `spec--{PARENT_DATE}--{PARENT_SLUG}--{NN}-{child-slug}.md`
- `{NN}` = two-digit sequence (01, 02, 03...)
- `{child-slug}` = kebab-case description of the child's focus

### Collision detection

Before writing each child spec, check if the target filename already exists in `specs/team/`. If it does:
- Append a numeric suffix: `--01-auth-2.md`, then `--01-auth-3.md`.
- STOP after 3 collisions for the same child: "Filename collision limit reached for [child]. Clean up specs/team/ and rerun."

### Create directory

Create `specs/team/` if it doesn't exist: `mkdir -p specs/team/`

### Write each child spec

Each child spec must contain all required sections and be fully self-contained:

```markdown
# Spec: [Child Title]

**Split from**: specs/applied/[parent-filename]

## Goal

[Child-specific goal text. Intentionally includes 2+ title keywords
(> 4 chars) from any dependency spec's title, to ensure pipeline-team's
keyword-matching algorithm detects dependency edges.]

## Current State

[Relevant portions of the parent's Current State. Only what this child
needs. If this child depends on another, reference the dependency's
concern area using its title keywords.]

## Decisions

[Decisions relevant to this child, re-numbered starting from D1.
Shared decisions duplicated in full, including rationale.
Each decision retains its original rationale text.]

## Technical Design

[Only the Technical Design subsection(s) relevant to this child.
Self-contained -- no references to sibling specs.]

## Edge Cases & Error Handling

[Edge cases relevant to this child. Cross-cutting cases duplicated
with a note: "Also applies to: [sibling spec name]".]

## Constraints & Invariants

[Project-wide constraints duplicated into every child.
Child-specific constraints only in the relevant child.]

## Testing Strategy

[Testing approach for this child's scope only.]

## Open Questions

[Relevant open questions, or "None".]
```

### Union coverage check

After generating all children, verify that every item from the parent spec appears in at least one child:
- Every decision (D1..DN)
- Every Technical Design element
- Every edge case row
- Every constraint

If anything is missing, add it to the most relevant child before writing files.

### Move parent to specs/applied/

Determine the parent's current location:
- If already in `specs/applied/` -> do not move (no double-move).
- If in `specs/` -> move to `specs/applied/`.
- If in `specs/team/` -> move to `specs/applied/`.

Create `specs/applied/` if it doesn't exist: `mkdir -p specs/applied/`

### Error recovery

If generation fails partway (e.g., disk error after writing 2 of 3 children):
- Do NOT move the parent spec.
- Print: "Generation failed after writing N of M children. Parent spec left in place. Partial children in specs/team/ may need cleanup."
- Exit.

---

## Phase 6 -- Summary

Print the completion summary:

```
## Split Complete

**Parent**: specs/[parent-filename] -> moved to specs/applied/
**Children** (N specs in specs/team/):
  - specs/team/[child-1-filename]
  - specs/team/[child-2-filename]
  - specs/team/[child-3-filename]

**Dependency graph:**
  [rendered graph from Phase 3]

**Wave schedule:**
  [rendered schedule from Phase 3]

**Next step**: Run `/pipeline-team` to execute these specs in parallel.
```
