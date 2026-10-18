---
name: antigravity-cli
description: "Expert guide for Google's Antigravity CLI (agy), the official successor to Gemini CLI. Use when the user mentions 'agy', 'antigravity', 'antigravity cli', 'gemini cli replacement', 'gemini cli migration', or any task involving the agy command-line tool including running prompts, managing plugins, resuming sessions, or automating agy in scripts and CI/CD pipelines."
version: 1.4.2
---

# Antigravity CLI (agy) Skill

Targets `agy` v1.4.2.

Use this skill to guide AI coding agents, automate CI/CD pipelines, orchestrate multi-agent swarms, configure Model Context Protocol (MCP) servers, manage custom Markdown agents, and control workspace environments with the `agy` CLI.

## Context: Gemini CLI Successor

The `agy` CLI is Google's official replacement for Gemini CLI, announced at Google I/O on May 19, 2026. Gemini CLI ceased serving requests for consumer and free users on June 18, 2026. Enterprise users on Gemini Code Assist Standard/Enterprise retain Gemini CLI access indefinitely.

Key architectural characteristics:
- **Go binary** (not Node.js) — instant startup latency, zero npm dependencies, bundled embedded `ripgrep` engine.
- **Unified agent platform** — shares the identical agent harness, runtime, and customization engine as the Antigravity IDE desktop application.
- **Plugin system** — replaces legacy Gemini CLI extensions (`agy plugin import gemini` to migrate).
- **High tool limit** — supports up to **512 tool calls** per turn on Gemini models for deep multi-step agentic execution.
- **Binary identification** — `agy` (primary executable), `antigravity` (certain Linux distributions). Set the `ANTIGRAVITY_CLI_ALIAS` environment variable to override binary resolution.
- **Default install path** — typically `~/.local/bin/agy`.

## Complete Command-Line Flag Reference

### Primary Command: `agy [flags] [prompt]`

| Flag | Shorthand | Description | Added / Updated |
| :--- | :--- | :--- | :--- |
| `--print` | `-p`, `--prompt` | Runs a single prompt non-interactively and prints the response. | v1.0.0+ |
| `--input-format <fmt>` | | Input format for print mode: `text` (default) or `stream-json`. `stream-json` reads NDJSON messages line-by-line from stdin for continuous multi-turn sessions. Requires `--output-format stream-json`. | v1.1.15+ |
| `--output-format <fmt>` | | Output format for print mode: `text` (default), `json`, or `stream-json` (strongly-typed NDJSON event stream). | v1.1.8+ |
| `--json-schema <schema>` | | Enforces a JSON schema on print mode output. Accepts an inline JSON schema string or a path to a schema file. Must have `"type": "object"` at the root; bare types fail at startup. | v1.1.8+, v1.2.14+ |
| `--effort <level>` | | Reasoning effort for the session: `low`, `medium`, `high`, `xhigh`, `max`. | v1.1.5+, v1.2.11+ |
| `--disable-slash-commands`| | Disables automatic expansion of leading slash commands and skills in headless print mode (`-p`). | v1.1.9+ |
| `--prompt-interactive` | `-i` | Runs an initial prompt interactively in the TUI and continues the session. | v1.0.0+ |
| `--continue` | `-c` | Continues the most recent conversation in the current workspace (with parent/child directory fallback). | v1.0.0+, v1.2.1+ |
| `--conversation <id>` | | Resumes a specific conversation by its unique ID. | v1.0.0+ |
| `--dangerously-skip-permissions` | | **CRITICAL FOR AUTOMATION.** Auto-approves all tool permission requests without interactive prompts. Since v1.3.2 it no longer auto-approves implementation plans in `/plan` mode; use `Artifact Review` in `/config` for that. | v1.0.0+, v1.3.2+ |
| `--sandbox` | | Runs terminal operations in a secure restricted sandbox. | v1.0.1+ |
| `--agent <agent>` | | Specifies a custom agent (Markdown or built-in) for the current CLI session. An unknown name makes headless `-p` runs exit `1` and list available agents; interactive sessions show an `Agent Not Found` warning and fall back to the default agent. | v1.1.1+, v1.2.11+, v1.3.4+ |
| `--model <model>` | | Model slug or identifier for the current CLI session. A base model name works without `--effort` (the CLI picks the model's default thinking level). | v1.0.5+, v1.1.22+, v1.3.4+ |
| `--mode <mode>` | | Sets agent execution mode: `default` (request review), `accept-edits`, `auto` (approves edits and safe commands, still prompts before irreversible actions), or `plan`. | v1.1.0+, v1.3.5+ |
| `--print-timeout <dur>` | | Timeout for print mode. **Default is `0s` (unlimited)** — runs until the turn completes. When set, a mid-turn expiration returns partial output and exits 0 with a stderr warning. | v1.1.28+, v1.2.6+ |
| `--add-dir <path>` | | Adds a directory to the workspace scope (repeatable). | v1.0.0+ |
| `--project <id\|name>` | | Explicitly sets the project ID or project name for the session. | v1.0.12+, v1.1.18+ |
| `--new-project` | | Creates a new isolated project context for the session. | v1.0.12+ |
| `--remote-control` | | Initiates a session-scoped remote connection on startup for remote following/control. | v1.2.6+ |
| `--log-file <path>` | | Overrides the CLI log file path. | v1.0.0+ |

## Complete Subcommands Reference

### 1. `agy mcp` (Model Context Protocol Management)
Added in `v1.1.16+`. Manage user-level `~/.gemini/config/mcp_config.json` directly from the CLI without manual JSON editing:

```bash
# List all configured MCP servers and their status
agy mcp list

# Add or update a stdio-based MCP server
agy mcp add fs npx -y @modelcontextprotocol/server-filesystem /path/to/work

# Add an stdio server with environment variables (repeatable -e)
agy mcp add --env GITHUB_TOKEN=ghp_xxx gh -- docker run -i ghcr.io/github/mcp

# Add an HTTP/SSE server with custom headers (repeatable -H)
agy mcp add --header "Authorization: Bearer sso_token" remote-api https://mcp.internal.net/sse

# Toggle server enablement
agy mcp disable remote-api
agy mcp enable remote-api

# Remove a server configuration
agy mcp remove fs
```

*Notes:*
- URLs starting with `http://` or `https://` automatically infer `--type http`.
- Use `--` before the command if any argument begins with a dash `-`.
- Plugin MCP servers are automatically namespaced as `<plugin>_<server>` to prevent name collisions (`v1.2.2+`).
- MCP servers can mark a tool as always loaded by setting `_meta: {"antigravity/alwaysLoad": true}` on it in their `tools/list` response, so the agent gets it directly instead of discovering it on demand (`v1.3.3+`).
- Plugin MCP servers can use variables in request headers (e.g. `"Authorization": "Bearer ${TOKEN}"`); they are resolved to your configured values (`v1.4.0+`).
- If an MCP server process exits unexpectedly, `/mcp` marks it `Error` instead of `Ready` (`v1.4.1+`).

### 2. `agy remote-control` (Remote Daemon Management)
Added in `v1.2.0+` and `v1.2.6+`. Manages the background Remote Control service registered with your operating system's service manager (systemd, launchd, or background process):

```bash
# Register and start the background daemon
agy remote-control start

# Start with a custom machine name and session scope
agy remote-control start --name "devbox-chicago" --session

# Inspect running daemon status, active tunnel, and PID
agy remote-control status

# Stop and unregister the background service
agy remote-control stop
```

*Inside interactive TUI:*
- Run `/remote-control` to open a session-scoped tunnel.
- Run `/remote-control off` to tear down the active tunnel.

### 3. `agy mic-serve` (Audio Dictation Forwarding)
Added in `v1.1.21+`. Forwards a local machine's microphone to an `agy` CLI session running on a remote host (e.g. over SSH) for terminal speech dictation (`/voice` or `F5`):

```bash
# Run on local client machine
agy mic-serve --addr 127.0.0.1:4713
```

### 4. `agy agent` / `agy agents`
List and inspect available custom and built-in agents.
```bash
# Human-readable list
agy agent

# Machine-readable list (agy agent rejects --output-format)
agy -p "/agents" --output-format json
```

### 5. `agy models`
List available models for the current sign-in credential or API key.
```bash
# Human-readable list
agy models

# Plain list (agy models rejects --output-format)
agy models
```

### 6. `agy plugin` / `agy plugins`
Manage agent plugins (stored in `~/.gemini/config/plugins/`):
```bash
# List installed plugins (marketplace and imported) with version and enabled state
agy plugin list
agy plugin list --json

# Install from the official marketplace, another marketplace, or GitHub repo subpaths with branch resolution
agy plugin install <plugin-name>@antigravity-plugins-official
agy plugin install security-auditor@marketplace
agy plugin install github.com/owner/repo/plugins/analyzer@main

# Migrate legacy Gemini CLI extensions or Claude extensions
agy plugin import gemini
agy plugin import claude

# Enable or disable plugins (enablement state persists in config.json)
agy plugin enable <name>
agy plugin disable <name>

# Validate a plugin directory or its plugin.json
agy plugin validate ./my-plugin

# CI-friendly: fail on warnings and emit a machine-readable report
agy plugin validate ./my-plugin --strict --json
```

*Notes:*
- `agy plugin list --json` added in `v1.3.4+`; it lists every installed plugin, not only imported ones.
- `agy plugin validate` (`v1.4.0+`, expanded in `v1.4.1+`) checks a plugin with the same rules the CLI uses to load it, including its `plugin.json` (unknown fields with "did you mean" hints, invalid names, components misplaced inside `.antigravity-plugin/`), skills, subagents, and rules. Each error and warning is reported with its file and line.
- Inside the TUI, `/plugin` (`v1.3.2+`) opens the interactive manager for the official marketplace (browse, install, `install`/`uninstall`/`enable`/`disable`/`list`, or `+ Install from local directory...`). Plugin names and manifests are case-insensitive (`v1.3.6+`).

### 7. `agy install`
Configures environment paths and shell configuration profiles (`.zshrc`, `.bashrc`):
```bash
agy install
agy install --dir /usr/local/bin
agy install --skip-path
agy install --skip-aliases
```

### 8. `agy update` & `agy changelog`
```bash
# Update the CLI binary to the latest release
agy update

# View changelog and release notes (also supported via agy -p "/changelog")
agy changelog
```

## Agentic Automation & Headless Scripting (`-p`)

### 1. Headless Slash-Command & Skill Expansion
As of `v1.1.9+`, headless print mode (`-p`) automatically expands slash commands and installed skills rather than treating them as literal text:

```bash
# Headless run that triggers the code-review skill
agy -p "/jbd-code-review-skill review the uncommitted diff" --dangerously-skip-permissions
```
To opt out and send leading slashes as literal text, pass `--disable-slash-commands`.

### 2. Zero-Turn Read-Only Command Queries
As of `v1.1.11+` and `v1.1.12+`, running read-only slash commands in print mode emits machine-readable data **without starting an agent turn, without consuming model quota, and without writing database records**:

```bash
# Check remaining quota in JSON format
agy -p "/quota" --output-format json

# Inspect token usage and costs
agy -p "/usage" --output-format json

# Check G1 credits balance
agy -p "/credits" --output-format json

# Export active permissions or installed skills
agy -p "/permissions" --output-format json
agy -p "/skills" --output-format json
```
Interactive-only commands (such as `/clear`) fail fast with a descriptive error rather than faking execution.

### 3. Bidirectional Continuous Streaming (`--input-format stream-json`)
As of `v1.1.15+`, scripts and test harnesses can maintain a persistent, multi-turn conversation over stdin/stdout using line-delimited NDJSON:

```bash
# Launch persistent streaming runner
agy -p --input-format stream-json --output-format stream-json --dangerously-skip-permissions
```

**Input protocol (one JSON object per line on stdin):**
```json
{"prompt": "Analyze main.go for performance bottlenecks"}
{"prompt": "Generate a patch fixing the memory allocation in WorkerPool"}
```

**Output protocol (one envelope per line: `{"event": "<name>", "<name>": {payload}}`):**
- `init`: session metadata (cwd, permission mode, tools).
- `step_update`: `state` (`ACTIVE`/`DONE`), `step_type`, incremental `text_delta`, `tool_info` (tool name, args, output), and `subagent_info` (`conversation_id`, `log_uri`).
- `result`: final answer, status, and token accounting (`input_tokens`, `output_tokens`, `thinking_tokens`, `cache_read_tokens`).

### 4. Enforcing Structured Output Schema (`--json-schema`)
Use `--json-schema` to guarantee that the final response conforms to a JSON schema:

```bash
agy -p "Extract all exported functions in auth.go" \
  --output-format stream-json \
  --json-schema '{"type":"object","properties":{"functions":{"type":"array","items":{"type":"string"}}},"required":["functions"]}' \
  --dangerously-skip-permissions
```
*Validation rule (`v1.2.14+`):* The root schema **must** have `"type": "object"`. Plain text or bare types (`string`) immediately terminate execution with exit code `1`.

### 5. Exit Codes & Structured Error Reporting
The `agy` CLI uses distinct exit codes to simplify automation error handling:
- **`0`**: Success, or graceful partial completion when `--print-timeout` expires.
- **`1`**: Startup configuration errors, invalid command flags, malformed `--json-schema`, or an unknown `--agent` name in `-p` mode (`v1.3.4+`).
- **`2`**: Print-mode usage errors, such as combining `--print` with `--prompt-interactive` (`v1.3.4+`).
- **`3`**: Fatal agent or model API error (`v1.2.6+`, `v1.2.10+`). When exit code 3 is returned, the CLI outputs a structured JSON error line on stderr:
  ```text
  AGY_ERROR: {"canonical_status": "RESOURCE_EXHAUSTED", "error_code": 429, "retryable": false, "error_id": "err_quota_exceeded"}
  ```

### 6. Non-Interactive Autonomous Decision Making
In headless `-p` runs:
- Implementation plan reviews (`--mode plan`) settle without blocking (`v1.1.28+`). For unattended interactive plan approval, set `Artifact Review` in `/config` (`--dangerously-skip-permissions` no longer implies it since `v1.3.2`).
- User confirmation questions (`ask_question`) auto-settle choices autonomously rather than hanging (`v1.1.12+`).
- Subagent permission denials are handled strictly without attempting unauthorized workarounds (`v1.2.15+`).
- Questions asked by subagents and workflow agents are skipped in `-p` runs so the subagent can continue (`v1.3.6+`).

## Delegating Work to agy (Host CLI Integration)

Coding assistants (Claude Code, OpenAI Codex, OpenCode, etc.) can delegate plan reviews, code reviews, and coding tasks to `agy` using headless print mode (`-p`).

Key mechanics:
- Always pass `--dangerously-skip-permissions` to avoid hanging on interactive prompts.
- Always scope the target directory with `--add-dir .` (or target path).
- Use `--mode plan` for read-only plan/architecture reviews.
- Omit `--mode plan` for code-writing tasks (agy will edit workspace files directly).
- Capture structured outputs using `--output-format json` (result in `.response`).
- Iterate on the same task using `--conversation <id>` with delta instructions.
- A fired `--print-timeout` still exits `0` with `status: SUCCESS` and partial output: check stderr for the `print timeout` marker, then request the final report on the same `--conversation <id>`.

**Read [resources/delegation_playbook.md](resources/delegation_playbook.md)** for complete delegation recipes, background execution patterns, error handling, and MCP/skill integration.

## Custom Agents & Customization System

Custom agents are Markdown files with YAML frontmatter in `.agents/agents/<name>.md` (project) or `~/.gemini/config/agents/<name>.md` (global), launched with `agy --agent <name>` or delegated to as subagents. Rules (`GEMINI.md`, `AGENTS.md`, `.agents/rules/*.md`) get a dedicated 20,000-token budget so they never evict skills, subagents, or MCP tools. Manifests (`skills.json`, `rules.json`, `agents.json`, `plugins.json`) are discovered across every `.agents/` folder up to the Git root. Plugins can ship skills, rules, MCP servers, hooks, and (since `v1.3.3`) subagents via a root `agents.json`.

**Read [resources/agents_and_customization.md](resources/agents_and_customization.md)** for the full agent schema, frontmatter fields, subagent messaging (`@<name>`), manifest scanning rules, plugin authoring, and skill metadata/hot reloading.

## Interactive Interface (TUI)

Key interactive features: modal Vim editing, `/model <name> [prompt]` one-shot switching, `/config` with `←`/`→` value cycling, `alt+m` opposite-send, `/compact [message]` on-demand compaction, `/diff` viewer (highlighting, folding, Commit Tree), artifact viewer with Kitty graphics, `/copy`, `/resume` grouping, `/skills reload`, and OSC 7501 terminal status (`AGY_CLI_OSC7501`).

**Read [resources/tui_reference.md](resources/tui_reference.md)** for keybindings, display modes, verbosity levels, and viewer details.

## Security, Sandbox & Permissions

### Sandbox Rules (`--sandbox`)
- **Read-Only `.git` Access (`v1.1.10+`)**: The terminal sandbox grants read-only access to `.git` metadata, protecting Git history from unauthorized rewriting.
- **System Temporary Directories (`v1.1.9+`)**: Automatic read and write access is granted to system temp directories (`/tmp`, `$TMPDIR`) without permission prompts.
- **Artifacts and Scratch Directories (`v1.2.10+`)**: Sandboxed commands have full read/write access to conversation artifact and scratch directories.
- **URL Reading (`v1.1.28+`)**: URL fetch tools default to prompting for confirmation before fetching external web pages.
- **macOS Application Support (`v1.3.3+`)**: `~/Library/Application Support` is blocked by default (apps keep credentials there); allow it explicitly in sandbox rules if needed.
- **Non-Interactive VCS Editors (`v1.3.6+`)**: Agent commands set `GIT_EDITOR`, `HGEDITOR`, `JJ_EDITOR` to `false` and `PAGER`, `GIT_PAGER`, `JJ_PAGER` to `cat`, so `git commit` without `-m` fails fast instead of hanging.

### Permission Allowlisting Rules
- **Subcommand Pinning**: Permission suggestions for script runners (`npm run <script>`, `go vet`, `repo status`, `jj config`) pin the specific subcommand so approving one task does not open access to arbitrary scripts (`v1.1.21+`, `v1.2.9+`, `v1.2.15+`).
- **Compound Shell Commands (`v1.1.8+`)**: Exact chained commands (e.g. `git fetch && git rebase`) can be saved as allow-always rules.
- **Rule Syntax**: Rules match exact command prefixes by default. Opt into regex pattern matching by prefixing the rule with `regex:`.

### Auto Mode (`--mode auto`, `v1.3.5+`)
Approves file edits and safe commands without asking; still prompts before irreversible actions. Commands that would leave the sandbox are reviewed by a separate agent first (`v1.3.6+`: allows the workspace's own format/lint/fix commands, blocks uploads of sensitive data).

```bash
# Launch a session in Auto mode
agy --mode auto -i "Refactor the auth package and fix any lint errors"

# Or switch inside the TUI: Shift+Tab cycles modes, or set Agent Mode in /config
```
*Availability:* macOS and Linux (Windows support is pending).

## Complete Environment Variables Reference

| Variable | Description | Introduced |
| :--- | :--- | :--- |
| `GEMINI_API_KEY` | Direct API key for Gemini API access without interactive sign-in (`modelProvider: "gemini"`). | v1.1.13+ |
| `GOOGLE_GEMINI_BASE_URL` | Custom endpoint URL when authenticating via `GEMINI_API_KEY`. | v1.1.13+ |
| `USE_ADC` | Set to `1` to authenticate via Application Default Credentials. | v1.0.11+ |
| `AGY_CLI_HIDE_ACCOUNT_INFO` | Set to `true` or `1` to hide user email and plan tier details from terminal headers. | v1.0.2+ |
| `AGY_CLI_HIDE_LOGO` | Set to `true` or `1` to suppress ASCII banner logo for screen readers or narrow viewports. | v1.1.19+ |
| `AGY_CLI_DISABLE_LATEX` | Set to `true` or `1` to disable terminal LaTeX math rendering globally. | v1.0.4+ |
| `CLI_GRAPHICS` | Set to `kitty` to force Kitty graphics protocol inside tmux or multiplexers. | v1.2.10+ |
| `AGY_CLI_DISABLE_ESCAPE_SEQUENCE_OPTIMIZATIONS` | Bypasses dirty-rectangle rendering optimizations for specialized terminal emulators. | v1.1.19+ |
| `AGY_CLI_CMD_OUTPUT_PERCENTAGE` | Configures max percentage height of command outputs in the TUI. | v1.0.11+ |
| `ANTIGRAVITY_CLI_ALIAS` | Overrides the detected binary executable name. | v1.0.0+ |
| `AGY_CLI_OSC7501` | `on` / `off` to force or disable OSC 7501 terminal status reporting. | v1.4.0+ |
| `AGY_CLI_OSC7501_MESSAGES` | Set to `brief` to show only the status, without pending command or question details. | v1.4.0+ |
| `AGY_ADC_AUTH` | ADC sign-in for Business accounts (Claude Opus 5.5 / Sonnet 5.5 available, telemetry off, Remote Control unsupported). | v1.3.3+, v1.3.6+, v1.4.1+ |

## Gemini CLI Flag & Feature Translation

| Gemini CLI | agy CLI v1.4.2 Equivalent | Notes |
| :--- | :--- | :--- |
| `gemini -p "prompt"` | `agy -p "prompt"` | Now auto-expands leading slash commands and skills. |
| `gemini --yolo` | `agy --dangerously-skip-permissions` | Complete tool auto-approval. |
| `gemini --resume <id>` | `agy --conversation <id>` | Resume session by ID. |
| `gemini -c` | `agy -c` | Continues most recent conversation with directory fallback. |
| `gemini -o stream-json` | `agy -p --output-format stream-json` | Strongly typed NDJSON event stream with token stats. |
| `N/A` | `agy -p --input-format stream-json` | Stdin NDJSON stream for multi-turn continuous runner. |
| `gemini -m <model>` | `agy --model <model>` | Explicit model selection. |
| `gemini --approval-mode plan` | `agy --mode plan` | Replaces `/planning` mode. |
| `gemini --approval-mode auto_edit` | `agy --mode accept-edits` or `agy --mode auto` | `auto` (v1.3.5+) also approves safe commands. |
| `gemini extensions list` | `agy plugin list` | Managed via plugin subsystem. |
| `gemini extensions install`| `agy plugin install <target>` | Supports marketplaces and GitHub subpaths. |
| `N/A` | `agy plugin import gemini` | One-click migration of Gemini extensions to plugins. |
| `N/A` | `agy mcp add\|list\|remove` | Built-in CLI management for MCP servers. |
| `N/A` | `agy remote-control start` | OS-registered background remote service daemon. |
| `N/A` | `agy plugin validate <path> --strict --json` | CI-ready plugin validation (v1.4.0+). |

## Troubleshooting & Failure Recovery

- **Headless Hangs (`-p`)**: Verify `--dangerously-skip-permissions` is passed. Without it, commands requiring authorization block silently on stdin.
- **Exit Code 3**: Inspect stderr for `AGY_ERROR: {...}` JSON. Common causes include quota exhaustion (`RESOURCE_EXHAUSTED`), model rate limits (`429`), or invalid model slugs.
- **Exit Code 1 on `--json-schema`**: Verify the root of the schema contains `"type": "object"`. Non-object schemas are rejected immediately.
- **Persistent Workspace State**: Remember that `agy` maintains internal workspace mappings. `cd`-ing into a directory does not automatically rebind the active project. Always pass `--add-dir .` or `--project <id|name>` when scoping is required.
- **Stale OAuth Credentials**: If enterprise or Google Cloud tokens expire or encounter clock skew, run `/logout` and `/login` to clear cached tokens.
- **Subagent Deadlocks**: Subagents respect permission boundaries. If a subagent lacks permissions in autonomous mode, it halts rather than attempting workarounds. Run with proper allowlisting in `settings.json` or pass `--dangerously-skip-permissions`.
- **Hung Model Requests**: A model connection with no data for 5 minutes is canceled and retried (`v1.3.6+`), with at most 30 seconds between retries (`v1.3.4+`). Socket disconnects (laptop sleep, Wi-Fi change) are retried too (`v1.4.1+`).
- **Plan Not Auto-Approved**: `/plan` waiting for review despite `--dangerously-skip-permissions` is intended since `v1.3.2`; set `Artifact Review` in `/config`.
- **`Eligibility check failed` at Launch**: Fixed in `v1.4.0`; run `agy update`.
- **Plugin Doesn't Load**: Run `agy plugin validate <path> --strict` for file and line level errors (`v1.4.0+`).
- **Plugin MCP Collisions**: If two plugins define conflicting MCP server names, the CLI automatically namespaces them as `<plugin>_<server>`. Reference them using their prefixed identifier.
