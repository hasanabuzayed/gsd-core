---
name: gsd:explore
description: Socratic ideation and idea routing — think through ideas before committing to plans
allowed-tools:
  - Read
  - Write
  - Bash
  - Grep
  - Glob
  - Agent
  - AskUserQuestion
---
<objective>
Open ended Socratic ideation session. Guide developer through idea exploration with probing questions. Optionally start research tasks. Direct results to GSD notes, todos, seeds, research questions, requirements, or new phases.

Accepts an optional topic argument: `/gsd:explore authentication strategy`
</objective>

<execution_context>
@~/.claude/gsd-core/workflows/explore.md
</execution_context>

<process>
Execute end to end.
</process>
