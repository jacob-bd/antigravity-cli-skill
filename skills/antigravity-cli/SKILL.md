---
name: antigravity-cli
description: "Use when an AI agent or user invokes Google's Antigravity CLI (agy), delegates plan reviews, code reviews, second-opinion audits, or coding tasks to agy or Gemini, checks agy quota or models, configures agy MCP servers, plugins, hooks, or custom agents, migrates from Gemini CLI, or automates agy in scripts, CI/CD, ACP editors, run_workflow, or swarms."
metadata:
  version: "1.5.12"
  author: jacob-bd
---
<!-- Version: 1.5.12 -->

# Antigravity CLI (`agy`) Skill

Targets `agy` **v1.5.12** (`~/.local/bin/agy`; override: `ANTIGRAVITY_CLI_ALIAS`). Shares the Go harness with Antigravity IDE (embedded `ripgrep`, zero npm deps). Run `agy --version` to verify (auto-updates).

## Host-Agent Execution Contract (Codex, Claude Code, OpenCode)

### 1. Resolve Binary & Host Sandbox
```bash
AGY=$(command -v agy || echo "$HOME/.local/bin/agy")
```
`agy` needs write access to `~/.gemini/antigravity-cli/`, outbound API network, and auth/keychain access. Escalate `agy` calls out of any outer host sandbox (Codex Seatbelt / Landlock); never nest `agy --sandbox`.

### 2. Pre-Flight Quota Check (0 Turns, 0 Tokens)
Skip `/usage` under `GEMINI_API_KEY` (API-key auth has no account quota; `/usage`, `/quota`, `/credits` exit `2`). Never probe `"/credits"` unconditionally (exits `1` without AI credits).
```bash
"$AGY" -p "/usage" --output-format json
```
Read `.command.data.groups[].buckets[]` (`id`, `name`, `remaining_fraction`, `reset_time`). `remaining_fraction` is share **left** (`1` = 100% left, `0` = used up). Gemini Pro, Gemini Flash, and Claude/GPT use separate pools (`groups[].description`).

| `remaining_fraction` | Action & Exact Output |
| :--- | :--- |
| `>= 0.20` | Omit quota from output and proceed. |
| `> 0` and `< 0.20` | Warn: `<model>: <round(remaining_fraction*100)>% left`, adding `(resets <reset_time>)` only if `reset_time != "1970-01-01T00:00:00Z"`. |
| `== 0` | Report `<model>: 0% left`; stop or switch `--model` to another pool (`"$AGY" models`). |

`1970-01-01T00:00:00Z` means no active reset window (never print as a date). Never call non-zero quota "full" or "exhausted".

### 3. Pick Model & Effort
Run `"$AGY" models` for live `<model>-<effort>` slugs on *this* account (rejects `--output-format`):
- **Gemini 4 Neon**: up to `xhigh` on Google sign-in and `GEMINI_API_KEY`.
- **Gemini 4 Carbon / Hydrogen / Argon**: `high` max on Google sign-in; Hydrogen offers `xhigh` with `GEMINI_API_KEY`, `AGY_ADC_AUTH`, or enterprise gateway.
- **Orchestrator & Claude**: `gemini-pro-orchestrator` where enabled (Enterprise / `AGY_ADC_AUTH` / Google sign-in); `max` effort is Claude-only. Unavailable `--effort` fails fast (exit `1`, 0 turns) with `(available: ...)`.

### 4. Select Flags & Run a Verified Recipe
Always pass `--add-dir .` (`cd` alone does not scope workspace) and set `--mode` or `--settings` explicitly (unconfigured sessions default to `auto` where available). Ephemeral `--settings '<json>'` overrides **only** specified keys — bare `{"agentMode":""}` inherits saved `"toolPermission":"always-proceed"` and allows writes, while `"toolPermission":"strict"` blocks even `read_file` in `-p`.

| Task | Required Mode / `--settings` | Notes |
| :--- | :--- | :--- |
| Pure read-only critique | `--settings '{"agentMode":"","toolPermission":"request-review","permissions":{"allow":[]}}'` | Allows workspace reads, blocks edits/shell, clears saved `global` allowlist. Not `--mode plan` (drafts `plan.md` & pauses). |
| Read-only Git diff review | `--settings '{"agentMode":"","toolPermission":"request-review","permissions":{"allow":[...]}}'` | Honors `permissions.allow`, blocks edits & other cmds. **`cd` into target repo first**. |
| Scoped coding & tests | `--mode accept-edits --settings '{"toolPermission":"always-proceed","artifactReviewPolicy":"always-proceed"}'` | Works across Personal, Pro/Ultra, and Enterprise without touching `settings.json`. |
| Interactive plan drafting | `--mode plan` | Drafts `plan.md`; add `"artifactReviewPolicy":"always-proceed"` to skip approval pause. |
| Bidirectional NDJSON | `agy --input-format stream-json --output-format stream-json` | Omit bare `-p` (or `-p ""`); send `{"event":"user","message":{"content":"..."}}` on stdin. |

```bash
# 1. Pure read-only plan / architecture critique (reads allowed, edits & shell blocked)
"$AGY" --print="Review plan.md for gaps, edge cases, and risks. Max 6 findings ranked by severity, ending with a verdict." \
  --add-dir . --disable-slash-commands \
  --settings '{"agentMode":"","toolPermission":"request-review","permissions":{"allow":[]}}' \
  --output-format json --print-timeout 10m > out.json 2> err.log

# 2. Read-only Git diff / code review (CWD must be inside target repo)
(cd /path/to/repo && "$AGY" --print="Review uncommitted git diff for bugs, races, and security issues. Max 8 findings." \
  --add-dir . --disable-slash-commands \
  --settings '{"agentMode":"","toolPermission":"request-review","permissions":{"allow":["command(git status)","command(git diff)","command(git log)","command(git show)"]}}' \
  --output-format json --print-timeout 8m) > out.json 2> err.log

# 3. Scoped implementation & tests
"$AGY" --print="Fix the timeout bug in src/payment/processor.py and run pytest tests/test_payment.py. Keep changes minimal." \
  --add-dir . --disable-slash-commands --mode accept-edits \
  --settings '{"toolPermission":"always-proceed","artifactReviewPolicy":"always-proceed"}' \
  --output-format json --print-timeout 10m > out.json 2> err.log
```

**Prompt & timeout rules:**
- Pass `--disable-slash-commands` for arbitrary/path-prefixed prompts (omit only for `/<skill>` or `/usage`). Use `--print="..."` (or argv array) for programmatic/dash-prefixed prompts.
- **Never pipe `git diff | agy -p "..."`**: `-p` ignores stdin and drops piped data. Let `agy` read files/git natively or pass files via `--add-dir`.
- Set `--print-timeout 8m`–`15m` (`0s` waits forever, including background tasks) and size host timeouts `+60s` above it (`agy` retries rate limits up to 4m and silent thinking up to 10m).

### 5. `--dangerously-skip-permissions` Guardrail
Pass `--dangerously-skip-permissions` **ONLY** when the user explicitly names that flag. It triggers host approval prompts in Codex/Claude Code and cannot bypass protected-command denylists or Enterprise admin policies. On denial, add only that rule to `--settings '{"permissions":{"allow":[...]}}'` and retry.

### 6. Mandatory Completion & Denial Check
Check exit code and `[ -s out.json ]` **before** parsing JSON (non-zero exits may leave stdout empty). Never treat exit `0` alone as success:
- Inspect `.status`, `.response` (or `.structured_output` with `--json-schema` + `--output-format json`), `.finished_reason` (`print_timeout`, `interrupted`, `error`; omitted when clean), `.denied_actions[]`, and `.conversation_id`.
- On `print_timeout`, recover via `--conversation <id>` ("Stop exploring; summarize collected findings without tools"). On non-empty `denied_actions`, add the needed `permissions.allow` rule (with `"toolPermission":"request-review"`) and retry.

## Core Headless Flags & Exit Codes

| Flag | Purpose |
| :--- | :--- |
| `-p`, `--print=<text>` | Single headless prompt (string arg required; never bare `-p --<flag>`). |
| `--settings <json\|path>` | Session-only `settings.json` overrides (`{...}` = inline JSON, else file). |
| `--mode <mode>` | `accept-edits`, `auto`, or `plan` (never `--mode default`). |
| `--model` / `--effort` | Slug from `agy models`; effort `low`, `medium`, `high`, `xhigh`, `max`. |
| `--add-dir <path>` | Add directory to workspace scope (repeatable; always pass). |
| `--output-format` / `--input-format` | Output: `text`, `json`, `stream-json`. Input: `text`, `stream-json`. |
| `--json-schema <json\|path>` | Structured output (`"type":"object"` root; pair with `json`, read `.structured_output`). |
| `--print-timeout <dur>` | Print deadline (`0s` = unlimited, waits for background tasks). |
| `--disable-slash-commands` | Treat leading `/` prompts as literal text in `-p`. |
| `--conversation <id>` / `-c` | Resume conversation by ID or continue latest in workspace. |
| `--agent <name>` | Custom Markdown or built-in agent (unknown exits `1` in `-p`). |

Run `agy --help` / `agy <subcmd> --help` or see [resources/cheat_sheet.md](resources/cheat_sheet.md) for all flags, subcommands, `--settings` enums, sandbox/auto rules, env vars, and Gemini CLI migration.

- **Exit `0`**: Success, mid-turn `--print-timeout` partial output (check `.finished_reason`), or headless denial (check `.denied_actions`).
- **Exit `1`**: Invalid flags/settings/schema, unavailable `--effort`, unknown `--agent`, `/credits` without credits, `PreInvocation` hook block, or host sandbox blocking `~/.gemini/antigravity-cli/`.
- **Exit `2`**: Flag conflict (`-p` + `-i`, bare `-p --<flag>`, bad `--input-format`/`--print-timeout`), interactive-only `/clear` in `-p`, or `GEMINI_API_KEY` with `/usage`, `/quota`, `/credits`, `--remote-control`, `--remote-agent`.
- **Exit `3`**: Fatal API/model error (`AGY_ERROR: {...}` on stderr) or `--print-timeout` killing active background tasks.

## Delegate to `agy` vs. Keep Locally
- **Delegate to `agy`**: Independent cross-model plan/code/security reviews, large-context or PDF/multimodal analysis, isolated worktree edits, and parallel `run_workflow` fan-out.
- **Keep locally**: Trivial single-file edits, tight host-context loops, or steps needing host-only MCP tools/state.
- Full decision guide & orchestration selector: [resources/delegation_playbook.md](resources/delegation_playbook.md).

## Where to Go Next
- **Deep delegation, `stream-json`, `git worktree` coding, parallel fan-out, Python wrapper**: [resources/delegation_playbook.md](resources/delegation_playbook.md)
- **All CLI flags, subcommands (`acp`, `mcp`, `plugin`), `--settings` schema, sandbox/auto, env vars, Gemini migration**: [resources/cheat_sheet.md](resources/cheat_sheet.md)
- **Custom agents, swarms, `run_workflow`, plugin authoring, hooks, skill metadata**: [resources/agents_and_customization.md](resources/agents_and_customization.md)
- **Interactive TUI, Vim mode, `/swarm board`, `/projects`, `/diff`, `/compact`, `/btw`, OSC 7501**: [resources/tui_reference.md](resources/tui_reference.md)
- **Categorized runnable automation recipes**: [examples/usage_patterns.md](examples/usage_patterns.md)

## Troubleshooting
- **`agy: command not found`**: Invoke `$HOME/.local/bin/agy` or prepend `$HOME/.local/bin` to `PATH`.
- **`open ~/.gemini/antigravity-cli/installation_id: operation not permitted`**: Escalate `agy` out of outer host sandbox and omit `--sandbox`.
- **Read-only run edited files (or denied `read_file`)**: Bare `{"agentMode":""}` inherits saved `always-proceed`, while `"strict"` blocks even `read_file` in `-p`; use `"toolPermission":"request-review"` with `"permissions":{"allow":[]}` (or explicit read-only commands).
- **`command(git ...)` denied when allowlisted**: `cd` into target repo first and use `"toolPermission":"request-review"` (not `"strict"`).
- **Review asks to approve a plan**: Passed `--mode plan`; use `--settings '{"agentMode":"","toolPermission":"request-review","permissions":{"allow":[]}}'`.
- **Exit `2` (`-p took "--..." as its prompt`)**: Omit `-p` (or pass `-p ""`) with `--input-format stream-json`.
- **Exit `2` on `/usage`, `/quota`, `/credits`, `--remote-control`, `--remote-agent`**: Active `GEMINI_API_KEY` disables these; skip `/usage` under API-key auth.
- **Exit `1` (`has no "<level>" effort`)**: Effort unsupported for model/sign-in (`max` is Claude-only); pick a slug from `agy models`.
- **Exit `3`**: Parse `sed -n 's/^AGY_ERROR:[[:space:]]*//p' err.log | jq .`; switch `--model` pool only if `/usage` shows `remaining_fraction == 0`.
