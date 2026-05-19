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
| `--print-timeout <dur>` | | Print mode timeout (default: 5m) |
| `--add-dir <path>` | | Add directory to workspace |
| `--log-file <path>` | | Override log file location |

## Key Commands
- `agy update` -- Update to latest version
- `agy changelog` -- See what's new
- `agy install` -- Configure PATH and shell
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
| `-o stream-json` | Not available (v1.0.0) |
| `-m <model>` | Not available (v1.0.0) |
| `--approval-mode plan` | `--sandbox` |

## Critical Reminders
- Always pair `-p` with `--dangerously-skip-permissions` in scripts
- First run may require interactive setup before `-p` works
- Default print timeout is 5 minutes -- increase with `--print-timeout` for long tasks
- No NDJSON streaming output in v1.0.0 -- you get plain text only
- Run `agy --help` to check for newly added flags after updates
