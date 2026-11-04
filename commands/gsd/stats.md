---
name: gsd:stats
description: Show project statistics: phases, plans, requirements, git data, timeline
effort: low
allowed-tools:
  - Read
  - Bash
  - Grep
requires: [phase, progress]
---
<objective>
Show full project stats: phase progress, plan metrics, requirement status, git history, and timeline.
</objective>

<execution_context>
@~/.claude/gsd-core/workflows/stats.md
</execution_context>

<process>
Run complete process.
</process>
