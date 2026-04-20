---
name: new-plan
description: >
  Use when starting multi-script or multi-session work that needs tracking, or when the
  user invokes /new-plan. Creates a planning document and registers it in the project CLAUDE.md.
user-invocable: true
---

# Create a New Planning Document

When the user invokes `/new-plan`, create a structured planning document and register it
in the project's document registry.

## 1. Gather Information

Ask the user:
- **Topic**: What is this plan for? (e.g., "wound healing scRNAseq", "adipocyte EV figures", "manuscript revision")
- **Category**: Planning doc (tracks status of ongoing work) or Data doc (describes datasets, file formats, experimental design)?
- **Style**: Data science (numbered scripts, inputs/outputs tracking) or General (tasks, milestones, components)?
- **Task tracking** (planning docs only): Checkboxes (simple open/completed lists) or Status table (phased rows with status column)?

If the user provides the topic directly as an argument (e.g., `/new-plan CGRP wound angiogenesis`),
use that and still ask the remaining questions.

Streamline the questions — if the answers are obvious from context, propose defaults and let
the user confirm. For example, a data science project asking about "clustering and annotation"
likely wants data science style with a status table.

## 2. Create the Document

Create a new `.md` file in the project's `.claude/` directory.

### Naming convention
- Use `SCREAMING_SNAKE_CASE` matching existing docs
- Planning docs: `{TOPIC}_PLAN.md` (e.g., `WOUND_HEALING_SCRNA_PLAN.md`)
- Data docs: `{TOPIC}_DATA.md` (e.g., `VISIUM_SKIN_DATA.md`)

---

## Planning Document Templates

### General planning doc (checkbox style)

For general projects (manuscripts, grant sections, lab resources) with simple task tracking.

```markdown
# {Topic} Plan

> Last updated: {today}

## Overview

{Brief description of what this plan covers}

## Goals

- [ ] Goal 1
- [ ] Goal 2

## Open Tasks

{Group tasks by category if there are many}

- [ ] Task 1
- [ ] Task 2

## Key Files

- (list key files and components as they become relevant)

## Key Decisions

- (record decisions with dates as they are made — accumulates, never trimmed)

## Working Notes

(free-form context for next session — what's in progress, gotchas, context that
doesn't fit elsewhere)

## Completed

- [x] (move completed items here)
```

### General planning doc (status table style)

For general projects that benefit from phased tracking.

```markdown
# {Topic} Plan

> Last updated: {today}

## Overview

{Brief description of what this plan covers}

## Goals

- [ ] Goal 1
- [ ] Goal 2

## Status

| Phase | Description | Status |
|-------|-------------|--------|
| 1 | {First phase} | Not started |

## Key Files

- (list key files and components as they become relevant)

## Key Decisions

- (record decisions with dates as they are made — accumulates, never trimmed)

## Working Notes

(free-form context for next session — what's in progress, gotchas, context that
doesn't fit elsewhere)
```

### Data science planning doc

For analysis projects with numbered `.Rmd` scripts, data pipelines, and input/output tracking.

```markdown
# {Topic} Plan

> Last updated: {today}

## Overview

{Brief description of what this plan covers}

## Goals

- [ ] Goal 1
- [ ] Goal 2

## Status

| Phase | Description | Status |
|-------|-------------|--------|
| 1 | {First phase} | Not started |

## Key Files

### Inputs
- (list input files as they become relevant — Seurat objects, count matrices, metadata)

### Outputs
- (list output files as they are created — processed objects, figures, tables)

### Scripts

Track all scripts here. When a script is superseded, move it to the Legacy section.

#### Active
| Script | Purpose | Status |
|--------|---------|--------|
| (add scripts as they are created) | | |

#### Legacy / Inactive
| Script | Replaced by | Notes |
|--------|-------------|-------|
| (move superseded scripts here) | | |

## Key Decisions

- (record decisions with dates as they are made — accumulates, never trimmed)
- Example: 2024-03-15 — Excluded sample WH_003 (low cell count <500 cells after QC)
- Example: 2024-03-18 — Using log-normalize not SCTransform; CellChat requires RNA assay

## Working Notes

(free-form context for next session — what's in progress, gotchas, context that
doesn't fit elsewhere)
```

---

## Data Document Templates

### General data doc

For documenting reference files, marker gene lists, or any structured reference material.

```markdown
# {Topic} Data

## Overview

{Brief description of what this document covers}

## Purpose

{Why this data exists and how it's used in the analysis}

## Key Files

| File | Description |
|------|-------------|
| | |

## Structure / Schema

{Describe the structure or format of the key files}

## Notes

{Any important notes about usage, edge cases, or conventions}
```

### Data science data doc

For documenting experimental datasets with biological context.

```markdown
# {Topic} Data

## Overview

{Brief description of the dataset}

## Experimental Design

{Describe the experiment that generated this data — conditions, timepoints, genotypes,
tissue types, number of biological replicates, sequencing platform}

## Samples

| Sample ID | Condition | Timepoint | Replicate | Notes |
|-----------|-----------|-----------|-----------|-------|
| | | | | |

## Key Files

| File | Description | Location |
|------|-------------|----------|
| | | |

## QC Summary

{Brief notes on cell counts, sequencing depth, any samples excluded and why}

## Processing Notes

{How data was processed — Cell Ranger version, genome reference, filtering thresholds,
any important preprocessing decisions}
```

---

## Maintaining Plans Over Time

### Collapse completed phases

When a phase is marked complete, collapse its detailed content to a **3-5 line summary**.
Keep the phase heading and a brief record of what was done and key outcomes, but remove
per-item checklists and working notes that are no longer needed. This prevents plans from
growing monotonically.

**Before collapsing, check:** Does any information in this phase feed into a future phase
that hasn't started yet? If so, ensure that information is captured in the future phase's
section before removing it from the completed phase.

Example — before collapsing:
```markdown
## Phase 2: QC and Filtering ✓ COMPLETE

### Per-sample QC results
(40 lines of per-sample metrics...)

### Filtering decisions
(20 lines of threshold exploration...)
```

After collapsing:
```markdown
## Phase 2: QC and Filtering ✓ COMPLETE

Processed 8 samples (4 wounded, 4 unwounded). Excluded WH_003 (482 cells, below 500
threshold). Final object: 31,847 cells. Thresholds: nFeature 200-6000, percent.mt <20.
Full QC plots in outputs/figures/01_qc/.
```

---

## 3. Register in Project CLAUDE.md

After creating the document, add it to the **Project Document Registry** in the project's
`.claude/CLAUDE.md`:

1. Read `.claude/CLAUDE.md` and find the "Project Document Registry" section
2. Add a new row to the appropriate table (Planning Documents or Data Documents)
3. Include the document name, topic, and whether it has a status table

If no registry exists, create the registry section at the bottom of CLAUDE.md:

```markdown
## Project Document Registry

### Planning Documents
| Document | Topic | Style |
|----------|-------|-------|
| `WOUND_HEALING_SCRNA_PLAN.md` | Wound healing scRNAseq analysis | Data science, status table |

### Data Documents
| Document | Topic |
|----------|-------|
| `VISIUM_SKIN_DATA.md` | Spatial transcriptomics experimental design |
```

## 4. Confirm

Tell the user:
- What file was created and where
- That it was registered in the project CLAUDE.md
- Suggest they start filling in the details or offer to help plan the work
