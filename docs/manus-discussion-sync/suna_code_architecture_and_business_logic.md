# Suna 系统架构与业务逻辑代码级梳理

**作者：Manus AI**

## 一、执行摘要

从代码结构看，**Suna（仓库内产品名更偏向 Kortix）不是一个单纯的聊天式 Agent 应用，而是一个围绕“账户级长期运行计算机”构建的自主执行平台**。它的核心不是一次对话调用，而是为每个账户分配、复用、代理和管理一个长期存在的沙箱/机器，再把 LLM、搜索、密钥、集成、部署、预览访问、更新与计费能力挂接在这个机器之上。

如果用一句话概括其架构：**Web/Mobile 前端 + Hono API 控制面 + DB 持久化 + Provider 抽象层 + 沙箱运行时容器 + 预览代理/隧道接入层**。其中，真正的业务主线不是“消息 -> 模型 -> 回复”，而是 **“用户/账户 -> 解析账户归属 -> 确保可用沙箱 -> 注入服务密钥 -> 通过代理访问沙箱内部能力 -> 持续管理其生命周期”**。

## 二、代码仓库分层结构

仓库是一个典型的 **monorepo**。顶层 `package.json` 把系统分成 `apps`、`packages`、`core` 三大类，开发命令也直接对应前端、API 与核心沙箱运行时三个部分。

| 顶层目录 | 角色 | 代码层意义 |
|---|---|---|
| `apps/api` | 后端控制面 | 认证、账户、沙箱生命周期、计费、代理、集成、管理接口 |
| `apps/web` | Web 前端 | 控制台、实例管理、集成配置、订阅、设置、会话 UI |
| `apps/mobile` | 移动端 | 移动侧入口 |
| `apps/frontend` | 兼容/旧前端别名或独立构建单元 | 与 Web 前端脚本有关 |
| `packages/db` | 数据模型层 | 数据库 schema、迁移、类型定义 |
| `packages/shared` | 共享逻辑 | 公共类型与库 |
| `packages/agent-tunnel` | Agent 隧道相关能力 | 远程连接与设备认证相关 |
| `core` | 沙箱运行时 | Docker 镜像、容器启动、OpenCode 运行环境、系统服务 |
| `supabase` | 身份与数据库依赖 | Supabase 相关迁移/配置 |
| `tests` / `apps/api/src/__tests__` | 测试层 | 生命周期、安全性、池化、预览代理、更新等测试 |

从这个结构可以看出，**Suna 的“平台主体”其实在 `apps/api`，而 `core` 更像被平台调度和接入的执行面**。

## 三、系统架构分层

### 3.1 表层：多终端产品入口

`apps/web/src` 的目录结构显示，前端并不是一个极简对话页，而是一个完整的产品控制台。它包含 `instance`、`integrations`、`scheduled-tasks`、`deployments`、`secrets`、`settings`、`thread`、`sidebar`、`onboarding`、`billing` 等模块。这说明 Suna 的产品概念是 **“带实例、集成、订阅和运维能力的 Agent 平台”**，而不是单个 Agent Demo。

### 3.2 中层：API 控制面

`apps/api/src/index.ts` 是整个后端的总入口。这里可以看到几个关键特征。

第一，它用 **Hono** 组织多个子服务，并在顶层统一处理 **CORS、请求上下文、日志、Sentry、健康检查**。这说明 API 服务本身承担的是平台控制面的职责，而不是只做一个轻量 BFF。

第二，它把多个子域明确挂载为独立路由前缀，例如 `/v1/router`、`/v1/billing`、`/v1/platform`、`/v1/pipedream`、`/v1/admin`、`/v1/providers`、`/v1/secrets`、`/v1/servers`、`/v1/queue`、`/v1/tunnel`。这意味着 **Suna 的业务功能是按域拆分的，而不是把所有逻辑糊在一个 Agent Controller 里**。

第三，它在进程启动时还会启动一系列后台任务，例如 **模型价格初始化、隧道服务、沙箱健康监控、Provision 轮询、池自动补充、访问控制缓存**。这表明平台不是请求驱动的静态 API，而是带有持续运维逻辑的控制面服务。

### 3.3 下层：沙箱执行面

`core/docker/docker-compose.yml` 定义了名为 `desktop` 的核心容器，也就是仓库里的沙箱运行时。这里最关键的设计不是容器本身，而是它的 **持久化分区模型**：

| 路径 | 作用 | 生命周期 |
|---|---|---|
| `/workspace` | 用户工作目录、代码、项目、用户安装包 | 持久化 |
| `/persistent` | OpenCode 状态、浏览器 profile、认证、密钥、内部状态 | 持久化 |
| `/ephemeral` | Kortix/OpenCode 运行时与系统逻辑 | 镜像更新时替换 |
| `/var/lib/docker` | 沙箱内部 Docker 数据 | 独立持久卷 |

这说明 Suna 的沙箱不是纯粹临时环境，而是一个 **长期运行、可升级、带工作记忆与工具状态的“账户计算机”**。同时，环境变量中可以看到 `KORTIX_API_URL`、`KORTIX_TOKEN`、`INTERNAL_SERVICE_KEY`、`SANDBOX_ID`、`PROJECT_ID` 等变量，说明沙箱在启动后会与 API 控制面保持一种 **受控双向关系**：平台给它身份和服务密钥，沙箱反向暴露内部服务给平台代理。

## 四、后端核心服务边界

从 `apps/api/src/index.ts` 与 `apps/api/src/platform/index.ts` 来看，Suna 的后端大致可以拆成以下几类服务边界。

| 服务域 | 主要职责 | 是否属于主业务主线 |
|---|---|---|
| `platform` | 账户初始化、沙箱创建/停止/重启/更新/SSH/备份 | 是 |
| `sandbox-proxy` | 预览访问、端口代理、分享链接、沙箱服务转发 | 是 |
| `router` | LLM 路由、模型访问、Web/Image Search、代理路由 | 是，但偏能力层 |
| `billing` | 订阅、收费、机器计费、Webhook | 是 |
| `providers` / `integrations` | 第三方 API/工具接入 | 支撑层 |
| `secrets` | 密钥存储与注入 | 支撑层 |
| `queue` | 会话级消息暂存与队列 | 辅助层 |
| `tunnel` | 设备认证、隧道连接 | 接入层 |
| `admin` | 平台管理、池管理、健康检查 | 运维层 |
| `deployments` | 部署能力 | 扩展业务层 |

因此，**Suna 的平台中心并不是“聊天路由器”，而是 `platform + sandbox-proxy + billing + auth` 这一组控制面能力**。

## 五、核心数据模型

### 5.1 账户是租户边界的中心

`packages/db/src/schema/kortix.ts` 显示，系统的租户边界主要围绕 **`accounts`** 和 **`account_members`** 建模，而不是围绕 thread、project 或 workspace 建模。

| 表 | 作用 | 说明 |
|---|---|---|
| `accounts` | 账户主体 | 可看作租户/组织/个人空间 |
| `account_members` | 用户与账户关系 | 支持 owner/admin/member |
| `sandboxes` | 账户拥有的机器/沙箱 | 平台最核心资源对象 |
| `pool_resources` | 预热池资源配置 | 定义 provider、规格、地区、目标容量 |
| `pool_sandboxes` | 预热池中的待领用沙箱 | 用于加速创建与复用 |
| `deployments` | 部署对象 | 与 sandbox 关联 |
| `api_keys` | 用户或沙箱级 API Key | 支撑 Agent 与平台互通 |

这说明 **Suna 的真正一等公民不是“Agent Session”，而是“Account 和 Sandbox”**。很多业务能力最终都落在 `sandboxes` 表上：provider、externalId、status、baseUrl、config、metadata、lastUsedAt、stripeSubscriptionItemId 等都集中在这个对象上。

### 5.2 沙箱对象承载了基础设施与业务状态的耦合

`sandboxes` 表不是单纯的基础设施记录。它既记录 **provider/externalId/baseUrl/status** 这种运行时信息，也记录 **metadata/config/isIncluded/stripeSubscriptionItemId** 这类业务状态。因此在 Suna 中，**沙箱对象本身就是“基础设施资源 + 商业对象 + 用户工作容器”的合体**。

这也是理解其业务逻辑的关键：只要读懂 `sandboxes` 的创建、复用、代理和更新，就读懂了 Suna 的半个系统。

## 六、认证与身份模型

`apps/api/src/middleware/auth.ts` 展示了 Suna 非常关键的一点：**它有双身份体系**。

| 身份类型 | 认证方式 | 典型调用方 | 用途 |
|---|---|---|---|
| 用户身份 | Supabase JWT | Web/App 用户 | 控制台、计费、平台配置、实例管理 |
| 系统/API 身份 | Kortix Token / Sandbox Token | 沙箱内部 Agent、服务调用方 | 路由访问、代理调用、沙箱内部服务 |
| 预览会话身份 | `__preview_session` Cookie | 浏览器 iframe / 预览访问 | 访问 `/v1/p/*` |

`combinedAuth` 的逻辑非常说明问题。它会依次尝试 Authorization Header、`X-Kortix-Token`、preview cookie、query token；然后优先识别是否是 **Kortix Token**，否则再走 **Supabase JWT**。这表明 Suna 不是简单的“用户请求后端”模型，而是 **“平台用户”和“沙箱内部 Agent/服务”都在调用 API”** 的双边架构。

换句话说，**Suna 的 API 既服务前端，也服务沙箱内部运行时**。

## 七、核心业务逻辑主线

### 7.1 账户初始化与租户解析

`apps/api/src/shared/resolve-account.ts` 显示，用户进入系统后，第一步不是直接创建会话，而是 **解析或懒迁移到账户体系**。逻辑顺序是：

1. 优先从 `account_members` 查找用户归属账户；
2. 若没有，再回退到旧表 `account_user`；
3. 若旧表命中，则懒迁移到新表 `accounts + account_members`；
4. 如果都没有，则创建一个以 `userId` 为主键的个人账户。

这段逻辑说明 **Suna 的控制平面是围绕 account 做统一归属的**，所有后续资源——尤其是 sandbox——都依附 account，而不是直接依附 user session。

### 7.2 “确保有沙箱”是平台主业务入口

`apps/api/src/platform/routes/account.ts` 的 `/init` 与 `apps/api/src/platform/services/ensure-sandbox.ts` 一起构成了 Suna 的主业务入口。

其主线不是“创建对话”，而是：

1. 用户通过 JWT 访问 `/v1/platform/init`；
2. 系统调用 `resolveAccountId(userId)`；
3. 根据 provider/serverType/location 等参数决定目标沙箱类型；
4. 调用 `ensureSandbox()`：
   - 先拿账户级 advisory lock，避免并发重复创建；
   - 查找已有 active/provisioning sandbox；
   - 尝试重新激活 stopped/archived sandbox；
   - 做计费/额度校验；
   - 若开启 pool，则先尝试从池中领用；
   - 若仍无可用实例，再真正调用 provider 创建新实例；
5. 为新沙箱创建 `sandbox` 类型 API key；
6. 把 `KORTIX_TOKEN` 注入沙箱环境。

这个流程说明：**Suna 的产品核心是“保证账户始终有一台可工作的机器”**。Agent 逻辑是运行在这台机器上的，而平台后端负责把“账户 -> 机器”这一层关系稳定维护起来。

### 7.3 沙箱创建与计费是耦合的

`apps/api/src/platform/routes/sandbox-cloud.ts` 把这一点表现得非常明确。对于 `justavps` 这类托管 VPS provider，创建机器之前会先：

1. 校验 server type 与 location 的合法性；
2. 查询或创建 Stripe customer；
3. 检查默认支付方式；
4. 直接创建独立的 Stripe subscription；
5. 再写入 `sandboxes` 表中的 provisioning row；
6. 创建 sandbox-scoped API key；
7. 优先尝试从 pool 领用；
8. 否则调用 provider.create；
9. 失败时回滚订阅、清理 provider 资源并把 sandbox 标记为 error。

这说明 **Suna 的“实例创建”不是纯技术动作，而是业务交易动作**。一台机器对应的不是临时 session，而可能是一个独立收费单元。

### 7.4 预热池是平台优化而非旁支

`pool_resources`、`pool_sandboxes` 表，加上 `ensure-sandbox.ts` 与 `sandbox-cloud.ts` 中的 `pool.grab()`、`pool.injectEnv()`，说明 Suna 有一套比较明确的 **预热池/池化复用逻辑**。它不是简单复用已有用户沙箱，而是：

1. 先准备 provider + serverType + location 维度的池资源；
2. 当用户申请新沙箱时优先 claim；
3. claim 成功后再为该 sandbox 写入 DB 记录；
4. 动态把新的 `KORTIX_TOKEN` 注入被领用的实例；
5. 未命中才真正走 provider.create。

因此，Suna 在代码层面已经体现出一种很典型的 **商业化云平台优化思路**：**把慢的机器准备提前，把用户请求阶段变成“认领 + 注入 + 激活”**。

### 7.5 预览代理把“平台入口”接到“沙箱内部服务”

`apps/api/src/sandbox-proxy/index.ts` 揭示了另一个很关键的执行链路。Suna 并不要求用户直接知道沙箱真实地址，而是通过 `/v1/p/:sandboxId/:port/*` 这样的统一入口做代理。

其逻辑大致是：

1. 通过 `combinedAuth` 做用户或沙箱 token 校验；
2. 根据 `sandboxId/externalId` 查数据库，解析 provider、baseUrl、serviceKey、proxyToken、slug；
3. 若是 JustAVPS，还会检查并刷新 proxy token；
4. 根据 provider 类型分流到 `local_docker`、`daytona`、`justavps` 不同代理方式；
5. 最终把请求转发到沙箱内部实际暴露的服务端口。

这意味着 **Suna 的后端不只是“创建机器”，还扮演“机器服务网关”**。用户访问的网页预览、内部服务、分享链接，本质上都经过平台控制面代理出去。

### 7.6 LLM Router 是能力层，不是系统唯一中心

`apps/api/src/router/index.ts` 显示，Suna 确实也有一套模型与搜索网关，包括 Web Search、Image Search、LLM、Anthropic-compatible 路由与 proxy 路由，并使用 `apiKeyAuth` 做访问控制。

但从整体架构上看，**router 更像被沙箱和平台消费的能力层**，而不是整个平台的绝对中心。因为它并不负责租户、机器、预览、计费和生命周期管理。

## 八、一个典型请求链路

为了把系统架构和业务逻辑连起来，可以把一个典型的用户启动流程写成下面这样。

| 步骤 | 发生位置 | 逻辑 |
|---|---|---|
| 1 | Web 前端 | 用户登录并进入控制台 |
| 2 | API `supabaseAuth` | 校验用户 JWT，提取 `userId` |
| 3 | `resolveAccountId` | 找到账户或完成懒迁移 |
| 4 | `/v1/platform/init` | 请求初始化账户对应的沙箱 |
| 5 | `ensureSandbox` | 查现有、尝试重启、检查额度、尝试 pool claim |
| 6 | Provider 层 | 必要时真正创建机器 |
| 7 | `createApiKey` | 生成 sandbox-scoped service key |
| 8 | DB `sandboxes` | 持久化 externalId/baseUrl/status/config |
| 9 | 沙箱容器 | 通过 `KORTIX_TOKEN` 与平台建立受控连接 |
| 10 | `/v1/p/:sandboxId/:port/*` | 用户后续通过统一代理入口访问沙箱内部服务 |

这个时序能说明一件事：**Suna 的中心业务不是一次性推理请求，而是账户机器的长期可用性管理**。

## 九、从代码角度看，Suna 的业务重心是什么

基于代码结构，我会把 Suna 的业务重心概括为以下四点。

### 9.1 它首先是“账户机器平台”，其次才是“Agent 应用”

沙箱生命周期、Provider 抽象、池化、计费、更新、预览代理这些代码量与结构复杂度，都说明系统把“机器”当成核心资产来经营。

### 9.2 它采用“控制面在外、执行面在内”的架构

API 控制面负责账户、身份、路由、计费、生命周期与代理；`core` 容器负责 Linux 环境、OpenCode、浏览器、工作目录与运行时状态。这是很典型的 **platform control plane + sandbox runtime plane** 模式。

### 9.3 它的多租边界是 account，而不是 workspace/project

虽然前端中有 project、thread、deployment 等概念，但从后端核心建模看，真正的租户边界仍然是 `accountId`。这意味着其天然更适合 **个人账户或小团队共享机器** 的产品模型，而不是强隔离的大型企业多工作区模型。

### 9.4 它的商业化能力已经部分进入核心路径

计费、订阅、支付方式校验、实例与订阅映射、失败回滚，这些都直接嵌入沙箱创建流程。说明 Suna 并非只是在产品外面套一层收费页，而是把收费单元绑定到了核心基础设施对象。

## 十、代码层面的优点与局限

| 维度 | 优点 | 局限 |
|---|---|---|
| 架构清晰度 | 平台、代理、计费、沙箱边界较清楚 | 仍有较多平台逻辑集中在 API 服务内 |
| 核心对象 | 以 account/sandbox 为中心，主线明确 | 对更细粒度 workspace/project tenancy 支持较弱 |
| 商业化可用性 | 机器生命周期与计费已深度集成 | 更复杂的配额、审计、组织策略仍需继续拆分 |
| 运行时模型 | 长期机器 + 持久工作区，适合复杂 Agent | 成本与隔离模型天然比无状态任务更重 |
| 可扩展性 | provider 抽象、池化、更新机制已具雏形 | 任务编排、异步执行与事件总线还不算完全独立 |

## 十一、结论

从代码层面看，**Suna 的真实系统架构并不是“一个 AI 聊天应用”，而是“一个围绕账户级长期沙箱构建的 Agent 操作系统平台”**。它的业务逻辑主轴是：

> **用户进入平台，归属到账户；账户映射到一台或多台可管理的沙箱；平台通过认证、池化、Provider、代理、计费与更新机制来保障这些沙箱长期可用；Agent 与工具运行在沙箱内部，平台再把这些能力以统一入口暴露给前端与外部。**

因此，如果你要从工程视角理解 Suna，最重要的不是先看 prompt 或 agent loop，而是先看这三条主线：**账户模型、沙箱生命周期、平台代理接入**。这三条线基本决定了它的系统骨架；模型路由、搜索、部署和集成则是在这副骨架上的扩展能力。

## 十二、代码参考文件

下列文件是本次梳理的主要依据：

| 文件 | 作用 |
|---|---|
| `README.md` | 产品定位与运行模式 |
| `package.json` | Monorepo 工作区与开发脚本 |
| `apps/api/src/index.ts` | API 总入口与子服务挂载 |
| `apps/api/src/platform/index.ts` | 平台域路由组织 |
| `apps/api/src/platform/routes/account.ts` | 账户初始化与 ensure sandbox 入口 |
| `apps/api/src/platform/routes/sandbox-cloud.ts` | 云端沙箱生命周期、计费与池化主逻辑 |
| `apps/api/src/platform/services/ensure-sandbox.ts` | 账户级沙箱获取、复用与初始化 |
| `apps/api/src/middleware/auth.ts` | 用户 JWT / Kortix Token 双身份认证 |
| `apps/api/src/shared/resolve-account.ts` | 账户解析与懒迁移 |
| `apps/api/src/router/index.ts` | LLM 与搜索网关入口 |
| `apps/api/src/sandbox-proxy/index.ts` | 预览代理与沙箱服务接入 |
| `packages/db/src/schema/kortix.ts` | 核心数据模型 |
| `core/docker/docker-compose.yml` | 沙箱运行时与持久化拓扑 |
