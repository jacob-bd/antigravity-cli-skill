# agy CLI Quick Reference (v1.5.7)

## Host-Agent Native Recipes (Zero `--dangerously-skip-permissions`)
Resolve binary first: `AGY=$(command -v agy || echo "$HOME/.local/bin/agy")`
Check quota first (0 turns/tokens): `"$AGY" -p "/usage" --output-format json` (warn user if target model's `remaining_fraction < 0.20`).

```bash
# 1. Plan / Architecture Review (read-only; do NOT use --mode plan)
"$AGY" --print="Review plan.md for gaps and risks. Max 6 findings + verdict." \
  --add-dir . --settings '{"agentMode":""}' --output-format json --print-timeout 10m

# 2. Git Diff / Code Review (read-only Git commands allowed)
"$AGY" --print="Review uncommitted git diff for bugs and security issues. Max 8 findings." \
  --add-dir . \
  --settings '{"agentMode":"","permissions":{"allow":["command(git status)","command(git diff)","command(git log)","command(git show)"]}}' \
  --output-format json --print-timeout 8m

# 3. Scoped Coding + Tests (works on all account tiers)
"$AGY" --print="Fix <bug> and run <tests>. Keep changes minimal." \
  --add-dir . --mode accept-edits \
  --settings '{"toolPermission":"always-proceed","artifactReviewPolicy":"always-proceed"}' \
  --output-format json --print-timeout 10m
```

## All Command-Line Flags
| Flag | Short | What it does | Added / Updated |
| :--- | :--- | :--- | :--- |
| `--print` | `-p`, `--prompt` | Non-interactive single prompt execution | v1.0.0+ |
| `--settings <json\|path>` | | Ephemeral session-only `settings.json` overrides (never saved) | v1.5.7+ |
| `--mode <mode>` | | Execution mode: `accept-edits`, `auto`, `plan` (`v1.5.7+` defaults to `auto` where available) | v1.1.0+, v1.5.7+ |
| `--model <model>` | | Model slug or tier (`--effort` optional) | v1.0.5+, v1.3.4+ |
| `--effort <level>` | | Reasoning effort (`low`, `medium`, `high`, `xhigh`, `max`) | v1.1.5+, v1.2.11+ |
| `--agent <agent>` | | Agent for session (unknown name exits `1` in `-p`) | v1.1.1+, v1.3.4+ |
| `--add-dir <path>` | | Add directory to workspace (repeatable) | v1.0.0+ |
| `--input-format <fmt>` | | Input format (`text`, `stream-json` for continuous stdin NDJSON) | v1.1.15+ |
| `--output-format <fmt>` | | Output format (`text`, `json`, `stream-json` NDJSON events) | v1.1.8+ |
| `--json-schema <schema>` | | Enforce JSON schema (requires `"type": "object"` root) | v1.1.8+, v1.2.14+ |
| `--print-timeout <dur>` | | Print timeout (default `0s` = unlimited; exits `3` if killing background tasks in `v1.5.0+`) | v1.1.28+, v1.5.0+ |
| `--disable-slash-commands` | | Disable automatic slash-command and skill expansion in `-p` | v1.1.9+ |
| `--prompt-interactive` | `-i` | Interactive mode with initial prompt | v1.0.0+ |
| `--continue` | `-c` | Continue most recent conversation (parent/child dir fallback) | v1.0.0+, v1.2.1+ |
| `--conversation <id>` | | Resume specific session by ID | v1.0.0+ |
| `--dangerously-skip-permissions` | | Broad tool auto-approval (blocked on protected-command denylist & Enterprise admin review) | v1.0.0+, v1.5.7+ |
| `--sandbox` | | OS terminal containment (no-op if unavailable; avoid nesting inside host sandboxes) | v1.0.1+, v1.4.4+ |
| `--remote-agent` | | Run conversations on cloud Remote Agents | v1.5.0+ |
| `--project <id\|name>` | | Set local or remote project ID or name | v1.0.12+, v1.5.0+ |
| `--new-project` | | Create isolated project context | v1.0.12+ |
| `--remote-control` | | Start session with active Remote Control tunnel | v1.2.6+ |
| `--log-file <path>` | | Override CLI log file path | v1.0.0+ |

## Key Subcommands
- `agy acp [--login] [--process-per-session] [--ide-mcp-port <port>]` -- Agent Client Protocol stdio server (`v1.5.0+`)
- `agy mcp list | add | enable | disable | remove` -- Manage MCP servers in `mcp_config.json`
- `agy remote-control start | status | stop` -- Manage OS background remote-control daemon
- `agy mic-serve [--addr 127.0.0.1:4713]` -- Forward local microphone over SSH for `/voice`
- `agy agent` / `agy agents` -- Human-readable agent list (use `agy -p "/agents" --output-format json` for JSON)
- `agy models` -- Human-readable model list (does not accept `--output-format`)
- `agy plugin list [--json]` -- View installed plugins
- `agy plugin validate [path] [--strict] [--json]` -- Validate plugin manifest, skills, agents, rules, hooks (`.findings[]`)
- `agy plugin install <target> | uninstall <name> | enable/disable <name> | link <mp> <target>` -- Manage plugins
- `agy plugin import gemini | claude` -- Import Gemini CLI or Claude Code plugins
- `agy update | changelog | install` -- Update binary, view changelog, configure PATH/aliases

## Gemini CLI Migration Cheat Sheet
| Gemini CLI | agy CLI v1.5.7 | Notes |
| :--- | :--- | :--- |
| `gemini -p "prompt"` | `agy -p "prompt"` | Auto-expands leading slash commands/skills |
| `gemini --yolo` | `agy --mode accept-edits --settings '{"toolPermission":"always-proceed"}'` | Or `--dangerously-skip-permissions` |
| `gemini --resume <id>` / `-c` | `agy --conversation <id>` / `agy -c` | Resumes by ID or continues latest |
| `gemini -o stream-json` | `agy -p --output-format stream-json` | Typed NDJSON stream with `finished_reason` |
| `N/A` | `agy -p --input-format stream-json` | Stdin NDJSON stream for multi-turn sessions |
| `gemini -m <model>` | `agy --model <model>` | Explicit model slug |
| `gemini --approval-mode plan` | `agy --settings '{"agentMode":""}'` (review) / `--mode plan` (draft) | Review critique vs interactive plan drafting |
| `gemini --approval-mode auto_edit` | `agy --mode accept-edits` / `--mode auto` | `auto` also approves safe commands |
| `gemini extensions list\|install` | `agy plugin list\|install` | Migrate via `agy plugin import gemini` |

## Environment Variables
- `GEMINI_API_KEY` / `GOOGLE_GEMINI_BASE_URL` -- Direct Gemini API key auth (`modelProvider: "gemini"`).
- `USE_ADC=1` / `AGY_ADC_AUTH` -- Application Default Credentials (Gemini 4 Carbon + PAYG Auto mode in `v1.5.5+`–`v1.5.6+`).
- `ANTIGRAVITY_DISABLE_EMBEDDED_RIPGREP=true` -- Disable bundled `ripgrep` extraction (`v1.5.5+`).
- `AGY_CLI_MAC_OPTION_AS_ALT=0` -- Disable default macOS Option-as-Alt shortcuts (`v1.5.2+`).
- `AGY_CLI_OSC7501=on|off` -- Terminal tab status reporting (`AGY_CLI_OSC7501_MESSAGES=brief`).
- `CLI_GRAPHICS=kitty` -- Force Kitty graphics mode inside `tmux` / `screen`.
- `AGY_CLI_HIDE_ACCOUNT_INFO=true` / `AGY_CLI_HIDE_LOGO=true` / `AGY_CLI_DISABLE_LATEX=true`.
- `AGY_CLI_DISABLE_ESCAPE_SEQUENCE_OPTIMIZATIONS=1` / `AGY_CLI_CMD_OUTPUT_PERCENTAGE` / `ANTIGRAVITY_CLI_ALIAS`.

## TUI Panels, Shortcuts & Keybindings
- `/swarm <task>` & `/swarm board` (`Alt+S` open, `Alt+X` dismiss tray, `v1.5.5+`–`v1.5.6+`) -- Multi-agent swarm coordinator + live board.
- `/projects` (`v1.5.0+`) -- Create, edit, and switch local or remote projects.
- `/btw <query>` (`v1.5.0+`) -- Side question that can call read-only tools live.
- `/compact [message]` (`v1.3.5+`) -- Summarize conversation to free context.
- `/model <name> [prompt]` -- Switch model or run a one-shot turn on `<name>`.
- `/effort <level>` -- Adjust reasoning effort (`low`–`max`).
- `/codesearch` (`/cs`, `/search`), `/copy [n|btw]`, `/voice` (`F5`), `/skills reload`, `/permissions`, `/plugin`.
- `/diff`: `←`/`→` switch files, `/` search, `[`/`]` Per Turn jump, `n`/`N` next match, `Ctrl+O` fold, `f` changes-only.
- `/config`: `←`/`→` cycles settings; `full screen (altscreen)` (`v1.5.0+` label); `alt+m` opposite-send; `Shift+Tab` cycle modes.
- `/resume`: `Ctrl+F` workspace grouping. Artifact Viewer: `m` mode, `t` outline, `Ctrl+D`/`Ctrl+U` scroll, `Ctrl+G` `$EDITOR`.

## Automation Output & Exit Codes
- **Check both exit code and JSON fields**: `finished_reason` (`print_timeout`, `interrupted`, `error`; omitted when clean) and `denied_actions[]`.
- **Exit 0**: Clean success, or mid-turn `--print-timeout` partial output.
- **Exit 1**: Invalid flags, non-object `--json-schema`, unknown `--agent` in `-p`, or `PreInvocation` hook rejection.
- **Exit 2**: Print-mode usage error (e.g. `-p` with `-i`).
- **Exit 3**: Fatal API/model error (`AGY_ERROR: {...}` on stderr) OR `--print-timeout` terminating active background tasks (`v1.5.0+`).
