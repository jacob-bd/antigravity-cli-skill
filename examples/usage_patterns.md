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

## Pattern 9: Listing and Specifying Models
List all available models in the CLI and specify a model when launching a single prompt.

```bash
# Step 1: List all available models
agy models

# Step 2: Run a print command using a specific model (e.g. gemini-2.5-pro)
agy --model gemini-2.5-pro -p "Explain quantum computing in one sentence" --dangerously-skip-permissions
```

## Pattern 10: Interactive Permissions Configuration
Allows you to view and modify permission rules directly within the TUI.

```bash
# Step 1: Open the interactive TUI
agy

# Step 2: In the chat prompt, type `/permissions`
# This opens the interactive permission rules manager panel, allowing you
# to manage workspace, shared settings, and CLI configuration settings.
```

## Pattern 11: Configuring MCP Launch Timeouts
If you have a slower MCP server that takes a long time to start or initialize, you can configure or disable its launch timeout.

In `~/.gemini/config/mcp_config.json`, add the `"timeout"` parameter to the specific server configuration:

```json
{
  "mcpServers": {
    "my-slow-server": {
      "command": "node",
      "args": ["/path/to/server.js"],
      "timeout": 30000
    },
    "my-unlimited-server": {
      "command": "python3",
      "args": ["/path/to/server.py"],
      "timeout": -1
    }
  }
}
```
*Note: Setting `"timeout"` to `-1` disables the launch timeout completely for that server.*

## Pattern 12: Stacked Status Line Configuration
If you write custom status line indicators using community plugins, you can display both the default Antigravity status line and your custom status line indicators vertically stacked in the TUI.

In `~/.gemini/antigravity-cli/settings.json`, configure the `"statusLine"` block:

```json
{
  "statusLine": {
    "stack_with_default": true
  }
}
```

## Pattern 13: Installing Plugins from GitHub Subpaths
Version 1.0.7 adds support for installing plugins directly from subpaths within GitHub repositories, including branch resolution.

```bash
# Install a plugin located in a repository subpath on a specific branch
agy plugin install github.com/owner/repo/subpath@branch-name
```


