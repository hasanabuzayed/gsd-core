@~/.claude/gsd-core/references/response-language-directive.md

# Forensics Workflow

This workflow investigates failed or stuck GSD workflows after an incident.
It examines git history, `.planning/` artifacts, and file system state.
It detects anomalies and generates a structured diagnostic report.


**Principle:** This is a read-only investigation. Do not change project files.
Only write the forensic report.

---

## Step 1: Get Problem Description

```bash
IFS= read -rd '' GSD_ARGS <<'GSD_ARGS_END' || true
$ARGUMENTS
GSD_ARGS_END
GSD_ARGS=${GSD_ARGS%$'\n'}
PROBLEM="$GSD_ARGS"
```

If `$ARGUMENTS` is empty, ask the user:
> "What went wrong? Describe the problem. For example, 'autonomous mode got stuck on phase 3',
> 'execute-phase failed silently', or 'costs seem unusually high'."


Record the problem description for the report.

## Step 2: Gather Evidence

Collect data from all available sources. If a source is missing, use the sources that exist.


### 2a. Git History

```bash
# Recent commits (last 30)
git log --oneline -30

# Commits with timestamps for gap analysis
git log --format="%H %ai %s" -30

# Files changed in recent commits (detect repeated edits)
git log --name-only --format="" -20 | sort | uniq -c | sort -rn | head -20

# Uncommitted work
git status --short
git diff --stat
```

Record:
- Commit timeline (dates, messages, frequency)
- Most-edited files (a possible signal of a stuck loop)
- Uncommitted changes (a possible crash/interruption indicator)


### 2b. Planning State

If these files exist, read them:

- `.planning/STATE.md` — current milestone, phase, progress, blockers, last session
- `.planning/ROADMAP.md` — phase list with status
- `.planning/config.json` — workflow configuration

Extract:
- Current phase and its status
- Last recorded session stop point
- Any blockers or flags

### 2c. Phase Artifacts

For each phase directory in `.planning/phases/*/`:

```bash
ls .planning/phases/*/
```

For each phase, identify the artifacts that exist:

- `{padded}-PLAN.md` or `{padded}-PLAN-*.md` (execution plans)
- `{padded}-SUMMARY.md` (completion summary)
- `{padded}-VERIFICATION.md` (quality verification)
- `{padded}-CONTEXT.md` (design decisions)
- `{padded}-RESEARCH.md` (pre-planning research)

Record which phases have complete artifact sets and which have gaps.


### 2d. Session Reports

If `.planning/reports/SESSION_REPORT.md` exists, read it.
If it exists, extract the last session outcomes, completed work, and token estimates.



### 2e. Git Worktree State

```bash
git worktree list
```

Look for orphaned worktrees from crashed agents.


## Step 3: Detect Anomalies

Compare the evidence with these anomaly patterns:


### Stuck Loop Detection

**Signal:** The same file appears in 3+ consecutive commits in a short time.


```bash
# Look for files committed repeatedly in sequence
git log --name-only --format="---COMMIT---" -20
```

Parse the commit boundaries. If any file appears in 3+ consecutive commits, flag it as follows:
- **Confidence HIGH** if the commit messages are similar (for example, "fix:", "fix:", "fix:" on the same file)
- **Confidence MEDIUM** if the file appears frequently but the commit messages vary


### Missing Artifact Detection

**Signal:** A phase appears complete but does not have the expected artifacts. The phase has commits and is past in the roadmap.


For each phase that should be complete:
- PLAN.md missing → the planning step was skipped
- SUMMARY.md missing → the phase was not properly closed
- VERIFICATION.md missing → the quality check was skipped


### Partial-plan Drift Detection

**Signal:** commits exist but SUMMARY.md is missing for the current or recently
active plan.

Run the same comparison as the execute-phase safe-resume verifier:
- Identify the active plan from STATE.md/phase artifacts.
- Search git history for that plan id.
- Compare the results against the expected SUMMARY.md path.

If production commits exist but SUMMARY.md is missing, flag a high-confidence partial-plan drift anomaly.
This usually means that an executor was interrupted after implementation commits but before atomic close-out.


### Abandoned Work Detection

**Signal:** A large gap separates the last commit from the current time. STATE.md shows that execution is in progress.


```bash
# Time since last commit
git log -1 --format="%ai"
```

If STATE.md shows an active phase, the last commit is >2 hours old, and uncommitted changes exist, flag potential abandonment or crash.


### Crash/Interruption Detection

**Signal:** Uncommitted changes exist, STATE.md shows that execution is in progress, and orphaned worktrees exist.


Combine:
- `git status` shows `modified/staged` files
- STATE.md has an active execution entry
- `git worktree list` shows worktrees beyond the main one

### Scope Drift Detection

**Signal:** Recent commits change files outside the expected scope of the current phase.

Read the current phase PLAN.md to find the expected file paths.
Compare them with the files that recent commits changed.
Flag any files that are clearly outside the domain of the phase.


### Test Regression Detection

**Signal:** Commit messages contain "fix test" or "revert", or test files have repeat commits.



```bash
git log --oneline -20 | grep -iE "fix test|revert|broken|regression|fail"
```

## Step 4: Generate Report

If needed, create the forensics directory:

```bash
mkdir -p .planning/forensics
```

Write to `.planning/forensics/report-$(date +%Y%m%d-%H%M%S).md`:

```markdown
# Forensic Report

**Generated:** {ISO timestamp}
**Problem:** {user's description}

---

## Evidence Summary

### Git Activity
- **Last commit:** {date} — "{message}"
- **Commits (last 30):** {count}
- **Time span:** {earliest} → {latest}
- **Uncommitted changes:** {yes/no — list if yes}
- **Active worktrees:** {count — list if >1}

### Planning State
- **Current milestone:** {version or "none"}
- **Current phase:** {number — name — status}
- **Last session:** {stopped_at from STATE.md}
- **Blockers:** {any flags from STATE.md}

### Artifact Completeness
| Phase | PLAN | CONTEXT | RESEARCH | SUMMARY | VERIFICATION |
|-------|------|---------|----------|---------|-------------|
{for each phase: name | ✅/❌ per artifact}

## Anomalies Detected

### {Anomaly Type} — {Confidence: HIGH/MEDIUM/LOW}
**Evidence:** {specific commits, files, or state data}
**Interpretation:** {what this likely means}

{repeat for each anomaly found}

## Root Cause Hypothesis

Based on the evidence above, the most likely explanation is:

{1-3 sentence hypothesis grounded in the anomalies}

## Recommended Actions

1. {Specific, actionable remediation step}
2. {Another step if applicable}
3. {Recovery command if applicable — e.g., `/gsd:resume-work`, `/gsd:execute-phase N`}

---

*Report generated by `/gsd:forensics`. All paths redacted for portability.*
```

**Redaction rules:**
- Replace absolute paths with relative paths (remove the `$HOME` prefix)
- Remove any API keys, tokens, or credentials found in git diff output
- Truncate large diffs to the first 50 lines


## Step 5: Present Report

Show the full forensic report inline.


## Step 6: Offer Interactive Investigation

> "Report saved to `.planning/forensics/report-{timestamp}.md`.
>
> I can examine any finding in more detail. Do you want me to:
> - Trace a specific anomaly to its root cause?
> - Read specific files referenced in the evidence?
> - Find out if someone reported a similar issue before?"


If the user asks follow-up questions, answer from the evidence already gathered.
If more files are specifically needed, read them.


## Step 7: Offer Issue Creation

If you found actionable anomalies (HIGH or MEDIUM confidence), ask:


> "Want me to create a GitHub issue for this? I will format the findings and redact paths."

If confirmed:
```bash
# Check if "bug" label exists before using it
BUG_LABEL=$(gh label list --repo open-gsd/gsd-core --search "bug" --json name -q '.[0].name' 2>/dev/null)
LABEL_FLAG=""
if [ -n "$BUG_LABEL" ]; then
  LABEL_FLAG="--label bug"
fi

gh issue create \
  --repo open-gsd/gsd-core \
  --title "bug: {concise description from anomaly}" \
  $LABEL_FLAG \
  --body "{formatted findings from report}"
```

## Step 8: Update STATE.md

```bash
_GSD_SHIM_NAME="gsd-tools.cjs"; _GSD_RUNTIME_ROOT="${RUNTIME_DIR:-$(git rev-parse --show-toplevel 2>/dev/null || pwd)}"; GSD_TOOLS="${_GSD_RUNTIME_ROOT}/gsd-core/bin/${_GSD_SHIM_NAME}"; _gsd_at() { for _p; do if [ -f "$_p" ]; then GSD_TOOLS="$_p"; return 0; fi; done; return 1; }; _gsd_id_ok() { case "$("$1" runtime-identity --raw 2>/dev/null || true)" in '{"packageName":"@opengsd/gsd-core"'*'}') return 0;; *) return 1;; esac; }; _gsd_homes() { set -- "${CLAUDE_CONFIG_DIR:-$HOME/.claude}" "${ANTIGRAVITY_CONFIG_DIR:-$HOME/.gemini/antigravity}" "$HOME/.gemini/antigravity-ide" "$HOME/.gemini/antigravity-cli" "${AUGMENT_CONFIG_DIR:-$HOME/.augment}" "${CLINE_CONFIG_DIR:-$HOME/.cline}" "${CODEBUDDY_CONFIG_DIR:-$HOME/.codebuddy}" "${CODEX_HOME:-$HOME/.codex}" "${COPILOT_CONFIG_DIR:-${COPILOT_HOME:-$HOME/.copilot}}" "${CURSOR_CONFIG_DIR:-$HOME/.cursor}" "${HERMES_HOME:-$HOME/.hermes}" "${KILO_CONFIG_DIR:-${XDG_CONFIG_HOME:-$HOME/.config}/kilo}" "${KIMI_CONFIG_DIR:-$HOME/.config/agents}" "$HOME/.agents" "${KIMI_CODE_HOME:-$HOME/.kimi-code}" "${OPENCODE_CONFIG_DIR:-${XDG_CONFIG_HOME:-$HOME/.config}/opencode}" "${PI_CODING_AGENT_DIR:-$HOME/.pi/agent}" "${QWEN_CONFIG_DIR:-$HOME/.qwen}" "${TRAE_CONFIG_DIR:-$HOME/.trae}" "${WINDSURF_CONFIG_DIR:-$HOME/.codeium/windsurf}" "${ZCODE_CONFIG_DIR:-$HOME/.zcode}" "${GROK_AGENTS_HOME:-$HOME/.agents}"; for _h; do _gsd_at "$_h/gsd-core/bin/${_GSD_SHIM_NAME}" && return 0; done; return 1; }; if _gsd_at "${_GSD_RUNTIME_ROOT}/gsd-core/bin/${_GSD_SHIM_NAME}" "${_GSD_RUNTIME_ROOT}/.claude/gsd-core/bin/${_GSD_SHIM_NAME}" "${_GSD_RUNTIME_ROOT}/.codex/gsd-core/bin/${_GSD_SHIM_NAME}"; then gsd_run() { node "$GSD_TOOLS" "$@"; }; elif _gsd_homes; then gsd_run() { node "$GSD_TOOLS" "$@"; }; elif unset -f gsd_run; _G="$(command -v gsd_run)"; [ -n "$_G" ] && _gsd_id_ok "$_G"; then GSD_TOOLS="$_G"; gsd_run() { "$GSD_TOOLS" "$@"; }; else echo "ERROR: gsd-tools.cjs not found at $GSD_TOOLS and no identity-proving gsd_run is on PATH. Run: npx -y @opengsd/gsd-core@latest --claude --local" >&2; exit 1; fi; GSD_IDENTITY_STATUS=unverified; _gsd_id_ok gsd_run && GSD_IDENTITY_STATUS=ok; export GSD_IDENTITY_STATUS; [ "$GSD_IDENTITY_STATUS" = ok ] || echo "WARNING: \"$GSD_TOOLS\" did not prove it is @opengsd/gsd-core - it is either a different package or an @opengsd/gsd-core older than the runtime-identity verb. See docs/how-to/diagnose-a-foreign-gsd-tools.md" >&2; if [ -n "${CLAUDE_ENV_FILE:-}" ] && [ -n "${GSD_TOOLS:-}" ]; then printf "export PATH='%s':\"\$PATH\"\n" "${GSD_TOOLS%/*}" >> "$CLAUDE_ENV_FILE" 2>/dev/null || true; fi
gsd_run query state.record-session \
  --stopped-at "Forensic investigation complete" \
  --resume-file ".planning/forensics/report-{timestamp}.md"
```
