# 1. OCMS 总体架构设计

OCMS，即 Omniscient Cognitive Memory System，全局认知记忆系统，不应被设计成传统 RAG 的“外部知识库”，而应被设计成 Agent 的长期认知底座。其核心目标是让 Agent 具备：

1. 可持续运行的短期上下文管理能力；
2. 可沉淀、可演化、可修正的长期记忆；
3. 可联想、可反思、可遗忘的认知拓扑；
4. 可被任意语言、任意 Agent 框架接入的独立记忆服务。

整体架构可分为六个核心子系统：

```text
┌────────────────────────────────────────────┐
│             Agent / LLM / Tool Runtime      │
└──────────────────────┬─────────────────────┘
                       │
┌──────────────────────▼─────────────────────┐
│         Memory Access Gateway / MCP API     │
│   REST / gRPC / WebSocket / MCP / SDK Adapter│
└──────────────────────┬─────────────────────┘
                       │
┌──────────────────────▼─────────────────────┐
│        Reasoning-Driven Memory Router       │
│ Query Rewrite / EPA / Intent Gate / Policy  │
└───────┬───────────────┬──────────────┬──────┘
        │               │              │
┌───────▼──────┐ ┌──────▼──────┐ ┌─────▼──────┐
│ L0 即时感知层 │ │ L1 剧集折叠层 │ │ L2 语义本体层 │
│ Ring Buffer  │ │ Summary Tree │ │ Graph/Fact KB│
└───────┬──────┘ └──────┬──────┘ └─────┬──────┘
        │               │              │
        └───────────────▼──────────────┘
              Hybrid Memory Store
     JSONL / SQLite / Graph / Tree / Vector
                       │
┌──────────────────────▼─────────────────────┐
│         Subconscious Processing Layer        │
│ DreamWave / Reflect / Decay / SBU / Compact  │
└────────────────────────────────────────────┘
```

设计原则：

| 原则       | 说明                                                              |
| -------- | --------------------------------------------------------------- |
| 记忆分层     | 不同生命周期的数据进入不同层级，避免短期上下文、摘要、长期事实混杂。                              |
| 摘要与事实分离  | L1 摘要只服务对话连续性，L2 事实才参与长期认知演化。文档也强调 OCMS 要将 Summary 与 Fact 物理剥离。 |
| 图-树-向量融合 | 图负责关系联想，树负责长文档结构推理，向量负责语义近邻召回。                                  |
| 检索先推理后召回 | 先分析查询意图、领域、实体、噪声，再发起多路检索，避免“911 图表”这类查询构建错误。                    |
| 记忆可遗忘    | 长期运行系统必须支持柔性衰减和强制删除，否则会形成记忆污染。                                  |
| 服务化与协议化  | 记忆系统独立运行，通过统一协议接入任意语言与任意 Agent。                                 |

---

# 2. 分层设计：L0、L1、L2 与潜意识处理层

## 2.1 L0 即时感知层：Sensory Register & Working Memory

L0 是 Agent 的“当前注意力窗口”，负责保存最新交互、当前任务状态、短期指代关系和未完成动作。

**核心职责：**

| 能力     | 说明                                 |
| ------ | ---------------------------------- |
| 原始事件捕获 | 保存用户输入、模型输出、工具调用、文件操作、代码执行结果等原始事件。 |
| 短期指代消解 | 处理“刚才那个”“上一步”“这个方案”等依赖最近上下文的表达。    |
| 当前任务状态 | 保存当前目标、约束、已完成步骤、待办步骤。              |
| 零损失回放  | L0 原文必须可被精确回放，不能只保存摘要。             |

**推荐实现：**

```text
L0 = In-Memory RingBuffer + Append-Only JSONL WAL
```

数据结构示例：

```json
{
  "event_id": "evt_20260430_000001",
  "session_id": "sess_xxx",
  "agent_id": "agent_researcher",
  "role": "user",
  "content": "请基于 03.记忆系统设计方案.md...",
  "timestamp": 1777575600,
  "modality": "text",
  "tokens": 812,
  "tags": ["memory-system", "OCMS", "design"],
  "trace_id": "trace_xxx"
}
```

**工程策略：**

L0 常驻内存只保留最近 5～10 轮原始交互，这与文档中对 L0 的定义一致：该层保持当前会话窗口内最近的 5 至 10 轮原始交互，用于保证高频对话下的语义精确和短期指代消解。

落盘采用追加写 JSONL，而不是频繁更新数据库。原因是 L0 写入频率最高，JSONL 能降低锁竞争，并且异常退出后可快速恢复。

---

## 2.2 L1 剧集折叠层：Episodic Compaction Layer

L1 是“可压缩但可回溯”的剧集记忆层，解决长对话超出 Token 预算后的上下文压缩问题。

**核心职责：**

| 能力            | 说明                            |
| ------------- | ----------------------------- |
| short_compact | 将 L0 原始交互压缩为结构化摘要。            |
| short_expand  | 根据摘要节点回溯原始 L0 事件。             |
| 摘要树维护         | 多级合并摘要，形成 Summary Tree。       |
| 决议提取          | 从对话中提取“用户偏好、已确认决策、实体变化、任务状态”。 |

**摘要节点结构：**

```json
{
  "summary_id": "sum_001",
  "level": 1,
  "source_events": ["evt_001", "evt_002", "evt_003"],
  "summary": "用户要求基于 03 文档设计完整 OCMS 方案...",
  "decisions": [
    "方案需要支持任意语言接入",
    "需要包含 tagMemory 数学实现"
  ],
  "preferences": [
    "用户偏好工程化落地方案，而非纯概念描述"
  ],
  "entities": [
    {"name": "OCMS", "type": "system"},
    {"name": "tagMemory", "type": "module"}
  ],
  "confidence": 0.86,
  "created_at": 1777575700
}
```

**压缩策略：**

```text
L0 原始事件
  -> 按 token 阈值 / 语义边界切片
  -> SLM/LLM 摘要
  -> 提取决议、偏好、实体、任务状态
  -> 写入 L1 Summary Tree
  -> 原始 L0 保留引用，不物理删除
```

文档强调 L1 不应简单截断历史，而应通过 short_compact 生成摘要，并保留对原始 L0 文本的索引，以便 short_expand 溯源，保证“可压缩但无损”。

---

## 2.3 L2 语义本体层：Semantic Ontology & Workspace

L2 是长期记忆核心，不再按照时间线保存对话，而是把对话中稳定、有价值、可复用的信息转化为事实、观点、实体和关系。

**推荐四类长期记忆：**

| 类型         | 说明      | 示例                                  |
| ---------- | ------- | ----------------------------------- |
| World      | 客观世界事实  | “SQLite WAL 支持读写并发。”                |
| Experience | 第一人称经验  | “上次用户在 Dubbo 协议升级问题上关注灰度迁移。”        |
| Opinion    | 带置信度的观点 | “对于轻量 Agent，SQLite + FTS5 优于独立向量库。” |
| Entity     | 实体画像    | 用户、项目、代码库、协议、模块、文档等。                |

文档中对 L2 的定义是：L1 提取出的事实块经过置信度评估后，通过 long_retain 写入长期工作区，数据被解构为世界、经验、观点和实体四类图谱结构。

**L2 事实结构：**

```json
{
  "fact_id": "fact_001",
  "type": "preference",
  "subject": "user",
  "predicate": "prefers",
  "object": "backend-minimal engineering constraints",
  "source": ["sum_001", "evt_001"],
  "confidence": 0.91,
  "salience": 0.74,
  "valid_from": 1777575700,
  "valid_to": null,
  "last_accessed_at": 1777575700,
  "embedding_id": "vec_001",
  "tags": ["preference", "backend", "minimal"]
}
```

**L2 写入原则：**

不是所有信息都进入 L2。推荐写入门槛：

```text
L2 写入条件 =
  信息稳定性 > 0.6
  AND 可复用性 > 0.5
  AND 来源可信度 > 0.7
  AND 非临时闲聊
```

**冲突处理：**

当新事实与旧事实冲突时，不直接覆盖，而是创建版本关系：

```text
fact_old --contradicted_by--> fact_new
fact_new --supersedes--> fact_old
```

再由潜意识层进行置信度仲裁。

---

## 2.4 潜意识处理层：Subconscious Processing Layer

潜意识层是后台守护系统，负责反思、巩固、遗忘、清洗和结构优化。

**核心任务：**

| 模块              | 作用                             |
| --------------- | ------------------------------ |
| DreamWaveEngine | 离线关联 L0/L1/L2 记忆，形成新的共鸣桥和长期规则。 |
| long_reflect    | 整合同一实体的离散事实，发现矛盾并调整置信度。        |
| long_forget     | 基于访问频率、时间、重要性进行柔性遗忘。           |
| SBU Engine      | 对必须删除的记忆执行同步回流遗忘。              |
| Index Optimizer | 重建向量索引、图边权重、目录树摘要。             |
| Memory Auditor  | 检测污染、重复、过期、幻觉型事实。              |

文档明确将潜意识层定义为独立于主交互线程的后台守护进程，负责 VCP 的 AgentDream 梦境涟漪计算与 SBU 机器遗忘调度。

---

# 3. 数据存储策略：图-树-向量异构融合模型

OCMS 不应只用一个向量库。完整的存储底座应由五类存储组成：

| 存储域    | 数据模型                  | 推荐引擎                                 | 主要用途                   |
| ------ | --------------------- | ------------------------------------ | ---------------------- |
| L0 会话流 | Append-only Event Log | JSONL / RocksDB / BadgerDB           | 原始交互、事件回放、审计           |
| L1 摘要树 | Tree / DAG            | SQLite / PostgreSQL / LiteFS         | 剧集摘要、层级压缩、short_expand |
| L2 事实库 | Relational + FTS      | SQLite WAL + FTS5 / PostgreSQL       | 结构化事实、实体、置信度           |
| 语义拓扑图  | Weighted Graph        | 内存图 + 持久化边表 / Neo4j 可选               | 标签共现、LIF 扩散、多跳联想       |
| 向量索引   | Dense Vector          | USearch / Qdrant / Milvus / pgvector | 语义近邻、多模态召回             |

文档中的异构存储表也采用类似划分：短时会话流使用 JSONL/KV，工作区事实库使用 SQLite WAL + FTS5，拓扑语义网络使用加权无向图，专业长文档使用目录树，多模态潜空间使用 USearch + mmap。

## 3.1 关系库核心表设计

### memory_event

保存 L0 原始事件。

```sql
CREATE TABLE memory_event (
  event_id TEXT PRIMARY KEY,
  session_id TEXT NOT NULL,
  agent_id TEXT NOT NULL,
  role TEXT NOT NULL,
  content TEXT NOT NULL,
  modality TEXT DEFAULT 'text',
  token_count INTEGER,
  created_at INTEGER NOT NULL,
  trace_id TEXT,
  metadata_json TEXT
);
```

### episodic_summary

保存 L1 摘要树节点。

```sql
CREATE TABLE episodic_summary (
  summary_id TEXT PRIMARY KEY,
  parent_summary_id TEXT,
  level INTEGER NOT NULL,
  summary TEXT NOT NULL,
  source_event_ids TEXT NOT NULL,
  decisions_json TEXT,
  preferences_json TEXT,
  entities_json TEXT,
  confidence REAL DEFAULT 0.8,
  created_at INTEGER NOT NULL
);
```

### memory_fact

保存 L2 长期事实。

```sql
CREATE TABLE memory_fact (
  fact_id TEXT PRIMARY KEY,
  fact_type TEXT NOT NULL,
  subject TEXT NOT NULL,
  predicate TEXT NOT NULL,
  object TEXT NOT NULL,
  confidence REAL NOT NULL,
  salience REAL NOT NULL,
  source_ids TEXT NOT NULL,
  valid_from INTEGER,
  valid_to INTEGER,
  last_accessed_at INTEGER,
  created_at INTEGER NOT NULL,
  updated_at INTEGER NOT NULL
);
```

### entity

```sql
CREATE TABLE entity (
  entity_id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  entity_type TEXT NOT NULL,
  aliases_json TEXT,
  profile_json TEXT,
  confidence REAL,
  created_at INTEGER,
  updated_at INTEGER
);
```

### tag_edge

保存同现图边。

```sql
CREATE TABLE tag_edge (
  src_tag TEXT NOT NULL,
  dst_tag TEXT NOT NULL,
  weight REAL NOT NULL,
  cooccur_count INTEGER NOT NULL,
  last_updated_at INTEGER,
  PRIMARY KEY (src_tag, dst_tag)
);
```

## 3.2 图模型：标签/实体同现拓扑

图的节点可以是：

```text
Tag、Entity、Fact、Topic、DocumentNode、UserPreference、Task
```

边类型：

```text
co_occurs_with
supports
contradicts
derived_from
supersedes
belongs_to
mentions
causes
used_by
```

边权重更新：

```text
w_ij(t+1) = decay(w_ij(t)) + cooccur_boost + salience_boost + confidence_boost
```

其中：

```text
decay(w) = w * exp(-λ * Δt)
cooccur_boost = α * same_window_count
salience_boost = β * avg_salience
confidence_boost = γ * avg_confidence
```

实现上，热图保存在内存：

```go
type Graph map[TagID]map[TagID]float64
```

冷数据保存在 `tag_edge` 表中，服务启动时加载 Top-N 热边，后台周期性压缩低权重边。

## 3.3 树模型：Vectorless RAG 目录树

专业长文档、代码库、产品文档不应粗暴向量切块，而应构建目录树：

```text
Document
 ├── Chapter 1
 │    ├── Section 1.1
 │    └── Section 1.2
 └── Chapter 2
      ├── Section 2.1
      └── Section 2.2
```

节点结构：

```json
{
  "node_id": "doc_001_sec_2_1",
  "doc_id": "doc_001",
  "parent_id": "doc_001_ch_2",
  "title": "记忆检索机制",
  "summary": "本节描述霰弹枪查询、LIF 脉冲扩散和测地线重排...",
  "start_offset": 12340,
  "end_offset": 18220,
  "children": [],
  "linked_facts": ["fact_031", "fact_044"],
  "linked_tags": ["Shotgun Query", "LIF", "Geodesic Rerank"]
}
```

文档也指出，OCMS 引入 Vectorless RAG 与目录树结构，针对 Markdown/PDF 构建内容目录树，每个节点保存 Node ID、起止位置和层级摘要，检索时基于上下级关系推理遍历。

## 3.4 向量模型：多索引分层

推荐不要只维护一个全局向量索引，而是维护多个 namespace：

```text
vec:l0_recent
vec:l1_summary
vec:l2_fact
vec:doc_tree_node
vec:multimodal_image
vec:tool_trace
```

向量元数据：

```json
{
  "vector_id": "vec_001",
  "namespace": "vec:l2_fact",
  "target_type": "fact",
  "target_id": "fact_001",
  "embedding_model": "bge-m3 / jina-embeddings / text-embedding-3-large",
  "dimension": 1024,
  "created_at": 1777575700
}
```

---

# 4. 记忆写入链路

完整写入流程如下：

```text
用户输入 / 工具结果 / Agent 输出
    ↓
Event Capture
    ↓
L0 RingBuffer + JSONL WAL
    ↓
Memory Classifier
    ├── 临时上下文：仅保留 L0
    ├── 剧集信息：进入 L1 摘要树
    └── 长期事实：进入 L2 Fact KB
    ↓
Tag Extraction + Entity Linking
    ↓
Embedding Generate
    ↓
Graph Edge Update
    ↓
Index Commit
```

## 写入决策评分

```text
retain_score =
  0.30 * salience
+ 0.25 * stability
+ 0.20 * user_preference_signal
+ 0.15 * task_reuse_value
+ 0.10 * source_confidence
- 0.20 * privacy_risk
- 0.15 * noise_score
```

建议阈值：

| retain_score | 动作            |
| ------------ | ------------- |
| < 0.35       | 只留 L0，过期删除    |
| 0.35～0.60    | 进入 L1 摘要      |
| 0.60～0.80    | 进入 L2 事实库     |
| > 0.80       | 进入 L2，并标记高显著性 |
| > 0.90 且多次验证 | 晋升为核心记忆       |

---

# 5. 记忆检索机制

OCMS 检索不是“输入一句话，查一次向量库”，而是一条多阶段认知检索流水线：

```text
Query
 ↓
Query Understanding / EPA
 ↓
Shotgun Query
 ↓
Hybrid Recall: BM25 + Vector + Graph + Tree
 ↓
LIF Spike Propagation
 ↓
SVD / Gram-Schmidt Dedup
 ↓
Geodesic Rerank
 ↓
Context Packing
 ↓
LLM Answer
```

---

## 5.1 霰弹枪查询：Shotgun Query

霰弹枪查询用于解决长上下文中的“单一查询向量语义稀释”问题。文档中提出，OCMS 会基于 0.70 的向量相似度阈值对当前窗口对话分段，并发起 N+1 次查询：1 个当前意图查询，N 个历史上下文分段查询。

流程：

```text
当前上下文 C
  ↓
按语义相似度切分为 S1, S2, ..., Sn
  ↓
构造查询：
Q0 = 当前用户问题
Q1 = S1 的摘要查询
Q2 = S2 的摘要查询
...
Qn = Sn 的摘要查询
  ↓
并发召回
```

伪代码：

```python
def shotgun_query(current_query, context_segments):
    queries = [rewrite_current_intent(current_query)]
    for seg in context_segments:
        if semantic_distance(seg, current_query) < 0.70:
            queries.append(build_segment_query(seg))
    return parallel_retrieve(queries)
```

适用场景：

| 场景     | 原因                    |
| ------ | --------------------- |
| 长对话    | 当前问题可能依赖前文多个片段        |
| 多任务混杂  | 单一查询会偏向最后一个话题         |
| 隐式指代   | “那个方案”“前面提到的结构”需要回溯   |
| 复杂研究任务 | 需要同时召回需求、约束、参考材料和历史结论 |

---

## 5.2 LIF 脉冲扩散：Leaky Integrate-and-Fire

LIF 脉冲扩散用于从初始标签触发多跳联想。文档说明，OCMS 会在同现拓扑图上以初始关键词为刺激源，让能量向相邻概念扩散，并进行 2～3 跳关联游走。

定义：

```text
V_i(t+1) = leak * V_i(t) + Σ_j W_ji * spike_j(t) + I_i(t)
```

其中：

| 符号      | 含义                |
| ------- | ----------------- |
| V_i     | 节点 i 的膜电位/激活能量    |
| leak    | 泄露系数，建议 0.65～0.90 |
| W_ji    | 标签 j 到标签 i 的边权重   |
| spike_j | 节点 j 是否放电         |
| I_i     | 查询注入的外部刺激         |
| θ       | 放电阈值              |

放电规则：

```text
spike_i(t) = 1 if V_i(t) >= θ else 0
```

推荐参数：

| 参数              |  默认值 | 说明                   |
| --------------- | ---: | -------------------- |
| max_hops        |    3 | 多数任务 2～3 跳足够，过高会引入噪声 |
| leak            | 0.75 | 控制能量衰减               |
| θ               | 0.60 | 节点放电阈值               |
| top_k_neighbors |   20 | 每跳最多扩散的邻居数量          |
| min_edge_weight | 0.05 | 过滤弱关系边               |

输出：

```json
{
  "activated_tags": [
    {"tag": "OCMS", "energy": 1.00, "hop": 0},
    {"tag": "tagMemory", "energy": 0.82, "hop": 1},
    {"tag": "LIF", "energy": 0.71, "hop": 2}
  ],
  "activation_path": [
    ["OCMS", "tagMemory", "LIF"]
  ]
}
```

---

## 5.3 SVD / Gram-Schmidt 去重

霰弹枪查询会召回大量候选记忆，必须去重。文档中提出用 SVD 潜主题去重和 Gram-Schmidt 正交化，选择能够覆盖“未解释主题能量”的片段，避免 Token 爆炸。

核心思想：

```text
候选记忆向量集合 M = {m1, m2, ..., mk}
已选择集合 S = {}
每次选择与当前已选集合最不冗余、同时与查询最相关的记忆
```

评分：

```text
score(m) = relevance(m, q) - redundancy(m, S) + novelty(m, residual_topic)
```

其中：

```text
redundancy(m, S) = max cosine(m, s), s ∈ S
```

推荐阈值：

| 参数                         | 建议        |
| -------------------------- | --------- |
| dedup_similarity_threshold | 0.86～0.92 |
| max_context_memories       | 8～20      |
| min_novelty_score          | 0.15      |
| max_tokens_per_memory      | 300～800   |

---

## 5.4 测地线重排：Geodesic Rerank

测地线重排用于弥补纯向量 KNN 的缺陷。比如“Bank”在金融语境中不应召回河岸文本。

文档中说明，Geodesic Rerank 会利用 LIF 扩散阶段缓存的 accumulatedEnergy，对候选片段绑定标签计算 GeoScore，再混合 KNN 空间距离与拓扑距离。

推荐公式：

```text
final_score =
  α * vector_score
+ β * bm25_score
+ γ * geo_score
+ δ * recency_score
+ ε * confidence_score
- ζ * noise_score
```

默认权重：

| 权重                 |  默认值 |
| ------------------ | ---: |
| α vector_score     | 0.35 |
| β bm25_score       | 0.15 |
| γ geo_score        | 0.25 |
| δ recency_score    | 0.10 |
| ε confidence_score | 0.10 |
| ζ noise_score      | 0.05 |

GeoScore：

```text
geo_score(chunk) =
  max accumulatedEnergy(tag)
  for tag in chunk.tags
```

如果候选片段没有足够标签，走回退链：

```text
L2: 使用精确标签测地线
L1: 使用摘要标签测地线
L0: 回退到向量 + BM25
```

---

# 6. tagMemory 核心功能：数学实现与参数调优

tagMemory 是 OCMS 的语义调焦系统，负责把“标签拓扑能量”注入向量检索过程，使查询不只是语义近邻搜索，而是受认知拓扑影响的动态搜索。

文档中列出了 tagMemory 的关键参数，包括 `noise_penalty`、`tagWeightRange`、`dynamicBoostRange`、`coreBoostRange`、`deduplicationThreshold`，并说明这些参数用于塑造智能体的认知焦点。

## 6.1 标签能量计算

每个标签的基础能量：

```text
E_tag =
  a * frequency_score
+ b * recency_score
+ c * salience_score
+ d * confidence_score
+ e * graph_activation_score
```

推荐权重：

| 项                      |   权重 |
| ---------------------- | ---: |
| frequency_score        | 0.15 |
| recency_score          | 0.15 |
| salience_score         | 0.25 |
| confidence_score       | 0.20 |
| graph_activation_score | 0.25 |

## 6.2 动态 Beta 向量融合

令：

```text
q = 原始查询向量
t = 标签能量聚合向量
β = 动态标签注入系数
```

融合向量：

```text
q' = normalize((1 - β) * q + β * t)
```

β 的推荐计算：

```text
β = clamp(
  β_min + k1 * logic_depth + k2 * semantic_coherence - k3 * noise,
  β_min,
  β_max
)
```

参数：

| 参数    |   默认 |
| ----- | ---: |
| β_min | 0.05 |
| β_max | 0.45 |
| k1    | 0.18 |
| k2    | 0.16 |
| k3    | 0.20 |

解释：

| 情况              | β 行为             |
| --------------- | ---------------- |
| 用户问题专业、严谨、上下文连贯 | β 增大，查询向核心标签靠拢   |
| 用户闲聊、发散、噪声高     | β 降低，避免过度联想      |
| 命中核心实体          | β 受 coreBoost 提升 |
| 标签冲突明显          | β 降低，并触发多路查询     |

## 6.3 核心参数建议

| 参数                     |          默认值 |         建议范围 | 调优方向                            |
| ---------------------- | -----------: | -----------: | ------------------------------- |
| noise_penalty          |         0.05 |    0.01～0.20 | 越高越保守，适合生产环境；越低越发散，适合研究型 Agent。 |
| tagWeightRange         | [0.05, 0.45] | [0.03, 0.60] | 控制标签引力对查询向量的影响。                 |
| dynamicBoostRange      |   [0.3, 2.0] |   [0.2, 3.0] | 控制专业查询下的认知聚焦强度。                 |
| coreBoostRange         | [1.20, 1.40] |   [1.0, 1.8] | 提升核心实体、用户偏好、长期项目的优先级。           |
| deduplicationThreshold |         0.88 |    0.80～0.95 | 越低合并越激进，越高保留越细。                 |

## 6.4 调优方法

建议建立离线评测集：

```text
query, expected_memory_ids, forbidden_memory_ids, expected_tags
```

指标：

| 指标               | 说明                 |
| ---------------- | ------------------ |
| Recall@K         | 正确记忆是否被召回          |
| MRR              | 正确记忆排名是否靠前         |
| Noise Rate       | 无关记忆比例             |
| Conflict Rate    | 冲突事实误召回比例          |
| Expansion Depth  | LIF 平均扩散跳数         |
| Token Efficiency | 每 1K Token 中有效记忆比例 |

调参顺序：

```text
先调召回宽度：top_k、β_max、tagWeightRange
再调噪声控制：noise_penalty、min_edge_weight
再调排序质量：geo_score 权重、confidence 权重
最后调长期记忆稳定性：decay、salience、confidence
```

---

# 7. 潜意识架构：梦境系统与机器遗忘引擎

## 7.1 DreamWaveEngine 梦境系统

DreamWaveEngine 是离线记忆巩固系统，不参与实时响应，适合在低负载周期运行。

文档中描述的 DreamWaveEngine 会在近期涟漪阶段聚焦 0～7 天未处理事实，随机抓取 3 个经验节点，并按 3/5/7 搜索步长寻找共鸣桥；中期回音阶段会把跨度拉到 7～90 天，尝试形成规则级本体认知。

推荐流程：

```text
每日低负载窗口触发
  ↓
选取近期高显著性 / 高冲突 / 未巩固记忆
  ↓
多跳召回相关 L1 / L2 记忆
  ↓
构造梦境任务：
    “这些记忆之间是否存在新关系？”
    “是否存在矛盾？”
    “是否能抽象出稳定规则？”
  ↓
生成候选关系 / 候选规则
  ↓
置信度校验
  ↓
写入 L2 或等待人工审核
```

梦境任务类型：

| 类型                   | 说明               |
| -------------------- | ---------------- |
| Resonance Bridge     | 在离散事件之间建立关联桥。    |
| Conflict Arbitration | 发现事实/观点冲突并调整置信度。 |
| Rule Distillation    | 从多次经验中提炼规则。      |
| Entity Consolidation | 合并重复实体、别名、项目画像。  |
| Memory Promotion     | 将高显著性记忆晋升为核心记忆。  |
| Memory Demotion      | 将低价值记忆降权或归档。     |

## 7.2 机器遗忘引擎

机器遗忘分两类：

### A. 柔性遗忘：Access-based Decay

用于普通过期信息。

```text
decay_score = exp(- ln(2) * Δt / half_life)
```

最终记忆权重：

```text
memory_weight =
  base_score
* decay_score
* confidence
* salience
```

建议半衰期：

| 记忆类型      |             半衰期 |
| --------- | --------------: |
| 临时闲聊      |           1～3 天 |
| 普通任务上下文   |          7～14 天 |
| 项目经验      |         30～90 天 |
| 用户偏好      |       180～365 天 |
| 核心身份/长期规则 | 不自动衰减，只人工或强规则修改 |

文档也强调，访问驱动衰减不是简单删除，而是未被访问的记忆在召回排序中按半衰期下降，新近高显著性事实自然覆盖旧记忆。

### B. 强制遗忘：SBU 同步回流遗忘

用于以下场景：

| 场景     | 示例               |
| ------ | ---------------- |
| 用户要求删除 | “忘记我的某个偏好/身份信息。” |
| 隐私合规   | 敏感信息需要彻底删除。      |
| 记忆污染   | 幻觉事实被写入长期记忆。     |
| 越狱攻击   | 恶意指令被误存为系统偏好。    |

SBU 流程：

```text
Forget Request
  ↓
定位目标记忆 Fact / Entity / Tag / Vector
  ↓
依赖闭包分析
  ↓
删除或降权关联节点
  ↓
删除向量索引
  ↓
删除摘要引用或标记 tombstone
  ↓
更新图边引用计数
  ↓
加入 forbidden memory filter
  ↓
审计与验证
```

文档中提到，SBU 在记忆网络路径中通过依赖闭包裁剪实体和附属数据，在参数对齐路径中通过随机参考对齐、向量扰动或特征流重定向来防止被删除内容回流。

工程上，如果无法控制底层大模型参数，可用以下替代策略：

```text
1. RAG 层硬删除；
2. 向量索引删除；
3. Graph 边删除；
4. Query-time forbidden filter；
5. Prompt-level non-recall policy；
6. 审计测试集验证不可召回。
```

---

# 8. 性能优化、扩展性与可观测性

## 8.1 性能优化

文档建议核心路由、L0/L1 上下文缓冲和底层控制流采用 Go，向量检索引擎采用 Rust/USearch，并通过 mmap 降低内存占用。

推荐技术策略：

| 方向    | 方案                                                      |
| ----- | ------------------------------------------------------- |
| L0 写入 | Append-only JSONL，批量 fsync                              |
| L1 摘要 | 异步 compact，避免阻塞主响应                                      |
| L2 事实 | SQLite WAL / PostgreSQL 分区表                             |
| 向量索引  | USearch mmap / Qdrant HNSW                              |
| 图计算   | 热图内存化，冷边落库                                              |
| 多租户   | tenant_id 分区 + namespace 隔离                             |
| 缓存    | Query plan cache、Embedding cache、Graph activation cache |
| 并发    | 读写分离，后台任务限流                                             |
| 压缩    | 低频 L0 归档为 zstd JSONL                                    |

## 8.2 扩展性设计

推荐服务拆分：

```text
memory-gateway
memory-router
memory-writer
memory-retriever
memory-graph
memory-vector
memory-dream
memory-forget
memory-observability
```

小规模部署可以单体运行：

```text
Go 单体服务 + SQLite WAL + USearch + JSONL
```

中大型部署可以拆成：

```text
Kubernetes + PostgreSQL + Qdrant/Milvus + Redis + Kafka + OpenTelemetry
```

## 8.3 可观测性

文档强调 OCMS 应提供“白盒”可观测性，包括查询如何被解构、检索波如何在同现拓扑图中漫游、动态 Beta 曲线、AgentDream 形成的新逻辑链路等。

建议暴露以下指标：

| 指标                     | 说明           |
| ---------------------- | ------------ |
| recall_latency_ms      | 记忆召回耗时       |
| write_latency_ms       | 记忆写入耗时       |
| compact_queue_depth    | L1 压缩队列长度    |
| dream_jobs_total       | 梦境任务数量       |
| forget_jobs_total      | 遗忘任务数量       |
| graph_activation_depth | 平均 LIF 扩散深度  |
| geo_rerank_gain        | 测地线重排带来的排名提升 |
| memory_pollution_rate  | 被判定为污染的记忆比例  |
| conflict_fact_count    | 冲突事实数量       |
| recall_hit_rate        | 用户问题召回命中率    |

Trace 示例：

```json
{
  "trace_id": "trace_001",
  "query": "帮我继续上次的后端 minimal harness 方案",
  "query_plan": {
    "intent": "continue_project",
    "entities": ["backend", "minimal", "harness"],
    "shotgun_queries": 4
  },
  "retrieval": {
    "vector_hits": 12,
    "graph_hits": 8,
    "tree_hits": 3,
    "final_memories": 6
  },
  "rerank": {
    "beta": 0.31,
    "geo_weight": 0.25,
    "activation_path": ["harness", "backend", "minimal"]
  }
}
```

---

# 9. 可插拔架构：支持任意语言接入

为了支持任意语言，OCMS 应作为独立记忆服务，而不是绑定某个 Agent 框架或语言 SDK。

## 9.1 接入协议

推荐同时提供四种接口：

| 接口        | 用途                        |
| --------- | ------------------------- |
| REST      | 最通用，适合所有语言快速接入            |
| gRPC      | 高性能服务间调用                  |
| WebSocket | 流式记忆事件、实时 Agent           |
| MCP       | 接入支持 MCP 的 Agent/IDE/工具生态 |

核心 API：

```text
POST /v1/memory/events          写入 L0 事件
POST /v1/memory/retain          显式写入长期记忆
POST /v1/memory/recall          检索记忆
POST /v1/memory/forget          遗忘记忆
POST /v1/memory/compact         触发摘要压缩
POST /v1/memory/reflect         触发反思
GET  /v1/memory/entities/{id}   查询实体画像
GET  /v1/memory/trace/{id}      查询检索链路
```

## 9.2 插件点设计

文档提到 OCMS 应支持 Hook 插件化，short_compact 摘要算法、多语言 tokenizer 等都可以热插拔。

推荐 Hook：

```text
BeforeWriteHook
AfterWriteHook
MemoryClassifierHook
EntityExtractorHook
TagExtractorHook
EmbeddingProviderHook
CompactHook
RecallRewriteHook
RerankHook
ForgetPolicyHook
DreamTaskHook
AuditHook
```

插件接口示例：

```typescript
export interface MemoryPlugin {
  name: string;
  version: string;

  beforeWrite?(event: MemoryEvent): Promise<MemoryEvent>;
  extractTags?(event: MemoryEvent): Promise<Tag[]>;
  compact?(events: MemoryEvent[]): Promise<EpisodicSummary>;
  rerank?(query: MemoryQuery, candidates: MemoryCandidate[]): Promise<MemoryCandidate[]>;
  forget?(request: ForgetRequest): Promise<ForgetPlan>;
}
```

## 9.3 SDK 结构

```text
sdks/
  typescript/
  python/
  go/
  java/
  rust/
  csharp/
  kotlin/
  swift/
```

所有 SDK 只做轻封装，核心协议保持一致：

```json
{
  "agent_id": "agent_x",
  "tenant_id": "tenant_y",
  "session_id": "sess_z",
  "query": "继续上次的设计方案",
  "options": {
    "recall_depth": 3,
    "include_l0": true,
    "include_l1": true,
    "include_l2": true,
    "max_tokens": 4000
  }
}
```

---

# 10. 关键技术选型

## 最小可落地版本

适合个人 Agent、本地助手、小团队：

| 模块    | 技术                         |
| ----- | -------------------------- |
| 主服务   | Go                         |
| L0    | JSONL + 内存 RingBuffer      |
| L1/L2 | SQLite WAL + FTS5          |
| 向量    | USearch / sqlite-vec       |
| 图     | Go 内存 Map + SQLite 边表      |
| 后台任务  | Go cron / asynq            |
| 可观测性  | OpenTelemetry + Prometheus |
| API   | REST + SSE                 |
| 插件    | WASM / 子进程插件               |

## 生产增强版本

适合企业、多 Agent、多租户：

| 模块   | 技术                                          |
| ---- | ------------------------------------------- |
| 主服务  | Go / Rust                                   |
| 事实库  | PostgreSQL                                  |
| 搜索   | OpenSearch / PostgreSQL FTS                 |
| 向量库  | Qdrant / Milvus                             |
| 图数据库 | Neo4j / NebulaGraph / PostgreSQL edge table |
| 消息队列 | Kafka / NATS                                |
| 缓存   | Redis                                       |
| 编排   | Kubernetes                                  |
| 可观测性 | OpenTelemetry + Tempo + Grafana             |
| 权限   | OPA / Casbin                                |
| API  | REST + gRPC + MCP                           |

---

# 11. 推荐实施路线

## Phase 1：MVP，先实现可用记忆闭环

目标：能写、能查、能压缩、能回溯。

实现内容：

```text
L0 RingBuffer
L0 JSONL WAL
L1 Summary Tree
L2 Fact 表
基础 Embedding 召回
BM25/FTS5 召回
REST API
```

验收标准：

```text
1. 支持会话恢复；
2. 支持“继续上次任务”；
3. 支持 L1 摘要回溯原文；
4. 支持显式保存/遗忘记忆。
```

## Phase 2：增强检索质量

实现内容：

```text
Shotgun Query
SVD/Gram-Schmidt 去重
tagMemory 动态 Beta
LIF 图扩散
Geodesic Rerank
```

验收标准：

```text
1. 长对话召回准确率显著高于单向量检索；
2. 多义词误召回下降；
3. 召回结果可解释，有 activation path。
```

## Phase 3：潜意识系统

实现内容：

```text
DreamWaveEngine
long_reflect
access-based decay
memory conflict arbitration
SBU forget pipeline
```

验收标准：

```text
1. 自动发现重复实体；
2. 自动降低冲突旧观点置信度；
3. 用户删除记忆后不可再被召回；
4. 低价值记忆自动降权。
```

## Phase 4：多语言与生态接入

实现内容：

```text
MCP Server
gRPC API
TypeScript/Python/Go/Java SDK
Plugin Hook
Dashboard
```

验收标准：

```text
1. 任意语言可写入/检索/遗忘记忆；
2. Agent 框架可通过 MCP 接入；
3. 管理后台可查看检索链路、图扩散和梦境巩固结果。
```

---

# 12. 最终落地形态

建议将 OCMS 定义为一个独立的“Agent Memory Runtime”：

```text
ocms/
  cmd/
    ocms-server/
    ocms-cli/
  internal/
    l0_buffer/
    l1_compactor/
    l2_workspace/
    graph/
    vector/
    tree_index/
    recall/
    tagmemory/
    dream/
    forget/
    observability/
  plugins/
    tokenizer/
    embedder/
    compactor/
    reranker/
  api/
    openapi.yaml
    memory.proto
    mcp_server.ts
  sdks/
    typescript/
    python/
    go/
    java/
  dashboard/
  docs/
```

最小启动命令：

```bash
ocms-server \
  --store sqlite://./ocms.db \
  --event-log ./data/events.jsonl \
  --vector usearch://./data/vector.index \
  --config ./config/ocms.yaml
```

核心定位：

```text
OCMS 不是“更复杂的 RAG”，而是 Agent 的认知操作系统。
L0 负责即时意识；
L1 负责剧集压缩；
L2 负责长期本体；
潜意识层负责反思、巩固和遗忘；
图-树-向量融合负责让记忆既可联想、可推理、又可语义召回；
可插拔协议层负责让任意语言和任意 Agent 都能接入。
```
