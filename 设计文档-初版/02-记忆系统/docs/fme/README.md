# FAP-ME / FME-1 规范文档索引

> FME-1（FAP Memory Engine 1）— FAP-1 协议官方记忆扩展引擎规范
> 版本：v1.0.0-rc.1
> 日期：2026-05-08

本目录为 FAP-ME 的分章规范，与 FAP-1 协议规范（`../../../01-协议设计/docs/fap/`）对应。整合落地方案见 `../../2-记忆系统设计最终版.md`，演进与差距分析见 `../../3-记忆系统深度分析与未来方案.md`。

## 核心原则文档（必读）

| 文档 | 主题 |
|---|---|
| [kernel-contract.md](./kernel-contract.md) | **FME Kernel Contract** — Kernel 必留组件与插件边界 |
| [fap1-binding.md](./fap1-binding.md) | **FAP-1 绑定规范** — capability、Mandate constraints、Receipt 绑定 |
| [signing-canonical.md](./signing-canonical.md) | **签名 canonical encoding 总锚** — AuditEvent / ForgetReceipt / RedactionReport / Grant / Handoff / Dream 的字段顺序与 hash/签名规则 |
| [outbox-reconciliation.md](./outbox-reconciliation.md) | **跨系统一致性** — HardForget outbox、mutation ledger、reconciler、ADVISORY_DELIVERED |
| [chain-state-concurrency.md](./chain-state-concurrency.md) | **chain_state 并发** — CAS、隔离模式、跨 schema 完整性验证 |
| [retain-score.md](./retain-score.md) | **分层量化标准** — retain_score / L0 折叠 / L2 写入门槛 |
| [scoring-formula.md](./scoring-formula.md) | **检索打分公式** — final_score 与各模式权重矩阵 |

## 章节规范（19 章）

| 序号 | 文档 | 主题 |
|---|---|---|
| 00 | [00-overview.md](./00-overview.md) | FAP-ME 总览 |
| 01 | [01-goals-and-non-goals.md](./01-goals-and-non-goals.md) | 设计目标与非目标 |
| 02 | [02-architecture.md](./02-architecture.md) | 总体架构（Kernel / Plugin / Cognitive Core 三区） |
| 03 | [03-memory-layers.md](./03-memory-layers.md) | 记忆分层模型（L0 / L1 / L2 / L3） |
| 04 | [04-cognitive-core.md](./04-cognitive-core.md) | Seahorse 风格认知内核（脑区分工） |
| 05 | [05-data-model.md](./05-data-model.md) | 数据模型（MemoryUnit / Episode / Tag / Version） |
| 06 | [06-control-plane.md](./06-control-plane.md) | 控制面 API |
| 07 | [07-data-plane.md](./07-data-plane.md) | 数据面流式记忆 |
| 08 | [08-retrieval-modes.md](./08-retrieval-modes.md) | 检索模式与流水线 |
| 09 | [09-plugin-runtime.md](./09-plugin-runtime.md) | 插件运行时 |
| 10 | [10-security-model.md](./10-security-model.md) | 安全模型 |
| 11 | [11-audit-and-receipt.md](./11-audit-and-receipt.md) | 审计与凭证 |
| 12 | [12-context-sharing.md](./12-context-sharing.md) | 上下文共享与 Handoff |
| 13 | [13-storage-design.md](./13-storage-design.md) | 存储设计 |
| 14 | [14-deployment-profiles.md](./14-deployment-profiles.md) | 部署形态 |
| 15 | [15-fap1-integration.md](./15-fap1-integration.md) | FAP-1 协议集成 |
| 16 | [16-rust-implementation.md](./16-rust-implementation.md) | Rust 实现与目录结构 |
| 17 | [17-testing-conformance.md](./17-testing-conformance.md) | 测试与一致性门禁 |
| 18 | [18-roadmap.md](./18-roadmap.md) | 实施路线图 |

## 专题规范

### 算法与认知内核

| 文档 | 主题 |
|---|---|
| [tide-algorithm.md](./tide-algorithm.md) | Tide 算法（Gram-Schmidt + 投影熵 + 引力场） |
| [lif-spike.md](./lif-spike.md) | LIF 脉冲扩散 |
| [geodesic-rerank.md](./geodesic-rerank.md) | 测地线重排（V8） |
| [tag-graph-v7.md](./tag-graph-v7.md) | TagGraph V7 有向序位拓扑 + 冷热分离 |
| [scoring-formula.md](./scoring-formula.md) | final_score 打分公式与权重矩阵 |
| [retain-score.md](./retain-score.md) | retain_score / L0 折叠 / L2 门槛量化 |

### 安全与治理

| 文档 | 主题 |
|---|---|
| [kernel-contract.md](./kernel-contract.md) | FME Kernel Contract |
| [signing-canonical.md](./signing-canonical.md) | 签名 canonical encoding 总锚 |
| [chain-state-concurrency.md](./chain-state-concurrency.md) | 审计链并发与跨隔离模式 verify |
| [content-safety.md](./content-safety.md) | 内容安全：Prompt Injection 防护 |
| [redaction-policy.md](./redaction-policy.md) | 脱敏策略与 RedactionReport（含 SBU 三层过滤分布） |
| [purpose-vocabulary.md](./purpose-vocabulary.md) | Purpose 标准化词汇表 |
| [multi-tenant.md](./multi-tenant.md) | 多租户隔离与插件配置 |
| [forget-engine.md](./forget-engine.md) | 遗忘引擎与 SBU 强制遗忘 |
| [outbox-reconciliation.md](./outbox-reconciliation.md) | 跨系统遗忘协调与最终一致 |
| [dream-state-machine.md](./dream-state-machine.md) | 梦境提案审批状态机 |

### 运行时

| 文档 | 主题 |
|---|---|
| [retrieval-fallback.md](./retrieval-fallback.md) | 检索降级链与背压 |
| [fap1-binding.md](./fap1-binding.md) | FAP-1 capability / Mandate / Receipt 绑定 |
| [problem-catalog.md](./problem-catalog.md) | 错误码目录 |

## 阅读顺序

- 首次阅读：00 → 01 → 02 → **fap1-binding.md** → **kernel-contract.md** → **signing-canonical.md** → 03 → 04 → 10 → 11
- 实现者：02 + 09 + 16 + signing-canonical + chain-state-concurrency + tide-algorithm + lif-spike + tag-graph-v7
- 运维/安全：10 + 11 + content-safety + redaction-policy + purpose-vocabulary + outbox-reconciliation + chain-state-concurrency
- 协议集成：fap1-binding + signing-canonical + 15 + 06 + 07 + 12
- 算法贡献者：tide-algorithm + lif-spike + geodesic-rerank + scoring-formula

## 与 FAP-1 协议的关系

```text
FAP-1 协议层      负责：身份、会话、授权、控制面、数据面、Agent 间通信
                  对应：../../../01-协议设计/docs/fap/

FAP-ME 引擎层     负责：记忆分层、写入、检索、共享、handoff、遗忘、审计、梦境整合
                  对应：本目录 docs/fme/

FAP-ME 是 FAP-1 的标准记忆扩展实现，遵循 FAP-1 的 Kernel Contract 原则：
  - 安全判定不外包给插件
  - 审计链不可绕过
  - 所有算法扩展点必须是 Plugin
```

## 版本

```text
规范: v1.0.0-rc.1
日期: 2026-05-08
状态: 待协议冻结（依赖 FAP-1 Phase 0 门禁 G0）
```
