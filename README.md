# Foster 2021 Wound Fibroblast Reanalysis: Progenitor Contribution to Wound Healing

## Summary

Reanalysis of Foster et al. 2021 PNAS (GSE178758) to ask whether progenitor-like
fibroblast states drive dermal wound healing, and whether the same populations
are implicated in SSc fibrosis. Uses scRNA-seq, scATAC-seq, and Visium spatial
transcriptomics from Rainbow-labeled wound fibroblasts (POD 0, 2, 7, 14).

## Core question

Do progenitor-like fibroblast populations (DPP4+/PI16+, SFRP2+, adipogenic
DLK1+) initiate and drive wound repair, and do they persist in SSc skin
fibrosis?

## Reproduction

Run scripts in numbered order:

1. `scripts/01_download_and_qc.Rmd` — download GSE178758, initial QC
2. `scripts/02_integration_clustering.Rmd` — integrate timepoints, cluster, annotate
3. `scripts/03_progenitor_scoring.Rmd` — score progenitor signatures across cells
4. `scripts/04_trajectory_analysis.Rmd` — pseudotime and RNA velocity
5. `scripts/05_scATAC_chromatin_priming.Rmd` — multi-lineage priming assessment
6. `scripts/06_spatial_visium.Rmd` — spatial mapping of progenitor states
7. `scripts/07_ssc_integration.Rmd` — integration with Tabib SSc dataset

## Data sources

- **GSE178758** — Foster et al. 2021 PNAS: scRNA-seq, scATAC-seq, Visium of
  stented mouse wounds, POD 0/2/7/14, inner vs outer regions, mCerulean+
  Rainbow-labeled fibroblasts
- **GSE138669** or current Tabib accession — SSc vs healthy dermal fibroblasts
  (pending accession confirmation)
- **GSE113854 / GSE113605** — Guerrero-Juarez/Plikus wound fibroblasts
  (optional secondary validation)

## Key caveats

- Foster scRNA-seq is sorted on mCerulean+ only — single Rainbow color, ~25%
  of recombined fibroblasts. Cluster proportions are biased.
- Lineage-negative sort excludes CD31+/CD45+/TER119+ cells. No myeloid
  contribution can be assessed from this dataset.
- Most cells are reticular (DLK1+/SCA1-); papillary and hypodermal are
  underrepresented.
- No clonal identity retained at the transcriptome level — Rainbow color was
  used for sorting only.

## Contact

Horsley Lab. Primary analyst: TBD.
