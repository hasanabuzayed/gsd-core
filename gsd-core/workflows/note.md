@~/.claude/gsd-core/references/response-language-directive.md

<purpose>
Capture ideas with one Write call and one confirmation line. Do not ask questions or show prompts.

**Text mode (`workflow.text_mode: true` in config or `--text` flag):** If `--text` is present in `$ARGUMENTS` OR `text_mode` from init JSON is `true`, set `TEXT_MODE=true`.

When TEXT_MODE is active, replace every `AskUserQuestion` call with a plain-text numbered list. Ask the user to type their choice number. Non-Claude runtimes need this because `AskUserQuestion` is not available. Examples include OpenAI Codex and Antigravity.

Run inline. Do not use Task, AskUserQuestion, or Bash.
</purpose>

<required_reading>
Before you start, read all files that the invoking prompt's execution_context references.
</required_reading>

<process>

<step name="storage_format">
**Note storage format.**

Store each note in a separate markdown file:

- **Project scope**: `.planning/notes/{YYYY-MM-DD}-{slug}.md` — use when `.planning/` exists in cwd
- **Global scope**: `~/.claude/notes/{YYYY-MM-DD}-{slug}.md` — use when `.planning/` does not exist or the `--global` flag is present

Each note file:

```markdown
---
date: "YYYY-MM-DD HH:mm"
promoted: false
---

{note text verbatim}
```

**`--global` flag**: Before you parse `$ARGUMENTS`, remove `--global` from anywhere in it. When the flag is present, use global scope even when `.planning/` exists.

**Important**: If `.planning/` does not exist, do NOT create it. Use global scope without a message.
</step>

<step name="parse_subcommand">
**After you remove --global, parse the subcommand from $ARGUMENTS.**

| Condition | Subcommand |
|-----------|------------|
| Arguments are exactly `list` (case-insensitive) | **list** |
| Arguments are exactly `promote <N>` where N is a number | **promote** |
| Arguments are empty (no text at all) | **list** |
| Anything else | **append** (the text IS the note) |

**Critical**: `list` is a subcommand only when it is the ENTIRE argument. `/gsd-note list of groceries` saves a note with text "list of groceries". `promote` is a subcommand only when exactly one number follows it.
</step>

<step name="append">
**Subcommand: append — create a timestamped note file.**

1. Find the scope (project or global) from the storage format above.
2. Make sure that the notes directory exists (`.planning/notes/` or `~/.claude/notes/`).
3. Generate the slug from the first ~4 meaningful words of the note text. Use lowercase letters and hyphens between words. Remove articles/prepositions from the start.
4. Generate the filename: `{YYYY-MM-DD}-{slug}.md`.
   - If a file with that name already exists, append `-2`, `-3`, and so on.
5. Write the file with frontmatter and note text (see storage format).
6. Confirm with exactly one line: `Noted ({scope}): {note text}`
   - Where `{scope}` is "project" or "global"

**Constraints:**
- **Never change the note text** — capture it verbatim, including typos
- **Never ask questions** — write and confirm
- **Timestamp format**: Use local time, `YYYY-MM-DD HH:mm` (24-hour, no seconds)
</step>

<step name="list">
**Subcommand: list — show notes from both scopes.**

1. If the directory exists, glob `.planning/notes/*.md` for project notes.
2. If the directory exists, glob `~/.claude/notes/*.md` for global notes.
3. For each file, read frontmatter to get `date` and `promoted` status.
4. Exclude files where `promoted: true` from active counts. Show them dimmed.
5. Sort by date. Number all active entries in sequence, starting at 1.
6. If total active entries > 20, show only the last 10. Include a note about how many entries you omitted.

**Output format:**

```
Notes:

Project (.planning/notes/):
  1. [2026-02-08 14:32] refactor the hook system to support async validators
  2. [promoted] [2026-02-08 14:40] add rate limiting to the API endpoints
  3. [2026-02-08 15:10] consider adding a --dry-run flag to build

Global (~/.claude/notes/):
  4. [2026-02-08 10:00] cross-project idea about shared config

{count} active note(s). Use `/gsd-note promote <N>` to convert to a todo.
```

If a scope has no directory or no entries, show: `(no notes)`
</step>

<step name="promote">
**Subcommand: promote — convert a note into a todo.**

1. Run the **list** logic to build the numbered index (both scopes).
2. Find entry N in the numbered list.
3. If N is invalid or refers to an already-promoted note, tell the user and stop.
4. **Need `.planning/` directory** — if it does not exist, warn: "Todos need a GSD project. Run `/gsd:new-project` to initialize one."
5. Make sure that the `.planning/todos/pending/` directory exists.
6. Generate the todo ID: `{NNN}-{slug}`. NNN is the next sequential number. Scan both `.planning/todos/pending/` and `.planning/todos/completed/` for the highest existing number. Increment it by 1. Add leading zeros to make 3 digits.

   The slug is the first ~4 meaningful words of the note text.
7. Extract the note text from the source file (body after frontmatter).
8. Create `.planning/todos/pending/{id}.md`:

```yaml
---
title: "{note text}"
status: pending
priority: P2
source: "promoted from /gsd-note"
created: {YYYY-MM-DD}
theme: general
---

## Goal

{note text}

## Context

Promoted from quick note captured on {original date}.

## Acceptance Criteria

- [ ] {primary criterion derived from note text}
```

9. Mark the source note file as promoted. Change its frontmatter to `promoted: true`.
10. Confirm: `Promoted note {N} to todo {id}: {note text}`
</step>

</process>

<edge_cases>
1. **"list" as note text**: `/gsd-note list of things` saves note "list of things". `list` is a subcommand only when it is the entire arg.
2. **No `.planning/`**: Use global `~/.claude/notes/`. This works in any directory.
3. **Promote without project**: Warn that todos need `.planning/`. Suggest `/gsd:new-project`.
4. **Large files**: When there are >20 active entries, `list` shows the last 10.
5. **Duplicate slugs**: If the slug is already in use on the same date, append `-2`, `-3`, and so on to the filename.
6. **`--global` position**: Remove it from anywhere. `--global my idea` and `my idea --global` both save "my idea" globally.
7. **Promote already-promoted**: Tell the user "Note {N} is already promoted" and stop.
8. **Empty note text after removing flags**: Treat as `list` subcommand.
</edge_cases>

<success_criteria>
- [ ] Append: The note file has correct frontmatter and verbatim text.
- [ ] Append: The capture is instant and asks no questions.
- [ ] List: Both scopes appear with sequential numbering.
- [ ] List: Promoted notes appear dimmed.
- [ ] Promote: The todo has the correct format.
- [ ] Promote: The source note is marked as promoted.
- [ ] Global fallback: It works when no `.planning/` exists.
</success_criteria>
