# 11 - 审计与凭证

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

```text
AuditEvent
├── event_id              UUID v7
├── prev_event_hash       哈希链前向引用
├── event_hash            本事件哈希
├── tenant_id
├── namespace
├── session_id
├── actor_did             操作者 DID
├── operation             memory.retrieve / store / share / forget / handoff
│                         dream.propose / dream.approve / chain.verify / ...
├── target_ref            memory_id / grant_id / proposal_id / receipt_id
├── mandate_id            FAP-1 Mandate id
├── fap_receipt_id        FAP-1 Receipt id
├── fap_invocation_id     FAP-1 invocation / message id
├── fap_envelope_hash     FAP-1 envelope canonical hash
├── fap_finality          PROVISIONAL / FINALIZED
├── capability_triple     capability + layer + action
├── content_hash
├── object_ref_hashes[]
├── index_version
├── timestamp
└── signature             AuditKernel 签名
```

所有字段必须使用 canonical encoding 参与签名。`tenant_id` 与 `namespace` 由 TenantKernel 注入，不能信任请求体。

## 3. 哈希链

```text
event_hash = SHA256(
  prev_event_hash ||
  tenant_id || namespace || session_id ||
  actor_did || operation || target_ref ||
  mandate_id ||
  fap_receipt_id || fap_invocation_id || fap_envelope_hash || fap_finality ||
  canonical(capability_triple) ||
  content_hash || canonical(object_ref_hashes) ||
  timestamp
)
```

每个 tenant/namespace 维护独立 chain head：

```text
ChainState {
  tenant_id
  namespace
  current_tip
  last_verified_at
  last_verified_event
}
```

实现不得使用单个全局 `Mutex<ChainState>` 作为服务端多租户瓶颈。服务端模式应使用按 `(tenant_id, namespace)` 分片的状态表或 compare-and-swap 更新；边缘模式可退化为单租户本地状态。

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

ForgetReceipt 必须区分本地提交与全局协调状态：

```text
ForgetReceipt
├── receipt_id
├── request_id
├── mode                         soft | hard
├── status                       COMMITTED_LOCALLY | GLOBALLY_RECONCILED | RECONCILE_FAILED
├── local_committed_at
├── globally_reconciled_at?
├── target_ids[]
├── cascaded_ids[]               级联清除的下游
├── local_mutations[]            SQLite/PG、本地索引、本地 CAS
├── external_mutations[]         S3/Kafka/Qdrant/远端 sink
├── pending_external_systems[]
├── mutation_ledger_id
├── reconciliation_deadline
├── audit_event_id
├── fap_receipt_id
└── signature                    JWS by AuditKernel
```

`COMMITTED_LOCALLY` 只表示本地事务已提交，不表示 S3/Kafka/Qdrant/远端接收方已经清除。跨系统完成后可签发同一 `receipt_id` 的更新版或 `ForgetReconciledReceipt`。详见 [forget-engine.md](./forget-engine.md) 与 [outbox-reconciliation.md](./outbox-reconciliation.md)。

### 7.4 RedactionReport

```text
RedactionReport
├── report_id
├── policy_id, policy_version
├── classifier_id, classifier_version
├── applied_at
├── executor_did
├── source_scope_hash
├── sbu_manifest
│   ├── scan_coverage            full | sampled | partial
│   ├── sample_rate
│   ├── patterns_count_scanned
│   └── source_unit_count
├── audit_sample_hashes[]        可选抽样审计
├── actions_taken[]
├── sbu_units_removed
├── sbu_tombstones_included
├── total_units_before / after
└── report_signature             JWS by executor
```

RedactionReport 证明的是“按该 policy 与 classifier_version 执行并移除了报告中声明的命中项”，不证明“源数据绝对无 SBU 残留”。接收方信任 = 信任 policy + 信任 classifier + 信任 executor。跨租户/跨组织共享必须携带 `sbu_manifest`。

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
| `MEMORY_FORGET / DELETE_HARD` | `FINALIZED` 后执行本地提交 |
| `MEMORY_DREAM / APPROVE` | `FINALIZED` 后应用 mutation |
| `MEMORY_ADMIN / ADMIN` | `FINALIZED` 后执行 |
| 普通 retrieve | 可接受 `PROVISIONAL`，但返回审计需补最终状态 |

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

仅持有 `compliance_audit` purpose，且三元组为 `MEMORY_AUDIT / AUDIT_CHAIN / QUERY` 的 mandate 可查询。详见 [purpose-vocabulary.md](./purpose-vocabulary.md)。

## 10. 不可绕过保证

```text
插件不能直接写 chain_state
插件不能伪造 event_hash
插件不能跳过 audit（每次 plugin.invoke 由 Orchestrator 自动写）
插件不能修改 ChainVerifyReport 结果
插件不能创建未绑定 FAP-1 Receipt 的高风险 FME AuditEvent
```

通过 [kernel-contract.md](./kernel-contract.md) 中的负例测试矩阵保证。
