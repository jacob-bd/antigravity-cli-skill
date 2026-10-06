# agy CLI Usage Patterns (v1.3.0)

Executable patterns and architectural recipes for automating, scripting, and orchestrating Google's Antigravity CLI.

---

## Pattern 1: Non-Interactive Code Generation
Run single-prompt code generation headless with auto-approved permissions. Note that in `v1.2.6+`, `--print-timeout` defaults to `0s` (unlimited), running until completion without premature timeouts.

```bash
# Standard non-interactive generation
agy -p "Generate a resilient Go HTTP server with graceful shutdown" \
  --dangerously-skip-permissions

# Explicitly bounded run with graceful partial output on timeout
agy -p "Run extensive benchmark suite on all crypto routines" \
  --dangerously-skip-permissions \
  --print-timeout 10m
```

---

## Pattern 2: Continuous Multi-Turn Streaming over Stdin/Stdout
Version `v1.1.15+` supports bidirectional, line-delimited NDJSON streaming via `--input-format stream-json` and `--output-format stream-json`. This allows persistent test harnesses, drivers, and background daemons to keep a continuous session alive without restarting the CLI:

```bash
# Launch a continuous streaming agent worker
agy -p \
  --input-format stream-json \
  --output-format stream-json \
  --dangerously-skip-permissions
```

**Sending turns to stdin:**
```json
{"prompt": "Audit src/auth/jwt.go for token replay risks"}
{"prompt": "Write a patch for jwt.go that enforces jti validation"}
```

**Receiving structured events on stdout:**
```json
{"event": "init", "session_id": "conv_9f81a", "model": "gemini-3.8-pro"}
{"event": "step_update", "step_type": "tool_call", "tool_info": {"name": "view_file", "parameters": {"AbsolutePath": "/workspace/src/auth/jwt.go"}}}
{"event": "result", "output": "Successfully implemented JTI replay cache...", "usage": {"prompt_tokens": 1280, "cache_read_tokens": 840, "candidates_tokens": 320}}
```

---

## Pattern 3: Headless Slash-Command and Skill Execution
Version `v1.1.9+` automatically expands leading slash commands and installed skills in print mode (`-p`):

```bash
# Headless run that triggers the code-review skill on the current git diff
agy -p "/jbd-code-review-skill review the uncommitted changes" \
  --dangerously-skip-permissions

# Opt out and treat leading slashes as literal text
agy -p "/not-a-command this is literal text" \
  --disable-slash-commands \
  --dangerously-skip-permissions
```

---

## Pattern 4: Zero-Turn Inspection of Read-Only Commands
Version `v1.1.11+` and `v1.1.12+` allow querying read-only slash commands in print mode to extract machine-readable JSON without starting an agent turn, without consuming quota, and without polluting conversation databases:

```bash
# Check quota buckets and reset windows in JSON
agy -p "/quota" --output-format json

# Export token usage and session spend
agy -p "/usage" --output-format json

# Inspect remaining G1 credits
agy -p "/credits" --output-format json

# List installed skills or active permission rules
agy -p "/skills" --output-format json
agy -p "/permissions" --output-format json
```

---

## Pattern 5: Managing MCP Servers via CLI
Version `v1.1.16+` introduces dedicated `mcp` subcommands to configure `mcp_config.json` without hand-editing JSON files:

```bash
# 1. List configured MCP servers
agy mcp list

# 2. Add a stdio MCP server
agy mcp add fs npx -y @modelcontextprotocol/server-filesystem /workspace

# 3. Add an stdio server with environment variables
agy mcp add --env GITHUB_TOKEN=ghp_secret gh -- docker run -i ghcr.io/github/mcp

# 4. Add an authenticated HTTP/SSE MCP server
agy mcp add --header "Authorization: Bearer my_api_key" internal-docs https://docs.internal.net/mcp

# 5. Disable and enable servers on the fly
agy mcp disable internal-docs
agy mcp enable internal-docs

# 6. Remove a server
agy mcp remove fs
```

---

## Pattern 6: Remote Control Background Service
Version `v1.2.0+` and `v1.2.6+` support managing the Remote Control background daemon:

```bash
# Register and start the background service with the OS service manager
agy remote-control start --name "build-server-01"

# Check daemon health, process ID, and active connection status
agy remote-control status

# Stop and unregister the background service
agy remote-control stop

# Or launch an interactive CLI session with a temporary tunnel
agy --remote-control
```

---

## Pattern 7: Direct Gemini API Key Authentication
Version `v1.1.13+` allows direct access to Gemini models via API key without interactive OAuth sign-in:

```bash
# Configure API key and optional custom endpoint
export GEMINI_API_KEY="AIzaSy..."
export GOOGLE_GEMINI_BASE_URL="https://generativelanguage.googleapis.com"

# Ensure modelProvider is configured in ~/.gemini/antigravity-cli/settings.json:
# { "modelProvider": "gemini" }

# Run prompts directly
agy -p "List 3 advantages of Go over Python" --dangerously-skip-permissions
```

---

## Pattern 8: One-Shot Model Prompting Mid-Conversation
Version `v1.1.27+` allows executing a single prompt against another model without altering your session default:

```bash
# Inside interactive TUI:
# Runs prompt on claude-3-7-sonnet and automatically reverts back to gemini-3.8-pro next turn
/model claude-3-7-sonnet Critique this architectural plan from a functional programming viewpoint

# In scripts, select models directly:
agy --model gemini-3.8-pro -p "High-complexity reasoning task" --dangerously-skip-permissions
```

---

## Pattern 9: Subagent Messaging & Delegation
Version `v1.2.9+` adds prompt syntax to route messages directly to subagents:

```bash
# Inside the TUI prompt, address subagents directly:
@code-reviewer Check the uncommitted diff in src/controllers/

# In automation, spawn a custom subagent:
agy --agent security-auditor -p "Scan dependencies in go.mod for CVEs" \
  --dangerously-skip-permissions
```

---

## Pattern 10: Enforcing Strict JSON Schemas
Version `v1.1.8+` and `v1.2.14+` enforce structured output conforming to a JSON schema:

```bash
# Schema must have "type": "object" at root
agy -p "Extract all HTTP routes defined in router.go" \
  --output-format stream-json \
  --json-schema '{"type":"object","properties":{"routes":{"type":"array","items":{"type":"string"}}},"required":["routes"]}' \
  --dangerously-skip-permissions
```

---

## Pattern 11: Custom Markdown Agent Definition
Define custom agents in `.agents/agents/<name>.md` or `~/.gemini/config/agents/<name>.md`:

```markdown
---
name: devops-specialist
description: Infrastructure as Code and CI/CD automation engineer
mainAgent: true
subagent: true
hidden: false
model: pro
commandExecutionPolicy: ask
inheritCustomizations: true
rules:
  - .agents/rules/security-baseline.md
agents:
  - terraform-linter
---

# DevOps Specialist Agent
You are an expert site reliability and automation engineer. Enforce immutable infrastructure, least-privilege IAM policies, and reproducible pipelines.
```

Launch with:
```bash
agy --agent devops-specialist -i "Review our GitHub Actions workflow"
```

---

## Pattern 12: Controlling Reasoning Effort
Version `v1.1.5+` and `v1.2.11+` support multi-tier reasoning effort (`low`, `medium`, `high`, `xhigh`, `max`):

```bash
# Launch session with maximum reasoning depth
agy --effort max -p "Formal proof of correctness for lock-free ring buffer" \
  --dangerously-skip-permissions

# Inside interactive TUI:
/effort high
```

---

## Pattern 13: Audio Dictation with Remote Forwarding
Version `v1.1.21+` provides speech-to-text dictation into the prompt:

```bash
# On local laptop (client forwarding local microphone):
agy mic-serve --addr 127.0.0.1:4713

# In remote SSH session:
# Press F5 or type /voice in prompt to begin dictation
```

---

## Pattern 14: Handling Headless Errors in CI/CD
Version `v1.2.6+` and `v1.2.10+` return exit code `3` and print a structured `AGY_ERROR` JSON payload on fatal failures:

```bash
#!/usr/bin/env bash
set -eo pipefail

OUTPUT=$(agy -p "Run database migration" --dangerously-skip-permissions 2> /tmp/agy_err.log) || EXIT_CODE=$?

if [ "${EXIT_CODE:-0}" -eq 3 ]; then
  echo "AGY Fatal Error Detected:"
  grep "^AGY_ERROR:" /tmp/agy_err.log | jq .
  exit 1
fi
```

---

## Pattern 15: Hot-Reloading Discovered Skills
Version `v1.2.4+` allows reloading skills without restarting long-running sessions:

```bash
# Inside interactive TUI prompt:
/skills reload
```
