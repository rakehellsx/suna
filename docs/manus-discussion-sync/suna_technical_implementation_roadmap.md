# Suna 商业化改造技术实施路线图

## 一、文档目标

本文基于此前对 **Suna** 的深度调研与商业化改造建议，进一步把方向性结论细化为一份可执行的 **技术实施路线图**。目标不是重复“应该做什么”，而是明确 **先做什么、后做什么、为什么这样排序、每一阶段交付什么、依赖什么团队、如何判断阶段完成**。路线图默认面向一个希望将 Suna 从“共享 AI Computer”演进为“多租可运营 Agent SaaS 平台”的团队。[1] [2] [3]

从现状看，Suna 已经具备平台 API、账户模型、沙箱 provider、多种执行环境、OpenCode runtime、预览代理和计费雏形，这意味着它不需要从零搭平台；真正要补的是 **多租控制面、任务化编排、分层执行资源、可审计状态与统一计费内核**。[1] [2] [3] [4] 因此，这份路线图的核心原则是：**保留现有 runtime 深度，优先补控制面与治理层，再推进执行面弹性化，最后再做企业级增强。**

## 二、路线图设计原则

路线图的排序遵循四个原则。第一，优先处理 **平台级不可逆技术债**，例如租户模型、权限边界、任务对象和 usage 事件模型；这些对象如果早期定义不清，后续所有计费、审计和产品分层都会重做。第二，优先构建 **可观测与可回放能力**，因为没有可靠的任务生命周期记录，沙箱优化和商业计费都会失真。第三，把高成本的执行资源优化放在控制面之后，因为没有策略与计量，复用和池化往往会先带来风险，再带来收益。第四，把企业特性放在后期集中建设，因为它们依赖前面的对象模型和策略引擎。

| 设计原则 | 含义 | 对排序的影响 |
|---|---|---|
| 先控制面后执行面 | 先定义租户、任务、权限、计量对象 | 避免后续执行层返工 |
| 先可观测后优化 | 先拿到 usage 与 trace，再做资源优化 | 防止“优化了但不可计量” |
| 先标准化再高级化 | 先统一模型，再做 dedicated / VPC / SLA | 便于产品分层 |
| 保留 Suna 优势 | 不抹掉长期状态与共享上下文能力 | 形成差异化竞争力 |

## 三、目标架构蓝图

建议最终目标架构分为三层：**Tenant Control Plane**、**Agent Orchestration Plane** 与 **Execution Plane**。Tenant Control Plane 负责租户、工作区、项目、RBAC、策略、密钥、套餐、配额、账单、审计和集成目录。Agent Orchestration Plane 负责任务创建、计划、步骤执行、工件归档、记忆索引、恢复点和调度决策。Execution Plane 则继续承载 Suna 现有的 OpenCode runtime、终端、浏览器、文件系统与实际进程执行。[1] [2] [3]

这三层不是三个完全独立的物理集群，也不要求一次性全部拆分成微服务。更现实的做法是 **先做逻辑分层，再按瓶颈逐步物理拆分**。例如第一阶段可以仍然保留单一 API 服务，但先把 tenancy、tasks、usage、policy 与 sandbox provider 的模块边界明确下来；第二阶段再把 usage pipeline、scheduler 和 sandbox manager 独立出来；第三阶段再为企业能力引入专门的 policy service 和 audit service。

## 四、阶段总览

这份路线图建议拆为 **四个实施阶段**，总周期可以按 6 到 12 个月组织，具体取决于团队规模与现有代码可控程度。四个阶段分别是：Phase 0 架构定标与观测补齐，Phase 1 多租控制面落地，Phase 2 执行面弹性化与沙箱分层，Phase 3 企业级治理与商业分层。

| 阶段 | 目标定位 | 建议周期 | 主要成果 |
|---|---|---|---|
| Phase 0 | 架构定标与可观测补齐 | 2-4 周 | 对象模型、事件模型、系统基线 |
| Phase 1 | 从共享机器走向可治理平台 | 8-12 周 | tenancy / workspace / project / task / usage 基础能力 |
| Phase 2 | 从长期独占实例走向弹性执行资源 | 8-12 周 | dedicated / warm / ephemeral、分层复用、调度与回收 |
| Phase 3 | 从可用产品走向企业可采购平台 | 8-16 周 | policy engine、VPC、合规审计、企业计费和 SLA |

## 五、Phase 0：架构定标与观测补齐

Phase 0 的目标不是发布用户可见功能，而是确保后面不会在错误的底座上狂奔。这个阶段需要先把几个最关键的 **平台对象** 定义下来，包括 `tenant`、`workspace`、`project`、`task`、`task_run`、`task_step`、`artifact`、`usage_event`、`policy_decision`、`audit_log`。这些对象中的大部分在当前 Suna 中要么不存在，要么散落在 account、sandbox 和内部逻辑中。[3] [4]

同时，这一阶段必须建立 **统一 trace 与 usage 事件模型**。建议所有模型调用、沙箱调用、浏览器会话、文件工件上传、网络代理和长时任务都生成结构一致的 usage 事件，并强制携带 `tenant_id`、`workspace_id`、`project_id`、`task_run_id`、`sandbox_id`、`provider`、`resource_type`、`quantity`、`unit`、`started_at`、`ended_at` 等字段。没有这一步，后面所有“优化复用率”“做更细计费”“看单租户成本”都会缺乏可信数据。

Phase 0 还要完成一次 **代码边界梳理**。建议把当前 API 服务中的平台层逻辑，至少逻辑上拆成以下子模块：identity、tenancy、projects、tasks、sandbox registry、sandbox manager、usage pipeline、billing adapter、policy、integrations、secret manager、artifact service。哪怕这些模块暂时仍在一个仓库或一个进程里，也要先在代码结构上做隔离。

| Phase 0 工作项 | 说明 | 输出物 |
|---|---|---|
| 领域模型定稿 | 统一 tenancy / task / usage / audit 对象 | ERD、表结构草案 |
| 事件模型定稿 | 统一 usage 与 trace schema | 事件 schema 文档 |
| 模块边界梳理 | 重画 API 内部模块边界 | 模块图、责任矩阵 |
| 基线指标接入 | 记录任务数、沙箱时长、冷启动、失败率 | Dashboard v1 |
| 迁移策略设计 | 明确 account 到 tenant/workspace 的迁移方式 | 数据迁移方案 |

### Phase 0 里程碑

完成 Phase 0 的标志，不是代码量，而是以下几个问题有了明确答案：第一，系统中的“商业对象”和“执行对象”是否已分离；第二，单次任务与长期实例之间的映射关系是否可表达；第三，usage 事件是否可统一采集；第四，后续各团队是否围绕同一套对象模型工作。如果这些问题没有定稿，不建议进入大规模开发。

## 六、Phase 1：多租控制面落地

Phase 1 是整个商用化路线图里最关键的阶段。它的任务是把当前以 `account + sandbox` 为中心的系统，升级为以 **tenant / workspace / project / task** 为中心的平台。[3] 这一阶段完成后，Suna 才真正从“共享机器产品”进入“可治理平台”的范畴。

第一项核心工作是 **tenancy 模型改造**。建议保留现有 `accounts` 兼容层，但在新模型中引入 `tenant` 作为付费与合同主体，`workspace` 作为协作边界，`project` 作为上下文边界。为了避免一次性大爆炸迁移，可以先做双写：老接口仍读取 account，新功能开始写入 workspace / project，并在 API 层引入映射层。

第二项核心工作是 **任务系统落地**。要引入 `tasks`、`task_runs`、`task_steps`、`artifacts` 四类核心表，并在所有实际执行链路里强制绑定 `task_run_id`。这意味着即便一个 agent 最终仍跑在长期实例里，平台也要知道这次执行是什么任务、执行了哪些步骤、产出了哪些工件、用了多少资源。

第三项核心工作是 **Secret Manager 与 Integration Boundary**。当前 Suna 更接近以 sandbox 为中心的集成注入方式；Phase 1 应把它升级为平台托管式 secret 模型，即 secret 默认属于 tenant / workspace / project，由平台判断何时签发短期凭证到 runtime。只有完成这一步，后续企业客户才可能接受把生产系统接进来。

第四项核心工作是 **基础 RBAC 与审计日志**。建议最少支持 organization admin、workspace admin、project editor、project runner、viewer 五种角色。所有敏感行为，例如连接新集成、下载工件、开通浏览器能力、提升任务预算、分享预览链接，都要写审计日志。

| Phase 1 模块 | 关键改造 | 依赖 |
|---|---|---|
| Tenancy | tenant/workspace/project 新模型与迁移层 | Phase 0 对象模型 |
| Tasks | tasks/task_runs/task_steps/artifacts | 事件模型、API 模块边界 |
| Secrets & Integrations | 平台托管 secrets、短期令牌签发 | 身份体系、审计日志 |
| RBAC | 角色、资源授权、基础策略检查 | tenancy 模型 |
| Usage v1 | 按 task_run 汇总模型与沙箱消耗 | usage 事件总线 |

### Phase 1 交付标准

Phase 1 完成后，平台应至少具备以下用户可见与平台可见结果：用户可以在 workspace 和 project 内操作 agent；每次执行都能在任务列表中追踪；管理员可以查看 usage、审计和工件；敏感集成不再默认长期驻留在 sandbox 文件系统内；平台可以统计每个 workspace / project 的基本资源消耗。只要这些能力缺失，后续执行面优化就很难稳定落地。

## 七、Phase 2：执行面弹性化与沙箱分层

Phase 2 的主题是：**把当前偏长期独占的执行模型，改造成多层次执行资源池。** 这一步直接决定单位经济模型，也是将 Suna 从“托管 AI 电脑”扩展为“商用 Agent 执行平台”的关键一跃。[2] [3]

这一阶段首先要引入三类执行环境：`dedicated`、`warm_session`、`ephemeral_task`。Dedicated 保留 Suna 原有优势，用于长时在线 agent、复杂浏览器态和高价值客户；Warm Session 用于连续会话和短期项目协作；Ephemeral Task 用于纯任务型与低敏感度执行。平台调度器需要根据租户套餐、任务风险等级、是否需要浏览器登录态、是否包含敏感文件、预估时长和当前集群压力，自动选择执行环境。

第二项工作是 **分层复用与回收**。当前 Suna 已有 pool sandboxes 雏形，但商用平台需要进一步明确：跨租户只复用只读镜像层和内容寻址缓存；同租户可复用工作目录缓存；同会话才允许复用浏览器态和热状态。与此同时，要把 `/persistent` 再拆成 `persistent_safe` 与 `persistent_sensitive` 两层，从而把可保留状态与高风险状态分离。

第三项工作是 **Scheduler 与 Sandbox Manager 独立化**。随着执行环境类型增多，不能再靠简单的 provider 初始化逻辑支撑。建议引入独立的调度模块，负责容量判断、池化补货、claim / release、TTL、快照恢复、熔断和回收策略。Sandbox Manager 则负责 provider 抽象、生命周期编排、health checking 与故障恢复。

第四项工作是 **恢复点与快照机制**。Warm 和 dedicated 环境都需要更系统的恢复能力，例如工作目录索引、浏览器会话摘要、OpenCode 状态快照和安全清理后的中间状态包。这样既能提高连续体验，又能降低完全长期保活的成本。

| Phase 2 模块 | 核心能力 | 关键输出 |
|---|---|---|
| Execution Classes | dedicated / warm / ephemeral 三类环境 | 调度策略与产品分层对接 |
| Scheduler | 环境选择、容量感知、队列与熔断 | scheduler service v1 |
| Sandbox Manager | provider 适配、生命周期、恢复 | sandbox manager v1 |
| Reuse & Pooling | 镜像缓存、依赖缓存、工作区缓存、热状态复用 | 复用策略引擎 |
| Snapshot & Recovery | 快照、恢复点、回滚 | 恢复机制 v1 |

### Phase 2 交付标准

Phase 2 完成后，应能回答以下问题：同一个项目的连续任务是否不再每次全冷启动；低价值任务是否能落在低成本 ephemeral 环境；高价值客户是否能得到 dedicated 体验；系统是否能明确知道哪些状态可复用、哪些必须销毁；在高峰期是否能基于容量感知缩短 warm TTL。只有这些能力稳定，平台才具备真正可扩张的成本结构。

## 八、Phase 3：企业级治理与商业分层

Phase 3 的重点，不再是“平台能不能跑”，而是“客户愿不愿意采购”。这一阶段要补的是企业信任所需的控制能力，包括 **策略引擎、企业审计、网络隔离、私有连接、SLA 与高级计费**。

首先要建设 **Policy Engine**。Phase 1 的 RBAC 只能解决“谁能做什么”，而企业客户需要的是“在什么条件下允许什么行为”。例如：哪些项目可访问公网、哪些任务可安装系统包、哪些环境可上传外部文件、哪些 workspace 允许使用外部模型、哪些客户只能在指定地域执行。这些都应该由独立策略引擎裁决。

其次是 **网络与部署边界**。建议引入 BYOVPC / Private Link / Static Egress IP 等能力，至少在架构上预留接口。对很多企业客户来说，能否将 agent 执行限制在受控网络边界内，是采购的底线之一。

第三是 **企业级审计与合规导出**。除普通 audit log 外，还应支持工件访问日志、敏感 secret 读取记录、管理员授权变更历史、策略变更历史以及账单明细导出。

第四是 **高级计费和 SLA**。此时可以将平台正式分层为 Free、Pro、Enterprise 三档，分别映射不同的执行环境、usage 上限、审计深度、集成能力和支持等级。

| Phase 3 模块 | 核心能力 | 商业价值 |
|---|---|---|
| Policy Engine | 条件化策略、地域/网络/模型限制 | 企业信任与合规 |
| Private Connectivity | BYOVPC、私网出口、静态出口 IP | 对接企业内网系统 |
| Advanced Audit | 审计导出、策略历史、secret 访问记录 | 安全与法务支持 |
| Enterprise Billing | 分层套餐、承诺用量、超额计费、对账 | 商业闭环 |
| Reliability & SLA | SLO、告警、值班、灾备 | 企业签约能力 |

## 九、模块依赖图（逻辑顺序）

从实施依赖上看，不建议把所有模块并行推进。最合理的顺序是：先对象模型，后任务系统；先 usage 事件，后计费引擎；先 tenancy 与 RBAC，后 secret manager；先调度器，再做多环境执行；先策略引擎，再做企业网络隔离。下面的逻辑顺序可以作为项目排期的骨架。

| 优先级 | 模块 | 前置依赖 | 不宜提前的原因 |
|---|---|---|---|
| P0 | tenancy 模型 | 无 | 所有平台对象都依赖它 |
| P0 | task & artifact 模型 | tenancy、事件模型 | 没有任务对象就无法计量 |
| P0 | usage 事件总线 | 事件模型 | 后续计费与优化都依赖 |
| P1 | RBAC + audit | tenancy、tasks | 没有资源对象无法授权 |
| P1 | secret manager | RBAC、audit | 没有边界和日志不应先做 |
| P1 | scheduler / sandbox manager | tasks、usage、provider 抽象 | 没有计量和任务模型难优化 |
| P2 | dedicated / warm / ephemeral | scheduler、pooling | 需要稳定调度器 |
| P2 | billing rating engine | usage、套餐对象 | 没有 usage 先做只能返工 |
| P3 | policy engine | tenancy、RBAC、usage、audit | 需要完整上下文 |
| P3 | VPC / Enterprise features | policy、billing、SLA | 依赖整个平台成熟度 |

## 十、建议的团队编排方式

如果团队规模在 6 到 10 人之间，建议采用 **平台控制面小队、执行面小队、产品基础设施小队** 三个流并行推进，而不是按照前后端或仓库目录切分。平台控制面小队负责 tenancy、RBAC、tasks、usage、billing object、audit；执行面小队负责 sandbox manager、provider、scheduler、pooling、snapshot；产品基础设施小队负责 auth、secrets、integrations、artifact、observability 和 release engineering。

如果团队更小，例如 3 到 5 人，则建议按阶段串行推进：先控制面，再执行面，最后企业功能。此时最应避免的，是同时尝试“重写前端体验、重构 runtime、做企业功能、做新模型接入”，因为这样几乎一定失控。

| 团队流 | 负责模块 | 阶段重点 |
|---|---|---|
| 控制面小队 | tenancy、RBAC、tasks、usage、billing、audit | Phase 0-1 |
| 执行面小队 | sandbox manager、scheduler、pooling、snapshot | Phase 2 |
| 基础设施小队 | auth、secrets、integrations、observability、CI/CD | 全阶段支撑 |

## 十一、建议的 12 个月时间轴

如果用 12 个月做较稳妥的商业化演进，建议按如下节奏推进：第 1 个月完成对象模型、事件模型与迁移设计；第 2-4 个月完成 tenancy、tasks、RBAC、usage、audit 基础能力；第 5-8 个月完成 scheduler、sandbox manager、三类执行环境和分层复用；第 9-12 个月完成 policy engine、企业计费、私网能力与 SLA 体系。

这个时间轴的关键不是“12 个月一定做完”，而是让每一季度都有可验证的阶段成果。第一季度验证“平台边界对不对”；第二季度验证“成本结构能不能优化”；第三季度验证“企业客户能不能采购”。

| 时间窗口 | 重点目标 | 验证指标 |
|---|---|---|
| 月 1 | 架构定标、对象模型、事件模型 | 架构评审通过、迁移方案冻结 |
| 月 2-4 | tenancy/tasks/usage/RBAC/audit | 任务可追踪、usage 可汇总、权限生效 |
| 月 5-8 | scheduler/manager/三类环境/分层复用 | 冷启动下降、复用率提升、单位成本下降 |
| 月 9-12 | policy/VPC/billing/SLA | 企业试点上线、审计可导出、合同可签 |

## 十二、核心 KPI 与阶段验收指标

技术路线图如果没有 KPI，很容易沦为功能清单。建议每一阶段至少绑定一组平台指标。Phase 0 关注“看得见”，例如任务 trace 覆盖率、usage 事件完整率、错误分类率。Phase 1 关注“管得住”，例如 workspace/project 覆盖率、审计日志覆盖率、secret 直接落盘比例下降。Phase 2 关注“跑得省”，例如冷启动时间、warm 命中率、单位任务沙箱成本、dedicated 占比。Phase 3 关注“卖得动”，例如企业试点成功率、策略命中率、审计导出可用性和月度账单差错率。

| 阶段 | 核心 KPI | 验收方式 |
|---|---|---|
| Phase 0 | trace 覆盖率、usage 事件完整率 | 技术验收 |
| Phase 1 | workspace/project 覆盖率、审计覆盖率 | 内部灰度 |
| Phase 2 | 冷启动下降、warm 命中率、单位成本下降 | 成本与性能验收 |
| Phase 3 | 企业试点成功率、账单准确率、SLA 达标率 | 商业验收 |

## 十三、风险清单与应对策略

这条路线最容易出问题的地方有四个。第一，**对象模型迁移失败**，导致老数据与新模型长期双轨且不可收敛；应对方式是坚持兼容层有明确下线时间，并让新功能只写新模型。第二，**执行面优化先于计量能力**，导致资源池做起来了但成本不可解释；应对方式是 usage 先行。第三，**策略系统过晚**，导致企业试点时发现平台权限体系太薄；应对方式是在 Phase 1 先做基础 RBAC，Phase 3 再升级策略引擎。第四，**长期实例与任务化执行冲突**，导致产品体验摇摆；应对方式是明确把 dedicated 作为高阶模式，而平台默认采用 task / session 模式。

## 十四、最终实施建议

如果把这份路线图压缩成最关键的三句话，那么第一句是：**先把 Suna 从“实例中心”改成“任务与租户中心”**。第二句是：**先建立事件、计量与审计，再优化沙箱和执行成本**。第三句是：**不要抹掉 Suna 的长期状态优势，而应把它包装成高阶、可控、可计费的产品能力。**

因此，最推荐的执行顺序是：

> **Phase 0 先冻结对象与事件模型，Phase 1 建立 tenancy / task / usage / audit 控制面，Phase 2 完成执行面分层与调度，Phase 3 再补企业治理与商业分层。**

按照这一路线推进，Suna 最终不会只是一个“更强的开源 Agent 项目”，而会演进成一个兼具 **执行深度、控制面成熟度与商业交付能力** 的平台型产品。

## References

[1]: https://github.com/kortix-ai/suna "kortix-ai/suna - GitHub README"
[2]: https://github.com/kortix-ai/suna/blob/main/MANIFESTO.md "MANIFESTO.md - kortix-ai/suna"
[3]: https://github.com/kortix-ai/suna/blob/main/core/docker/docker-compose.yml "core/docker/docker-compose.yml - kortix-ai/suna"
[4]: https://github.com/kortix-ai/suna/tree/main/apps/api/src "apps/api/src - kortix-ai/suna"
[5]: https://github.com/kortix-ai/suna/blob/main/packages/db/src/schema/kortix.ts "packages/db/src/schema/kortix.ts - kortix-ai/suna"
