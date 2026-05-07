# http-json-binding.md

HTTP/JSON Fallback Binding，作为 TransportPlugin 的一个参考实现规范。

## 1. 定位

HTTP/JSON 是 FAP Core Lite 的**强制保底**传输，保证低门槛部署。
它是 **TransportPlugin 的一种**，与 QUIC TransportPlugin、WebTransport TransportPlugin 并列。

可插拔说明：

```text
同一 Agent 可同时启用多个 TransportPlugin
Agent Card.interfaces 按 priority 声明优先级
客户端可按偏好选择
Core Lite 构建只强制包含 HTTP/JSON TransportPlugin
```

## 2. Endpoint 映射

```text
POST /fap/v1/session/hello          ClientHello → ServerHello
POST /fap/v1/session/auth           AuthInit → AuthAck / AuthReject
POST /fap/v1/session/resume         ResumeRequest → ResumeAck
POST /fap/v1/capability/bind        CapabilityBind → CapabilityAck
GET  /fap/v1/capabilities           列出已绑定能力

POST /fap/v1/invoke                 InvokeRequest → InvokeResult
GET  /fap/v1/invoke/:id             查询状态
POST /fap/v1/invoke/:id/cancel      InvokeCancel
GET  /fap/v1/invoke/:id/stream      SSE 接收 ProgressEvent

POST /fap/v1/batch                  BatchInvokeRequest → BatchInvokeResult

POST /fap/v1/data/upload            multipart，映射 DataOpen+DataChunk+DataCommit
GET  /fap/v1/data/:object_id        流式下载
POST /fap/v1/data/:object_id/lease/renew  ObjectLeaseRenew

GET  /fap/v1/receipts/:id           查询 Receipt
GET  /fap/v1/receipts/chain         查询 hash chain
```

## 3. Header 映射

Envelope.Header 字段 ↔ HTTP Header：

```text
X-FAP-Protocol-Version      ← protocol_version
X-FAP-Message-Id            ← message_id
X-FAP-Session-Id            ← session_id
X-FAP-Sequence-No           ← sequence_no
X-FAP-Idempotency-Key       ← idempotency_key
X-FAP-Source-Agent          ← source_agent_id
X-FAP-Target-Agent          ← target_agent_id
X-FAP-Created-At            ← created_at (ISO8601)
X-FAP-Expires-At            ← expires_at (ISO8601)
X-FAP-Trace-Id              ← trace_id
X-FAP-Parent-Message-Id     ← parent_message_id
X-FAP-Payload-Sha256        ← hex(payload_sha256)
X-FAP-Prev-Hash             ← hex(prev_hash)
X-FAP-Priority              ← CONTROL / AUDIT / DATA
X-FAP-Signatures            ← JSON array of { alg, kid, value_b64 }

X-FAP-Extension-*           ← Header.extensions 前缀映射
```

## 4. Body 编码

JSON 使用 **proto3 canonical JSON**（`google.protobuf.util.JsonFormat`）：

```text
Timestamp → RFC 3339 string "2026-05-07T00:00:00Z"
bytes → base64 standard
enum → SCREAMING_SNAKE_CASE 字符串
int64/uint64 → string（JS 兼容）
Struct → 原生 JSON 对象
```

## 5. 签名

签名规则与 QUIC 路径**完全一致**：

```text
HTTP JSON 只是线格式
签名计算仍基于 Envelope proto deterministic encoding
客户端发送前:
  1. 构造 Envelope
  2. signed_bytes = proto deterministic encode (去 signatures)
  3. sign
  4. JSON-serialize 并发 HTTP 请求
服务端接收后:
  1. JSON → Envelope proto
  2. signed_bytes = proto deterministic encode (去 signatures)
  3. verify
```

**严禁对 JSON 字节直接签名**，会因 JSON 非规范而互操作失败。

## 6. SSE 流

`GET /fap/v1/invoke/:id/stream` 返回 `text/event-stream`：

```text
event: progress
id: <sequence_no>
data: {"progressEvent": {...}}

event: result
id: <sequence_no>
data: {"invokeResult": {...}}

event: pressure
id: <sequence_no>
data: {"flowCredit": {"pressure": "SLOW_DOWN", ...}}

event: heartbeat
id: <sequence_no>
data: {}
```

客户端断线 → 携带 `Last-Event-ID` 重连恢复。

## 7. 背压降级

HTTP 下 FlowCredit 为主要背压手段：

```text
transport_hint = HTTP_PRIMARY
客户端应按 recommended_chunk_bytes 调整 multipart 分片
收到 PAUSE → 暂停上传 retry_after_ms
服务端持久化 session 状态以支持 Resume
```

## 8. multipart 上传

`POST /fap/v1/data/upload`：

```text
multipart/form-data
  part "envelope": JSON（含 DataOpen）
  part "data": bytes 主体
  part "commit": JSON（含 DataCommit，含 sha256）
```

服务端原子提交，返回 ObjectRef（含 lease_id）。

## 9. 可替换性

虽然是强制保底，实现仍是插件：

```text
实现者可替换:
  HTTP 框架（Axum/Actix/Spring/Express）
  JSON 编解码库
  SSE 底层实现
  multipart 解析

只要符合本 binding 的线格式与语义即互操作。
```
