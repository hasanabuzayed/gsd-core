# GSD Loop: Discover

Use this playbook interactively with a user present. Do not schedule this playbook. Do not write production code or open pull requests.

This playbook charts or advances one discovery map. It divides complex goals into delivery slices for subsequent implementation.

## 1. Setup

Before starting, resolve the repository name:

```bash
gh repo view --json nameWithOwner
```

Ensure that the `gsd:map` label exists:

```bash
gh label create gsd:map --color ededed --description "Multi-session discovery map" 2>/dev/null || true
```

Use GitHub API version `2026-03-10` for commands in this playbook.

## 2. Select the Mode

If the user provides an issue number or URL, advance that map. Otherwise, chart a new map.

A valid map is an open issue that has the `gsd:map` label. Do not identify a map by title alone.

## 3. Chart a Map

### 3.1 Explore the Repository

Before asking questions, inspect the codebase, repository documentation, and related issues. Reserve user questions for product decisions and priorities.

### 3.2 Define Scope and Boundaries

Interview the user in brief rounds. Identify:

- The destination state.
- Explicit non-goals.
- Known risks and constraints.

### 3.3 Create the Discovery Map Issue

Create the tracking issue with label `gsd:map`:

```markdown
# [Map] TITLE

## Destination
Define the objective in 1-2 paragraphs.

## Delivery Slices
- [ ] Slice 1: Short summary
- [ ] Slice 2: Short summary

## Decisions so far
List resolved architectural and product choices.

## Non-goals
List items out of scope.
```

## 4. Advance a Map

When advancing an existing map:

1. Fetch the latest map issue content.
2. Resolve exactly one open decision frontier.
3. Update the map issue with the decision record under `## Decisions so far`.
4. When all slices are ready, set `## Graduation` to:
   ```markdown
   Ready for `gsd-loop-spec`.
   ```
5. Report the updated status to the user and stop.
