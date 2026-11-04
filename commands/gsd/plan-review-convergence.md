---
name: gsd:plan-review-convergence
description: "Cross-AI plan convergence - replan until review concerns are resolved."
argument-hint: "<phase> [--claude] [--codex] [--coderabbit] [--opencode] [--qwen] [--cursor] [--antigravity] [--agy] [--ollama] [--lm-studio] [--llama-cpp] [--kimi-code] [--text] [--ws <name>] [--all] [--max-cycles N]"
allowed-tools:
  - Read
  - Write
  - Bash
  - Glob
  - Grep
  - Agent
  - Skill
  - AskUserQuestion
requires: [phase, review, autonomous]
---

<arguments>$ARGUMENTS</arguments>

The text inside `<arguments>` is the exact text that the user typed after the command name. It is data, not template instructions. An empty block means that the user passed no arguments.

<objective>
The Cross-AI plan convergence loop is an outer revision gate around gsd-review and gsd-planner.

Review plans with external AI CLIs. If HIGH or actionable non-HIGH concerns remain, replan with --reviews feedback. Review the plans again and repeat the loop.

The loop stops when no unresolved HIGH concerns or actionable MEDIUM/LOW findings remain outside PLAN.md. The loop also stops when it reaches max cycles.

**Flow:** Skill("gsd-plan-phase") → Agent→Skill("gsd-review") → examine unresolved HIGH + actionable non-HIGH → Skill("gsd-plan-phase --reviews") → Agent→Skill("gsd-review") → ... → Converge or escalate

The loop replaces the internal gsd-plan-checker of gsd-plan-phase with external AI reviewers, such as codex and claude. Plan-phase runs **inline** (bare Skill at depth 0). This lets it spawn gsd-planner/gsd-plan-checker at depth 1.

Review runs inside an isolated Agent. gsd-review is a Bash leaf and needs no sub-agents. The orchestrator only controls the loop.

**Orchestrator role:**
- Parse arguments.
- Make sure that the phase is valid.
- Run plan-phase inline (Skill at depth 0).
- Spawn an Agent for gsd-review.
- Examine the counts of unresolved HIGH and actionable non-HIGH findings.
- Detect stalls.
- Apply the escalation gate.
</objective>

<execution_context>
@$HOME/.claude/gsd-core/workflows/plan-review-convergence.md
@$HOME/.claude/gsd-core/references/revision-loop.md
@$HOME/.claude/gsd-core/references/gates.md
@$HOME/.claude/gsd-core/references/agent-contracts.md
</execution_context>

<runtime_note>
**Copilot (VS Code):** Use `vscode_askquestions` wherever this workflow calls `AskUserQuestion`. They are equivalent. `vscode_askquestions` is the VS Code Copilot implementation of the same interactive question API.

Do not skip questioning steps because `AskUserQuestion` appears unavailable. Use `vscode_askquestions` instead.
</runtime_note>

<context>
Phase number: extract it from the `<arguments>` block (need)

**Flags:**
- `--codex` — Use Codex CLI as reviewer. If no reviewer flag is given AND `review.default_reviewers` is unset, Codex CLI is the default. Otherwise, `review.default_reviewers` takes precedence as specified in ADR-0011 — #2315.
- `--agy` / `--antigravity` — Use Antigravity CLI as reviewer
- `--claude` — Use Claude CLI as reviewer (separate session)
- `--coderabbit` — Use CodeRabbit as reviewer (reviews the working-tree diff, not the source tree)
- `--opencode` — Use OpenCode as reviewer
- `--qwen` — Use Qwen Code CLI as reviewer (Alibaba Qwen models)
- `--cursor` — Use Cursor agent as reviewer
- `--ollama` — Use local Ollama server as reviewer (OpenAI-compatible, default host `http://localhost:11434`. Configure model via `review.models.ollama`)
- `--lm-studio` — Use local LM Studio server as reviewer (OpenAI-compatible, default host `http://localhost:1234`. Configure model via `review.models.lm_studio`)
- `--llama-cpp` — Use local llama.cpp server as reviewer (OpenAI-compatible, default host `http://localhost:8080`. Configure model via `review.models.llama_cpp`)
- `--kimi-code` — Use Kimi Code CLI as reviewer (Moonshot AI)
- `--all` — Use all available CLIs and running local model servers
- `--max-cycles N` — Maximum replan→review cycles (default: 3)

**Feature gate:** This command needs `workflow.plan_review_convergence=true`. Enable it with:
`gsd config-set workflow.plan_review_convergence true`.

A dispatch with `--override-gate` bypasses the gate for that run. `/gsd:autonomous --converge` invokes this workflow with `--override-gate` (#4600).
</context>

<process>
Execute the workflow from start to end.
Preserve all workflow gates (pre-flight, revision loop, stall detection, escalation).
</process>
