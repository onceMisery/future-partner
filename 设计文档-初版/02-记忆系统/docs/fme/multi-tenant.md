# multi-tenant.md

> 多租户隔离与插件级别配置覆盖。补充最终版"插件全局单例"无法满足不同租户差异化需求的问题。

## 1. 设计原则

```text
租户隔离强制在 Kernel 内执行（TenantKernel）
插件可按租户配置覆盖（不同 tenant 用不同实现/参数）
租户数据物理隔离（PG schema / SQLite 文件 / Qdrant collection）
跨租户访问必须显式 cross_tenant 授权 + VC
```

## 2. 隔离维度

```text
tenant_id              主键
namespace              业务命名空间（多个 namespace 可共享 tenant 资源）
owner_subject_did      用户级
owner_agent_did        Agent 级
visibility             private | project | team | org | foreign
security_labels        sbu | secret | pii | internal
mandate_scope          mandate 限定的子集
```

每条 MemoryUnit 必须含 tenant_id。任何查询自动过滤。

## 3. TenantKernel

```rust
pub struct TenantKernel {
    tenant_store: Arc<TenantStore>,
    isolation_mode: IsolationMode,
}

pub enum IsolationMode {
    LogicalRow,     // 同一 DB，行级 tenant_id 过滤（默认）
    SchemaPerTenant, // PG 多 schema，每 tenant 独立 schema
    DbPerTenant,    // 单独 DB（极端隔离）
}

impl TenantKernel {
    /// 强制注入 tenant 过滤（插件不能拼装 SQL）
    pub fn inject_filter(
        &self,
        session: &Session,
        base_filter: &SearchFilter,
    ) -> Result<SearchFilter>;

    /// 验证跨租户操作授权
    pub fn check_cross_tenant(
        &self,
        from_tenant: &TenantId,
        to_tenant: &TenantId,
        mandate: &Mandate,
    ) -> Result<()>;
}
```

## 4. 数据物理隔离

### 4.1 单机/Edge Lite

```text
默认：每 tenant 一个 SQLite 文件
  .fap-me/tenants/<tenant_id>/tenant.db

或：单 SQLite，行级 tenant_id 过滤
  .fap-me/all-tenants.db
```

### 4.2 服务端（PostgreSQL）

```text
默认：单 DB + 行级 tenant_id 过滤 + RLS（Row-Level Security）

CREATE POLICY tenant_isolation ON memory_unit
  USING (tenant_id = current_setting('app.tenant_id'));

ALTER TABLE memory_unit ENABLE ROW LEVEL SECURITY;
```

高隔离模式：每 tenant 一个 schema：

```text
schema "tenant_acme":
  memory_unit
  episode
  semantic_fact
  ...

schema "tenant_globex":
  ...
```

### 4.3 向量索引

```text
Qdrant：
  collection per tenant：fapme_<tenant_id>
  namespace 必须进入 payload filter，禁止跨 namespace 统计混合
  
HNSW (单机)：
  index file per tenant/namespace：indexes/<tenant_id>/<namespace>/hnsw_v1.rkyv
```

### 4.4 对象存储

```text
S3：bucket prefix per tenant：
  s3://fap-me-prod/tenants/<tenant_id>/namespaces/<namespace>/objects/<sha256>

本地 CAS：
  .fap-me/tenants/<tenant_id>/namespaces/<namespace>/objects/
```

## 5. 插件级别配置覆盖

### 5.1 配置结构

```toml
[plugins.embedding]
default = "openai-text-embedding-3"

[plugins.embedding.tenant_overrides.medical_org]
provider = "biomedical-bert-v2"
config = { model_name = "BioBERT-base-v1.1" }

[plugins.embedding.tenant_overrides.legal_org]
provider = "legal-bert"

[plugins.dream_worker.tenant_overrides.research_lab]
provider = "experimental-llm-dreamer"
allow_high_risk_auto_approve = false

[plugins.compact_strategy.tenant_overrides.high_volume_tenant]
provider = "default"
config = {
  token_threshold = 8192,
  turn_threshold = 20,
}
```

### 5.2 解析逻辑

```rust
impl PluginRegistry {
    pub fn resolve_for_tenant(
        &self,
        tenant_id: &TenantId,
        kind: PluginKind,
    ) -> Result<Arc<dyn Plugin>> {
        // 1. 查找 tenant 覆盖
        if let Some(override_cfg) = self.tenant_overrides.read().get(tenant_id) {
            if let Some(plugin_id) = override_cfg.get(kind) {
                return self.lookup(plugin_id);
            }
        }
        // 2. 默认实现
        self.lookup_default(kind)
    }
}
```

### 5.3 Plugin trait 的 configure_for_tenant

```rust
pub trait Plugin: Send + Sync {
    ...
    fn configure_for_tenant(
        &self,
        tenant_id: &TenantId,
        config: &TenantPluginConfig,
    ) -> Result<Box<dyn Plugin>>;
}
```

允许同一 plugin 类型按 tenant 实例化不同配置：

```text
EmbeddingProvider "openai" 实例 A：tenant_acme，model = ada-002
EmbeddingProvider "openai" 实例 B：tenant_globex，model = text-embedding-3
```

## 6. 跨租户访问

### 6.1 默认禁止

```text
TenantKernel.inject_filter 始终注入 WHERE tenant_id = :session_tenant
任何插件无法访问其他 tenant
```

### 6.2 显式允许

```text
1. signed constraint memory.purpose 必须是 cross_tenant=true 的 purpose
   （如 knowledge_transfer 或 cross_agent_collaboration）
2. FAP-1 Mandate constraints 显式列出允许的 namespace / cross_tenant
3. 跨组织时必须 requires_vc = true（VC 验证）
4. RedactionPolicy.cross_org_safe = true
5. RedactionReport 必须携带 sbu_manifest
6. 写审计：cross_tenant.access
```

### 6.3 数据流向

```text
跨租户访问只读：source tenant → target session
不允许跨租户写入（避免数据污染）
跨租户检索结果必须经 RedactionPolicy 处理
```

## 7. 配额与限流

```text
按 tenant 配置：
  storage_quota_bytes      存储配额
  vector_quota             向量数量上限
  retrieve_qps             检索 QPS
  store_qps                写入 QPS
  dream_jobs_per_day       梦境任务数上限

超限：
  返回错误码 fap-me/quota-exceeded
  写审计 quota.exceeded
```

## 8. 多租户审计隔离

```text
每个 tenant/namespace 独立的 audit chain head
每个 tenant/namespace 独立的 chain_state 行
跨 tenant 的审计查询需 admin mandate
```

## 9. 配置示例

```toml
# fap-me.toml

[tenants]
isolation_mode = "schema_per_tenant"
default_quota_bytes = 10_737_418_240   # 10 GB
default_retrieve_qps = 100
default_store_qps = 50

[tenants.acme]
quota_bytes = 107_374_182_400          # 100 GB
retrieve_qps = 500
store_qps = 200
isolation_mode = "db_per_tenant"

[tenants.research_lab]
allow_experimental_plugins = true
allow_high_risk_auto_approve = false
```

## 10. 启动时验证

```text
启动时：
  1. 加载所有 tenant_id 列表
  2. 对每个 tenant：
     - 验证 storage 后端可达
     - 验证 vector index 后端可达
     - 验证 plugin 可解析（含 override）
  3. 任一失败 → 启动失败 + 详细日志
```

## 11. 租户生命周期

| 操作 | 触发 | 副作用 |
|---|---|---|
| 创建 | admin API | 创建 schema / collection / 配额 |
| 暂停 | admin API | 拒绝新请求；保留数据 |
| 恢复 | admin API | 接受新请求 |
| 注销 | admin API + 合规审批 | 触发 ForgetEngine.tenant_purge → ForgetReceipt |

注销必须：

```text
1. 提交 elevated mandate（compliance_audit purpose）
2. ForgetEngine 完整级联清理：
   - 所有 memory_unit
   - 所有 vector index entries
   - 所有 object_ref + 实际 object
   - 所有 context_grant
   - 所有 handoff_packet（标记 redacted）
   - 保留 audit_event（合规要求）
3. 写最终审计 tenant.purged
4. 返回 ForgetReceipt；本地状态为 COMMITTED_LOCALLY，外部 S3/Qdrant/Kafka 等系统通过 mutation ledger 达成 GLOBALLY_RECONCILED
```

## 12. 与最终版的差距

```text
最终版                       本文档
-----                        -----
插件全局单例                 PluginRegistry.tenant_overrides + configure_for_tenant
tenant 隔离仅 SQL 过滤       三种 IsolationMode + RLS + schema/db per tenant
无配额机制                   storage_quota / qps / dream_jobs 限流
无生命周期定义               创建 / 暂停 / 注销 + 合规清退
```

详见 `../../3-记忆系统深度分析与未来方案.md` §3.3。
