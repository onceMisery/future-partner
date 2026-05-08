# 01 - 设计目标与非目标

## 1. 核心目标

| 目标 | 说明 |
|---|---|
| FAP-1 兼容 | 完全兼容 FAP-1 协议层（Discovery/Session/Control/Data） |
| Kernel Contract | 安全判定与审计链不可被插件替换；插件只提供算法或适配 |
| 可插拔架构 | 向量化、索引、拓扑、折叠、精排、梦境、遗忘、审计输出全部 Plugin |
| 认知拓扑能力 | 支持 LIF 脉冲扩散 + V7 有向序位拓扑 + Tide 引力场 + 测地线重排 |
| 分层量化 | retain_score、L0 折叠、L2 写入门槛、final_score 全部公式化、可调参 |
| 跨 Agent 安全 | RedactionReport 可验证签名与执行留痕；HandoffPacket 含 FAP-1 Receipt 与 FME 审计链头 |
| 强遗忘语义 | SBU 同步回流遗忘；HardForget 本地强一致、跨系统最终一致；ForgetReceipt 区分本地提交与全局协调 |
| 部署形态多元 | 单机/边缘（SQLite WAL）→ 服务端/多租户（PG + Qdrant + Redis + Kafka） |
| 可观测 | retrieval_latency / hit_rate / dream_jobs / chain_integrity 全指标 |
| 可演进 | 双索引切换、Schema breaking CI、向后兼容门禁 |

## 2. 非目标

```text
不替代 FAP-1 协议层
不依赖某个具体的 LLM Provider
不强制使用某个具体的向量数据库
不复制 VCPToolBox / Mem0 / Letta 的实现结构
不要求所有部署都使用全部 7 种检索模式
不在 V1 引入 CRDT 多副本同步
不在 V1 引入潜空间通信
不强制启用 DID/VC（仅跨组织时启用）
```

## 3. Core Lite 约束

FAP-ME 必须能在受限环境运行。最小 Profile（边缘 Agent）：

```text
依赖：SQLite WAL + 本地 HNSW + JSON Schema
不依赖：Qdrant / Redis / Kafka / WASM 沙箱
不依赖：DID/VC（使用 mTLS + JWT）
不依赖：分布式节点
检索模式：basic + hybrid（其余可缺省）
```

## 4. Kernel 必留组件（不可插件化）

```text
PolicyKernel              所有记忆操作的唯一授权入口
AuditKernel               哈希链生成与完整性验证
TenantKernel              租户/命名空间强制过滤
MandateVerifier           FAP-1 Mandate + FME constraints 验证
ContentSafetyGuard        Prompt Injection 防护，写入前强制执行
```

详见 [kernel-contract.md](./kernel-contract.md)。

## 5. Plugin 扩展点清单

Core 只定义 Trait；以下全部是 Plugin：

| 扩展点 | Plugin Trait |
|---|---|
| 向量化 | EmbeddingProvider |
| 向量索引 | VectorIndex |
| 标签拓扑 | TagGraph |
| L0→L1 折叠 | CompactStrategy |
| 精排 | Reranker |
| 梦境联想 | DreamWorker（只提议，不写入） |
| 遗忘规划 | ForgetEngine（只规划，不删除） |
| 审计输出 | AuditSink |
| 对象存储 | ObjectStore |

详见 [09-plugin-runtime.md](./09-plugin-runtime.md)。

## 6. 与 FAP-1 协议的分层定位

```text
FAP Core / FAP Core Lite      稳定通信、安全、审计
                              FAP-ME 复用 FAP-1 的 Session/Auth/Receipt 链路

FAP-ME Kernel                 记忆专属安全：SBU、FME Mandate constraints、内容安全
FAP-ME Plugin                 算法可替换层
FAP-ME Cognitive Core         脑区分工实现（Seahorse 风格）
FAP-ME Layers                 L0 / L1 / L2 / L3
```

## 7. 不在 V1 范围

```text
EXT-潜空间记忆                 实验性
EXT-联邦学习记忆共享           实验性
EXT-跨节点 CRDT 记忆同步       FAP-1 EXT 范围
EXT-多模态嵌入                 Plugin 可后续注册，不在 Core 测试矩阵
```
