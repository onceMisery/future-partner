# 13 - 分布式节点

## 1. 节点角色

| 角色 | 说明 |
|---|---|
| Main Node | 协议入口、能力路由、审计中心 |
| Worker Node | 提供本地插件能力 |
| Storage Node | 对象存储与文件解析 |
| GPU Node | 多模态生成、推理 |
| Memory Node | 检索与记忆服务 |

## 2. 节点注册

```text
Worker 启动
  → 加载本地插件
  → 生成 NodeHello
  → 连接 Main Node (QUIC)
  → Auth
  → RegisterCapabilities
  → 启动 Heartbeat
  → 状态 = READY
```

## 3. NodeHealth FSM + 断路器

```text
HEALTHY
  → DEGRADED     (error_rate > 10% in 1min)
  → UNHEALTHY    (error_rate > 30% 或 heartbeat 连续 miss ×3)
  → QUARANTINED  (手动隔离 或 UNHEALTHY 持续 5min)
  → 15min 自动恢复窗 → HEALTHY

QUARANTINED 节点的 capabilities 从路由表临时剔除。
```

## 4. 远程调用

```text
Client InvokeRequest
  → Main Node
  → Capability Router (跳过 QUARANTINED)
  → Remote Node 选择（locality + load + health）
  → RemoteInvoke (forward 原 InvokeRequest)
  → Worker 执行插件
  → ProgressEvent 回传
  → InvokeResult 回传
  → Main Node 生成 Receipt（MAIN + EXECUTOR 签名）
```

## 5. NodeHealthPlugin

节点健康策略可插拔：

| Plugin | 说明 |
|---|---|
| SimpleNodeHealthPlugin | 基于 error_rate + heartbeat（默认） |
| AdaptiveNodeHealthPlugin | 自适应阈值 |
| MLNodeHealthPlugin | 机器学习预测 |
