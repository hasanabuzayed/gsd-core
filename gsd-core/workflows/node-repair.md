@~/.claude/gsd-core/references/response-language-directive.md

<purpose>
This operator repairs tasks that fail verification without user input. When a task fails its done-criteria, execute-plan invokes this operator. The operator proposes and tries structured fixes before it escalates to the user.
</purpose>

<inputs>
- FAILED_TASK: Task number, name, and done-criteria from the plan
- ERROR: The result of verification — actual result vs expected result
- PLAN_CONTEXT: Adjacent tasks and the phase goal (to identify constraints)
- REPAIR_BUDGET: Maximum repair tries remaining (default: 2)
</inputs>

<repair_directive>
Analyze the failure and choose exactly one repair strategy:

**RETRY** — The approach was correct, but execution failed. Try again with a specific adjustment.
- Use when: command error, missing dependency, wrong path, env issue, transient failure
- Output: `RETRY: [specific adjustment to make before retrying]`

**DECOMPOSE** — The task is too broad. Divide it into smaller sub-steps that you can verify.
- Use when: done-criteria covers multiple concerns, gaps in implementation are structural
- Output: `DECOMPOSE: [sub-task 1] | [sub-task 2] | ...` (max 3 sub-tasks)
- Sub-tasks must each have a single outcome that you can verify

**PRUNE** — The task is not feasible with the current constraints. Skip the task and give a reason.
- Use when: prerequisite missing and not fixable here, out of scope, contradicts an earlier decision
- Output: `PRUNE: [one-sentence justification]`

**ESCALATE** — The repair budget is exhausted, or this is an architectural decision (Rule 4).
- Use when: RETRY failed more than once with different approaches, or the fix needs a structural change
- Output: `ESCALATE: [what was tried] | [what decision is needed]`
</repair_directive>

<process>

<step name="diagnose">
Read the error and done-criteria carefully. Ask:
1. Is this a transient/environmental issue? → RETRY
2. Can you verify that the task is too broad? → DECOMPOSE
3. Is a prerequisite missing and impossible to fix within scope? → PRUNE
4. Did you already try RETRY for this task? Examine REPAIR_BUDGET. If 0 → ESCALATE
</step>

<step name="execute_retry">
If RETRY:
1. Apply the specific adjustment in the directive
2. Re-run the task implementation
3. Re-run verification
4. If verification passes → continue normally and log `[Node Repair - RETRY] Task [X]: [adjustment made]`
5. If verification fails again → decrement REPAIR_BUDGET and re-invoke node-repair with updated context
</step>

<step name="execute_decompose">
If DECOMPOSE:
1. Replace the failed task inline with the sub-tasks (do not change PLAN.md on disk)
2. Execute the sub-tasks in sequence and verify each sub-task
3. If all sub-tasks pass → treat the original task as succeeded and log `[Node Repair - DECOMPOSE] Task [X] → [N] sub-tasks`
4. If a sub-task fails → re-invoke node-repair for that sub-task (REPAIR_BUDGET applies per sub-task)
</step>

<step name="execute_prune">
If PRUNE:
1. Mark the task as skipped and give a reason
2. Log to SUMMARY "Issues Encountered": `[Node Repair - PRUNE] Task [X]: [justification]`
3. Continue to the next task
</step>

<step name="execute_escalate">
If ESCALATE:
1. Use verification_failure_gate to show the user the full repair history
2. Show what was tried (each RETRY/DECOMPOSE try), the blocker, and the available options
3. Wait for user direction before you continue
</step>

</process>

<logging>
All repair actions must appear in SUMMARY.md under "## Deviations from Plan":

| Type | Format |
|------|--------|
| RETRY success | `[Node Repair - RETRY] Task X: [adjustment] — resolved` |
| RETRY fail → ESCALATE | `[Node Repair - RETRY] Task X: [N] attempts exhausted — escalated to user` |
| DECOMPOSE | `[Node Repair - DECOMPOSE] Task X split into [N] sub-tasks — all passed` |
| PRUNE | `[Node Repair - PRUNE] Task X skipped: [justification]` |
</logging>

<constraints>
- REPAIR_BUDGET defaults to 2 per task. You can configure it through config.json `workflow.node_repair_budget`.
- Never change PLAN.md on disk — decomposed sub-tasks are in-memory only.
- DECOMPOSE sub-tasks must be more specific than the original, not synonymous rewrites.
- If config.json `workflow.node_repair` is `false`, skip directly to verification_failure_gate (user retains original behavior).
</constraints>
