@~/.claude/gsd-core/references/response-language-directive.md

<purpose>
This workflow audits all UAT and verification files across phases. It finds every outstanding item (pending, skipped, blocked, human_needed). It optionally compares the items with the codebase to detect stale docs and produces a prioritized human test plan.
</purpose>

<process>

<step name="initialize">
Run the CLI audit:

```bash
_GSD_SHIM_NAME="gsd-tools.cjs"; _GSD_RUNTIME_ROOT="${RUNTIME_DIR:-$(git rev-parse --show-toplevel 2>/dev/null || pwd)}"; GSD_TOOLS="${_GSD_RUNTIME_ROOT}/gsd-core/bin/${_GSD_SHIM_NAME}"; _gsd_at() { for _p; do if [ -f "$_p" ]; then GSD_TOOLS="$_p"; return 0; fi; done; return 1; }; _gsd_id_ok() { case "$("$1" runtime-identity --raw 2>/dev/null || true)" in '{"packageName":"@opengsd/gsd-core"'*'}') return 0;; *) return 1;; esac; }; _gsd_homes() { set -- "${CLAUDE_CONFIG_DIR:-$HOME/.claude}" "${ANTIGRAVITY_CONFIG_DIR:-$HOME/.gemini/antigravity}" "$HOME/.gemini/antigravity-ide" "$HOME/.gemini/antigravity-cli" "${AUGMENT_CONFIG_DIR:-$HOME/.augment}" "${CLINE_CONFIG_DIR:-$HOME/.cline}" "${CODEBUDDY_CONFIG_DIR:-$HOME/.codebuddy}" "${CODEX_HOME:-$HOME/.codex}" "${COPILOT_CONFIG_DIR:-${COPILOT_HOME:-$HOME/.copilot}}" "${CURSOR_CONFIG_DIR:-$HOME/.cursor}" "${HERMES_HOME:-$HOME/.hermes}" "${KILO_CONFIG_DIR:-${XDG_CONFIG_HOME:-$HOME/.config}/kilo}" "${KIMI_CONFIG_DIR:-$HOME/.config/agents}" "$HOME/.agents" "${KIMI_CODE_HOME:-$HOME/.kimi-code}" "${OPENCODE_CONFIG_DIR:-${XDG_CONFIG_HOME:-$HOME/.config}/opencode}" "${PI_CODING_AGENT_DIR:-$HOME/.pi/agent}" "${QWEN_CONFIG_DIR:-$HOME/.qwen}" "${TRAE_CONFIG_DIR:-$HOME/.trae}" "${WINDSURF_CONFIG_DIR:-$HOME/.codeium/windsurf}" "${ZCODE_CONFIG_DIR:-$HOME/.zcode}" "${GROK_AGENTS_HOME:-$HOME/.agents}"; for _h; do _gsd_at "$_h/gsd-core/bin/${_GSD_SHIM_NAME}" && return 0; done; return 1; }; if _gsd_at "${_GSD_RUNTIME_ROOT}/gsd-core/bin/${_GSD_SHIM_NAME}" "${_GSD_RUNTIME_ROOT}/.claude/gsd-core/bin/${_GSD_SHIM_NAME}" "${_GSD_RUNTIME_ROOT}/.codex/gsd-core/bin/${_GSD_SHIM_NAME}"; then gsd_run() { node "$GSD_TOOLS" "$@"; }; elif _gsd_homes; then gsd_run() { node "$GSD_TOOLS" "$@"; }; elif unset -f gsd_run; _G="$(command -v gsd_run)"; [ -n "$_G" ] && _gsd_id_ok "$_G"; then GSD_TOOLS="$_G"; gsd_run() { "$GSD_TOOLS" "$@"; }; else echo "ERROR: gsd-tools.cjs not found at $GSD_TOOLS and no identity-proving gsd_run is on PATH. Run: npx -y @opengsd/gsd-core@latest --claude --local" >&2; exit 1; fi; GSD_IDENTITY_STATUS=unverified; _gsd_id_ok gsd_run && GSD_IDENTITY_STATUS=ok; export GSD_IDENTITY_STATUS; [ "$GSD_IDENTITY_STATUS" = ok ] || echo "WARNING: \"$GSD_TOOLS\" did not prove it is @opengsd/gsd-core - it is either a different package or an @opengsd/gsd-core older than the runtime-identity verb. See docs/how-to/diagnose-a-foreign-gsd-tools.md" >&2; if [ -n "${CLAUDE_ENV_FILE:-}" ] && [ -n "${GSD_TOOLS:-}" ]; then printf "export PATH='%s':\"\$PATH\"\n" "${GSD_TOOLS%/*}" >> "$CLAUDE_ENV_FILE" 2>/dev/null || true; fi
AUDIT=$(gsd_run query audit-uat --raw)
```

Parse the JSON for the `results` array and the `summary` object.

If `summary.total_items` is 0 AND `summary.parse_gap_files` is 0:
```
## All Clear

No outstanding UAT or verification items found across all phases.
All tests are passing, resolved, or diagnosed with fix plans.
```
Stop here.

If `summary.total_items` is 0 but `summary.parse_gap_files` is greater than 0, this is NOT all-clear. One or more files have test blocks that the audit could not parse. These blocks may hide outstanding work.

Continue to the `categorize` and `present` steps below. Even when `items` is empty for a mixed project, `present` renders the Unparsed section from these `parse_gap: true` entries.
</step>

<step name="categorize">
Group the items by what is actionable NOW and what needs prerequisites:

**Testable Now** (no external dependencies):
- `pending` — tests that never ran
- `human_uat` — items for human verification
- `skipped_unresolved` — skipped with no clear blocking reason

**Needs Prerequisites:**
- `server_blocked` — needs a running external server
- `device_needed` — needs a physical device (not a simulator)
- `build_needed` — needs a release/preview build
- `third_party` — needs configuration for an external service

For each item in "Testable Now", use Grep/Read to make sure that the underlying feature still exists in the codebase:
- If the test references a component/function that no longer exists, mark the item as `stale`
- If the test references code that was significantly rewritten, mark the item as `needs_update`
- Otherwise, mark the item as `active`
</step>

<step name="present">
If `summary.parse_gap_files` is greater than 0, render this section FIRST. Render it before the `## UAT Audit Report` below. If `total_items` is also 0, render it standalone.

Filter `results` for entries with `parse_gap: true`:
```
## Unparsed UAT Files ({parse_gap_files} files)

| Phase | File | Unparsed Blocks |
|-------|------|------------------|
| {phase} | {file_path} | {unparsed_blocks} |
...

These files have `### N.` test blocks with no readable `result:` line. Fix the file's structure, then re-run the audit.
```
This section fires for ANY project with `parse_gap_files > 0`. The two sections are not mutually exclusive. A mixed project can also have real outstanding `items` in other phases.

Even if its phase belongs to an already-archived milestone, an outstanding item still matters. A deferred human-UAT scenario or a `skipped` live-stack test is exactly the kind of item that gets archived still-open. The audit lists the parse gap of an archived phase in this same table, with no filter on `archived_milestone`.

If `total_items` is 0, stop after this section. There is no `## UAT Audit Report` to present. Otherwise, continue below.

Present the audit report:

```
## UAT Audit Report

**{total_items} outstanding items across {total_files} files in {phase_count} phases**

### Testable Now ({count})

| # | Phase | Test | Description | Status |
|---|-------|------|-------------|--------|
| 1 | {phase} | {test_name} | {expected} | {active/stale/needs_update} |
...

### Needs Prerequisites ({count})

| # | Phase | Test | Blocked By | Description |
|---|-------|------|------------|-------------|
| 1 | {phase} | {test_name} | {category} | {expected} |
...

### Stale (can be closed) ({count})

| # | Phase | Test | Why Stale |
|---|-------|------|-----------|
| 1 | {phase} | {test_name} | {reason} |
...

---

## Recommended Actions

1. **Close stale items:** `/gsd:verify-work {phase}` — mark stale tests as resolved
2. **Run active tests:** Human UAT test plan below
3. **When prerequisites met:** Retest blocked items with `/gsd:verify-work {phase}`
```
</step>

<step name="test_plan">
Generate a human UAT test plan for "Testable Now" + "active" items only:

Group the items by what you can test together (same screen, same feature, same prerequisite):

```
## Human UAT Test Plan

### Group 1: {category — e.g., "Billing Flow"}
Prerequisites: {what needs to be running/configured}

1. **{Test name}** (Phase {N})
   - Navigate to: {where}
   - Do: {action}
   - Expected: {expected behavior}

2. **{Test name}** (Phase {N})
   ...

### Group 2: {category}
...
```
</step>

</process>
