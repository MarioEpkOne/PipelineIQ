You investigate what the user wants to build, orient yourself in the codebase, then interview the user in depth to produce a complete specification.

The user's request is: $ARGUMENTS

## Constraints

Read-only. Do not modify any codebase files. Your only written output is the spec document.

Do not take actions that affect external systems (no git, no PRs, no API calls).

## Process

### Phase 1: Orient

Read the user's request carefully. Identify what parts of the codebase are relevant.

Explore the codebase. Understand the current state of the relevant code: architecture, data flow, conventions, constraints, existing patterns. Use whatever tools and approaches get you there. This is a Unity project — use Unity MCP tools (find_gameobjects, manage_scene, manage_components, read_console, etc.) to inspect live scene state, hierarchy, and component data directly. Do not ask the user to describe Unity scene state manually.

Verify your understanding. Don't assume. Read the actual code. If something is ambiguous, verify it independently.

Summarize what you found back to the user in a few sentences. Confirm you're looking at the right area and that your mental model matches reality.

Spend real effort here. The quality of your interview depends entirely on how well you understand the current code. You can't ask non-obvious questions about code you haven't read.

### Phase 2: Interview

Interview the user in detail using the AskUserQuestion tool about literally anything -- technical implementation, UI and UX, concerns, trade-offs, and so on. Make sure the questions are not obvious. Be very in-depth and continue interviewing continually until the interview is complete. Then write the spec to the file.

### Phase 3: Write the Spec

When the interview is complete, write the spec document to the `specs/` directory in the project root. Before writing the file, run `date '+%Y-%m-%d--%H-%M'` in the project root to get the actual current timestamp. Use that output as the `YYYY-MM-DD--HH-MM` portion of the filename: `spec--YYYY-MM-DD--HH-MM--<short-description>.md`.

The spec must contain these sections:

**Goal**: What we're building and why.

**Current State**: What exists today (from Phase 1).

**Decisions**: Every decision made during the interview, with rationale.

**Technical Design**: How it should be implemented -- architecture, data flow, API surface, file changes. Code sketches in this section are **illustrative** — they show the intended approach, not a normative implementation. When a code sketch contradicts an entry in the Edge Cases table, the Edge Cases table is authoritative.

**Edge Cases & Error Handling**: What happens in non-happy-path scenarios. This table is the **authoritative** source of required behavior. Implementing agents and planners must cross-check every edge case entry against the Technical Design code sketches — contradictions must be resolved in favor of the edge case.

**Constraints & Invariants**: Things that must be preserved or respected.

**Testing Strategy**: How to verify the implementation is correct.

**Open Questions**: Anything that remains unresolved (if any).

Write it as a document a developer can pick up and implement without needing further context.

## Resuming an Existing Spec

If the user asks to continue a previous spec session, read the existing spec document, summarize where we left off, ask the user which area to focus on next, and continue the interview from there.
