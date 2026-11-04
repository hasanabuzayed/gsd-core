<purpose>
This command audits implemented frontend code against 6 visual pillars. It works on any project, whether GSD manages it or not. It produces a scored UI-REVIEW.md with findings that users can act on.
</purpose>

<required_reading>
@~/.claude/gsd-core/references/ui-brand.md
</required_reading>

<available_agent_types>
Use the exact names of valid GSD subagent types. Do not use 'general-purpose' as a fallback.
- gsd-ui-auditor — Audits UI against design requirements
</available_agent_types>

<process>

## 0. Initialize

```bash
IFS= read -rd '' GSD_ARGS <<'GSD_ARGS_END' || true
$ARGUMENTS
GSD_ARGS_END
GSD_ARGS=${GSD_ARGS%$'\n'}
_GSD_SHIM_NAME="gsd-tools.cjs"; _GSD_RUNTIME_ROOT="${RUNTIME_DIR:-$(git rev-parse --show-toplevel 2>/dev/null || pwd)}"; GSD_TOOLS="${_GSD_RUNTIME_ROOT}/gsd-core/bin/${_GSD_SHIM_NAME}"; _gsd_at() { for _p; do if [ -f "$_p" ]; then GSD_TOOLS="$_p"; return 0; fi; done; return 1; }; _gsd_id_ok() { case "$("$1" runtime-identity --raw 2>/dev/null || true)" in '{"packageName":"@opengsd/gsd-core"'*'}') return 0;; *) return 1;; esac; }; _gsd_homes() { set -- "${CLAUDE_CONFIG_DIR:-$HOME/.claude}" "${ANTIGRAVITY_CONFIG_DIR:-$HOME/.gemini/antigravity}" "$HOME/.gemini/antigravity-ide" "$HOME/.gemini/antigravity-cli" "${AUGMENT_CONFIG_DIR:-$HOME/.augment}" "${CLINE_CONFIG_DIR:-$HOME/.cline}" "${CODEBUDDY_CONFIG_DIR:-$HOME/.codebuddy}" "${CODEX_HOME:-$HOME/.codex}" "${COPILOT_CONFIG_DIR:-${COPILOT_HOME:-$HOME/.copilot}}" "${CURSOR_CONFIG_DIR:-$HOME/.cursor}" "${HERMES_HOME:-$HOME/.hermes}" "${KILO_CONFIG_DIR:-${XDG_CONFIG_HOME:-$HOME/.config}/kilo}" "${KIMI_CONFIG_DIR:-$HOME/.config/agents}" "$HOME/.agents" "${KIMI_CODE_HOME:-$HOME/.kimi-code}" "${OPENCODE_CONFIG_DIR:-${XDG_CONFIG_HOME:-$HOME/.config}/opencode}" "${PI_CODING_AGENT_DIR:-$HOME/.pi/agent}" "${QWEN_CONFIG_DIR:-$HOME/.qwen}" "${TRAE_CONFIG_DIR:-$HOME/.trae}" "${WINDSURF_CONFIG_DIR:-$HOME/.codeium/windsurf}" "${ZCODE_CONFIG_DIR:-$HOME/.zcode}" "${GROK_AGENTS_HOME:-$HOME/.agents}"; for _h; do _gsd_at "$_h/gsd-core/bin/${_GSD_SHIM_NAME}" && return 0; done; return 1; }; if _gsd_at "${_GSD_RUNTIME_ROOT}/gsd-core/bin/${_GSD_SHIM_NAME}" "${_GSD_RUNTIME_ROOT}/.claude/gsd-core/bin/${_GSD_SHIM_NAME}" "${_GSD_RUNTIME_ROOT}/.codex/gsd-core/bin/${_GSD_SHIM_NAME}"; then gsd_run() { node "$GSD_TOOLS" "$@"; }; elif _gsd_homes; then gsd_run() { node "$GSD_TOOLS" "$@"; }; elif unset -f gsd_run; _G="$(command -v gsd_run)"; [ -n "$_G" ] && _gsd_id_ok "$_G"; then GSD_TOOLS="$_G"; gsd_run() { "$GSD_TOOLS" "$@"; }; else echo "ERROR: gsd-tools.cjs not found at $GSD_TOOLS and no identity-proving gsd_run is on PATH. Run: npx -y @opengsd/gsd-core@latest --claude --local" >&2; exit 1; fi; GSD_IDENTITY_STATUS=unverified; _gsd_id_ok gsd_run && GSD_IDENTITY_STATUS=ok; export GSD_IDENTITY_STATUS; [ "$GSD_IDENTITY_STATUS" = ok ] || echo "WARNING: \"$GSD_TOOLS\" did not prove it is @opengsd/gsd-core - it is either a different package or an @opengsd/gsd-core older than the runtime-identity verb. See docs/how-to/diagnose-a-foreign-gsd-tools.md" >&2; if [ -n "${CLAUDE_ENV_FILE:-}" ] && [ -n "${GSD_TOOLS:-}" ]; then printf "export PATH='%s':\"\$PATH\"\n" "${GSD_TOOLS%/*}" >> "$CLAUDE_ENV_FILE" 2>/dev/null || true; fi
RESPONSE_LANGUAGE=$(gsd_run query config-get response_language --raw --default "" 2>/dev/null || echo "")
PHASE_ARG=$(echo "$GSD_ARGS" | sed -E 's/--ws[[:space:]]+[A-Za-z0-9._-]+//g' | xargs)
GSD_WS=$(echo " $GSD_ARGS" | sed -nE 's/.* --ws +([A-Za-z0-9][A-Za-z0-9._-]*).*/--ws \1/p' | head -n 1)
INIT=$(gsd_run query init.phase-op ${GSD_WS:+--ws=${GSD_WS##* }} "${PHASE_ARG}")
if [[ "$INIT" == @file:* ]]; then INIT=$(cat "${INIT#@file:}"); fi
AGENT_SKILLS_UI_REVIEWER=$(gsd_run query agent-skills gsd-ui-auditor ${GSD_WS:+--ws=${GSD_WS##* }})
# workflow.ui_interaction_capture (default false): read here and handed to the auditor
# through its <config> block — the agent carries no gsd_run resolver of its own.
INTERACTION_CAPTURE=$(gsd_run query config-get workflow.ui_interaction_capture --raw 2>/dev/null || echo "false")
[ "$INTERACTION_CAPTURE" = "true" ] || INTERACTION_CAPTURE="false"
```

**If `response_language` is set:** You MUST use `{response_language}` for all user-facing output of this workflow. This output includes narration between tool calls, status updates, progress notes, findings, questions, prompts, and explanations.

Keep technical terms, code, file paths, and subagent prompts in English. Translate only user-facing output.

Parse: `phase_dir`, `phase_number`, `phase_name`, `phase_slug`, `padded_phase`, `commit_docs`.

```bash
UI_AUDITOR_MODEL=$(gsd_run query resolve-model gsd-ui-auditor --raw)
```

Show the banner:
```
### GSD ► UI AUDIT — PHASE {N}: {name}
```

## 1. Detect Input State

```bash
SUMMARY_FILES=$(ls "${PHASE_DIR}"/*-SUMMARY.md 2>/dev/null)
UI_SPEC_FILE=$(ls "${PHASE_DIR}"/*-UI-SPEC.md 2>/dev/null | head -1)
UI_REVIEW_FILE=$(ls "${PHASE_DIR}"/*-UI-REVIEW.md 2>/dev/null | head -1)
```

**If `SUMMARY_FILES` is empty:** Exit with "Phase {N} not executed. Run /gsd:execute-phase {N} first."

**Text mode (`workflow.text_mode: true` in config or `--text` flag):** If `$ARGUMENTS` contains `--text` OR `text_mode` from init JSON is `true`, set `TEXT_MODE=true`. If TEXT_MODE is active, replace every `AskUserQuestion` call with a plain-text numbered list. Ask the user to type their choice number.

Non-Claude runtimes, such as OpenAI Codex and Antigravity, need this because `AskUserQuestion` is not available.
**If `UI_REVIEW_FILE` non-empty:**

**If `--auto`:** Automatically select "View". Keep the existing UI-REVIEW.md unchanged and exit without a new audit. Log: `[auto] UI-REVIEW.md exists — reusing as-is.`

"Re-audit" is the choice that regenerates the review. An unattended run uses an existing artifact instead (#4776).

**Otherwise:** Use AskUserQuestion:
- header: "Existing UI Review"
- question: "UI-REVIEW.md already exists for Phase {N}."
- options:
  - "Re-audit — run fresh audit"
  - "View — display current review and exit"

If "View": Show the file and exit.
If "Re-audit": Continue.

## 2. Gather Context Paths

Build a file list for the auditor:
- All SUMMARY.md files in the phase directory
- All PLAN.md files in the phase directory
- UI-SPEC.md, if it exists (audit baseline)
- CONTEXT.md, if it exists (locked decisions)

## 3. Spawn gsd-ui-auditor

```
◆ Spawning UI auditor... (runs in a subagent — no output until it returns, ~1–5 min; expected, not a freeze)
```

Build the prompt:

```markdown
Read ~/.claude/agents/gsd-ui-auditor.md for instructions.

<objective>
Conduct 6-pillar visual audit of Phase {phase_number}: {phase_name}
{If UI-SPEC exists: "Audit against UI-SPEC.md design contract."}
{If no UI-SPEC: "Audit against abstract 6-pillar standards."}
</objective>

<required_reading>
- {summary_paths} (Execution summaries)
- {plan_paths} (Execution plans — what was intended)
- {ui_spec_path} (UI Design Contract — audit baseline, if exists)
- {context_path} (User decisions, if exists)
</required_reading>

${AGENT_SKILLS_UI_REVIEWER}

<config>
phase_dir: {phase_dir}
padded_phase: {padded_phase}
interaction_capture: {interaction_capture}
</config>
```

Omit null file paths.

`interaction_capture` is always present. Its value is `$INTERACTION_CAPTURE` from step 0. It is `true` only when `workflow.ui_interaction_capture` is on.

The auditor uses this value to select a branch in `<screenshot_approach>`. If the value is `false` or no Chrome binary resolves, the auditor uses its Playwright-only static capture.

<!-- #2508 runtime-aware-dispatch -->

> **Runtime-aware dispatch (#2508 Phase 4).** GSD workflows dispatch specialized subagents by role. Before dispatch on a built-in-only runtime (kimi-code — three built-ins only), resolve the role with `gsd_run query resolve-dispatch-type --requested <role> --raw`.

> The command returns the role unchanged on named-dispatch runtimes (Claude/OpenCode/…). On kimi-code, it maps the role to `coder`/`explore`/`plan` by role-suffix. In both cases, `${AGENT_SKILLS_<ROLE>}` carries the persona (Phase 3).

> See @gsd-core/references/runtime-aware-dispatch.md.

<!-- #2517 model-omit-on-inherit -->

> **Model omission (#2517).** If the value for `model` (`UI_AUDITOR_MODEL`) is `"inherit"` or empty, omit the parameter entirely. An empty value causes a 404 on runtimes without native tier aliases. This is the default on non-Claude runtimes.

> If you omit the parameter, the agent inherits the orchestrator's model. See @gsd-core/references/model-profile-resolution.md.

```
Agent(
  prompt=ui_audit_prompt,
  subagent_type="gsd-ui-auditor",
  model="{UI_AUDITOR_MODEL}",
  description="UI Audit Phase {N}"
)
```

> **ORCHESTRATOR RULE — CODEX RUNTIME**: After you call Agent() above, stop work on this task immediately. While the subagent is active, do not read more files, edit code, or run tests related to this task. Wait for the subagent to return its result.

> This prevents duplicate work, conflicting edits, and wasted context. Resume only when the subagent result is available.

## 4. Handle Return

**If `## UI REVIEW COMPLETE`:**

Show the score summary:

```
### GSD ► UI AUDIT COMPLETE ✓

**Phase {N}: {Name}** — Overall: {score}/24

| Pillar | Score |
|--------|-------|
| Copywriting | {N}/4 |
| Visuals | {N}/4 |
| Color | {N}/4 |
| Typography | {N}/4 |
| Spacing | {N}/4 |
| Experience Design | {N}/4 |

Top fixes:
1. {fix}
2. {fix}
3. {fix}

Full review: {path to UI-REVIEW.md}

---

## ▶ Next

`/clear` then:

- `/gsd:verify-work {N}` — UAT testing before phase completion

---
```

## Automated UI Verification (when Playwright-MCP is available)

If `mcp__playwright__*` tools are accessible in this session:

1. Use `mcp__playwright__navigate` or an equivalent Playwright-MCP tool to navigate to each UI component in the phase's UI-SPEC.md.
2. Use `mcp__playwright__screenshot` to take a screenshot of each component.
3. Compare the components against the spec's visual requirements: dimensions, color palette, layout, spacing scale, and typography.
4. Automatically report dimension, color, or layout discrepancies as more findings in the applicable pillar section of UI-REVIEW.md.
5. In the findings, flag items that need human judgment (brand feel, content tone) as `needs_human_review: true`.
   Show these items to the user separately after the automated pass completes.

If Playwright-MCP is not available in this session, skip this section entirely. The audit uses the standard code-only review described above instead. No configuration change is needed.

The workflow detects the availability of `mcp__playwright__*` tools at runtime.

## 5. Commit (if configured)

```bash
gsd_run query commit "docs(${padded_phase}): UI audit review" --files "${PHASE_DIR}/${PADDED_PHASE}-UI-REVIEW.md"
```

</process>

<success_criteria>
- [ ] Phase validated
- [ ] SUMMARY.md files found (execution complete)
- [ ] Existing review handled (re-audit/view)
- [ ] gsd-ui-auditor spawned with correct context
- [ ] UI-REVIEW.md created in phase directory
- [ ] Score summary shown to user
- [ ] Next steps shown
</success_criteria>
