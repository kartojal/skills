# AGENTS.md

Instructions for AI coding agents contributing to this repository.

## What This Repo Is

A library of Claude AI skills. Each skill is a focused, self-contained capability for Claude Code in VS Code and Cursor.

## Structure

```
skills/
├── audit/           # Security review of Solidity changes while you develop
├── audit-helper/    # Full audit prep for security researchers
├── lint/            # Solidity linter (NatSpec, naming, best practices)
marketplace.json     # Plugin marketplace manifest
AGENTS.md            # This file (read by all agents)
CLAUDE.md            # Imports AGENTS.md for Claude Code
```

## Rules

- One skill, one purpose.
- No fabricated examples - outputs must reflect real model responses.
- No secrets, API keys, or personal data.
- Only list models in `model_compatibility` that have been tested.
- Do not update `marketplace.json` without also moving or renaming the skill directory.
