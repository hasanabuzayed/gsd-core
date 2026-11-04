<purpose>
Analyze freeform text from the user. Route the request to the most appropriate GSD command. This dispatcher never does the work itself.

Match the user intent to the best command. Confirm the route. Pass control to the command.
</purpose>

<required_reading>
Before starting, read all files that the invoking prompt's execution_context references.
</required_reading>

<process>

```bash
_GSD_SHIM_NAME="gsd-tools.cjs"; _GSD_RUNTIME_ROOT="${RUNTIME_DIR:-$(git rev-parse --show-toplevel 2>/dev/null || pwd)}"; GSD_TOOLS="${_GSD_RUNTIME_ROOT}/gsd-core/bin/${_GSD_SHIM_NAME}"; _gsd_at() { for _p; do if [ -f "$_p" ]; then GSD_TOOLS="$_p"; return 0; fi; done; return 1; }; _gsd_id_ok() { case "$("$1" runtime-identity --raw 2>/dev/null || true)" in '{"packageName":"@opengsd/gsd-core"'*'}') return 0;; *) return 1;; esac; }; _gsd_homes() { set -- "${CLAUDE_CONFIG_DIR:-$HOME/.claude}" "${ANTIGRAVITY_CONFIG_DIR:-$HOME/.gemini/antigravity}" "$HOME/.gemini/antigravity-ide" "$HOME/.gemini/antigravity-cli" "${AUGMENT_CONFIG_DIR:-$HOME/.augment}" "${CLINE_CONFIG_DIR:-$HOME/.cline}" "${CODEBUDDY_CONFIG_DIR:-$HOME/.codebuddy}" "${CODEX_HOME:-$HOME/.codex}" "${COPILOT_CONFIG_DIR:-${COPILOT_HOME:-$HOME/.copilot}}" "${CURSOR_CONFIG_DIR:-$HOME/.cursor}" "${HERMES_HOME:-$HOME/.hermes}" "${KILO_CONFIG_DIR:-${XDG_CONFIG_HOME:-$HOME/.config}/kilo}" "${KIMI_CONFIG_DIR:-$HOME/.config/agents}" "$HOME/.agents" "${KIMI_CODE_HOME:-$HOME/.kimi-code}" "${OPENCODE_CONFIG_DIR:-${XDG_CONFIG_HOME:-$HOME/.config}/opencode}" "${PI_CODING_AGENT_DIR:-$HOME/.pi/agent}" "${QWEN_CONFIG_DIR:-$HOME/.qwen}" "${TRAE_CONFIG_DIR:-$HOME/.trae}" "${WINDSURF_CONFIG_DIR:-$HOME/.codeium/windsurf}" "${ZCODE_CONFIG_DIR:-$HOME/.zcode}" "${GROK_AGENTS_HOME:-$HOME/.agents}"; for _h; do _gsd_at "$_h/gsd-core/bin/${_GSD_SHIM_NAME}" && return 0; done; return 1; }; if _gsd_at "${_GSD_RUNTIME_ROOT}/gsd-core/bin/${_GSD_SHIM_NAME}" "${_GSD_RUNTIME_ROOT}/.claude/gsd-core/bin/${_GSD_SHIM_NAME}" "${_GSD_RUNTIME_ROOT}/.codex/gsd-core/bin/${_GSD_SHIM_NAME}"; then gsd_run() { node "$GSD_TOOLS" "$@"; }; elif _gsd_homes; then gsd_run() { node "$GSD_TOOLS" "$@"; }; elif unset -f gsd_run; _G="$(command -v gsd_run)"; [ -n "$_G" ] && _gsd_id_ok "$_G"; then GSD_TOOLS="$_G"; gsd_run() { "$GSD_TOOLS" "$@"; }; else echo "ERROR: gsd-tools.cjs not found at $GSD_TOOLS and no identity-proving gsd_run is on PATH. Run: npx -y @opengsd/gsd-core@latest --claude --local" >&2; exit 1; fi; GSD_IDENTITY_STATUS=unverified; _gsd_id_ok gsd_run && GSD_IDENTITY_STATUS=ok; export GSD_IDENTITY_STATUS; [ "$GSD_IDENTITY_STATUS" = ok ] || echo "WARNING: \"$GSD_TOOLS\" did not prove it is @opengsd/gsd-core - it is either a different package or an @opengsd/gsd-core older than the runtime-identity verb. See docs/how-to/diagnose-a-foreign-gsd-tools.md" >&2; if [ -n "${CLAUDE_ENV_FILE:-}" ] && [ -n "${GSD_TOOLS:-}" ]; then printf "export PATH='%s':\"\$PATH\"\n" "${GSD_TOOLS%/*}" >> "$CLAUDE_ENV_FILE" 2>/dev/null || true; fi
RESPONSE_LANGUAGE=$(gsd_run query config-get response_language --raw --default "" 2>/dev/null || echo "")
```

**If `response_language` is set:** All user-facing output of this workflow MUST be in `{response_language}`. This includes narration between tool calls, status updates, progress notes, findings, questions, prompts, and explanations. Technical terms, code, file paths, and subagent prompts stay in English. Only user-facing output is translated.

<step name="validate">
**Make sure that input exists.**

**Text mode (`workflow.text_mode: true` in config or `--text` flag):** If `--text` is present in `$ARGUMENTS` OR `text_mode` from init JSON is `true`, set `TEXT_MODE=true`. When TEXT_MODE is active, replace every `AskUserQuestion` call with a plain-text numbered list. Ask the user to type their choice number.

Non-Claude runtimes, such as OpenAI Codex and Antigravity, need this because `AskUserQuestion` is not available.

If `$ARGUMENTS` is empty, ask via AskUserQuestion:

```
What would you like to do? Describe the task, bug, or idea and I'll route it to the right GSD command.
```

Wait for the response before you continue.
</step>

<step name="check_project">
**Check if the project exists.**

```bash
INIT=$(gsd_run query state.load 2>/dev/null)
```

Track whether `.planning/` exists. Some routes need it, but others do not.
</step>

<step name="route">
**Match intent to command.**

Compare `$ARGUMENTS` with these routing rules. The rules are in **most-specific first** order. Apply the **first matching** rule. Never let a generic keyword rule override a more specific operation that also matches.

Generic keywords include "set up", "spike", and "review". More specific operations include "set up this existing codebase", "wrap up the spike findings", and "review the changed source code".

| If the text describes... | Route to | Why |
|--------------------------|----------|-----|
| First-time setup for an existing codebase, brownfield onboarding, "onboard this codebase" | `/gsd:onboard` | Safe map → docs ingest → project setup sequence |
| Starting a new greenfield project, "set up", "initialize" (no existing codebase named) | `/gsd:new-project` | Needs full project initialization |
| Mapping or analyzing an existing codebase map | `/gsd:map-codebase` | Codebase discovery or refresh |
| A bug, error, crash, failure, or something broken | `/gsd:debug` | Needs systematic investigation |
| Wrapping up spikes, "package the spikes", "consolidate spike findings" | `/gsd:spike --wrap-up` | Package spike findings into reusable skill |
| Wrapping up sketches, "package the designs", "consolidate sketch findings" | `/gsd:sketch --wrap-up` | Package sketch findings into reusable skill |
| Spiking, "test if", "will this work", "experiment", "prove this out", validate feasibility | `/gsd:spike` | Throwaway experiment to validate feasibility |
| Sketching, "mockup", "what would this look like", "prototype the UI", "design this", explore visual direction | `/gsd:sketch` | Throwaway HTML mockups to explore design |
| Reviewing changed source code for bugs, security issues, or code quality ("code review the changes") | `/gsd:code-review` | Source review of phase-changed files |
| Requesting peer review of phase plans from another AI CLI ("plan review", "review the plan") | `/gsd:review` | Cross-AI plan review |
| Reviewing or hardening implemented UI ("visual audit", "review the UI") | `/gsd:ui-review` | Retroactive 6-pillar visual audit |
| Verifying security mitigations of a completed phase ("security check", "secure phase N") | `/gsd:secure-phase` | Retroactive threat-mitigation verification |
| Auditing milestone completion against original intent ("audit the milestone") | `/gsd:audit-milestone` | Milestone audit against original intent |
| An autonomous audit-to-fix pass ("audit and fix", "audit the repo and fix what it finds") | `/gsd:audit-fix` | Audit-to-fix pipeline |
| Generating or updating project documentation ("update the docs", "documentation update") | `/gsd:docs-update` | Docs verified against the codebase |
| Exploring, researching, comparing, or "how does X work" | `/gsd:explore` | Socratic ideation and idea routing |
| Discussing vision, "how should X look", brainstorming | `/gsd:discuss-phase` | Needs context gathering |
| Planning a specific phase or "plan phase N" | `/gsd:plan-phase` | Direct planning request |
| Executing a phase or "build phase N", "run phase N" (SDD dependency-aware wave execution) | `/gsd:execute-phase` | Direct execution request |
| Adding, inserting, removing, or editing phases in the roadmap ("multi-phase", roadmap phase management) | `/gsd:phase` | Roadmap phase CRUD |
| A complex task: refactoring, migration, multi-file architecture, system redesign | `/gsd:plan-phase` | Needs a full phase with plan/build cycle |
| Running all remaining phases automatically | `/gsd:autonomous` | Full autonomous execution |
| A review or quality concern about existing work | `/gsd:verify-work` | Needs verification |
| Checking progress, status, "where am I" | `/gsd:progress` | Status check |
| Resuming work, "pick up where I left off" | `/gsd:resume-work` | Session restoration |
| A note, idea, or "remember to..." | `/gsd:capture` | Capture for later |
| Adding tests, "write tests", "test coverage" | `/gsd:add-tests` | Test generation |
| Completing a milestone, shipping, releasing | `/gsd:complete-milestone` | Milestone lifecycle |
| A specific, actionable, small task (add feature, fix typo, update config) | `/gsd:quick` | Self-contained, single executor |

**Need `.planning/` directory:** All routes need it except `/gsd:new-project`, `/gsd:onboard`, `/gsd:map-codebase`, `/gsd:spike`, `/gsd:sketch`, and `/gsd:help`. If the project does not exist and the route needs it, suggest `/gsd:onboard` for existing codebases or `/gsd:new-project` for greenfield projects.

**Ambiguity handling:** If the text could reasonably match multiple routes, ask the user via AskUserQuestion. Give the top 2-3 options. For example:

```
"Refactor the authentication system" could be:
1. /gsd:plan-phase — Full planning cycle (recommended for multi-file refactors)
2. /gsd:quick — Quick execution (if scope is small and clear)

Which approach fits better?
```
</step>

<step name="display">
**Show the routing decision.**

```
### GSD ► ROUTING

**Input:** {first 80 chars of $ARGUMENTS}
**Routing to:** {chosen command}
**Reason:** {one-line explanation}
```
</step>

<step name="confirm">
**Confirm the route before dispatch (REQ-DO-03).**

Before you invoke anything, ask the user to confirm the route shown via AskUserQuestion:

```
Route to {chosen command}?
1. Yes — proceed with {chosen command} (recommended)
2. Choose a different command
3. Cancel — do not dispatch
```

- **Yes / proceed:** Continue to the dispatch step.
- **Choose a different command:** Present the 2-3 next-best routes from the routing table as options. Return to display + confirm with the new selection.
- **Cancel:** Stop. Do not invoke any command.

**TEXT_MODE:** Present the same choices as a plain-text numbered list. Ask the user to type their choice number, exactly like other AskUserQuestion calls in this workflow.
</step>

<step name="dispatch">
**Invoke the chosen command with only the arguments it accepts.**

Read the chosen command's frontmatter `argument-hint` in `commands/gsd/<name>.md`. Forward **only arguments that command accepts**. Do NOT pass the full freeform sentence.

- If the command expects a phase number or flags only, extract the phase number / flags from the input. Examples include `/gsd:verify-work [phase number]`, `/gsd:plan-phase`, and `/gsd:execute-phase`. If none was provided, extract it from context or ask via AskUserQuestion. Remove the surrounding prose.

- If the command explicitly accepts a freeform task description, forward the relevant portion of `$ARGUMENTS` as the description. Examples include `/gsd:quick`, `/gsd:debug`, `/gsd:spike`, and `/gsd:sketch`.

- If the command takes no arguments, invoke it without arguments.

After you invoke the command, stop. The dispatched command handles everything from here.
</step>

</process>

<success_criteria>
- [ ] Make sure that input is not empty
- [ ] Match intent to exactly one GSD command
- [ ] If needed, resolve ambiguity through a user question
- [ ] For routes that need the project, make sure that it exists
- [ ] Show the routing decision before dispatch
- [ ] The user confirms the route before dispatch (REQ-DO-03), with a TEXT_MODE equivalent
- [ ] Invoke the command with only the arguments it accepts, as specified by argument-hint
- [ ] Use freeform text only where the command takes a freeform description
- [ ] Do no work directly — dispatcher only
</success_criteria>
