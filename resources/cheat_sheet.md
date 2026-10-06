# agy CLI Quick Reference (v1.3.0)

## All Command-Line Flags
| Flag | Short | What it does | Added / Updated |
| :--- | :--- | :--- | :--- |
| `--print` | `-p`, `--prompt` | Non-interactive single prompt execution | v1.0.0+ |
| `--input-format <fmt>` | | Input format (`text`, `stream-json` for continuous stdin streaming) | v1.1.15+ |
| `--output-format <fmt>`| | Output format (`text` (default), `json`, `stream-json` NDJSON events) | v1.1.8+ |
| `--json-schema <schema>`| | Enforce JSON schema (inline string or file; requires `"type": "object"` root) | v1.1.8+, v1.2.14+ |
| `--effort <level>` | | Reasoning effort (`low`, `medium`, `high`, `xhigh`, `max`) | v1.1.5+, v1.2.11+ |
| `--disable-slash-commands`| | Opt out of automatic slash-command and skill expansion in `-p` mode | v1.1.9+ |
| `--prompt-interactive`| `-i` | Interactive mode with initial prompt | v1.0.0+ |
| `--continue` | `-c` | Continue most recent conversation (with parent/child dir fallback) | v1.0.0+, v1.2.1+ |
| `--conversation <id>` | | Resume specific session by ID | v1.0.0+ |
| `--dangerously-skip-permissions` | | Auto-approve all tool permission prompts (crucial for headless scripts) | v1.0.0+ |
| `--sandbox` | | Restricted terminal execution mode | v1.0.1+ |
| `--agent <agent>` | | Agent for current session (Markdown or built-in) | v1.1.1+ |
| `--model <model>` | | Model slug or tier for current session | v1.0.5+, v1.1.22+ |
| `--mode <mode>` | | Set agent execution mode (`default`, `accept-edits`, `plan`) | v1.1.0+ |
| `--print-timeout <dur>`| | Print mode timeout (default: `0s` / unlimited; mid-turn returns partial) | v1.1.28+, v1.2.6+ |
| `--add-dir <path>` | | Add directory to workspace (repeatable) | v1.0.0+ |
| `--project <id\|name>` | | Explicitly set project ID or project name | v1.0.12+, v1.1.18+ |
| `--new-project` | | Create a new isolated project context | v1.0.12+ |
| `--remote-control` | | Start session with active Remote Control tunnel | v1.2.6+ |
| `--log-file <path>` | | Override CLI log file path | v1.0.0+ |

## Key Subcommands
- `agy mcp list` -- List all configured MCP servers and status
- `agy mcp add [flags] <name> <cmd|url> [args...]` -- Add stdio or HTTP MCP server (`--type`, `--env`, `--header`)
- `agy mcp enable/disable <name>` -- Toggle MCP server enablement
- `agy mcp remove <name>` -- Delete an MCP server configuration
- `agy remote-control start` -- Start OS-registered background remote service daemon (`--name`, `--session`)
- `agy remote-control status` -- View background daemon status, active tunnel, and PID
- `agy remote-control stop` -- Stop and unregister background remote service
- `agy mic-serve` -- Serve local microphone over loopback/SSH (`--addr`) for `/voice`
- `agy agent` / `agy agents` -- List available agents (supports `--output-format json|stream-json`)
- `agy models` -- List available models (supports `--output-format json|stream-json`)
- `agy plugin list` -- View installed plugins
- `agy plugin install <target>` -- Install plugin (`name@marketplace` or `owner/repo/subpath@branch`)
- `agy plugin import gemini` -- Migrate legacy Gemini CLI extensions
- `agy plugin import claude` -- Import Claude extensions
- `agy plugin enable/disable <name>` -- Toggle plugin enablement
- `agy update` -- Update to the latest CLI release
- `agy changelog` -- View release notes
- `agy install` -- Configure PATH and shell aliases (`--dir`, `--skip-path`, `--skip-aliases`)

## Gemini CLI Migration Cheat Sheet
| Gemini CLI | agy CLI v1.3.0 | Notes |
| :--- | :--- | :--- |
| `gemini -p "prompt"` | `agy -p "prompt"` | Auto-expands slash commands and skills in v1.1.9+ |
| `gemini --yolo` | `agy --dangerously-skip-permissions` | Complete tool permission auto-approval |
| `gemini --resume <id>` | `agy --conversation <id>` | Resumes conversation by ID |
| `gemini -c` | `agy -c` | Continues most recent conversation |
| `gemini -o stream-json` | `agy -p --output-format stream-json` | Strongly typed NDJSON event stream with token stats |
| `N/A` | `agy -p --input-format stream-json` | Stdin NDJSON stream for multi-turn continuous runner |
| `gemini -m <model>` | `agy --model <model>` | Specify model slug |
| `gemini --approval-mode plan` | `agy --mode plan` | Auto-approved in headless print mode |
| `gemini extensions list` | `agy plugin list` | Managed via plugin subsystem |
| `gemini extensions install`| `agy plugin install <target>` | Supports marketplaces and GitHub subpaths |
| `N/A` | `agy plugin import gemini` | Automatic migration of Gemini extensions |

## Environment Variables
- `GEMINI_API_KEY` -- Direct API key authentication without interactive login (`modelProvider: "gemini"`).
- `GOOGLE_GEMINI_BASE_URL` -- Custom endpoint URL for Gemini API requests.
- `USE_ADC=1` -- Authenticate via Application Default Credentials.
- `AGY_CLI_HIDE_ACCOUNT_INFO=true` -- Hide user email and plan tier details from terminal header.
- `AGY_CLI_HIDE_LOGO=true` -- Hide ASCII banner art in header for screen readers or narrow viewports.
- `AGY_CLI_DISABLE_LATEX=true` -- Globally disable LaTeX math rendering in terminal.
- `CLI_GRAPHICS=kitty` -- Force Kitty graphics mode inside tmux or multiplexers.
- `AGY_CLI_DISABLE_ESCAPE_SEQUENCE_OPTIMIZATIONS=1` -- Bypass dirty-rectangle renderer optimizations.
- `AGY_CLI_CMD_OUTPUT_PERCENTAGE` -- Set max height of command outputs in TUI.
- `ANTIGRAVITY_CLI_ALIAS` -- Override automatic binary name detection.

## TUI Panels, Shortcuts & Keybindings
- **Modal Vim Editing**: Enable in `/settings` -> `Editor Mode` -> `Vim`. Supports counts (`3dw`, `2di(`), Normal/Insert/Visual/Visual Line modes, and `vim.*` keybindings.
- `/model <name>`: Switch active default model.
- `/model <name> <prompt>`: Execute a single turn on `<name>` and automatically revert back to original model.
- `/effort <level>`: Adjust reasoning effort (`low`, `medium`, `high`, `xhigh`, `max`) with timeline-gauge.
- `/codesearch` (aliases `/cs`, `/search`): Regex/literal (`-F`) workspace search with path filters (`f:`).
- `/copy`: Copy most recent response. `/copy <n>` copies n-th response. `/copy btw` copies `/btw` response.
- `/voice` or `F5`: Start/stop voice speech-to-text dictation.
- `/skills reload`: Hot-reload skills and slash commands without restarting.
- `/permissions`: Interactive TUI panel to manage permission allowlists.
- `/resume`: Resume picker. `Ctrl+F` toggles directory grouping (`pickerGrouping` setting).
- Artifact Viewer: `m` cycles display mode (Image, ASCII fallback, Raw source); `t` opens outline; `Left`/`Right` pans diagrams; `Ctrl+D`/`Ctrl+U` half-page scrolls; `Ctrl+G` opens in `$EDITOR`.
- `/config`: Use `←` / `→` arrow keys to change highlighted settings immediately without opening dropdowns.

## Automation Patterns & Exit Codes
- **Continuous Stdin Streaming**: `agy -p --input-format stream-json --output-format stream-json --dangerously-skip-permissions`
- **Headless Skill Execution**: `agy -p "/my-skill test this feature" --dangerously-skip-permissions`
- **Zero-Turn Stats**: `agy -p "/quota" --output-format json` (no quota spent, no agent turn started)
- **Subagent Direct Prompting**: `@<subagent-name> <task instruction>`
- **Exit Code 0**: Clean success (or graceful partial return on timeout expiration).
- **Exit Code 1**: Invalid flags or invalid non-object `--json-schema`.
- **Exit Code 3**: Fatal agent/model API error with structured `AGY_ERROR: {...}` line on stderr.
