# purpose-vocabulary.md

> 标准化 FAP-ME Purpose 词汇表与授权三元组，解决自由文本 purpose、字符串式操作名和实现间模糊匹配导致的授权不一致问题。

## 1. 背景

FAP-ME 不再定义独立的 FME Mandate schema。记忆授权必须复用 FAP-1 `Mandate`，并通过 FME 约束扩展表达记忆语义：

```text
FAP-1 Mandate.capabilities   声明可调用的 capability
FAP-1 Mandate.constraints    声明 memory.purpose / memory.allowed_triples / memory.sbu_access 等约束
FME PurposeRegistry          给出每个标准 purpose 允许的三元组上界
```

旧写法把 `retrieve`、`retrieve:L1`、`memory.forget:hard`、`admin.rebuild` 放在同一个字符串数组中，混合了 capability、layer 和 risk/action 维度。实现如果使用 `contains`、前缀或后缀匹配，容易把 `memory.retrieve` 误授权为 `memory.retrieve:L0`，或把 `admin.rebuild` 误映射到不受控的管理操作。

本文档规定：策略匹配必须使用结构化三元组精确匹配，禁止基于字符串包含、前缀、后缀或通配符推断权限。

## 2. 标准 Purpose 列表（V1）

| Purpose | 含义 |
|---|---|
| `code_review` | 代码审查 |
| `project_debugging` | 项目排障 |
| `knowledge_transfer` | 跨项目/跨组织知识共享 |
| `compliance_audit` | 合规审计 |
| `memory_maintenance` | 记忆维护（梦境审批、遗忘、重建） |
| `cross_agent_collaboration` | 跨 Agent 协作 |

## 3. 授权三元组

```proto
enum MemoryCapability {
  MEMORY_CAPABILITY_UNSPECIFIED = 0;
  MEMORY_RETRIEVE = 1;
  MEMORY_STORE = 2;
  MEMORY_SHARE = 3;
  MEMORY_HANDOFF = 4;
  MEMORY_FORGET = 5;
  MEMORY_DREAM = 6;
  MEMORY_AUDIT = 7;
  MEMORY_ADMIN = 8;
}

enum MemoryLayer {
  MEMORY_LAYER_UNSPECIFIED = 0;
  L0 = 1;
  L1 = 2;
  L2 = 3;
  L3 = 4;
  AUDIT_CHAIN = 5;
}

enum MemoryAction {
  MEMORY_ACTION_UNSPECIFIED = 0;
  READ = 1;
  WRITE = 2;
  SHARE_CREATE = 3;
  HANDOFF_CREATE = 4;
  HANDOFF_RECEIVE = 5;
  DELETE_SOFT = 6;
  DELETE_HARD = 7;
  PROPOSE = 8;
  APPROVE = 9;
  REJECT = 10;
  QUERY = 11;
  ADMIN = 12;
}

message MemoryCapabilityTriple {
  MemoryCapability capability = 1;
  MemoryLayer layer = 2;
  MemoryAction action = 3;
}
```

精确匹配规则：

```text
requested_triple ∈ mandate.allowed_triples
requested_triple ∈ purpose_registry[purpose].allowed_triples
```

`MEMORY_LAYER_UNSPECIFIED` 只能用于与层无关的 capability，例如 `MEMORY_HANDOFF/HANDOFF_CREATE`。不能用它表示“任意层”。

## 4. 词汇表 JSON Schema

```json
{
  "$schema": "fap-me-purpose-registry-v1",
  "version": "1.0",
  "purpose_registry": {
    "code_review": {
      "allowed_triples": [
        {"capability": "MEMORY_RETRIEVE", "layer": "L1", "action": "READ"},
        {"capability": "MEMORY_RETRIEVE", "layer": "L2", "action": "READ"},
        {"capability": "MEMORY_STORE", "layer": "L1", "action": "WRITE"}
      ],
      "sbu_access": "DENY",
      "cross_tenant": false,
      "max_ttl_hours": 8,
      "requires_vc": false,
      "requires_elevated_mandate": false,
      "description": "代码审查场景：可读历史 PR、issue、L2 项目知识，可在 L1 写入审查记录。"
    },

    "project_debugging": {
      "allowed_triples": [
        {"capability": "MEMORY_RETRIEVE", "layer": "L0", "action": "READ"},
        {"capability": "MEMORY_RETRIEVE", "layer": "L1", "action": "READ"},
        {"capability": "MEMORY_RETRIEVE", "layer": "L2", "action": "READ"},
        {"capability": "MEMORY_STORE", "layer": "L1", "action": "WRITE"},
        {"capability": "MEMORY_HANDOFF", "layer": "MEMORY_LAYER_UNSPECIFIED", "action": "HANDOFF_CREATE"}
      ],
      "sbu_access": "DENY",
      "cross_tenant": false,
      "max_ttl_hours": 24,
      "requires_vc": false,
      "requires_elevated_mandate": false,
      "description": "项目排障：可访问当前会话上下文与历史排障记录，可在 L1 留存调查过程，可创建 handoff。"
    },

    "knowledge_transfer": {
      "allowed_triples": [
        {"capability": "MEMORY_RETRIEVE", "layer": "L1", "action": "READ"},
        {"capability": "MEMORY_RETRIEVE", "layer": "L2", "action": "READ"},
        {"capability": "MEMORY_SHARE", "layer": "MEMORY_LAYER_UNSPECIFIED", "action": "SHARE_CREATE"},
        {"capability": "MEMORY_HANDOFF", "layer": "MEMORY_LAYER_UNSPECIFIED", "action": "HANDOFF_CREATE"}
      ],
      "sbu_access": "REDACTED_ONLY",
      "cross_tenant": true,
      "max_ttl_hours": 48,
      "requires_vc": true,
      "requires_elevated_mandate": false,
      "description": "跨组织知识共享：可读 L2 语义、共享 L1 摘要；强制 VC 验证；不可触达 SBU 原文。"
    },

    "compliance_audit": {
      "allowed_triples": [
        {"capability": "MEMORY_RETRIEVE", "layer": "L1", "action": "READ"},
        {"capability": "MEMORY_RETRIEVE", "layer": "L2", "action": "READ"},
        {"capability": "MEMORY_AUDIT", "layer": "AUDIT_CHAIN", "action": "QUERY"},
        {"capability": "MEMORY_FORGET", "layer": "MEMORY_LAYER_UNSPECIFIED", "action": "DELETE_HARD"}
      ],
      "sbu_access": "REDACTED_ONLY",
      "cross_tenant": false,
      "max_ttl_hours": 4,
      "requires_vc": false,
      "requires_elevated_mandate": true,
      "description": "合规审计：可查询审计链、可强制遗忘；SBU 仅以脱敏形式访问；需 elevated mandate。"
    },

    "memory_maintenance": {
      "allowed_triples": [
        {"capability": "MEMORY_RETRIEVE", "layer": "L1", "action": "READ"},
        {"capability": "MEMORY_RETRIEVE", "layer": "L2", "action": "READ"},
        {"capability": "MEMORY_RETRIEVE", "layer": "L3", "action": "READ"},
        {"capability": "MEMORY_STORE", "layer": "L1", "action": "WRITE"},
        {"capability": "MEMORY_DREAM", "layer": "MEMORY_LAYER_UNSPECIFIED", "action": "PROPOSE"},
        {"capability": "MEMORY_DREAM", "layer": "MEMORY_LAYER_UNSPECIFIED", "action": "APPROVE"},
        {"capability": "MEMORY_FORGET", "layer": "MEMORY_LAYER_UNSPECIFIED", "action": "DELETE_SOFT"},
        {"capability": "MEMORY_ADMIN", "layer": "MEMORY_LAYER_UNSPECIFIED", "action": "ADMIN"}
      ],
      "sbu_access": "DENY",
      "cross_tenant": false,
      "max_ttl_hours": 2,
      "requires_vc": false,
      "requires_elevated_mandate": false,
      "description": "记忆维护：梦境提案与审批、软遗忘、索引重建。短 TTL（2h）。"
    },

    "cross_agent_collaboration": {
      "allowed_triples": [
        {"capability": "MEMORY_RETRIEVE", "layer": "L0", "action": "READ"},
        {"capability": "MEMORY_RETRIEVE", "layer": "L1", "action": "READ"},
        {"capability": "MEMORY_SHARE", "layer": "MEMORY_LAYER_UNSPECIFIED", "action": "SHARE_CREATE"},
        {"capability": "MEMORY_HANDOFF", "layer": "MEMORY_LAYER_UNSPECIFIED", "action": "HANDOFF_CREATE"},
        {"capability": "MEMORY_HANDOFF", "layer": "MEMORY_LAYER_UNSPECIFIED", "action": "HANDOFF_RECEIVE"}
      ],
      "sbu_access": "DENY",
      "cross_tenant": true,
      "max_ttl_hours": 12,
      "requires_vc": false,
      "requires_elevated_mandate": false,
      "description": "跨 Agent 协作：用于多 Agent 共同完成任务；handoff 必须含 RedactionReport。"
    }
  }
}
```

## 5. FAP-1 Mandate 约束映射

FME 约束必须进入 FAP-1 `Mandate.constraints`，不得放入未签名的请求体字段：

```json
{
  "capabilities": ["memory.retrieve", "memory.store", "memory.handoff"],
  "constraints": [
    {"type": "memory.purpose", "value": "project_debugging"},
    {"type": "memory.allowed_triples", "value": [
      {"capability": "MEMORY_RETRIEVE", "layer": "L0", "action": "READ"},
      {"capability": "MEMORY_RETRIEVE", "layer": "L1", "action": "READ"},
      {"capability": "MEMORY_STORE", "layer": "L1", "action": "WRITE"},
      {"capability": "MEMORY_HANDOFF", "layer": "MEMORY_LAYER_UNSPECIFIED", "action": "HANDOFF_CREATE"}
    ]},
    {"type": "memory.namespaces", "value": ["project-x"]},
    {"type": "memory.sbu_access", "value": "DENY"},
    {"type": "memory.cross_tenant", "value": false}
  ]
}
```

详见 [fap1-binding.md](./fap1-binding.md)。

## 6. 字段语义

### `allowed_triples`

允许的记忆操作三元组。`capability`、`layer`、`action` 三个维度必须同时匹配。

### `sbu_access`

| 值 | 含义 |
|---|---|
| `DENY` | 完全禁止访问 SBU |
| `REDACTED_ONLY` | 只允许访问脱敏后视图 |
| `RAW_ALLOWED` | 允许访问原文，必须 elevated mandate 且写高风险审计 |

### `cross_tenant`

是否允许跨租户访问。`true` 时通常要求 `requires_vc = true`，并触发更严格的 RedactionReport 要求。

### `max_ttl_hours`

Mandate 最大允许有效期。请求方申请的 TTL 不能超过此值。

### `requires_vc`

跨组织时是否要求 Verifiable Credential。

### `requires_elevated_mandate`

是否要求 elevated mandate（由组织 owner 或合规角色颁发，而非自助申请）。

## 7. 校验流程

```text
MandateVerifier.verify(fap_mandate):
  1. 复用 FAP-1 验签、issuer/subject、expires_at、revocation、budget 校验
  2. 从 signed constraints 读取 memory.purpose
  3. purpose 必须在 PurposeRegistry 中存在，否则拒绝
  4. mandate.expires_at - now ≤ purpose.max_ttl_hours，否则拒绝
  5. 从 capability_id + request payload 推导 requested_triple
  6. requested_triple 必须精确存在于 mandate.memory.allowed_triples
  7. requested_triple 必须精确存在于 purpose.allowed_triples
  8. request namespace/layer/sbu/cross_tenant 必须被 signed constraints 覆盖
  9. purpose.requires_vc 时检查 FAP-1 identity proof / VC
  10. purpose.requires_elevated_mandate 时检查 issuer 是否具备 elevated role
```

不满足任一项即拒绝该调用，并写审计。禁止：

```text
op_string.contains(...)
op_string.starts_with(...)
capability 后缀推断 layer
缺省 layer 解释为 all layers
请求体字段覆盖已签名约束
```

## 8. PolicyKernel 集成

```rust
pub struct CapabilityTriple {
    capability: MemoryCapability,
    layer: MemoryLayer,
    action: MemoryAction,
}

impl PolicyKernel {
    pub fn authorize(
        &self,
        session: &Session,
        requested: CapabilityTriple,
        scope: &MemoryScope,
    ) -> Result<AuthorizedOp, PolicyError> {
        let mandate = session.fap_mandate();
        let purpose = self.purpose_registry
            .get(mandate.memory_purpose()?)
            .ok_or(PolicyError::UnknownPurpose)?;

        if !mandate.memory_allowed_triples().contains(&requested) {
            return Err(PolicyError::TripleNotAllowedByMandate);
        }

        if !purpose.allowed_triples.contains(&requested) {
            return Err(PolicyError::TripleNotAllowedForPurpose);
        }

        if !mandate.memory_scope().covers(scope) {
            return Err(PolicyError::ScopeNotCoveredByMandate);
        }

        if scope.includes_sbu && !purpose.sbu_access.allows(mandate.sbu_access()) {
            return Err(PolicyError::SbuAccessDenied);
        }

        Ok(AuthorizedOp { requested, scope: scope.clone() })
    }
}
```

## 9. 词汇表演进

```text
V1.0   定义 6 个标准 purpose 和结构化三元组
V1.1   可新增 purpose 或新增更细 action（需能力协商）
V2.0   不兼容变更，需 protocol/profile 协商
```

演进规则：

1. 不允许静默扩大已发布 purpose 的权限；新增三元组必须进入新 registry 版本。
2. 不允许静默修改已发布三元组的语义。
3. 安全收紧必须通过 mandate revocation、registry emergency override 或新版本协商完成，不能假装向后兼容。
4. 已签发 mandate 按签发时绑定的 registry version 校验，除非被显式撤销。
5. 自定义 purpose 只能在私有命名空间内使用，跨组织必须双方显式声明同一 registry hash。

## 10. Agent Card 声明

Agent Card 必须声明支持的 purpose registry 版本和 hash：

```json
{
  "memory": {
    "purpose_registry_version": "1.0",
    "purpose_registry_hash": "sha256:...",
    "fme_capability_triples": true
  }
}
```

发起跨 Agent 调用时，发起方与接收方必须协商相同版本或可验证的兼容映射。

## 11. 自定义 Purpose（不推荐）

组织可定义私有 purpose，但必须满足：

```text
- 不能使用 StandardPurpose 中的字符串
- 默认不能跨组织使用
- 必须在 Agent Card 中声明命名空间与 registry hash
- 必须使用同一 CapabilityTriple schema
- 必须经 PolicyKernel 精确匹配，不能落回字符串匹配
```

强烈建议优先使用标准词汇表。

## 12. 与旧版的差距

```text
旧版                         本文档
----                         ----
purpose 自由文本              标准 purpose + registry hash
allowed_ops 字符串            CapabilityTriple 结构化精确匹配
retrieve:L1 等混合命名        capability / layer / action 分离
FME Mandate 独立 schema       FAP-1 Mandate + FME constraints
contains / 前缀实现风险       禁止模糊匹配
```
