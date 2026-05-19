# Antigravity CLI (agy) Skill

Use this skill to leverage the power of the `agy` CLI for coding tasks, multi-agent orchestration, and workspace management.

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

## Known Limitations (v1.0.0)

> [!CAUTION]
> agy v1.0.0 is the initial release. Several capabilities from Gemini CLI are **not yet available**. Do NOT attempt these flags -- they will fail.

| Missing Capability | Gemini CLI Equivalent | Status |
| :--- | :--- | :--- |
| JSON/NDJSON streaming output | `-o stream-json` | Not available |
| Model selection at spawn time | `-m <model>` | Not available |
| Yolo shorthand | `--yolo` | Use `--dangerously-skip-permissions` |
| Session resume by flag name | `--resume <id>` | Use `--conversation <id>` |
| Plan/approval mode | `--approval-mode plan` | Use `--sandbox` (partial equivalent) |
| MCP server control | `--allowed-mcp-server-names=` | Not available |
| Thinking/reasoning level control | `GEMINI_THINKING_LEVEL` env var | Not available |
| Config home isolation | `GEMINI_CLI_HOME` env var | Not available |

**What this means for automation:** Without NDJSON streaming, you cannot parse tool calls, thinking tokens, session IDs, or usage stats in real time. Print mode returns a single text blob when the agent finishes.

## Subcommands

### `plugin` (alias: `plugins`)
Manage the capabilities of your agent.
- `agy plugin list`: See what's installed.
- `agy plugin install <target>`: Add new powers (e.g., `plugin@marketplace`).
- `agy plugin import gemini`: Migrate your Gemini CLI extensions to Antigravity plugins.
- `agy plugin import claude`: Import Claude Code extensions as plugins.
- `agy plugin enable/disable <name>`: Toggle specific functionality.
- `agy plugin uninstall <name>`: Remove a plugin.
- `agy plugin validate [path]`: Validate a plugin definition.

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

### 4. Workspace Management
Always ensure you are in the correct directory. Use `--add-dir` if you need to bring in external context from other directories.

### 5. Migrating from Gemini CLI
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
