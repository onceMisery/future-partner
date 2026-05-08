# 03 - 记忆分层模型

> 量化触发与门槛见 [retain-score.md](./retain-score.md)；折叠策略实现见 [09-plugin-runtime.md](./09-plugin-runtime.md) 中的 `CompactStrategy`。

## 1. 四层模型总览

```text
L0  Immediate Context       即时上下文        秒~小时        ring buffer + WAL
L1  Episodic Memory         剧集记忆          天~月          SQLite/PG + HNSW
L2  Semantic Ontology       语义本体          月~年          fact + tag graph + vector
L3  Subconscious            潜意识后台         不参与主路径    后台任务调度器
```

跃迁方向：

```text
L0 ──fold──→ L1 ──extract──→ L2 ──forget/decay──→ tombstone
                  ↑                ↑
              CompactStrategy   ConfidenceScoring
```

L3 不存储新内容，只对 L0/L1/L2 做后台维护：梦境整合、压缩、重索引、遗忘扫描、拓扑修复。

`CoreMemory` 不是第五层，也不是 L4。若需要表达“高保留价值的长期事实”，应作为 L2/L3 相关策略标签或 retention tier 表达，例如 `core_memory = true`、`retention_tier = protected`，不能改变 L0-L3 的分层模型。

---

## 2. L0 - Immediate Context（即时上下文）

### 职责

```text
保存当前 session / turn / tool call / streaming result / 临时推理状态
```

### 特征

| 项 | 设计 |
|---|---|
| 生命周期 | 秒~小时 |
| 写入方式 | append-only |
| 存储 | 内存 ring buffer + SQLite WAL（断电不丢） |
| 是否进入长期记忆 | 默认不进入 |
| 典型内容 | 用户当前问题、工具返回、代码 diff、临时约束、handoff 现场 |
| 控制面操作 | `AppendContext` `CreateSnapshot` `ReadContextWindow` |
| 安全策略 | 默认私有；handoff 时必须创建 ContextGrant |

### 折叠触发

折叠由 `CompactStrategy` 插件决定，但触发优先级在 Kernel 内固定：

```text
ForceFold        token 超过阈值的 2 倍，立刻折叠
Fold             轮次达到 TURN_THRESHOLD（默认 10）
FoldAtBoundary   semantic_boundary_score > 0.70
FoldOnTimeout    silence_secs > 300
Continue         其余
```

完整公式见 [retain-score.md](./retain-score.md)。

### 设计目标

> 保证 Agent 当前工作现场不会丢，但不污染长期记忆。

---

## 3. L1 - Episodic Memory（剧集记忆）

### 职责

```text
把 L0 多轮上下文折叠成任务级、对话级、事件级 Episode
```

### 特征

| 项 | 设计 |
|---|---|
| 生命周期 | 天~月 |
| 写入方式 | L0 fold 后提交 |
| 存储 | SQLite/PostgreSQL + chunk table + HNSW vector |
| 典型内容 | "完成了某次排障"、"某次 PR 修改过程"、"多 Agent 协作记录" |
| 控制面操作 | `StoreEpisode` `FoldEpisode` `UpdateEpisode` `ShareEpisode` |
| 检索方式 | BM25 + Vector + Tag 过滤 |
| 安全策略 | namespace ACL + FAP-1 Mandate constraints |

### 在 Handoff 中的角色

L1 是多 Agent 协作的核心。Handoff 传递的不是完整 L0，而是：

```text
L0 当前现场（snapshot 引用） + L1 相关 Episode 摘要 + L2 关键事实引用
```

详见 [12-context-sharing.md](./12-context-sharing.md)。

---

## 4. L2 - Semantic Ontology（长期语义本体）

### 职责

```text
保存长期事实、偏好、项目知识、实体关系、Tag 拓扑、抽象经验
```

### 特征

| 项 | 设计 |
|---|---|
| 生命周期 | 月~年 |
| 写入方式 | L1 提炼、用户确认、策略自动合并 |
| 存储 | semantic facts + tag graph + vector index |
| 典型内容 | 用户偏好、项目架构、API 约束、团队规范、长期事实 |
| 控制面操作 | `UpsertFact` `MergeOntology` `RetrieveMemory` `ForgetMemory` |
| 检索方式 | HNSW + LIF + Tag 共现拓扑 + 重排 |
| 安全策略 | 强 ACL；SBU 禁止自动提升 |

### 写入门槛（量化）

L2 不能被低质量对话直接污染。写入条件：

```text
stability        > 0.6
reusability      > 0.5
source_confidence > 0.7
NOT is_ephemeral_chat
NOT has_active_contradiction
```

公式与冲突仲裁见 [retain-score.md](./retain-score.md)。

### 写入流水线

```text
L1 Episode → Semantic Extractor → Confidence Scoring
           → Policy Check → Conflict Resolver → L2 Merge
```

---

## 5. L3 - Subconscious（潜意识后台层）

### 职责

```text
非实时记忆整理：梦境整合、压缩、去重、遗忘、重索引、拓扑修复
```

### 特征

| 项 | 设计 |
|---|---|
| 生命周期 | 后台任务（每个任务有自己的 TTL） |
| 是否参与主路径 | **不直接参与** |
| 触发方式 | 定时（cron） / 阈值（token、graph 健康分） / 用户手动 / 管理员审批 |
| 典型任务 | DreamRun / CompressEpisode / RebuildIndex / PruneTagGraph / ForgetSweep |
| 安全策略 | SBU 默认不可进入梦境；高风险操作需审批 |

### 与 Cerebellum 的关系

L3 的任务由 Cognitive Core 中的 Cerebellum 调度执行：

```text
L3 任务声明（"做什么"） ← Cerebellum 调度（"什么时候、怎么做"）
                       ← DreamWorker / ForgetEngine 插件提供算法
                       ← Kernel 执行实际 mutations
```

详见 [04-cognitive-core.md](./04-cognitive-core.md)、[dream-state-machine.md](./dream-state-machine.md)。

---

## 6. 跨层访问规则

| 操作 | L0 | L1 | L2 | L3 |
|---|---|---|---|---|
| 自由读取 | 同 session | namespace ACL | namespace ACL | 仅 admin |
| 自由写入 | append-only | extractor 写入 | 需置信度门槛 | 不直接写入 |
| Handoff | 必须 grant | 摘要 share | 引用 share | ❌ |
| Forget | 立刻清除 | tombstone + cascade | tombstone + cascade | 自动审计 |
| 检索默认 | 仅 session 内 | tenant 内 | tenant 内 | ❌ |

## 7. 跨层数据世系（Lineage）

每条 L1/L2 记忆必须保留 `lineage_parent_ids`：

```text
L1.lineage_parent_ids = [L0_event_id_1, L0_event_id_2, ...]
L2.lineage_parent_ids = [L1_episode_id_1, L1_episode_id_2, ...]
```

用于：

- HardForget 的级联追溯（删除 L0 时下游全部级联）
- 审计追溯（哪些上游证据导致了某条 L2 fact）
- DreamWorker 推荐时的解释性输出
