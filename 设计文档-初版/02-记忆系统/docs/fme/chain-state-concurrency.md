# chain-state-concurrency.md

> 审计哈希链头 `chain_state` 在并发写入与多种隔离模式下的一致性规范。本文档负责回答：
>
> 1. 谁可以更新 `chain_state.current_tip`？
> 2. 多并发写入如何避免链分叉？
> 3. SchemaPerTenant / DbPerTenant 模式下，跨 schema 的周期完整性验证如何调度？

## 1. 设计原则

```text
1. 每条 (tenant_id, namespace) 的链头有且只有 AuditKernel 可以更新
2. 更新使用 CAS（compare-and-swap）：base_revision 必须等于当前 revision
3. CAS 失败 → 重新计算 prev_event_hash 后重试，最多 N 次后转告警
4. 不允许全局单 Mutex；服务端必须支持租户级并发
5. SchemaPerTenant / DbPerTenant 下保留一份"chain_state 注册中心"，使周期 verify 任务可跨 schema 调度
6. 任何 chain head 异常（断裂 / CAS 永久失败 / 注册中心丢失）= 红色告警 + 拒绝该 (tenant, namespace) 后续写入
```

## 2. chain_state 表

权威 DDL 见 [13-storage-design.md §3](./13-storage-design.md)，要点：

```sql
CREATE TABLE chain_state (
  tenant_id TEXT NOT NULL,
  namespace TEXT NOT NULL,
  current_tip BLOB NOT NULL,
  revision INTEGER NOT NULL,                 -- 单调递增
  last_verified_at INTEGER,
  last_verified_event TEXT,
  PRIMARY KEY (tenant_id, namespace)
);
```

`revision` 是 CAS 的版本号；每次成功 append 后 `revision += 1`。

## 3. CAS 更新算法

```rust
impl AuditKernel {
    pub fn append(&self, input: AuditEventInput) -> Result<AuditReceipt, ChainError> {
        let key = (input.tenant_id, input.namespace);
        for attempt in 0..MAX_CAS_RETRIES {                  // 默认 5
            // 1. 读 chain head
            let st = self.chain_store.get(&key)?;            // (current_tip, revision)
            // 2. 计算 event_hash
            let event = build_event_with_prev(&input, st.current_tip);
            let event_hash = sha256_canonical(&event);
            // 3. CAS：only if revision unchanged
            match self.chain_store.cas_update(
                &key,
                expected_revision: st.revision,
                new_tip: event_hash,
                new_revision: st.revision + 1,
                event_to_append: event.clone(),
            ) {
                Ok(()) => {
                    // append 成功；通知 sinks
                    self.fanout(event)?;
                    return Ok(AuditReceipt::from(event));
                }
                Err(ChainError::CasConflict) => continue,    // 重试
                Err(e) => return Err(e),
            }
        }
        // CAS 永久冲突 → 告警 + 锁定该 (tenant, namespace) 的写入
        self.lock_tenant_chain(&key, "cas-permanent-conflict");
        Err(ChainError::CasExhausted)
    }
}
```

`cas_update` 的具体后端实现：

| 后端 | 实现 |
|---|---|
| SQLite WAL | `UPDATE chain_state SET current_tip=?, revision=? WHERE tenant_id=? AND namespace=? AND revision=?` 在 WAL 事务中插入 audit_event + 更新 chain_state |
| PostgreSQL | 同上，使用 SERIALIZABLE 或 READ COMMITTED + 显式行锁（`SELECT ... FOR UPDATE`） |
| 多节点 PG | 同上；CAS 由 PG 行锁与 MVCC 自然实现 |

服务端模式必须保证：

```text
audit_event INSERT 与 chain_state UPDATE 在同一本地事务内提交
两者要么全部生效要么全部回滚
否则会出现 chain_state 已更新但 event 未持久化的"虚链"
```

## 4. 隔离模式与 chain_state 位置

`multi-tenant.md §3` 定义三种隔离模式。`chain_state` 表的位置随模式变化：

| 隔离模式 | chain_state 位置 |
|---|---|
| `LogicalRow` | 全局 chain_state 表，按 (tenant_id, namespace) 分行 |
| `SchemaPerTenant` | 每个 tenant schema 有自己的 chain_state 表（与该 tenant 的 audit_event 同 schema） |
| `DbPerTenant` | 每个 tenant DB 有自己的 chain_state 表 |

## 5. 全局 chain_state 注册中心

`SchemaPerTenant` / `DbPerTenant` 模式下，单纯 `SELECT * FROM chain_state` 无法枚举全部 chain head。必须存在一个独立的"chain_state 注册中心"：

```sql
-- 全局 metadata DB / 控制面 DB
CREATE TABLE chain_state_registry (
  tenant_id TEXT NOT NULL,
  namespace TEXT NOT NULL,
  isolation_mode TEXT NOT NULL,           -- LogicalRow | SchemaPerTenant | DbPerTenant
  storage_locator TEXT NOT NULL,          -- "schema:tenant_acme" | "db:tenant_acme_pg" | "shared"
  registered_at INTEGER NOT NULL,
  last_seen_at INTEGER NOT NULL,
  PRIMARY KEY (tenant_id, namespace)
);
```

注册时机：

```text
- 首次 audit_event.append 时，AuditKernel 检查并注册
- tenant 创建 / namespace 创建时由 admin API 主动写入
- 周期心跳更新 last_seen_at
```

注册中心由 control plane 维护，不参与热路径写入，仅用于：

```text
1. ChainIntegrityScheduler 枚举所有 chain head
2. 容灾恢复时定位每个 chain 的存储位置
3. 监控仪表板按 tenant 聚合
```

## 6. 周期完整性验证调度

```rust
pub struct ChainIntegrityScheduler {
    registry: Arc<ChainStateRegistry>,
    audit_kernel: Arc<AuditKernel>,
    schedule: cron::Schedule,            // 默认 "30 0 * * *"
    parallelism: usize,                   // 默认 8
}

impl ChainIntegrityScheduler {
    pub async fn run_daily(&self) -> Result<()> {
        let entries = self.registry.list_active().await?;
        // 按 storage_locator 分组，避免对同一 DB 过载
        let by_locator = group_by(entries, |e| &e.storage_locator);
        let semaphore = Arc::new(Semaphore::new(self.parallelism));

        for (locator, group) in by_locator {
            let kernel = self.audit_kernel.clone();
            tokio::spawn(async move {
                let _permit = semaphore.acquire().await?;
                for entry in group {
                    match kernel.verify_chain(
                        TenantRef::new(&entry.tenant_id, &entry.namespace),
                        from: last_verified_event(&entry),
                        to: latest_event(&entry),
                    ).await {
                        Ok(_) => kernel.touch_verified(&entry).await?,
                        Err(broken) => {
                            kernel.append_tampering_alert(&entry, broken).await?;
                            kernel.lock_tenant_chain(&entry, "chain-broken");
                        }
                    }
                }
            });
        }
        Ok(())
    }
}
```

容量约束：单次完整性验证不应阻塞业务路径；调度器必须是 Cerebellum 后台任务（不阻塞主路径），并在 `last_verified_at` 更新失败时告警。

## 7. 失败处理

| 失败 | 行为 |
|---|---|
| CAS 冲突（暂时） | 重试 ≤ MAX_CAS_RETRIES |
| CAS 永久冲突 | 锁定 chain head 写入，告警 `fap-me/chain-cas-exhausted` |
| 链断裂（哈希不匹配） | 写入 TamperingAlert + 锁 + 告警 `fap-me/tampering-detected` |
| 注册中心丢失某 tenant | 启动时拒绝该 tenant 写入；告警 `fap-me/chain-registry-missing` |
| 注册中心与 chain head 不一致 | 取 chain head 实际值，registry 自动同步并告警 |
| 主从切换后 revision 回退 | 拒绝写入；要求 admin 介入重建 chain head |

`fap-me/chain-cas-exhausted` 与 `fap-me/chain-registry-missing` 应加入 [problem-catalog.md](./problem-catalog.md)。

## 8. 边缘 / Edge Lite

```text
单租户 / 单 namespace：单进程内单 chain，不需要外部 CAS
退化为 Mutex<ChainState>（仅边缘）
注册中心退化为本地 JSON
```

边缘模式不要求注册中心高可用；但启动时必须能从本地 metadata 中枚举所有 chain，否则拒绝启动。

## 9. 测试 / 验收要求

Conformance 矩阵中必须覆盖：

```text
1. 1 个 tenant、单 namespace、串行写入 N 次 → revision 单调
2. 1 个 tenant、单 namespace、并发 ≥ 32 路写入 → 全部最终持久化、链完整、revision 连续
3. 模拟 CAS 永久冲突 → 锁定写入并告警
4. SchemaPerTenant 下注册中心枚举 ≥ 100 tenant → 周期 verify 全部完成
5. 注入"chain head 与 audit_event 不一致" → TamperingAlert 必须写入
6. 主从切换后 chain head revision 回退 → 拒绝新写入
```

至少 ≥ 8 个 Conformance 用例（与 [17-testing-conformance.md §5](./17-testing-conformance.md) 中"审计链完整性 ≥ 8"对齐）。

## 10. 与最终版的差距

```text
最终版                                  本文档
-----                                   -----
chain_state 并发模型未规格化            CAS + base_revision + 重试预算
SchemaPerTenant 下完整性 verify 未定义   chain_state_registry 注册中心 + ChainIntegrityScheduler
"应使用 CAS 或分片"措辞                  明确实现：SQLite WAL / PG 行锁；可降级为 Mutex（仅 Edge Lite）
```
