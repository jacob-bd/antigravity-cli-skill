# agy Interactive Interface (TUI) Reference (v1.4.2)

Detailed reference moved out of `SKILL.md` to keep it under agy's 24 KB skill body limit.

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
- **Opposite Send (`alt+m`, `v1.4.0+`)**: Press `alt+m` (`Option+m` on macOS when Option acts as Alt) to send a message using the opposite of your `Queued Messages` setting: send right away in `queue` mode, or queue in `send-immediately` mode.
- **Verbosity Modes (`v1.2.10+`)**:
  - `high`: Full tool calls, parameter payloads, and thoughts.
  - `medium` (default since `v1.3.0`): Groups related tool calls into concise summaries (e.g. `Explored 14 files`) while preserving command outputs.
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

### 7. Conversation Compaction (`/compact`)
Added in `v1.3.5+`. Free up the context window on demand:
- `/compact`: Summarizes the conversation and ends the turn.
- `/compact <message>`: Summarizes the conversation, then answers your message.
- A `Compacting conversation...` label is shown while it runs (cancel with `Esc`), followed by how many steps were summarized and how much smaller the context became.

### 8. `/diff` Viewer Enhancements
- **Syntax highlighting & folding (`v1.3.5+`)**: Token colors per color scheme with tinted added/removed lines. `Ctrl+O` expands or collapses the fold nearest the cursor and `f` toggles changes-only vs. whole file.
- **Navigation (`v1.3.1+`, `v1.3.2+`)**: `←`/`→` open the next or previous file, `n`/`N` continue across files, and `Shift+Tab` switches to the previous view.
- **Commit Tree (`v1.4.0+`)**: Shows the latest 100 commits with scrolling; `Enter` opens the selected commit.

### 9. Terminal Status Reporting (OSC 7501)
Added in `v1.4.0+`. In supporting terminals the tab or window shows whether the CLI is working, waiting for input, done, or failed. Control it with `AGY_CLI_OSC7501=on|off` and `AGY_CLI_OSC7501_MESSAGES=brief`.
