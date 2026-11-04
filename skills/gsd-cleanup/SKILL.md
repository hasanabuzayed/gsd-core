---
name: gsd-cleanup
description: "Archive phase directories that belong to completed milestones"
allowed-tools:
  - Read
  - Write
  - Bash
  - Grep
  - AskUserQuestion
---

<objective>
Move phase directories from completed milestones into `.planning/milestones/v{X.Y}-phases/`.

Use this skill when `.planning/phases/` contains directories from past milestones.
</objective>

<execution_context>
@~/.claude/gsd-core/workflows/cleanup.md
</execution_context>

<process>
Run the complete process.
Identify completed milestones.
Show a dry‑run summary.
Archive directories after confirmation.
</process>
