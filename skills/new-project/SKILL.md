---
name: new-project
description: >
  Use when creating a new project, setting up a project from scratch, or when the user
  invokes /new-project. Scaffolds directory structure, renv, git, and Claude Code
  configuration. Supports data science and general project types. R-primary lab.
user-invocable: true
---

# Create a New Project

When the user invokes `/new-project`, scaffold a complete project with version control and
Claude Code configuration. The project type determines which conventions, directories, and
environments to set up.

Run this skill from **inside the target project directory** (which may be empty or newly created).

---

## 1. Gather Project Information

### Question 1: Project type
- **Data science** (default) — Analysis project with numbered `.Rmd` scripts, `data/` +
  `outputs/` directories, renv, Quarto analysis documents
- **General** — Any other project (tools, packages, lab resources, scripts without full
  data science conventions)

### Question 2: Project basics (all types)
- **Project name**: Default to current directory name
- **Brief description**: 1–2 sentences for CLAUDE.md and README

### For Data Science projects, also ask:

#### Question 3: Python needed?
- **No** (default) — R only
- **Yes** — R + Python (ask which packages are needed)

#### Question 4: Layout
- **Flat** (default) — Single `scripts/` directory, for projects with a single analytical thread
- **Sectioned** — Subdirectories under `scripts/`, `data/`, `outputs/` for larger projects
  with multiple analytical threads (e.g., `scrna`, `spatial`, `proteomics`)

If sectioned, ask for section names.

#### Question 5: YCRC Bouchet cluster?
- **Yes** — This project will also run on Yale's Bouchet HPC cluster
- **No** (default) — Local analysis only

If yes:
- Adds `batch/` directory (tracked in git — SLURM batch scripts)
- Adds `logs/` directory (gitignored — SLURM output, ephemeral)
- Adds a "Dual Environment: Local + Cluster" section to CLAUDE.md
- Notes cluster path conventions (see below)
- Batch scripts use `BASEDIR=$(git rev-parse --show-toplevel)` — no hardcoded paths

### For all types:

#### Question: GitHub
- **Personal account** (`vhorsley`) — default for most lab projects
- **Other org** — if the project belongs to a collaboration or external org

#### Question: Private or public?
- **Private** (default, recommended for unpublished data)
- **Public**

#### Question: Changelog?
- **Yes** (recommended) — creates `CHANGELOG.md` to track changes over time
- **No** — skip; can always add later

---

## 2. Create Directory Structure

### Data Science — Flat layout

```bash
mkdir -p data/raw data/processed scripts/scratch outputs/figures outputs/tables docs .claude
touch data/raw/.gitkeep data/processed/.gitkeep scripts/scratch/.gitkeep \
      outputs/figures/.gitkeep outputs/tables/.gitkeep docs/.gitkeep
```

### Data Science — Sectioned layout

For each section (e.g., `scrna`, `spatial`):

```bash
mkdir -p data/raw data/processed outputs/figures outputs/tables docs .claude
mkdir -p scripts/{section} scripts/scratch
mkdir -p data/{section} outputs/{section}
touch data/raw/.gitkeep data/processed/.gitkeep scripts/scratch/.gitkeep docs/.gitkeep
```

### Cluster directories (if Bouchet = yes)

```bash
mkdir -p batch logs
```

`batch/` is tracked in git (SLURM scripts are version-controlled).
`logs/` is gitignored (SLURM output is ephemeral).

### General layout

```bash
mkdir -p .claude
```

Do NOT create `data/`, `outputs/`, or `scripts/` — let the user organize as appropriate.

---

## 3. R + renv (Data Science — always; General — if using R)

### Initialize renv

```r
renv::init()
install.packages(c("tidyverse", "here"))
renv::snapshot()
```

### Lab policy

- Every R data science project uses renv
- Snapshot after every package install: `renv::snapshot()`
- Commit `renv.lock`, `renv/activate.R`, `.Rprofile` to git
- Do NOT commit `renv/library/` or `renv/staging/`

---

## 4. Python environment (only if Python = yes)

```bash
conda create -n {project_name} python=3.11 numpy pandas matplotlib ipykernel -y
conda activate {project_name}
conda env export --from-history > environment.yml
```

Ask the user which additional packages to install.

### Lab policy

- Never install into the `base` environment
- One conda environment per project, named to match the project
- Always include `ipykernel` for Quarto compatibility
- Use `--from-history` for portable `environment.yml`

---

## 5. Write .gitignore

### Data Science .gitignore

```
# Raw data — read-only, often large; document source in README instead
data/raw/

# Generated outputs (reproducible from scripts)
outputs/

# R artifacts
.Rhistory
.RData
.Rproj.user/
renv/library/
renv/staging/
renv/local/
*_cache/

# Python artifacts
__pycache__/
*.py[cod]
.venv/
venv/

# Quarto rendering
*_files/
.quarto/
*.html

# Claude Code
.claude/worktrees/

# OS files
.DS_Store
Thumbs.db

# IDE settings
.vscode/
.positron/
*.Rproj

# Secrets
.env
*.pem
credentials.json
```

If cluster project, also add:

```
# SLURM logs (ephemeral)
logs/
```

**Note on `data/raw/`**: Default is to gitignore raw data (often large files). For small
datasets (<10 MB total), you can remove this line and commit the data directly. Ask the user.

### General .gitignore

```
# R artifacts
.Rhistory
.RData
.Rproj.user/

# Python artifacts
__pycache__/
*.py[cod]
.venv/
venv/

# Quarto rendering
*_files/
.quarto/

# Claude Code
.claude/worktrees/

# OS files
.DS_Store
Thumbs.db

# IDE settings
.vscode/
.positron/

# Secrets
.env
*.pem
credentials.json
```

---

## 6. Generate .claude/CLAUDE.md

Fill in project-specific details throughout. Remove any sections not relevant to this project.

### Data Science template

```markdown
# {Project Name}

{Brief description}

**Project type:** Data science
**PI:** Valerie Horsley
**Owner:** {user name}
**Started:** {date}

## Environment

### R
- Version: {R version}
- Package management: renv (see `renv.lock`)
- Restore with: `renv::restore()`

### Python (if applicable)
- Conda env: `{project_name}`
- Activate with: `conda activate {project_name}`
- Restore with: `conda env create -f environment.yml`

## Project Structure

```
{project_name}/
├── data/
│   ├── raw/          ← original files, never modified
│   └── processed/    ← cleaned/transformed data
├── scripts/
│   ├── 01_qc.Rmd          ← numbered in order of execution
│   ├── 02_preprocessing.Rmd
│   └── scratch/           ← exploratory files (not numbered)
├── outputs/
│   ├── figures/
│   └── tables/
├── docs/              ← notes, protocols, manuscript notes
└── README.md
```

## Conventions

- Numbered scripts (`01_`, `02_`, etc.) are the canonical pipeline — run in order
- Scratch files in `scripts/scratch/` are exploratory — consolidate before finishing
- Raw data in `data/raw/` is read-only — scripts never write there
- All outputs go in `outputs/` with descriptive names
- Use `here::here()` for all file paths — no hardcoded absolute paths
- Set `set.seed()` before any stochastic operation (clustering, UMAP)
- Run `renv::snapshot()` after installing new packages

<!-- IF CLUSTER: include this section if Bouchet = yes -->
## Dual Environment: Local + Bouchet

This project runs on both local machines and Yale's Bouchet HPC cluster.

**Cluster access:**
- SSH: `ssh {netid}@bouchet.ycrc.yale.edu`
- Web portal: https://ood-bouchet.ycrc.yale.edu
- Find your storage paths: run `mydirectories` after logging in

**Cluster paths (Roberts filesystem):**
- Project storage: `/nfs/roberts/project/{pi_group}/{netid}/{project_name}/`
- Scratch (60-day purge): accessible via `~/scratch_pi_{pi_netid}`

**Conventions:**
- Batch scripts in `batch/` use `BASEDIR=$(git rev-parse --show-toplevel)` — no hardcoded paths
- Large data files and outputs live on the cluster; only code and metadata are committed to git
- SLURM logs go in `logs/` (gitignored — ephemeral)
- Scratch storage is purged after 60 days — do not use for long-term storage
<!-- END IF CLUSTER -->

## Key Files

| File | Purpose |
|------|---------|
| `data/raw/` | Input data — never modified |
| `scripts/01_*.Rmd` | Start here |
| `outputs/figures/` | All figures |
| `renv.lock` | R package versions |
```

### General project template

```markdown
# {Project Name}

{Brief description}

**Project type:** General
**PI:** Valerie Horsley
**Owner:** {user name}
**Started:** {date}

## Conventions

- Use `here::here()` for file paths where applicable
- Document dependencies in README

## Notes

(Add project-specific context here as it develops)
```

### Project reminders file

Create `.claude/project-reminders.txt`:

```
CRITICAL REMINDERS (re-injected after context compaction):
1. (Add project-specific rules here as the project develops)
2. Check planning documents before modifying scripts
3. Never silently default unmatched data or metadata
```

Tell the user: *"I created `.claude/project-reminders.txt` — edit this as you discover rules
that Claude keeps forgetting after long sessions."*

---

## 7. Generate README.md

### Data Science README

````markdown
# {Project Name}

{Brief description}

**PI:** Valerie Horsley | **Contact:** valerie.horsley@yale.edu

## Setup

### R
```r
# renv auto-activates via .Rprofile
renv::restore()
```

### Python (if applicable)
```bash
conda env create -f environment.yml
conda activate {project_name}
```

## Data

(Document data sources and how to obtain them)

## Running the Analysis

Run numbered scripts in order:
1. `scripts/01_qc.Rmd` — {brief description}
2. `scripts/02_preprocessing.Rmd` — {brief description}
...
````

### General README

````markdown
# {Project Name}

{Brief description}

**PI:** Valerie Horsley | **Contact:** valerie.horsley@yale.edu

## Setup

{Document prerequisites and setup steps as the project develops}

## Usage

{Document how to use the project}
````

---

## 8. Create CHANGELOG.md (if requested)

```markdown
# Changelog

## {today's date}

### Added
- Initial project setup
```

---

## 9. Git + GitHub

```bash
git init
git add .
git commit -m "Initial project setup"
```

Create the remote:

```bash
# Personal account, private (default)
gh repo create vhorsley/{project_name} --private --source=. --push

# Personal account, public
gh repo create vhorsley/{project_name} --public --source=. --push
```

---

## 10. Summary

### Data Science summary

```
Project "{project_name}" created successfully!

  Type: Data science ({flat/sectioned})
  Languages: {R only / R + Python}
  renv: initialized
  Cluster: {Bouchet / local only}
  GitHub: {url}

Next steps:
  1. Add data to data/raw/ (or document source in README)
  2. Create your first script: scripts/01_import.Rmd
  3. Edit .claude/project-reminders.txt as the project develops
```

### General summary

```
Project "{project_name}" created successfully!

  Type: General
  GitHub: {url}

Next steps:
  1. Start adding code and documentation
  2. Update .claude/CLAUDE.md as the project develops
```
