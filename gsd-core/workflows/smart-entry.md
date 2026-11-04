<purpose>
GSD smart entry is a state-aware launcher. Detect the current project situation with `gsd_run smart-entry --json`. Show a short menu of the applicable next actions. Dispatch to exactly one existing GSD command. This is a launcher/router only. It never does the work itself.

This is a *menu* launcher, not a second router. For in-project progress (planning → executing → verify-pending), the recommended action is `/gsd:progress --next`. That action delegates to the single gated advancement engine (`workflows/next.md`: Route 0 resume-incomplete-phase + Gates 1-3).

smart-entry adds value only where `--next` cannot reach. These situations are pre-project, remediation (paused/blocked/verify-failed), and lifecycle exits (idle-stranded/complete). See `docs/adr/1787-gsd-next-smart-entry.md`.
</purpose>

<required_reading>
Before you start, read all files that the invoking prompt's `execution_context` references.
</required_reading>

<process>

<step name="text_mode">
**TEXT_MODE handling (non-Claude runtimes).**

If `--text` is present in `$ARGUMENTS` OR `text_mode` from init JSON is `true`, set `TEXT_MODE=true`. When TEXT_MODE is active, replace every `AskUserQuestion` call with a plain-text numbered list. Ask the user to type their choice number. Non-Claude runtimes, such as OpenAI Codex and Antigravity, need this because `AskUserQuestion` is not available.
</step>

<step name="resolve">
**Resolve the gsd_run shim.**

Run this resolver block exactly. It finds `gsd-tools.cjs` across every supported runtime home and defines a `gsd_run` function. If it cannot find the tool, it prints the standard install hint and exits non-zero.

```bash
```
</step>

<step name="detect">
**Detect the situation.**

```bash
_GSD_SHIM_NAME="gsd-tools.cjs"; _GSD_RUNTIME_ROOT="${RUNTIME_DIR:-$(git rev-parse --show-toplevel 2>/dev/null || pwd)}"; GSD_TOOLS="${_GSD_RUNTIME_ROOT}/gsd-core/bin/${_GSD_SHIM_NAME}"; _gsd_at() { for _p; do if [ -f "$_p" ]; then GSD_TOOLS="$_p"; return 0; fi; done; return 1; }; _gsd_id_ok() { case "$("$1" runtime-identity --raw 2>/dev/null || true)" in '{"packageName":"@opengsd/gsd-core"'*'}') return 0;; *) return 1;; esac; }; _gsd_homes() { set -- "${CLAUDE_CONFIG_DIR:-$HOME/.claude}" "${ANTIGRAVITY_CONFIG_DIR:-$HOME/.gemini/antigravity}" "$HOME/.gemini/antigravity-ide" "$HOME/.gemini/antigravity-cli" "${AUGMENT_CONFIG_DIR:-$HOME/.augment}" "${CLINE_CONFIG_DIR:-$HOME/.cline}" "${CODEBUDDY_CONFIG_DIR:-$HOME/.codebuddy}" "${CODEX_HOME:-$HOME/.codex}" "${COPILOT_CONFIG_DIR:-${COPILOT_HOME:-$HOME/.copilot}}" "${CURSOR_CONFIG_DIR:-$HOME/.cursor}" "${HERMES_HOME:-$HOME/.hermes}" "${KILO_CONFIG_DIR:-${XDG_CONFIG_HOME:-$HOME/.config}/kilo}" "${KIMI_CONFIG_DIR:-$HOME/.config/agents}" "$HOME/.agents" "${KIMI_CODE_HOME:-$HOME/.kimi-code}" "${OPENCODE_CONFIG_DIR:-${XDG_CONFIG_HOME:-$HOME/.config}/opencode}" "${PI_CODING_AGENT_DIR:-$HOME/.pi/agent}" "${QWEN_CONFIG_DIR:-$HOME/.qwen}" "${TRAE_CONFIG_DIR:-$HOME/.trae}" "${WINDSURF_CONFIG_DIR:-$HOME/.codeium/windsurf}" "${ZCODE_CONFIG_DIR:-$HOME/.zcode}" "${GROK_AGENTS_HOME:-$HOME/.agents}"; for _h; do _gsd_at "$_h/gsd-core/bin/${_GSD_SHIM_NAME}" && return 0; done; return 1; }; if _gsd_at "${_GSD_RUNTIME_ROOT}/gsd-core/bin/${_GSD_SHIM_NAME}" "${_GSD_RUNTIME_ROOT}/.claude/gsd-core/bin/${_GSD_SHIM_NAME}" "${_GSD_RUNTIME_ROOT}/.codex/gsd-core/bin/${_GSD_SHIM_NAME}"; then gsd_run() { node "$GSD_TOOLS" "$@"; }; elif _gsd_homes; then gsd_run() { node "$GSD_TOOLS" "$@"; }; elif unset -f gsd_run; _G="$(command -v gsd_run)"; [ -n "$_G" ] && _gsd_id_ok "$_G"; then GSD_TOOLS="$_G"; gsd_run() { "$GSD_TOOLS" "$@"; }; else echo "ERROR: gsd-tools.cjs not found at $GSD_TOOLS and no identity-proving gsd_run is on PATH. Run: npx -y @opengsd/gsd-core@latest --claude --local" >&2; exit 1; fi; GSD_IDENTITY_STATUS=unverified; _gsd_id_ok gsd_run && GSD_IDENTITY_STATUS=ok; export GSD_IDENTITY_STATUS; [ "$GSD_IDENTITY_STATUS" = ok ] || echo "WARNING: \"$GSD_TOOLS\" did not prove it is @opengsd/gsd-core - it is either a different package or an @opengsd/gsd-core older than the runtime-identity verb. See docs/how-to/diagnose-a-foreign-gsd-tools.md" >&2; if [ -n "${CLAUDE_ENV_FILE:-}" ] && [ -n "${GSD_TOOLS:-}" ]; then printf "export PATH='%s':\"\$PATH\"\n" "${GSD_TOOLS%/*}" >> "$CLAUDE_ENV_FILE" 2>/dev/null || true; fi
RESPONSE_LANGUAGE=$(gsd_run query config-get response_language --raw --default "" 2>/dev/null || echo "")
SNAPSHOT=$(gsd_run smart-entry --json 2>/dev/null)
```

**If `response_language` is set:** You MUST show all user-facing output of this workflow in `{response_language}`. This output includes narration between tool calls, status updates, progress notes, findings, questions, prompts, and explanations.

Keep technical terms, code, file paths, and subagent prompts in English. Translate only user-facing output.

Parse `SNAPSHOT` as JSON. It has the shape:

```json
{
  "situation": "executing",
  "recommended": "progress-next",
  "summary": "Phase 2 of 5 · 60% · executing",
  "signals": { "...": "..." },
  "actions": [
    { "id": "progress-next", "label": "Advance to the next step", "command": "/gsd:progress --next", "recommended": true },
    { "id": "execute-phase", "label": "Continue executing phase 2", "command": "/gsd:execute-phase", "recommended": false }
  ]
}
```

`situation` is one of: `no-project`, `paused`, `blocked`, `verify-failed`, `needs-first-phase`, `planning`, `executing`, `verify-pending`, `idle-stranded`, `complete`, `unknown`.

**Fallback (never strand the user):** `smart-entry --json` can fail for two reasons. Each reason has a different recovery. Parse `SNAPSHOT`. If it is empty, not valid JSON, or missing `actions`, apply the first matching recovery below. Do NOT error.

1. **`gsd-tools` itself is broken** (the failure is a `Cannot find module ...` / Node crash, not only an empty result). Run `gsd_run state-snapshot` to probe the tool layer. If THAT also errors, the whole tool layer is down. Routing to `/gsd:progress` would also fail because it needs gsd-tools. **Recover by reading state directly:**
   - Read `.planning/STATE.md` (frontmatter + body) with the Read tool. Extract `status` from frontmatter `status:` or body `**Status:**`. Extract `Phase:` from the body. If a nested `progress:` frontmatter object is present, extract `total_phases`/`percent` from it. Extract any `## Blockers` items.
   - Synthesize a minimal result. Set `situation` to your best guess from the status text (`executing`/`verifying`/`planning`/`complete`/`paused`). Set `summary` to a one-line read ("Phase N of M · status"). Build an `actions` list from status. For example, use verifying → `/gsd:verify-work`, executing → `/gsd:execute-phase`, else `/gsd:progress`. Always include `/gsd:quick` and `/gsd:help`.
   - Print one line first: `smart-entry unavailable (gsd-tools error) — reading state directly. The gsd-tools layer may need a rebuild (rm tsconfig.build.tsbuildinfo && npm run build).`
   - Proceed to the `present` step with this synthesized result.

2. **Only `smart-entry` is unavailable** (for example, older gsd-core without the subcommand. `state-snapshot` still works). Print one line first: `smart-entry unavailable — showing progress.` Run `/gsd:progress` and stop.
</step>

<step name="present">
**Present the menu.**

Show the `summary` line to orient the user. Then offer the actions.

**If TEXT_MODE is false:** call `AskUserQuestion` with:
- `header`: a short label derived from `situation`. For example, `executing` → "Continue work", `blocked` → "Unblock", `no-project` → "Get started", `complete` → "What next?".
- `question`: the `summary` line, then "What would you like to do?"
- `options`: the first 4 entries of `actions[]` in order. For each, set `label` to the action's `label` and `description` to the action's `command`. The recommended action is already first. Show it as the first option. The user may also type a custom command, which the system handles automatically.

**If TEXT_MODE is true:** print the `summary`. Then print a numbered list of ALL `actions[]` (not capped to 4 — text has no limit). Then ask the user to type the number of their choice:

```
{summary}

  1. {actions[0].label}  ({actions[0].command})
  2. {actions[1].label}  ({actions[1].command})
  ...

Type a number, or describe what you want to do.
```

Wait for the user's response before you continue. Map the chosen number to the corresponding action.
</step>

<step name="display">
**Show the routing decision.**

```
### GSD ► SMART ENTRY

**Situation:** {situation}
**Routing to:** {chosen command}
```
</step>

<step name="dispatch">
**Dispatch and stop.**

Invoke the chosen action's `command`. If the user typed a free-form response instead of an action, treat it as freeform intent. Route it via `/gsd:progress --do "<their text>"`.

After you invoke the command, **stop**. The dispatched command owns everything from here. Do not continue. Do not chain. Do not re-enter this workflow.
</step>

</process>

<success_criteria>
- [ ] Situation detected via `gsd_run smart-entry --json`
- [ ] Summary shown to orient the user
- [ ] Menu offered (AskUserQuestion, or numbered list under TEXT_MODE)
- [ ] Routing decision shown before dispatch
- [ ] Exactly one command dispatched
- [ ] Any detection failure falls back to /gsd:progress (never strands the user)
- [ ] No work done directly — launcher only
</success_criteria>
