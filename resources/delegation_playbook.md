# Delegating Work to agy (Host CLI Guide)

This guide is for AI coding assistants (Claude Code, OpenAI Codex CLI, OpenCode, Cursor, Aider, etc.) that want to delegate tasks to Google's Antigravity CLI (`agy`) as an external reasoning, review, or coding worker.

By delegating heavy-lifting, plan reviews, or code reviews to `agy`, host assistants can save context window capacity, leverage Google Gemini models (e.g. Gemini 3.8 Pro/Flash), and benefit from diverse cross-model verification.

---

## The Golden Mechanics of `agy -p`

When delegating commands from another agent or terminal environment, remember:

1. **Always pass `--dangerously-skip-permissions`**: Without this flag, `agy` will block waiting for interactive user permission prompts on stdin and appear to hang.
2. **Explicit Workspace Scoping (`--add-dir .`)**: When launching `agy` in a repository or directory, pass `--add-dir .` (or `--add-dir /path/to/target`). `agy` uses persistent workspace mappings; passing `--add-dir` ensures tools can read and write within that target path.
3. **Set a Safe Print Timeout (`--print-timeout <duration>`)**: Headless print mode defaults to `0s` (unlimited). For automated scripts, pass `--print-timeout 5m` or `10m` so processes never become zombies if network drops.
4. **Choose the Output Format (`--output-format json` or `stream-json`)**:
   - `json`: Emits `{"conversation_id": "...", "status": "SUCCESS", "response": "..."}` upon completion. The response string is in `.response`.
   - `stream-json`: Emits line-delimited NDJSON events (`init`, `step_update`, `result`). Useful for streaming progress indicators or logging tool steps.
5. **Continuous Multi-Turn (`--conversation <id>`)**:
   Every completion returns a `conversation_id`. Host agents can ask follow-up questions or request iterative revisions on the exact same context by passing `--conversation <id>`.

---

## Delegation Recipes

### 1. Adversarial Plan / Architecture Review (Read-Only)

**Goal:** Get a second-opinion critique of an architectural proposal, RFC, or planned refactor without making changes.

**Mechanics:**
- Pass `--mode plan` to prevent autonomous file writes.
- Pass `--output-format json` to capture the structured markdown or JSON response.
- Use `--effort high` or `--effort max` for complex architectural scrutiny.

```bash
agy -p "Perform an adversarial plan review of docs/rfc-001.md. Challenge assumptions, edge cases, failure modes, and data loss risks. Do not implement changes." \
  --add-dir . \
  --mode plan \
  --effort high \
  --output-format json \
  --dangerously-skip-permissions \
  --print-timeout 5m
```

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

```bash
# Example: Review uncommitted changes
agy -p "Review the git diff in this repository for race conditions, security vulnerabilities, and edge case bugs. Return findings categorized by severity. Do not modify any files." \
  --add-dir . \
  --output-format json \
  --dangerously-skip-permissions \
  --print-timeout 4m
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

```bash
# Example: Fix a failing test suite
agy -p "Fix the bug in src/payment/processor.py causing test_refund_timeout to fail. Run pytest tests/test_payment.py to verify. Keep changes minimal." \
  --add-dir . \
  --dangerously-skip-permissions \
  --output-format json \
  --print-timeout 8m
```

*Host AI follow-up:*
After `agy` exits 0, the host AI runs `git diff` to inspect changes, runs tests, and presents the diff to the user.

---

### 4. Follow-up & Iteration on Previous Turn

**Goal:** Ask `agy` to fix an issue identified in a review or refine its previous work.

**Mechanics:**
- Supply the `--conversation <id>` captured from the previous response.
- Do not repeat the entire repository context; just provide the delta instruction.

```bash
agy -p "In your previous review you flagged a race condition in WorkerPool. Please now generate the code fix directly in src/worker.go." \
  --conversation "5a70ead4-f77b-43bf-a797-ec0ac5e61816" \
  --add-dir . \
  --dangerously-skip-permissions \
  --output-format json \
  --print-timeout 5m
```

---

### 5. Running in the Background (Asynchronous Delegation)

When the host AI tool supports background tasks (e.g. Claude Code's `run_in_background: true`, Codex sub-processes, or standard shell backgrounding):

```bash
# Run detached and log output
nohup agy -p "Refactor db/migrations to use v2 schema" \
  --add-dir . \
  --dangerously-skip-permissions \
  --output-format json \
  --print-timeout 15m > /tmp/agy_task.json 2> /tmp/agy_task.err &
AGY_PID=$!

# The host AI can monitor /tmp/agy_task.json or check process status with kill -0 $AGY_PID
```

---

## Exit Codes & Error Handling for Host Scripts

| Exit Code | Meaning | Host AI Action |
| :--- | :--- | :--- |
| `0` | Success (or partial result on graceful `--print-timeout`) | Parse `.response` from stdout JSON. |
| `1` | Configuration error, invalid CLI flags, invalid `--json-schema` | Check stderr; fix invocation parameters. |
| `2` | Usage error (e.g. combining `--print` with `-i`) | Fix argument conflict. |
| `3` | Fatal API or model error (Quota exhausted, auth fail) | Stderr contains structured `AGY_ERROR: {...}` JSON. Parse error code and report to user. |

---

## Loading Custom Skills and MCP Servers During Delegation

If the host assistant wants `agy` to utilize a specialized skill or MCP tool during its run:

1. **Pre-installed Skills**: Skills located in `~/.gemini/config/skills/` or `<workspace>/.agents/skills/` are auto-discovered.
2. **Explicit Skill Invocations**: Use leading slash command syntax in the prompt (e.g. `agy -p "/jbd-code-review-skill review src/" --dangerously-skip-permissions`).
3. **MCP Tool Access**: Ensure required MCP servers are configured in `agy mcp list`. If not, configure them dynamically via `agy mcp add <name> <command>` before triggering the prompt.
