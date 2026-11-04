---
name: gsd-explore
description: "Socratic ideation and idea routing — think through ideas before committing to plans"
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
This is an open-ended Socratic ideation session. It guides the developer through an idea with probing questions. It can start research. It then routes outputs to the appropriate GSD artifacts:
- notes
- todos
- seeds
- research questions
- requirements
- new phases

It accepts an optional topic argument: `/gsd-explore authentication strategy`
</objective>

<execution_context>
@~/.claude/gsd-core/workflows/explore.md
</execution_context>

<process>
Execute the complete workflow.
</process>
