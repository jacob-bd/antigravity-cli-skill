# agy CLI Quick Reference

## All Flags
| Flag | Short | What it does |
| :--- | :--- | :--- |
| `--print` | `-p` | Non-interactive single prompt |
| `--output-format <fmt>` | | Output format (`text` (default), `json`, `stream-json`) (v1.1.8+) |
| `--json-schema <schema>` | | Enforce JSON schema string or file on output (v1.1.8+) |
| `--effort <level>` | | Reasoning effort (`low`, `medium`, `high`) (v1.1.5+) |
| `--prompt-interactive` | `-i` | Interactive mode with initial prompt |
| `--continue` | `-c` | Continue last conversation |
| `--conversation <id>` | | Resume specific session |
| `--dangerously-skip-permissions` | | Auto-approve everything |
| `--sandbox` | | Restricted execution mode |
| `--agent <agent>` | | Agent for the current CLI session |
| `--model <model>` | | Model for the current CLI session |
| `--mode <mode>` | | Set agent execution mode (`accept-edits`, `plan`) |
| `--print-timeout <dur>` | | Print mode timeout (default: 5m) |
| `--add-dir <path>` | | Add directory to workspace |
| `--log-file <path>` | | Override log file location |
| `--project <id>` | | Explicitly set project ID |
| `--new-project` | | Create a new project for this session |

## Key Commands
- `agy update` -- Update to latest version
- `agy changelog` -- See what's new
- `agy install` -- Configure PATH and shell
- `agy agents` / `agy agent` -- List available agents
- `agy models` -- List available models for CLI sessions
- `agy plugin list` -- View installed plugins
- `agy plugin install <target>` -- Add a plugin (supports `plugin@marketplace`, GitHub subpaths `owner/repo/subpath@branch`, and auto-resolves Git submodules in `v1.0.9+`)
- `agy plugin import gemini` -- Migrate Gemini CLI extensions
- `agy plugin import claude` -- Import Claude extensions
- `agy plugin enable/disable <name>` -- Toggle a plugin

## Gemini CLI Migration Cheat Sheet
| Gemini CLI | agy |
| :--- | :--- |
| `gemini -p "prompt"` | `agy -p "prompt"` |
| `--yolo` | `--dangerously-skip-permissions` |
| `--resume <id>` | `--conversation <id>` |
| `-o stream-json` | `agy -p --output-format stream-json` (v1.1.8+) |
| `-m <model>` | `agy --model <model>` (v1.0.5+) |
| `--approval-mode plan` | `--mode plan` (v1.1.0+) |

## Environment Variables, Sandbox & Settings
- `AGY_CLI_HIDE_ACCOUNT_INFO=true` -- Hides email & plan tier from header logs.
- `AGY_CLI_DISABLE_LATEX=true` -- Disables LaTeX math rendering globally (v1.0.4+).
- `AGY_CLI_CMD_OUTPUT_PERCENTAGE` -- Set max height of command outputs in TUI (v1.0.11+).
- `USE_ADC=1` -- Authenticate via Application Default Credentials (v1.0.11+).
- **Proceed-in-Sandbox mode** (v1.0.1+) -- Automatically approves secure commands inside the sandbox.
- **Hardened Sandbox Checks** (v1.0.9+) -- Enforces exact-match verification for PowerShell, redirects, and unparseable commands. Adds `.git` to dangerous paths.
- **Optimized Customizations Permissions** (v1.0.9+) -- Automatically grants read-only access to built-in customizations, avoiding startup prompts.
- **Permission & Flag Fixes** (v1.0.10+) -- Escapes regex metacharacters in saved rules, fixes environment flag parsing, ensures "ask" permissions in `settings.json` are preserved across configuration writes, and resolves bash mode argument escaping (defaulting shell resolution to PowerShell).
- **Project Specific Permissions** (v1.0.12+) -- Project config (`~/.gemini/config/projects/`) takes precedence over global settings.
- **Strict Permission Rule Matching** (v1.1.0+) -- Exact prefix matching by default, regex requires `regex:` prefix.
- **Relaxed Redirection Checks** (v1.1.0+) -- Allows stdout redirection commands without separate approval rules.
- **Workspace URI Verification** (v1.1.0+) -- Resolves false warnings on valid in-workspace file operations.
- **Compound Command Permissions** (v1.1.8+) -- Chained shell commands (e.g. `git fetch && git rebase`) can be saved as allow-always rules.
- **System Temp Read Access** (v1.1.6+) -- Default read permissions granted for system temporary directory.
- **`copyOnSelect` Setting** (v1.1.8+) -- Toggle auto-copying mouse text selection to clipboard in `/settings`.
- **UseG1Credits setting** (v1.0.3+) -- Toggles automatic use of G1 credits when standard quota is depleted.
- **SQLite Storage** (v1.0.4+) -- Centralized SQLite database format (.db, .db-wal) for conversation history.
- **MCP Launch Timeout** (v1.0.7+) -- Configure launch timeout in `mcp_config.json` per server block, or set to `-1` to disable.
- **Wayland Clipboard** (v1.0.7+) -- Native `wl-paste` support on Linux, falling back to `xclip` on X11.
- **Preserved Settings** (v1.0.7+) -- Unknown fields in `settings.json` are preserved across read/write/merge.

## Key Subcommands & TUI Panels
- `/codesearch` (aliases `/cs`, `/search`) (v1.1.3+) -- Interactively search code with regex, literal (`-F`), or path globs (`f:`).
- `/effort` (v1.1.5+) -- View and set reasoning effort (`low`, `medium`, `high`) with timeline gauge.
- `/copy <n>` (v1.1.6+) -- Copy the n-th most recent response to clipboard.
- Stacked slash commands (v1.1.4+) -- Prefix prompt with multiple slash commands (e.g. `/plan /grill-me <prompt>`).
- Markdown Custom Agents (`agent.md`) (v1.1.6+) -- Define custom agents using Markdown with YAML frontmatter.
- `/statusline help` / `delete` / `reset` / `enable` / `disable` -- Manage custom statusline.
- `/statusline` setting `stack_with_default: true` (v1.0.6+) -- Renders default and custom status lines stacked vertically.
- `/credits` -- Opens G1 credits balance info panel and purchase link.
- `/permissions` -- Interactive panel to view and modify permission rules directly (v1.0.5+).
- `/open` and `/add-dir` (v1.0.6+) -- Support shell-style path auto-completion in TUI.
- **Fuzzy Autocomplete** -- Fuzzy command matching (e.g., `/el` suggests `/help` and `/model`) (v1.0.6+).
- **Optimistic Rendering** -- User prompts render instantly to minimize perceived lag (v1.0.6+).
- **AltScreen Tool Confirmations** (v1.0.11+) -- Expanded view (`ctrl+g`) for tool confirmations. On artifact view, `ctrl+g` opens `$EDITOR`.
- **Execution Modes** (v1.1.0+) -- Cycle modes (`default`, `accept-edits`, `plan`) in TUI via `shift+tab` or persist via `Agent Mode` in `/settings`.

## Automation Patterns
- **Structured Streaming**: `agy -p "..." --output-format stream-json --dangerously-skip-permissions`
- **Multi-turn**: `agy -p "..."` then `agy -c -p "..."`
- **Injection**: `agy -p "$(cat file) \n\n prompt"`
- **Add Dir**: `agy -p "..." --add-dir ./path`

## Critical Reminders
- **Workspace Scoping**: `agy` uses persistent state. **CWD DOES NOT SCOPE THE SESSION**. Use `--add-dir` or explicit injection.
- **Tool calls limit**: Gemini models support up to 512 tool calls per turn (v1.0.7+).
- **Subagents Filtering**: Subagent sessions are filtered from the `/resume` list (v1.0.6+).
- Always pair `-p` with `--dangerously-skip-permissions` in scripts.
- First run may require interactive setup before `-p` works.
- Default print timeout is 5 minutes -- increase with `--print-timeout` for long tasks.
- Stream JSON with `--output-format stream-json` for typed NDJSON events, tool call info, and token accounting in scripts.
- Run `agy --help` to check for newly added flags after updates.

