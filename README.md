# Horsley Lab Claude Skills

Custom [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skills for the Horsley Lab at Yale. These skills teach Claude our lab's conventions for data analysis, figure preparation, and scientific writing.

## What are skills?

Claude Code skills are markdown files that give Claude specific instructions for particular tasks — how to organize an analysis project, how to audit a script, how to structure a figure. Each skill has a short **description** that tells Claude when to activate it, and a **body** with the actual instructions.

There are two kinds:

- **Automatic skills** load on their own when Claude detects a relevant task. For example, when you ask Claude to set up a new analysis project, `horsley-lab-conventions` loads automatically.
- **User-invoked skills** (slash commands) are triggered by typing a command like `/audit-script`. These run specific workflows on demand.

## Prerequisites

You need [Claude Code](https://docs.anthropic.com/en/docs/claude-code/overview) installed before adding lab skills.

## Install

⚠️ The install commands must be run in the **Claude Code terminal**, not the sidebar. Open your terminal, type `claude` to launch Claude Code, then run:

```
/plugin marketplace add https://github.com/vhorsley/horsley-lab-skills
/plugin install horsley-lab-skills
```

## Updates

Skills update automatically when you restart Claude Code. To update manually:

```
/plugin uninstall horsley-lab-skills
/plugin install horsley-lab-skills
```

## Skill reference

### Workflows (slash commands)

| Skill | Description |
|-------|-------------|
| `/audit-script` | Systematic audit of data analysis scripts for bugs, analytical reasoning, data handling, style, and reproducibility. Run this before sending a script for review. |
| `/new-project` | Scaffold a new analysis project with the right folder structure, renv, git, and CLAUDE.md in one step |
| `/new-plan` | Create a planning document for multi-session analyses — tracks status, scripts, inputs/outputs, and key decisions |
| `/done` | End-of-session wrap-up: summarizes work, updates planning docs, consolidates scratch files, and commits |
| `/learn-code` | Interactive walkthrough of any analysis script, section by section. Great for understanding code or learning a new analysis approach |

### Auto-loading skills

| Skill | Description |
|-------|-------------|
| `horsley-lab-conventions` | Standard project and script organization for R-based data analysis — loads when setting up a new project |
| `data-handling` | Best practices for Seurat objects, joins, filtering, and annotating analytical decisions — loads when writing analysis code |
| `cleanup-scripts` | End-of-session script consolidation — moves scratch files into the numbered `.Rmd` pipeline |

### Figures and writing

| Skill | Description |
|-------|-------------|
| `horsley-figures` | Drafts and organizes figures |

## Improving skills

If a skill does something wrong, doesn't handle a situation you ran into, or you have an idea for a new one — open a GitHub issue or email [valerie.horsley@yale.edu](mailto:valerie.horsley@yale.edu).

When reporting, include which skill was involved, what happened, and what you expected.
