# GSD Loop: Build

Each execution pass completes one unit of work: repair one rework PR, or implement one ready issue. Stop after one pass.

## 1. Setup

Before starting work, verify repository context:

```bash
gh repo view --json nameWithOwner
DEFAULT_BRANCH=$(gh repo view --json defaultBranchRef --jq .defaultBranchRef.name)
```

Ensure that the queue labels exist:

```bash
for l in gsd:ready gsd:blocked gsd:approved gsd:rework gsd:escalated; do
  gh label create "$l" --color ededed 2>/dev/null || true
done
```

## 2. Check for Dead Passes

Before claiming new work, inspect the local git state:

- If uncommitted changes exist on a `gsd/NNN-*` branch with an open PR, resume the repair flow.
- If uncommitted changes exist on a `gsd/NNN-*` branch with no PR, resume implementation.
- If uncommitted changes exist on an unrelated branch, stop immediately and report to the user.
- Do not stash or discard uncommitted changes from other branches.

## 3. Repair Queue

Give priority to pull requests carrying `gsd:rework`:

```bash
gh pr list --state open --label gsd:rework --limit 200 \
  --json number,title,headRefName,headRefOid,labels,updatedAt,url
```

Skip pull requests that also carry `gsd:escalated`.

For the oldest rework pull request:

1. Check out the pull request branch.
2. Read the reviewer feedback comments.
3. Fix only the identified issues.
4. Execute the verification suite.
5. Push the commit to the branch.
6. Drop the `gsd:rework` label and stop.

## 4. Build Queue

When no repair work remains, claim the oldest `gsd:ready` issue:

```bash
ISSUE=$(gh issue list --state open --label gsd:ready --limit 1 --json number --jq .[0].number)
```

1. Assign the issue to yourself:
   ```bash
   gh issue edit "$ISSUE" --add-assignee "@me"
   ```
2. Create and check out a dedicated branch:
   ```bash
   git checkout -b "gsd/$ISSUE-implementation" "$DEFAULT_BRANCH"
   ```
3. Implement the required changes. Follow project conventions.
4. Execute all verification tests:
   ```bash
   npm test
   nix flake check
   ```
5. Commit and push the branch:
   ```bash
   git commit -m "feat: resolve issue #$ISSUE"
   git push -u origin "gsd/$ISSUE-implementation"
   ```
6. Open the pull request and reference the issue:
   ```bash
   gh pr create --title "feat: resolve issue #$ISSUE" --body "Closes #$ISSUE"
   ```
7. Remove `gsd:ready` from the issue and stop.
