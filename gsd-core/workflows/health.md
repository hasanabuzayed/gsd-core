<purpose>
Examine the integrity of the `.planning/` directory and report problems that need action. The examination finds missing files, invalid configurations, inconsistent state, and orphaned plans. It can also repair problems that support automatic repair.
</purpose>

<required_reading>
Before starting, read all files that the invoking prompt's execution_context references.
</required_reading>

<process>
```bash
_GSD_SHIM_NAME="gsd-tools.cjs"; _GSD_RUNTIME_ROOT="${RUNTIME_DIR:-$(git rev-parse --show-toplevel 2>/dev/null || pwd)}"; GSD_TOOLS="${_GSD_RUNTIME_ROOT}/gsd-core/bin/${_GSD_SHIM_NAME}"; _gsd_at() { for _p; do if [ -f "$_p" ]; then GSD_TOOLS="$_p"; return 0; fi; done; return 1; }; _gsd_id_ok() { case "$("$1" runtime-identity --raw 2>/dev/null || true)" in '{"packageName":"@opengsd/gsd-core"'*'}') return 0;; *) return 1;; esac; }; _gsd_homes() { set -- "${CLAUDE_CONFIG_DIR:-$HOME/.claude}" "${ANTIGRAVITY_CONFIG_DIR:-$HOME/.gemini/antigravity}" "$HOME/.gemini/antigravity-ide" "$HOME/.gemini/antigravity-cli" "${AUGMENT_CONFIG_DIR:-$HOME/.augment}" "${CLINE_CONFIG_DIR:-$HOME/.cline}" "${CODEBUDDY_CONFIG_DIR:-$HOME/.codebuddy}" "${CODEX_HOME:-$HOME/.codex}" "${COPILOT_CONFIG_DIR:-${COPILOT_HOME:-$HOME/.copilot}}" "${CURSOR_CONFIG_DIR:-$HOME/.cursor}" "${HERMES_HOME:-$HOME/.hermes}" "${KILO_CONFIG_DIR:-${XDG_CONFIG_HOME:-$HOME/.config}/kilo}" "${KIMI_CONFIG_DIR:-$HOME/.config/agents}" "$HOME/.agents" "${KIMI_CODE_HOME:-$HOME/.kimi-code}" "${OPENCODE_CONFIG_DIR:-${XDG_CONFIG_HOME:-$HOME/.config}/opencode}" "${PI_CODING_AGENT_DIR:-$HOME/.pi/agent}" "${QWEN_CONFIG_DIR:-$HOME/.qwen}" "${TRAE_CONFIG_DIR:-$HOME/.trae}" "${WINDSURF_CONFIG_DIR:-$HOME/.codeium/windsurf}" "${ZCODE_CONFIG_DIR:-$HOME/.zcode}" "${GROK_AGENTS_HOME:-$HOME/.agents}"; for _h; do _gsd_at "$_h/gsd-core/bin/${_GSD_SHIM_NAME}" && return 0; done; return 1; }; if _gsd_at "${_GSD_RUNTIME_ROOT}/gsd-core/bin/${_GSD_SHIM_NAME}" "${_GSD_RUNTIME_ROOT}/.claude/gsd-core/bin/${_GSD_SHIM_NAME}" "${_GSD_RUNTIME_ROOT}/.codex/gsd-core/bin/${_GSD_SHIM_NAME}"; then gsd_run() { node "$GSD_TOOLS" "$@"; }; elif _gsd_homes; then gsd_run() { node "$GSD_TOOLS" "$@"; }; elif unset -f gsd_run; _G="$(command -v gsd_run)"; [ -n "$_G" ] && _gsd_id_ok "$_G"; then GSD_TOOLS="$_G"; gsd_run() { "$GSD_TOOLS" "$@"; }; else echo "ERROR: gsd-tools.cjs not found at $GSD_TOOLS and no identity-proving gsd_run is on PATH. Run: npx -y @opengsd/gsd-core@latest --claude --local" >&2; exit 1; fi; GSD_IDENTITY_STATUS=unverified; _gsd_id_ok gsd_run && GSD_IDENTITY_STATUS=ok; export GSD_IDENTITY_STATUS; [ "$GSD_IDENTITY_STATUS" = ok ] || echo "WARNING: \"$GSD_TOOLS\" did not prove it is @opengsd/gsd-core - it is either a different package or an @opengsd/gsd-core older than the runtime-identity verb. See docs/how-to/diagnose-a-foreign-gsd-tools.md" >&2; if [ -n "${CLAUDE_ENV_FILE:-}" ] && [ -n "${GSD_TOOLS:-}" ]; then printf "export PATH='%s':\"\$PATH\"\n" "${GSD_TOOLS%/*}" >> "$CLAUDE_ENV_FILE" 2>/dev/null || true; fi
RESPONSE_LANGUAGE=$(gsd_run query config-get response_language --raw --default "" 2>/dev/null || echo "")
```

**If `response_language` is set:** You MUST use `{response_language}` for all user-facing output of this workflow. This output includes narration between tool calls, status updates, progress notes, findings, questions, prompts, and explanations.

Keep technical terms, code, file paths, and subagent prompts in English. Translate only user-facing output.

<step name="parse_args">
**Parse arguments:**

Find if the command arguments contain the `--repair`, `--backfill`, or `--context` flags.

```
REPAIR_FLAG=""
BACKFILL_FLAG=""
CONTEXT_MODE=""
if arguments contain "--repair"; then
  REPAIR_FLAG="--repair"
fi
if arguments contain "--backfill"; then
  BACKFILL_FLAG="--backfill"
fi
if arguments contain "--context"; then
  CONTEXT_MODE="true"
fi
```

If `CONTEXT_MODE` is set, go to the `context_check` step and skip the integrity validation steps. The two modes are independent. Context use does not relate to the health of the `.planning/` directory.
</step>

<step name="context_check">
**Run only when `--context` is set.**

The model that runs this workflow reports the current session's approximate `tokensUsed` and the active model's `contextWindow`. Use the values that your runtime shows.

These values come from Claude Code's `/context` slash command output or the model's own session telemetry. If the runtime shows neither value, use AskUserQuestion to ask the user once for both numbers.

**TEXT_MODE fallback:** When `text_mode` is true (config or `--text` flag), the runtime is non-Claude. `AskUserQuestion` is not available. Examples of non-Claude runtimes include Codex and Antigravity.

Replace the prompt with a plain-text two-question sequence ("Approximate tokens used? Context window size?"). Read the answers as plain text from the user's response.

```bash
gsd_run query validate.context \
  --tokens-used "$TOKENS_USED" \
  --context-window "$CONTEXT_WINDOW"
```

The query prints a one-line status (`Context utilization: NN% (state)`). For the warning and critical states, it also prints a recommendation line.

Print the SDK output verbatim and end the workflow. Do **not** include `.planning/` health output. The two modes are independent diagnostics.
</step>

<step name="run_health_check">
**Run health validation:**

```bash
gsd_run query validate.health $REPAIR_FLAG $BACKFILL_FLAG
```

Parse JSON output:
- `status`: "healthy" | "degraded" | "broken"
- `errors[]`: Critical issues (code, message, fix, repairable)
- `warnings[]`: Non-critical issues
- `info[]`: Informational notes
- `repairable_count`: Number of auto-fixable issues
- `repairs_performed[]`: Actions taken if --repair was used

**Isolation/worktrees compatibility check (#2486):** The SDK is runtime-neutral. This check runs here and uses the same negotiation as the execution-workflow guards.

It finds a config with an explicit `workflow.use_worktrees: true` on a runtime that declares `dispatch.isolation` as `none`. For example, a worktree-capable install that shares the repo can supply this config. `/gsd:execute-phase` and `/gsd:quick` fail closed on that value.

The gate uses the **declared capability, not the runtime name** (#2584). The resolver fail-closes unknown/undeclared/`undocumented` values to `none`.

Use `inspect-dispatch-isolation`, the **sentinel-free** inspection verb. Never use `dispatch-isolation`. Its #3045 contract records the resolved decision to the executor-dispatch sentinel as an unconditional side effect.

A read-only diagnostic must not be able to stamp a sentinel that the isolation guards then enforce against real dispatches. "Sentinel-free" is the precise claim and the only one this check depends on.

Like every `gsd-tools` invocation, inspection still runs the shared CLI bootstrap. The bootstrap may self-heal a stale `.planning/active-workstream` pointer or rebuild compiled modules. These actions already exist, do not depend on the verb, and cannot block a dispatch. Writing the sentinel can block a dispatch.

The worktrees read deliberately has no `--default`/fallback, as in `settings.md`. W025 claims that the config **sets** the key to a non-false value. An absent key (empty output) must remain distinguishable from an explicit `true`.

A `|| echo "true"` fallback removes that distinction. It makes the check fire on a config that never set the key. This is correct only on an emit where `_stampNonClaudeRuntimeDefaults` rewrote the line to `--default false`.

It is wrong on the un-stamped source/Claude emit. The `[ -n … ]` guard below removes the dependency on stamping (#2486 review, Major 1).

A failed query is **not** a capability verdict. `|| echo "none"` would give the same value for "this runtime declares no primitive" and "the query did not answer". W025's text asserts the former.

This tells a Claude Code user that their runtime declares no executor-isolation primitive, which is false. Track the two cases separately and report which one occurred.

Note the limit of that signal (#2486 review). The verb itself fail-closes an unknown, undeclared, or out-of-vocabulary runtime to `none` and exits 0. It does the same for any internal resolution error.

So `INSPECTED_RESOLVED=true` means "the query answered", not "the runtime published a declaration". W025's resolved-case wording below is true for every path that reaches it. This includes a declared `none` and an internally fail-closed `none`.

Distinguishing those two cases would need provenance from the verb. That is out of scope here.

```bash
_INSPECTED_RAW=$(gsd_run query inspect-dispatch-isolation --raw 2>/dev/null)
_ISOLATION_RC=$?
if [ $_ISOLATION_RC -ne 0 ] || [ -z "$_INSPECTED_RAW" ]; then
  INSPECTED_ISOLATION=none
  INSPECTED_RESOLVED=false      # no verdict learned — not the same as "declares none"
else
  INSPECTED_ISOLATION="$_INSPECTED_RAW"
  INSPECTED_RESOLVED=true
fi
case "$INSPECTED_ISOLATION" in
  harness-worktree|orchestrator-worktree|none) ;;
  *) INSPECTED_ISOLATION=none; INSPECTED_RESOLVED=false ;;   # out of vocabulary is not a verdict either
esac

USE_WORKTREES=$(gsd_run query config-get workflow.use_worktrees --raw 2>/dev/null)
if [ "$INSPECTED_ISOLATION" = "none" ] && [ -n "$USE_WORKTREES" ] && [ "$USE_WORKTREES" != "false" ]; then
  if [ "$INSPECTED_RESOLVED" = "true" ]; then
    echo "W025: the project config sets workflow.use_worktrees to a non-false value, but this runtime has no usable executor-isolation primitive — dispatch.isolation resolves to none, declared as none, or fail-closed because the capability could not be determined — so /gsd:execute-phase and /gsd:quick will fail closed. Fix: run /gsd:settings and answer No to Worktrees, or set workflow.use_worktrees: false in the project config (the active workstream's config.json when one is active). Set it explicitly rather than deleting the key — an absent key resolves to false only on an emit whose default was stamped to false, and to true otherwise."
  else
    echo "W025: the project config sets workflow.use_worktrees to a non-false value, and GSD could not resolve this runtime's executor-isolation capability ('gsd_run query inspect-dispatch-isolation' failed or returned nothing) — so it cannot tell whether /gsd:execute-phase and /gsd:quick will fail closed on that value. This is a report of an unverifiable config, NOT a finding that the runtime declares no primitive. Fix: re-run once the gsd-tools shim resolves; if the warning persists, run /gsd:settings and answer No to Worktrees."
  fi
fi
```

If the check prints, append it to the Warnings section of the report as `[W025]` with the printed fix. Include it in the warning count that the report shows. If `validate.health` returned `healthy`, report `Status: DEGRADED`.

A config that makes the execution workflows fail closed is not a healthy planning state. The problem does not support automatic repair. An explicit `true` may be intentional for a worktree-capable install that shares the same `.planning/config.json`. The user decides on the remedy (#2486).
</step>

<step name="format_output">
**Format and show results:**

```
### GSD Health Check

Status: HEALTHY | DEGRADED | BROKEN
Errors: N | Warnings: N | Info: N
```

**If repairs were done:**
```
## Repairs Performed

- ✓ config.json: Created with defaults
- ✓ STATE.md: Regenerated from roadmap
```

**If errors exist:**
```
## Errors

- [E001] config.json: JSON parse error at line 5
  Fix: Run /gsd:health --repair to reset to defaults

- [E002] PROJECT.md not found
  Fix: Run /gsd:new-project to create
```

**If warnings exist:**
```
## Warnings

- [W002] STATE.md references phase 5, but only phases 1-3 exist
  Fix: Review STATE.md manually before changing it; repair will not overwrite an existing STATE.md

- [W005] Phase directory "1-setup" doesn't follow NN-name format
  Fix: Rename to match pattern (e.g., 01-setup)
```

**If info exists:**
```
## Info

- [I001] 02-implementation/02-01-PLAN.md has no SUMMARY.md
  Note: May be in progress
```

**Footer (if repairable issues exist and --repair was NOT used):**
```
---
N issues can be auto-repaired. Run: /gsd:health --repair
```
</step>

<step name="offer_repair">
**If repairable issues exist and --repair was NOT used:**

Ask the user if they want to run repairs:

```
Would you like to run /gsd:health --repair to fix N issues automatically?
```

If the user says yes, re-run with the --repair flag and show the results.
</step>

<step name="verify_repairs">
**If repairs were done:**

Re-run the health check without --repair to make sure that the problems are resolved:

```bash
gsd_run query validate.health
```

Report final status.
</step>

</process>

<error_codes>
| Code | Severity | Description | Repairable |
|------|----------|-------------|------------|
| E001 | error | .planning/ directory not found | No |
| E002 | error | PROJECT.md not found | No |
| E003 | error | ROADMAP.md not found | No |
| E004 | error | STATE.md not found | No |
| E005 | error | config.json parse error | No |
| E010 | error | CWD resolves to the user's home directory — health check would target the wrong .planning/ | No |
| W001 | warning | PROJECT.md missing need section | No |
| W002 | warning | STATE.md references invalid phase | No |
| W003 | warning | config.json not found | Yes |
| W004 | warning | config.json invalid field value | No |
| W005 | warning | Phase directory naming mismatch | No |
| W006 | warning | Phase in ROADMAP but no directory | No |
| W007 | warning | Phase on disk but not in ROADMAP | No |
| W008 | warning | config.json: workflow.nyquist_validation absent (defaults to enabled but agents may skip) | Yes |
| W009 | warning | Phase has Validation Architecture in RESEARCH.md but no VALIDATION.md | No |
| W010 | warning | GSD agent installation missing or incomplete | No |
| W011 | warning | STATE.md current-phase status disagrees with ROADMAP.md checkbox | No |
| W012 | warning | config.json invalid branching_strategy value | No |
| W013 | warning | config.json context_window not a positive integer | No |
| W014 | warning | config.json phase_branch_template missing {phase} placeholder | No |
| W015 | warning | config.json milestone_branch_template missing {milestone} placeholder | No |
| W016 | warning | config.json: workflow.ai_integration_phase absent (defaults to enabled but agents may skip AI-integration-phase planning) | Yes |
| W017 | warning | Orphan git worktree (path no longer exists on disk) | No |
| W018 | warning | MILESTONES.md missing entry for archived milestone snapshot | Yes (`--backfill`) |
| W019 | warning | Unrecognized .planning/ root file — not a canonical GSD artifact | No |
| W020 | warning | Worktree health scan degraded — git worktree list timed out, failed, or a finding could not be verified | No |
| W021 | warning | Phase's integer prefix implies a different milestone than its ROADMAP section (phase_id_convention: milestone-prefixed) | No |
| W022 | warning | config.json models entry malformed (unknown phase type, invalid tier, or non-object value) | No |
| W023 | warning | Phase directories collide on normalized key | No |
| W024 | warning | STATE.md was written many commits ago — treat its contents as approximate | No |
| W026 | warning | STATE says milestone complete but ROADMAP lists an unstarted phase for that milestone | No |
| W027 | warning | Stale git worktree (not changed in a long time) | No |
| W028 | warning | A GSD-owned install scope shadows another on this machine | No |
| W029 | warning | .planning/ matches a gitignore rule but is still tracked by git (gitignore has no effect on already-tracked files) | No |
| W030 | warning | Phase verification report status is outside the closed set | Yes (known legacy statuses only) |
| I001 | info | Plan without SUMMARY (may be in progress) | No |
| I010 | info | Resolved CWD reported alongside the E010 home-directory guard | No |

Note: this table is **generated**. Do not hand-edit it. `node scripts/gen-health-docs.cjs --write` produces it from the `RULES` table in `src/health-diagnostic.cts` and 3 pre-checks.

The table has 31 rules as of #3309. Each `Rule` entry has a static `description`/`repairable`. See `src/health-diagnostic-types.cts`.

The pre-checks stay outside the rule table by design. They are `E001`, `E010`, and `I010`. They are safety rails in `cmdValidateHealth`, `src/verify.cts`, and never `.planning/` findings.

`scripts/lint-health-diagnostic-rule-table.cjs` already enforces the 1:1 code invariant that this table depends on. There are no duplicate codes. Severity is always a `Rule` property and is never set per emit call.

Before you assign a new code, add a `Rule` entry under `src/health-diagnostic-rules/`. Its `code` is the next free number that the lint guard has not yet seen. Run `node scripts/gen-health-docs.cjs --write` to regenerate this table. If the table drifts, `npm run lint:generated-sync` fails.

`W025` is the `workflow.use_worktrees`/`dispatch.isolation` check (#2486). It is a workflow-layer diagnostic that this file's own `run_health_check` step emits directly, not `cmdValidateHealth`. Its documentation stays in that step, not in this generated table.

</error_codes>

<repair_actions>
| Action | Effect | Risk |
|--------|--------|------|
| createConfig | Create config.json with defaults | None |
| resetConfig | Delete + recreate config.json | Loses custom settings |
| regenerateState | Create STATE.md from ROADMAP structure when it is missing | Loses session history |
| addNyquistKey | Add workflow.nyquist_validation: true to config.json | None — matches existing default |
| addAiIntegrationPhaseKey | Add workflow.ai_integration_phase: true to config.json | None — matches existing default |
| backfillMilestones | Synthesize missing MILESTONES.md entries from `.planning/milestones/vX.Y-ROADMAP.md` snapshots | None — additive only. Triggered by `--backfill` flag |
| repairVerificationStatus | Rewrite a known legacy verification `status` (`pass_with_caveats`) to `passed`, keeping the original as `legacy_status` | None — only the `status` line changes. Unknown statuses are never touched |

**Not repairable (too risky):**
- PROJECT.md, ROADMAP.md content
- Phase directory renaming
- Orphaned plan cleanup

</repair_actions>

<stale_task_cleanup>
**Windows-specific:** Examine Claude Code task directories for stale directories that accumulate on crash/freeze. When subagents are force-killed, these directories remain and consume disk space.

When `--repair` is active, find and remove them:

```bash
# Check for stale task directories (older than 24 hours)
TASKS_DIR="$HOME/.claude/tasks"
if [ -d "$TASKS_DIR" ]; then
  STALE_COUNT=$( (find "$TASKS_DIR" -maxdepth 1 -type d -mtime +1 2>/dev/null || true) | wc -l )
  if [ "$STALE_COUNT" -gt 0 ]; then
    echo "⚠️  Found $STALE_COUNT stale task directories in ~/.claude/tasks/"
    echo "   These are leftover from crashed subagent sessions."
    echo "   Run: rm -rf ~/.claude/tasks/*  (safe — only affects dead sessions)"
  fi
fi
```

Report as info diagnostic: `I002 | info | Stale subagent task directories found | Yes (--repair removes them)`
</stale_task_cleanup>
