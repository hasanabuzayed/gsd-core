# GSD Loop: Spec

Use this playbook interactively with a user present. Do not run this playbook unattended.

This playbook turns rough ideas or discovery maps into self-sufficient GitHub issues for autonomous build agents.

## 1. Setup

Before creating issues, verify the target repository:

```bash
gh repo view --json nameWithOwner
```

Ensure that the queue labels exist:

```bash
for l in gsd:ready gsd:blocked gsd:approved gsd:rework gsd:escalated; do
  gh label create "$l" --color ededed 2>/dev/null || true
done
```

## 2. Process Input

When the user provides a discovery map:

1. Confirm that the issue carries the `gsd:map` label.
2. Confirm that `## Graduation` equals `Ready for gsd-loop-spec.`.
3. Validate the slice structure:
   ```bash
   node loop/scripts/validate-discovery-map.mjs /path/to/map.md
   ```

When the user provides a direct idea without a map, interview the user to define acceptance outcomes.

## 3. Specify the Contract

Draft the specification issue with these required sections:

```markdown
# [Slice] TITLE

## Problem Statement
Describe the problem and context in 1-2 paragraphs.

## Target Outcomes
- [ ] Explicit verifiable outcome 1
- [ ] Explicit verifiable outcome 2

## Verification Gates
List exact shell commands that prove completion:
- `npm test`
- `nix flake check`

## Non-goals
List items explicitly excluded from this task.
```

## 4. File the Issue

1. Confirm the specification with the user.
2. Create the GitHub issue:
   ```bash
   gh issue create --title "TITLE" --body-file spec.md --label "gsd:ready"
   ```
3. Output the created issue URL to the user and stop.
