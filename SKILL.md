---
name: antigravity-cli
description: "Expert guide for Google's Antigravity CLI (agy), the official successor to Gemini CLI. Use when the user mentions 'agy', 'antigravity', 'antigravity cli', 'gemini cli replacement', 'gemini cli migration', or any task involving the agy command-line tool including running prompts, managing plugins, resuming sessions, or automating agy in scripts and CI/CD pipelines."
version: 1.3.0
---

# Antigravity CLI (agy) Skill

Targets locally installed `agy` v1.3.0 (with complete feature coverage through v1.2.17).

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
| `--dangerously-skip-permissions` | | **CRITICAL FOR AUTOMATION.** Auto-approves all tool permission requests without interactive prompts. | v1.0.0+ |
| `--sandbox` | | Runs terminal operations in a secure restricted sandbox. | v1.0.1+ |
| `--agent <agent>` | | Specifies a custom agent (Markdown or built-in) for the current CLI session. | v1.1.1+, v1.2.11+ |
| `--model <model>` | | Model slug or identifier for the current CLI session. | v1.0.5+, v1.1.22+ |
| `--mode <mode>` | | Sets agent execution mode: `default` (request review), `accept-edits`, or `plan`. | v1.1.0+ |
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

# Machine-readable output for programmatic inspection
agy agent --output-format json
agy agent --output-format stream-json
```

### 5. `agy models`
List available models for the current sign-in credential or API key.
```bash
# Human-readable list
agy models

# Machine-readable output for tooling
agy models --output-format json
```

### 6. `agy plugin` / `agy plugins`
Manage agent plugins (stored in `~/.gemini/config/plugins/`):
```bash
# List installed plugins
agy plugin list

# Install from marketplace or GitHub repo subpaths with branch resolution
agy plugin install security-auditor@marketplace
agy plugin install github.com/owner/repo/plugins/analyzer@main

# Migrate legacy Gemini CLI extensions or Claude extensions
agy plugin import gemini
agy plugin import claude

# Enable or disable plugins (enablement state persists in config.json)
agy plugin enable <name>
agy plugin disable <name>

# Validate plugin definition manifest
agy plugin validate ./my-plugin
```

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

**Output protocol (strongly-typed NDJSON events on stdout):**
- `init`: session metadata, active model, and tool declarations.
- `step_update`: incremental text deltas, `tool_info` (tool name, args, output), and `subagent_info` (`conversation_id`, `log_uri`).
- `result`: final answer, token accounting (`cache_read_tokens`, `prompt_tokens`, `candidates_tokens`), and status.

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
- **`1`**: Startup configuration errors, invalid command flags, malformed `--json-schema`.
- **`3`**: Fatal agent or model API error (`v1.2.6+`, `v1.2.10+`). When exit code 3 is returned, the CLI outputs a structured JSON error line on stderr:
  ```text
  AGY_ERROR: {"canonical_status": "RESOURCE_EXHAUSTED", "error_code": 429, "retryable": false, "error_id": "err_quota_exceeded"}
  ```

### 6. Non-Interactive Autonomous Decision Making
In headless `-p` runs:
- Implementation plan reviews (`--mode plan`) are approved automatically without blocking (`v1.1.28+`).
- User confirmation questions (`ask_question`) auto-settle choices autonomously rather than hanging (`v1.1.12+`).
- Subagent permission denials are handled strictly without attempting unauthorized workarounds (`v1.2.15+`).

## Custom Markdown Agents (`agent.md`)

Antigravity CLI uses Markdown files with YAML frontmatter to define custom primary agents and specialized subagents (`v1.1.6+` through `v1.2.5+`).

### Storage Locations
- **Project Agents**: `.agents/agents/<name>.md` (automatically discovered in active workspace).
- **Global Agents**: `~/.gemini/config/agents/<name>.md`.

### Complete Agent Schema
```markdown
---
name: backend-architect
description: Senior backend systems engineer specialized in Go and distributed systems
mainAgent: true
subagent: true
hidden: false
model: pro
commandExecutionPolicy: ask
inheritCustomizations: true
excludeDefaultComponents: false
rules:
  - .agents/rules/go-standards.md
agents:
  - database-specialist
  - test-engineer
---

# Backend Architect System Prompt
You are a Staff Software Engineer. Focus on high-throughput architecture, clean concurrency models, and resilient error recovery.
```

### Frontmatter Fields Reference
- `name` *(string)*: Unique identifier.
- `description` *(string)*: Human-readable description shown in `/agents`.
- `mainAgent` *(boolean)*: If `true`, agent can be launched directly via `agy --agent <name>`.
- `subagent` *(boolean)*: If `true`, agent can be delegated to by other agents.
- `hidden` *(boolean)*: If `true`, hides the agent from `/agents` picker menus.
- `model` *(string)*: Model tier (`flash`, `pro`, `inherit`) or explicit model slug.
- `commandExecutionPolicy` *(string)*: Tool execution policy (`ask`, `allow`).
- `inheritCustomizations` *(boolean)*: Master switch to inherit ambient skills, rules, and plugins (`v1.1.14+`).
- `excludeDefaultComponents` *(boolean)*: Opts out of default prompt sections and baseline tools (`v1.2.1+`).
- `rules` *(array of strings)*: Explicit rule file paths applied unconditionally (`v1.1.15+`).
- `agents` *(array of strings)*: Explicit subagent dependencies declared for this agent (`v1.1.27+`).

### Subagent Orchestration & Messaging
- **Direct Subagent Messaging Syntax (`v1.2.9+`)**: In the prompt, type `@<subagent-name> <instruction>` to dispatch tasks directly to a specific subagent with shell-style autocomplete.
- **Roster Auto-Injection (`v1.2.5+`)**: Any custom agent declaring `invoke_subagent` in its tools automatically receives the full subagent catalog and usage instructions in its system prompt.
- **Built-in Image Generator (`v1.2.16+`)**: The agent automatically delegates visual asset creation to the built-in `image-generator` subagent with up to 3 automatic verification iterations.
- **Isolated Worktrees (`v1.2.10+`)**: Subagent Git operations are isolated in `worktrees/` inside the application data directory to prevent workspace collision.

## Customization System & Token Budgeting

### Dedicated 20,000-Token Rule Budget (`v1.2.7+`)
User and workspace rules (`GEMINI.md`, `AGENTS.md`, `.agents/rules/*.md`) are allocated a dedicated 20,000-token budget. Rules are truncated cleanly along newline boundaries, ensuring large rule sets **never evict skills, workflows, subagents, or MCP tools**.

### Manifest Directory Scanning Rules (`v1.2.10+`)
Directory paths in `skills.json`, `rules.json`, `agents.json`, and `plugins.json` load only items located directly inside the directory (matching `.agents/skills/` behavior). To load nested sub-items, explicitly specify them in `include_only`:
```json
{
  "path": "shared_skills",
  "include_only": ["security/vuln-scanner", "compliance/license-check"]
}
```

### Hierarchical Configuration Loading (`v1.2.16+`)
Manifests (`skills.json`, `rules.json`, etc.) are discovered and merged across every `.agents/` folder between the current working directory and the project Git root.

### Skill Metadata & Hot Reloading
- **Skill Visual Branding (`v1.1.20+`)**: Add `metadata.icon` (e.g. `icon: 🛡️`) in `SKILL.md` frontmatter to render icons across `/skills` and autocomplete popups.
- **Silent Skills (`v1.1.12+`)**: Add `disable-slash-command: true` in `SKILL.md` frontmatter to keep a skill model-invocable without cluttering the interactive `/` slash command menu.
- **Live Reload (`v1.2.4+`)**: Run `/skills reload` to reload all skills and commands dynamically without restarting the CLI session.

## Interactive Interface, Keybindings & Navigation

### 1. Modal Vim Editing Mode
Added in `v1.1.11+` and expanded in `v1.2.9+` and `v1.2.12+`. Configure in `/settings` under `Editor Mode` -> `Vim`:
- **Modes**: Normal, Insert, Visual, Visual Line. Mode status badge rendered in status line.
- **Motions & Counts**: Full support for count multipliers (e.g. `3dw`, `2d3w`, `3dd`, `3x`, `[count]G`, `2di(`).
- **Submissions**: Submit directly from Normal mode via `Ctrl+S` or `Ctrl+Enter`.
- **Insert First Option**: Opens prompt in Insert mode where bare `Enter` submits and `Shift+Enter` / `Ctrl+J` inserts newlines.
- **Keybinding scopes**: Full customization in `keybindings.json` under `vim.*`.

### 2. One-Shot Model Switching (`/model`)
Added in `v1.1.22+` and `v1.1.27+`:
- Switch default model: `/model <name>`.
- **One-shot execution**: `/model <name> <prompt>` executes a single turn using `<name>` and automatically reverts back to the original model on the subsequent turn.

### 3. Settings Fast Navigation (`/config` & `/settings`)
- As of `v1.2.16+`, press `←` / `→` arrow keys on any highlighted setting to cycle values immediately without opening dropdown menus.
- **Queued Messages (`v1.2.14+`)**: Configure `"queuedMessages": "queue"` (default) or `"send-immediately"` to interrupt the active turn when follow-up messages are entered.
- **Verbosity Modes (`v1.2.10+`)**:
  - `high`: Full tool calls, parameter payloads, and thoughts.
  - `medium`: Groups related tool calls into concise summaries (e.g. `Explored 14 files`) while preserving command outputs.
  - `low`: Minimal output with token metric tables.

### 4. Artifact Detail Viewer
- **Inline Kitty Graphics (`v1.2.7+`)**: Renders LaTeX formulas (`$$...$$`, fenced `latex`/`math`) and Mermaid diagrams directly in the terminal viewport on Kitty-compatible terminals.
- **Display Modes (`m`)**: Press `m` in the artifact viewer to cycle between Image, Unicode ASCII fallback, and Raw source.
- **Diagram Panning**: Use `Left` / `Right` arrow keys to horizontally pan oversized diagrams.
- **Outline Navigation (`t`)**: Press `t` to open a table-of-contents outline and jump directly to sections (`v1.1.12+`).
- **Half-page scrolling**: Use `Ctrl+D` and `Ctrl+U` (`v1.1.26+`).
- **External Editor**: Press `Ctrl+G` to open the artifact in `$EDITOR`.

### 5. Response & Content Clipboard (`/copy`)
- `/copy`: Copies the most recent response.
- `/copy <n>`: Copies the n-th most recent response (`v1.1.6+`).
- `/copy btw`: Copies the active `/btw` side-question response (`v1.2.3+`).

### 6. Workspace Grouping in `/resume`
Toggle between a flat list and conversations grouped by directory using `Ctrl+F` (`v1.1.25+`). Persist the default with `"pickerGrouping": "workspace"` in `settings.json`.

## Security, Sandbox & Permissions

### Sandbox Rules (`--sandbox`)
- **Read-Only `.git` Access (`v1.1.10+`)**: The terminal sandbox grants read-only access to `.git` metadata, protecting Git history from unauthorized rewriting.
- **System Temporary Directories (`v1.1.9+`)**: Automatic read and write access is granted to system temp directories (`/tmp`, `$TMPDIR`) without permission prompts.
- **Artifacts and Scratch Directories (`v1.2.10+`)**: Sandboxed commands have full read/write access to conversation artifact and scratch directories.
- **URL Reading (`v1.1.28+`)**: URL fetch tools default to prompting for confirmation before fetching external web pages.

### Permission Allowlisting Rules
- **Subcommand Pinning**: Permission suggestions for script runners (`npm run <script>`, `go vet`, `repo status`, `jj config`) pin the specific subcommand so approving one task does not open access to arbitrary scripts (`v1.1.21+`, `v1.2.9+`, `v1.2.15+`).
- **Compound Shell Commands (`v1.1.8+`)**: Exact chained commands (e.g. `git fetch && git rebase`) can be saved as allow-always rules.
- **Rule Syntax**: Rules match exact command prefixes by default. Opt into regex pattern matching by prefixing the rule with `regex:`.

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

## Gemini CLI Flag & Feature Translation

| Gemini CLI | agy CLI v1.3.0 Equivalent | Notes |
| :--- | :--- | :--- |
| `gemini -p "prompt"` | `agy -p "prompt"` | Now auto-expands leading slash commands and skills. |
| `gemini --yolo` | `agy --dangerously-skip-permissions` | Complete tool auto-approval. |
| `gemini --resume <id>` | `agy --conversation <id>` | Resume session by ID. |
| `gemini -c` | `agy -c` | Continues most recent conversation with directory fallback. |
| `gemini -o stream-json` | `agy -p --output-format stream-json` | Strongly typed NDJSON event stream with token stats. |
| `N/A` | `agy -p --input-format stream-json` | Stdin NDJSON stream for multi-turn continuous runner. |
| `gemini -m <model>` | `agy --model <model>` | Explicit model selection. |
| `gemini --approval-mode plan` | `agy --mode plan` | Replaces `/planning` mode. Auto-approved in headless `-p`. |
| `gemini extensions list` | `agy plugin list` | Managed via plugin subsystem. |
| `gemini extensions install`| `agy plugin install <target>` | Supports marketplaces and GitHub subpaths. |
| `N/A` | `agy plugin import gemini` | One-click migration of Gemini extensions to plugins. |
| `N/A` | `agy mcp add\|list\|remove` | Built-in CLI management for MCP servers. |
| `N/A` | `agy remote-control start` | OS-registered background remote service daemon. |

## Troubleshooting & Failure Recovery

- **Headless Hangs (`-p`)**: Verify `--dangerously-skip-permissions` is passed. Without it, commands requiring authorization block silently on stdin.
- **Exit Code 3**: Inspect stderr for `AGY_ERROR: {...}` JSON. Common causes include quota exhaustion (`RESOURCE_EXHAUSTED`), model rate limits (`429`), or invalid model slugs.
- **Exit Code 1 on `--json-schema`**: Verify the root of the schema contains `"type": "object"`. Non-object schemas are rejected immediately.
- **Persistent Workspace State**: Remember that `agy` maintains internal workspace mappings. `cd`-ing into a directory does not automatically rebind the active project. Always pass `--add-dir .` or `--project <id|name>` when scoping is required.
- **Stale OAuth Credentials**: If enterprise or Google Cloud tokens expire or encounter clock skew, run `/logout` and `/login` to clear cached tokens.
- **Subagent Deadlocks**: Subagents respect permission boundaries. If a subagent lacks permissions in autonomous mode, it halts rather than attempting workarounds. Run with proper allowlisting in `settings.json` or pass `--dangerously-skip-permissions`.
- **Plugin MCP Collisions**: If two plugins define conflicting MCP server names, the CLI automatically namespaces them as `<plugin>_<server>`. Reference them using their prefixed identifier.
