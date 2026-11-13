# agy Custom Agents, Swarms & Customization Reference (v1.5.7)

Detailed reference moved out of `SKILL.md` to keep it under agy's 24 KB skill body limit.

## Custom Markdown Agents (`agent.md`)

Antigravity CLI uses Markdown files with YAML frontmatter to define custom primary agents and specialized subagents (`v1.1.6+` through `v1.2.5+`).

### Storage Locations
- **Project Agents**: `.agents/agents/<name>.md` (automatically discovered in active workspace).
- **Global Agents**: `~/.gemini/config/agents/<name>.md`.

### Complete Agent Schema
```markdown
---
name: backend-architect
description: Senior backend systems engineer specialized in Go and distributed systems
mainAgent: true
subagent: true
hidden: false
model: pro
commandExecutionPolicy: ask
inheritCustomizations: true
excludeDefaultComponents: false
rules:
  - .agents/rules/go-standards.md
agents:
  - database-specialist
  - test-engineer
---

# Backend Architect System Prompt
You are a Staff Software Engineer. Focus on high-throughput architecture, clean concurrency models, and resilient error recovery.
```

### Frontmatter Fields Reference
- `name` *(string)*: Unique identifier.
- `description` *(string)*: Human-readable description shown in `/agents`.
- `mainAgent` *(boolean)*: If `true`, agent can be launched directly via `agy --agent <name>`.
- `subagent` *(boolean)*: If `true`, agent can be delegated to by other agents.
- `hidden` *(boolean)*: If `true`, hides the agent from `/agents` picker menus (note: built-in `browser` is subagent-only as of `v1.5.1+`).
- `model` *(string)*: Model tier (`flash`, `pro`, `inherit`) or explicit model slug.
- `commandExecutionPolicy` *(string)*: Tool execution policy (`ask`, `allow`). Note: `v1.5.2+` ensures plugin agents set to `allow` cannot override Enterprise admin-enforced review policies.
- `inheritCustomizations` *(boolean)*: Master switch to inherit ambient skills, rules, and plugins (`v1.1.14+`).
- `excludeDefaultComponents` *(boolean)*: Opts out of default prompt sections and baseline tools (`v1.2.1+`).
- `rules` *(array of strings)*: Explicit rule file paths applied unconditionally (`v1.1.15+`).
- `agents` *(array of strings)*: Explicit subagent dependencies declared for this agent (`v1.1.27+`).

### Subagent Orchestration & Messaging
- **Direct Subagent Messaging Syntax (`v1.2.9+`)**: In the prompt, type `@<subagent-name> <instruction>` to dispatch tasks directly to a specific subagent with shell-style autocomplete.
- **Roster Auto-Injection (`v1.2.5+`)**: Any custom agent declaring `invoke_subagent` in its tools automatically receives the full subagent catalog and usage instructions in its system prompt.
- **Subagent Model & Status Display (`v1.5.1+`, `v1.5.3+`)**: Subagent rows above the prompt and `/agents` display each subagent's active model (and the specific tool when blocked, e.g. `Blocked · AskQuestion`). Custom status-line JSON includes a `model` field on each `subagents` entry.
- **Built-in Image Generator (`v1.2.16+`)**: Visual asset requests delegate to the built-in `image-generator` subagent with up to 3 verification iterations.
- **Dirty-State Worktrees & Non-VCS Copies (`v1.2.10+`, `v1.5.5+`)**: Subagent and swarm Git worktrees under `worktrees/` inherit your staged, unstaged, and untracked local changes. Outside version control, agents can receive an isolated workspace copy (up to 100,000 entries / 4 GiB).
- **Cross-Subagent Permissions (`v1.4.6+`)**: Approving a permission in the main agent or any subagent applies across the entire conversation tree.
- **Compaction Fidelity (`v1.4.10+`)**: Compacting a subagent retains the parent agent's 10 most recent messages to preserve delegation continuity.

### Swarms & Multi-Agent Teams (`v1.5.5+`–`v1.5.6+`)
- **`/swarm <task>` *(Preview & Quota Heavy)***: Turns the session into a coordinator that spawns parallel worker agents (each in its own conversation, able to fork from existing members or run any model from `/model`) communicating via a shared message board. Workers pause instead of retrying when their model hits quota, and a monitor worker alerts the coordinator on stalls.
- **`/swarm board` (`Alt+S`)**: Live full-screen board (`Roster`, `Posts`, `Message` panes) with Kitty graphics bee animations (`v1.5.6+`). Note: headless `-p` runs and Remote Control cannot open `/swarm board` interactively (`v1.5.6+`–`v1.5.7+`).
- **`/teamwork` Run Isolation (`v1.5.5+`)**: Each `/teamwork` run stores its state in an isolated directory under the CLI's app data directory rather than `.agents/teamwork/`, preventing concurrent runs in the same workspace from colliding.

## Customization System & Token Budgeting

### Dedicated 20,000-Token Rule Budget (`v1.2.7+`)
User and workspace rules (`GEMINI.md`, `AGENTS.md`, `.agents/rules/*.md`) receive a dedicated 20,000-token budget. Rules truncate cleanly along newline boundaries and **never evict skills, workflows, subagents, or MCP tools**.

### Manifest Directory Scanning & Workspace Confinement (`v1.2.10+`, `v1.5.2+`)
- Directory paths in `skills.json`, `rules.json`, `agents.json`, and `plugins.json` load only items directly inside the directory unless nested sub-items are listed in `include_only`:
  ```json
  {
    "path": "shared_skills",
    "include_only": ["security/vuln-scanner", "compliance/license-check"]
  }
  ```
- **Untrusted Repo Protection (`v1.5.2+`)**: Workspace `.agents/plugins.json` and `.agents/skills.json` entries only load paths inside the workspace or its repository root; absolute paths, `../`, or symlink escapes are skipped.

### Hierarchical Configuration Loading (`v1.2.16+`)
Manifests (`skills.json`, `rules.json`, etc.) are discovered and merged across every `.agents/` folder between the current working directory and the project Git root.

### Plugin Authoring, Import & Validation (`v1.3.2+`–`v1.5.6+`)
- Keep the manifest in `.antigravity-plugin/plugin.json`, `.gemini-plugin/plugin.json`, or root `plugin.json`.
- Declare subagents in root `agents.json` (`v1.3.3+`), skills in `plugin.json` or `skills/`, and rules in `rules/` (never root `AGENTS.md`/`GEMINI.md`).
- Store persistent plugin data in `PLUGIN_DATA`, ship binaries in `bin/`, and use built-in `${HOME}` in `sidecar.json` `workspaceUris`/`accessGrants` (`v1.5.0+`).
- Frontmatter `description:` supports unquoted colons (`v1.3.6+`).
- **`agy plugin validate [path] [--strict] [--json]`**:
  - Emits machine-readable findings under `.findings[]` (`v1.4.0+`).
  - Validates `name` (kebab-case), `category` (`v1.5.3+`), `author`/`homepage`/`repository`, `logo`/`iconUri`, skills, agents, rules, `hooks.json`, sidecars, context providers, and `extensions["com.google.antigravity"]` (`v1.5.2+`).
  - Warns when hook commands use `${PLUGIN_ROOT}` or `$PLUGIN_ROOT` instead of relative paths (hooks already run in the plugin folder; fails only with `--strict`, `v1.5.6+`).
- **`agy plugin import claude` (`v1.4.11+`, `v1.5.5+`–`v1.5.6+`)**: Imports Claude Code marketplace plugins, remote `"type": "http"` MCP servers, `.mcp.json`, manifest `"skills"` arrays, `commands/` as named skills, and `PreToolUse`/`PostToolUse`/`Stop`/`SessionStart` hooks (mapping tool matchers like `Bash` -> `run_command` with 60s default timeout).
- **Atomic Installs & Security (`v1.5.2+`, `v1.5.6+`)**: Installs swap atomically with rollback on failure, reject unsafe plugin names, symlinks outside the plugin, FIFOs, and device files.
- **Optional API Keys (`v1.4.12+`)**: `auth.json` supports `"optional": true` for unauthenticated free-tier usage.

### Hook Capabilities for Automation

Hooks (`hooks.json`) intercept the agent lifecycle and receive enriched JSON on stdin:
- **Context fields (`v1.4.7+`, `v1.5.4+`)**: `lastUserInput`, `hookEventName`, `cwd`, `toolCall.id` (for matching `PreToolUse` to `PostToolUse`), and accurate custom model names (`v1.5.3+`).
- **Plugin Hook Env Vars (`v1.5.3+`)**: Plugin hooks receive `PLUGIN_ROOT` and `PLUGIN_DATA` in their environment (though relative paths remain more portable across older CLIs).
- **Stop Hook Loop Guard (`v1.5.6+`)**: `Stop` hooks receive `"continuation_active": true` when the agent is already continuing from an earlier Stop hook request, preventing infinite test/lint loops.
- **`PreInvocation` Termination (`v1.4.11+`)**: Return `terminationBehavior: "terminate"` to stop before calling the model; headless `-p` exits non-zero (`1`):
  ```json
  {
    "terminationBehavior": "terminate",
    "errorMessage": {
      "userMessage": "This prompt contains sensitive data and was blocked by policy."
    }
  }
  ```
- **Enterprise Policy Precedence (`v1.5.7+`)**: `PreToolUse` returning `auto_approve` cannot bypass organization admin-enforced review policies for unsandboxed commands or out-of-workspace file access.

### Skill Metadata & Hot Reloading
- **Skill Visual Branding (`v1.1.20+`)**: Add `metadata.icon` (e.g. `icon: 🛡️`) in `SKILL.md` frontmatter.
- **Silent Skills (`v1.1.12+`)**: Add `disable-slash-command: true` to keep a skill model-invocable without showing it in `/`.
- **Live Reload (`v1.2.4+`)**: Run `/skills reload` to reload skills/commands without restarting (also preserves cloud skills in `--remote-agent` sessions in `v1.5.2+`).
