# 00 - FAP-ME 总览

## 定位

FAP-ME（FAP Memory Engine，FME-1）是 **FAP-1 协议的官方记忆扩展实现**，采用**认知内核 + 协议外壳**双层架构：协议层负责通信契约，引擎层负责认知计算。

```text
FAP-ME = 面向 FAP-1 协议的可插拔认知记忆引擎
```

## 与 FAP-1 的职责划分

```text
FAP-1 负责：
  Discovery / Session / Auth / Control Plane / Data Plane / Agent 间通信

FAP-ME 负责：
  记忆分层 / 写入 / 检索 / 共享 / handoff / 遗忘 / 审计 / 梦境整合
```

## 三区架构模型

```text
Kernel Zone（不可替换，Core 唯一实现）：
  PolicyKernel | AuditKernel | TenantKernel | MandateVerifier | ContentSafetyGuard

Plugin Zone（可替换，算法/适配层）：
  EmbeddingProvider | VectorIndex | TagGraph | CompactStrategy
  Reranker | DreamWorker | ForgetEngine | AuditSink

Cognitive Core（Seahorse 风格脑区分工）：
  Thalamus | Cortex | Synapse | Hippocampus | Cerebellum
```

详见 [02-architecture.md](./02-architecture.md) 与 [kernel-contract.md](./kernel-contract.md)。

## 四层记忆模型

```text
L0 Immediate Context     即时上下文：秒~小时级，不污染长期记忆
L1 Episodic Memory       剧集记忆：天~月级，由 L0 折叠产生
L2 Semantic Ontology     语义本体：月~年级，由 L1 提炼，需置信度门槛
L3 Subconscious          潜意识后台层：不参与主路径，做梦境/压缩/遗忘
```

详见 [03-memory-layers.md](./03-memory-layers.md)。

## 七种检索模式

```text
basic            HNSW Top-K              快速回忆
hybrid           BM25 + HNSW + filter    生产默认
tide             Gram-Schmidt + 投影熵   复杂语义、弱信号挖掘
tagmemo          LIF + Tag 共现拓扑      联想式召回
tagmemo_geodesic tagmemo + 测地线重排    多义词消歧
dream            高 hop + 低阈值          后台整理
sbu_safe         检索后强脱敏             敏感场景
```

详见 [08-retrieval-modes.md](./08-retrieval-modes.md)。

## 核心设计原则

| 原则 | 实施 |
|---|---|
| Kernel Contract | 安全判定 / 审计链 / 租户隔离 / Mandate 验证 / 内容安全均在 Kernel，不可外包 |
| 算法扩展点全插件化 | 向量化 / 索引 / 拓扑图 / 折叠 / 精排 / 梦境 / 遗忘 / 审计输出全部 Plugin |
| 插件不参与安全判定 | 插件输入输出经 PolicyKernel 过滤；不能直接访问原始 MemoryUnit |
| 单机/服务端共用契约 | 边缘部署用 SQLite WAL；服务端部署用 PG + Qdrant + Redis + Kafka |
| 协议层与引擎层解耦 | 控制面只传操作意图；数据面只传上下文流；引擎独立计算 |

## 核心差异（相对 VCPToolBox / 传统 RAG）

```text
认知拓扑 + Kernel Contract       vs 单一向量库 + 业务层判权
有向序位 Tag 图 (V7)             vs 无向共现图
Tide + 引力场                    vs 仅余弦相似度
七种检索模式                      vs 单一检索
可验证 RedactionReport           vs 不可验证的脱敏
DreamProposal 状态机             vs 直接写入长期记忆
ContentSafetyGuard 内置          vs 无 Prompt Injection 防护
```

## 关键规范

- [kernel-contract](./kernel-contract.md) Kernel 边界
- [retain-score](./retain-score.md) 分层量化标准
- [scoring-formula](./scoring-formula.md) 检索打分
- [tide-algorithm](./tide-algorithm.md) Tide 完整规格
- [10-security-model](./10-security-model.md) 安全模型
- [11-audit-and-receipt](./11-audit-and-receipt.md) 审计链

## 版本

```text
规范: v1.0.0-rc.1
日期: 2026-05-08
状态: 待协议冻结（依赖 FAP-1 Phase 0 门禁 G0）
```
