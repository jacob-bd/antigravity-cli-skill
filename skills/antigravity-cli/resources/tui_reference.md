# `agy` Interactive Interface (TUI) & Remote Control Reference (v1.5.12)

Reference for interactive TUI editing, slash commands, altscreen controls, `/swarm board`, `--remote-agent` sessions, Remote Control browser view, and OSC 7501 status reporting.

## Contents

- [Modal Vim Editing](#modal-vim-editing)
- [Model, Effort & Mode Switching](#model-effort--mode-switching)
- [Settings, Rendering & Altscreen](#settings-rendering--altscreen)
- [Swarms & Live Board](#swarms--live-board)
- [Local Projects & Remote Agent Sessions](#local-projects--remote-agent-sessions)
- [Quick TUI Commands & Workflows](#quick-tui-commands--workflows)
- [Artifact Viewer & Media Paste](#artifact-viewer--media-paste)
- [Remote Control Browser View](#remote-control-browser-view)
- [Terminal Status Reporting (OSC 7501)](#terminal-status-reporting-osc-7501)

---

## Modal Vim Editing

Enable in `/settings` under `Editor Mode` -> `Vim`:
- **Modes**: Normal, Insert, Visual, Visual Line (active mode badge appears in the status line).
- **Motions & Counts**: Numeric counts work across operators, motions, and text objects (`3dw`, `2d3w`, `3dd`, `3x`, `[count]G`, `2di(`), plus `r<space>`, `f<space>`, `df<space>`, and full Unicode/emoji handling.
- **Vim-Accurate Undo/Redo**: `u` undoes one command at a time (no time-based grouping); changes entering Insert mode (`cw`, `o`) undo together with everything typed before `Esc`; undoing or redoing an operator (`dd`, `diw`) places the cursor at the start of the changed text.
- **Submission**: Submit from Normal mode with `Ctrl+S` or `Ctrl+Enter`. Enable `Insert First` to start in Insert mode where `Enter` submits and `Shift+Enter` / `Ctrl+J` inserts newlines.
- **Custom Keybindings**: Configure `vim.*` scopes in `keybindings.json`.

---

## Model, Effort & Mode Switching

- **Switch Default Model**: `/model <name>` (base names work without `--effort`; picker supports live fuzzy search).
- **One-Shot Model Turn**: `/model <name> <prompt>` runs a single turn on `<name>` and automatically reverts to the session default on the next turn (queued prompts wait until their turn starts before switching).
- **Orchestrator Planner Selector**: When multiple Orchestrator planners (such as Carbon and Argon) are available, `/model` displays a single `Orchestrator` row with a `Planner` choice switched via `←`/`→` or `Tab` (`--model` and saved settings still use full names).
- **Effort & Modes**: `/effort <level>` adjusts reasoning effort (`low`–`max`, depending on model/account). `Shift+Tab` cycles execution modes.

---

## Settings, Rendering & Altscreen

- **Fast `/config` Navigation**: Press `←` / `→` on a highlighted setting to cycle and save immediately.
- **Full Screen (`altscreen`) Rendering Mode** (setting key `altScreenMode`):
  - **Click-to-Expand & Hover**: Click tool groups, thoughts, attached images, or multi-line command output to expand/collapse individually (instead of `Ctrl+O`); expandable targets and pinned prompts highlight on hover.
  - **Scroll Position Indicator**: While scrolled up, the line above the prompt shows remaining content below (e.g., `↓ 25 lines · 2 turns`).
  - **Clean Markdown & Copy**: Headings render styled (bold, italic, underline) without raw `##`–`######` markers, CJK bold punctuation renders cleanly, and mouse-copying a wrapped prompt preserves original text without inserting line breaks at screen wrap points.
- **Queued Messages**: Configure `"queuedMessages": "queue"` (default) or `"send-immediately"`. Press `alt+m` to send using the opposite behavior. Native macOS Option shortcuts work out of the box (set `AGY_CLI_MAC_OPTION_AS_ALT=0` to revert).
- **Verbosity Modes** (default `medium`): `high` (full tool calls, payloads, thoughts), `medium` (grouped summaries like `Explored 14 files` + command outputs), `low` (minimal output + token metric tables).
- **`GEMINI_API_KEY` TUI Behavior**: Unsupported commands (`/usage`, `/quota`, `/credits`, `/remote-control`, `/voice`, `/record`), the red `AI: Out of credits` badge, and the `Use AI Credits` config row are automatically hidden when connecting with an API key.

---

## Swarms & Live Board

- **`/swarm <task>`**: Starts a coordinator + parallel worker swarm over a shared message board.
- **Active-Items Tray**: Shows `● Swarm(<title>)` with latest post/count while running and `Done` when finished (`Enter` opens board; `alt+x` / `swarm.dismiss` dismisses).
- **`/swarm board` (`alt+s` / `swarm.board`)**: Full-screen view with `Roster`, `Posts`, and `Message` panes (plus Kitty graphics bee animations / pixel-art queen):
  - `Tab` / `Shift+Tab`: Switch panes
  - `z`: Zoom focused pane
  - `Enter`: Show or hide message
  - `[` / `]`: Step through posts
  - `p`: Filter feed to one member's posts and mentions
  - `o`: Open member subagent details (`Esc` returns to board preserving selection, filter, focus, and zoom)
  - `f`: Follow new posts; `Esc`: Back out

---

## Local Projects & Remote Agent Sessions

- **`/projects`**: List projects (current marked), create a project from a name and folder paths (absolute, `~`, or relative), rename/edit folders, or switch projects (starts a new conversation).
- **`--remote-agent` Sessions**:
  - Header shows `remote workspace`; `@`-mention searches remote files; `/config` separates `TUI` and `Remote Agent` tabs; `/skills` lists cloud plugin skills under `Remote agent skills`.
  - **`accept-edits` Toggle**: `Shift+Tab` switches between default mode and `accept-edits`. Default mode asks before each file edit in the remote workspace; switch to `accept-edits` for automatic edits.

---

## Quick TUI Commands & Workflows

| Command / Shortcut | Purpose |
| :--- | :--- |
| `!<cmd>` | Run a local shell command inline without burning a model turn (streams stdout/stderr in order). |
| `/btw <query>` | Ask a side question in a card with live read-only tool calls (permission-requiring tools auto-deny; runs until finished or `Esc`). Copy result with `/copy btw`. |
| `/diff` | Interactive diff viewer: `←`/`→` switch files, `/` live search (`n`/`N` next/prev), `[`/`]` Per Turn jump, `Ctrl+O` fold, `f` changes-only, Commit Tree (`Enter` opens commit). |
| `/rewind` | Inspect relative timestamps and rewind or fork conversation state and workspace snapshots. |
| `/fork` | Branch the current conversation cleanly. Switches immediately without re-loading full history; for large artifact/scratch folders, points the forked agent at the original folder instead of copying (fast, but **deleting the original conversation also deletes those shared files**). |
| `/resume [<id-prefix>]` | Open the conversation picker (`Ctrl+F` toggles directory grouping) or jump directly to any unique ID prefix (e.g., short ID in status line). |
| `/rename <title>` | Rename the active conversation title. |
| `/compact [message]` | Summarize conversation history to free context; if `message` is given, answers it afterward (`Esc` cancels). |
| `/copy [n\|btw]` | Copy latest response, `n`-th response, or active `/btw` answer to clipboard (works over SSH). |
| `/skills reload` | Hot-reload discovered skills and slash commands without restarting the session. |
| `/codesearch` (`/cs`, `/search`) | Run workspace code search directly from the TUI. |
| `/voice` (`F5`) | Voice dictation (auto-finalizes at 3m30s; pair with `agy mic-serve` over SSH; unavailable with `GEMINI_API_KEY`). |

---

## Artifact Viewer & Media Paste

- **Inline Kitty Graphics**: Renders LaTeX math and Mermaid flowcharts/sequence diagrams on Kitty-compatible terminals, with Unicode ASCII fallback (`CLI_GRAPHICS=kitty` forces image mode inside `tmux`/`screen` when passthrough is configured).
- **Controls**: `m` cycles display mode (Image, ASCII, Raw); `t` opens outline; `Left`/`Right` pans wide diagrams; `Ctrl+D`/`Ctrl+U` half-page scrolls; `Ctrl+=`/`Ctrl+-` zooms; `Ctrl+G` opens in `$EDITOR`.
- **Image Paste & Drag-and-Drop**: `Cmd+V` / `Ctrl+Shift+V` and file drag-and-drop attach images directly (including over SSH with X11/Wayland forwarding).

---

## Remote Control Browser View

When connected via `--remote-control`, `/remote-control`, or `agy remote-control start`:
- **Split View (desktop/tablet)**: Keep multiple conversations side by side via `Split Right` or `Split Down` in the sidebar conversation menu or title-bar `⋮` menu.
- **Overview Panel (`Cmd+Option+O` / `Ctrl+Alt+O`)**: Lists the conversation's artifacts, modified files, subagents, and background tasks; side-panel file/review tabs persist across reload and open with their sidebar collapsed (`Toggle sidebar` expands it).
- **Navigation & Quoting**: Hover/click the prompt navigator rail (or press `[` / `]`) to jump between prompts; select response text and click `Quote` to quote it in your next message (clicking an older quote auto-loads and scrolls to it).
- **Subagent Drag-and-Drop & Attachments**: Drag a subagent tab from the side panel into the message box to insert an `@` mention; attach files up to **20 MB**; links open with `Cmd+click` (`Ctrl+click` on Windows/Linux) or middle-click.
- **Timers & Scrollback**: Active timers show live countdowns in `Background Tasks` and the status line (`Waiting for 1h timer · 59m left`); scrolling up shows `Loading older messages…` and a `Retry` link on failure.

---

## Terminal Status Reporting (OSC 7501)

Reports `working`, `waiting`, `done`, or `failed` state to supporting terminal tabs/windows. Stays in `working` while paused waiting for non-daemon background tasks or subagents to finish (daemon tasks such as dev servers do not hold it in `working`). Control with `AGY_CLI_OSC7501=on|off` and `AGY_CLI_OSC7501_MESSAGES=brief`.
