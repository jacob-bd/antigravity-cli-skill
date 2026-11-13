# agy CLI Usage Patterns (v1.5.7)

Executable patterns and architectural recipes for automating, scripting, and orchestrating Google's Antigravity CLI. Host coding assistants (Codex, Claude Code, OpenCode) should prefer ephemeral `--settings` overrides over `--dangerously-skip-permissions`.

---

## Pattern 1: Non-Interactive Code Generation
Run single-prompt code generation headless with ephemeral `--settings` (`v1.5.7+`). Note that in `v1.2.6+`, `--print-timeout` defaults to `0s` (unlimited), waiting for completion and background tasks (`v1.5.0+`).

```bash
# Scoped non-interactive generation without --dangerously-skip-permissions
agy -p "Generate a resilient Go HTTP server with graceful shutdown" \
  --add-dir . \
  --mode accept-edits \
  --settings '{"toolPermission":"always-proceed","artifactReviewPolicy":"always-proceed"}'

# Explicitly bounded run with structured completion check
agy -p "Run extensive benchmark suite on all crypto routines" \
  --add-dir . \
  --mode accept-edits \
  --settings '{"toolPermission":"always-proceed","artifactReviewPolicy":"always-proceed"}' \
  --output-format json \
  --print-timeout 10m
```

---

## Pattern 2: Continuous Multi-Turn Streaming over Stdin/Stdout
Version `v1.1.15+` supports bidirectional, line-delimited NDJSON streaming via `--input-format stream-json` and `--output-format stream-json`. This allows persistent test harnesses, drivers, and background daemons to keep a continuous session alive without restarting the CLI:

```bash
# Launch a continuous streaming agent worker
agy -p \
  --add-dir . \
  --mode accept-edits \
  --settings '{"toolPermission":"always-proceed","artifactReviewPolicy":"always-proceed"}' \
  --input-format stream-json \
  --output-format stream-json
```

**Sending turns to stdin:**
```json
{"prompt": "Audit src/auth/jwt.go for token replay risks"}
{"prompt": "Write a patch for jwt.go that enforces jti validation"}
```

**Receiving structured events on stdout:**
```json
{"event": "init", "conversation_id": "conv_9f81a", "init": {"cwd": "/workspace", "permission_mode": "always-proceed"}}
{"event": "step_update", "step_update": {"step_index": 1, "state": "ACTIVE", "step_type": "tool_call", "tool_info": {"name": "view_file"}}}
{"event": "result", "result": {"conversation_id": "conv_9f81a", "status": "SUCCESS", "response": "Successfully implemented JTI replay cache...", "usage": {"input_tokens": 1280, "cache_read_tokens": 840, "output_tokens": 320}}}
```

---

## Pattern 3: Headless Slash-Command and Skill Execution
Version `v1.1.9+` automatically expands leading slash commands and installed skills in print mode (`-p`):

```bash
# Headless read-only skill run on uncommitted git diff
agy -p "/jbd-code-review-skill review the uncommitted changes" \
  --add-dir . \
  --settings '{"agentMode":"","permissions":{"allow":["command(git status)","command(git diff)","command(git log)","command(git show)"]}}'

# Opt out and treat leading slashes as literal text
agy -p "/not-a-command this is literal text" \
  --disable-slash-commands \
  --settings '{"agentMode":""}'
```

---

## Pattern 4: Zero-Turn Inspection of Read-Only Commands
Version `v1.1.11+` and `v1.1.12+` allow querying read-only slash commands in print mode to extract machine-readable JSON without starting an agent turn, without consuming quota, and without polluting conversation databases:

```bash
# Check quota buckets and reset windows in JSON (alias: "/quota")
agy -p "/usage" --output-format json

# List installed skills, active permission rules, or available custom agents
agy -p "/skills" --output-format json
agy -p "/permissions" --output-format json
agy -p "/agents" --output-format json

# Optional: inspect AI credits (note: exits 1 on accounts without AI credits)
agy -p "/credits" --output-format json
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

# Use session-only --settings or persist {"modelProvider": "gemini"} in settings.json
agy -p "List 3 advantages of Go over Python" \
  --settings '{"modelProvider":"gemini","agentMode":""}'
```

---

## Pattern 8: One-Shot Model Prompting Mid-Conversation
Version `v1.1.27+` allows executing a single prompt against another model without altering your session default (`v1.3.4+` lets base names work without `--effort`):

```bash
# Inside interactive TUI:
# Runs prompt on claude-sonnet-5-5-medium and reverts to default model next turn
/model claude-sonnet-5-5-medium Critique this architectural plan from a functional programming viewpoint

# In headless scripts, select models explicitly (list slugs with `agy models`):
agy --model gemini-4-carbon-high -p "High-complexity reasoning task" \
  --add-dir . --settings '{"agentMode":""}' --output-format json
```

---

## Pattern 9: Subagent Messaging & Delegation
Version `v1.2.9+` adds prompt syntax to route messages directly to subagents:

```bash
# Inside the TUI prompt, address subagents directly:
@code-reviewer Check the uncommitted diff in src/controllers/

# In automation, spawn a custom subagent in read-only mode:
agy --agent security-auditor -p "Scan dependencies in go.mod for CVEs" \
  --add-dir . --settings '{"agentMode":""}' --output-format json
```

---

## Pattern 10: Enforcing Strict JSON Schemas
Version `v1.1.8+` and `v1.2.14+` enforce structured output conforming to a JSON schema:

```bash
# Schema must have "type": "object" at root
agy -p "Extract all HTTP routes defined in router.go" \
  --add-dir . \
  --settings '{"agentMode":""}' \
  --output-format stream-json \
  --json-schema '{"type":"object","properties":{"routes":{"type":"array","items":{"type":"string"}}},"required":["routes"]}'
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
  --add-dir . --settings '{"agentMode":""}' --output-format json

# Inside interactive TUI:
/effort high
```

---

## Pattern 13: Two-Phase Plan Review and Automated Execution
Host agents (Codex, Claude Code, OpenCode) can coordinate complex changes through `agy` using native `--settings` without `--dangerously-skip-permissions`: first critique the plan read-only (`agentMode: ""`), then implement on the same conversation (`--mode accept-edits`):

```bash
#!/usr/bin/env bash
set -eo pipefail

# Phase 1: Adversarial plan review (read-only; returns critique directly in .response)
agy --print="Review architecture of src/auth/ and critique our migration plan to JWT v2. Max 6 findings ranked by severity, ending with a verdict." \
  --add-dir . \
  --settings '{"agentMode":""}' \
  --effort high \
  --output-format json \
  --print-timeout 10m > plan.json

CONV_ID=$(jq -r .conversation_id plan.json)
PLAN=$(jq -r .response plan.json)
echo "Review received from agy session $CONV_ID"

# (Host agent or human evaluates findings here...)

# Phase 2: Resume conversation and implement approved steps + run tests
agy --print="Implement Step 1 and Step 2 directly in the codebase and run tests." \
  --conversation "$CONV_ID" \
  --add-dir . \
  --mode accept-edits \
  --settings '{"toolPermission":"always-proceed","artifactReviewPolicy":"always-proceed"}' \
  --output-format json \
  --print-timeout 15m > exec.json
```

---

## Pattern 14: Handling Headless Errors in CI/CD
Version `v1.2.6+`, `v1.2.10+`, and `v1.5.0+` return exit code `3` and print a structured `AGY_ERROR` JSON payload on fatal API failures or when `--print-timeout` terminates active background tasks:

```bash
#!/usr/bin/env bash
set -eo pipefail

OUTPUT=$(agy -p "Run database migration" \
  --add-dir . --mode accept-edits \
  --settings '{"toolPermission":"always-proceed"}' \
  2> /tmp/agy_err.log) || EXIT_CODE=$?

if [ "${EXIT_CODE:-0}" -eq 3 ]; then
  echo "AGY Exit Code 3 Detected:"
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

---

## Pattern 16: Auto Mode for Hands-Off Refactors
Version `v1.3.5+` adds `--mode auto` (macOS, Linux, Windows 11 24H2+ in `v1.4.4+`, PAYG `AGY_ADC_AUTH` in `v1.5.6+`; default mode in `v1.5.7+` where available): file edits and safe commands are approved without prompting, irreversible actions ask, and sandbox escapes are reviewed by a Gemini Flash reviewer (`v1.5.4+`):

```bash
# Start an interactive Auto mode session
agy --mode auto -i "Rename the Config struct to Settings across the repo and fix the build"

# Inside the TUI, press Shift+Tab to cycle modes, or set Agent Mode in /config
```

---

## Pattern 17: Validating Plugins in CI
Version `v1.4.0+` and `v1.4.1+` validate a plugin (manifest, skills, subagents, rules) with the same rules the loader uses. `--strict` fails on warnings and `--json` emits a machine-readable report:

```bash
#!/usr/bin/env bash
set -eo pipefail

# Fail the pipeline on any error or warning
agy plugin validate ./my-plugin --strict

# Capture a JSON report for annotations
agy plugin validate ./my-plugin --strict --json > plugin-report.json || EXIT_CODE=$?
jq . plugin-report.json
exit "${EXIT_CODE:-0}"
```

---

## Pattern 18: Installing and Auditing Plugins
Version `v1.3.2+` adds the official marketplace, and `v1.3.4+` adds `--json` to `plugin list`:

```bash
# Install from the official marketplace without opening the CLI
agy plugin install <plugin-name>@antigravity-plugins-official

# Audit what's installed (marketplace and imported), including enabled state
agy plugin list --json | jq '.'

# Inside the TUI, browse and manage interactively
/plugin
```

---

## Pattern 19: Failing Fast on a Misspelled Agent
Version `v1.3.4+` makes `--agent` exit `1` in headless runs when the name matches no agent, instead of silently running the default agent. Use it as a guard in scripts:

```bash
if ! agy --agent security-auditor -p "Scan go.mod for CVEs" \
  --add-dir . --settings '{"agentMode":""}' --output-format json; then
  echo "agy failed (unknown agent or runtime error); available agents:"
  agy -p "/agents" --output-format json | jq '.command.data.agents'
  exit 1
fi
```

---

## Pattern 20: Compacting a Long Conversation
Version `v1.3.5+` adds `/compact` to free up the context window on demand:

```bash
# Inside the interactive TUI:
/compact                          # Summarize the conversation and end the turn
/compact Now continue with the DB migration   # Summarize, then answer the message
```

---

## Pattern 21: Unattended Plan Drafting Approval
Since `v1.3.2`, `--dangerously-skip-permissions` no longer auto-approves implementation plans in `/plan` mode. For unattended plan drafting without mutating `settings.json`, pass `artifactReviewPolicy: "always-proceed"` via `--settings` (`v1.5.7+`):

```bash
agy --mode plan -i "Plan the migration from REST to gRPC" \
  --settings '{"artifactReviewPolicy":"always-proceed"}'
```

---

## Pattern 22: Always-Loaded MCP Tools
Version `v1.3.3+` lets an MCP server mark a tool as always loaded so the agent gets it directly instead of discovering it on demand. In the server's `tools/list` response:

```json
{
  "name": "search_docs",
  "description": "Search internal documentation",
  "inputSchema": { "type": "object", "properties": { "query": { "type": "string" } } },
  "_meta": { "antigravity/alwaysLoad": true }
}
```

---

## Pattern 23: Detecting Incomplete Runs & Denials via JSON Output
Version `v1.4.11+` adds `finished_reason` (`print_timeout`, `interrupted`, `error`; omitted on clean runs) and headless mode surfaces `.denied_actions[]` when tools lack permission. In `v1.5.0+`, `--print-timeout` exits `3` if active background tasks had to be terminated:

```bash
#!/usr/bin/env bash
set -eo pipefail

agy -p "Review the auth module for security vulnerabilities" \
  --add-dir . \
  --settings '{"agentMode":""}' \
  --output-format json \
  --print-timeout 8m > out.json 2> err.log || EXIT_CODE=$?

REASON=$(jq -r '.finished_reason // empty' out.json)
DENIED=$(jq -r '.denied_actions | length // 0' out.json)
CONV_ID=$(jq -r '.conversation_id' out.json)

if [ "${EXIT_CODE:-0}" -eq 3 ]; then
  echo "Exit 3 (API error or background-task timeout). Check err.log."
elif [ "$DENIED" -gt 0 ]; then
  echo "Tools denied:" && jq '.denied_actions' out.json
else
  case "$REASON" in
    "")             echo "Complete: $(jq -r .response out.json)" ;;
    print_timeout)  echo "Partial (timed out). Resume with --conversation $CONV_ID" ;;
    interrupted)    echo "Canceled before completion." ;;
    error)          echo "Failed. Check err.log and AGY_ERROR on stderr." ;;
  esac
fi
```

---

## Pattern 24: Blocking Prompts with Hook Termination
Version `v1.4.11+` allows `PreInvocation` hooks to terminate a turn before the model is called. Useful for prompt screening, policy enforcement, or data loss prevention:

```json
// hooks.json
{
  "PreInvocation": [
    {
      "command": "./hooks/screen-prompt.sh"
    }
  ]
}
```

The hook script receives `lastUserInput` (`v1.4.7+`) in the JSON on stdin and can return:
```json
{
  "terminationBehavior": "terminate",
  "errorMessage": {
    "userMessage": "Blocked: prompt contains production credentials."
  }
}
```
In headless `-p` runs, this causes exit code `1`, enabling CI gate behavior.

---

## Pattern 25: Plugin Validation with Expanded Checks in CI
Version `v1.4.3+` through `v1.5.6+` expanded `agy plugin validate` to cover hooks (including warning on `${PLUGIN_ROOT}`), sidecars, component lists, context providers, kebab-case `name`, `category`, `author`/`homepage`/`repository`, and `logo`/`iconUri` paths. Findings appear under `.findings[]`:

```bash
#!/usr/bin/env bash
set -eo pipefail

# Full validation covering manifest, skills, agents, rules, hooks, sidecars, context providers
agy plugin validate ./my-plugin --strict --json > plugin-report.json || EXIT_CODE=$?

# Inspect warnings or errors
jq '.findings[]' plugin-report.json
exit "${EXIT_CODE:-0}"
```

---

## Pattern 26: Optional Plugin API Keys
Version `v1.4.12+` supports `"optional": true` for API key providers in `auth.json`, so plugins work without a key at a free rate limit:

```json
// auth.json
{
  "providers": [
    {
      "name": "ACME_API_KEY",
      "description": "API key for ACME service (optional, raises rate limits)",
      "optional": true
    }
  ]
}
```

---

## Pattern 27: MCP Root Conversation ID for Nested Subagents
Version `v1.4.4+` passes `_meta["antigravity.google/root_conversation_id"]` in MCP `tools/call` requests. MCP servers can use this to correlate tool calls from nested subagents back to the top-level conversation:

```python
# In an MCP server handler
def handle_tool_call(request):
    root_id = request.get("_meta", {}).get("antigravity.google/root_conversation_id")
    # Use root_id for logging, tracing, or session correlation
```

---

## Pattern 28: Pre-Flight Quota Check & Low-Usage Warning
Before delegating a heavy task, inspect zero-turn `/usage` JSON for the target model (`v1.1.11+`). Warn the user if `remaining_fraction < 0.20` (or `0`), or suggest switching to another quota pool (Gemini Pro, Gemini Flash, and Claude/GPT models maintain separate pools):

```bash
#!/usr/bin/env bash
set -euo pipefail

AGY=$(command -v agy || echo "$HOME/.local/bin/agy")
MODEL_ID="${1:-gemini-4-carbon-high}"

"$AGY" -p "/usage" --output-format json | jq -r --arg id "$MODEL_ID" '
  .command.data.groups[].buckets[]
  | select(.id == $id)
  | if .remaining_fraction <= 0 then
      "EXHAUSTED: \($id) at 0% (resets \(.reset_time)). Switch --model to another pool."
    elif .remaining_fraction < 0.20 then
      "LOW QUOTA: \($id) at \((.remaining_fraction * 100) | round)% (resets \(.reset_time)). Warn user before heavy runs."
    else
      "OK: \($id) at \((.remaining_fraction * 100) | round)% remaining."
    end'
```

---

## Pattern 29: Ephemeral Session Overrides (`--settings`)
Version `v1.5.7+` adds `--settings` (inline JSON or file path) to apply temporary `settings.json` keys for a single run without modifying `~/.gemini/antigravity-cli/settings.json`:

```bash
# Force strict request-review behavior (v1.5.7+ strict overrides auto and accept-edits)
agy -i "Audit sensitive billing scripts" \
  --settings '{"agentMode":"","toolPermission":"strict"}'

# Load a project-specific session settings template from file
agy -p "Run nightly repository check" \
  --add-dir . \
  --settings ./ci-agy-settings.json \
  --output-format json
```

---

## Pattern 30: Embedding `agy` via Agent Client Protocol (`agy acp`)
Version `v1.5.0+`–`v1.5.4+` lets editors and IDEs embed `agy` as a persistent ACP stdio server instead of spawning one-off `-p` processes:

```bash
# Authenticate once over SSH or in a headless terminal
agy acp --login

# Launch stdio ACP server with per-session worker isolation and IDE MCP bridge
agy acp --process-per-session --ide-mcp-port 9123 --add-dir /path/to/repo
```

---

## Pattern 31: Cloud Remote Agent Sessions (`--remote-agent`)
Version `v1.5.0+`–`v1.5.3+` runs conversations on cloud Remote Agents instead of locally (`--project` selects the remote project by ID or name; cloud plugin skills are available in `v1.5.2+`):

```bash
# Open a remote workspace session on a named cloud project
agy --remote-agent --project "backend-monorepo" -i "Investigate flaky CI test in pkg/queue"
```

---

## Pattern 32: Coordinating Parallel Swarms (`/swarm`)
Version `v1.5.5+`–`v1.5.6+` adds `/swarm` for multi-worker parallel execution over a shared message board (note: quota heavy; `/swarm board` is interactive-only):

```text
# Inside interactive TUI:
/swarm Migrate all v1 API handlers to v2, update integration tests, and verify error schemas

# Press Alt+S to open /swarm board (Tab switches Roster/Posts/Message, o opens member, Esc returns)
# Press Alt+X to dismiss the finished Swarm tray row
```

---

## Pattern 33: Loop-Safe `Stop` Hooks (`continuation_active`)
Version `v1.5.4+` and `v1.5.6+` pass `hookEventName`, `cwd`, `toolCall.id`, and `continuation_active` to hook commands on stdin so `Stop` hooks never loop infinitely:

```bash
#!/usr/bin/env bash
# hooks/verify-tests-once.sh
INPUT=$(cat)
ACTIVE=$(echo "$INPUT" | jq -r '.continuation_active // false')

# Already continuing from a prior Stop hook? Allow exit instead of looping.
if [ "$ACTIVE" = "true" ]; then
  exit 0
fi

# Otherwise inspect cwd and ask the agent to run tests before stopping
CWD=$(echo "$INPUT" | jq -r '.cwd')
if [ -f "$CWD/go.mod" ] && ! (cd "$CWD" && go test ./... >/dev/null 2>&1); then
  echo '{"decision":"block","reason":"go test ./... failed; fix tests before finishing."}'
fi
```
