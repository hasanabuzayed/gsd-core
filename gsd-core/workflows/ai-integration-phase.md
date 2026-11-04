<purpose>
Generate an AI design contract (AI-SPEC.md) for phases that build AI systems. The workflow coordinates gsd-framework-selector → gsd-ai-researcher → gsd-domain-researcher → gsd-eval-planner with a validation gate. The workflow runs between discuss-phase and plan-phase in the GSD lifecycle.

AI-SPEC.md fixes four things before the planner creates tasks:
1. Framework selection, with reasons and alternatives
2. Implementation guidance from official docs, with correct syntax, patterns, and pitfalls
3. Domain context, with practitioner rubric ingredients, failure modes, and regulatory constraints
4. Evaluation strategy, with dimensions, rubrics, tooling, reference dataset, and guardrails

This prevents the two most common AI development failures:
- Selection of the wrong framework for the use case
- Evaluation as an afterthought.
</purpose>

<required_reading>
@~/.claude/gsd-core/references/ai-frameworks.md
@~/.claude/gsd-core/references/ai-evals.md
</required_reading>

<process>

## 1. Initialize

```bash
IFS= read -rd '' GSD_ARGS <<'GSD_ARGS_END' || true
$ARGUMENTS
GSD_ARGS_END
GSD_ARGS=${GSD_ARGS%$'\n'}
_GSD_SHIM_NAME="gsd-tools.cjs"; _GSD_RUNTIME_ROOT="${RUNTIME_DIR:-$(git rev-parse --show-toplevel 2>/dev/null || pwd)}"; GSD_TOOLS="${_GSD_RUNTIME_ROOT}/gsd-core/bin/${_GSD_SHIM_NAME}"; _gsd_at() { for _p; do if [ -f "$_p" ]; then GSD_TOOLS="$_p"; return 0; fi; done; return 1; }; _gsd_id_ok() { case "$("$1" runtime-identity --raw 2>/dev/null || true)" in '{"packageName":"@opengsd/gsd-core"'*'}') return 0;; *) return 1;; esac; }; _gsd_homes() { set -- "${CLAUDE_CONFIG_DIR:-$HOME/.claude}" "${ANTIGRAVITY_CONFIG_DIR:-$HOME/.gemini/antigravity}" "$HOME/.gemini/antigravity-ide" "$HOME/.gemini/antigravity-cli" "${AUGMENT_CONFIG_DIR:-$HOME/.augment}" "${CLINE_CONFIG_DIR:-$HOME/.cline}" "${CODEBUDDY_CONFIG_DIR:-$HOME/.codebuddy}" "${CODEX_HOME:-$HOME/.codex}" "${COPILOT_CONFIG_DIR:-${COPILOT_HOME:-$HOME/.copilot}}" "${CURSOR_CONFIG_DIR:-$HOME/.cursor}" "${HERMES_HOME:-$HOME/.hermes}" "${KILO_CONFIG_DIR:-${XDG_CONFIG_HOME:-$HOME/.config}/kilo}" "${KIMI_CONFIG_DIR:-$HOME/.config/agents}" "$HOME/.agents" "${KIMI_CODE_HOME:-$HOME/.kimi-code}" "${OPENCODE_CONFIG_DIR:-${XDG_CONFIG_HOME:-$HOME/.config}/opencode}" "${PI_CODING_AGENT_DIR:-$HOME/.pi/agent}" "${QWEN_CONFIG_DIR:-$HOME/.qwen}" "${TRAE_CONFIG_DIR:-$HOME/.trae}" "${WINDSURF_CONFIG_DIR:-$HOME/.codeium/windsurf}" "${ZCODE_CONFIG_DIR:-$HOME/.zcode}" "${GROK_AGENTS_HOME:-$HOME/.agents}"; for _h; do _gsd_at "$_h/gsd-core/bin/${_GSD_SHIM_NAME}" && return 0; done; return 1; }; if _gsd_at "${_GSD_RUNTIME_ROOT}/gsd-core/bin/${_GSD_SHIM_NAME}" "${_GSD_RUNTIME_ROOT}/.claude/gsd-core/bin/${_GSD_SHIM_NAME}" "${_GSD_RUNTIME_ROOT}/.codex/gsd-core/bin/${_GSD_SHIM_NAME}"; then gsd_run() { node "$GSD_TOOLS" "$@"; }; elif _gsd_homes; then gsd_run() { node "$GSD_TOOLS" "$@"; }; elif unset -f gsd_run; _G="$(command -v gsd_run)"; [ -n "$_G" ] && _gsd_id_ok "$_G"; then GSD_TOOLS="$_G"; gsd_run() { "$GSD_TOOLS" "$@"; }; else echo "ERROR: gsd-tools.cjs not found at $GSD_TOOLS and no identity-proving gsd_run is on PATH. Run: npx -y @opengsd/gsd-core@latest --claude --local" >&2; exit 1; fi; GSD_IDENTITY_STATUS=unverified; _gsd_id_ok gsd_run && GSD_IDENTITY_STATUS=ok; export GSD_IDENTITY_STATUS; [ "$GSD_IDENTITY_STATUS" = ok ] || echo "WARNING: \"$GSD_TOOLS\" did not prove it is @opengsd/gsd-core - it is either a different package or an @opengsd/gsd-core older than the runtime-identity verb. See docs/how-to/diagnose-a-foreign-gsd-tools.md" >&2; if [ -n "${CLAUDE_ENV_FILE:-}" ] && [ -n "${GSD_TOOLS:-}" ]; then printf "export PATH='%s':\"\$PATH\"\n" "${GSD_TOOLS%/*}" >> "$CLAUDE_ENV_FILE" 2>/dev/null || true; fi
GSD_WS=$(echo " $GSD_ARGS" | sed -nE 's/.* --ws +([A-Za-z0-9][A-Za-z0-9._-]*).*/--ws \1/p' | head -n 1)
INIT=$(gsd_run query init.plan-phase ${GSD_WS:+--ws=${GSD_WS##* }} "$PHASE")
if [[ "$INIT" == @file:* ]]; then INIT=$(cat "${INIT#@file:}"); fi
```

Parse the JSON for: `phase_dir`, `phase_number`, `phase_name`, `phase_slug`, `padded_phase`, `has_context`, `has_research`, `commit_docs`, `response_language`.

**If `response_language` is set:** You MUST show all user-facing output of this workflow in `{response_language}`. This output includes narration between tool calls, status updates, progress notes, findings, questions, prompts, and explanations. Keep technical terms, code, file paths, and subagent prompts in English. Translate only user-facing output.

**File paths:** `state_path`, `roadmap_path`, `requirements_path`, `context_path`.

Resolve agent models:
```bash
SELECTOR_MODEL=$(gsd_run query resolve-model gsd-framework-selector --pick model 2>/dev/null || true)
RESEARCHER_MODEL=$(gsd_run query resolve-model gsd-ai-researcher --pick model 2>/dev/null || true)
DOMAIN_MODEL=$(gsd_run query resolve-model gsd-domain-researcher --pick model 2>/dev/null || true)
PLANNER_MODEL=$(gsd_run query resolve-model gsd-eval-planner --pick model 2>/dev/null || true)
```

Examine the config:
```bash
AI_PHASE_ENABLED=$(gsd_run query config-get workflow.ai_integration_phase --raw 2>/dev/null || echo "true")
```

**If `AI_PHASE_ENABLED` is `false`:**
```
AI phase is disabled in config. Enable via /gsd:settings.
```
Exit workflow.

**If `planning_exists` is false:** Show an error. Run `/gsd:new-project` first.

## 2. Parse and Validate Phase

Extract the phase number from $ARGUMENTS.

If the phase number is not provided, this orchestrator, not `gsd-tools.cjs`, detects the next unplanned phase:
- Run `gsd_run query roadmap.analyze`.
- Read its `next_phase` field.

This is the first phase whose `disk_status` is `no_directory`, `empty`, `discussed`, or `researched`. The phase is not yet planned.

The `query roadmap.get-phase` command below needs an explicit `${PHASE}`. It does not auto-detect the phase.

```bash
PHASE_INFO=$(gsd_run query roadmap.get-phase "${PHASE}")
```

**If `found` is false:** Show an error with the available phases.

## 3. Check Prerequisites

**If `has_context` is false:**
```
No CONTEXT.md found for Phase {N}.
Recommended: run /gsd:discuss-phase {N} first to capture framework preferences.
Continuing without user decisions — framework selector will ask all questions.
```
Continue (non-blocking).

## 4. Check Existing AI-SPEC

```bash
AI_SPEC_FILE=$(ls "${PHASE_DIR}"/*-AI-SPEC.md 2>/dev/null | head -1)
```

**Text mode (`workflow.text_mode: true` in config or `--text` flag):** If `--text` is present in `$ARGUMENTS` OR `text_mode` from init JSON is `true`, set `TEXT_MODE=true`. When TEXT_MODE is active, replace every `AskUserQuestion` call with a plain-text numbered list. Ask the user to type their choice number. Non-Claude runtimes, such as OpenAI Codex and Antigravity, need this because `AskUserQuestion` is not available.
**If exists:**

**If `--auto`:** Auto-select "Skip". Keep the existing AI-SPEC untouched. Exit. Log:
`[auto] AI-SPEC exists — reusing as-is.`

Skip is the auto choice because it is the only non-destructive choice. "Update" re-runs framework selection with the existing spec as baseline. This can rewrite decisions a person already recorded in the spec. Nobody is present to notice (#4776).

**Otherwise:** Use AskUserQuestion:
- header: "Existing AI-SPEC"
- question: "AI-SPEC.md already exists for Phase {N}. What would you like to do?"
- options:
  - "Update — re-run with existing as baseline"
  - "View — display current AI-SPEC and exit"
  - "Skip — keep current AI-SPEC and exit"

If "View": Show the file contents. Exit.
If "Skip": Exit.
If "Update": Continue to step 5.

## 5. Spawn gsd-framework-selector

Display:
```
### GSD ► AI DESIGN CONTRACT — PHASE {N}: {name}

◆ Step 1/4 — Framework Selection...
```

Spawn `gsd-framework-selector` with:
```markdown
Read ~/.claude/agents/gsd-framework-selector.md for instructions.

<objective>
Select the right AI framework for Phase {phase_number}: {phase_name}
Goal: {phase_goal}
</objective>

<required_reading>
{context_path if exists}
{requirements_path if exists}
</required_reading>

<phase_context>
Phase: {phase_number} — {phase_name}
Goal: {phase_goal}
</phase_context>
```

Parse selector output for: `primary_framework`, `system_type`, `model_provider`, `eval_concerns`, `alternative_framework`.

**If selector fails or returns empty:** Exit with error — "Framework selection failed. Re-run /gsd:ai-integration-phase {N} or answer the framework question in /gsd:discuss-phase {N} first."

## 6. Initialize AI-SPEC.md

Copy template:
```bash
cp "$HOME/.claude/gsd-core/templates/AI-SPEC.md" "${PHASE_DIR}/${PADDED_PHASE}-AI-SPEC.md"
```

Before Step 7, fill in header fields:
- Phase number and name
- System classification (from selector)
- Selected framework (from selector)
- Alternative considered (from selector)

## 7. Spawn gsd-ai-researcher

> **Order note (prevents a tool-level last-writer-wins race):** Steps 7 and 8 write separate sections of AI-SPEC.md. They MUST run sequentially. Wait for Step 7 to complete before you spawn Step 8. Both agents use only the `Edit` tool, never `Write`, to change AI-SPEC.md.

> A `Write` on a shared file replaces the entire file without notice. It overwrites the other agent's work. `Edit` targets only the applicable lines. See #3096 for a confirmed 40%-incidence race on parallel dispatch.

Display:
```
◆ Step 2/4 — Researching {primary_framework} docs + AI systems best practices...
```

Spawn `gsd-ai-researcher` with:
```markdown
Read ~/.claude/agents/gsd-ai-researcher.md for instructions.

**Tool discipline (mandatory):**
Use the Edit tool exclusively when modifying AI-SPEC.md — NEVER use Write on this file.
Write replaces the entire file and will overwrite work from parallel or sequential sibling agents.
Before editing, verify the section you are about to write is still a template placeholder.

<objective>
</objective>

<required_reading>
{ai_spec_path}
{context_path if exists}
</required_reading>

<input>
framework: {primary_framework}
system_type: {system_type}
model_provider: {model_provider}
ai_spec_path: {ai_spec_path}
phase_context: Phase {phase_number}: {phase_name} — {phase_goal}
</input>
```

## 8. Spawn gsd-domain-researcher

> **Wait for Step 7 to complete before you spawn this step** (see the order note in Step 7).

Display:
```
◆ Step 3/4 — Researching domain context and expert evaluation criteria...
```

Spawn `gsd-domain-researcher` with:
```markdown
Read ~/.claude/agents/gsd-domain-researcher.md for instructions.

**Tool discipline (mandatory):**
Use the Edit tool exclusively when modifying AI-SPEC.md — NEVER use Write on this file.
Write replaces the entire file and will overwrite work from parallel or sequential sibling agents.
Before editing, verify the section you are about to write is still a template placeholder.

<objective>
</objective>

<required_reading>
{ai_spec_path}
{context_path if exists}
{requirements_path if exists}
</required_reading>

<input>
system_type: {system_type}
phase_name: {phase_name}
phase_goal: {phase_goal}
ai_spec_path: {ai_spec_path}
</input>
```

## 9. Spawn gsd-eval-planner

Display:
```
◆ Step 4/4 — Designing evaluation strategy from domain + technical context...
```

Spawn `gsd-eval-planner` with:
```markdown
Read ~/.claude/agents/gsd-eval-planner.md for instructions.

<objective>
Design evaluation strategy for Phase {phase_number}: {phase_name}
Write Sections 5, 6, and 7 of AI-SPEC.md
AI-SPEC.md now contains domain context (Section 1b) — use it as your rubric starting point.
</objective>

<required_reading>
{ai_spec_path}
{context_path if exists}
{requirements_path if exists}
</required_reading>

<input>
system_type: {system_type}
framework: {primary_framework}
model_provider: {model_provider}
phase_name: {phase_name}
phase_goal: {phase_goal}
ai_spec_path: {ai_spec_path}
</input>
```

## 10. Validate AI-SPEC Completeness

Read the completed AI-SPEC.md. Make sure that:
- Section 2 has a framework name (not placeholder)
- Section 1b has at least one domain rubric ingredient (Good/Bad/Stakes)
- Section 3 has a non-empty code block (entry point pattern)
- Section 4b has a Pydantic example
- Section 5 has at least one row in the dimensions table
- Section 6 has at least one guardrail or explicit "N/A for internal tool" note
- Checklist section at end has 3+ items checked

**If validation fails:** Show the specific missing sections. Ask the user if they want to re-run the specific step or continue anyway.

## 11. Commit

```bash
gsd_run query commit "docs({phase_slug}): generate AI-SPEC.md — {primary_framework} + domain context + eval strategy" --files "${AI_SPEC_FILE}"
```

## 12. Display Completion

```
### GSD ► AI-SPEC COMPLETE — PHASE {N}: {name}

◆ Framework: {primary_framework}
◆ System Type: {system_type}
◆ Domain: {domain_vertical from Section 1b}
◆ Eval Dimensions: {eval_concerns}
◆ Tracing Default: Arize Phoenix (or detected existing tool)
◆ Output: {ai_spec_path}

Next step:
  /gsd:plan-phase {N}   — planner will consume AI-SPEC.md
```

</process>

<success_criteria>
- [ ] Select the framework with reasons (Section 2)
- [ ] Create AI-SPEC.md from the template
- [ ] Research framework docs + AI best practices (Sections 3, 4, 4b populated)
- [ ] Research domain context + expert rubric ingredients (Section 1b populated)
- [ ] Base the eval strategy on domain context (Sections 5-7 populated)
- [ ] Set Arize Phoenix (or detected tool) as the tracing default in Section 7
- [ ] Validate AI-SPEC.md (Sections 1b, 2, 3, 4b, 5, 6 all non-empty)
- [ ] If commit_docs is enabled, commit
- [ ] Show the next step to the user
</success_criteria>
