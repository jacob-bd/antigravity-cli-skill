# Delegating Work to `agy` (Host Agent Playbook)

This playbook is for AI coding agents (**Codex CLI, Codex App, Claude Code, OpenCode, Cursor, Aider**) delegating reviews, second-opinion reasoning, or implementation tasks to Google's Antigravity CLI (`agy` v1.5.7).

Delegating to `agy` preserves host context, gives access to Gemini 4 Carbon / Hydrogen / Argon and Gemini 3.8 Flash, and provides independent cross-model verification.

---

## Pre-Flight: Binary & Quota Check (Zero Cost)

1. **Resolve the executable** (non-login shells often omit `~/.local/bin`):
   ```bash
   AGY=$(command -v agy || echo "$HOME/.local/bin/agy")
   ```
2. **Check model quota before heavy delegation** (`0` turns, `0` tokens spent):
   ```bash
   "$AGY" -p "/usage" --output-format json
   ```
   Check `.command.data.groups[].buckets[]` for the model you plan to use (`id`, `name`, `remaining_fraction`, `reset_time`):
   - **Warn the user** before launching if `remaining_fraction < 0.20` (or `0`), including the percentage left and `reset_time` (ISO-8601 UTC).
   - **Quota pool fallback**: Gemini Pro, Gemini Flash, and Claude/GPT models use **separate quota pools** (`groups[].description`). If your preferred model's pool is low or exhausted, suggest switching `--model` to an available pool (`"$AGY" models` lists slugs).
   - Do not run `"/credits"` unconditionally in scripts; accounts without AI credits exit `1`.

```bash
# Example: inspect remaining fraction for gemini-4-carbon-high
"$AGY" -p "/usage" --output-format json | jq -r '
  .command.data.groups[].buckets[]
  | select(.id == "gemini-4-carbon-high")
  | "\(.name): \((.remaining_fraction * 100) | round)% remaining (resets \(.reset_time))"'
```

---

## Native Headless Mechanics (Zero `--dangerously-skip-permissions`)

Why host agents must avoid `--dangerously-skip-permissions`:
- Host coding agents (Codex CLI, Codex App, Claude Code) flag `--dangerously-skip-permissions` in command-safety rules and halt to ask the human for approval even for read-only reviews.
- In `agy` `-p` mode, unapproved tools **never hang on stdin** — they fail immediately and appear in stdout JSON under `.denied_actions[]`.
- `agy` `v1.5.7` adds ephemeral `--settings '<json>'` overrides that configure permissions for the single invocation without touching `~/.gemini/antigravity-cli/settings.json`.

Core rules:
1. **Always pass `--add-dir .`** (or target path): `agy` tracks persistent workspace mappings; `cd` alone does not scope a directory.
2. **Never pass `--mode plan` for critiques or code reviews**: `--mode plan` is an interactive plan-drafting mode that writes an internal `plan.md` artifact and stops asking for plan approval. Use `--settings '{"agentMode":""}'` so `agy` reads workspace files natively, returns its critique directly in `.response`, and engine-blocks edits/shell commands.
3. **Never pass `--mode default`** (rejected with a warning; valid `--mode` values are `accept-edits`, `auto`, `plan`). Always set `--mode` or `--settings` explicitly because `v1.5.7+` defaults unconfigured sessions to `auto` where available and warns on Business/Enterprise accounts.
4. **Do not nest `--sandbox` inside an outer agent sandbox**: If the host agent runs in macOS Seatbelt or Linux Landlock (e.g., Codex), nested `--sandbox` fails or blocks `~/.gitconfig` and state files. Request host network/sandbox access for the `agy` call itself when needed.
5. **Set `--print-timeout` (e.g. `8m`–`15m`)**: Default is `0s` (unlimited, waiting on background tasks in `v1.5.0+`). Always run the completion check below before trusting output.
6. **Attach prompt text with `--print='...'`** when passing programmatic or dash-prefixed text to avoid CLI flag parsing ambiguity.

---

## Native Delegation Recipes

### 1. Plan / Architecture Review (Read-Only, Zero Prompts)

Runs workspace file reads/searches natively, engine-blocks writes and shell commands, and avoids both `--dangerously-skip-permissions` and `--mode plan`.

```bash
"$AGY" --print="Perform an adversarial plan review of docs/rfc-001.md. Challenge assumptions, edge cases, failure modes, and data loss risks. Return at most 6 findings ranked by severity, ending with a verdict. Reserve time for the final report." \
  --add-dir . \
  --settings '{"agentMode":""}' \
  --effort high \
  --output-format json \
  --print-timeout 10m > out.json 2> err.log
```

### 2. Git Diff / Code Review (Read-Only Git Inspection)

Allows read-only Git commands (`status`, `diff`, `log`, `show`) while engine-blocking file edits and arbitrary commands.

```bash
"$AGY" --print="Review the uncommitted git diff in this repository for race conditions, security vulnerabilities, and edge-case bugs. Return at most 8 findings categorized by severity." \
  --add-dir . \
  --settings '{"agentMode":"","permissions":{"allow":["command(git status)","command(git diff)","command(git log)","command(git show)"]}}' \
  --output-format json \
  --print-timeout 8m > out.json 2> err.log
```

*Structured schema variant:*
```bash
"$AGY" --print="Audit src/auth/ for vulnerabilities. Return findings conforming to schema." \
  --json-schema ./review_schema.json \
  --add-dir . \
  --settings '{"agentMode":""}' \
  --output-format json \
  --print-timeout 8m > out.json 2> err.log
```

### 3. Scoped Implementation & Test Execution (Write Mode)

Uses `--mode accept-edits` plus ephemeral `toolPermission: "always-proceed"` and `artifactReviewPolicy: "always-proceed"`. Works uniformly across Personal, Pro/Ultra, and Business/Enterprise accounts without triggering host `--dangerously-skip-permissions` heuristics.

```bash
"$AGY" --print="Fix the bug in src/payment/processor.py causing test_refund_timeout to fail. Run pytest tests/test_payment.py to verify. Keep changes minimal." \
  --add-dir . \
  --mode accept-edits \
  --settings '{"toolPermission":"always-proceed","artifactReviewPolicy":"always-proceed"}' \
  --output-format json \
  --print-timeout 10m > out.json 2> err.log
```

*Tighter command allowlist alternative* (restrict shell execution strictly to the test runner):
```bash
"$AGY" --print="Fix the bug in src/payment/processor.py and run pytest tests/test_payment.py." \
  --add-dir . \
  --mode accept-edits \
  --settings '{"agentMode":"accept-edits","permissions":{"allow":["command(pytest)"]}}' \
  --output-format json \
  --print-timeout 10m > out.json 2> err.log
```

### 4. Follow-Up & Timeout Recovery (`--conversation <id>`)

Every completion returns `.conversation_id`. Resume on the exact same context with delta instructions:

```bash
CONVERSATION_ID=$(jq -r .conversation_id out.json)

"$AGY" --print="Stop all exploration. Using only the evidence already collected, produce the final report now: at most 6 findings ranked by severity, 600 words maximum, ending with a verdict. Do not call any tools." \
  --conversation "$CONVERSATION_ID" \
  --add-dir . \
  --settings '{"agentMode":""}' \
  --output-format json \
  --print-timeout 3m > recovery.json 2> err.log
```

### 5. Long-Running Delegation from Host Python Code

Use an argv array (no shell quoting hazards), private stdout/stderr files, a host wall-clock deadline, and full completion/denial checks:

```python
import json, os, shutil, subprocess

agy_bin = shutil.which("agy") or os.path.expanduser("~/.local/bin/agy")
cmd = [
    agy_bin,
    "--print=Refactor db/migrations to the v2 schema and run the migration tests.",
    "--add-dir", ".",
    "--mode", "accept-edits",
    "--settings", json.dumps({
        "toolPermission": "always-proceed",
        "artifactReviewPolicy": "always-proceed",
    }),
    "--output-format", "json",
    "--print-timeout", "15m",
]
with open("agy.out.json", "w") as out, open("agy.err.log", "w") as err:
    proc = subprocess.run(cmd, stdout=out, stderr=err, text=True, timeout=20 * 60)

data = json.load(open("agy.out.json"))
reason = data.get("finished_reason")
denied = data.get("denied_actions") or []
outcome = reason or ("denied" if denied else ("complete" if data.get("response") else "failed"))
print(proc.returncode, outcome, data.get("conversation_id"))
```

---

## Completion Detection, `denied_actions`, and Timeouts

### Mandatory Completion Check

Treat a run as complete only when **all** checks pass:

| Check | Complete | Incomplete / Action Needed |
| :--- | :--- | :--- |
| Exit code | `0` | `1`: config/flag/hook error; `2`: usage conflict; `3`: `AGY_ERROR` API failure or background-task `--print-timeout` (`v1.5.0+`) |
| `finished_reason` | **absent** | `print_timeout`, `interrupted`, or `error` (even when exit code is `0`!) |
| `denied_actions` | **absent / empty** | Non-empty array (e.g. `[{"action":"write_file","display_name":"Edit"}]`): add needed allow-rule/mode in `--settings` |
| `response` | non-empty | Empty with `num_turns: 0` and zero `usage`: nothing was collected |

```bash
REASON=$(jq -r '.finished_reason // empty' out.json)
DENIED=$(jq -r '.denied_actions | length // 0' out.json)
if [ -n "$REASON" ]; then
  OUTCOME="$REASON"
elif [ "$DENIED" -gt 0 ]; then
  OUTCOME="denied"
else
  OUTCOME="complete"
fi
```

### Bounded Review Prompting

Prevent timeouts by bounding the worker in the prompt itself:
- **Name exact files/directories** to inspect instead of letting the worker scan the whole repo.
- **Cap output** (e.g., max 6 findings, 600 words, final verdict).
- **Reserve a finalization window** ("spend at most 5 minutes exploring, then write the report").
- **Size `--print-timeout` above the exploration budget** (`8m`–`10m` for a `5m` budget).
- **Use `--output-format stream-json`** when live progress monitoring is needed.

---

## Permissions & Isolation Boundaries

- `--settings '{"agentMode":""}'` restricts `agy` to read-only file/search tools unless `permissions.allow` grants specific commands; unapproved write/command calls fail immediately into `.denied_actions`.
- `--settings '{"toolPermission":"always-proceed"}'` and `--dangerously-skip-permissions` still **cannot bypass** `agy`'s built-in protected-command denylist (`v1.5.2+`) or Gemini Enterprise admin-enforced review policies (`v1.5.2+`, `v1.5.7+`), where headless runs deny restricted actions rather than prompting.
- `--add-dir <path>` extends workspace scope; it does not isolate other paths.
- For strict read-only guarantees on untrusted code, run `agy` against a temporary copy or `git worktree`.

---

## Reading `stream-json` Events

Each stdout line is one JSON object with a top-level `event` key and matching payload object:

```json
{"event": "init", "conversation_id": "9f1c7a2e-0000-0000-0000-000000000000", "init": {"cwd": "/repo", "permission_mode": "always-proceed", "tools": ["read_file", "run_command"]}}
{"event": "step_update", "step_update": {"conversation_id": "9f1c7a2e-0000-0000-0000-000000000000", "step_index": 1, "state": "ACTIVE", "step_type": "agent_response", "text_delta": "ok"}}
{"event": "step_update", "step_update": {"conversation_id": "9f1c7a2e-0000-0000-0000-000000000000", "step_index": 1, "state": "DONE", "step_type": "agent_response", "text_delta": "\n", "duration_seconds": 2.38, "usage": {"input_tokens": 23463, "output_tokens": 36, "thinking_tokens": 35, "cache_read_tokens": 0, "total_tokens": 23499}}}
{"event": "result", "result": {"conversation_id": "9f1c7a2e-0000-0000-0000-000000000000", "status": "SUCCESS", "finished_reason": "print_timeout", "response": "ok\n", "duration_seconds": 36.47, "num_turns": 1, "usage": {"input_tokens": 23463, "output_tokens": 36, "thinking_tokens": 35, "cache_read_tokens": 0, "total_tokens": 23499}}}
```

- Key parsers only on `event`, `step_update.state`, `step_update.step_type`, `step_update.text_delta`, and `result.*`; ignore unknown fields.
- Tool steps carry `tool_info`; subagent steps carry `subagent_info`.
- `finished_reason` appears only when ended early (`print_timeout`, `interrupted`, `error`).

---

## Persistent Editor Integration: `agy acp`

When integrating `agy` into an IDE or persistent agent client instead of one-shot `-p` subprocesses, use `agy acp` (`v1.5.0+`–`v1.5.4+`):
- Runs an Agent Client Protocol (ACP) JSON-RPC server over stdio (`--stdio` default).
- Supports `--process-per-session`, `--ide-mcp-port <port>`, `--agent <name>`, `--add-dir <path>`, `auto` / `accept-edits` / `plan` / `default` modes, `session/resume`, and `agy acp --login` for terminal auth over SSH.

---

## Exit Codes Reference

| Exit Code | Meaning | Host Agent Action |
| :--- | :--- | :--- |
| `0` | Success, or mid-turn `--print-timeout` partial output | Check `finished_reason` and `denied_actions` in JSON; resume with `--conversation` if partial. |
| `1` | Config/flag error, invalid `--json-schema`, unknown `--agent`, or `PreInvocation` hook termination | Fix flags/schema or inspect hook error message on stderr. |
| `2` | Print-mode usage conflict (e.g., `-p` + `-i`) | Fix conflicting flags. |
| `3` | Fatal API/model error (`AGY_ERROR: {...}` on stderr) OR `--print-timeout` terminated active background tasks (`v1.5.0+`) | Parse `AGY_ERROR`; check quota with `/usage` or resume `--conversation` if timed out. |

---

## Loading Skills and MCP Servers During Delegation

1. **Auto-discovered skills**: `~/.gemini/config/skills/` and `<workspace>/.agents/skills/` (note `v1.5.2+` restricts `.agents/skills.json` and `.agents/plugins.json` entries to paths inside the workspace/repo root).
2. **Explicit skill invocation**: Start the prompt with `/<skill-name>` (e.g., `--print="/jbd-code-review-skill review src/"`).
3. **MCP tools**: Verify with `"$AGY" mcp list`; add missing servers via `"$AGY" mcp add <name> <cmd|url>` before delegating.
