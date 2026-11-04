@~/.claude/gsd-core/references/response-language-directive.md

<purpose>
Direct parallel codebase mapper agents to analyze the codebase. Produce structured documents in .planning/codebase/.

Each agent has fresh context. Each agent examines a specific focus area and **writes documents directly**.

The orchestrator receives only confirmation and line counts. Then it writes a summary.

Output: .planning/codebase/ folder with 7 structured documents about the codebase state.
</purpose>

<available_agent_types>
Valid GSD subagent types (use exact names — do not fall back to 'general-purpose'):
- gsd-codebase-mapper — Maps project structure and dependencies
</available_agent_types>

<philosophy>
**Why dedicated mapper agents:**
- Fresh context per domain (no token contamination)
- Agents write documents directly (no context transfer back to orchestrator)
- Orchestrator only summarizes what was created (minimal context usage)
- Faster execution (agents run simultaneously)

**Document quality over length:**
Include enough detail for use as a reference. Give practical examples, especially code patterns, priority over arbitrary brevity.

**Always include file paths:**
Claude uses the documents as reference material for planning/executing. Always include actual file paths formatted with backticks: `src/services/user.ts`.
</philosophy>

<process>

<step name="parse_paths_flag" priority="first">
Parse an optional `--paths <p1,p2,...>` argument.

If the post-execute codebase-drift gate in `/gsd:execute-phase` supplies it, the workflow uses **incremental-remap mode**. The workflow also uses this mode if a user runs `/gsd:map-codebase --paths apps/accounting,packages/ui`:

- Pass `--paths <p1>,<p2>,...` to each spawned `gsd-codebase-mapper` agent's prompt. Agents restrict their Glob/Grep/Bash exploration to the listed repo-relative prefixes only. They do not scan the whole repo.
- Reject path values that contain `..`, start with `/`, or include shell
  metacharacters (`;`, `` ` ``, `$`, `&`, `|`, `<`, `>`). If all provided
  paths are invalid, fall back to a normal whole-repo run.
- The mapper does NOT stamp the `last_mapped_commit` baseline. The `stamp_codebase_map` step below stamps it deterministically on every run, incremental or full. See that step for the reason.

**Explicit contract — pass `--paths` through a single normalized variable.** Downstream steps (`spawn_agents`, `sequential_mapping`, and any Agent-mode prompt construction) MUST use `${PATH_SCOPE_HINT}`.

This makes sure that every mapper receives the same deterministic scope. Without this contract, incremental-remap can silently return to a whole-repo scan.

```bash
# Validated, comma-separated paths (empty if --paths absent or all rejected):
SCOPED_PATHS="<validated paths or empty>"
if [ -n "$SCOPED_PATHS" ]; then
  PATH_SCOPE_HINT="--paths $SCOPED_PATHS"
else
  PATH_SCOPE_HINT=""
fi
```

All mapper prompts built later in this workflow MUST include `${PATH_SCOPE_HINT}`. In full-repo mode, expand it to empty.

When `--paths` is absent, behave exactly as before. Scan the full repo and refresh all 7 documents.
</step>

<step name="init_context" priority="first">
Load codebase mapping context:

```bash
IFS= read -rd '' GSD_ARGS <<'GSD_ARGS_END' || true
$ARGUMENTS
GSD_ARGS_END
GSD_ARGS=${GSD_ARGS%$'\n'}
_GSD_SHIM_NAME="gsd-tools.cjs"; _GSD_RUNTIME_ROOT="${RUNTIME_DIR:-$(git rev-parse --show-toplevel 2>/dev/null || pwd)}"; GSD_TOOLS="${_GSD_RUNTIME_ROOT}/gsd-core/bin/${_GSD_SHIM_NAME}"; _gsd_at() { for _p; do if [ -f "$_p" ]; then GSD_TOOLS="$_p"; return 0; fi; done; return 1; }; _gsd_id_ok() { case "$("$1" runtime-identity --raw 2>/dev/null || true)" in '{"packageName":"@opengsd/gsd-core"'*'}') return 0;; *) return 1;; esac; }; _gsd_homes() { set -- "${CLAUDE_CONFIG_DIR:-$HOME/.claude}" "${ANTIGRAVITY_CONFIG_DIR:-$HOME/.gemini/antigravity}" "$HOME/.gemini/antigravity-ide" "$HOME/.gemini/antigravity-cli" "${AUGMENT_CONFIG_DIR:-$HOME/.augment}" "${CLINE_CONFIG_DIR:-$HOME/.cline}" "${CODEBUDDY_CONFIG_DIR:-$HOME/.codebuddy}" "${CODEX_HOME:-$HOME/.codex}" "${COPILOT_CONFIG_DIR:-${COPILOT_HOME:-$HOME/.copilot}}" "${CURSOR_CONFIG_DIR:-$HOME/.cursor}" "${HERMES_HOME:-$HOME/.hermes}" "${KILO_CONFIG_DIR:-${XDG_CONFIG_HOME:-$HOME/.config}/kilo}" "${KIMI_CONFIG_DIR:-$HOME/.config/agents}" "$HOME/.agents" "${KIMI_CODE_HOME:-$HOME/.kimi-code}" "${OPENCODE_CONFIG_DIR:-${XDG_CONFIG_HOME:-$HOME/.config}/opencode}" "${PI_CODING_AGENT_DIR:-$HOME/.pi/agent}" "${QWEN_CONFIG_DIR:-$HOME/.qwen}" "${TRAE_CONFIG_DIR:-$HOME/.trae}" "${WINDSURF_CONFIG_DIR:-$HOME/.codeium/windsurf}" "${ZCODE_CONFIG_DIR:-$HOME/.zcode}" "${GROK_AGENTS_HOME:-$HOME/.agents}"; for _h; do _gsd_at "$_h/gsd-core/bin/${_GSD_SHIM_NAME}" && return 0; done; return 1; }; if _gsd_at "${_GSD_RUNTIME_ROOT}/gsd-core/bin/${_GSD_SHIM_NAME}" "${_GSD_RUNTIME_ROOT}/.claude/gsd-core/bin/${_GSD_SHIM_NAME}" "${_GSD_RUNTIME_ROOT}/.codex/gsd-core/bin/${_GSD_SHIM_NAME}"; then gsd_run() { node "$GSD_TOOLS" "$@"; }; elif _gsd_homes; then gsd_run() { node "$GSD_TOOLS" "$@"; }; elif unset -f gsd_run; _G="$(command -v gsd_run)"; [ -n "$_G" ] && _gsd_id_ok "$_G"; then GSD_TOOLS="$_G"; gsd_run() { "$GSD_TOOLS" "$@"; }; else echo "ERROR: gsd-tools.cjs not found at $GSD_TOOLS and no identity-proving gsd_run is on PATH. Run: npx -y @opengsd/gsd-core@latest --claude --local" >&2; exit 1; fi; GSD_IDENTITY_STATUS=unverified; _gsd_id_ok gsd_run && GSD_IDENTITY_STATUS=ok; export GSD_IDENTITY_STATUS; [ "$GSD_IDENTITY_STATUS" = ok ] || echo "WARNING: \"$GSD_TOOLS\" did not prove it is @opengsd/gsd-core - it is either a different package or an @opengsd/gsd-core older than the runtime-identity verb. See docs/how-to/diagnose-a-foreign-gsd-tools.md" >&2; if [ -n "${CLAUDE_ENV_FILE:-}" ] && [ -n "${GSD_TOOLS:-}" ]; then printf "export PATH='%s':\"\$PATH\"\n" "${GSD_TOOLS%/*}" >> "$CLAUDE_ENV_FILE" 2>/dev/null || true; fi
GSD_WS=$(echo " $GSD_ARGS" | sed -nE 's/.* --ws +([A-Za-z0-9][A-Za-z0-9._-]*).*/--ws \1/p' | head -n 1)
INIT=$(gsd_run query init.map-codebase ${GSD_WS:+--ws=${GSD_WS##* }})
if [[ "$INIT" == @file:* ]]; then INIT=$(cat "${INIT#@file:}"); fi
AGENT_SKILLS_MAPPER=$(gsd_run query agent-skills gsd-codebase-mapper ${GSD_WS:+--ws=${GSD_WS##* }})
```

Extract from init JSON: `mapper_model`, `commit_docs`, `codebase_dir`, `existing_maps`, `has_maps`, `codebase_dir_exists`, `subagent_timeout`, `date`.
</step>

<step name="check_existing">
Use `has_maps` from init context to find out if .planning/codebase/ already exists.

If `codebase_dir_exists` is true:
```bash
ls -la .planning/codebase/
```

**If exists:**

```
.planning/codebase/ already exists with these documents:
[List files found]

What's next?
1. Refresh - Delete existing and remap codebase
2. Update - Keep existing, only update specific documents
3. Skip - Use existing codebase map as-is
```

Wait for user response.

If "Refresh": Delete .planning/codebase/. Continue to create_structure.
If "Update": Ask which documents to update. Record the selection for the stamp step below. Continue to spawn_agents (filtered):

```bash
# Comma-separated filenames the user selected, e.g. "STACK.md,CONCERNS.md":
UPDATED_DOCS="<selected documents>"
```

If "Skip": Exit workflow

`UPDATED_DOCS` narrows `stamp_codebase_map`. Leave it empty on every other path (Refresh, first run, `--paths`). These paths regenerate all seven documents.

An Update run does not touch the documents the user did not select. If the workflow stamps those documents at HEAD, it claims freshness that they do not have.

**If does not exist:**
Continue to create_structure.
</step>

<step name="create_structure">
Create .planning/codebase/ directory:

```bash
mkdir -p .planning/codebase
```

**Expected output files:**
- STACK.md (from tech mapper)
- INTEGRATIONS.md (from tech mapper)
- ARCHITECTURE.md (from arch mapper)
- STRUCTURE.md (from arch mapper)
- CONVENTIONS.md (from quality mapper)
- TESTING.md (from quality mapper)
- CONCERNS.md (from concerns mapper)

Continue to spawn_agents.
</step>

<step name="detect_runtime_capabilities">
Before you spawn agents, find out whether the current runtime supports the `Agent` tool for subagent delegation.

**How to detect:** Find out if you have access to an `Agent` tool. The name may be `Agent` or `agent`, depending on the runtime.

If you do NOT have an `Agent`/`agent` tool, skip `spawn_agents` and `collect_confirmations`. Also skip them if you only have tools like `browser_subagent`, which is for web browsing, NOT code analysis. Go directly to `sequential_mapping` instead.

**CRITICAL:** Never use `browser_subagent` or `Explore` as a substitute for `Agent`. The `browser_subagent` tool is only for web page interaction. It will fail for codebase analysis. If `Agent` is unavailable, do the mapping sequentially in-context.
</step>

<step name="spawn_agents" condition="Agent tool is available">
Spawn 4 parallel gsd-codebase-mapper agents.

<!-- #2508 runtime-aware-dispatch -->

> **Runtime-aware dispatch (#2508 Phase 4).** GSD workflows dispatch specialized subagents by role. Before dispatch on a built-in-only runtime (kimi-code — three built-ins only), resolve the role with `gsd_run query resolve-dispatch-type --requested <role> --raw`. On named-dispatch runtimes (Claude/OpenCode/…) the command returns the role unchanged. On kimi-code the command maps the role to `coder`/`explore`/`plan` by role-suffix. In both cases, `${AGENT_SKILLS_<ROLE>}` carries the persona (Phase 3). See @gsd-core/references/runtime-aware-dispatch.md.

<!-- #2517 model-omit-on-inherit -->

> **Model omission (#2517).** If the value for `model` (`mapper_model`) is `"inherit"` or empty, omit the `model` parameter entirely. An empty value causes a 404 on runtimes without native tier aliases. This is the default on non-Claude runtimes. If you omit `model`, the agent inherits the orchestrator's model. See @gsd-core/references/model-profile-resolution.md.

Use Agent tool with `subagent_type="gsd-codebase-mapper"`, `model="{mapper_model}"`, and `run_in_background=true` for parallel execution.

**CRITICAL:** Use the dedicated `gsd-codebase-mapper` agent, NOT `Explore` or `browser_subagent`. The mapper agent writes documents directly.

Print: "Spawning 4 parallel codebase mapper agents (each runs in a subagent — no output until they return, ~1–5 min. Expected, not a freeze)"

**Agent 1: Tech Focus**

```text
Agent(
  subagent_type="gsd-codebase-mapper",
  model="{mapper_model}",
  run_in_background=true,
  description="Map codebase tech stack",
  prompt="Focus: tech
Today's date: {date}

Analyze this codebase for technology stack and external integrations.

Write these documents to {codebase_dir}/:
- STACK.md - Languages, runtime, frameworks, dependencies, configuration
- INTEGRATIONS.md - External APIs, databases, auth providers, webhooks

IMPORTANT: Set all date stamps (`**Analysis Date:**`, footer `*... analysis: ...*`, `<!-- refreshed: ... -->`) to {date}, overwriting any existing date.

Scope: ${PATH_SCOPE_HINT:-(full repo)} — when --paths is supplied, restrict exploration to those prefixes only.

Explore thoroughly. Write documents directly using templates. Return confirmation only.
${AGENT_SKILLS_MAPPER}"
)
```

**Agent 2: Architecture Focus**

```text
Agent(
  subagent_type="gsd-codebase-mapper",
  model="{mapper_model}",
  run_in_background=true,
  description="Map codebase architecture",
  prompt="Focus: arch
Today's date: {date}

Analyze this codebase architecture and directory structure.

Write these documents to {codebase_dir}/:
- ARCHITECTURE.md - Pattern, layers, data flow, abstractions, entry points
- STRUCTURE.md - Directory layout, key locations, naming conventions

IMPORTANT: Set all date stamps (`**Analysis Date:**`, footer `*... analysis: ...*`, `<!-- refreshed: ... -->`) to {date}, overwriting any existing date.

Scope: ${PATH_SCOPE_HINT:-(full repo)} — when --paths is supplied, restrict exploration to those prefixes only.

Explore thoroughly. Write documents directly using templates. Return confirmation only.
${AGENT_SKILLS_MAPPER}"
)
```

**Agent 3: Quality Focus**

```text
Agent(
  subagent_type="gsd-codebase-mapper",
  model="{mapper_model}",
  run_in_background=true,
  description="Map codebase conventions",
  prompt="Focus: quality
Today's date: {date}

Analyze this codebase for coding conventions and testing patterns.

Write these documents to {codebase_dir}/:
- CONVENTIONS.md - Code style, naming, patterns, error handling
- TESTING.md - Framework, structure, mocking, coverage

IMPORTANT: Set all date stamps (`**Analysis Date:**`, footer `*... analysis: ...*`, `<!-- refreshed: ... -->`) to {date}, overwriting any existing date.

Scope: ${PATH_SCOPE_HINT:-(full repo)} — when --paths is supplied, restrict exploration to those prefixes only.

Explore thoroughly. Write documents directly using templates. Return confirmation only.
${AGENT_SKILLS_MAPPER}"
)
```

**Agent 4: Concerns Focus**

```
Agent(
  subagent_type="gsd-codebase-mapper",
  model="{mapper_model}",
  run_in_background=true,
  description="Map codebase concerns",
  prompt="Focus: concerns
Today's date: {date}

Analyze this codebase for technical debt, known issues, and areas of concern.

Write this document to {codebase_dir}/:
- CONCERNS.md - Tech debt, bugs, security, performance, fragile areas

IMPORTANT: Set all date stamps (`**Analysis Date:**`, footer `*... analysis: ...*`, `<!-- refreshed: ... -->`) to {date}, overwriting any existing date.

Scope: ${PATH_SCOPE_HINT:-(full repo)} — when --paths is supplied, restrict exploration to those prefixes only.

Explore thoroughly. Write document directly using template. Return confirmation only.
${AGENT_SKILLS_MAPPER}"
)
```

> **ORCHESTRATOR RULE — CODEX RUNTIME**: After all 4 Agent() calls above with `run_in_background=true`, do NOT read source files while the subagents are active. Do NOT analyze the codebase or independently write mapping documents while the subagents are active. Wait for all 4 agents to complete before you continue to collect_confirmations. This prevents duplicate work and wasted context.

Continue to collect_confirmations.
</step>

<step name="collect_confirmations">
Wait for all 4 background agents to finish. Then read each agent's output file to collect confirmations.

Each `Agent(...)` call above with `run_in_background=true` returns an `async_launched` result. The result carries an `outputFile` path (and `canReadOutputFile: true`). The 4 agents run at the same time. When each agent finishes, a completion message arrives in this conversation. Do NOT issue a separate blocking call to wait for them.

**After all 4 agents report completion, read each agent's output file (single message with 4 Read calls):**
```
Read tool:
  file_path: "{outputFile from that agent's async_launched result}"
```

> Allow up to `workflow.subagent_timeout` for the slowest agent to finish before you treat it as failed. You can configure the timeout with `workflow.subagent_timeout` in `.planning/config.json` (milliseconds). Default: 300000 (5 minutes). Increase it for large codebases or slower models.

Each output file contains that agent's completion confirmation. Parse the confirmation marker (see below) from the file contents.

**Expected confirmation format from each agent:**
```
## Mapping Complete

**Focus:** {focus}
**Documents written:**
- `.planning/codebase/{DOC1}.md` ({N} lines)
- `.planning/codebase/{DOC2}.md` ({N} lines)

Ready for orchestrator summary.
```

**What you receive:** Only file paths and line counts. NOT document contents.

If any agent failed, note the failure and continue with successful documents.

Continue to verify_output.
</step>

<step name="sequential_mapping" condition="Agent tool is NOT available (e.g. Antigravity, Codex)">
When the `Agent` tool is unavailable, do codebase mapping sequentially in the current context. This replaces `spawn_agents` and `collect_confirmations`.

**IMPORTANT:** Do NOT use `browser_subagent`, `Explore`, or any browser-based tool. Use only file system tools (Read, Bash, Write, Grep, Glob, list_dir, view_file, grep_search, or equivalent tools available in your runtime).

**IMPORTANT:** Set all date stamps (`**Analysis Date:**`, footer, `<!-- refreshed -->`) to `{date}` from init context. Overwrite any existing date. Update runs seed from files with concrete before dates. Replacing only `[YYYY-MM-DD]` placeholders is not enough. NEVER guess the date.

**SCOPE:** When `${PATH_SCOPE_HINT}` is non-empty (`--paths` was supplied), restrict every pass below to the validated path prefixes in `${SCOPED_PATHS}`. Do NOT scan files outside those prefixes. When `${PATH_SCOPE_HINT}` is empty, do a full-repo scan.

Do all 4 mapping passes sequentially:

**Pass 1: Tech Focus**
- Explore package.json/Cargo.toml/go.mod/requirements.txt, config files, dependency trees
- Write `.planning/codebase/STACK.md` — Languages, runtime, frameworks, dependencies, configuration
- Write `.planning/codebase/INTEGRATIONS.md` — External APIs, databases, auth providers, webhooks

**Pass 2: Architecture Focus**
- Explore directory structure, entry points, module boundaries, data flow
- Write `.planning/codebase/ARCHITECTURE.md` — Pattern, layers, data flow, abstractions, entry points
- Write `.planning/codebase/STRUCTURE.md` — Directory layout, key locations, naming conventions

**Pass 3: Quality Focus**
- Explore code style, error handling patterns, test files, CI config
- Write `.planning/codebase/CONVENTIONS.md` — Code style, naming, patterns, error handling
- Write `.planning/codebase/TESTING.md` — Framework, structure, mocking, coverage

**Pass 4: Concerns Focus**
- Explore TODOs, known issues, fragile areas, security patterns
- Write `.planning/codebase/CONCERNS.md` — Tech debt, bugs, security, performance, fragile areas

Use the same document templates as the `gsd-codebase-mapper` agent. Include actual file paths formatted with backticks.

Continue to verify_output.
</step>

<step name="verify_output">
Make sure that all documents were created successfully:

```bash
ls -la .planning/codebase/
wc -l .planning/codebase/*.md
```

**Verification checklist:**
- All 7 documents exist
- No empty documents (each should have >20 lines)

If any documents are missing or empty, note which agents possibly failed.

Continue to stamp_codebase_map.
</step>

<step name="stamp_codebase_map">
Stamp the drift baseline into every document that the mapper just wrote:

```bash
gsd_run stamp-codebase-map ${UPDATED_DOCS:+--files "$UPDATED_DOCS"}
```

This writes `last_mapped_commit: <HEAD sha>` and `last_mapped_at: <date>` into the YAML frontmatter of each existing `.planning/codebase/*.md`. It runs on every mapping run, incremental (`--paths`) and full. `--files` narrows it to the documents that an Update run refreshed. `--paths` needs no narrowing because the workflow regenerates all seven documents with scoped content.

**Why this is a shell step and not an instruction to the mapper.** The stamp is the only machine-readable freshness marker. The `verify codebase-drift` gate reads it to decide what to diff HEAD against.

On an Update run, the mapper restamps human-readable markers (`**Analysis Date:**`, `<!-- refreshed: ... -->`) without conditions. A mapper can decide its work is already done and rewrite only the dates. The document then still looks fresh to a human.

If the same agent writes the machine-readable stamp, it reproduces exactly the failure that the stamp exists to detect. A confident agent cannot skip a shell step.

The command is non-blocking. If it runs outside a git repo or no documents exist, it emits `skipped` with a `reason`. If any entry in `failed` is non-empty, report `stamped` and `commit` in the summary. Otherwise continue silently.

Run the step in this position, before `commit_codebase_map`. The stamp then lands on documents the mapper just wrote. The same commit includes its markdown whitespace normalization. If you run `stamp-codebase-map` by hand against an already-committed map, it also reflows that map's whitespace.

Continue to scan_for_secrets.
</step>

<step name="scan_for_secrets">
**CRITICAL SECURITY CHECK:** Before you commit, scan output files for accidentally leaked secrets.

Run secret pattern detection:

```bash
# Check for common API key patterns in generated docs
grep -E '(sk-[a-zA-Z0-9]{20,}|sk_live_[a-zA-Z0-9]+|sk_test_[a-zA-Z0-9]+|ghp_[a-zA-Z0-9]{36}|gho_[a-zA-Z0-9]{36}|glpat-[a-zA-Z0-9_-]+|AKIA[A-Z0-9]{16}|xox[baprs]-[a-zA-Z0-9-]+|-----BEGIN.*PRIVATE KEY|eyJ[a-zA-Z0-9_-]+\.eyJ[a-zA-Z0-9_-]+\.)' .planning/codebase/*.md 2>/dev/null && SECRETS_FOUND=true || SECRETS_FOUND=false
```

**If SECRETS_FOUND=true:**

```
⚠️  SECURITY ALERT: Potential secrets detected in codebase documents!

Found patterns that look like API keys or tokens in:
[show grep output]

This would expose credentials if committed.

**Action required:**
1. Review the flagged content above
2. If these are real secrets, they must be removed before committing
3. Consider adding sensitive files to Claude Code "Deny" permissions

Pausing before commit. Reply "safe to proceed" if the flagged content is not actually sensitive, or edit the files first.
```

Wait for user confirmation before you continue to commit_codebase_map.

**If SECRETS_FOUND=false:**

Continue to commit_codebase_map.
</step>

<step name="commit_codebase_map">
Commit the codebase map:

```bash
gsd_run query commit "docs: map existing codebase" --files .planning/codebase/*.md
```

Continue to offer_next.
</step>

<step name="offer_next">
Show the completion summary and next steps.

**Get line counts:**
```bash
wc -l .planning/codebase/*.md
```

**Output format:**

```
Codebase mapping complete.

Created .planning/codebase/:
- STACK.md ([N] lines) - Technologies and dependencies
- ARCHITECTURE.md ([N] lines) - System design and patterns
- STRUCTURE.md ([N] lines) - Directory layout and organization
- CONVENTIONS.md ([N] lines) - Code style and patterns
- TESTING.md ([N] lines) - Test structure and practices
- INTEGRATIONS.md ([N] lines) - External services and APIs
- CONCERNS.md ([N] lines) - Technical debt and issues

---

## ▶ Next Up — [${PROJECT_CODE}] ${PROJECT_TITLE}

**Initialize project** — use codebase context for planning

`/clear` then:

`/gsd:new-project`

---

**Also available:**
- Re-run mapping: `/gsd:map-codebase`
- Review specific file: `cat .planning/codebase/STACK.md`
- Edit any document before proceeding

---
```

End workflow.
</step>

</process>

<success_criteria>
- .planning/codebase/ directory created
- If Agent tool available: 4 parallel gsd-codebase-mapper agents spawned with run_in_background=true
- If Agent tool NOT available: 4 sequential mapping passes done inline (never using browser_subagent)
- All 7 codebase documents exist
- No empty documents (each should have >20 lines)
- Clear completion summary with line counts
- User offered clear next steps in GSD style
</success_criteria>
