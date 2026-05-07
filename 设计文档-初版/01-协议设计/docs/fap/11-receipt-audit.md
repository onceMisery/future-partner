# 11 - Receipt 审计

> 多签与 hash chain 详见 [receipt-multisig.md](./receipt-multisig.md)。
> 按风险分级的执行路径与 finality 默认策略见 [risk-execution-paths.md](./risk-execution-paths.md)。

## 1. 目标

Receipt 是可验证审计凭证，证明：请求确实由某主体发起、能力确实由某执行节点执行、输出确实对应该输入、高风险动作确实基于有效 Mandate、审计中心确实接收并确认、Receipt 未被篡改、Receipt 链未断裂。

## 2. 可插拔审计组件

```text
SignaturePlugin:       ed25519 / ecdsa / rsa-pss / ...
AuditSinkPlugin:       本地文件 / Rekor / 自研 transparency log / ...
CheckpointStorePlugin: SQLite / 对象存储 / 分布式 KV / ...
```

## 3. 签名角色

| 角色 | 必选 | 说明 |
|---|---|---|
| REQUESTER | 可选 | 请求方签名 |
| EXECUTOR_NODE | 必选 | 执行节点签名 |
| MAIN_NODE | 必选 | 主节点签名 |
| AUDIT_CENTER | high 必选 | 审计中心签名 |
| HUMAN_APPROVER | critical 必选 | 人工审批签名或审批引用 |
| STORAGE_NODE | 涉远程对象时必选 | 对象 hash 与可用性 |

## 4. 策略 → 签名

| 风险 | 必选签名 |
|---|---|
| low | MAIN_NODE |
| medium | MAIN_NODE + EXECUTOR_NODE |
| high | MAIN_NODE + EXECUTOR_NODE + AUDIT_CENTER |
| critical | MAIN_NODE + EXECUTOR_NODE + AUDIT_CENTER + HUMAN_APPROVER |

## 5. 异步补签

调用返回：EXECUTOR + MAIN 在线签 → finality=PROVISIONAL 返回。
后台：AUDIT_CENTER / HUMAN_APPROVER 异步签 → finality=FINALIZED。

## 6. Hash Chain 锚定

```text
Genesis: session 首 Receipt 的 previous_receipt_hash =
  SHA256(session_id || server_nonce || handshake_transcript_hash)

节点重启：从 SQLite session.hash_chain_tip 恢复，启动做 chain 完整性校验。
```

## 7. Checkpoint 与 Transparency Log

每日或每 N=10000 条 Receipt → 生成 checkpoint = Merkle root。
推送到 AuditSinkPlugin（默认本地 append-only log；可选 Sigstore Rekor / 自研 transparency log / 公链锚定）。
