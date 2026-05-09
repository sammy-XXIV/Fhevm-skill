# fhevm-skill — AI Agent Skill for Zama FHEVM

A production-ready skill that teaches any AI coding agent how to write, test, and deploy confidential smart contracts on Zama FHEVM — correctly, first try.

**Submission for:** Zama Developer Program Mainnet Season 2 — Bounty Track  
**Built from:** Real production bugs discovered while building a confidential lending protocol on Zama FHEVM Sepolia.

---

## What's inside

| File | What it does |
|------|-------------|
| `SKILL.md` | 30 sections — full FHEVM development workflow from setup to production |
| `fhevm-lint.js` | Static linter, zero dependencies — catches 15 anti-patterns before deployment |
| `templates/ConfidentialVault.sol` | Production-ready vault contract with full ACL and decrypt flow |
| `templates/ConfidentialLending.sol` | Full lending protocol — deposit, borrow, repay, liquidate in FHE |

---

## Why this skill exists

AI coding agents have no built-in FHE knowledge. Without guidance they:
- Use `FHE.div` — doesn't exist, contract won't compile
- Put `FHE.allow` inside `view` functions — compile error
- Skip `FHE.allowTransient` before token transfers — silent revert
- Use inline `FHE.asEuint64(0)` for comparisons — unreliable results
- Try `approve()` on ERC-7984 tokens — reverts silently

Every bug in this skill was discovered building a real protocol on Sepolia — not from reading docs.

---

## Quick start

```bash
git clone https://github.com/sammy-XXIV/fhevm-skill
cd fhevm-skill

# Lint your contracts
node fhevm-lint.js path/to/contracts/

# Load SKILL.md as agent context
# Then prompt: "Build a confidential vault on Zama FHEVM"
```

---

## The linter

```bash
node fhevm-lint.js contracts/
```

Catches 15 anti-patterns with line number, offending code, and exact fix:

```
✖ line 17 [AP-001] FHE.div does not exist in the Zama library
    Code: euint64 result = FHE.div(balance, FHE.asEuint64(100));
    Fix:  Cross-multiply instead: FHE.le(FHE.mul(a,100), FHE.mul(b,66))

✖ line 24 [AP-009] FHE.allowTransient missing before confidentialTransferFrom
    Code: token.confidentialTransferFrom(msg.sender, address(this), amount);
    Fix:  Add FHE.allowTransient(amount, address(token)) immediately before

⚠ line 6  [AP-012] Possible missing ZamaEthereumConfig inheritance
    Code: contract MyVault {
    Fix:  contract MyVault is ZamaEthereumConfig { ... }
```

Exit code 1 if errors found — integrates with CI.

---

## SKILL.md coverage

| Requirement | Sections |
|-------------|----------|
| Encrypted types | §3, §25 |
| FHE operations | §5 |
| Access control | §6, §20 |
| Input proofs | §7 |
| Decryption patterns | §9 |
| Frontend integration | §11, §19 |
| Testing | §12, §27 |
| Common anti-patterns | §13, §28 |

**Production-only patterns not in official docs:**
- Backend-mediated decryption (CORS blocks frontend SDK)
- Stale handle bug after every FHE storage update
- `FHE.div` workaround via cross-multiplication
- `_encryptedZero` in constructor to fix inline zero bug
- cWETH 8 decimals vs 18 decimal errors
- Render cold start — wake backend on page load
- `estimateGas` blocked by FHEVM plugin on Sepolia
- `setOperator` vs `approve` on ERC-7984 tokens

---

## Built from production

Every anti-pattern in this skill caused a real bug during development of a confidential lending protocol on Zama FHEVM Sepolia. This is not a docs summary — it is a record of what actually breaks in production.
