# 12 - 上下文共享与 Handoff

> 凭证细节见 [11-audit-and-receipt.md](./11-audit-and-receipt.md)；脱敏可验证性见 [redaction-policy.md](./redaction-policy.md)。

## 1. 设计原则

```text
不直接共享数据库
所有跨 Agent 上下文必须经过 ContextGrant
SBU 默认不可跨 Agent 共享
跨 Agent 脱敏必须可验证（RedactionReport 签名）
HandoffPacket 必须含审计链头，便于接收方追溯
```

## 2. 共享流程

```text
Agent A ──ShareContextRequest──▶ FAP-ME Gateway
            │
            │  ① Verify mTLS / DID-VC + Intent Mandate
            ▼
        ContentSafetyGuard（仅写场景）
            │
            ▼
        PolicyKernel.authorize（包含 purpose + scope）
            │
            ▼
        Hippocampus.create_snapshot(scope) → snapshot_id + content_hash
            │
            ▼
        PolicyKernel.redact(snapshot, policy_id)
            → (redacted_units, RedactionReport)
            │
            ▼
        Issue ContextGrant（绑定 snapshot + report）
            │
            ▼
        Sign + Audit
            │
            ▼
        ContextGrant returned to Agent A
            │
            ▼ Agent A → Agent B（通过 FAP Control Plane）
            │
        Agent B ──RedeemGrant──▶ FAP-ME Gateway
            │
            ▼
        Verify grant signature + expires_at + revoked
            │
            ▼
        StreamSnapshot（已脱敏，按 ContextGrant 限制）
            │
            ▼
        Audit B 的访问
```

## 3. ContextSnapshot

```text
ContextSnapshot
├── snapshot_id
├── tenant_id, namespace
├── source_session_id
├── content_hash           内容哈希，用于 grant 绑定校验
├── memory_unit_ids[]      包含的 MemoryUnit
├── created_by_agent_did
├── created_at
└── expires_at
```

特性：

- 不可变：任何修改产生新 snapshot
- 与 ContextGrant 绑定：grant.snapshot_ids 必须存在且未过期
- 内容哈希：grant 验证时核对 snapshot 内容是否被篡改

## 4. ContextGrant

```text
ContextGrant
├── grant_id
├── issuer_did               发行方
├── receiver_agent_did       接收方
├── tenant_id, namespace
├── snapshot_ids[]
├── allowed_layers           [L0, L1, L2]
├── allowed_ops              [read, append, merge]
├── redaction_policy_id      已应用的脱敏策略
├── purpose                  标准化 purpose（见 purpose-vocabulary.md）
├── mandate_id               关联 Mandate
├── expires_at
└── signature                issuer 的 JWS 签名
```

`allowed_ops` 严格限制接收方能做什么：

| op | 含义 |
|---|---|
| `read` | 只读 snapshot 内容 |
| `append` | 可在自己 namespace 中创建 foreign episode 引用 |
| `merge` | 可将内容合并到自己的 L1（不进入 L2） |

**默认禁止 `merge`**；需 mandate.purpose ∈ {knowledge_transfer} 时才允许。

## 5. Grant 撤销

```text
ContextGrant.revoke
  → 写审计
  → grant_revocation_table.insert(grant_id, revoked_at)
  → ≤ 60s 全集群可见
  → 后续 RedeemGrant 立即拒绝
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
├── l0_snapshot_ref            ContextSnapshot 引用
├── l1_episode_refs[]
├── l2_fact_refs[]
├── context_grant              内嵌 grant
├── sbu_redaction_report       RedactionReport（可验证签名）
├── redaction_policy_id
├── audit_chain_head           接收方追溯起点
└── packet_signature
```

详见 [redaction-policy.md](./redaction-policy.md)。

## 7. 接收方导入策略

接收方默认不能将 handoff 内容写入长期记忆。

| 权限（grant.allowed_ops） | 行为 |
|---|---|
| `read_l0_snapshot` | 只读现场，过期销毁 |
| `import_l1_temp` | 导入为 foreign episode，不进入 L2 |
| `append_project_l1` | 可追加项目共享剧集（带 foreign 标记） |
| `merge_l2` | 需要高权限和高置信度，purpose 必须为 knowledge_transfer |
| `view_sbu` | 默认禁止，必须显式授权且 RedactionReport 标注 sbu_units_removed = 0 |

## 8. 跨 Agent 脱敏验证

接收方收到 HandoffPacket 后必须：

```rust
let report = packet.sbu_redaction_report;
PolicyKernel.verify_redaction_report(&report, &original_scope)?;
```

校验内容：

```text
1. report_signature 由 executor_did 持有的私钥签名
2. policy_id 是双方都信任的 RedactionPolicy
3. source_scope_hash 与 packet 引用的 snapshot 一致
4. sbu_units_removed >= 期望值（如 grant 标注禁止 SBU 时必须 = sbu_count_in_source）
```

校验失败：拒绝导入并写审计。

## 9. Agent 间通信复用 FAP-1

实际跨 Agent 传输使用 FAP-1 Control Plane：

```text
HandoffPacket → FAP-1 InvokeRequest（capability = "memory.handoff.receive"）
ContextGrant  → FAP-1 InvokeRequest（capability = "memory.context.redeem"）
RecallStream → FAP-1 Data Plane（详见 07-data-plane.md）
```

不重复定义传输协议；FAP-1 负责传输，FAP-ME 负责语义。

## 10. 冲突仲裁

接收方本地存在冲突事实时：

```rust
match resolve_l2_conflict(local_fact, incoming_fact) {
    Supersede { keep_lineage }       → 标记 local 为 superseded
    Fork { mark_for_dream_arbitration } → 等 DreamWorker 推荐
    Reject { reason }                → 拒绝 incoming，写审计
}
```

详见 [retain-score.md](./retain-score.md)。

## 11. 失败模式

| 失败 | 行为 |
|---|---|
| Grant 过期 | 拒绝 redeem，写审计 |
| Grant 已撤销 | 同上 |
| RedactionReport 签名验证失败 | 拒绝 handoff，写审计 |
| Snapshot content_hash 不一致 | 拒绝 redeem，告警 |
| 接收方 mandate 不允许 op | 拒绝，写审计 |
| Foreign episode 试图升级到 L2 | 拒绝（默认禁止） |
