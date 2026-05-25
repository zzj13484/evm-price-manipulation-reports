# [漏洞标题 — 简明描述影响]

| 字段 | 内容 |
|------|------|
| **Target** | `0x...`（链：Ethereum Mainnet / 其他） |
| **Contract** | ContractName |
| **Severity** | Critical / High / Medium / Low / Informational |
| **Category** | Price manipulation / Oracle / Access control / Logic / Other |
| **Reporter** | 赵志杰 |
| **Date** | YYYY-MM-DD |
| **Status** | Draft |

---

## Summary

（2–4 句话：漏洞是什么、攻击者能做什么、是否需要特殊权限。）

## Affected scope

- **Deployed address:** `0x...`
- **Verified source:** Etherscan link（如有）
- **In-scope functions:** `functionName(...)` 等
- **Out of scope:** （若适用）

## Root cause

（技术根因：错误定价、预言机可被操纵、缺少访问控制、重入、精度问题等。附关键代码片段或文件路径。）

```solidity
// 示例：粘贴最小相关代码
```

## Attack scenario

1. （前置条件：资金、角色、市场状态）
2. （步骤 1）
3. （步骤 2）
4. （结果：获利 / 清算 / 铸币等）

## Impact

- **Confidentiality:** N/A / Low / ...
- **Integrity:** ...
- **Availability:** ...
- **Financial impact:** （可量化则写估算思路）

## Proof of concept

（Foundry / Hardhat / 手动交易步骤。无完整 PoC 时写「待补充」并说明复现思路。）

```bash
# 复现命令占位
```

## Recommendations

1. （修复建议 1）
2. （修复建议 2）

## References

- 审计备注 / 内部检测：链接或文件路径
- 同类历史漏洞：（如有）

---

## Platform checklist（提交前自检）

- [ ] 标题与 Summary 一致、无夸大
- [ ] 地址与链 ID 正确
- [ ] Impact 与 Severity 匹配平台定级标准
- [ ] PoC 可独立复现或步骤足够清晰
- [ ] 不含未公开的第三方 0day 误报
