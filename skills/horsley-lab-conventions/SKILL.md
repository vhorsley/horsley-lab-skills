---
name: horsley-lab-conventions
description: >
  Horsley Lab standard project and script organization conventions for R-based
  data analysis. Use when setting up a new project, when someone asks how to
  organize their analysis, or when reviewing a project structure. Also use when
  a lab member seems to be working without a clear folder structure.
---

# Horsley Lab Project Conventions

## Standard Project Structure

```
project-name/
├── data/
│   ├── raw/          ← original files, never modified
│   └── processed/    ← cleaned/transformed data
├── scripts/
│   ├── 01_qc.Rmd          ← numbered in order of execution
│   ├── 02_preprocessing.Rmd
│   ├── 03_analysis.Rmd
│   └── scratch/           ← working/exploratory files (not numbered)
├── outputs/
│   ├── figures/
│   └── tables/
├── docs/              ← notes, protocols, manuscript drafts
└── README.md
```

## Script Naming Rules

- **Numbered scripts** (`01_`, `02_`, etc.) are the canonical analysis pipeline —
  they should run in order and reproduce all results
- **Scratch files** in `scripts/scratch/` are exploratory — name them anything,
  but consolidate useful code into a numbered script before finishing
- Use `.Rmd` for all numbered scripts so code and notes live together
- Use lowercase with underscores: `03_cell_clustering.Rmd` not `3-CellClustering.R`

## Data Rules

- Raw data in `data/raw/` is **read-only** — scripts never write there
- Every processed file should be traceable to the script that created it
- Large files (>50MB) go in a `data/external/` folder and are noted in README,
  not committed to GitHub

## Outputs

- Figures go in `outputs/figures/` with descriptive names: `umap_cluster_labels.pdf`
- Every figure should be regeneratable from a numbered script
- Keep a `outputs/figures/figure_log.md` noting which script made which figure

## README minimum contents

Every project should have a README with:
1. One-sentence project summary
2. How to reproduce the analysis (which scripts, in what order)
3. Data sources and any access notes
4. Contact (who owns this project)
