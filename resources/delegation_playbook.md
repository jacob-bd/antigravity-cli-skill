# Delegating Work to agy (Host CLI Guide)

This guide is for AI coding assistants (Claude Code, OpenAI Codex CLI, OpenCode, Cursor, Aider, etc.) that want to delegate tasks to Google's Antigravity CLI (`agy`) as an external reasoning, review, or coding worker.

By delegating heavy-lifting, plan reviews, or code reviews to `agy`, host assistants can save context window capacity, leverage Google Gemini models (e.g. Gemini 3.8 Pro/Flash), and benefit from diverse cross-model verification.

---

## The Golden Mechanics of `agy -p`

When delegating commands from another agent or terminal environment, remember:

1. **Always pass `--dangerously-skip-permissions` in headless runs**: Without this flag, `agy` will block waiting for interactive user permission prompts on stdin and appear to hang. It is broad tool auto-approval, not an isolation boundary (see "Permissions and Isolation").
2. **Explicit Workspace Scoping (`--add-dir .`)**: When launching `agy` in a repository or directory, pass `--add-dir .` (or `--add-dir /path/to/target`). `agy` uses persistent workspace mappings; passing `--add-dir` ensures tools can read and write within that target path. It grants scope; it is not proof of exclusive filesystem isolation.
3. **Set a Safe Print Timeout (`--print-timeout <duration>`)**: Headless print mode defaults to `0s` (unlimited). For automated scripts, pass `--print-timeout 8m` or `10m` so processes never become zombies if the network drops. A fired timeout still exits `0` with `status: SUCCESS` and partial output, so always run the completion check before trusting a result.
4. **Choose the Output Format (`--output-format json` or `stream-json`)**:
   - `json`: Emits `{"conversation_id": "...", "status": "SUCCESS", "response": "...", "duration_seconds": ..., "num_turns": ..., "usage": {...}}` when the process exits. The response string is in `.response`.
   - `stream-json`: Emits line-delimited NDJSON events (`init`, `step_update`, `result`) for live progress; see "Reading stream-json Events".
5. **Continuous Multi-Turn (`--conversation <id>`)**: Every completion returns a `conversation_id`. Host agents can ask follow-up questions, request iterative revisions, or recover a cut-off run on the exact same context by passing `--conversation <id>`.
6. **Attach the Prompt to the Flag (`--print='...'`)**: Use the attached argv form (`agy --print='analyze src/'`) to avoid prompt/flag ambiguity when the text begins with dashes or is passed programmatically. The separate positional form `agy -p "..."` is fine for simple prompts.

---

## Incomplete Runs: Detection, Recovery, and Bounded Reviews

### The false-success trap

When `--print-timeout` fires mid-turn, `agy` exits with code **`0`** and JSON `"status": "SUCCESS"` plus whatever partial output exists. stdout alone cannot tell a finished review from a cut-off one. The marker appears on stderr:

```text
[agy] print timeout after 6m0s with turn in progress; returning partial output
```

### Mandatory completion check

Treat a run as finished only after all of these pass:

| Check | Complete | Incomplete |
| :--- | :--- | :--- |
| Exit code | `0` | non-zero: take the error path below |
| stdout JSON | parses | unparseable means a failed run |
| stderr | no timeout marker | contains `print timeout` / `returning partial output` |
| `response` | non-empty report | empty response with `num_turns: 0` and zero `usage`: nothing was collected |

Record a host-side outcome such as `complete` / `partial_timeout` / `failed`, and never silently label partial output as finished.

```bash
agy -p "$PROMPT" --output-format json ... > out.json 2> err.log
grep -q 'print timeout' err.log && OUTCOME=partial_timeout || OUTCOME=complete
```

### Recovery recipe: finish from collected evidence

A timed-out run has already done the expensive exploration, so do not restart the whole review. Resume the same conversation, forbid new exploration, and request the final deliverable under a small separate budget:

```bash
CONVERSATION_ID=$(jq -r .conversation_id out.json)

agy -p "Stop all exploration. Using only the evidence already collected, produce the final report now: at most 6 findings ranked by severity, 600 words maximum, ending with a verdict. Do not use any tools." \
  --conversation "$CONVERSATION_ID" \
  --add-dir . \
  --mode plan \
  --dangerously-skip-permissions \
  --output-format json \
  --print-timeout 3m
```

### Preventing timeouts: bounded review instructions

A timeout is a budget boundary, not evidence of a deadlock. Bound the work in the request itself:

- **Scope the search**: name the exact files or directories to inspect so the worker does not wander the repository.
- **Cap the deliverable**: request a small number of source-backed findings, a word budget, and a final verdict.
- **Reserve a finalization interval**: instruct it to stop exploring in time to write the report (e.g. "spend at most 5 minutes exploring, then report with what you have"). Models often exceed the first budget, which is exactly why the recovery recipe exists.
- **Size the timeout above the budget**: exploration budget 5m means `--print-timeout 8m`-`10m`. Use `--print-timeout 0` only when the host enforces its own wall-clock deadline.
- **Watch long runs**: `--output-format stream-json` surfaces live `step_update` events so hosts can detect stalls instead of waiting blind.

---

## Permissions and Isolation: What the Flags Do and Do Not Guarantee

- `--dangerously-skip-permissions` is broad tool auto-approval for the session; it does not sandbox the process or restrict which files can be touched.
- `--mode plan` stops the agent from implementing changes; it is a behavioral guardrail, not an OS-enforced read-only filesystem.
- `--add-dir <path>` extends the workspace scope; it is not an isolation boundary for other paths.
- `--sandbox` (macOS/Linux) enables terminal restrictions; effective scope still comes from the instructions you give.
- For review work that needs stronger isolation, point `agy` at a temporary copy of the tree and compare the copy afterwards; this pattern is in production use by hosts.

---

## Delegation Recipes

### 1. Adversarial Plan / Architecture Review (Read-Only)

**Goal:** Get a second-opinion critique of an architectural proposal, RFC, or planned refactor without making changes.

**Mechanics:**
- Pass `--mode plan` to prevent autonomous file writes.
- Pass `--output-format json` to capture the structured markdown or JSON response.
- Use `--effort high` or `--effort max` for complex architectural scrutiny.
- Bound the review and give it a realistic timeout (see "Incomplete Runs"); expect occasional partial timeouts and recover rather than rerunning.

```bash
agy -p "Perform an adversarial plan review of docs/rfc-001.md. Challenge assumptions, edge cases, failure modes, and data loss risks. At most 6 findings, end with a verdict. Reserve time for the final report. Do not implement changes." \
  --add-dir . \
  --mode plan \
  --effort high \
  --output-format json \
  --dangerously-skip-permissions \
  --print-timeout 10m
```

*Completion check:* grep stderr for `print timeout`. If present, treat the run as `partial_timeout` and use the recovery recipe with the returned `conversation_id`.

*Parsing response:*
```bash
RESPONSE=$(cat output.json | jq -r .response)
CONVERSATION_ID=$(cat output.json | jq -r .conversation_id)
```

---

### 2. Code Review on Diff (Read-Only)

**Goal:** Review an uncommitted git diff or branch changes.

**Mechanics:**
- Pass git status and diff context either directly in the prompt or instruct `agy` to inspect git.
- If running under `--sandbox`, note that git config in `~/.gitconfig` might be restricted on some platforms; passing the diff or file paths explicitly is recommended.
- You can optionally enforce a JSON schema on findings using `--json-schema <schema_file>`.
- Apply the same completion check as Recipe 1; a partial timeout here means findings are incomplete.

```bash
# Example: Review uncommitted changes
agy -p "Review the git diff in this repository for race conditions, security vulnerabilities, and edge case bugs. Return findings categorized by severity, at most 8 findings. Do not modify any files." \
  --add-dir . \
  --output-format json \
  --dangerously-skip-permissions \
  --print-timeout 8m
```

*Enforcing structured findings:*
```bash
agy -p "Audit src/auth/ for vulnerabilities. Return findings conforming to schema." \
  --json-schema ./review_schema.json \
  --add-dir . \
  --output-format json \
  --dangerously-skip-permissions
```

---

### 3. Delegating Implementation / Coding Tasks (Write Mode)

**Goal:** Have `agy` implement a fix, write a feature, or refactor code in the repository.

**Mechanics:**
- **Do not** pass `--mode plan`. Use standard execution mode or `--mode auto`.
- Explicitly pass `--add-dir .` (or the target root) so `agy` has write permissions on the directory.
- Specify tests or verification commands so `agy` runs them before finishing.
- Run on a dedicated git branch or worktree if the host AI wants to review before merging.
- Exit `0` does not prove the work finished; a `partial_timeout` can leave half-edited files. Run the completion check before trusting the diff.

```bash
# Example: Fix a failing test suite
agy -p "Fix the bug in src/payment/processor.py causing test_refund_timeout to fail. Run pytest tests/test_payment.py to verify. Keep changes minimal." \
  --add-dir . \
  --dangerously-skip-permissions \
  --output-format json \
  --print-timeout 10m
```

*Host AI follow-up:*
After `agy` exits `0` **and** passes the completion check, the host AI runs `git diff` to inspect changes, runs tests, and presents the diff to the user.

---
### 4. Follow-up & Iteration on Previous Turn

**Goal:** Ask `agy` to fix an issue identified in a review or refine its previous work. This is also the timeout-recovery path.

**Mechanics:**
- Supply the `--conversation <id>` captured from the previous response.
- Do not repeat the entire repository context; just provide the delta instruction.
- For recovery, forbid new exploration and request the final report from evidence already collected.

```bash
agy -p "In your previous review you flagged a race condition in WorkerPool. Please now generate the code fix directly in src/worker.go." \
  --conversation "5a70ead4-f77b-43bf-a797-ec0ac5e61816" \
  --add-dir . \
  --dangerously-skip-permissions \
  --output-format json \
  --print-timeout 5m
```

---

### 5. Long-Running Delegation from Host Code

Process liveness does not establish task completion, and `--print-timeout` is not an orphan-process guarantee. Prefer an argv-array subprocess with a host wall-clock deadline, explicit stdout/stderr files, and the completion check:

```python
import json, subprocess

cmd = [
    "agy",
    "--print=Refactor db/migrations to the v2 schema and run the migration tests.",
    "--add-dir", ".",
    "--dangerously-skip-permissions",
    "--output-format", "json",
    "--print-timeout", "15m",
]
with open("agy.out.json", "w") as out, open("agy.err.log", "w") as err:
    subprocess.run(cmd, stdout=out, stderr=err, text=True, timeout=20 * 60)

stderr_text = open("agy.err.log").read()
data = json.load(open("agy.out.json"))
outcome = "partial_timeout" if "print timeout" in stderr_text else ("complete" if data.get("response") else "failed")
print(outcome, data.get("conversation_id"))
```

- Track the process you started and terminate it on cancellation (argv arrays also avoid shell-quoting traps).
- Keep stdout/stderr in private files; do not stream raw tool output or token accounting into ordinary status logs.
- Shell backgrounding with `nohup ... 2> err.log &` still works for quick scripts, but check the stderr marker and JSON before treating the task as done.

---

## Reading `stream-json` Events (Real Envelopes)

Each stdout line is one JSON object with a top-level `event` field and a payload object under the same name. Real (sanitized) shapes:

```json
{"event": "init", "conversation_id": "9f1c7a2e-0000-0000-0000-000000000000", "init": {"cwd": "/repo", "permission_mode": "always-proceed", "tools": ["read_file", "run_command"]}}
{"event": "step_update", "step_update": {"conversation_id": "9f1c7a2e-0000-0000-0000-000000000000", "step_index": 1, "state": "ACTIVE", "step_type": "agent_response", "text_delta": "ok"}}
{"event": "step_update", "step_update": {"conversation_id": "9f1c7a2e-0000-0000-0000-000000000000", "step_index": 1, "state": "DONE", "step_type": "agent_response", "text_delta": "\n", "duration_seconds": 2.38, "usage": {"input_tokens": 23463, "output_tokens": 36, "thinking_tokens": 35, "cache_read_tokens": 0, "total_tokens": 23499}}}
{"event": "result", "result": {"conversation_id": "9f1c7a2e-0000-0000-0000-000000000000", "status": "SUCCESS", "response": "ok\n", "duration_seconds": 36.47, "num_turns": 1, "usage": {"input_tokens": 23463, "output_tokens": 36, "thinking_tokens": 35, "cache_read_tokens": 0, "total_tokens": 23499}}}
```

- Parser rule: key only on `event`, `step_update.state`, `step_update.step_type`, `step_update.text_delta`, and `result.*`; ignore unknown fields. There is no flat `type` field.
- Tool steps carry `tool_info` (name, args, output) and subagent steps carry `subagent_info` inside their `step_update` payload.
- Keep token counts, account details, and raw tool output out of ordinary status logs.

---

## Exit Codes & Error Handling for Host Scripts

| Exit Code | Meaning | Host AI Action |
| :--- | :--- | :--- |
| `0` | Success, or a `--print-timeout` cut returning partial output | Run the completion check (stderr marker) before parsing `.response`; recover incomplete runs on the same conversation. |
| `1` | Configuration error, invalid CLI flags, invalid `--json-schema` | Check stderr; fix invocation parameters. |
| `2` | Usage error (e.g. combining `--print` with `-i`) | Fix argument conflict. |
| `3` | Fatal API or model error (Quota exhausted, auth fail) | Stderr contains structured `AGY_ERROR: {...}` JSON. Parse error code and report to user. |

---

## Loading Custom Skills and MCP Servers During Delegation

If the host assistant wants `agy` to utilize a specialized skill or MCP tool during its run:

1. **Pre-installed Skills**: Skills located in `~/.gemini/config/skills/` or `<workspace>/.agents/skills/` are auto-discovered.
2. **Explicit Skill Invocations**: Use leading slash command syntax in the prompt (e.g. `agy -p "/jbd-code-review-skill review src/" --dangerously-skip-permissions`).
3. **MCP Tool Access**: Ensure required MCP servers are configured in `agy mcp list`. If not, configure them dynamically via `agy mcp add <name> <command>` before triggering the prompt.
