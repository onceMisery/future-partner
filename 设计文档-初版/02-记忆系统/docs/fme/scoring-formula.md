# scoring-formula.md

> 检索模式总览见 [08-retrieval-modes.md](./08-retrieval-modes.md)；算法专题见 [tide-algorithm.md](./tide-algorithm.md)、[lif-spike.md](./lif-spike.md)、[geodesic-rerank.md](./geodesic-rerank.md)。

## 1. final_score 公式

```text
final_score =
  w_vector   * vector_score
+ w_keyword  * bm25_score
+ w_spike    * lif_spike_score
+ w_tag      * tag_topology_score
+ w_recency  * recency_score
+ w_trust    * source_confidence
- w_dup      * duplication_penalty
- w_sbu      * sensitivity_penalty
+ w_geo      * geodesic_score                 (仅 tagmemo_geodesic)
```

约束：所有正项 + 负项之和落在 [-1, +1.5]，候选间相对排序优先于绝对值。

## 2. 各子项定义

### vector_score

```text
vector_score = max(0, cosine_similarity(query_embedding, candidate_embedding))
```

候选 embedding 必须与 query 在同一索引版本（embedding_model 相同）。

### bm25_score

```text
bm25_score = BM25(query_text, candidate_text) 标准实现
归一化到 [0, 1]：bm25_score = sigmoid(raw_bm25 / bm25_scale)
默认 bm25_scale = 5.0
```

### lif_spike_score

```text
LIF 扩散后，候选 MemoryUnit 关联 Tag 的累积能量：
  lif_spike_score = Σ accumulated_energy(tag_i) / |tags|
  归一化到 [0, 1]
```

详见 [lif-spike.md](./lif-spike.md)。

### tag_topology_score

```text
候选与 query Tag 的拓扑接近度：
  tag_topology_score = avg(min_distance(query_tag, candidate_tag))
  其中 min_distance 在 V7 有向图中按势能加权
  归一化到 [0, 1]
```

### recency_score

```text
recency_score = exp(-age_days / half_life_days)
默认 half_life_days：
  L0: 1
  L1: 30
  L2: 180
```

### source_confidence

直接读取 MemoryUnit.source_confidence，范围 [0, 1]。

### duplication_penalty

```text
当候选与已选结果中某条 cosine > dedup_threshold（默认 0.92）时：
  duplication_penalty = (1 - 1/rank_in_dedup_cluster) * cosine_similarity
否则 0
```

### sensitivity_penalty

```text
若 candidate.security_labels 含 "sbu" 且 mandate 不允许：
  sensitivity_penalty = 1.0  → 必然过滤（不仅是降权）
若含 "internal" 且 visibility = foreign：
  sensitivity_penalty = 0.5
否则 0
```

### geodesic_score

```text
仅在 tagmemo_geodesic 模式下生效：
  geo_score = 候选 Tag 在 LIF accumulatedEnergy 场中的命中均能量
  finalScore = (1 - alpha) * knn_score + alpha * normalized_geo_score
```

详见 [geodesic-rerank.md](./geodesic-rerank.md)。

## 3. 默认权重矩阵

| 模式 | vector | keyword | spike | tag | recency | trust | dup | sbu | geo |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| basic | 0.85 | 0.00 | 0.00 | 0.05 | 0.05 | 0.05 | 0.10 | 1.00 | - |
| hybrid | 0.55 | 0.20 | 0.00 | 0.10 | 0.10 | 0.05 | 0.15 | 1.00 | - |
| tide | 0.45 | 0.15 | 0.00 | 0.20 | 0.10 | 0.10 | 0.15 | 1.00 | - |
| tagmemo | 0.40 | 0.10 | 0.20 | 0.15 | 0.10 | 0.05 | 0.15 | 1.00 | - |
| tagmemo_geodesic | 0.30 | 0.10 | 0.20 | 0.15 | 0.10 | 0.05 | 0.15 | 1.00 | 0.30 |
| dream | 0.25 | 0.05 | 0.30 | 0.20 | 0.05 | 0.15 | 0.20 | 1.00 | - |
| sbu_safe | 0.50 | 0.20 | 0.05 | 0.10 | 0.10 | 0.05 | 0.10 | 1.00 | - |

注：
- `w_sbu = 1.0` 表示 SBU 在该模式下硬过滤（不仅是降权）。
- `sbu_safe` 模式仍保留 `w_sbu = 1.0` 作为最后一道硬过滤。即使上游预期已脱敏，检索流水线也不能依赖该前提。
- 权重可按租户在 PluginConfig 覆盖。

## 4. 归一化

候选打分前，每个子分数必须经过版本化校准归一化，而不是按单次候选池做 min-max：

```text
score_norm = clamp01(calibrator_v(raw_score))
```

校准器来源：

| 子项 | 校准方式 |
|---|---|
| vector_score | cosine 已在 [0,1]，按 embedding_model + index_version 做温度校准 |
| bm25_score | sigmoid(raw_bm25 / bm25_scale)，`bm25_scale` 固定进 scoring_version |
| lif_spike_score | 除以理论上限或离线 P95，按 tag_graph_version 固定 |
| tag_topology_score | 距离转相似度后按图版本校准 |
| recency_score | 指数衰减天然稳定 |
| source_confidence | 来源 trust 分数，禁止候选池内归一化 |

禁止把生产 final_score 建立在单次候选池 `min(scores)/max(scores)` 上。原因：

```text
1. 不同请求之间不可比
2. 离群值会改变所有候选得分
3. 审计回放难以稳定复现
4. 部分结果或 fallback 会改变 min/max，导致排序漂移
```

允许在同一候选池内使用 robust rank feature 作为额外 tie-breaker，但必须写入 `scoring_version`，且不能替代上述稳定校准器。

## 5. 可解释性输出

每个 RecallResult 必须暴露 ScoreBreakdown：

```proto
message ScoreBreakdown {
  float vector_score = 1;
  float keyword_score = 2;
  float spike_score = 3;
  float tag_topology_score = 4;
  float recency_score = 5;
  float trust_score = 6;
  float duplication_penalty = 7;
  float sensitivity_penalty = 8;
  optional float geodesic_score = 9;
  float final_score = 10;
}
```

便于：
- 审计可解释性
- 离线评估
- 权重调优

## 6. 调优指南

### 6.1 检索结果"太相似"

- 增加 `w_dup`（默认 0.15 → 0.25）
- 降低 `dedup_threshold`（默认 0.92 → 0.88）

### 6.2 检索结果"太陈旧"

- 增加 `w_recency`（默认 0.10 → 0.20）
- 缩短 half_life_days

### 6.3 召回质量低

- 提高 `w_vector`，降低 `w_spike`
- 切换到 hybrid 模式

### 6.4 联想不足

- 提高 `w_spike` 与 `w_tag`
- 切换到 tagmemo / tagmemo_geodesic

### 6.5 多义词错误

- 启用 tagmemo_geodesic
- 调高 alpha（默认 0.30 → 0.50）

## 7. 权重热更新

```toml
[scoring.weights.tagmemo]
w_vector = 0.40
w_keyword = 0.10
w_spike = 0.20
w_tag = 0.15
w_recency = 0.10
w_trust = 0.05
w_dup = 0.15
w_sbu = 1.00
```

通过配置文件或 admin API 热更新；变更立即生效，下一次检索使用新权重。

## 8. 评估指标

```text
hit_rate@K            Top-K 中有命中的比例
nDCG@K                折损累计增益
recall_latency_p99    P99 延迟
score_explainability  ScoreBreakdown 完整性
```

详见 [17-testing-conformance.md](./17-testing-conformance.md)。
