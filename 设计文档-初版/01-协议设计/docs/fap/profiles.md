# profiles.md

FAP-1 Profile 重排，避免 HP 默认暗含分布式。

## 1. Profile 序列

```text
Core Lite
  → Core Secure
  → Plugin Runtime
  → Data Plane
  → HP Local
  → HP Cluster / Distributed
  → EXT
```

每个 Profile 独立开启，后一级不能强依赖后续级。

## 2. Profile 定义

### Core Lite

```text
必选。
最小可用协议内核，单机可运行。

含:
  Agent Card
  HTTP/JSON Transport（强制保底）
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
  FAP-Inspector CLI
```

### Core Secure

```text
生产必选。
在 Core Lite 之上补齐安全、审计、版本治理。

含:
  SemVer 协商（双向区间）
  Multisig Receipt（同步或异步补签）
  Hash Chain Checkpoint
  Mandate 委托链 + 撤销
  Replay Cache bloom + 共享
  Transparency Log 对接点
  Plugin Manifest 签名校验
```

### Plugin Runtime

```text
可选。
支持外部 Capability 插件。

含:
  Plugin Lifecycle（STAGED → ACTIVE → DRAINING）
  Rust native / Sidecar / WASM runtime
  Plugin permission sandbox
  capability_id + schema_version 并存
  Plugin Health + Rollback
  plugin_api SemVer
```

### Data Plane

```text
可选。
支持大对象、流式传输。

含:
  ObjectRef + Lease
  DataOpen / DataCommit（走 Envelope 签名）
  DataChunkMeta + raw chunk stream（走 Merkle 分片流，不走 Protobuf bytes 主路径）
  FlowCredit / DataPause / DataResume
  RemoteFileResolver（本地）
  Object GC
  
不含: 跨节点 replica（需 HP Cluster）
```

### HP Local

```text
可选。
单机高性能增强。

含:
  Dynamic Tool Projection
  Context Folding L0~L4
  DAG BatchInvoke
  Execution Scheduler
  Warm Worker Pool
  Semantic Capability Routing
  Local Vector Index
  
不含: 分布式节点
```

### HP Cluster / Distributed

```text
可选。
多节点协作。

含:
  NodeHello / RegisterCapabilities / RemoteInvoke
  NodeHealth FSM + 断路器
  Heartbeat
  Object Replica + 跨节点 RemoteFileResolver
  Capability Router（跨节点）
  
依赖: Data Plane + HP Local
```

### EXT

```text
实验，默认关闭，编译期可剔除。

含:
  DID/VC
  Latent Channel
  Gossip Discovery
  CRDT Workspace
  Federated Learning
  
每个 EXT 独立 feature flag。
```

## 3. 依赖关系

```text
Core Lite (独立)
Core Secure → Core Lite
Plugin Runtime → Core Secure
Data Plane → Core Secure
HP Local → Plugin Runtime
HP Cluster → HP Local + Data Plane
EXT → Core Secure (+ 视特性可选 Plugin Runtime / Data Plane)
```

## 4. Agent Card 声明

Core Lite 是协议层唯一必选 Profile；Core Secure 是生产部署模板的必选项，但不能破坏 Core Lite 在开发、边缘和低功耗场景下独立协商运行。

```json
{
  "advertisedProfiles": [
    { "id": "fap.core-lite", "version": "1.0.0", "required": true },
    { "id": "fap.core-secure", "version": "1.0.0", "required": false, "deploymentRequired": "production" },
    { "id": "fap.plugin-runtime", "version": "1.0.0", "required": false },
    { "id": "fap.data-plane", "version": "1.0.0", "required": false },
    { "id": "fap.hp-local", "version": "1.0.0", "required": false },
    { "id": "fap.hp-cluster", "version": "1.0.0", "required": false, "defaultEnabled": false },
    { "id": "fap.ext", "version": "0.1.0", "required": false, "defaultEnabled": false }
  ]
}
```

Profile ID 采用 kebab-case + `fap.` 前缀。

## 5. Hello 协商规则

```text
客户端声明 supported_profiles: 包含客户端支持的所有 Profile
服务端 selected_profiles: 取两方都支持的最小公共子集
rejected_profiles: 明确声明哪些 Profile 无法启用
  rejection reasons:
    NOT_IMPLEMENTED
    DISABLED_BY_CONFIG
    DEPENDENCY_UNMET
    VERSION_INCOMPATIBLE
    
客户端收到 rejected_profiles 后应:
  若 rejected 包含 Core Lite → 拒绝会话
  若当前 deployment policy 要求 Core Secure，而 Core Secure 被 rejected → 拒绝会话
  若 rejected 仅含可选 Profile → 降级运行
```

## 6. Profile 冲突

```text
HP Local + HP Cluster 不能独启用 HP Cluster 而不启用 HP Local
EXT Latent + Data Plane: Latent 必须能 fallback 到 Data Plane
Plugin Runtime 与 Capability Registry: 若 Plugin Runtime 关闭，Capability Registry 只能承载 Core 内置能力
```

## 7. 向后兼容

```text
Profile 升级:
  Profile MAJOR 变化视为破坏兼容，Hello 协商时可能 rejected
  Profile MINOR 变化: 向后兼容，自动启用新特性

移除 Profile:
  不允许移除已发布 Profile
  可声明 deprecated 但保留 ≥ 2 个 MAJOR 周期
```
