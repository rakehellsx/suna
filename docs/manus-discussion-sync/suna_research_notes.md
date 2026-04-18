# Suna 深度调研笔记

## 官方仓库与定位

- 官方 GitHub：`https://github.com/kortix-ai/suna`
- 官方定位：**The Open-Source Operating System for Running Autonomous Companies**。
- README 强调：一个长期运行的共享机器，多个 agent 共享同一文件系统、数据库、凭证与历史。
- README 明确写到：`Full Linux Ubuntu sandbox`、`persistent memory`、`60+ skills`、`3,000+ integrations`、`cron/webhook triggers`、`multi-channel access`。
- README 明确写到：**The agent runtime is OpenCode**。

## 初步判断

- Suna 更像“公司级共享 agent 操作系统”，而不是严格意义上的强隔离多租户 SaaS。
- 它与 Manus 的相似点在于：通用任务执行、Linux 沙箱、长期运行、技能/记忆/触发器。
- 它与 Manus 的潜在差异在于：其官方叙述强调“one shared machine”与“shared context”，更偏单组织内共享上下文，而非默认租户强隔离。

## 下一步待核实问题

1. 自托管部署结构与依赖组成。
2. Web/API/core runtime 的分层方式。
3. 沙箱是单机容器、远程 sandbox 还是可切换后端。
4. 多用户、工作区、项目、权限边界是否存在。
5. OpenCode 在其中是内嵌 runtime、外部依赖还是经过二次封装。
6. 本地与云托管版本在能力上是否存在差异。

## 官方文档与代码进一步确认

### 部署与依赖

- 自托管文档显示：Suna 提供 Docker Setup 与 Manual Setup 两种方式。
- 当前两种方式都依赖 **cloud Supabase**，文档明确说本地 Supabase 暂不支持。
- Setup Wizard 会配置 Supabase、Daytona、主 LLM provider、搜索 API、MCP、Composio 等。
- 启动后的核心服务至少包括：**Redis、Backend、Frontend**。

### 仓库与运行时结构

- 顶层 monorepo 至少包含 `apps`、`core`、`packages`、`supabase`、`tests` 等目录。
- `pnpm dev:core` / `pnpm dev:sandbox` 指向 `core/docker/docker-compose.yml`，说明 sandbox/core runtime 是单独层。
- API 代码依赖 `@daytonaio/sdk` 与 `@supabase/supabase-js`，表明云沙箱和认证/数据库都不是弱依赖。

### 沙箱与执行模型

- 配置里出现了多个 sandbox provider：`local_docker`、`daytona`、`justavps`。
- API 层存在 `/v1/platform`、`/v1/p` 这类 sandbox lifecycle 与 preview proxy 路由，说明前端/平台与 sandbox 是分层的。
- 本地模式下，API 会自动注册本地 Docker sandbox，并向 sandbox 注入 `KORTIX_TOKEN`、`INTERNAL_SERVICE_KEY` 等认证信息。
- 代码中存在 sandbox proxy、preview subdomain、WebSocket PTY 代理，说明终端/预览/长连接都通过平台侧统一转发。
- 文档要求提供 Daytona API key，说明 Suna 的云端或远程执行环境强依赖 Daytona 能力。
- 代码里还有 JustAVPS provider 与对应的 snapshot / health / repair / proxy token 逻辑，说明其托管版可能带有更完整的远程机器生命周期管理。

### 多用户与隔离线索

- 测试中明确存在 **cross-user isolation** 用例，说明不同用户不能访问他人的 sandbox keys。
- 数据模型和查询中存在 `accountId`、`sandboxes`、账户下多个 sandbox、管理员查看全量 sandbox 等实现。
- 集成系统支持“将一个集成自动链接到该账户下所有 active sandboxes”，说明隔离边界至少是 **account**，而不是整个实例完全共享。
- 同时，官方 README 与 MANIFESTO 强调“one shared machine / shared filesystem / shared credentials / shared history”，这更像 **单个组织/账户内部共享上下文**，而非 SaaS 默认租户硬隔离叙事。

### 初步综合判断

- Suna 不是只有一个本地 agent 进程，而是已经形成了 **前端 + API + 数据库/认证 + sandbox provider + proxy + admin** 的平台型形态。
- 它既有“共享机器/公司操作系统”的产品理念，也有“account-sandbox 边界”的后端实现，因此更像 **组织内共享上下文 + 账户级隔离** 的混合模型。
- 从代码看，它不像“agent 直接在前端容器里裸跑”，而更像 **平台控制层调用本地或远程 sandbox provider**。

## 产品理念与平台结构补充

Suna（文档与代码中也大量使用 Kortix 品牌）并不把自己描述成一个传统的“多会话聊天 Agent 平台”，而是描述成一个 **会长期运行、可持续积累上下文的 AI computer / operating system**。其核心叙事不是“每次对话临时创建一个隔离执行环境”，而是“为一个账户或组织提供一台长期存在的机器，所有 agent 在其上共享文件系统、记忆、凭证与历史”。这意味着它的产品哲学天然更偏向 **长期驻留、共享上下文、组织内复利**，而不是面向强隔离 SaaS 的短生命周期 task sandbox。

从运行结构上看，Suna 已经不是单一仓库中的 demo agent，而是清晰分层的平台。顶层至少分为前端、API、core sandbox runtime、数据库 schema、测试与部署脚本等部分。`apps/api` 作为平台控制层，负责认证、账户初始化、sandbox 生命周期、proxy、计费、集成、管理后台与访问控制；`core/docker/docker-compose.yml` 则定义了真正运行 agent 的沙箱容器。该容器内部明确采用 **OpenCode 作为 agent runtime**，并开放终端、桌面、浏览器与预览相关端口。

core compose 文件揭示了一个非常重要的设计：Suna 把状态划分为 `/workspace`、`/persistent`、`/ephemeral` 三个逻辑区。`/workspace` 用于保存用户代码与项目文件，`/persistent` 用于保存 OpenCode 数据库、浏览器 profile、secrets 与系统内部状态，`/ephemeral` 则承载可更新的 runtime 层。这说明它并不只是简单“起一个 Ubuntu 容器”，而是在认真设计 **持久工作区、运行时升级和状态分层**。

数据库 schema 进一步说明其隔离边界并非“整个实例只有一个全局用户”。Suna 有 `accounts`、`account_members`、`sandboxes` 等正式模型，说明它支持 **账户、成员、账户下多沙箱** 的关系。`sandboxes` 表中包含 `accountId`、provider、status、externalId、baseUrl、lastUsedAt，以及与 Stripe subscription item 的关联；同时还定义了 `pool_resources` 与 `pool_sandboxes`，这说明它已经实现了 **预创建/预热池** 的资源模型，而不是每次都从零创建实例。

`/v1/platform/init` 的路由实现显示，Suna 在平台层会先解析当前用户的 `accountId`，再根据 provider 和套餐策略执行 `ensureSandbox`。也就是说，从实现上它仍然是 **平台控制层为账户确保一个可用沙箱**，而不是前端直接连一个本地 agent 进程。对于本地 Docker 模式，它还实现了 image pull 进度、stale provisioning row 自动修复、active sandbox 返回等平台化逻辑。这些都表明它具备相当成熟的控制面特征。
