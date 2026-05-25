# Security Policy

## Scope

This repository publishes **draft vulnerability disclosures** for smart contracts on **Ethereum Mainnet (chain ID 1)**. Findings are limited to issues the reporter believes are exploitable under realistic on-chain conditions (e.g., flash-loan–assisted spot price manipulation).

## Reporting to vendors

Primary submission is through official bug bounty / audit platforms (Immunefi, Cantina, Sherlock, or project-specific programs). **Do not** exploit live contracts beyond responsible testing on a local fork.

## Coordinated disclosure

1. **Draft** — Report written and archived in this repo (`Status: Draft`).
2. **Submitted** — Filed with the vendor/platform (`Status: Submitted`).
3. **Acknowledged** — Vendor confirms receipt (`Status: Acknowledged`).
4. **Fixed / Mitigated** — Patch or operational mitigation verified (`Status: Resolved`).
5. **Disclosed** — Public details updated; PoC may be added if policy allows.

Timelines follow each platform’s rules. This GitHub archive may lag platform-private copies during the embargo period.

## What not to report here

- Phishing, key compromise, or off-chain social engineering
- Issues already fixed on-chain with no remaining vulnerable deployments listed in our reports
- Pure centralization / governance risk without a user-exploitable PM path (see `supplemental/`)

## Contact

Open a GitHub issue labeled `security` for questions about these disclosures. For new vulnerabilities in third-party contracts, use the vendor’s official channel.
