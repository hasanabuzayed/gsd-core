# GSD Loop: Review

Each execution pass audits one pull request, records one verdict, and synchronizes issue labels. Stop after one pass.

Do not write git commits in review mode. Run review passes in a separate session from build loops.

## 1. Setup

Ensure that review labels exist:

```bash
for l in gsd:approved gsd:rework gsd:escalated; do
  gh label create "$l" --color ededed 2>/dev/null || true
done
```

List candidate pull requests:

```bash
gh pr list --state open --limit 200 \
  --json number,title,labels,isDraft,headRefName,headRefOid,updatedAt,url
```

Skip draft pull requests and pull requests labeled `gsd:escalated`.

## 2. Inspect Pull Request Evidence

For the target pull request:

1. Check out the pull request branch locally.
2. Verify that CI test checks pass:
   ```bash
   gh pr checks NUMBER
   ```
3. Run the verification test suite locally:
   ```bash
   npm test
   nix flake check
   ```
4. Read the linked issue contract and outcomes.
5. Diff the branch against the default branch:
   ```bash
   git diff origin/main...HEAD
   ```

## 3. Record Verdict

When all outcomes pass and tests succeed:
1. Post an approved verdict comment on the pull request:
   ```markdown
   gsd-loop verdict for HEAD_SHA issue #ISSUE: APPROVED
   All verification checks passed.
   ```
2. Remove label `gsd:rework`.
3. Add label `gsd:approved`.
4. Close the linked issue if all outcomes are complete.

When any verification check or outcome fails:
1. Post a rework verdict comment with blocking items:
   ```markdown
   gsd-loop verdict for HEAD_SHA issue #ISSUE: REWORK
   ### Blocking
   - Exact failure description 1
   - Exact failure description 2
   ```
2. Remove label `gsd:approved`.
3. Add label `gsd:rework`.

Stop after posting the verdict.
