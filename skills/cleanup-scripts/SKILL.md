---
name: cleanup-scripts
description: >
  Session-scoped script cleanup for Horsley Lab R analysis projects. Checks
  scripts/scratch/ for working files that need consolidation into numbered .Rmd
  scripts, flags unnumbered scripts that should be numbered, and verifies
  script-output correspondence. Use when finishing a coding session, when
  scripts/scratch/ has accumulated files, or when the user says "clean up
  scripts", "consolidate scratch", or "wrap up this session".
---

# Script Cleanup (Session Scope)

Check and consolidate working scripts from the current session into the
numbered `.Rmd` pipeline.

---

## When to Run

- Before wrapping up a coding session
- When `scripts/scratch/` has accumulated files
- When the user asks to clean up or consolidate scripts
- After a long iterative session with lots of exploratory file creation

---

## Steps

### 1. Check `scripts/scratch/`

List all files in `scripts/scratch/`. For each file:

- **Identify the target script:** Based on filename and conversation context,
  determine which numbered `.Rmd` script this code belongs to. If none fits,
  it may need a new numbered `.Rmd`.
- **Report to user:** Show each scratch file, its proposed destination, and
  ask for confirmation before consolidating.
- **Consolidate:** Add the code to the target `.Rmd` in a new chunk with a
  descriptive chunk label. Or create a new numbered `.Rmd` if needed.
- **Clean up:** Delete the scratch file after successful consolidation.

If `scripts/scratch/` is empty or doesn't exist, report "No scratch files."

### 2. Check for Convention Violations

Scan `scripts/` for files that should be numbered but aren't:

- `.R` or `.Rmd` files at the top level of `scripts/` without a number prefix
- Ask the user: assign a number and rename, move to `scratch/`, or archive?

Skip: `scripts/scratch/`, `scripts/old/`, already-numbered files.

### 3. Verify Script-Output Correspondence

For each numbered `.Rmd` in `scripts/`:
- Check whether `outputs/figures/` contains files whose names suggest they
  came from that script
- Flag scripts with no apparent outputs (may just not have been run yet — fine)

### 4. Report

Summarize:
- Scratch files consolidated
- Convention violations found and resolved
- Any scripts with no corresponding outputs

---

## Notes

- **Never delete without asking.** Always show what will be moved or deleted first.
- **This is session-scoped** — it uses conversation context to understand what
  belongs where.
- If no conventions are set up yet for this project, offer to create the standard
  folder structure using the `horsley-lab-conventions` skill.
