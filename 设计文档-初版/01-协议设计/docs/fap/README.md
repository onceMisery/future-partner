# FAP-1 规范文档索引

> FAP-1（Future Agent Protocol 1）原生 Agent 通信协议规范
> 版本：v1.0.0-rc.2
> 日期：2026-05-07

本目录为 FAP-1 协议的分章规范。与之对应的整合落地方案见 `../4.FAP-1最终实现方案.md`。

## 核心原则文档（必读）

| 文档 | 主题 |
|---|---|
| [kernel-contract.md](./kernel-contract.md) | **Kernel Contract** — Core 必留组件与插件边界 |
| [profiles.md](./profiles.md) | **Profile 重排** — Core Lite → ... → EXT |
| [version-lines.md](./version-lines.md) | **四条版本线** — protocol / plugin_api / capability schema / agent card schema |

## 章节规范（23 章）

| 序号 | 文档 | 主题 |
|---|---|---|
| 00 | [00-overview.md](./00-overview.md) | 协议总览 |
| 01 | [01-goals-and-non-goals.md](./01-goals-and-non-goals.md) | 目标与非目标 |
| 02 | [02-architecture.md](./02-architecture.md) | 总体架构 |
| 03 | [03-wire-protocol.md](./03-wire-protocol.md) | Wire Protocol |
| 04 | [04-discovery-agent-card.md](./04-discovery-agent-card.md) | 发现层与 Agent Card |
| 05 | [05-session-layer.md](./05-session-layer.md) | 会话层 |
| 06 | [06-control-plane.md](./06-control-plane.md) | 控制面 |
| 07 | [07-data-plane.md](./07-data-plane.md) | 数据面 |
| 08 | [08-capability-model.md](./08-capability-model.md) | 能力模型 |
| 09 | [09-plugin-runtime.md](./09-plugin-runtime.md) | 插件运行时 |
| 10 | [10-security-model.md](./10-security-model.md) | 安全模型 |
| 11 | [11-receipt-audit.md](./11-receipt-audit.md) | Receipt 审计 |
| 12 | [12-high-performance-profile.md](./12-high-performance-profile.md) | 高性能 Profile |
| 13 | [13-distributed-node.md](./13-distributed-node.md) | 分布式节点 |
| 14 | [14-model-adapter.md](./14-model-adapter.md) | 模型侧适配 |
| 15 | [15-mcp-a2a-compatibility.md](./15-mcp-a2a-compatibility.md) | MCP / A2A 兼容 |
| 16 | [16-extension-profile.md](./16-extension-profile.md) | 扩展 Profile |
| 17 | [17-storage-design.md](./17-storage-design.md) | 存储设计 |
| 18 | [18-rust-implementation.md](./18-rust-implementation.md) | Rust 实现 |
| 19 | [19-java-sdk-and-sidecar.md](./19-java-sdk-and-sidecar.md) | Java SDK 与 Sidecar |
| 20 | [20-testing-conformance.md](./20-testing-conformance.md) | 测试与一致性 |
| 21 | [21-roadmap.md](./21-roadmap.md) | 路线图 |
| 22 | [22-rollback.md](./22-rollback.md) | 回滚策略 |

## 专题规范

### 安全与审计

| 文档 | 主题 |
|---|---|
| [signing-canonical.md](./signing-canonical.md) | 签名字节规范化 + 测试向量 |
| [security.md](./security.md) | 安全模型详解 |
| [receipt-multisig.md](./receipt-multisig.md) | Multisig Receipt |
| [risk-execution-paths.md](./risk-execution-paths.md) | 风险分级执行路径 |

### 协议治理

| 文档 | 主题 |
|---|---|
| [kernel-contract.md](./kernel-contract.md) | Kernel Contract 边界 |
| [profiles.md](./profiles.md) | Profile 重排 |
| [version-lines.md](./version-lines.md) | 四条版本线 |
| [semver-policy.md](./semver-policy.md) | SemVer 策略（按版本线细分） |
| [schema-evolution.md](./schema-evolution.md) | Schema 演进规则 |
| [problem-catalog.md](./problem-catalog.md) | 错误码目录 |

### 运行时

| 文档 | 主题 |
|---|---|
| [plugin-lifecycle.md](./plugin-lifecycle.md) | 插件发布模型与版本并存 |
| [data-plane-merkle.md](./data-plane-merkle.md) | Data Plane Merkle 化 |
| [data-backpressure.md](./data-backpressure.md) | 背压 |

### 互操作

| 文档 | 主题 |
|---|---|
| [http-json-binding.md](./http-json-binding.md) | HTTP/JSON Fallback |
| [mcp-bridge-semantics.md](./mcp-bridge-semantics.md) | MCP Bridge 语义 |

### 工程门禁

| 文档 | 主题 |
|---|---|
| [phase-gates.md](./phase-gates.md) | 路线图 Phase 门禁 |
| [testing-matrix.md](./testing-matrix.md) | 测试四门禁 + 三档性能目标 |

## 阅读顺序

- 首次阅读：00 → 01 → 02 → **kernel-contract.md** → **profiles.md** → 04 → 05 → 10 → 11
- 实现者：03 + signing-canonical + plugin-lifecycle + data-plane-merkle + 18/19
- 运维/安全：security + receipt-multisig + risk-execution-paths + phase-gates
- 协议贡献者：version-lines + schema-evolution + phase-gates + testing-matrix

## 版本

```text
规范: v1.0.0-rc.2
日期: 2026-05-07
状态: 待协议冻结（Phase 0 门禁 G0 尚未通过）
```
