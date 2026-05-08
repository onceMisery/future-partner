# 12 - 上下文共享与 Handoff

> 凭证细节见 [11-audit-and-receipt.md](./11-audit-and-receipt.md)；脱敏报告边界见 [redaction-policy.md](./redaction-policy.md)；FAP-1 绑定见 [fap1-binding.md](./fap1-binding.md)。

## 1. 设计原则

```text
不直接共享数据库
所有跨 Agent 上下文必须经过 ContextGrant
SBU 默认不可跨 Agent 原文共享
跨 Agent 脱敏必须附 RedactionReport，但报告不是“绝对无遗漏证明”
HandoffPacket 必须含 FAP-1 Receipt 与 FME 审计链头，便于接收方追溯
```

Grant 的撤销只能阻止后续 `RedeemGrant`。一旦接收方已经读取或导入内容，必须通过合规删除协作、HardForget outbox 或接收方本地策略处理，不能把访问撤销表述为“已复制数据撤销”。

## 2. 共享流程

```text
Agent A ──FAP-1 Invoke(memory.share)──▶ FAP-ME Gateway
            │
            │  ① Verify mTLS / DID-VC + FAP-1 Mandate
            ▼
        MandateVerifier（FME constraints）
            │
            ▼
        PolicyKernel.authorize（purpose + capability triple + scope）
            │
            ▼
        Hippocampus.create_snapshot(scope) → snapshot_id + content_hash
            │
            ▼
        PolicyKernel.redact(snapshot, policy_id)
            → (redacted_units, RedactionReport)
            │
            ▼
        Issue ContextGrant（绑定 snapshot + report + FAP receipt）
            │
            ▼
        Sign + Audit
            │
            ▼
        ContextGrant returned to Agent A
            │
            ▼ Agent A → Agent B（通过 FAP Control Plane）
            │
        Agent B ──FAP-1 Invoke(memory.context.redeem)──▶ FAP-ME Gateway
            │
            ▼
        Verify grant signature + expires_at + revoked + receiver DID
            │
            ▼
        StreamSnapshot（已脱敏，按 ContextGrant 限制）
            │
            ▼
        Audit B 的访问
```

FME 不重复定义跨 Agent 传输协议；FAP-1 负责传输、会话、Receipt 与 Data Plane，FME 负责记忆语义。

## 3. ContextSnapshot

```text
ContextSnapshot
├── snapshot_id
├── tenant_id, namespace
├── source_session_id
├── content_hash             内容哈希，用于 grant 绑定校验
├── memory_unit_ids[]        包含的 MemoryUnit
├── source_scope_hash        RedactionReport 绑定
├── created_by_agent_did
├── created_at
└── expires_at
```

特性：

- 不可变：任何修改产生新 snapshot
- 与 ContextGrant 绑定：grant.snapshot_ids 必须存在且未过期
- 内容哈希：grant 验证时核对 snapshot 内容是否被篡改
- source_scope_hash：用于验证 RedactionReport 覆盖范围声明

## 4. ContextGrant

```text
ContextGrant
├── grant_id
├── issuer_did
├── receiver_agent_did
├── tenant_id, namespace
├── snapshot_ids[]
├── allowed_triples[]          capability/layer/action
├── access_mode                READ_ONLY | IMPORT_FOREIGN_EPISODE | APPEND_PROJECT_L1 | MERGE_L2
├── redaction_policy_id
├── redaction_report_id
├── purpose                    标准化 purpose（见 purpose-vocabulary.md）
├── mandate_id                 FAP-1 Mandate id
├── fap_receipt_id
├── expires_at
└── signature                  issuer 的 JWS 签名
```

Grant 权限必须由 `allowed_triples` 与 `access_mode` 共同表达：

| 字段 | 职责 |
|---|---|
| `allowed_triples` | 接收方可调用的 FME capability/layer/action |
| `access_mode` | 接收方对共享内容的导入语义 |

禁止重新引入 `allowed_ops` 字符串数组。

## 5. Grant 撤销

```text
ContextGrant.revoke
  → 写 FAP-1 Receipt
  → 写 FME AuditEvent
  → grant_revocation_table.insert(grant_id, revoked_at)
  → ≤ 60s 全集群可见
  → 后续 RedeemGrant 立即拒绝
```

撤销语义：

```text
未 redeem 的 grant              后续不可读取
已 redeem 但未导入的 stream      后续 chunk 停止
已导入为 foreign episode         需要接收方执行本地 forget 协作
已合并到 L1/L2                  必须依赖 lineage + HardForget outbox 追踪
```

## 6. HandoffPacket

```text
HandoffPacket
├── packet_id
├── task_id
├── from_agent_did, to_agent_did
├── goal
├── current_state_summary
├── completed_steps[]
├── failed_attempts[]
├── open_questions[]
├── constraints[]
├── tool_state_refs[]
├── l0_snapshot_ref              ContextSnapshot 引用
├── l1_episode_refs[]
├── l2_fact_refs[]
├── context_grant                内嵌 grant
├── sbu_redaction_report         RedactionReport（签名报告）
├── redaction_policy_id
├── audit_chain_head             FME 审计链追溯起点
├── fap_receipt_id
└── packet_signature
```

详见 [redaction-policy.md](./redaction-policy.md)。

## 7. 接收方导入策略

接收方默认不能将 handoff 内容写入长期记忆。

| `access_mode` | 行为 |
|---|---|
| `READ_ONLY` | 只读 snapshot 内容，过期销毁 |
| `IMPORT_FOREIGN_EPISODE` | 导入为 foreign episode，不进入 L2 |
| `APPEND_PROJECT_L1` | 可追加项目共享剧集，带 foreign 标记 |
| `MERGE_L2` | 需要高权限、高置信度和 `knowledge_transfer` purpose |

SBU 原文访问不通过 `access_mode` 表达，必须由 FAP-1 Mandate constraint `memory.sbu_access = RAW_ALLOWED` 显式授权，并要求 elevated mandate、强审计和接收方本地隔离策略。

## 8. 跨 Agent 脱敏报告验证

接收方收到 HandoffPacket 后必须：

```rust
let report = packet.sbu_redaction_report;
PolicyKernel.verify_redaction_report(&report, &packet.source_scope_hash)?;
```

校验内容：

```text
1. report_signature 由 executor_did 持有的私钥签名
2. policy_id / policy_version 是双方都信任的 RedactionPolicy
3. classifier_id / classifier_version 被报告声明
4. source_scope_hash 与 packet 引用的 snapshot 一致
5. 跨租户/跨组织时 sbu_manifest 必须存在，且 scan_coverage 满足接收方策略
6. actions_taken 与 grant 的 sbu_access 要求一致
```

校验通过并不等价于“证明无 SBU 残留”。它只说明可信 executor 按声明的 policy 与 classifier 版本执行过脱敏。校验失败：拒绝导入并写审计。

## 9. Agent 间通信复用 FAP-1

实际跨 Agent 传输使用 FAP-1：

```text
HandoffPacket → FAP-1 InvokeRequest（capability_id = "memory.handoff.receive"）
ContextGrant  → FAP-1 InvokeRequest（capability_id = "memory.context.redeem"）
RecallStream  → FAP-1 Data Plane（详见 07-data-plane.md）
```

大对象、长上下文和附件必须走 FAP-1 Data Plane 的 `DataOpen/DataChunkMeta/DataCommit/DataAck`，不能塞入控制面 envelope。

## 10. 冲突仲裁

接收方本地存在冲突事实时：

```rust
match resolve_l2_conflict(local_fact, incoming_fact) {
    Supersede { keep_lineage } => mark_superseded(local_fact),
    Fork { mark_for_dream_arbitration } => create_dream_proposal(),
    Reject { reason } => audit_reject(incoming_fact, reason),
}
```

详见 [retain-score.md](./retain-score.md)。

## 11. 失败模式

| 失败 | 行为 |
|---|---|
| Grant 过期 | 拒绝 redeem，写审计 |
| Grant 已撤销 | 同上 |
| RedactionReport 签名验证失败 | 拒绝 handoff，写审计 |
| `sbu_manifest` 缺失（跨租户/跨组织） | 拒绝导入 |
| Snapshot content_hash 不一致 | 拒绝 redeem，告警 |
| 接收方 mandate 不允许 requested_triple | 拒绝，写审计 |
| Foreign episode 试图升级到 L2 | 拒绝（默认禁止） |
| 已复制数据被源侧 revoke | 触发接收方 forget 协作，不声称自动撤销 |
