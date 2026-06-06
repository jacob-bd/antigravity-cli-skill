# agy CLI Quick Reference

## All Flags
| Flag | Short | What it does |
| :--- | :--- | :--- |
| `--print` | `-p` | Non-interactive single prompt |
| `--prompt-interactive` | `-i` | Interactive mode with initial prompt |
| `--continue` | `-c` | Continue last conversation |
| `--conversation <id>` | | Resume specific session |
| `--dangerously-skip-permissions` | | Auto-approve everything |
| `--sandbox` | | Restricted execution mode |
| `--model <model>` | | Model for the current CLI session |
| `--print-timeout <dur>` | | Print mode timeout (default: 5m) |
| `--add-dir <path>` | | Add directory to workspace |
| `--log-file <path>` | | Override log file location |

## Key Commands
- `agy update` -- Update to latest version
- `agy changelog` -- See what's new
- `agy install` -- Configure PATH and shell
- `agy models` -- List available models for CLI sessions
- `agy plugin list` -- View installed plugins
- `agy plugin install <target>` -- Add a plugin
- `agy plugin import gemini` -- Migrate Gemini CLI extensions
- `agy plugin import claude` -- Import Claude extensions
- `agy plugin enable/disable <name>` -- Toggle a plugin

## Gemini CLI Migration Cheat Sheet
| Gemini CLI | agy |
| :--- | :--- |
| `gemini -p "prompt"` | `agy -p "prompt"` |
| `--yolo` | `--dangerously-skip-permissions` |
| `--resume <id>` | `--conversation <id>` |
| `-o stream-json` | Not available (v1.0.6) |
| `-m <model>` | `agy --model <model>` (v1.0.5+) |
| `--approval-mode plan` | `--sandbox` |

## Environment Variables, Sandbox & Settings
- `AGY_CLI_HIDE_ACCOUNT_INFO=true` -- Hides email & plan tier from header logs.
- `AGY_CLI_DISABLE_LATEX=true` -- Disables LaTeX math rendering globally (v1.0.4+).
- **Proceed-in-Sandbox mode** (v1.0.1+) -- Automatically approves secure commands inside the sandbox.
- **UseG1Credits setting** (v1.0.3+) -- Toggles automatic use of G1 credits when standard quota is depleted.
- **SQLite Storage** (v1.0.4+) -- Centralized SQLite database format (.db, .db-wal) for conversation history.

## Key Subcommands & TUI Panels
- `/statusline help` -- Show help configuration.
- `/statusline delete` / `reset` -- Reset to default.
- `/statusline enable` / `on` -- Enable statusline.
- `/statusline disable` / `off` -- Disable statusline.
- `/credits` -- Opens G1 credits balance info panel and purchase link.
- `/permissions` -- Interactive panel to view and modify permission rules directly (v1.0.5+).
- **Path Auto-Completion** -- Shell-style path autocomplete for `/open` and `/add-dir` (v1.0.6+).
- **Fuzzy Autocomplete** -- Fuzzy command matching (e.g., `/el` suggests `/help` and `/model`) (v1.0.6+).
- **Optimistic Rendering** -- User prompts render instantly to minimize perceived lag (v1.0.6+).

## Automation Patterns
- **Multi-turn**: `agy -p "..."` then `agy -c -p "..."`
- **Injection**: `agy -p "$(cat file) \n\n prompt"`
- **Add Dir**: `agy -p "..." --add-dir ./path`

## Critical Reminders
- **Workspace Scoping**: `agy` uses persistent state. **CWD DOES NOT SCOPE THE SESSION**. Use `--add-dir` or explicit injection.
- Always pair `-p` with `--dangerously-skip-permissions` in scripts
- First run may require interactive setup before `-p` works
- Default print timeout is 5 minutes -- increase with `--print-timeout` for long tasks
- No NDJSON streaming output in v1.0.6 -- you get plain text only
- Run `agy --help` to check for newly added flags after updates
