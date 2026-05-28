# agy CLI Usage Patterns

## Pattern 1: Non-Interactive Code Generation
Useful for generating snippets or files without manual intervention.

```bash
agy -p "Generate a robust python script for data cleaning" --dangerously-skip-permissions
```

For long-running generation tasks, increase the timeout:
```bash
agy -p "Build a full REST API with auth, tests, and docs" \
  --dangerously-skip-permissions \
  --print-timeout 30m
```

## Pattern 2: Context-Aware Refactoring
Resuming a conversation to apply changes to an existing project.

```bash
# First, establish context
agy -i "Analyze this repository for security vulnerabilities" --add-dir .

# Later, apply a fix
agy -c "Fix the vulnerabilities found in the auth module" --dangerously-skip-permissions
```

## Pattern 3: Multi-Agent Collaboration
Using `agy` as a coordinator to delegate tasks to "worker" agents.

```bash
# Task for the 'Designer' agent
agy -p "Create a CSS design system for a dark theme" \
  --dangerously-skip-permissions > design.css

# Task for the 'Developer' agent using the design
agy -p "Build a landing page using design.css" \
  --dangerously-skip-permissions
```

## Pattern 4: Plugin Management for Specific Tasks
Extending capabilities on the fly.

```bash
agy plugin install chrome-devtools
agy -p "Audit the accessibility of localhost:3000" --dangerously-skip-permissions
```

## Pattern 5: Migrating from Gemini CLI
If you have existing Gemini CLI workflows, migrate step by step.

```bash
# Step 1: Import your Gemini CLI extensions as plugins
agy plugin import gemini

# Step 2: Verify they imported
agy plugin list

# Step 3: Update your scripts
# Before (Gemini CLI):
#   gemini -p "fix the bug" --yolo
# After (agy):
#   agy -p "fix the bug" --dangerously-skip-permissions
```

## Pattern 6: Multi-Directory Workspace
Bring in context from multiple directories for cross-project work.

```bash
agy -i "Compare the API contracts between these two services" \
  --add-dir ./service-a \
  --add-dir ./service-b
```

## Pattern 7: Logging for CI/CD Pipelines
Capture detailed execution logs for debugging in automated environments.

```bash
agy -p "Run the test suite and fix any failures" \
  --dangerously-skip-permissions \
  --print-timeout 15m \
  --log-file /tmp/agy-ci-$(date +%s).log
```

## Pattern 8: Managing and Purchasing G1 Credits
When model usage standard quota is exhausted, you can check credit balance or trigger automatic credit usage.

```bash
# Open the interactive CLI TUI
agy

# Once inside the interactive TUI, type `/credits` in the chat prompt
# to open the credits panel, view G1 credit balance details, and obtain
# a direct link to purchase more credits.
```

