# Antigravity CLI (agy) Skill

Use this skill to leverage the power of the `agy` CLI for coding tasks, multi-agent orchestration, and workspace management.

## Overview

The `agy` CLI is the primary interface for the Antigravity agentic system. It allows you to start conversations, run non-interactive prompts, manage plugins, and orchestrate complex workflows.

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
| `--log-file <path>` | | Overrides the default CLI log file path. |

## Subcommands

### `plugin` (alias: `plugins`)
Manage the capabilities of your agent.
- `agy plugin list`: See what's installed.
- `agy plugin install <target>`: Add new powers (e.g., `plugin@marketplace`).
- `agy plugin import`: Import from Gemini or Claude.
- `agy plugin enable/disable <name>`: Toggle specific functionality.

### `update`
`agy update`: Keep the system at the cutting edge.

### `changelog`
`agy changelog`: View version history and new features.

## Agentic Workflows & Best Practices

### 1. The "Print Mode" Trap (`-p`)
> [!WARNING]
> Running `agy -p` is excellent for quick tasks, but it has "disabilities" in an automated environment:
> - **Permissions**: You MUST use `--dangerously-skip-permissions` or the command will hang silently.
> - **Output**: Output capture can be tricky. Use `--log-file` if you need to track execution details, but be aware that standard stdout might be buffered or handled by the system's own logging.

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

### 4. Workspace Management
Always ensure you are in the correct directory. Use `--add-dir` if you need to bring in external context.

## Troubleshooting

- **Hangs/Timeouts**: Usually caused by missing `--dangerously-skip-permissions`.
- **Permission Denied**: Check if `--sandbox` is restrictive or if the OS requires manual approval for certain commands.
- **Lost Context**: Use `agy --conversation <id>` to recover a specific state.

---
*Created for Bro by Antigravity*
