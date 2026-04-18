# Suna 数据库与服务拆分清单

## 一、文档目标

本文是对 **Suna 商业化技术实施路线图** 的进一步细化，重点回答两个工程问题：第一，数据库层面应该拆出哪些核心域、哪些表、哪些读写边界；第二，服务层面应该如何从当前偏单体的平台 API，逐步演进为更适合多租 SaaS 的分层服务体系。[1] [2] [3]

这里的“拆分”并不意味着一开始就做彻底的物理微服务化。更现实也更稳妥的做法是：**先做逻辑域拆分、接口拆分和表模型拆分，再根据瓶颈逐步做数据库物理隔离与服务独立部署。** 这样既能避免过早复杂化，也能保证后续商业化时不会被单体结构卡死。

## 二、拆分总原则

Suna 当前已经具备 `accounts`、`account_members`、`sandboxes`、`pool_sandboxes` 等对象，以及平台 API、provider、计费与沙箱生命周期逻辑。[3] [4] 但这些对象仍然偏向“账户 + 长期机器”的产品抽象。要演进为商业化平台，数据库和服务都应围绕 **tenant / workspace / project / task / usage / policy** 这些更稳定的 SaaS 对象重新组织。

拆分时建议遵循三个原则。第一，**商业对象与执行对象分离**，即 tenant、workspace、subscription、invoice 不应与 sandbox、task_run、step_event 混在一套服务边界里。第二，**控制面状态与执行面状态分离**，即 RBAC、policy、secret、billing 应尽量留在控制面；浏览器态、工作目录、进程树等留在执行面。第三，**高变更高吞吐数据与低变更主数据分离**，例如 usage events、step logs、artifact metadata 与 tenant profile、workspace settings 应当分别建模，避免互相拖累。

| 拆分原则 | 含义 | 对数据库/服务的影响 |
|---|---|---|
| 商业对象与执行对象分离 | 合同、租户、套餐与任务、沙箱分开 | billing/tenancy 不与 sandbox 强耦合 |
| 控制面与执行面分离 | 策略、权限、密钥与 runtime 状态分开 | policy/secret 不落在 sandbox 服务里 |
| 主数据与事件数据分离 | 低频配置与高频日志/计量分开 | OLTP 表与 usage/event 表分层 |

## 三、建议的数据库域拆分

从数据库视角看，建议将整体数据模型拆为 **八个逻辑域**：Identity & Tenancy、Projects & Collaboration、Tasks & Execution、Sandbox Registry & Pooling、Secrets & Integrations、Usage & Billing、Policy & Audit、Artifacts & Knowledge。短期内这些域可以仍然共用一个 PostgreSQL 实例；中期可拆为多个 schema；长期再按写入压力和组织边界逐步独立数据库。

| 数据库逻辑域 | 主要职责 | 建议阶段 |
|---|---|---|
| Identity & Tenancy | 用户、租户、工作区、成员、角色 | Phase 0-1 |
| Projects & Collaboration | 项目、上下文边界、共享设置 | Phase 1 |
| Tasks & Execution | 任务、运行、步骤、状态机 | Phase 1 |
| Sandbox Registry & Pooling | 沙箱、provider、池化、快照 | Phase 2 |
| Secrets & Integrations | 集成、凭证、密钥映射、授权 | Phase 1 |
| Usage & Billing | usage events、账单项、订阅、额度 | Phase 1-3 |
| Policy & Audit | 策略、决策、审计日志 | Phase 1-3 |
| Artifacts & Knowledge | 工件、文件索引、记忆索引、知识对象 | Phase 1-2 |

## 四、数据库拆分清单

### 4.1 Identity & Tenancy 域

这一域是整个平台的根。它用于表达谁在使用系统、谁为系统付费、谁属于哪个协作边界，以及后续 RBAC、审计、计费、project 归属的根键。

建议新增或重构以下核心表：

| 表名 | 用途 | 关键字段 | 说明 |
|---|---|---|---|
| `tenants` | 付费与合同主体 | `tenant_id`, `name`, `plan_id`, `status` | 高于现有 account 概念 |
| `tenant_members` | 用户与租户关系 | `tenant_id`, `user_id`, `role` | 顶层成员关系 |
| `workspaces` | 团队/业务边界 | `workspace_id`, `tenant_id`, `name`, `visibility` | 商业 SaaS 的核心协作单元 |
| `workspace_members` | 用户与工作区权限 | `workspace_id`, `user_id`, `role` | 支持更细粒度授权 |
| `workspace_settings` | 工作区级配置 | `workspace_id`, `default_region`, `allowed_models` | 策略与默认值 |
| `plans` | 套餐定义 | `plan_id`, `name`, `limits`, `features` | 给 billing 与 policy 共用 |

**迁移建议** 是保留现有 `accounts` 作为兼容层，将其逐步映射为 `tenant` 或 `workspace`。如果当前产品更偏“一账户对应一团队”，可以先将 `accounts -> workspaces`，再引入 `tenants` 作为更高层商业主体。

### 4.2 Projects & Collaboration 域

Suna 当前的问题不是没有共享，而是共享边界过粗。Project 域的目标，就是把过去默认挂在 account 或 sandbox 上的上下文、凭证、文件与协作关系，收敛到更合理的业务边界上。

| 表名 | 用途 | 关键字段 | 说明 |
|---|---|---|---|
| `projects` | 业务上下文边界 | `project_id`, `workspace_id`, `name`, `status` | 核心共享边界 |
| `project_members` | 项目成员关系 | `project_id`, `user_id`, `role` | 支持跨工作区受限协作可选 |
| `project_settings` | 项目默认执行策略 | `project_id`, `default_execution_class`, `retention_policy` | 影响调度与回收 |
| `project_contexts` | 项目级上下文入口 | `project_id`, `context_type`, `ref_id` | 连接知识库、工件、仓库等 |
| `project_integrations` | 项目授权的集成映射 | `project_id`, `integration_id`, `scope` | 明确集成边界 |

这一域与 tenancy 域强耦合，但与 sandbox 不应直接强耦合。也就是说，项目不是“某台机器里的一个文件夹”，而是平台控制面里的正式对象。

### 4.3 Tasks & Execution 域

这是从“实例中心”切换到“任务中心”的关键域。所有 agent 行为，无论最终落在哪类 sandbox 上，都应该具备统一的任务执行记录。

| 表名 | 用途 | 关键字段 | 说明 |
|---|---|---|---|
| `tasks` | 任务定义对象 | `task_id`, `workspace_id`, `project_id`, `created_by`, `intent` | 用户可见任务 |
| `task_runs` | 一次具体执行尝试 | `task_run_id`, `task_id`, `status`, `sandbox_id`, `started_at` | 用于重试与计费 |
| `task_steps` | 步骤级执行记录 | `step_id`, `task_run_id`, `step_type`, `status`, `started_at` | 对应工具调用或计划节点 |
| `task_step_events` | 过程事件 | `step_event_id`, `step_id`, `event_type`, `payload` | 流式日志与状态流 |
| `task_results` | 最终结果索引 | `task_run_id`, `summary`, `result_ref` | 用户最终可见输出 |
| `task_bindings` | 任务与资源绑定 | `task_run_id`, `sandbox_id`, `project_id`, `execution_class` | 便于调度分析 |

建议把 `task_steps` 与 `task_step_events` 分开。前者是状态机节点，后者是高吞吐事件流。否则表会快速膨胀，影响任务主查询。

### 4.4 Sandbox Registry & Pooling 域

Suna 已经有 `sandboxes`、`pool_resources` 和 `pool_sandboxes` 基础。[4] 这一域不需要推翻重来，但需要从“账户机器表”升级为“执行资源注册表”。

| 表名 | 用途 | 关键字段 | 说明 |
|---|---|---|---|
| `sandboxes` | 正式分配给租户/项目的执行环境 | `sandbox_id`, `tenant_id`, `workspace_id`, `project_id`, `provider`, `status` | 当前表可扩展 |
| `sandbox_allocations` | 资源分配记录 | `allocation_id`, `sandbox_id`, `task_run_id`, `claimed_at`, `released_at` | 解决一台环境被谁用过 |
| `sandbox_snapshots` | 快照与恢复点 | `snapshot_id`, `sandbox_id`, `snapshot_type`, `status`, `created_at` | Phase 2 关键对象 |
| `sandbox_sessions` | 会话级热状态 | `session_id`, `sandbox_id`, `workspace_id`, `expires_at` | warm session 模型 |
| `pool_resources` | 池规格与库存目标 | `resource_id`, `provider`, `server_type`, `desired_count` | 现有表保留强化 |
| `pool_sandboxes` | 预热实例池 | `id`, `provider`, `status`, `metadata` | 现有表保留强化 |
| `sandbox_health_checks` | 健康检查历史 | `check_id`, `sandbox_id`, `status`, `latency_ms` | 支撑 SLA |
| `sandbox_recycle_logs` | 回收与清理记录 | `log_id`, `sandbox_id`, `action`, `result` | 审计和优化必需 |

这里最重要的新增表是 `sandbox_allocations`。因为商业化后，必须能回答“某个 task_run 在什么时间绑定了什么环境、用了多久、是谁触发的”。

### 4.5 Secrets & Integrations 域

当前 Suna 的 secret 更接近运行时注入逻辑。商用化后，需要把 secret 从“文件系统状态”提升为“平台主数据 + 临时下发凭证”。

| 表名 | 用途 | 关键字段 | 说明 |
|---|---|---|---|
| `integrations` | 集成目录 | `integration_id`, `provider`, `category`, `status` | GitHub、Slack、AWS 等 |
| `integration_connections` | 具体连接实例 | `connection_id`, `tenant_id`, `provider`, `owner_type`, `owner_id` | 连接归属 tenant/workspace/project |
| `secret_records` | 密钥元数据 | `secret_id`, `owner_type`, `owner_id`, `kind`, `rotation_policy` | 不直接存明文 |
| `secret_versions` | 加密后的版本记录 | `secret_version_id`, `secret_id`, `ciphertext_ref`, `created_at` | 支持轮转 |
| `runtime_secret_leases` | 运行时短期租约 | `lease_id`, `secret_id`, `sandbox_id`, `task_run_id`, `expires_at` | 关键安全对象 |
| `integration_scopes` | 授权范围 | `connection_id`, `resource_scope`, `permissions` | 明确最小权限 |

建议 `secret_records` 与 `secret_versions` 分开，便于做轮转、审计和外部 KMS 集成。

### 4.6 Usage & Billing 域

这一域决定平台是否真的能商业化。其关键不是 Stripe 接没接好，而是 **计量对象是否统一**。

| 表名 | 用途 | 关键字段 | 说明 |
|---|---|---|---|
| `usage_events` | 原始计量事件 | `usage_event_id`, `tenant_id`, `task_run_id`, `resource_type`, `quantity`, `unit` | Phase 0 必须定义 |
| `usage_rollups` | 聚合视图 | `rollup_id`, `scope_type`, `scope_id`, `window_start`, `window_end` | 降低查询成本 |
| `subscriptions` | 订阅关系 | `subscription_id`, `tenant_id`, `plan_id`, `status` | 对接 Stripe 等 |
| `billing_accounts` | 账务主体配置 | `billing_account_id`, `tenant_id`, `currency`, `tax_region` | 企业账务基础 |
| `billing_line_items` | 出账明细项 | `line_item_id`, `subscription_id`, `resource_type`, `amount` | 对账核心 |
| `credit_ledgers` | 额度与赠送记录 | `ledger_id`, `tenant_id`, `delta`, `reason` | 免费额度、补偿等 |
| `invoices` | 发票/账单对象 | `invoice_id`, `billing_account_id`, `period_start`, `total_amount` | 用户可见账单 |

推荐把 `usage_events` 作为事件型高写入表，必要时后续独立到单独 schema，甚至单独存储层。

### 4.7 Policy & Audit 域

策略与审计是企业化的基础，不应继续散落在各业务表中。

| 表名 | 用途 | 关键字段 | 说明 |
|---|---|---|---|
| `policies` | 策略定义 | `policy_id`, `scope_type`, `scope_id`, `policy_type`, `document` | 条件化策略 |
| `policy_bindings` | 策略绑定关系 | `binding_id`, `policy_id`, `target_type`, `target_id` | 支持多层覆盖 |
| `policy_decisions` | 裁决结果 | `decision_id`, `task_run_id`, `actor_type`, `action`, `decision`, `reason` | 可解释性基础 |
| `audit_logs` | 通用审计日志 | `audit_log_id`, `actor_id`, `action`, `resource_type`, `resource_id` | 全平台必备 |
| `admin_actions` | 管理员高风险行为 | `action_id`, `admin_id`, `target_type`, `payload` | 企业审计强化 |
| `compliance_exports` | 导出记录 | `export_id`, `tenant_id`, `export_type`, `status` | 企业客户常见要求 |

### 4.8 Artifacts & Knowledge 域

Suna 的长期状态优势，最终要沉淀到可控的工件和知识对象，而不是全靠 runtime 内部状态。

| 表名 | 用途 | 关键字段 | 说明 |
|---|---|---|---|
| `artifacts` | 文件与结果索引 | `artifact_id`, `task_run_id`, `project_id`, `type`, `storage_ref` | 所有产出统一归档 |
| `artifact_access_logs` | 工件访问记录 | `log_id`, `artifact_id`, `actor_id`, `action` | 企业审计需要 |
| `knowledge_items` | 结构化记忆对象 | `knowledge_id`, `project_id`, `kind`, `content_ref` | 替代隐式长期记忆 |
| `knowledge_indexes` | 检索索引 | `index_id`, `knowledge_id`, `embedding_ref` | 向量检索可接入 |
| `repo_links` | 代码仓与项目绑定 | `link_id`, `project_id`, `provider`, `repo_ref` | 软件开发场景重要 |

## 五、建议的服务拆分清单

从服务视角，建议分成 **九个逻辑服务**。在 Phase 0-1，这些服务可以仍然共处同一进程，但要先清楚每个服务的责任边界、接口与数据所有权。到了 Phase 2-3，再根据吞吐与团队边界做物理拆分。

| 服务 | 主要职责 | 主要拥有的数据域 |
|---|---|---|
| Identity Service | 用户身份、登录态、服务身份、签名令牌 | Identity |
| Tenancy Service | tenant/workspace/project/member/RBAC | Tenancy、Projects |
| Task Orchestration Service | tasks、task_runs、task_steps、状态机 | Tasks & Execution |
| Sandbox Manager Service | provider、sandbox 生命周期、池化、回收 | Sandbox Registry |
| Secret & Integration Service | 密钥、连接、令牌租约、集成目录 | Secrets & Integrations |
| Usage Metering Service | usage event 接收、校验、聚合 | Usage |
| Billing Service | 订阅、账单项、额度、发票 | Billing |
| Policy & Audit Service | 策略裁决、审计记录、导出 | Policy & Audit |
| Artifact & Knowledge Service | 工件索引、知识对象、文件访问 | Artifacts & Knowledge |

### 5.1 Identity Service

该服务负责用户认证、服务对服务认证、sandbox 身份与签名令牌，不应继续由各业务模块各自处理 token 逻辑。它不一定需要脱离现有 auth 提供方，但需要成为平台统一的身份适配层。

### 5.2 Tenancy Service

Tenancy Service 是商用化改造的第一优先级服务。它拥有 tenant、workspace、project、membership、role assignment 等核心对象，并对外提供资源归属、角色检查和基础授权查询。它不负责复杂策略裁决，但负责回答“某人属于哪个边界”。

### 5.3 Task Orchestration Service

这是平台的任务控制面。它负责任务创建、运行、步骤状态机、恢复点、任务绑定、工件挂接与结果归档。即便底层仍调用 OpenCode runtime，这个服务也应该成为“平台视角下的任务真相源”。

### 5.4 Sandbox Manager Service

该服务承接当前 provider 初始化、pooling、claim/release、health check、snapshot、TTL、回收等逻辑。建议 Phase 2 从 API 中独立出来，因为它会逐渐成为高并发、高状态变化的基础设施服务。

### 5.5 Secret & Integration Service

该服务不只是“保存密钥”，更负责 OAuth 连接、权限 scope、临时租约签发和运行时吊销。它必须与 policy、audit 强联动。

### 5.6 Usage Metering Service

该服务是所有资源事件的统一入口。模型调用、浏览器会话、沙箱 CPU/内存时长、artifact 带宽、出网流量都应汇入这里。它与 Billing Service 分离，原因是“采集”与“定价”不应耦合。

### 5.7 Billing Service

Billing Service 负责 subscription、credit、rating、invoice、payment provider adapter。它消费 Usage Metering Service 的聚合数据，而不是直接从运行时抓数据。

### 5.8 Policy & Audit Service

前期可只做审计写入与简单策略查询；后期升级为完整 Policy Engine。之所以建议单独成域，是因为它最终会成为企业功能的核心，不宜寄生在 tenancy 或 task 服务中。

### 5.9 Artifact & Knowledge Service

这个服务把“长期共享状态”从黑箱 runtime 中抽出一部分，变成平台可治理对象。它可以统一接对象存储、向量索引与代码仓引用。

## 六、服务拆分顺序建议

不建议一开始就把九个服务全部独立部署。更稳妥的顺序如下：

| 顺序 | 建议先后 | 原因 |
|---|---|---|
| 1 | Tenancy Service | 决定所有资源归属 |
| 2 | Task Orchestration Service | 没有任务真相源就无法平台化 |
| 3 | Usage Metering Service | 先计量再优化再计费 |
| 4 | Secret & Integration Service | 企业客户接入前必须可控 |
| 5 | Policy & Audit Service | 权限与企业治理基础 |
| 6 | Sandbox Manager Service | 执行面复杂后必须独立 |
| 7 | Billing Service | usage 稳定后再做定价闭环 |
| 8 | Artifact & Knowledge Service | 作为长期状态治理层逐步显式化 |
| 9 | Identity Service 物理独立化 | 可后置，逻辑上先统一即可 |

也就是说，**Phase 1 更适合做逻辑服务拆分，Phase 2 再做 Sandbox Manager 和 Usage Metering 的物理独立，Phase 3 再视企业需求拆出 Policy 与 Billing。**

## 七、数据库拆分顺序建议

数据库也不建议一步走到“一个服务一个库”。更现实的路线是三步走：

第一步是 **单库多 schema / 明确 owner**。例如 `tenancy.*`、`tasks.*`、`sandbox.*`、`usage.*`、`billing.*`、`policy.*`、`artifact.*`。这一步就足以让团队先建立数据 ownership。

第二步是 **事件域先独立**。通常最先需要物理隔离的是 `usage_events`、`task_step_events`、`audit_logs` 这类高写入表。因为它们会迅速膨胀，且对主业务事务一致性的要求不同。

第三步是 **强边界域再独立**。随着企业需求增加，可逐步把 billing、policy、secret 这些边界敏感域独立数据库或独立加密存储。

| 阶段 | 数据库策略 | 推荐拆分对象 |
|---|---|---|
| Step 1 | 单库多 schema | tenancy/tasks/sandbox/usage/policy |
| Step 2 | 高吞吐事件域独立 | usage_events、task_step_events、audit_logs |
| Step 3 | 强边界高敏感域独立 | secrets、billing、policy |

## 八、现有 Suna 对象的映射建议

为了降低改造风险，建议对现有对象做映射，而不是直接废弃。

| 现有对象 | 建议映射 | 处理方式 |
|---|---|---|
| `accounts` | `tenant` 或 `workspace` 兼容层 | 过渡期双写/映射 |
| `account_members` | `workspace_members` / `tenant_members` | 逐步拆分 |
| `sandboxes` | `sandboxes` + `sandbox_allocations` | 保留主表，增加分配表 |
| `pool_resources` | `pool_resources` | 保留并增强 |
| `pool_sandboxes` | `pool_sandboxes` | 保留并增强 |
| 现有 API key / integration 逻辑 | `integration_connections` + `runtime_secret_leases` | 从静态注入转为租约模型 |

## 九、推荐的最小可行拆分版本

如果只做一个 **MVP 级商用化拆分版本**，我建议最少完成以下内容：数据库新增 `tenants`、`workspaces`、`projects`、`tasks`、`task_runs`、`task_steps`、`usage_events`、`audit_logs`、`secret_records`、`runtime_secret_leases`、`sandbox_allocations`；服务上先逻辑拆出 Tenancy、Task Orchestration、Usage Metering、Secret & Integration 四个核心模块。

这样做的原因是，这一版已经足以支撑多租基本边界、任务可追踪、usage 可计量、secret 可控，同时又不会让拆分成本过高。等这部分稳定后，再进入 Sandbox Manager、Policy、Billing 的进一步物理独立化。

## 十、最终建议

如果把整份清单压缩成一句话，那么我最推荐的拆分路线是：

> **数据库先按 tenancy / tasks / sandbox / usage / policy / artifact 六大域重组，服务先逻辑拆成 tenancy、task orchestration、usage metering、secret & integration、sandbox manager 五条主线，再按吞吐与安全边界逐步物理独立。**

这条路线的优势在于，它既不会破坏 Suna 当前已有的 runtime 与 provider 深度，又能把平台一步步推进到适合商用化运营的结构上。

## References

[1]: /home/ubuntu/suna_technical_implementation_roadmap.md "Suna 商业化改造技术实施路线图"
[2]: /home/ubuntu/suna_commercialization_recommendations.md "基于 Suna 调研的可商用化改造建议"
[3]: https://github.com/kortix-ai/suna/blob/main/packages/db/src/schema/kortix.ts "packages/db/src/schema/kortix.ts - kortix-ai/suna"
[4]: https://github.com/kortix-ai/suna/tree/main/apps/api/src "apps/api/src - kortix-ai/suna"
