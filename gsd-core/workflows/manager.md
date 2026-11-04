<purpose>

This interactive command center manages a milestone from a single terminal. It shows a dashboard with the status of all phases.

It dispatches discuss inline and runs plan/execute inline. When dispatch-should-flatten returns false, it runs plan/execute in the background. It returns to the dashboard after each action.

It supports parallel phase work from one terminal.

</purpose>

<required_reading>

Before you start, read all files that the invoking prompt's execution_context references.

</required_reading>

<process>

<step name="initialize" priority="first">

## 1. Initialize

Start with manager init:

```bash
IFS= read -rd '' GSD_ARGS <<'GSD_ARGS_END' || true
$ARGUMENTS
GSD_ARGS_END
GSD_ARGS=${GSD_ARGS%$'\n'}
_GSD_SHIM_NAME="gsd-tools.cjs"; _GSD_RUNTIME_ROOT="${RUNTIME_DIR:-$(git rev-parse --show-toplevel 2>/dev/null || pwd)}"; GSD_TOOLS="${_GSD_RUNTIME_ROOT}/gsd-core/bin/${_GSD_SHIM_NAME}"; _gsd_at() { for _p; do if [ -f "$_p" ]; then GSD_TOOLS="$_p"; return 0; fi; done; return 1; }; _gsd_id_ok() { case "$("$1" runtime-identity --raw 2>/dev/null || true)" in '{"packageName":"@opengsd/gsd-core"'*'}') return 0;; *) return 1;; esac; }; _gsd_homes() { set -- "${CLAUDE_CONFIG_DIR:-$HOME/.claude}" "${ANTIGRAVITY_CONFIG_DIR:-$HOME/.gemini/antigravity}" "$HOME/.gemini/antigravity-ide" "$HOME/.gemini/antigravity-cli" "${AUGMENT_CONFIG_DIR:-$HOME/.augment}" "${CLINE_CONFIG_DIR:-$HOME/.cline}" "${CODEBUDDY_CONFIG_DIR:-$HOME/.codebuddy}" "${CODEX_HOME:-$HOME/.codex}" "${COPILOT_CONFIG_DIR:-${COPILOT_HOME:-$HOME/.copilot}}" "${CURSOR_CONFIG_DIR:-$HOME/.cursor}" "${HERMES_HOME:-$HOME/.hermes}" "${KILO_CONFIG_DIR:-${XDG_CONFIG_HOME:-$HOME/.config}/kilo}" "${KIMI_CONFIG_DIR:-$HOME/.config/agents}" "$HOME/.agents" "${KIMI_CODE_HOME:-$HOME/.kimi-code}" "${OPENCODE_CONFIG_DIR:-${XDG_CONFIG_HOME:-$HOME/.config}/opencode}" "${PI_CODING_AGENT_DIR:-$HOME/.pi/agent}" "${QWEN_CONFIG_DIR:-$HOME/.qwen}" "${TRAE_CONFIG_DIR:-$HOME/.trae}" "${WINDSURF_CONFIG_DIR:-$HOME/.codeium/windsurf}" "${ZCODE_CONFIG_DIR:-$HOME/.zcode}" "${GROK_AGENTS_HOME:-$HOME/.agents}"; for _h; do _gsd_at "$_h/gsd-core/bin/${_GSD_SHIM_NAME}" && return 0; done; return 1; }; if _gsd_at "${_GSD_RUNTIME_ROOT}/gsd-core/bin/${_GSD_SHIM_NAME}" "${_GSD_RUNTIME_ROOT}/.claude/gsd-core/bin/${_GSD_SHIM_NAME}" "${_GSD_RUNTIME_ROOT}/.codex/gsd-core/bin/${_GSD_SHIM_NAME}"; then gsd_run() { node "$GSD_TOOLS" "$@"; }; elif _gsd_homes; then gsd_run() { node "$GSD_TOOLS" "$@"; }; elif unset -f gsd_run; _G="$(command -v gsd_run)"; [ -n "$_G" ] && _gsd_id_ok "$_G"; then GSD_TOOLS="$_G"; gsd_run() { "$GSD_TOOLS" "$@"; }; else echo "ERROR: gsd-tools.cjs not found at $GSD_TOOLS and no identity-proving gsd_run is on PATH. Run: npx -y @opengsd/gsd-core@latest --claude --local" >&2; exit 1; fi; GSD_IDENTITY_STATUS=unverified; _gsd_id_ok gsd_run && GSD_IDENTITY_STATUS=ok; export GSD_IDENTITY_STATUS; [ "$GSD_IDENTITY_STATUS" = ok ] || echo "WARNING: \"$GSD_TOOLS\" did not prove it is @opengsd/gsd-core - it is either a different package or an @opengsd/gsd-core older than the runtime-identity verb. See docs/how-to/diagnose-a-foreign-gsd-tools.md" >&2; if [ -n "${CLAUDE_ENV_FILE:-}" ] && [ -n "${GSD_TOOLS:-}" ]; then printf "export PATH='%s':\"\$PATH\"\n" "${GSD_TOOLS%/*}" >> "$CLAUDE_ENV_FILE" 2>/dev/null || true; fi
GSD_WS=$(echo " $GSD_ARGS" | sed -nE 's/.* --ws +([A-Za-z0-9][A-Za-z0-9._-]*).*/--ws \1/p' | head -n 1)
INIT=$(gsd_run query init.manager ${GSD_WS:+--ws=${GSD_WS##* }})
if [[ "$INIT" == @file:* ]]; then INIT=$(cat "${INIT#@file:}"); fi
```

Parse the JSON for `milestone_version`, `milestone_name`, `phase_count`, `completed_count`, `in_progress_count`, `phases`, `recommended_actions`, `all_complete`, `waiting_signal`, `manager_flags`, and `response_language`.

Also parse the optional fields `queued_milestone_version`, `queued_milestone_name`, and `queued_phases`. SDK fix `2495-2496-2497` added these fields. They may be absent on older SDK versions. If a field is missing, treat it as empty.

**If `response_language` is set:** Present all user-facing output of this workflow in `{response_language}`. This MUST include narration between tool calls, status updates, progress notes, findings, questions, prompts, and explanations.

Keep technical terms, code, file paths, and subagent prompts in English. Translate only user-facing output. Keep subagent dispatch prompts (discuss/plan/execute) in English. Include `response_language` in their spawn args as specified by the dispatched workflow.

`manager_flags` contains passthrough flags from config for each step:
- `manager_flags.discuss` — append to `/gsd:discuss-phase` args, for example `"--auto --analyze"`
- `manager_flags.plan` — append to the plan agent init command
- `manager_flags.execute` — append to the execute agent init command

These are empty strings by default. Set them with: `gsd_run query config-set manager.flags.discuss "--auto --analyze"`

**If there is an error:** Show the error message and exit.

Show the startup banner:

```
### GSD ► MANAGER

 {milestone_version} — {milestone_name}
 {phase_count} phases · {completed_count} complete

 ✓ Discuss → inline    ◆ Plan/Execute → inline (background when FLATTEN=false)
 Dashboard auto-refreshes when background work is active.

---
```

Go to the dashboard step.

</step>

<step name="dashboard">

## 2. Dashboard (Refresh Point)

**Each time you reach this step**, read the state from disk again. This gets changes from background agents:

```bash
IFS= read -rd '' GSD_ARGS <<'GSD_ARGS_END' || true
$ARGUMENTS
GSD_ARGS_END
GSD_ARGS=${GSD_ARGS%$'\n'}
GSD_WS=$(echo " $GSD_ARGS" | sed -nE 's/.* --ws +([A-Za-z0-9][A-Za-z0-9._-]*).*/--ws \1/p' | head -n 1)
INIT=$(gsd_run query init.manager ${GSD_WS:+--ws=${GSD_WS##* }})
if [[ "$INIT" == @file:* ]]; then INIT=$(cat "${INIT#@file:}"); fi
```

Parse the full JSON. Build the dashboard from the JSON. Symbols: `✓` done, `◆` active, `○` pending, `·` queued. Progress bar: 20-char `█░`.

**Status mapping** (disk_status → D P E Status):

- `complete` → `✓ ✓ ✓` `✓ Complete`
- `executed` → `✓ ✓ ◆` `◆ Verification required`
- `partial` → `✓ ✓ ◆` `◆ Executing...`
- `planned` → `✓ ✓ ○` `○ Ready to execute`
- `discussed` → `✓ ○ ·` `○ Ready to plan`
- `researched` → `◆ · ·` `○ Ready to plan`
- `empty`/`no_directory` + `is_next_to_discuss` → `○ · ·` `○ Ready to discuss`
- `empty`/`no_directory` otherwise → `· · ·` `· Up next`
- If `is_active`, replace the status icon with `◆` and append `(active)`

If any phases have `is_active`, show `◆ Background: {action} Phase {N}, ...` above the grid.

Use `display_name` (not `name`) for the Phase column. If clipped, it is pre-truncated to 20 chars with `…`. Pad all phase names to the same width for alignment.

Use `deps_display` from the init JSON for the Deps column. It shows the phases that this phase depends on, for example `1,3`. If there are none, it shows `—`.

Example output:

```
### GSD ► DASHBOARD
 ████████████░░░░░░░░ 60%  (3/5 phases)
 ◆ Background: Planning Phase 4
 | # | Phase                | Deps | D | P | E | Status              |
 |---|----------------------|------|---|---|---|---------------------|
 | 1 | Foundation           | —    | ✓ | ✓ | ✓ | ✓ Complete          |
 | 2 | API Layer            | 1    | ✓ | ✓ | ◆ | ◆ Executing (active)|
 | 3 | Auth System          | 1    | ✓ | ✓ | ○ | ○ Ready to execute  |
 | 4 | Dashboard UI & Set…  | 1,2  | ✓ | ◆ | · | ◆ Planning (active) |
 | 5 | Notifications        | —    | ○ | · | · | ○ Ready to discuss  |
 | 6 | Polish & Final Mail… | 1-5  | · | · | · | · Up next           |
```

**Queued section (next milestone preview):**

If `queued_phases` is present and non-empty, show a compact preview of the next milestone's phases directly below the main table. This shows future work without adding it to the active-milestone grid.

If `queued_phases` is empty or missing, skip this section entirely. For example, this occurs when the active milestone is the last one in the roadmap.

Use `queued_milestone_version` and `queued_milestone_name` for the header. The phases are not discussed yet. Show them without D/P/E columns.

Show only the number, name (pre-truncated `display_name`), dependencies (`deps_display`), and fixed `· Queued` status. Phase-name padding should match the active-table column width for visual alignment.

Example:

```
### ◆ Queued — {queued_milestone_version} {queued_milestone_name}  ({queued_phases.length} phases)
 | # | Phase                | Deps | Status       |
 |---|----------------------|------|--------------|
 | 31| Email Logs           | —    | · Queued     |
 | 32| Today's Sheets       | 31   | · Queued     |
 | 33| Resend Backfill      | 31   | · Queued     |
 | 34| Business Day Audit   | 31   | · Queued     |
```

Queued phases are NOT eligible for the Continue action menu. They are in a future milestone and must wait for the current milestone to ship. The preview is only for awareness of future work.

**Recommendations section:**

If `all_complete` is true:

```
### MILESTONE COMPLETE

All {phase_count} phases verified complete. Ready for final steps:
  → /gsd:verify-work — run acceptance testing
  → /gsd:complete-milestone — archive and wrap up
```

**Text mode (`workflow.text_mode: true` in config or `--text` flag):** If `$ARGUMENTS` contains `--text` OR `text_mode` from init JSON is `true`, set `TEXT_MODE=true`.

When TEXT_MODE is active, replace every `AskUserQuestion` call with a plain-text numbered list. Ask the user to type their choice number. Non-Claude runtimes need this when `AskUserQuestion` is not available. Examples include OpenAI Codex and Antigravity.

Ask the user via AskUserQuestion:
- **question:** "All phases complete. What next?"
- **options:** "Verify work" / "Complete milestone" / "Exit manager"

Handle responses:
- "Verify work": `Skill(skill="gsd-verify-work")`  then loop to dashboard.
- "Complete milestone": `Skill(skill="gsd-complete-milestone")` then exit.
- "Exit manager": Go to exit step.

**If NOT all_complete**, build compound options from `recommended_actions`:

**Compound option logic:** Group background actions (plan/execute) together. When a single inline action (discuss) exists, pair it with them. Show the fewest options possible. One option can dispatch multiple background agents plus one inline action.

**Building options:**

1. Collect all background actions (execute and plan recommendations). There can be multiple of each.
2. Collect verification actions (`verify`) for implementation-complete phases without a passing canonical verification.
3. If an inline action exists, collect it (the discuss recommendation). There will be at most one because discuss is sequential.
4. Build compound options:

   **If there are ANY recommended actions (background, inline, or both):**
   Create ONE primary "Continue" option that dispatches ALL of them together:
   - Label: `"Continue"` — always this exact word
   - Below the label, list every action that will occur. List ALL recommended actions. Do not limit or truncate the list:
     ```
     Continue:
       → Execute Phase 32 (background)
       → Plan Phase 34 (background)
       → Verify Phase 33
       → Discuss Phase 35 (inline)
     ```
   - This dispatches all background agents first. It then runs verification actions inline. If an inline discuss exists, it runs that next.
   - If there is no inline discuss, the dashboard refreshes after it spawns background agents and runs inline verification.

   **Important:** The Continue option must include EVERY action from `recommended_actions`, not just 2. If there are 3 actions, list 3. If there are 5, list 5.

4. Always add:
   - `"Refresh dashboard"`
   - `"Exit manager"`

Show the recommendations in a compact format:

```
### ▶ Next Steps

Continue:
  → Execute Phase 32 (background)
  → Plan Phase 34 (background)
  → Discuss Phase 35 (inline)
```

**Auto-refresh:** If background agents are active (`is_active` is true for any phase), set a 60-second auto-refresh cycle. After you show the action menu, wait for user input. If no input arrives within 60 seconds, automatically refresh the dashboard.

You can configure this interval with `manager_refresh_interval` in GSD config. The default is 60 seconds. Set it to 0 to disable auto-refresh.

Present via AskUserQuestion:
- **question:** "What would you like to do?"
- **options:** (compound options as built above + refresh + exit, AskUserQuestion auto-adds "Other")

**On "Other" (free text):** Parse the intent. If it mentions a phase number and action, dispatch that action. If the intent is unclear, show the available actions and return to action_menu.

Go to the handle_action step with the selected action.

</step>

<step name="handle_action">

## 4. Handle Action

### Refresh Dashboard

Loop back to dashboard step.

### Exit Manager

Go to exit step.

### Compound Action (background + inline)

When the user selects a compound option, behavior depends on runtime support for background dispatch of nesting-capable orchestrators. The Plan Phase N / Execute Phase N handlers below resolve this with `gsd_run query dispatch-should-flatten` (#1708):

- **If `FLATTEN` is `false`:** The host can run a nesting-capable orchestrator in the background, for example codex or cursor. **Spawn all background agents first** (plan/execute). Dispatch them in parallel with the Plan Phase N / Execute Phase N handlers below. Then run verification actions. Then run the inline discuss. The background agents continue while you verify/discuss.
- **Otherwise (`FLATTEN` is `true` — run inline):** Run the chosen plan/execute step(s) **inline** via their handlers below, in order. Then run verification actions. Then run the inline discuss. There is no overlap.

Inline verification:

For each verification recommendation, dispatch by the recommended action's `command`:
- If `command` contains `execute-phase`, run `Skill(skill="gsd-execute-phase", args="{PHASE_NUM} {manager_flags.execute}")`.
- If `command` contains `verify-work`, run `Skill(skill="gsd-verify-work", args="{PHASE_NUM}")`.
- If `command` is missing or unrecognized, stop and show the recommendation row instead of guessing.

Inline discuss:

```
Skill(skill="gsd-discuss-phase", args="{PHASE_NUM} {manager_flags.discuss}")
```

After discuss completes, loop back to dashboard step.

### Discuss Phase N

Discussion is interactive and needs user input. Run it inline with any configured flags:

```
Skill(skill="gsd-discuss-phase", args="{PHASE_NUM} {manager_flags.discuss}")
```

After discuss completes, loop back to dashboard step.

### Plan Phase N

Planning runs autonomously. **First determine whether background dispatch is safe.** Background dispatch is only safe when a background agent can nest the pipeline's subagents on the runtime. These subagents include plan-checker / worktree executors / verifier.

The documentation-sourced dispatch capability in the registry determines this (#1708). Claude Code's background agents have no `Agent`/`Task` tool. Every other runtime prohibits nested subagents or disables them by default. Run **inline** everywhere except where `dispatch-should-flatten` returns `false`.

```bash
FLATTEN=$(gsd_run query dispatch-should-flatten --raw 2>/dev/null || echo "true")
```

**If `FLATTEN` is `false`:** Spawn a background agent that delegates to the Skill pipeline with any configured flags:

```
Agent(
  description="Plan phase {N}: {phase_name}",
  run_in_background=true,
  prompt="You are running the GSD plan-phase workflow for phase {N} of the project.

Working directory: {cwd}
Phase: {N} — {phase_name}
Goal: {goal}
Manager flags: {manager_flags.plan}

Run the plan-phase Skill with any configured manager flags:
Skill(skill=\"gsd-plan-phase\", args=\"{N} --auto {manager_flags.plan}\")

This delegates to the full plan-phase pipeline including local patches, research, plan-checker, and all quality gates.

Important: You are running in the background. Do NOT use AskUserQuestion — make autonomous decisions based on project context. If you hit a blocker, write it to STATE.md as a blocker and stop. Do NOT silently work around permission or file access errors — let them fail so the manager can surface them with resolution hints. Do NOT use --no-verify on git commits."
)
```

> **ORCHESTRATOR RULE — BACKGROUND DISPATCH**: After you call Agent() above with `run_in_background=true`, do NOT do planning work for this phase independently. Return to the dashboard immediately. Wait for the background agent to report its result.

> When the subagent result is available, and only then, resume planning-related work. Never call `ScheduleWakeup` or any host wake/sleep-scheduling tool while you wait (#4079). A partial-args wake call shows a red validation error. The dashboard loop is the wait.

Display:

```
◆ Spawning planner for Phase {N}: {phase_name}... (runs in a subagent — no output until it returns, ~1–5 min; expected, not a freeze)
```

Loop back to dashboard step.

**Otherwise (`FLATTEN` is `true` — run inline):** Run plan inline so the plan-checker and quality gates run. Do NOT wrap it in `Agent(run_in_background=true, …)`:

```
Skill(skill="gsd-plan-phase", args="{N} --auto {manager_flags.plan}")
```

Display while it runs:

```
◆ Planning Phase {N}: {phase_name}... (runs inline so the plan-checker runs — the dashboard resumes when it returns, ~1–5 min; expected, not a freeze)
```

Then loop back to dashboard step.

### Execute Phase N

Execution runs autonomously. **First determine whether background dispatch is safe.** Background dispatch is only safe when a background agent can nest the pipeline's subagents on the runtime. These subagents include plan-checker / worktree executors / verifier.

The documentation-sourced dispatch capability in the registry determines this (#1708). Claude Code's background agents have no `Agent`/`Task` tool. Every other runtime prohibits nested subagents or disables them by default. Run **inline** everywhere except where `dispatch-should-flatten` returns `false`.

```bash
FLATTEN=$(gsd_run query dispatch-should-flatten --raw 2>/dev/null || echo "true")
```

**If `FLATTEN` is `false`:** Spawn a background agent that delegates to the Skill pipeline with any configured flags:

```
Agent(
  description="Execute phase {N}: {phase_name}",
  run_in_background=true,
  prompt="You are running the GSD execute-phase workflow for phase {N} of the project.

Working directory: {cwd}
Phase: {N} — {phase_name}
Goal: {goal}
Manager flags: {manager_flags.execute}

Run the execute-phase Skill with any configured manager flags:
Skill(skill=\"gsd-execute-phase\", args=\"{N} {manager_flags.execute}\")

This delegates to the full execute-phase pipeline including local patches, branching, wave-based execution, verification, and all quality gates.

Important: You are running in the background. Do NOT use AskUserQuestion — make autonomous decisions. Do NOT use --no-verify on git commits — let pre-commit hooks run normally. If you hit a permission error, file lock, or any access issue, do NOT work around it — let it fail and write the error to STATE.md as a blocker so the manager can surface it with resolution guidance."
)
```

> **ORCHESTRATOR RULE — BACKGROUND DISPATCH**: After you call Agent() above with `run_in_background=true`, do NOT do execution work for this phase independently. Return to the dashboard immediately. Wait for the background agent to report its result.

> When the subagent result is available, and only then, resume execution-related work. Never call `ScheduleWakeup` or any host wake/sleep-scheduling tool while you wait (#4079). A partial-args wake call shows a red validation error. The dashboard loop is the wait.

Display:

```
◆ Spawning executor for Phase {N}: {phase_name}... (runs in a subagent — no output until it returns, ~1–5 min; expected, not a freeze)
```

Loop back to dashboard step.

**Otherwise (`FLATTEN` is `true` — run inline):** Run execute inline so worktree isolation and the verifier run. Do NOT wrap it in `Agent(run_in_background=true, …)`:

```
Skill(skill="gsd-execute-phase", args="{N} {manager_flags.execute}")
```

Display while it runs:

```
◆ Executing Phase {N}: {phase_name}... (runs inline so worktree isolation and verification run — the dashboard resumes when it returns; expected, not a freeze)
```

Then loop back to dashboard step.

</step>

<step name="background_completion">

## 5. Background Agent Completion

When you receive notice that a background agent completed:

1. Read the result message from the agent.
2. Show a brief notification:

```
✓ {description}
  {brief summary from agent result}
```

3. Loop back to dashboard step.

**If the agent reported an error or blocker:**

Classify the error:

**Permission / tool access error** (for example, tool not allowed, permission denied, sandbox restriction):
- Parse the error to find which tool or command the runtime blocked.
- Show the error clearly. Then offer to fix it:
  - **question:** "Phase {N} failed — permission denied for `{tool_or_command}`. Want me to add it to settings.local.json so it is allowed?"
  - **options:** "Add permission and retry" / "Run this phase inline instead" / "Skip and continue"
  - "Add permission and retry": Use `Skill(skill="update-config")` to add the permission to `settings.local.json`, then re-spawn the background agent. Loop to dashboard.
  - "Run this phase inline instead": Dispatch the same action inline via the appropriate Skill. If planning failed, use `Skill(skill="gsd-plan-phase", args="{N}")`. If execution failed, use `Skill(skill="gsd-execute-phase", args="{N}")`. Then return to the dashboard.
  - "Skip and continue": Loop to dashboard (phase stays in current state).

**Other errors** (for example, git lock, file conflict, logic error):
- Show the error. Then offer options via AskUserQuestion:
  - **question:** "Background agent for Phase {N} encountered an issue: {error}. What next?"
  - **options:** "Retry" / "Run inline instead" / "Skip and continue" / "View details"
  - "Retry": Re-spawn the same background agent. Loop to dashboard.
  - "Run inline instead": Dispatch the action inline via the appropriate Skill. If planning failed, use `Skill(skill="gsd-plan-phase", args="{N}")`. If execution failed, use `Skill(skill="gsd-execute-phase", args="{N}")`. Then return to the dashboard.
  - "Skip and continue": Loop to dashboard (phase stays in current state).
  - "View details": Read the STATE.md blockers section. Show it. Then show the options again.

</step>

<step name="exit">

## 6. Exit

Display final status with progress bar:

```
### GSD ► SESSION END

 {milestone_version} — {milestone_name}
 {PROGRESS_BAR} {progress_pct}%  ({completed_count}/{phase_count} phases)

 Resume anytime: /gsd:manager

---
```

**Note:** Any background agents still active will continue to completion. Their results will be visible on the next `/gsd:manager` or `/gsd:progress` invocation.

</step>

</process>

<success_criteria>
- [ ] The dashboard shows all phases with correct status indicators (D/P/E/V columns)
- [ ] The progress bar shows the correct completion percentage
- [ ] Dependency resolution: blocked phases show which deps are missing
- [ ] Recommendations use this priority: execute > plan > discuss
- [ ] Discuss phases run inline via Skill() — interactive questions work
- [ ] Plan phases run inline (or as background Task agents on Codex) — the dashboard resumes when complete
- [ ] Execute phases run inline (or as background Task agents on Codex) — the dashboard resumes when complete
- [ ] Dashboard refreshes get changes from background agents via disk state
- [ ] Background agent completion triggers a notification and dashboard refresh
- [ ] Background agent errors show retry/skip options
- [ ] The all-complete state offers verify-work and complete-milestone
- [ ] Exit shows final status with resume instructions
- [ ] "Other" free-text input is parsed for phase number and action
- [ ] The manager loop continues until the user exits or the milestone completes
- [ ] The queued section shows when `queued_phases` is non-empty. It is skipped when absent or empty
</success_criteria>
