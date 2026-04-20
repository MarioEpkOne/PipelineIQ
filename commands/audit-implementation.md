POST-IMPLEMENTATION AUDIT — analyzes what the implementing agent did wrong and why, proposes specific changes to CLAUDE.md / impl.md / impl-plan.md for approval, and appends new patterns to learnings.md.

The user's request is: $ARGUMENTS

## Purpose

This skill is a process autopsy, not a QA check. The goal is to understand *why* failures happened so the skills and rules can be improved for future implementations. Run this after every /impl. For auditing whether an implementation *plan* conforms to the impl-plan skill spec, use `/audit-plan` instead.

**Re-audit mode (after /fix):** If this audit is being run after a `/fix` session, limit scope to: (1) verify each change the fixer stated it made in its working log, and (2) confirm no regressions were introduced. A full independent evaluation and complete failure taxonomy are not needed — skip Phase 2 and condense Phase 3 to only the fixer's stated changes. Note "Re-audit — scoped to fixer's stated changes" at the top of the audit document.

---

## Phase 1 — Find Inputs

**Find the working log:**
- If an argument was given: search `Working Logs/` for a file whose name contains the argument (case-insensitive). If multiple match, list them and ask.
- If no argument: use the most recently modified `.md` file in `Working Logs/`.
- If none found: tell the user and exit.

Read the working log in full.

**Follow the chain:**
- From the working log's `**Impl plan:**` header field: find and read the impl plan from `Implementation Plans/`.
- From the impl plan's `**Spec:**` header field: find and read the spec from `specs/`.

If a link is missing, continue without that document but flag the gap in the audit output. If the spec cannot be found, skip Phase 2 (independent evaluator requires the spec) and note "Independent evaluation skipped — spec not found."

---

## Phase 2 — Independent Evaluator Sub-Agent

Launch a sub-agent using the Agent tool. Pass it the following prompt, with the full spec content pasted in at the end:

---
*Sub-agent prompt (fill in [SPEC CONTENT] before launching):*

```
You are an independent evaluator. You have NOT seen the working log or implementation plan for this feature. Do not read files in Working Logs/, Implementation Plans/, or Retros/.

Your inputs are:
1. The spec document pasted below
2. The current state of the codebase (you may read source files referenced in the spec's Technical Design section)

Your task:
For each goal and expected behavior stated in the spec's Goal and Edge Cases sections, inspect the current codebase state (read source files, run tests, check the build) to decide whether the behavior is in place.

CRITICAL RULE: Distinguish static evidence (value written in the source) from runtime evidence (value observed when the code runs). For properties computed only at runtime (layout, async state, rendered output), static inspection is NOT valid verification — say so explicitly and mark as "requires runtime verification".

For each file mentioned in the spec's Technical Design section that you need to inspect: you may read it. Do not read any other files.

Output exactly this structure:

## Goals — Static Verification
For each goal from the spec:
- **[Goal summary]**: APPEARS MET / APPEARS UNMET / CANNOT VERIFY STATICALLY
  - Evidence: [what you read, what value was observed, or why it cannot be verified]

## Properties That Cannot Be Verified Without Runtime Observation
List any runtime-computed properties you inspected where only static/saved values were available. These are NOT confirmed correct.

## Contradictions Found
Any cases where the current code state directly contradicts what the spec expected.

---
[SPEC CONTENT]
```

---

Wait for the sub-agent to complete. Store its verdict for use in Phase 3.

If the sub-agent fails or times out: retry once after 3 seconds. If it fails again, note "Independent evaluation unavailable" in the audit and continue without it.

If the sub-agent's output references specific error messages or file contents that only appear in the working log (suggesting it read the log despite instructions): note "Evaluator isolation may be compromised — verdict may be anchored" and treat it with lower confidence.

---

## Phase 3 — Analyze Failures

Cross-reference the sub-agent verdict, the working log, and the impl plan. For every problem or deviation found, assign one or more of these categories:

| Code | Meaning |
|---|---|
| `STATIC_VS_RUNTIME_GAP` | Agent "verified" a property by static inspection when only runtime observation would be valid |
| `SPEC_DRIFT` | Agent deviated from the spec's stated goal — by choice or because forced |
| `PLAN_DEVIATION` | Agent deviated from impl plan steps — by choice or because forced |
| `INCOMPLETE_TASK` | Working log has unchecked Post-Implementation Checklist items |
| `TEST_FAILURE` | Test suite has failures that were not disclosed or not fixed |
| `BUILD_FAILURE` | Build or typecheck fails |
| `RULE_VIOLATION` | A CLAUDE.md hard rule was broken |

For each failure: identify what happened, why it happened (root cause, not symptom), and the evidence.

---

## Phase 4 — Propose Skill Changes

For each failure category found, check whether a rule in `CLAUDE.md`, `impl.md`, or `impl-plan.md` already addresses it. If not, propose a specific addition or change.

Format all proposals as exact diff blocks with the target file and insertion location:

```diff
+ line to add
- line to remove
```

---

## Phase 5 — Write Audit Document

Before writing the file, run `date '+%Y-%m-%d--%H-%M'` to get the actual current timestamp. Save to `Retros/audit-impl--YYYY-MM-DD--HH-MM--<description>.md` (replacing `YYYY-MM-DD--HH-MM` with the output of that command; use the same description slug as the working log).

Use this structure:

```markdown
# Implementation Audit: <Feature Name>
**Date**: YYYY-MM-DD
**Status**: COMPLETE / INCOMPLETE
**Working log**: Working Logs/wlog--...md
**Impl plan**: Implementation Plans/impl--...md
**Spec**: specs/spec--...md

---

## Independent Evaluator Verdict
[Summary of sub-agent findings. If evaluation was skipped or isolation may be compromised, say so here.]

## Goals — Static Verification
| Goal | Status | Evidence |
|---|---|---|
| [goal] | APPEARS MET / APPEARS UNMET / CANNOT VERIFY STATICALLY | [value observed or "requires runtime verification"] |

## Properties Not Verifiable Without Runtime Observation
[List from sub-agent]

---

## Failures & Root Causes

### [Short failure title]
**Category**: [category code(s)]
**What happened**: [1-2 sentences]
**Why**: [root cause]
**Evidence**: [quote from working log or finding]

*(Repeat for each failure)*

---

## Verification Gaps
[All cases where static inspection was used to verify a runtime value — explicitly flagged as UNCONFIRMED even if the working log called them confirmed]

---

## Actionable Errors

Structured list of errors that a fixer agent can act on. Each entry is self-contained.

### Error 1: [Short title]
- **Category**: [category code from Failures & Root Causes]
- **File(s)**: [exact file paths involved]
- **What broke**: [1-2 sentences — expected behavior vs. actual]
- **Evidence**: [specific error message, observed value, or test output]
- **Suggested fix**: [concrete action — "change X to Y in file Z", not "investigate"]

*(Repeat for each actionable error)*

**Not actionable (requires human judgment or runtime verification):**
- [list items that cannot be auto-fixed, with explanation of why]

## Rule Violations
[Any CLAUDE.md rules broken — note whether intentional and what tradeoff was made]

## Task Completeness
- **Unchecked items**: [list from working log Post-Implementation Checklist, or "None"]

---

## Proposed Skill Changes

### CLAUDE.md — [short title]
**Insert after**: [section/line reference]
\`\`\`diff
+ line to add
\`\`\`
**Why**: [which failure this prevents]
[ ] Apply?

*(Repeat for impl.md and impl-plan.md as needed)*

---

## Proposed learnings.md Additions
Copy-paste these into learnings.md under the relevant section:

\`\`\`
- YYYY-MM-DD [slug]: [pattern in 1-2 sentences]. → [which skill to update]
\`\`\`
```

The "Actionable Errors" section is critical — it is what the `/fix` skill and the pipeline's fix loop parse. Every error in "Failures & Root Causes" must appear either as an actionable error entry or in the "not actionable" list. Do not omit errors from both places.

If no failures were found: write a minimal audit ("No failures identified. No proposed changes.") and save it. Still required even for clean impls.

---

## After Writing

Tell the user:
- Audit saved to `Retros/audit-impl--...md`
- List proposed skill changes (titles only) and say: "Review the retro and reply 'apply [title]' for any changes you want me to make, or 'apply all' to apply everything."
- Remind them to copy the `learnings.md` additions manually if they want to keep them.

When the user says "apply [title]" or "apply all": read the current target file, apply only the approved diff(s), and confirm.
