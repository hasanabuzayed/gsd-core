<purpose>
This workflow configures third-party integrations for GSD:

- Search API keys (Brave / Firecrawl / Exa)
- Code-review CLI routing (`review.models.<cli>`)
- Agent-skill injection (`agent_skills.<agent-type>`)

The workflow uses `gsd-tools` to write to `.planning/config.json`.
It preserves unrelated keys and never overwrites them.

This command is separate from `/gsd:settings` (workflow toggles) and any `/gsd-settings-advanced` tuning surface.
API keys and cross-tool routing relate to *connectivity*, not workflow or tuning settings.
</purpose>

<security>
**API keys are secrets.** The workflow writes them as plaintext to `.planning/config.json`.
Secrets are on disk at that path. File permissions are the security boundary.
The UI must never show, echo, or log the plaintext value. The workflow follows these rules:

- **Masking convention: `****<last-4>`** (for example, `sk-abc123def456` → `****f456`).
  The UI shows strings shorter than 8 characters as `****` with no tail.
  This prevents disclosure of a meaningful fraction of the bytes of a short secret.
  The UI shows unset values as `(unset)`.
- **AskUserQuestion descriptions, confirmation tables, and log lines never echo plaintext.**
  The workflow does not write plaintext to any file under `.planning/` other than `config.json` itself.
- **`config-set` masks output** for keys in the secret set
  (`brave_search`, `firecrawl`, `exa_search`). See `gsd-core/bin/lib/secrets.cjs`.
- **Agent-type and CLI slug validation.** Before any write, make sure that `agent_skills.<agent-type>` slug inputs match `^[a-zA-Z0-9_-]+$`.
  Reject inputs that contain path separators (`/`, `\`, `..`), whitespace, or shell metacharacters.
  This prevents skill-injection attacks on that open namespace (dynamic key pattern).

  For `review.models.<cli>`, do not examine slug shape. Membership in the closed, registry-derived settable set is the gate.
  See the review-models section below. This gate includes slug shape.
  Slug shape alone never makes a `review.models.*` key writable.
</security>

<required_reading>
Before you start, read all files that the invoking prompt's execution_context references.
</required_reading>

<process>

<step name="ensure_and_load_config">
Make sure that the config exists. Resolve the active config path (flat vs workstream, #2282):

```bash
_GSD_SHIM_NAME="gsd-tools.cjs"; _GSD_RUNTIME_ROOT="${RUNTIME_DIR:-$(git rev-parse --show-toplevel 2>/dev/null || pwd)}"; GSD_TOOLS="${_GSD_RUNTIME_ROOT}/gsd-core/bin/${_GSD_SHIM_NAME}"; _gsd_at() { for _p; do if [ -f "$_p" ]; then GSD_TOOLS="$_p"; return 0; fi; done; return 1; }; _gsd_id_ok() { case "$("$1" runtime-identity --raw 2>/dev/null || true)" in '{"packageName":"@opengsd/gsd-core"'*'}') return 0;; *) return 1;; esac; }; _gsd_homes() { set -- "${CLAUDE_CONFIG_DIR:-$HOME/.claude}" "${ANTIGRAVITY_CONFIG_DIR:-$HOME/.gemini/antigravity}" "$HOME/.gemini/antigravity-ide" "$HOME/.gemini/antigravity-cli" "${AUGMENT_CONFIG_DIR:-$HOME/.augment}" "${CLINE_CONFIG_DIR:-$HOME/.cline}" "${CODEBUDDY_CONFIG_DIR:-$HOME/.codebuddy}" "${CODEX_HOME:-$HOME/.codex}" "${COPILOT_CONFIG_DIR:-${COPILOT_HOME:-$HOME/.copilot}}" "${CURSOR_CONFIG_DIR:-$HOME/.cursor}" "${HERMES_HOME:-$HOME/.hermes}" "${KILO_CONFIG_DIR:-${XDG_CONFIG_HOME:-$HOME/.config}/kilo}" "${KIMI_CONFIG_DIR:-$HOME/.config/agents}" "$HOME/.agents" "${KIMI_CODE_HOME:-$HOME/.kimi-code}" "${OPENCODE_CONFIG_DIR:-${XDG_CONFIG_HOME:-$HOME/.config}/opencode}" "${PI_CODING_AGENT_DIR:-$HOME/.pi/agent}" "${QWEN_CONFIG_DIR:-$HOME/.qwen}" "${TRAE_CONFIG_DIR:-$HOME/.trae}" "${WINDSURF_CONFIG_DIR:-$HOME/.codeium/windsurf}" "${ZCODE_CONFIG_DIR:-$HOME/.zcode}" "${GROK_AGENTS_HOME:-$HOME/.agents}"; for _h; do _gsd_at "$_h/gsd-core/bin/${_GSD_SHIM_NAME}" && return 0; done; return 1; }; if _gsd_at "${_GSD_RUNTIME_ROOT}/gsd-core/bin/${_GSD_SHIM_NAME}" "${_GSD_RUNTIME_ROOT}/.claude/gsd-core/bin/${_GSD_SHIM_NAME}" "${_GSD_RUNTIME_ROOT}/.codex/gsd-core/bin/${_GSD_SHIM_NAME}"; then gsd_run() { node "$GSD_TOOLS" "$@"; }; elif _gsd_homes; then gsd_run() { node "$GSD_TOOLS" "$@"; }; elif unset -f gsd_run; _G="$(command -v gsd_run)"; [ -n "$_G" ] && _gsd_id_ok "$_G"; then GSD_TOOLS="$_G"; gsd_run() { "$GSD_TOOLS" "$@"; }; else echo "ERROR: gsd-tools.cjs not found at $GSD_TOOLS and no identity-proving gsd_run is on PATH. Run: npx -y @opengsd/gsd-core@latest --claude --local" >&2; exit 1; fi; GSD_IDENTITY_STATUS=unverified; _gsd_id_ok gsd_run && GSD_IDENTITY_STATUS=ok; export GSD_IDENTITY_STATUS; [ "$GSD_IDENTITY_STATUS" = ok ] || echo "WARNING: \"$GSD_TOOLS\" did not prove it is @opengsd/gsd-core - it is either a different package or an @opengsd/gsd-core older than the runtime-identity verb. See docs/how-to/diagnose-a-foreign-gsd-tools.md" >&2; if [ -n "${CLAUDE_ENV_FILE:-}" ] && [ -n "${GSD_TOOLS:-}" ]; then printf "export PATH='%s':\"\$PATH\"\n" "${GSD_TOOLS%/*}" >> "$CLAUDE_ENV_FILE" 2>/dev/null || true; fi
RESPONSE_LANGUAGE=$(gsd_run query config-get response_language --raw --default "" 2>/dev/null || echo "")
gsd_run query config-ensure-section
if [[ -z "${GSD_CONFIG_PATH:-}" ]]; then
  if [[ -f .planning/active-workstream ]]; then
    WS=$(tr -d '\n\r' < .planning/active-workstream)
    GSD_CONFIG_PATH=".planning/workstreams/${WS}/config.json"
  else
    GSD_CONFIG_PATH=".planning/config.json"
  fi
fi
```

**If `response_language` is set:** All user-facing output of this workflow MUST be in `{response_language}`.
This includes narration between tool calls, status updates, progress notes, findings, questions, prompts, and explanations.
Technical terms, code, file paths, and subagent prompts stay in English. Translate only user-facing output.

Store `$GSD_CONFIG_PATH`. Use it for every next read/write.
</step>

<step name="read_current">
Read the current config. Compute a masked view to show the user.
For each integration field, compute one of:

- `(unset)` — the field is null / missing
- `****<last-4>` — the secret field has a value (never show plaintext)
- `<value>` — non-secret routing/skill string, shown as-is

```bash
BRAVE=$(gsd_run query config-get brave_search --raw --default null)
FIRECRAWL=$(gsd_run query config-get firecrawl --raw --default null)
EXA=$(gsd_run query config-get exa_search --raw --default null)
SEARCH_GITIGNORED=$(gsd_run query config-get search_gitignored --raw --default false)
```

For each secret key (`brave_search`, `firecrawl`, `exa_search`), show `****<last-4>` when set, never the raw string.
Never echo the plaintext to stdout, stderr, or any log.
</step>

<step name="section_1_search_integrations">

**Text mode (`workflow.text_mode: true` or `--text` flag):** Set `TEXT_MODE=true`.
Replace every `AskUserQuestion` call with a plain-text numbered list. Non-Claude runtimes need this mode.

Ask the user what they want to do for each search API key.
For keys that are already set, show `**** already set`. Offer Leave / Replace / Clear.
For unset keys, offer Skip / Set.

```text
AskUserQuestion([
  {
    question: "Brave Search API key — used for web research during plan/discuss phases",
    header: "Brave",
    multiSelect: false,
    options: [
      // When already set:
      { label: "Leave (**** already set)", description: "Keep current value" },
      { label: "Replace", description: "Enter a new API key" },
      { label: "Clear", description: "Remove the stored key" }
      // When unset, use the two-option shape: Skip / Set.
    ]
  },
  {
    question: "Firecrawl API key — used for deep-crawl scraping",
    header: "Firecrawl",
    multiSelect: false,
    options: [ /* same Leave/Replace/Clear or Skip/Set */ ]
  },
  {
    question: "Exa Search API key — used for semantic search",
    header: "Exa",
    multiSelect: false,
    options: [ /* same Leave/Replace/Clear or Skip/Set */ ]
  },
  {
    question: "Include gitignored files in local code searches?",
    header: "Gitignored",
    multiSelect: false,
    options: [
      { label: "No (Recommended)", description: "Respect .gitignore. Safer — excludes secrets, node_modules, build artifacts." },
      { label: "Yes", description: "Include gitignored files. Useful when secrets/artifacts genuinely contain searchable intent." }
    ]
  }
])
```

For each "Set" or "Replace", ask for the key value in a text-input prompt.
**You must not echo the answer** in next question descriptions or confirmation text. Write the value via:

```bash
gsd_run query config-set brave_search "<value>"     # masked in output
gsd_run query config-set firecrawl "<value>"        # masked in output
gsd_run query config-set exa_search "<value>"       # masked in output
gsd_run query config-set search_gitignored true|false
```

For "Clear", write `null`:

```bash
gsd_run query config-set brave_search null
```
</step>

<step name="section_2_review_models">

`review.models.<cli>` is a closed, registry-derived map. It tells the review workflow which model id a reviewer lane uses.
It is not an open namespace. A `review.models.<cli>` key is settable only when that lane's capability declares a `modelConfigKey`.

`config-set` accepts exactly those keys (federated from the capability registry).
No dynamic-key regex governs this namespace. Any other slug fails with `Unknown config key`.

Settable keys (the shipped registry's model-bearing lanes):

`review.models.agy` (Antigravity), `review.models.claude`, `review.models.codex`,
`review.models.cursor`, `review.models.kimi-code`,
`review.models.llama_cpp`, `review.models.lm_studio`, `review.models.ollama`,
`review.models.opencode`.

Reviewer lanes `qwen` and `coderabbit` declare no `modelConfigKey`. There is nothing to configure for them here.
Whether they should have a per-lane model key is a separate question, out of scope for this workflow.
If the user asks for one of those, say exactly that and skip.

```text
AskUserQuestion([
  {
    question: "Review model CLI mapping — what next?",
    header: "Review",
    multiSelect: false,
    options: [
      { label: "Configure CLI", description: "Pick a reviewer lane and set/clear its model id" },
      { label: "Done", description: "Finish this section" }
    ]
  }
])
```

If the user selects "Configure CLI", ask:

```text
AskUserQuestion([
  {
    question: "Which reviewer lane do you want to configure? (Common lanes below; any settable lane from the list above works — or type its slug)",
    header: "CLI",
    multiSelect: false,
    options: [
      { label: "Claude", description: "review.models.claude — defaults to session model when unset" },
      { label: "Codex", description: "review.models.codex — bare model id injected into --model, e.g. 'gpt-5'" },
      { label: "OpenCode", description: "review.models.opencode — bare model id injected into --model, e.g. 'claude-sonnet-4'" }
    ]
  }
])
```

For a slug received as free text, make sure that it is in the settable set above.
If it is not one of the settable keys, print:

```text
Rejected: review.models.<slug> is not settable — only the reviewer lanes whose
keys are enumerated above can be configured here. (qwen and coderabbit have no
per-lane model key.)
```

Then prompt again.

For the selected lane, show the current value (or `(unset)`). Offer Leave / Replace / Clear.
Then prompt for the model id string with a text input. Write via:

```bash
gsd_run query config-set review.models.<cli> "<model id>"
```

After each update, return to the "Review model CLI mapping — what next?" question.
Loop until the user selects "Done".
</step>

<step name="section_3_agent_skills">

`agent_skills.<agent-type>` injects extra skill names into an agent's spawn frontmatter.
The user can extend the slug. Make sure that the free-text input matches `^[a-zA-Z0-9_-]+$`.
Reject inputs with path separators, spaces, or shell metacharacters.

```text
AskUserQuestion([
  {
    question: "Agent skills mapping — what next?",
    header: "Agent Skills",
    multiSelect: false,
    options: [
      { label: "Configure agent", description: "Pick an agent type and set/clear skills" },
      { label: "Done", description: "Finish this section" }
    ]
  }
])
```

If the user selects "Configure agent", ask:

```text
AskUserQuestion([
  {
    question: "Configure agent_skills for which agent type?",
    header: "Agent Type",
    multiSelect: false,
    options: [
      { label: "gsd-executor", description: "Skills injected when spawning executor agents" },
      { label: "gsd-planner", description: "Skills injected when spawning planner agents" },
      { label: "gsd-verifier", description: "Skills injected when spawning verifier agents" },
      { label: "Custom…", description: "Enter a custom agent-type slug" }
    ]
  }
])
```

For "Custom…", prompt for a slug. Make sure that it matches `^[a-zA-Z0-9_-]+$`.
If it fails validation, print:

```text
Rejected: agent-type '<slug>' must match [a-zA-Z0-9_-]+ (no path separators,
spaces, or shell metacharacters).
```

Then prompt again.

For a selected slug, prompt for the skill list with a text input. A comma-separated reply is fine.
If a current value exists, show it. Offer Leave / Replace / Clear.

Before you write, split the reply. The resolver never splits strings.
A comma-joined string would be stored as ONE skill path that silently fails resolution at spawn time.
See #3651 — `gsd-core/references/planning-config.md`:
"Paths cannot be comma-joined into one string. Each path must be its own
array element".

Split on commas. Trim each entry. Remove empty entries.
Reject any entry that contains a quote character (`'` or `"`). You cannot write it safely through the single-quoted form.
Write the JSON array form:

```bash
gsd_run query config-set agent_skills.<slug> '["skills/alpha","skills/beta"]'
```

You may write a single skill as one bare string or a one-element array. Both resolve identically.

After each update, return to the "Agent skills mapping — what next?" question.
Loop until "Done".
</step>

<step name="confirm">
Show the masked confirmation table. **No plaintext API keys appear in this output under any circumstance.**

```text
### GSD ► INTEGRATIONS UPDATED

Search Integrations
| Field              | Value             |
|--------------------|-------------------|
| brave_search       | ****<last-4>      |  (or "(unset)")
| firecrawl          | ****<last-4>      |
| exa_search         | ****<last-4>      |
| search_gitignored  | true | false      |

Code Review CLI Routing
| Lane        | Model id                             |
|-------------|--------------------------------------|
| <lane>      | <value or (unset)>                   |
| ...         | ... one row per lane the user set    |

Agent Skills Injection
| Agent Type       | Skills                    |
|------------------|---------------------------|
| <slug>           | <skill-a, skill-b>        |
| ...              | ...                       |

Notes:
- API keys are stored plaintext in .planning/config.json. The confirmation
  table above never displays plaintext — keys appear as ****<last-4>.
- Plaintext is not echoed back by this workflow, not written to any log,
  and not displayed in error messages.

Quick commands:
- /gsd:settings — workflow toggles and model profile
- /gsd-set-profile <profile> — switch model profile
```
</step>

</process>

<success_criteria>
- [ ] Read the current config from `$GSD_CONFIG_PATH`
- [ ] Show the user three sections: Search Integrations, Review CLI Routing, Agent Skills Injection
- [ ] Write API keys as plaintext only to `config.json`. Never echo, log, or show them
- [ ] Use `****<last-4>` for set keys and `(unset)` for null in the masked confirmation table
- [ ] Before writing, make sure that `agent_skills.<agent-type>` slugs match `[a-zA-Z0-9_-]+`
- [ ] Accept `review.models.<cli>` slugs only from the registry-derived settable set
- [ ] Write skill lists as JSON arrays (never comma-joined strings)
- [ ] Preserve all keys outside the three sections this workflow owns when merging the config
</success_criteria>
