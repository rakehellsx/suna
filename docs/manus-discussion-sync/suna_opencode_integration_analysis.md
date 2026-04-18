# Suna 与 OpenCode 的协作关系及部署方式分析

**作者：Manus AI**

## 一、结论摘要

从代码实现看，**Suna 并不是把 OpenCode 当作一个外部 SaaS 来调用**，而是把它作为 **每个沙箱内部的核心 Agent Runtime** 来运行；而 Suna 自己的平台 API、账户模型、沙箱生命周期管理、代理层与权限控制，则位于 OpenCode 之外，形成一层更高的 **platform control plane**。换句话说，Suna 与 OpenCode 的关系更接近 **“平台控制层 + 沙箱内嵌 Agent Runtime”**，而不是“前端直接连 OpenCode”或“OpenCode 独立托管，Suna 仅作 UI”。[1] [2] [3]

同时，OpenCode 在 Suna 中的部署方式也比较明确：它被 **直接安装进沙箱镜像**，在容器启动后由 **Kortix Master 的 ServiceManager** 管理和拉起，监听固定端口 `4096`；而对外暴露时，并不是直接把这个端口裸露给客户端，而是先经过 **Kortix Master** 和更外层的平台代理，再转发给 OpenCode。[4] [5] [6] [7]

这里还需要补一个表述上的精确区分。说 **“OpenCode 和沙箱装在一起”**，在工程语义上是 **基本成立** 的，因为 OpenCode 的二进制、默认配置和启动脚本都被打进了沙箱镜像，运行时进程也确实在沙箱容器内部启动。[5] [12] [14] 但如果进一步追问，OpenCode 并 **不等于** 沙箱本身。Suna 的一个沙箱实例实际上同时包含 **Linux 执行环境、持久化工作区、浏览器 profile、Kortix Master 网关，以及 OpenCode Runtime**；OpenCode 只是其中负责 Agent 会话、工具调用与运行时编排的那一层。[2] [4] [5] [7]

| 问题 | 结论 |
|---|---|
| Suna 与 OpenCode 如何协作 | Suna 平台层负责任务入口、账户/沙箱管理、代理与治理；OpenCode 作为沙箱内 Agent Runtime 承担会话、工具、Agent 执行与 Web UI |
| OpenCode 是否在沙箱里运行 | 是，运行在每个沙箱容器内部 |
| 能否简单说“OpenCode 和沙箱装在一起” | 可以粗略这么说，但更准确是“OpenCode 内嵌在沙箱实例中，属于沙箱内的一个核心运行时组件” |
| OpenCode 是否独立部署为平台级共享服务 | 不是，代码显示它是沙箱内实例级运行 |
| 谁负责把请求转给 OpenCode | 沙箱内是 Kortix Master，平台外层还有 path-based proxy |
| OpenCode 的状态放哪里 | 主要持久化到 `/workspace/.local/share/opencode/` 等工作区持久目录 |
| 一个完整沙箱实例包含什么 | Linux 环境、Kortix Master、OpenCode、浏览器与持久化工作区 |

## 二、Suna 与 OpenCode 的协作边界

Suna 顶层 README 已经明确写出，系统提供的是一个可持续运行的 Linux 云电脑，而 **Agent runtime 就是 OpenCode**。[1] MANIFESTO 进一步说明，一个 Kortix 实例本身就是带完整桌面环境与 OpenCode agent framework 的 Docker sandbox，内部不仅有 Agent，还包括技能、记忆、会话历史和若干后台服务。[2]

这意味着在架构边界上，Suna 至少分为两层。第一层是 **Suna/Kortix 平台层**，负责账户、成员、沙箱生命周期、外部 API、鉴权、平台代理、云端路由与账户级活动沙箱选择。第二层是 **沙箱内部运行时层**，这里真正承载 Agent 执行、OpenCode Session、OpenCode 工具、技能、插件和面向工作区的持久状态。[3] [8] [9]

从代码上看，这个边界非常清晰。`apps/api/src/platform/routes/sandbox-cloud.ts` 与 `ensure-sandbox.ts` 这类文件负责的是“拿到哪个沙箱”“启动、停止、归档、复用哪个沙箱”；而一旦请求真正进入沙箱内部，就会被转给 `core/kortix-master`，由它继续代理到 OpenCode 或直接处理文件、代理、分享、偏好设置等网关能力。[3] [9] [10]

可以把两者关系概括为下面这个分层。

| 层级 | 主要组件 | 职责 |
|---|---|---|
| 平台控制层 | `apps/api`、平台路由、账户解析、云沙箱路由 | 用户鉴权、账户/沙箱绑定、外层代理、云端控制 |
| 沙箱网关层 | `core/kortix-master` | 沙箱内统一入口、OpenCode 代理、文件/端口/分享/偏好 API |
| Agent Runtime 层 | OpenCode | Session、消息、工具、Agent 执行、Web UI、配置与技能加载 |
| 持久化工作区层 | `/workspace` 下各目录 | OpenCode 会话状态、浏览器 profile、技能配置、缓存、用户文件 |

## 三、请求是如何从 Suna 走到 OpenCode 的

Suna 的后端入口 `apps/api/src/index.ts` 很关键。它不仅负责普通 API，还负责 **基于路径的沙箱代理和 WebSocket 升级代理**。代码中明确说明了 `/v1/p/{sandboxId}/{port}/*` 这种路径格式，用于把请求转发到目标沙箱；而且特别指出这条链路会继续代理到沙箱内的 Kortix Master，再由 Kortix Master 转发给 OpenCode，例如 PTY、SSE-over-WS 等长连接也沿着这条链路工作。[6]

进入沙箱后，`core/s6-services/svc-kortix-master/run` 会启动 Kortix Master，固定监听 `8000`，同时把 `OPENCODE_HOST=localhost` 和 `OPENCODE_PORT=4096` 注入进去，说明 Kortix Master 在设计上就被当作 **OpenCode 的上游统一网关**。[4]

`core/kortix-master/src/index.ts` 又进一步坐实了这件事。它在启动时显式导入 `proxyToOpenCode`，检查 OpenCode 健康状态，必要时请求恢复 `opencode-serve` 服务；并在大部分未被网关路由截获的请求上，统一走 catch-all proxy 转发到 OpenCode。[7] `spec-merger.ts` 甚至会把 Kortix Master 自己的 OpenAPI 与 OpenCode 的 `/doc` 实时合并，说明从产品视角，Suna 希望把 OpenCode 与沙箱网关暴露成一个统一能力平面。[11]

因此，一次典型调用链更接近下面这样。

| 步骤 | 请求流向 | 说明 |
|---|---|---|
| 1 | 前端 / 客户端 → `apps/api` | 进入 Suna 平台 API |
| 2 | `apps/api` → `/v1/p/{sandboxId}/{port}` 代理 | 解析目标沙箱与目标端口 |
| 3 | 代理进入沙箱 `8000` 端口 | 命中沙箱内 Kortix Master |
| 4 | Kortix Master 路由判断 | 文件、分享、端口代理等由自己处理 |
| 5 | 其他大多数 Agent API | 代理到 `localhost:4096` 的 OpenCode |
| 6 | OpenCode 处理 session / message / tool | 真正执行 Agent Runtime 逻辑 |

这说明 **Suna 与 OpenCode 并不是平级关系**。OpenCode 更像“沙箱内核”，而 Suna/Kortix 则是“平台与网关外壳”。

## 四、OpenCode 在 Suna 项目中是怎么部署的

从 Dockerfile 看，OpenCode 并不是运行时临时下载或 sidecar 挂上去的，而是 **在构建沙箱镜像时直接安装**。`core/docker/Dockerfile` 中先定义了 `OPENCODE_VERSION`，随后在镜像构建阶段通过 `npm install -g "opencode-ai@${OPENCODE_VERSION}"` 安装全局二进制，并在之后应用补丁 `patches/apply.sh`，说明 Suna 不只是使用原版 OpenCode，还对其做了定制化修补。[5]

更重要的是，Dockerfile 明确区分了三类区域：`/workspace` 为持久层，`/ephemeral` 为 shipped runtime，`/opt` 为第三方依赖。文档中把 `.local/share/opencode/`、`.opencode/`、`.cache/opencode/` 都归到 `/workspace` 下的持久目录，说明 OpenCode 的会话数据库、日志、快照、配置和技能安装结果，都设计成 **跟随工作区持久化**，而不是跟随镜像层。[5]

`run-opencode-serve.sh` 则展示了它的真实运行方式。脚本会设置：

> `HOME=/workspace`、`OPENCODE_STORAGE_BASE=${KORTIX_PERSISTENT_ROOT}/opencode`、`AUTH_JSON_PATH=${OPENCODE_STORAGE_BASE}/auth.json`、`OPENCODE_CONFIG_DIR=/ephemeral/kortix-master/opencode`，最后执行 `opencode serve --port 4096 --hostname 0.0.0.0`。[12]

这说明 OpenCode 在 Suna 中的部署形态是：**二进制与默认配置打进镜像，运行时进程在沙箱内启动，核心数据落到持久卷，配置目录默认来自镜像内的 Kortix Master 附带配置树**。[5] [12]

## 五、OpenCode 是由谁启动和托管的

表面上看，Suna 有一个 `svc-opencode-serve` s6 服务，但真正的实现已经不是传统意义上的 s6 长驻进程。其 `run` 文件明确写着：这个 s6 service 只是一个 **占位符**，真正的 OpenCode API 进程已经由 **Kortix Master 的 central ServiceManager** 端到端接管。[13]

`service-manager.ts` 里对 `opencode-serve` 的定义更直接。它把这个服务声明为 `adapter: "spawn"` 的 builtin service，启动命令是 `bash /ephemeral/kortix-master/scripts/run-opencode-serve.sh`，端口是 `4096`，健康检查是 TCP，且默认 `autoStart: true`。[14] 这说明 OpenCode 并不是独立的 Docker container，也不是系统级 daemon，而是 **沙箱容器内由 Kortix Master 进程管理的受控子服务**。

| 组件 | 在 Suna 中的角色 | 证据 |
|---|---|---|
| `svc-opencode-serve` | 占位服务槽，不负责真正托管 | [13] |
| `ServiceManager` | OpenCode 的真实生命周期管理者 | [14] |
| `run-opencode-serve.sh` | 启动包装器 | [12] |
| `opencode serve --port 4096` | OpenCode 实际启动命令 | [12] |
| Kortix Master | 发现异常后可触发恢复与重启 | [7] [15] |

这也是为什么 `runtime-reload.ts` 中会区分“dispose OpenCode instance”和“full restart services”两种模式：前者是对 OpenCode 实例做热重载，后者则通过 ServiceManager / s6 层级做更彻底的代码与服务重启。[15]

## 六、OpenCode 的配置、状态和持久化模型

Suna 对 OpenCode 的持久化设计是比较重的，不只是保 session，而是把 **几乎所有与工作区相关的 Agent 状态** 都放进持久卷。

Dockerfile 中点名了以下目录：`.local/share/opencode/` 作为 OpenCode state、sessions 与 SQLite DB 所在地，`.opencode/` 用于 OpenCode 配置、skills、agents，`.cache/opencode/` 作为运行时缓存。[5] 这意味着容器重建后，只要 `/workspace` 这块卷还在，OpenCode 的会话和用户安装结果就仍可恢复。

另外，`marketplace.ts` 还说明，Suna 会确保真实工作区里存在 `.opencode` 目录，并维护 `ocx.jsonc`、`opencode.jsonc` 等配置文件；如果有历史路径，还会把旧的 `.kortix/.opencode` 迁移到新目录。这说明 **OpenCode 的用户级配置并不完全固定在镜像内，而是支持工作区级增量变更与持久保留**。[16]

`legacy-migrate.ts` 和 `auth-sync.ts` 则进一步表明，OpenCode 的 SQLite 数据库和认证文件都被当作长期状态来直接操作或同步；它们的路径默认指向 `OPENCODE_STORAGE_BASE` 下的 `opencode.db` 与 `auth.json`。[17] [18]

从部署语义看，可以把它总结为：

| 类别 | 默认位置 | 是否持久化 | 说明 |
|---|---|---|---|
| OpenCode 二进制 | 全局安装路径 | 否，跟镜像走 | 镜像构建时安装 [5] |
| OpenCode 默认配置树 | `/ephemeral/kortix-master/opencode` | 否，跟镜像走 | shipped runtime [5] [12] |
| OpenCode 会话与 SQLite DB | `/workspace/.local/share/opencode/` | 是 | 会话、日志、快照 [5] |
| OpenCode 缓存 | `/workspace/.cache/opencode/` | 是 | 运行时缓存 [5] |
| OpenCode 用户配置与技能 | `/workspace/.opencode/` | 是 | Marketplace 与用户侧安装结果 [5] [16] |
| 浏览器 profile | `/workspace/.browser-profile/` | 是 | 为 Agent 的浏览器能力保状态 [5] |

## 七、Suna 为什么要在 OpenCode 外面再包一层 Kortix Master

从代码可以看出，Kortix Master 并不是简单反代，而是在补 OpenCode 不负责、或平台需要自己掌控的能力。

首先，`files.ts` 说明文件管理是 **authoritative file I/O layer**，故意不再简单代理给 OpenCode，而是直接由沙箱网关控制二进制下载、上传、删除等行为。[19] 其次，`proxy.ts`、`web-proxy.ts`、`share.ts`、`share-proxy.ts` 则提供通用端口代理、网页代理和公开分享链路，这些都属于 **沙箱级基础设施能力**，不是典型 Agent Runtime 自带能力。[20] [21] [22] [23]

再者，`preferences.ts` 和 `tasks.ts` 展示了两种不同集成方式：有些功能直接通过 HTTP 调 OpenCode，例如 `/config` 或 session/task 相关接口；有些则由 Suna/Kortix 自己维护一层更高抽象，比如项目、分享、文件、连接器等。[24] [25]

这说明 Suna 对 OpenCode 的使用方式并不是“原样暴露”，而是 **把 OpenCode 内核嵌入到自己的沙箱 OS / gateway 中，再由平台层统一暴露给前端和外部 API**。这也解释了为什么它更像 Manus 类产品，而不是单纯的 OpenCode 包装壳。

## 八、回答你的两个具体问题

### 1. 它与 OpenCode 是如何协作的？

如果用一句话概括：**Suna 用 OpenCode 做沙箱内 Agent Runtime，用 Kortix Master 做沙箱内统一网关，用平台 API 做沙箱外控制面。**[1] [4] [7] [14]

也就是说，OpenCode 负责“Agent 真正工作”的那一层，包括 session、message、tool、agent、Web UI 等；而 Suna 负责“Agent 被如何托管、如何被找到、如何被鉴权、如何被复用、如何同沙箱和账户绑定”的那一层。[3] [7] [9] [14]

### 2. 在 Suna 项目中，OpenCode 是怎么部署的？

也是一句话概括：**OpenCode 被打进沙箱镜像，在每个沙箱容器内以本地服务形式启动，默认监听 4096 端口，由 Kortix Master 的 ServiceManager 管理，状态持久化到 `/workspace`。**[5] [12] [14]

如果换成更口语的说法，可以理解为：**OpenCode 和沙箱是“装在一起”的。** 但更严格地说，应该表达为：**OpenCode 被内嵌部署在沙箱实例中，是沙箱内部的核心 Agent Runtime 组件之一。**[4] [5] [12] 它不是单独的共享云服务，也不是前端浏览器里运行的组件，更不是只在开发环境启动一次的本地依赖，而是 **每个沙箱实例的一部分**。因此，Suna 的基本运行单元并不是“一个全局 OpenCode”，而是“一个沙箱 = Linux 执行环境 + 一份 OpenCode Runtime + 一层 Kortix Master + 浏览器/文件系统状态 + 持久化工作区”。[2] [4] [5] [12]

为了避免概念混淆，可以用下面这张表来理解。

| 说法 | 是否准确 | 更精确的解释 |
|---|---|---|
| OpenCode 和沙箱装在一起 | 基本准确 | OpenCode 被打进沙箱镜像，并在沙箱容器内部启动 |
| OpenCode 就是沙箱 | 不准确 | 沙箱还包含 Linux 环境、网关、浏览器、持久卷等 |
| Suna 在平台侧统一部署一个 OpenCode 给所有用户共用 | 不准确 | 代码显示 OpenCode 是按沙箱实例内嵌运行 |
| OpenCode 是沙箱内部的 Agent Runtime | 准确 | 它负责 session、message、tool、agent 与 Web UI |

## References

[1]: https://github.com/kortix-ai/suna "Suna README"
[2]: https://github.com/kortix-ai/suna/blob/main/MANIFESTO.md "Suna MANIFESTO"
[3]: https://github.com/kortix-ai/suna/blob/main/apps/api/src/platform/routes/sandbox-cloud.ts "Suna platform sandbox cloud routes"
[4]: https://github.com/kortix-ai/suna/blob/main/core/s6-services/svc-kortix-master/run "Kortix Master s6 run script"
[5]: https://github.com/kortix-ai/suna/blob/main/core/docker/Dockerfile "Suna sandbox Dockerfile"
[6]: https://github.com/kortix-ai/suna/blob/main/apps/api/src/index.ts "Suna API server entry"
[7]: https://github.com/kortix-ai/suna/blob/main/core/kortix-master/src/index.ts "Kortix Master entry"
[8]: https://github.com/kortix-ai/suna/blob/main/apps/api/src/platform/index.ts "Suna platform router entry"
[9]: https://github.com/kortix-ai/suna/blob/main/apps/api/src/platform/services/ensure-sandbox.ts "Suna ensure sandbox service"
[10]: https://github.com/kortix-ai/suna/blob/main/apps/api/src/sandbox-proxy/index.ts "Suna sandbox proxy entry"
[11]: https://github.com/kortix-ai/suna/blob/main/core/kortix-master/src/services/spec-merger.ts "Kortix Master spec merger"
[12]: https://github.com/kortix-ai/suna/blob/main/core/kortix-master/scripts/run-opencode-serve.sh "OpenCode serve startup wrapper"
[13]: https://github.com/kortix-ai/suna/blob/main/core/s6-services/svc-opencode-serve/run "OpenCode serve placeholder s6 service"
[14]: https://github.com/kortix-ai/suna/blob/main/core/kortix-master/src/services/service-manager.ts "Kortix Master ServiceManager"
[15]: https://github.com/kortix-ai/suna/blob/main/core/kortix-master/src/services/runtime-reload.ts "Kortix runtime reload service"
[16]: https://github.com/kortix-ai/suna/blob/main/core/kortix-master/src/routes/marketplace.ts "Kortix marketplace workspace preparation"
[17]: https://github.com/kortix-ai/suna/blob/main/core/kortix-master/src/routes/legacy-migrate.ts "Kortix legacy OpenCode DB migration"
[18]: https://github.com/kortix-ai/suna/blob/main/core/kortix-master/src/services/auth-sync.ts "Kortix auth sync service"
[19]: https://github.com/kortix-ai/suna/blob/main/core/kortix-master/src/routes/files.ts "Kortix files route"
[20]: https://github.com/kortix-ai/suna/blob/main/core/kortix-master/src/routes/proxy.ts "Kortix dynamic port proxy"
[21]: https://github.com/kortix-ai/suna/blob/main/core/kortix-master/src/routes/web-proxy.ts "Kortix web forward proxy"
[22]: https://github.com/kortix-ai/suna/blob/main/core/kortix-master/src/routes/share.ts "Kortix share route"
[23]: https://github.com/kortix-ai/suna/blob/main/core/kortix-master/src/routes/share-proxy.ts "Kortix share proxy route"
[24]: https://github.com/kortix-ai/suna/blob/main/core/kortix-master/src/routes/preferences.ts "Kortix preferences route"
[25]: https://github.com/kortix-ai/suna/blob/main/core/kortix-master/src/routes/tasks.ts "Kortix tasks route"
