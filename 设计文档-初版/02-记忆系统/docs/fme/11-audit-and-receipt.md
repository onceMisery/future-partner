# 11 - 审计与凭证

> 所有签名对象的 canonical 字段顺序与 hash 公式见 [signing-canonical.md](./signing-canonical.md)。本文档负责语义、流程与 trait 定义。

## 1. 设计原则

```text
所有记忆操作 → 写 FME AuditEvent
所有 FAP-1 调用 → 绑定 FAP-1 Receipt
审计哈希链不可被插件改写
审计完整性可定期验证
AuditSink 可插件化，但 hash chain 与 fan-out 策略由 AuditKernel 控制
凭证（ContextGrant、ForgetReceipt、HandoffPacket、RedactionReport）必须签名
```

FME 审计不是 FAP-1 Receipt 的替代品。FAP-1 Receipt 负责协议调用可追责；FME AuditEvent 负责记忆语义可追责。两者必须通过稳定字段强绑定，避免出现两条链互相无法证明同一操作的情况。

## 2. AuditEvent 结构

`AuditEvent` 的 canonical 字段顺序与 hash 输入字段在 [signing-canonical.md §2](./signing-canonical.md) 冻结。本节仅提供阅读概览：

```text
AuditEvent
├── event_id              UUID v7
├── prev_event_hash       哈希链前向引用
├── event_hash            本事件哈希
├── tenant_id
├── namespace
├── session_id
├── actor_did             操作者 DID
├── operation             memory.retrieve / memory.store / memory.share
│                         memory.forget.soft / memory.forget.hard
│                         memory.handoff.create / memory.handoff.receive
│                         memory.dream.propose / memory.dream.approve
│                         memory.audit.query / memory.admin.rebuild
│                         chain.verify / plugin.invoke / ...
├── target_ref            memory_id / grant_id / proposal_id / receipt_id
├── mandate_id            FAP-1 Mandate id
├── fap_receipt_id        FAP-1 Receipt id
├── fap_invocation_id     == FAP-1 Envelope.message_id
├── fap_envelope_hash     FAP-1 envelope canonical hash
├── fap_finality          OPTIMISTIC | VERIFIED | FINALIZED
├── capability_triple     capability + layer + action
├── content_hash
├── object_ref_hashes[]
├── index_version
├── timestamp_micros      Unix 微秒
└── signature             AuditKernel 签名（覆盖 event_hash）
```

`fap_invocation_id` 与 FAP-1 `Envelope.message_id` 是同一字段；任何文档表述不得引入新别名。`fap_finality` 词汇仅允许 `OPTIMISTIC | VERIFIED | FINALIZED` 三档（FAP-1 标准），不允许 `PROVISIONAL`。

`tenant_id` 与 `namespace` 由 TenantKernel 注入，不能信任请求体。

## 3. 哈希链

`event_hash` 公式见 [signing-canonical.md §2.2](./signing-canonical.md)。本节只描述 chain head 维护与并发：

```text
ChainState {
  tenant_id
  namespace
  current_tip
  revision                单调递增（CAS 比较的 base）
  last_verified_at
  last_verified_event
}
```

每个 `(tenant_id, namespace)` 维护独立 chain head。并发更新模型、SchemaPerTenant / DbPerTenant 下的全局注册中心、leader 选举见 [chain-state-concurrency.md](./chain-state-concurrency.md)。

实现不得使用单个全局 `Mutex<ChainState>` 作为服务端多租户瓶颈。

## 4. AuditKernel API

```rust
pub struct AuditKernel {
    sinks: Vec<Arc<dyn AuditSink>>,                 // sink 可插件化
    chain_store: Arc<dyn ChainStateStore>,          // 按 tenant/namespace 分片
    fanout_policy: AuditFanoutPolicy,
    integrity_scheduler: Arc<IntegrityScheduler>,
}

pub enum AuditFanoutPolicy {
    FailClosedAll,       // 所有 sink 必须成功，适用于合规关键路径
    Quorum { min: u8 },  // 至少 N 个 sink 成功
    OutboxReplay,        // 本地链先提交，外部 sink 走 outbox reconciliation
}

impl AuditKernel {
    /// 写入审计事件，原子更新本地 chain_state。
    pub fn append(&self, input: AuditEventInput) -> Result<AuditReceipt>;

    /// 验证哈希链完整性（点对点或全量）。
    pub fn verify_chain(
        &self,
        tenant: TenantRef,
        from: EventId,
        to: EventId,
    ) -> Result<ChainVerifyReport>;

    /// 检测断链（防篡改告警）。
    pub fn detect_tampering(&self, tenant: TenantRef) -> Option<TamperingAlert>;
}
```

合规关键操作默认使用 `FailClosedAll` 或 `OutboxReplay`。如果外部 sink 是 S3/Kafka/远端归档，不能假装与本地事务 ACID；应使用 [outbox-reconciliation.md](./outbox-reconciliation.md) 的 ledger 与 reconciler。

## 5. AuditSink Plugin

```rust
pub trait AuditSink: Plugin {
    /// 仅写入；不计算 hash chain，不决定 finality。
    fn append(&self, event: &SignedAuditEvent) -> Result<()>;
    fn read_range(&self, tenant: TenantRef, from: EventId, to: EventId) -> Result<Vec<SignedAuditEvent>>;
}
```

实现示例：

| Sink | 用途 |
|---|---|
| `audit-sink-sqlite` | 边缘部署，append-only WAL |
| `audit-sink-kafka` | 服务端流式审计 |
| `audit-sink-s3` | 长期归档，CAS 不可变 |
| `audit-sink-otel` | OpenTelemetry trace 联动 |

Sink 可以插件化，但只能接收 `SignedAuditEvent`。插件不能改写 hash、signature、tenant 或 finality。

## 6. 完整性验证

### 6.1 周期性自检

```text
默认间隔：每天 00:30
范围：当天所有 tenant/namespace 的 chain
失败处理：写 TamperingAlert 事件 → 触发告警通道
```

### 6.2 主动验证

```rust
let report = audit_kernel.verify_chain(tenant_ref, from, to)?;
match report {
    ChainVerifyReport::Ok { count } => { ... }
    ChainVerifyReport::Broken { broken_at, expected, actual } => {
        // 立即告警
    }
}
```

### 6.3 断链处理

```text
检测到断链 → 写 TamperingAlert 审计 → 锁定 tenant/namespace 写入 → 等待人工介入
绝不自动修复 → 否则掩盖攻击痕迹
```

## 7. 凭证类型

### 7.1 ContextGrant

```text
ContextGrant
├── grant_id
├── issuer_did
├── receiver_agent_did
├── tenant_id, namespace
├── snapshot_ids[]             绑定的 ContextSnapshot
├── allowed_triples[]          capability/layer/action
├── redaction_policy_id
├── redaction_report_id
├── purpose                    标准化 purpose
├── mandate_id                 FAP-1 Mandate id
├── fap_receipt_id
├── expires_at
└── signature                  JWS by issuer
```

详见 [12-context-sharing.md](./12-context-sharing.md)。

### 7.2 HandoffPacket

附带 RedactionReport 与 ContextGrant，详见 [redaction-policy.md](./redaction-policy.md)。

### 7.3 ForgetReceipt

`ForgetReceipt` 的权威 schema 与双签名规则见 [signing-canonical.md §3](./signing-canonical.md)。本节仅描述语义：

```text
ForgetReceipt
├── receipt_id
├── request_id
├── mode                          SOFT | HARD
├── status                        COMMITTED_LOCALLY | GLOBALLY_RECONCILED | RECONCILE_FAILED
├── 阶段一（必填）
│   ├── committed_locally_at_micros
│   ├── locally_committed_mutations[]
│   ├── target_ids[]
│   ├── cascaded_ids[]
│   └── local_signature
├── 阶段二（GLOBALLY_RECONCILED 时填）
│   ├── globally_reconciled_at_micros
│   ├── external_evidence[]            SystemReconcileEvidence
│   └── reconciled_signature
├── 进行中
│   ├── pending_external_systems[]
│   └── reconciliation_deadline_micros
├── 失败（RECONCILE_FAILED 时填）
│   ├── intervention_reason
│   └── failed_mutations[]
├── 关联
│   ├── mutation_ledger_id
│   ├── audit_event_id
│   └── fap_receipt_id
└── 取消
    └── cancelled_after_local_commit    bool
```

`COMMITTED_LOCALLY` 只表示本地事务已提交，不表示 S3/Kafka/Qdrant/远端接收方已经清除。跨系统完成后由 reconciler 在同一 `receipt_id` 上更新状态、追加 `reconciled_signature`。详见 [forget-engine.md](./forget-engine.md) 与 [outbox-reconciliation.md](./outbox-reconciliation.md)。

`cancelled_after_local_commit = true` 表示请求方在阶段一完成后取消了 `wait_for_reconciled` 等待，但 mutation_ledger 中的 mutation 仍由 reconciler 异步达成；阶段二完成时仍会签发 `reconciled_signature`。

### 7.4 RedactionReport

`RedactionReport` 的权威 schema、canonical 字段顺序、签名规则见 [signing-canonical.md §4](./signing-canonical.md) 与 [redaction-policy.md §8](./redaction-policy.md)。本节仅描述语义边界：

```text
RedactionReport 证明：
  ✅ 该 executor_did 按声明的 policy_id/policy_version 与 classifier_version 执行过脱敏
  ✅ source_scope_hash 与被共享 snapshot 匹配
  ✅ 报告中 actions_taken 已执行

RedactionReport 不证明：
  ❌ 源数据中所有 SBU 都被识别
  ❌ classifier 没有漏检
  ❌ policy 规则覆盖了所有危险模式
```

跨租户/跨组织共享必须携带 `sbu_manifest`，且 `classifier_version ≥ 接收方 mandate.classifier_version_min`。验签流程见 [redaction-policy.md §10](./redaction-policy.md)。

### 7.5 DreamProposal（含审批轨迹）

```text
DreamProposal
├── proposal_id
├── status                      状态机字段
├── risk_level
├── sbu_blocked
├── mutations[]
├── reason
├── proposer_did
├── approver_did                非 null 当 status = APPROVED 后
├── approval_audit_event_id
├── fap_receipt_id
├── created_at, expires_at
└── signature
```

详见 [dream-state-machine.md](./dream-state-machine.md)。

## 8. 哈希链与 FAP-1 Receipt 的关系

```text
FAP-1 Receipt              负责协议级调用的审计、risk/finality、invoke 生命周期
FAP-ME AuditEvent          负责记忆语义、tenant/namespace、memory target、cascade 范围

强绑定字段：
  fap_receipt_id
  fap_invocation_id
  fap_envelope_hash
  fap_finality
  mandate_id
```

高风险操作要求：

| 操作 | 最低 FAP-1 finality |
|---|---|
| `memory.forget.hard` (DELETE_HARD) | `FINALIZED` 后执行本地提交 |
| `memory.dream.approve` (APPROVE) | `FINALIZED` 后应用 mutation |
| `memory.admin.rebuild` (ADMIN) | `FINALIZED` 后执行 |
| 普通 `memory.retrieve` | 可接受 `OPTIMISTIC`，但返回审计需补最终状态 |

详见 FAP-1 [11-receipt-audit.md](../../../01-协议设计/docs/fap/11-receipt-audit.md)。

## 9. 审计查询

```proto
message MemoryAuditQuery {
  string tenant_id = 1;
  string namespace = 2;
  optional string session_id = 3;
  optional string actor_did = 4;
  optional string target_ref = 5;
  optional google.protobuf.Timestamp from = 6;
  optional google.protobuf.Timestamp to = 7;
  uint32 limit = 8;
  string mandate_id = 9;
  string fap_receipt_id = 10;
}
```

仅持有 `compliance_audit` purpose，且 `requested_triple = (MEMORY_AUDIT, AUDIT_CHAIN, QUERY)` 的 mandate 可查询。详见 [purpose-vocabulary.md](./purpose-vocabulary.md)。

## 10. 不可绕过保证

```text
插件不能直接写 chain_state
插件不能伪造 event_hash
插件不能跳过 audit（每次 plugin.invoke 由 Orchestrator 自动写）
插件不能修改 ChainVerifyReport 结果
插件不能创建未绑定 FAP-1 Receipt 的高风险 FME AuditEvent
```

通过 [kernel-contract.md](./kernel-contract.md) 中的负例测试矩阵保证。
