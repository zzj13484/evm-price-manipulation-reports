# [Title — concise impact statement]

| Field | Value |
|-------|-------|
| **Report ID** | ETH-PM-2026-XXX |
| **Target** | `0x...` (Ethereum Mainnet, chain ID 1) |
| **Contract** | ContractName |
| **Severity** | Critical / High / Medium / Low |
| **Category** | Price manipulation / Oracle / Access control / Logic |
| **Reporter** | Zhao Zhijie (赵志杰) |
| **Date** | YYYY-MM-DD |
| **Status** | Draft / Submitted / Acknowledged / Resolved |

**Language:** English · [中文](../zh-CN/eth_0x..._ContractName.md)

---

## Summary

(2–4 sentences: what the bug is, what an attacker can do, whether privileged access is required.)

## Verified deployments

(List isomorphic addresses if applicable. Same vulnerability class — no separate report needed.)

| Address | Notes |
|---------|-------|
| `0x...` | Primary target for this report |

**Etherscan:** https://etherscan.io/address/0x...

## Affected scope

- **In-scope functions:** `functionName(...)`
- **Out of scope:** (if any)

## Root cause

(Technical root cause with minimal relevant code.)

```solidity
// Minimal excerpt
```

## Attack scenario

1. Preconditions
2. Step 1 — manipulate spot price
3. Step 2 — trigger victim path
4. Step 3 — arbitrage / extract value

## Impact

| Dimension | Assessment |
|-----------|------------|
| **Financial** | ... |
| **Integrity** | ... |
| **Availability** | ... |

## Proof of concept

**Status:** Conceptual / Foundry fork (link when available)

```solidity
// PoC skeleton
```

## Recommendations

1. ...
2. ...

## References

- Etherscan verified source
- Related reports in this repository

## Disclosure timeline

| Date | Event |
|------|-------|
| YYYY-MM-DD | Initial audit |
| TBD | Vendor submission |
| TBD | Fix verification |

---

## Platform checklist

- [ ] Title matches Summary
- [ ] Address and chain ID correct
- [ ] Severity aligned with impact
- [ ] PoC reproducible or steps clear
- [ ] No overstatement of exploitability
