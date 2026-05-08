# outbox-reconciliation.md

> HardForget 与跨系统记忆 mutation 的最终一致性方案。基于 transactional outbox + saga + reconciliation。

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

```sql
CREATE TABLE mutation_ledger (
  ledger_id TEXT PRIMARY KEY,
  request_id TEXT NOT NULL,                  -- ForgetRequest 幂等键
  receipt_id TEXT NOT NULL,                  -- ForgetReceipt
  mutation_kind TEXT NOT NULL,               -- vector_delete | s3_delete | kafka_emit | ...
  target_system TEXT NOT NULL,               -- qdrant_prod | s3_us_east | kafka_audit | ...
  payload_json TEXT NOT NULL,                -- 幂等执行所需参数
  status TEXT NOT NULL,                      -- PENDING | IN_FLIGHT | SUCCESS | FAILED | NEEDS_INTERVENTION
  attempts INTEGER NOT NULL DEFAULT 0,
  last_error TEXT,
  scheduled_at INTEGER NOT NULL,
  started_at INTEGER,
  completed_at INTEGER,
  next_retry_at INTEGER,
  signature TEXT NOT NULL                    -- mutation 描述的本地签名（防 tampering）
);
CREATE INDEX idx_mutation_ledger_status ON mutation_ledger(status, next_retry_at);
CREATE INDEX idx_mutation_ledger_receipt ON mutation_ledger(receipt_id);
```

## 5. ForgetReceipt 双状态 schema

```proto
enum ForgetReceiptStatus {
  COMMITTED_LOCALLY = 0;       // 阶段一完成
  GLOBALLY_RECONCILED = 1;     // 阶段二完成
  RECONCILE_FAILED = 2;        // 阶段二永久失败（NEEDS_INTERVENTION）
}

message ForgetReceipt {
  string receipt_id = 1;
  string request_id = 2;
  ForgetMode mode = 3;
  ForgetReceiptStatus status = 4;

  // 阶段一信息（必填）
  google.protobuf.Timestamp committed_locally_at = 5;
  repeated string locally_committed_mutations = 6;
  bytes local_signature = 7;

  // 阶段二信息（reconciled 时填）
  optional google.protobuf.Timestamp globally_reconciled_at = 8;
  repeated SystemReconcileEvidence external_evidence = 9;
  optional bytes reconciled_signature = 10;

  // 进行中信息
  repeated string pending_external_systems = 11;
  optional google.protobuf.Timestamp reconciliation_deadline = 12;

  // 失败时
  optional string intervention_reason = 13;
  repeated FailedMutation failed_mutations = 14;
}

message SystemReconcileEvidence {
  string system = 1;             // qdrant_prod / s3_us_east / kafka_audit
  string evidence_token = 2;     // 外部系统返回的删除凭证（如 S3 delete marker）
  google.protobuf.Timestamp at = 3;
}

message FailedMutation {
  string ledger_id = 1;
  string target_system = 2;
  string last_error = 3;
  uint32 attempts = 4;
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

```rust
pub struct ForgetReconciler {
    ledger: Arc<dyn MutationLedgerStore>,
    executors: HashMap<MutationKind, Box<dyn MutationExecutor>>,
    audit_kernel: Arc<AuditKernel>,
}

#[async_trait]
pub trait MutationExecutor: Send + Sync {
    async fn execute(&self, mutation: &PendingMutation) -> Result<ReconcileEvidence>;
    fn kind(&self) -> MutationKind;
    fn is_idempotent(&self) -> bool;        // 必须为 true
}

impl ForgetReconciler {
    pub async fn run_loop(&self) {
        loop {
            let pending = self.ledger.fetch_due(MAX_BATCH).await?;
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
            Ok(evidence) => {
                self.ledger.set_success(m.ledger_id, evidence).await?;
                self.maybe_finalize_receipt(m.receipt_id).await?;
            }
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
        if self.ledger.all_success(receipt_id).await? {
            // 所有 mutation 成功 → 签发第二阶段 Receipt
            let evidence = self.ledger.gather_evidence(receipt_id).await?;
            let receipt = self.sign_reconciled_receipt(receipt_id, evidence)?;
            self.audit_kernel.append(reconciled_event(receipt)).await?;
            // 通知订阅者
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
