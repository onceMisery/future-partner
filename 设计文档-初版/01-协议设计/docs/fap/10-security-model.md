# 10 - 安全模型

详见 [security.md](./security.md)。

## 1. 可插拔安全组件

```text
AuthPlugin:         mTLS+JWT / DID-VC / 企业 IAM / API Key
SignaturePlugin:    ed25519 / ecdsa / rsa-pss / hmac / 国密 SM2
KeyResolverPlugin:  本地 JWKS / 远程 JWKS / X.509 / DID Document / HSM
RevocationPlugin:   OCSP / CRL / 自研撤销表 / Transparency Log
```

## 2. 安全链路

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

## 3. Mandate 委托链

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
  "issuer_signature": { "alg": "ed25519", "kid": "...", "value": "..." }
}
```

委托规则：子 capabilities ⊆ 父 capabilities；子 delegation_depth = 父 + 1。

## 4. 撤销

```proto
message AuthRevoke {
  string session_id = 1;
  string mandate_id = 2;
  string reason = 3;
  uint32 retry_after_ms = 4;
  RevokeScope scope = 5;
}
```

策略：每 InvokeRequest 查撤销表（缓存 TTL ≤ 60s）；每 5 分钟 session-wide sync；high/critical 每次 re-check。

## 5. Replay 防护

```text
校验：message_id / idempotency_key / created_at / expires_at / payload_sha256 / prev_hash / session_id / nonce
Cache：time-windowed bloom + exact cache
多节点：必须共享（Redis/etcd）
```

## 6. 沙箱

- WASM: Wasmtime，资源限制 + 系统调用白名单
- Sidecar: 独立进程，cgroups + seccomp + SELinux/AppArmor
- Rust Native: 仅 Core 内置，无额外沙箱
