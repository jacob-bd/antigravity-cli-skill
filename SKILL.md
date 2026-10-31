---
name: antigravity-cli
description: "Expert guide for Google's Antigravity CLI (agy), the official successor to Gemini CLI. Use when the user mentions 'agy', 'antigravity', 'antigravity cli', 'gemini cli replacement', 'gemini cli migration', or any task involving the agy command-line tool including running prompts, managing plugins, resuming sessions, or automating agy in scripts and CI/CD pipelines."
version: 1.4.12
---

# Antigravity CLI (agy) Skill

Targets `agy` v1.4.12.

Use this skill to guide AI coding agents, automate CI/CD pipelines, orchestrate multi-agent swarms, configure Model Context Protocol (MCP) servers, manage custom Markdown agents, and control workspace environments with the `agy` CLI.

## Context: Gemini CLI Successor

The `agy` CLI is Google's official replacement for Gemini CLI. It is a compiled Go binary (instant startup, zero npm dependencies, embedded `ripgrep`), sharing the agent harness and runtime with the Antigravity IDE desktop application. It supports up to **512 tool calls** per turn on Gemini models for deep multi-step agentic execution. Executable is typically at `~/.local/bin/agy` (override with `ANTIGRAVITY_CLI_ALIAS`).

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
- MCP `tools/call` requests include `_meta["antigravity.google/root_conversation_id"]` to correlate subagent calls back to the parent conversation (`v1.4.4+`).
- On-demand MCP tool results up to 10 KB return directly inline without temporary file spills (`v1.4.7+`).
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
Forwards a local client microphone to an `agy` session running on a remote host (e.g. over SSH) for speech dictation: `agy mic-serve --addr 127.0.0.1:4713` (`v1.1.21+`).

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
- `agy plugin list --json` (`v1.3.4+`): lists every installed plugin with version and enabled state.
- `agy plugin validate` (`v1.4.0+`–`v1.4.9+`): validates plugins with the same rules the loader uses, checking `plugin.json` (syntax, fields, kebab-case `name`, `author`/`homepage`), skills, subagents, rules, `hooks.json`, sidecars, and context providers. Use `--strict --json` in CI.
- `agy plugin install` (`v1.4.8+`): prints loaded skills, agents, rules, MCP servers, and hooks upon installation.
- `agy plugin import claude` (`v1.4.11+`): imports Claude Code marketplace plugins, remote HTTP MCP servers, and `.mcp.json`.
- Plugin `auth.json` supports `"optional": true` for API key providers to allow unauthenticated free-tier usage (`v1.4.12+`).
- Inside the TUI, `/plugin` (`v1.3.2+`) opens the interactive manager.

### 7. `agy install`
Configures environment PATH and shell aliases: `agy install [--dir <path>] [--skip-path] [--skip-aliases]`.

### 8. `agy update` & `agy changelog`
Update binary (`agy update`) or view release notes (`agy changelog` or `agy -p "/changelog"`).

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
- **`1`**: Startup configuration errors, invalid flags, malformed `--json-schema`, unknown `--agent` name in `-p` mode (`v1.3.4+`), or prompt blocked by hook (`v1.4.11+`).
- **`2`**: Print-mode usage errors, such as combining `--print` with `--prompt-interactive` (`v1.3.4+`).
- **`3`**: Fatal agent or model API error (`v1.2.6+`, `v1.2.10+`). When exit code 3 is returned, the CLI outputs a structured JSON error line on stderr:
  ```text
  AGY_ERROR: {"canonical_status": "RESOURCE_EXHAUSTED", "error_code": 429, "retryable": false, "error_id": "err_quota_exceeded"}
  ```

*Completion Detection (`v1.4.11+`):* Results in `--output-format json` and `stream-json` include `finished_reason` (`print_timeout`, `interrupted`, `error`) when a turn terminates early. The field is omitted entirely on clean, complete runs.

### 6. Non-Interactive Autonomous Decision Making
In headless `-p` runs:
- Implementation plan reviews (`--mode plan`) settle without blocking (`v1.1.28+`). For unattended interactive plan approval, set `Artifact Review` in `/config` (`--dangerously-skip-permissions` no longer implies it since `v1.3.2`).
- User confirmation questions (`ask_question`) auto-settle choices autonomously rather than hanging (`v1.1.12+`).
- Subagent permission denials are handled strictly without attempting unauthorized workarounds (`v1.2.15+`).
- Permission grants apply across the main agent and all subagents (`v1.4.6+`).
- Questions asked by subagents and workflow agents are skipped in `-p` runs so the subagent can continue (`v1.3.6+`).
- Context compaction in subagents preserves the parent agent's 10 most recent messages (`v1.4.10+`).

## Delegating Work to agy (Host CLI Integration)

Coding assistants (Claude Code, OpenAI Codex, OpenCode, etc.) can delegate plan reviews, code reviews, and coding tasks to `agy` using headless print mode (`-p`).

Key mechanics:
- Always pass `--dangerously-skip-permissions` to avoid hanging on interactive prompts.
- Always scope the target directory with `--add-dir .` (or target path).
- Use `--mode plan` for read-only plan/architecture reviews.
- Omit `--mode plan` for code-writing tasks (agy will edit workspace files directly).
- Capture structured outputs using `--output-format json` (result in `.response`).
- Iterate on the same task using `--conversation <id>` with delta instructions.
- A fired `--print-timeout` still exits `0` with `status: SUCCESS` and partial output: inspect `finished_reason` in JSON (`v1.4.11+`) or grep stderr for `print timeout`, then request final synthesis on the same `--conversation <id>`.

**Read [resources/delegation_playbook.md](resources/delegation_playbook.md)** for complete delegation recipes, background execution patterns, error handling, and MCP/skill integration.

## Custom Agents & Customization System

Custom agents are Markdown files with YAML frontmatter in `.agents/agents/<name>.md` (project) or `~/.gemini/config/agents/<name>.md` (global), launched with `agy --agent <name>` or delegated to as subagents. Rules (`GEMINI.md`, `AGENTS.md`, `.agents/rules/*.md`) get a dedicated 20,000-token budget so they never evict skills, subagents, or MCP tools. Manifests (`skills.json`, `rules.json`, `agents.json`, `plugins.json`) are discovered across every `.agents/` folder up to the Git root. Plugins can ship skills, rules, MCP servers, hooks, and subagents via a root `agents.json`. Hooks (`hooks.json`) support `lastUserInput` inspection (`v1.4.7+`) and `terminationBehavior: "terminate"` to gate turns before model calls (`v1.4.11+`).

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
- **Sensitive Credentials (`v1.4.6+`)**: Sandboxed commands cannot read or modify `~/.docker`, `~/.kube`, or `~/_netrc` without explicit approval.
- **Google Domain Isolation (`v1.4.9+`)**: Approving general Google sites does not permit access to sensitive apps (`mail.google.com`, `drive.google.com`, `console.cloud.google.com`).
- **Non-Interactive VCS Editors (`v1.3.6+`)**: Agent commands set `GIT_EDITOR`, `HGEDITOR`, `JJ_EDITOR` to `false` and `PAGER`, `GIT_PAGER`, `JJ_PAGER` to `cat`, so `git commit` without `-m` fails fast instead of hanging.

### Permission Allowlisting Rules
- **Subcommand Pinning**: Permission suggestions for script runners (`npm run <script>`, `go vet`, `repo status`, `jj config`) pin the specific subcommand so approving one task does not open access to arbitrary scripts (`v1.1.21+`, `v1.2.9+`, `v1.2.15+`).
- **Compound Shell Commands (`v1.1.8+`)**: Exact chained commands (e.g. `git fetch && git rebase`) can be saved as allow-always rules. Multi-line script rules apply cleanly (`v1.4.9+`).
- **Rule Syntax**: Rules match exact command prefixes by default. Opt into regex pattern matching by prefixing the rule with `regex:`.

### Auto Mode (`--mode auto`, `v1.3.5+`)
Approves file edits and safe commands without asking; still prompts before irreversible actions. Commands that would leave the sandbox are reviewed by a separate agent first (`v1.3.6+`: allows the workspace's own format/lint/fix commands, blocks uploads of sensitive data; `v1.4.7+`–`v1.4.9+`: prompts less often for routine tasks).

```bash
# Launch a session in Auto mode
agy --mode auto -i "Refactor the auth package and fix any lint errors"
```
*Availability:* macOS, Linux, and Windows 11 version 24H2+ (`v1.4.4+`).

## Complete Environment Variables Reference

| Variable | Description | Introduced |
| :--- | :--- | :--- |
| `GEMINI_API_KEY` | Direct API key for Gemini API access without interactive sign-in (`modelProvider: "gemini"`). | v1.1.13+ |
| `GOOGLE_GEMINI_BASE_URL` | Custom endpoint URL when authenticating via `GEMINI_API_KEY`. | v1.1.13+ |
| `USE_ADC` | Set to `1` to authenticate via Application Default Credentials. | v1.0.11+ |
| `AGY_ADC_AUTH` | ADC sign-in for Business accounts (Claude Opus/Sonnet 5.5 available; loads skills, rules, plugins, hooks correctly in v1.4.7+). | v1.3.3+, v1.4.7+ |
| `ANTIGRAVITY_CLI_ALIAS` | Overrides the detected binary executable name. | v1.0.0+ |
| `CLI_GRAPHICS` | Set to `kitty` to force Kitty graphics protocol inside tmux or multiplexers. | v1.2.10+ |
| `AGY_CLI_OSC7501` | `on` / `off` to force or disable OSC 7501 terminal status reporting (`AGY_CLI_OSC7501_MESSAGES=brief` for compact). | v1.4.0+ |
| `AGY_CLI_HIDE_ACCOUNT_INFO` | Set to `true` to hide user email/plan in terminal header. Also: `AGY_CLI_HIDE_LOGO`, `AGY_CLI_DISABLE_LATEX`. | v1.0.2+ |

## Gemini CLI Flag Translation

| Gemini CLI | agy CLI Equivalent | Notes |
| :--- | :--- | :--- |
| `gemini -p "prompt"` | `agy -p "prompt"` | Auto-expands leading slash commands and skills |
| `gemini --yolo` | `agy --dangerously-skip-permissions` | Complete tool auto-approval |
| `gemini --resume <id>` / `-c` | `agy --conversation <id>` / `agy -c` | Resumes session / continues recent |
| `gemini -o stream-json` | `agy -p --output-format stream-json` | NDJSON event stream with usage stats |
| `gemini -m <model>` | `agy --model <model>` | Explicit model selection |
| `gemini --approval-mode plan` | `agy --mode plan` | Plan mode; check `finished_reason` in output |
| `gemini --approval-mode auto_edit` | `agy --mode accept-edits` / `--mode auto` | `auto` also approves safe commands |
| `gemini extensions list\|install` | `agy plugin list\|install` | Managed via plugin subsystem (`agy plugin import gemini`) |

See [resources/cheat_sheet.md](resources/cheat_sheet.md) for full migration comparison.

## Troubleshooting & Failure Recovery

- **Headless Hangs (`-p`)**: Verify `--dangerously-skip-permissions` is passed. Without it, commands requiring authorization block silently on stdin.
- **Exit Code 3**: Inspect stderr for `AGY_ERROR: {...}` JSON. Common causes include quota exhaustion (`RESOURCE_EXHAUSTED`), model rate limits (`429`), or invalid model slugs.
- **Exit Code 1 on `--json-schema`**: Verify the root of the schema contains `"type": "object"`. Non-object schemas are rejected immediately.
- **Persistent Workspace State**: Remember that `agy` maintains internal workspace mappings. `cd`-ing into a directory does not automatically rebind the active project. Always pass `--add-dir .` or `--project <id|name>` when scoping is required.
- **Stale OAuth Credentials**: If enterprise or Google Cloud tokens expire or encounter clock skew, run `/logout` and `/login` to clear cached tokens.
- **Subagent Deadlocks**: Subagents respect permission boundaries. If a subagent lacks permissions in autonomous mode, it halts rather than attempting workarounds. Run with proper allowlisting in `settings.json` or pass `--dangerously-skip-permissions`.
- **Hung Model Requests**: A model connection with no data for 5 minutes is canceled and retried (`v1.3.6+`), with at most 30 seconds between retries (`v1.3.4+`). Socket disconnects (laptop sleep, Wi-Fi change) are retried too (`v1.4.1+`).
- **Plan Not Auto-Approved**: `/plan` waiting for review despite `--dangerously-skip-permissions` is intended since `v1.3.2`; set `Artifact Review` in `/config`.
- **Incomplete Headless Runs**: In `v1.4.11+`, check `finished_reason` in `--output-format json` results. A run returning exit code `0` with `finished_reason: "print_timeout"` is partial. Resume on the same session via `--conversation <id>`.
- **`Eligibility check failed` at Launch**: Fixed in `v1.4.0`; run `agy update`.
- **Plugin Doesn't Load**: Run `agy plugin validate <path> --strict` for file and line level errors (`v1.4.0+`).
- **Plugin MCP Collisions**: If two plugins define conflicting MCP server names, the CLI automatically namespaces them as `<plugin>_<server>`. Reference them using their prefixed identifier.
