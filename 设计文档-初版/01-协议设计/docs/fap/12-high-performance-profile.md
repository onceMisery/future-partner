# 12 - 高性能 Profile (FAP-HP)

FAP-HP 是一组高性能优化插件的集合，全部可选。

## 1. 特性清单

```text
dynamic_tool_projection
context_folding
dag_batch_invoke
async_task
warm_worker_pool
distributed_capability_node
remote_file_resolver
semantic_capability_routing
sqlite_wal_local_store
usearch_mmap_vector_index
```

## 2. 动态工具注入

```text
用户上下文
  → Intent Parser
  → Capability Candidate Recall (向量检索)
  → Risk Filter
  → Semantic Rerank
  → Top-K Tool Projection
  → Context Folding (按 token 预算)
  → 注入模型 Prompt
```

Projection 接口：

```rust
#[async_trait::async_trait]
pub trait CapabilityProjectionPlugin: Send + Sync {
    async fn project(
        &self,
        ctx: ProjectionContext,
        capabilities: Vec<Capability>,
        budget: TokenBudget,
    ) -> Result<Vec<ProjectedCapability>, FapError>;
}
```

## 3. 上下文折叠等级

| 等级 | 内容 | Token 成本 |
|---|---|---|
| L0 | 只注入能力名 | ~10 |
| L1 | 注入一句话描述 | ~30 |
| L2 | 注入参数摘要 | ~80 |
| L3 | 注入完整调用说明 | ~200 |
| L4 | 注入示例 + 安全约束 + 错误处理 | ~500 |

调度器根据 token 预算自动选择等级。

## 4. 调度策略

```text
短任务:      local cold start
高频任务:    warm worker pool
长任务:      async task
远程重任务:  remote node
批量 DAG:    parallel scheduler + 拓扑排序
```

## 5. 向量索引插件

| Plugin | 说明 |
|---|---|
| USearchVectorIndexPlugin | mmap，高性能（默认） |
| SqliteVecIndexPlugin | sqlite-vec，轻量 |
| PgVectorIndexPlugin | pgvector，分布式 |
| WeaviateIndexPlugin | Weaviate，云原生 |
