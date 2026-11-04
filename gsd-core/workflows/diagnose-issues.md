@~/.claude/gsd-core/references/response-language-directive.md

<purpose>
Coordinate parallel debug agents to investigate UAT gaps and find root causes.

After UAT finds gaps, spawn one debug agent per gap. Each agent investigates independently with symptoms filled from UAT. Collect root causes. Update UAT.md gaps with the diagnosis. Then pass the diagnoses to plan-phase --gaps.

Keep the orchestrator limited to these tasks:
- Parse gaps
- Spawn agents
- Collect results
- Update UAT
</purpose>

<available_agent_types>
Valid GSD subagent types: Use the exact names. Do not use 'general-purpose' instead.
- gsd-debugger — Diagnoses and fixes problems
</available_agent_types>

<paths>
DEBUG_DIR=.planning/debug

Debug files use the `.planning/debug/` path. This hidden directory has a leading dot.
</paths>

<core_principle>
**Diagnose before planning fixes.**

UAT shows what is broken (symptoms). Debug agents find the cause (root cause). plan-phase --gaps then creates fixes based on actual causes, not guesses.

Without diagnosis: "Comment does not refresh" → guess at fix → maybe wrong
With diagnosis: "Comment does not refresh" → "useEffect missing dependency" → fix for the cause
</core_principle>

<process>

<step name="parse_gaps">
**Extract gaps from UAT.md:**

Read the "Gaps" section (YAML format):
```yaml
- truth: "Comment appears immediately after submission"
  status: failed
  reason: "User reported: works but doesn't show until I refresh the page"
  severity: major
  test: 2
  artifacts: []
  missing: []
```

For each gap, also read the related test from the "Tests" section to get the full context.

Build the gap list:
```
gaps = [
  {truth: "Comment appears immediately...", severity: "major", test_num: 2, reason: "..."},
  {truth: "Reply button positioned correctly...", severity: "minor", test_num: 5, reason: "..."},
  ...
]
```
</step>

<step name="report_plan">
**Read worktree config:**

```bash
_GSD_SHIM_NAME="gsd-tools.cjs"; _GSD_RUNTIME_ROOT="${RUNTIME_DIR:-$(git rev-parse --show-toplevel 2>/dev/null || pwd)}"; GSD_TOOLS="${_GSD_RUNTIME_ROOT}/gsd-core/bin/${_GSD_SHIM_NAME}"; _gsd_at() { for _p; do if [ -f "$_p" ]; then GSD_TOOLS="$_p"; return 0; fi; done; return 1; }; _gsd_id_ok() { case "$("$1" runtime-identity --raw 2>/dev/null || true)" in '{"packageName":"@opengsd/gsd-core"'*'}') return 0;; *) return 1;; esac; }; _gsd_homes() { set -- "${CLAUDE_CONFIG_DIR:-$HOME/.claude}" "${ANTIGRAVITY_CONFIG_DIR:-$HOME/.gemini/antigravity}" "$HOME/.gemini/antigravity-ide" "$HOME/.gemini/antigravity-cli" "${AUGMENT_CONFIG_DIR:-$HOME/.augment}" "${CLINE_CONFIG_DIR:-$HOME/.cline}" "${CODEBUDDY_CONFIG_DIR:-$HOME/.codebuddy}" "${CODEX_HOME:-$HOME/.codex}" "${COPILOT_CONFIG_DIR:-${COPILOT_HOME:-$HOME/.copilot}}" "${CURSOR_CONFIG_DIR:-$HOME/.cursor}" "${HERMES_HOME:-$HOME/.hermes}" "${KILO_CONFIG_DIR:-${XDG_CONFIG_HOME:-$HOME/.config}/kilo}" "${KIMI_CONFIG_DIR:-$HOME/.config/agents}" "$HOME/.agents" "${KIMI_CODE_HOME:-$HOME/.kimi-code}" "${OPENCODE_CONFIG_DIR:-${XDG_CONFIG_HOME:-$HOME/.config}/opencode}" "${PI_CODING_AGENT_DIR:-$HOME/.pi/agent}" "${QWEN_CONFIG_DIR:-$HOME/.qwen}" "${TRAE_CONFIG_DIR:-$HOME/.trae}" "${WINDSURF_CONFIG_DIR:-$HOME/.codeium/windsurf}" "${ZCODE_CONFIG_DIR:-$HOME/.zcode}" "${GROK_AGENTS_HOME:-$HOME/.agents}"; for _h; do _gsd_at "$_h/gsd-core/bin/${_GSD_SHIM_NAME}" && return 0; done; return 1; }; if _gsd_at "${_GSD_RUNTIME_ROOT}/gsd-core/bin/${_GSD_SHIM_NAME}" "${_GSD_RUNTIME_ROOT}/.claude/gsd-core/bin/${_GSD_SHIM_NAME}" "${_GSD_RUNTIME_ROOT}/.codex/gsd-core/bin/${_GSD_SHIM_NAME}"; then gsd_run() { node "$GSD_TOOLS" "$@"; }; elif _gsd_homes; then gsd_run() { node "$GSD_TOOLS" "$@"; }; elif unset -f gsd_run; _G="$(command -v gsd_run)"; [ -n "$_G" ] && _gsd_id_ok "$_G"; then GSD_TOOLS="$_G"; gsd_run() { "$GSD_TOOLS" "$@"; }; else echo "ERROR: gsd-tools.cjs not found at $GSD_TOOLS and no identity-proving gsd_run is on PATH. Run: npx -y @opengsd/gsd-core@latest --claude --local" >&2; exit 1; fi; GSD_IDENTITY_STATUS=unverified; _gsd_id_ok gsd_run && GSD_IDENTITY_STATUS=ok; export GSD_IDENTITY_STATUS; [ "$GSD_IDENTITY_STATUS" = ok ] || echo "WARNING: \"$GSD_TOOLS\" did not prove it is @opengsd/gsd-core - it is either a different package or an @opengsd/gsd-core older than the runtime-identity verb. See docs/how-to/diagnose-a-foreign-gsd-tools.md" >&2; if [ -n "${CLAUDE_ENV_FILE:-}" ] && [ -n "${GSD_TOOLS:-}" ]; then printf "export PATH='%s':\"\$PATH\"\n" "${GSD_TOOLS%/*}" >> "$CLAUDE_ENV_FILE" 2>/dev/null || true; fi
USE_WORKTREES=$(gsd_run query config-get workflow.use_worktrees --raw 2>/dev/null || echo "true")
RUNTIME=$(gsd_run query config-get runtime --default claude --raw 2>/dev/null || echo "claude")
```

**Resolve isolation now (#2584/#2652).** Read @gsd-core/references/dispatch-isolation-gate.md.
Run its `Resolve ISOLATION`, `Single-agent dispatch sites`, and `Resolve the harness flag` blocks in order.

The blocks set `ISOLATION`/`HARNESS_FLAG` through `query dispatch-isolation`.
`ISOLATION`, not `RUNTIME`, controls the worktree decision at spawn.
The `spawn_agents` step below uses both variables.

**Report diagnosis plan to user:**

```
## Diagnosing {N} Gaps

Spawning parallel debug agents to investigate root causes:

| Gap (Truth) | Severity |
|-------------|----------|
| Comment appears immediately after submission | major |
| Reply button positioned correctly | minor |
| Delete removes comment | blocker |

Each agent will:
1. Create .planning/debug/{slug}.md with symptoms pre-filled
2. Investigate autonomously (read code, form hypotheses, test)
3. Return root cause

This runs in parallel - all gaps investigated simultaneously.
```
</step>

<step name="spawn_agents">
**Load agent skills:**

```bash
IFS= read -rd '' GSD_ARGS <<'GSD_ARGS_END' || true
$ARGUMENTS
GSD_ARGS_END
GSD_ARGS=${GSD_ARGS%$'\n'}
GSD_WS=$(echo " $GSD_ARGS" | sed -nE 's/.* --ws +([A-Za-z0-9][A-Za-z0-9._-]*).*/--ws \1/p' | head -n 1)
AGENT_SKILLS_DEBUGGER=$(gsd_run query agent-skills gsd-debugger ${GSD_WS:+--ws=${GSD_WS##* }})
EXPECTED_BASE=$(git rev-parse HEAD)
```

**Pre-dispatch worktree base-check (#2649, mirrors execute-phase #683/#1369 and quick #1941).**
Claude Code uses `isolation="worktree"` to fork new worktrees from `origin/HEAD`, not the current local HEAD.
The documented GSD steady state is to commit every step and push only on request.
If local HEAD advances without an intervening `git push`, `origin/HEAD` stays at an old ancestor.

In that case, the debug agent's `worktree_branch_check` guard stops with a fatal base-mismatch error *after* the worktree exists.
It does not automatically switch modes.
Run the same pre-dispatch check that the four sibling dispatch sites run.
If the fork base cannot be resolved reliably, automatically switch to sequential (main-tree) debug-agent dispatch.
The verify-only `<worktree_branch_check>` guard below stays active as a backstop in both cases.

```bash
if [ "$ISOLATION" = "harness-worktree" ]; then
  _DIAG_SHOULD_DEGRADE=$(gsd_run query worktree.base-check --mode "$ISOLATION" --pick shouldDegrade 2>/dev/null || true)
  if [ "$_DIAG_SHOULD_DEGRADE" = "true" ]; then
    _DIAG_DEGRADE_MSG=$(gsd_run query worktree.base-check --mode "$ISOLATION" --pick message 2>/dev/null || true)
    [ -n "$_DIAG_DEGRADE_MSG" ] && printf '%s\n' "$_DIAG_DEGRADE_MSG" >&2
    echo "⚠ [#2649] Worktree fork base diverged from orchestrator HEAD — auto-degrading to sequential mode for diagnosis to avoid a base-mismatch halt." >&2
    ISOLATION=none
    USE_WORKTREES=false
  fi
fi

# Re-record after the base-check degrade, immediately before the spawn below, so the
# #3045 sentinel matches the dispatch the guard is about to see (#3045).
gsd_run query dispatch-isolation --raw --force-isolation "$ISOLATION" >/dev/null 2>&1 || true

# Model resolution for the debugger spawns below (#3602).
DEBUGGER_MODEL=$(gsd_run query resolve-model gsd-debugger --raw)
```

**Spawn debug agents — use parallel dispatch only when each agent is isolated:**

**`ISOLATION` decides the fan-out, not only the flag (#2652).** If `ISOLATION = "harness-worktree"`, spawn all agents in a single message.
Each agent gets its own worktree, so concurrent edits cannot collide.

If `ISOLATION = "none"`, spawn agents **one at a time**.
Wait for each agent to return before you spawn the next agent.
This condition includes the `orchestrator-worktree` fallback and the #2649 base-check degrade.
In these cases, all agents run against the **primary checkout**.

The `orchestrator-worktree` degrade prevents parallel dispatch of unisolated debuggers.
A flag degrade with parallel dispatch would announce sequential mode but do the opposite.

For each gap, fill this prompt as `filled_debug_prompt`. The `Template placeholders` below define every `{…}` field. Then spawn:

```markdown
<objective>
Investigate issue: {slug}

**Summary:** {truth}
</objective>

<symptoms>
expected: {expected}
actual: {actual}
errors: {errors}
reproduction: {reproduction}
timeline: {timeline}
</symptoms>

<mode>
symptoms_prefilled: true
goal: find_root_cause_only
</mode>

<debug_file>
Create: .planning/debug/{slug}.md
</debug_file>
```

Print: `◆ Spawning diagnostics agent... (each runs in a subagent — no output until they return, ~1–5 min; expected, not a freeze)`

If `ISOLATION = "harness-worktree"`, prepare WORKTREE_GUARD before you spawn the agent.
Read `gsd-core/references/worktree-branch-check.md`.
Replace `{EXPECTED_BASE}` with `$EXPECTED_BASE`.
Use the resulting `<worktree_branch_check>` block (the runnable guard) as WORKTREE_GUARD below.

If `ISOLATION = "none"`, set `WORKTREE_GUARD` to the empty string instead.
The agent runs on the main working tree, where the guard's HEAD assertion cannot hold.
Otherwise, every diagnostic agent stops on a base mismatch that it was not meant to check (#2652).

If `ISOLATION = "harness-worktree"`, **replace `{harnessFlag}` in the `Agent()` call below** with `$HARNESS_FLAG` followed by a comma.
Otherwise, replace it with the empty string.
This is the same build-time substitution that `execute-phase.md` uses.
`{harnessFlag}` is a template placeholder, not a shell variable.

<!-- #2508 runtime-aware-dispatch -->

> **Runtime-aware dispatch (#2508 Phase 4).** GSD workflows dispatch specialized subagents by role.
> kimi-code is a built-in-only runtime with three built-ins only.
> Before dispatch on a built-in-only runtime, resolve the role to a built-in with `gsd_run query resolve-dispatch-type --requested <role> --raw`.

> On named-dispatch runtimes (Claude/OpenCode/…), the command returns the role unchanged.
> On kimi-code, the command maps the role to `coder`/`explore`/`plan` by role-suffix.
> In both cases, the persona uses `${AGENT_SKILLS_<ROLE>}` (Phase 3).
> See @gsd-core/references/runtime-aware-dispatch.md.

<!-- #2517 model-omit-on-inherit -->

> **Model omission (#2517).** If `DEBUGGER_MODEL` is `"inherit"` or empty, omit the `model` parameter entirely.
> An empty value causes a 404 on runtimes without native tier aliases.
> This is the default on non-Claude runtimes.

> If you omit the parameter, the agent inherits the orchestrator's model.
> See @gsd-core/references/model-profile-resolution.md.

```
Agent(
  prompt=filled_debug_prompt + "\n\n" + WORKTREE_GUARD + "\n\n<required_reading>\n- {phase_dir}/{phase_num}-UAT.md\n- {state_path}\n</required_reading>\n${AGENT_SKILLS_DEBUGGER}",
  subagent_type="gsd-debugger",
  model="{DEBUGGER_MODEL}",
  {harnessFlag}
  description="Debug: {truth_short}"
)
```

> **ORCHESTRATOR RULE — CODEX RUNTIME**: After you call Agent() above to spawn debug agent(s), stop work on this task immediately.
> While the subagent(s) are active, do not read more files, edit code, or run tests related to these gaps.
> Wait for all subagents to return before you continue.
> This prevents duplicate work, conflicting edits, and wasted context.

**Only if `ISOLATION = "harness-worktree"`, spawn all agents in a single message (parallel execution).**
If `ISOLATION = "none"`, spawn one agent per message. Wait for each agent to return.
See the fan-out rule above (#2652).

Template placeholders:
- `{truth}`: The expected behavior that failed
- `{expected}`: From UAT test
- `{actual}`: Verbatim user description from reason field
- `{errors}`: Any error messages from UAT (or "None reported")
- `{reproduction}`: "Test {test_num} in UAT"
- `{timeline}`: "Discovered during UAT"
- `{slug}`: Generated from truth
</step>

<step name="collect_results">
**Collect root causes from agents:**

Each agent returns with:
```
## ROOT CAUSE FOUND

**Debug Session:** ${DEBUG_DIR}/{slug}.md

**Root Cause:** {specific cause with evidence}

**Evidence Summary:**
- {key finding 1}
- {key finding 2}
- {key finding 3}

**Files Involved:**
- {file1}: {what's wrong}
- {file2}: {related issue}

**Suggested Fix Direction:** {brief hint for plan-phase --gaps}
```

Parse each return to extract:
- root_cause: The diagnosed cause
- files: Files involved
- debug_path: Path to the debug session file
- fix_hint: A NON-BINDING example route for the gap closure plan. The binding payload is `root_cause`.
  A gap plan that removes the root cause by a smaller or different mechanism closes the gap in full.

If an agent returns `## INVESTIGATION INCONCLUSIVE`:
- Set root_cause to "Investigation inconclusive - manual review needed"
- Note which problem needs manual attention
- Include the remaining possibilities from the agent return
</step>

<step name="update_uat">
**Update UAT.md gaps with diagnosis:**

For each gap in the Gaps section, add the artifacts and missing fields:

```yaml
- truth: "Comment appears immediately after submission"
  status: failed
  reason: "User reported: works but doesn't show until I refresh the page"
  severity: major
  test: 2
  root_cause: "useEffect in CommentList.tsx missing commentCount dependency"
  artifacts:
    - path: "src/components/CommentList.tsx"
      issue: "useEffect missing dependency"
  missing:
    - "Add commentCount to useEffect dependency array"
    - "Trigger re-render when new comment added"
  debug_session: .planning/debug/comment-not-refreshing.md
```

Update status in frontmatter to "diagnosed".

Commit the updated UAT.md:
```bash
gsd_run query commit "docs({phase_num}): add root causes from diagnosis" --files ".planning/phases/XX-name/{phase_num}-UAT.md"
```
</step>

<step name="report_results">
**Report diagnosis results and pass control:**

Show:
```
### GSD ► DIAGNOSIS COMPLETE

| Gap (Truth) | Root Cause | Files |
|-------------|------------|-------|
| Comment appears immediately | useEffect missing dependency | CommentList.tsx |
| Reply button positioned correctly | CSS flex order incorrect | ReplyButton.tsx |
| Delete removes comment | API missing auth header | api/comments.ts |

Debug sessions: ${DEBUG_DIR}/

Proceeding to plan fixes...
```

Return to the verify-work orchestrator for automatic planning.
Do NOT offer manual next steps. verify-work handles the rest.
</step>

</process>

<context_efficiency>
Agents start with symptoms filled from UAT (no symptom gathering).
Agents only diagnose. plan-phase --gaps handles fixes (no fix application).
</context_efficiency>

<failure_handling>
**If an agent fails to find the root cause:**
- Mark the gap as "needs manual review"
- Continue with other gaps
- Report the incomplete diagnosis

**If an agent times out:**
- Examine .planning/debug/{slug}.md for partial progress
- You can resume with /gsd:debug

**If all agents fail:**
- The cause is systemic, for example permissions or git
- Report for manual investigation
- Use plan-phase --gaps without root causes instead (less precise)
</failure_handling>

<success_criteria>
- [ ] Parse gaps from UAT.md
- [ ] Spawn debug agents in parallel
- [ ] Collect root causes from all agents
- [ ] Update UAT.md gaps with artifacts and missing
- [ ] Save debug sessions to ${DEBUG_DIR}/
- [ ] Return to verify-work for automatic planning
</success_criteria>
