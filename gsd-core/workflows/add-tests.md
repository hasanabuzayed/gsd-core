<purpose>
Generate unit and E2E tests for a completed phase. Use its SUMMARY.md, CONTEXT.md, and implementation. Classify each changed file as TDD (unit), E2E (browser), or Skip. Present a test plan for user approval. Then generate tests that follow RED-GREEN conventions.

Users currently write `/gsd:quick` prompts by hand to generate tests after each phase. This workflow standardizes the process with classification, quality gates, and gap reporting.
</purpose>

<required_reading>
Before you start, read all files that the invoking prompt's execution_context references.
</required_reading>

<process>

<step name="parse_arguments">
Parse `$ARGUMENTS` for:
- Store the phase number (integer, decimal, or letter-suffix) as `$PHASE_ARG`.
- Store the text after the phase number as `$EXTRA_INSTRUCTIONS` (optional).

Example: `/gsd:add-tests 12 focus on edge cases` → `$PHASE_ARG=12`, `$EXTRA_INSTRUCTIONS="focus on edge cases"`

If the user does not provide a phase argument, show:

```
ERROR: Phase number required
Usage: /gsd:add-tests <phase> [additional instructions]
Example: /gsd:add-tests 12
Example: /gsd:add-tests 12 focus on edge cases in the pricing module
```

Exit.
</step>

<step name="init_context">
Load phase operation context:

```bash
IFS= read -rd '' GSD_ARGS <<'GSD_ARGS_END' || true
$ARGUMENTS
GSD_ARGS_END
GSD_ARGS=${GSD_ARGS%$'\n'}
_GSD_SHIM_NAME="gsd-tools.cjs"; _GSD_RUNTIME_ROOT="${RUNTIME_DIR:-$(git rev-parse --show-toplevel 2>/dev/null || pwd)}"; GSD_TOOLS="${_GSD_RUNTIME_ROOT}/gsd-core/bin/${_GSD_SHIM_NAME}"; _gsd_at() { for _p; do if [ -f "$_p" ]; then GSD_TOOLS="$_p"; return 0; fi; done; return 1; }; _gsd_id_ok() { case "$("$1" runtime-identity --raw 2>/dev/null || true)" in '{"packageName":"@opengsd/gsd-core"'*'}') return 0;; *) return 1;; esac; }; _gsd_homes() { set -- "${CLAUDE_CONFIG_DIR:-$HOME/.claude}" "${ANTIGRAVITY_CONFIG_DIR:-$HOME/.gemini/antigravity}" "$HOME/.gemini/antigravity-ide" "$HOME/.gemini/antigravity-cli" "${AUGMENT_CONFIG_DIR:-$HOME/.augment}" "${CLINE_CONFIG_DIR:-$HOME/.cline}" "${CODEBUDDY_CONFIG_DIR:-$HOME/.codebuddy}" "${CODEX_HOME:-$HOME/.codex}" "${COPILOT_CONFIG_DIR:-${COPILOT_HOME:-$HOME/.copilot}}" "${CURSOR_CONFIG_DIR:-$HOME/.cursor}" "${HERMES_HOME:-$HOME/.hermes}" "${KILO_CONFIG_DIR:-${XDG_CONFIG_HOME:-$HOME/.config}/kilo}" "${KIMI_CONFIG_DIR:-$HOME/.config/agents}" "$HOME/.agents" "${KIMI_CODE_HOME:-$HOME/.kimi-code}" "${OPENCODE_CONFIG_DIR:-${XDG_CONFIG_HOME:-$HOME/.config}/opencode}" "${PI_CODING_AGENT_DIR:-$HOME/.pi/agent}" "${QWEN_CONFIG_DIR:-$HOME/.qwen}" "${TRAE_CONFIG_DIR:-$HOME/.trae}" "${WINDSURF_CONFIG_DIR:-$HOME/.codeium/windsurf}" "${ZCODE_CONFIG_DIR:-$HOME/.zcode}" "${GROK_AGENTS_HOME:-$HOME/.agents}"; for _h; do _gsd_at "$_h/gsd-core/bin/${_GSD_SHIM_NAME}" && return 0; done; return 1; }; if _gsd_at "${_GSD_RUNTIME_ROOT}/gsd-core/bin/${_GSD_SHIM_NAME}" "${_GSD_RUNTIME_ROOT}/.claude/gsd-core/bin/${_GSD_SHIM_NAME}" "${_GSD_RUNTIME_ROOT}/.codex/gsd-core/bin/${_GSD_SHIM_NAME}"; then gsd_run() { node "$GSD_TOOLS" "$@"; }; elif _gsd_homes; then gsd_run() { node "$GSD_TOOLS" "$@"; }; elif unset -f gsd_run; _G="$(command -v gsd_run)"; [ -n "$_G" ] && _gsd_id_ok "$_G"; then GSD_TOOLS="$_G"; gsd_run() { "$GSD_TOOLS" "$@"; }; else echo "ERROR: gsd-tools.cjs not found at $GSD_TOOLS and no identity-proving gsd_run is on PATH. Run: npx -y @opengsd/gsd-core@latest --claude --local" >&2; exit 1; fi; GSD_IDENTITY_STATUS=unverified; _gsd_id_ok gsd_run && GSD_IDENTITY_STATUS=ok; export GSD_IDENTITY_STATUS; [ "$GSD_IDENTITY_STATUS" = ok ] || echo "WARNING: \"$GSD_TOOLS\" did not prove it is @opengsd/gsd-core - it is either a different package or an @opengsd/gsd-core older than the runtime-identity verb. See docs/how-to/diagnose-a-foreign-gsd-tools.md" >&2; if [ -n "${CLAUDE_ENV_FILE:-}" ] && [ -n "${GSD_TOOLS:-}" ]; then printf "export PATH='%s':\"\$PATH\"\n" "${GSD_TOOLS%/*}" >> "$CLAUDE_ENV_FILE" 2>/dev/null || true; fi
GSD_WS=$(echo " $GSD_ARGS" | sed -nE 's/.* --ws +([A-Za-z0-9][A-Za-z0-9._-]*).*/--ws \1/p' | head -n 1)
INIT=$(gsd_run query init.phase-op ${GSD_WS:+--ws=${GSD_WS##* }} "${PHASE_ARG}")
if [[ "$INIT" == @file:* ]]; then INIT=$(cat "${INIT#@file:}"); fi
```

Extract these values from init JSON: `phase_dir`, `phase_number`, `phase_name`, `response_language`.

**If `response_language` is set:** You MUST present all user-facing output of this workflow in `{response_language}`. This output includes narration between tool calls, status updates, progress notes, findings, questions, prompts, and explanations. Keep technical terms, code, file paths, and subagent prompts in English. Translate only user-facing output.

Make sure that the phase directory exists. If it does not exist, show:
```
ERROR: Phase directory not found for phase ${PHASE_ARG}
Ensure the phase exists in .planning/phases/
```
Exit.

Read the phase artifacts in this order of priority:
1. `${phase_dir}/*-SUMMARY.md` — the implementation and changed files
2. `${phase_dir}/CONTEXT.md` — acceptance criteria and decisions
3. `${phase_dir}/*-VERIFICATION.md` — scenarios that the user verified, if UAT was done

If no SUMMARY.md exists, show:
```
ERROR: No SUMMARY.md found for phase ${PHASE_ARG}
This command works on completed phases. Run /gsd:execute-phase first.
```
Exit.

Present the banner:
```
### GSD ► ADD TESTS — Phase ${phase_number}: ${phase_name}
```
</step>

<step name="analyze_implementation">
Find the "Files Changed" section of SUMMARY.md or an equivalent section. Extract its list of files that the phase changed.

For each file, select one of three categories:

| Category | Criteria | Test Type |
|----------|----------|-----------|
| **TDD** | Pure functions for which you can write `expect(fn(input)).toBe(output)` | Unit tests |
| **E2E** | UI behavior that browser automation can test | Playwright/E2E tests |
| **Skip** | Files that tests cannot meaningfully test or that tests already cover | None |

**TDD classification — apply when:**
- Business logic: calculations, pricing, tax rules, validation
- Data transformations: mapping, filtering, aggregation, formatting
- Parsers: CSV, JSON, XML, custom format parsing
- Validators: input validation, schema validation, business rules
- State machines: status transitions, workflow steps
- Utilities: string manipulation, date handling, number formatting

**E2E classification — apply when:**
- Keyboard shortcuts: key bindings, modifier keys, chord sequences
- Navigation: page transitions, routing, breadcrumbs, back/forward
- Form interactions: submit, validation errors, field focus, autocomplete
- Selection: row selection, multi-select, shift-click ranges
- Drag and drop: reordering, moving between containers
- Modal dialogs: open, close, confirm, cancel
- Data grids: sorting, filtering, inline editing, column resize

**Skip classification — apply when:**
- UI layout/styling: CSS classes, visual appearance, responsive breakpoints
- Configuration: config files, environment variables, feature flags
- Glue code: dependency injection setup, middleware registration, routing tables
- Migrations: database migrations, schema changes
- Simple CRUD: basic create/read/update/delete with no business logic
- Type definitions: records, DTOs, interfaces with no logic

Read each file to make sure that its classification is correct. Do not classify a file based on its filename alone.
</step>

<step name="present_classification">
Present the classification to the user for confirmation before you continue:

**Text mode (`workflow.text_mode: true` in config or `--text` flag):** If `--text` is in `$ARGUMENTS` OR `text_mode` from init JSON is `true`, set `TEXT_MODE=true`. When TEXT_MODE is active, replace every `AskUserQuestion` call with a plain-text numbered list. Ask the user to type their choice number.

Non-Claude runtimes such as OpenAI Codex and Antigravity need this because `AskUserQuestion` is not available.

```
AskUserQuestion(
  header: "Test Classification",
  question: |
    ## Files classified for testing

    ### TDD (Unit Tests) — {N} files
    {list of files with brief reason}

    ### E2E (Browser Tests) — {M} files
    {list of files with brief reason}

    ### Skip — {K} files
    {list of files with brief reason}

    {if $EXTRA_INSTRUCTIONS: "Additional instructions: ${EXTRA_INSTRUCTIONS}"}

    How would you like to proceed?
  options:
    - "Approve and generate test plan"
    - "Adjust classification (I'll specify changes)"
    - "Cancel"
)
```

If the user selects "Adjust classification", apply their changes and present the classification again.
If the user selects "Cancel", exit gracefully.
</step>

<step name="discover_test_structure">
Before you generate the test plan, find the project's existing test structure:

```bash
# Find existing test directories
find . -type d -name "*test*" -o -name "*spec*" -o -name "*__tests__*" 2>/dev/null | head -20
# Find existing test files for convention matching
find . -type f \( -name "*.test.*" -o -name "*.spec.*" -o -name "*Tests.fs" -o -name "*Test.fs" \) 2>/dev/null | head -20
# Check for test runners
ls package.json *.sln 2>/dev/null || true
```

Identify:
- Test directory structure (where to find unit tests and E2E tests)
- Naming conventions (such as `.test.ts`, `.spec.ts`, `*Tests.fs`)
- Test runner commands (how to run unit tests and E2E tests)
- Test framework (such as xUnit, NUnit, Jest, Playwright)

If the test structure is ambiguous, ask the user:
```
AskUserQuestion(
  header: "Test Structure",
  question: "I found multiple test locations. Where should I create tests?",
  options: [list discovered locations]
)
```
</step>

<step name="generate_test_plan">
For each approved file, create a detailed test plan.

**For TDD files**, plan tests that follow RED-GREEN-REFACTOR:
1. Identify functions/methods in the file that tests can test.
2. For each function, list input scenarios, expected outputs, and edge cases.
3. The code already exists, so tests may pass immediately. That is OK. Make sure that they test the RIGHT behavior.

**For E2E files**, plan tests that follow RED-GREEN gates:
1. Identify user scenarios from CONTEXT.md/VERIFICATION.md.
2. For each scenario, describe the user action, expected outcome, and assertions.
3. A RED gate means that you make sure that the test would fail if the feature were broken.

Present the complete test plan:

```
AskUserQuestion(
  header: "Test Plan",
  question: |
    ## Test Generation Plan

    ### Unit Tests ({N} tests across {M} files)
    {for each file: test file path, list of test cases}

    ### E2E Tests ({P} tests across {Q} files)
    {for each file: test file path, list of test scenarios}

    ### Test Commands
    - Unit: {discovered test command}
    - E2E: {discovered e2e command}

    Ready to generate?
  options:
    - "Generate all"
    - "Cherry-pick (I'll specify which)"
    - "Adjust plan"
)
```

If the user selects "Cherry-pick", ask which tests to include.
If the user selects "Adjust plan", apply the changes and present the plan again.
</step>

<step name="execute_tdd_generation">
For each approved TDD test:

1. Before you write the test, **create test file** that follows the project conventions you found (directory, naming, imports).

2. **Write test** with a clear arrange/act/assert structure:
   ```
   // Arrange — set up inputs and expected outputs
   // Act — call the function under test
   // Assert — verify the output matches expectations
   ```

3. **Run the test**:
   ```bash
   {discovered test command}
   ```

4. **Evaluate result:**
   - **Test passes**: The implementation satisfies the test. After the test passes, make sure that it tests meaningful behavior, not just that the code compiles.
   - **Test fails with assertion error**: This may be a genuine bug that the test found. Flag it:
     ```
     ⚠️ Potential bug found: {test name}
     Expected: {expected}
     Actual: {actual}
     File: {implementation file}
     ```
     Do NOT fix the implementation. This is a test-generation command, not a fix command. Record the finding.
   - **Test fails with error (such as import or syntax)**: This is a test error. Fix the test and run it again.
</step>

<step name="execute_e2e_generation">
For each approved E2E test:

1. **Check for existing tests** that cover the same scenario:
   ```bash
   grep -r "{scenario keyword}" {e2e test directory} 2>/dev/null || true
   ```
   If you find a test, extend it rather than duplicate it.

2. **Create test file** for the user scenario from CONTEXT.md/VERIFICATION.md.

3. **Run the E2E test**:
   ```bash
   {discovered e2e command}
   ```

4. **Evaluate result:**
   - **GREEN (passes)**: Record success.
   - **RED (fails)**: Determine whether the cause is a test problem or a genuine application bug. Flag bugs:
     ```
     ⚠️ E2E failure: {test name}
     Scenario: {description}
     Error: {error message}
     ```
   - **Cannot run**: Report the blocker. Do NOT mark the test as complete.
     ```
     🛑 E2E blocker: {reason tests cannot run}
     ```

**No-skip rule:** If E2E tests cannot run because of missing dependencies or environment problems, report the blocker. Mark the test as incomplete. Never mark success without running the test.
</step>

<step name="summary_and_commit">
Create a test coverage report and present it to the user:

```
### GSD ► TEST GENERATION COMPLETE

## Results

| Category | Generated | Passing | Failing | Blocked |
|----------|-----------|---------|---------|---------|
| Unit     | {N}       | {n1}    | {n2}    | {n3}    |
| E2E      | {M}       | {m1}    | {m2}    | {m3}    |

## Files Created/Modified
{list of test files with paths}

## Coverage Gaps
{areas that couldn't be tested and why}

## Bugs Discovered
{any assertion failures that indicate implementation bugs}
```

Record test generation in project state:
```bash
gsd_run query state-snapshot
```

If there are tests to commit that pass:

```bash
git add {test files}
git commit -m "test(phase-${phase_number}): add unit and E2E tests from add-tests command" -- {test files}
```

Present next steps:

```
---

## ▶ Next Up — [${PROJECT_CODE}] ${PROJECT_TITLE}

{if bugs discovered:}
**Fix discovered bugs:** `/gsd:quick fix the {N} test failures discovered in phase ${phase_number}`

{if blocked tests:}
**Resolve test blockers:** {description of what's needed}

{otherwise:}
**All tests passing!** Phase ${phase_number} is fully tested.

---

**Also available:**
- `/gsd:add-tests {next_phase}` — test another phase
- `/gsd:verify-work {phase_number}` — run UAT verification

---
```
</step>

</process>

<success_criteria>
- [ ] Load phase artifacts (SUMMARY.md, CONTEXT.md, optionally VERIFICATION.md).
- [ ] Classify all changed files into TDD/E2E/Skip categories.
- [ ] Present the classification to the user and get approval.
- [ ] Find the project test structure (directories, conventions, runners).
- [ ] Present the test plan to the user and get approval.
- [ ] Generate TDD tests with arrange/act/assert structure.
- [ ] Generate E2E tests for user scenarios.
- [ ] Run all tests. Do not mark untested tests as passing.
- [ ] Flag bugs that tests find. Do not fix them.
- [ ] Commit test files with the proper message.
- [ ] Document coverage gaps.
- [ ] Present next steps to the user.
</success_criteria>
