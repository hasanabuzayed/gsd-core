<purpose>
Audit gaps in Nyquist validation for a completed phase. Generate missing tests. Update VALIDATION.md.
</purpose>

<required_reading>
@~/.claude/gsd-core/references/ui-brand.md
</required_reading>

<available_agent_types>
Valid GSD subagent types (use exact names — do not use 'general-purpose' as a fallback):
- gsd-nyquist-auditor — Examines verification coverage
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
AGENT_SKILLS_AUDITOR=$(gsd_run query agent-skills gsd-nyquist-auditor ${GSD_WS:+--ws=${GSD_WS##* }})
```

**If `response_language` is set:** You MUST show all user-facing output of this workflow in `{response_language}`. This includes narration between tool calls, status updates, progress notes, findings, questions, prompts, and explanations. Keep technical terms, code, file paths, and subagent prompts in English. Translate only user-facing output.

Parse: `phase_dir`, `phase_number`, `phase_name`, `phase_slug`, `padded_phase`.

```bash
AUDITOR_MODEL=$(gsd_run query resolve-model gsd-nyquist-auditor --raw)
VERIFY_POST_HOOKS_JSON=$(gsd_run loop render-hooks verify:post --raw)
```

Resolve active step hooks from `VERIFY_POST_HOOKS_JSON` where `kind == "step"` and `ref.skill == "validate-phase"`.

If no active validate-phase step hook exists, exit with "Nyquist validation is disabled. Enable via /gsd:settings."

Show the banner: `GSD > VALIDATE PHASE {N}: {name}`

## 1. Detect Input State

```bash
VALIDATION_FILE=$(ls "${PHASE_DIR}"/*-VALIDATION.md 2>/dev/null | head -1)
SUMMARY_FILES=$(ls "${PHASE_DIR}"/*-SUMMARY.md 2>/dev/null)
```

- **State A** (`VALIDATION_FILE` non-empty): Audit the existing file.
- **State B** (`VALIDATION_FILE` empty, `SUMMARY_FILES` non-empty): Reconstruct from artifacts.
- **State C** (`SUMMARY_FILES` empty): Exit with "Phase {N} not executed. Run /gsd:execute-phase {N} ${GSD_WS} first."

## 2. Discovery

### 2a. Read Phase Artifacts

Read all PLAN and SUMMARY files. Extract task lists, requirement IDs, key-files changed, and verify blocks.

### 2b. Build Requirement-to-Task Map

Per task: `{ task_id, plan_id, wave, requirement_ids, has_automated_command }`

### 2c. Detect Test Infrastructure

State A: Parse the existing VALIDATION.md Test Infrastructure table.
State B: Scan the filesystem:

```bash
find . -name "pytest.ini" -o -name "jest.config.*" -o -name "vitest.config.*" -o -name "pyproject.toml" 2>/dev/null | head -10
find . \( -name "*.test.*" -o -name "*.spec.*" -o -name "test_*" \) -not -path "*/node_modules/*" 2>/dev/null | head -40
```

### 2d. Cross-Reference

Match each requirement to existing tests by filename, imports, and test descriptions. Record: requirement → test_file → status.

## 3. Gap Analysis

Classify each requirement:

| Status | Criteria |
|--------|----------|
| COVERED | A test exists, targets behavior, and passes |
| PARTIAL | A test exists but fails or is incomplete |
| MISSING | No test found |

Build: `{ task_id, requirement, gap_type, suggested_test_path, suggested_command }`

If there are no gaps, skip to Step 6 and set `nyquist_compliant: true`.

## 4. Present Gap Plan

**Text mode (`workflow.text_mode: true` in config or `--text` flag):** If `--text` is present in `$ARGUMENTS` OR `text_mode` from init JSON is `true`, set `TEXT_MODE=true`. When TEXT_MODE is active, replace every `AskUserQuestion` call with a plain-text numbered list. Ask the user to type their choice number. Non-Claude runtimes, such as OpenAI Codex and Antigravity, need this because `AskUserQuestion` is not available.

Call AskUserQuestion with the gap table and options:
1. "Fix all gaps" → Step 5
2. "Skip — mark manual-only" → add to Manual-Only, Step 6
3. "Cancel" → exit

## 5. Spawn gsd-nyquist-auditor

Print: `◆ Spawning nyquist auditor... (runs in a subagent — no output until it returns, ~1–5 min; expected, not a freeze)`

<!-- #2508 runtime-aware-dispatch -->

> **Runtime-aware dispatch (#2508 Phase 4).** GSD workflows dispatch specialized subagents by role. Before dispatch on a built-in-only runtime (kimi-code — three built-ins only), resolve the role to a built-in with `gsd_run query resolve-dispatch-type --requested <role> --raw`. On named-dispatch runtimes (Claude/OpenCode/…), the command returns the role unchanged. On kimi-code, the command maps the role to `coder`/`explore`/`plan` by role-suffix. The persona uses `${AGENT_SKILLS_<ROLE>}` (Phase 3) in both cases. See @gsd-core/references/runtime-aware-dispatch.md.

<!-- #2517 model-omit-on-inherit -->

> **Model omission (#2517).** If the value for `model` (`AUDITOR_MODEL`) is `"inherit"` or empty, omit the `model` parameter entirely. An empty value causes a 404 on runtimes without native tier aliases, the default on non-Claude runtimes. If you omit the parameter, the subagent inherits the orchestrator's model. See @gsd-core/references/model-profile-resolution.md.

```
Agent(
  prompt="Read ~/.claude/agents/gsd-nyquist-auditor.md for instructions.\n\n" +
    "<required_reading>{PLAN, SUMMARY, impl files, VALIDATION.md}</required_reading>" +
    "<gaps>{gap list}</gaps>" +
    "<test_infrastructure>{framework, config, commands}</test_infrastructure>" +
    "<constraints>Never modify impl files. Max 3 debug iterations. Escalate impl bugs.</constraints>" +
    "${AGENT_SKILLS_AUDITOR}",
  subagent_type="gsd-nyquist-auditor",
  model="{AUDITOR_MODEL}",
  description="Fill validation gaps for Phase {N}"
)
```

> **ORCHESTRATOR RULE — CODEX RUNTIME**: After you call Agent() above, stop work on this task immediately. While the subagent is active, follow this rule. Do not read more files, edit code, or run tests related to this task. Wait for the subagent to return its result. This prevents duplicate work, conflicting edits, and wasted context. When the subagent result is available, resume work only then.

Handle return:
- `## GAPS FILLED` → record tests + map updates, Step 6
- `## PARTIAL` → record resolved, move escalated to manual-only, Step 6
- `## ESCALATE` → move all to manual-only, Step 6

## 6. Generate/Update VALIDATION.md

**State B (create):**
1. Read template from `~/.claude/gsd-core/templates/VALIDATION.md`
2. Fill the frontmatter (**set `status: validated`**), Test Infrastructure, Per-Task Map, Manual-Only, and Sign-Off
3. Write to `${PHASE_DIR}/${PADDED_PHASE}-VALIDATION.md`

**State A (update):**
1. Update Per-Task Map statuses. Add escalated items to Manual-Only. Update frontmatter (**set `status: validated`**).
2. Append to the audit trail through the owning verb after the Per-Task Map update above, not instead of it.

   The verb compares against the LAST `## Validation Audit` block. When nothing material changed, the verb skips the write entirely (#5105 R3). For VALIDATION, idempotence alone is not enough (#4981). VALIDATION content legitimately changes with the plan/SUMMARY set:

```bash
gsd_run query verification.append-audit "${PHASE_DIR}/${PADDED_PHASE}-VALIDATION.md" \
  --heading "Validation Audit" \
  --rows "{\"Gaps found\": {N}, \"Resolved\": {M}, \"Escalated\": {K}}"
```

## 7. Commit

```bash
git add {test_files}
git commit -m "test(phase-${PHASE}): add Nyquist validation tests"

gsd_run query commit "docs(phase-${PHASE}): add/update validation strategy" \
  --files "${PHASE_DIR}/${PADDED_PHASE}-VALIDATION.md"
```

## 8. Results + Routing

**Compliant:**
```
GSD > PHASE {N} IS NYQUIST-COMPLIANT
All requirements have automated verification.
▶ Next: /gsd:audit-milestone ${GSD_WS}
```

**Partial:**
```
GSD > PHASE {N} VALIDATED (PARTIAL)
{M} automated, {K} manual-only.
▶ Retry: /gsd:validate-phase {N} ${GSD_WS}
```

Show the `/clear` reminder.

</process>

<success_criteria>
- [ ] Examine Nyquist config (exit if disabled)
- [ ] Detect input state (A/B/C)
- [ ] Exit cleanly for State C
- [ ] Read PLAN/SUMMARY files and build the requirement map
- [ ] Detect test infrastructure
- [ ] Classify gaps (COVERED/PARTIAL/MISSING)
- [ ] Show the user gate with the gap table
- [ ] Spawn the auditor with complete context
- [ ] Handle all three return formats
- [ ] Create or update VALIDATION.md
- [ ] Commit test files separately
- [ ] Show results with routing
</success_criteria>
