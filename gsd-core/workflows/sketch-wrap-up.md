@~/.claude/gsd-core/references/response-language-directive.md

<purpose>
Curate sketch design findings and put them in a persistent project skill for future UI implementation.
Read from `.planning/sketches/`.
Write the project-local skill to `./.claude/skills/sketch-findings-[project]/`.
Write the summary to `.planning/sketches/WRAP-UP-SUMMARY.md`.
This workflow is a companion to `/gsd:sketch`.
</purpose>

<required_reading>
Before starting, read all files that the invoking prompt's execution_context references.
</required_reading>

<process>

<step name="banner">
```
### GSD ► SKETCH WRAP-UP
```
</step>

<step name="gather">
## Gather Sketch Inventory

1. Read `.planning/sketches/MANIFEST.md` for the design direction and reference points.
2. Glob `.planning/sketches/*/README.md`.
   Parse YAML frontmatter from each file.
3. Determine if `./.claude/skills/sketch-findings-*/SKILL.md` exists for this project.
   - If it exists, read its `processed_sketches` list.
     Remove those sketches from the candidates.
   - If it does not exist, all sketches are candidates.

If no unprocessed sketches exist:
```
No unprocessed sketches found in `.planning/sketches/`.
Run `/gsd:sketch` first to create design explorations.
```
Exit.

Read the `commit_docs` config:
```bash
_GSD_SHIM_NAME="gsd-tools.cjs"; _GSD_RUNTIME_ROOT="${RUNTIME_DIR:-$(git rev-parse --show-toplevel 2>/dev/null || pwd)}"; GSD_TOOLS="${_GSD_RUNTIME_ROOT}/gsd-core/bin/${_GSD_SHIM_NAME}"; _gsd_at() { for _p; do if [ -f "$_p" ]; then GSD_TOOLS="$_p"; return 0; fi; done; return 1; }; _gsd_id_ok() { case "$("$1" runtime-identity --raw 2>/dev/null || true)" in '{"packageName":"@opengsd/gsd-core"'*'}') return 0;; *) return 1;; esac; }; _gsd_homes() { set -- "${CLAUDE_CONFIG_DIR:-$HOME/.claude}" "${ANTIGRAVITY_CONFIG_DIR:-$HOME/.gemini/antigravity}" "$HOME/.gemini/antigravity-ide" "$HOME/.gemini/antigravity-cli" "${AUGMENT_CONFIG_DIR:-$HOME/.augment}" "${CLINE_CONFIG_DIR:-$HOME/.cline}" "${CODEBUDDY_CONFIG_DIR:-$HOME/.codebuddy}" "${CODEX_HOME:-$HOME/.codex}" "${COPILOT_CONFIG_DIR:-${COPILOT_HOME:-$HOME/.copilot}}" "${CURSOR_CONFIG_DIR:-$HOME/.cursor}" "${HERMES_HOME:-$HOME/.hermes}" "${KILO_CONFIG_DIR:-${XDG_CONFIG_HOME:-$HOME/.config}/kilo}" "${KIMI_CONFIG_DIR:-$HOME/.config/agents}" "$HOME/.agents" "${KIMI_CODE_HOME:-$HOME/.kimi-code}" "${OPENCODE_CONFIG_DIR:-${XDG_CONFIG_HOME:-$HOME/.config}/opencode}" "${PI_CODING_AGENT_DIR:-$HOME/.pi/agent}" "${QWEN_CONFIG_DIR:-$HOME/.qwen}" "${TRAE_CONFIG_DIR:-$HOME/.trae}" "${WINDSURF_CONFIG_DIR:-$HOME/.codeium/windsurf}" "${ZCODE_CONFIG_DIR:-$HOME/.zcode}" "${GROK_AGENTS_HOME:-$HOME/.agents}"; for _h; do _gsd_at "$_h/gsd-core/bin/${_GSD_SHIM_NAME}" && return 0; done; return 1; }; if _gsd_at "${_GSD_RUNTIME_ROOT}/gsd-core/bin/${_GSD_SHIM_NAME}" "${_GSD_RUNTIME_ROOT}/.claude/gsd-core/bin/${_GSD_SHIM_NAME}" "${_GSD_RUNTIME_ROOT}/.codex/gsd-core/bin/${_GSD_SHIM_NAME}"; then gsd_run() { node "$GSD_TOOLS" "$@"; }; elif _gsd_homes; then gsd_run() { node "$GSD_TOOLS" "$@"; }; elif unset -f gsd_run; _G="$(command -v gsd_run)"; [ -n "$_G" ] && _gsd_id_ok "$_G"; then GSD_TOOLS="$_G"; gsd_run() { "$GSD_TOOLS" "$@"; }; else echo "ERROR: gsd-tools.cjs not found at $GSD_TOOLS and no identity-proving gsd_run is on PATH. Run: npx -y @opengsd/gsd-core@latest --claude --local" >&2; exit 1; fi; GSD_IDENTITY_STATUS=unverified; _gsd_id_ok gsd_run && GSD_IDENTITY_STATUS=ok; export GSD_IDENTITY_STATUS; [ "$GSD_IDENTITY_STATUS" = ok ] || echo "WARNING: \"$GSD_TOOLS\" did not prove it is @opengsd/gsd-core - it is either a different package or an @opengsd/gsd-core older than the runtime-identity verb. See docs/how-to/diagnose-a-foreign-gsd-tools.md" >&2; if [ -n "${CLAUDE_ENV_FILE:-}" ] && [ -n "${GSD_TOOLS:-}" ]; then printf "export PATH='%s':\"\$PATH\"\n" "${GSD_TOOLS%/*}" >> "$CLAUDE_ENV_FILE" 2>/dev/null || true; fi
COMMIT_DOCS=$(gsd_run query config-get commit_docs --raw 2>/dev/null || echo "true")
```
</step>

<step name="curate">
## Curate Sketches One-at-a-Time

Show each unprocessed sketch in ascending order. For each sketch, show:

- **Sketch number and name**
- **Design question:** from frontmatter
- **Winner:** the selected variant, if any
- **Tags:** from frontmatter
- **Key decisions:** a summary of the visual decisions

Then ask the user:

### CHECKPOINT: Decision Need

Sketch {NNN}: {name} — Winner: Variant {X}

{key design decisions summary}

---

**→ Include / Exclude / Partial / Let me look at it**

**If "Let me look at it":**
1. Provide: `open .planning/sketches/NNN-name/index.html`
2. Remind the user which variant won and what to look for.
3. After the user looks at it, return to the include/exclude/partial decision.

**If "Partial":**
Ask what specifically to include or exclude from this sketch's decisions.
</step>

<step name="group">
## Auto-Group by Design Area

After you curate all sketches:

1. Read the tags, names, and content of all included sketches.
2. Propose design-area groups. For example:
   - "**Layout & Navigation** — sketches 001, 004"
   - "**Form Controls** — sketches 002, 005"
   - "**Color & Typography** — sketches 003"
3. Show the groups for approval. The user may merge, split, rename, or rearrange them.

Each group becomes one reference file in the generated skill.
</step>

<step name="skill_name">
## Determine Output Skill Name

Derive the skill name from the project directory name: `./.claude/skills/sketch-findings-[project-dir-name]/`

If a skill already exists at that path (append mode), update it in place.
</step>

<step name="copy_sources">
## Copy Source Files

For each included sketch:

1. Copy the winning variant's HTML file into `sources/NNN-sketch-name/`.
   Alternatively, copy the full index.html with all variants into that directory.
2. Copy the winning theme.css into `sources/themes/`.
3. Exclude node_modules, build artifacts, and .DS_Store.
</step>

<step name="synthesize">
## Synthesize Reference Files

For each design-area group, write a reference file at `references/[design-area-name].md`:

```markdown
# [Design Area Name]

## Design Decisions
[For each validated decision: what was chosen, why it won over alternatives, the key visual properties (colors, spacing, border radius, typography)]

## CSS Patterns
[Key CSS snippets from winning variants — layout structures, component patterns, animation patterns. Extracted and cleaned up for reference.]

## HTML Structures
[Key HTML patterns from winning variants — page layout, component markup, navigation structures.]

## What to Avoid
[Design directions that were tried and rejected. Why they didn't work.]

## Origin
Synthesized from sketches: NNN, NNN
Source files available in: sources/NNN-sketch-name/
```
</step>

<step name="write_skill">
## Write SKILL.md

Create (or update) the generated skill's SKILL.md:

```markdown
---
name: sketch-findings-[project-dir-name]
description: Validated design decisions, CSS patterns, and visual direction from sketch experiments. Auto-loaded during UI implementation on [project-dir-name].
---

<context>
## Project: [project-dir-name]

[Design direction paragraph from MANIFEST.md]
[Reference points mentioned during intake]

Sketch sessions wrapped: [date(s)]
</context>

<design_direction>
## Overall Direction

[Summary of the validated visual direction: palette, typography, spacing system, layout approach, interaction patterns]
</design_direction>

<findings_index>
## Design Areas

| Area | Reference | Key Decision |
|------|-----------|--------------|
| [Name] | references/[name].md | [One-line summary] |

## Theme

The winning theme file is at `sources/themes/default.css`.

## Source Files

Original sketch HTML files are preserved in `sources/` for complete reference.
</findings_index>

<metadata>
## Processed Sketches

[List of sketch numbers wrapped up]

- 001-sketch-name
- 002-sketch-name
</metadata>
```
</step>

<step name="write_summary">
## Write Planning Summary

Write `.planning/sketches/WRAP-UP-SUMMARY.md` for project history:

```markdown
# Sketch Wrap-Up Summary

**Date:** [date]
**Sketches processed:** [count]
**Design areas:** [list]
**Skill output:** `./.claude/skills/sketch-findings-[project]/`

## Included Sketches
| # | Name | Winner | Design Area |
|---|------|--------|-------------|

## Excluded Sketches
| # | Name | Reason |
|---|------|--------|

## Design Direction
[consolidated design direction summary]

## Key Decisions
[layout, palette, typography, spacing, interaction patterns]
```
</step>

<step name="update_claude_md">
## Update Project CLAUDE.md

Add an auto-load routing line:

```
- **Sketch findings for [project]** (design decisions, CSS patterns, visual direction) → `Skill("sketch-findings-[project-dir-name]")`
```

If this routing line already exists (append mode), leave it unchanged.
</step>

<step name="commit">
If `COMMIT_DOCS` is true, commit all artifacts:

```bash
gsd_run query commit "docs(sketch-wrap-up): package [N] sketch findings into project skill" --files .planning/sketches/WRAP-UP-SUMMARY.md
```
</step>

<step name="report">
```
### GSD ► SKETCH WRAP-UP COMPLETE ✓

**Curated:** {N} sketches ({included} included, {excluded} excluded)
**Design areas:** {list}
**Skill:** `./.claude/skills/sketch-findings-[project]/`
**Summary:** `.planning/sketches/WRAP-UP-SUMMARY.md`
**CLAUDE.md:** routing line added

The sketch-findings skill will auto-load when building the UI.
```

---

## ▶ Next Up

**Explore frontier sketches** — see what else is worth sketching based on previous exploration.

Run `/gsd:sketch` with no argument. Its frontier mode analyzes the sketch landscape and proposes consistency and frontier sketches.

---

**Also available:**
- `/gsd:plan-phase` — start implementation of the real UI
- `/gsd:ui-phase` — generate a UI design contract for a frontend phase
- `/gsd:sketch [idea]` — sketch a specific new design area
- `/gsd:explore` — continue exploration

---
</step>

</process>

<success_criteria>
- [ ] The workflow shows every unprocessed sketch for individual curation.
- [ ] The workflow proposes design-area groups and the user approves them.
- [ ] The sketch-findings skill exists at `./.claude/skills/` with SKILL.md, references/, and sources/.
- [ ] The winning theme.css is in the skill sources.
- [ ] Reference files contain design decisions, CSS patterns, HTML structures, and anti-patterns.
- [ ] `.planning/sketches/WRAP-UP-SUMMARY.md` exists for project history.
- [ ] Project CLAUDE.md has an auto-load routing line.
- [ ] The workflow shows the summary.
- [ ] The workflow shows next-step options, including frontier sketch exploration via `/gsd:sketch`.
</success_criteria>
