# Antigravity CLI (`agy`) Reference & Cheat Sheet (v1.5.12)

Single authoritative reference for `agy` flags, subcommands, `--settings` keys, models, sandbox/admin rules, environment variables, exit codes, and Gemini CLI migration. For the host-agent execution contract, quota decision table, and core copy-pasteable delegation recipes, see [../SKILL.md](../SKILL.md) and [delegation_playbook.md](delegation_playbook.md).

## Contents

- [Command-Line Flags](#command-line-flags)
- [Subcommands](#subcommands)
- [Ephemeral `--settings` Schema & Enums](#ephemeral---settings-schema--enums)
- [Models & Reasoning Effort](#models--reasoning-effort)
- [Security, Sandbox, Auto Mode & Admin Controls](#security-sandbox-auto-mode--admin-controls)
- [Environment Variables](#environment-variables)
- [Exit Codes & Headless Failure Semantics](#exit-codes--headless-failure-semantics)
- [Gemini CLI to Antigravity CLI Migration](#gemini-cli-to-antigravity-cli-migration)

---

## Command-Line Flags

Usage: `agy [flags] [prompt]`

### Headless, Output & Streaming

| Flag | Short | Description |
| :--- | :--- | :--- |
| `--print <prompt>` | `-p`, `--prompt` | Run a single prompt non-interactively and print the response. **Takes a string argument**; use `--print='...'` for dash-prefixed text and never write bare `-p --<flag>`. |
| `--output-format <fmt>` | | Print mode output format: `text` (default), `json`, or `stream-json` (NDJSON events). Invalid values silently fall back to `text`. |
| `--input-format <fmt>` | | Print mode input format: `text` (default) or `stream-json` (reads one `{"event":"user","message":{"content":"..."}}` NDJSON line per turn from stdin; requires `--output-format stream-json`; CLI-answered slash commands like `/usage` exit `2`). |
| `--json-schema <schema>` | | Inline JSON schema string or file path to enforce structured output. Requires `"type": "object"` at root AND `--output-format json` or `stream-json` (else exit `1`). |
| `--print-timeout <dur>` | | Optional wall-clock limit for print mode (default `0s` = unlimited, waits for background tasks). Exits `3` if terminating active background tasks, or `0` with `finished_reason: "print_timeout"` mid-turn. |
| `--disable-slash-commands` | | Send leading `/` prompts as literal text in print mode instead of expanding slash commands or skills. |

### Model, Effort & Agent

| Flag | Short | Description |
| :--- | :--- | :--- |
| `--model <slug>` | | Model slug for the session (list available slugs with `agy models`). Base names work without `--effort`. |
| `--effort <level>` | | Reasoning effort: `low`, `medium`, `high`, `xhigh`, `max`. Fails fast with exit `1` if the selected model or sign-in does not offer `<level>`. |
| `--agent <name>` | | Custom Markdown or built-in agent for the session. Unknown name exits `1` in print mode. |

### Permissions, Mode & Workspace Scope

| Flag | Short | Description |
| :--- | :--- | :--- |
| `--settings <json\|path>` | | Session-only `settings.json` overrides (never saved to disk; requires `agy` v1.5.7+). Values starting with `{` parse as inline JSON; other values load from a file path. |
| `--mode <mode>` | | Execution mode: `accept-edits`, `auto`, or `plan`. Unconfigured sessions default to `auto` where available. `--mode default` is invalid (warns and continues with default mode). |
| `--add-dir <path>` | | Add a directory to workspace scope (repeatable). Always pass in automation; `cd` alone does not add a directory to workspace scope. |
| `--sandbox` | | Enable OS terminal containment (no-op if host lacks sandbox support). Do not nest inside an outer host-agent sandbox (Seatbelt/Landlock). |
| `--dangerously-skip-permissions` | | Broad tool auto-approval (still blocked by protected-command denylist and Enterprise admin-enforced review; does not auto-approve `/plan` artifacts). Avoid in host-agent wrappers unless the user explicitly asks for it. |

### Session, Remote, Interactive & Diagnostics

| Flag | Short | Description |
| :--- | :--- | :--- |
| `--continue` | `-c` | Continue the most recent conversation in the workspace (with parent/child directory fallback). |
| `--conversation <id>` | | Resume a specific conversation by ID. |
| `--prompt-interactive <prompt>` | `-i` | Run an initial prompt in the TUI and keep the session open. Conflicts with `-p` (exit `2`). |
| `--remote-agent` | | Run conversations on cloud Remote Agents (`--project` names a remote project by ID or name). Exits `2` under `GEMINI_API_KEY`. |
| `--project <id\|name>` | | Explicitly set a local or remote project ID or name. |
| `--new-project` | | Create a new isolated project context for this session. |
| `--remote-control` | | Create a Remote Control tunnel on startup. Unsupported with `AGY_ADC_AUTH` and exits `2` under `GEMINI_API_KEY`. |
| `--log-file <path>` | | Override the CLI log file path. |

---

## Subcommands

### `agy acp` (Agent Client Protocol Server)

Runs `agy` as a stdio JSON-RPC ACP server (`--stdio` default) for editors and IDEs, supporting images, embedded file context, mid-turn steering, `default`/`accept-edits`/`auto`/`plan` modes, `session/resume`, per-session models, live context-window usage updates, immediate post-turn background/subagent streaming, and automatic cleanup of subagents/background tasks on cancel or session close.

```bash
agy acp                              # Start stdio ACP server
agy acp --login                      # Terminal Google sign-in (SSH / headless editors; falls back to saved login)
agy acp --process-per-session        # Isolate each session in its own worker process
```

Key flags: `--add-dir`, `--agent`, `--ide-mcp-port <port>` (requires `nc` in `$PATH`), `--sandbox`, `--dangerously-skip-permissions`, `--include-default-customizations`, `--log-file`.

### `agy mcp` (MCP Server Management)

Manages `~/.gemini/config/mcp_config.json` declaratively. Flags must appear before `<name>`; use `--` before commands containing dash flags.

```bash
agy mcp list
agy mcp add fs npx -y @modelcontextprotocol/server-filesystem /path/to/work
agy mcp add --env GITHUB_TOKEN=ghp_xxx gh -- docker run -i ghcr.io/github/mcp
agy mcp add --header "Authorization: Bearer token" remote-api https://mcp.internal.net/mcp
agy mcp enable <name> | agy mcp disable <name> | agy mcp remove <name>
```

- `http://` and `https://` URLs automatically select Streamable HTTP (`--type http`). **Legacy HTTP+SSE (`/sse`, MCP spec 2024-11-05) is not supported**; point `url` at the Streamable HTTP endpoint (typically `/mcp`) or wrap legacy SSE servers with a stdio bridge such as `mcp-remote`.
- Plugin MCP servers auto-namespace as `<plugin>_<server>` and expand header variables like `${TOKEN}`.
- Tools returning `_meta: {"antigravity/alwaysLoad": true}` in `tools/list` load eagerly; `tools/call` forwards `_meta["antigravity.google/root_conversation_id"]` for nested subagents; on-demand tool outputs up to 10 KB return inline.

### `agy plugin` / `agy plugins`

Manages plugins in `~/.gemini/config/plugins/`.

```bash
agy plugin list [--json]
agy plugin install <plugin>@antigravity-plugins-official
agy plugin install github.com/owner/repo/plugins/analyzer@main
agy plugin import gemini | agy plugin import claude
agy plugin enable <name> | agy plugin disable <name> | agy plugin uninstall <name>
agy plugin validate [target] [--strict] [--json]
agy plugin link <marketplace> <target>
```

- `agy plugin validate [target] [--strict] [--json]` accepts a local directory or `plugin.json` (default: `.`), a `.zip` file or `https://` zip URL, a GitHub repo or subfolder URL (`https://github.com/<owner>/<repo>/tree/<branch>/<path>`), or a `marketplace.json` file/URL. Remote targets are fetched to a temp folder and only read (never installed or executed). JSON findings appear under `.findings[]`.
- `agy plugin import claude` converts Claude Code marketplace plugins, remote HTTP MCP servers, `.mcp.json`, manifest `"skills"` arrays, `commands/` into named skills, and `PreToolUse`/`PostToolUse`/`Stop`/`SessionStart` hooks.
- Installs swap atomically with rollback on failure, reject unsafe names/symlink escapes/FIFOs/device files, and handle UTF-8 BOM `plugin.json` files.

### Other Subcommands

- `agy remote-control start [--name <host>] [--session] | status | stop` — Manage the OS background daemon (`systemd`, `launchd`, or background process fallback in containers).
- `agy mic-serve [--addr 127.0.0.1:4713]` — Forward local microphone over SSH for `/voice`.
- `agy models` — Human-readable list of `<model>-<effort>` slugs available to the current account (rejects `--output-format`).
- `agy agent` / `agy agents` — Human-readable list of custom agents (rejects `--output-format`; use `agy -p "/agents" --output-format json` for JSON).
- `agy install [--dir <path>] [--skip-path] [--skip-aliases]`, `agy update`, `agy changelog` — Install PATH configuration, update binary, or inspect release notes. The binary auto-updates in the background; always check `agy --version` before asserting the installed version.

---

## Ephemeral `--settings` Schema & Enums

Requires `agy` v1.5.7+. Only keys present in `--settings` are overridden; unspecified keys inherit from `~/.gemini/antigravity-cli/settings.json`. Unknown keys fail fast with exit `1`.

| Key | Valid Values | Behavior Notes |
| :--- | :--- | :--- |
| `agentMode` | `""`, `"accept-edits"`, `"auto"`, `"plan"` | `""` selects default (request-review) mode. `"default"` is rejected (exit `1`). |
| `toolPermission` | `"strict"`, `"request-review"`, `"always-proceed"` | **`"request-review"`** allows workspace file reads (`read_file`/`view_file`) while denying unapproved writes/commands and honoring `permissions.allow` — use with `"permissions":{"allow":[]}` for pure file-read reviews (clears saved `global` allow rules) or explicit `permissions.allow` rules for Git/test reviews. **`"strict"`** blocks **every** tool including `read_file` inside `--add-dir .` in `-p` mode (use only in interactive TUI or zero-tool pure-text prompts). **`"always-proceed"`** auto-approves tools even when `agentMode: ""`. Always set `toolPermission` explicitly on read-only runs so a saved `"always-proceed"` setting cannot leak writes. |
| `artifactReviewPolicy` | `"asks-for-review"`, `"always-proceed"`, `"agent-decides"` | Set `"always-proceed"` alongside `toolPermission: "always-proceed"` for unattended coding or `/plan` runs. |
| `permissions` | `{"allow": [...], "deny": [...], "ask": [...]}` | Replaces the `global` permission scope for this session while `shared` scope rules stay active. Rules use prefix matching (or `regex:` prefix). When using `command(git ...)` allowlists, **`cd` into the target repo first** so the model runs `git diff` rather than `cd /repo && git diff` (which fails prefix matching). |
| `modelProvider` | `"gemini"`, etc. | Set `"gemini"` with `GEMINI_API_KEY` for direct Gemini API routing without touching `settings.json`. |

---

## Models & Reasoning Effort

- Run `agy models` to discover exact `<model>-<effort>` slugs available to the active account and sign-in method.
- **Gemini 4 Neon** (`gemini-4-neon-{low,medium,high,xhigh}`): Supports `xhigh` on both Google sign-in and `GEMINI_API_KEY`.
- **Gemini 4 Carbon / Hydrogen / Argon**: Top out at `high` on Google sign-in; Hydrogen offers `xhigh` with `GEMINI_API_KEY`, `AGY_ADC_AUTH`, or an enterprise gateway.
- **Orchestrator** (`gemini-pro-orchestrator`): Available on Google sign-in (where enabled), Gemini Enterprise, and `AGY_ADC_AUTH`; plans with Gemini 4 Carbon (High) (or Argon where offered) and executes with Gemini 4 Hydrogen (High).
- **Gemini 3.x Flash** (`gemini-3.8-flash-{low,medium,high}`, etc.) & **Gemini 3.1 Pro** (`low`/`high`): Fast/separate quota pools.
- **Claude Opus / Sonnet 5.5** (where enabled): Only models that support `--effort max`. Passing `--effort max` on Gemini models fails fast with exit `1`.
- **Quota inspection**: Run `agy -p "/usage" --output-format json` (0 turns, 0 tokens). Under `GEMINI_API_KEY`, `/usage`, `/quota`, and `/credits` exit `2` (no account quota; skip pre-flight checks when using an API key).

---

## Security, Sandbox, Auto Mode & Admin Controls

- **Terminal Sandbox (`--sandbox`)**:
  - Read-only `.git`; read/write `/tmp`, `$TMPDIR`, and the conversation's artifact/scratch directories.
  - Blocks reading `~/.config` (`$XDG_CONFIG_HOME`), Windows `AppData\Roaming`, `~/.git-credentials`, `~/Library/Application Support`, `~/.docker`, `~/.kube`, `~/_netrc`, and sensitive Google origins (`mail`, `drive`, `docs`, `accounts`, `console.cloud`). Common developer tool configs (Git, Go, npm, Python) remain readable; other paths can be allowed explicitly via sandbox rules.
  - Sets `GIT_EDITOR`/`HGEDITOR`/`JJ_EDITOR=false` and `PAGER`/`GIT_PAGER`/`JJ_PAGER=cat`.
  - Network proxy normalizes `%2F`, backslashes, and `..` before checking path rules; HTTP MCP custom headers are never forwarded across cross-origin redirects.
- **Auto Mode (`--mode auto`)**:
  - Default mode for unconfigured sessions where available (macOS, Linux, Windows 11 24H2+, pay-as-you-go `AGY_ADC_AUTH`; unavailable on Business/Enterprise admin-managed accounts).
  - Honors `Non-Workspace Access` in `/config`. `toolPermission: "strict"` takes precedence over `auto` and `accept-edits`.
  - Sandbox-escape reviewer searches conversation/files directly and uses medium thinking on Gemini 4 Hydrogen for faster, lower-timeout approvals; allows safe GitHub API/existing-clone/public-SSH-key reads while blocking private keys and unsafe uploads.
- **Permission Rule Syntax**: Prefix match by default; `regex:` prefix for regular expressions; multi-line rules supported. On Windows, commands executed via `cmd`/`powershell`/`pwsh` require an exact match. `mcp(server/tool)` deny/ask rules also apply to eager-loaded MCP tools and `/browser` DevTools tools.
- **Retry & Timeout Guarantees**:
  - Server-requested rate-limit delays retry up to **4 minutes** (spend-cap, prepaid-balance, and weekly quota limits stop immediately).
  - Silent high-effort thinking requests wait up to **10 minutes** before being treated as stalled.
  - Transient connection resets, mid-stream drops, and post-wake network errors retry automatically.
- **Enterprise / Admin Settings**:
  - `pluginControls.pluginsDisabled` (blocks plugins pre-login and stops running plugin MCP/processes immediately if toggled mid-session) and `pluginControls.marketplaceDisabled`.
  - `gateway.routeWebFetch` (blocks web fetches outside `proxy.address`), `featureControls.remoteControlEnabled`, and `models.list[].supportsThoughtCirculation`.
  - Remote enterprise model gateways require `https://` (`localhost`/`127.0.0.0/8`/`::1` exempt).

---

## Environment Variables

All variable names below are verified against `agy` v1.5.12:

| Variable | Purpose |
| :--- | :--- |
| `GEMINI_API_KEY` | Direct Gemini API authentication (`modelProvider: "gemini"`) without OAuth. |
| `GOOGLE_GEMINI_BASE_URL` | Custom Gemini API base URL when using `GEMINI_API_KEY`. |
| `AGY_ADC_AUTH` / `USE_ADC=1` | Authenticate with Google Cloud Application Default Credentials (supports Gemini 4 Carbon/Hydrogen, Orchestrator, and PAYG Auto mode). |
| `AGY_LLM_GATEWAY_MODELS` | Additional model entries routed through an enterprise model gateway. |
| `ANTIGRAVITY_DISABLE_EMBEDDED_RIPGREP=true` | Disable bundled `ripgrep` extraction (avoids endpoint-security blocks; falls back to `PATH` `rg` or built-in search). |
| `AGY_CLI_MAC_OPTION_AS_ALT=0` | Revert native macOS Option-as-Alt shortcuts (`alt+m`, `alt+s`). |
| `AGY_CLI_OSC7501=on\|off` | Force or disable OSC 7501 terminal tab status reporting. |
| `AGY_CLI_OSC7501_MESSAGES=brief` | Show OSC 7501 status only without pending-action details. |
| `CLI_GRAPHICS=kitty` | Force Kitty graphics mode inside `tmux` or GNU `screen` with passthrough. |
| `AGY_CLI_HIDE_ACCOUNT_INFO=true` | Hide email address and plan tier from the startup header. |
| `AGY_CLI_HIDE_LOGO=true` | Suppress startup ASCII logo art while keeping version and account lines. |
| `AGY_CLI_DISABLE_LATEX=true` | Disable inline LaTeX math rendering. |
| `AGY_CLI_DISABLE_ESCAPE_SEQUENCE_OPTIMIZATIONS=1` | Bypass renderer dirty-rectangle/diffing optimizations. |
| `AGY_CLI_CMD_OUTPUT_PERCENTAGE` | Customize max TUI height of command outputs as a percentage of terminal height. |
| `ANTIGRAVITY_CLI_ALIAS` | Override CLI executable alias detection. |
| `PYTHONUTF8` | Agent-run commands on Windows set `PYTHONUTF8=1` unless already defined. |

---

## Exit Codes & Headless Failure Semantics

Always check exit code AND `[ -s out.json ]` before parsing JSON, then verify `.finished_reason` and `.denied_actions[]`:

| Exit Code | Meaning | Action |
| :--- | :--- | :--- |
| `0` | Clean completion, **or** mid-turn `--print-timeout` partial output, **or** headless tool denial (`status: "SUCCESS"`, `response: ""`, non-empty `.denied_actions[]`). | Check `.finished_reason` (`print_timeout`, `interrupted`, `error` — absent on clean finish) and `.denied_actions[]`. Resume partial timeouts with `--conversation <id>`. |
| `1` | Invalid flags/enum/unknown `--settings` key, invalid `--json-schema` (missing `--output-format json\|stream-json` or non-object root), unavailable `--effort`, unknown `--agent`, `PreInvocation` hook termination, `/credits` on accounts without AI credits, or host sandbox blocking `~/.gemini/antigravity-cli/installation_id`. | Fix flags/schema/model, inspect stderr, or escalate out of outer host sandbox. |
| `2` | Usage/mode conflict: `-p` with `-i`, bare `-p --<flag>` (missing prompt string), `--input-format stream-json` without `--output-format stream-json`, CLI-answered slash commands (e.g. `/usage`) inside `--input-format stream-json`, invalid `--input-format`/`--print-timeout`, interactive-only `/clear` in `-p`, or `GEMINI_API_KEY` combined with `/usage`, `/quota`, `/credits`, `--remote-control`, or `--remote-agent`. | Fix CLI invocation or skip unsupported commands/flags under `GEMINI_API_KEY`. |
| `3` | Fatal API/model error (`AGY_ERROR: {...}` JSON line on stderr) **or** `--print-timeout` terminating active background tasks. | Parse `AGY_ERROR` on stderr (`sed -n 's/^AGY_ERROR:[[:space:]]*//p'`); check `/usage` or resume `--conversation`. |

**Silent fallbacks to watch for:**
- `--mode default` logs a warning to stderr and **exits `0`** using the saved/default mode.
- Invalid `--output-format` (e.g. `yaml`) **silently falls back to `text` and exits `0`**.
- Default `-p` (`--input-format text`) **silently ignores piped stdin** — never run `git diff | agy -p "review"`.

---

## Gemini CLI to Antigravity CLI Migration

| Gemini CLI | `agy` CLI (v1.5.12) | Notes |
| :--- | :--- | :--- |
| `gemini -p "prompt"` | `agy -p "prompt" --add-dir .` | Auto-expands leading slash commands/skills (add `--disable-slash-commands` for arbitrary text). |
| `gemini --yolo` | `agy --mode accept-edits --settings '{"toolPermission":"always-proceed","artifactReviewPolicy":"always-proceed"}'` | Native ephemeral approval across all account tiers without triggering host approval prompts. |
| `gemini --resume <id>` / `-c` | `agy --conversation <id>` / `agy -c` | Resumes by conversation ID or continues latest workspace session. |
| `gemini -o stream-json` | `agy -p "prompt" --output-format stream-json` | Typed NDJSON events (`init`, `step_update` with `step_type: "tool"`, `result` with `finished_reason`). Never pass bare `-p --output-format`. |
| *(no direct equivalent)* | `agy --input-format stream-json --output-format stream-json` | Continuous multi-turn NDJSON runner over stdin/stdout (omit `-p` or pass `-p ""`). |
| `gemini -m <model>` | `agy --model <model>` | Base model name or `<model>-<effort>` slug from `agy models`. |
| `gemini --approval-mode plan` | `agy --settings '{"agentMode":"","toolPermission":"request-review","permissions":{"allow":[]}}'` (critique) / `--mode plan` (interactive draft) | Never use `--mode plan` for headless critiques, and never use `"toolPermission":"strict"` when `agy` must read files (`strict` denies `read_file` in `-p`). Add `command(git ...)` entries to `permissions.allow` when read-only Git commands are needed. |
| `gemini --approval-mode auto_edit` | `agy --mode accept-edits` or `agy --mode auto` | `auto` also runs commands in the sandbox and reviews sandbox escapes. |
| `gemini extensions list\|install` | `agy plugin list\|install` | Migrate existing extensions with `agy plugin import gemini`. |
