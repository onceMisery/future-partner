# problem-catalog.md

标准 Problem 错误码目录。

## 1. Problem 消息结构

```proto
message Problem {
  string type = 1;              // URI，稳定错误标识
  string title = 2;             // 简短人类可读
  uint32 status = 3;            // 类 HTTP 状态码
  string detail = 4;            // 详细说明
  string instance = 5;          // message_id / invocation_id
  bool retryable = 6;
  uint32 retry_after_ms = 7;
  map<string, string> data = 8; // 结构化附加信息
}
```

`type` 约定格式：`https://fap.dev/problems/<kebab-case-code>`。

## 2. 标准错误码

### 2.1 版本与协议

| code | type | status | retryable | 说明 |
|---|---|---|---|---|
| VERSION_INCOMPATIBLE | /version-incompatible | 400 | false | Hello 协商失败 |
| WIRE_VERSION_MISMATCH | /wire-version-mismatch | 400 | false | wire_major/minor 不匹配 |
| SCHEMA_UNKNOWN | /schema-unknown | 400 | false | 未知消息类型 |
| SIGNATURE_INVALID | /signature-invalid | 401 | false | 签名校验失败 |
| PAYLOAD_HASH_MISMATCH | /payload-hash-mismatch | 400 | false | payload_sha256 不一致 |

### 2.2 认证授权

| code | status | retryable | 说明 |
|---|---|---|---|
| AUTH_FAILED | 401 | false | 认证失败 |
| AUTH_REVOKED | 401 | false | 凭证已撤销 |
| AUTH_EXPIRED | 401 | false | 凭证过期 |
| MANDATE_REQUIRED | 403 | false | 高风险需 mandate |
| MANDATE_EXPIRED | 403 | false | mandate 过期 |
| MANDATE_INSUFFICIENT_SCOPE | 403 | false | mandate 不覆盖该能力 |
| MANDATE_DELEGATION_TOO_DEEP | 403 | false | 委托深度超限 |
| MANDATE_ISSUER_UNTRUSTED | 403 | false | mandate 签发人不可信 |

### 2.3 能力与调用

| code | status | retryable | 说明 |
|---|---|---|---|
| CAPABILITY_NOT_FOUND | 404 | false | 未注册 |
| CAPABILITY_DISABLED | 503 | true | 临时禁用 |
| CAPABILITY_VERSION_MISMATCH | 400 | false | schema_version 不兼容 |
| RISK_POLICY_DENY | 403 | false | 策略拒绝 |
| INVOCATION_TIMEOUT | 504 | true | 超时 |
| INVOCATION_CANCELLED | 499 | false | 被取消 |
| INVALID_DAG | 400 | false | BatchInvoke DAG 有环或引用不存在 |

### 2.4 幂等与重放

| code | status | retryable | 说明 |
|---|---|---|---|
| REPLAY_DETECTED | 409 | false | message_id 重复 |
| IDEMPOTENCY_CONFLICT | 409 | false | idempotency_key 与不同请求冲突 |
| EXPIRES_AT_OUT_OF_WINDOW | 400 | false | expires_at 缺失或超 max_ttl |

### 2.5 数据面

| code | status | retryable | 说明 |
|---|---|---|---|
| OBJECT_UNAVAILABLE | 503 | true | 所有副本不可达 |
| OBJECT_EXPIRED | 410 | false | ObjectRef 过期 |
| OBJECT_HASH_MISMATCH | 400 | false | sha256 不一致 |
| LEASE_EXPIRED | 410 | false | lease 过期 |
| LEASE_RENEW_DENIED | 403 | false | 续租被拒（超上限） |
| PAYLOAD_TOO_LARGE | 413 | false | 超 maxDataChunkBytes |
| STREAM_CLOSED | 410 | false | 流已关闭 |

### 2.6 审计

| code | status | retryable | 说明 |
|---|---|---|---|
| RECEIPT_CHAIN_BROKEN | 500 | false | hash chain 断裂，节点进入 DEGRADED |
| CHECKPOINT_MISSING | 500 | false | 审计 checkpoint 缺失 |
| SIGNATURE_ROLE_MISSING | 500 | false | required role 未签 |

### 2.7 节点与集群

| code | status | retryable | 说明 |
|---|---|---|---|
| NODE_UNHEALTHY | 503 | true | 节点非健康，已熔断 |
| NODE_QUARANTINED | 503 | true | 节点被隔离 |
| NODE_UNAUTHORIZED | 401 | false | 节点认证失败 |

### 2.8 资源与限流

| code | status | retryable | 说明 |
|---|---|---|---|
| RATE_LIMIT_EXCEEDED | 429 | true | 限流，`retry_after_ms` 必填 |
| QUOTA_EXCEEDED | 429 | false | 配额用尽 |
| BUDGET_EXCEEDED | 402 | false | mandate 预算用尽 |

### 2.9 通用

| code | status | retryable | 说明 |
|---|---|---|---|
| TIMEOUT | 504 | true | 超时 |
| INTERNAL_ERROR | 500 | true | 未分类错误，应有 trace_id |
| NOT_IMPLEMENTED | 501 | false | 方法未实现 |
| BAD_REQUEST | 400 | false | 请求不合法 |

## 3. 使用规则

```text
MUST:
  所有协议拒绝必须返回 Problem
  Problem.type 必须稳定（一旦发布不可变）
  retryable=true 时建议提供 retry_after_ms
  instance 至少含 message_id

SHOULD:
  data 字段提供结构化线索（如冲突的 idempotency_key）

MUST NOT:
  将业务错误用于协议层 Problem
  把 detail 字段当作日志堆栈（放 trace_id 即可）
```

## 4. 扩展

组织私有错误码使用反向域名：

```text
type: https://example.com/fap-problems/custom-domain-violation
```

不得使用与标准错误码相同的 code 名。
