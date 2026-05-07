# receipt-multisig.md

Multisig Receipt 审计凭证规范。

## 1. 目标

Receipt 不是调用日志，是**可验证审计凭证**，必须证明：

```text
请求确实由某主体发起
能力确实由某执行节点执行
输出确实对应该输入
高风险动作确实基于有效 Mandate
审计中心确实接收并确认
Receipt 未被篡改
Receipt 链未断裂
```

## 2. 可插拔视角

```text
Receipt 格式、hash chain、多签策略固化为协议契约
但签名算法、审计 Sink、Transparency Log 均为插件:

  SignaturePlugin:       ed25519 / ecdsa / rsa-pss / ...
  AuditSinkPlugin:       本地文件 / Rekor / 自研 transparency log / ...
  CheckpointStorePlugin: SQLite / 对象存储 / 分布式 KV / ...
```

## 3. 签名角色

| 角色 | 必选 | 说明 |
|---|---|---|
| REQUESTER | 可选 | 请求方签名，证明请求意图 |
| EXECUTOR_NODE | 必选 | 执行节点签名 |
| MAIN_NODE | 必选 | 主节点签名 |
| AUDIT_CENTER | high 必选 | 审计中心签名 |
| HUMAN_APPROVER | critical 必选 | 人工审批签名或审批引用 |
| STORAGE_NODE | 涉远程对象时必选 | 对象 hash 与可用性 |

## 4. Receipt 定义

```proto
message Receipt {
  string receipt_id = 1;
  string invocation_id = 2;
  string batch_id = 3;

  string subject = 4;
  string capability_id = 5;
  string mandate_id = 6;

  string executor_node_id = 7;
  string main_node_id = 8;

  string status = 9;

  bytes request_hash = 10;
  bytes response_hash = 11;
  bytes transcript_hash = 12;
  bytes previous_receipt_hash = 13;

  repeated ObjectReceipt object_receipts = 14;

  google.protobuf.Timestamp created_at = 15;

  ReceiptFinality finality = 16;

  MultisigPolicy multisig_policy = 20;
  repeated ReceiptSignature signatures = 21;
}

enum ReceiptFinality {
  RECEIPT_FINALITY_UNSPECIFIED = 0;
  RECEIPT_PENDING = 1;
  RECEIPT_PROVISIONAL = 2;
  RECEIPT_FINALIZED = 3;
}
```

## 5. 策略 → 签名

| 风险 | 必选签名 |
|---|---|
| low | MAIN_NODE |
| medium | MAIN_NODE + EXECUTOR_NODE |
| high | MAIN_NODE + EXECUTOR_NODE + AUDIT_CENTER |
| critical | MAIN_NODE + EXECUTOR_NODE + AUDIT_CENTER + HUMAN_APPROVER |

## 6. 异步补签

降低延迟：

```text
调用返回:
  EXECUTOR + MAIN 在线签 → finality=PROVISIONAL 返回

后台:
  AUDIT_CENTER 异步签 → finality 仍 PROVISIONAL
  HUMAN_APPROVER 异步签或引用预授权 mandate → finality=FINALIZED
  推送 ReceiptFinalized 事件

约束:
  PROVISIONAL 可返回客户端，不得用于对账/结算
  超 24h 未 FINALIZED 触发告警
  critical 无法获得 HUMAN_APPROVER → 回滚并生成补偿 Receipt
```

## 7. 预授权 Mandate

为减少 HUMAN_APPROVER 同步等待：

```text
用户预签一个受约束的 blanket approval mandate
Executor 在 mandate 约束内直接生成含 HUMAN_APPROVER 签名的 Receipt

mandate 声明:
  approver_signing_delegated = true
  max_risk = critical
  constraints = [...]
  expires_at
  budget_units
```

## 8. Hash Chain 锚定

```text
Genesis:
  session 首 Receipt 的 previous_receipt_hash =
    SHA256(session_id || server_nonce || handshake_transcript_hash)

跨会话:
  不强制延续
  同 subject 可聚合（可选）

节点重启:
  从 SQLite session.hash_chain_tip 恢复
  启动做 chain 完整性校验
  发现断链 → DEGRADED 模式，拒绝 high/critical 直到人工介入
```

## 9. Checkpoint 与 Transparency Log

```text
每日 或 每 N=10000 条 Receipt → 生成 checkpoint
checkpoint = Merkle root of 该周期所有 receipt_hash

推送到 AuditSinkPlugin:
  默认: 本地 append-only log
  可选: Sigstore Rekor
  可选: 自研 transparency log
  可选: 公链锚定

查询接口:
  GET /fap/v1/audit/checkpoints
  GET /fap/v1/audit/checkpoints/:id/proof
  可验证任意 Receipt 的 Merkle 证明
```

## 10. 验证流程

```text
1. 校验 receipt_id 格式
2. 校验 request_hash 与 invocation 记录一致
3. 校验 response_hash 与 invocation 记录一致
4. 校验 transcript_hash
5. 校验 previous_receipt_hash 指向链内
6. 读取 multisig_policy
7. 检查 required_signers 齐全（按 finality 状态判断）
8. 校验每个签名 kid 和证书链
9. 校验签名时间在会话有效期内
10. 校验 high 动作有 AUDIT_CENTER 签名
11. 校验 critical 动作有 HUMAN_APPROVER 签名或 mandate 引用
12. 校验 object_receipts 与实际 ObjectRef 一致
13. 校验 Receipt 在对应 checkpoint Merkle tree 内（可选强度）
```

## 11. 签名字节

复用 signing-canonical.md 规则：

```text
signed_bytes = protobuf_deterministic_encode(Receipt 去除 signatures)
所有签名者对同一份 signed_bytes 签
顺序由 multisig_policy.required_signers 决定
```

## 12. Storage

见 17-storage-design.md 中的 `receipt` 和 `receipt_checkpoint` 表。
