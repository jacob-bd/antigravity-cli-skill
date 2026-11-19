# `agy` Custom Agents, Workflows, Swarms & Customization (v1.5.12)

Reference for custom Markdown agents, dynamic Python workflows (`run_workflow`), multi-agent swarms, customization manifests, plugin authoring/validation, lifecycle hooks (`hooks.json`), and skill metadata.

## Contents

- [Custom Markdown Agents](#custom-markdown-agents)
- [Dynamic Workflows (`run_workflow`)](#dynamic-workflows-run_workflow)
- [Multi-Agent Swarms & Teamwork](#multi-agent-swarms--teamwork)
- [Customization Discovery, Rules & Manifests](#customization-discovery-rules--manifests)
- [Plugins: Authoring, Import & Validation](#plugins-authoring-import--validation)
- [Lifecycle Hooks (`hooks.json`)](#lifecycle-hooks-hooksjson)
- [Skill Metadata & Live Reload](#skill-metadata--live-reload)

---

## Custom Markdown Agents

Antigravity CLI defines custom primary agents and specialized subagents as Markdown files with YAML frontmatter.

### Storage Locations
- **Project Agents**: `.agents/agents/<name>.md` (auto-discovered walking up to the repository root).
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

### Frontmatter Fields
- `name` *(string)*: Unique identifier.
- `description` *(string)*: Description shown in `/agents` (supports unquoted colons).
- `mainAgent` *(boolean)*: If `true`, launchable directly via `agy --agent <name>`.
- `subagent` *(boolean)*: If `true`, callable as a subagent by other agents.
- `hidden` *(boolean)*: If `true`, hides the agent from `/agents` pickers (built-in `browser` is subagent-only).
- `model` *(string)*: Model tier (`flash`, `pro`, `inherit`) or explicit model slug.
- `commandExecutionPolicy` *(string)*: Tool execution policy (`ask`, `allow`). Plugin agents set to `allow` cannot override Enterprise admin-enforced review policies.
- `inheritCustomizations` *(boolean)*: Inherit ambient skills, rules, and plugins.
- `excludeDefaultComponents` *(boolean)*: Opt out of default prompt sections and baseline tools.
- `rules` *(string[])*: Explicit rule file paths applied unconditionally.
- `agents` *(string[])*: Explicit subagent dependencies declared for this agent.

### Subagent Orchestration & Isolation
- **Direct Subagent Messaging**: Type `@<subagent-name> <instruction>` in the prompt to message a running or completed subagent directly (with autocomplete).
- **Roster Auto-Injection**: Custom agents listing `invoke_subagent` in their tools automatically receive the available subagent roster and instructions.
- **Model & Blocked-Tool Display**: Subagent rows above the prompt and `/agents` show each subagent's active model and the exact tool it is waiting on (e.g., `Blocked · AskQuestion`); status-line JSON includes `model` on each `subagents` entry.
- **Built-in Image Generator**: Image generation requests delegate to the built-in `image-generator` subagent (up to 3 attempts, saved to conversation artifacts).
- **Dirty-State Worktrees & Non-VCS Copies**: Subagent and swarm Git worktrees under `<appDataDir>/worktrees/` inherit staged, unstaged, and untracked local changes. Outside version control, agents can receive an isolated directory copy (up to 100,000 entries / 4 GiB).
- **Cross-Subagent Permissions & Compaction**: Approving a permission applies across the entire conversation tree. Compacting a subagent retains the parent's 10 most recent messages.

---

## Dynamic Workflows (`run_workflow`)

`run_workflow` is a built-in model tool (not a CLI flag) that executes a Python script using the CLI's bundled Python (no local Python install required) to orchestrate parallel or staged agent work programmatically. Ask for a workflow in your prompt when work splits into loops, fan-out/aggregate passes, or multi-stage pipelines.

### Script Primitives

| Primitive | Behavior |
| :--- | :--- |
| `phase(name)` | Starts a named UI phase grouping subsequent logs and agent cards. |
| `log(*args)` | Appends a line to workflow logs and the UI. |
| `agent(prompt=None, schema=None, role=None, type_name="self", workspace="inherit", model=None)` | Defines a workflow agent (lazy until awaited or called). `schema` (JSON schema dict/string) is required; the agent returns structured data via its `finish` tool. `await agent(...)` runs once; `a = agent(...)` followed by `await a(prompt, schema)` keeps multi-turn context across calls. |
| `parallel(items, fn)` | Runs `fn(item)` concurrently and returns results in item order; if one fails, remaining tasks cancel. |
| `pipeline(items, fn)` | Runs and awaits `fn(item)` sequentially in item order. |
| `call_tool(name, **args)` | Invokes allowed tools directly from Python; raises `ToolError` (`.denied` is `True` on permission/hook denial; workflows cannot prompt the user). |
| `trajectory(last=None, types=None, tools=None, min_index=None, max_index=None)` | Reads earlier parent-conversation steps captured before `run_workflow` started. |

- **`workspace` values**: `"inherit"` (shared workspace, best for readers), `"share"` (separate working copy sharing the repo), or `"branch"` (isolated clone for parallel editors).
- **`model` tiers**: `"inherit"` (default), `"flash_lite"`, `"flash"`, or `"pro"`.
- **Example**:
  ```python
  phase("Discover")
  files = await agent(
      "List Go files under pkg/auth that parse JWTs. Paths only.",
      schema={"type": "array", "items": {"type": "string"}},
  )
  phase("Audit")
  findings = await parallel(files, lambda f: agent(
      f"Review {f} for signature bypass bugs. Reply in 3 bullets or 'OK'.",
      schema={"type": "string"},
      role=f"Audit {f}",
      model="flash",
  ))
  for item in findings:
      log(item)
  ```

---

## Multi-Agent Swarms & Teamwork

- **`/swarm <task>` *(Preview & Quota Heavy)***: Turns the current session into a coordinator that spawns parallel worker agents collaborating over a shared message board (`swarm`, `post_message`, `read_board`, `swarm_wait`).
  - **Worker sizing & sharding**: Pass `n` (homogeneous copies) or `tasks` (per-worker slices) to `swarm{action:"add"}` to spawn `<name>-1`..`<name>-<k>` in one call.
  - **Context & workspace isolation**: Pass `fork: "<member>"` (or `"coordinator"`) to branch from warm context; pass `workspace: "share"` or `"branch"` for parallel editors (commit/land work before `kill`, which permanently deletes the workspace).
  - **Channels**: Large swarms can split board traffic into channels (`swarm{action:"channel", op:"create|subscribe|unsubscribe|list|archive"}`) and address whole areas with `@#channel` while keeping `main` for cross-area decisions.
  - **Quota & stalls**: Workers pause on quota errors instead of looping; switch a waiting worker's model in place via `swarm{action:"model", name, model}` without losing conversation history. Optional `monitor` and `narrator` workers watch for stalls and maintain the UI overview.
  - **Headless / Remote Control caveat**: `/swarm board` (`Alt+S`) is interactive-TUI only and cannot open in `-p` runs or Remote Control.
- **`/teamwork` Run Isolation**: Each `/teamwork` run stores state in its own directory under `<appDataDir>` rather than `.agents/teamwork/`, preventing concurrent workspace runs from colliding.

---

## Customization Discovery, Rules & Manifests

- **Dedicated Rule Budget**: Always-on and global rules (`GEMINI.md`, `AGENTS.md`, `.agents/rules/*.md`) have a **24,000-byte per-file cap** (truncated on line boundaries) and a dedicated **20,000-token aggregate budget** so rules never evict skills, workflows, subagents, or MCP tools. Over-budget rules are demoted to file path pointers for on-demand reading.
- **Hierarchical `.agents/` Loading**: Manifests (`skills.json`, `plugins.json`, `rules.json`, `agents.json`) merge from every `.agents/` directory between the working directory and the repository root.
- **One-Level Directory Scanning**: Directory entries in manifests scan only direct children unless nested sub-items are listed in `include_only`:
  ```json
  {
    "entries": [
      {
        "path": "shared_skills",
        "include_only": ["security/vuln-scanner", "compliance/license-check"],
        "exclude": ["experimental-.*"]
      }
    ]
  }
  ```
- **Workspace Confinement**: Workspace `.agents/plugins.json` and `.agents/skills.json` only load paths inside the workspace or repository root; absolute paths, `../`, and symlink escapes are skipped.

---

## Plugins: Authoring, Import & Validation

- **Manifest & Layout**: Place `plugin.json` at the plugin root (or `.antigravity-plugin/plugin.json` / `.gemini-plugin/plugin.json`). Put skills in `skills/<skill>/SKILL.md`, agents in `agents/<agent>.md` (or root `agents.json`), rules in `rules/AGENTS.md` (not root `AGENTS.md`), MCP servers in `mcp_config.json`, binaries in `bin/`, and persistent data in `PLUGIN_DATA`.
- **Native Plugin `hooks.json` Path**: Native plugins **must** place `hooks.json` directly at the plugin root (`<plugin_dir>/hooks.json`). `hooks/hooks.json` is only staged automatically when importing Claude Code or Gemini CLI extensions.
- **Sidecar Paths & Optional Auth**: `sidecar.json` supports `${HOME}` in `workspaceUris`/`accessGrants`; `auth.json` supports `"optional": true` on API key providers for free-tier unauthenticated use.
- **Pre-Install & CI Validation (`agy plugin validate [target] [--strict] [--json]`)**:
  - `target` accepts a local directory or `plugin.json` (default `.`), a `.zip` file or `https://` zip URL, a GitHub repo or subfolder URL (`https://github.com/<owner>/<repo>/tree/<branch>/<path>`), or a `marketplace.json` file/URL (fetched to a temp dir, never installed or run).
  - Validates `name` (kebab-case), `category`, `author`/`homepage`/`repository`, `logo`/`iconUri`, skills (including folder-name fallback), agents, rules, `hooks.json`, sidecars, context providers, and `extensions["com.google.antigravity"]`. Handles UTF-8 BOM in `plugin.json` and `hooks.json`.
  - Emits JSON findings under `.findings[]`. Warns if hook commands reference `${PLUGIN_ROOT}` / `$PLUGIN_ROOT` instead of relative paths (fails only with `--strict`).
- **Claude & Gemini Import (`agy plugin import claude|gemini`)**: Converts Claude Code marketplace plugins, remote HTTP MCPs, `.mcp.json`, manifest `"skills"` arrays, `commands/` into named skills, and `PreToolUse`/`PostToolUse`/`Stop`/`SessionStart` hooks (mapping tool matchers such as `Bash` -> `run_command`, 60s default timeout).
- **Atomic Installs & Enterprise Controls**: Installs swap atomically with rollback on failure and reject symlinks escaping the plugin, FIFOs, and device files. When managed admin settings set `pluginControls.pluginsDisabled: true`, plugin MCP servers/processes cannot start before sign-in completes and running plugin processes stop immediately if disabled mid-session.

---

## Lifecycle Hooks (`hooks.json`)

Hooks intercept the agent lifecycle in `.agents/hooks.json`, `~/.gemini/config/hooks.json`, or `<plugin_dir>/hooks.json`.

**Important JSON conventions (protojson):**
- Every top-level key in `hooks.json` is a **named hook group** (e.g., `"prompt-guard"`, `"test-gate"`), not a bare event name.
- All stdin/stdout JSON keys use **camelCase** (`conversationId`, `hookEventName`, `lastUserInput`, `continuationActive`, `toolCall.id`).
- Hook commands run synchronously (`sh -c` on Unix, `cmd /c` on Windows; default timeout 30s) with the directory containing `hooks.json` as their working directory. Plugin hooks also receive `PLUGIN_ROOT` and `PLUGIN_DATA` environment variables.

### Structure & Event Types

```json
{
  "safety-gate": {
    "PreToolUse": [
      {
        "matcher": "run_command",
        "hooks": [
          { "type": "command", "command": "./hooks/check-cmd.sh", "timeout": 15 }
        ]
      }
    ],
    "PreInvocation": [
      { "type": "command", "command": "./hooks/screen-prompt.sh" }
    ],
    "Stop": [
      { "type": "command", "command": "./hooks/verify-once.sh" }
    ]
  }
}
```

- **Grouped events (require `matcher` + `hooks` array)**: `PreToolUse`, `PostToolUse` (matcher regex against tool name such as `"*"`, `"run_command"`, `"run_command|view_file"`).
- **Flat events (array of handler objects directly)**: `SessionStart`, `PreInvocation`, `PostInvocation`, `Stop`.
- **Common stdin fields**: `conversationId`, `workspacePaths`, `transcriptPath`, `artifactDirectoryPath`, `modelName`, `lastUserInput`, `hookEventName`, `cwd` (agent's working directory).

### Event Contracts

1. **`PreToolUse`**:
   - Stdin adds `toolCall: { "id", "name", "args" }` and `stepIdx`.
   - Stdout: `{"decision": "allow" | "deny" | "ask" | "force_ask", "reason": "...", "permissionOverrides": [...], "overwrite": { ... }}`.
   - `overwrite` performs a shallow top-level merge into tool call arguments before execution.
   - **Admin precedence**: `PreToolUse` returning `"allow"` cannot bypass organization admin-enforced review policies for unsandboxed commands or out-of-workspace file access.
2. **`PostToolUse`**:
   - Stdin adds matching `toolCall` (`id` matches `PreToolUse`), `stepIdx`, and optional `error`. Stdout: `{}`.
3. **`SessionStart` / `PreInvocation` / `PostInvocation`**:
   - To block a prompt before calling the model in `PreInvocation` (causing `-p` to exit `1`), nest `errorMessage` inside `injectSteps` and set `terminationBehavior: "terminate"`:
     ```json
     {
       "injectSteps": [
         {
           "errorMessage": {
             "userMessage": "Blocked: prompt contains sensitive credentials."
           }
         }
       ],
       "terminationBehavior": "terminate"
     }
     ```
   - `PostInvocation` also supports `"terminationBehavior": "force_continue"`.
   - `injectSteps` also supports `{"systemMessage": {"systemMessage": "...", "metadata": {"user_markdown": "..."}}}` and `{"userMessage": "..."}`.
4. **`Stop`**:
   - Stdin adds `executionNum`, `terminationReason` (`NO_TOOL_CALL`, `MAX_INVOCATIONS`, `ERROR`, `USER_CANCELED`, etc.), `fullyIdle`, and **`continuationActive`** (`true` while the agent is already continuing from an earlier `Stop` hook in this turn — check it to prevent infinite loops).
   - Stdout: Return **`{"decision": "continue", "reason": "..."}`** to block stopping and inject `reason` as a system message. Any other `decision` value (or `{}`) allows the agent to stop. See [../examples/usage_patterns.md](../examples/usage_patterns.md) for a complete runnable `Stop` hook script.

---

## Skill Metadata & Live Reload

- **Folder-Name Fallback**: If a skill's `SKILL.md` omits `name` in frontmatter, `agy` names it after its parent directory instead of colliding as `SKILL`.
- **Visual Branding**: Add `metadata.icon` (e.g., `icon: 🛡️`) in `SKILL.md` frontmatter.
- **Silent Skills**: Set `disable-slash-command: true` to keep a skill model-invocable without listing it in the `/` menu.
- **Live Reload**: Run `/skills reload` inside the TUI to reload local skills without restarting (preserves cloud skills in `--remote-agent` sessions).
