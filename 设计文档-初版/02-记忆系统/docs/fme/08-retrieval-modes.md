# 08 - 检索模式与流水线

> 公式详见 [scoring-formula.md](./scoring-formula.md)；算法专题见 [tide-algorithm.md](./tide-algorithm.md)、[lif-spike.md](./lif-spike.md)、[geodesic-rerank.md](./geodesic-rerank.md)。

## 1. 检索模式总表

| 模式 | 用途 | 计算成本 | 说明 |
|---|---|---:|---|
| `basic` | 快速回忆 | 低 | HNSW Top-K |
| `hybrid` | 普通生产默认 | 中 | BM25 + HNSW + metadata filter |
| `tide` | 复杂语义 | 中高 | Gram-Schmidt / 投影熵 / 弱信号 |
| `tagmemo` | 联想式召回 | 高 | Tag 共现拓扑 + LIF 扩散 |
| `tagmemo_geodesic` | 多义词、跨域 query | 高 | tagmemo + 测地线重排 |
| `dream` | 后台整理 | 高 | 更大 hop、更低阈值、更强去重 |
| `sbu_safe` | 敏感场景 | 中 | 检索后强脱敏和权限收缩 |

## 2. 标准检索流程

```text
Retrieve Request
  → Verify Session + Mandate
  → Scope Filter（tenant / namespace / layer / visibility）
  → Query Normalize + Sanitizer
  → Embedding
  → Thalamus（Tide 意图分析，可选）
  → 并行：L0 Context Search / L1 Hybrid Recall / L2 HNSW Recall
  → Synapse（LIF Tag Expansion，可选）
  → Candidate Pool
  → Dedup + Rerank
  → PolicyKernel.redact（SBU 脱敏）
  → AuditKernel.append
  → RecallResultStream
```

## 3. Query 规范化

查询预处理必须统一：

```text
去 HTML
去 JSON 包装 / Tool Marker
语言置信度检测
Tag 提示抽取
术语归一化（如 bugfix → bug fix）
长度裁剪
```

与写入路径一样，查询文本也必须经过基础 Sanitizer，但不走阻断式 ContentSafetyGuard。

## 4. 检索模式细节

### 4.1 basic

```text
输入：query embedding
执行：Cortex.HNSW Top-K
输出：按 vector_score 排序
```

适用：
- 边缘部署 Core Lite
- 低延迟要求（P95 < 20ms）
- 召回目标明确、歧义低

### 4.2 hybrid

```text
输入：query text + query embedding
执行：BM25 + HNSW + metadata filter
输出：按 final_score 排序
```

适用：
- 生产默认
- 文本中有术语与字面命中价值
- 需要平衡语义与关键字

### 4.3 tide

```text
输入：query embedding + TagCentroid
执行：Thalamus.decompose → focus_level / weak_signal_tags
输出：修正 query 向量 + 候选 tag 层
```

适用：
- 模糊 query
- 长问题、多主题问题
- 需要捕获被主主题掩盖的弱信号

详见 [tide-algorithm.md](./tide-algorithm.md)。

### 4.4 tagmemo

```text
输入：seed Tags + LifParams
执行：Synapse.expand（LIF）
输出：SpikeTrace + activated Tag 候选
```

适用：
- 需要联想式召回
- 项目上下文中术语存在强共现关系
- 语义引力大于字面相似度

详见 [lif-spike.md](./lif-spike.md)。

### 4.5 tagmemo_geodesic

```text
输入：tagmemo 结果 + lastEnergyField + KNN 候选
执行：测地线重排 finalScore = (1-α) * knn + α * geo
输出：语义地形贴地排序
```

适用：
- 多义词
- 跨域 query
- KNN 直线距离容易穿过"语义山峰"时

详见 [geodesic-rerank.md](./geodesic-rerank.md)。

### 4.6 dream

```text
输入：L1 样本 + 宽松参数（更多 hop / 更低 threshold）
执行：高召回 + 强去重 + 梦境提案
输出：DreamProposal，不直接写入长期记忆
```

适用：
- 后台整理
- 周期性本体压缩
- 检测跨 Episode 弱关联

### 4.7 sbu_safe

```text
输入：任意检索模式结果
执行：PolicyKernel.redact + SBU 硬过滤 + sensitivity_penalty
输出：强脱敏结果 + RedactionReport
```

`sbu_safe` 不能假设上游候选已经安全。即使 RedactionPolicy 已执行，返回前仍必须按 `security_labels` 与 Mandate constraints 做最后一轮硬过滤。

适用：
- 合规审计
- 跨 Agent 上下文共享
- 默认不暴露 SBU 原文的场景

## 5. 打分公式

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
```

默认权重矩阵：

| 模式 | vector | keyword | spike | tag | recency | trust |
|---|---:|---:|---:|---:|---:|---:|
| basic | 0.85 | 0.00 | 0.00 | 0.05 | 0.05 | 0.05 |
| hybrid | 0.55 | 0.20 | 0.00 | 0.10 | 0.10 | 0.05 |
| tagmemo | 0.40 | 0.10 | 0.20 | 0.15 | 0.10 | 0.05 |
| tagmemo_geodesic | 0.30 | 0.10 | 0.20 | 0.15 | 0.10 | 0.05 |
| dream | 0.25 | 0.05 | 0.30 | 0.20 | 0.05 | 0.15 |

详见 [scoring-formula.md](./scoring-formula.md)。

## 6. 可解释性要求

RecallResult 必须在可用时暴露：

```text
vector_score
bm25_score
spike_pathway（激活路径）
weak_signal_tags（Tide）
geodesic_geo_score（tagmemo_geodesic）
redaction_applied（sbu_safe）
```

## 7. 候选池与去重

```text
候选池上限：500
去重顺序：
  1. memory_id 去重
  2. content_hash 去重
  3. 语义相似去重（cross-encoder / centroid 近似）
```

## 8. 超时与降级

降级链：

```text
tagmemo_geodesic → tagmemo → hybrid → basic
```

具体阈值见 [retrieval-fallback.md](./retrieval-fallback.md)。

## 9. 默认 Profile 建议

| Profile | 默认模式 | 可启用增强 |
|---|---|---|
| Edge Lite | basic / hybrid | - |
| Standalone | hybrid / tide | tagmemo |
| Enterprise | hybrid / tagmemo / sbu_safe | tagmemo_geodesic / dream |
| Research | 全开 | 参数热更新 |
