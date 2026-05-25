# Report index

| Report ID | File prefix | Chain | Severity | Category |
|-----------|-------------|-------|----------|----------|
| ETH-PM-2026-001 | `eth_0x0bafc174…_Vogue` | Ethereum | High | V4 spot oracle / batch settlement |
| ETH-PM-2026-002 | `eth_0x1d442…_Rover` | Ethereum | High | V3 slot0 + Router zero min-out |
| ETH-PM-2026-003 | `eth_0x7f622…_MasterChef` | Ethereum | High | V2 Router zero min-out in `reward()` |
| ETH-PM-2026-004 | `eth_0x7d3a6d…_Zap` | Ethereum | High | Curve zero min_dy / min mint |
| ETH-PM-2026-005 | `eth_0x71ea…_Vault` | Ethereum | High | Stale exchangePrice vs live oracle |

## Isomorphic deployments (same report)

| Family | Additional addresses |
|--------|---------------------|
| Vogue | `0xb3c368bc…`, `0x19f4b121…`, `0x9743de23…`, `0x32956d93…` |
| Rover | `0x491c3a40…`, `0x1d8227d7…`, `0xb9ae20b2…` |
| MasterChef | `0xdae1acc2…` |

## Languages

- **English:** `reports/en/`
- **中文：** `reports/zh-CN/`

Each file uses the naming convention:

```
{chain}_{0xaddress}_{ContractName}.md
```

Example: `eth_0x0bafc174bf07ec3d0eafdee06df7d07377108c0c_Vogue.md`
