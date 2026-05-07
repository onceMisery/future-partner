# 05 - 会话层

## 1. 状态机

```text
NEW
  → DISCOVERED
  → HELLO_EXCHANGED
  → AUTHENTICATED
  → CAPABILITY_BOUND
  → ACTIVE
  → DRAINING
  → CLOSED
```

异常：AUTH_FAILED / POLICY_REJECTED / EXPIRED / RESUMING / BROKEN。

## 2. 标准流程

```text
Client → GET Agent Card
Client → ClientHello
Server → ServerHello（含 version_commitment）
Client → AuthInit（交给 AuthPlugin）
Server → AuthAck / AuthReject
Client → CapabilityBind
Server → CapabilityAck
Client → InvokeRequest / BatchInvokeRequest / DataOpen
Server → ProgressEvent / InvokeResult / Receipt
```

## 3. 版本协商

```text
ClientHello:
  supported_protocol_range { min, max }
  supported_profiles
  supported_plugin_api_versions

ServerHello:
  selected_protocol_version
  selected_profiles
  rejected_profiles
  selected_plugin_api_version
  compatibility_warnings
  version_commitment (防降级)
```

## 4. 防降级攻击

```text
version_commitment =
  HMAC(session_key,
       SHA256(client_hello.supported_protocol_range
              || client_hello.supported_profiles
              || client_hello.client_nonce))

客户端本地重算并比对，不一致则拒绝。
```

## 5. 会话恢复

```text
客户端断线
  → 重连
  → ResumeRequest(session_id, last_received_sequence_no)
  → 服务端校验会话有效 + 鉴权凭证未过期
  → ResumeAck(replay_from_sequence_no)
  → 重放未确认 ProgressEvent / Result / Receipt
```

## 6. 0-RTT 策略

```text
默认: 禁用 0-RTT
Resume: 允许 0-RTT，Early Data 只接受幂等消息
  InvokeRequest 必须带 idempotency_key
  Replay Cache 命中时返回缓存结果
high/critical: 强制 full handshake，0-RTT 拒绝
```

## 7. Session 模型

```rust
pub struct Session {
    pub session_id: String,
    pub state: SessionState,
    pub subject: Option<String>,
    pub granted_capabilities: Vec<String>,
    pub sequence_no: u64,
    pub hash_chain_tip: Vec<u8>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
    pub expires_at: DateTime<Utc>,
}

pub enum SessionState {
    New,
    HelloExchanged,
    Authenticated,
    CapabilityBound,
    Active,
    Resuming,
    Draining,
    Closed,
}
```

## 8. 多连接一致性

一个 session 可跨多条物理连接（不同 TransportPlugin 可并存）：

```text
session_id 唯一
每个物理连接附带 session_id
消息 sequence_no 严格递增
切换 Transport 时 Resume
```

## 9. 会话超时

```text
默认 session TTL: 30 分钟
可由 Agent Card.limits.maxSessionTTLSeconds 覆盖
DRAINING 窗口: 60 秒，完成在途 Invoke 后 CLOSED
```
