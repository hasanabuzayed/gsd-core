@~/.claude/gsd-core/references/response-language-directive.md

<purpose>
Extract decisions, lessons, patterns, and surprises from completed phase artifacts into a structured LEARNINGS.md file. This captures institutional knowledge that would otherwise be lost between phases.
</purpose>

<required_reading>
Before you start, read all files that the invoking prompt's execution_context references.
</required_reading>

<objective>
Analyze the completed phase artifacts: PLAN.md, SUMMARY.md, VERIFICATION.md, UAT.md, and STATE.md.

Extract structured learnings into 4 categories: decisions, lessons, patterns, and surprises. Include source attribution for each extracted item.

The output is a LEARNINGS.md file with YAML frontmatter that contains metadata about the extraction.
</objective>

<process>

<step name="initialize">
Parse arguments and load project state:

```bash
IFS= read -rd '' GSD_ARGS <<'GSD_ARGS_END' || true
$ARGUMENTS
GSD_ARGS_END
GSD_ARGS=${GSD_ARGS%$'\n'}
_GSD_SHIM_NAME="gsd-tools.cjs"; _GSD_RUNTIME_ROOT="${RUNTIME_DIR:-$(git rev-parse --show-toplevel 2>/dev/null || pwd)}"; GSD_TOOLS="${_GSD_RUNTIME_ROOT}/gsd-core/bin/${_GSD_SHIM_NAME}"; _gsd_at() { for _p; do if [ -f "$_p" ]; then GSD_TOOLS="$_p"; return 0; fi; done; return 1; }; _gsd_id_ok() { case "$("$1" runtime-identity --raw 2>/dev/null || true)" in '{"packageName":"@opengsd/gsd-core"'*'}') return 0;; *) return 1;; esac; }; _gsd_homes() { set -- "${CLAUDE_CONFIG_DIR:-$HOME/.claude}" "${ANTIGRAVITY_CONFIG_DIR:-$HOME/.gemini/antigravity}" "$HOME/.gemini/antigravity-ide" "$HOME/.gemini/antigravity-cli" "${AUGMENT_CONFIG_DIR:-$HOME/.augment}" "${CLINE_CONFIG_DIR:-$HOME/.cline}" "${CODEBUDDY_CONFIG_DIR:-$HOME/.codebuddy}" "${CODEX_HOME:-$HOME/.codex}" "${COPILOT_CONFIG_DIR:-${COPILOT_HOME:-$HOME/.copilot}}" "${CURSOR_CONFIG_DIR:-$HOME/.cursor}" "${HERMES_HOME:-$HOME/.hermes}" "${KILO_CONFIG_DIR:-${XDG_CONFIG_HOME:-$HOME/.config}/kilo}" "${KIMI_CONFIG_DIR:-$HOME/.config/agents}" "$HOME/.agents" "${KIMI_CODE_HOME:-$HOME/.kimi-code}" "${OPENCODE_CONFIG_DIR:-${XDG_CONFIG_HOME:-$HOME/.config}/opencode}" "${PI_CODING_AGENT_DIR:-$HOME/.pi/agent}" "${QWEN_CONFIG_DIR:-$HOME/.qwen}" "${TRAE_CONFIG_DIR:-$HOME/.trae}" "${WINDSURF_CONFIG_DIR:-$HOME/.codeium/windsurf}" "${ZCODE_CONFIG_DIR:-$HOME/.zcode}" "${GROK_AGENTS_HOME:-$HOME/.agents}"; for _h; do _gsd_at "$_h/gsd-core/bin/${_GSD_SHIM_NAME}" && return 0; done; return 1; }; if _gsd_at "${_GSD_RUNTIME_ROOT}/gsd-core/bin/${_GSD_SHIM_NAME}" "${_GSD_RUNTIME_ROOT}/.claude/gsd-core/bin/${_GSD_SHIM_NAME}" "${_GSD_RUNTIME_ROOT}/.codex/gsd-core/bin/${_GSD_SHIM_NAME}"; then gsd_run() { node "$GSD_TOOLS" "$@"; }; elif _gsd_homes; then gsd_run() { node "$GSD_TOOLS" "$@"; }; elif unset -f gsd_run; _G="$(command -v gsd_run)"; [ -n "$_G" ] && _gsd_id_ok "$_G"; then GSD_TOOLS="$_G"; gsd_run() { "$GSD_TOOLS" "$@"; }; else echo "ERROR: gsd-tools.cjs not found at $GSD_TOOLS and no identity-proving gsd_run is on PATH. Run: npx -y @opengsd/gsd-core@latest --claude --local" >&2; exit 1; fi; GSD_IDENTITY_STATUS=unverified; _gsd_id_ok gsd_run && GSD_IDENTITY_STATUS=ok; export GSD_IDENTITY_STATUS; [ "$GSD_IDENTITY_STATUS" = ok ] || echo "WARNING: \"$GSD_TOOLS\" did not prove it is @opengsd/gsd-core - it is either a different package or an @opengsd/gsd-core older than the runtime-identity verb. See docs/how-to/diagnose-a-foreign-gsd-tools.md" >&2; if [ -n "${CLAUDE_ENV_FILE:-}" ] && [ -n "${GSD_TOOLS:-}" ]; then printf "export PATH='%s':\"\$PATH\"\n" "${GSD_TOOLS%/*}" >> "$CLAUDE_ENV_FILE" 2>/dev/null || true; fi
PHASE_ARG=$(echo "$GSD_ARGS" | sed -E 's/--ws[[:space:]]+[A-Za-z0-9._-]+//g' | xargs)
GSD_WS=$(echo " $GSD_ARGS" | sed -nE 's/.* --ws +([A-Za-z0-9][A-Za-z0-9._-]*).*/--ws \1/p' | head -n 1)
INIT=$(gsd_run query init.phase-op ${GSD_WS:+--ws=${GSD_WS##* }} "${PHASE_ARG}")
if [[ "$INIT" == @file:* ]]; then INIT=$(cat "${INIT#@file:}"); fi
```

Parse these fields from init JSON: `phase_found`, `phase_dir`, `phase_number`, `phase_name`, `padded_phase`.

If the phase is not found, exit with the error: "Phase {PHASE_ARG} not found."
</step>

<step name="collect_artifacts">
Read the phase artifacts. Learning extraction needs PLAN.md and SUMMARY.md. VERIFICATION.md, UAT.md, and STATE.md are optional.

**Need artifacts:**
- `${PHASE_DIR}/*-PLAN.md` — all plan files for the phase
- `${PHASE_DIR}/*-SUMMARY.md` — all summary files for the phase

If PLAN.md or SUMMARY.md files are missing, exit with the error: "Need artifacts missing. Learning extraction needs PLAN.md and SUMMARY.md."

**Optional artifacts:** If available, read them. If not found, skip them.
- `${PHASE_DIR}/*-VERIFICATION.md` — verification results
- `${PHASE_DIR}/*-UAT.md` — user acceptance test results
- `.planning/STATE.md` — project state with decisions and blockers

Track which optional artifacts are missing for the `missing_artifacts` frontmatter field.
</step>

<step name="extract-learnings">
Analyze all collected artifacts. Extract learnings into 4 categories:

### 1. Decisions
These are technical and architectural decisions made during the phase. Look for:
- Explicit decisions documented in PLAN.md or SUMMARY.md
- Technology choices and their rationale
- Trade-offs that the phase evaluated
- Design decisions recorded in STATE.md

Each decision entry must include:
- **What** the phase decided
- **Why** the phase made the decision (rationale)
- **Source:** attribution to the artifact that contains the decision (for example, "Source: 03-01-PLAN.md")

### 2. Lessons
These are things learned during execution that were not known before. Look for:
- Unexpected complexity in SUMMARY.md
- Issues discovered during verification in VERIFICATION.md
- Failed approaches documented in SUMMARY.md
- UAT feedback that revealed gaps

Each lesson entry must include:
- **What** the phase learned
- **Context** for the lesson
- **Source:** attribution to the start artifact

### 3. Patterns
These are reusable patterns, approaches, or techniques that the phase discovered. Look for:
- Successful implementation patterns in SUMMARY.md
- Testing patterns from VERIFICATION.md or UAT.md
- Workflow patterns that worked well
- Code organization patterns from PLAN.md

Each pattern entry must include:
- **Pattern** name/description
- **When to use** it
- **Source:** attribution to the start artifact

### 4. Surprises
These are unexpected findings, behaviors, or outcomes. Look for:
- Things that took longer or shorter than estimated
- Unexpected dependencies or interactions
- Edge cases not expected in planning
- Performance or behavior that differed from expectations

Each surprise entry must include:
- **What** caused surprise
- **Impact** of the surprise
- **Source:** attribution to the start artifact
</step>

<step name="capture_thought_integration">
**What this step is:** `capture_thought` is an **optional convention**, not a bundled GSD tool. GSD does not ship one and does not need one. This step is a hook for users who run a memory / knowledge-base MCP server. The server exposes a tool with this exact name. Examples include ExoCortex-style servers, `claude-mem`, and `mem0`-style servers.

If any MCP server in the current session provides a `capture_thought` tool with the signature below, route each learning through it. Include metadata. If no such tool is present, the step does nothing and gives no message. `LEARNINGS.md` is always the primary output.

**Detection:** Examine the tools in the current session for a tool named `capture_thought`. Do not assume that any specific MCP server is connected.

**If available**, call once per extracted learning:

```
capture_thought({
  category: "decision" | "lesson" | "pattern" | "surprise",
  phase: PHASE_NUMBER,
  content: LEARNING_TEXT,
  source: ARTIFACT_NAME
})
```

**If not available** (no MCP server in the session exposes this tool, or the runtime does not support it), skip the step without a message. Continue the workflow. The workflow must not fail or warn. This is expected behavior for users who do not run a knowledge-base MCP.
</step>

<step name="write_learnings">
Write the LEARNINGS.md file to the phase directory. If a previous LEARNINGS.md exists, overwrite it. Replace the file entirely.

Output path: `${PHASE_DIR}/${PADDED_PHASE}-LEARNINGS.md`

The file must have YAML frontmatter with these fields:
```yaml
---
phase: {PHASE_NUMBER}
phase_name: "{PHASE_NAME}"
project: "{PROJECT_NAME}"
generated: "{ISO_DATE}"
counts:
  decisions: {N}
  lessons: {N}
  patterns: {N}
  surprises: {N}
missing_artifacts:
  - "{ARTIFACT_NAME}"
---
```

Individual items may carry an optional `graduated:` annotation. `graduation.md` adds it when a cluster is promoted:
```markdown
**Graduated:** {target-file}:{ISO_DATE}
```
The graduation workflow appends this annotation after the item's existing fields. The annotation prevents future graduation scans from showing the item again.

Do not add this field during extraction. Only the graduation workflow writes it.

The body follows this structure:
```markdown
# Phase {PHASE_NUMBER} Learnings: {PHASE_NAME}

## Decisions

### {Decision Title}
{What was decided}

**Rationale:** {Why}
**Source:** {artifact file}

---

## Lessons

### {Lesson Title}
{What was learned}

**Context:** {context}
**Source:** {artifact file}

---

## Patterns

### {Pattern Name}
{Description}

**When to use:** {applicability}
**Source:** {artifact file}

---

## Surprises

### {Surprise Title}
{What was surprising}

**Impact:** {impact description}
**Source:** {artifact file}
```
</step>

<step name="calibrate_estimates">
Rebuild the estimate-vs-actual calibration from every completed phase (#2632, ADR-2629).

```bash
gsd_run query estimate-calibrate
```

This pairs each phase's PLAN `estimate` with its SUMMARY `actuals`. It writes
`.planning/estimation-calibration.json` and reports the resulting correction factor.
The planner reads it on the next `/gsd:plan-phase`. Estimates improve for THIS project
over time.

Report the returned `factor`, `sample_count`, and `confidence` in the summary output.
`applied: false` means fewer than 3 phases carry both an estimate and actuals. This is
expected early and is not an error.

The verb rebuilds from scratch each run. It is safe to run it again, and it never accumulates duplicates.

The verb skips phases that lack either side instead of making a guess. A fabricated sample would
steer every future estimate.
</step>

<step name="update_state">
Update STATE.md with the learning extraction activity:

```bash
gsd_run query state.update "Last Activity" "$(date +%Y-%m-%d)"
```
</step>

<step name="report">
```
---------------------------------------------------------------

## Learnings Extracted: Phase {X} — {Name}

Decisions:  {N}
Lessons:    {N}
Patterns:   {N}
Surprises:  {N}
Total:      {N}

Output: {PHASE_DIR}/{PADDED_PHASE}-LEARNINGS.md

Missing artifacts: {list or "none"}

Next steps:
- Review extracted learnings for accuracy
- /gsd:progress — see overall project state
- /gsd:execute-phase {next} — continue to next phase

---------------------------------------------------------------
```
</step>

</process>

<success_criteria>
- [ ] Find and read phase artifacts successfully
- [ ] Extract all 4 categories: decisions, lessons, patterns, surprises
- [ ] Include source attribution for each extracted item
- [ ] Write LEARNINGS.md with correct YAML frontmatter
- [ ] Track missing optional artifacts in frontmatter
- [ ] If the tool is available, try capture_thought integration
- [ ] Update STATE.md with extraction activity
- [ ] Give the user a summary report
</success_criteria>

<critical_rules>
- Learning extraction needs PLAN.md and SUMMARY.md — exit with clear error if missing
- VERIFICATION.md, UAT.md, and STATE.md are optional. If present, extract from them. If not found, skip them gracefully.
- Every extracted learning must have source attribution back to the start artifact
- If you run extract-learnings twice on the same phase, it must overwrite (replace) the previous LEARNINGS.md, not append.
- Do not fabricate learnings. Only extract what the artifacts explicitly document.
- If capture_thought is unavailable, the workflow must not fail. Use file-only output as a graceful degradation.
- LEARNINGS.md frontmatter must include counts for all 4 categories and list any missing_artifacts
</critical_rules>
