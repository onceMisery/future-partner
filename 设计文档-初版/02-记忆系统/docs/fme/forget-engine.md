# forget-engine.md

> 遗忘引擎与跨系统最终一致。本地强一致 + 外部 saga + 双状态 ForgetReceipt。详见 [outbox-reconciliation.md](./outbox-reconciliation.md)。

## 1. 设计原则

```text
ForgetEngine 只规划，不直接删除（mutations 由 Kernel 执行）
本地系统：单事务原子完成
跨系统：outbox + saga + reconciler，最终一致
ForgetReceipt 双状态：COMMITTED_LOCALLY → GLOBALLY_RECONCILED
请求方可选阻塞等待或异步轮询
SBU 强制遗忘不可被插件跳过
审计链不可删除（审计本身豁免遗忘）
```

## 2. 遗忘模式

| 模式 | 含义 | 默认 finality | risk_level |
|---|---|---|---|
| `SOFT` | tombstone + 不再返回（本地强一致即可完成） | VERIFIED | high |
| `HARD` | 级联清除 raw / embedding / snapshot / grant，含跨系统 | FINALIZED | critical |

## 3. 双阶段流程

### 3.1 阶段一：本地强一致（同步完成）

```text
ForgetRequest
  → FAP-1 Mandate 验证（capability = memory.forget.hard）
  → PolicyKernel.authorize
  → Find Memory Lineage（追溯所有下游）
  → ForgetEngine.plan（插件规划）
  → ForgetEngine.local_mutations（输出本地 mutation list）
  → ForgetEngine.external_mutations（输出跨系统 mutation list）
  → Kernel 单事务执行所有本地 mutation：
      revoke ContextGrant（本地）
      delete L0 snapshots（本地）
      tombstone L1 / L2（本地行级）
      [HARD] purge 本地 raw content
      mark vector deleted（本地索引文件）
      update tag graph 引用（本地）
      mutation_ledger insert（pending external mutations）
      audit_event insert（status = COMMITTED_LOCALLY）
  → COMMIT
  → 签发 ForgetReceipt { status: COMMITTED_LOCALLY, pending_external_systems[] }
```

### 3.2 阶段二：跨系统最终一致（异步达成）

详见 [outbox-reconciliation.md](./outbox-reconciliation.md)。

```text
Cerebellum 的 ForgetReconciler 异步处理 mutation_ledger：
  - vector_index.mark_deleted（远程 Qdrant）
  - object_store.delete（远程 S3）
  - audit_sink.fan_out（远程 Kafka / S3 归档）
  - 通知接收方撤销缓存（如对端已收 ContextGrant）

完成所有外部 mutation 后：
  - 签发 ForgetReconciledReceipt
  - 追加 audit_event（operation = forget.reconciled，绑定 reconciled 证据）
  - 通知请求方（若订阅）
```

## 4. ForgetEngine trait

```rust
pub trait ForgetEngine: Plugin {
    fn plan(
        &self,
        request: &ForgetRequest,
        lineage: &MemoryLineage,
    ) -> Result<ForgetPlan>;

    /// 本地 mutation：在单事务内执行
    fn local_mutations(&self, plan: &ForgetPlan) -> Vec<MemoryMutation>;

    /// 跨系统 mutation：写入 mutation_ledger 由 reconciler 异步达成
    fn external_mutations(&self, plan: &ForgetPlan) -> Vec<ExternalMutation>;
}

pub struct ExternalMutation {
    pub kind: ExternalMutationKind,         // VECTOR_DELETE | S3_DELETE | KAFKA_EMIT | ...
    pub target_system: String,
    pub idempotency_key: String,
    pub payload: serde_json::Value,
}
```

## 5. 级联清单（本地 vs 外部）

### 5.1 本地（单事务）

```text
1. memory_unit.deleted_at
2. memory_unit_security 删除
3. memory_lineage 删除
4. memory_tag 引用减少（Tag 行不立刻删，等 reconciler 判断孤立）
5. context_snapshot 中含此 memory_id 的快照失效
6. context_grant 中绑定此 snapshot 的 grant 撤销（本地表）
7. 本地 HNSW 索引文件 mark_deleted
8. 本地 CAS 对象删除（content_ref 指向时）
9. mutation_ledger 写入 pending external mutations
10. audit_event 写入 (status = COMMITTED_LOCALLY)
```

### 5.2 外部（saga）

```text
- 远端 vector index（Qdrant collection）              [Enforceable]
- 远端 object store（S3 bucket）                      [Enforceable]
- 跨节点 audit sink（Kafka audit topic）              [Enforceable]
- 远端审计归档（S3 / OpenTelemetry collector）         [Enforceable]
- 接收方缓存通知（已 redeem 过的 ContextGrant 持有方）  [Advisory]
- HandoffPacket 接收方：通知撤销建议                   [Advisory]
```

每条外部 mutation 进 mutation_ledger，由 reconciler 重试至 SUCCESS / ADVISORY_DELIVERED 或 NEEDS_INTERVENTION。`Advisory` 标记的 mutation 通知送达即结案，详见 [outbox-reconciliation.md §13.1](./outbox-reconciliation.md)。

## 6. ForgetRequest 选项

```proto
message ForgetInput {
  string tenant_id = 1;
  ForgetMode mode = 2;
  ForgetTarget target = 3;
  string reason = 4;

  // saga 行为
  bool wait_for_reconciled = 5;
  optional google.protobuf.Duration wait_timeout = 6;
  optional string callback_url = 7;
}
```

| 行为 | 描述 |
|---|---|
| 默认（异步） | 立刻返回 COMMITTED_LOCALLY |
| `wait_for_reconciled = true` | 阻塞至 GLOBALLY_RECONCILED 或超时 |
| `callback_url` | 异步达成时回调 |
| 等待中取消 | 阶段一已签 `local_signature`，receipt 含 `cancelled_after_local_commit = true`；ledger 中的 mutation 仍由 reconciler 异步达成；阶段二完成时仍签 `reconciled_signature` |

## 7. ForgetReceipt（双状态）

权威 schema 与双签名规则见 [signing-canonical.md §3](./signing-canonical.md)。本节摘要：

```proto
enum ForgetReceiptStatus {
  COMMITTED_LOCALLY = 0;
  GLOBALLY_RECONCILED = 1;
  RECONCILE_FAILED = 2;
}

// 完整字段见 signing-canonical.md §3.1
// 双签名：
//   local_signature        阶段一签发，覆盖 (receipt_id, mode, target_ids[],
//                                          cascaded_ids[], locally_committed_mutations[],
//                                          committed_locally_at_micros, ...)
//   reconciled_signature   阶段二签发，覆盖 (receipt_id, GLOBALLY_RECONCILED,
//                                           globally_reconciled_at_micros,
//                                           external_evidence[], failed_mutations[])
```

详见 [outbox-reconciliation.md §5](./outbox-reconciliation.md)。

## 8. SBU 强制遗忘

```text
ForgetTarget::SBU 触发：
  - 扫描 memory_unit_security WHERE label IN ('sbu', 'secret')
  - 对每条 → HardForget
  - 强制级联所有 lineage 子节点
  - 强制撤销所有 ContextGrant 中包含这些 memory 的 grant
  - 不可被插件跳过（Kernel 直接驱动）
  - 全过程写审计

mandate.capabilities 必须含 memory.forget.hard
mandate.constraints[type="memory.purpose"] 必须为 compliance_audit 或 memory_maintenance
```

SBU 遗忘默认 `wait_for_reconciled = true` + 30 分钟超时（合规要求体验）。

## 9. 异步索引清理

```text
HNSW 等向量索引的实际"压实"是高代价操作：
  - 阶段一：mark_deleted（O(1) 软删除，本地事务）
  - 阶段二：reconciler 在外部远端 vector index 上 mark_deleted
  - 阶段三：Cerebellum 周期 compaction（每周 / deleted 占比 > 20%）
           调用 fork_version 重建并切换索引

请求方可见的"已删"：
  阶段一完成（检索时被过滤）

请求方可见的"物理清除"：
  阶段三完成（视情况可能滞后数天）
```

## 10. Forget 风险与限制

```text
Tenant 级遗忘需 elevated mandate（compliance_audit purpose）
不可遗忘 audit_event（审计链不可删）
不可遗忘 forget_receipt（凭证留痕）
不可遗忘 mandate（凭证留痕）
不可遗忘 chain_state（链状态留痕）

可遗忘但需谨慎：
  context_grant     连同对应 snapshot 一起处理
  handoff_packet    保留索引但内容 redacted；通知对端
```

## 11. 用户行使遗忘权（GDPR/CCPA）

```text
用户提交：DataSubjectRequest
  → 转换为 ForgetRequest {
      target: Lineage(rooted at user-owned memory),
      mode: HARD,
      wait_for_reconciled: false,        # 立即给用户响应
      callback_url: "..."                 # 30 分钟内回调达成
    }
  → mandate.constraints[type="memory.purpose"] = "compliance_audit"
  → 标准 forget 流程
  → 立即返回 ForgetReceipt（COMMITTED_LOCALLY）
  → ≤ 30 分钟内（典型）回调 ForgetReconciledReceipt
  → ≤ 30 天（合规上限）必须达成 GLOBALLY_RECONCILED 或 RECONCILE_FAILED
```

## 12. 与 DreamWorker 的边界

```text
DreamWorker          建议合并/压缩，可改写记忆但不删除原始数据
ForgetEngine         真正物理删除，含跨系统 saga

互不干扰，但都受 Kernel 严格管控。
```

## 13. 失败处理

| 阶段 | 失败 | 行为 |
|---|---|---|
| 阶段一 | 本地事务回滚 | 写审计 forget.failed，返回错误 fap-me/forget-failed |
| 阶段二 | 单 mutation 失败但可重试 | 计数 attempts，按指数退避重试 |
| 阶段二 | 单 mutation 永久失败 | 转 NEEDS_INTERVENTION，告警，等待人工 |
| 阶段二 | 超过 reconciliation_deadline | 状态转 RECONCILE_FAILED，红色告警（合规风险） |

详见 [outbox-reconciliation.md](./outbox-reconciliation.md) §11。

## 14. 监控指标

```text
forget_total{mode}                            SOFT vs HARD 计数
forget_local_duration_seconds                 阶段一耗时（同步路径关键）
forget_reconcile_duration_seconds{system}     阶段二各系统耗时
forget_pending_external_total                 等待 reconcile 的项数
forget_intervention_total                     人工介入项数（红色）
forget_sbu_total                              SBU 强制遗忘
forget_tenant_purge_total                     tenant 级遗忘（合规）
```

## 15. 与最终版的差距

```text
最终版                                  本文档
-----                                   -----
单事务覆盖 raw + embedding + snapshot   本地事务 + 外部 saga
+ grant + vector + object + audit       双状态 ForgetReceipt
                                        mutation_ledger 幂等 ledger
ForgetReceipt 单状态（暗示已完成）       COMMITTED_LOCALLY / GLOBALLY_RECONCILED / RECONCILE_FAILED
失败时无补偿语义                         NEEDS_INTERVENTION + 人工介入流程
不允许请求方等待最终一致                wait_for_reconciled / callback_url
finality 未与 risk 绑定                  HARD 默认 FINALIZED
```
