---
name: antigravity-cli
description: "Expert guide for Google's Antigravity CLI (agy), the official successor to Gemini CLI. Use when the user mentions 'agy', 'antigravity', 'antigravity cli', 'gemini cli replacement', 'gemini cli migration', or any task involving the agy command-line tool including running prompts, managing plugins, resuming sessions, or automating agy in scripts and CI/CD pipelines."
---

# Antigravity CLI (agy) Skill

Targets locally installed `agy` v1.0.10.

Use this skill to work with the `agy` CLI for coding tasks, multi-agent orchestration, and workspace management.

## Context: Gemini CLI Successor

The `agy` CLI is Google's official replacement for Gemini CLI, announced at Google I/O on May 19, 2026. Gemini CLI stops serving requests for consumer and free users on **June 18, 2026**. Enterprise users on Gemini Code Assist Standard/Enterprise retain Gemini CLI access indefinitely.

Key differences from Gemini CLI:
- **Go binary** (not Node.js) -- faster cold startup, no npm dependency chain
- **Plugin system** replaces Gemini CLI Extensions (`agy plugin import gemini` to migrate)
- **Unified platform** -- shares the same agent harness as the Antigravity IDE desktop app
- **Tool calls limit**: Supports up to **512 tool calls** per turn for Gemini models, allowing for highly complex, multi-step agentic tasks.
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
| `--model <model>` | | Specifies the model to use for the current CLI session. |
| `--print-timeout <duration>` | | Timeout for print mode (default: `5m0s`). Increase for long tasks. |
| `--log-file <path>` | | Overrides the default CLI log file path. |

## Known Limitations (verified with installed v1.0.9)

> [!CAUTION]
> This skill is verified against locally installed agy v1.0.10. Several capabilities from Gemini CLI are **not yet available**. Do NOT attempt these flags -- they will fail.

| Missing Capability | Gemini CLI Equivalent | Status |
| :--- | :--- | :--- |
| JSON/NDJSON streaming output | `-o stream-json` | Not available |
| Reset workspace context | N/A | Not available (verified v1.0.10) |
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
- `agy plugin install <target>`: Add new powers (e.g., `plugin@marketplace`). Also supports installing directly from GitHub subpaths with branch resolution (e.g., `owner/repo/subpath@branch`). External plugin installation automatically resolves and initializes Git submodules (`v1.0.9+`).
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

### `models`
List available models for CLI sessions.
- `agy models`: Display a list of available model names.

### `update`
`agy update`: Update the CLI to the latest version.

### `changelog`
`agy changelog`: View version history and release notes.

## Agentic Workflows & Best Practices

### The "Print Mode" Trap (`-p`)
> [!WARNING]
> Running `agy -p` is excellent for quick tasks, but it has critical caveats in an automated environment:
> - **Permissions**: You MUST use `--dangerously-skip-permissions` or the command will hang silently waiting for approval.
> - **First run**: agy may require initial interactive setup before print mode works. If `-p` hangs with zero output on a fresh install, run `agy` interactively first to complete auth/setup.
> - **Timeout**: Print mode has a default 5-minute timeout (`--print-timeout`). For long-running tasks, increase it: `--print-timeout 30m`.
> - **Output**: stdout may be buffered. Use `--log-file` if you need to track execution details.

### Spawning Subagents
You can use `agy` to spawn other agents to handle sub-tasks.
```bash
agy -p "Review this code: $(cat main.py)" --dangerously-skip-permissions
```
By nesting CLI calls, you can create hierarchical agent structures.

### Resuming Conversations
To maintain context across different execution steps:
1. Start with a prompt: `agy -i "Let's build a React app"`
2. Follow up later: `agy -c "Now add a login page"`
3. Resume a specific session: `agy --conversation <id>`

## Advanced Automation Patterns

### Multi-turn Continuity (`-c` + `-p`)
You can chain non-interactive prompts by combining the continue flag (`-c`) with print mode (`-p`). This is the preferred way for agents to perform multi-step tasks without a TTY:

```bash
# Turn 1: Initial request
agy -p "Initialize a new project" --dangerously-skip-permissions

# Turn 2: Follow-up using context from Turn 1
agy -c -p "Now add a basic index.html" --dangerously-skip-permissions
```

> [!NOTE]
> As of `v1.0.9+`, resuming in headless print mode (`-c`/`-p` or `--conversation`/`-p`) correctly prints *only* the newly generated response rather than dumping the entire historical conversation transcript.

### Explicit Content Injection
To ensure the agent has the correct context (bypassing persistent workspace issues), inject file content directly into the prompt:

```bash
agy -p "$(cat README.md)\n\nBased on this file, what is the project goal?" --dangerously-skip-permissions
```

### Targeted Workspace Addition
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
3. Be aware that v1.0.10 has no native command to "reset" or "clear" the workspace context.

### Environment Variables & Permissions Configuration

#### Environment Variables
- `AGY_CLI_HIDE_ACCOUNT_INFO`: Set to `true` or `1` to hide user email and plan tier details from the terminal header, preserving privacy during screen shares or CI/CD logs.
- `ANTIGRAVITY_CLI_ALIAS`: Overrides automatic binary name detection.
- `AGY_CLI_DISABLE_LATEX`: Set to `true` or `1` to globally disable LaTeX math rendering in the terminal viewport (added in `v1.0.4`).

#### Tool Permissions & Sandbox Mode
- **Sandbox Mode (`--sandbox`)**: Restricts terminal operations to a secure runtime environment. In `v1.0.6+`, `--sandbox` propagation is fixed in headless print mode (`-p` / `--print`), ensuring sandbox isolation is correctly enforced during non-interactive execution.
- **Proceed-in-Sandbox Mode**: Automatically approves terminal commands that run inside the secure sandbox. Manual approval is requested only when a command attempts to bypass the sandbox, making automated non-interactive tasks much smoother.
- **Hardened Sandbox Checks (`v1.0.9+`)**: Enforces strict exact-match verification for PowerShell scripts, complex shell redirections (`>`, `2>&1`), and unparseable strings. Additionally, the `.git` directory is added to the core list of dangerous paths to prevent unauthorized repository modifications.
- **Optimized Customizations Permissions (`v1.0.9+`)**: Automatically grants read-only access to the built-in customizations directory, eliminating redundant permission prompts on startup.
- **Permission & Flag Fixes (`v1.0.10+`)**: Escapes regex metacharacters in saved permission rules to prevent infinite loops, fixes environment flag parsing, ensures "ask" permissions in `settings.json` are preserved across configuration writes, and resolves bash mode argument escaping (defaulting shell resolution to PowerShell).
- **Interactive Permissions Configuration (`/permissions`)**: Added in `v1.0.5`. Allows users to add, edit, or remove permission rules directly inside the TUI. Supports configuring permissions for workspace levels, shared settings, and CLI-specific configuration settings.
- **Integrated Permissions System**: Integrates CLI permissioning with the rest of the Antigravity system, merging project-level permissions, shared user settings, and CLI-specific configuration rules.
- **MCP Config & Launch Options**: 
  - **MCP URL Support**: Configure MCP servers using URLs inside `mcp_config.json`.
  - **Configurable Launch Timeout**: A configurable timeout for launching MCP servers is supported in `v1.0.7+`. Specify a custom duration or set it to `-1` to disable the timeout completely (placed within the server definition block in `mcp_config.json`).
  - **Preserving Unknown Settings**: Unknown fields inside `settings.json` are preserved during read, write, and merge operations, preventing configurations from being silently wiped when upgrading/downgrading.
  - **Clipboard Support**: Linux now has native Wayland clipboard support via `wl-paste`, falling back to `xclip` on X11, prioritizing copied files over raw image data. clipboard image/file reading has been fixed for Windows and Wayland-only Linux distributions, and clipboard size verification has been added to prevent OOM errors on large clipboard files (`v1.0.8+`).

### Interactive Interface & Commands

#### `/statusline` Subcommand
The statusline command is fully case-insensitive and supports direct subcommand arguments:
- `/statusline help`: Shows help for configuring custom statuslines.
- `/statusline delete` / `/statusline reset`: Reverts to the default statusline.
- `/statusline enable` / `/statusline on`: Enables statusline rendering.
- `/statusline disable` / `/statusline off`: Disables statusline rendering.
- **Statusline Optimization**: In `v1.0.5`, statusline layout is improved by merging active tips and artifact statuses on a single line, with truncation to prevent collisions on narrow screens.
- **Statusline Stacking**: In `v1.0.6+`, a `stack_with_default` flag can be set in the `statusLine` configuration to render both the default status line and custom status line output stacked vertically.
- **Quota and Mode Info**: In `v1.0.8+`, quota usage and execution mode are rendered in the status line.

#### G1 Credits & `/credits` Panel
Version `v1.0.3` adds full support for G1 credits:
- **Automatic Credit Usage**: When standard model quota runs out, the CLI can automatically utilize G1 credits.
- **`UseG1Credits` Setting**: A new TUI setting enables/disables automatic G1 credit usage.
- **Real-Time Display**: Remaining G1 credits are displayed in real-time in the terminal's status bar.
- **`/credits` Panel**: Open the in-CLI credits panel to view credit balance details and access a direct link to purchase additional G1 credits.

#### Keyboard Shortcuts & UI Updates
- **Slash Commands Caret (`>`)**: All user slash commands and interactive shell inputs in message history are rendered with a caret prefix (`>`) to clearly distinguish them from agent-generated output.
- **Slash Command History (`v1.0.8+`)**: Up arrow key in the TUI prompt editor replays previously entered slash commands.
- **Paste Guard (`v1.0.8+`)**: A per-line guard replaces extremely long single-line TUI pastes with an expandable placeholder to prevent performance lag.
- **Improved Shortcuts**: The `/help` shortcuts tab sorts all keybindings by their primary key.
- **New Keybindings**: Additional built-in shortcuts include:
  - `ctrl+r`: Reload / Search history. As of `v1.0.5`, you can open this Artifact Review panel while answering pending questions or tool permission confirmations, preserving your current progress when toggling back.
  - `ctrl+o`: Open file/url
  - `alt+j` / `ctrl+k`: UI focus and navigation overrides
- **Scrolling Shortcuts**: General scrolling (`PageUp`/`PageDown`/`GoToTop`/`GoToBottom`) is fully supported across both Commands and Shortcuts tabs.
- **Session deletion**: In the `/resume` screen, deleting a conversation is bound to `ctrl+delete` (changed from `ctrl+d` to avoid terminal exit conflicts).

#### LaTeX Math Rendering
Added in `v1.0.4`. Renders beautiful LaTeX mathematical formulas directly in the terminal viewport. You can disable this by setting the `AGY_CLI_DISABLE_LATEX` environment variable.

#### SQLite Conversation Storage & /resume Performance
As of `v1.0.4`, the CLI uses SQLite (`.db` and `.db-wal`) as its default conversation storage format. Performance of `/resume` is highly optimized in `v1.0.5` through lazy loading conversation details, filtering of empty conversations, and fast scanning of SQLite database files.
- **Subagent Filtering**: Subagent conversations are automatically skipped from `/resume` to keep the picker focused solely on direct user-initiated conversations (v1.0.6+).
- **Archival Timestamp**: Conversation archiving now correctly saves the archival status timestamp (v1.0.7+).
- **Redesigned Picker (`v1.0.8+`)**: `/resume` conversation picker aligned workspace columns and added adaptive column dropping (workspace, time, steps) for narrow terminals.

#### Centralized Project Discovery
Decoupled in `v1.0.4` from local workspace directories. Workspace-to-project mappings are centralized in `~/.gemini/antigravity-cli/cache/projects.json` for clutter-free repositories and single-map lookups.

#### Autocomplete Alias Resolution
As of `v1.0.5`, tab completion for slash commands resolves to the matched alias (e.g., `/se` autocompletes to `/settings` instead of the primary `/config` command).

#### TUI & Autocomplete Enhancements
- **Path Auto-Completion**: Added in `v1.0.6`. Supports shell-style path auto-completion when typing path arguments for the `/open` and `/add-dir` commands in the interactive TUI.
- **Fuzzy Slash Command Suggestions**: Added in `v1.0.6`. Autocompletion for slash commands now supports fuzzy and partial substring matching (e.g., typing `/el` suggests `/help` and `/model`). Autocomplete prefix bugs (like `/conv` vs `/conv-switch`) are resolved (`v1.0.8+`).
- **Unconditional `@` Mentions Typeahead**: Suggestions trigger whenever `@` is typed without preceding whitespace (e.g., after opening parenthesis `(`).
- **Optimistic Rendering**: Added in `v1.0.6`. Implements optimistic rendering for user chat prompts, immediately displaying the user's prompt in the viewport to reduce perceived input lag.
- **Esc Key Stream Interrupt**: Entering a prompt immediately after pressing `Esc` (to interrupt stream) now accepts input without swallowing/rejecting it.
- **Artifact Viewer Improvements**: Gutter numbering and line mapping in the artifact viewer are revamped in `v1.0.7+` to accurately align viewport lines with 1-based source line numbers, correctly handling wrapped lines and collapsed Mermaid diagrams. Layout boundary overflow and scrolling bugs in the detail view with inline comments are resolved. Gutter layout rendering complexity during long sessions is also optimized to prevent hangs (`v1.0.8+`).
- **Dynamic Skill & Command Autocomplete**: Custom skills and system slash commands are dynamically reloaded and instantly discovered upon conversation switch or `/add-dir` (`v1.0.8+`).
- **Builtin Guide Skill (`v1.0.10+`)**: Includes the `antigravity_guide` builtin skill to provide instant, in-context reference guides for Antigravity 2.0, CLI, IDE, and SDK.
- **Glamour Parsing Error Handling**: Graceful fallback to raw text with a warning banner on bubbletea parsing errors (e.g. nested checkboxes inside list emphasis), preventing TUI crashes (`v1.0.9+`). Renders cleaner headings and block padding with upgraded Glamour v2.0.1 (`v1.0.10+`).
- **TUI & Git Enhancements (`v1.0.10+`)**: Scrolling in commit history navigation immediately loads/displays changed files and diffs; ASCII node graphs (`git log --graph`) are enabled for visual parity; short (6-char) commit hashes are matched to long (64-char) hashes; system errors/warnings use a dedicated alert message type; and CLI log file path is accessible in the `/help` menu.

#### Models & Quota Page (`v1.0.8+`)
- **Redesigned Interface**: Enabled by default, replacing the legacy usage page. It handles disabled quota buckets by displaying a dimmed "Disabled" status and omitting progress bars.
- **Settings Inheritance**: CLI inherits the `use_ai_credits` setting from global user settings on startup.
- **Transient Statusline Errors**: Propagates configuration write failures as transient error flashes on the TUI status line.
- **Hook Configurations**: `/hooks` command now correctly writes to the shared configuration directory (`~/.gemini/config/hooks.json`).

#### `/tasks` and `/btw` Updates (`v1.0.8+`)
- **`/tasks` Redesign**: Redesigned list and detail views with start times on the left, right-aligned status, and capped panel height for better readability.
- **`/btw` Optimization**: Improved token efficiency, streaming responses, and fixed premature truncation.

### Migrating from Gemini CLI
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
| gemini -p "prompt" | `agy -p "prompt"` | Same semantics |
| gemini --yolo | `agy --dangerously-skip-permissions` | Longer but same effect |
| gemini --resume <id> | `agy --conversation <id>` | Different flag name |
| gemini -o stream-json | N/A | Not available in verified v1.0.10 |
| gemini -m <model> | `agy --model <model>` | Supported in v1.0.5+ |
| gemini --approval-mode plan | `agy --sandbox` | Partial equivalent |

## Troubleshooting

- **Hangs/Timeouts (print mode)**: Usually caused by missing `--dangerously-skip-permissions`. Can also indicate first-run setup is needed -- run `agy` interactively once to initialize.
- **Hangs/Timeouts (long tasks)**: Increase `--print-timeout` beyond the default 5 minutes.
- **Permission Denied**: Check if `--sandbox` is restricting the operation, or if the OS requires manual approval.
- **Lost Context**: Use `agy --conversation <id>` to recover a specific session state.
- **Extensions missing after migration**: Run `agy plugin import gemini` to port Gemini CLI extensions.
- **Binary not found**: Check `~/.local/bin/agy` or run `agy install` to configure PATH. On some Linux distros, the binary is named `antigravity`.
