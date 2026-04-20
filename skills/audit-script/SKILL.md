---
name: audit-script
description: >
  Systematic audit of data analysis scripts for bugs, analytical reasoning, data handling, style,
  and reproducibility. Includes domain verification phase that researches tools, file formats, and
  methods to catch domain-specific errors (not just code bugs). Use when auditing a script,
  reviewing code for correctness, checking for bugs, preparing a script for publication, or when
  the user says "audit this script", "review this code", "check this for bugs", or "is this
  script correct". Three modes: thorough (collaborative section-by-section), fast (Claude-driven
  with discussion), and report-only. Do NOT load for quick one-off questions about a single line
  or function.
user-invocable: true
---

# Audit Script — Systematic Code Review for Data Science

This skill systematically evaluates data analysis scripts for correctness, analytical soundness,
and quality. It surfaces bugs, questionable analytical choices, data handling problems, style
issues, and reproducibility gaps — producing a structured audit report with severity levels and
action items.

Unlike `/learn-code` (which teaches team members to understand code), this skill is for **critical
evaluation** — finding what's wrong, fragile, or misleading. The user is a collaborator, not a
student. The tone is direct and analytical.

### Core Philosophy: Simplicity First

**The goal of an audit is NOT to make scripts handle every possible edge case.** Data science
scripts should be simple, clean, easy to read, and well-annotated. Adding defensive code for
hypothetical problems makes scripts harder to read, which is the opposite of what we want.

The audit should:
- **Flag real bugs** that produce wrong results in the script's actual use case
- **Flag analytical decisions** that affect interpretation (undocumented, questionable, or missing)
- **Flag clarity problems** — code that's hard to follow, poorly annotated, or unnecessarily complex
- **Note theoretical issues as awareness items**, not action items — "be aware that this join
  silently drops unmatched rows" is useful context; "add 20 lines of validation for a fixed
  input file" is over-engineering
- **Actively flag over-engineering** — unnecessary validation, defensive code for impossible cases,
  and abstraction-for-its-own-sake are style findings, not good practices

**Context matters.** A one-time script that processes a known, fixed Seurat object needs different
treatment than a reusable pipeline that will run on multiple datasets. The audit must calibrate
its recommendations to the script's actual role.

---

## Entry Flow

When the skill is invoked (via `/audit-script` or auto-loaded from context):

### 1. Identify the Script

Check for:
- An IDE selection (highlighted code in the editor)
- A currently open file in the editor
- A file path mentioned in conversation

If none, ask: "Which script would you like to audit?"

Read the full script before proceeding.

### 2. Ask Mode

Present the user with three options:

- **Thorough (default)** — Collaborative, section-by-section deep audit. You are a co-auditor:
  reading code, running chunks, inspecting data objects, catching issues yourself. Claude guides
  the systematic walk-through and adds observations. This is pair code review.
- **Fast** — Claude reads the whole script and identifies issues independently, then discusses
  findings together with the user.
- **Report only** — Claude audits independently and produces a report. User reads it on their
  own time.

### 3. Ask for Specific Concerns

"Is there anything in particular you're worried about or want me to focus on?"

This lets the user flag known weak points, steer attention to a specific category, or provide
context about what the script is supposed to do.

---

## Domain Verification Phase

**This phase runs in all modes** (thorough, fast, report-only) before the code audit begins.
Its purpose is to close the gap between code-level review and domain-specific correctness by
researching the actual tools, data formats, and analytical methods the script uses — then
auditing the code against that verified knowledge.

### Why This Matters

The most dangerous bugs in bioinformatics are **misunderstandings of what the tools and data
actually do.** A script can be syntactically correct and still produce wrong results because the
author didn't know that:
- `NormalizeData()` with default `scale.factor = 10000` produces log1p-normalized counts, not
  counts per million — downstream tests may assume different input
- `FindAllMarkers()` uses a one-vs-rest comparison — not the same as pairwise cluster DE
- `SCTransform()` residuals are not interchangeable with log-normalized counts for all
  downstream methods
- `inner_join()` silently drops unmatched rows
- Seurat's `subset()` resets PCA/UMAP embeddings — you may need to rerun dimensional reduction
- Spatial `Visium` spot coordinates are in pixel space unless explicitly converted
- `RunHarmony()` integrates by embedding, not by correcting the count matrix
- Cell Ranger feature-barcode matrices include all detected barcodes, not just cells —
  filtering thresholds matter

These are **domain assumptions** — facts about tools and methods that the code depends on but
doesn't state. The domain verification phase makes them explicit and checks them.

### How It Works

#### 1. Inventory Tools, Formats, and Methods

After reading the script, identify every external dependency the code relies on:

- **Data formats** being read or written (Seurat objects, AnnData/H5AD, 10x H5, MEX format,
  CSV/TSV metadata, RDS, loom, SpatialExperiment, etc.)
- **R packages and functions** called (Seurat, Harmony, SCTransform, scran, DESeq2, edgeR,
  limma, Monocle3, CellChat, STARsolo, spaceranger, ggplot2, dplyr, etc.)
- **Statistical methods** or analytical approaches (normalization, clustering, differential
  expression, trajectory analysis, cell-cell communication, spatial analysis, etc.)
- **Package-specific behaviors** (how Seurat handles assay slots, how dplyr handles NAs in
  groupby, how ggplot2 drops NAs silently, etc.)

#### 2. Research Critical Assumptions

For each tool/method, use **WebSearch and WebFetch** to pull the relevant documentation and
identify the critical behaviors the code must handle correctly. Focus on:

- **Object structure:** What does one slot or assay contain? (Raw counts? Normalized? Scaled?
  Which assay is "active"?)
- **Default behaviors:** What does the function do silently? (Which assay does it pull from?
  Does it overwrite existing results? Does it subset?)
- **Parameter semantics:** What do specific values mean? (resolution in FindClusters, dims in
  RunUMAP, latent.vars in FindMarkers)
- **Input assumptions:** What does the function expect? (log-normalized, raw counts, SCT
  residuals?)
- **Edge cases:** What happens with very small clusters, all-zero genes, samples with very
  different cell counts, missing metadata?
- **Known gotchas:** What do people commonly get wrong with this tool? (Seurat GitHub issues,
  Bioconductor forums, method papers)

Produce a **Domain Assumptions Checklist** — a concrete list of facts that the code depends on,
each verified against documentation. Format:

```
DOMAIN ASSUMPTIONS CHECKLIST
─────────────────────────────
Package/Method: Seurat normalization + DE
  ✓ NormalizeData() log1p-normalizes to scale.factor (default 10000), stores in "data" slot
  ✓ ScaleData() stores in "scale.data" slot — used by PCA, not by FindMarkers by default
  ✓ FindAllMarkers() uses Wilcoxon rank-sum by default, one cluster vs. all others
  ✓ FindMarkers() with test.use = "DESeq2" requires raw counts (RNA counts assay, slot = "counts")
  ? Whether SCTransform residuals or log-normalized counts are active assay — need to check

Package/Method: Harmony integration
  ✓ RunHarmony() corrects the PCA embedding, NOT the count matrix
  ✓ Downstream UMAP and clustering use corrected embedding — must specify reduction = "harmony"
  ✓ DE should still use uncorrected counts, not Harmony-corrected values

Package/Method: dplyr joins on metadata
  ✓ inner_join() drops unmatched rows on both sides — silent data loss
  ✓ left_join() keeps all rows from left table, NAs for unmatched right
  ? Whether all cell barcodes in metadata match those in Seurat object
```

Mark each assumption: ✓ (verified against docs), ✗ (contradicted by docs — potential BUG),
? (couldn't verify — flag for manual review).

#### 3. Audit Code Against Checklist

With the checklist in hand, trace through the code and verify that each assumption is handled
correctly:

- An assumption marked ✗ becomes a **BUG** or **CONCERN** finding
- An assumption marked ? becomes a **WARNING** with "needs manual domain review"
- An assumption marked ✓ that the code handles incorrectly becomes a **BUG**
- An assumption marked ✓ that the code handles correctly is noted as a **Good Practice**

#### 4. Recommend Assumption Blocks

After the audit, recommend that the script include an explicit **ASSUMPTIONS block** documenting
the critical domain assumptions the code depends on. This makes future audits faster and helps
team members understand what the code takes for granted:

```r
# ASSUMPTIONS (verified against Seurat v5 docs, 2024):
# - Active assay is "RNA"; NormalizeData output is in "data" slot (log1p, scale.factor = 10000)
# - SCTransform was NOT run — log-normalized workflow throughout
# - FindAllMarkers uses Wilcoxon one-vs-rest; not equivalent to pairwise comparison
# - Harmony corrects embedding only; DE uses uncorrected RNA counts
# - Metadata join on cell barcodes verified to have no unmatched rows (checked 2024-03-15)
```

### Depth Scaling

- **Report-only:** Quick checklist from background knowledge + targeted web searches for
  unfamiliar tools or methods. Flag unknowns as ? rather than spending time researching deeply.
- **Fast:** Full research phase with web searches. Produce verified checklist. Flag remaining
  unknowns for discussion.
- **Thorough:** Full research phase, then walk through the checklist with the user before
  starting the code audit. The user adds domain knowledge ("we verified this threshold
  experimentally"), resolves ? items, and may flag additional assumptions the checklist missed.

---

## The 5 Audit Categories

Every section of the script is evaluated against these categories. Each finding is tagged with
its category and severity.

### 1. Correctness (bugs)
- Off-by-one errors, wrong variable references, typos in column names or metadata fields
- Logic errors (wrong condition, inverted filter, incorrect formula)
- Functions used incorrectly (wrong assay slot, wrong arguments, misunderstood return values)
- Order dependencies (e.g., running FindMarkers before NormalizeData)
- Mismatches between what the code does and what the comments say

### 2. Analytical Reasoning
- Is the statistical test appropriate for this data and question?
- Is the normalization method appropriate for the experimental design?
  (e.g., log-normalize vs. SCTransform vs. raw counts for pseudobulk DE)
- Are thresholds justified or arbitrary? (min.pct, logfc.threshold, resolution)
- Are comparisons properly controlled? (batch effects accounted for?)
- Could the analysis be misleading even if technically correct?
  (e.g., visualizing UMAP from unintegrated data after Harmony was run)
- Are there alternative approaches that would be more appropriate?

### 3. Data Handling
- Silent row/cell drops (joins, filters, NA removal)
- Unvalidated assumptions about object structure (which assay is active, slot contents)
- Missing checks on cell counts, gene counts, or metadata completeness
- Unchecked NAs propagating through calculations
- Joins on metadata that could introduce duplicates or lose cells
- Aggregation that hides important variation across samples or conditions

### 4. Style & Organization
- Code clarity and readability
- Variable naming (are Seurat object names descriptive of their state?)
- Comments (too few, misleading, or unnecessary)
- Chunk organization and labels in .Rmd files
- Magic numbers without explanation (e.g., `res = 0.4` with no justification)
- Repeated code that should be a function or loop
- **Over-engineering** — unnecessary defensive code, validation for impossible cases,
  abstractions that add complexity without benefit. Simple, readable code is a feature.

### 5. Reproducibility
- Hardcoded paths or absolute file paths (should use relative paths or `here::here()`)
- Missing `set.seed()` before clustering or UMAP
- Package versions not documented (consider `sessionInfo()` at end of script)
- Missing input file documentation (which Seurat object? from which script?)
- Outputs not clearly tied to input versions
- Results that depend on random initialization without seed

---

## Severity Levels

- **BUG** — Incorrect behavior; produces wrong results *in the script's actual use case*. Must fix.
- **CONCERN** — Analytically questionable; may produce misleading results. Should investigate.
- **WARNING** — Not wrong, but fragile or risky. Should address.
- **NOTE** — Style, clarity, or minor improvement. Nice to fix.
- **FYI** — A pattern worth being aware of, but not something to change. Used for theoretical
  fragilities that don't apply in the script's actual context. Informational only.

### Severity Calibration

Before assigning severity, consider:
- **Is this a real problem or a theoretical one?** If the script processes a known, fixed Seurat
  object and the "issue" only manifests with different input, it's FYI, not BUG.
- **Would the fix make the script simpler or more complex?** If more complex, the cure may be
  worse than the disease.
- **Is this a one-time script or a reusable pipeline?** One-time scripts should be correct for
  their specific task. Reusable pipelines need more robustness.

---

## Thorough Mode: Collaborative Section-by-Section Audit

The user is a co-auditor. Claude does NOT pre-digest the script — both work through it together.
The process of finding issues is as valuable as the findings themselves.

### 1. Script Overview

Read the script and present:
- What it does (plain language)
- What data it processes and what it produces
- A numbered map of logical sections

Ask: "Does this match your understanding of what this script should do?" Mismatches between
intent and implementation are a finding category.

### 2. Domain Verification (collaborative)

Run the full Domain Verification Phase. In thorough mode:
- Research tools and methods, produce the Domain Assumptions Checklist
- Present the checklist to the user before starting the code walk-through
- Walk through each assumption: "I found that FindAllMarkers uses one-vs-rest comparisons —
  does that match your analytical intent?"
- The user adds domain knowledge, resolves ? items, and may flag assumptions the checklist missed
- This step builds shared understanding of what the code *should* do before examining whether
  it actually does

### 3. Section-by-Section Audit

For each logical section:

**a. Present the code chunk** (~15–20 lines max at a time)

**b. User reads and runs it.** Encourage the user to:
- Read the code before Claude explains anything
- Run the chunk in their R console or knit the chunk
- Inspect intermediate objects (`str()`, `dim()`, `head()`, `table()`, `summary()`)
- For Seurat objects: check `Assays()`, `Reductions()`, `DefaultAssay()`, `ncol()`, `nrow()`
- Flag anything that looks off or that they don't understand

**c. Claude probes and suggests checks** — targeted to what's most likely to go wrong with
this specific type of code. Ask questions, suggest diagnostics, and raise concerns. The user
adds domain context and responds. Findings are documented as they emerge.

**For data loading / object construction:**
- "Let's verify dimensions — how many cells and features did we load? Is that what you expect?"
- "What's the active assay? Let's check `DefaultAssay()` before we proceed"
- "Are there NAs in key metadata columns? `table(is.na(obj$condition))`"
- "Does the cell count match the expected number after filtering?"

**For normalization / preprocessing:**
- "Is this log-normalization or SCTransform? Let's confirm which assay slot has the values
  we'll use downstream"
- "Is `ScaleData()` being run on all genes or just HVGs? Does that affect downstream steps?"
- "If SCTransform was used earlier, are we consistent about which assay is active here?"

**For dimensionality reduction / integration:**
- "Which reduction are we using for UMAP — PCA or Harmony? Let's confirm
  `Reductions(obj)` includes what we expect"
- "Was `set.seed()` called before RunUMAP? Results won't be reproducible otherwise"
- "Are we carrying the Harmony embedding into clustering, or accidentally using uncorrected PCA?"

**For clustering:**
- "What resolution is this, and how was it chosen?"
- "How many clusters did we get? Is that biologically reasonable for this tissue/experiment?"
- "Are there any very small clusters that might be artifacts?"

**For differential expression:**
- "Which assay and slot are being used for DE? Is that the right one for this test?"
- "Is this one-vs-rest or pairwise? Does that match the biological question?"
- "Are we correcting for batch or other covariates that could confound results?"
- "What's the distribution of p-values — does it look well-calibrated?"

**For cell type annotation:**
- "Are the marker genes used for annotation consistent with published markers for this tissue?"
- "Are any cluster labels ambiguous or unsupported by the data?"
- "How are doublets being handled?"

**For spatial data (Visium/STARmap/MERFISH):**
- "Are spot coordinates in pixel space or tissue space? Has the conversion been applied?"
- "Is the spatial neighborhood graph constructed correctly for the resolution of this tissue?"
- "Are we accounting for the fact that spots may contain multiple cell types?"

**For plotting:**
- "Does this plot accurately represent the underlying data?"
- "Are color scales, axis limits, and labels correct?"
- "Are there enough cells per group to make this visualization meaningful?"

**d. Run diagnostics together** when something is suspicious:
- "Let's check — run `anti_join()` on these metadata tables and see how many rows don't match"
- "Try `table(obj$seurat_clusters)` — is the cluster size distribution what you'd expect?"
- "Comment out this filter and check how many cells we'd lose"

**e. Document findings** — tag with category, severity, lines, and recommendation.

### 4. Cross-Section Analysis

After all individual sections:
- Trace data flow across the full script together
- Check: do preprocessing decisions in section A affect correctness in section D?
- Look for cascading issues (e.g., wrong active assay early → wrong DE test later)
- Verify the overall analytical argument holds together
- Look for things the script should be doing but isn't

### 5. Produce and Save Audit Report

Compile findings into the structured report format below.
Save to `outputs/audit_reports/{script_name}_audit_report.md` in the project root.
Create the directory if it doesn't exist.

### Pacing

Every 2–3 sections, briefly check in: "How's the depth? Want to go faster or deeper?"

---

## Fast Mode: Claude-Driven Audit with Discussion

Claude works through the script independently, then discusses findings with the user.

### 1. Full Script Read

Read the entire script.

### 2. Domain Verification

Run the full Domain Verification Phase. Research tools and methods via web searches, produce
verified Domain Assumptions Checklist, flag remaining unknowns (?) for discussion.

### 3. Systematic Analysis

Apply the 5-category checklist across all sections:
- Trace data flow from input to output
- Check analytical reasoning and statistical assumptions
- Look for silent data loss, unvalidated object states, missing checks
- Evaluate style, organization, and reproducibility
- Run diagnostics where possible (dimension checks, NA counts, join validation)
- Check code against the Domain Assumptions Checklist

### 4. Produce and Save Audit Report

Full structured report with all findings, including Domain Assumptions Checklist.
Save to `outputs/audit_reports/{script_name}_audit_report.md`.

### 5. Collaborative Review of Findings

Present findings ordered by severity (BUG first):
- For each finding: show the code, explain the issue, discuss implications
- Present unresolved domain assumptions (? items) for the user's domain input
- User adds context, agrees/disagrees, reclassifies severity
- Together decide: fix now, defer, mark as acceptable
- Report updated with collaborative decisions and final dispositions

---

## Report-Only Mode

Same as fast mode steps 1–4. No collaborative review. Produces the report and saves it.
Findings are marked as "Unreviewed" in the status column.

Best for: batch auditing multiple scripts, quick quality snapshots, or when the user will
review the report in a separate session.

---

## Diagnostic Capabilities

When auditing, Claude should actively run diagnostics (in Claude-driven modes) or suggest them
(in collaborative mode):

- **Dimension checks:** `dim()`, `ncol()`, `nrow()` before and after key operations
- **Object state checks:** `DefaultAssay()`, `Assays()`, `Reductions()`, `Layers()` for Seurat objects
- **NA checks:** Track where NAs enter metadata and how they flow through the script
- **Join validation:** `anti_join()` to check unmatched rows on both sides
- **Distribution checks:** `summary()`, `hist()`, `VlnPlot()` for key variables, especially
  before statistical tests
- **Cluster checks:** `table(obj$seurat_clusters)` — are cluster sizes reasonable?
- **Seed checks:** Is `set.seed()` called before any stochastic operation?

---

## Audit Report Format

```markdown
# Script Audit Report: {script_name}

**Date:** {date}
**Script:** {path/to/script}
**Auditor:** Claude Code {+ user name, if collaborative}
**Mode:** {Thorough / Fast / Report only}

## Summary

- **Total findings:** {N}
- **By severity:** {N} BUG, {N} CONCERN, {N} WARNING, {N} NOTE, {N} FYI
- **By category:** {N} Correctness, {N} Analytical, {N} Data Handling, {N} Style, {N} Reproducibility
- **Overall assessment:** {1-2 sentence summary of script quality and most critical issues}

## Domain Assumptions Checklist

| Tool/Method | Assumption | Verified? | Code Handles? | Finding |
|-------------|-----------|:---------:|:-------------:|---------|
| {tool} | {assumption} | ✓ / ✗ / ? | Yes / No / N/A | {ref or "OK"} |

## Findings

### BUG-1: {Short description}
- **Category:** {Correctness / Analytical / Data Handling / Style / Reproducibility}
- **Section:** {section name or chunk label}
- **Lines:** {line range}
- **Description:** {What the issue is}
- **Impact:** {What goes wrong because of this}
- **Recommendation:** {How to fix it}
- **Status:** {Open / Discussed — {outcome} / Fixed / Unreviewed}

### CONCERN-1: {Short description}
...

### WARNING-1: {Short description}
...

### NOTE-1: {Short description}
...

### FYI-1: {Short description}
- **Category:** {category}
- **Lines:** {line range}
- **Description:** {What the pattern is and why it's worth knowing about}
- **Why not an action item:** {Why this doesn't need to change in this script's context}

## Sections Reviewed

| Section / Chunk | Lines | Issues Found | Notes |
|-----------------|-------|-------------|-------|
| {name} | {range} | BUG-1, WARN-2 | {brief note} |
| {name} | {range} | None | Clean |

## Analytical Decisions Inventory

| Section | Decision | Current Choice | Justification | Alternatives | Risk Level |
|---------|----------|---------------|---------------|-------------|------------|
| ... | ... | ... | ... | ... | ... |

## Action Items

| Priority | Finding | Action | Owner |
|----------|---------|--------|-------|
| 1 | BUG-1 | Fix immediately | {name} |
| 2 | CONCERN-1 | Investigate | {name} |
```

**Always save the report** to `outputs/audit_reports/{script_name}_audit_report.md` in the
project root. Create the `outputs/audit_reports/` directory if it doesn't exist. Every audit
must produce a saved report file — this is not optional.

---

## Audit Principles

1. **Trace the data, not just the code.** The most important bugs in single-cell analysis are
   data flow bugs — wrong assay slot, wrong object state, silent cell drops. Follow the data
   from input to output.
2. **Question analytical defaults.** Just because `resolution = 0.5` or `min.pct = 0.1` is
   common doesn't mean it's right for this dataset. Every default is a choice.
3. **Check what's NOT in the script.** Missing seed setting, missing validation of object state,
   missing documentation of thresholds — these are findings too.
4. **Severity is about impact, not aesthetics.** A confusing variable name is a NOTE. A
   confusing variable name that leads someone to use the wrong cluster label is a BUG.
5. **Be specific.** "This join might lose cells" is not helpful. "This inner_join on line 47
   drops 312 cells because barcodes in `meta` don't match the Seurat object" is actionable.
6. **Run diagnostics, don't guess.** When something looks suspicious, actually run the code
   to verify before reporting it as a finding.
7. **Credit good practices.** Note when the script does something well — especially clear
   annotations of thresholds, good use of `set.seed()`, or thoughtful DE design.
8. **Verify domain assumptions, don't assume.** When the code depends on package behavior,
   look it up. A verified assumption is worth ten educated guesses.
9. **Simplicity is a virtue, not a gap.** A script that does its job cleanly without handling
   every edge case is well-written, not incomplete. If a fix would make the script longer and
   harder to read, reconsider whether it's worth reporting as an action item — it may be FYI.
10. **Calibrate to the script's role.** A one-time script on a known Seurat object needs
    different rigor than a reusable pipeline. Don't treat every script as if it will be rerun
    on unknown inputs.

---

## Claude Code Behavior

When this skill is active:

- **Be direct, not hedging.** "This join silently drops 312 cells" not "This join might
  potentially have some issues with cell counts."
- **Show evidence.** When flagging an issue, show the specific code and explain exactly what
  goes wrong. Run diagnostics where possible.
- **Distinguish fact from opinion.** "This uses an inner join that drops cells" (fact) vs.
  "A left join would be safer here" (recommendation). Both valid; clearly distinguished.
- **Don't over-report.** Not every line needs a finding. If a section is clean, say so and
  move on. Audit fatigue from low-severity noise degrades the value of real findings.
- **Protect simplicity.** Never push scripts toward unnecessary complexity. If a recommendation
  would add code to handle a theoretical edge case that won't occur, use FYI severity instead.
  Actively flag existing over-engineering as a style finding.
- **Respect the author's context.** In collaborative mode, the author may have reasons for
  choices that aren't documented. Ask before assuming something is wrong.
- **Track uncertainty.** "This might be intentional, but if not, it would cause..." is better
  than a false positive or a missed bug.
- **In thorough mode: don't pre-digest.** Let the user read and run the code first. Ask
  questions, don't give answers. The user finding issues themselves is the point.
- **In fast mode: be comprehensive.** You're working alone — don't skip sections or categories.
- **Do not use subagents for audits.** Run the audit directly in the current conversation.
