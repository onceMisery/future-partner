# FAP-1 规范文档索引

> FAP-1（Future Agent Protocol 1）原生 Agent 通信协议规范
> 版本：v1.0.0-rc.1
> 日期：2026-05-07

本目录为 FAP-1 协议的分章规范。与之对应的整合落地方案见 `../4.FAP-1最终实现方案.md`。

## 章节规范（22 章）

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

## 专题规范（9 篇）

| 文档 | 主题 |
|---|---|
| [signing-canonical.md](./signing-canonical.md) | 签名字节规范化规则 + 测试向量 |
| [problem-catalog.md](./problem-catalog.md) | 标准错误码目录 |
| [semver-policy.md](./semver-policy.md) | SemVer 策略 |
| [schema-evolution.md](./schema-evolution.md) | Proto Schema 演进规则 |
| [http-json-binding.md](./http-json-binding.md) | HTTP/JSON Fallback Binding |
| [mcp-bridge-semantics.md](./mcp-bridge-semantics.md) | MCP Bridge 语义映射 |
| [data-backpressure.md](./data-backpressure.md) | 数据面背压机制 |
| [receipt-multisig.md](./receipt-multisig.md) | Multisig Receipt |
| [security.md](./security.md) | 安全模型详解 |

## 阅读顺序建议

- 首次阅读：00 → 01 → 02 → 04 → 05 → 06 → 08 → 10 → 11
- 实现者：03 (wire) + 专题 9 篇 + 18/19
- 运维/安全：10/11/security.md/receipt-multisig.md
- 协议贡献者：schema-evolution.md + semver-policy.md + 20/21/22
