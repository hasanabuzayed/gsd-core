@~/.claude/gsd-core/references/response-language-directive.md

<purpose>
Detect the current project state. Automatically advance to the next logical GSD workflow step.
Read the project state to determine the progression: discuss → plan → execute → verify → complete.
</purpose>

<required_reading>
Before you start, read all files that the invoking prompt's execution_context references.
</required_reading>

<process>

<step name="detect_state">
Read the project state to determine the current position:

```bash
_GSD_SHIM_NAME="gsd-tools.cjs"; _GSD_RUNTIME_ROOT="${RUNTIME_DIR:-$(git rev-parse --show-toplevel 2>/dev/null || pwd)}"; GSD_TOOLS="${_GSD_RUNTIME_ROOT}/gsd-core/bin/${_GSD_SHIM_NAME}"; _gsd_at() { for _p; do if [ -f "$_p" ]; then GSD_TOOLS="$_p"; return 0; fi; done; return 1; }; _gsd_id_ok() { case "$("$1" runtime-identity --raw 2>/dev/null || true)" in '{"packageName":"@opengsd/gsd-core"'*'}') return 0;; *) return 1;; esac; }; _gsd_homes() { set -- "${CLAUDE_CONFIG_DIR:-$HOME/.claude}" "${ANTIGRAVITY_CONFIG_DIR:-$HOME/.gemini/antigravity}" "$HOME/.gemini/antigravity-ide" "$HOME/.gemini/antigravity-cli" "${AUGMENT_CONFIG_DIR:-$HOME/.augment}" "${CLINE_CONFIG_DIR:-$HOME/.cline}" "${CODEBUDDY_CONFIG_DIR:-$HOME/.codebuddy}" "${CODEX_HOME:-$HOME/.codex}" "${COPILOT_CONFIG_DIR:-${COPILOT_HOME:-$HOME/.copilot}}" "${CURSOR_CONFIG_DIR:-$HOME/.cursor}" "${HERMES_HOME:-$HOME/.hermes}" "${KILO_CONFIG_DIR:-${XDG_CONFIG_HOME:-$HOME/.config}/kilo}" "${KIMI_CONFIG_DIR:-$HOME/.config/agents}" "$HOME/.agents" "${KIMI_CODE_HOME:-$HOME/.kimi-code}" "${OPENCODE_CONFIG_DIR:-${XDG_CONFIG_HOME:-$HOME/.config}/opencode}" "${PI_CODING_AGENT_DIR:-$HOME/.pi/agent}" "${QWEN_CONFIG_DIR:-$HOME/.qwen}" "${TRAE_CONFIG_DIR:-$HOME/.trae}" "${WINDSURF_CONFIG_DIR:-$HOME/.codeium/windsurf}" "${ZCODE_CONFIG_DIR:-$HOME/.zcode}" "${GROK_AGENTS_HOME:-$HOME/.agents}"; for _h; do _gsd_at "$_h/gsd-core/bin/${_GSD_SHIM_NAME}" && return 0; done; return 1; }; if _gsd_at "${_GSD_RUNTIME_ROOT}/gsd-core/bin/${_GSD_SHIM_NAME}" "${_GSD_RUNTIME_ROOT}/.claude/gsd-core/bin/${_GSD_SHIM_NAME}" "${_GSD_RUNTIME_ROOT}/.codex/gsd-core/bin/${_GSD_SHIM_NAME}"; then gsd_run() { node "$GSD_TOOLS" "$@"; }; elif _gsd_homes; then gsd_run() { node "$GSD_TOOLS" "$@"; }; elif unset -f gsd_run; _G="$(command -v gsd_run)"; [ -n "$_G" ] && _gsd_id_ok "$_G"; then GSD_TOOLS="$_G"; gsd_run() { "$GSD_TOOLS" "$@"; }; else echo "ERROR: gsd-tools.cjs not found at $GSD_TOOLS and no identity-proving gsd_run is on PATH. Run: npx -y @opengsd/gsd-core@latest --claude --local" >&2; exit 1; fi; GSD_IDENTITY_STATUS=unverified; _gsd_id_ok gsd_run && GSD_IDENTITY_STATUS=ok; export GSD_IDENTITY_STATUS; [ "$GSD_IDENTITY_STATUS" = ok ] || echo "WARNING: \"$GSD_TOOLS\" did not prove it is @opengsd/gsd-core - it is either a different package or an @opengsd/gsd-core older than the runtime-identity verb. See docs/how-to/diagnose-a-foreign-gsd-tools.md" >&2; if [ -n "${CLAUDE_ENV_FILE:-}" ] && [ -n "${GSD_TOOLS:-}" ]; then printf "export PATH='%s':\"\$PATH\"\n" "${GSD_TOOLS%/*}" >> "$CLAUDE_ENV_FILE" 2>/dev/null || true; fi
# Get state snapshot
gsd_run query state.json 2>/dev/null || echo "{}"
```

Also read:
- `.planning/STATE.md` — current phase, progress, plan counts
- `.planning/ROADMAP.md` — milestone structure and phase list

Extract:
- `current_phase` — which phase is active
- `plan_of` / `plans_total` — plan execution progress
- `progress` — overall percentage
- `status` — active, paused, or another status.

If no `.planning/` directory exists:
```
No GSD project detected. Run `/gsd:new-project` to get started.
```
Exit.
</step>

<step name="safety_gates">
Before routing, run hard-stop checks. Unless the user passed `--force`, exit on the first hit.

If the user passed the `--force` flag, skip all gates, Route 0, and the prior-phase completeness prompt.
Print a one-line warning: `⚠ --force: skipping safety gates`
Then proceed directly to `determine_next_action`.
Under `--force`, the workflow does not reach Route 0 or `prior_phase_completeness`.

**Gate 1: Unresolved checkpoint**
Determine whether `.planning/.continue-here.md` exists:
```bash
[ -f .planning/.continue-here.md ]
```
If found:
```
⛔ Hard stop: Unresolved checkpoint

`.planning/.continue-here.md` exists — a previous session left
unfinished work that needs manual review before advancing.

Read the file, resolve the issue, then delete it to continue.
Use `--force` to bypass this check.
```
Exit (do not route).

**Gate 2: Error state**
Determine whether STATE.md contains `status: error` or `status: failed`:
If found:
```
⛔ Hard stop: Project in error state

STATE.md shows status: {status}. Resolve the error before advancing.
Run `/gsd:health` to diagnose, or manually fix STATE.md.
Use `--force` to bypass this check.
```
Exit.

**Gate 3: Unchecked verification**
Determine whether the current phase has a VERIFICATION.md with any `FAIL` items without overrides:
If found:
```
⛔ Hard stop: Unchecked verification failures

VERIFICATION.md for phase {N} has {count} unresolved FAIL items.
Address the failures or add overrides before advancing to the next phase.
Use `--force` to bypass this check.
```
Exit.

After all three hard-stop gates pass, continue to `resume_incomplete_phase`.
</step>

<step name="resume_incomplete_phase">
**Hard invariant:** A phase can have PLAN.md files without matching SUMMARY.md files. The phase must be completed before `/gsd:progress --next` routes to any forward action.

This gate catches a common failure. A session can stop during execution because of a hang, token exhaustion, or an API connection drop. STATE.md's `current_phase` can advance past the phase with unfinished work.

Without this gate, `/gsd:progress --next` would route by `current_phase` and silently skip the partially executed phase.

**If the user passed `--no-resume`, skip this step** and continue to `prior_phase_completeness`.
At `safety_gates`, `--force` already bypassed all gates and Route 0. The workflow does not reach this step under `--force`.

**Why Route 0 runs here (after Gates 1-3, before the prior-phase defer prompt):** This hard invariant does not depend on the value of `current_phase`. It must run before any routing rule that reads `current_phase`. Gates 1-3 are low-cost checks of repo/state validity. They must always run. If the workflow skips them on the resume path, it risks advancing a project with a broken state.

In the default case (no flag), Route 0 automatically resumes the phase. The prior-phase completeness-scan DEFER PROMPT must NOT run in this case. The prompt would force a double decision: prompt first, then resume anyway. This would override the user's choice.

In this position, Route 0 resumes silently by default, without a defer prompt. `--no-resume` skips Route 0 and continues to the prior-phase defer prompt in `prior_phase_completeness`. At `safety_gates`, `--force` goes directly to `determine_next_action`. It never reaches Route 0 or `prior_phase_completeness`.

Scan ALL phases in ROADMAP order, from lowest-numbered to highest, for incomplete execution. Use `gsd_run query roadmap.analyze` to get the phase list.

For each phase number `N`, query `gsd_run query find-phase <N>` JSON. Inspect its `plans` and `summaries` arrays. A phase has **incomplete-execution** when `plans.length > summaries.length` (at least one PLAN.md has no matching SUMMARY.md).

Stop at the first such phase. Record its phase number as `INCOMPLETE_PHASE`. This is the lowest-numbered phase that needs continued execution.

Illustrative bash:

```bash
INCOMPLETE_PHASE=""
ROADMAP_JSON=$(gsd_run query roadmap.analyze)
ROADMAP_SCOPE=$(echo "$ROADMAP_JSON" | jq -r '.scope // "complete"')
if [ $? -ne 0 ] || [ -z "$ROADMAP_JSON" ]; then
  echo "⚠ WARNING: resume-incomplete-phase scan could not run (roadmap.analyze failed)." >&2
  echo "  The incomplete-phase invariant (#160) could not be verified." >&2
  echo "  Proceeding to prior-phase completeness check — review project state carefully." >&2
  # Fall through to prior_phase_completeness rather than silently skipping
elif [ "$ROADMAP_SCOPE" != "complete" ]; then
  # #3184/#3165: roadmap.analyze succeeded and returned a well-formed document,
  # but its milestone window did not see all of its input, so `.phases[]` is a
  # NON-answer rather than a real empty. Looping it would run the invariant over
  # a phase list the scan could not populate and report "clean" — the silent
  # disarm #3165 reports. Treated as scan-failed, same as an outright failure.
  echo "⚠ WARNING: resume-incomplete-phase scan could not be scoped (roadmap.analyze scope: $ROADMAP_SCOPE)." >&2
  echo "  The milestone window did not cover the whole ROADMAP, so the phase list is incomplete." >&2
  echo "  The incomplete-phase invariant (#160) could not be verified — review project state carefully." >&2
  # Fall through to prior_phase_completeness rather than silently passing
else
  for PHASE_NUM in $(echo "$ROADMAP_JSON" | jq -r '.phases[] | (.number // .phase_number // empty)'); do
    PHASE_JSON=$(gsd_run query find-phase "$PHASE_NUM")
    if [ $? -ne 0 ] || [ -z "$PHASE_JSON" ]; then
      echo "⚠ WARNING: Could not query phase $PHASE_NUM — skipping in resume scan." >&2
      continue
    fi
    PLAN_COUNT=$(echo "$PHASE_JSON" | jq '(.plans // []) | length')
    SUMMARY_COUNT=$(echo "$PHASE_JSON" | jq '(.summaries // []) | length')
    if [ "${PLAN_COUNT:-0}" -gt "${SUMMARY_COUNT:-0}" ]; then
      INCOMPLETE_PHASE="$PHASE_NUM"
      break
    fi
  done
fi
```

**If `INCOMPLETE_PHASE` is non-empty:** route to `/gsd:execute-phase $INCOMPLETE_PHASE` and exit. Before you invoke it, show a one-line notice:

```
▶ Resuming incomplete Phase ${INCOMPLETE_PHASE} (plans without summaries detected)
  /gsd:execute-phase ${INCOMPLETE_PHASE}
  (use --no-resume to skip this check and defer via the prior-phase prompt)
```

Then invoke it via SlashCommand. Do not continue to the next steps.

**If `INCOMPLETE_PHASE` is empty:** continue to `prior_phase_completeness`.
</step>

<step name="prior_phase_completeness">
**Prior-phase completeness scan:** This scan runs if the user passed `--no-resume` and skipped Route 0. It also runs if Route 0 found no incomplete-execution phases in the default case.

Under `--force`, the workflow does NOT reach this scan. At `safety_gates`, that flag goes directly to `determine_next_action`.

Scan all phases before the current phase in ROADMAP.md order for incomplete work. For each before phase number `N`, use `gsd_run query find-phase <N>` JSON to inspect that phase. The JSON includes plans, summaries, incomplete_plans, and other fields.

Detect three categories of incomplete work:
1. **Plans without summaries** — a PLAN.md exists in a before phase directory, but no matching SUMMARY.md exists. Execution started but did not complete.
2. **Verification failures not overridden** — a before phase has a VERIFICATION.md with `FAIL` items without an override annotation.
3. **CONTEXT.md without plans** — a before phase directory has a CONTEXT.md but no PLAN.md files. Discussion happened, but planning never ran.

If you find no incomplete before work, continue to `determine_next_action` silently without interruption.

If you find incomplete before work, show a structured completeness report:
```
⚠ Prior phase has incomplete work

Phase {N} — "{name}" has unresolved items:
  • Plan {N}-{M} ({slug}): executed but no SUMMARY.md
  [... additional items ...]

Advancing before resolving these may cause:
  • Verification gaps — future phase verification won't have visibility into what prior phases shipped
  • Context loss — plans that ran without summaries leave no record for future agents

Options:
  [C] Continue and defer these items to backlog
  [S] Stop and resolve manually (recommended)
  [F] Force advance without recording deferral

Choice [S]:
```

**If the user chooses "Stop" (S or Enter/default):** Exit without routing.

**If the user chooses "Continue and defer" (C):**
1. For each incomplete item, create a backlog entry in `ROADMAP.md` under `## Backlog`. Use the existing `999.x` numbering scheme:
```markdown
### Phase 999.{N}: Follow-up — Phase {src} incomplete plans (BACKLOG)

**Goal:** Resolve plans that ran without producing summaries during Phase {src} execution
**Source phase:** {src}
**Deferred at:** {date} during /gsd:progress --next advancement to Phase {dest}
**Plans:**
- [ ] {N}-{M}: {slug} (ran, no SUMMARY.md)
```
2. Commit the deferral record:
```bash
gsd_run query commit "docs: defer incomplete Phase {src} items to backlog" \
  --files .planning/ROADMAP.md
```
3. Continue routing to `determine_next_action` immediately — no second prompt.

**If the user chooses "Force" (F):** Continue to `determine_next_action` without recording deferral.
</step>

<step name="spike_sketch_notice">
Examine pending spike/sketch work and show a notice. This does not change routing:

```bash
# Check for pending spikes (verdict: PENDING in any README)
PENDING_SPIKES=$(grep -rl 'verdict: PENDING' .planning/spikes/*/README.md 2>/dev/null | wc -l | tr -d ' ')

# Check for pending sketches (winner: null in any README)
PENDING_SKETCHES=$(grep -rl 'winner: null' .planning/sketches/*/README.md 2>/dev/null | wc -l | tr -d ' ')
```

If either count is > 0, show this notice before routing:
```
⚠ Pending exploratory work:
  {PENDING_SPIKES} spike(s) with unresolved verdicts in .planning/spikes/
  {PENDING_SKETCHES} sketch(es) without a winning variant in .planning/sketches/

  Resume with `/gsd:spike` or `/gsd:sketch`, or continue with phase work below.
```

Show lines only for non-zero counts. If both are 0, skip this notice entirely.
</step>

<step name="determine_next_action">
Apply routing rules based on the state:

**Route 1: No phases exist yet → discuss**
If ROADMAP has phases but no phase directories exist on disk:
→ Next action: `/gsd:discuss-phase <first-phase>`

**Route 2: Phase exists but has no CONTEXT.md or RESEARCH.md → discuss**
If the current phase directory exists but has neither CONTEXT.md nor RESEARCH.md:
→ Next action: `/gsd:discuss-phase <current-phase>`

**Route 3: Phase has context but no plans → plan**
If the current phase has CONTEXT.md (or RESEARCH.md) but no PLAN.md files:
→ Next action: `/gsd:plan-phase <current-phase>` (or `/gsd:plan-review-convergence <current-phase>` when `PLAN_STRATEGY=converge`)

**Route 4: Phase has plans but incomplete summaries → execute**
If plans exist but not all have matching summaries:
→ Next action: `/gsd:execute-phase <current-phase>`

**Route 5: All plans have summaries → verify and complete**
If all plans in the current phase have summaries:
→ Next action: `/gsd:verify-work`

**Route 6: Phase complete, next phase exists → advance**
If the current phase is complete and the next phase exists in ROADMAP:
→ Next action: `/gsd:discuss-phase <next-phase>`

**Route 7: All phases complete → complete milestone**
If all phases are complete:
→ Next action: `/gsd:complete-milestone`

**Route 8: Paused → resume**
If STATE.md shows paused_at:
→ Next action: `/gsd:resume-work`
</step>

<step name="show_and_execute">
Parse the arguments passed to this workflow to detect the plan strategy. Build convergence pass-through args:

```bash
IFS= read -rd '' GSD_ARGS <<'GSD_ARGS_END' || true
$ARGUMENTS
GSD_ARGS_END
GSD_ARGS=${GSD_ARGS%$'\n'}
PLAN_STRATEGY="local"
if echo "$GSD_ARGS" | grep -qE '(^|[[:space:]])\-\-(converge|cross-ai)([[:space:]]|$)'; then
  PLAN_STRATEGY="converge"
fi

CONVERGENCE_ARGS=""
# Lane flags derived from the declared roster (#2800/#2272); --all and --text are convergence
# controls, not reviewer lanes, so they stay literal.
for REVIEW_FLAG in $(gsd_run review-lane flags) --all --text; do
  if echo "$GSD_ARGS" | grep -qE "(^|[[:space:]])${REVIEW_FLAG}([[:space:]]|$)"; then
    CONVERGENCE_ARGS="${CONVERGENCE_ARGS} ${REVIEW_FLAG}"
  fi
done

MAX_CYCLES_ARG=""
if echo "$GSD_ARGS" | grep -qE '\-\-max-cycles\s+[0-9]+'; then
  MAX_CYCLES_ARG=$(echo "$GSD_ARGS" | grep -oE '\-\-max-cycles\s+[0-9]+' | awk '{print $2}')
  CONVERGENCE_ARGS="${CONVERGENCE_ARGS} --max-cycles ${MAX_CYCLES_ARG}"
fi
```

If `PLAN_STRATEGY` is `converge`, fail fast unless the convergence feature gate is enabled:

```bash
if [ "$PLAN_STRATEGY" = "converge" ]; then
  CONVERGENCE_ENABLED=$(gsd_run query config-get workflow.plan_review_convergence --raw 2>/dev/null || echo "false")
  if [ "$CONVERGENCE_ENABLED" != "true" ]; then
    printf '%s\n' \
      '/gsd:progress --next --converge is disabled (workflow.plan_review_convergence=false).' \
      '' \
      'Enable plan convergence with:' \
      '' \
      '  gsd config-set workflow.plan_review_convergence true' \
      '' \
      'Then re-run with --converge.'
    exit 1
  fi
fi
```

Show the determination:

```
## GSD Next

**Current:** Phase [N] — [name] | [progress]%
**Status:** [status description]

▶ **Next step:** `/gsd-[command] [args]`
  [One-line explanation of why this is the next step]
```

Then immediately invoke the determined command via SlashCommand.
Do not ask for confirmation. The purpose of `/gsd:progress --next` is advancement without interruption.

**Route 3 convergence override:** If the routing decision is Route 3 (plan) and `PLAN_STRATEGY=converge`, invoke `/gsd:plan-review-convergence <current-phase> ${CONVERGENCE_ARGS}` instead of `/gsd:plan-phase <current-phase>`.

**If the user passed `--auto`:** after the determined command completes, automatically re-invoke `/gsd:progress --next --auto`. Forward `--converge`/`--cross-ai` and any reviewer flags that the user originally passed.

This continues chaining to the next step. Repeat until one of:
- A milestone completes (`/gsd:complete-milestone` is reached)
- A blocking decision is needed (safety gate triggers, prior-phase completeness prompt, user input needed)
- An error or paused state is detected

When a blocker stops the chain, show:
```
⛔ Auto-chain stopped: [reason — e.g. safety gate, blocking decision required]

Resume with: `/gsd:progress --next --auto` once resolved.
```
</step>

</process>

<success_criteria>
- [ ] The workflow correctly detects the project state.
- [ ] Gates 1-3 (repo/state validity) always run first, even on the resume path.
- [ ] Route 0 (resume_incomplete_phase) runs AFTER Gates 1-3 and BEFORE the prior-phase defer prompt. The default case (no flag) has no double decision.
- [ ] Default (no flag): Route 0 silently resumes the incomplete phase and exits. The user never sees the prior-phase defer prompt.
- [ ] `--no-resume`: The workflow skips Route 0. The prior_phase_completeness defer prompt runs as before.
- [ ] `--force`: The workflow skips everything (Gates, Route 0, prior_phase_completeness) and goes directly to `determine_next_action`.
- [ ] The scan uses `gsd_run` (canonical resolver form). The scan shows errors instead of suppressing them.
- [ ] If a `roadmap.analyze` result has a `scope` other than `complete`, the workflow treats the scan as failed (warn + fall through). It never treats the result as a clean empty scan (#3184/#3165).
- [ ] The predicate is plans-without-summaries (`plans.length > summaries.length`). It is consistent with `determine_next_action` Route 4.
- [ ] The workflow correctly determines the next action from routing rules.
- [ ] The workflow immediately invokes the command without user confirmation.
- [ ] The workflow shows clear status before it invokes the command.
- [ ] `--converge` routes Route 3 planning through `gsd-plan-review-convergence`.
- [ ] The workflow accepts `--cross-ai` as an alias for `--converge`.
- [ ] If `workflow.plan_review_convergence=false`, `--converge` fails fast with enable instructions.
- [ ] `--converge` forwards reviewer selector flags and `--max-cycles N`.
- [ ] If the user does not request convergence, default planning remains `gsd-plan-phase`.
</success_criteria>
