---
name: data-handling
description: >
  Data handling best practices for R-based data analysis scripts in the Horsley Lab.
  Use when writing data manipulation code, analysis pipelines, or .Rmd scripts that
  process single-cell RNA-seq, spatial transcriptomics, bulk RNA-seq, proteomics, or
  other biological datasets (filtering, joining, normalizing, clustering). Also applies
  to Python analysis scripts. Do NOT load for general scripting, infrastructure code,
  or configuration management.
user-invocable: false
---

# Data Handling Best Practices

When writing data analysis code, follow these practices to ensure transparency,
reproducibility, and catch errors early. The Horsley Lab works primarily in R with
Seurat, but these principles apply to Python workflows as well.

---

## 1. Organize Inputs at the Top

Group all data reads at the top of each script or in a dedicated setup chunk, with
comments distinguishing external data from other scripts' outputs:

```r
# --- Inputs (from other scripts) ---
seu <- readRDS(here("data/processed/02_seu_normalized.rds"))
mdata <- read_csv(here("data/processed/metadata_annotated.csv"))

# --- Inputs (external data) ---
marker_genes <- read_csv(here("data/raw/published_skin_markers.csv"))
```

This makes dependencies self-documenting: reading the top of any script shows exactly
what it needs and where those files come from.

---

## 2. Show Data at Key Steps

Include summaries whenever datasets are created or substantially altered.

### Seurat objects

```r
# After loading
seu <- readRDS(here("data/processed/02_seu_normalized.rds"))
cat("Loaded Seurat object:", ncol(seu), "cells,", nrow(seu), "features\n")
cat("Active assay:", DefaultAssay(seu), "\n")
cat("Available assays:", paste(Assays(seu), collapse = ", "), "\n")
cat("Available reductions:", paste(Reductions(seu), collapse = ", "), "\n")
cat("Samples:", paste(unique(seu$orig.ident), collapse = ", "), "\n")

# After filtering
seu <- subset(seu, subset = nFeature_RNA > 200 & percent.mt < 20)
cat("After QC filter:", ncol(seu), "cells retained\n")

# After clustering
seu <- FindClusters(seu, resolution = 0.5)
cat("Clusters found:", length(unique(seu$seurat_clusters)), "\n")
print(table(seu$seurat_clusters))

# After cell type annotation
cat("Cell type distribution:\n")
print(table(seu$cell_type))
```

### Metadata joins

```r
# Before join — check key format on both sides
cat("Cell barcodes in Seurat (first 3):", head(colnames(seu), 3), "\n")
cat("Cell barcodes in metadata (first 3):", head(mdata$barcode, 3), "\n")
cat("Overlap:", sum(colnames(seu) %in% mdata$barcode), "of", ncol(seu), "cells\n")

# After join
seu <- AddMetaData(seu, metadata = mdata %>% column_to_rownames("barcode"))
cat("After metadata join:", sum(!is.na(seu$new_column)), "cells with annotation\n")
```

### Bulk RNA-seq / counts matrices

```r
# After loading
counts <- read_csv(here("data/raw/counts.csv")) %>% column_to_rownames("gene_id")
cat("Loaded counts matrix:", nrow(counts), "genes,", ncol(counts), "samples\n")

# After filtering
keep <- rowSums(counts > 1) >= 3
counts_filtered <- counts[keep, ]
cat("After expression filter:", sum(keep), "of", nrow(counts), "genes retained\n")
```

**When to show data:**
- After loading any data object
- After filtering or subsetting
- After joins or metadata additions
- After normalization or transformation
- After clustering or dimensionality reduction
- After cell type annotation
- Before final output or plotting

---

## 3. Annotate Analytical Decisions

Any operation that transforms, scales, filters, or interprets data should be annotated
with the reasoning. Document the "why" directly in code comments or markdown text.

```r
# Log-normalize to 10,000 counts per cell (library size normalization)
# Using log-normalize rather than SCTransform because downstream CellChat
# requires log-normalized counts in the RNA assay
seu <- NormalizeData(seu, normalization.method = "LogNormalize", scale.factor = 10000)

# Filter low-quality cells: nFeature_RNA > 200 removes empty droplets,
# < 6000 removes likely doublets; percent.mt < 20 removes dying cells
# Thresholds chosen based on QC plots from 01_qc.Rmd
seu <- subset(seu, subset = nFeature_RNA > 200 &
                             nFeature_RNA < 6000 &
                             percent.mt < 20)
cat("After QC filter:", ncol(seu), "cells retained\n")

# Resolution 0.5 chosen after testing 0.2-1.0 — gives biologically
# interpretable clusters without over-splitting fibroblast populations
seu <- FindClusters(seu, resolution = 0.5)

# Using Wilcoxon for DE (default) — appropriate for scRNAseq cluster markers;
# pseudobulk DESeq2 used for sample-level comparisons in 05_de.Rmd
markers <- FindAllMarkers(seu, only.pos = TRUE, min.pct = 0.25,
                          logfc.threshold = 0.25)
```

**What to annotate:**
- Normalization method and why it suits this data
- QC filtering thresholds and their rationale
- Clustering resolution and how it was chosen
- Statistical test choice and assumptions
- Which assay is active and why
- Integration method (Harmony, RPCA, etc.) and why
- Any deviation from defaults
- Sample exclusions and why

---

## 4. Seurat-Specific Data Handling

### Always check the active assay

```r
# Check before any analysis
cat("Active assay:", DefaultAssay(seu), "\n")

# Set explicitly rather than assuming
DefaultAssay(seu) <- "RNA"  # for DE, CellChat, most downstream analysis
# DefaultAssay(seu) <- "SCT"  # if using SCTransform workflow
```

### Verify assay slot contents

```r
# Check which slot has data before using it
cat("RNA counts slot (raw, first 3 genes):\n")
print(seu[["RNA"]]@counts[1:3, 1])
cat("RNA data slot (normalized, first 3 genes):\n")
print(seu[["RNA"]]@data[1:3, 1])
```

### Track cell counts through the pipeline

```r
cat("=== Cell counts through pipeline ===\n")
cat("After loading:        ", ncol(seu_raw), "cells\n")
cat("After QC filter:      ", ncol(seu_qc), "cells\n")
cat("After doublet removal:", ncol(seu_clean), "cells\n")
cat("After integration:    ", ncol(seu_integrated), "cells\n")
```

### Check reductions before using them

```r
# Verify the reduction exists
cat("Available reductions:", paste(Reductions(seu), collapse = ", "), "\n")

# If using Harmony, confirm downstream steps use harmony embedding NOT pca
# otherwise batch correction has no effect on clustering/UMAP
seu <- RunUMAP(seu, reduction = "harmony", dims = 1:30)
seu <- FindNeighbors(seu, reduction = "harmony", dims = 1:30)
```

### Spatial data (Visium)

```r
# After loading spatial object
cat("Spots:", ncol(seu_spatial), "\n")
cat("Images:", paste(Images(seu_spatial), collapse = ", "), "\n")

# Verify spatial coordinates are present
coords <- GetTissueCoordinates(seu_spatial)
cat("Spatial coordinate range — x:", range(coords$x), "y:", range(coords$y), "\n")

# Note: Visium spots contain multiple cell types — document deconvolution
# approach if used (e.g., RCTD, SPOTlight, BayesSpace)
```

---

## 5. Validate to Prevent Silent Data Loss

### Report cell/row counts before and after joins

```r
cat("Before join:", nrow(mdata), "rows\n")
mdata_joined <- mdata %>% inner_join(annotations, by = "cell_id")
cat("After inner join:", nrow(mdata_joined), "rows\n")
# inner_join drops unmatched rows on both sides — always check
```

### Check for unmatched keys

```r
unmatched <- mdata %>% anti_join(annotations, by = "cell_id")
if (nrow(unmatched) > 0) {
  cat("WARNING:", nrow(unmatched), "rows will not match\n")
  cat("Unmatched IDs (first 5):", head(unmatched$cell_id, 5), "\n")
}
```

### Validate expected columns exist

```r
required_cols <- c("cell_id", "condition", "sample_id")
missing <- setdiff(required_cols, names(mdata))
if (length(missing) > 0) stop("Missing columns: ", paste(missing, collapse = ", "))
```

### Assert expected data characteristics

```r
stopifnot("No cells after filter" = ncol(seu) > 0)
stopifnot("Unexpected NAs in condition" = !any(is.na(seu$condition)))
stopifnot("Cell counts don't match metadata" = ncol(seu) == nrow(mdata))
```

---

## 6. Hidden Sources of Data Loss

### Seurat-specific
- `subset()` resets PCA/UMAP embeddings — rerun dimensional reduction after subsetting
- `FindAllMarkers()` uses one-vs-rest, not pairwise — results differ from pairwise DE
- `SCTransform()` residuals are NOT interchangeable with log-normalized counts for all
  downstream tools — CellChat and some trajectory methods require log-normalized RNA assay
- `NormalizeData()` stores in `data` slot; `ScaleData()` stores in `scale.data` —
  `FindMarkers()` uses `data` slot by default, not `scale.data`
- `RunHarmony()` corrects the PCA embedding only — DE should use uncorrected counts
- `AddMetaData()` silently ignores cells not present in the Seurat object
- Harmony-corrected UMAP requires `reduction = "harmony"` in both `FindNeighbors()`
  and `RunUMAP()` — easy to accidentally use uncorrected PCA

### R/dplyr-specific
- `inner_join()` silently drops unmatched rows on both sides
- `lm()`, `glm()` drop rows with NAs silently — check `na.action`
- `factor()` drops unused levels when subsetting — use `droplevels()` intentionally
- `as.numeric()` on character introduces NAs silently
- `mean()`, `sum()` with `na.rm = TRUE` silently ignore NAs — report count:
  `cat("NAs ignored:", sum(is.na(x)), "\n")`
- Bioconductor packages (DESeq2, edgeR, AnnotationDbi) mask dplyr functions —
  always use `dplyr::filter()`, `dplyr::select()`, `dplyr::rename()` explicitly
- ggplot2 removes rows with NAs silently and clips data outside axis limits

### Bulk RNA-seq specific
- DESeq2 and edgeR apply independent filtering — genes can be dropped silently
- `calcNormFactors()` assumes most genes are NOT differentially expressed
- `voom()` requires raw counts, not pre-normalized data

---

## 7. Rmd Document Patterns

Put validation in chunks with `include = FALSE` but keep summaries visible:

````r
```{r validate-join, include=FALSE, message=TRUE}
# Validation (hidden in rendered output)
unmatched <- mdata %>% anti_join(annotations, by = "cell_id")
if (nrow(unmatched) > 0) {
  message("WARNING: ", nrow(unmatched), " rows unmatched in annotation join")
}
stopifnot(nrow(mdata_joined) > 0)
```

```{r show-result}
# Summary (visible in rendered output)
cat("Final dataset:", nrow(mdata_joined), "cells with annotations\n")
glimpse(mdata_joined)
```
````

**Key pattern:**
- `cat()` / `message()` — diagnostics; hide with `include = FALSE`
- `print()` / `glimpse()` — data summaries; keep visible
- `knitr::kable()` — formatted tables for rendered output

---

## Claude Code Behavior

### Show Your Work — Communicate During Coding

When writing analysis code interactively, **do not just write code and move on**. The
user is a scientist who needs to stay informed about what's happening to the data.
Treat coding as a conversation, not a monologue.

**After every significant data operation, report:**
- Object state: "Loaded Seurat object: 8,432 cells × 33,538 genes. Active assay: RNA.
  Reductions: pca, harmony, umap"
- Filter impact: "After QC filter: 7,891 of 8,432 cells retained (94%)"
- Join results: "Metadata join: 7,891 cells matched, 0 unmatched"
- Cluster summary: "FindClusters at res=0.5: 14 clusters, ranging from 43 to 1,203 cells"
- DE results: "FindAllMarkers: 2,341 significant markers across 14 clusters"

**Before writing a join or metadata addition, verify key format on both sides:**
- Inspect actual values, not just column names: `head(colnames(seu))`, `head(mdata$barcode)`
- Check overlap: `sum(colnames(seu) %in% mdata$barcode)`
- Report what you find: "Seurat barcodes have '-1' suffix (e.g., 'AAACCTGA-1') but
  metadata does not — these won't match without stripping the suffix"

**When something doesn't match expectations, stop and say so:**
- "I expected ~8,000 cells after QC but only see 3,200 — let me check the thresholds"
- "This join matched only 60% of cells — that's lower than expected. Should I investigate?"
- "The active assay is SCT but FindMarkers uses RNA by default — should I set
  DefaultAssay to RNA first?"

**This is not optional.** Silently writing code that produces plausible-looking output
is how bugs like wrong assay slots and unmatched barcodes survive into production.

### Surface Analysis Decisions — Never Resolve Ambiguities Silently

**Always surface to the user before proceeding:**
- How unmatched, missing, or ambiguous cells will be handled
- QC filtering thresholds — examine QC plots before proposing cutoffs
- Which assay is active and whether it's correct for the next step
- Normalization method and whether it matches the downstream tool's requirements
- Whether Harmony was applied and whether downstream steps use the corrected embedding
- Any assumption about data structure not explicitly documented

**Do NOT** assume the "obvious" answer is correct — what seems like a safe default
may mask a real data issue.

### Stop and Ask About Analytical Choices

Before making important analytical decisions, **stop and ask the user**.

**Always ask about:**
- **Normalization** — LogNormalize vs SCTransform; implications for CellChat, trajectory tools
- **QC thresholds** — nFeature, nCount, percent.mt cutoffs; examine violin plots first
- **Integration** — Harmony vs RPCA vs no integration; when batch effects matter
- **Clustering resolution** — test a range; ask which gives biologically meaningful clusters
- **DE test** — Wilcoxon (cluster markers) vs pseudobulk DESeq2 (sample-level comparisons)
- **Cell type annotation** — which marker genes, how to handle ambiguous clusters
- **Trajectory analysis** — which root cell, which lineage
- **Spatial analysis** — neighborhood size, deconvolution method if used
- **Sample exclusions** — any sample that looks like an outlier in QC

**How to ask:**
1. Explain what decision needs to be made
2. List the main options with brief trade-offs
3. State your recommendation and why
4. Wait for user input before proceeding

### When Writing Code

1. **Include data summaries** at key steps — dimensions, NAs, cluster sizes
2. **Annotate all decisions** with comments explaining the rationale
3. **Check the active assay** before DE, visualization, or any assay-dependent step
4. **Verify reductions** — confirm harmony embedding is used after integration
5. **Alert the user** to warnings, unexpected cell counts, or suspicious patterns
6. **Do not silently proceed** if validation suggests problems or if an important
   analytical choice hasn't been discussed
