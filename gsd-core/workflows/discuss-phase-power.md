@~/.claude/gsd-core/references/response-language-directive.md

<purpose>
Power user mode for discuss-phase generates ALL questions first. It writes the questions to a JSON state file and an HTML companion UI. It then waits for the user to answer at their own pace. When the user signals readiness, it processes all answers in one pass. It then generates CONTEXT.md.

**When to use:** Use this mode for large phases with many gray areas. Also use it when users prefer to answer questions offline or asynchronously instead of in the chat session.
</purpose>

<trigger>
This workflow runs when the `--power` flag is present in ARGUMENTS to `/gsd:discuss-phase`.

The caller (discuss-phase.md) already did these actions:
- Made sure that the phase exists
- Supplied init context: `phase_dir`, `padded_phase`, `phase_number`, `phase_name`, `phase_slug`, `has_spec`, `spec_path`

Start at **Step 1** immediately.
</trigger>

<step name="check_spec">
Find out whether the phase has a SPEC.md from the spec-phase workflow. SPEC.md locks requirements before implementation decisions. Init already resolved this. Use `has_spec` and `spec_path`.

Both use one shared definition of "the phase SPEC": a plain `*-SPEC.md`, never `*-AI-SPEC.md` or `*-UI-SPEC.md`. Do not glob for it.

**If `has_spec` is true:**
1. Read the SPEC.md file at `spec_path`.
2. Count the requirements (numbered items in `## Requirements`).
3. Show: `Found SPEC.md — {N} requirements locked. Focusing on implementation decisions.`
4. Set `spec_loaded = true`.
5. Store requirements, boundaries, and acceptance criteria as `<locked_requirements>`. Put these directly into CONTEXT.md without asking again.

**If `has_spec` is false:** Continue with `spec_loaded = false`.

**Note:** `*-AI-SPEC.md` (from the ai-integration-phase workflow) and `*-UI-SPEC.md` (from the ui-phase workflow) are different artifacts and never count as the phase SPEC.
</step>

<step name="analyze">
Identify gray areas as in standard discuss-phase mode.

1. Load before context (PROJECT.md, REQUIREMENTS.md, STATE.md, before CONTEXT.md files).
2. Examine the codebase for reusable assets and patterns relevant to this phase.
3. Read the phase goal from ROADMAP.md.
4. Identify ALL gray areas: specific implementation decisions the user should consider. If `spec_loaded = true`, the locked requirements are already answered. Identify gray areas only about HOW to implement, never about WHAT to build or WHY.
5. For each gray area, generate 2–4 concrete options with tradeoff descriptions.

Group questions by topic into sections. Examples are "Visual Style", "Data Model", "Interactions", and "Error Handling". Each section should have 2–6 questions.

Do NOT ask the user anything at this stage. Record everything internally. Then continue to generate.
</step>

<step name="generate_json">
Write all questions to:

```
{phase_dir}/{padded_phase}-QUESTIONS.json
```

**JSON structure:**

```json
{
  "phase": "{padded_phase}-{phase_slug}",
  "generated_at": "ISO-8601 timestamp",
  "stats": {
    "total": 0,
    "answered": 0,
    "chat_more": 0,
    "remaining": 0
  },
  "sections": [
    {
      "id": "section-slug",
      "title": "Section Title",
      "questions": [
        {
          "id": "Q-01",
          "title": "Short question title",
          "context": "Codebase info, prior decisions, or constraints relevant to this question",
          "options": [
            {
              "id": "a",
              "label": "Option label",
              "description": "Tradeoff or elaboration for this option"
            },
            {
              "id": "b",
              "label": "Another option",
              "description": "Tradeoff or elaboration"
            },
            {
              "id": "c",
              "label": "Custom",
              "description": ""
            }
          ],
          "answer": null,
          "chat_more": "",
          "status": "unanswered"
        }
      ]
    }
  ]
}
```

**Field rules:**
- `stats.total`: count all questions across all sections.
- `stats.answered`: count questions where `answer` is not null and not an empty string.
- `stats.chat_more`: count questions where `chat_more` has content.
- `stats.remaining`: `total - answered`
- `question.id`: use sequential IDs across all sections — Q-01, Q-02, Q-03, ...
- `question.context`: use a concrete annotation about the codebase or an earlier decision, not a generic annotation.
- `question.answer`: use null until the user sets it. After the user answers, use the selected option id or free-text.
- `question.status`: "unanswered" | "answered" | "chat-more" (has chat_more but no answer yet)
</step>

<step name="generate_html">
Write a self-contained HTML companion file to:

```
{phase_dir}/{padded_phase}-QUESTIONS.html
```

The file must be a single self-contained HTML file with inline CSS and JavaScript. Do not use external dependencies.

**Layout:**

```
┌─────────────────────────────────────────────────────┐
│  Phase {N}: {phase_name} — Discussion Questions      │
│  ┌──────────────────────────────────────────────┐   │
│  │  12 total  |  3 answered  |  9 remaining     │   │
│  └──────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────┤
│  ▼ Visual Style (3 questions)                        │
│   ┌──────────┐ ┌──────────┐ ┌──────────┐            │
│   │ Q-01     │ │ Q-02     │ │ Q-03     │            │
│   │ Layout   │ │ Density  │ │ Colors   │            │
│   │ ...      │ │ ...      │ │ ...      │            │
│   └──────────┘ └──────────┘ └──────────┘            │
│  ▼ Data Model (2 questions)                          │
│   ...                                                │
└─────────────────────────────────────────────────────┘
```

**Stats bar:**
- Show total questions, answered count, and remaining count.
- Show a simple CSS progress bar (green fill = answered / total).

**Section headers:**
- Let a click show/hide questions in the section.
- Show the answered count for the section, for example "2/4 answered".

**Question cards (3-column grid):**
Each card contains:
- Show a question ID badge, for example "Q-01", and the title.
- Show a context annotation in gray italic text.
- Show an option list with radio buttons, bold labels, and description text.
- Show a Chat more textarea with an orange border when content is present.
- Highlight the card green when the question is answered.

**JavaScript behavior:**
- When the user selects a radio button, mark the question as answered in page state. Update the stats bar.
- When the user enters text in the textarea, update chat_more content in page state. If content is present, show an orange border.
- Put a "Save answers" button at the top and bottom. Each button serializes page state back to the JSON file path.

**Save mechanism:**
If the File System Access API is available, the Save button uses it to write the updated JSON back. Otherwise, the button generates a downloadable JSON file. The user can save this file over the original. Include clear instructions in the UI:

```
After answering, click "Save answers" — or download the JSON and replace the original file.
Then return to Claude and say "refresh" to process your answers.
```

**Answered question styling:**
- Card border: `2px solid #22c55e` (green)
- Card background: `#f0fdf4` (light green tint)

**Unanswered question styling:**
- Card border: `1px solid #e2e8f0` (gray)
- Card background: `white`

**Chat more textarea:**
- Placeholder: "Add context, nuance, or clarification for this question..."
- Normal border: `1px solid #e2e8f0`
- Active (has content) border: `2px solid #f97316` (orange)
</step>

<step name="notify_user">
After you write both files, print this message to the user:

```
Questions ready for Phase {N}: {phase_name}

  HTML (open in browser/IDE):   {phase_dir}/{padded_phase}-QUESTIONS.html
  JSON (state file):            {phase_dir}/{padded_phase}-QUESTIONS.json

  {total} questions across {section_count} topics.

Open the HTML file, answer the questions at your own pace, then save.

When ready, tell me:
  "refresh"   — process your answers and update the file
  "finalize"  — generate CONTEXT.md from all answered questions
  "explain Q-05"   — elaborate on a specific question
  "exit power mode" — return to standard one-by-one discussion (answers carry over)
```
</step>

<step name="wait_loop">
Enter wait mode. Claude listens for user commands and handles each command:

---

**"refresh"** (or "process answers", "update", "re-read"):

1. Read `{phase_dir}/{padded_phase}-QUESTIONS.json`.
2. Recalculate stats: count answered, chat_more, and remaining.
3. Write the updated stats back to the JSON.
4. Generate the HTML file again with the updated state. Highlight answered cards green and update the progress bar.
5. Report to the user:

```
Refreshed. Updated state:
  Answered:  {answered} / {total}
  Remaining: {remaining}
  Chat-more: {chat_more}

  {phase_dir}/{padded_phase}-QUESTIONS.html updated.

Answer more questions, then say "refresh" again, or say "finalize" when done.
```

---

**"finalize"** (or "done", "generate context", "write context"):

Proceed to the **finalize** step.

---

**"explain Q-{N}"** (or "more info on Q-{N}", "elaborate Q-{N}"):

1. Find the question by ID in the JSON.
2. Explain why this decision matters. Explain how it affects the downstream plan. Include more relevant context from the codebase.
3. Return to wait mode.

---

**"exit power mode"** (or "switch to interactive"):

1. Read all currently answered questions from JSON.
2. Load answers into the internal accumulator as if the user answered them interactively.
3. For any unanswered questions, continue with the standard `discuss_areas` step from discuss-phase.md.
4. Generate CONTEXT.md as normal.

---

**Any other message:**
Respond helpfully. Then remind the user of available commands:
```
(Power mode active — say "refresh", "finalize", "explain Q-N", or "exit power mode")
```
</step>

<step name="finalize">
Process all answered questions from the JSON file and generate CONTEXT.md.

1. Read `{phase_dir}/{padded_phase}-QUESTIONS.json`.
2. Filter to questions where `answer` is not null/empty.
3. Group decisions by section.
4. For each answered question, format a decision entry:
   - Decision: the selected option label, or custom text for a free-form answer
   - Rationale: the option description, plus `chat_more` content if present
   - Status: "Decided" if fully answered, "Needs clarification" if only chat_more with no option selected

5. Write CONTEXT.md in the standard context template format. Include these sections and references:
   - If `spec_loaded = true`, put the `<spec_lock>` section immediately after `<domain>` (template: `discuss-phase/templates/context.md`). Do NOT copy SPEC.md requirements into `<decisions>`.
   - If `spec_loaded = true`, put the SPEC.md file in `<canonical_refs>` with the note "Locked requirements — MUST read before planning".
   - Put all answered questions in the `<decisions>` section. Group them by section.
   - Put unanswered questions in the `<deferred_ideas>` section for future discussion.
   - Put any chat_more content that adds nuance in the `<specifics>` section.
   - Put reusable assets found during analysis in the `<code_context>` section.
   - Include the `<canonical_refs>` section (MANDATORY — paths to relevant specs/docs).

6. If fewer than 50% of questions were answered, warn the user:
```
Warning: Only {answered}/{total} questions answered ({pct}%).
CONTEXT.md generated with available decisions. Unanswered questions listed as deferred.
Consider running /gsd:discuss-phase {N} again to refine before planning.
```

7. Print completion message:
```
CONTEXT.md written: {phase_dir}/{padded_phase}-CONTEXT.md

  Decisions captured: {answered}
  Deferred:          {remaining}

Next step: /gsd:plan-phase {N}
```
</step>

<success_criteria>
- The generated JSON has a clear structure and covers all identified gray areas.
- The HTML companion file is self-contained and usable without a server.
- The stats bar shows accurate answered/remaining counts after each refresh.
- The HTML highlights answered questions green.
- The generated CONTEXT.md uses the same format as standard discuss-phase output.
- Unanswered questions stay as deferred items. The workflow does not silently discard them.
- The `canonical_refs` section is always present in CONTEXT.md (MANDATORY).
- The user knows how to refresh, finalize, explain, or exit power mode.
</success_criteria>
