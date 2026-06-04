# Promotion Log

## 2026-04-23: Impl-plan should check CLAUDE.md for stale references when changing defaults
**Applied to**: commands/impl-plan.md
**Was**: MEDIUM (1 occurrence)
**Seen in**: spec-splitter-command pipeline run (2026-04-22)
**Diff applied**:
```diff
 | **Edge Cases** | Every edge case has a step or explicit handling ... |
+| **Documentation references** | If any step changes a default value, scan path, CLI argument format, or naming convention, add a sub-step to search CLAUDE.md and any nested CLAUDE.md files for references to the old value. Include those files in the plan's Scope section and update them in the same step. Stale documentation invariants are a recurring audit finding that costs a fix cycle. |
 | **Constraints** | Every constraint is enforced somewhere in the steps or post-implementation checklist |
```

## 2026-06-04: Plan edits one doc sentence but leaves an adjacent stale count/list
**Routed to**: CVapplication `.claude/pipeline-kb/planner.md`, `.claude/pipeline-kb/implementer.md`
**Was**: LOW (1 occurrence) — project-specific portion only; universal table-row generalization retained in `learnings.md` pending a pipelineIQ plugin update
**Seen in**: spec--interactive-forex-positions, project: /mnt/c/Users/Epkone/CVapplication
**Entries written**:
- planner: "When a plan step edits a documentation region (CLAUDE.md, ADRs), sweep the surrounding paragraph/table for stale counts and enumerations — not just the one sentence being changed. CVapp's CLAUDE.md describes the Lambda's mock Forex tool set both as a count ("six mock Forex tools") and as an enumerated list, with an adjacent MockAgent sentence; a change to that set must update every count/list in the same step or the auditor flags SPEC_DRIFT (a fix-cycle cost)."
- implementer: "When changing the Lambda's mock Forex tool set, update every place CLAUDE.md counts or lists it (the "six mock Forex tools" count, the enumerated tool names, and the adjacent MockAgent sentence) in the same edit — stale doc counts are a recurring SPEC_DRIFT finding here."

## 2026-06-04: Edit tool mangles typographic / non-ASCII characters
**Routed to**: CVapplication `.claude/pipeline-kb/implementer.md`
**Was**: LOW (1 occurrence) — project-specific (CVapp CZ typographic quotes); universal non-ASCII generalization retained in `learnings.md` pending a pipelineIQ plugin update
**Seen in**: spec--content-accuracy-fixes, project: /mnt/c/Users/Epkone/CVapplication
**Entries written**:
- implementer: "CZ resume/cover-letter content uses typographic quotes (U+201E „ … U+201C ") in src/data/resume.ts and src/data/cover-letter.ts. The Edit tool may silently substitute an ASCII " (U+0022) for a closing curly quote, producing "unterminated string literal" typecheck errors. After editing such strings, byte-verify the glyphs (or write them via a script with explicit \uXXXX escapes) and run `npm run typecheck` before moving on. Treat a typecheck failure on a line you just edited with special characters as a glyph-encoding issue, not a logic bug."
