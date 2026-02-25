# Pashov Audit Group Skills

> Modular, agent-agnostic skills for Claude Code, Codex, Copilot, Cursor, Windsurf, and beyond. Built by Pashov Audit Group [www.pashov.com](https://www.pashov.com/)

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Contributions Welcome](https://img.shields.io/badge/contributions-welcome-brightgreen.svg)](CONTRIBUTING.md)
[![CI](https://github.com/pashov/skills/actions/workflows/ci.yml/badge.svg)](https://github.com/pashov/skills/actions/workflows/ci.yml)

Drop a skill into your AI environment and it gains a focused, reusable capability - like a plugin, but for AI agents.

|                | Supported                                         |
| -------------- | ------------------------------------------------- |
| **Models**     | Claude · ChatGPT · Gemini                         |
| **Agents**     | Claude Code · Codex · OpenCode · GitHub Copilot   |
| **IDEs**       | VS Code · Cursor · Windsurf                       |
| **Extensions** | Claude Code · GitHub Copilot · Gemini Code Assist |

---

## Skills

| Skill                                      | Description                                                                                      | Category           |
| ------------------------------------------ | ------------------------------------------------------------------------------------------------ | ------------------ |
| [lint](skills/lint/)                       | Lints Solidity code - unused imports, NatSpec, formatting, naming, custom errors, best practices | Secure Development |
| [security-scan](skills/security-scan/) | Fast security feedback on Solidity changes while you develop                                     | Secure Development |
| [start-audit](skills/start-audit/)         | Full audit prep for security researchers - builds, architecture diagrams, threat model           | Security Research  |

---

## Install

**Don't want to clone the whole repo?** Grab just what you need:

```bash
# Sparse-checkout a single skill
git clone --filter=blob:none --sparse https://github.com/pashov/skills
cd skills && git sparse-checkout set skills/security-scan
```

```bash
# Or fetch just the instruction file
curl -fsSL https://raw.githubusercontent.com/pashov/skills/main/skills/security-scan/SKILL.md \
  -o security-scan.SKILL.md
```

**Drop it into your agent:**

| Agent                    | Where to put it                                                                              |
| ------------------------ | -------------------------------------------------------------------------------------------- |
| Claude Code (global)     | `~/.claude/skills/security-scan/`                                                          |
| Claude Code (project)    | `.claude/skills/security-scan/`                                                            |
| GitHub Copilot (project) | `.github/skills/security-scan/`                                                            |
| GitHub Copilot (global)  | `~/.copilot/skills/security-scan/`                                                         |
| Cursor                   | `.cursor/rules/security-scan.md`                                                           |
| Windsurf                 | `.windsurf/rules/security-scan.md`                                                         |
| Codex                    | `$skill-installer install https://github.com/pashov/skills/tree/main/skills/security-scan` |
| Any agent                | Paste `SKILL.md` contents into your system prompt or context window                          |

For Cursor and Windsurf, copy the contents of `SKILL.md` directly into the rules file.

**Claude Code** — then invoke by name in a new conversation:

```
/security-scan path/to/Contract.sol
```

---

## Contributing

We welcome new skills, improvements, and fixes. One skill, one purpose - see [AGENTS.md](AGENTS.md) for contribution rules.

1. Copy an existing skill as a starting point.
2. Fill in `SKILL.md` - frontmatter `name` and `description` are required.
3. Add a `README.md` with usage examples.
4. Open a pull request.

---

## Structure

```
skills/
└── skill-name/
    ├── SKILL.md         # Required - frontmatter + instructions
    ├── README.md        # Usage and examples
    ├── scripts/         # Executable helpers
    ├── references/      # Docs loaded into context
    └── assets/          # Templates and static files
```

---

## Security · Code of Conduct · License

Report vulnerabilities via [Security Policy](SECURITY.md). This project follows the [Contributor Covenant](CODE_OF_CONDUCT.md). [MIT](LICENSE) © contributors.

## Security Consulting

Reach out for a Pashov Audit Group security audit on [Telegram @pashovkrum](https://t.me/pashovkrum)
