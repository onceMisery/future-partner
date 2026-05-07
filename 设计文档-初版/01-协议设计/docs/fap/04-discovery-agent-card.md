# 04 - 发现层与 Agent Card

## 1. 发现

### 1.1 默认发现（HTTPS）

```text
GET /.well-known/agent-card.json
```

### 1.2 DiscoveryPlugin

发现机制本身可插拔：

| Plugin | 说明 |
|---|---|
| WellKnownHttpDiscoveryPlugin | 默认，HTTPS well-known |
| DnsDiscoveryPlugin | DNS TXT 记录 |
| GossipDiscoveryPlugin | EXT，默认关闭 |
| RegistryDiscoveryPlugin | 中心化注册中心 |
| MdnsDiscoveryPlugin | 局域网 mDNS |

## 2. Agent Card 结构

```json
{
  "name": "future-agent-runtime",
  "version": "1.0.0",
  "description": "FAP-1 compatible agent runtime",
  "protocol": {
    "name": "fap",
    "version": "1.0.0",
    "minCompatibleVersion": "1.0.0",
    "compatibilityPolicy": "semver",
    "acceptPrereleases": "same-major"
  },
  "runtime": {
    "name": "fap-rust-runtime",
    "version": "0.1.0",
    "features": {
      "has_quic": true,
      "has_wasm_plugin": true,
      "has_distributed_node": false,
      "has_did_vc": false,
      "has_latent_channel": false
    }
  },
  "interfaces": [
    {
      "type": "fap.quic.proto",
      "endpoint": "https://agent.example.com/fap/v1/session",
      "priority": 1,
      "version": "1.0.0"
    },
    {
      "type": "fap.https.json",
      "endpoint": "https://agent.example.com/fap/v1/json",
      "priority": 2,
      "version": "1.0.0"
    }
  ],
  "advertisedProfiles": [
    { "id": "fap.core", "version": "1.0.0", "required": true },
    { "id": "fap.hp", "version": "0.1.0", "required": false, "defaultEnabled": true },
    { "id": "fap.ext", "version": "0.1.0", "required": false, "defaultEnabled": false }
  ],
  "security": [
    { "type": "mtls-jwt", "default": true },
    { "type": "did-vc", "default": false, "profile": "fap.ext" }
  ],
  "limits": {
    "maxControlFrameBytes": 1048576,
    "maxDataChunkBytes": 4194304,
    "maxConcurrentStreams": 128,
    "maxBatchSize": 256,
    "maxEnvelopeTTLSeconds": 900
  },
  "capabilitiesEndpoint": "/fap/v1/capabilities",
  "pluginApi": {
    "version": "1.0.0",
    "minCompatibleVersion": "1.0.0"
  },
  "cardSignature": {
    "alg": "ed25519",
    "kid": "key-agent-2026",
    "value": "<base64>"
  }
}
```

## 3. 校验

```text
1. 取回 agent-card.json
2. 若 cardSignature 存在，校验签名（kid 指向已信任 issuer）
3. protocol.major == client.protocol.major
4. protocol.version >= client.minCompatibleVersion
5. required profile 必须都能满足
6. optional profile 可降级关闭
7. pluginApi.major 必须一致
```

## 4. Card 签名

```text
cardSignature.signed_bytes = deterministic JSON encoding
  (去掉 cardSignature 字段本身)
  UTF-8 NFC 归一化
  object keys 字典序
  无多余空白
签名者为 Agent 自己或 Agent Card 颁发机构
```

注意：Agent Card 是 JSON 场景的例外，签名对 canonical JSON 做；但一旦进入 FAP Envelope（例如远程 Card fetch 的 InvokeRequest），签名按 proto canonical。

## 5. 生成流程

```text
启动服务
  → 加载配置
  → 加载插件 manifest
  → 汇总 capability
  → 汇总 transport（枚举已注册 TransportPlugin）
  → 汇总 security scheme（枚举 AuthPlugin）
  → 生成 Agent Card
  → 可选签名
  → 暴露 /.well-known/agent-card.json
```

## 6. 动态性

Agent Card 应是"动态生成"而非静态文件；插件的启用/禁用、能力的注册/下线都应即时反映到 Card。可设置 ETag / Last-Modified 支持缓存。
