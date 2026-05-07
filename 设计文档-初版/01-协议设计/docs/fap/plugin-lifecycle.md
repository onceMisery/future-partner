# plugin-lifecycle.md

插件发布与版本并存模型。

## 1. 发布状态机

```text
UPLOADED
  → VALIDATED       (manifest 校验 + 签名校验通过)
  → STAGED          (隔离环境加载成功)
  → DRY_RUN_PASSED  (schema dry-run + 样本 Invoke 通过)
  → ACTIVE          (进入路由表)
  → DRAINING        (收到升级/停用信号)
  → STOPPED
  → UNLOADED
```

异常状态：FAILED_VALIDATION / FAILED_STAGING / FAILED_DRY_RUN / DEGRADED。

## 2. 发布流程

```text
1. UPLOADED:
   插件 artifact + manifest 上传到 Core
   计算 artifact hash
   
2. VALIDATED:
   manifest schema 校验
   signature 校验（若配置要求）
   permissions 不超系统允许的最大集合
   min_core_version / plugin_api MAJOR 匹配
   
3. STAGED:
   在隔离环境启动插件（WASM sandbox / Sidecar）
   不注册到路由表
   Init / Start 成功
   Health 返回 UP
   
4. DRY_RUN_PASSED:
   对每个 capability 跑 schema dry-run（样本输入 → 输入 schema 校验）
   可选：运行 golden test input → 比对 golden output
   可选：运行 fuzz 样本确认不崩溃
   
5. ACTIVE:
   注册 capability 到路由表
   更新 Agent Card
   对外可调用
   
6. DRAINING:
   从路由表剔除新 invocation
   已分配的 invocation 完成或超时
   期间 health 仍 UP
   
7. STOPPED:
   所有 in-flight invocation 已完成
   插件进程/WASM 实例 stop
   
8. UNLOADED:
   artifact 可被删除
   manifest 进入历史表
```

## 3. 升级：版本并存

### 3.1 同 capability_id 多版本并存

```text
capability_id + schema_version 构成唯一键:
  capability "code.review" 可同时存在:
    version=1.0.0, schema_version=a1b2c3d4
    version=1.1.0, schema_version=e5f6g7h8
    version=2.0.0, schema_version=i9j0k1l2

路由策略:
  InvokeRequest 未指定版本 → 取最新 ACTIVE
  InvokeRequest 指定 version=^1.0 → 取匹配的最新 ACTIVE
  InvokeRequest 指定 schema_version=a1b2c3d4 → 精确匹配
```

### 3.2 升级流程

```text
新版本上传
  → VALIDATED → STAGED → DRY_RUN_PASSED

双写模式:
  新版本 ACTIVE（路由表同时含新旧）
  新 invocation 按路由策略分配
  客户端可显式指定 schema_version 钉版

旧版本进入 DRAINING:
  不接新 invocation
  在途完成后 STOPPED

回滚:
  若新版本 DEGRADED → 旧版本 ACTIVE 保持
  若旧版本已 STOPPED 但新版本异常 → 从 UNLOADED 前可重启
```

### 3.3 Schema 不兼容的升级

```text
MAJOR 升级（schema 不兼容）:
  旧客户端继续调旧版本
  新客户端调新版本
  至少保留 2 个 MINOR 周期

MINOR 升级（schema 兼容）:
  所有客户端可无感升级
  compatibility_warnings 用于标记新字段

PATCH 升级:
  完全透明
```

## 4. 路由中的版本选择

```proto
message InvokeRequest {
  string capability_id = 2;
  // ...
  // options 中可指定版本约束
  map<string, string> options = 100;
}
```

约定 options key：

```text
fap.capability.version_requirement = "^1.0"
fap.capability.schema_version = "a1b2c3d4"
fap.capability.min_version = "1.0.0"
fap.capability.prefer = "stable" | "latest"
```

## 5. 并存存储

SQLite 的 `capability` 表主键改为 `(capability_id, version)`：

```sql
CREATE TABLE capability (
  capability_id TEXT NOT NULL,
  version TEXT NOT NULL,
  plugin_id TEXT NOT NULL,
  name TEXT NOT NULL,
  schema_version TEXT NOT NULL,
  min_fap_version TEXT NOT NULL,
  risk_level TEXT NOT NULL,
  requires_mandate INTEGER NOT NULL,
  supports_batch INTEGER NOT NULL,
  schema_json TEXT NOT NULL,
  state TEXT NOT NULL,         -- STAGED / ACTIVE / DRAINING / STOPPED
  activated_at INTEGER,
  updated_at INTEGER NOT NULL,
  PRIMARY KEY (capability_id, version)
);

CREATE INDEX idx_capability_state ON capability(capability_id, state);
```

## 6. 插件升级事件

```proto
message PluginLifecycleEvent {
  string plugin_id = 1;
  string version = 2;
  PluginLifecycleState new_state = 3;
  google.protobuf.Timestamp occurred_at = 4;
  string reason = 5;
}

enum PluginLifecycleState {
  PLUGIN_STATE_UNSPECIFIED = 0;
  PLUGIN_UPLOADED = 1;
  PLUGIN_VALIDATED = 2;
  PLUGIN_STAGED = 3;
  PLUGIN_DRY_RUN_PASSED = 4;
  PLUGIN_ACTIVE = 5;
  PLUGIN_DRAINING = 6;
  PLUGIN_STOPPED = 7;
  PLUGIN_UNLOADED = 8;
  PLUGIN_FAILED = 99;
}
```

生命周期事件必须写入审计日志，便于回溯。

## 7. 原子性

```text
状态转移必须原子:
  VALIDATED → STAGED: 失败不保留部分状态
  STAGED → ACTIVE: 失败回到 STAGED
  ACTIVE → DRAINING: 不可逆，但 DRAINING 内可 cancel

路由表更新必须原子:
  添加新版本 ACTIVE 与移除旧版本 ACTIVE 在同一事务
  或采用 read-copy-update 模式
```

## 8. 失败回滚

```text
VALIDATED 失败 → 删除 artifact，记录 FAILED_VALIDATION
STAGED 失败 → 清理 sandbox，记录 FAILED_STAGING
DRY_RUN 失败 → 插件不 ACTIVE，原版本继续服务
ACTIVE 后 health 异常 → DEGRADED，保留路由但优先调度其它版本
```
