<purpose>
Capture an idea, task, or issue from a GSD session as a structured todo for later work. This lets you use the "thought → capture → continue" flow without losing context.
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

Extract these values from init JSON: `commit_docs`, `date`, `timestamp`, `todo_count`, `todos`, `pending_dir`, `todos_dir_exists`, `response_language`.

**If `response_language` is set:** You MUST show all user-facing output of this workflow in `{response_language}`. This output includes narration between tool calls, status updates, progress notes, findings, questions, prompts, and explanations.

Keep technical terms, code, file paths, and subagent prompts in English. Translate only user-facing output.

Make sure that the directories exist:
```bash
mkdir -p .planning/todos/pending .planning/todos/completed
```

Note the existing areas from the todos array. Use them for consistency in the infer_area step.
</step>

<step name="extract_content">
**With arguments:** Use the arguments as the title/focus.
- `/gsd-add-todo Add auth token refresh` → title = "Add auth token refresh"

**Without arguments:** Examine the recent conversation. Extract these items:
- The specific problem, idea, or task that the conversation discussed
- Relevant file paths that the conversation mentioned
- Technical details (error messages, line numbers, constraints)

Formulate:
- `title`: A descriptive title of 3-10 words (an action verb is preferred)
- `problem`: What is wrong or why this is needed
- `solution`: Approach hints, or "TBD" if this is only an idea
- `files`: Relevant paths with line numbers from the conversation
</step>

<step name="infer_area">
Infer area from file paths:

| Path pattern | Area |
|--------------|------|
| `src/api/*`, `api/*` | `api` |
| `src/components/*`, `src/ui/*` | `ui` |
| `src/auth/*`, `auth/*` | `auth` |
| `src/db/*`, `database/*` | `database` |
| `tests/*`, `__tests__/*` | `testing` |
| `docs/*` | `docs` |
| `.planning/*` | `planning` |
| `scripts/*`, `bin/*` | `tooling` |
| No files or unclear | `general` |

If a similar match exists, use the existing area from step 2.
</step>

<step name="infer_severity">
Infer a **suggested** severity from the blocker/major/minor/cosmetic taxonomy that `verify-work.md`'s `severity_inference` uses. Before you write, CONFIRM the severity with the user. Never auto-assign it without confirmation.

An incorrect severity corrupts backlog triage. The field gives a signal for backlog triage.

Suggest from the user's natural-language description:

| User says | Suggest |
|-----------|---------|
| "crashes", "error", "exception", "fails completely", "data loss" | blocker |
| "does not work", "nothing happens", "wrong behavior" | major |
| "works but...", "slow", "weird", "minor issue" | minor |
| "color", "spacing", "alignment", "looks off" | cosmetic |

If the severity is unclear, set the suggestion to **major**.

**Text mode (`workflow.text_mode: true` in config or `--text` flag):** If `$ARGUMENTS` contains `--text` OR `text_mode` from init JSON is `true`, set `TEXT_MODE=true`. When TEXT_MODE is active, replace the `AskUserQuestion` below with a plain-text numbered list of the four options. Ask the user to type their choice number.

This is necessary for non-Claude runtimes (OpenAI Codex, Antigravity, etc.) where `AskUserQuestion` is unavailable.

Confirm with AskUserQuestion (present the suggested value first):
- header: "Severity?"
- question: "Suggested severity: [suggested]. Confirm or change:"
- options:
  - "blocker" — breaks a workflow or loses data. Fix first
  - "major" — wrong behavior with no workaround
  - "minor" — works, but with a workaround or annoyance
  - "cosmetic" — visual/polish only

Use the confirmed value for `severity` in the create_file frontmatter.
</step>

<step name="check_duplicates">
```bash
# Search for key words from title in existing todos
grep -l -i "[key words from title]" .planning/todos/pending/*.md 2>/dev/null || true
```

If you find a potential duplicate:
1. Read the existing todo.
2. Compare the scope.


**Text mode (`workflow.text_mode: true` in config or `--text` flag):** If `$ARGUMENTS` contains `--text` OR `text_mode` from init JSON is `true`, set `TEXT_MODE=true`. When TEXT_MODE is active, replace every `AskUserQuestion` call with a plain-text numbered list. Ask the user to type their choice number.

This is necessary for non-Claude runtimes (OpenAI Codex, Antigravity, etc.) where `AskUserQuestion` is not available.

If the scope overlaps, use AskUserQuestion:
- header: "Duplicate?"
- question: "Similar todo exists: [title]. What would you like to do?"
- options:
  - "Skip" — keep existing todo
  - "Replace" — update existing with new context
  - "Add anyway" — create as separate todo
</step>

<step name="create_file">
Use values from init context: `timestamp` and `date` are already available.

Generate a slug for the title:
```bash
slug=$(gsd_run query generate-slug "$title" --raw)
```

Write to `.planning/todos/pending/${date}-${slug}.md`:

```markdown
---
created: [timestamp]
title: [title]
area: [area]
severity: [blocker|major|minor|cosmetic — confirmed in infer_severity step]
files:
  - [file:lines]
---

## Problem

[problem description - enough context for future Claude to understand weeks later]

## Solution

[approach hints or "TBD"]
```
</step>

<step name="update_state">
If `.planning/STATE.md` exists:

1. Re-run `gsd_run query init.todos ${GSD_WS:+--ws=${GSD_WS##* }}` with the same workstream flag as the first call. Get a fresh JSON snapshot after the write. The count and rendering must reflect the change you just made.

2. **Fail-safe check (#2618):** If the JSON failed to parse, do NOT touch the "### Pending Todos" section. If `pending_read_ok` is not `true`, do NOT touch the section. If `pending_todos_markdown` is not a string, do NOT touch the section. In these cases, leave the section exactly as-is and continue to the next step. A partial or malformed `init.todos` result must never overwrite a good existing section.

3. Otherwise, replace the entire body of "### Pending Todos" under "## Accumulated Context" with the literal value of `pending_todos_markdown`. Use the value verbatim. `init.todos` already renders one bullet per pending todo and caps the length.

   Do not reformat, re-wrap, re-order, or hand-edit the bullets. Do not append to the old body. Replace the entire body. This replaces an old run-on-sentence section without a migration step.
</step>

<step name="git_commit">
Commit the todo and any updated state:

```bash
gsd_run query commit "docs: capture todo - [title]" --files .planning/todos/pending/[filename] .planning/STATE.md
```

The tool automatically respects `commit_docs` config and gitignore.

Confirm: "Committed: docs: capture todo - [title]"
</step>

<step name="confirm">
```
Todo saved: .planning/todos/pending/[filename]

  [title]
  Area: [area]
  Files: [count] referenced

---

Would you like to:

1. Continue with current work
2. Add another todo
3. View all todos (/gsd:capture --list)
```
</step>

</process>

<success_criteria>
- [ ] Directory structure exists
- [ ] Todo file exists with valid frontmatter
- [ ] Problem section has enough context for future Claude
- [ ] No duplicates (examined and resolved)
- [ ] Area is consistent with existing todos
- [ ] If STATE.md exists, it reflects the update
- [ ] Todo and state are committed to git
</success_criteria>
