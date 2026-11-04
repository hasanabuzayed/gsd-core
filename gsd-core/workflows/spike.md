<purpose>
Explore an idea through focused experiments. Build experiments to experience parts of a future app and test feasibility. Produce verified knowledge for the real build.

Save artifacts to `.planning/spikes/`. This workflow is a companion to `/gsd:spike --wrap-up`.

The workflow supports two modes:
- **Idea mode** (default) — the user describes an idea to spike.
- **Frontier mode** — no argument or "frontier" / "what should I spike?" — the workflow analyzes existing spikes and proposes integration and frontier spikes.
</purpose>

<required_reading>
Before you start, read all files that the invoking prompt's execution_context references.
</required_reading>

<process>
```bash
_GSD_SHIM_NAME="gsd-tools.cjs"; _GSD_RUNTIME_ROOT="${RUNTIME_DIR:-$(git rev-parse --show-toplevel 2>/dev/null || pwd)}"; GSD_TOOLS="${_GSD_RUNTIME_ROOT}/gsd-core/bin/${_GSD_SHIM_NAME}"; _gsd_at() { for _p; do if [ -f "$_p" ]; then GSD_TOOLS="$_p"; return 0; fi; done; return 1; }; _gsd_id_ok() { case "$("$1" runtime-identity --raw 2>/dev/null || true)" in '{"packageName":"@opengsd/gsd-core"'*'}') return 0;; *) return 1;; esac; }; _gsd_homes() { set -- "${CLAUDE_CONFIG_DIR:-$HOME/.claude}" "${ANTIGRAVITY_CONFIG_DIR:-$HOME/.gemini/antigravity}" "$HOME/.gemini/antigravity-ide" "$HOME/.gemini/antigravity-cli" "${AUGMENT_CONFIG_DIR:-$HOME/.augment}" "${CLINE_CONFIG_DIR:-$HOME/.cline}" "${CODEBUDDY_CONFIG_DIR:-$HOME/.codebuddy}" "${CODEX_HOME:-$HOME/.codex}" "${COPILOT_CONFIG_DIR:-${COPILOT_HOME:-$HOME/.copilot}}" "${CURSOR_CONFIG_DIR:-$HOME/.cursor}" "${HERMES_HOME:-$HOME/.hermes}" "${KILO_CONFIG_DIR:-${XDG_CONFIG_HOME:-$HOME/.config}/kilo}" "${KIMI_CONFIG_DIR:-$HOME/.config/agents}" "$HOME/.agents" "${KIMI_CODE_HOME:-$HOME/.kimi-code}" "${OPENCODE_CONFIG_DIR:-${XDG_CONFIG_HOME:-$HOME/.config}/opencode}" "${PI_CODING_AGENT_DIR:-$HOME/.pi/agent}" "${QWEN_CONFIG_DIR:-$HOME/.qwen}" "${TRAE_CONFIG_DIR:-$HOME/.trae}" "${WINDSURF_CONFIG_DIR:-$HOME/.codeium/windsurf}" "${ZCODE_CONFIG_DIR:-$HOME/.zcode}" "${GROK_AGENTS_HOME:-$HOME/.agents}"; for _h; do _gsd_at "$_h/gsd-core/bin/${_GSD_SHIM_NAME}" && return 0; done; return 1; }; if _gsd_at "${_GSD_RUNTIME_ROOT}/gsd-core/bin/${_GSD_SHIM_NAME}" "${_GSD_RUNTIME_ROOT}/.claude/gsd-core/bin/${_GSD_SHIM_NAME}" "${_GSD_RUNTIME_ROOT}/.codex/gsd-core/bin/${_GSD_SHIM_NAME}"; then gsd_run() { node "$GSD_TOOLS" "$@"; }; elif _gsd_homes; then gsd_run() { node "$GSD_TOOLS" "$@"; }; elif unset -f gsd_run; _G="$(command -v gsd_run)"; [ -n "$_G" ] && _gsd_id_ok "$_G"; then GSD_TOOLS="$_G"; gsd_run() { "$GSD_TOOLS" "$@"; }; else echo "ERROR: gsd-tools.cjs not found at $GSD_TOOLS and no identity-proving gsd_run is on PATH. Run: npx -y @opengsd/gsd-core@latest --claude --local" >&2; exit 1; fi; GSD_IDENTITY_STATUS=unverified; _gsd_id_ok gsd_run && GSD_IDENTITY_STATUS=ok; export GSD_IDENTITY_STATUS; [ "$GSD_IDENTITY_STATUS" = ok ] || echo "WARNING: \"$GSD_TOOLS\" did not prove it is @opengsd/gsd-core - it is either a different package or an @opengsd/gsd-core older than the runtime-identity verb. See docs/how-to/diagnose-a-foreign-gsd-tools.md" >&2; if [ -n "${CLAUDE_ENV_FILE:-}" ] && [ -n "${GSD_TOOLS:-}" ]; then printf "export PATH='%s':\"\$PATH\"\n" "${GSD_TOOLS%/*}" >> "$CLAUDE_ENV_FILE" 2>/dev/null || true; fi
RESPONSE_LANGUAGE=$(gsd_run query config-get response_language --raw --default "" 2>/dev/null || echo "")
```

**If `response_language` is set:** You MUST present all user-facing output of this workflow in `{response_language}`. This output includes narration between tool calls, status updates, progress notes, findings, questions, prompts, and explanations.

Keep technical terms, code, file paths, and subagent prompts in English. Translate only user-facing output.

<step name="banner">
```
### GSD ► SPIKING
```

Parse `$ARGUMENTS` for:
- `--quick` flag → set `QUICK_MODE=true`
- `--text` flag → set `TEXT_MODE=true`
- `frontier` or empty → set `FRONTIER_MODE=true`
- Remaining text → the idea to spike

**Text mode:** If TEXT_MODE is enabled, replace AskUserQuestion calls with plain-text numbered lists.
</step>

<step name="route">
## Routing

- **FRONTIER_MODE is true** → Jump to `frontier_mode`
- **Otherwise** → Continue to `setup_directory`
</step>

<step name="frontier_mode">
## Frontier Mode — Propose What to Spike Next

### Load the Spike Landscape

If no `.planning/spikes/` directory exists, tell the user that there is nothing to analyze. Offer to start with an idea instead.

Otherwise, load the following in this order:

**a. MANIFEST.md** — read every idea section under `## Ideas`. Include each idea's paragraph and its own scoped Requirements. Read the `## Spikes` table with verdicts. Each row has an idea tag.

**b. Findings skills** — glob `./.claude/skills/spike-findings-*/SKILL.md`. Read each file that exists and its `references/*.md` files. These contain selected knowledge from before wrap-ups.

**c. CONVENTIONS.md** — if `.planning/spikes/CONVENTIONS.md` exists, read it for the established stack and patterns.

**d. All spike READMEs** — read `.planning/spikes/*/README.md` for verdicts, results, investigation trails, and tags.

### Analyze for Integration Spikes

Review every pair and cluster of VALIDATED spikes. Look for:

- **Shared resources:** Two spikes use the same API, database, state, or data format. Tests examined them independently.
- **Data handoffs:** Spike A produces output that Spike B consumes. The formats were assumed compatible but never proven.
- **Timing/ordering:** Spikes work in isolation but have sequence dependencies in the real flow.
- **Resource contention:** Spikes work individually but may compete for connections, memory, rate limits, or tokens when combined.

If integration risks exist, present concrete spike proposals with names and Given/When/Then validation questions. If no meaningful integration risks exist, say so and skip this category.

### Analyze for Frontier Spikes

Think about different possibilities for each idea section in MANIFEST.md. Consider what was proven so far for each idea. Consider:

- **Gaps in the vision:** Assumed but unproven capabilities.
- **Discovered dependencies:** Findings that reveal new questions.
- **Alternative approaches:** Different approaches for PARTIAL or INVALIDATED spikes.
- **Adjacent capabilities:** Capabilities that would improve the idea if feasible.
- **Comparison opportunities:** Approaches that worked but felt heavy.

Present concrete frontier spike proposals with Given/When/Then. Number them from the highest existing spike number. Order them by risk.

### Get Alignment and Execute

Present all integration and frontier candidates. Ask the user which spikes to run.

When the user picks spikes, append their definitions to the existing table in `.planning/spikes/MANIFEST.md`. Set each row's Idea column to the idea key(s) that the spike extends or validates. Proceed directly to build them, starting at `research`.
</step>

<step name="setup_directory">
If `.planning/spikes/` does not exist, create it:

```bash
mkdir -p .planning/spikes
```

Find existing spikes to determine numbering:
```bash
ls -d .planning/spikes/[0-9][0-9][0-9]-* 2>/dev/null | sort | tail -1
```

Read the `commit_docs` config:
```bash
COMMIT_DOCS=$(gsd_run query config-get commit_docs --raw 2>/dev/null || echo "true")
```
</step>

<step name="detect_stack">
Examine the project's tech stack to guide spike technology choices.

**Examine conventions first.** If `.planning/spikes/CONVENTIONS.md` exists, follow its stack and patterns. These are validated choices that the user expects to continue.

**Then examine the project stack:**
```bash
ls package.json pyproject.toml Cargo.toml go.mod 2>/dev/null
```

Use the project's language/framework by default. For greenfield projects with no conventions and no existing stack, pick the fastest path to a runnable result.

Unless the spike specifically needs them, avoid:
- Complex package management beyond `npm install` or `pip install`
- Build tools, bundlers, or transpilers
- Docker, containers, or infrastructure
- Env files or config systems — hardcode everything
</step>

<step name="load_prior_context">
If `.planning/spikes/` has existing content, load context in this priority order:

**a. Conventions:** If `.planning/spikes/CONVENTIONS.md` exists, read it.

**b. Findings skills:** Glob for `./.claude/skills/spike-findings-*/SKILL.md`. Read each file that exists and its `references/*.md` files.

**c. Manifest:** Read `.planning/spikes/MANIFEST.md` for the index of all spikes.

**d. Related READMEs:** Use the new idea to find related earlier spikes. Match tags, names, technologies, or domain overlap. Read only those `.planning/spikes/*/README.md` files. Skip unrelated files.

Compare against all this earlier work:
- **Skip already-validated questions.** Note the earlier spike number and continue.
- **Use earlier findings.** Do not repeat failed approaches. Use their Research and Results sections.
- **Reuse earlier research.** Use findings from that research instead of repeating it.
- **Follow established conventions.** Mention any deviation.
- **When you present the decomposition, mention relevant earlier work.**

If no `.planning/spikes/` exists, skip this step.
</step>

<step name="decompose">
**If `QUICK_MODE` is true:** Skip decomposition and alignment. Take the user's idea as a single spike question. Assign it the next available number. Jump to `research`.

Break the idea into 2-5 independent questions. Frame each as Given/When/Then. Present them as a table:

```
| # | Spike | Type | Validates (Given/When/Then) | Risk |
|---|-------|------|-----------------------------|------|
| 001 | websocket-streaming | standard | Given a WS connection, when LLM streams tokens, then client receives chunks < 100ms | **High** |
| 002a | pdf-parse-pdfjs | comparison | Given a multi-page PDF, when parsed with pdfjs, then structured text is extractable | Medium |
| 002b | pdf-parse-camelot | comparison | Given a multi-page PDF, when parsed with camelot, then structured text is extractable | Medium |
```

**Spike types:**
- **standard** — one approach answering one question
- **comparison** — same question, different approaches. Shared number with letter suffix.

Good spikes ask specific feasibility questions with observable output.
Bad spikes are too broad, have no observable output, or consist only of reading/planning.

Order the spikes by risk. Run the spike most likely to stop the idea first.
</step>

<step name="align">
**If `QUICK_MODE` is true:** Skip.

### CHECKPOINT: Decision Need

{spike table from decompose step}

---

**→ Build all in this order, or adjust the list?**
</step>

<step name="research">
## Research and Briefing Before Each Spike

This step runs **before each individual spike**, not once at the start.

**a. Present a spike briefing:**

> **Spike NNN: Descriptive Name**
> [2-3 sentences: what this spike is, why it matters, key risk or unknown.]

**b. Research the current state of the art.** Use context7 (resolve-library-id → query-docs) for libraries/frameworks. For APIs/services without a context7 entry, use web search. Read the documentation.

**c. Surface competing approaches** as a table:

| Approach | Tool/Library | Pros | Cons | Status |
|----------|-------------|------|------|--------|
| ... | ... | ... | ... | ... |

**Chosen approach:** [which one and why]

If 2+ credible approaches exist, plan to build quick variants within the spike and compare them.

**d. Capture research findings** in a `## Research` section in the README.

**For pure logic with no external dependencies, skip this step when unnecessary.**
</step>

<step name="create_manifest">
Create or update `.planning/spikes/MANIFEST.md`.

**Assign an idea key.** Derive a short, stable, kebab-case slug (2-4 words) that summarizes the current idea. For example, a slug is `realtime-llm-streaming`.

Reuse the exact same idea key for every spike in this session and any later session that continues the same idea. Only create a new idea key when the current idea does not continue one already indexed in MANIFEST.md. Never reuse an existing idea key for an unrelated idea. Never merge two different ideas under one key.

If `.planning/spikes/MANIFEST.md` does not exist, create it:

```markdown
# Spike Manifest

## Ideas

### {idea-key}
[One paragraph describing this idea]

**Requirements:**
[Design decisions that emerged from the user's choices while spiking THIS idea. Non-negotiable
for the real build of this idea. Updated as spikes progress. Never copy or merge requirements
from a different idea key into this list.]

- [e.g., "Must use streaming JSON output, not single-response"]
- [e.g., "Must support reconnection on network failure"]

## Spikes

| # | Idea | Name | Type | Validates | Verdict | Tags |
|---|------|------|------|-----------|---------|------|
```

If `.planning/spikes/MANIFEST.md` already exists:

- **Same idea key already has a `### {idea-key}` section under `## Ideas`:** As new requirements emerge, append them to that section's Requirements list. Never overwrite or rewrite its `## Idea` paragraph.
- **New idea key, not yet present:** Append a new `### {idea-key}` subsection under `## Ideas`, after any existing idea sections. Never change, merge into, or delete another idea's section.
- **A pre-#1700 MANIFEST.md with the old flat shape** (a single top-level `## Idea` / `##
  Requirements` pair, no `## Ideas` heading): treat its existing content as one implicit idea.
  Derive an idea key from its `## Idea` paragraph, migrate it in place to `## Ideas` >
  `### {idea-key}`. Preserve the paragraph and every existing Requirements bullet and
  Spikes row verbatim. Then continue as above. Do this migration once. After
  `## Ideas` exists, do not repeat it.

Every row appended to `## Spikes` carries an **Idea** column set to the idea key it belongs to.

**Track requirements as they emerge.** When the user expresses a preference during spiking, add it immediately to the current idea's Requirements list. Never add it to a different idea's list.
</step>

<step name="reground">
## Re-Ground Before Each Spike

Before each spike, not only the first, re-read `.planning/spikes/MANIFEST.md` and `.planning/spikes/CONVENTIONS.md`. This prevents drift during long sessions.

Examine the current idea's `### {idea-key}` Requirements list. Make sure that the spike does not contradict any established requirement for this idea. Do not apply another idea's requirements.
</step>

<step name="build_spikes">
## Build Each Spike Sequentially

**Depth over speed.** The goal is to understand, not to reach a quick verdict. Never declare VALIDATED after a single happy-path test. Investigate surprising findings. Test edge cases. Document the investigation trail, not only the conclusion.

**Comparison spikes** use a shared number with a letter suffix: `NNN-a-name` / `NNN-b-name`. Build them back-to-back. Then compare them head-to-head.

### For Each Spike:

**a.** Create `.planning/spikes/NNN-descriptive-name/`

**b.** By default, give the user something they can experience. The bias should favor a simple UI or interactive demo, not stdout that only Claude reads. The user wants to *feel* the spike work, not only hear that it works.

**The default is: build something the user can interact with.** This could be:
- A simple HTML page that shows the result visually
- A web UI with a button that triggers the action and shows the response
- A page that displays data flowing through a pipeline
- A minimal interface where the user can try different inputs and see outputs

**Only use stdout/CLI verification when the spike is about a fact, not a feeling:**
- Pure data transformation where the answer is "yes it parses correctly"
- Binary yes/no questions (does this API authenticate? does this library exist?)
- Benchmark numbers (how fast is X? how much memory does Y use?)

When in doubt, build the UI. It takes a few extra minutes but produces a spike the user can demo and feel confident about.

**If the spike needs runtime observability,** build a forensic log layer:
1. Event log array with ISO timestamps and category tags
2. Export mechanism (server: GET endpoint, CLI: JSON file, browser: Export button)
3. Log summary (event counts, duration, errors, metadata)
4. Analysis helpers if volume warrants it

**c.** Build the code. Start with the simplest version. Then deepen it.

**d.** When findings warrant it, iterate:
- **Surprising surface?** Write a follow-up test that isolates and explores it.
- **Answer feels shallow?** Test edge cases — large inputs, concurrent requests, malformed data, network failures.
- **Assumption wrong?** Adjust. Note the pivot in the README.

Complex questions are expected to need multiple files per spike. For example: `test-basic.js`, `test-edge-cases.js`, `benchmark.js`.

**e.** Write `README.md` with YAML frontmatter:

```markdown
---
spike: NNN
idea: {idea-key}
name: descriptive-name
type: standard
validates: "Given [precondition], when [action], then [expected outcome]"
verdict: PENDING
related: []
tags: [tag1, tag2]
---

# Spike NNN: Descriptive Name

## What This Validates
[Given/When/Then]

## Research
[Docs checked, approach comparison table, chosen approach, gotchas. Omit if no external deps.]

## How to Run
[Command(s)]

## What to Expect
[Concrete observable outcomes]

## Observability
[If forensic log layer exists. Omit otherwise.]

## Investigation Trail
[Updated as spike progresses. Document each iteration: what tried, what revealed, what tried next.]

## Results
[Verdict, evidence, surprises, log analysis findings.]
```

**f.** Auto-link related spikes silently.

**g.** Run and verify:
- Self-verifiable: run the spike. If findings warrant deeper investigation, iterate. Update the verdict.
- Needs human judgment: present the checkpoint box:

### CHECKPOINT: Verification Need

**Spike {NNN}: {name}**
**How to run:** {command}
**What to expect:** {concrete outcomes}

---

**→ Does this match what you expected? Describe what you see.**

**h.** Update `.planning/spikes/MANIFEST.md` with the spike's row. Set the Idea column to this spike's idea key.

**i.** Commit (if `COMMIT_DOCS` is true):
```bash
gsd_run query commit "docs(spike-NNN): [VERDICT] — [key finding]" --files .planning/spikes/NNN-descriptive-name/ .planning/spikes/MANIFEST.md
```

**j.** Report:
```
◆ Spike NNN: {name}
  Verdict: {VALIDATED ✓ / INVALIDATED ✗ / PARTIAL ⚠}
  Key findings: {not just verdict — investigation trail, surprises, edge cases explored}
  Impact: {effect on remaining spikes}
```

Do not rush to a verdict. A spike that says "VALIDATED — it works" without detail is almost always incomplete.

**k.** If core assumption invalidated:

### CHECKPOINT: Decision Need

Core assumption invalidated by Spike {NNN}.
{what was invalidated and why}

---

**→ Continue with remaining spikes / Pivot approach / Abandon**
</step>

<step name="update_conventions">
## Update Conventions

After you build all spikes in this session, update `.planning/spikes/CONVENTIONS.md` with patterns that emerged or became established.

```markdown
# Spike Conventions

Patterns and stack choices established across spike sessions. New spikes follow these unless the question requires otherwise.

## Stack
[What we use for frontend, backend, scripts, and why]

## Structure
[Common file layouts, port assignments, naming patterns]

## Patterns
[Recurring approaches: how we handle auth, how we style, how we serve]

## Tools & Libraries
[Preferred packages with versions that worked, and any to avoid]
```

Only include patterns that repeated across 2+ spikes or that the user explicitly chose. If `CONVENTIONS.md` already exists, update its sections with new patterns from this session.

Commit (if `COMMIT_DOCS` is true):
```bash
gsd_run query commit "docs(spikes): update conventions" --files .planning/spikes/CONVENTIONS.md
```
</step>

<step name="report">
```
### GSD ► SPIKE COMPLETE ✓

## Verdicts

| # | Name | Type | Verdict |
|---|------|------|---------|
| 001 | {name} | standard | ✓ VALIDATED |
| 002a | {name} | comparison | ✓ WINNER |

## Key Discoveries
{surprises, gotchas, investigation trail highlights}

## Feasibility Assessment
{overall viability}

## Signal for the Build
{what to use, avoid, watch out for}
```

---

## ▶ Next Up

**Package findings** — put spike knowledge into an implementation blueprint

`/gsd:spike --wrap-up`

---

**Also available:**
- `/gsd:spike` — spike more ideas (or run with no argument for frontier mode)
- `/gsd:plan-phase` — start planning the real implementation
- `/gsd:explore` — continue exploring the idea

---
</step>

</process>

<success_criteria>
- [ ] `.planning/spikes/` exists (auto-creates if needed, no project init need)
- [ ] Earlier spikes and findings skills informed the work before building
- [ ] Work follows conventions (or documents deviation)
- [ ] Research used current docs for each spike before coding
- [ ] Depth over speed — tests cover edge cases, investigation follows surprising findings, and the investigation trail is documented
- [ ] Comparison spikes were built back-to-back with a head-to-head verdict
- [ ] Spikes that need human interaction have a forensic log layer
- [ ] MANIFEST.md tracks requirements from user choices as they emerge, scoped to the idea key that produced them
- [ ] CONVENTIONS.md exists or includes updates for patterns that emerged
- [ ] Each spike README has complete frontmatter (including its idea key), Investigation Trail, and Results
- [ ] MANIFEST.md is current (with Idea and Type columns, and each idea's own scoped Requirements section)
- [ ] Commits use `docs(spike-NNN): [VERDICT]` format
- [ ] The consolidated report includes next-step routing
</success_criteria>
