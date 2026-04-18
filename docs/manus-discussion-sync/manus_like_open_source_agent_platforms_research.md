# 接近 Manus 的开源 Agent 平台与沙箱方案调研

**作者：Manus AI**  
**日期：2026-04-18**

## 摘要

如果把目标限定为 **更接近 Manus 的“自主执行型 Agent 平台”**，而不是一般的聊天应用、RAG 平台或工作流编排器，那么候选范围会明显缩小。真正相关的项目通常需要同时覆盖以下几层能力：其一，Agent 具备连续规划与任务执行能力；其二，具备 **浏览器、终端、文件系统、代码执行** 等可操作的工具面；其三，存在 **沙箱或隔离执行环境**，以承接高风险操作；其四，最好具备某种 **团队、多用户、工作区或多租户** 能力。

基于这一标准，本次调研得到的结论是：**目前开源生态里，很少有单一项目能同时完整覆盖“Manus 式通用 Agent + 沙箱 + 多租户平台化”三层能力。** 更常见的形态有两类。第一类是 **接近产品层的 Agent 平台**，例如 Suna、OpenHands、sandboxed.sh、agentbot-opensource；第二类是 **执行底座/沙箱平台**，例如 OpenSandbox、AIO Sandbox、Kubernetes Agent Sandbox。这意味着，如果你要做商业化多租户产品，现实路径往往不是“直接拿一个开源项目替代 Manus”，而是 **“平台层 + 沙箱层”组合式选型**。[1] [2] [3] [4] [5] [6] [7]

## 调研标准

为了避免把 Dify、Flowise 这类更偏工作流/应用编排的平台与 Manus 混在一起，本次调研采用更严格的筛选标准。

| 维度 | 说明 |
|---|---|
| Agent 自主性 | 是否支持多步任务推进，而不只是单轮问答或静态流程 |
| 工具执行面 | 是否具备浏览器、终端、文件、代码执行等能力 |
| 沙箱/隔离 | 是否内置或明确依赖隔离执行环境 |
| 平台化程度 | 是否有 Dashboard、任务控制、API、集成或工作区概念 |
| 多租户/团队能力 | 是否明确支持多用户、工作区、团队或容器级隔离 |
| 与 Manus 相似度 | 是否接近“通用自主执行代理”，而不只是 coding agent 或单一 sandbox |

按照这个标准，OpenHands、Suna、sandboxed.sh、agentbot-opensource 更接近“平台层候选”；OpenSandbox、AIO Sandbox、Kubernetes Agent Sandbox 更接近“执行层候选”；OpenManus 更接近“框架候选”。

## 候选项目总览

| 项目 | 类型 | 与 Manus 的接近度 | 多租户/团队 | 沙箱/隔离 | 结论 |
|---|---|---:|---|---|---|
| **Suna / Kortix** | 通用自主执行型 Agent 平台 | 高 | 有平台化与多通道特征，但公开资料中多租边界需再核实 | Full Linux Ubuntu sandbox | 当前最接近 Manus 交互形态的开源候选之一 [2] |
| **OpenHands** | Cloud coding agent 平台 | 中高 | 企业/规模化部署明确，但开源版多租边界不如企业版清晰 | Docker/Kubernetes sandboxed runtime | 很强，但更偏 coding agent [1] |
| **sandboxed.sh** | 自托管 autonomous agent orchestrator | 中高 | 适合多任务/多 mission，但更像运维编排器 | Isolated Linux workspaces | 很适合 OpenCode/Claude Code 自托管 [5] |
| **agentbot-opensource** | 多租户 agent 平台 | 中 | 直接声明 multi-tenant | Docker isolation | 多租户表述最明确，但方向更偏 OpenClaw/A2A 生态 [6] |
| **OpenManus** | Agent framework | 中低 | 未见明确多租户平台能力 | 未见强沙箱平台表述 | 更像框架，而不是即用型平台 [3] |
| **OpenSandbox** | 通用沙箱平台 | 平台相似度低，底座相似度高 | 可支撑多租架构，但自身不是完整平台 | Docker/K8s + 强隔离容器 | 很适合做底层执行层 [7] |
| **AIO Sandbox** | 一体化沙箱环境 | 平台相似度低，底座相似度高 | 未强调多租平台 | Browser/Shell/File/VSCode/MCP 单容器 | 很适合作为本地或自托管执行面 [4] |
| **Kubernetes Agent Sandbox** | Kubernetes 隔离执行层 | 平台相似度低，底座相似度高 | 适合多租 K8s 场景 | gVisor/Kata 等解耦隔离层 | 很适合作为生产级隔离层 [8] |

## 一、最像 Manus 的平台层候选

### 1. Suna / Kortix

Suna 的仓库将自己定义为 **“The Open-Source Operating System for Running Autonomous Companies”**，并明确写到它提供 **Full Linux Ubuntu sandbox、persistent memory、60+ skills、3,000+ integrations、cron/webhook triggers、multi-channel access**，同时指出 **“The agent runtime is OpenCode”**。[2] 这一点非常关键，因为它说明 Suna 不是单纯调用某个 LLM API，而是在构建一个更像“云电脑 + Agent runtime + 长期上下文”的系统。

从能力结构看，Suna 与 Manus 的相似点主要在于：它强调 **通用任务执行**，而不只聚焦代码；它以 **Linux 机器** 为核心抽象，让 Agent 能操作 bash、文件、API、文档、数据库与网页；它还有技能、长期记忆、触发器和多通道访问能力。[2] 这与 Manus 类产品的“一个 agent 在一个受控执行环境中长期工作”的思路很接近。

不过，Suna 的设计也有一个明显特点：它强调 **“One shared machine where every agent sees the same filesystem, the same databases, the same credentials, the same history.”**[2] 这对于单企业内的“公司操作系统”很有吸引力，但对严格的商业化多租户 SaaS 来说，也意味着需要额外审视隔离边界。也就是说，**Suna 很像 Manus 的产品体验，但它更偏“单组织共享操作系统”而非默认强多租户隔离的平台基座**。

### 2. OpenHands

OpenHands 官方站点将其定义为 **“The Open Platform for Cloud Coding Agents”**，并强调 **open-source, customizable, run locally or at scale**，同时明确写到 **“Secure, sandboxed runtime you control”**，支持 **isolated Docker or Kubernetes environments, self-hosted or cloud**。[1] 这说明 OpenHands 在工程成熟度和部署形态上都已经明显平台化。

OpenHands 的强项在于：它不是单纯的 coding assistant，而是更接近 **autonomous coding agent**，可面向漏洞修复、PR review、迁移、测试、故障排查等场景，并且支持 API、SDK、GitHub、GitLab、CI/CD、Slack 等集成。[1] 从“Agent + 远程执行 + 沙箱”的角度看，它是目前开源生态中最成熟的候选之一。

但它与 Manus 的差异同样明显。OpenHands 更聚焦 **软件工程 outer loop 自动化**，而 Manus 类产品通常更广义，既处理代码任务，也处理网页事务、信息搜集、文档、文件、浏览器和通用知识工作。因此，如果你的目标是 **做工程团队内部的自主 coding agent 平台**，OpenHands 非常强；如果你的目标是 **做通用个人助理/通用工作代理**，它仍然偏窄。[1]

### 3. sandboxed.sh

sandboxed.sh 的定位是 **“Self-hosted orchestrator for AI autonomous agents”**，支持在隔离 Linux 工作区中运行 **Claude Code、OpenCode、Amp**，并具备 **Mission Control、Isolated Workspaces、Git-backed Library、Telegram Integration、Automations、Model Routing** 等能力。[5] 这使它在开源项目里呈现出一个很有意思的位置：它既不是纯底层沙箱，也不是完整企业 SaaS，而是一个 **自托管的 agent 编排与任务平台**。

它与 Manus 的相似点，在于它确实覆盖了 **任务控制、隔离工作区、agent runtime、技能与自动化** 这几个关键层。[5] 尤其是它对 **OpenCode** 的直接支持，使它很适合那些想把 OpenCode 运行在隔离 Linux 工作区中的团队。相比 OpenHands，它更像“多 agent runtime 的统一 orchestrator”；相比 Suna，它又更偏“为自主 agent 运行提供控制面”，而非更完整的终端产品体验。

如果你的目标是 **自托管 OpenCode/Claude Code，并希望有任务控制、隔离工作区与自动化调度**，sandboxed.sh 是一个很值得深入看的高相关候选。[5]

### 4. agentbot-opensource

agentbot-opensource 在仓库描述中直接声称自己是 **“Open-source multi-tenant AI agent platform. Docker isolation, multi-channel, SIWE auth.”**[6] 从关键词来看，它是这批候选中 **多租户表述最直接** 的一个。此外，仓库结构中可见 backend、dashboard、gateway、memory、skills、sdk 等模块，也说明它不是一个简单 demo，而是按平台思路拆分的。[6]

不过，agentbot-opensource 的生态与定位也比较特殊。它强调 OpenClaw、A2A、webhook bus、链上支付等机制，[6] 这使它更像一个 **特定 agent 经济/网络生态中的多租平台**，而不是标准意义上接近 Manus 的“通用工作代理”。换言之，它在 **多租户** 这个维度很值得研究，但在 **通用性和产品路径** 上，与 Manus 不完全同向。

## 二、框架层候选

### OpenManus

OpenManus 官网把自己定义为 **“Open-source framework for building AI agents”**。[3] 从当前公开页面能看到的关键信息来看，它更像一个 **Agent 开发框架**：强调通用 agent、工具集成、开源社区，而不是完整的平台化产品。

这意味着 OpenManus 的适用方式更偏“二次开发”。如果你想研究开源社区如何复刻 Manus 的基础能力，它值得参考；但如果你要找 **即用型、带多租管理和沙箱执行面的开源替代品**，OpenManus 本身还不够。[3]

## 三、沙箱/执行底座层候选

### 1. OpenSandbox

OpenSandbox 将自己定义为 **“general-purpose sandbox platform for AI applications”**，提供多语言 SDK、统一 sandbox API，以及 Docker/Kubernetes runtime，并支持 coding agents、GUI agents、agent evaluation、code execution 等场景。[7] 它还强调 gVisor、Kata Containers、Firecracker microVM 等强隔离能力。[7]

这类项目与 Manus 的关系，不是“成品替代”，而是 **基础设施层对应**。如果你想做商业化多租户平台，OpenSandbox 很适合承担 **执行层/沙箱层** 的职责，而把任务编排、会话状态、计费、权限、租户治理放在上层控制面实现。

### 2. AIO Sandbox

AIO Sandbox 的 README 非常直接：它把 **Browser、Shell、File、VSCode、Jupyter、MCP** 都放进一个统一容器里，并强调共享文件系统、MCP-compatible APIs 和一体化 agent sandbox environment。[4] 这让它很适合 **本地开发、自托管单租环境、或作为上层 agent 的统一执行环境**。

但它的问题也很明确：它更像 **单容器执行面**，而不是一个自带多租调度、工作区治理和配额策略的平台。因此，AIO Sandbox 很适合做“Manus 类系统的 runtime substrate”，但不适合直接当成完整多租户产品。

### 3. Kubernetes Agent Sandbox

Kubernetes Agent Sandbox 提供的是 **secure, isolated execution layer**，专门用于在 Kubernetes 上安全运行会生成和执行不可信代码的 autonomous agents。[8] 它强调标准化 Kubernetes API 与隔离后端解耦，可支持 gVisor、Kata Containers 等不同后端。[8]

如果你的重点是 **生产级多租户隔离**，这个方向的价值很高。它可以作为控制面下方的 **通用执行层 CRD/资源抽象**，特别适合企业级集群中需要严格隔离的 agent workloads。但同样地，它本身并不提供完整的终端产品层。

## 哪些项目更适合什么目标

| 你的目标 | 更推荐的项目/组合 | 原因 |
|---|---|---|
| 找最像 Manus 的开源产品体验 | **Suna / Kortix** | 通用自主执行、Linux sandbox、技能、记忆、触发器、OpenCode runtime [2] |
| 做工程团队内部 coding agent 平台 | **OpenHands** | 平台成熟、沙箱明确、云端/自托管都强，但更偏 coding [1] |
| 自托管 OpenCode/Claude Code 任务平台 | **sandboxed.sh** | 编排层明显，隔离工作区与任务控制做得较完整 [5] |
| 研究多租户平台形态 | **agentbot-opensource** | 直接强调 multi-tenant 与 Docker isolation [6] |
| 自研 Manus 类平台的底层执行面 | **OpenSandbox** 或 **Kubernetes Agent Sandbox** | 生产级隔离与统一 sandbox API 更强 [7] [8] |
| 快速搭一个统一的本地执行沙箱 | **AIO Sandbox** | Browser/Shell/File/MCP/VSCode 一体化 [4] |
| 做 Agent 框架级二次开发 | **OpenManus** | 更像框架，而不是产品化平台 [3] |

## 推荐的选型思路

如果你的目标是 **找一个直接接近 Manus 的开源替代品**，我会把优先级排成：**Suna > OpenHands > sandboxed.sh > OpenManus**。这里的排序依据不是成熟度单一维度，而是 **产品形态接近度**。Suna 最像“通用自主执行代理”；OpenHands 最成熟，但偏 coding；sandboxed.sh 很适合自托管 runtime orchestration；OpenManus 更适合作为框架起点。

如果你的目标是 **做商业化多租户产品**，我反而不建议只盯“哪个最像 Manus”，而要优先看 **“平台层 + 沙箱层”的组合**。一个更现实的架构组合可能是：

| 层次 | 推荐候选 |
|---|---|
| Agent / 编排层 | Suna、OpenHands、sandboxed.sh、或自研 |
| 沙箱 / 执行层 | OpenSandbox、Kubernetes Agent Sandbox、AIO Sandbox |
| 多租户控制面 | 自研工作区、权限、配额、计费、审计与资源调度 |

这背后的原因是：**开源项目中最成熟的通常要么是“Agent 平台”，要么是“沙箱底座”，但很少有人把多租户、计费、权限、长期会话、浏览器、终端、文件、任务编排、审计和产品交互全部一次性做好。** 因此，真正面向商用的路线，往往是 **借开源项目补齐某几层，而不是期待一个项目完全复刻 Manus。**

## 结论

综合来看，若以“像 Manus”作为唯一标准，**Suna / Kortix** 是目前最值得优先关注的开源候选；若以“工程成熟度和可部署性”作为标准，**OpenHands** 非常强，但范围更偏 coding；若以“OpenCode 自托管 + 隔离工作区 + 任务控制”为目标，**sandboxed.sh** 很实用；若以“多租户平台研究”为目标，**agentbot-opensource** 值得看；若以“生产级执行底座”为目标，则应重点关注 **OpenSandbox** 与 **Kubernetes Agent Sandbox**。[1] [2] [5] [6] [7] [8]

> **一句话总结：现在开源世界里，“最像 Manus 的产品层”与“最像 Manus 的沙箱层”通常不在同一个项目里，真正可商用的方案更像是组合架构，而不是单项目替代。**

## References

[1]: https://openhands.dev/ "OpenHands | The Open Platform for Cloud Coding Agents"
[2]: https://github.com/kortix-ai/suna "GitHub - kortix-ai/suna: The Autonomous Company Operating System"
[3]: https://openmanus.github.io/ "OpenManus - Open-source Framework for Building AI Agents"
[4]: https://github.com/agent-infra/sandbox "GitHub - agent-infra/sandbox: All-in-One Sandbox for AI Agents"
[5]: https://github.com/Th0rgal/sandboxed.sh "GitHub - Th0rgal/sandboxed.sh: Self-hosted orchestrator for AI autonomous agents"
[6]: https://github.com/Eskyee/agentbot-opensource "GitHub - Eskyee/agentbot-opensource: Open-source multi-tenant AI agent platform"
[7]: https://github.com/alibaba/OpenSandbox "GitHub - alibaba/OpenSandbox: Secure, Fast, and Extensible Sandbox runtime for AI agents"
[8]: https://agent-sandbox.sigs.k8s.io/ "Agent Sandbox"
