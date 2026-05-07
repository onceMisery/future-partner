# 00 - FAP-1 协议总览

## 定位

FAP-1（Future Agent Protocol 1）是面向 Agent-to-Agent / Agent-to-Tool / Agent-to-Memory 的原生通信协议，采用**可插拔架构**：传输、认证、签名、存储、发现、审计、桥接等所有扩展点都是 Plugin，Core 只定义接口与契约。

## 三层协议模型

```text
FAP Core:           稳定通信、安全、会话、能力、数据、审计
                    最小形态 Core Lite，可退化为 HTTP/JSON + Protobuf + SQLite

FAP-HP Local:       动态工具注入、上下文折叠、DAG 批量、本地调度
                    推荐按需启用，全部由 Plugin 提供

FAP-HP Cluster:     分布式节点、跨节点对象副本、远程能力路由
                    可选启用，依赖 HP Local + Data Plane

FAP-EXT Profile:    DID/VC、潜空间、Gossip、CRDT、联邦学习
                    实验能力，默认关闭，编译期可剔除
```

## 四平面

```text
Discovery Plane:   Agent Card、能力目录
Session Plane:     Hello、版本协商、Auth、Resume
Invocation Plane:  Control + Data + Audit 三股流共享通道
Extension Plane:   EXT 特性
```

Invocation Plane 内消息按 Priority 出队：Control > Audit > Data。

## 核心差异（相对 VCPToolBox）

```text
可插拔传输         vs 单一 WebSocket
插件运行时三模态   vs Node.js 单一
Multisig Receipt   vs 无/弱审计
Mandate 委托链     vs API Key
DAG 批量调用       vs 异步并行
ObjectRef + Lease  vs Base64 回源
SemVer + 降级防护  vs 无版本策略
EXT 可编译剔除     vs 混在核心
```

## 关键规范

- [signing-canonical](./signing-canonical.md) 签名字节规则
- [semver-policy](./semver-policy.md) 版本治理
- [receipt-multisig](./receipt-multisig.md) 审计凭证
- [security](./security.md) 安全模型
- [data-backpressure](./data-backpressure.md) 背压
- [problem-catalog](./problem-catalog.md) 错误码
- [schema-evolution](./schema-evolution.md) 演进
- [http-json-binding](./http-json-binding.md) HTTP Fallback
- [mcp-bridge-semantics](./mcp-bridge-semantics.md) MCP 映射

## 版本

```text
规范: v1.0.0-rc.1
日期: 2026-05-07
状态: 待协议冻结
```
