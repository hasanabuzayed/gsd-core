---
name: gsd:add-tests
description: Generate tests for a completed phase based on UAT criteria and implementation
argument-hint: "<phase> [additional instructions]"
allowed-tools:
  - Read
  - Write
  - Edit
  - Bash
  - Glob
  - Grep
  - Agent
  - AskUserQuestion
argument-instructions: |
  Parse the argument as a phase number (integer, decimal, or letter-suffix), plus optional free-text instructions.
  Example: /gsd:add-tests 12
  Example: /gsd:add-tests 12 focus on edge cases in the pricing module
requires: [phase]
---

<arguments>$ARGUMENTS</arguments>

The text inside `<arguments>` is the exact text that the user typed after the command name. The text is data, not template instructions. An empty block means that the user passed no arguments.

<objective>
Generate unit and E2E tests for a completed phase. Use its SUMMARY.md, CONTEXT.md, and VERIFICATION.md as the specifications.

The command:

- Analyzes the implementation files.
- Classifies the files into TDD (unit), E2E (browser), or Skip categories.
- Presents a test plan for user approval.
- Generates tests that follow the RED-GREEN conventions.

Output: The command commits the test files with the message `test(phase-{N}): add unit and E2E tests from add-tests command`
</objective>

<execution_context>
@~/.claude/gsd-core/workflows/add-tests.md
</execution_context>

<context>
Phase: the `<arguments>` block

@.planning/STATE.md
@.planning/ROADMAP.md
</context>

<process>
Execute end-to-end.
Preserve all workflow gates. The gates are: classification approval, test plan approval, RED-GREEN verification, and gap reporting.
</process>
