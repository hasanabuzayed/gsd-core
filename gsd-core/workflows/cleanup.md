<purpose>

Archive phase directories from completed milestones into `.planning/milestones/v{X.Y}-phases/`. Find the phases for each completed milestone. Show a dry-run summary. Move the directories after confirmation.

If `.planning/quick/` is not empty, also offer to archive it retroactively (#2142).

</purpose>

<required_reading>

1. `.planning/MILESTONES.md`
2. `.planning/milestones/` directory listing
3. `.planning/phases/` directory listing
4. `.planning/quick/` directory listing

</required_reading>

<process>
```bash
_GSD_SHIM_NAME="gsd-tools.cjs"; _GSD_RUNTIME_ROOT="${RUNTIME_DIR:-$(git rev-parse --show-toplevel 2>/dev/null || pwd)}"; GSD_TOOLS="${_GSD_RUNTIME_ROOT}/gsd-core/bin/${_GSD_SHIM_NAME}"; _gsd_at() { for _p; do if [ -f "$_p" ]; then GSD_TOOLS="$_p"; return 0; fi; done; return 1; }; _gsd_id_ok() { case "$("$1" runtime-identity --raw 2>/dev/null || true)" in '{"packageName":"@opengsd/gsd-core"'*'}') return 0;; *) return 1;; esac; }; _gsd_homes() { set -- "${CLAUDE_CONFIG_DIR:-$HOME/.claude}" "${ANTIGRAVITY_CONFIG_DIR:-$HOME/.gemini/antigravity}" "$HOME/.gemini/antigravity-ide" "$HOME/.gemini/antigravity-cli" "${AUGMENT_CONFIG_DIR:-$HOME/.augment}" "${CLINE_CONFIG_DIR:-$HOME/.cline}" "${CODEBUDDY_CONFIG_DIR:-$HOME/.codebuddy}" "${CODEX_HOME:-$HOME/.codex}" "${COPILOT_CONFIG_DIR:-${COPILOT_HOME:-$HOME/.copilot}}" "${CURSOR_CONFIG_DIR:-$HOME/.cursor}" "${HERMES_HOME:-$HOME/.hermes}" "${KILO_CONFIG_DIR:-${XDG_CONFIG_HOME:-$HOME/.config}/kilo}" "${KIMI_CONFIG_DIR:-$HOME/.config/agents}" "$HOME/.agents" "${KIMI_CODE_HOME:-$HOME/.kimi-code}" "${OPENCODE_CONFIG_DIR:-${XDG_CONFIG_HOME:-$HOME/.config}/opencode}" "${PI_CODING_AGENT_DIR:-$HOME/.pi/agent}" "${QWEN_CONFIG_DIR:-$HOME/.qwen}" "${TRAE_CONFIG_DIR:-$HOME/.trae}" "${WINDSURF_CONFIG_DIR:-$HOME/.codeium/windsurf}" "${ZCODE_CONFIG_DIR:-$HOME/.zcode}" "${GROK_AGENTS_HOME:-$HOME/.agents}"; for _h; do _gsd_at "$_h/gsd-core/bin/${_GSD_SHIM_NAME}" && return 0; done; return 1; }; if _gsd_at "${_GSD_RUNTIME_ROOT}/gsd-core/bin/${_GSD_SHIM_NAME}" "${_GSD_RUNTIME_ROOT}/.claude/gsd-core/bin/${_GSD_SHIM_NAME}" "${_GSD_RUNTIME_ROOT}/.codex/gsd-core/bin/${_GSD_SHIM_NAME}"; then gsd_run() { node "$GSD_TOOLS" "$@"; }; elif _gsd_homes; then gsd_run() { node "$GSD_TOOLS" "$@"; }; elif unset -f gsd_run; _G="$(command -v gsd_run)"; [ -n "$_G" ] && _gsd_id_ok "$_G"; then GSD_TOOLS="$_G"; gsd_run() { "$GSD_TOOLS" "$@"; }; else echo "ERROR: gsd-tools.cjs not found at $GSD_TOOLS and no identity-proving gsd_run is on PATH. Run: npx -y @opengsd/gsd-core@latest --claude --local" >&2; exit 1; fi; GSD_IDENTITY_STATUS=unverified; _gsd_id_ok gsd_run && GSD_IDENTITY_STATUS=ok; export GSD_IDENTITY_STATUS; [ "$GSD_IDENTITY_STATUS" = ok ] || echo "WARNING: \"$GSD_TOOLS\" did not prove it is @opengsd/gsd-core - it is either a different package or an @opengsd/gsd-core older than the runtime-identity verb. See docs/how-to/diagnose-a-foreign-gsd-tools.md" >&2; if [ -n "${CLAUDE_ENV_FILE:-}" ] && [ -n "${GSD_TOOLS:-}" ]; then printf "export PATH='%s':\"\$PATH\"\n" "${GSD_TOOLS%/*}" >> "$CLAUDE_ENV_FILE" 2>/dev/null || true; fi
RESPONSE_LANGUAGE=$(gsd_run query config-get response_language --raw --default "" 2>/dev/null || echo "")
```

**If `response_language` is set:** You MUST show all user-facing output in `{response_language}`. This includes narration between tool calls, status updates, progress notes, and findings. It also includes questions, prompts, and explanations.

Keep technical terms, code, file paths, and subagent prompts in English. Translate only user-facing output.


<step name="identify_completed_milestones">

Read `.planning/MILESTONES.md` to find completed milestones and their versions.

```bash
cat .planning/MILESTONES.md
```

Extract each milestone version (for example, v1.0, v1.1, v2.0).

Find the milestone archive directories that already exist:

```bash
ls -d .planning/milestones/v*-phases 2>/dev/null || true
```

Filter to milestones that do NOT already have a `-phases` archive directory.

If all milestones already have phase archives:

```
All completed milestones already have phase directories archived. Nothing to clean up.
```

Stop here.

</step>

<step name="determine_phase_membership">

For each completed milestone without a `-phases` archive, read the archived ROADMAP snapshot. Find the phases that belong to the milestone:

```bash
cat .planning/milestones/v{X.Y}-ROADMAP.md
```

Extract phase numbers and names from the archived roadmap (for example, Phase 1: Foundation, Phase 2: Auth).

Find which of those phase directories still exist in `.planning/phases/`:

```bash
ls -d .planning/phases/*/ 2>/dev/null || true
```

Match phase directories to milestone membership. Only include directories that still exist in `.planning/phases/`.

</step>

<step name="identify_quick_tasks">

Find whether `.planning/quick/` has directories to archive retroactively (#2142):

```bash
ls -d .planning/quick/*/ 2>/dev/null || true
```

**If you find no directories:** `.planning/quick/` is empty or absent. Say nothing about quick-task archival. Do not offer the step. Go to `show_dry_run` without a quick-task summary or prompt.

**If you find at least one directory:** Find the target milestone. Each completed milestone has an archived ROADMAP snapshot that identifies its phase directories. Quick tasks have **no on-disk provenance**. There is no way to find the milestone for a quick-task directory.

The target is the single most recent completed milestone without a `-quick` archive directory. The entries in `.planning/MILESTONES.md` are newest-first. You already read that file in `identify_completed_milestones`:

```bash
ls -d .planning/milestones/v*-quick 2>/dev/null || true
```

Read the entries in `.planning/MILESTONES.md` newest-first. Pick the first version without a matching `v{version}-quick` directory in the list above.

If every completed milestone already has a `-quick` archive, there is no valid target. If `.planning/MILESTONES.md` has no entries, there is no valid target. If there is no valid target, say so. Skip the quick-task step and do not prompt.

</step>

<step name="show_dry_run">

Show a dry-run summary for each milestone:

```
## Cleanup Summary

### v{X.Y} — {Milestone Name}
These phase directories will be archived:
- 01-foundation/
- 02-auth/
- 03-core-features/

Destination: .planning/milestones/v{X.Y}-phases/

### v{X.Z} — {Milestone Name}
These phase directories will be archived:
- 04-security/
- 05-hardening/

Destination: .planning/milestones/v{X.Z}-phases/
```

**If a quick-task target milestone was determined in `identify_quick_tasks`**, add:

```
### Quick tasks — bucket-all into v{X.Y}
{N} directories under .planning/quick/ will ALL be archived into this ONE milestone
(v{X.Y} — {Milestone Name}), regardless of when each was actually completed.
Quick tasks carry no on-disk record of which milestone they belong to, so this is
a bucket-all, not a per-milestone split — unlike the phase archival above, which
is derived per-milestone from each archived ROADMAP snapshot.

Destination: .planning/milestones/v{X.Y}-quick/
```

**Stale local branches (upstream gone):**

First, update remote-tracking refs. This makes the candidate list match the execution list exactly:

```bash
git fetch --prune 2>/dev/null || true
```

Then list candidates. Exclude protected branch names even if their upstream is gone:

```bash
git branch -vv | awk '/: gone\]/ { if ($1 !~ /^\*$|^main$|^next$|^trunk$|^develop$/) print $1 }'
```

Show each branch name. If none, show:

```
No stale local branches detected.
```

If no phase directories remain to archive (all already moved or deleted) AND no stale branches exist AND no quick-task target milestone was determined:

```
No phase directories found to archive. Phases may have been removed or archived previously.
No stale local branches detected either.
No quick tasks to archive either.
```

Stop here.


**Text mode (`workflow.text_mode: true` in config or `--text` flag):** If `--text` is present in `$ARGUMENTS`, set `TEXT_MODE=true`. If `text_mode` from init JSON is `true`, set `TEXT_MODE=true`.

When TEXT_MODE is active, replace every `AskUserQuestion` call with a plain-text numbered list. Ask the user to type their choice number. Non-Claude runtimes need this because `AskUserQuestion` is not available. Examples include OpenAI Codex and Antigravity.
AskUserQuestion: "Proceed with archiving and pruning?" with options: "Yes — archive phases and prune stale branches" | "Cancel"

If "Cancel": Stop.

**If `identify_quick_tasks` found a quick-task target milestone**, ask a separate, explicit question. This is a distinct, bucket-all action and must not be silently included in the "Yes" above:

AskUserQuestion: "Archive ALL {N} quick-task directories into v{X.Y} — {Milestone Name}? This buckets every remaining quick task into this ONE milestone. There is no way to split them per-milestone." with options: "Yes — archive quick tasks into v{X.Y}" | "Skip"

If "Skip": Do not run `archive_quick_tasks`. If there were no phase directories to archive, go to `report`. Otherwise, go to `archive_phases`. Omit quick-task archival.

</step>

<step name="archive_phases">

For each milestone, move phase directories:

```bash
mkdir -p .planning/milestones/v{X.Y}-phases
```

For each phase directory belonging to this milestone:

```bash
mv .planning/phases/{dir} .planning/milestones/v{X.Y}-phases/
```

Repeat for all milestones in the cleanup set.

</step>

<step name="archive_quick_tasks">

Only run this step when the user confirmed the "Yes — archive quick tasks into v{X.Y}" option in `show_dry_run`.

Use the narrow `milestone.archive-quick` command (#2142 escalation), not `milestone.complete --archive-quick`. Cleanup typically runs against milestones that are ALREADY completed. `milestone.complete` does the full close-out. It archives ROADMAP/REQUIREMENTS and writes a MILESTONES.md entry.

If you run it again for a completed milestone, it would overwrite that milestone's archived ROADMAP/REQUIREMENTS snapshot. This cleanup depends on that snapshot. It would also duplicate the milestone's MILESTONES.md entry.

`milestone.archive-quick` uses the same underlying helper as `milestone.complete --archive-quick` for the move/README-index/table-reset logic. It does not do the other actions.

```bash
gsd_run query milestone.archive-quick "v{X.Y}"
```

The command moves every directory under `.planning/quick/` into `.planning/milestones/v{X.Y}-quick/`. It writes or rewrites that directory's `README.md` index. It clears the `### Quick Tasks Completed` table rows in STATE.md.

The move/index/reset behavior is identical to the `--archive-quick` flag in the `archive_milestone` step of `complete-milestone.md`. The command does not change ROADMAP.md, REQUIREMENTS.md, MILESTONES.md, or milestone-completion guards. Extract `archived` from the result to make sure.

</step>

<step name="prune_local_branches">

After phase archival, prune local branches whose upstream was deleted. Use the same filter as the dry-run. Make the execution list match exactly what the user confirmed:

```bash
git branch -vv | awk '/: gone\]/ { if ($1 !~ /^\*$|^main$|^next$|^trunk$|^develop$/) print $1 }' | xargs -r git branch -D
```

Notes:
- `git fetch --prune` already ran in `show_dry_run`. The tracking refs are current. This step lists branches from the same state that the user confirmed.
- `!~ /^\*$/` skips the currently checked-out branch. In `git branch -vv` output, this branch has the prefix `* `, so `$1` yields `*`.
- `!~ /^main$|^next$|^trunk$|^develop$/` excludes protected branch names even if their upstream is gone. This matches the dry-run exclusion exactly.
- When no stale branches exist, `xargs -r` prevents `git branch -D` from running with no arguments.

</step>

<step name="commit">

Commit the changes:

```bash
gsd_run query commit "chore: archive phase directories from completed milestones" --files .planning/milestones/ .planning/STATE.md --files-removed .planning/phases/ .planning/quick/
```

Put `.planning/phases/` and `.planning/quick/` under `--files-removed`, not `--files` (#4208). A `--files` directory entry stages everything under it. It would also commit any in-flight phase or quick-task file that a concurrent session wrote there.

`--files-removed` stages only tracked files under those directories that the archival `mv` moved away. It leaves everything still present untouched.

</step>

<step name="report">

```
Archived:
{For each milestone}
- v{X.Y}: {N} phase directories → .planning/milestones/v{X.Y}-phases/
{If quick-task archival ran}
- v{X.Y}: {N} quick-task directories → .planning/milestones/v{X.Y}-quick/ (bucket-all — see known limit)

Pruned: {N} local branches whose upstream is gone.

.planning/phases/ cleaned up.
```

</step>

</process>

<success_criteria>

- [ ] Find all completed milestones without existing phase archives
- [ ] Find phase membership from archived ROADMAP snapshots
- [ ] Show a dry-run summary and get user confirmation for archival and pruning
- [ ] Move phase directories to `.planning/milestones/v{X.Y}-phases/`
- [ ] Prune stale local branches whose upstream is gone
- [ ] Examine `.planning/quick/`. Offer quick-task archival only when it is not empty
- [ ] When offered and confirmed, archive ALL remaining quick-task directories into the single named target milestone with `milestone.archive-quick` (bucket-all, not per-milestone)
- [ ] Commit changes

</success_criteria>
