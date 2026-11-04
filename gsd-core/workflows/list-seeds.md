@~/.claude/gsd-core/references/response-language-directive.md

<purpose>
List captured seeds for browsing and audit. You can use an optional status filter.

The workflow is read-only and never changes seeds.
</purpose>

<required_reading>
Before you start, read all files that the invoking prompt's execution_context references.
</required_reading>

<process>

<step name="load_seeds">
Load seed context. An optional status filter may follow `--list-seeds`.

Examples: `dormant`, `active`, and `triggered`.

```bash
_GSD_SHIM_NAME="gsd-tools.cjs"; _GSD_RUNTIME_ROOT="${RUNTIME_DIR:-$(git rev-parse --show-toplevel 2>/dev/null || pwd)}"; GSD_TOOLS="${_GSD_RUNTIME_ROOT}/gsd-core/bin/${_GSD_SHIM_NAME}"; _gsd_at() { for _p; do if [ -f "$_p" ]; then GSD_TOOLS="$_p"; return 0; fi; done; return 1; }; _gsd_id_ok() { case "$("$1" runtime-identity --raw 2>/dev/null || true)" in '{"packageName":"@opengsd/gsd-core"'*'}') return 0;; *) return 1;; esac; }; _gsd_homes() { set -- "${CLAUDE_CONFIG_DIR:-$HOME/.claude}" "${ANTIGRAVITY_CONFIG_DIR:-$HOME/.gemini/antigravity}" "$HOME/.gemini/antigravity-ide" "$HOME/.gemini/antigravity-cli" "${AUGMENT_CONFIG_DIR:-$HOME/.augment}" "${CLINE_CONFIG_DIR:-$HOME/.cline}" "${CODEBUDDY_CONFIG_DIR:-$HOME/.codebuddy}" "${CODEX_HOME:-$HOME/.codex}" "${COPILOT_CONFIG_DIR:-${COPILOT_HOME:-$HOME/.copilot}}" "${CURSOR_CONFIG_DIR:-$HOME/.cursor}" "${HERMES_HOME:-$HOME/.hermes}" "${KILO_CONFIG_DIR:-${XDG_CONFIG_HOME:-$HOME/.config}/kilo}" "${KIMI_CONFIG_DIR:-$HOME/.config/agents}" "$HOME/.agents" "${KIMI_CODE_HOME:-$HOME/.kimi-code}" "${OPENCODE_CONFIG_DIR:-${XDG_CONFIG_HOME:-$HOME/.config}/opencode}" "${PI_CODING_AGENT_DIR:-$HOME/.pi/agent}" "${QWEN_CONFIG_DIR:-$HOME/.qwen}" "${TRAE_CONFIG_DIR:-$HOME/.trae}" "${WINDSURF_CONFIG_DIR:-$HOME/.codeium/windsurf}" "${ZCODE_CONFIG_DIR:-$HOME/.zcode}" "${GROK_AGENTS_HOME:-$HOME/.agents}"; for _h; do _gsd_at "$_h/gsd-core/bin/${_GSD_SHIM_NAME}" && return 0; done; return 1; }; if _gsd_at "${_GSD_RUNTIME_ROOT}/gsd-core/bin/${_GSD_SHIM_NAME}" "${_GSD_RUNTIME_ROOT}/.claude/gsd-core/bin/${_GSD_SHIM_NAME}" "${_GSD_RUNTIME_ROOT}/.codex/gsd-core/bin/${_GSD_SHIM_NAME}"; then gsd_run() { node "$GSD_TOOLS" "$@"; }; elif _gsd_homes; then gsd_run() { node "$GSD_TOOLS" "$@"; }; elif unset -f gsd_run; _G="$(command -v gsd_run)"; [ -n "$_G" ] && _gsd_id_ok "$_G"; then GSD_TOOLS="$_G"; gsd_run() { "$GSD_TOOLS" "$@"; }; else echo "ERROR: gsd-tools.cjs not found at $GSD_TOOLS and no identity-proving gsd_run is on PATH. Run: npx -y @opengsd/gsd-core@latest --claude --local" >&2; exit 1; fi; GSD_IDENTITY_STATUS=unverified; _gsd_id_ok gsd_run && GSD_IDENTITY_STATUS=ok; export GSD_IDENTITY_STATUS; [ "$GSD_IDENTITY_STATUS" = ok ] || echo "WARNING: \"$GSD_TOOLS\" did not prove it is @opengsd/gsd-core - it is either a different package or an @opengsd/gsd-core older than the runtime-identity verb. See docs/how-to/diagnose-a-foreign-gsd-tools.md" >&2; if [ -n "${CLAUDE_ENV_FILE:-}" ] && [ -n "${GSD_TOOLS:-}" ]; then printf "export PATH='%s':\"\$PATH\"\n" "${GSD_TOOLS%/*}" >> "$CLAUDE_ENV_FILE" 2>/dev/null || true; fi
SEEDS=$(gsd_run list-seeds "$STATUS_FILTER")
if [[ "$SEEDS" == @file:* ]]; then SEEDS=$(cat "${SEEDS#@file:}"); fi
```

If `$ARGUMENTS` includes a filter token, replace `$STATUS_FILTER` with that token. Otherwise, omit it.

Extract these items from the JSON:
- `count`
- `seeds[]`: Each seed has `seed_id`, `status`, `scope`, `trigger_when`, `planted`, and `title`.
- `summary`: A `{ status: count }` map.
</step>

<step name="empty_case">
If `count` is 0:
```
No seeds found.

Plant one with /gsd:capture --seed "<forward-looking idea>".
```
If a status filter was given and nothing matched, say: `No seeds with status "<filter>".`

Exit.
</step>

<step name="render_table">
Show the seeds as a table sorted by `seed_id`. The tool already sorts the seeds.

Truncate `trigger_when` and `title` to keep the table readable.

```
Seeds

---
ID        Status     Scope    Trigger                  Title
SEED-001  dormant    large    when websockets land     Real-time collaboration
SEED-006  triggered  medium   MILE-04 planning         Remove legacy auth crates

---
<count> seeds  (<summary rendered as "N status" pairs, e.g. "1 dormant, 1 triggered">)
```

Then offer next actions as plain text. Make no changes to seeds here:
```
- /gsd:capture --seed --enrich <ID>   enrich a seed with trigger, why, and scope
- /gsd:capture --list-seeds <status>  filter by status
```
</step>

</process>

<success_criteria>
- [ ] The list shows ID, status, scope, trigger, and title for each seed
- [ ] If a status filter was provided, the list uses it
- [ ] The empty / no-match case includes guidance
- [ ] The summary line shows total and per-status counts
- [ ] The workflow did not change seed files (read-only)
</success_criteria>
