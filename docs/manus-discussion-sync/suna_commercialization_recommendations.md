# 基于 Suna 调研的可商用化改造建议

## 一、执行摘要

基于前面的调研，**Suna 已经具备相当强的产品雏形**：它不是一个只会对话的 Agent 前端，而是一个包含账户、平台 API、沙箱供应、OpenCode runtime、代理、计费和集成系统的完整开源体系。[1] [2] [3] 但它的原生产品哲学更偏向 **“长期在线的共享 AI 计算机”**，默认强调组织内部的共享上下文、长期记忆与同一实例上的多 agent 协作。[1] [4]

这套思路非常适合单组织、单团队、长期运营型 AI workforce 场景；但如果目标是做 **商业化公有云 SaaS**、服务多个客户、支持工作区、多成员、配额、审计、SLA 与细粒度计费，那么 Suna 还需要从“共享机器产品”进一步演进为“多租治理型 Agent 平台”。换句话说，Suna 目前最强的是 **Agent 计算机范式**，而要商用，需要再补上一整层 **租户治理、资源抽象、弹性调度与商业化控制面**。[1] [2] [3]

因此，最合理的商用化路线并不是推翻 Suna，而是保留其 **OpenCode + 长期状态 + 工具执行 + 机器抽象** 的优势，同时对其进行一次明确的 **平台化重构**：把“共享机器”重新封装为“租户控制下的受控工作环境”，把“实例级共享”进一步细化为“账户、工作区、项目、任务”四级边界，把“长期驻留 runtime”升级为“可池化、可复用、可审计、可计费的执行资源”。

## 二、改造目标：从共享 AI Computer 走向多租商用平台

从调研结果看，Suna 当前的强项与短板都很清晰。强项在于它已经有真实 Linux 沙箱、持久化状态分层、平台 API、provider 抽象、preview proxy、计费代码和沙箱池化雏形。[2] [3] [5] 短板在于它的默认产品抽象仍然偏“每个账户拥有一台长期机器”，这会让租户治理、资源利用率、隔离强度和成本模型更偏向 **托管计算机**，而不是 **标准化 Agent 平台**。[1] [4]

所以，商业化改造的目标不应只写成“支持更多用户”，而应明确成以下四条：第一，增强 **多租隔离**，避免上下文、凭证、文件和浏览器状态在错误边界内共享；第二，提高 **资源效率**，让沙箱和远程机器从“长期独占”走向“可池化、可分层复用”；第三，建立 **平台控制面**，把权限、审计、计费、风控、任务编排从 runtime 中抽离出来；第四，形成 **分层产品路线**，让 BYOC、私有化和托管云三条交付模式能共享同一内核。

| 商用化目标 | 现状 | 改造方向 |
|---|---|---|
| 多租隔离 | 以 account 为主，组织内共享色彩强 | 增加 workspace / project / task 级边界 |
| 资源效率 | 偏长期实例、偏独占机器 | 引入池化、快照、分层复用 |
| 平台治理 | 已有 API、认证、计费雏形 | 强化审计、权限、配额、SLA 控制 |
| 商业交付 | BYOC + 托管雏形 | 标准化为 Free / Pro / Enterprise 分层 |

## 三、总体架构改造建议

最重要的建议是：**保留 Suna 的 runtime 优势，但改变默认产品抽象。** 目前 Suna 的叙事是“每个账户有一台长期在线的 AI 计算机”；商用平台则更需要“每个租户有一个受控的 Agent 工作空间，平台按任务和会话调度执行环境”。这两者并不冲突，但后者必须成为控制面的第一原则。

因此，我建议把整体架构改造成三层：上层是 **Tenant Control Plane**，中层是 **Agent Orchestration Plane**，下层是 **Sandbox / Machine Execution Plane**。Tenant Control Plane 负责账户、工作区、项目、成员、权限、套餐、配额、审计、计费、策略和密钥管理；Agent Orchestration Plane 负责计划、工具路由、任务状态、会话记忆索引、重试和调度；Execution Plane 则继续承载 OpenCode runtime、终端、浏览器、文件系统和实际进程执行。

这一步的意义非常大，因为它会把“机器状态”和“产品状态”拆开。机器状态应该留在沙箱；而任务、计费、权限、配额、审计、策略等产品状态，应该尽量留在平台控制面。只有这样，Suna 才能从一个优秀的自托管 AI computer，演进成一个可运营、可治理、可大规模托管的 SaaS 平台。

| 分层 | 主要职责 | 是否建议从现有 Suna 中抽离 |
|---|---|---|
| Tenant Control Plane | 租户、工作区、套餐、配额、RBAC、审计、密钥管理 | 强烈建议抽离并强化 |
| Agent Orchestration Plane | 任务、步骤、记忆索引、调度、重试、工具路由 | 建议显式化 |
| Execution Plane | OpenCode runtime、终端、浏览器、文件系统、进程 | 保留并增强 |

## 四、租户模型改造：从 Account 扩展到 Workspace / Project / Task

当前 Suna 的实现已经有 `accounts`、`account_members` 和 `sandboxes` 等对象，说明它有账户边界，也支持多成员。[5] 但对于商用平台来说，这个模型还不够细。因为商业 SaaS 往往并不是“一个账户 = 一台机器 = 全部上下文”，而是需要更细粒度地表示多个团队、多个项目、多个安全域和多个执行上下文。

我建议新增并强化四级对象：`tenant`、`workspace`、`project`、`task_run`。其中 `tenant` 对应付费主体，`workspace` 对应团队或业务单元，`project` 对应一个共享上下文域，例如某个客户项目、某个产品线或某个仓库集合，`task_run` 则对应具体执行实例。这样做的直接收益是：你可以把长期共享上下文收敛在 project 或 workspace 内，而不是整个账户级别无限膨胀。

这一步不是为了把 Suna 改成“没有长期记忆”，而是为了让长期记忆 **有合法边界**。例如：一个销售团队的知识不应默认与一个财务自动化 agent 共用；一个客户 A 的浏览器登录态不应与客户 B 的文档处理 agent 共享。商用系统真正需要的是“有选择的共享”，而不是“默认全局共享”。

| 新对象 | 建议作用 | 与现有 Suna 的关系 |
|---|---|---|
| tenant | 计费与合同主体 | 高于 account 的商业对象 |
| workspace | 团队与权限边界 | 承接多人协作 |
| project | 上下文、凭证、文件的业务边界 | 拆解原本过大的共享实例 |
| task_run | 单次执行记录 | 用于调度、审计与计费 |

## 五、沙箱与机器模型改造：从长期独占走向分层资源池

Suna 当前已经具备多个 provider、pool_sandboxes 和预热资源的基础。[2] [5] 这是非常好的起点，因为商用化的关键之一就是 **把“机器”转化为“可调度资源”**。但我建议进一步把执行环境明确分成三类：**Dedicated Sandbox**、**Warm Session Sandbox**、**Ephemeral Task Sandbox**。

Dedicated Sandbox 适合高价值客户、长时间在线 agent、复杂浏览器登录态和持续工作流；Warm Session Sandbox 适合同一会话或同一项目内的短期连续交互；Ephemeral Task Sandbox 则适合纯任务型、可重建、低敏感度的执行需求。这样一来，Suna 就不再只有“长期机器”这一种产品抽象，而能针对不同成本和隔离等级做差异化交付。

这里的关键不是取消长期实例，而是 **把长期实例变成高阶服务，而不是默认服务**。默认用户应该优先落到 warm / ephemeral 体系中，只有高价值或确实需要长期状态沉淀的场景，才升级到 dedicated 机器。这样做能显著改善资源利用率，也更适合商业化定价。

| 执行环境类型 | 适用场景 | 成本 | 隔离 | 推荐套餐 |
|---|---|---|---|---|
| Dedicated Sandbox | 长时间在线 agent、持续浏览器态、复杂项目 | 高 | 强 | Enterprise / Pro+ |
| Warm Session Sandbox | 连续对话、短期项目迭代 | 中 | 中强 | Pro |
| Ephemeral Task Sandbox | 报告生成、代码运行、一次性任务 | 低 | 最强 | Free / 基础版 |

## 六、沙箱复用策略改造：从实例复用走向分层复用

Suna 已经开始处理 pool 和 claim，但对商用平台来说，还需要更进一步的 **分层复用设计**。我建议把复用对象分成四层：只读镜像层、依赖缓存层、工作目录层、会话热状态层。跨租户复用只发生在前两层；同租户复用集中在工作目录层；同会话复用只保留热状态层。这样既能降低冷启动成本，又不会轻易突破租户边界。

从具体实现上看，可以保留现有 `/workspace`、`/persistent`、`/ephemeral` 设计，但要对 `/persistent` 再做拆分。当前 `/persistent` 里混合了 OpenCode DB、auth、browser profile、secret 等高敏感状态。[3] 对商用平台来说，这些状态不应该默认随实例长期驻留，也不应该在不加策略的情况下被任意复用。更合理的做法是把 `/persistent` 进一步拆为 **safe persistent state** 与 **sensitive session state**，前者可做短 TTL 保留，后者应更严格地绑定主体与会话。

> 商用多租环境下，最应该复用的是镜像、缓存和工作区索引；最不应该默认复用的是浏览器态、密钥、令牌和隐式上下文。

## 七、权限与安全模型改造

Suna 已经有双向认证、sandbox token 注入和 cross-user isolation 测试，但这还只是基础。[2] [5] 真正商用化后，安全模型应从“认证是否通过”提升到“**策略是否允许**”。我建议把权限系统做成四层：身份层、能力层、资源层、行为层。

身份层决定调用者是谁；能力层决定该租户是否可使用浏览器、shell、网络或部署能力；资源层决定其可访问哪些 project、哪些 secrets、哪些集成；行为层则决定其能否执行特定动作，例如是否允许写外网、是否允许 apt install、是否允许上传外部文件、是否允许调用高成本模型。只有把权限拆到这一层，Suna 才能真正服务企业客户。

此外，建议把 OpenCode 内的权限与平台权限彻底区分开。平台应成为 **最终策略裁决者**，而 runtime 只负责执行被批准的动作。否则一旦 runtime 内部能力过强，平台控制面就很容易被架空。

| 权限层 | 需要控制的内容 | 改造建议 |
|---|---|---|
| 身份层 | user、service、sandbox、admin | 统一身份体系与签名令牌 |
| 能力层 | shell、browser、deploy、network、MCP | 做成套餐与策略开关 |
| 资源层 | workspace、project、secret、integration | 显式授权，不靠隐式继承 |
| 行为层 | apt install、出网、长时任务、共享链接 | 引入策略引擎与审计 |

## 八、秘密管理与集成改造

Suna 非常强调共享凭证和集成，这是其“AI computer”优势之一。[1] [4] 但这恰恰也是商用化中的高风险点。建议把当前以 sandbox 为中心的 secret 注入模型，改造为 **平台密钥托管 + 短期凭证下发 + 最小权限代理访问** 三段式架构。

也就是说，默认情况下，集成凭证不应该长驻在 runtime 文件系统中，而应优先托管在平台侧的 Secret Manager 中；沙箱需要访问时，由平台签发短期访问令牌，或通过平台代理代发请求；只有在确实需要本地 CLI 或浏览器交互的场景，才把有限期的秘密注入到 sandbox，并在任务完成或 TTL 到期后主动回收。

这样做的商业价值非常高。企业客户并不只关心“能不能连上 GitHub / Slack / AWS”，更关心 **凭证是否可审计、可撤销、可轮转、可按项目隔离**。这类能力是 SaaS 成交的前提条件之一。

## 九、任务与状态管理改造

Suna 当前更擅长“长期实例里的连续工作”，但商用平台往往还需要强任务语义。建议在控制面中新增 `tasks`、`task_runs`、`task_steps`、`task_artifacts`、`task_usage_events` 等对象，让每次实际执行都具备可追踪生命周期。这样既不会削弱其长期上下文优势，又能补齐企业最关心的 **审计、重试、失败恢复和成本归因**。

更具体地说，可以保留实例内长期记忆，但把一次执行显式记录为：任务创建、计划生成、沙箱绑定、步骤执行、工件归档、资源回收、usage 冻结、账单换算。这样一来，Suna 就从“有持续上下文的 agent computer”进化成“有持续上下文、同时又具备企业级执行记录的 agent platform”。

## 十、计费模型改造

从代码上看，Suna 已有 Stripe、credits 和付费托管机器供应逻辑。[2] 但商用平台的计费不能只按“是否给你一台托管机器”来分。建议把计费拆成四层：**seat / workspace fee、model usage、sandbox usage、premium capability surcharge**。

seat / workspace fee 用来覆盖控制面和协作能力；model usage 对应 token 或推理成本；sandbox usage 对应 CPU/内存/时长/浏览器时长/出网成本；premium capability surcharge 则对应如长时常驻 agent、dedicated machine、私网连接、合规审计、企业支持等高级能力。这样定价会比单纯卖“机器套餐”更灵活，也更能反映真实成本结构。

同时，建议把 usage event 采集做成统一总线。模型网关、sandbox provider、代理服务、文件服务和浏览器服务都只负责上报自己最可信的原始计量，最终由统一的 rating engine 生成账单项。这样未来无论是 Free、Pro、Enterprise，还是 BYOC 与托管混合模式，都能在一个计费内核里统一处理。

| 计费层 | 计费对象 | 适用原因 |
|---|---|---|
| Seat / Workspace | 用户席位、团队空间 | 反映协作与控制面价值 |
| Model Usage | 输入输出 token、模型调用 | 对应推理成本 |
| Sandbox Usage | CPU、内存、时长、浏览器、带宽 | 对应执行成本 |
| Premium Capability | dedicated、SLA、VPC、审计、私有模型 | 对应企业增值 |

## 十一、产品分层与 GTM 建议

如果直接把当前 Suna 作为统一产品推向市场，容易陷入一个问题：轻用户觉得太重，企业用户又觉得治理不够。因此更合理的路线是做三层产品。

第一层是 **BYOC / Self-Hosted**，继续保留现有 Suna 的强项，让技术型团队快速体验长期在线 agent computer。第二层是 **Managed Pro**，由平台托管 warm sandbox 和部分 dedicated sandbox，提供较完善的 UI、project、task、usage 与集成功能。第三层是 **Enterprise**，提供独立控制平面、VPC 接入、审计日志导出、策略引擎、自定义模型网关与更强的隔离配置。

这种分层的好处在于：现有开源生态可以直接成为获客入口，而真正赚钱的部分来自受控托管、治理能力和企业集成。换句话说，Suna 的开源价值更多在于 **证明 runtime 与产品体验**，商业化价值则更多来自 **控制面和托管能力**。

## 十二、实施优先级建议

如果只从落地效率和商业价值来看，我建议分三期推进。第一期先补最关键的 **多租控制面能力**，包括 workspace / project 模型、任务对象、usage 采集、审计日志和 secret manager。第二期再做 **沙箱与执行面重构**，包括 dedicated / warm / ephemeral 三类执行环境、分层复用、快照恢复和复用策略。第三期再向企业版深化，加入策略引擎、BYOVPC、合规日志、客户托管密钥与更强 RBAC。

| 阶段 | 目标 | 核心交付 |
|---|---|---|
| Phase 1 | 从共享机器走向可治理平台 | workspace/project/task、审计、usage、secret manager |
| Phase 2 | 从独占实例走向弹性执行资源 | dedicated/warm/ephemeral、池化、快照、分层复用 |
| Phase 3 | 从产品可用走向企业可采购 | 策略引擎、VPC、合规、企业计费、SLA |

## 十三、风险与取舍

需要特别说明的是，Suna 的最大魅力恰恰也是商业化改造中最大的取舍点：**共享上下文与长期驻留**。这是它区别于一般 agent 框架的核心优势，因此不建议为了商用而完全抹掉这部分特性。真正应该做的，是把这种共享能力从“默认全局共享”改造成“可配置、可审计、可边界化的共享”。

也就是说，商业化改造不应把 Suna 变成另一个纯任务型无状态 agent 平台，否则会失去它最独特的竞争力；但也不能继续沿用“一个账户一台机器，长期共享全部状态”的默认模式，否则很难支撑高密度多租运营。正确做法是：**保留长期实例作为高价值能力，同时把平台默认执行模式改造成任务化、会话化、项目化。**

## 十四、最终建议

如果你计划基于 Suna 做商业化产品，我的总建议可以概括为一句话：

> **保留 Suna 的 Agent Computer 内核，但必须补上一整层 SaaS 级 Tenant Control Plane，把“共享机器”改造成“受控工作环境”，把“长期实例”改造成“高阶服务”，把“平台能力”从 runtime 中彻底显式化。**

这样改造之后，Suna 会非常有竞争力。因为市面上很多开源 Agent 项目要么只有控制面没有真实执行深度，要么只有沙箱没有完整产品形态；而 Suna 的特别之处，正是在于它已经同时拥有两者的雏形。[1] [2] [3] [5] 真正欠缺的，不是 Agent 能力本身，而是把这套能力 **标准化、治理化、计量化、商业化**。

## References

[1]: https://github.com/kortix-ai/suna "kortix-ai/suna - GitHub README"
[2]: https://github.com/kortix-ai/suna/blob/main/MANIFESTO.md "MANIFESTO.md - kortix-ai/suna"
[3]: https://github.com/kortix-ai/suna/blob/main/core/docker/docker-compose.yml "core/docker/docker-compose.yml - kortix-ai/suna"
[4]: https://github.com/kortix-ai/suna/tree/main/apps/api/src "apps/api/src - kortix-ai/suna"
[5]: https://github.com/kortix-ai/suna/blob/main/packages/db/src/schema/kortix.ts "packages/db/src/schema/kortix.ts - kortix-ai/suna"
