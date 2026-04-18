# Suna 各模块源码解读清单

**作者：Manus AI**

## 一、文档目的

这份清单不是简单罗列目录，而是从**读源码的角度**给出一条更高效的理解路径。目标是帮助你在阅读 Suna 时，不是陷入零散文件，而是先抓住它的系统主轴：**账户模型、沙箱生命周期、平台代理接入、Agent 运行时、前端控制台**。因此，下面的清单会同时回答五个问题：每个模块是干什么的、应该先看哪些文件、这些文件解决什么问题、阅读时该关注哪些实现细节、以及它在整个系统里的优先级。

## 二、建议的总体阅读顺序

如果你是第一次深入 Suna 源码，我建议不要按目录从上往下盲读，而是按“从系统骨架到局部实现”的顺序推进。**最佳顺序是：产品入口与仓库结构 → API 总入口 → 账户与认证 → 沙箱生命周期 → 预览代理 → 数据模型 → 沙箱运行时 → Router/集成/计费 → 前端控制台。**

| 阅读阶段 | 目标 | 推荐优先级 |
|---|---|---|
| 1 | 先理解产品定位与 Monorepo 结构 | 极高 |
| 2 | 先看 API 总入口和平台路由装配 | 极高 |
| 3 | 再看账户解析与认证中间件 | 极高 |
| 4 | 再看沙箱创建、复用、池化、更新逻辑 | 极高 |
| 5 | 再看预览代理与平台到沙箱的连接方式 | 极高 |
| 6 | 然后看数据库 schema，把核心对象串起来 | 高 |
| 7 | 再看沙箱运行时 `core` | 高 |
| 8 | 最后看 Router、Integrations、Billing、Deployments 等扩展域 | 中高 |
| 9 | 最后回到前端，理解产品交互如何驱动后端 | 中高 |

## 三、一级模块源码解读清单

### 3.1 产品与仓库骨架模块

这一层的目的不是看业务逻辑，而是先建立“地图感”。你如果不知道仓库是怎么分层的，后面会一直分不清哪些代码是控制面，哪些代码是执行面，哪些代码只是前端展示。

| 模块 | 关键文件/目录 | 作用 | 阅读关注点 | 优先级 |
|---|---|---|---|---|
| 产品定位 | `README.md` | 解释 Suna/Kortix 想解决什么问题 | 它到底是聊天应用、Agent 平台还是长期运行计算机 | 极高 |
| Monorepo 入口 | `package.json` | 定义开发脚本与工作区 | `apps/api`、`apps/web`、`core` 谁是主体 | 极高 |
| 顶层目录结构 | `apps/` `packages/` `core/` | 确认前后端与运行时边界 | 控制面与沙箱是否解耦 | 极高 |

**阅读建议**：这一层不要停留太久，但一定先读。它决定你后面对代码的解释框架。

### 3.2 API 总入口与服务装配模块

这部分是整个后端的“总线层”。如果你只看单个业务文件，很容易误以为 Suna 是某个单点服务；但一旦读了总入口，你会发现它是多个服务域拼起来的控制面。

| 模块 | 关键文件 | 作用 | 阅读关注点 | 优先级 |
|---|---|---|---|---|
| API 总入口 | `apps/api/src/index.ts` | 注册全局中间件、健康检查、子路由、后台任务 | 哪些能力是一级子域；启动时有哪些持续服务 | 极高 |
| 平台域装配 | `apps/api/src/platform/index.ts` | 聚合平台相关路由 | `account`、`sandbox`、`backup`、`ssh`、`update` 如何分层 | 极高 |
| Router 装配 | `apps/api/src/router/index.ts` | 聚合 LLM、搜索、代理路由 | Router 是系统核心还是能力子域 | 高 |

**阅读建议**：读 `index.ts` 时，重点标出每一个 `app.route()` 的一级前缀，然后自己画一张“后端路由脑图”。这一步能极大提升后续源码阅读效率。

### 3.3 认证与身份模型模块

Suna 的一个核心特征是，它不是只有“用户登录态”这一种身份。代码里至少有**平台用户身份、沙箱服务身份、预览访问身份**三层。理解这一点后，很多代理和 token 注入逻辑才会顺畅。

| 模块 | 关键文件 | 作用 | 阅读关注点 | 优先级 |
|---|---|---|---|---|
| 统一认证中间件 | `apps/api/src/middleware/auth.ts` | 实现 `apiKeyAuth`、`supabaseAuth`、`combinedAuth` | JWT 与 Kortix Token 如何共存；preview cookie 如何工作 | 极高 |
| 账户解析 | `apps/api/src/shared/resolve-account.ts` | 将用户映射到账户，并兼容旧模型迁移 | `account_members` 与旧 `account_user` 的桥接方式 | 极高 |
| 平台角色 | `apps/api/src/shared/platform-roles.ts` | 管理平台角色与管理员逻辑 | admin/super_admin 权限边界 | 中高 |
| 预览权限 | `apps/api/src/shared/preview-ownership.ts` | 判断谁能访问某个 sandbox 预览 | 用户态与沙箱态访问边界 | 高 |
| 沙箱访问控制 | `apps/api/src/shared/sandbox-access.ts` | 按账户范围查找可访问沙箱 | account 是否为真正租户边界 | 高 |

**阅读建议**：这组文件要和数据库 schema 一起看。因为身份逻辑最终都会落到 `accountId`、`sandboxId`、`api key` 这些对象上。

### 3.4 账户初始化与平台主业务入口模块

这是 Suna 最关键的一层。很多人看 Suna，会下意识先找聊天路由或 Agent loop；但从代码上看，它真正的主业务入口其实是“为账户确保一台可用机器”。

| 模块 | 关键文件 | 作用 | 阅读关注点 | 优先级 |
|---|---|---|---|---|
| 账户初始化路由 | `apps/api/src/platform/routes/account.ts` | 提供 `/providers`、`/init`、`/init/local` | 用户进系统后如何拿到账户和沙箱 | 极高 |
| 确保沙箱服务 | `apps/api/src/platform/services/ensure-sandbox.ts` | 核心的 create-or-return 逻辑 | 查现有、重启旧实例、池化 claim、新建实例 | 极高 |
| Provider 抽象入口 | `apps/api/src/platform/providers/*` | 对接 `daytona`、`local_docker`、`justavps` | provider 能力接口是否统一 | 极高 |

**阅读建议**：如果你只选三组文件读出 Suna 的核心，我会建议就是：`account.ts`、`ensure-sandbox.ts`、`sandbox-cloud.ts`。

### 3.5 沙箱生命周期与云实例管理模块

这一层比“账户初始化”更偏基础设施控制面。前者解决“有无机器”，这一层解决“机器怎么收费、怎么停、怎么重启、怎么归档、怎么恢复”。

| 模块 | 关键文件 | 作用 | 阅读关注点 | 优先级 |
|---|---|---|---|---|
| 云端沙箱生命周期 | `apps/api/src/platform/routes/sandbox-cloud.ts` | 创建、获取、列出、停止、重启、归档沙箱 | `sandboxes` 表是如何驱动业务主线的 | 极高 |
| 沙箱更新 | `apps/api/src/platform/routes/sandbox-update.ts` `apps/api/src/update/*` | 负责版本更新和容器更新 | 升级是否通过镜像替换 + 状态迁移 | 高 |
| 沙箱备份 | `apps/api/src/platform/routes/sandbox-backups.ts` | 管理备份与恢复 | 备份粒度是工作区还是整实例 | 中高 |
| SSH 管理 | `apps/api/src/platform/routes/ssh.ts` | 暴露 SSH 访问管理 | 远程运维如何接入 | 中高 |
| 健康与轮询 | `apps/api/src/platform/services/sandbox-health.ts` `sandbox-provision-poller.ts` | 监控 provisioning / 健康状态 | 状态机是同步推进还是轮询修复 | 高 |

**阅读建议**：看这一组时，建议你专门记一张“沙箱状态流转表”，至少标出 `provisioning`、`active`、`stopped`、`archived`、`pooled`、`error` 六个状态的进入条件。

### 3.6 池化与资源复用模块

如果你关心商业化、多租、成本优化，那么这一层很重要。Suna 并非每次都临时创建新实例，而是引入了池化资源与待领用沙箱。

| 模块 | 关键文件 | 作用 | 阅读关注点 | 优先级 |
|---|---|---|---|---|
| 池化总入口 | `apps/api/src/pool/index.ts` | 暴露池化能力 | pool 子系统如何与平台主流程耦合 | 高 |
| 自动补充 | `apps/api/src/pool/auto-replenish.ts` | 持续补池 | 期望库存如何维护 | 高 |
| 资源清单 | `apps/api/src/pool/inventory.ts` | 管理可用池资源 | provider、serverType、location 三元组如何管理 | 高 |
| 环境注入 | `apps/api/src/pool/env-injector.ts` | 给领用实例动态注入 token/env | claim 后如何从“公用池实例”变成“账户实例” | 极高 |
| 资源与统计 | `apps/api/src/pool/resources.ts` `stats.ts` | 资源配置与监控 | 池命中率和健康度如何评估 | 中高 |

**阅读建议**：如果你研究的是“如何把 Suna 商业化”，这组模块的优先级要上调，因为它直接关系成本结构和启动时延。

### 3.7 预览代理与沙箱接入模块

这是理解“平台如何把沙箱能力暴露给用户”的关键模块。很多系统只做实例管理，但 Suna 还要把沙箱内服务通过统一入口代理出去。

| 模块 | 关键文件 | 作用 | 阅读关注点 | 优先级 |
|---|---|---|---|---|
| 代理入口 | `apps/api/src/sandbox-proxy/index.ts` | 统一处理 `/v1/p/:sandboxId/:port/*` | provider 分流、token 刷新、serviceKey 注入 | 极高 |
| 本地预览代理 | `apps/api/src/sandbox-proxy/routes/local-preview.ts` | 本地 Docker 沙箱代理 | 本地模式与云模式差异 | 高 |
| 通用预览 | `apps/api/src/sandbox-proxy/routes/preview.ts` | 预览访问逻辑 | Daytona 等 provider 的预览路径 | 高 |
| 认证入口 | `apps/api/src/sandbox-proxy/routes/auth.ts` | 设置 preview cookie | preview session 的浏览器侧体验 | 高 |
| 分享链接 | `apps/api/src/sandbox-proxy/routes/share.ts` | 生成可分享访问地址 | 临时分享与公开分享边界 | 中高 |
| 项目代理 | `apps/api/src/routes/kortix-projects.ts` | 代理 `/v1/kortix/*` 到沙箱内部 | API 控制面如何桥接沙箱内部服务 | 高 |

**阅读建议**：这一组最好和 `auth.ts` 对照阅读，否则会看不明白为什么同一个路由既支持 JWT，又支持 kortix token，又支持 cookie。

### 3.8 数据模型与持久化模块

如果前面的业务代码告诉你“系统在做什么”，那数据库 schema 告诉你的就是“系统真正把什么对象当作一等公民”。

| 模块 | 关键文件 | 作用 | 阅读关注点 | 优先级 |
|---|---|---|---|---|
| 核心 schema | `packages/db/src/schema/kortix.ts` | 定义账户、成员、沙箱、池、部署、API Key 等核心对象 | 哪些对象是平台中心 | 极高 |
| 兼容 schema | `packages/db/src/schema/public.ts` | 公共 schema 或 legacy 兼容 | 与旧系统的关系 | 中高 |
| DB 入口 | `packages/db/src/client.ts` `src/index.ts` | DB 连接与类型导出 | Drizzle 的组织方式 | 中 |
| 迁移 | `packages/db/drizzle/*` `supabase/migrations/*` | 结构演进历史 | 多租和计费逻辑是后补还是一开始就有 | 高 |

**阅读建议**：读 schema 时不要只看表名，要把**主业务对象的关联关系**自己重画一遍，尤其是 `accounts -> account_members -> sandboxes -> api_keys -> deployments` 这条链。

### 3.9 计费与订阅模块

这一层在 Suna 中不是外围系统，而是与沙箱创建深度耦合的。尤其是托管 VPS 场景，订阅会直接参与机器创建和回滚。

| 模块 | 关键文件 | 作用 | 阅读关注点 | 优先级 |
|---|---|---|---|---|
| 计费入口 | `apps/api/src/billing/index.ts` | 计费路由装配 | billing 子域整体边界 | 高 |
| 客户与订阅仓储 | `apps/api/src/billing/repositories/*` | 持久化客户、信用、订阅关系 | 账户与账单的绑定方式 | 高 |
| 订阅服务 | `apps/api/src/billing/services/subscriptions.ts` | Stripe 订阅逻辑 | 机器创建与订阅生成的耦合方式 | 极高 |
| 定价与层级 | `apps/api/src/billing/services/tiers.ts` | 计算规格、计划与价格映射 | server type 与 price 的映射方式 | 高 |
| Webhook | `apps/api/src/billing/webhooks/*` | 支付事件回调 | 异步对账与状态修复 | 高 |

**阅读建议**：如果你只想理解产品逻辑，可先粗读；如果你要做商用化改造，这一层必须精读。

### 3.10 Router、模型与搜索能力模块

这一层是 Suna 的能力网关。它很重要，但从系统骨架角度看，它不是最先要读的部分。

| 模块 | 关键文件 | 作用 | 阅读关注点 | 优先级 |
|---|---|---|---|---|
| Router 入口 | `apps/api/src/router/index.ts` | 聚合 LLM、Web Search、Image Search | 能力层与控制面如何分离 | 高 |
| LLM 路由 | `apps/api/src/router/routes/llm.ts` | 模型调用统一入口 | 请求如何转给外部模型供应商 | 高 |
| Anthropic 兼容 | `apps/api/src/router/routes/anthropic.ts` | 兼容式接口 | 多模型接口标准化方式 | 中高 |
| 搜索 | `apps/api/src/router/routes/search-web.ts` `search-image.ts` | 搜索能力 | Search 是平台内建能力还是第三方封装 | 中高 |
| 计费检查 | `apps/api/src/router/services/billing.ts` | 在 Router 层做额度判断 | 模型调用与信用扣减耦合点 | 中高 |

**阅读建议**：这一层适合在你已经理解“账户-沙箱-平台”主线后再看，否则容易误判它是系统主中心。

### 3.11 集成、密钥与外部能力接入模块

Suna 希望让 Agent 在“公司操作系统”里长期工作，因此集成与密钥管理是非常重要的支撑层。

| 模块 | 关键文件 | 作用 | 阅读关注点 | 优先级 |
|---|---|---|---|---|
| Providers 注册表 | `apps/api/src/providers/registry.ts` | 定义外部 API/模型/工具 provider | provider 是业务提供商，不是基础设施 provider | 中高 |
| Provider 路由 | `apps/api/src/providers/routes.ts` | 连接/断开 provider | 外部工具接入流程 | 中高 |
| Secrets 路由 | `apps/api/src/secrets/routes.ts` | 管理平台密钥 | secret 是存在 DB 还是注入沙箱 | 高 |
| Integrations 总入口 | `apps/api/src/integrations/index.ts` | 集成能力聚合 | 与前端集成设置的对应关系 | 中高 |
| Credential Store | `apps/api/src/integrations/credential-store.ts` | 存储集成凭证 | 租户级安全边界 | 高 |

**阅读建议**：如果你关注 Agent 如何真正“干活”，这组模块和 `core` 运行时要一起看。

### 3.12 队列、隧道与辅助运行模块

这组模块更多是“平台运行辅助件”，不是主业务骨架，但能帮助你理解 Suna 并不是一个请求-响应式静态网站，而是一个持续运行的平台。

| 模块 | 关键文件 | 作用 | 阅读关注点 | 优先级 |
|---|---|---|---|---|
| Queue 路由 | `apps/api/src/queue/routes.ts` | 会话消息队列与排序 | 是轻量消息暂存还是任务编排系统 | 中高 |
| Queue Drainer | `apps/api/src/queue/drainer.ts` | 队列消费 | 前端队列与后台执行关系 | 中 |
| Tunnel 总入口 | `apps/api/src/tunnel/index.ts` | 隧道能力装配 | 沙箱与外部设备如何连接 | 中高 |
| Device Auth | `apps/api/src/tunnel/routes/device-auth.ts` | CLI/设备认证 | 用户侧接入流程 | 中 |
| Servers 路由 | `apps/api/src/servers/index.ts` | 服务器接入/同步 | 托管与自带机器的模型差异 | 中高 |

### 3.13 管理后台与运维模块

如果你后续想做自托管、SaaS 运维或企业版改造，这一层要认真看。

| 模块 | 关键文件 | 作用 | 阅读关注点 | 优先级 |
|---|---|---|---|---|
| Admin 入口 | `apps/api/src/admin/index.ts` | 管理接口集合 | 平台侧能看到哪些运维对象 | 中高 |
| Sandbox Pool Admin | `apps/api/src/platform/routes/sandbox-pool-admin.ts` | 池化管理接口 | 预热池的人工干预方式 | 高 |
| Setup 路由 | `apps/api/src/setup/index.ts` | 自托管初始化 | 本地模式与云模式差异 | 高 |
| Ensure Schema | `apps/api/src/ensure-schema.ts` | 启动时 schema 自检 | 部署与启动依赖 | 中 |

### 3.14 沙箱运行时与 OpenCode 执行环境模块

这是 Suna 的执行面，也是最像“Manus/OpenCode runtime substrate”的部分。它解释了真正的 Agent 计算机是怎么落地的。

| 模块 | 关键文件 | 作用 | 阅读关注点 | 优先级 |
|---|---|---|---|---|
| 运行时拓扑 | `core/docker/docker-compose.yml` | 定义容器、卷、端口、环境变量 | `/workspace`、`/persistent`、`/ephemeral` 三层状态分离 | 极高 |
| 镜像构建 | `core/docker/Dockerfile` | 构建沙箱镜像 | 系统工具、浏览器、运行时如何打包 | 极高 |
| 启动脚本 | `core/init-scripts/*` `core/scripts/*` | 引导运行时初始化 | 密钥、包恢复、运行时服务启动顺序 | 高 |
| 服务编排 | `core/s6-services/*` `core/services/*` | 容器内服务管理 | 多进程如何在一个沙箱内协调 | 高 |
| 测试 | `core/tests/*` | 运行时验证 | 哪些行为被视作核心能力 | 中高 |

**阅读建议**：理解这层时，要把它看成“长期机器操作系统”，不是普通 Docker 容器。重点关注**状态持久化设计**与**平台注入身份**这两个点。

### 3.15 前端控制台与产品交互模块

前端不是为了样式，而是帮你理解“后端这些抽象最终如何进入产品交互”。特别是 `instance`、`billing`、`integrations`、`thread`、`settings` 等模块，可以帮助反推后端哪些接口是真正产品主路径。

| 模块 | 关键目录 | 作用 | 阅读关注点 | 优先级 |
|---|---|---|---|---|
| 页面路由 | `apps/web/src/app/*` | 页面与产品信息架构 | 主要产品入口有哪些 | 高 |
| Hooks 层 | `apps/web/src/hooks/*` | 前端调用后端的封装 | 哪些后端接口被高频使用 | 高 |
| 实例模块 | `apps/web/src/components/instance/*` | 机器/实例 UI | sandbox 生命周期如何映射到前端 | 高 |
| 计费模块 | `apps/web/src/components/billing/*` | 订阅与价格 UI | 计费逻辑如何暴露给用户 | 中高 |
| 集成模块 | `apps/web/src/components/integrations/*` | Provider 接入 UI | 第三方能力如何配置 | 中高 |
| 线程/会话模块 | `apps/web/src/components/thread/*` | 对话/线程 UI | Agent 交互在产品中的位置 | 中高 |
| 平台调用 Hooks | `apps/web/src/hooks/platform/*` `hooks/instance/*` | 实例与平台接口调用 | 前端如何驱动 `/v1/platform/*` | 极高 |

## 四、按目标导向的阅读路线

如果你的目标不同，阅读顺序也应该不同。下面给出三条常见路线。

| 你的目标 | 最优阅读路线 |
|---|---|
| 想快速搞懂 Suna 整体架构 | `README` → `package.json` → `apps/api/src/index.ts` → `platform/index.ts` → `auth.ts` → `account.ts` → `ensure-sandbox.ts` → `sandbox-cloud.ts` → `kortix.ts` → `sandbox-proxy/index.ts` → `core/docker/docker-compose.yml` |
| 想研究多租户与商业化 | `resolve-account.ts` → `kortix.ts` → `sandbox-cloud.ts` → `ensure-sandbox.ts` → `pool/*` → `billing/*` → `sandbox-pool-admin.ts` |
| 想研究 Agent 执行环境与沙箱 | `core/docker/docker-compose.yml` → `core/docker/Dockerfile` → `sandbox-proxy/index.ts` → `routes/kortix-projects.ts` → `providers/*` → `auth.ts` |

## 五、建议你重点做的源码笔记

为了避免看完就忘，我建议你在阅读时同步维护四张表。

| 笔记主题 | 你应该记录什么 |
|---|---|
| 领域对象表 | account、member、sandbox、poolSandbox、apiKey、deployment 的字段与关系 |
| 状态流转表 | sandbox 从 provisioning 到 active/stopped/error/archived/pooled 的变化条件 |
| 请求链路表 | `/v1/platform/init`、`/v1/platform/sandbox`、`/v1/p/*` 的调用时序 |
| 身份模型表 | Supabase JWT、Kortix Token、Preview Cookie 各自的调用范围 |

## 六、我建议的“最小必读文件集”

如果你时间有限，只读下面这组文件，也能把 Suna 的系统骨架抓住七八成。

| 文件 | 为什么必读 |
|---|---|
| `README.md` | 明确产品不是普通聊天应用 |
| `package.json` | 建立仓库分层认知 |
| `apps/api/src/index.ts` | 看到完整后端服务地图 |
| `apps/api/src/middleware/auth.ts` | 看懂双身份模型 |
| `apps/api/src/shared/resolve-account.ts` | 看懂租户边界 |
| `apps/api/src/platform/routes/account.ts` | 看懂账户初始化主入口 |
| `apps/api/src/platform/services/ensure-sandbox.ts` | 看懂“账户 -> 沙箱”的核心逻辑 |
| `apps/api/src/platform/routes/sandbox-cloud.ts` | 看懂沙箱生命周期与计费耦合 |
| `apps/api/src/sandbox-proxy/index.ts` | 看懂平台如何接入沙箱内部服务 |
| `packages/db/src/schema/kortix.ts` | 看懂数据模型主轴 |
| `core/docker/docker-compose.yml` | 看懂执行面与状态持久化模型 |

## 七、结论

如果从源码解读价值来看，**Suna 最值得优先阅读的不是前端线程 UI，也不是某个单独的模型调用文件，而是“账户、认证、沙箱生命周期、代理接入、数据库 schema、运行时容器”这六块**。因为它们决定了 Suna 为什么更像一个 **Agent 平台/计算机操作系统**，而不是一个简单的 AI 对话应用。

换句话说，读 Suna 源码最有效的方式，不是先问“它怎么聊天”，而是先问：

> **它如何识别用户属于哪个账户、如何为该账户确保一台机器、如何把平台能力和密钥注入这台机器、又如何把机器中的服务安全地代理回产品界面。**

只要把这条主线读通，后面无论是计费、集成、部署、调度、前端控制台还是 Agent 运行细节，都会变得更容易理解。
