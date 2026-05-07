# 06 - 控制面

控制面承载结构化控制消息，不承载大文件/视频/音频/图片/大日志/长文本流/embedding 大数组（这些进数据面）。

## 1. Envelope

见 [03-wire-protocol.md](./03-wire-protocol.md)。

## 2. 校验顺序

```text
1. Decode Protobuf
2. 校验 wire_major/wire_minor
3. 校验 message_id 格式
4. 校验 expires_at ≤ session.max_ttl
5. 校验 Replay Cache
6. 校验 session_id
7. 校验 payload_sha256
8. 校验 prev_hash 指向链内
9. 校验 signatures（按 signing-canonical）
10. 分发 handler
```

## 3. Problem 错误

见 [problem-catalog.md](./problem-catalog.md)。

## 4. 优先级

```text
Priority 0: Control (Hello/Auth/Capability/Cancel/Problem)
Priority 1: Audit (Receipt)
Priority 2: Data (DataOpen/DataChunkMeta/DataCommit)
```

控制面消息必须优先于数据面消息出队。
