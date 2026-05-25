# Ethereum Price Manipulation Security Disclosures

Independent security research on **price manipulation (PM)** vulnerabilities in verified Ethereum mainnet smart contracts.

**Reporter:** Zhao Zhijie (赵志杰)  
**Research focus:** Spot AMM / oracle pricing, zero-slippage router sinks, vault share mispricing  
**Status:** Draft reports — pending vendor confirmation via official bug bounty platforms

---

## Repository layout

```
security-disclosures/
├── README.md                 # This file
├── LICENSE                     # MIT License
├── SECURITY.md                 # Disclosure policy
├── reports/
│   ├── INDEX.md                # Master index (bilingual)
│   ├── zh-CN/                  # Chinese reports (full technical detail)
│   └── en/                     # English reports (submission-ready)
├── supplemental/
│   ├── zh-CN/                  # Negative controls & non-PM notes
│   └── en/
└── templates/
    ├── REPORT_TEMPLATE.en.md
    └── REPORT_TEMPLATE.zh-CN.md
```

---

## Reports at a glance

| ID | Contract | Primary address | Severity | EN | 中文 |
|----|----------|-----------------|----------|----|------|
| ETH-PM-2026-001 | Vogue (QU!D) | [`0x0bafc174…`](https://etherscan.io/address/0x0bafc174bf07ec3d0eafdee06df7d07377108c0c) | High | [en](reports/en/eth_0x0bafc174bf07ec3d0eafdee06df7d07377108c0c_Vogue.md) | [zh-CN](reports/zh-CN/eth_0x0bafc174bf07ec3d0eafdee06df7d07377108c0c_Vogue.md) |
| ETH-PM-2026-002 | Rover (QU!D) | [`0x1d442…`](https://etherscan.io/address/0x1d442eeed53b89f08a5b07a256a9d0fa99bc1f97) | High | [en](reports/en/eth_0x1d442eeed53b89f08a5b07a256a9d0fa99bc1f97_Rover.md) | [zh-CN](reports/zh-CN/eth_0x1d442eeed53b89f08a5b07a256a9d0fa99bc1f97_Rover.md) |
| ETH-PM-2026-003 | MasterChef | [`0x7f622…`](https://etherscan.io/address/0x7f6229786703f01a8bfcede5f94c36179c467acd) | High | [en](reports/en/eth_0x7f6229786703f01a8bfcede5f94c36179c467acd_MasterChef.md) | [zh-CN](reports/zh-CN/eth_0x7f6229786703f01a8bfcede5f94c36179c467acd_MasterChef.md) |
| ETH-PM-2026-004 | Zap (yYB) | [`0x7d3a6d…`](https://etherscan.io/address/0x7d3a6d1085fe898965cbc0b47a5a652965438cac) | High | [en](reports/en/eth_0x7d3a6d1085fe898965cbc0b47a5a652965438cac_Zap.md) | [zh-CN](reports/zh-CN/eth_0x7d3a6d1085fe898965cbc0b47a5a652965438cac_Zap.md) |
| ETH-PM-2026-005 | Vault (Naturelab) | [`0x71ea…`](https://etherscan.io/address/0x71ea0eb2605bd63fe69012a60c75bdbd22e8b3d3) | High | [en](reports/en/eth_0x71ea0eb2605bd63fe69012a60c75bdbd22e8b3d3_Vault.md) | [zh-CN](reports/zh-CN/eth_0x71ea0eb2605bd63fe69012a60c75bdbd22e8b3d3_Vault.md) |

**Supplemental (not PM submission reports):** [Negative controls & governance notes](supplemental/en/PM_Audit_Supplement_Controls.md)

---

## Methodology (summary)

1. Large-scale bytecode collection on Ethereum mainnet (non–Uniswap-V2-Pair–biased sampling).
2. Static **price-manipulation taint analysis** (pricing reads → external swap/mint sinks).
3. Fetch **Etherscan verified source** for high-confidence hits.
4. Manual audit: trigger source vs cross-contract sink, isomorphic deployment deduplication.
5. Bilingual disclosure documents for platform submission and public archive.

---

## Disclosure status

| Step | Status |
|------|--------|
| Source audit & report drafting | ✅ Complete (draft) |
| Submit to official bug bounty platforms | ⏳ Planned |
| Vendor confirmation / fix verification | ⏳ Pending |
| Public GitHub archive | ⏳ This repository |

Intended platforms include [Immunefi](https://immunefi.com/), [Cantina](https://cantina.xyz/), and project-specific programs where in scope.

---

## License

Reports and documentation in this repository are released under the [MIT License](LICENSE) unless a platform’s disclosure policy requires otherwise for submitted copies.

---

## Contact

For coordination on these findings, open a GitHub issue in this repository or contact the reporter through the platform where the report was originally filed.
