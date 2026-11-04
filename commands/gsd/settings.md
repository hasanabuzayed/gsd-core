---
name: gsd:settings
description: Configure GSD workflow toggles and model profile
allowed-tools:
  - Read
  - Write
  - Bash
  - Grep
  - AskUserQuestion
requires: [quick]
---

<objective>
Use a multi-question prompt to configure GSD workflow agents and the model profile.

Route to the settings workflow. The workflow does these tasks:
- Make sure that the config exists.
- Read and parse the current settings.
- Use an interactive multi-question prompt for the model profile and workflow toggles.
  The toggles include research, plan_check, verifier, drift guard, TDD, code review, worktrees, compact content, and more.
  See `gsd-core/workflows/settings.md` for the current set.
- Merge and write the config.
- Show confirmation with quick command references.
</objective>

<execution_context>
@~/.claude/gsd-core/workflows/settings.md
</execution_context>

<process>
Execute the workflow from start to end.
</process>
