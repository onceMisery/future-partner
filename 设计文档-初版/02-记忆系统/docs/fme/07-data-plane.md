# 07 - 数据面流式记忆

> 数据面**完全复用** FAP-1 Data Plane（详见 FAP-1 [data-plane-merkle.md](../../../01-协议设计/docs/fap/data-plane-merkle.md) 与 [07-data-plane.md](../../../01-协议设计/docs/fap/07-data-plane.md)）。FME 不定义自定义 Frame，仅定义语义层 payload 类型。

## 1. 设计原则

```text
不重新定义 frame、签名、Merkle、断点续传、背压
完全使用 FAP-1 的：
  DataOpen          流开始
  DataChunkMeta     chunk 元数据 + Merkle 节点 hash
  DataAck           确认与流控
  DataCommit        流结束 + Merkle root
  DataReset         异常重置

FME 仅声明：
  payload 类型与 schema（如 RecallResultFrame、ContextFrame）
  写入路径的安全检查顺序
```

## 2. FME Payload 类型

| stream 用途 | payload 类型 | 方向 | DataOpen 携带 |
|---|---|---|---|
| 检索结果流 | `RecallResultFrame` | Engine → Agent | mode、tenant_id、namespace |
| 上下文写入流 | `ContextFrame` | Agent → Engine | session_id、layer=L0 |
| 长文档上传 | `ObjectRef` 引用 | Agent → Engine | upload_session_id |
| 审计订阅流 | `AuditEventFrame` | Engine → Agent | tenant_id、time_range |
| 梦境提案推送 | `DreamProposalFrame` | Engine → Agent | tenant_id |
| 共享上下文兑现流 | `ContextFrame` | Engine → Agent | grant_id |

## 3. 大对象通过 ObjectRef，不切片走 Frame

```text
小内容（< 64KB）：直接内嵌在 InvokeRequest.input 中
中等内容（64KB ~ 1MB）：FAP-1 Data Plane chunk 流式（自动 Merkle）
大对象（> 1MB）：上传到 ObjectStore 得到 ObjectRef
                 InvokeRequest 仅传 ObjectRef
                 Engine 按需通过 ObjectStorePlugin 拉取
```

不再使用最终版自定义的 `MemoryChunkFrame.chunk_bytes`——那种自定义切片绕过了 FAP-1 的 Merkle 完整性。

## 4. 写入路径强制顺序（修复 P0-5）

任何写入路径（含 ContextFrameStream）必须按以下顺序：

```text
1. FAP-1 Frame 解码（DataOpen / DataChunkMeta）
2. Merkle 完整性验证（chunk 链）
3. payload 解码（如 ContextFrame）
4. ContentSafetyGuard.check_write    ← 阻断式
5. SBU 自动标记（PolicyKernel.classify）
6. PolicyKernel.authorize            ← 校验是否允许写入此层
7. TenantKernel filter 注入
8. Hippocampus.put（事务）
9. AuditKernel.append（绑定 fap_receipt_id 待 DataCommit 后写）
```

**禁止**任何"立刻 append L0，安全检测后置"的实现。任何越过 ContentSafetyGuard 的 chunk 必须被丢弃且写审计 `security.bypass_attempt`。

## 5. RecallResultFrame

```proto
message RecallResultFrame {
  uint32 rank = 1;
  string memory_id = 2;
  float final_score = 3;
  ScoreBreakdown breakdown = 4;
  string content_preview = 5;
  optional ObjectRef content_object_ref = 6;     // 长内容
  repeated string highlight_tags = 7;
  optional SpikePathway spike_path = 8;
  RetrievalSource source = 9;
  optional GeodesicScoreInfo geo_info = 10;
}

message ScoreBreakdown {
  float vector_score = 1;
  float keyword_score = 2;
  float spike_score = 3;
  float tag_topology_score = 4;
  float recency_score = 5;
  float trust_score = 6;
  float duplication_penalty = 7;
  float sensitivity_penalty = 8;
  optional float geodesic_score = 9;
  float final_score = 10;
}
```

打分公式见 [scoring-formula.md](./scoring-formula.md)。

## 6. ContextFrame

```proto
message ContextFrame {
  string session_id = 1;
  ContextFrameKind kind = 2;        // USER_INPUT | TOOL_RESULT | SYSTEM_NOTE | DIFF
  bytes content = 3;
  google.protobuf.Timestamp at = 4;
  repeated string actor_dids = 5;
}
```

写入路径严格按 §4 顺序处理。

## 7. AuditEventFrame

```proto
message AuditEventFrame {
  AuditEvent event = 1;             // 含 fap_receipt_id 反向绑定
  optional bytes prev_event_hash = 2;
  bytes event_hash = 3;
}
```

仅 `memory.audit.query` capability 持有者可订阅。

## 8. DreamProposalFrame

```proto
message DreamProposalFrame {
  DreamProposal proposal = 1;
  DreamProposalStatus status = 2;
  optional string transition_reason = 3;
}
```

## 9. 长文档上传流（替代最终版的 MemoryChunkStream）

```text
1. Agent 调用 capability `memory.store` 但 input 含 upload_session_id
2. Agent 通过 FAP-1 Data Plane 开启上传流（DataOpen，目标 = upload_session_id）
3. 按 FAP-1 chunk 协议上传（自动 Merkle / 背压 / 断点续传）
4. DataCommit 时 Engine 收到完整 Merkle root
5. Engine：
   - 按 §4 顺序处理（ContentSafetyGuard 等）
   - 通过 ObjectStorePlugin 持久化得到 ObjectRef
   - 完成 capability invocation，返回 StoreReceipt
```

## 10. ObjectRef 复用 FAP-1

```proto
// FAP-1 已有，FME 不重新定义
message ObjectRef {
  string object_id = 1;
  string media_type = 2;
  uint64 size_bytes = 3;
  bytes sha256 = 4;
  string uri = 5;
  string lease_id = 6;
  google.protobuf.Timestamp expires_at = 7;
}
```

`ObjectStorePlugin` 即 FAP-ME `ObjectStore` 插件，统一接口（详见 [09-plugin-runtime.md](./09-plugin-runtime.md)）。

## 11. 背压与流控

完全继承 FAP-1：

```text
窗口默认 64 frames inflight
max_inflight ≤ 256
DataAck 控制 window_remaining
超限时发送方暂停发送
```

详见 FAP-1 [data-backpressure.md](../../../01-协议设计/docs/fap/data-backpressure.md)。

## 12. Resume 与断点续传

```text
ResumeToken {
  stream_id
  last_acked_sequence
  merkle_node_hash
  payload_sha256_so_far
}
```

由 FAP-1 提供；FME 不重复实现。

## 13. 安全约束

```text
所有 frame 必须在已建立的会话内
  → session_id 在 FAP-1 Envelope 中验证

所有写入流必须经过 ContentSafetyGuard（按 §4 顺序）
  → 任何绕过路径在 conformance 测试中拒绝编译

所有召回结果流必须经过 PolicyKernel.redact
  → SBU 内容在脱敏后才能进入流

审计流仅向授权 actor 推送
  → mandate.capabilities 必须含 memory.audit.query
```

## 14. 与最终版的差距

```text
最终版                                  本文档
-----                                   -----
自定义 DataFrame                         复用 FAP-1 DataOpen/DataChunkMeta/DataCommit/DataAck
自定义 MemoryChunkFrame                   长文档通过 ObjectRef 或 FAP-1 chunk 流
自定义 ContextFrame 流                    payload 类型保留，frame 框架是 FAP-1
ContextFrameStream 安全顺序矛盾           §4 强制顺序：decode → safety → SBU label → authorize → append
独立背压实现                             复用 FAP-1 背压
独立 Resume 实现                         复用 FAP-1 Resume
```
