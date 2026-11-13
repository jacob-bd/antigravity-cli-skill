# agy Interactive Interface (TUI) Reference (v1.5.7)

Detailed reference moved out of `SKILL.md` to keep it under agy's 24 KB skill body limit.

## Interactive Interface, Keybindings & Navigation

### 1. Modal Vim Editing Mode
Added in `v1.1.11+` and expanded in `v1.2.9+`, `v1.2.12+`, `v1.4.9+`, and `v1.5.6+`. Configure in `/settings` under `Editor Mode` -> `Vim`:
- **Modes**: Normal, Insert, Visual, Visual Line. Mode status badge rendered in status line.
- **Motions & Counts**: Count multipliers (`3dw`, `2d3w`, `3dd`, `3x`, `[count]G`, `2di(`), plus `r<space>`, `f<space>`, `df<space>` (`v1.4.9+`).
- **Undo/Redo Cursor (`v1.5.6+`)**: `u` and redo return the cursor to where the edit was made instead of jumping to the end of the prompt.
- **Submissions**: Submit from Normal mode via `Ctrl+S` or `Ctrl+Enter`.
- **Insert First Option**: Opens prompt in Insert mode where bare `Enter` submits and `Shift+Enter` / `Ctrl+J` inserts newlines.
- **Keybinding scopes**: Customize in `keybindings.json` under `vim.*`.

### 2. One-Shot Model Switching (`/model`)
- Switch default model: `/model <name>` (accepts base names without `--effort` since `v1.3.4+`).
- **One-shot execution (`v1.1.27+`)**: `/model <name> <prompt>` executes a single turn on `<name>` and automatically reverts back to the original model.

### 3. Settings Fast Navigation (`/config` & `/settings`)
- **Arrow Cycling (`v1.2.16+`)**: Press `←` / `→` on any highlighted setting to cycle and save immediately.
- **Rendering Mode (`v1.5.0+`)**: `no flickering (altscreen)` was renamed to `full screen (altscreen)` (saved setting key `altScreenMode` unchanged).
- **Queued Messages (`v1.2.14+`)**: `"queuedMessages": "queue"` (default) or `"send-immediately"`.
- **Opposite Send (`alt+m`, `v1.4.0+`, native macOS Option `v1.5.2+`)**: Sends using the opposite of your `Queued Messages` setting. macOS Option shortcuts work out of the box since `v1.5.2+` (set `AGY_CLI_MAC_OPTION_AS_ALT=0` to revert).
- **Verbosity Modes (`v1.2.10+`, default `medium` `v1.3.0+`)**:
  - `high`: Full tool calls, payloads, and thoughts.
  - `medium`: Groups related tool calls into summaries (e.g. `Explored 14 files`) while keeping command outputs.
  - `low`: Minimal output with token metric tables.

### 4. Swarms & Live Board (`/swarm`, `v1.5.5+`–`v1.5.6+`)
- **`/swarm <task>`**: Starts a coordinator + parallel worker swarm with a shared message board.
- **Active-Items Tray**: Shows `● Swarm(<title>)` with latest post/count while running and `Done` when finished (`Enter` opens board; `alt+x` / `swarm.dismiss` dismisses).
- **`/swarm board` (`alt+s` / `swarm.board`)**: Full-screen view with `Roster`, `Posts`, and `Message` panes (plus Kitty graphics bee animations / pixel-art queen in `v1.5.6+`):
  - `Tab` / `Shift+Tab`: Switch panes
  - `z`: Zoom focused pane
  - `Enter`: Show or hide message
  - `[` / `]`: Step through posts
  - `p`: Filter feed to one member's posts and mentions
  - `o`: Open member subagent details (`Esc` returns to board preserving state, `v1.5.6+`)
  - `f`: Follow new posts; `Esc`: Back out

### 5. Local Projects & Remote Agent Sessions (`v1.5.0+`–`v1.5.2+`)
- **`/projects` (`v1.5.0+`)**: List projects (with current marked), create a project from a name and folder paths (absolute, `~`, or relative), rename/edit folders, or switch projects (adds folders to workspace and starts a new conversation).
- **`--remote-agent` TUI (`v1.5.0+`–`v1.5.2+`)**: Header shows `remote workspace`; `@`-mention searches remote files; `/config` separates `TUI` and `Remote Agent` tabs; `/skills` lists connected cloud plugin skills under `Remote agent skills`.

### 6. Side Questions (`/btw`, `v1.5.0+`)
- `/btw <query>` can call read-only tools live inside the `/btw` card and reports `Called N tools` when done; permission-requiring tools are auto-denied so quick questions never block.

### 7. Artifact Detail Viewer & Native Media Paste
- **Inline Kitty Graphics (`v1.2.7+`)**: Renders LaTeX and Mermaid diagrams on Kitty-compatible terminals.
- **Controls**: `m` cycles display mode (Image, ASCII, Raw); `t` opens outline; `Left`/`Right` pans diagrams; `Ctrl+D`/`Ctrl+U` half-page scrolls; `Ctrl+G` opens in `$EDITOR`.
- **Image Paste & Drag-and-Drop (`v1.4.9+`)**: Terminal `Cmd+V` / `Ctrl+Shift+V` and file drag-and-drop attach images directly (including over SSH with X11/Wayland forwarding).

### 8. Clipboard (`/copy`), Resume (`/resume`), Compact (`/compact`) & `/diff`
- `/copy`: Copies latest response; `/copy <n>` copies n-th response; `/copy btw` copies `/btw` answer.
- `/resume`: `Ctrl+F` toggles directory grouping (`pickerGrouping`).
- `/compact [message]`: Summarizes conversation to free context; with a message, answers it afterward (`Esc` cancels).
- `/diff` Viewer: Syntax highlighting & folding (`Ctrl+O` fold, `f` changes-only), `/` live search (`n`/`N` next/prev match), `←`/`→` switch files, `[`/`]` Per Turn jump, Commit Tree (`Enter` opens commit).

### 9. Terminal Status Reporting (OSC 7501)
Added in `v1.4.0+`. Shows working / waiting / done / failed state in supporting terminal tabs. Control via `AGY_CLI_OSC7501=on|off` and `AGY_CLI_OSC7501_MESSAGES=brief`.
