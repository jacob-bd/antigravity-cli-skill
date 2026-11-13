---
name: antigravity-cli
description: "Use when an AI agent or user invokes Google's Antigravity CLI (agy), delegates plan reviews, code reviews, or coding tasks to agy, migrates from Gemini CLI, or automates agy in scripts, CI/CD, ACP editors, or multi-agent swarms."
version: 1.5.7
---

# Antigravity CLI (`agy`) Skill

Targets `agy` **v1.5.7**. Executable is typically `~/.local/bin/agy` (override with `ANTIGRAVITY_CLI_ALIAS`). Shares the Go agent harness with the Antigravity IDE (zero npm dependencies, embedded `ripgrep`, up to 512 tool calls/turn on Gemini models).

## Host-Agent Execution Contract (Codex, Claude Code, OpenCode)

Follow this contract when invoking `agy` from another coding agent so calls run natively without triggering host approval prompts or false-success bugs:

1. **Resolve the binary first**: Non-login agent subshells often omit `~/.local/bin`:
   `AGY=$(command -v agy || echo "$HOME/.local/bin/agy")`
2. **Pre-flight quota check (0 turns, 0 tokens spent)**: Before delegating a task, check remaining allowance for the model you plan to use:
   ```bash
   "$AGY" -p "/usage" --output-format json
   ```
   Inspect `.command.data.groups[].buckets[]` (`id`, `name`, `remaining_fraction`, `reset_time`).
   - If the target model's `remaining_fraction < 0.20` (or `0`), **warn the user** with the remaining percentage and `reset_time` (ISO-8601 UTC) before running heavy work.
   - Gemini Pro, Gemini Flash, and Claude/GPT models have **separate quota pools**. If one pool is low, suggest switching `--model` to a model in a healthier pool (`"$AGY" models` lists available slugs).
   - Do not probe `"/credits"` unconditionally in scripts: accounts without AI credits exit `1`.
3. **Use ephemeral `--settings` instead of `--dangerously-skip-permissions`**:
   `agy` v1.5.7+ accepts `--settings '<json>'` for session-only config overrides (never written to `settings.json`). Unapproved tools in `-p` mode do not hang on stdin; `agy` denies them immediately and reports `.denied_actions[]`.
   - **Plan / architecture critique (read-only)**:
     ```bash
     "$AGY" -p "Review plan.md for gaps, edge cases, and risks. Max 6 findings ranked by severity, end with a verdict." \
       --add-dir . --settings '{"agentMode":""}' --output-format json --print-timeout 10m
     ```
   - **Git diff / code review (read-only)**:
     ```bash
     "$AGY" -p "Review uncommitted git diff for bugs, races, and security issues. Max 8 findings." \
       --add-dir . \
       --settings '{"agentMode":"","permissions":{"allow":["command(git status)","command(git diff)","command(git log)","command(git show)"]}}' \
       --output-format json --print-timeout 8m
     ```
   - **Scoped implementation & tests (write + run)**:
     ```bash
     "$AGY" -p "Fix <issue> and run <test command>. Keep changes minimal." \
       --add-dir . --mode accept-edits \
       --settings '{"toolPermission":"always-proceed","artifactReviewPolicy":"always-proceed"}' \
       --output-format json --print-timeout 10m
     ```
     Works identically across Personal, Pro/Ultra, and Business/Enterprise accounts without modifying user settings.
4. **Hard prohibitions for host agents**:
   - **Do NOT pass `--dangerously-skip-permissions`** for reviews, queries, or standard coding tasks. It triggers human-approval prompts in Codex/Claude Code and is still blocked by built-in protected-command denylists and Enterprise admin review policies (`v1.5.2+`, `v1.5.7+`). Reserve it strictly as an explicit user-authorized fallback.
   - **Do NOT pass `--mode plan`** to critique an existing plan or diff. `--mode plan` is an interactive plan-drafting mode that creates a `plan.md` artifact and stops asking for approval. Use `--settings '{"agentMode":""}'` so `agy` reads files natively, returns the critique in `.response`, and engine-blocks edits.
   - **Do NOT pass `--mode default`** (rejected with a warning; valid `--mode` values are `accept-edits`, `auto`, `plan`).
   - **Do NOT rely on unconfigured default mode**: `v1.5.7+` defaults sessions with no saved `agentMode` (including `-p`) to `auto` where available, and prints warnings on Business/Enterprise accounts. Pass `--mode` or `--settings` explicitly.
   - **Do NOT nest `--sandbox`** inside an outer agent sandbox (e.g., Codex Seatbelt / Landlock). Nested OS sandboxes fail or block `git` and `~/.gemini/antigravity-cli/`. Instead, request host network/sandbox access for the `agy` invocation when needed.
5. **Mandatory completion & denial checks**:
   Never treat exit code `0` alone as success:
   - Parse stdout JSON: check `.status`, `.response`, `.finished_reason` (`print_timeout`, `interrupted`, `error` — omitted on clean completion), `.denied_actions[]`, and `.conversation_id`.
   - If `finished_reason == "print_timeout"`, recover on the same context via `--conversation <id>` ("Stop exploring; summarize findings collected so far").
   - If `denied_actions` is non-empty, add only the needed rule to `--settings '{"permissions":{"allow":[...]}}'` and retry.

See [resources/delegation_playbook.md](resources/delegation_playbook.md) for Codex/Claude Code host integration, recovery scripts, and `stream-json` parsing.

## Command-Line Flag Reference (`agy [flags] [prompt]`)

| Flag | Short | Description | Added / Updated |
| :--- | :--- | :--- | :--- |
| `--print` | `-p`, `--prompt` | Non-interactive single prompt execution. Prefer `--print='...'` when prompt text starts with `-`. | v1.0.0+ |
| `--settings <json\|path>` | | Session-only `settings.json` overrides (inline JSON or file path); never saved to disk. | v1.5.7+ |
| `--mode <mode>` | | Execution mode: `accept-edits`, `auto`, or `plan` (`v1.5.7+` defaults unconfigured sessions to `auto` where available). | v1.1.0+, v1.5.7+ |
| `--model <model>` | | Model slug (from `agy models`). Base names work without `--effort`. | v1.0.5+, v1.3.4+ |
| `--effort <level>` | | Reasoning effort: `low`, `medium`, `high`, `xhigh`, `max`. | v1.1.5+, v1.2.11+ |
| `--agent <agent>` | | Custom Markdown or built-in agent. Unknown name exits `1` in `-p` mode. | v1.1.1+, v1.3.4+ |
| `--add-dir <path>` | | Add directory to workspace scope (repeatable). Always pass in automation. | v1.0.0+ |
| `--output-format <fmt>` | | Print mode output: `text` (default), `json`, or `stream-json` (NDJSON events). | v1.1.8+ |
| `--input-format <fmt>` | | Print mode input: `text` (default) or `stream-json` (multi-turn NDJSON; requires `--output-format stream-json`). | v1.1.15+ |
| `--json-schema <schema>` | | Inline JSON schema string or file path (`"type": "object"` required at root; else exit `1`). | v1.1.8+, v1.2.14+ |
| `--print-timeout <dur>` | | Print mode deadline (`0s` default = unlimited, waits for background tasks). Exits `3` if terminating active background tasks (`v1.5.0+`), or `0` with `finished_reason: "print_timeout"` mid-turn. | v1.1.28+, v1.5.0+ |
| `--disable-slash-commands` | | Send leading `/` prompts as literal text in `-p` mode instead of expanding skills/commands. | v1.1.9+ |
| `--prompt-interactive` | `-i` | Run initial prompt in TUI and keep session open. Conflicts with `-p` (exit `2`). | v1.0.0+, v1.3.4+ |
| `--continue` | `-c` | Continue most recent conversation in workspace (parent/child dir fallback). | v1.0.0+, v1.2.1+ |
| `--conversation <id>` | | Resume a specific conversation by ID. | v1.0.0+ |
| `--dangerously-skip-permissions` | | Broad tool auto-approval (still blocks protected-command denylist and Enterprise admin-reviewed actions; does not auto-approve `/plan` artifacts). Avoid in host-agent wrappers when `--settings` works. | v1.0.0+, v1.5.2+, v1.5.7+ |
| `--sandbox` | | OS terminal containment (no-op if unavailable on host in `v1.4.4+`). | v1.0.1+, v1.4.4+ |
| `--remote-agent` | | Run conversations on cloud Remote Agents (`--project` picks remote project by ID/name). | v1.5.0+ |
| `--project <id\|name>` | | Explicitly set local or remote project ID/name. | v1.0.12+, v1.5.0+ |
| `--new-project` | | Create a new isolated project context. | v1.0.12+ |
| `--remote-control` | | Start session with an active Remote Control tunnel (unsupported with `AGY_ADC_AUTH`). | v1.2.6+, v1.3.3+ |
| `--log-file <path>` | | Override CLI log file path. | v1.0.0+ |

## Subcommands Reference

### 1. `agy acp` (Agent Client Protocol Server)
Added in `v1.5.0+` (expanded `v1.5.1+`, `v1.5.4+`). Runs `agy` as a stdio JSON-RPC ACP server so external editors/IDEs can drive sessions with images, embedded file context, mid-turn steering, `default`/`accept-edits`/`auto`/`plan` modes, `session/resume`, and per-session models:

```bash
agy acp                              # Start stdio ACP server
agy acp --login                      # Terminal Google sign-in (SSH / headless editors)
agy acp --process-per-session        # Isolate each session in its own worker process
```
Key flags: `--add-dir`, `--agent`, `--ide-mcp-port <port>` (requires `nc`), `--sandbox`, `--dangerously-skip-permissions`, `--include-default-customizations`, `--log-file`.

### 2. `agy mcp` (MCP Server Management)
Manages `~/.gemini/config/mcp_config.json` without hand-editing JSON (`v1.1.16+`):
```bash
agy mcp list
agy mcp add fs npx -y @modelcontextprotocol/server-filesystem /path/to/work
agy mcp add --env GITHUB_TOKEN=ghp_xxx gh -- docker run -i ghcr.io/github/mcp
agy mcp add --header "Authorization: Bearer token" remote-api https://mcp.internal.net/sse
agy mcp enable <name> | agy mcp disable <name> | agy mcp remove <name>
```
- `http://`/`https://` infers `--type http`. Use `--` before commands with dash flags.
- Plugin MCP servers auto-namespace as `<plugin>_<server>` (`v1.2.2+`) and expand header variables like `${TOKEN}` (`v1.4.0+`).
- `_meta: {"antigravity/alwaysLoad": true}` in `tools/list` bypasses on-demand discovery (`v1.3.3+`).
- `tools/call` forwards `_meta["antigravity.google/root_conversation_id"]` for nested subagents (`v1.4.4+`).
- On-demand tool outputs up to 10 KB return inline (`v1.4.7+`); tools declaring `explanation`, `toolSummary`, `toolAction`, or `waitForPreviousTools` receive model-sent values (`v1.5.2+`).

### 3. `agy plugin` / `agy plugins`
Manages plugins in `~/.gemini/config/plugins/`:
```bash
agy plugin list [--json]
agy plugin install <plugin>@antigravity-plugins-official
agy plugin install github.com/owner/repo/plugins/analyzer@main
agy plugin import gemini | agy plugin import claude
agy plugin enable <name> | agy plugin disable <name> | agy plugin uninstall <name>
agy plugin validate [path] [--strict] [--json]
agy plugin link <marketplace> <target>
```
- `agy plugin validate --strict --json`: validates `plugin.json` (`name` kebab-case, `category`, `author`/`homepage`, `logo`/`iconUri`), skills, agents, rules, `hooks.json`, sidecars, context providers (`v1.4.0+`–`v1.5.3+`), and warns on `${PLUGIN_ROOT}` in hook commands (`v1.5.6+`; hooks already execute in the plugin directory). Report findings are under `.findings[]`.
- `agy plugin import claude` converts marketplace plugins, remote HTTP MCPs, `.mcp.json`, manifest `"skills"` folders, `commands/` into named skills, and `PreToolUse`/`PostToolUse`/`Stop`/`SessionStart` hooks (`v1.4.11+`, `v1.5.5+`–`v1.5.6+`).
- Installs swap atomically with rollback on failure, reject unsafe names or symlink escapes (`v1.5.2+`, `v1.5.6+`), and work under `AGY_ADC_AUTH` and enterprise gateways (`v1.5.1+`).

### 4. `agy remote-control`, `agy mic-serve`, `agy agent`, `agy models`, `agy install`, `agy update`, `agy changelog`
- `agy remote-control start [--name <host>] [--session] | status | stop`: OS daemon (`systemd`/`launchd`/background). In TUI: `/remote-control [on|off]`. Unsupported with `AGY_ADC_AUTH`.
- `agy mic-serve [--addr 127.0.0.1:4713]`: Forward local microphone over SSH for `/voice`.
- `agy agent` / `agy agents` and `agy models`: Human-readable lists (both reject `--output-format`; use `agy -p "/agents" --output-format json` for JSON agent lists). Current models include Gemini 4 Carbon (`gemini-4-carbon-{low,medium,high}`), Gemini 4 Hydrogen, Gemini 4 Argon, Gemini 3.8 Flash, and Claude Opus/Sonnet 5.5 (where enabled).
- `agy install [--dir <path>] [--skip-path] [--skip-aliases]`, `agy update`, `agy changelog` (note: live release notes are at `https://antigravity.google/docs/changelog` if `agy changelog` lags behind binary version).

## Headless Automation (`-p`) Details

- **Zero-Turn Read-Only Commands**: `agy -p "/usage" --output-format json` (alias `"/quota"`), `"/skills"`, `"/permissions"`, `"/agents"` return JSON with `0` turns and `0` quota spent. Interactive-only commands (`/clear`) fail fast.
- **Bidirectional NDJSON Streaming**: `agy -p --input-format stream-json --output-format stream-json` reads `{"prompt": "..."}` lines from stdin and emits `init`, `step_update` (`state`, `step_type`, `text_delta`, `tool_info`, `subagent_info`), and `result` (`status`, `response`, `finished_reason`, `denied_actions`, `usage`).
- **JSON Schema Enforcement**: `--json-schema` requires `"type": "object"` at the root (`v1.2.14+`).
- **Exit Codes**:
  - `0`: Complete success, or mid-turn `--print-timeout` partial output (check `.finished_reason`: `print_timeout`, `interrupted`, `error`, or omitted on clean finish).
  - `1`: Config/flag error, invalid `--json-schema`, unknown `--agent` (`v1.3.4+`), or `PreInvocation` hook rejection (`v1.4.11+`).
  - `2`: Print-mode usage conflict (e.g., `-p` + `-i`).
  - `3`: Fatal model/API error (`AGY_ERROR: {...}` on stderr) or `--print-timeout` terminating active background tasks (`v1.5.0+`).
- **Headless Behavioral Safeguards**: Skips background title-gen requests (`v1.5.2+`), caps post-result shutdown at 30s (`v1.5.5+`), auto-settles main `ask_question` and skips subagent questions (`v1.3.6+`), shares permission approvals across subagents (`v1.4.6+`), and preserves parent's 10 latest messages when compacting subagents (`v1.4.10+`).

## Custom Agents, Swarms, Hooks & TUI

- **Custom Agents & Rules**: Markdown agents live in `.agents/agents/<name>.md` or `~/.gemini/config/agents/<name>.md`. Rules (`GEMINI.md`, `AGENTS.md`, `.agents/rules/*.md`) have a dedicated 20,000-token budget (`v1.2.7+`). Workspace `.agents/{plugins,skills}.json` manifests only load paths inside the workspace/repo root (`v1.5.2+`).
- **Swarms (`/swarm`, `v1.5.5+`–`v1.5.6+`)**: Coordinates parallel worker agents over a shared message board; `/swarm board` (`Alt+S`) opens live Roster/Posts/Message panes. Swarm/subagent Git worktrees inherit staged, unstaged, and untracked changes (`v1.5.5+`).
- **Hooks (`hooks.json`)**: Stdin includes `lastUserInput` (`v1.4.7+`), `hookEventName`, `cwd`, `toolCall.id` (`v1.5.4+`), and `continuation_active` on `Stop` hooks (`v1.5.6+`). `PreInvocation` hooks can return `{"terminationBehavior":"terminate","errorMessage":{"userMessage":"..."}}` to block prompts (`v1.4.11+`). Plugin hooks receive `PLUGIN_ROOT` and `PLUGIN_DATA` (`v1.5.3+`).
- **Deep References**:
  - [resources/delegation_playbook.md](resources/delegation_playbook.md) — Host-agent delegation recipes, sandbox boundaries, `stream-json`, ACP.
  - [resources/agents_and_customization.md](resources/agents_and_customization.md) — Agent schema, swarms, manifests, plugins, hooks, skill metadata.
  - [resources/tui_reference.md](resources/tui_reference.md) — TUI shortcuts, `/swarm board`, `/projects`, `/diff`, `/compact`, `/btw`, Vim mode.
  - [resources/cheat_sheet.md](resources/cheat_sheet.md) — Compact quick-reference card & Gemini CLI migration table.
  - [examples/usage_patterns.md](examples/usage_patterns.md) — 32 runnable automation recipes.

## Security, Sandbox, Auto Mode & Environment Variables

- **Sandbox (`--sandbox`)**: Read-only `.git` (`v1.1.10+`); read/write `/tmp` and `$TMPDIR` (`v1.1.9+`) plus artifact/scratch dirs (`v1.2.10+`); blocks `~/Library/Application Support` (`v1.3.3+`), `~/.docker`, `~/.kube`, `~/_netrc` (`v1.4.6+`), and sensitive Google origins (`mail`/`drive`/`docs`/`accounts`/`console.cloud`, `v1.4.9+`). Sets `GIT_EDITOR`/`HGEDITOR`/`JJ_EDITOR=false` and `PAGER`/`GIT_PAGER`/`JJ_PAGER=cat`.
- **Auto Mode (`--mode auto`)**: Default mode in `v1.5.7+` where available (macOS, Linux, Windows 11 24H2+; includes pay-as-you-go `AGY_ADC_AUTH` in `v1.5.6+`, unavailable on Business/Enterprise admin accounts). Reviews sandbox escapes with Gemini Flash (`v1.5.4+`), allowing safe GitHub API/existing-clone/public-SSH-key reads while blocking private keys and unsafe uploads (`v1.5.6+`). `toolPermission: "strict"` takes precedence over `auto` and `accept-edits` (`v1.5.7+`).
- **Rule Syntax**: Prefix match by default; `regex:` prefix for regex rules. Multi-line rules apply (`v1.4.9+`).

| Environment Variable | Purpose | Introduced |
| :--- | :--- | :--- |
| `GEMINI_API_KEY` / `GOOGLE_GEMINI_BASE_URL` | Direct Gemini API auth (`modelProvider: "gemini"`) without OAuth. | v1.1.13+ |
| `USE_ADC=1` / `AGY_ADC_AUTH` | Application Default Credentials (Gemini 4 Carbon & Auto mode for PAYG `AGY_ADC_AUTH` in `v1.5.5+`–`v1.5.6+`). | v1.0.11+, v1.3.3+ |
| `ANTIGRAVITY_DISABLE_EMBEDDED_RIPGREP=true` | Disable bundled `ripgrep` extraction (avoids endpoint-security blocks). | v1.5.5+ |
| `AGY_CLI_MAC_OPTION_AS_ALT=0` | Revert native macOS Option-as-Alt shortcuts (`alt+m`, `alt+s`). | v1.5.2+ |
| `CLI_GRAPHICS=kitty` | Force Kitty graphics inside `tmux` / `screen`. | v1.2.10+ |
| `AGY_CLI_OSC7501=on\|off` | Force/disable OSC 7501 tab status (`AGY_CLI_OSC7501_MESSAGES=brief`). | v1.4.0+ |
| `ANTIGRAVITY_CLI_ALIAS` | Override binary executable name detection. | v1.0.0+ |
| `AGY_CLI_HIDE_ACCOUNT_INFO` / `HIDE_LOGO` / `DISABLE_LATEX` | Hide email/plan header, ASCII banner, or LaTeX rendering (`true`). | v1.0.2+ |

## Gemini CLI Flag Translation

| Gemini CLI | agy CLI Equivalent | Notes |
| :--- | :--- | :--- |
| `gemini -p "prompt"` | `agy -p "prompt"` | Expands leading slash commands/skills (`--disable-slash-commands` to disable) |
| `gemini --yolo` | `agy --mode accept-edits --settings '{"toolPermission":"always-proceed"}'` | Or `--dangerously-skip-permissions` when unrestricted approval is intended |
| `gemini --resume <id>` / `-c` | `agy --conversation <id>` / `agy -c` | Resumes session / continues most recent |
| `gemini -o stream-json` | `agy -p --output-format stream-json` | Typed NDJSON stream with `finished_reason` and `usage` |
| `gemini -m <model>` | `agy --model <model>` | Base model slug or tier |
| `gemini --approval-mode plan` | `agy --settings '{"agentMode":""}'` (for reviews) / `--mode plan` (to draft a plan) | Critique vs interactive plan drafting |
| `gemini --approval-mode auto_edit` | `agy --mode accept-edits` / `--mode auto` | `auto` also approves safe commands |
| `gemini extensions list\|install` | `agy plugin list\|install` | Migrate with `agy plugin import gemini` |

## Troubleshooting

- **`agy: command not found`**: Call `$HOME/.local/bin/agy` directly or prepend `$HOME/.local/bin` to `PATH`.
- **Host agent asks human to approve `agy` call**: Remove `--dangerously-skip-permissions` and use `--settings` recipes above.
- **Review returns `"Please review and approve the plan"`**: You passed `--mode plan` for a critique; switch to `--settings '{"agentMode":""}'`.
- **Headless JSON shows `denied_actions` and empty `response`**: Add the required `permissions.allow` rule in `--settings` (or use `--mode accept-edits` + `toolPermission: "always-proceed"` for coding tasks).
- **Nested sandbox / network failure inside Codex**: Do not pass `--sandbox` inside an outer sandbox; run `agy` with host network/sandbox escalation.
- **Exit Code 3 / Low Quota**: Check `AGY_ERROR` on stderr or run `agy -p "/usage" --output-format json`; switch `--model` to another quota pool if exhausted.
- **Enterprise Gateway HTTP 400/404**: Remote gateways require `https://` (`v1.5.5+`); trailing `/v1beta` and Gemini 4 Carbon thinking suffixes are handled in `v1.5.3+`/`v1.5.7+`.
