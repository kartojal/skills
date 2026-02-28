# Pashov Audit Group Skills

> Claude Code skills for Solidity security and development — built by [Pashov Audit Group](https://www.pashov.com/).

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Contributions Welcome](https://img.shields.io/badge/contributions-welcome-brightgreen.svg)](CONTRIBUTING.md)

---

## Skills

| Skill                                | Description                                                                    |
| ------------------------------------ | ------------------------------------------------------------------------------ |
| [audit](skills/audit/)               | Fast security feedback on Solidity changes while you develop (typically <5 min)|
| [audit-helper](skills/audit-helper/) | Full audit prep — architecture, fund flows, integrations, and threat model     |
| [lint](skills/lint/)                 | Lints Solidity — imports, NatSpec, naming, visibility, custom errors, and more |

---

## Install & Run

Works with Claude Code in **VS Code**, **Cursor**, and the terminal. Clone this repo, then copy the skill folder into Claude Code's commands directory.

**Global** — available in every project:

```bash
cp -r skills/audit ~/.claude/commands/
```

**Local** — available in the current project only:

```bash
cp -r skills/audit .claude/commands/
```

The skill is then invocable as `/audit`. Replace `audit` with any skill name from the table above.

---

## Contributing · Security · License

We welcome improvements and fixes. See [CONTRIBUTING.md](CONTRIBUTING.md) for the PR process.

Report vulnerabilities via [Security Policy](SECURITY.md). This project follows the [Contributor Covenant](CODE_OF_CONDUCT.md). [MIT](LICENSE) © contributors.

## Contact

For a Pashov Audit Group security engagement, reach out on [Telegram @pashovkrum](https://t.me/pashovkrum).
