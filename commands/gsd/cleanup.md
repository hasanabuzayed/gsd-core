---
name: gsd:cleanup
description: Archive accumulated phase directories from completed milestones
allowed-tools:
  - Read
  - Write
  - Bash
  - Grep
  - AskUserQuestion
requires: [phase]
---
<objective>
Archive phase directories from completed milestones into `.planning/milestones/v{X.Y}-phases/`.
If `.planning/phases/` contains directories from past milestones, use this command.
</objective>

<execution_context>
@~/.claude/gsd-core/workflows/cleanup.md
</execution_context>

<process>
Run the command end to end.
Identify completed milestones.
Show a dry-run summary.
Archive the phase directories on confirmation.
</process>
