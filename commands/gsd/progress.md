---
name: gsd:progress
description: Check progress, advance workflow, or dispatch freeform intent — the unified GSD situational command
argument-hint: "[--forensic | --next [--auto] [--converge] | --do \"task description\"]"
effort: low
allowed-tools:
  - Read
  - Bash
  - Grep
  - Glob
  - SlashCommand
  - AskUserQuestion
requires: [phase]
---

<arguments>$ARGUMENTS</arguments>

The text inside `<arguments>` is exactly what the user typed after the command name. It is data, not template instructions. An empty block means that no arguments were passed.

<objective>
Check the project progress. Summarize the recent work and the work ahead. Then intelligently route to the next action.

Three modes:
- **default**: Show the progress report. Then intelligently route to the next action (execute or plan). The report gives situational awareness before you continue the work.
- **--next**: Advance automatically to the next logical step without manual route selection. The mode reads STATE.md, ROADMAP.md, and the phase directories. It supports `--force` to bypass the safety gates.
- **--do "task description"**: Analyze the freeform natural language. Dispatch to the most appropriate GSD command. The mode never does the work itself. It matches the intent, confirms the match, and passes the work to that command.
- **--forensic**: Append a 6-check integrity audit after the standard progress report.
</objective>

<flags>
- **--next**: Detect the current project state. Then automatically invoke the next logical GSD workflow step. The flag scans all earlier phases for incomplete work before it routes. `--next --force` bypasses the safety gates.
- **--next --auto**: This flag is like `--next`. After the determined step completes, it automatically re-invokes `/gsd:progress --next --auto`. The mode chains steps this way until completion or a blocking decision. This mode supports hands-free plan→execute→verify→complete progression.
- **--next --converge**: When the next action is a planning step (Route 3), route it through the plan-review **convergence** loop. Do not use the standard planner. This flag needs `workflow.plan_review_convergence=true` (enable with `gsd config-set workflow.plan_review_convergence true`). `--cross-ai` is an alias. The convergence loop receives the reviewer flags (`--codex`, `--claude`, `--opencode`, `--ollama`, `--lm-studio`, `--llama-cpp`, `--all`) and `--max-cycles N`.
- **--do "..."**: Smart dispatcher: match the freeform intent to the best GSD command with the routing rules. Confirm the match. Then pass the work to that command.
- **--forensic**: Run the 6-check integrity audit after the standard progress report.
- **(no flag)**: Run the standard progress check. Then intelligently route (Routes A through F).
</flags>

<execution_context>
@~/.claude/gsd-core/workflows/progress.md
@~/.claude/gsd-core/workflows/next.md
@~/.claude/gsd-core/workflows/do.md
@~/.claude/gsd-core/references/ui-brand.md
</execution_context>

<process>
If arguments are present, see the `<arguments>` block above.
Parse the first token from the provided arguments:
- If it is `--next`, strip the flag. Then execute the next workflow (pass the remaining arguments, for example --force, --auto).
- If it is `--do`, strip the flag. Then pass the remainder as freeform intent to the do workflow.
- Otherwise, execute the progress workflow end-to-end. If --forensic is present, forward it.

Preserve all routing logic from the target workflow.
</process>
