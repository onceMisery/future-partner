# 22 - 回滚策略

## 1. 传输回滚

```text
QUIC → HTTP JSON → MCP Bridge → A2A Bridge
```

任何 TransportPlugin 失败，可降级到下一优先级 TransportPlugin。

## 2. 功能回滚（feature flag）

```yaml
features:
  fap_core: true
  http_json: true
  quic: true
  plugin_runtime: true
  fap_hp: true

  did_vc: false
  latent_channel: false
  gossip_discovery: false
  crdt_workspace: false
  semantic_routing: false
```

## 3. 插件回滚

```text
disable plugin
  → drain running invocation
  → rollback manifest version
  → restore previous artifact
  → rebuild capability registry
  → emit audit event
```

## 4. 协议版本回滚

```text
客户端发现服务端不支持新版本
  → 降级到 minCompatibleVersion
  → 禁用新版本特性
  → 记录 compatibility_warnings
```

## 5. 会话回滚

```text
会话异常
  → 进入 DRAINING
  → 完成在途 Invoke
  → 生成补偿 Receipt
  → CLOSED
  → 客户端可 Resume 或重新 Hello
```

## 6. 数据回滚

```text
DataCommit 失败
  → 删除临时 chunk
  → 释放 lease
  → 返回 Problem: OBJECT_UNAVAILABLE
  → 客户端可重试
```

## 7. Receipt 回滚

```text
Multisig 超时未齐
  → finality 保持 PROVISIONAL
  → 触发告警
  → 人工介入决定：补签 / 作废 / 补偿
```

## 8. 节点回滚

```text
节点 UNHEALTHY
  → 进入 QUARANTINED
  → 从路由表剔除
  → 15min 自动恢复窗
  → 或人工恢复
```

## 9. 审计回滚

```text
Receipt chain 断裂
  → 节点进入 DEGRADED
  → 拒绝 high/critical 请求
  → 人工修复 chain
  → 恢复 HEALTHY
```

## 10. 存储回滚

```text
SQLite WAL 损坏
  → 从最近 checkpoint 恢复
  → 重放 WAL
  → 若失败，从备份恢复
  → 记录数据丢失范围
```
