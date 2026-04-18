# Suna 深度调研报告

## 一、结论摘要

**Suna**（仓库与文档中也大量使用 **Kortix** 品牌）并不是一个传统意义上的“聊天式 Agent UI + 若干工具调用”的轻量应用，而是一个更激进的 **AI Computer / Agent Operating System**。它的核心目标不是为每次任务临时起一个无状态执行环境，而是为一个账户或组织提供一台 **长期在线、持续积累上下文、可被多个 agent 共同使用的共享机器**。[1] [2]

如果用我们前面讨论的框架来定位，Suna 更接近 **“平台控制层 + 长生命周期沙箱/机器层”** 的混合形态。平台层负责认证、账户、沙箱生命周期、预览代理、计费、管理后台和集成管理；真正执行 agent 的，是一个运行 **OpenCode** 的 Ubuntu 沙箱容器或远程机器。[1] [3] [4]

从产品理念看，Suna 与 Manus 有明显相似点：都强调 agent 不是纯聊天，而是能够使用终端、文件系统、浏览器与长期状态来完成真实工作。但从默认抽象看，Suna 更像 **“公司级共享 AI 计算机”**，而 Manus 更像 **“中心编排的通用任务代理平台”**。前者强调 **共享上下文与长期驻留**，后者通常更强调 **任务编排、规范化执行与平台治理**。

## 二、项目定位与产品哲学

Suna 的 README 将自己定义为：

> **The Open-Source Operating System for Running Autonomous Companies**。[1]

这个表述非常关键。它说明 Suna 不是把 agent 当作“一个会话中的功能”，而是把 agent 当作“运行在公司操作系统之上的持续劳动力”。README 进一步强调：一台共享机器上，所有 agent 看到的是同一个文件系统、同一个数据库、同一组凭证与同一段历史，上下文不会按工具或会话切碎，而是在系统层持续累积。[1]

这一定位在 MANIFESTO 里被展开得更彻底。文档明确提出：一个 Kortix 实例是一台完整 Linux 计算机，其上运行 AI agent framework；agent 具有 root 权限，可以安装包、写脚本、调 API、部署服务，并且通过文件系统和记忆系统长期积累能力。[2] 这说明 Suna 的第一性原理不是“安全最小化的函数调用”，而是“给 agent 一台真实可生长的机器”。

| 维度 | Suna 的官方定位 | 对架构的含义 |
|---|---|---|
| 核心比喻 | AI Computer / Operating System | 不是单次任务执行器，而是长期驻留环境 |
| 默认上下文模型 | 共享文件系统、共享历史、共享凭证 | 更偏组织内共享上下文 |
| Agent 角色 | 工作者、自动化执行者、编排者 | 强调持续运行与自治 |
| 运行基础 | 完整 Linux 机器 + OpenCode | 强依赖真实终端/文件/浏览器能力 |

## 三、开源范围与项目形态

从仓库结构和 README 看，Suna 已经具备相当完整的平台轮廓，而不是一个单文件 demo。其 monorepo 至少包含顶层应用、API、core runtime、数据库 schema、测试与部署脚本等部分。[1] [5]

顶层 `package.json` 中可见多个开发入口：`dev:web`、`dev:api`、`dev:mobile`、`dev:core`、`dev:sandbox` 等，这说明前端、API 与 core sandbox runtime 是彼此分离的可独立开发模块。[5] README 还给出统一的 `kortix` CLI 用于启动、停止、日志、更新与重置整套系统，说明它不是单纯的代码库，而是带有安装器与运维抽象的产品化仓库。[1]

从开源完整度来看，Suna 的代码已经暴露出以下关键能力：平台 API、账户初始化、sandbox provider 接入、preview proxy、WebSocket 终端代理、计费路由、管理后台、集成系统、数据库 schema 与测试代码。[4] [6] 这意味着它并非只开源“前端壳子”或“沙箱底座”，而是已经开源了大量控制面逻辑。

## 四、系统架构分层

综合 README、文档和 API 代码，Suna 的结构大体可以分成四层。

第一层是 **交互层**，包括前端和移动端。文档显示启动后至少暴露前端和 API 服务；顶层脚本也包含移动端开发入口。[3] [5]

第二层是 **平台控制层**，也就是 `apps/api`。这一层负责 Supabase 鉴权、账户解析、`/v1/platform/init` 初始化、sandbox provider 选择、API key 管理、preview proxy、管理后台、集成映射、计费订阅与 webhook 等。[4] [6] [7]

第三层是 **沙箱/机器供应层**。配置和代码中可以看到多个 provider：`local_docker`、`daytona`、`justavps`。[6] 这意味着 Suna 并不把执行环境写死在单一本地容器里，而是支持本地 Docker、Daytona 远程沙箱与 JustAVPS 远程机器三种来源。

第四层是 **沙箱内部运行时层**，即 core docker 镜像。README 直言 “The agent runtime is OpenCode”，而 core compose 也明确把 OpenCode 相关目录、数据库、缓存、auth 与持久状态挂载出来。[1] [4]

| 层次 | 主要组件 | 责任 |
|---|---|---|
| 交互层 | Web / Mobile | 用户界面、发起任务、查看状态 |
| 控制层 | `apps/api` | 认证、账户、沙箱生命周期、代理、计费、集成 |
| 供应层 | Local Docker / Daytona / JustAVPS | 提供具体运行实例 |
| 运行时层 | Core sandbox + OpenCode | 真正执行 agent、工具与长期状态 |

## 五、Agent 运行机制

Suna 的 README 与 MANIFESTO 都清楚说明，它的 agent runtime 是 **OpenCode**。[1] [2] 在 MANIFESTO 中，作者进一步声明一个实例里存在一个主 orchestrator 和若干专门 agent，例如 research、browser automation、web development、presentations、spreadsheets、image generation 等。[2] 这说明 Suna 并不是只包了一层 OpenCode UI，而是已经把 OpenCode 作为底层 agent runtime，再往上叠加自己的 agent 组织方式、技能体系和平台外壳。

它的默认执行哲学也非常鲜明：**Everything is files**。文档称 agent、skill、tool、memory、command 都可以是文本文件或脚本，文件系统是基础协调层。[2] 这意味着 Suna 非常适合“长期积累、自我扩展、在同一环境中衍生新能力”的模式，但也意味着它默认就比“严格短生命周期任务 sandbox”更强调状态持久化。

从这个角度看，Suna 更适合 **长期运营一个实例**，而不是把每次交互视为“新任务、新容器、新上下文”。这与我们前面讨论的商业化多租 SaaS 模式构成了鲜明对照。

## 六、沙箱模型与状态管理

Suna 的沙箱设计，是本次调研中最值得注意的部分之一。`core/docker/docker-compose.yml` 明确把状态划分为三个逻辑区：`/workspace`、`/persistent`、`/ephemeral`。[4]

其中，`/workspace` 保存用户代码、仓库、项目文件以及用户安装包；`/persistent` 保存 OpenCode 数据库、浏览器 profile、秘密信息与系统内部持久状态；`/ephemeral` 保存运行时镜像层，也就是随镜像升级被替换的部分。[4] 这种分层设计非常成熟，说明作者已经认真考虑过“长期运行 + 可升级 + 状态不丢失”的问题。

> `/workspace` 用于用户工作区，`/persistent` 用于 OpenCode DB、secrets、browser profile 等内部状态，`/ephemeral` 则是每次更新都会替换的运行时层。[4]

同时，compose 文件还暴露了多个端口：Kortix Master、桌面、HTTPS 桌面、Presentation Viewer、Browser Stream、Browser Viewer、SSH 和静态站点端口。[4] 这说明 Suna 的沙箱并非只是一个 shell executor，而是一个含桌面、浏览器与代理组件的 **完整工作环境**。

| 状态区 | 用途 | 生命周期 |
|---|---|---|
| `/workspace` | 用户代码、仓库、项目文件、用户安装包 | 持久化保留 |
| `/persistent` | OpenCode DB、secrets、auth、浏览器 profile、系统状态 | 持久化保留 |
| `/ephemeral` | Kortix server、OpenCode config、运行时层 | 镜像更新时替换 |

## 七、平台控制面与沙箱生命周期

Suna 的平台 API 并不是可有可无的配套，而是系统的关键中枢。`apps/api/src/platform/index.ts` 将路由拆分为 provider 列表、`/init`、sandbox update、SSH、webhook、备份、local bridge、API keys 等模块。[7] 这说明平台层掌控的是 **账户与实例生命周期**，而不是只做聊天转发。

`/v1/platform/init` 的实现尤其关键。它会先通过鉴权解析当前 `userId`，再映射到 `accountId`，然后调用 `ensureSandbox` 去确保该账户拥有一个可用 sandbox。[8] 这说明 Suna 的默认资源归属单位不是“线程”或“单次任务”，而是 **账户**。若启用了内部计费，并请求 `justavps` 这类托管 provider，代码还会先检查套餐是否满足 Pro 级别。[8]

本地 Docker 模式也不是简单的 `docker run`。路由实现中包含了 image pull 进度轮询、stale provisioning row 自动修复、active sandbox 返回、失败状态标记等逻辑。[8] 这反映出 Suna 已经具备 **平台级资源控制面**，而不是“开发者自己看日志排查”的原型阶段。

## 八、多租户、账户边界与权限模型

如果只看 README，容易误以为 Suna 是“整个实例完全共享、没有租户边界”的系统，因为它反复强调 “one shared machine” 和 “shared filesystem / shared credentials / shared history”。[1] 但深入代码后可以看到，情况更准确地说是：**Suna 在产品叙事上强调组织内共享上下文，但在后端实现上仍然有明确的 account 边界**。

数据库 schema 中存在 `accounts`、`account_members`、`sandboxes` 三组核心对象。[9] 这意味着它支持账户、多成员和账户下多 sandbox。`sandboxes` 表中显式记录 `accountId`，而 `account_members` 表则建立用户与账户的多对一映射。[9]

测试代码也进一步证明这一点。在 API key 与 sandbox 相关测试中，存在明确的 **cross-user isolation** 用例，验证其他用户无法列出、创建或操作不属于自己的 sandbox keys。[6] 这说明 Suna 并非没有隔离，而是采用 **账户级隔离**，同时允许同一账户内共享较多上下文。

这与典型的 SaaS 多租模型有一个本质差别：Suna 更像 **每个账户拥有一个或多个长期机器**，而不是 **平台为每次任务临时分配一个绝对隔离沙箱**。对于单组织协作来说，这种模型非常有吸引力；但如果要做高密度 B2B2C 或大规模公有云多租 SaaS，就需要进一步补强租户治理与资源弹性策略。

| 观察点 | Suna 的表现 | 解释 |
|---|---|---|
| 是否存在账户模型 | 是 | `accounts`、`account_members`、`sandboxes` 明确存在[9] |
| 是否存在跨用户隔离 | 是 | 测试覆盖 cross-user isolation[6] |
| 是否强调共享上下文 | 是 | 官方叙事极强[1] [2] |
| 默认隔离单位 | 更像 account / instance | 而不是单次任务 |

## 九、沙箱供应、池化与复用能力

Suna 在这部分比很多开源 Agent 项目成熟。配置与代码显示它支持多个 sandbox provider：`local_docker`、`daytona` 与 `justavps`。[6] 这意味着同一平台 API 可以对接不同的执行后端。

更重要的是，它的数据库 schema 和测试中出现了 `pool_resources` 与 `pool_sandboxes`，以及相应的 pool 自动补货、claim、过期、inventory 清理与 env 注入逻辑。[9] [10] 这说明 Suna 已经开始系统化地处理 **预热池、预分配实例、快速认领与元数据清洗** 这些问题，而不是每次临时创建远程机器。

这点与你前面问到的“如何在多租场景下降低沙箱成本”是高度相关的。Suna 至少已经具备以下思路：预创建 pool sandbox、按 provider 和 server type 建资源池、在被认领后剥离 pool placeholder token、并对 stale / error 状态做清理。[9] [10] 这说明它并不是只解决“能运行”，而是在向 **商用基础设施效率** 迈进。

## 十、认证、代理与沙箱 API 交互模型

Suna 的平台与沙箱之间存在一套显式的双向认证机制。配置文件说明 `INTERNAL_SERVICE_KEY` 用于 **API → sandbox** 的服务端调用认证，而 `KORTIX_TOKEN` 则用于 **sandbox → API** 的反向认证。[6] 代码中还有把这些密钥注入 sandbox `/env` 接口的逻辑，并在必要时自动同步和重试。[6]

这很重要，因为它说明 Suna 不是“agent 在平台内直接操作所有资源”，而是存在一个清晰的 **控制面/执行面边界**。平台层通过 `/v1/p` preview proxy、WebSocket PTY 代理、subdomain preview 等方式转发流量到 sandbox。[6] 这与我们之前分析的 **“Agent 调用沙箱 API”** 路线高度一致。

因此，从架构判断上，Suna 明显不是“把所有控制逻辑都塞进沙箱里”的形态，而更像：**平台控制层负责账户、路由、鉴权与资源生命周期；真实 agent runtime 在 sandbox 内运行；二者通过代理和服务密钥互联。**

## 十一、计费与商业化线索

Suna 的平台代码中不仅有认证和 sandbox，还存在相当完整的计费逻辑，包括 Stripe webhook、订阅、credits、计划判断和 sandbox 供应触发。[6] 文档侧也列出了 Billing & Subscriptions 页面入口，自托管文档中同样明确包含 Supabase 和 API key 配置。[3]

在 `billing` 相关代码里可以看到：免费用户默认不配置托管 sandbox，而是 “connect their own”；付费用户则可以在 checkout 或 setup 流程中触发 sandbox provisioning。[6] 这说明 Suna 的商业模型不是“每个人都默认给一台托管机器”，而是存在 **BYOC/本地模式与托管付费模式并存** 的路线。

这在商业上是合理的，因为长期在线的 agent 机器成本远高于一次性任务沙箱。Suna 的实现本质上是在把“机器拥有权”和“平台控制权”做分层：轻用户连接自己的环境，重度用户为托管机器买单。

## 十二、与 Manus 的相似与差异

如果把 Suna 与 Manus 放在同一坐标系里，它们的共同点很多。两者都强调：agent 不是聊天玩具，而是要能操纵真实工具、访问真实环境、处理真实文件并持续工作。两者也都更接近 **通用执行型 Agent 平台**，而不是只做 prompt 编排的 workflow 工具。

但二者的默认系统哲学并不完全相同。Suna 更像“每个账户拥有一台长期在线的 AI 计算机”；Manus 则更像“平台侧有统一编排器，按任务调用受控执行环境”。前者的优势是状态沉淀深、上下文复利强；后者的优势是治理标准化、弹性更强、商业化多租更容易做细粒度控制。

| 对比维度 | Suna | Manus 类平台 |
|---|---|---|
| 默认抽象 | AI Computer / 共享机器 | 通用任务代理平台 |
| 运行时 | OpenCode 驱动的长期实例 | 更像中心编排 + 受控执行器 |
| 状态模型 | 强持久化、共享上下文 | 更强调任务级状态与平台管理 |
| 隔离单位 | Account / Instance 更强 | Task / Session / Workspace 更常见 |
| 优势 | 上下文复利、长期记忆、组织共享 | 治理、弹性、标准化、多租更容易 |
| 风险 | 状态膨胀、共享边界复杂 | 冷启动、跨任务上下文保留更难 |

## 十三、适用场景与局限

如果你的目标是做一个 **组织内部长期运行的 AI 操作系统**，让多个 agent 共用上下文、工具、文件和长期记忆，那么 Suna 是当前开源方案里相当有特色、而且已经具备平台雏形的项目。[1] [2] [4] 它尤其适合以下场景：技术型创业团队、单组织内的自动化协作、长期运营一套 AI workforce、BYOC 或半托管模式。

但如果你的目标是做一个像典型公有云 SaaS 那样的 **高密度多租、严格任务隔离、成本与配额细粒度控制** 的 Agent 平台，那么 Suna 的默认模型并不是天然最优。它更像是“先给每个账户一台脑机合一的长期机器”，而不是“先构造一套高度抽象和细粒度计量的任务执行网格”。这并不代表它不能演进到那一步，只是它的出发点不同。

## 十四、最终判断

综合官方叙事、文档与代码，我的最终判断如下。

Suna 不是一个“只会调用工具的 Agent UI”，而是一个已经具备 **控制层、认证层、计费层、sandbox 供应层和 OpenCode runtime 层** 的开源 Agent 平台。[1] [3] [4] [5] [6] [7]

它也不是简单的“把 OpenCode 跑起来”。真正关键的是，它在 OpenCode 之外，已经搭建了账户、沙箱 provider、预览代理、token 注入、sandbox 池化、计费和集成映射等平台基础设施。[6] [8] [9] [10]

所以，如果要一句话概括：

> **Suna 更像一个以 OpenCode 为核心 runtime、以长期共享机器为产品抽象、并且已经初步平台化的开源 Manus 近邻。**

但它与 Manus 的关系更准确的说法是“邻近但不等同”：两者都属于自主执行型 Agent 平台，但 **Suna 更像共享 AI 计算机，Manus 更像统一编排的代理平台**。

## References

[1]: https://github.com/kortix-ai/suna "kortix-ai/suna - GitHub README"
[2]: https://github.com/kortix-ai/suna/blob/main/MANIFESTO.md "MANIFESTO.md - kortix-ai/suna"
[3]: https://mintlify.wiki/kortix-ai/suna/self-hosting/setup "Setup Guide - Kortix"
[4]: https://github.com/kortix-ai/suna/blob/main/core/docker/docker-compose.yml "core/docker/docker-compose.yml - kortix-ai/suna"
[5]: https://github.com/kortix-ai/suna/blob/main/package.json "package.json - kortix-ai/suna"
[6]: https://github.com/kortix-ai/suna/tree/main/apps/api/src "apps/api/src - kortix-ai/suna"
[7]: https://github.com/kortix-ai/suna/blob/main/apps/api/src/platform/index.ts "platform/index.ts - kortix-ai/suna"
[8]: https://github.com/kortix-ai/suna/blob/main/apps/api/src/platform/routes/account.ts "platform/routes/account.ts - kortix-ai/suna"
[9]: https://github.com/kortix-ai/suna/blob/main/packages/db/src/schema/kortix.ts "packages/db/src/schema/kortix.ts - kortix-ai/suna"
[10]: https://github.com/kortix-ai/suna/blob/main/apps/api/src/__tests__/pool.test.ts "pool.test.ts - kortix-ai/suna"
