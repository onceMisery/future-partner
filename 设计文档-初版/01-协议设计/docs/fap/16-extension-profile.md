# 16 - 扩展 Profile (FAP-EXT)

全部默认关闭，Core Lite 构建中可编译剔除。

## 1. DID/VC

```text
Phase 4+，AuthPlugin
优先支持 did:web，再支持 did:key
必须绑定 session challenge
必须支持撤销检查
不得替代 Core AuthZ
高风险仍需 Mandate
```

## 2. 潜空间通信

```text
Phase 5+，ExtensionPlugin
必须有 text shadow（人类可读投影）
必须有 human-readable projection
禁用于 high/critical risk capability
必须可回退到 Data Plane
```

## 3. Gossip 发现

```text
Phase 5+，DiscoveryPlugin
只传播 Agent Card 摘要
不传播 token / mandate
必须 TTL + 签名 + 限速
```

## 4. CRDT 工作区

```text
Phase 5+，WorkspacePlugin
只同步协作状态
不承载授权 / 审计 / 执行决策
```

## 5. 编译期剔除

```toml
# Cargo features
[features]
default = ["core-lite"]
core-lite = []
core-full = ["quic", "wasm-plugin"]
distributed = ["core-full"]
ext-did-vc = []
ext-latent = []
ext-gossip = []
ext-crdt = []
```

Core Lite 构建产物禁止链接任何 EXT 代码。
