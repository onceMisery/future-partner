# 01 - 设计目标与非目标

## 1. 核心目标

| 目标 | 说明 |
|---|---|
| 独立协议 | 不依赖 MCP/A2A/VCP，可桥接但不绑定 |
| 可插拔架构 | 所有扩展点必须是 Plugin（传输/认证/存储/签名/发现/审计/桥接） |
| 轻量化内核 | Core Lite 可在单机、边缘、低功耗设备运行 |
| 高性能 | QUIC 多路复用、DAG 批量、零拷贝数据面 |
| 强安全 | mTLS + JWT + Mandate 委托链 + Replay 防护 |
| 可审计 | 关键调用生成可验证 Multisig Receipt |
| 可演进 | SemVer + 降级防护 + Schema breaking CI |
| 可观测 | OpenTelemetry + FAP-Inspector 工具链 |

## 2. 非目标

```text
不默认启用潜空间通信
不默认启用 Gossip 网络
不默认启用 CRDT 协同
不默认启用全量 DID/VC
不强制所有模型使用 JSON Function Calling
不复制 VCPToolBox 实现结构
不绑定某个具体 Transport / Storage / TLS 库
```

## 3. Core Lite 约束

```text
FAP Core Lite 不依赖 FAP-HP
FAP Core Lite 不依赖 WASM
FAP Core Lite 不依赖 QUIC
FAP Core Lite 不依赖 DID/VC
FAP Core Lite 不依赖分布式节点
FAP Core Lite 必须可退化为 HTTP/JSON + Protobuf + SQLite
```

## 4. 可插拔清单

Core 只定义接口；以下全部是插件：

| 扩展点 | Plugin Trait |
|---|---|
| 传输 | TransportPlugin |
| 认证 | AuthPlugin |
| 签名 | SignaturePlugin |
| 密钥解析 | KeyResolverPlugin |
| 撤销 | RevocationPlugin |
| 存储 | StoragePlugin |
| 向量索引 | 属于 StoragePlugin 子接口 |
| 对象存储 | ObjectStorePlugin |
| 发现 | DiscoveryPlugin |
| 审计 Sink | AuditSinkPlugin |
| 桥接 | BridgePlugin |
| 业务能力 | CapabilityPlugin |
| 上下文 | ContextPlugin |
| 路由 | CapabilityRouterPlugin |
| 节点健康 | NodeHealthPlugin |

## 5. 分层定位

```text
FAP Core:     稳定、安全、强类型、可审计
FAP-HP:       高性能插件执行、动态工具注入、上下文折叠、分布式节点、批量调用
FAP-EXT:      潜空间、Gossip、CRDT、DID/VC、联邦学习
```
