---
name: security-scan
description: Fast, focused security feedback on Solidity code while you develop - before you commit, not after an auditor does. Built for developers, not security researchers. Use when the user asks to "review my changes for security issues", "check this contract", "security-scan", or wants a quick sanity check before pushing. Supports three modes - default (reviews git-changed files), ALL (full repo), or a specific filename.
---

# Smart Contract Security Review

Fast, focused security feedback while you're developing. Catch real issues early - before they reach an audit or mainnet.

Before scanning any code (unless a tight time budget applies — see Time Budget below), read the full attack vector reference:

```
references/attack-vectors.md
```

It contains 49 attack vectors with precise detection patterns and false-positive signals. Use it as your scanning checklist for every file.

Always read the report formatting specification before producing output:

```
references/report-formatting.md
```

It defines the disclaimer, severity classification, output format, and ordering rules. Follow it exactly.

## Mode Selection

- **Default** (no arguments): run `git diff HEAD --name-only`, filter for `.sol` files. Stop and say so if there are no changed Solidity files.
- **ALL**: scan all `.sol` files in the repo (exclude `lib/`, `out/`, `node_modules/`, `.git/`).
- **`$filename`**: scan that specific file only.
- **`--max-run-time=N`** (optional, in seconds): cap the scan to a time budget. Adjusts scope, depth, and severity coverage based on the tier below. `--fast` is an alias for `--max-run-time=60`.
- **`--confidence=N`** (optional, default `80`): minimum confidence score (0–100) a finding must reach to be reported. Lower values cast a wider net; higher values report only near-certain issues. Example: `--confidence=70` for a broad sweep, `--confidence=95` for a tight, high-signal report.

## Time Budget

When `--max-run-time=N` is set, apply the constraints for the matching tier. Tiers are cumulative — a higher budget includes everything from lower tiers.

| Budget | Severities | Reference files | Assets | File cap (ALL mode) |
|---|---|---|---|---|
| ≤ 30s | CRITICAL only | Skip — use built-in knowledge | Skip | 2 files |
| ≤ 60s | CRITICAL, HIGH | Skip — use built-in knowledge | Skip | 5 files |
| ≤ 120s | CRITICAL, HIGH, MEDIUM | Skip — use built-in knowledge | Load | All files |
| ≤ 300s | All severities | Read `attack-vectors.md` | Load | All files |
| No limit (default) | All severities | Read `attack-vectors.md` | Load | All files |

**Built-in high-yield vectors** (used when `attack-vectors.md` is skipped): reentrancy (single, cross-function, read-only), missing/incorrect access control, unprotected initializer, oracle spot-price manipulation, flash loan price manipulation, unchecked arithmetic in value flows, msg.value reuse in loop/multicall, delegatecall to user-controlled address, signature replay, abi.encodePacked hash collision, price slippage (no min output).

---

## Confidence Scoring

Every finding that passes the false-positive checks is assigned a confidence score from 0 to 100 before being reported. Findings below the active threshold (default 85) are suppressed — not listed, but counted in the Scope section.

**Scoring criteria:**

| Score    | Meaning                                                                                                                                                                             |
| -------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 90–100   | Detection pattern matches exactly. No FP signals apply. Clear, step-by-step attack path. Function is public/external and reachable without preconditions. Meaningful value at risk. |
| 75–89    | Pattern matches clearly but a FP signal partially applies (e.g., CEI partially followed, guard present but incomplete). Attack requires some preconditions or specific state.       |
| 50–74    | Pattern matches but multiple FP signals partially reduce certainty. Requires elevated preconditions (specific role, multi-step setup, or low-probability state). Impact is limited. |
| Below 50 | Theoretical match only. Should rarely be reached — the FP filter should have eliminated these.                                                                                      |

**Scoring process:**

1. Start at 100.
2. For each FP signal from `attack-vectors.md` that partially applies, deduct 10–25 points depending on how strongly it mitigates the risk.
3. If the attack path requires multiple non-trivial preconditions, deduct 10–15 points per additional step.
4. If the value at risk is minimal (e.g., governance of an undeployed contract, dust amounts), deduct 10–20 points.
5. If a prior finding in `assets/findings/` is closely related, deduct 5–10 points (known surface, partially addressed risk).

Do not game the score upward to make a report look thorough. When in doubt, score lower and let the threshold filter decide.

## Context Loading

Before scanning, load if present:

- **`assets/false-positives.md`** — per the False Positives section above.
- **`assets/findings/`** — prior audit reports. Use as context to avoid duplicating known issues. Mark previously known findings as such.

## Review Process

For each file in scope:

1. Read the full file.
2. Scan against all 49 vectors in `references/attack-vectors.md`. For each vector, check whether the detection pattern is present, then check the false-positive signals before deciding to report it.
3. Check the finding against `false-positives.md` entries. Skip any that match.
4. Only carry forward findings where the detection pattern matches AND the false-positive conditions do not fully apply.
5. Assign a confidence score (0–100) per the Confidence Scoring section above.
6. Suppress findings whose confidence score is below the active threshold (default 85; overridden by `--confidence=N`).
7. Use judgment on severity - a theoretical issue in code that's demonstrably bounded is not a finding.

Prioritize findings that are:

- Directly exploitable with a concrete attack path
- In functions handling value (ETH, tokens, governance power)
- In code that was changed (in default mode)

## Output Format

Follow `references/report-formatting.md` exactly. Summary: disclaimer first, then a findings table (number, severity, title), then detailed findings sections in the same order, then Scope. Severity levels: CRITICAL, HIGH, MEDIUM, LOW only — do not report INFO findings.

## Constraints

- Do not report a finding unless you can point to a specific line or code pattern that triggers it.
- Do not report theoretical issues that are structurally prevented by the codebase (check false-positive signals).
- Never fabricate findings to appear thorough.
- Do not report INFO findings. Minimum severity is LOW.
