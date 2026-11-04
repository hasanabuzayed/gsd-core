@~/.claude/gsd-core/references/response-language-directive.md

<purpose>
Edit any field of an existing phase in ROADMAP.md in place. Always preserve the phase number and position.

Do not edit in-progress or completed phases unless the user passes --force. Validate depends_on references before writing. Show a diff and ask for confirmation before writing.
</purpose>

<required_reading>
Before you start, read all files that the invoking prompt's execution_context references.
</required_reading>

<process>

<step name="parse_arguments">
Parse the command arguments:
- First argument: phase number to edit (integer or decimal)
- Optional flag: --force (allow editing in_progress/completed phases)

Examples:
  `/gsd-edit-phase 5`       → phase = 5, force = false
  `/gsd-edit-phase 5 --force` → phase = 5, force = true
  `/gsd-edit-phase 12.1`    → phase = 12.1, force = false

If the user provides no argument:

```
ERROR: Phase number required
Usage: /gsd-edit-phase <phase-number> [--force]
Example: /gsd-edit-phase 5
Example: /gsd-edit-phase 5 --force
```

Exit.
</step>

<step name="init_context">
Load phase operation context:

```bash
IFS= read -rd '' GSD_ARGS <<'GSD_ARGS_END' || true
$ARGUMENTS
GSD_ARGS_END
GSD_ARGS=${GSD_ARGS%$'\n'}
_GSD_SHIM_NAME="gsd-tools.cjs"; _GSD_RUNTIME_ROOT="${RUNTIME_DIR:-$(git rev-parse --show-toplevel 2>/dev/null || pwd)}"; GSD_TOOLS="${_GSD_RUNTIME_ROOT}/gsd-core/bin/${_GSD_SHIM_NAME}"; _gsd_at() { for _p; do if [ -f "$_p" ]; then GSD_TOOLS="$_p"; return 0; fi; done; return 1; }; _gsd_id_ok() { case "$("$1" runtime-identity --raw 2>/dev/null || true)" in '{"packageName":"@opengsd/gsd-core"'*'}') return 0;; *) return 1;; esac; }; _gsd_homes() { set -- "${CLAUDE_CONFIG_DIR:-$HOME/.claude}" "${ANTIGRAVITY_CONFIG_DIR:-$HOME/.gemini/antigravity}" "$HOME/.gemini/antigravity-ide" "$HOME/.gemini/antigravity-cli" "${AUGMENT_CONFIG_DIR:-$HOME/.augment}" "${CLINE_CONFIG_DIR:-$HOME/.cline}" "${CODEBUDDY_CONFIG_DIR:-$HOME/.codebuddy}" "${CODEX_HOME:-$HOME/.codex}" "${COPILOT_CONFIG_DIR:-${COPILOT_HOME:-$HOME/.copilot}}" "${CURSOR_CONFIG_DIR:-$HOME/.cursor}" "${HERMES_HOME:-$HOME/.hermes}" "${KILO_CONFIG_DIR:-${XDG_CONFIG_HOME:-$HOME/.config}/kilo}" "${KIMI_CONFIG_DIR:-$HOME/.config/agents}" "$HOME/.agents" "${KIMI_CODE_HOME:-$HOME/.kimi-code}" "${OPENCODE_CONFIG_DIR:-${XDG_CONFIG_HOME:-$HOME/.config}/opencode}" "${PI_CODING_AGENT_DIR:-$HOME/.pi/agent}" "${QWEN_CONFIG_DIR:-$HOME/.qwen}" "${TRAE_CONFIG_DIR:-$HOME/.trae}" "${WINDSURF_CONFIG_DIR:-$HOME/.codeium/windsurf}" "${ZCODE_CONFIG_DIR:-$HOME/.zcode}" "${GROK_AGENTS_HOME:-$HOME/.agents}"; for _h; do _gsd_at "$_h/gsd-core/bin/${_GSD_SHIM_NAME}" && return 0; done; return 1; }; if _gsd_at "${_GSD_RUNTIME_ROOT}/gsd-core/bin/${_GSD_SHIM_NAME}" "${_GSD_RUNTIME_ROOT}/.claude/gsd-core/bin/${_GSD_SHIM_NAME}" "${_GSD_RUNTIME_ROOT}/.codex/gsd-core/bin/${_GSD_SHIM_NAME}"; then gsd_run() { node "$GSD_TOOLS" "$@"; }; elif _gsd_homes; then gsd_run() { node "$GSD_TOOLS" "$@"; }; elif unset -f gsd_run; _G="$(command -v gsd_run)"; [ -n "$_G" ] && _gsd_id_ok "$_G"; then GSD_TOOLS="$_G"; gsd_run() { "$GSD_TOOLS" "$@"; }; else echo "ERROR: gsd-tools.cjs not found at $GSD_TOOLS and no identity-proving gsd_run is on PATH. Run: npx -y @opengsd/gsd-core@latest --claude --local" >&2; exit 1; fi; GSD_IDENTITY_STATUS=unverified; _gsd_id_ok gsd_run && GSD_IDENTITY_STATUS=ok; export GSD_IDENTITY_STATUS; [ "$GSD_IDENTITY_STATUS" = ok ] || echo "WARNING: \"$GSD_TOOLS\" did not prove it is @opengsd/gsd-core - it is either a different package or an @opengsd/gsd-core older than the runtime-identity verb. See docs/how-to/diagnose-a-foreign-gsd-tools.md" >&2; if [ -n "${CLAUDE_ENV_FILE:-}" ] && [ -n "${GSD_TOOLS:-}" ]; then printf "export PATH='%s':\"\$PATH\"\n" "${GSD_TOOLS%/*}" >> "$CLAUDE_ENV_FILE" 2>/dev/null || true; fi
GSD_WS=$(echo " $GSD_ARGS" | sed -nE 's/.* --ws +([A-Za-z0-9][A-Za-z0-9._-]*).*/--ws \1/p' | head -n 1)
INIT=$(gsd_run query init.phase-op ${GSD_WS:+--ws=${GSD_WS##* }} "${target}")
if [[ "$INIT" == @file:* ]]; then INIT=$(cat "${INIT#@file:}"); fi
```

Read `roadmap_exists` from the init JSON. If it is false:
```
ERROR: No roadmap found (.planning/ROADMAP.md)
Run /gsd:new-project to initialize.
```
Exit.
</step>

<step name="load_phase">
Read the current phase section from ROADMAP.md:

```bash
PHASE_DATA=$(gsd_run query roadmap get-phase "${target}")
```

Parse the JSON result. If `found` is false:

```
ERROR: Phase {target} not found in ROADMAP.md

Available phases can be seen with /gsd:progress.
```

Exit.

Extract from the result:
- `phase_name` — the phase title
- `goal` — the phase goal/description
- `success_criteria` — array of criteria
- `section` — full raw section text (preserves depends_on, requirements, plans, and other content)

Also parse the full section text to extract more fields not in the SDK result:
- `depends_on` — from `**Depends on:** ...` or `**Depends on**: ...` line
- `requirements` — from `**Requirements:** ...` block if present
</step>

<step name="check_phase_status">
Find the phase status from disk. Compare it with the current phase in STATE.md:

```bash
ANALYZE=$(gsd_run query roadmap analyze)
```

Find the phase entry in the `phases` array. Extract `disk_status`.

Map disk_status to a user-friendly status:
- `complete` → status = `completed`
- `planned`, `partial`, or `executed` → status = `in_progress`
- `empty`, `no_directory`, `discussed`, `researched` → status = `future`

If status is `in_progress` or `completed` AND `--force` was NOT passed:

```
ERROR: Cannot edit Phase {target} — status is {status}

Editing an in-progress or completed phase may invalidate executed plans.

To edit anyway, run:
  /gsd-edit-phase {target} --force
```

Exit.

If the user passed `--force` and status is `in_progress` or `completed`, show the user a warning and continue:

```
WARNING: Editing Phase {target} which is {status}. Proceeding due to --force.
```
</step>

<step name="present_current_values">
Show the current phase fields clearly:

```
Current values for Phase {target}: {phase_name}

Title:            {phase_name}
Goal:             {goal}
Depends on:       {depends_on or "(none)"}
Requirements:     {requirements or "(none)"}
Success Criteria:
  1. {criterion_1}
  2. {criterion_2}
  ...
```

Then ask the user what they want to change:

```
What would you like to do?

  [1] Edit specific fields (title, goal, depends_on, requirements, success_criteria)
  [2] Regenerate all fields from a clarified intent
  [3] Cancel

Enter choice (1, 2, or 3):
```

Wait for user input.
</step>

<step name="collect_edits">

**If user chose [3] Cancel:** Exit cleanly.

**If user chose [1] Edit specific fields:**

Ask which fields to edit. For each field the user wants to change, ask for the new value. Update only fields for which the user gives an explicit answer. For empty answers, preserve the existing value.

```
Which fields do you want to update? (comma-separated or "all")
Options: title, goal, depends_on, requirements, success_criteria
```

For each selected field, ask:

```
New value for {field} [current: {current_value}]:
```

Build an `updates` map of {field → new_value} for non-empty answers.

**If user chose [2] Regenerate all from clarified intent:**

Ask the user:

```
Describe the revised intent for Phase {target} (replace the current description):
```

Wait for user input. Use the clarified intent to rewrite all fields:
- Generate a clear, concise `title` from the intent
- Write a complete `goal` statement
- If the original had `requirements`, produce updated `requirements`
- Generate `success_criteria` (3-5 measurable criteria)
- Preserve `depends_on` unless the user explicitly mentioned changing it
</step>

<step name="validate_depends_on">
If you update `depends_on` or preserve it as non-empty, validate that every referenced phase number exists in ROADMAP.md:

```bash
ALL_PHASES=$(gsd_run query roadmap analyze)
```

Parse the `phases` array to get all valid phase numbers.

For each phase number referenced in `depends_on`:
- Normalize it (strip whitespace and the "Phase" prefix if present)
- Make sure that it is in the valid phase numbers set
- It must not reference itself (phase {target})

If any reference is invalid:

```
ERROR: depends_on references invalid phase(s): {bad_refs}

Valid phase numbers: {valid_list}

Fix the depends_on field and try again.
```

Exit (do not write).
</step>

<step name="show_diff_and_confirm">
Apply the changes to the original `section` text to build the updated phase section:

- For `title`: replace the heading text after `Phase {N}:`
- For `goal`: replace the `**Goal:**` line value
- For `depends_on`: replace or add the `**Depends on:**` line
- For `requirements`: replace or add the requirements block
- For `success_criteria`: replace the numbered list under `**Success Criteria**:`
- For full regeneration: rebuild the entire section from the new field values

Show a unified-style diff of the old and new sections:

```
Proposed changes to Phase {target}:

--- current
+++ updated
@@ ...
- **Goal:** {old_goal}
+ **Goal:** {new_goal}
...

Apply these changes? (y/n):
```

Wait for confirmation. If the user says `n`, exit without writing.
</step>

<step name="write_updated_phase">
Before writing, capture the current milestone scope (window scope + phase set). Use it to verify the write:

```bash
SCOPE_BEFORE=$(gsd_run query roadmap milestone-scope)
```

Write the updated phase back in place in ROADMAP.md.

Read the full ROADMAP.md content. Find the phase section by its header (`## Phase {N}:` or `### Phase {N}:`). Replace exactly the old section text with the new section text.

All content before and after the section must stay unchanged. This includes other phases, milestone headers, and the summary checklist.

After writing, derive the milestone scope again and compare it with the captured scope:

```bash
SCOPE_AFTER=$(gsd_run query roadmap milestone-scope)
```

If `scope` or `phases` differs from SCOPE_BEFORE, the replacement text introduced a heading that stops the current milestone window. This is a level 1-3 heading with a version token, a ✅/📋/🚧/🔄 marker, or the word "Milestone".

Restore ROADMAP.md by replacing the new section text with the original section text. Then report the error and stop. Do NOT update STATE.md and do NOT retry the write:

```
ERROR: edit rejected — milestone scope changed
Phase set before: {SCOPE_BEFORE phases} / after: {SCOPE_AFTER phases}
ROADMAP.md rolled back to the original section.
The updated phase text must not introduce a milestone-scoping heading; rewrite the offending line as plain text or a list item.
```

Exit.

If the milestone scope is unchanged, update STATE.md Roadmap Evolution:

```bash
gsd_run query state.add-roadmap-evolution \
  --phase {target} \
  --action edited \
  --note "edited fields: {changed_field_list}"
```
</step>

<step name="completion">
Show the completion summary:

```
Phase {target} updated in ROADMAP.md.

Fields changed: {changed_field_list}

---

## What's Next

- `/gsd:progress` — view updated roadmap
- `/gsd:plan-phase {target}` — re-plan this phase (if needed)
- `/gsd:discuss-phase {target}` — discuss implementation approach

---
```
</step>

</process>

<anti_patterns>
- Do not renumber the phase. The number and position must stay exactly the same
- When you edit one phase, do not change other phases
- Do not skip depends_on validation. Invalid references block writes
- Do not write without showing a diff and getting confirmation
- Do not edit in_progress/completed phases without --force
- Do not use raw Write on ROADMAP.md without reading it first. Always replace the section in place
- Do not change the phase directory structure. Only ROADMAP.md changes
- Do not introduce a milestone-scoping heading in field values. This heading stops the current milestone window. The post-write scope check rolls back the edit. A milestone-scoping heading is level 1-3 with a version token, ✅/📋/🚧/🔄 marker, or "Milestone"
- Do not commit the change. The user makes that decision
</anti_patterns>

<success_criteria>
Edit-phase is complete when:

- [ ] Phase {target} found and loaded from ROADMAP.md
- [ ] Status check done. in_progress/completed blocked without --force
- [ ] Current values presented to user
- [ ] User chose edit mode (specific fields or full regeneration)
- [ ] depends_on references validated. Invalid references blocked
- [ ] Diff shown and confirmed by user
- [ ] Updated phase written back in place. Number, position, and status preserved
- [ ] Milestone scope verified unchanged after the write (rollback + error on mismatch)
- [ ] STATE.md Roadmap Evolution updated
- [ ] User informed of next steps
</success_criteria>
