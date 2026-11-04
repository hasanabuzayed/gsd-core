@~/.claude/gsd-core/references/response-language-directive.md

# Add Backlog Item Workflow

`/gsd:capture --backlog` (`commands/gsd/capture.md`) starts this workflow.

This workflow adds an idea to the ROADMAP.md backlog parking lot. It uses 999.x numbering.

Backlog items are ideas without a sequence that are not ready for active planning.
They are outside the normal phase sequence and collect context over time.

<process>

## Step 1: Read ROADMAP.md

Find existing backlog entries:

```bash
cat .planning/ROADMAP.md
```

## Step 2: Find next backlog number

```bash
_GSD_SHIM_NAME="gsd-tools.cjs"; _GSD_RUNTIME_ROOT="${RUNTIME_DIR:-$(git rev-parse --show-toplevel 2>/dev/null || pwd)}"; GSD_TOOLS="${_GSD_RUNTIME_ROOT}/gsd-core/bin/${_GSD_SHIM_NAME}"; _gsd_at() { for _p; do if [ -f "$_p" ]; then GSD_TOOLS="$_p"; return 0; fi; done; return 1; }; _gsd_id_ok() { case "$("$1" runtime-identity --raw 2>/dev/null || true)" in '{"packageName":"@opengsd/gsd-core"'*'}') return 0;; *) return 1;; esac; }; _gsd_homes() { set -- "${CLAUDE_CONFIG_DIR:-$HOME/.claude}" "${ANTIGRAVITY_CONFIG_DIR:-$HOME/.gemini/antigravity}" "$HOME/.gemini/antigravity-ide" "$HOME/.gemini/antigravity-cli" "${AUGMENT_CONFIG_DIR:-$HOME/.augment}" "${CLINE_CONFIG_DIR:-$HOME/.cline}" "${CODEBUDDY_CONFIG_DIR:-$HOME/.codebuddy}" "${CODEX_HOME:-$HOME/.codex}" "${COPILOT_CONFIG_DIR:-${COPILOT_HOME:-$HOME/.copilot}}" "${CURSOR_CONFIG_DIR:-$HOME/.cursor}" "${HERMES_HOME:-$HOME/.hermes}" "${KILO_CONFIG_DIR:-${XDG_CONFIG_HOME:-$HOME/.config}/kilo}" "${KIMI_CONFIG_DIR:-$HOME/.config/agents}" "$HOME/.agents" "${KIMI_CODE_HOME:-$HOME/.kimi-code}" "${OPENCODE_CONFIG_DIR:-${XDG_CONFIG_HOME:-$HOME/.config}/opencode}" "${PI_CODING_AGENT_DIR:-$HOME/.pi/agent}" "${QWEN_CONFIG_DIR:-$HOME/.qwen}" "${TRAE_CONFIG_DIR:-$HOME/.trae}" "${WINDSURF_CONFIG_DIR:-$HOME/.codeium/windsurf}" "${ZCODE_CONFIG_DIR:-$HOME/.zcode}" "${GROK_AGENTS_HOME:-$HOME/.agents}"; for _h; do _gsd_at "$_h/gsd-core/bin/${_GSD_SHIM_NAME}" && return 0; done; return 1; }; if _gsd_at "${_GSD_RUNTIME_ROOT}/gsd-core/bin/${_GSD_SHIM_NAME}" "${_GSD_RUNTIME_ROOT}/.claude/gsd-core/bin/${_GSD_SHIM_NAME}" "${_GSD_RUNTIME_ROOT}/.codex/gsd-core/bin/${_GSD_SHIM_NAME}"; then gsd_run() { node "$GSD_TOOLS" "$@"; }; elif _gsd_homes; then gsd_run() { node "$GSD_TOOLS" "$@"; }; elif unset -f gsd_run; _G="$(command -v gsd_run)"; [ -n "$_G" ] && _gsd_id_ok "$_G"; then GSD_TOOLS="$_G"; gsd_run() { "$GSD_TOOLS" "$@"; }; else echo "ERROR: gsd-tools.cjs not found at $GSD_TOOLS and no identity-proving gsd_run is on PATH. Run: npx -y @opengsd/gsd-core@latest --claude --local" >&2; exit 1; fi; GSD_IDENTITY_STATUS=unverified; _gsd_id_ok gsd_run && GSD_IDENTITY_STATUS=ok; export GSD_IDENTITY_STATUS; [ "$GSD_IDENTITY_STATUS" = ok ] || echo "WARNING: \"$GSD_TOOLS\" did not prove it is @opengsd/gsd-core - it is either a different package or an @opengsd/gsd-core older than the runtime-identity verb. See docs/how-to/diagnose-a-foreign-gsd-tools.md" >&2; if [ -n "${CLAUDE_ENV_FILE:-}" ] && [ -n "${GSD_TOOLS:-}" ]; then printf "export PATH='%s':\"\$PATH\"\n" "${GSD_TOOLS%/*}" >> "$CLAUDE_ENV_FILE" 2>/dev/null || true; fi
NEXT=$(gsd_run query phase.next-decimal 999 --raw)
```

If no 999.x phases exist yet, `phase.next-decimal` returns `999.1`.

Sparse numbering is acceptable, for example 999.1, 999.3.

Always use `phase.next-decimal` and never guess.

## Step 3: Write ROADMAP entry

**Write the ROADMAP entry BEFORE you create the directory.**

The directory shows reliably that the phase is already registered.
This prevents false duplicate detection in any hook that checks for existing 999.x directories (#2280).

Add the entry under a `## Backlog` section.
If the section does not exist, create it at the end of ROADMAP.md:

```markdown
## Backlog

### Phase {NEXT}: {description} (BACKLOG)

**Goal:** [Captured for future planning]
**Requirements:** TBD
**Plans:** 0 plans

Plans:
- [ ] TBD (promote with /gsd:review-backlog when ready)
```

## Step 4: Create the phase directory

If `.planning/config.json` sets the `project_code` prefix, apply it.

This makes the backlog directory name consistent with all other phase-creation paths:

```bash
IFS= read -rd '' GSD_ARGS <<'GSD_ARGS_END' || true
$ARGUMENTS
GSD_ARGS_END
GSD_ARGS=${GSD_ARGS%$'\n'}
SLUG=$(gsd_run query generate-slug "$GSD_ARGS" --raw)
PROJECT_CODE=$(gsd_run query config-get project_code --raw 2>/dev/null || echo "")
PREFIX=$([ -n "$PROJECT_CODE" ] && echo "${PROJECT_CODE}-" || echo "")
PHASE_DIR=".planning/phases/${PREFIX}${NEXT}-${SLUG}"
mkdir -p "${PHASE_DIR}"
touch "${PHASE_DIR}/.gitkeep"
```

## Step 5: Commit

```bash
IFS= read -rd '' GSD_ARGS <<'GSD_ARGS_END' || true
$ARGUMENTS
GSD_ARGS_END
GSD_ARGS=${GSD_ARGS%$'\n'}
gsd_run query commit "docs: add backlog item ${NEXT} — ${GSD_ARGS}" --files .planning/ROADMAP.md "${PHASE_DIR}/.gitkeep"
```

## Step 6: Report

```
## 📋 Backlog Item Added

Phase {NEXT}: {description}
Directory: {PHASE_DIR}/

This item lives in the backlog parking lot.
Use /gsd:discuss-phase {NEXT} to explore it further.
Use /gsd:review-backlog to promote items to active milestone.
```

</process>

<notes>
- 999.x numbering keeps backlog items outside the active phase sequence.
- The workflow creates phase directories immediately so /gsd:discuss-phase and /gsd:plan-phase work on them.
- There is no `Depends on:` field. Backlog items have no sequence by definition.
- Sparse numbering is acceptable (999.1, 999.3). The workflow always uses next-decimal.
- Use /gsd:review-backlog to promote backlog items to the active milestone.
</notes>
