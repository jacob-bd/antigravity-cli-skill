<p align="center">
  <img src="assets/banner.png" alt="Google Antigravity CLI Skill" width="600">
</p>

# Antigravity CLI (agy) Skill

An AI agent skill for working with Google's Antigravity CLI (`agy`) -- the official successor to Gemini CLI.

## What This Is

This is a skill file that teaches AI coding agents (Claude Code, Gemini CLI, Codex, Cursor, etc.) how to use the `agy` command-line tool effectively. It covers:

- Complete flag reference for agy v1.0.6
- Known v1.0.6 limitations (flags that don't exist yet)
- Gemini CLI to agy migration guide and flag mapping
- Best practices for automation and scripting
- Troubleshooting common issues

## Background

Google announced at I/O 2026 (May 19) that Gemini CLI is transitioning to Antigravity CLI. Consumer/free users lose Gemini CLI access on June 18, 2026. The `agy` CLI is a Go binary that shares the same agent harness as the Antigravity IDE desktop application.

## Files

| File | Description |
| :--- | :--- |
| `SKILL.md` | Main skill definition -- install this in your AI tool |
| `resources/cheat_sheet.md` | Quick reference card with all flags and migration mapping |
| `examples/usage_patterns.md` | Common usage patterns with runnable examples |

## Installation

Copy or symlink `SKILL.md` to your AI tool's skill directory:

```bash
# Claude Code
cp SKILL.md ~/.claude/skills/antigravity-cli/SKILL.md

# Gemini CLI
cp SKILL.md ~/.gemini/skills/antigravity-cli/SKILL.md

# Or install via Universal Skills Manager if available
```

## Status

This skill tracks agy v1.0.6. It will be updated as Google adds features like NDJSON streaming output.
