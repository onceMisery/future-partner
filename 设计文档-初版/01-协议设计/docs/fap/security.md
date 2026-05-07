# security.md

FAP-1 安全模型详解。

## 1. 安全原则

```text
1. 组织内默认 mTLS + JWT
2. 高风险动作必须 Intent Mandate
3. 跨组织 DID/VC 为可选 AuthPlugin
4. 所有认证/授权/签名机制都是可插拔 Plugin
5. 插件不可绕过 Core 安全链路
6. 审计凭证不可伪造、不可篡改、可验证
```

## 2. 可插拔视角

```text
AuthPlugin:
  mTLS + JWT（默认）
  DID / VC（EXT）
  企业 IAM（Okta / Azure AD / 自研）
  API Key（遗留兼容）

SignaturePlugin:
  ed25519（默认）
  ecdsa-p256-sha256
  rsa-pss-sha256
  hmac-sha256（仅会话内）
  国密 SM2（可选扩展）

KeyResolverPlugin:
  本地 JWKS
  远程 JWKS
  X.509 证书链
  DID Document
  HSM

RevocationPlugin:
  OCSP
  CRL
  自研撤销表
  Transparency Log
```

Core 只定义接口；具体实现由插件替换。

## 3. 安全校验链

```text
Transport TLS
  → Header Validation
  → Replay Cache
  → Session Check
  → AuthN (AuthPlugin)
  → AuthZ (PolicyPlugin)
  → Mandate Check (含撤销查询)
  → Capability Policy
  → Risk Policy
  → Plugin Sandbox
  → Receipt (SignaturePlugin)
```

## 4. 认证

### 4.1 默认 mTLS + JWT

```text
Transport 层: mTLS（相互证书）
应用层: JWT 承载 subject / scope
JWT 校验:
  iss / aud / exp / nbf / jti
  签名（SignaturePlugin + KeyResolverPlugin）
  撤销（RevocationPlugin）
```

### 4.2 DID/VC (EXT)

见 10-security-model.md 第 11.7 节。

### 4.3 防降级

```text
ServerHello 必须返回:
  version_commitment = HMAC(session_key,
    SHA256(client_hello.supported_protocol_range ||
           client_hello.supported_profiles ||
           client_hello.client_nonce))

客户端本地重算并比对。
中间人篡改 → 拒绝。
```

## 5. 授权

### 5.1 Scope

JWT 或 mandate 声明：

```text
capabilities: ["code.review", "file.read"]
resources:    ["repo:future-partner"]
constraints:  ["no-production", "max-risk:high"]
expires_at
budget_units
```

### 5.2 Intent Mandate 委托链

```json
{
  "mandate_id": "m-001",
  "issuer": "did:web:alice.example",
  "subject": "agent:code-agent",
  "capabilities": ["file.write", "code.patch"],
  "resources": ["repo:future-partner"],
  "constraints": [...],
  "expires_at": "2026-05-07T00:00:00Z",
  "budget_units": 1000,
  "parent_mandate_id": null,
  "delegation_depth": 0,
  "max_delegation_depth": 2,
  "issuer_signature": {
    "alg": "ed25519",
    "kid": "alice-key-2026",
    "value": "<base64>"
  }
}
```

委托规则：

```text
子 capabilities ⊆ 父 capabilities
子 resources    ⊆ 父 resources
子 expires_at   ≤ 父 expires_at
子 delegation_depth = 父 + 1
子 delegation_depth ≤ 父 max_delegation_depth
```

### 5.3 高风险能力

```text
shell.exec
file.write / file.delete
email.send
browser.control
database.mutate
payment.execute
production.deploy
plugin.create / plugin.modify

必须满足:
  requires_mandate = true
  requires_receipt = true
  requires_sandbox = true
```

## 6. 撤销

```proto
message AuthRevoke {
  string session_id = 1;
  string mandate_id = 2;
  string reason = 3;
  uint32 retry_after_ms = 4;
  RevokeScope scope = 5;
}

enum RevokeScope {
  REVOKE_SCOPE_UNSPECIFIED = 0;
  REVOKE_MANDATE = 1;
  REVOKE_SESSION = 2;
  REVOKE_SUBJECT = 3;
}
```

策略：

```text
每 InvokeRequest 查撤销表（缓存 TTL ≤ 60s）
每 5 分钟 session-wide sync
撤销 → session DRAINING → 完成在途 → 拒绝新 Invoke
high/critical 每次 re-check
low/medium 可会话级缓存
```

## 7. Replay 防护

```text
校验维度:
  message_id 未使用
  idempotency_key 未冲突
  created_at 在时钟偏移窗内
  expires_at 未过期且 ≤ session.max_ttl（默认 15min）
  payload_sha256 匹配
  prev_hash 指向链内
  session_id 有效
  nonce 绑定当前 session

Cache 实现:
  time-windowed bloom + exact cache
  精确 cache 保留 max(expires_at) + 5min
  多节点部署必须共享（Redis/etcd）
  未共享时 idempotency 粒度限制到 node-local
  拒绝 expires_at 缺失或 > 15min 的消息
```

## 8. 沙箱

### 8.1 WASM

```text
Wasmtime 或等价运行时
资源限制: memory / cpu / wall_clock / output_bytes
系统调用白名单
网络/文件/subprocess 按 manifest 声明授权
```

### 8.2 Sidecar

```text
独立进程（容器 / systemd-nspawn / gVisor）
通过 gRPC over UDS 通信
cgroups 限制
seccomp 过滤系统调用
SELinux/AppArmor profile
```

### 8.3 Rust Native

```text
仅限 Core 内置插件
审计链路可信
不做额外沙箱（已编译进 binary）
```

## 9. 插件签名

```text
插件 Manifest 含 signature 字段
加载前:
  1. 校验 manifest 签名
  2. 校验 artifact hash
  3. 校验 issuer 在受信列表
  4. 校验 permission 不超权

建议:
  生产环境仅加载受信插件
  开发环境可关闭签名校验但记 warn
```

## 10. 审计

```text
Receipt Multisig
Hash Chain Genesis + Checkpoint
Transparency Log Anchor
详见 receipt-multisig.md
```

## 11. 0-RTT 策略

```text
默认: 禁用 0-RTT
Resume: 允许 0-RTT，Early Data 只接受幂等消息
  InvokeRequest 必须带 idempotency_key
  Replay Cache 命中时返回缓存结果
high/critical: 强制 full handshake
```

## 12. 密钥管理

```text
KeyResolverPlugin 实现:
  本地 JWKS (JSON 文件)
  远程 JWKS (HTTPS)
  X.509 (PEM)
  HSM (PKCS#11)
  云 KMS (AWS/GCP/Azure)
  DID Resolver

密钥轮换:
  kid 必填
  轮换触发 auth_context 失效
  老 kid 保留至少 2 × max_session_ttl
```

## 13. 威胁模型速查

| 威胁 | 缓解 |
|---|---|
| 中间人降级 | version_commitment |
| 消息重放 | replay cache + idempotency |
| 签名伪造 | canonical signing + kid 信任链 |
| mandate 滥用 | 委托链 + 撤销 |
| 插件越权 | manifest permission + sandbox |
| DID resolver DoS | 缓存 + 异步预取 + soft-fail 低风险 |
| 审计中心篡改 | hash chain + checkpoint + transparency log |
| 节点失联 object 丢失 | replica + lease + availability check |
| 大流量淹没 Control | priority queue + backpressure |
| 0-RTT 重放 | 仅幂等消息 + replay cache |
