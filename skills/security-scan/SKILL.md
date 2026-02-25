---
name: security-scan
description: Fast, focused security feedback on Solidity code while you develop - before you commit, not after an auditor does. Built for developers, not security researchers. Use when the user asks to "review my changes for security issues", "check this contract", "security-scan", or wants a quick sanity check before pushing. Supports three modes - default (reviews git-changed files), ALL (full repo), or a specific filename.
---

# Smart Contract Security Review

Fast, focused security feedback while you're developing. Catch real issues early - before they reach an audit or mainnet.

Before scanning any code (unless `--fast` is set), read the full attack vector reference:
```
references/attack-vectors.md
```
It contains 52 attack vectors with precise detection patterns and false-positive signals. Use it as your scanning checklist for every file.

## Mode Selection

- **Default** (no arguments): run `git diff HEAD --name-only`, filter for `.sol` files. Stop and say so if there are no changed Solidity files.
- **ALL**: scan all `.sol` files in the repo (exclude `lib/`, `out/`, `node_modules/`, `.git/`).
- **`$filename`**: scan that specific file only.
- **`--fast`** (optional flag, combinable with any mode): complete in under 1 minute. See Fast Mode below.

## Fast Mode

When `--fast` is set, apply these constraints instead of the default process:

- **Skip** reading `references/attack-vectors.md` — use built-in knowledge of Solidity attack vectors.
- **Skip** loading `assets/false-positives.md` and `assets/findings/`.
- **Check only CRITICAL and HIGH severity vectors.** Do not report MEDIUM, LOW, or INFO.
- **ALL mode:** cap at the 5 files most recently changed (`git diff HEAD --name-only` order).
- **Output:** omit the PoC field. Replace with a single sentence: what an attacker does and what they gain.

Focus built-in scanning on the highest-yield vectors: reentrancy (single, cross-function, read-only), missing/incorrect access control, unprotected initializer, oracle spot-price manipulation, flash loan price manipulation, unchecked arithmetic in value flows, msg.value reuse in loop/multicall, delegatecall to user-controlled address, signature replay, ecrecover zero-address, abi.encodePacked hash collision, and price slippage (no min output).

---

## False Positives

If `assets/false-positives.md` exists, load it before scanning. Each entry has a location and a reason; vector is optional. Skip any finding whose location and vector match an entry.

Format:
```
### ContractName.functionName
- **Vector:** <optional>
- **Reason:** <why this is not a finding>
```

Matching rules:
- Location match: finding's contract+function equals the entry's heading (case-insensitive).
- If the entry specifies a vector, skip only findings for that vector at that location. If no vector is specified, skip all findings at that location.

Report the count and entry names of skipped findings at the bottom of the Scope section.

## Context Loading

Before scanning, load if present:
- **`assets/false-positives.md`** — per the False Positives section above.
- **`assets/findings/`** — prior audit reports. Use as context to avoid duplicating known issues. Mark previously known findings as such.

## Review Process

For each file in scope:

1. Read the full file.
2. Scan against all 52 vectors in `references/attack-vectors.md`. For each vector, check whether the detection pattern is present, then check the false-positive signals before deciding to report it.
3. Before reporting a finding, check it against `false-positives.md` entries. Skip any that match.
4. Only report findings where the detection pattern matches AND the false-positive conditions do not apply.
5. Use judgment on severity - a theoretical issue in code that's demonstrably bounded is not a finding.

Prioritize findings that are:
- Directly exploitable with a concrete attack path
- In functions handling value (ETH, tokens, governance power)
- In code that was changed (in default mode)

## Output Format

```
# Security Review

## Summary
<1-3 sentences: severity distribution, files reviewed, most critical finding>

## Findings

### [CRITICAL|HIGH|MEDIUM|LOW|INFO] Title
- **Location:** ContractName.functionName (line N)
- **Vector:** <vector name from attack-vectors.md>
- **Issue:** <what is wrong and why it matters>
- **Impact:** <what an attacker can do>
- **PoC:** <minimal attack scenario - one paragraph, no full exploit code> *(omitted in --fast mode: replaced by one sentence)*
- **Fix:** <concrete code-level recommendation>

## Scope
<files reviewed, mode>
False positives skipped: N (entry, ...)
```

Order findings Critical first. Omit severity levels that have no findings.

## Constraints

- Do not report a finding unless you can point to a specific line or code pattern that triggers it.
- Do not report theoretical issues that are structurally prevented by the codebase (check false-positive signals).
- Never fabricate findings to appear thorough.
- Keep PoC concise - attack scenario, not a full working exploit.
