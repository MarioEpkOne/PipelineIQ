Review learnings.md, classify each entry as universal or project-specific, route approved entries to commands/*.md or project KB files, curate existing KB entries, and archive everything to promotion-log.md.

The user's request is: $ARGUMENTS

## Step 1 -- Read learnings.md

Read `learnings.md` at the project root. If running from a project other than PipelineIQ, locate PipelineIQ's root by finding where `commands/pipeline.md` is installed and resolving upward. If empty or contains only headers with no entries, tell the user and stop.

Parse all entries. Group by severity tier (HIGH, MEDIUM, LOW). Number them 1, 2, 3... for reference throughout this workflow, ordered by severity (HIGH first, then MEDIUM, then LOW) and within each tier by occurrence count (highest first).

## Step 1.5 -- Detect target project context

Determine whether project-specific routing is available:
- Check if the current working directory (target project root) has a `.claude/` directory
- Check if the target project root has a `CLAUDE.md` file
- If neither `.claude/` nor `CLAUDE.md` exists: set PROJECT_KB_AVAILABLE = false, warn the user ("Project-specific routing unavailable -- no .claude/ directory or CLAUDE.md found. All learnings will be treated as universal pipeline improvements."), proceed with universal-only workflow
- If `.claude/` exists but `CLAUDE.md` does not: project-specific routing is partially available -- agent-specific KB files CAN be created in `.claude/pipeline-kb/` but project-wide gotchas cannot be routed to `CLAUDE.md`. Warn user about the limitation.
- If both exist: set PROJECT_KB_AVAILABLE = true, read any existing `.claude/pipeline-kb/*.md` files for the curation step

## Step 2 -- Analyze each entry against the codebase

For each entry, in order of severity (HIGH first):

- Read the target file referenced in **Suggested diff**
- Verify the diff still applies (the old text exists in the file at roughly the expected location)
- Check whether the problem described actually exists in the current command file
- Assess whether the suggested change would conflict with or contradict existing logic
- Check the real benefit: Is this a meaningful improvement, or cosmetic/trivial?
- If the diff is stale (old text not found in target file), attempt to regenerate it by re-reading the target file and the suggestion. If unable to generate a valid diff, mark as "stale -- needs manual review"
- If the problem described in the learning has already been fixed in the current file, mark as "Already resolved" with evidence (cite the line/section where the fix exists)

Don't form opinions from the suggestion text alone -- always verify against the actual source.

## Step 2.5 -- Classify each entry

For each entry, determine routing classification:
- **Universal**: pipeline process friction affecting all projects -> diff to `commands/*.md` (existing behavior)
- **Project-specific**: characteristic of this particular project -> bullet point(s) to `.claude/pipeline-kb/<agent>.md` and/or `## Pipeline Learnings` in `CLAUDE.md`
- **Hybrid**: universal principle revealed by project-specific incident -> split into universal diff AND project-specific KB entry

For project-specific and hybrid entries, also determine:
- **Target agents**: which of the 4 primary agents (planner, implementer, auditor, fixer) should receive the entry
- **Project-wide vs. agent-specific**: all agents (-> `CLAUDE.md ## Pipeline Learnings`) or specific ones (-> KB files)?
- **Reframed text**: for each target agent, write the bullet in language appropriate to that agent's role

Classification guidance (evaluated in priority order -- first match wins):
1. If observation references project-specific entities (API endpoints, language requirements, framework choices, dependency names, domain terms) -> **project-specific** (regardless of diff target)
2. If observation is a general principle but evidence is project-specific -> **hybrid** (split into universal diff AND project-specific KB entry)
3. If "Suggested diff" targets `commands/*.md` AND observation is purely about pipeline mechanics with no project-specific entities -> **universal**
4. When uncertain, default to **project-specific** (lower blast radius; can be promoted to universal in a future review cycle)

Note: The "Suggested diff" target in learnings.md is Phase 6's routing suggestion, not an authoritative classification. Phase 6 always targets `commands/*.md` because that's the only valid diff destination. Content-based signals override this heuristic.

## Step 3 -- Present analysis

Show a severity-grouped table with a "Routing" column:

```
SEVERITY | # | Title | Occ | Verdict | Routing | Why
---------|---|-------|-----|---------|---------|----
```

Severity thresholds (set by Phase 6 based on occurrence count): HIGH = 4+, MEDIUM = 2-3, LOW = 1.

Verdicts:
- **Apply** -- HIGH severity (4+ occurrences), diff verified, safe to apply now
- **Watch** -- MEDIUM severity (2-3 occurrences), approaching promotion threshold, or diff needs manual review
- **Already resolved** -- The problem described appears to have been fixed externally. Ask user whether to archive to promotion-log.md or keep watching.
- **Stale** -- Diff does not apply cleanly and could not be regenerated. Needs manual review regardless of severity.
- **Skip** -- LOW severity or insufficient evidence
- **Route** -- project-specific, route to KB files and/or CLAUDE.md
- **Split** -- hybrid, split into universal diff + project KB entry

If PROJECT_KB_AVAILABLE is false, entries that would be project-specific are shown with routing "Universal (project KB unavailable)".

After the table, show proposed KB entries grouped by destination:

```
Proposed project KB entries:

  CLAUDE.md ## Pipeline Learnings:
    (entries or "none for this batch")

  .claude/pipeline-kb/implementer.md:
    - [bullet entries]

  .claude/pipeline-kb/auditor.md:
    - [bullet entries]

  (etc. for each agent with entries)
```

## Step 3 continued -- KB curation

If existing KB files were found in Step 1.5, append a curation report:
- For each existing KB file, list every entry with a status (OK / STALE / CONTRADICTED)
- Flag entries that appear contradicted by newer learnings, by current code, or that reference entities no longer in the project
- If any KB file exceeds 50 lines, warn: "<file> has N entries (M lines). Consider pruning stale entries to keep agent context focused."
- Detect duplicate entries (new entry already present verbatim in target file): skip with note "Skipped -- already present in <file>."

Include a recommendation line for any stale entries found.

## Step 4 -- Ask for confirmation

Single consolidated question covers: which universal diffs to apply, which project KB entries to route, which existing KB entries to prune. User can override any decision (e.g. "make #2 universal instead", "skip #1", "keep entry [1] in implementer.md").

Wait for the user's response before proceeding.

## Step 5 -- Apply approved universal diffs

For each approved universal item, in order:

1. Read the target file fresh (the one named in the **Suggested diff** field)
2. Apply the edit using the Edit tool (targeted edit, not full rewrite)
3. If the diff doesn't apply cleanly: report and skip. Do not force.
4. If the next approved item targets the same file as one you just edited, re-read the file before attempting the diff -- the prior edit may have shifted context lines. If the diff no longer applies, attempt to regenerate it from the suggestion text against the current file state. If regeneration fails, report and skip.
5. After editing, confirm: "Item N done -- [short description of change] in [file]:[line]"

## Step 5.5 -- Write project KB entries

For each approved project-specific routing:

1. If targeting `CLAUDE.md ## Pipeline Learnings`:
   - Read `CLAUDE.md`
   - Find the `## Pipeline Learnings` section; if it doesn't exist, create it at the end of the file (before any trailing content)
   - Append the bullet point(s) under that section
2. If targeting `.claude/pipeline-kb/<agent>.md`:
   - Create `.claude/pipeline-kb/` directory if it doesn't exist (mkdir -p)
   - Read the file if it exists; if not, create with header `# <Agent> Project Knowledge\n\n`
   - Check for duplicates: if the bullet already exists verbatim, skip with note "Skipped -- already present in <file>."
   - Append the bullet point(s)
3. If the file after appending has only the header and no entries, do not write it

For approved KB pruning:

1. Read the target KB file
2. Remove the flagged entries
3. If the file has no remaining entries (only the header), delete the file

## Step 6 -- Archive to promotion-log.md

Append to `promotion-log.md` in the PipelineIQ project root (create with a `# Promotion Log` header if it doesn't exist).

For applied universal items, use this format:

```
## YYYY-MM-DD: [Title]
**Applied to**: [file path]
**Was**: [severity] ([N] occurrences)
**Seen in**: [full occurrence history from the entry]
**Diff applied**:
[the exact diff that was applied]
```

For routed project-specific items, use this format:

```
## YYYY-MM-DD: [Title]
**Routed to**: .claude/pipeline-kb/implementer.md, .claude/pipeline-kb/auditor.md
**Was**: [severity] ([N] occurrences)
**Seen in**: [full occurrence history]
**Entries written**:
- implementer: "[bullet text]"
- auditor: "[bullet text]"
```

For hybrid items: show both the universal diff and the project KB entry.

For items marked "Already resolved" that user chose to archive: move with note "Resolved externally".

## Step 7 -- Recount and re-sort learnings.md

After removing applied/archived items, rewrite `learnings.md` with the remaining entries properly organized:
- HIGH tier first, then MEDIUM, then LOW
- Within each tier: highest occurrence count first, then most recent date
- Preserve the tier headers and `---` separators even if a tier is empty
- Do NOT clear the file; only applied/archived items are removed

## Step 8 -- Commit

Stage and commit all modified files (command files edited, learnings.md, promotion-log.md, KB files written, CLAUDE.md if updated). Use a message like:

```
chore: apply pipeline learnings -- [summary]

Applied N universal, N project-specific, N hybrid. Pruned M stale KB entries.
See promotion-log.md for details.

Co-Authored-By: Claude <noreply@anthropic.com>
```

If nothing is git-trackable (e.g., only gitignored files were changed), skip the commit but still tell the user what was done.

Tell the user: "N items applied, N watching, N skipped. Applied items moved to promotion-log.md."

---

## Important constraints

- **Never apply a change you haven't verified against the current source.** The suggestion may be outdated.
- **Don't ask multiple confirmation questions.** One table, one question, then act.
- **Keep edits minimal and targeted.** Don't refactor surrounding code while applying a suggestion.
- **If you're unsure which file to edit**, use Grep to find the right location rather than guessing.
- **Build after code changes if the project has a build step** -- check for `npm run build` or similar in CLAUDE.md/package.json and remind the user if a rebuild is needed.
- **Universal diffs target `commands/` only.** Do not apply diffs to skills, CLAUDE.md, or files outside the pipeline command directory.
- **Only `/learnings-review` removes entries from `learnings.md`.** Phase 6 never removes entries.
- **`promotion-log.md` is the audit trail.** Every applied change is archived there with full context.
- **Rollback**: If an applied change causes problems, the original diff is preserved in `promotion-log.md`. Reverse the diff manually or revert the commit created in Step 8.
- **KB files are plain bullet lists with no metadata.** Metadata lives in `promotion-log.md`.
- **KB files use the format**: `# <Agent> Project Knowledge` header followed by bullet points.
- **When routing to multiple agents**, reframe each bullet for the agent's role.
