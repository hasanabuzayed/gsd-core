<purpose>
List all pending todos. Let the user select a todo. Load the full context for the selected todo. Route to the applicable action.
</purpose>

<required_reading>
Before you start, read all files that the invoking prompt's execution_context references.
</required_reading>

<process>

<step name="init_context">
Load todo context:

```bash
IFS= read -rd '' GSD_ARGS <<'GSD_ARGS_END' || true
$ARGUMENTS
GSD_ARGS_END
GSD_ARGS=${GSD_ARGS%$'\n'}
_GSD_SHIM_NAME="gsd-tools.cjs"; _GSD_RUNTIME_ROOT="${RUNTIME_DIR:-$(git rev-parse --show-toplevel 2>/dev/null || pwd)}"; GSD_TOOLS="${_GSD_RUNTIME_ROOT}/gsd-core/bin/${_GSD_SHIM_NAME}"; _gsd_at() { for _p; do if [ -f "$_p" ]; then GSD_TOOLS="$_p"; return 0; fi; done; return 1; }; _gsd_id_ok() { case "$("$1" runtime-identity --raw 2>/dev/null || true)" in '{"packageName":"@opengsd/gsd-core"'*'}') return 0;; *) return 1;; esac; }; _gsd_homes() { set -- "${CLAUDE_CONFIG_DIR:-$HOME/.claude}" "${ANTIGRAVITY_CONFIG_DIR:-$HOME/.gemini/antigravity}" "$HOME/.gemini/antigravity-ide" "$HOME/.gemini/antigravity-cli" "${AUGMENT_CONFIG_DIR:-$HOME/.augment}" "${CLINE_CONFIG_DIR:-$HOME/.cline}" "${CODEBUDDY_CONFIG_DIR:-$HOME/.codebuddy}" "${CODEX_HOME:-$HOME/.codex}" "${COPILOT_CONFIG_DIR:-${COPILOT_HOME:-$HOME/.copilot}}" "${CURSOR_CONFIG_DIR:-$HOME/.cursor}" "${HERMES_HOME:-$HOME/.hermes}" "${KILO_CONFIG_DIR:-${XDG_CONFIG_HOME:-$HOME/.config}/kilo}" "${KIMI_CONFIG_DIR:-$HOME/.config/agents}" "$HOME/.agents" "${KIMI_CODE_HOME:-$HOME/.kimi-code}" "${OPENCODE_CONFIG_DIR:-${XDG_CONFIG_HOME:-$HOME/.config}/opencode}" "${PI_CODING_AGENT_DIR:-$HOME/.pi/agent}" "${QWEN_CONFIG_DIR:-$HOME/.qwen}" "${TRAE_CONFIG_DIR:-$HOME/.trae}" "${WINDSURF_CONFIG_DIR:-$HOME/.codeium/windsurf}" "${ZCODE_CONFIG_DIR:-$HOME/.zcode}" "${GROK_AGENTS_HOME:-$HOME/.agents}"; for _h; do _gsd_at "$_h/gsd-core/bin/${_GSD_SHIM_NAME}" && return 0; done; return 1; }; if _gsd_at "${_GSD_RUNTIME_ROOT}/gsd-core/bin/${_GSD_SHIM_NAME}" "${_GSD_RUNTIME_ROOT}/.claude/gsd-core/bin/${_GSD_SHIM_NAME}" "${_GSD_RUNTIME_ROOT}/.codex/gsd-core/bin/${_GSD_SHIM_NAME}"; then gsd_run() { node "$GSD_TOOLS" "$@"; }; elif _gsd_homes; then gsd_run() { node "$GSD_TOOLS" "$@"; }; elif unset -f gsd_run; _G="$(command -v gsd_run)"; [ -n "$_G" ] && _gsd_id_ok "$_G"; then GSD_TOOLS="$_G"; gsd_run() { "$GSD_TOOLS" "$@"; }; else echo "ERROR: gsd-tools.cjs not found at $GSD_TOOLS and no identity-proving gsd_run is on PATH. Run: npx -y @opengsd/gsd-core@latest --claude --local" >&2; exit 1; fi; GSD_IDENTITY_STATUS=unverified; _gsd_id_ok gsd_run && GSD_IDENTITY_STATUS=ok; export GSD_IDENTITY_STATUS; [ "$GSD_IDENTITY_STATUS" = ok ] || echo "WARNING: \"$GSD_TOOLS\" did not prove it is @opengsd/gsd-core - it is either a different package or an @opengsd/gsd-core older than the runtime-identity verb. See docs/how-to/diagnose-a-foreign-gsd-tools.md" >&2; if [ -n "${CLAUDE_ENV_FILE:-}" ] && [ -n "${GSD_TOOLS:-}" ]; then printf "export PATH='%s':\"\$PATH\"\n" "${GSD_TOOLS%/*}" >> "$CLAUDE_ENV_FILE" 2>/dev/null || true; fi
GSD_WS=$(echo " $GSD_ARGS" | sed -nE 's/.* --ws +([A-Za-z0-9][A-Za-z0-9._-]*).*/--ws \1/p' | head -n 1)
INIT=$(gsd_run query init.todos ${GSD_WS:+--ws=${GSD_WS##* }})
if [[ "$INIT" == @file:* ]]; then INIT=$(cat "${INIT#@file:}"); fi
```

Extract these fields from the init JSON: `todo_count`, `todos`, `pending_dir`, `response_language`.

**If `response_language` is set:** All user-facing output of this workflow MUST use `{response_language}`. This output includes narration between tool calls, status updates, progress notes, findings, questions, prompts, and explanations.

Technical terms, code, file paths, and subagent prompts stay in English. Translate only user-facing output.

If `todo_count` is 0:
```
No pending todos.

Todos are captured during work sessions with /gsd-add-todo.

---

Would you like to:

1. Continue with current phase (/gsd:progress)
2. Add a todo now (/gsd-add-todo)
```

Exit.
</step>

<step name="parse_filter">
Examine the arguments for an area filter:
- `/gsd:capture --list` → show all
- `/gsd:capture --list api` → filter to area:api only
</step>

<step name="list_todos">
Use the `todos` array from the init context. If the user specified an area, the array is already filtered by area.

Parse the array. Show it as a numbered list:

```
Pending Todos:

1. Add auth token refresh (api, 2d ago)
2. Fix modal z-index issue (ui, 1d ago)
3. Refactor database connection pool (database, 5h ago)

---

Reply with a number to view details, or:
- `/gsd:capture --list [area]` to filter by area
- `q` to exit
```

Format age as relative time from the created timestamp.
</step>

<step name="handle_selection">
Wait for user to reply with a number.

If the selection is valid, load the selected todo. Then proceed.

If the selection is invalid, show: "Invalid selection. Reply with a number (1-[N]) or `q` to exit."
</step>

<step name="load_context">
Read the complete todo file. Show:

```
## [title]

**Area:** [area]
**Created:** [date] ([relative time] ago)
**Files:** [list or "None"]

### Problem
[problem section content]

### Solution
[solution section content]
```

If the `files` field has entries, read each file. Give a short summary of each file.
</step>

<step name="check_roadmap">
Look for the roadmap. You can use init progress or examine whether the file exists directly.

If `.planning/ROADMAP.md` exists:
1. Examine whether the todo's area matches an upcoming phase.
2. Examine whether the todo's files overlap with a phase's scope.
3. Note any match for the action options.
</step>

<step name="offer_actions">
**If todo maps to a roadmap phase:**


**Text mode (`workflow.text_mode: true` in config or `--text` flag):** If `--text` is present in `$ARGUMENTS` OR `text_mode` from init JSON is `true`, set `TEXT_MODE=true`.

When TEXT_MODE is active, replace every `AskUserQuestion` call with a plain-text numbered list. Ask the user to type their choice number.

Non-Claude runtimes where `AskUserQuestion` is not available need this. OpenAI Codex and Antigravity are examples of these runtimes.
Use AskUserQuestion:
- header: "Action"
- question: "This todo relates to Phase [N]: [name]. What would you like to do?"
- options:
  - "Work on it now" — move to done, start working
  - "Add to phase plan" — include when planning Phase [N]
  - "Brainstorm approach" — think through before deciding
  - "Put it back" — return to list

**If no roadmap match:**

Use AskUserQuestion:
- header: "Action"
- question: "What would you like to do with this todo?"
- options:
  - "Work on it now" — move to done, start working
  - "Create a phase" — /gsd:phase with this scope
  - "Brainstorm approach" — think through before deciding
  - "Put it back" — return to list
</step>

<step name="execute_action">
**Work on it now:**
```bash
mv ".planning/todos/pending/[filename]" ".planning/todos/completed/"
```
Update STATE.md todo count. Show the problem/solution context. Start work or ask how to proceed.

**Add to phase plan:**
Note the todo reference in the phase planning notes. Keep the todo in pending. Return to the list or exit.

**Create a phase:**
Show: `/gsd:phase "[description from todo]"`
Keep the todo in pending. The user runs the command in a fresh context.

**Brainstorm approach:**
Keep the todo in pending. Start a discussion about the problem and approaches.

**Put it back:**
Return to list_todos step.
</step>

<step name="update_state">
If `.planning/STATE.md` exists:

1. Re-run `gsd_run query init.todos ${GSD_WS:+--ws=${GSD_WS##* }}` to get a fresh JSON snapshot after the write. Use the same workstream flag as the first call. The count and rendering must reflect the change just made.

2. **Fail-safe check (#2618):** If the JSON failed to parse, do NOT touch the "### Pending Todos" section. If `pending_read_ok` is not `true` or `pending_todos_markdown` is not a string, do NOT touch the section. Leave it exactly as-is and continue to the next step. A partial or malformed `init.todos` result must never overwrite a good existing section.

3. Otherwise, replace the entire body of "### Pending Todos" under "## Accumulated Context" with the literal value of `pending_todos_markdown`. Use the value verbatim. `init.todos` already renders one bullet per pending todo and caps its length.

   Do not reformat, re-wrap, re-order, or hand-edit the bullets. Do not append to the old body. Replace the entire body. This replaces an old run-on-sentence section cleanly without a migration step.
</step>

<step name="git_commit">
If the todo moved to completed/, commit the change:

```bash
git rm --cached .planning/todos/pending/[filename] 2>/dev/null || true
gsd_run query commit "docs: start work on todo - [title]" --files .planning/todos/completed/[filename] .planning/STATE.md
```

The tool respects `commit_docs` config and gitignore automatically.

Confirm: "Committed: docs: start work on todo - [title]"
</step>

</process>

<success_criteria>
- [ ] List all pending todos with title, area, and age
- [ ] If the user specified an area filter, apply it
- [ ] Load the selected todo's full context
- [ ] Examine the roadmap context for a phase match
- [ ] Offer applicable actions
- [ ] Execute the selected action
- [ ] If the todo count changed, update STATE.md
- [ ] If the todo moved to completed/, commit the changes to git
</success_criteria>
