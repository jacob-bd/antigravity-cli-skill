# Delegating Work to `agy` (Host Agent Playbook — v1.5.12)

Playbook for AI coding agents (**Claude Code, Codex CLI, Codex App, OpenCode, Cursor, Aider**) and CI orchestrators delegating reviews, codebase research, parallel fan-outs, or scoped implementation to Google's Antigravity CLI (`agy` v1.5.12).

> **Quick start:** For binary resolution, the 3-row quota table, and the 3 canonical base recipes (pure read-only critique, Git diff review, scoped write + test), see [../SKILL.md](../SKILL.md). This playbook covers task selection, prompt contracts, advanced workflows, hardened wrappers, and exact headless schemas.

## Contents
- [Delegate to `agy` vs. Keep Locally](#delegate-to-agy-vs-keep-locally)
- [Orchestration Mode Selector](#orchestration-mode-selector)
- [6-Part Delegation Prompt Contract & Safe Input Passing](#6-part-delegation-prompt-contract--safe-input-passing)
- [Advanced Task Recipes](#advanced-task-recipes)
- [Ephemeral `--settings` Schema & Permission Rules](#ephemeral---settings-schema--permission-rules)
- [Hardened Wrappers, Completion Checks & Timeout Sizing](#hardened-wrappers-completion-checks--timeout-sizing)
- [Streaming NDJSON (`stream-json`) Driver & Consumer](#streaming-ndjson-stream-json-driver--consumer)
- [Zero-Turn Inspection Commands](#zero-turn-inspection-commands)
- [Security, Sandbox, Auto Mode, Remote Agents & ACP](#security-sandbox-auto-mode-remote-agents--acp)

---

## Delegate to `agy` vs. Keep Locally

| Situation | Recommended Owner | Why |
| :--- | :--- | :--- |
| Adversarial plan, architecture, security, or PR review | **Delegate to `agy`** | Cross-model second opinion (Gemini 4 Carbon/Hydrogen/Neon/Argon) avoids host confirmation bias. |
| Large-repo archaeology or multi-file dependency tracing | **Delegate to `agy`** | Protects host context window; `agy` uses embedded `ripgrep` and up to 512 tool calls/turn on Gemini. |
| Multi-module parallel audits or staged pipelines | **Delegate to `agy` (`run_workflow`)** | Bundled Python fans out typed `agent()` calls in parallel with schema-checked aggregation. |
| Conflict-free speculative or parallel coding | **Delegate to `agy` in a `git worktree`** | Keeps main working tree untouched while `agy` edits and runs tests in isolation. |
| Multi-MB PDF, screenshot, or UI mockup analysis | **Delegate to `agy`** | Handles large PDFs and visual inputs natively for text-only host agents. |
| Small file reads, 1-file surgical edits, or tight conversational loops | **Keep in host agent** | Avoids subprocess startup latency, duplicate context loading, and extra quota spend. |
| Target `agy` quota pool is at `0% left` and no alternate pool fits | **Keep in host agent** | Prevents exit-3 quota failures and retries. |

---

## Orchestration Mode Selector

| Need | Primitive | Notes |
| :--- | :--- | :--- |
| Single review, audit, or CI gate | `agy -p "..." --output-format json` | Single JSON envelope with `.conversation_id`, `.response`, `.structured_output`, `.denied_actions[]`. |
| Iterative fix loop or timeout recovery | `agy --conversation <id> -p "..."` | Resumes exact session history; host inspects `git diff` or findings between turns. |
| Persistent multi-turn driver or live tool progress | `agy --input-format stream-json --output-format stream-json` | Zero process-restart overhead; omit bare `-p` (or pass `-p ""`). |
| Headless parallel fan-out / map-reduce | Prompt `agy -p` to run `run_workflow` | Bundled Python (`agent`, `parallel`, `pipeline`, `phase`, `schema`, `workspace: "share"\|"branch"`). |
| Interactive collaborative multi-agent team | `/swarm` (TUI) | Shared board + live `/swarm board`; preview & quota-heavy; board UI is interactive-only. |
| Persistent IDE / editor integration | `agy acp [--process-per-session]` | Stdio JSON-RPC ACP server with `session/resume`, context-window reporting, and clean cancellation. |
| Native Python orchestration service | `google-antigravity` Python SDK (`pip install google-antigravity`) | Async `Agent` context manager with typed `response.thoughts` and `response.tool_calls` streams. |

---

## 6-Part Delegation Prompt Contract & Safe Input Passing

Bound every headless prompt so the worker does not wander into a timeout:

1. **Goal & role**: State the job explicitly (`"Adversarial security reviewer"`, `"Root-cause investigator"`).
2. **Exact scope paths**: Name the files/directories to inspect. If plans or specs live outside the repo (e.g., `/tmp/specs` or `~/.claude/plans`), grant access with repeatable `--add-dir <path>`.
3. **Write & command boundaries**: State whether edits are forbidden or restricted to specific paths and test commands.
4. **Evidence rule**: Require `file:line` citations for every finding and tell the agent to mark unknowns explicitly rather than guessing.
5. **Hard output cap or schema**: Cap findings and length (`"Max 6 findings ranked by severity, 600 words, final verdict"`) or pass `--json-schema`.
6. **Exploration budget**: Reserve synthesis time (`"Spend at most 5 minutes exploring, then write the final report"`).

### Safe Input Passing Rules

- **Never pipe stdin alongside `-p "<text>"`**: In default `--input-format text` mode, `agy -p "<text>"` ignores piped stdin (so `git diff | agy -p "Review this"` drops the diff silently), and `cat diff.txt | agy -p ""` exits `1` (`empty prompt`). Instead, let `agy` read files and run allowlisted `git diff` natively (which also preserves full codebase cross-referencing), write long prompts to a file and pass `--print="$(cat /tmp/prompt.txt)"`, use a Python `subprocess.run` argv array, or use `--input-format stream-json`.
- **Use `--print='...'` and `--disable-slash-commands` for arbitrary text**: `--print=...` prevents dash-prefixed text from being parsed as flags, and `--disable-slash-commands` prevents prompts starting with a path like `"/src/auth/jwt.go has a race..."` from triggering slash-command expansion. Omit `--disable-slash-commands` only when intentionally invoking `/<skill-name>` or zero-turn `/usage`, `/config`, etc.
- **Always `cd` into the target repo when using `command(git ...)` allowlists**: Permission rules are prefix-matched on the command string. If `agy` is started outside the repo with `--add-dir /other/repo`, the model runs `cd /other/repo && git diff`, which does **not** match `command(git diff)` and lands in `.denied_actions[]`.

---

## Advanced Task Recipes

> **Critical read-only rule:** `--settings` only overrides keys you specify. If the user's saved `settings.json` has `"toolPermission": "always-proceed"`, bare `--settings '{"agentMode":""}'` still allows file writes. Always set `"toolPermission": "request-review"` with `"permissions": {"allow": []}` (pure read-only file reviews: allows workspace `read_file`, clears saved `global` allow rules, and blocks writes/commands) or `"permissions": {"allow": [...]}` (allowlisted read-only commands such as `git diff`). Do **not** use `"toolPermission": "strict"` for headless file reviews: `"strict"` denies even `read_file` in `-p` mode (reserve it for interactive TUI or zero-tool pure-text prompts).

### 1. PR / Branch-Range Code Review with Concrete `--json-schema`

Requires `--output-format json` (or `stream-json`; omitting it exits `1`) and a root `"type": "object"` schema. On success, `agy` populates a pre-parsed `.structured_output` object alongside `.response`. `--json-schema` enforcement also persists cleanly across `--conversation <id>` follow-ups.

```bash
AGY=$(command -v agy || echo "$HOME/.local/bin/agy")

cat <<'EOF' > /tmp/agy-review-schema.json
{
  "type": "object",
  "properties": {
    "verdict": { "type": "string", "enum": ["approve", "request_changes", "needs_discussion"] },
    "summary": { "type": "string" },
    "findings": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "severity": { "type": "string", "enum": ["critical", "high", "medium", "low"] },
          "file": { "type": "string" },
          "line": { "type": "integer" },
          "title": { "type": "string" },
          "evidence": { "type": "string" },
          "suggested_fix": { "type": "string" }
        },
        "required": ["severity", "file", "line", "title", "evidence", "suggested_fix"]
      }
    }
  },
  "required": ["verdict", "summary", "findings"]
}
EOF

# Must run inside the target Git repo so command(git ...) prefix rules match
cd /path/to/repo
"$AGY" --disable-slash-commands \
  --print="Review git diff main...HEAD for bugs, concurrency races, and security issues. Cross-reference callers in the repo. Do not edit files. Max 8 findings with file:line evidence." \
  --add-dir . \
  --settings '{"agentMode":"","toolPermission":"request-review","permissions":{"allow":["command(git status)","command(git diff)","command(git log)","command(git show)"]}}' \
  --json-schema /tmp/agy-review-schema.json \
  --output-format json \
  --print-timeout 10m > /tmp/review.json 2> /tmp/review.err || EXIT_CODE=$?

if [ "${EXIT_CODE:-0}" -eq 0 ] && [ -s /tmp/review.json ]; then
  jq '.structured_output' /tmp/review.json
fi
```

### 2. Zero-Write Patch Proposal (Diff-Only Coding)

Generates a concrete unified diff without touching disk, allowing the host agent to inspect and apply changes cleanly. `"toolPermission": "request-review"` with `"permissions": {"allow": []}` allows workspace reads while clearing saved `global` allow rules and blocking file writes and shell commands.

```bash
AGY=$(command -v agy || echo "$HOME/.local/bin/agy")

"$AGY" --disable-slash-commands \
  --print="Read src/auth/jwt.go and src/auth/jwt_test.go. Propose a minimal unified diff that enforces jti replay validation and adds a unit test. Do not modify any files; output the unified diff and a 3-bullet risk summary." \
  --add-dir . \
  --settings '{"agentMode":"","toolPermission":"request-review","permissions":{"allow":[]}}' \
  --output-format json \
  --print-timeout 8m > /tmp/patch-proposal.json 2> /tmp/patch-proposal.err
```

### 3. Isolated `git worktree` Coding (Conflict-Free Parallel Edits)

Prevents `agy` edits from colliding with uncommitted changes in the host agent's main working tree.

```bash
#!/usr/bin/env bash
set -euo pipefail

AGY=$(command -v agy || echo "$HOME/.local/bin/agy")
WORKTREE="/tmp/agy-worktree-fix-timeout"
BRANCH="agy/fix-refund-timeout"

cleanup() {
  git worktree remove --force "$WORKTREE" 2>/dev/null || true
  git branch -D "$BRANCH" 2>/dev/null || true
}
trap cleanup EXIT

git worktree add "$WORKTREE" -b "$BRANCH" HEAD

# Option A: Broad write + test execution inside isolated worktree
(
  cd "$WORKTREE"
  "$AGY" --disable-slash-commands \
    --print="Fix the timeout bug in src/payment/processor.py causing test_refund_timeout to fail, then run pytest tests/test_payment.py. Keep changes minimal, commit nothing, and report changed files plus test output." \
    --add-dir "$WORKTREE" \
    --mode accept-edits \
    --settings '{"toolPermission":"always-proceed","artifactReviewPolicy":"always-proceed"}' \
    --output-format json \
    --print-timeout 12m > /tmp/agy-fix.json 2> /tmp/agy-fix.err
)

# Capture tracked and newly created files as a patch before cleanup removes the worktree
git -C "$WORKTREE" status --short
git -C "$WORKTREE" add -N .
git -C "$WORKTREE" diff > /tmp/agy-fix.patch
# Verify cleanly in the main repo before applying: git apply --check /tmp/agy-fix.patch
```

*Least-privilege alternative* (allows edits in `accept-edits` mode and permits **only** `pytest`, denying all other shell commands; `cd "$WORKTREE"` so `command(pytest)` prefix matching succeeds, and do not duplicate `agentMode` inside `--settings` when `--mode` is already set):

```bash
(
  cd "$WORKTREE"
  "$AGY" --disable-slash-commands \
    --print="Fix src/payment/processor.py and run pytest tests/test_payment.py." \
    --add-dir "$WORKTREE" \
    --mode accept-edits \
    --settings '{"toolPermission":"request-review","permissions":{"allow":["command(pytest)"]}}' \
    --output-format json \
    --print-timeout 12m > /tmp/agy-fix.json 2> /tmp/agy-fix.err
)
```

### 4. Codebase Research & Allowlisted Root-Cause Debugging

**Pure read-only codebase archaeology** (zero writes, zero shell commands):

```bash
"$AGY" --disable-slash-commands \
  --print="Trace how OAuth tokens are loaded, refreshed, and attached to outbound HTTP requests under pkg/auth/ and pkg/client/. Cite file:line for every hop, list edge cases, and stay under 500 words." \
  --add-dir . \
  --settings '{"agentMode":"","toolPermission":"request-review","permissions":{"allow":[]}}' \
  --output-format json \
  --print-timeout 8m > /tmp/research.json 2> /tmp/research.err
```

**Allowlisted root-cause debugging** (can run targeted test/repro commands and inspect Git history, while edits remain blocked):

```bash
cd /path/to/repo
"$AGY" --disable-slash-commands \
  --print="Reproduce the failure in tests/test_queue.py using pytest and git log. Identify the exact root cause and introducing commit if visible. Do not edit files; cite file:line and command output." \
  --add-dir . \
  --settings '{"agentMode":"","toolPermission":"request-review","permissions":{"allow":["command(pytest)","command(git log)","command(git blame)","command(git show)"]}}' \
  --output-format json \
  --print-timeout 10m > /tmp/debug.json 2> /tmp/debug.err
```

### 5. Headless `run_workflow` Parallel Fan-Out

`run_workflow` is a built-in model tool backed by bundled Python (no host Python required). It supports `phase()`, `parallel()`, `pipeline()`, `agent(prompt, schema, role, workspace="inherit"|"share"|"branch", model="inherit"|"flash_lite"|"flash"|"pro")`, `call_tool()`, and `trajectory()`. Prefer `run_workflow` over `/swarm` for headless `-p` fan-outs because every child agent returns schema-validated output cleanly into one final response.

```bash
"$AGY" --disable-slash-commands \
  --print="Use the run_workflow tool to audit every package under services/ in parallel:
1. Phase 'Discover': list top-level subdirectories under services/.
2. Phase 'Audit': run parallel() with one read-only agent per directory (model='flash', workspace='inherit', schema={'type':'object','properties':{'service':{'type':'string'},'risks':{'type':'array','items':{'type':'string'}}},'required':['service','risks']}) checking for missing auth checks or unclosed resources.
3. Synthesize a consolidated Markdown table ranked by severity. Do not edit any files." \
  --add-dir . \
  --settings '{"agentMode":"","toolPermission":"request-review","permissions":{"allow":[]}}' \
  --output-format json \
  --print-timeout 15m > /tmp/fanout.json 2> /tmp/fanout.err
```

### 6. Parallel Host-Shell Fan-Out (Bounded Concurrency)

Launch 2–4 independent read-only reviews concurrently from the host shell with isolated output files. Keep concurrency modest (2–4) to avoid triggering per-minute rate limits.

```bash
#!/usr/bin/env bash
set -euo pipefail

AGY=$(command -v agy || echo "$HOME/.local/bin/agy")
OUT_DIR="/tmp/agy-parallel-audit"
mkdir -p "$OUT_DIR"

TARGETS=("src/auth" "src/billing" "src/webhooks")
PIDS=()

for dir in "${TARGETS[@]}"; do
  slug=$(echo "$dir" | tr '/' '_')
  "$AGY" --disable-slash-commands \
    --print="Audit ${dir}/ for security flaws and unhandled errors. Max 5 findings with file:line citations. Do not edit files." \
    --add-dir . \
    --settings '{"agentMode":"","toolPermission":"request-review","permissions":{"allow":[]}}' \
    --output-format json \
    --print-timeout 10m > "${OUT_DIR}/${slug}.json" 2> "${OUT_DIR}/${slug}.err" &
  PIDS+=("$!")
done

FAILED=0
for pid in "${PIDS[@]}"; do
  if ! wait "$pid"; then
    FAILED=1
  fi
done

for f in "${OUT_DIR}"/*.json; do
  if [ -s "$f" ]; then
    echo "=== $f ==="
    jq -r '{status, finished_reason, denied: (.denied_actions // [] | length), response}' "$f"
  fi
done
exit "$FAILED"
```

### 7. Multimodal & Large-PDF Offloading

Text-only host agents can delegate multi-MB PDFs, architecture diagrams, or UI screenshots directly to `agy` by granting `--add-dir` to the asset folder:

```bash
"$AGY" --disable-slash-commands \
  --print="Inspect /tmp/assets/spec-v4.pdf and /tmp/assets/checkout-bug.png. Extract all API error-code requirements from the PDF and list every discrepancy visible in the screenshot. Cite page numbers." \
  --add-dir /tmp/assets \
  --settings '{"agentMode":"","toolPermission":"request-review","permissions":{"allow":[]}}' \
  --output-format json \
  --print-timeout 10m > /tmp/multimodal.json 2> /tmp/multimodal.err
```

### 8. Timeout Recovery & Multi-Turn Follow-Up (`--conversation <id>`)

Every `-p --output-format json` run returns `.conversation_id` (even on `print_timeout`). Resume on the exact same context to finalize without re-reading the codebase. Note that `num_turns` increments while `duration_seconds` reports cumulative elapsed wall time since conversation creation.

```bash
CONV_ID=$(jq -r '.conversation_id // empty' /tmp/review.json)

"$AGY" --disable-slash-commands \
  --print="Stop exploring. Using only the evidence already gathered, write the final report now: max 6 findings ranked by severity, 600 words max, ending with a verdict. Do not call any tools." \
  --conversation "$CONV_ID" \
  --add-dir . \
  --settings '{"agentMode":"","toolPermission":"request-review","permissions":{"allow":[]}}' \
  --output-format json \
  --print-timeout 3m > /tmp/recovery.json 2> /tmp/recovery.err
```

---

## Ephemeral `--settings` Schema & Permission Rules

`--settings` accepts either inline JSON (any value starting with `{`) or a path to a JSON file. Overrides apply only to the current invocation and are never written to `~/.gemini/antigravity-cli/settings.json`. Unknown keys or invalid enum values fail fast with exit `1`.

| Key | Valid Values | Headless Effect |
| :--- | :--- | :--- |
| `agentMode` | `""`, `"accept-edits"`, `"auto"`, `"plan"` | `""` selects default request-review mode (never pass `"default"` — exits `1`). |
| `toolPermission` | `"strict"`, `"request-review"`, `"always-proceed"` | `"request-review"` allows workspace `read_file`, blocks unapproved writes/commands, and honors `permissions.allow` (pair with `"permissions": {"allow": []}` for pure file reviews). `"strict"` prompts on **every** tool including `read_file`, so headless `-p` auto-denies file reads (use `"strict"` only for interactive TUI or zero-tool pure-text prompts). `"always-proceed"` auto-approves tools even if `agentMode: ""`. (`""` and `"ask"` exit `1`.) |
| `artifactReviewPolicy` | `"asks-for-review"`, `"always-proceed"`, `"agent-decides"` | Controls plan/artifact review gates (`""` and `"request-review"` exit `1`). |
| `permissions` | `{"allow": [...], "deny": [...], "ask": [...]}` | Rule syntax: `command(git diff)` (prefix match by default), `regex:...`, `mcp(server/tool)`. On Windows `cmd`/PowerShell, command rules require an exact match. |
| `modelProvider` | `"gemini"` (with `GEMINI_API_KEY`) | Routes directly to the Gemini API without OAuth sign-in. |

**Permission scope caveat:** Passing `permissions` inside `--settings` replaces the **`global`** scope rules for the session, while **`shared`** scope rules remain active. For headless read-only file reviews, pass `"toolPermission": "request-review"` with `"permissions": {"allow": []}` to clear saved `global` command rules while keeping workspace `read_file` enabled (use `"toolPermission": "strict"` only when no tools at all—not even `read_file`—should run).

**Admin & protected-command ceilings:** Neither `"toolPermission": "always-proceed"` nor `--dangerously-skip-permissions` can bypass `agy`'s built-in protected-command denylist or Gemini Enterprise admin-enforced review policies (restricted actions are denied in headless mode rather than prompting).

---

## Hardened Wrappers, Completion Checks & Timeout Sizing

### Model, Effort & Timeout Sizing Rules

- **Discover live slugs**: Run `"$AGY" models` to see exact slugs for the active sign-in. `gemini-4-neon` offers `low`, `medium`, `high`, and `xhigh` on Google sign-in and `GEMINI_API_KEY`. On Google sign-in, Carbon, Hydrogen, and Argon top out at `high`; Hydrogen also offers `xhigh` with `GEMINI_API_KEY`, `AGY_ADC_AUTH`, or an enterprise gateway. `gemini-pro-orchestrator` appears where enabled (planning with Carbon/Argon High and executing with Hydrogen High). `--effort max` is Claude-only (`claude-opus-5-5-max`, `claude-sonnet-5-5-max`) and exits `1` on Gemini models.
- **Size timeouts for retries and deep thinking**: `agy` automatically retries per-minute rate limits for up to **4 minutes**, waits up to **10 minutes** for silent high-effort model thinking, retries transient connection drops automatically, and caps post-result shutdown at **30 seconds**. Set `--print-timeout` to `8m`–`15m` for high-effort tasks (never `1m`–`2m`), and set the host subprocess wall-clock timeout to `print_timeout + 60s`.
- **Do not blindly retry exit `3` quota errors**: Spend-cap, weekly quota, and low AI-credit balance errors stop immediately with exit `3` and an `AGY_ERROR: {...}` line on stderr. Inspect `AGY_ERROR` and switch `--model` to another quota pool instead of looping.
- **`GEMINI_API_KEY` exit code `2`**: When using `GEMINI_API_KEY`, `/usage`, `/quota`, `/credits`, `--remote-agent`, and `--remote-control` exit with code `2`. Skip quota pre-flight checks when `GEMINI_API_KEY` is active.

### Mandatory Completion & Denial Verification

Never trust exit code `0` alone, and never run `jq` or `json.loads` before checking exit code and non-empty stdout:

| Signal | Clean Completion | Action Needed |
| :--- | :--- | :--- |
| Exit code | `0` | `1`: bad flag/schema/unknown `--agent`/no AI credits/hook block; `2`: flag conflict, bare `-p --flag`, interactive command in `-p`, or `GEMINI_API_KEY` unsupported command; `3`: `AGY_ERROR` API failure or `--print-timeout` killing background tasks. |
| `finished_reason` | **absent** | `print_timeout`, `interrupted`, or `error` (can occur even with exit `0`). |
| `denied_actions` | **absent / empty** | Non-empty array (e.g., `[{"action":"write_file","display_name":"Edit"}]` or `[{"action":"command","display_name":"Bash"}]`); stderr also prints a `jetski: no output produced` notice. |
| `response` / `structured_output` | non-empty | Empty with `num_turns: 0` indicates nothing was produced. |

**Silent-fallback gotchas:** `--mode default` prints a warning and **continues with exit `0`** using the saved mode; an unrecognized `--output-format` (e.g., `yaml`) silently falls back to `text` and exits `0`. Always pass exact values (`accept-edits|auto|plan`, `text|json|stream-json`).

### Hardened Bash Wrapper

```bash
#!/usr/bin/env bash
set -euo pipefail

AGY=$(command -v agy || echo "$HOME/.local/bin/agy")
OUT="/tmp/agy-run.json"
ERR="/tmp/agy-run.err"
EXIT_CODE=0

"$AGY" --disable-slash-commands \
  --print="Audit src/auth/ for token validation bugs. Max 6 findings with file:line citations." \
  --add-dir . \
  --settings '{"agentMode":"","toolPermission":"request-review","permissions":{"allow":[]}}' \
  --output-format json \
  --print-timeout 10m > "$OUT" 2> "$ERR" || EXIT_CODE=$?

if [ "$EXIT_CODE" -ne 0 ]; then
  echo "agy exited $EXIT_CODE" >&2
  AGY_ERR=$(sed -n 's/^AGY_ERROR:[[:space:]]*//p' "$ERR")
  if [ -n "$AGY_ERR" ]; then
    printf '%s\n' "$AGY_ERR" | jq . >&2 || printf '%s\n' "$AGY_ERR" >&2
  else
    tail -n 20 "$ERR" >&2
  fi
  exit "$EXIT_CODE"
fi

if [ ! -s "$OUT" ] || ! jq empty "$OUT" 2>/dev/null; then
  echo "agy exited 0 without valid JSON stdout; inspect $ERR" >&2
  exit 1
fi

REASON=$(jq -r '.finished_reason // empty' "$OUT")
DENIED=$(jq -r '(.denied_actions // []) | length' "$OUT")
CONV_ID=$(jq -r '.conversation_id // empty' "$OUT")

if [ -n "$REASON" ]; then
  echo "Incomplete ($REASON). Resume with: agy --conversation $CONV_ID ..." >&2
  exit 1
elif [ "$DENIED" -gt 0 ]; then
  echo "Tools denied:" >&2
  jq '.denied_actions' "$OUT" >&2
  exit 1
fi

jq -r '.response' "$OUT"
```

### Hardened Python Subprocess Wrapper

```python
import json
import os
import shutil
import subprocess
from pathlib import Path

agy_bin = shutil.which("agy") or os.path.expanduser("~/.local/bin/agy")
out_path = Path("/tmp/agy.out.json")
err_path = Path("/tmp/agy.err.log")

prompt = (
    "Refactor db/migrations/004_users.sql to add an index on email "
    "and run pytest tests/test_migrations.py. Keep changes minimal."
)
cmd = [
    agy_bin,
    "--disable-slash-commands",
    f"--print={prompt}",
    "--add-dir", ".",
    "--mode", "accept-edits",
    "--settings", json.dumps({
        "toolPermission": "always-proceed",
        "artifactReviewPolicy": "always-proceed",
    }),
    "--output-format", "json",
    "--print-timeout", "15m",
]

try:
    with out_path.open("w") as out_f, err_path.open("w") as err_f:
        # Host timeout = print-timeout (15m) + 60s buffer for shutdown cap
        proc = subprocess.run(cmd, stdout=out_f, stderr=err_f, text=True, timeout=16 * 60)
except subprocess.TimeoutExpired:
    raise SystemExit("Host timeout expired while waiting for agy")

raw = out_path.read_text().strip()
err_text = err_path.read_text()

if proc.returncode != 0:
    for line in err_text.splitlines():
        if line.startswith("AGY_ERROR:"):
            raise SystemExit(f"agy exit {proc.returncode}: {line[len('AGY_ERROR:'):].strip()}")
    raise SystemExit(f"agy exit {proc.returncode}: {err_text[-500:]}")

try:
    data = json.loads(raw) if raw else {}
except json.JSONDecodeError as exc:
    raise SystemExit(f"Invalid JSON on stdout: {exc}")

reason = data.get("finished_reason")
denied = data.get("denied_actions") or []
if reason:
    outcome = f"incomplete:{reason}"
elif denied:
    outcome = "denied"
elif data.get("response") or data.get("structured_output"):
    outcome = "complete"
else:
    outcome = "empty"

print(proc.returncode, outcome, data.get("conversation_id"))
```

---

## Streaming NDJSON (`stream-json`) Driver & Consumer

`-p` / `--print` takes a string argument; passing bare `agy -p --input-format stream-json ...` exits `2`. For bidirectional streaming, **omit `-p`** (or pass `-p ""`). Send each turn on stdin as `{"event":"user","message":{"content":"..."}}` (optional `"role":"user"` inside `message`; omitting `"event"` exits `1`). CLI-answered slash commands like `/usage` exit `2` in `--input-format stream-json` mode and must be run as standalone `agy -p "/usage"` invocations.

### Verified Event Envelopes

```json
{"event":"init","conversation_id":"9f1c7a2e-0000-0000-0000-000000000000","init":{"model":"gemini-3.8-flash-low","cwd":"/repo","tools":["read_file","run_command"],"permission_mode":"request-review"}}
{"event":"step_update","step_update":{"conversation_id":"9f1c7a2e-0000-0000-0000-000000000000","step_index":2,"state":"ACTIVE","step_type":"tool","tool_name":"run_command","tool_info":{"name":"run_command","parameters":{"CommandLine":"git status"}}}}
{"event":"step_update","step_update":{"conversation_id":"9f1c7a2e-0000-0000-0000-000000000000","step_index":2,"state":"DONE","step_type":"tool","tool_name":"run_command","duration_seconds":0.29,"tool_info":{"name":"run_command","parameters":{"CommandLine":"git status"}}}}
{"event":"step_update","step_update":{"conversation_id":"9f1c7a2e-0000-0000-0000-000000000000","step_index":3,"state":"DONE","step_type":"agent_response","text_delta":"Clean working tree.\n","duration_seconds":1.42,"usage":{"input_tokens":12400,"output_tokens":28,"thinking_tokens":22,"cache_read_tokens":9800,"total_tokens":12428}}}
{"event":"result","result":{"conversation_id":"9f1c7a2e-0000-0000-0000-000000000000","status":"SUCCESS","response":"Clean working tree.\n","duration_seconds":2.23,"num_turns":1,"usage":{"input_tokens":12400,"output_tokens":28,"thinking_tokens":22,"cache_read_tokens":9800,"total_tokens":12428}}}
```

- Tool steps use `"step_type": "tool"` (not `"tool_call"`), with `tool_name` and `tool_info` on `step_update` (`tool_info.error` appears on `state: "ERROR"`).
- Subagent steps carry `subagent_info`.
- `finished_reason` (`print_timeout`, `interrupted`, `error`) appears on `result` only when ended early; `denied_actions[]` appears when tools were auto-denied.

### Runnable Python `stream-json` Consumer

```python
import json
import os
import shutil
import subprocess
import sys

agy_bin = shutil.which("agy") or os.path.expanduser("~/.local/bin/agy")
cmd = [
    agy_bin,
    "--add-dir", ".",
    "--settings", json.dumps({
        "agentMode": "",
        "toolPermission": "request-review",
        "permissions": {"allow": []},
    }),
    "--input-format", "stream-json",
    "--output-format", "stream-json",
]

proc = subprocess.Popen(
    cmd, stdin=subprocess.PIPE, stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True
)
assert proc.stdin and proc.stdout

proc.stdin.write(
    json.dumps({"event": "user", "message": {"content": "Summarize the package layout in 5 bullets."}}) + "\n"
)
proc.stdin.flush()
proc.stdin.close()

final_result = None
for line in proc.stdout:
    line = line.strip()
    if not line:
        continue
    evt = json.loads(line)
    kind = evt.get("event")
    if kind == "step_update":
        su = evt.get("step_update", {})
        if su.get("step_type") == "tool" and su.get("state") == "ACTIVE":
            print(f"[tool] {su.get('tool_name')}", file=sys.stderr)
    elif kind == "result":
        final_result = evt.get("result")

rc = proc.wait(timeout=60)
if rc != 0 or not final_result:
    raise SystemExit(f"stream-json failed (rc={rc}): {proc.stderr.read() if proc.stderr else ''}")

print(json.dumps({
    "conversation_id": final_result.get("conversation_id"),
    "finished_reason": final_result.get("finished_reason"),
    "denied_actions": final_result.get("denied_actions", []),
    "response": final_result.get("response", ""),
}, indent=2))
```

---

## Zero-Turn Inspection Commands

These read-only slash commands run in `-p` mode with **0 turns and 0 tokens spent** and leave no conversation behind. Always pass `--output-format json` in scripts (text mode formats epoch-zero timestamps in local time, e.g. `1969-12-31 19:00 EST`, whereas JSON keeps canonical `1970-01-01T00:00:00Z`).

| Command | JSON Payload Path | Caveats |
| :--- | :--- | :--- |
| `agy -p "/usage" --output-format json` (alias `"/quota"`) | `.command.data.groups[].buckets[]` | Exits `2` with `GEMINI_API_KEY`; may include Claude buckets not shown in `agy models`. |
| `agy -p "/credits" --output-format json` | `.command.data` | Exits `1` on accounts without AI credits; exits `2` with `GEMINI_API_KEY`. |
| `agy -p "/config" --output-format json` | `.command.data.config` | Inspect active `agentMode`, `toolPermission`, `artifactReviewPolicy`, `nonWorkspaceAccess`. |
| `agy -p "/model" --output-format json` | `.command.data` | Shows active default model without calling the API. |
| `agy -p "/effort" --output-format json` | `.command.data` | Shows active reasoning effort level. |
| `agy -p "/skills" --output-format json` | `.command.data.skills[]` | Lists discovered global, workspace, and built-in skills. |
| `agy -p "/permissions" --output-format json` | `.command.data` | Inspect merged `global`/`shared`/session allow, deny, and ask rules (works with `--settings`). |
| `agy -p "/agents" --output-format json` | `.command.data.agents` | Lists available custom and built-in agents (unlike `agy agents`, which rejects `--output-format`). |
| `agy -p "/hooks" --output-format json` | `.command.data` | Lists active lifecycle hooks loaded from `hooks.json`. |
| `agy -p "/help" --output-format json` | `.command.data` | Lists every slash command answered non-interactively in `-p` mode. |
| `agy -p "/changelog" --output-format json` | `.command.data` | Returns release notes for the installed binary. |

---

## Security, Sandbox, Auto Mode, Remote Agents & ACP

- **Host outer sandbox escalation**: `agy` requires read/write access to `~/.gemini/antigravity-cli/`, outbound network, and credential/keychain access. Running `agy models`, `agy agents`, or `agy -p` inside a restrictive host sandbox (such as macOS Seatbelt or Linux Landlock) fails with `Failed to start: open ~/.gemini/antigravity-cli/installation_id: operation not permitted`. Escalate the `agy` invocation out of the host sandbox, and never nest `agy --sandbox` inside an outer sandbox.
- **`agy --sandbox` defaults**: Keeps `.git` read-only and blocks reads to `~/.config` (`$XDG_CONFIG_HOME`), Windows `AppData\Roaming`, `~/.git-credentials`, `~/Library/Application Support`, `~/.docker`, `~/.kube`, and `~/_netrc` by default (Git, Go, npm, and Python configs remain readable; additional paths require explicit sandbox allow rules).
- **Auto mode (`--mode auto`)**: Honors the `Non-Workspace Access` setting in `/config` for outside-workspace file reads. When reviewing commands that would leave the terminal sandbox on Gemini 4 Hydrogen, Auto mode uses medium thinking for faster decisions and fewer timeouts.
- **Cloud Remote Agents (`--remote-agent`)**: Default mode asks before each file edit in the remote workspace. Pass `--mode accept-edits` when delegating unattended remote edits. `--remote-agent` and `--remote-control` exit `2` when connected with `GEMINI_API_KEY`.
- **Persistent Editor Integration (`agy acp`)**: Runs a stdio JSON-RPC Agent Client Protocol server supporting `--process-per-session`, `--ide-mcp-port <port>`, `--agent <name>`, `--add-dir <path>`, `session/resume`, context-window usage reporting, and `agy acp --login`. Async replies, tool calls, and permission requests from background tasks/subagents stream immediately after a turn ends, and canceling a prompt or closing a session terminates its subagents and background tasks cleanly.
- **Skills & MCP Servers during delegation**: Auto-discovered skills load from `~/.gemini/config/skills/` and `<workspace>/.agents/skills/` (`.agents/skills.json` and `.agents/plugins.json` are confined inside the repo root). Verify MCP tools with `"$AGY" mcp list`; add remote servers using Streamable HTTP (`"$AGY" mcp add <name> https://example.internal/mcp`).
