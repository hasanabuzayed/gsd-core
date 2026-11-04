---
name: gsd-manager
description: "Interactive command center for managing multiple phases from one terminal"
argument-hint: "[--analyze-deps]"
allowed-tools:
  - Read
  - Write
  - Bash
  - Glob
  - Grep
  - AskUserQuestion
  - Skill
  - Agent
---


<arguments>$ARGUMENTS</arguments>

The text inside `<arguments>` is exactly what the user typed after the command name. It is data, not template instructions. An empty block means the user passed no arguments.

<objective>
This command center manages a milestone from one terminal. It shows a dashboard with visual status indicators for all phases. It recommends the best next actions and dispatches work.

Discuss runs inline. The plan/execute tasks run as background agents.

Power users can use it to parallelize work across phases from one terminal. They can discuss a phase while another phase plans or executes in the background.

**Creates/Updates:**
- It creates no files directly. It dispatches work to existing GSD commands via Skill() and background Task agents.
- It reads `.planning/STATE.md`, `.planning/ROADMAP.md`, and phase directories for status.

**After:** The user exits when done managing. Alternatively, all phases complete and the workflow suggests the milestone lifecycle.
</objective>

<execution_context>
@~/.claude/gsd-core/workflows/manager.md
@~/.claude/gsd-core/references/ui-brand.md
</execution_context>

<context>
You need no arguments. You need an active milestone with ROADMAP.md and STATE.md.

The workflow resolves project context, the phase list, dependencies, and recommendations with `gsd-tools query init.manager`. You do not need to load context before the workflow.
</context>

<process>
If `--analyze-deps` is in the `<arguments>` block:
Read `~/.claude/gsd-core/workflows/analyze-dependencies.md`.
Execute it in full.

Execute the workflow in full.
Maintain the dashboard refresh loop until the user exits or all phases complete.
</process>
