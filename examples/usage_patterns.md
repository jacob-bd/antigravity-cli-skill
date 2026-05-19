# agy CLI Usage Patterns

## Pattern 1: Non-Interactive Code Generation
Useful for generating snippets or files without manual intervention.

```bash
agy -p "Generate a robust python script for data cleaning" --dangerously-skip-permissions
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
agy -p "Create a CSS design system for a turquoise theme" --dangerously-skip-permissions > design.css

# Task for the 'Developer' agent using the design
agy -p "Build a landing page using design.css" --dangerously-skip-permissions
```

## Pattern 4: Plugin Management for Specific Tasks
Extending capabilities on the fly.

```bash
agy plugin install chrome-devtools
agy -p "Audit the accessibility of localhost:3000" --dangerously-skip-permissions
```
