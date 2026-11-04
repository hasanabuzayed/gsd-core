@~/.claude/gsd-core/references/response-language-directive.md

<internal_workflow>

**This is an INTERNAL workflow — NOT a user-facing command.**

There is no `/gsd-transition` command. `execute-phase` automatically calls this workflow during auto-advance.

The orchestrator also calls it inline after phase verification. Users should never be told to run `/gsd-transition`.

**Valid user commands for phase progression:**
- `/gsd:discuss-phase {N}` — discuss a phase before planning
- `/gsd:plan-phase {N}` — plan a phase
- `/gsd:execute-phase {N}` — execute a phase
- `/gsd:progress` — see roadmap progress

</internal_workflow>

<required_reading>

**Read these files NOW:**

1. `.planning/STATE.md`
2. `.planning/PROJECT.md`
3. `.planning/ROADMAP.md`
4. Current phase's plan files (`*-PLAN.md`)
5. Current phase's summary files (`*-SUMMARY.md`)

</required_reading>

<purpose>

Mark the current phase complete. Advance to the next phase.

Track progress and update PROJECT.md at this point.

"Planning next phase" = "current phase is done"

</purpose>

<process>

<step name="post_completion_mode" priority="first">

**Invocation mode — read this FIRST.** This workflow has two modes:

1. **Standalone transition** (normal path): This workflow marks the phase complete and transitions it. Run EVERY step below in order. Run `verify_completion`, then `update_roadmap_and_state`, then the post-processing. `update_roadmap_and_state` calls `gsd_run query phase.complete`.

2. **Post-completion delegation** (`execute-phase` calls this after its auto-chain completion — #1526): execute-phase's `update_roadmap` step already called `phase.complete`. Verification already passed in execute-phase's `verify_phase_goal`. SKIP `verify_completion` and `update_roadmap_and_state`. A second call to `phase.complete` would write STATE.md/ROADMAP.md again.

   Run `cleanup_handoff` to clear stale `.continue-here` handoffs after completion. Then start at `evolve_project`. Run every step from there through `offer_next_phase`. This is the post-processing parity set: graduation scan, session-continuity, project-reference, accumulated-context, current-position/progress.

   `archive_prompts` is a documented no-op in either mode.

If the caller states that phase completion and verification already ran, use post-completion mode. If in doubt, run standalone (mode 1). It is idempotent enough to be safe, but slower.
</step>

<step name="load_project_state" priority="first">

Before transition, read project state:

```bash
cat .planning/STATE.md 2>/dev/null || true
cat .planning/PROJECT.md 2>/dev/null || true
```

Parse the current position to make sure that you transition the correct phase.
Note accumulated context that may need an update after transition.

</step>

<step name="verify_completion">

Make sure that the current phase has all plan summaries:

```bash
(ls .planning/phases/XX-current/*-PLAN.md 2>/dev/null || true) | sort
(ls .planning/phases/XX-current/*-SUMMARY.md 2>/dev/null || true) | sort
```

**Verification logic:**

- Count PLAN files
- Count SUMMARY files
- If counts match: all plans complete
- If counts do not match: incomplete

<config-check>

```bash
cat .planning/config.json 2>/dev/null || true
```

</config-check>

**Check for verification debt in this phase:**

```bash
_GSD_SHIM_NAME="gsd-tools.cjs"; _GSD_RUNTIME_ROOT="${RUNTIME_DIR:-$(git rev-parse --show-toplevel 2>/dev/null || pwd)}"; GSD_TOOLS="${_GSD_RUNTIME_ROOT}/gsd-core/bin/${_GSD_SHIM_NAME}"; _gsd_at() { for _p; do if [ -f "$_p" ]; then GSD_TOOLS="$_p"; return 0; fi; done; return 1; }; _gsd_id_ok() { case "$("$1" runtime-identity --raw 2>/dev/null || true)" in '{"packageName":"@opengsd/gsd-core"'*'}') return 0;; *) return 1;; esac; }; _gsd_homes() { set -- "${CLAUDE_CONFIG_DIR:-$HOME/.claude}" "${ANTIGRAVITY_CONFIG_DIR:-$HOME/.gemini/antigravity}" "$HOME/.gemini/antigravity-ide" "$HOME/.gemini/antigravity-cli" "${AUGMENT_CONFIG_DIR:-$HOME/.augment}" "${CLINE_CONFIG_DIR:-$HOME/.cline}" "${CODEBUDDY_CONFIG_DIR:-$HOME/.codebuddy}" "${CODEX_HOME:-$HOME/.codex}" "${COPILOT_CONFIG_DIR:-${COPILOT_HOME:-$HOME/.copilot}}" "${CURSOR_CONFIG_DIR:-$HOME/.cursor}" "${HERMES_HOME:-$HOME/.hermes}" "${KILO_CONFIG_DIR:-${XDG_CONFIG_HOME:-$HOME/.config}/kilo}" "${KIMI_CONFIG_DIR:-$HOME/.config/agents}" "$HOME/.agents" "${KIMI_CODE_HOME:-$HOME/.kimi-code}" "${OPENCODE_CONFIG_DIR:-${XDG_CONFIG_HOME:-$HOME/.config}/opencode}" "${PI_CODING_AGENT_DIR:-$HOME/.pi/agent}" "${QWEN_CONFIG_DIR:-$HOME/.qwen}" "${TRAE_CONFIG_DIR:-$HOME/.trae}" "${WINDSURF_CONFIG_DIR:-$HOME/.codeium/windsurf}" "${ZCODE_CONFIG_DIR:-$HOME/.zcode}" "${GROK_AGENTS_HOME:-$HOME/.agents}"; for _h; do _gsd_at "$_h/gsd-core/bin/${_GSD_SHIM_NAME}" && return 0; done; return 1; }; if _gsd_at "${_GSD_RUNTIME_ROOT}/gsd-core/bin/${_GSD_SHIM_NAME}" "${_GSD_RUNTIME_ROOT}/.claude/gsd-core/bin/${_GSD_SHIM_NAME}" "${_GSD_RUNTIME_ROOT}/.codex/gsd-core/bin/${_GSD_SHIM_NAME}"; then gsd_run() { node "$GSD_TOOLS" "$@"; }; elif _gsd_homes; then gsd_run() { node "$GSD_TOOLS" "$@"; }; elif unset -f gsd_run; _G="$(command -v gsd_run)"; [ -n "$_G" ] && _gsd_id_ok "$_G"; then GSD_TOOLS="$_G"; gsd_run() { "$GSD_TOOLS" "$@"; }; else echo "ERROR: gsd-tools.cjs not found at $GSD_TOOLS and no identity-proving gsd_run is on PATH. Run: npx -y @opengsd/gsd-core@latest --claude --local" >&2; exit 1; fi; GSD_IDENTITY_STATUS=unverified; _gsd_id_ok gsd_run && GSD_IDENTITY_STATUS=ok; export GSD_IDENTITY_STATUS; [ "$GSD_IDENTITY_STATUS" = ok ] || echo "WARNING: \"$GSD_TOOLS\" did not prove it is @opengsd/gsd-core - it is either a different package or an @opengsd/gsd-core older than the runtime-identity verb. See docs/how-to/diagnose-a-foreign-gsd-tools.md" >&2; if [ -n "${CLAUDE_ENV_FILE:-}" ] && [ -n "${GSD_TOOLS:-}" ]; then printf "export PATH='%s':\"\$PATH\"\n" "${GSD_TOOLS%/*}" >> "$CLAUDE_ENV_FILE" 2>/dev/null || true; fi
# #5118: the verification status is read through its ONE owner — never with
# awk/sed/grep over the report (that bypassed the closed enum, the staleness
# check and the phase-pinned report resolution, #3492). A non-zero exit is the
# owner's hard error (a status outside passed | gaps_found | human_needed).
VERIFY_STATUS=$(gsd_run query verification.status .planning/phases/XX-current --pick status) || { echo "verification.status refused this phase's report — see the error above." >&2; exit 1; }
```

**If VERIFY_STATUS is not `passed`:**

Stop before confirming:

```
Verification incomplete: ${VERIFY_STATUS}

Resolve before transition. Review: `/gsd:audit-uat`
```

This preliminary check blocks obviously unresolved verification before the authoritative gate below.

`gsd_run query phase.complete` in `update_roadmap_and_state` remains the authoritative stale-aware gate. It fail-closes unless canonical verification status is `passed`.

**If all plans complete:**

<if mode="yolo">

```
⚡ Auto-approved: Transition Phase [X] → Phase [X+1]
Phase [X] complete — all [Y] plans finished.

Proceeding to mark done and advance...
```

Go directly to the cleanup_handoff step.

</if>

<if mode="interactive" AND="gates.confirm_transition != false">

Ask: "Phase [X] complete — all [Y] plans finished. Ready to mark done and move to Phase [X+1]?"

Wait for confirmation before you continue.

</if>

**If plans incomplete:**

**SAFETY RAIL.** Skipping incomplete plans is destructive. ALWAYS prompt in every mode.

Show:

```
Phase [X] has incomplete plans:
- {phase}-01-SUMMARY.md ✓ Complete
- {phase}-02-SUMMARY.md ✗ Missing
- {phase}-03-SUMMARY.md ✗ Missing

⚠️ Safety rail: Skipping plans requires confirmation (destructive action)

Options:
1. Continue current phase (execute remaining plans)
2. Mark complete anyway (skip remaining plans)
3. Review what's left
```

Wait for user decision.

</step>

<step name="cleanup_handoff">

Look for remaining handoffs:

```bash
ls .planning/phases/XX-current/.continue-here*.md 2>/dev/null || true
```

If you find handoffs, delete them. The phase is complete and the handoffs are stale.

</step>

<step name="update_roadmap_and_state">

**Delegate ROADMAP.md and STATE.md updates to `gsd_run query phase.complete`:**

```bash
TRANSITION=$(gsd_run query phase.complete "${current_phase}")
```

The CLI does these actions:
- Marks the phase checkbox as `[x]` complete with today's date
- Updates the plan count to the final count (for example, "3/3 plans complete")
- Updates the Progress table (Status → Complete, adds date)
- Advances STATE.md to the next phase (Current Phase, Status → Ready to plan, Current Plan → Not started)
- Detects whether this is the last phase in the milestone

Extract from result: `completed_phase`, `plans_executed`, `next_phase`, `next_phase_name`, `is_last_phase`.

</step>

<step name="archive_prompts">

If prompts were generated for the phase, they stay in place.
The `completed/` subfolder pattern from create-meta-prompts handles archival.

</step>

<step name="evolve_project">

Update PROJECT.md with the learnings from the completed phase.

**Read phase summaries:**

```bash
_SUMMARIES=( .planning/phases/XX-current/*-SUMMARY.md )
if [ -e "${_SUMMARIES[0]}" ]; then cat "${_SUMMARIES[@]}"; fi
```

**Assess requirement changes:**

1. **Requirements validated?**
   - Did this phase ship any Active requirements?
   - Move them to Validated with a phase reference: `- ✓ [Requirement] — Phase X`

2. **Requirements invalidated?**
   - Did the phase find any Active requirements unnecessary or wrong?
   - Move them to Out of Scope with a reason: `- [Requirement] — [why invalidated]`

3. **Requirements emerged?**
   - Did building reveal any new requirements?
   - Add them to Active: `- [ ] [New requirement]`

4. **Decisions to log?**
   - Extract decisions from SUMMARY.md files.
   - Add them to the Key Decisions table.
   - If you know the outcome, add it to the table.

5. **"What This Is" still accurate?**
   - If the product changed meaningfully, update the description.
   - Keep it current and accurate.

**Update PROJECT.md:**

Make the edits inline. Update "Last updated" footer:

```markdown
---
*Last updated: [date] after Phase [X]*
```

**Example evolution:**

Before:

```markdown
### Active

- [ ] JWT authentication
- [ ] Real-time sync < 500ms
- [ ] Offline mode

### Out of Scope

- OAuth2 — complexity not needed for v1
```

After (Phase 2 shipped JWT auth, discovered rate limiting needed):

```markdown
### Validated

- ✓ JWT authentication — Phase 2

### Active

- [ ] Real-time sync < 500ms
- [ ] Offline mode
- [ ] Rate limiting on sync endpoint

### Out of Scope

- OAuth2 — complexity not needed for v1
```

**Step complete when:**

- [ ] Phase summaries reviewed for learnings
- [ ] Validated requirements moved from Active
- [ ] Invalidated requirements moved to Out of Scope with reason
- [ ] Emerged requirements added to Active
- [ ] New decisions logged with rationale
- [ ] "What This Is" updated if product changed
- [ ] "Last updated" footer reflects this transition

</step>

<step name="graduation_scan">

Scan LEARNINGS.md files from recent phases for recurring patterns. Show promotion candidates to the developer.

**Invoke the graduation helper:**

```text
@~/.claude/gsd-core/workflows/graduation.md
```

`graduation.md` handles this entire step. It handles guard checks, clustering, backlog filtering, HITL prompting, promotion writes, and STATE.md updates.

The guard checks cover the feature flag, window size, and threshold.

**This step is always non-blocking:** The developer sees graduation candidates and decides what to do. No action needs to occur for the transition to continue.

If the graduation scan produces no qualifying clusters, it prints a single `[graduation: no qualifying clusters]` line and returns.

**Step complete when:**

- [ ] graduation.md guard checks passed (or skipped with silent no-op)
- [ ] Recurring clusters surfaced (or `[graduation: no qualifying clusters]` printed)
- [ ] Each cluster resolved as Promote / Defer / Dismiss (or all skipped)

</step>

<step name="update_current_position_after_transition">

**Note:** `gsd_run query phase.complete` already made the basic position updates in the update_roadmap_and_state step. These updates include Current Phase, Status, Current Plan, and Last Activity.

Read STATE.md to make sure that the updates are correct. If the progress bar needs an update, use:

```bash
PROGRESS=$(gsd_run query progress.bar --raw)
```

Update the progress bar line in STATE.md with the result.

**Step complete when:**

- [ ] Phase number incremented to next phase (done by phase complete)
- [ ] Plan status reset to "Not started" (done by phase complete)
- [ ] Status shows "Ready to plan" (done by phase complete)
- [ ] Progress bar reflects total completed plans

</step>

<step name="update_project_reference">

Update Project Reference section in STATE.md.

```markdown
## Project Reference

See: .planning/PROJECT.md (updated [today])

**Core value:** [Current core value from PROJECT.md]
**Current focus:** [Next phase name]
```

Update the date and current focus to reflect the transition.

</step>

<step name="review_accumulated_context">

Review and update Accumulated Context section in STATE.md.

**Decisions:**

- Note recent decisions from this phase (3-5 max)
- Full log lives in PROJECT.md Key Decisions table

**Blockers/Concerns:**

- Review blockers from completed phase
- If addressed in this phase: Remove from list
- If still relevant for future: Keep with "Phase X" prefix
- Add any new concerns from completed phase's summaries

**Example:**

Before:

```markdown
### Blockers/Concerns

- ⚠️ [Phase 1] Database schema not indexed for common queries
- ⚠️ [Phase 2] WebSocket reconnection behavior on flaky networks unknown
```

After (if database indexing was addressed in Phase 2):

```markdown
### Blockers/Concerns

- ⚠️ [Phase 2] WebSocket reconnection behavior on flaky networks unknown
```

**Step complete when:**

- [ ] Recent decisions noted (full log in PROJECT.md)
- [ ] Resolved blockers removed from list
- [ ] Unresolved blockers kept with phase prefix
- [ ] New concerns from completed phase added

</step>

<step name="update_session_continuity_after_transition">

Update Session Continuity section in STATE.md to reflect transition completion.

**Format:**

```markdown
Last session: [today]
Stopped at: Phase [X] complete, ready to plan Phase [X+1]
Resume file: None
```

**Step complete when:**

- [ ] Last session timestamp updated to current date and time
- [ ] Stopped at describes phase completion and next phase
- [ ] Resume file confirmed as None (transitions do not use resume files)

</step>

<step name="offer_next_phase">

**MANDATORY: Make sure of the milestone status before you show the next steps.**

**Use the transition result from `gsd_run query phase.complete`:**

The `is_last_phase` field from the phase complete result shows:
- `is_last_phase: false` → More phases remain → Go to **Route A**
- `is_last_phase: true` → Last phase done → **Check for workstream collisions first**

The `next_phase` and `next_phase_name` fields give you the next phase details.

If you need more context, use:
```bash
ROADMAP=$(gsd_run query roadmap.analyze)
```

This returns all phases with goals, disk status, and completion information.

**Section-manifest gate (#2994):** The `verify_completion` step above already establishes `gsd_run`.

Get the dedicated `init.transition` bundle for the workstream-collision-check gate below:

```bash
IFS= read -rd '' GSD_ARGS <<'GSD_ARGS_END' || true
$ARGUMENTS
GSD_ARGS_END
GSD_ARGS=${GSD_ARGS%$'\n'}
GSD_WS=$(echo " $GSD_ARGS" | sed -nE 's/.* --ws +([A-Za-z0-9][A-Za-z0-9._-]*).*/--ws \1/p' | head -n 1)
INIT_TRANSITION=$(gsd_run query init.transition ${GSD_WS:+--ws=${GSD_WS##* }})
if [[ "$INIT_TRANSITION" == @file:* ]]; then INIT_TRANSITION=$(cat "${INIT_TRANSITION#@file:}"); fi
```

Extract from `INIT_TRANSITION`: `other_active_workstreams`, `section_manifest`.

---

<!-- gsd:section id="workstream-collision-check" when="state:workstream-active" -->
If `section_manifest` from `INIT_TRANSITION` is `null`, read and execute `gsd-core/workflows/transition/steps/workstream-collision-check.md`.

If `"workstream-collision-check"` is in its `included` list, read and execute that file. Otherwise (flat mode), skip it. Do not read the file. Go directly to **Route B**.
<!-- /gsd:section -->

---

**Route A: More phases remain in milestone**

Read ROADMAP.md to get the next phase's name and goal.

**Look for CONTEXT.md in the next phase:**

```bash
ls .planning/phases/*[X+1]*/*-CONTEXT.md 2>/dev/null || true
```

**If next phase exists:**

<if mode="yolo">

**If CONTEXT.md exists:**

```
Phase [X] marked complete.

Next: Phase [X+1] — [Name]

⚡ Auto-continuing: Plan Phase [X+1] in detail
```

Exit skill and invoke SlashCommand("/gsd:plan-phase [X+1] --auto ${GSD_WS}")

**If CONTEXT.md does NOT exist:**

```
Phase [X] marked complete.

Next: Phase [X+1] — [Name]

⚡ Auto-continuing: Discuss Phase [X+1] first
```

Exit skill and invoke SlashCommand("/gsd:discuss-phase [X+1] --auto ${GSD_WS}")

</if>

<if mode="interactive" AND="gates.confirm_transition != false">

**If CONTEXT.md does NOT exist:**

```
## ✓ Phase [X] Complete

---

## ▶ Next Up — [${PROJECT_CODE}] ${PROJECT_TITLE}

**Phase [X+1]: [Name]** — [Goal from ROADMAP.md]

`/clear` then:

`/gsd:discuss-phase [X+1] ${GSD_WS}` — gather context and clarify approach

---

**Also available:**
- `/gsd:plan-phase [X+1] ${GSD_WS}` — skip discussion, plan directly
- `/gsd:plan-phase --research-phase [X+1] ${GSD_WS}` — investigate unknowns

---
```

**If CONTEXT.md exists:**

```
## ✓ Phase [X] Complete

---

## ▶ Next Up — [${PROJECT_CODE}] ${PROJECT_TITLE}

**Phase [X+1]: [Name]** — [Goal from ROADMAP.md]
<sub>✓ Context gathered, ready to plan</sub>

`/clear` then:

`/gsd:plan-phase [X+1] ${GSD_WS}`

---

**Also available:**
- `/gsd:discuss-phase [X+1] ${GSD_WS}` — revisit context
- `/gsd:plan-phase --research-phase [X+1] ${GSD_WS}` — investigate unknowns

---
```

</if>

---

**Route B1: Workstream done, other workstreams still active**

Use this route when `is_last_phase: true` AND the collision check found other active workstreams.

Do NOT suggest completing the milestone or advancing to the next milestone. Other workstreams are still active.

**Clear auto-advance chain flag** — the workstream boundary is the stopping point:

```bash
gsd_run query config-set workflow._auto_chain_active false
```

<if mode="yolo">

Override auto-advance: do NOT auto-continue to milestone completion.
Show the blocking information and stop.

</if>

Show (all modes):

```
## ✓ Phase {X}: {Phase Name} Complete

This workstream's phases are complete. Other workstreams are still active:

| Workstream | Status | Phase | Progress |
|------------|--------|-------|----------|
| {name}     | {status} | {current_phase} | {completed_phases}/{phase_count} |
| ...        | ...    | ...   | ...      |

---

## Next Steps

Archive this workstream:

`/gsd:workstreams complete {current_ws_name} ${GSD_WS}`

See overall milestone progress:

`/gsd:workstreams progress ${GSD_WS}`

<sub>Milestone completion will be available once all workstreams finish.</sub>

---
```

Do NOT suggest `/gsd:complete-milestone` or `/gsd:new-milestone`.
Do NOT auto-invoke any further slash commands.

**Stop here.** The user must explicitly decide what to do next.

---

**Route B: All phases complete (milestone ready to close)**

**Use this route only when:**
- `is_last_phase: true` AND no other active workstreams exist (or flat mode)

**Clear auto-advance chain flag** — the milestone boundary is the stopping point:

```bash
gsd_run query config-set workflow._auto_chain_active false
```

<if mode="yolo">

```
Phase {X} marked complete.

🎉 Milestone {version} is 100% complete — all {N} phases finished!

⚡ Auto-continuing: Complete milestone and archive
```

Exit skill and invoke SlashCommand("/gsd:complete-milestone {version} ${GSD_WS}")

</if>

<if mode="interactive" AND="gates.confirm_transition != false">

```
## ✓ Phase {X}: {Phase Name} Complete

🎉 Milestone {version} is 100% complete — all {N} phases finished!

---

## ▶ Next Up — [${PROJECT_CODE}] ${PROJECT_TITLE}

**Complete Milestone {version}** — archive and prepare for next

`/clear` then:

`/gsd:complete-milestone {version} ${GSD_WS}`

---

**Also available:**
- Review accomplishments before archiving

---
```

</if>

</step>

</process>

<implicit_tracking>
Progress tracking is IMPLICIT: planning phase N implies that phases 1-(N-1) are complete.

There is no separate progress step. Forward motion IS progress.
</implicit_tracking>

<partial_completion>

If user wants to move on but phase is not fully complete:

```
Phase [X] has incomplete plans:
- {phase}-02-PLAN.md (not executed)
- {phase}-03-PLAN.md (not executed)

Options:
1. Mark complete anyway (plans weren't needed)
2. Defer work to later phase
3. Stay and finish current phase
```

Respect the user's judgment. The user knows whether the work matters.

**If marking complete with incomplete plans:**

- Update ROADMAP: "2/3 plans complete" (not "3/3")
- Note in transition message which plans were skipped

</partial_completion>

<success_criteria>

Transition is complete when:

- [ ] Current phase plan summaries verified (all exist or user chose to skip)
- [ ] Any stale handoffs deleted
- [ ] ROADMAP.md updated with completion status and plan count
- [ ] PROJECT.md evolved (requirements, decisions, description if needed)
- [ ] STATE.md updated (position, project reference, context, session)
- [ ] Progress table updated
- [ ] User knows next steps

</success_criteria>
