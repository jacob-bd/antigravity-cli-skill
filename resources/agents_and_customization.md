# agy Custom Agents & Customization System Reference (v1.4.2)

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
- `hidden` *(boolean)*: If `true`, hides the agent from `/agents` picker menus.
- `model` *(string)*: Model tier (`flash`, `pro`, `inherit`) or explicit model slug.
- `commandExecutionPolicy` *(string)*: Tool execution policy (`ask`, `allow`).
- `inheritCustomizations` *(boolean)*: Master switch to inherit ambient skills, rules, and plugins (`v1.1.14+`).
- `excludeDefaultComponents` *(boolean)*: Opts out of default prompt sections and baseline tools (`v1.2.1+`).
- `rules` *(array of strings)*: Explicit rule file paths applied unconditionally (`v1.1.15+`).
- `agents` *(array of strings)*: Explicit subagent dependencies declared for this agent (`v1.1.27+`).

### Subagent Orchestration & Messaging
- **Direct Subagent Messaging Syntax (`v1.2.9+`)**: In the prompt, type `@<subagent-name> <instruction>` to dispatch tasks directly to a specific subagent with shell-style autocomplete.
- **Roster Auto-Injection (`v1.2.5+`)**: Any custom agent declaring `invoke_subagent` in its tools automatically receives the full subagent catalog and usage instructions in its system prompt.
- **Built-in Image Generator (`v1.2.16+`)**: The agent automatically delegates visual asset creation to the built-in `image-generator` subagent with up to 3 automatic verification iterations.
- **Isolated Worktrees (`v1.2.10+`)**: Subagent Git operations are isolated in `worktrees/` inside the application data directory to prevent workspace collision.

## Customization System & Token Budgeting

### Dedicated 20,000-Token Rule Budget (`v1.2.7+`)
User and workspace rules (`GEMINI.md`, `AGENTS.md`, `.agents/rules/*.md`) are allocated a dedicated 20,000-token budget. Rules are truncated cleanly along newline boundaries, ensuring large rule sets **never evict skills, workflows, subagents, or MCP tools**.

### Manifest Directory Scanning Rules (`v1.2.10+`)
Directory paths in `skills.json`, `rules.json`, `agents.json`, and `plugins.json` load only items located directly inside the directory (matching `.agents/skills/` behavior). To load nested sub-items, explicitly specify them in `include_only`:
```json
{
  "path": "shared_skills",
  "include_only": ["security/vuln-scanner", "compliance/license-check"]
}
```

### Hierarchical Configuration Loading (`v1.2.16+`)
Manifests (`skills.json`, `rules.json`, etc.) are discovered and merged across every `.agents/` folder between the current working directory and the project Git root.

### Plugin Authoring (`v1.3.2+`)
- Keep the manifest in a `.antigravity-plugin/` or `.gemini-plugin/` folder, or at the plugin root as `plugin.json`.
- List skills in `plugin.json`, or load every skill from the plugin's `skills/` folder.
- Declare subagents with a root-level `agents.json`, the same way `skills.json` and `rules.json` declare skills and rules (`v1.3.3+`).
- Store data in the per-plugin folder exposed as `PLUGIN_DATA`, and ship command-line tools in a `bin/` folder that the agent can run by name.
- Put `AGENTS.md`/`GEMINI.md` rules in `rules/`, not at the plugin root (`agy plugin validate` warns otherwise).
- Frontmatter in skills, rules, workflows, and Markdown agents may contain an unquoted colon in `description:` (`v1.3.6+`).

### Skill Metadata & Hot Reloading
- **Skill Visual Branding (`v1.1.20+`)**: Add `metadata.icon` (e.g. `icon: 🛡️`) in `SKILL.md` frontmatter to render icons across `/skills` and autocomplete popups.
- **Silent Skills (`v1.1.12+`)**: Add `disable-slash-command: true` in `SKILL.md` frontmatter to keep a skill model-invocable without cluttering the interactive `/` slash command menu.
- **Live Reload (`v1.2.4+`)**: Run `/skills reload` to reload all skills and commands dynamically without restarting the CLI session.
