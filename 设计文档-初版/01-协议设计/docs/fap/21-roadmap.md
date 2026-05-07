# 21 - 路线图

> Phase 门禁与依赖关系见 [phase-gates.md](./phase-gates.md)。
> Profile 依赖关系见 [profiles.md](./profiles.md)。
> 测试四门禁与三档性能目标见 [testing-matrix.md](./testing-matrix.md)。

每个 Phase 对应一个 Profile，完成后必须通过门禁才能进入下一阶段。

## Phase 0：协议冻结（1 周）

交付：

```text
proto/fap/v1/fap.proto
proto/fap/v1/plugin_api.proto
proto/fap/v1/ALLOCATIONS.md
schema/agent-card.schema.json
schema/plugin-manifest.schema.json
docs/fap/*.md（全部规范文档）
```

验收：Rust/Java/TS/Python 代码生成通过；Golden Protobuf 样本生成；buf breaking CI 就绪。

## Phase 1：Core Lite MVP（2 周）

只做：

```text
Agent Card
HTTP JSON Gateway
Protobuf Envelope
ClientHello / ServerHello（含 version_commitment）
JWT 或本地 Token
Capability Registry
InvokeRequest / InvokeResult
Basic Receipt（MAIN_NODE 单签）
SQLite WAL
Idempotency
Replay Cache
AuthRevoke
FAP-Inspector CLI 初版
```

明确不做：QUIC / WASM / 分布式节点 / DID/VC / FAP-HP / 潜空间 / Gossip / CRDT。

验收：低功耗设备可运行；单机开发者 10 分钟内启动；fap inspect 能看到完整消息流；重复请求不会重复执行；Receipt hash chain 可查看；version 降级攻击被拒绝；golden corpus 100% 通过。

## Phase 2：Core Hardening（2 周）

```text
SemVer 协商（含 prerelease 策略）
Multisig Receipt（同步模式）
Hash Chain Checkpoint
Mandate 委托链 + 撤销
Replay Cache bloom + 共享
Backpressure HTTP fallback
Plugin Manifest 校验 + 签名验证
FAP-Inspector UI 初版
Interop 矩阵（Rust ↔ Java）
```

## Phase 3：Plugin Runtime（2~3 周）

```text
Rust native plugin
Sidecar plugin (gRPC)
WASM plugin (experimental)
Plugin permission model
Plugin rollback
Plugin hot reload
plugin_api SemVer 检查
```

## Phase 4：QUIC + Data Plane（3 周）

```text
QUIC Control Stream
DataOpen / DataChunk / DataCommit / DataAck
FlowCredit / DataPause / DataResume
ObjectRef TTL + Lease + Renew
Object GC
Transparency Log 对接
Multisig 异步补签 + ReceiptFinalized
```

## Phase 5：FAP-HP（3 周）

```text
Dynamic Tool Projection
Context Folding L0~L4
DAG BatchInvoke
Execution Scheduler
Warm Worker Pool
Semantic Capability Routing
USearch mmap Index
```

## Phase 6：Distributed Node（3 周）

```text
NodeHello
RegisterCapabilities
RemoteInvoke
RemoteFileResolver
Object Replica
NodeHealth FSM + 断路器
Heartbeat
```

## Phase 7：FAP-EXT（持续）

```text
DID/VC Plugin
Latent Channel Plugin
Gossip Discovery Plugin
CRDT Workspace Plugin
Federated Learning Plugin
```

全部默认关闭，可 feature flag 开启，全部有 fallback，全部不能绕过 Core 安全链路。
