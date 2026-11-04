---
name: gsd-settings
description: "Configure GSD workflow toggles and model profile"
allowed-tools:
  - Read
  - Write
  - Bash
  - Grep
  - AskUserQuestion
---


<objective>
Configure GSD workflow agents and the model profile with an interactive prompt that asks multiple questions.

Route to the settings workflow. The workflow does these tasks:
- Makes sure that the config exists
- Reads and parses the current settings
- Uses an interactive prompt that asks multiple questions about the model profile and workflow toggles
- Merges and writes the config
- Shows confirmation with quick command references

The workflow toggles include research, plan_check, verifier, drift guard, TDD, code review, worktrees, compact content, and more.
See `gsd-core/workflows/settings.md` for the current set.
</objective>

<execution_context>
@~/.claude/gsd-core/workflows/settings.md
</execution_context>

<process>
Execute the workflow from start to end.
</process>
