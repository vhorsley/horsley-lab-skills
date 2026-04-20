# Horsley Lab Claude Skills

Custom [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skills for the Horsley Lab at Yale. These skills teach Claude our lab's conventions for data analysis, figure preparation, and scientific writing.

## What are skills?

Claude Code skills are markdown files that give Claude specific instructions for particular tasks — how to organize an analysis project, how to audit a script, how to structure a figure. Each skill has a short **description** that tells Claude when to activate it, and a **body** with the actual instructions.

There are two kinds:

- **Automatic skills** load on their own when Claude detects a relevant task. For example, when you ask Claude to set up a new analysis project, `horsley-lab-conventions` loads automatically.
- **User-invoked skills** (slash commands) are triggered by typing a command like `/audit-script`. These run specific workflows on demand.

## Prerequisites

You need [Claude Code](https://docs.anthropic.com/en/docs/claude-code) installed before adding lab skills. See [Anthropic's install guide](https://docs.anthropic.com/en/docs/claude-code/overview) to get started.

## Install

**In Positron / VS Code:**

1. Type `/plugins` in the Claude Code chat panel to open the plugin manager
2. Go to the **Marketplaces** tab
3. Add `vhorsley/horsley-lab-skills`
4. Switch to the **Plugins** tab and install `horsley-lab-skills`

**In the terminal CLI:**

```
/plugin marketplace add vhorsley/horsley-lab-skills
/plugin install horsley-lab-skills
```

Skills are then available as `/horsley-lab-skills:skill-name` (e.g., `/horsley-lab-skills:audit-script`).

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
| `/audit-script` | Systematic audit of data analysis scripts for bugs, analytical reasoning, data handling, style, and reproducibility |

### Data analysis conventions

| Skill | Description |
|-------|-------------|
| `horsley-lab-conventions` | Standard project and script organization for R-based data analysis |
| `cleanup-scripts` | End-of-session script consolidation — moves scratch files into numbered `.Rmd` pipeline |

### Figures and writing

| Skill | Description |
|-------|-------------|
| `horsley-figures` | Drafts and organizes figures |

## Improving skills

If a skill does something wrong, doesn't handle a situation you ran into, or you have an idea for a new one — open a GitHub issue or email [valerie.horsley@yale.edu](mailto:valerie.horsley@yale.edu).

When reporting, include which skill was involved, what happened, and what you expected.
