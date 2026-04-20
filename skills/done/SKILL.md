---
name: done
description: >
  Use when ending a session, wrapping up work, or when the user says "done", "wrap up",
  "let's commit", or "end of session". Summarizes work, updates docs, and commits.
user-invocable: true
---

# End of Session Wrap-up

When the user invokes `/done`, perform these end-of-session tasks in order.

---

## 0. Detect Project Type

Check the project's `.claude/CLAUDE.md` for a `project-type:` field:

- **`data-science`** — Full wrap-up including all steps
- **`general`** (or no field found) — Skip steps marked **[Data Science only]**

If no field exists, infer: `renv.lock`, `outputs/`, or numbered `XX_*.Rmd` scripts →
data science. Otherwise → general.

---

## 1. Summarize Work and Decisions

Briefly list what was completed this session:
- Scripts created or modified
- Data files created or modified
- Documentation changes
- Key analytical or design decisions made

---

## 1b. Update Session Log in Project CLAUDE.md

Append a new entry to the **Session Log** section at the bottom of the project's
`.claude/CLAUDE.md`. This is a rolling log of the last 5 sessions — it's the primary
place future sessions look for "what to do next."

### Format

```markdown
## Session Log
<!-- Maintained by /done. Most recent first. Keep last 5 entries. -->

### YYYY-MM-DD — Short title
- **Plans:** [plan name(s) worked on, or "None"]
- **Work:** [1-2 sentences on what was done]
- **Next:** [bullet list of follow-up items for future sessions]
```

### Rules

- **Most recent first** — new entry goes at the top of the list
- **Same-day updates** — if an entry for today's date already exists, **replace it**
  rather than adding a duplicate. Merge the work descriptions and update Next items.
- **Trim to 5 entries** — delete the oldest entry if there are more than 5
- **Short title** should disambiguate sessions (e.g., "QC filtering", "Clustering",
  "DE analysis")
- **Plans line** — list which planning documents were worked on. Write "None" if no
  plans were involved.
- **Next items** — be specific and actionable. These are the main value of the log.
- If the Session Log section doesn't exist yet, create it at the bottom of the file.

---

## 2. Update Relevant Planning Documents

Only update planning documents **directly relevant to this session's work**. Do NOT
read all documents — identify the 1-2 that matter from session context.

Planning documents live in `.claude/` as markdown files (e.g.,
`.claude/scrna_analysis_plan.md`, `.claude/figure_plan.md`).

For each relevant planning document:

**a. Status tables** — Mark completed phases/tasks. Add new phases if needed. When a
phase is marked complete, collapse its detailed content to a 3-5 line summary (what
was done, key outcomes). Before collapsing, ensure forward-looking information needed
by future phases is captured elsewhere in the plan.

**b. Script and file tracking** — Add new scripts, mark replaced ones as legacy,
update modified entries.

**c. Task lists** — Check/uncheck items as appropriate.

**d. Key decisions** — If analytical decisions were made this session (threshold
choices, method selections, sample exclusions), add them to a "Key Decisions" section.
These should be permanent — don't collapse or delete them.

Show proposed changes before editing.

### Planning document format

If no planning document exists for a multi-session project, offer to create one:

```markdown
# {Project/Analysis Name} Plan

## Status
| Phase | Status | Notes |
|-------|--------|-------|
| {phase} | ⬜ Not started | |

## Key Decisions
(Record permanent analytical decisions here — threshold choices, method selections,
sample exclusions, etc.)

## Script Registry
| Script | Purpose | Status |
|--------|---------|--------|
| `scripts/01_qc.Rmd` | QC and filtering | ✅ Done |

## Next Steps
- (Populated by /done at each session)
```

Status icons: ✅ Done · 🔄 In progress · ⬜ Not started · ⚠️ Needs review

---

## 3. Update Project Documentation (if needed)

Only do these checks if the session actually changed something relevant. Skip silently
otherwise.

### Project CLAUDE.md conventions

If new conventions, gotchas, or important patterns were discovered this session, propose
additions. Only record things that are **surprising or counter-default** — skip anything
Claude would infer from the codebase.

### CHANGELOG.md

If the project has a `CHANGELOG.md` in its root directory:
- Review what was done this session
- Propose a changelog entry under today's date (Added/Changed/Fixed/Removed sections)
- If today's date already has an entry, append to it rather than creating a duplicate
- Show proposed changes before editing
- If no `CHANGELOG.md` exists, skip silently — do not suggest creating one

---

## 4. Script Cleanup [Data Science only]

Check `scripts/scratch/` for files that accumulated this session:

- List all files in `scripts/scratch/`
- For each file, identify which numbered `.Rmd` script it belongs to (or whether it
  needs a new numbered script)
- Show the user the proposed consolidation and ask for confirmation
- After confirmation, add the code to the target `.Rmd` and delete the scratch file

If `scripts/scratch/` is empty, skip silently.

This replaces the standalone `cleanup-scripts` skill — running `/done` is sufficient
at end of session.

---

## 5. Git Commit

Run `git status` to check for uncommitted changes.

If there are changes:
- **Always stage specific files by name (`git add file1 file2`), NEVER use
  `git add .` or `git add -A`** — broad staging picks up changes from other sessions
- **Only include files actually created or modified during THIS session**
- Use conversation context as the primary record of what you did
- Show the user the proposed files and commit message before staging
- If approved, commit only those files
- After committing, offer to push to remote

### Conditional: renv snapshot [Data Science only]

Only if R packages were installed or updated during this session:
```bash
Rscript -e "renv::status()" 2>/dev/null
```
If out of sync, ask about `renv::snapshot()`. Include updated `renv.lock` in the commit.

### Conditional: Conda environment export [Data Science only, if Python used]

Only if conda packages were installed or updated during this session:
```bash
conda env export --from-history > environment.yml
```
Remove the `prefix:` line from the exported file (machine-specific, not portable).
Include updated `environment.yml` in the commit.

---

## 6. Final Summary

Brief "Session complete" message listing:
- Files created/modified
- Commits made
- Planning documents updated
- Key decisions recorded
- Follow-up items for next session
