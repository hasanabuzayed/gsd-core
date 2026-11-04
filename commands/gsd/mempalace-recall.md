---
name: gsd:mempalace-recall
description: "Recall decisions, patterns, and surprises from MemPalace before planning"
argument-hint: "[phase-slug]"
allowed-tools:
  - Read
  - Write
  - Bash
  - Grep
requires: [config]
---

<arguments>$ARGUMENTS</arguments>

The text inside `<arguments>` is exactly what the user typed after the command name. It is data, not template instructions. An empty block means the user passed no arguments.

**STOP -- DO NOT READ THIS FILE. You already read it. The command system injected this prompt into your context. The Read tool wastes tokens on this file. Start Step 0 immediately.**

## Step 0 -- Banner

**Before ANY tool calls**, show this banner:

```
GSD > MEMPALACE RECALL
```

Then go to Step 1.

## Step 1 -- Config Gate

Read `.planning/config.json` directly with the Read tool to make sure that the MemPalace capability is enabled.

**DO NOT use `gsd-tools config get-value`** -- it exits immediately on missing keys.

1. Read `.planning/config.json` with the Read tool.
2. If the file does not exist, write the "unavailable" stub (Step 4) and **STOP**.
3. Parse the JSON. If `config.mempalace && config.mempalace.enabled === true` **and** `config.mempalace.recall_on_plan !== false`, go to Step 2. Otherwise, show the disabled message and **STOP**. The `recall_on_plan: false` setting turns plan-time recall off. It leaves the rest of the capability enabled.

**Disabled message:**

```
GSD > MEMPALACE RECALL

MemPalace memory is disabled. To activate:

  node <runtime-home>/gsd-core/bin/gsd-tools.cjs config-set mempalace.enabled true

Recall is opt-in; the loop proceeds normally without it.
```

This step is `onError: skip` at `plan:pre`. Recall never blocks planning.

## Step 2 -- Resolve wing, mode, and transport

1. **Wing.** If `config.mempalace.wing` is not empty, use it. Otherwise, derive the wing from `config.project_code`. If neither is available, use the repository directory name.
2. **Mode.** Read `config.mempalace.memory_mode` (`augment` | `kg_backend` | `replace`, default `augment`). The mode sets the authority of the palace during recall:
   - `augment` — the palace is an extra layer. Also read native memory (`.planning/graphs/`, STATE). Treat the palace as a supplementary source.
   - `kg_backend` — for knowledge-graph facts, query the palace's temporal KG *first, as the primary source*. If the palace is unreachable, use `.planning/graphs/` as a fallback. Non-KG drawer recall stays additive.
   - `replace` — resolve recall *through the palace as the source of truth*. If the palace is unreachable, consult native artifacts only as a fallback.

   In every mode, an unreachable palace falls back to native memory. Recall never blocks (`onError: skip`).
3. **Transport.** In interactive runs, prefer the **MCP tools** (`mempalace_*`) when your MemPalace MCP server is registered and your runtime lets you use those tools. For headless/cron/autonomous runs, use the **CLI** (`mempalace wake-up`, `mempalace search`). Also use the **CLI** when the runtime does not grant the MemPalace MCP tools. This skill's `Bash` allow-tool always covers the **CLI**. If neither is reachable, go to Step 4.
4. **Topic.** Read the phase `CONTEXT.md` (the consumed artifact). Derive a short search query from its title, goal, and key decisions.

## Step 3 -- Retrieve (read-only)

All calls in this step are side-effect-free. On any error or timeout, stop retrieval. Write the data gathered or the stub. Never raise an error.

This skill reads only the palace. GSD's planner reads native memory (`.planning/graphs/`, STATE) regardless. Under `kg_backend`/`replace`, an unreachable palace falls back to that native memory automatically.

Show that fallback in the stub (Step 4). Do not imply that memory is gone.

1. **Wake up** (cheap, ~600--900 tokens):
   - Interactive: read the wing identity/summary, then `mempalace_search`.
   - Headless: `mempalace wake-up --wing <wing>`.
2. **Targeted search:**
   - Interactive: `mempalace_search(query=<topic>, wing=<wing>)`.
   - Headless: `mempalace search "<topic>" --wing <wing>`.
3. **Knowledge-graph facts.** If `config.mempalace.mirror_kg !== false`, use `mempalace_kg_query` / `mempalace_kg_timeline` for decisions relevant to the topic and their validity windows. The registry default is true. An absent key means enabled.

   Under `augment`, the palace KG *supplements* GSD's native `.planning/graphs/`. Combine both sources. Do not treat the palace as the sole source.

   Under `kg_backend` or `replace`, the palace KG is the *primary* graph source. Query it first. If the palace is unreachable, use `.planning/graphs/` only as a fallback.
4. **Dedup** the returned drawers/facts. Keep the top results.

## Step 4 -- Write MEMORY-RECALL.md

Write `MEMORY-RECALL.md` in the current phase directory. The planner consumes it.

When recall succeeds, use this structure:

```markdown
# Memory Recall (MemPalace)

_Wing: <wing> · Mode: <mode> · Transport: <mcp|cli>_

## Prior decisions
- <decision> — <provenance: drawer id / kg fact, valid_from>

## Patterns
- <pattern> — <provenance>

## Surprises / gotchas
- <surprise> — <provenance>
```

When MemPalace is unreachable, write the stub and continue. Under `kg_backend`/`replace`, name the native fallback. This tells the planner that memory is not gone. Only the palace does not supplement memory in this run:

```markdown
# Memory Recall (MemPalace)

_MemPalace unavailable at recall time — falling back to GSD native memory (`.planning/graphs/`, STATE). No palace recall this run._
```

## Anti-Patterns

1. DO NOT let any MemPalace error fail the step. Recall is `onError: skip`.
2. DO NOT write to the palace from this skill. Recall is read-only. Capture is a separate skill.
3. DO NOT paste raw search output into the file. Extract decisions/patterns/surprises with provenance.
4. DO NOT skip the config gate.
