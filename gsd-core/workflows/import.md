# Import Workflow

This workflow imports external plans, detects conflicts, and delegates work to agents.

- **--from**: Import an external plan → detect conflicts → write PLAN.md → validate with gsd-plan-checker

A future PR plans to add `--prd` mode. This mode extracts a PRD into PROJECT.md + REQUIREMENTS.md + ROADMAP.md.

---

<step name="banner">

Show the stage banner:
```bash
_GSD_SHIM_NAME="gsd-tools.cjs"; _GSD_RUNTIME_ROOT="${RUNTIME_DIR:-$(git rev-parse --show-toplevel 2>/dev/null || pwd)}"; GSD_TOOLS="${_GSD_RUNTIME_ROOT}/gsd-core/bin/${_GSD_SHIM_NAME}"; _gsd_at() { for _p; do if [ -f "$_p" ]; then GSD_TOOLS="$_p"; return 0; fi; done; return 1; }; _gsd_id_ok() { case "$("$1" runtime-identity --raw 2>/dev/null || true)" in '{"packageName":"@opengsd/gsd-core"'*'}') return 0;; *) return 1;; esac; }; _gsd_homes() { set -- "${CLAUDE_CONFIG_DIR:-$HOME/.claude}" "${ANTIGRAVITY_CONFIG_DIR:-$HOME/.gemini/antigravity}" "$HOME/.gemini/antigravity-ide" "$HOME/.gemini/antigravity-cli" "${AUGMENT_CONFIG_DIR:-$HOME/.augment}" "${CLINE_CONFIG_DIR:-$HOME/.cline}" "${CODEBUDDY_CONFIG_DIR:-$HOME/.codebuddy}" "${CODEX_HOME:-$HOME/.codex}" "${COPILOT_CONFIG_DIR:-${COPILOT_HOME:-$HOME/.copilot}}" "${CURSOR_CONFIG_DIR:-$HOME/.cursor}" "${HERMES_HOME:-$HOME/.hermes}" "${KILO_CONFIG_DIR:-${XDG_CONFIG_HOME:-$HOME/.config}/kilo}" "${KIMI_CONFIG_DIR:-$HOME/.config/agents}" "$HOME/.agents" "${KIMI_CODE_HOME:-$HOME/.kimi-code}" "${OPENCODE_CONFIG_DIR:-${XDG_CONFIG_HOME:-$HOME/.config}/opencode}" "${PI_CODING_AGENT_DIR:-$HOME/.pi/agent}" "${QWEN_CONFIG_DIR:-$HOME/.qwen}" "${TRAE_CONFIG_DIR:-$HOME/.trae}" "${WINDSURF_CONFIG_DIR:-$HOME/.codeium/windsurf}" "${ZCODE_CONFIG_DIR:-$HOME/.zcode}" "${GROK_AGENTS_HOME:-$HOME/.agents}"; for _h; do _gsd_at "$_h/gsd-core/bin/${_GSD_SHIM_NAME}" && return 0; done; return 1; }; if _gsd_at "${_GSD_RUNTIME_ROOT}/gsd-core/bin/${_GSD_SHIM_NAME}" "${_GSD_RUNTIME_ROOT}/.claude/gsd-core/bin/${_GSD_SHIM_NAME}" "${_GSD_RUNTIME_ROOT}/.codex/gsd-core/bin/${_GSD_SHIM_NAME}"; then gsd_run() { node "$GSD_TOOLS" "$@"; }; elif _gsd_homes; then gsd_run() { node "$GSD_TOOLS" "$@"; }; elif unset -f gsd_run; _G="$(command -v gsd_run)"; [ -n "$_G" ] && _gsd_id_ok "$_G"; then GSD_TOOLS="$_G"; gsd_run() { "$GSD_TOOLS" "$@"; }; else echo "ERROR: gsd-tools.cjs not found at $GSD_TOOLS and no identity-proving gsd_run is on PATH. Run: npx -y @opengsd/gsd-core@latest --claude --local" >&2; exit 1; fi; GSD_IDENTITY_STATUS=unverified; _gsd_id_ok gsd_run && GSD_IDENTITY_STATUS=ok; export GSD_IDENTITY_STATUS; [ "$GSD_IDENTITY_STATUS" = ok ] || echo "WARNING: \"$GSD_TOOLS\" did not prove it is @opengsd/gsd-core - it is either a different package or an @opengsd/gsd-core older than the runtime-identity verb. See docs/how-to/diagnose-a-foreign-gsd-tools.md" >&2; if [ -n "${CLAUDE_ENV_FILE:-}" ] && [ -n "${GSD_TOOLS:-}" ]; then printf "export PATH='%s':\"\$PATH\"\n" "${GSD_TOOLS%/*}" >> "$CLAUDE_ENV_FILE" 2>/dev/null || true; fi
RESPONSE_LANGUAGE=$(gsd_run query config-get response_language --raw --default "" 2>/dev/null || echo "")
```

**If `response_language` is set:** You MUST show all user-facing output of this workflow in `{response_language}`.

This output includes narration between tool calls, status updates, progress notes, findings, questions, prompts, and explanations.

Keep technical terms, code, file paths, and subagent prompts in English. Translate only user-facing output.

```
### GSD ► IMPORT
```

</step>

<step name="parse_arguments">

Parse `$ARGUMENTS` to find the execution mode:

- If `--from` is present, extract FILEPATH, the next token after `--from`. Set MODE=plan.
- If `--prd` is present, show a message that `--prd` is not yet implemented. Exit:
  ```
  GSD > --prd mode is planned for a future release. Use --from to import plan files.
  ```
- If neither flag is present, show usage and exit:

```
Usage: /gsd:import --from <path>

  --from <path>   Import an external plan file into GSD format
```

**Validate the file path:**

Make sure that the path does not contain traversal sequences. Make sure that the file exists:

```bash
case "{FILEPATH}" in
  *..* ) echo "SECURITY_ERROR: path contains traversal sequence"; exit 1 ;;
esac
test -f "{FILEPATH}" || echo "FILE_NOT_FOUND"
```

If FILE_NOT_FOUND, show the error and exit:

```
### ERROR

File not found: {FILEPATH}

**To fix:** Verify the file path and try again.
```

</step>

---

## Path A: MODE=plan (--from)

<step name="plan_load_context">

Load project context for conflict detection:

1. Read `.planning/ROADMAP.md`. Extract the phase structure, phase numbers, and dependencies.
2. Read `.planning/PROJECT.md`. Extract the project constraints, tech stack, and scope boundaries.
   **If PROJECT.md does not exist:** Skip constraint checks that rely on it. Show:
   ```
   GSD > Note: No PROJECT.md found. Conflict checks against project constraints will be skipped.
   ```
3. Read `.planning/REQUIREMENTS.md`. Extract existing requirements for overlap and contradiction checks.
   **If REQUIREMENTS.md does not exist:** Skip requirement conflict checks. Continue.
4. Find all CONTEXT.md files across phase directories:
   ```bash
   find .planning/phases/ -name "*-CONTEXT.md" -o -name "CONTEXT.md" 2>/dev/null
   ```
   Read each CONTEXT.md file that you find. Extract locked decisions, which are decisions in a `<decisions>` block.

Store the loaded context for conflict detection in the next step.

</step>

<step name="plan_read_input">

Read the imported file at FILEPATH.

Find the format:
- **GSD PLAN.md format**: The file has YAML frontmatter with `phase:`, `plan:`, and `type:` fields.
- **Freeform document**: The file has any other format, such as a markdown spec, design doc, or task list.

Extract from the imported content:
- **Phase target**: The phase for this plan, from frontmatter or inferred from content
- **Plan objectives**: What the plan aims to accomplish
- **Tasks listed**: Individual work items in the plan
- **Files change**: Any files named as targets
- **Dependencies**: Any referenced prerequisites

</step>

<step name="plan_conflict_detection">

Run conflict checks against the loaded project context. Read `gsd-core/references/doc-conflict-engine.md`. Apply its report format, severity semantics, and safety-gate behavior here. Operation noun: `import`.

### BLOCKER checks (any one prevents import):

- Plan targets a phase number that does not exist in ROADMAP.md → [BLOCKER]
- Plan specifies a tech stack that contradicts PROJECT.md constraints → [BLOCKER]
- Plan contradicts a locked decision in any CONTEXT.md `<decisions>` block → [BLOCKER]
- Plan contradicts an existing requirement in REQUIREMENTS.md → [BLOCKER]

### WARNING checks (user confirmation need):

- Plan partially overlaps existing requirement coverage in REQUIREMENTS.md → [WARNING]
- Plan has `depends_on` that references plans that are not yet complete → [WARNING]
- Plan changes files that overlap with existing incomplete plans → [WARNING]
- Plan phase number conflicts with existing phase numbering in ROADMAP.md → [WARNING]

### INFO checks (informational, no action needed):

- Plan uses a library not currently in the project tech stack → [INFO]
- Plan adds a new phase to the ROADMAP.md structure → [INFO]

Show the full Conflict Detection Report in the format from `gsd-core/references/doc-conflict-engine.md`.

**If any [BLOCKER] exists:** Apply the safety gate from the reference. Exit WITHOUT writing any files. If blockers exist, do not write PLAN.md.

**If only WARNINGS and/or INFO (no blockers):**

**Text mode (`workflow.text_mode: true` in config or `--text` flag):**

If `--text` is present in `$ARGUMENTS` OR `text_mode` from init JSON is `true`, set `TEXT_MODE=true`.

When TEXT_MODE is active, replace every `AskUserQuestion` call with a plain-text numbered list. Ask the user to type their choice number.

Non-Claude runtimes, such as OpenAI Codex and Antigravity, need this because `AskUserQuestion` is not available.

Use AskUserQuestion with the approve-revise-abort pattern. See `gsd-core/references/gate-prompts.md`.

- question: "Review the warnings above. Proceed with import?"
- header: "Approve?"
- options: Approve | Abort

If the user selects "Abort", exit cleanly with the message "Import cancelled."

</step>

<step name="plan_convert">

Convert the imported content to GSD PLAN.md format.

Make sure that PLAN.md has all necessary frontmatter fields:
```yaml
---
phase: "{NN}-{slug}"
plan: "{NN}-{MM}"
type: "feature|refactor|config|test|docs"
wave: 1
depends_on: []
files_modified: []
autonomous: true
must_haves:
  truths: []
  artifacts: []
---
```

**Reject PBR naming conventions in source content:**
If the imported plan references PBR plan naming, rename all references to the GSD `{NN}-{MM}-PLAN.md` convention during conversion. Examples of PBR plan naming are `PLAN-01.md` and `plan-01.md`.

Apply the GSD naming convention for the output filename:
- Format: `{NN}-{MM}-PLAN.md` (for example, `04-01-PLAN.md`)
- NEVER use `PLAN-01.md`, `plan-01.md`, or any other format
- NN = phase number (zero-padded), MM = plan number within the phase (zero-padded)

Query `init.phase-op` for the phase number extracted in `plan_read_input` to find the target directory. This applies the `project_code` prefix from `.planning/config.json`:

```bash
IFS= read -rd '' GSD_ARGS <<'GSD_ARGS_END' || true
$ARGUMENTS
GSD_ARGS_END
GSD_ARGS=${GSD_ARGS%$'\n'}
GSD_WS=$(echo " $GSD_ARGS" | sed -nE 's/.* --ws +([A-Za-z0-9][A-Za-z0-9._-]*).*/--ws \1/p' | head -n 1)
INIT=$(gsd_run query init.phase-op ${GSD_WS:+--ws=${GSD_WS##* }} "{NN}")
if [[ "$INIT" == @file:* ]]; then INIT=$(cat "${INIT#@file:}"); fi
expected_phase_dir=$(echo "$INIT" | node -e "process.stdout.write(JSON.parse(require('fs').readFileSync('/dev/stdin','utf8')).expected_phase_dir)")
```

If the directory does not exist, create it:
```bash
mkdir -p "${expected_phase_dir}"
```

Set `phase_dir="${expected_phase_dir}"` for use in the next steps.

Write the PLAN.md file to the target directory.

</step>

<step name="plan_validate">

Delegate validation to gsd-plan-checker:

Print: "Delegating to gsd-plan-checker (runs in a subagent — no output until it returns, ~1–5 min. Expected, not a freeze)"

```bash
CHECKER_MODEL=$(gsd_run query resolve-model gsd-plan-checker --raw)
```

<!-- #2508 runtime-aware-dispatch -->

> **Runtime-aware dispatch (#2508 Phase 4).** GSD workflows dispatch specialized subagents by role.

> Before dispatch on a built-in-only runtime (kimi-code — three built-ins only), resolve the role to a built-in. Use `gsd_run query resolve-dispatch-type --requested <role> --raw`.

> On named-dispatch runtimes (Claude/OpenCode/…), the role returns unchanged. On kimi-code, the role maps to `coder`/`explore`/`plan` by role-suffix. In all cases, the persona uses `${AGENT_SKILLS_<ROLE>}` (Phase 3). See @gsd-core/references/runtime-aware-dispatch.md.

<!-- #2517 model-omit-on-inherit -->

> **Model omission (#2517).** If the value for `model` (`CHECKER_MODEL`) is `"inherit"` or empty, omit the `model` parameter entirely. An empty value causes a 404 on runtimes without native tier aliases. This is the default on non-Claude runtimes.

> If you omit `model`, the subagent inherits the orchestrator's model. See @gsd-core/references/model-profile-resolution.md.

```
Agent({
  subagent_type: "gsd-plan-checker",
  model: "{CHECKER_MODEL}",
  prompt: "Validate: ${phase_dir}/{plan}-PLAN.md — check frontmatter completeness, task structure, and GSD conventions. Report any issues."
})
```

> **ORCHESTRATOR RULE — CODEX RUNTIME**: After you call Agent() above, stop work on this task immediately. Do not read more files, edit code, or run tests related to this task while the subagent is active. Wait for the subagent to return its result.

> This prevents duplicate work, conflicting edits, and wasted context. Resume only when the subagent result is available.

Handle the checker return by severity, never by the sentinel alone. Count BLOCKER + WARNING entries in the YAML issues block. Count an entry with missing or unrecognized severity as a BLOCKER (fail closed).

If a `## VERIFICATION PASSED` return lists a BLOCKER or WARNING entry in the YAML issues block or in prose, count it as `## ISSUES FOUND`. Count the prose entries as BLOCKER/WARNING entries (#5578). A mention such as "no warnings" or a zero count is not an entry.

If the return is `## VERIFICATION PASSED` with no such entry, or the count is zero — every entry is explicitly INFO — show `ℹ advisory — {dimension}: {description}` per INFO entry. Treat the plan as imported. INFO is advisory and never blocks an import (#3724).

Otherwise:
- Show the blocking issues to the user.
- Ask the user to resolve issues before you consider the plan imported.
- Do not delete the written file. The user can fix and re-validate it manually.

If the checker returns clean:
- Show: "Plan validation passed"

</step>

<step name="plan_finalize">

Update `.planning/ROADMAP.md` to reflect the new plan:
- Add the plan to the Plans list under the correct phase section
- Include the plan name and description

If appropriate, update `.planning/STATE.md` (for example, increment total plan count).

Commit the imported plan and updated files:
```bash
gsd_run query commit "docs({phase}): import plan from {basename FILEPATH}" --files .planning/phases/{phase}/{plan}-PLAN.md .planning/ROADMAP.md
```

Show completion:
```
### GSD ► IMPORT COMPLETE
```

Show: plan filename written, phase directory, validation result, next steps.

</step>

---

## Anti-Patterns

Do NOT:
- Violate the shared conflict-engine contract in `gsd-core/references/doc-conflict-engine.md` (no markdown tables, no new severity labels, no bypass of the BLOCKER gate)
- Write PLAN.md files as `PLAN-01.md` or `plan-01.md` — always use `{NN}-{MM}-PLAN.md`
- Use `pbr:plan-checker` or `pbr:planner` — use `gsd-plan-checker` and `gsd-planner`
- Write `.planning/.active-skill` — this is a PBR pattern with no GSD equivalent
- Reference `pbr-tools`, `pbr:`, or `PLAN-BUILD-RUN` anywhere
- If blockers exist, write any PLAN.md file — the safety gate must hold
- Skip path validation on the --from file argument
