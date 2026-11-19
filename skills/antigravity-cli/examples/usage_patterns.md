# agy CLI Usage Patterns (v1.5.12)

Runnable, copy-pasteable recipes for automating, scripting, and orchestrating Google's Antigravity CLI (`agy` v1.5.12). Host coding agents (Claude Code, Codex, OpenCode) should prefer ephemeral `--settings` overrides over `--dangerously-skip-permissions`.

## Contents

**Headless Delegation & Recovery**
- [Pattern 1: Scoped Headless Implementation & Verification](#pattern-1-scoped-headless-implementation--verification)
- [Pattern 2: Two-Phase Plan Critique and Automated Execution](#pattern-2-two-phase-plan-critique-and-automated-execution)
- [Pattern 3: Pre-Flight Quota Check & `GEMINI_API_KEY` Handling](#pattern-3-pre-flight-quota-check--gemini_api_key-handling)
- [Pattern 4: Detecting Incomplete Runs, Denials & `AGY_ERROR` in CI/CD](#pattern-4-detecting-incomplete-runs-denials--agy_error-in-cicd)
- [Pattern 5: Ephemeral Session Overrides (`--settings`)](#pattern-5-ephemeral-session-overrides---settings)

**Structured Output & Streaming**
- [Pattern 6: Enforcing Strict JSON Schemas (`--json-schema`)](#pattern-6-enforcing-strict-json-schemas---json-schema)
- [Pattern 7: Continuous Multi-Turn Streaming (`stream-json`)](#pattern-7-continuous-multi-turn-streaming-stream-json)
- [Pattern 8: Headless Skill Execution & Zero-Turn Inspection](#pattern-8-headless-skill-execution--zero-turn-inspection)

**Models, Workflows & Multi-Agent**
- [Pattern 9: Discovering Models & Controlling Reasoning Effort](#pattern-9-discovering-models--controlling-reasoning-effort)
- [Pattern 10: Headless Parallel Fan-Out with `run_workflow`](#pattern-10-headless-parallel-fan-out-with-run_workflow)
- [Pattern 11: Custom Markdown Agent & Fail-Fast Guard](#pattern-11-custom-markdown-agent--fail-fast-guard)
- [Pattern 12: Auto Mode for Hands-Off Refactors](#pattern-12-auto-mode-for-hands-off-refactors)
- [Pattern 13: Coordinating Interactive Swarms (`/swarm`)](#pattern-13-coordinating-interactive-swarms-swarm)

**Plugins, Hooks & MCP**
- [Pattern 14: Validating, Installing & Auditing Plugins in CI](#pattern-14-validating-installing--auditing-plugins-in-ci)
- [Pattern 15: Optional Plugin API Keys (`auth.json`)](#pattern-15-optional-plugin-api-keys-authjson)
- [Pattern 16: Managing MCP Servers via CLI](#pattern-16-managing-mcp-servers-via-cli)
- [Pattern 17: Always-Loaded MCP Tools & Subagent Root Correlation](#pattern-17-always-loaded-mcp-tools--subagent-root-correlation)
- [Pattern 18: Screening Prompts with `PreInvocation` Hooks](#pattern-18-screening-prompts-with-preinvocation-hooks)
- [Pattern 19: Loop-Safe `Stop` Hooks (`continuationActive`)](#pattern-19-loop-safe-stop-hooks-continuationactive)

**Remote, ACP & Authentication**
- [Pattern 20: Embedding `agy` via ACP, Cloud Remote Agents & Remote Control](#pattern-20-embedding-agy-via-acp-cloud-remote-agents--remote-control)
- [Pattern 21: Direct Gemini API Key Authentication (`GEMINI_API_KEY`)](#pattern-21-direct-gemini-api-key-authentication-gemini_api_key)

---

## Pattern 1: Scoped Headless Implementation & Verification

Run non-interactive implementation with ephemeral `--settings`, explicit `--print-timeout`, and JSON output. `--print-timeout` defaults to `0s` (unlimited, waiting on background tasks); always set an explicit bound in automation.

```bash
AGY=$(command -v agy || echo "$HOME/.local/bin/agy")

"$AGY" --disable-slash-commands \
  --print="Generate a resilient Go HTTP server with graceful shutdown in cmd/server/main.go and run go test ./..." \
  --add-dir . \
  --mode accept-edits \
  --settings '{"toolPermission":"always-proceed","artifactReviewPolicy":"always-proceed"}' \
  --output-format json \
  --print-timeout 12m > /tmp/codegen.json 2> /tmp/codegen.err
```

---

## Pattern 2: Two-Phase Plan Critique and Automated Execution

Phase 1 runs a guaranteed read-only review (`"toolPermission": "request-review"` with `"permissions": {"allow": []}` allows workspace `read_file`, clears saved `global` allow rules, and blocks file writes and shell commands even when saved `settings.json` has `"always-proceed"`). Phase 2 resumes the exact same conversation to implement approved changes.

```bash
#!/usr/bin/env bash
set -euo pipefail

AGY=$(command -v agy || echo "$HOME/.local/bin/agy")

# Phase 1: Pure read-only adversarial critique (never use --mode plan for reviews)
"$AGY" --disable-slash-commands \
  --print="Review src/auth/ and critique docs/jwt-v2-plan.md. Max 6 findings ranked by severity with file:line citations, ending with a verdict." \
  --add-dir . \
  --settings '{"agentMode":"","toolPermission":"request-review","permissions":{"allow":[]}}' \
  --effort high \
  --output-format json \
  --print-timeout 10m > /tmp/plan.json 2> /tmp/plan.err

CONV_ID=$(jq -r '.conversation_id // empty' /tmp/plan.json)
if [ -z "$CONV_ID" ]; then
  echo "Phase 1 failed; inspect /tmp/plan.err" >&2
  exit 1
fi
jq -r '.response' /tmp/plan.json

# Phase 2: Resume the same conversation and implement approved steps + tests
"$AGY" --disable-slash-commands \
  --print="Implement Step 1 and Step 2 directly in the codebase and run pytest tests/test_auth.py." \
  --conversation "$CONV_ID" \
  --add-dir . \
  --mode accept-edits \
  --settings '{"toolPermission":"always-proceed","artifactReviewPolicy":"always-proceed"}' \
  --output-format json \
  --print-timeout 15m > /tmp/exec.json 2> /tmp/exec.err

jq -r '{status, finished_reason, denied: (.denied_actions // [] | length), response}' /tmp/exec.json
```

---

## Pattern 3: Pre-Flight Quota Check & `GEMINI_API_KEY` Handling

Check zero-turn `/usage` JSON before heavy runs. `remaining_fraction` is the share **left** (`1` = 100% left); `1970-01-01T00:00:00Z` means no reset window is active. Stay silent at `>= 0.20`, warn below `0.20`, and switch pools at `0`. When connected via `GEMINI_API_KEY`, `/usage` exits `2` because API keys have no account quota pools.

```bash
#!/usr/bin/env bash
set -euo pipefail

AGY=$(command -v agy || echo "$HOME/.local/bin/agy")
TARGET_MODEL="${1:-}"

if [ -n "${GEMINI_API_KEY:-}" ]; then
  echo "GEMINI_API_KEY is set; skipping OAuth /usage quota check (billed in AI Studio)."
  exit 0
fi

USAGE_JSON="/tmp/agy-usage.json"
USAGE_ERR="/tmp/agy-usage.err"
RC=0
"$AGY" -p "/usage" --output-format json > "$USAGE_JSON" 2> "$USAGE_ERR" || RC=$?

if [ "$RC" -eq 2 ]; then
  echo "No account quota pools available for current auth method (exit 2)."
  exit 0
elif [ "$RC" -ne 0 ] || [ ! -s "$USAGE_JSON" ]; then
  cat "$USAGE_ERR" >&2
  exit "$([ "$RC" -ne 0 ] && echo "$RC" || echo 1)"
fi

# Discover available slugs with `agy models` or inspect all buckets below 20%
jq -r --arg id "$TARGET_MODEL" '
  .command.data.groups[].buckets[]
  | select($id == "" or .id == $id)
  | (if .reset_time == "1970-01-01T00:00:00Z" then "" else " (resets \(.reset_time))" end) as $reset
  | if .remaining_fraction <= 0 then
      "\(.name) [\(.id)]: 0% left\($reset) — switch --model to another pool"
    elif .remaining_fraction < 0.20 then
      "\(.name) [\(.id)]: \((.remaining_fraction * 100) | round)% left\($reset)"
    else
      empty
    end' "$USAGE_JSON"
```

---

## Pattern 4: Detecting Incomplete Runs, Denials & `AGY_ERROR` in CI/CD

Always check exit code and `[ -s out.json ]` **before** running `jq`, and strip the `AGY_ERROR:` prefix before piping stderr JSON to `jq`. Exit `3` signals either a fatal API/model error or `--print-timeout` terminating active background tasks.

```bash
#!/usr/bin/env bash
set -euo pipefail

AGY=$(command -v agy || echo "$HOME/.local/bin/agy")
OUT="/tmp/ci-review.json"
ERR="/tmp/ci-review.err"
EXIT_CODE=0

"$AGY" --disable-slash-commands \
  --print="Audit src/auth/ for security vulnerabilities. Max 8 findings with file:line citations." \
  --add-dir . \
  --settings '{"agentMode":"","toolPermission":"request-review","permissions":{"allow":[]}}' \
  --output-format json \
  --print-timeout 10m > "$OUT" 2> "$ERR" || EXIT_CODE=$?

if [ "$EXIT_CODE" -eq 3 ]; then
  echo "AGY exit 3 (API error or background-task timeout):" >&2
  AGY_ERR=$(sed -n 's/^AGY_ERROR:[[:space:]]*//p' "$ERR")
  if [ -n "$AGY_ERR" ]; then
    printf '%s\n' "$AGY_ERR" | jq . >&2 || printf '%s\n' "$AGY_ERR" >&2
  else
    cat "$ERR" >&2
  fi
  exit 3
elif [ "$EXIT_CODE" -ne 0 ]; then
  echo "AGY exited $EXIT_CODE:" >&2
  cat "$ERR" >&2
  exit "$EXIT_CODE"
fi

if [ ! -s "$OUT" ] || ! jq empty "$OUT" 2>/dev/null; then
  echo "Empty or invalid JSON output; inspect $ERR" >&2
  exit 1
fi

REASON=$(jq -r '.finished_reason // empty' "$OUT")
DENIED=$(jq -r '(.denied_actions // []) | length' "$OUT")
CONV_ID=$(jq -r '.conversation_id // empty' "$OUT")

if [ "$DENIED" -gt 0 ]; then
  echo "Tools denied in headless mode:" >&2
  jq '.denied_actions' "$OUT" >&2
  exit 1
fi

case "$REASON" in
  "")             jq -r '.response' "$OUT" ;;
  print_timeout)  echo "Partial (timed out). Resume with --conversation $CONV_ID" >&2; exit 1 ;;
  interrupted)    echo "Interrupted before completion." >&2; exit 1 ;;
  error)          echo "Turn ended with error. Check $ERR." >&2; exit 1 ;;
  *)              echo "Unexpected finished_reason: $REASON. Check $ERR." >&2; exit 1 ;;
esac
```

---

## Pattern 5: Ephemeral Session Overrides (`--settings`)

`--settings` applies session-only `settings.json` overrides (inline JSON starting with `{`, or a JSON file path) without modifying `~/.gemini/antigravity-cli/settings.json`.

```bash
# Force strict review interactively (prompts on every tool including read_file;
# do NOT use "strict" for headless -p file reviews — use "request-review" + "permissions":{"allow":[]} instead)
agy -i "Audit sensitive billing scripts" \
  --settings '{"agentMode":"","toolPermission":"strict"}'

# Unattended interactive /plan drafting without mutating saved settings.json
agy --mode plan -i "Plan the migration from REST to gRPC" \
  --settings '{"artifactReviewPolicy":"always-proceed"}'

# Load a checked-in CI settings template file
agy --disable-slash-commands \
  --print="Run nightly repository health check" \
  --add-dir . \
  --settings ./ci-agy-settings.json \
  --output-format json \
  --print-timeout 10m
```

---

## Pattern 6: Enforcing Strict JSON Schemas (`--json-schema`)

`--json-schema` requires `--output-format json` (or `stream-json`; otherwise exits `1`) and a root `"type": "object"` schema. In `--output-format json`, read the pre-parsed `.structured_output` object directly.

```bash
AGY=$(command -v agy || echo "$HOME/.local/bin/agy")

"$AGY" --disable-slash-commands \
  --print="Extract all HTTP routes defined in router.go." \
  --add-dir . \
  --settings '{"agentMode":"","toolPermission":"request-review","permissions":{"allow":[]}}' \
  --output-format json \
  --json-schema '{"type":"object","properties":{"routes":{"type":"array","items":{"type":"string"}}},"required":["routes"]}' \
  --print-timeout 5m > /tmp/routes.json 2> /tmp/routes.err

jq '.structured_output' /tmp/routes.json
```

---

## Pattern 7: Continuous Multi-Turn Streaming (`stream-json`)

Use `--input-format stream-json --output-format stream-json` for persistent multi-turn drivers. **Omit `-p`** (or pass `-p ""`): `-p` is a string flag, so bare `agy -p --add-dir ...` exits `2`. CLI-answered slash commands like `/usage` exit `2` in `stream-json` input mode and must be run as separate `agy -p "/usage"` calls.

```bash
# Launch a continuous read-only streaming worker (omit -p)
agy \
  --add-dir . \
  --settings '{"agentMode":"","toolPermission":"request-review","permissions":{"allow":[]}}' \
  --input-format stream-json \
  --output-format stream-json
```

**Send turns to stdin (one JSON object per line; `"event": "user"` is required):**
```json
{"event":"user","message":{"content":"Audit src/auth/jwt.go for token replay risks"}}
{"event":"user","message":{"content":"Summarize the top 2 fixes as a unified diff without editing files"}}
```

**Receive typed NDJSON events on stdout (`step_type` for tools is `"tool"`):**
```json
{"event":"init","conversation_id":"conv_9f81a","init":{"cwd":"/workspace","permission_mode":"request-review","tools":["read_file","run_command"]}}
{"event":"step_update","step_update":{"conversation_id":"conv_9f81a","step_index":1,"state":"ACTIVE","step_type":"tool","tool_name":"read_file","tool_info":{"name":"read_file","parameters":{"Path":"/workspace/src/auth/jwt.go"}}}}
{"event":"result","result":{"conversation_id":"conv_9f81a","status":"SUCCESS","response":"Found missing jti replay cache...","duration_seconds":4.12,"num_turns":1,"usage":{"input_tokens":1280,"cache_read_tokens":840,"output_tokens":320,"total_tokens":1600}}}
```

---

## Pattern 8: Headless Skill Execution & Zero-Turn Inspection

Print mode automatically expands leading `/<skill-name>` and read-only slash commands. Pass `--disable-slash-commands` when a prompt contains arbitrary or path-prefixed text.

```bash
# Headless read-only skill run on uncommitted git diff (cd into repo first for prefix rules)
cd /path/to/repo
agy -p "/jbd-code-review-skill review the uncommitted changes" \
  --add-dir . \
  --settings '{"agentMode":"","toolPermission":"request-review","permissions":{"allow":["command(git status)","command(git diff)","command(git log)","command(git show)"]}}' \
  --output-format json \
  --print-timeout 10m

# Treat leading slash as literal text
agy --disable-slash-commands \
  --print="/etc/nginx/nginx.conf has a proxy_pass loop; explain why." \
  --add-dir . \
  --settings '{"agentMode":"","toolPermission":"request-review","permissions":{"allow":[]}}' \
  --output-format json

# Zero-turn inspection commands (0 turns, 0 tokens, no saved conversation)
agy -p "/config" --output-format json
agy -p "/model" --output-format json
agy -p "/skills" --output-format json
agy -p "/permissions" --output-format json
agy -p "/agents" --output-format json
agy -p "/hooks" --output-format json
```

---

## Pattern 9: Discovering Models & Controlling Reasoning Effort

Available models and effort ceilings depend on the sign-in method. Always check `agy models` first — an unavailable `--effort` fails fast with exit `1` and lists valid levels. `--effort max` is Claude-only and exits `1` on Gemini models.

```bash
# List exact <model>-<effort> slugs available to this account
agy models

# High-effort reasoning on Gemini 4 Neon (xhigh available on Google sign-in & GEMINI_API_KEY)
agy --model gemini-4-neon --effort xhigh \
  --disable-slash-commands \
  --print="Verify lock-free ring buffer memory ordering in pkg/ring/ring.go." \
  --add-dir . \
  --settings '{"agentMode":"","toolPermission":"request-review","permissions":{"allow":[]}}' \
  --output-format json \
  --print-timeout 12m

# Enterprise / AGY_ADC_AUTH Orchestrator (plans with Carbon/Argon High, executes with Hydrogen High)
agy --model gemini-pro-orchestrator \
  --disable-slash-commands \
  --print="Implement pagination across pkg/api/ and update integration tests." \
  --add-dir . \
  --mode accept-edits \
  --settings '{"toolPermission":"always-proceed","artifactReviewPolicy":"always-proceed"}' \
  --output-format json \
  --print-timeout 15m

# Claude max effort (only on accounts with Claude models enabled)
agy --model claude-opus-5-5 --effort max \
  --print="Critique the distributed consensus design in docs/raft.md." \
  --add-dir . \
  --settings '{"agentMode":"","toolPermission":"request-review","permissions":{"allow":[]}}' \
  --output-format json \
  --print-timeout 12m
```

---

## Pattern 10: Headless Parallel Fan-Out with `run_workflow`

Ask `agy` to invoke its built-in `run_workflow` tool (bundled Python; no local Python needed) to fan work out across parallel agents with structured schemas and aggregate the result in one `-p` run.

```bash
agy --disable-slash-commands \
  --print="Use run_workflow to review every .go file under internal/auth/ in parallel:
1. Phase 'Discover': find all .go files under internal/auth/ excluding *_test.go.
2. Phase 'Review': call parallel() with agent(..., model='flash', workspace='inherit', schema={'type':'object','properties':{'file':{'type':'string'},'issues':{'type':'array','items':{'type':'string'}}},'required':['file','issues']}) checking each file for nil dereferences and context leaks.
3. Return a single prioritized Markdown report. Do not edit files." \
  --add-dir . \
  --settings '{"agentMode":"","toolPermission":"request-review","permissions":{"allow":[]}}' \
  --output-format json \
  --print-timeout 15m > /tmp/workflow-review.json 2> /tmp/workflow-review.err
```

---

## Pattern 11: Custom Markdown Agent & Fail-Fast Guard

Define reusable agents in `.agents/agents/<name>.md` or `~/.gemini/config/agents/<name>.md`. Unknown `--agent` names fail fast with exit `1` in headless `-p` mode.

```markdown
---
name: security-auditor
description: Application security reviewer focused on auth, injection, and supply-chain risks
mainAgent: true
subagent: true
hidden: false
model: pro
commandExecutionPolicy: ask
inheritCustomizations: true
rules:
  - .agents/rules/security-baseline.md
---

# Security Auditor
Review code strictly for exploitable vulnerabilities. Cite exact file:line locations and remediation steps.
```

```bash
#!/usr/bin/env bash
set -euo pipefail

if ! agy --agent security-auditor \
  --disable-slash-commands \
  --print="Scan go.mod and internal/auth/ for known security issues." \
  --add-dir . \
  --settings '{"agentMode":"","toolPermission":"request-review","permissions":{"allow":[]}}' \
  --output-format json \
  --print-timeout 8m > /tmp/audit.json 2> /tmp/audit.err; then
  echo "agy failed (unknown agent or runtime error); available agents:" >&2
  agy -p "/agents" --output-format json | jq '.command.data.agents' >&2
  exit 1
fi
```

---

## Pattern 12: Auto Mode for Hands-Off Refactors

In Auto mode, file edits and safe commands proceed without prompts, irreversible actions ask, and sandbox-escape checks run on a fast reviewer model. Auto mode also honors `Non-Workspace Access` from `/config`.

```bash
# Interactive Auto mode session
agy --mode auto -i "Rename the Config struct to Settings across the repo and verify go test ./..."
```

---

## Pattern 13: Coordinating Interactive Swarms (`/swarm`)

Use `/swarm` in the interactive TUI for collaborative parallel teams over a shared message board (preview & quota-heavy; `/swarm board` is TUI-only). For headless `-p` fan-outs, use [Pattern 10 (`run_workflow`)](#pattern-10-headless-parallel-fan-out-with-run_workflow).

```text
# Inside interactive TUI:
/swarm Migrate all v1 API handlers to v2, update integration tests, and verify error schemas

# Press Alt+S to open /swarm board (Tab switches Roster/Posts/Message, o opens member, Esc returns)
# Press Alt+X to dismiss the finished Swarm tray row
```

---

## Pattern 14: Validating, Installing & Auditing Plugins in CI

`agy plugin validate [target]` inspects a local folder/`plugin.json`, a `.zip` archive, an `https://` archive URL, a GitHub repo/subfolder URL, or a `marketplace.json` without installing or running anything. Findings appear under `.findings[]`.

```bash
#!/usr/bin/env bash
set -euo pipefail

# Validate a local plugin strictly and capture machine-readable findings
EXIT_CODE=0
agy plugin validate ./my-plugin --strict --json > /tmp/plugin-report.json || EXIT_CODE=$?
jq '.findings[]' /tmp/plugin-report.json
if [ "$EXIT_CODE" -ne 0 ]; then
  exit "$EXIT_CODE"
fi

# Validate a remote GitHub subfolder plugin before installing
agy plugin validate "https://github.com/owner/repo/tree/main/plugins/analyzer" --strict

# Install from official marketplace and audit installed state in JSON
agy plugin install analyzer@antigravity-plugins-official
agy plugin list --json | jq '.[] | {name, enabled}'
```

---

## Pattern 15: Optional Plugin API Keys (`auth.json`)

Mark a plugin's API key provider `"optional": true` so users can run at a free/unauthenticated rate limit without configuring credentials first.

```json
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

## Pattern 16: Managing MCP Servers via CLI

Manage `~/.gemini/config/mcp_config.json` with `agy mcp`. Remote servers use Streamable HTTP (`/mcp`); use `--` before commands carrying dash flags.

```bash
# List configured MCP servers
agy mcp list

# Add a stdio MCP server
agy mcp add fs npx -y @modelcontextprotocol/server-filesystem /workspace

# Add a stdio server with environment variables
agy mcp add --env GITHUB_TOKEN=ghp_secret gh -- docker run -i ghcr.io/github/mcp

# Add an authenticated Streamable HTTP MCP server
agy mcp add --header "Authorization: Bearer my_api_key" internal-docs https://docs.internal.net/mcp

# Toggle or remove a server
agy mcp disable internal-docs
agy mcp enable internal-docs
agy mcp remove fs
```

---

## Pattern 17: Always-Loaded MCP Tools & Subagent Root Correlation

Mark high-frequency MCP tools with `_meta.antigravity/alwaysLoad` to bypass on-demand discovery, and read `_meta["antigravity.google/root_conversation_id"]` in `tools/call` to trace nested subagent calls back to the root conversation.

```json
{
  "name": "search_docs",
  "description": "Search internal documentation",
  "inputSchema": { "type": "object", "properties": { "query": { "type": "string" } } },
  "_meta": { "antigravity/alwaysLoad": true }
}
```

```python
def handle_tool_call(request: dict) -> str:
    root_id = request.get("_meta", {}).get("antigravity.google/root_conversation_id")
    return f"Correlated tool call for root conversation: {root_id}"
```

---

## Pattern 18: Screening Prompts with `PreInvocation` Hooks

In `hooks.json`, every top-level key is a **named hook**. `PreInvocation` receives camelCase JSON on stdin (`lastUserInput`, `hookEventName`, `cwd`) and terminates before calling the model by returning `injectSteps` with an `errorMessage` plus `"terminationBehavior": "terminate"` (causing headless `-p` to exit `1`).

```json
{
  "prompt-screener": {
    "PreInvocation": [
      {
        "type": "command",
        "command": "./hooks/screen-prompt.sh",
        "timeout": 10
      }
    ]
  }
}
```

```bash
#!/usr/bin/env bash
# hooks/screen-prompt.sh
set -euo pipefail
INPUT=$(cat)
PROMPT=$(echo "$INPUT" | jq -r '.lastUserInput // ""')

if echo "$PROMPT" | grep -qE '(AKIA[0-9A-Z]{16}|BEGIN PRIVATE KEY)'; then
  cat <<'EOF'
{
  "injectSteps": [
    {
      "errorMessage": {
        "userMessage": "Blocked: prompt appears to contain production credentials."
      }
    }
  ],
  "terminationBehavior": "terminate"
}
EOF
  exit 0
fi

echo '{}'
```

---

## Pattern 19: Loop-Safe `Stop` Hooks (`continuationActive`)

Check camelCase `.continuationActive` on stdin so a `Stop` hook never triggers an infinite loop. Return `{"decision":"continue","reason":"..."}` to ask the agent to keep working, or `{}` to let it stop.

```json
{
  "verify-tests-once": {
    "Stop": [
      {
        "type": "command",
        "command": "./hooks/verify-tests-once.sh",
        "timeout": 60
      }
    ]
  }
}
```

```bash
#!/usr/bin/env bash
# hooks/verify-tests-once.sh
set -euo pipefail
INPUT=$(cat)
ACTIVE=$(echo "$INPUT" | jq -r '.continuationActive // false')

# Already continuing from an earlier Stop hook? Allow exit instead of looping.
if [ "$ACTIVE" = "true" ]; then
  echo '{}'
  exit 0
fi

CWD=$(echo "$INPUT" | jq -r '.cwd // "."')
if [ -f "$CWD/go.mod" ] && ! (cd "$CWD" && go test ./... >/dev/null 2>&1); then
  echo '{"decision":"continue","reason":"go test ./... failed; fix the failing tests before finishing."}'
  exit 0
fi

echo '{}'
```

---

## Pattern 20: Embedding `agy` via ACP, Cloud Remote Agents & Remote Control

Use `agy acp` for persistent IDE/editor integration, `--remote-agent` for cloud workspaces, and `agy remote-control` for background daemon access. Note that `--remote-agent` and `--remote-control` require OAuth/account auth and exit `2` when connected with `GEMINI_API_KEY`.

```bash
# Authenticate once over SSH or in a headless terminal for ACP
agy acp --login

# Start stdio ACP server with per-session worker isolation and IDE MCP bridge
agy acp --process-per-session --ide-mcp-port 9123 --add-dir /path/to/repo

# Cloud Remote Agent session (pass --mode accept-edits so remote edits proceed without prompts)
agy --remote-agent --project "backend-monorepo" --mode accept-edits \
  -i "Investigate flaky CI test in pkg/queue"

# Background Remote Control daemon lifecycle
agy remote-control start --name "build-server-01"
agy remote-control status
agy remote-control stop
```

---

## Pattern 21: Direct Gemini API Key Authentication (`GEMINI_API_KEY`)

Authenticate directly against the Gemini API without OAuth sign-in. Skip `/usage`, `/quota`, `/credits`, `--remote-agent`, and `--remote-control` (all exit `2` with `GEMINI_API_KEY`).

```bash
export GEMINI_API_KEY="AIzaSy-your-real-key"
export GOOGLE_GEMINI_BASE_URL="https://generativelanguage.googleapis.com"

# Gemini 4 Neon (low, medium, high, xhigh) is available with GEMINI_API_KEY
agy --model gemini-4-neon --effort xhigh \
  --disable-slash-commands \
  --print="Compare Go channel fan-out vs errgroup for bounded worker pools." \
  --settings '{"modelProvider":"gemini","agentMode":"","toolPermission":"request-review","permissions":{"allow":[]}}' \
  --output-format json \
  --print-timeout 5m
```
