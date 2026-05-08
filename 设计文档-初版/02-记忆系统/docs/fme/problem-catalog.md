# problem-catalog.md

> FAP-ME 错误码目录。格式遵循 FAP-1 [problem-catalog.md](../../../01-协议设计/docs/fap/problem-catalog.md)。

## 1. 错误格式

```json
{
  "type": "fap-me/<category>-<reason>",
  "title": "Human readable title",
  "status": 4xx | 5xx,
  "detail": "More detailed explanation",
  "trace_id": "...",
  "metadata": { ... }
}
```

`type` 字段是稳定的机器可读 ID；`title` 和 `detail` 可本地化。

## 2. 错误分类

### 2.1 Mandate / Authorization (40x)

| type | status | 含义 |
|---|---:|---|
| `fap-me/mandate-not-found` | 401 | Mandate ID 不存在 |
| `fap-me/mandate-expired` | 401 | Mandate 已过期 |
| `fap-me/mandate-revoked` | 401 | Mandate 已撤销 |
| `fap-me/mandate-signature-invalid` | 401 | Mandate 签名验证失败 |
| `fap-me/mandate-purpose-unknown` | 400 | purpose 不在标准词汇表 |
| `fap-me/mandate-purpose-mismatch` | 403 | purpose 与请求 capability triple 不匹配 |
| `fap-me/capability-triple-denied` | 403 | capability/layer/action 三元组未被授权 |
| `fap-me/mandate-scope-violation` | 403 | scope 超出 purpose 允许 |
| `fap-me/mandate-ttl-exceeded` | 400 | mandate TTL 超过 purpose.max_ttl_hours |
| `fap-me/elevated-mandate-required` | 403 | 需 elevated mandate |
| `fap-me/vc-required` | 403 | 跨组织需 VC 验证 |

### 2.2 Tenant / Isolation (40x)

| type | status | 含义 |
|---|---:|---|
| `fap-me/tenant-not-found` | 404 | Tenant 不存在 |
| `fap-me/tenant-suspended` | 403 | Tenant 已暂停 |
| `fap-me/cross-tenant-denied` | 403 | 跨 tenant 访问未授权 |
| `fap-me/namespace-not-found` | 404 | namespace 不存在 |
| `fap-me/quota-exceeded` | 429 | 超出 tenant 配额 |
| `fap-me/qps-throttled` | 429 | 超 QPS 限流 |

### 2.3 Content Safety (40x)

| type | status | 含义 |
|---|---:|---|
| `fap-me/content-injection-detected` | 400 | Prompt Injection 拦截 |
| `fap-me/content-too-long` | 413 | 内容超长度限制 |
| `fap-me/content-too-many-tokens` | 413 | embedding 输入超 token 限制 |
| `fap-me/content-disallowed-chars` | 400 | 禁用字符 |
| `fap-me/content-pii-violation` | 400 | PII 检测拦截（block 模式） |

### 2.4 Memory Operations (40x)

| type | status | 含义 |
|---|---:|---|
| `fap-me/memory-not-found` | 404 | MemoryUnit 不存在 |
| `fap-me/memory-already-exists` | 409 | idempotency_key 冲突 |
| `fap-me/memory-revision-conflict` | 409 | 乐观锁失败 |
| `fap-me/layer-not-allowed` | 403 | 不允许访问该层 |
| `fap-me/sbu-access-denied` | 403 | 未授权访问 SBU |
| `fap-me/l2-eligibility-failed` | 400 | L2 写入门槛未通过 |
| `fap-me/conflict-resolution-rejected` | 409 | 冲突仲裁拒绝 |

### 2.5 Context Sharing & Handoff (40x)

| type | status | 含义 |
|---|---:|---|
| `fap-me/snapshot-not-found` | 404 | Snapshot 不存在 |
| `fap-me/snapshot-expired` | 410 | Snapshot 已过期 |
| `fap-me/snapshot-content-mismatch` | 409 | snapshot content_hash 不一致 |
| `fap-me/grant-not-found` | 404 | ContextGrant 不存在 |
| `fap-me/grant-expired` | 410 | ContextGrant 已过期 |
| `fap-me/grant-revoked` | 410 | ContextGrant 已撤销 |
| `fap-me/grant-signature-invalid` | 401 | Grant 签名错误 |
| `fap-me/grant-triple-not-allowed` | 403 | 接收方 capability triple 不在 grant.allowed_triples |
| `fap-me/redaction-report-invalid` | 400 | RedactionReport 验证失败 |
| `fap-me/redaction-policy-untrusted` | 400 | 策略未在 trusted_policies |
| `fap-me/redaction-manifest-missing` | 400 | 跨租户/跨组织共享缺少 sbu_manifest |
| `fap-me/redaction-classifier-too-old` | 400 | classifier_version 低于接收方要求 |
| `fap-me/handoff-signature-invalid` | 401 | HandoffPacket 签名错误 |

### 2.6 Index / Storage (5xx)

| type | status | 含义 |
|---|---:|---|
| `fap-me/index-version-mismatch` | 500 | embedding 模型与索引版本不一致 |
| `fap-me/index-corruption` | 500 | 索引文件损坏 |
| `fap-me/index-rebuild-in-progress` | 503 | 索引重建中 |
| `fap-me/storage-unavailable` | 503 | 存储后端不可达 |
| `fap-me/storage-quota-exhausted` | 507 | 存储已满 |

### 2.7 Audit Chain (5xx)

| type | status | 含义 |
|---|---:|---|
| `fap-me/chain-broken` | 500 | 审计链断裂（防篡改告警） |
| `fap-me/chain-verify-failed` | 500 | 链完整性验证失败 |
| `fap-me/audit-sink-unavailable` | 503 | AuditSink 不可达 |
| `fap-me/tampering-detected` | 500 | 检测到链篡改 |

### 2.8 Dream / Forget (4xx + 5xx)

| type | status | 含义 |
|---|---:|---|
| `fap-me/dream-blocked` | 403 | DreamProposal 含 SBU，永久阻断 |
| `fap-me/dream-expired` | 410 | DreamProposal 超过 72h 未审批 |
| `fap-me/dream-already-applied` | 409 | DreamProposal 已应用 |
| `fap-me/dream-apply-failed` | 500 | mutation 应用失败 |
| `fap-me/forget-failed` | 500 | 遗忘流程失败 |
| `fap-me/forget-cascade-incomplete` | 500 | 级联清理不完整 |
| `fap-me/forget-reconcile-pending` | 202 | 本地已提交遗忘，外部系统仍在协调 |
| `fap-me/forget-reconcile-failed` | 500 | 外部系统协调失败，需要人工介入 |

### 2.9 Plugin (5xx)

| type | status | 含义 |
|---|---:|---|
| `fap-me/plugin-not-found` | 404 | Plugin 未注册 |
| `fap-me/plugin-version-conflict` | 409 | Plugin 版本冲突 |
| `fap-me/plugin-unavailable` | 503 | Plugin 健康检查失败 |
| `fap-me/plugin-timeout` | 504 | Plugin 调用超时 |
| `fap-me/plugin-permission-denied` | 403 | Plugin 试图访问越权资源 |
| `fap-me/wasm-sandbox-violation` | 500 | WASM 沙箱拦截 |
| `fap-me/wasm-memory-exceeded` | 500 | WASM 内存超限 |
| `fap-me/wasm-timeout` | 504 | WASM 执行超时 |

### 2.10 Retrieval (4xx + 5xx)

| type | status | 含义 |
|---|---:|---|
| `fap-me/retrieval-mode-not-supported` | 400 | 检索模式不支持 |
| `fap-me/retrieval-timeout` | 504 | 总检索超时 |
| `fap-me/retrieval-fallback` | 200* | 已降级（warning，不是错误） |
| `fap-me/retrieval-partial-result` | 200* | 部分结果（warning） |
| `fap-me/sbu-safe-cannot-fallback` | 504 | sbu_safe 不可降级 |
| `fap-me/topology-explosion-guard` | 200* | LIF 拓扑爆炸保护触发（warning） |

*`fap-me/retrieval-fallback` 与 `partial-result` 是 warning，不是失败。HTTP 可以用 200 + `Warning` 或响应 metadata；gRPC/streaming 必须放入 frame metadata、trailer 或 FAP-1 problem extension，不能只依赖 HTTP header。

## 3. 标准响应

```json
HTTP/1.1 403 Forbidden
Content-Type: application/problem+json

{
  "type": "fap-me/sbu-access-denied",
  "title": "SBU access denied",
  "status": 403,
  "detail": "Mandate purpose 'code_review' does not allow SBU access.",
  "trace_id": "abc-def-123",
  "metadata": {
    "purpose": "code_review",
    "required_sbu_access": "redacted_only or true"
  }
}
```

## 4. 与 FAP-1 错误码的关系

```text
FAP-1 错误码     协议级（Envelope / Replay / Auth）
FAP-ME 错误码    记忆扩展级（FME constraints / SBU / Redaction / Plugin / ...）

两者命名空间不冲突：
  FAP-1：fap/<category>-<reason>
  FAP-ME：fap-me/<category>-<reason>

实现层应同时支持。
```

## 5. 警告与降级

```text
warning metadata 用于：
  retrieval-fallback   降级提示
  partial-result       部分结果
  topology-guard       保护性截断
  redaction-applied    脱敏已应用

HTTP 格式：
  Warning: 199 fap-me "fap-me/retrieval-fallback" "Fell back from tagmemo to hybrid"

FAP-1 Data Plane / gRPC 格式：
  frame.metadata.problem_warnings[] 或 trailer.metadata.problem_warnings[]
```

## 6. 客户端处理建议

```text
401 / 403：刷新凭证或拒绝
404：可能是异步删除或权限不足
409：重试（如果能）或冲突仲裁
410：资源已不可用，停止重试
413：缩小输入
429：退避重试（指数退避）
500：报告错误，不重试
503：退避重试（短）
504：检索可降级；其他改用其他模式
```

## 7. 监控与告警

```text
按 type 分组监控：
  fap-me/chain-broken                   红色告警（防篡改）
  fap-me/tampering-detected             红色告警
  fap-me/forget-cascade-incomplete      红色告警（本地合规失败）
  fap-me/forget-reconcile-failed        红色告警（跨系统协调失败）
  fap-me/redaction-manifest-missing     红色告警（跨组织共享策略错误）

  fap-me/dream-expired                  黄色告警（审批流程问题）
  fap-me/quota-exceeded                 黄色告警（容量规划）
  fap-me/retrieval-fallback rate > 5%   黄色告警（性能）

  fap-me/plugin-timeout                 信息告警（关注）
```
