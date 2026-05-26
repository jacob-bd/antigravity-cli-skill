---
name: antigravity-cli
description: "Expert guide for Google's Antigravity CLI (agy), the official successor to Gemini CLI. Use when the user mentions 'agy', 'antigravity', 'antigravity cli', 'gemini cli replacement', 'gemini cli migration', or any task involving the agy command-line tool including running prompts, managing plugins, resuming sessions, or automating agy in scripts and CI/CD pipelines."
version: "1.1.2"
---

# Antigravity CLI (agy) Skill

Use this skill to work with the `agy` CLI for coding tasks, multi-agent orchestration, and workspace management.

## Context: Gemini CLI Successor

The `agy` CLI is Google's official replacement for Gemini CLI, announced at Google I/O on May 19, 2026. Gemini CLI stops serving requests for consumer and free users on **June 18, 2026**. Enterprise users on Gemini Code Assist Standard/Enterprise retain Gemini CLI access indefinitely.

Key differences from Gemini CLI:
- **Go binary** (not Node.js) -- faster cold startup, no npm dependency chain
- **Plugin system** replaces Gemini CLI Extensions (`agy plugin import gemini` to migrate)
- **Unified platform** -- shares the same agent harness as the Antigravity IDE desktop app
- **Binary name**: `agy` (primary), `antigravity` (some Linux distros). Env var `ANTIGRAVITY_CLI_ALIAS` overrides detection.
- **Install location**: typically `~/.local/bin/agy`

## Complete Flag Reference

### Primary Command: `agy`

| Flag | Alias | Description |
| :--- | :--- | :--- |
| `--print` | `-p`, `--prompt` | Runs a single prompt non-interactively and prints the response. |
| `--prompt-interactive`| `-i` | Runs an initial prompt interactively and continues the session. |
| `--continue` | `-c` | Continues the most recent conversation in the current workspace. |
| `--conversation <id>` | | Resumes a specific conversation by its ID. |
| `--dangerously-skip-permissions` | | **CRITICAL for automation.** Auto-approves all tool permission requests. |
| `--add-dir <path>` | | Adds a directory to the workspace (repeatable). |
| `--sandbox` | | Runs in a sandbox with terminal restrictions enabled. |
| `--print-timeout <duration>` | | Timeout for print mode (default: `5m0s`). Increase for long tasks. |
| `--log-file <path>` | | Overrides the default CLI log file path. |

## Known Limitations (v1.0.2)

> [!CAUTION]
> agy v1.0.2 is the current release. Several capabilities from Gemini CLI are **not yet available**. Do NOT attempt these flags -- they will fail.

| Missing Capability | Gemini CLI Equivalent | Status |
| :--- | :--- | :--- |
| JSON/NDJSON streaming output | `-o stream-json` | Not available |
| Model selection at spawn time | `-m <model>` | Not available |
| Reset workspace context | N/A | Not available (v1.0.2) |
| Yolo shorthand | `--yolo` | Use `--dangerously-skip-permissions` |
| Session resume by flag name | `--resume <id>` | Use `--conversation <id>` |
| Plan/approval mode | `--approval-mode plan` | Use `--sandbox` (partial equivalent) |
| MCP server control | `--allowed-mcp-server-names=` | Not available |
| Thinking/reasoning level control | `GEMINI_THINKING_LEVEL` env var | Not available |
| Config home isolation | `GEMINI_CLI_HOME` env var | Not available |

**What this means for automation:** Without NDJSON streaming, you cannot parse tool calls, thinking tokens, session IDs, or usage stats in real time. Print mode returns a single text blob when the agent finishes.

## Subcommands

### `plugin` (alias: `plugins`)
Manage the capabilities of your agent. Downloaded plugins are stored directly in `~/.gemini/config/` for instant discoverability.
- `agy plugin list`: See what's installed.
- `agy plugin install <target>`: Add new powers (e.g., `plugin@marketplace`).
- `agy plugin import gemini`: Migrate your Gemini CLI extensions to Antigravity plugins.
- `agy plugin import claude`: Import Claude Code extensions as plugins.
- `agy plugin enable/disable <name>`: Toggle specific functionality.
- `agy plugin uninstall <name>`: Remove a plugin.
- `agy plugin validate [path]`: Validate a plugin definition.
- `agy plugin link <mp> <target>`: Generate a link to a marketplace.

### `install`
Configure environment paths and shell settings.
- `agy install`: Set up PATH and shell aliases.
- `agy install --dir <path>`: Custom directory target for PATH configuration.
- `agy install --skip-path`: Skip shell profile PATH appending.
- `agy install --skip-aliases`: Skip shell profile alias purging.

### `update`
`agy update`: Update the CLI to the latest version.

### `changelog`
`agy changelog`: View version history and release notes.

## Agentic Workflows & Best Practices

### 1. The "Print Mode" Trap (`-p`)
> [!WARNING]
> Running `agy -p` is excellent for quick tasks, but it has critical caveats in an automated environment:
> - **Permissions**: You MUST use `--dangerously-skip-permissions` or the command will hang silently waiting for approval.
> - **First run**: agy may require initial interactive setup before print mode works. If `-p` hangs with zero output on a fresh install, run `agy` interactively first to complete auth/setup.
> - **Timeout**: Print mode has a default 5-minute timeout (`--print-timeout`). For long-running tasks, increase it: `--print-timeout 30m`.
> - **Output**: stdout may be buffered. Use `--log-file` if you need to track execution details.

### 2. Spawning Subagents
You can use `agy` to spawn other agents to handle sub-tasks.
```bash
agy -p "Review this code: $(cat main.py)" --dangerously-skip-permissions
```
By nesting CLI calls, you can create hierarchical agent structures.

### 3. Resuming Conversations
To maintain context across different execution steps:
1. Start with a prompt: `agy -i "Let's build a React app"`
2. Follow up later: `agy -c "Now add a login page"`
3. Resume a specific session: `agy --conversation <id>`

## Advanced Automation Patterns

### 1. Multi-turn Continuity (`-c` + `-p`)
You can chain non-interactive prompts by combining the continue flag (`-c`) with print mode (`-p`). This is the preferred way for agents to perform multi-step tasks without a TTY:

```bash
# Turn 1: Initial request
agy -p "Initialize a new project" --dangerously-skip-permissions

# Turn 2: Follow-up using context from Turn 1
agy -c -p "Now add a basic index.html" --dangerously-skip-permissions
```

### 2. Explicit Content Injection
To ensure the agent has the correct context (bypassing persistent workspace issues), inject file content directly into the prompt:

```bash
agy -p "$(cat README.md)\n\nBased on this file, what is the project goal?" --dangerously-skip-permissions
```

### 3. Targeted Workspace Addition
Use `--add-dir` to explicitly bring external directories into the current session context:

```bash
agy -p "Analyze this code" --add-dir ./src --dangerously-skip-permissions
```

## Workspace Management & Persistent State

> [!CAUTION]
> **Persistent Workspace State Warning**
> `agy` maintains its own persistent internal workspace context across sessions. It does **NOT** automatically scope itself to your shell's current working directory (CWD).
> - `cd`-ing into a directory will **not** change the agent's focus.
> - Running `agy -p` without explicit content may result in answers based on a previous, unrelated project.
> - **Silent Failure Mode**: If no explicit files are provided, `agy` will answer from the last session context without warning.

To manage this:
1. Use `--add-dir` to explicitly scope the session.
2. Use the "Explicit Content Injection" pattern for small files.
3. Be aware that v1.0.2 has no native command to "reset" or "clear" the workspace context.

### 4. Workspace Management
Because `agy` uses persistent state, do not rely on your shell's CWD. Always use `--add-dir` or explicit file injection to ensure the agent is working on the correct codebase.

### 5. Environment Variables & Permissions Configuration

#### Environment Variables
- `AGY_CLI_HIDE_ACCOUNT_INFO`: Set to `true` or `1` to hide user email and plan tier details from the terminal header, preserving privacy during screen shares or CI/CD logs.
- `ANTIGRAVITY_CLI_ALIAS`: Overrides automatic binary name detection.

#### Tool Permissions & Sandbox Mode
- **Sandbox Mode (`--sandbox`)**: Restricts terminal operations to a secure runtime environment.
- **Proceed-in-Sandbox Mode**: Added in `v1.0.1`. Automatically approves terminal commands that run inside the sandbox. Manual approval is requested only when a command attempts to bypass the sandbox, making automated non-interactive tasks much smoother.

### 6. Interactive Interface & Commands

#### `/statusline` Subcommand
The statusline command is fully case-insensitive and supports direct subcommand arguments:
- `/statusline help`: Shows help for configuring custom statuslines.
- `/statusline delete` / `/statusline reset`: Reverts to the default statusline.
- `/statusline enable` / `/statusline on`: Enables statusline rendering.
- `/statusline disable` / `/statusline off`: Disables statusline rendering.

#### Keyboard Shortcuts & UI Updates
- **Slash Commands Caret (`>`)**: All user slash commands and interactive shell inputs in message history are rendered with a caret prefix (`>`) to clearly distinguish them from agent-generated output.
- **Improved Shortcuts**: The `/help` shortcuts tab sorts all keybindings by their primary key.
- **New Keybindings**: Additional built-in shortcuts include:
  - `ctrl+r`: Reload / Search history
  - `ctrl+o`: Open file/url
  - `alt+j` / `ctrl+k`: UI focus and navigation overrides
- **Scrolling Shortcuts**: General scrolling (`PageUp`/`PageDown`/`GoToTop`/`GoToBottom`) is fully supported across both Commands and Shortcuts tabs.
- **Session deletion**: In the `/resume` screen, deleting a conversation is bound to `ctrl+delete` (changed from `ctrl+d` to avoid terminal exit conflicts).

### 7. Migrating from Gemini CLI
If you previously used Gemini CLI:
1. Import your extensions: `agy plugin import gemini`
2. Note flag differences (see Known Limitations table above)
3. Replace `gemini -p` with `agy -p` in scripts
4. Replace `--yolo` with `--dangerously-skip-permissions`
5. Replace `--resume <id>` with `--conversation <id>`

## Gemini CLI Flag Mapping

Quick reference for translating Gemini CLI commands to agy:

| Gemini CLI | agy Equivalent | Notes |
| :--- | :--- | :--- |
| `gemini -p "prompt"` | `agy -p "prompt"` | Same semantics |
| `gemini --yolo` | `agy --dangerously-skip-permissions` | Longer but same effect |
| `gemini --resume <id>` | `agy --conversation <id>` | Different flag name |
| `gemini -o stream-json` | N/A | Not available in v1.0.0 |
| `gemini -m <model>` | N/A | Not available in v1.0.0 |
| `gemini --approval-mode plan` | `agy --sandbox` | Partial equivalent |

## Troubleshooting

- **Hangs/Timeouts (print mode)**: Usually caused by missing `--dangerously-skip-permissions`. Can also indicate first-run setup is needed -- run `agy` interactively once to initialize.
- **Hangs/Timeouts (long tasks)**: Increase `--print-timeout` beyond the default 5 minutes.
- **Permission Denied**: Check if `--sandbox` is restricting the operation, or if the OS requires manual approval.
- **Lost Context**: Use `agy --conversation <id>` to recover a specific session state.
- **Extensions missing after migration**: Run `agy plugin import gemini` to port Gemini CLI extensions.
- **Binary not found**: Check `~/.local/bin/agy` or run `agy install` to configure PATH. On some Linux distros, the binary is named `antigravity`.
