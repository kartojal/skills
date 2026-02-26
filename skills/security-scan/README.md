# security-scan

Fast, simple, stupid but effective security feedback while developing or before committing to version control.

Built for **developers** who want to catch issues early - not a replacement for a formal audit, but the check you should run every time you touch a contract.

## Usage

```bash
# Review only what changed (default)
/security-scan

# Review a specific file
/security-scan src/Vault.sol

# Review the entire repo
/security-scan ALL

# Time-limited scan (in seconds) - adjusts scope and depth to fit the budget
/security-scan --max-run-time=30   # CRITICAL only, 2 files max, built-in vectors
/security-scan --max-run-time=60   # CRITICAL + HIGH, 5 files max (alias: --fast)
/security-scan --max-run-time=120  # CRITICAL + HIGH + MEDIUM, all files
/security-scan --max-run-time=300  # all severities, reads full attack vector reference
/security-scan --fast              # alias for --max-run-time=60

# Confidence threshold - only report findings at or above N/100 (default: 80)
/security-scan --confidence=65    # broader sweep, includes more uncertain findings
/security-scan --confidence=95    # tight report, near-certain issues only
```

## What it does

- **Default mode**: runs `git diff HEAD` and reviews only the `.sol` files you've changed
- **File mode**: reviews a single contract you specify
- **ALL mode**: scans the full repo at its current state

It reads your code, applies a standard checklist of Solidity vulnerabilities, and gives you a structured report with severity, confidence score, impact, and a concrete fix for each finding. Findings below the confidence threshold are suppressed — the report stays signal, not noise.

## False positives

Add an entry to `assets/false-positives.md` and the scanner will skip it:

```markdown
### Vault.withdraw
- **Vector:** Reentrancy
- **Reason:** All state updates happen before the external call. CEI is followed.

### Ownable.transferOwnership
- **Reason:** Owner is a 4/6 multisig. Centralization is intentional.
```

Vector is optional — omitting it skips all findings at that location. The scan report always lists what was skipped and why.

## Carrying over previous findings

Drop prior audit report `.md` files into `assets/findings/`. The reviewer will use them as context - avoiding duplicate findings and focusing on new attack surface.

## Who this is for

Developers writing Solidity who want a security gut-check as part of their normal workflow. If you're preparing for a formal audit, see [`start-audit`](../start-audit/) instead.
