# outbox-reconciliation.md

> HardForget 与跨系统记忆 mutation 的最终一致性方案。基于 transactional outbox + saga + reconciliation。`ForgetReceipt` / `mutation_ledger` 的权威 schema 见 [signing-canonical.md](./signing-canonical.md) 与 [13-storage-design.md](./13-storage-design.md)。

## 1. 设计原则

```text
本地强一致：
  本地 SQLite/PG 行 + 本地索引文件 + 本地 CAS 对象 = 单事务

跨系统最终一致：
  外部 vector DB / 远程对象存储 / 外部审计 sink / 远端接收方
  = outbox + saga + 幂等 mutation ledger + reconciler

ForgetReceipt 双状态：
  COMMITTED_LOCALLY        本地完成
  GLOBALLY_RECONCILED      所有外部系统达成最终一致
  请求方可选阻塞等待或异步轮询
```

## 2. 为什么不能单事务

```text
SQL 事务边界：
  ✅ SQLite WAL（同进程）
  ✅ PostgreSQL（单实例）
  ✅ 本地 mmap 索引文件（fsync 保证）
  ✅ 本地 CAS 对象（rename 原子性）

无法纳入：
  ❌ Qdrant collection delete（远程 RPC）
  ❌ S3 object delete（远程 RPC）
  ❌ Kafka audit fan-out（异步消费）
  ❌ 远端 ContextGrant 接收方的本地缓存（不在我们控制下）

强制单事务 = 谎言。HardForget 必然是分布式过程。
```

## 3. 双阶段架构

```text
┌──────────────────────────────────────────────────────────────────┐
│  阶段一：本地强一致（单事务）                                      │
│                                                                   │
│  SQL transaction：                                                │
│    1. memory_unit set deleted_at                                  │
│    2. memory_unit_security 删除                                   │
│    3. memory_lineage 删除                                         │
│    4. context_grant 标记 revoked                                  │
│    5. mutation_ledger insert（pending mutations 列表）            │
│    6. audit_event insert（status=COMMITTED_LOCALLY）              │
│  COMMIT                                                           │
│                                                                   │
│  → 本地用户无法再读取已删数据                                       │
│  → 返回 ForgetReceipt { status: COMMITTED_LOCALLY, mutations[] } │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  阶段二：跨系统异步达成（saga）                                    │
│                                                                   │
│  Reconciler（Cerebellum 后台任务）：                              │
│    for mutation in mutation_ledger where status = PENDING:        │
│      try:                                                         │
│        execute(mutation)        # vector_index.mark_deleted /     │
│                                 # s3.delete / kafka.send / ...    │
│        mutation_ledger.set status = SUCCESS                       │
│      except:                                                      │
│        mutation_ledger.attempts += 1                              │
│        if attempts > MAX_RETRIES:                                 │
│          mutation_ledger.set status = NEEDS_INTERVENTION          │
│          告警                                                     │
│                                                                   │
│  完成所有 mutation 后：                                           │
│    audit_event.update status = GLOBALLY_RECONCILED                │
│    签发 ForgetReconciledReceipt                                   │
│    通知请求方（若有 wait_for_reconciled 订阅）                    │
└──────────────────────────────────────────────────────────────────┘
```

## 4. mutation_ledger 表

权威 DDL 见 [13-storage-design.md §3](./13-storage-design.md)。要点：

```text
- 主键 ledger_id；UNIQUE(target_system, idempotency_key) 保证跨重试幂等
- tenant_id / namespace 列：reconciler 按租户分片
- mutation_kind 命名空间：forget.* | dream.* | admin.*
- shard_key：(tenant_id, mutation_kind) → shard_id，用于 reconciler 多 worker 分片
- status：PENDING | IN_FLIGHT | SUCCESS | FAILED | NEEDS_INTERVENTION | ADVISORY_DELIVERED
- attempts / next_retry_at / last_error：指数退避状态
- signature：mutation 描述本地签名（防 reconciler 写入路径篡改）
```

`status` 状态转移：

```text
PENDING ──fetch──▶ IN_FLIGHT
IN_FLIGHT ──ok──▶ SUCCESS
IN_FLIGHT ──retriable err──▶ PENDING (attempts++, next_retry_at)
IN_FLIGHT ──permanent err──▶ NEEDS_INTERVENTION
IN_FLIGHT ──notify-only target ack──▶ ADVISORY_DELIVERED   [P1-5]
```

`ADVISORY_DELIVERED` 用于不可强制 mutation 的目标系统（如远端 ContextGrant 接收方）：通知已送达即结案，不进 NEEDS_INTERVENTION 告警队列。详见 §13.1。

## 5. ForgetReceipt 双状态 schema

权威 schema 与双签名规则见 [signing-canonical.md §3](./signing-canonical.md)。本节摘要：

```proto
enum ForgetReceiptStatus {
  COMMITTED_LOCALLY = 0;       // 阶段一完成
  GLOBALLY_RECONCILED = 1;     // 阶段二完成
  RECONCILE_FAILED = 2;        // 阶段二永久失败（NEEDS_INTERVENTION）
}

// 完整字段见 signing-canonical.md §3.1：
//   receipt_id / request_id / mode / status
//   committed_locally_at_micros / locally_committed_mutations[] /
//     target_ids[] / cascaded_ids[] / local_signature
//   globally_reconciled_at_micros? / external_evidence[] / reconciled_signature?
//   pending_external_systems[] / reconciliation_deadline_micros
//   intervention_reason? / failed_mutations[]
//   mutation_ledger_id / audit_event_id / fap_receipt_id
//   cancelled_after_local_commit
```

阶段一签名 `local_signature` 在 status = `COMMITTED_LOCALLY` 时立刻生成；
阶段二签名 `reconciled_signature` 在 status = `GLOBALLY_RECONCILED` 时由 reconciler 追加签发；
两段签名各自覆盖独立的 canonical payload，详见 [signing-canonical.md §3.2](./signing-canonical.md)。

## 5.1 SystemReconcileEvidence / FailedMutation

```proto
message SystemReconcileEvidence {
  string system          = 1;         // qdrant_prod / s3_us_east / kafka_audit
  string evidence_token  = 2;         // 外部系统返回的删除凭证（如 S3 delete marker）
  int64  at_micros       = 3;
}

message FailedMutation {
  string ledger_id       = 1;
  string target_system   = 2;
  string last_error      = 3;
  uint32 attempts        = 4;
}
```

## 6. 请求方接口

### 6.1 默认（异步）

```text
ForgetRequest
  → 同步等待阶段一
  → 立即返回 ForgetReceipt { status: COMMITTED_LOCALLY }
  → 请求方可后续查询：
      ForgetStatusQuery(request_id) → ForgetReceipt（最新状态）
```

### 6.2 阻塞等待最终一致

```text
ForgetRequest { wait_for_reconciled: true, timeout_secs: 1800 }
  → 同步等待阶段一
  → 继续等待阶段二
    → 完成 → 返回 ForgetReceipt { status: GLOBALLY_RECONCILED }
    → 超时 → 返回 ForgetReceipt { status: COMMITTED_LOCALLY, pending_external_systems[] }
            （不抛错；请求方仍可后续轮询）
```

### 6.3 订阅

```text
ForgetRequest { notify_on_reconciled: true, callback_url: "..." }
  → 立即返回 COMMITTED_LOCALLY
  → reconciler 完成后向 callback_url POST ForgetReconciledReceipt
```

## 7. 幂等性保证

每个 mutation 必须幂等：

```text
vector_delete       多次调用 mark_deleted(memory_id) 等价
s3_delete           多次 DELETE Object 等价（S3 原生幂等）
kafka_emit          带 idempotency_key 的 ProducerRecord
context_grant_revoke 多次撤销同一 grant 等价
```

`mutation_ledger.payload_json` 中包含 `idempotency_key`，确保重试不会产生副作用。

## 8. 重试策略

```text
指数退避 + 抖动：
  base = 30s
  cap = 3600s
  retry_at = min(base * 2^attempts + jitter(0~base), cap)

最大重试次数：
  short-lived 失败（网络）：MAX_ATTEMPTS = 50（≈ 1 day）
  long-lived 失败（认证）：MAX_ATTEMPTS = 10
  
超过 → status = NEEDS_INTERVENTION
       触发告警
       不再自动重试，等待人工
```

## 9. 故障情景

### 9.1 阶段一失败

```text
SQL 事务回滚 → 不写 mutation_ledger → 不返回 ForgetReceipt
返回错误码 fap-me/forget-failed
```

### 9.2 阶段二部分失败

```text
mutation_ledger 中部分 mutation 状态 = SUCCESS，部分 = FAILED
ForgetReceipt 维持 COMMITTED_LOCALLY
继续重试失败项
请求方查询时看到 pending_external_systems[]
```

### 9.3 阶段二永久失败

```text
mutation status = NEEDS_INTERVENTION
ForgetReceipt 转 RECONCILE_FAILED
告警 + 写审计 forget.intervention_required
合规人员介入：
  - 手动执行该 mutation
  - 或确认外部系统已无目标数据（提供证据 token）
  - 完成后管理员标记 mutation_ledger.status = SUCCESS
```

## 10. 与 FAP-1 finality 的对应

```text
HardForget capability 默认 finality = FINALIZED
FAP-1 InvokeRequest 在 FINALIZED 模式下：
  - 阶段一完成时签发 first-stage Receipt（status=committed_locally）
  - 阶段二完成时签发 second-stage Receipt（status=reconciled）
  - 两份 Receipt 通过 invocation_id 关联

请求方根据 fap_finality 与 receipt status 决定是否信任完成。
```

## 11. Reconciler 实现

### 11.1 Trait

```rust
pub struct ForgetReconciler {
    ledger: Arc<dyn MutationLedgerStore>,
    executors: HashMap<MutationKind, Box<dyn MutationExecutor>>,
    audit_kernel: Arc<AuditKernel>,
    leader: Arc<dyn LeaderElection>,            // §11.2
    shard_assignment: ShardAssignment,           // §11.3
}

#[async_trait]
pub trait MutationExecutor: Send + Sync {
    async fn execute(&self, mutation: &PendingMutation) -> Result<ReconcileEvidence>;
    fn kind(&self) -> MutationKind;
    fn is_idempotent(&self) -> bool;        // 必须为 true
    fn delivery_semantics(&self) -> DeliverySemantics;
}

pub enum DeliverySemantics {
    /// 可强制：失败转 NEEDS_INTERVENTION
    Enforceable,
    /// 不可强制：送达即结案，转 ADVISORY_DELIVERED
    Advisory,
}
```

`Advisory` 适用于受控边界外的目标（远端 Agent / 外部 API），详见 §13.1。

### 11.2 Leader 选举

多 gateway 实例并发跑 reconciler 时必须避免对同一 ledger 行重复调用执行器。允许的实现：

| 后端 | 机制 |
|---|---|
| PostgreSQL | `pg_try_advisory_lock(shard_id)` 持锁的实例为该 shard 的 leader |
| Etcd / Consul | `lease + key prefix /fap-me/reconciler/leader/{shard_id}` |
| Redis | `SET NX PX` + 续约（仅在 PG 不可用时降级使用） |
| 单机 / Edge Lite | 进程内单 leader（无需外部协调） |

leader 失联（lease 失效）后由其他实例接管。已 `IN_FLIGHT` 的 mutation 由超时回收（§11.4）处理。

### 11.3 分片

`mutation_ledger.shard_key` = `hash((tenant_id, mutation_kind)) mod NUM_SHARDS`，默认 `NUM_SHARDS = 32`。

```text
worker_i 处理 shard_key ∈ assigned_shards(worker_i)
分片分配通过 leader 协调：
  - 静态：按 worker_id 取模（适合稳定实例数）
  - 动态：leader 根据健康度重新分配（适合自动伸缩）
```

工程上保证：

```text
1. 同一 ledger 行同一时刻只有一个 worker 在 IN_FLIGHT
2. shard 重新分配时，旧 owner 必须先把所有 IN_FLIGHT 标回 PENDING
3. UNIQUE(target_system, idempotency_key) 兜底：即便 leader 切换瞬间双发，外部系统也只生效一次
```

### 11.4 超时回收

`IN_FLIGHT` 状态超过 `IN_FLIGHT_TIMEOUT`（默认 5 分钟）的 mutation 由 reconciler 自动回收：

```text
SELECT * FROM mutation_ledger
 WHERE status = 'IN_FLIGHT' AND started_at < now() - 300s
   AND shard_key ∈ my_shards
FOR UPDATE SKIP LOCKED
  → 标回 PENDING，attempts++
  → 若 attempts > MAX_ATTEMPTS：转 NEEDS_INTERVENTION
```

worker 崩溃后无人续期 → 5 分钟内自动回收并重派。

### 11.5 主循环

```rust
impl ForgetReconciler {
    pub async fn run_loop(&self) {
        loop {
            if !self.leader.am_leader_for_any_shard().await? {
                tokio::time::sleep(LEADER_POLL).await;
                continue;
            }
            let shards = self.leader.my_shards().await?;
            self.reclaim_orphaned(&shards).await?;
            let pending = self.ledger.fetch_due(&shards, MAX_BATCH).await?;
            for m in pending {
                self.try_execute(m).await;
            }
            tokio::time::sleep(POLL_INTERVAL).await;
        }
    }

    async fn try_execute(&self, m: PendingMutation) {
        self.ledger.set_in_flight(m.ledger_id).await?;
        let executor = self.executors.get(&m.mutation_kind)
            .expect("missing executor");
        match executor.execute(&m).await {
            Ok(evidence) => match executor.delivery_semantics() {
                DeliverySemantics::Enforceable => {
                    self.ledger.set_success(m.ledger_id, evidence).await?;
                    self.maybe_finalize_receipt(m.receipt_id).await?;
                }
                DeliverySemantics::Advisory => {
                    self.ledger.set_advisory_delivered(m.ledger_id, evidence).await?;
                    self.maybe_finalize_receipt(m.receipt_id).await?;
                }
            },
            Err(e) if e.is_retriable() => {
                self.ledger.bump_attempts(m.ledger_id, &e).await?;
            }
            Err(e) => {
                self.ledger.set_intervention(m.ledger_id, &e).await?;
                self.audit_kernel.append(intervention_event(m, e)).await?;
            }
        }
    }

    async fn maybe_finalize_receipt(&self, receipt_id: ReceiptId) -> Result<()> {
        // 所有 mutation 状态 ∈ {SUCCESS, ADVISORY_DELIVERED} 时进入阶段二
        if self.ledger.all_terminal(receipt_id).await? {
            let evidence = self.ledger.gather_evidence(receipt_id).await?;
            let receipt = self.sign_reconciled_receipt(receipt_id, evidence)?;
            self.audit_kernel.append(reconciled_event(receipt)).await?;
            self.notify_subscribers(receipt).await?;
        }
        Ok(())
    }
}
```

## 12. 监控指标

```text
mutation_ledger_pending_total{system}          待处理项数
mutation_ledger_in_flight_total
mutation_ledger_intervention_total{system}     人工介入项数（红色告警）
forget_reconcile_duration_seconds{system}      达成最终一致的耗时
forget_receipt_finalized_total                 签发 reconciled receipt 数
forget_max_pending_age_seconds                 最老 pending 项的年龄

告警建议：
  intervention_total > 0                     红色（必须人工处理）
  pending_age > 24h（默认 GDPR 窗口预警）   黄色
  pending_age > 72h                          红色（合规风险）
```

## 13. 不仅限于 HardForget

同一 saga 模式适用于所有跨系统 mutation：

```text
DreamProposal.apply 涉及外部 vector index 重建
admin.rebuild 涉及外部 Qdrant collection 替换
context_grant 跨租户撤销通知接收方

均通过 mutation_ledger 统一管理，
但 schema 中的 mutation_kind 命名空间区分
（forget.* / dream.* / admin.*）
```

`mutation_ledger` 统一管理所有需要最终一致的 mutation。

### 13.1 ADVISORY_DELIVERED：不可强制 mutation 的边界

`mutation_ledger.status = ADVISORY_DELIVERED` 用于受控边界外的目标系统：

```text
适用场景：
  forget.context_grant_revoke      远端 Agent / 已 redeem 的接收方
  forget.handoff_advisory          已分发 HandoffPacket 的接收方
  audit.fan_out_advisory           不在合规归档范围的可选 sink

行为：
  executor 把通知送达对端（API call / event publish）并收到 ack
  → status = ADVISORY_DELIVERED
  → 不再重试
  → 不进 NEEDS_INTERVENTION 告警队列
  → 进入 receipt 阶段二的 external_evidence[]，但 evidence_token 标 "advisory"

请求方理解：
  ADVISORY_DELIVERED 表示通知送达，不表示对端真删
  对端是否真删由对端审计承担
  源侧诚实声明撤销已通知，不声称数据自动消失（与 12-context-sharing §1 一致）

转 NEEDS_INTERVENTION 的边界：
  通知送达失败（无法到达对端）→ 重试 → 永久失败 → NEEDS_INTERVENTION（注意：是"通知送达"失败）
  对端拒绝 ack（语义错误）→ 直接 NEEDS_INTERVENTION
```

ADVISORY 状态与 SUCCESS 状态在 receipt 终结判定中等价（都属于 terminal），合规 SLA 仅以"送达"为衡量。

## 14. 合规含义

```text
GDPR Article 17：
  数据主体行使遗忘权后，控制者必须在"无不当延迟"内删除
  通常解释为 ≤ 30 天
  允许跨系统异步达成

CCPA：
  类似要求；具体窗口 ≤ 45 天

本文档保证：
  ForgetReceipt.committed_locally_at 在 ≤ 1 秒内签发（用户体验）
  ForgetReceipt.globally_reconciled_at 通常 ≤ 30 分钟
                                       极端情况 ≤ 30 天（合规上限）
  超过 24h 触发预警；超过 72h 红色告警
```

## 15. 与最终版的差距

```text
最终版                                  本文档
-----                                   -----
ForgetReceipt 暗示"已完成"              双状态：COMMITTED_LOCALLY / GLOBALLY_RECONCILED
单事务覆盖所有外部系统（不可能）        本地事务 + 外部 saga + reconciler
失败时无补偿语义                         NEEDS_INTERVENTION + 人工介入流程
不允许请求方等待最终一致                wait_for_reconciled 选项
不允许跨系统幂等保证                    mutation_ledger 幂等性 + 重试 + 证据 token
```
