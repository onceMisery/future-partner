# geodesic-rerank.md

> 测地线重排是 V8 的工程哲学体现：复用 LIF 阶段缓存的 `accumulatedEnergy` 作为语义地形图，零额外计算地修正 KNN 排序。

## 1. 算法目标

```text
KNN 余弦距离等价于在高维空间画一条"穿山直线"。
但 embedding 空间并非平坦——标签共现矩阵在语义空间中构成"地形"。

测地线重排 = KNN 直线 + 拓扑贴地距离的混合
```

## 2. 核心洞察

```text
情况 A：平坦区域（日常对话）
  Q ─────────── Chunk1   KNN 与测地线一致 → 零风险

情况 B：存在语义山峰（跨域 / 多义词）
  Q ─── ╱╲ ─── Chunk2   KNN 穿山（余弦近但语义远）
       │  └──── Chunk3   测地线绕山（余弦远但 Tag 拓扑近）
                         → 重排把 Chunk3 提上来
```

## 3. 算法实现

```rust
pub struct GeodesicReranker {
    pub alpha: f32,             // 测地线混合权重，默认 0.30
    pub min_geo_samples: usize, // 最小采样密度门槛，默认 4
}

pub struct GeodesicResult {
    pub memory_id: MemoryId,
    pub knn_score: f32,
    pub geo_score: f32,
    pub final_score: f32,
}

impl GeodesicReranker {
    pub fn rerank(
        &self,
        candidates: Vec<KnnHit>,
        energy_field: &HashMap<TagId, f32>,
        chunk_to_tags: &dyn Fn(&MemoryId) -> Vec<TagId>,
    ) -> Vec<GeodesicResult> {
        // 1. 距离场为空 → 退化（L0 防御）
        if energy_field.is_empty() {
            return candidates.into_iter()
                .map(|c| GeodesicResult {
                    memory_id: c.memory_id,
                    knn_score: c.score,
                    geo_score: 0.0,
                    final_score: c.score,
                })
                .collect();
        }

        // 2. 计算每个候选的 geo_score
        let mut intermediate: Vec<_> = candidates.into_iter().map(|c| {
            let tags = chunk_to_tags(&c.memory_id);
            let hits: Vec<f32> = tags.iter()
                .filter_map(|t| energy_field.get(t).copied())
                .collect();
            // 3. 采样密度不足 → 该 chunk geo_score = 0（L1 防御）
            let geo_score = if hits.len() < self.min_geo_samples {
                0.0
            } else {
                hits.iter().sum::<f32>() / hits.len() as f32
            };
            (c.memory_id, c.score, geo_score)
        }).collect();

        // 4. 归一化 geo_score
        let max_geo = intermediate.iter()
            .map(|(_, _, g)| *g)
            .fold(0.0_f32, f32::max);

        // 5. 全部 geo = 0 → 跳过归一化，纯 KNN（L2 防御）
        if max_geo <= f32::EPSILON {
            return intermediate.into_iter().map(|(id, knn, geo)| {
                GeodesicResult { memory_id: id, knn_score: knn, geo_score: geo, final_score: knn }
            }).collect();
        }

        // 6. 混合排序
        intermediate.into_iter().map(|(id, knn, geo)| {
            let geo_norm = geo / max_geo;
            let final_score = (1.0 - self.alpha) * knn + self.alpha * geo_norm;
            GeodesicResult {
                memory_id: id,
                knn_score: knn,
                geo_score: geo_norm,
                final_score,
            }
        }).collect()
    }
}
```

## 4. 三层防御链

| 层级 | 条件 | 行为 |
|---|---|---|
| L0 | `energy_field` 为空 | 整个 rerank 退化，返回 KNN 原序 |
| L1 | chunk 的 `hit_count < min_geo_samples` | 该 chunk geo_score = 0 |
| L2 | 所有 chunk 的 max_geo = 0 | 归一化跳过，全部走纯 KNN |

**最坏情况 = 不改动**（纯 KNN 排序结果原样返回）。

## 5. min_geo_samples 的作用

设定门槛 4 的两个原因：

```text
1. 统计可靠性：只命中 1~3 个 Tag 的 chunk 无法可靠估计测地线距离
2. 过滤"万能 Tag 误拉"：仅命中 1 个高频 Tag 的 chunk 被自动剔除
```

调高门槛 → 更保守，召回更精确；调低 → 更激进，召回更广。

## 6. 默认参数

| 参数 | 默认值 | 建议范围 | 说明 |
|---|---:|---:|---|
| `alpha` | 0.30 | 0.0~1.0 | 0 = 纯 KNN；1 = 纯测地线 |
| `min_geo_samples` | 4 | 2~10 | 最小采样密度门槛 |

## 7. 候选池约束

```text
geodesic_rerank 只重排，不截断
候选池在 rerank 前需保留充分广度（≥ Top-K × 2）
否则地形信息不足
```

## 8. 与检索流水线的集成

```text
KNN 搜索 → TagBoost 向量增强（可选）→ [V8] 测地线重排
        → TimeDecay → Reranker（cross-encoder 精排）→ 最终截断
```

测地线重排位于 cross-encoder 精排之前，保证精排拥有完整候选空间。

## 9. 与 LIF 的协作

```text
1. retrieval pipeline 进入 tagmemo 阶段
2. Synapse.expand(seed_tags) → SpikeTrace { energy_field }
3. TagMemoEngine.lastEnergyField = SpikeTrace.energy_field
4. KNN 检索完成后，调用 geodesic_rerank(candidates, lastEnergyField, ...)
5. 返回重排后的 candidates
```

energy_field 由 LIF 阶段免费产出，不引入额外计算成本。

## 10. 性能预算

```text
计算复杂度 O(|candidates| × avg_tags_per_chunk)
单次预算 ≤ 20ms
SQL 查询 chunk_id → tags 映射：批量 IN 查询，2 次 SQL
内存：临时 HashMap，≤ 10MB
```

## 11. 热调参

```toml
[geodesic_rerank]
alpha = 0.3
min_geo_samples = 4
```

可按 tenant 覆盖。

## 12. 何时启用

```text
启用：
  跨域 query
  多义词 query
  数据集 Tag 共现强（V7 拓扑稠密）

不启用：
  Tag 稀疏（命中样本不足）
  Edge Lite Profile（性能优先）
  Tag 提取质量低（geo_score 不可信）
```

## 13. V8 工程哲学

```text
"最好的优化不是引入新计算，而是发现已有计算中被丢弃的宝藏。"

LIF 距离场本来是 spike propagation 的副产品，被丢弃在内存中。
V8 只是教会系统在检索结束后回头看一眼这张图。
```

详见 `../../方案探索/VCPToolBox 研究与多智能体编排.md` 与 LIF 文档 [lif-spike.md](./lif-spike.md)。
