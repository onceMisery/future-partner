# retrieval-fallback.md

> 检索降级链与背压机制。补充最终版"高级检索超时可降级"中缺失的具体阈值与触发逻辑。

## 1. 设计目标

```text
任何检索模式超时后必须有降级路径，不返回空结果
霰弹枪并发不可无限增长
LIF 扩散 / Tide 计算 / Geodesic 重排 各自有独立预算
基础检索（basic）永不降级，是最后兜底
```

## 2. 降级链

```text
tagmemo_geodesic
    │ LIF 超时(80ms) → 仅向量 + geodesic
    │ 总超时(300ms) → tagmemo
    │
    ▼
tagmemo
    │ 总超时(300ms) → hybrid
    │ Tag 拓扑不可用 → hybrid
    │
    ▼
hybrid
    │ 总超时(300ms) → basic
    │ BM25 不可用 → basic
    │
    ▼
basic
    │ 永不降级，保证可用性
```

`tide` / `dream` / `sbu_safe` 的降级遵循同样原则：

```text
tide → hybrid → basic
dream → tagmemo → hybrid → basic
sbu_safe → 不允许降级到无脱敏模式（直接拒绝并告警）
```

## 3. 超时阈值

```rust
pub struct RetrievalPipeline {
    pub timeout_ms: u64,                    // 默认 300ms（总）
    pub shotgun_max_concurrent: usize,      // 默认 8
    pub shotgun_timeout_ms: u64,            // 默认 100ms（单次）
    pub lif_timeout_ms: u64,                // 默认 80ms
    pub tide_timeout_ms: u64,               // 默认 50ms
    pub geodesic_timeout_ms: u64,           // 默认 20ms
    pub reranker_timeout_ms: u64,           // 默认 80ms
    pub candidate_pool_max: usize,          // 默认 500
}
```

| 子步骤 | 默认预算 | 超时行为 |
|---|---:|---|
| Tide.decompose | 50ms | 跳过 Tide，使用原 query |
| LIF expand | 80ms | 跳过 LIF，候选仅来自 HNSW |
| Geodesic rerank | 20ms | 跳过重排，使用 KNN 排序 |
| Reranker | 80ms | 跳过精排，使用 final_score |
| 单次霰弹枪检索 | 100ms | 该次失败，其他保留 |
| 总检索 | 300ms | 降级到下一档 |

## 4. 降级判定

```rust
pub async fn retrieve(&self, req: &RetrieveRequest) -> RetrieveResult {
    let start = Instant::now();
    match req.mode {
        RetrievalMode::TagmemoGeodesic => {
            match timeout(self.timeout_ms, self.tagmemo_geodesic(req)).await {
                Ok(Ok(r)) => return r,
                _ => {
                    audit_fallback("tagmemo_geodesic", "tagmemo");
                    self.tagmemo(req).await
                }
            }
        }
        RetrievalMode::Tagmemo => {
            match timeout(self.timeout_ms, self.tagmemo(req)).await {
                Ok(Ok(r)) => return r,
                _ => {
                    audit_fallback("tagmemo", "hybrid");
                    self.hybrid(req).await
                }
            }
        }
        RetrievalMode::Hybrid => { ... }
        RetrievalMode::Basic => {
            // 永不降级
            self.basic(req).await
        }
        RetrievalMode::SbuSafe => {
            // 不可降级到无脱敏
            match timeout(self.timeout_ms, self.sbu_safe(req)).await {
                Ok(Ok(r)) => return r,
                _ => {
                    audit_fallback("sbu_safe", "FAIL");
                    Err(RetrievalError::Timeout)
                }
            }
        }
        _ => self.execute_direct(req).await,
    }
}
```

## 5. 背压机制

### 5.1 候选池上限

```text
候选池大小 = Σ 各路召回的 Top-K
  L0 search:        Top-20
  L1 hybrid recall: Top-100
  L2 HNSW recall:   Top-100
  Synapse expand:   Top-50

总和 ≤ candidate_pool_max（默认 500）
超限时按召回路径优先级截断（HNSW > BM25 > Spike）
```

### 5.2 霰弹枪并发上限

霰弹枪 = 多次并发向量检索（基于历史 segment）：

```text
并发上限 = shotgun_max_concurrent（默认 8）

实现：
  let semaphore = Arc::new(Semaphore::new(shotgun_max_concurrent));
  for segment in history_segments {
    let permit = semaphore.acquire().await?;
    spawn_with_permit(permit, retrieve(segment));
  }
```

### 5.3 拓扑爆炸保护

LIF 扩散过程中：

```text
若 active_nodes 增长率 > GROWTH_RATE_THRESHOLD（默认 5x per hop）：
  立即停止扩散
  保留当前已激活节点
  写审计：lif.topology_explosion_guard_triggered
```

## 6. 降级审计

每次降级必须写审计：

```text
operation = "memory.retrieve.fallback"
metadata = {
  requested_mode: "tagmemo_geodesic",
  actual_mode: "tagmemo",
  reason: "timeout" | "lif_unavailable" | "tag_graph_unavailable",
  duration_ms: 320,
}
```

便于：
- 排查性能问题
- 评估降级率
- 调整默认阈值

## 7. 健康度反馈

降级率（fallback_rate）作为系统健康度指标：

```text
fallback_rate = fallback_count / total_retrieve_count

健康阈值：
  < 1%       绿色（正常）
  1% ~ 5%    黄色（可调优）
  > 5%       红色（需排查）
```

## 8. 部分结果返回

```text
若总超时但有部分召回完成：
  返回已完成的部分（带 partial_result = true 标记）
  剩余结果不再等待
  写审计：retrieval.partial_result
```

部分结果不影响 final_score 排序的正确性（只是少了某些路径的贡献）。

## 9. SbuSafe 模式特殊处理

```text
sbu_safe 模式不可降级到 hybrid / basic
原因：脱敏处理是该模式的核心语义

超时处理：
  返回错误（fap-me/timeout）
  写审计 retrieval.sbu_safe_timeout
  调用方需自行重试或改用其他 purpose
```

## 10. 缓存与预热

```text
L1 cache（请求级）：
  query embedding 缓存（idempotency_key 重放）

L2 cache（短期）：
  Tag 质心向量（用于 Tide）
  hot tag graph（用于 LIF）
  TTL：5min
  失效触发：tag_edge 写入量超阈值

预热：
  服务启动时加载 Top-N 热边
  embedding 模型常驻
```

## 11. 默认参数表

| 参数 | 默认值 | 备注 |
|---|---:|---|
| `timeout_ms` | 300 | 总检索超时 |
| `shotgun_max_concurrent` | 8 | 霰弹枪最大并发 |
| `shotgun_timeout_ms` | 100 | 单次霰弹枪超时 |
| `lif_timeout_ms` | 80 | LIF 扩散超时 |
| `tide_timeout_ms` | 50 | Tide 分解超时 |
| `geodesic_timeout_ms` | 20 | 测地线重排超时 |
| `reranker_timeout_ms` | 80 | 精排超时 |
| `candidate_pool_max` | 500 | 候选池上限 |
| `lif_growth_rate_threshold` | 5.0 | LIF 拓扑爆炸阈值 |

## 12. 失败模式与监控

```text
监控指标：
  retrieval_fallback_total{from, to, reason}
  retrieval_partial_result_total
  retrieval_total_timeout_total
  shotgun_semaphore_wait_p99
  lif_topology_guard_triggered_total

告警建议：
  fallback_rate > 5%（持续 10min）→ 黄色告警
  total_timeout_rate > 1%         → 红色告警
  topology_guard 触发频率 > 10/h  → 黄色告警（拓扑健康度问题）
```
