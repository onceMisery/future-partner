# signing-canonical.md

签名字节规范化规则与测试向量。

## 1. 目标

确保跨实现（Rust / Java / TS / Python）签名字节一致；任何 Envelope 字段被中途篡改，签名校验失败。

## 2. 签名字节定义

```text
signed_bytes = protobuf_deterministic_encode(EnvelopeSigningView)

EnvelopeSigningView = Envelope 去掉 signatures 字段
                   = Envelope { header, body }
```

## 3. 规范化要求

### 3.1 Protobuf 编码

必须使用 **proto3 deterministic serialization**（参考 [Protobuf Canonical Form](https://protobuf.dev/programming-guides/serialization-not-canonical/)）：

```text
MUST:
  字段按 tag 升序编码
  repeated 字段按原顺序编码
  map 字段必须按 key 字典序（UTF-8 字节序）
  未知字段必须丢弃（不参与签名）
  packed repeated 按 packed 编码
  嵌套 message 递归使用 deterministic encoding

MUST NOT:
  使用 JSON 作为签名载体
  允许字段顺序不一致
```

### 3.2 类型细节

| 类型 | 规则 |
|---|---|
| `google.protobuf.Timestamp` | seconds + nanos，nanos 必须 [0, 999_999_999] |
| `google.protobuf.Struct` | 内部 field `fields: map<string, Value>` 按 key 排序 |
| `bytes` | 原字节，长度前缀由 varint 编码 |
| `string` | UTF-8 NFC 归一化 |
| `map<string, string>` | 按 key 字典序 |
| `Header.extensions` | 同 map 规则 |

### 3.3 签名字段自身处理

`signatures` 字段永远**不参与**签名计算。多方签名时：

```text
签名者按 required_signers 顺序依次签名
每个签名者独立计算 signed_bytes
最终 signed_bytes 对所有签名者相同（因为都去掉 signatures）
```

## 4. 签名算法

默认支持：

| alg | 说明 |
|---|---|
| `ed25519` | 推荐，小签名 |
| `ecdsa-p256-sha256` | 兼容 X.509 |
| `rsa-pss-sha256` | 兼容老 PKI |
| `hmac-sha256` | 仅限 session 内对称场景 |

`alg` 字段用小写连字符形式，不接受简写。

## 5. 签名流程

```text
1. 构造 Envelope（含所有业务字段）
2. 清空 signatures 字段
3. deterministic encode → signed_bytes
4. sig = sign(private_key, signed_bytes)
5. 把 Signature { alg, kid, value=sig } 追加到 signatures
6. 若多方签名，下一签名者重复步骤 2-5（不含已追加的签名）
```

验证流程：

```text
1. 解码 Envelope
2. 取出 signatures 列表，清空 Envelope.signatures
3. deterministic encode → signed_bytes
4. 逐个 Signature 按 kid 查公钥，verify(pub_key, signed_bytes, sig)
5. 任一失败 → 整个 Envelope 拒绝
```

## 6. 测试向量

所有实现必须通过以下向量。字节以 hex 表示。

### 6.1 向量 1：空 Invoke（仅 Header）

```text
Envelope:
  header:
    protocol_version: "1.0.0"
    message_id: "01HXQWE1ZZT0TESTMSG00001"
    session_id: "sess-test-0001"
    sequence_no: 1
    source_agent_id: "agent:alice"
    target_agent_id: "agent:bob"
    created_at: { seconds: 1714906800, nanos: 0 }
    expires_at: { seconds: 1714906900, nanos: 0 }
    wire_major: 1
    wire_minor: 0
    priority: PRIORITY_CONTROL
  body: (none)
  signatures: []

signed_bytes (hex):
  <由实现生成后录入 golden corpus；CI 校验跨实现一致>

alg: ed25519
private_key (hex): 9d61b19deffd5a60ba844af492ec2cc44449c5697b326919703bac031cae7f60
expected_signature (hex):
  <由 ref 实现计算；验证器校验>
```

### 6.2 向量 2：InvokeRequest（含 Struct）

```text
body.invoke_request:
  invocation_id: "inv-0001"
  capability_id: "code.review"
  input: Struct { "path": "src/main.rs", "mode": "strict" }
  timeout_ms: 30000
  declared_risk_level: RISK_MEDIUM
  mandate_id: "m-001"
  options: {}
```

关键校验：`Struct` 内 `fields` map 必须按 key 字典序 `"mode"` 在 `"path"` 之前。

### 6.3 向量 3：多签 Receipt

```text
Receipt:
  receipt_id: "rcpt-0001"
  invocation_id: "inv-0001"
  subject: "agent:code-agent"
  capability_id: "code.review"
  status: "SUCCESS"
  request_hash: <32 bytes>
  response_hash: <32 bytes>
  transcript_hash: <32 bytes>
  previous_receipt_hash: <32 bytes>
  finality: RECEIPT_FINALIZED
  signatures:
    [
      { role: MAIN_NODE, alg: ed25519, kid: main-key-2026, value: ... },
      { role: EXECUTOR_NODE, alg: ed25519, kid: exec-key-2026, value: ... },
      { role: AUDIT_CENTER, alg: ed25519, kid: audit-key-2026, value: ... }
    ]

签名顺序:
  1. MAIN_NODE 先签 Receipt（signatures=[]）
  2. MAIN 的签名 push 到 signatures
  3. EXECUTOR_NODE 对 Receipt（signatures=[MAIN 的]）？否 —— 错！
     EXECUTOR 也对 signatures=[] 签
  4. AUDIT_CENTER 对 signatures=[] 签

  最终 signatures = [MAIN, EXECUTOR, AUDIT]
  所有签名者都对同一份 signed_bytes 做签名。
```

### 6.4 向量 4：map<string, string> 排序

```text
Header.extensions:
  "com.example.b" = "2"
  "com.example.a" = "1"

编码时必须按 key 字典序:
  "com.example.a" 先编码
  "com.example.b" 后编码
```

## 7. golden corpus 格式

```text
conformance/golden/signing/
  vector-001-empty-invoke/
    envelope.json      <- 人类可读
    envelope.bin       <- proto binary
    signed_bytes.bin   <- 规范化编码
    private_key.hex
    expected_signature.hex
    description.md
```

## 8. CI 要求

```text
每次 proto 变更:
  1. 重新生成 signed_bytes.bin
  2. 对比旧版 signed_bytes.bin
  3. 若 signed_bytes 变化而 proto tag 未变 → 拒绝合并
  4. 所有 SDK 实现跑 vector，一致才通过
```
