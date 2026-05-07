# risk-execution-paths.md

风险分级执行路径。避免所有调用都支付 high/critical 成本。

## 1. 问题

```text
若默认所有 Invoke 都走完整 Multisig（MAIN + EXECUTOR + AUDIT + HUMAN）:
  - low 风险调用也支付同步多签 RTT
  - AUDIT_CENTER 成为单点瓶颈
  - HUMAN_APPROVER 阻塞所有请求
  
结论: 按风险分级选择执行路径
```

## 2. 四条执行路径

| 路径 | 适用风险 | 特征 |
|---|---|---|
| FAST | low | 单签 Receipt，同步完成 |
| VERIFIED | medium | 双签（MAIN+EXECUTOR）同步 |
| AUDITED_ASYNC | high（默认） | 双签同步 + AUDIT 异步补签，返回 PROVISIONAL |
| AUDITED_SYNC | high（严格配置） | 三签全同步，返回 FINALIZED |
| APPROVED_ASYNC | critical（默认） | 双签同步 + AUDIT/HUMAN 异步，返回 PROVISIONAL，不得用于结算 |
| APPROVED_SYNC | critical（严格配置） | 四签全同步，返回 FINALIZED |

## 3. 路径选择

```text
输入:
  capability.risk_level
  mandate.constraints（如 "require-finalized"）
  caller preference（InvokeRequest.options.finality_policy）
  system policy（tenant/session 级配置）

决策:
  1. 若 caller 指定 require_finalized=true → 强制 FINALIZED 路径
  2. 若 mandate 包含 require-finalized → 强制 FINALIZED 路径
  3. 否则按 risk_level 默认:
     low        → FAST
     medium     → VERIFIED
     high       → AUDITED_ASYNC
     critical   → APPROVED_ASYNC
  4. 若 AUDIT_CENTER 不可达:
     high AUDITED_ASYNC → 仍可返回 PROVISIONAL，标记待补签
     high AUDITED_SYNC  → fail-closed（critical 同理）
```

## 4. Finality 状态

```proto
enum ReceiptFinality {
  RECEIPT_FINALITY_UNSPECIFIED = 0;
  RECEIPT_PENDING = 1;         // 尚未满足路径下任何必选签名
  RECEIPT_PROVISIONAL = 2;     // 关键签名已齐（可以返回客户端）
  RECEIPT_FINALIZED = 3;       // 所有 required 签名已齐
  RECEIPT_COMPENSATED = 4;     // 补偿（最终失败后的回滚标记）
}
```

## 5. 响应契约

```text
PROVISIONAL Receipt:
  可返回给客户端
  客户端可用于展示、继续流程
  不得用于对账、结算、不可逆操作

FINALIZED Receipt:
  完整审计凭证
  可用于任何目的

调用方如何确认 FINALIZED:
  订阅 ReceiptFinalized 事件
  或轮询 GET /receipts/:id
```

## 6. SLA

```text
AUDITED_ASYNC 的 AUDIT 补签 SLA:
  P95 < 30s
  P99 < 5min
  超 24h 未 FINALIZED → 告警

APPROVED_ASYNC 的 HUMAN 补签 SLA:
  预授权 blanket mandate 下 < 5s
  真实人工 P95 < 4h（视业务）
  超 SLA → 告警 + 可选自动取消 + 补偿 Receipt
```

## 7. 预授权 Mandate（降低 HUMAN 等待）

```json
{
  "mandate_id": "blanket-approval-001",
  "issuer": "user:alice",
  "subject": "agent:code-agent",
  "capabilities": ["production.deploy"],
  "constraints": [
    "environment=staging",
    "max-risk:critical",
    "max-budget:10000"
  ],
  "expires_at": "2026-05-08T00:00:00Z",
  "approver_signing_delegated": true,
  "issuer_signature": { ... }
}
```

```text
带 approver_signing_delegated=true 的 mandate:
  Executor 在 mandate 约束内可直接生成含 HUMAN_APPROVER 的签名
  即 APPROVED_SYNC 无需真实人工等待
  但审计时该 HUMAN_APPROVER 签名引用 mandate_id 可追溯
```

## 8. InvokeRequest 指定

```text
options:
  "fap.finality" = "fast" | "verified" | "audited-async" | "audited-sync" | "approved-async" | "approved-sync"
  "fap.require-finalized" = "true" | "false"
  "fap.max-wait-ms" = "3000"
```

不合法组合（如 low 请求 approved-sync）返回 Problem: INVALID_FINALITY_POLICY。

## 9. 补偿 Receipt

```text
APPROVED_ASYNC 的 HUMAN 拒签 / 超时:
  生成 Receipt.finality = COMPENSATED
  previous_receipt_hash 指向原 PROVISIONAL Receipt
  executor 必须执行回滚（若动作不可逆则标记 IRREVERSIBLE_LOSS）
  审计记录完整轨迹
```

## 10. 性能影响

```text
基准 (本地，参考值):
  FAST:                 p50 < 3ms
  VERIFIED:             p50 < 5ms
  AUDITED_ASYNC:        p50 < 5ms（等同 VERIFIED，审计异步）
  AUDITED_SYNC:         p50 < 30ms（+ 审计中心 RTT）
  APPROVED_ASYNC:       p50 < 5ms
  APPROVED_SYNC:        依赖 HUMAN，通常 > 1s

结论:
  默认异步路径保证 p95 < 50ms 对所有风险级
  同步路径仅在严格场景启用
```
