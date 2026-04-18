# 开源多租户 Agent 平台候选清单

## 调研目标

筛选并比较同时涉及以下一个或多个维度的开源项目：

1. Agent 平台或 Agent 编排能力。
2. 多用户、多工作区、团队或多租户能力。
3. 沙箱、隔离执行环境、浏览器/终端/file runtime，或对外部沙箱的集成。
4. 开源可自部署，而非纯 SaaS。

## 第一轮候选

### 平台/平台型候选

- Agentbot（声称多租户 AI Agent 平台）
- UnicomAI/wanwu（企业级多租户 Agent 平台）
- AnythingLLM（多用户/工作区型 AI 平台，需确认 Agent 与沙箱能力）
- Dify（工作区、多用户、Agent/Workflow，需确认沙箱策略）
- Flowise（多用户/工作区，需确认 Agent 与执行隔离）
- OpenWebUI（多用户工作区，需确认 Agent 与执行插件能力）

### 沙箱/执行面候选

- agent-infra/sandbox
- alibaba/OpenSandbox
- agent-sandbox/agent-sandbox
- kubernetes-sigs/agent-sandbox
- mycosavant/manus-open-sandbox
- e2b（需确认核心是否开源，以及是否适合列入）

## 下一步重点

1. 确认哪些是真正“开源 + 可自部署”。
2. 区分“完整平台”与“执行底座/沙箱”。
3. 识别是否真的支持多租户，而不只是单用户 self-host。
4. 梳理沙箱是内置、外置集成，还是完全没有。

## 已确认候选：OpenHands

- 官方定位为 **The Open Platform for Cloud Coding Agents**。
- 明确强调 **open-source, customizable, run locally or at scale**。
- 明确强调 **secure, sandboxed runtime you control**，支持 **isolated Docker or Kubernetes environments**。
- 更偏向代码/软件工程 agent，而不是通用网页事务代理。
- 从官网表述看，具备企业与规模化部署倾向，但需要继续确认开源版本中的多租户/团队边界。
- 初步判断：与 Manus 相似点在于自主执行、沙箱、远程运行；差异在于更聚焦 coding agent，而非更广义的通用任务执行代理。

## 已确认候选：Suna / Kortix

- 项目定位为 **The Open-Source Operating System for Running Autonomous Companies**。
- README 明确强调：**Full Linux Ubuntu sandbox、persistent memory、60+ skills、3,000+ integrations、cron/webhook triggers、multi-channel access**。
- 明确写到 **The agent runtime is OpenCode**，这与我们前面讨论的“OpenCode + 沙箱/平台层”非常相关。
- 从产品形态上看，Suna 比 OpenHands 更接近通用自主执行型 Agent，而不只是 coding agent。
- 其核心抽象更像“共享机器/公司操作系统”，强调持久上下文和多 Agent 共用同一环境；这与严格多租隔离模型之间存在潜在张力，需要继续核实其多用户/团队边界。
- 初步判断：在开源项目中，Suna 可能是目前最接近 Manus 交互形态的候选之一。

## 已确认候选：OpenManus

OpenManus 官网将其定义为 **Open-source framework for building AI agents**。从当前公开页面可见，其更像一个通用 Agent 开发框架或基础框架，而不是完整的多租户自主执行平台。初步判断是：它与 Manus 的相似点在于“通用 agent”方向，但在产品层、团队层、多租户层以及完整沙箱执行层上，公开信息明显弱于 OpenHands 或 Suna。

## 已确认候选：agent-infra/sandbox

AIO Sandbox 的定位非常清晰：它是一个 **all-in-one sandbox environment**，把 **Browser、Shell、File、VSCode、Jupyter、MCP** 集成在同一个容器中。它非常适合作为 Manus 类平台的执行底座或本地/自托管运行时，但它本身不是完整的 Agent 平台。其价值在于为上层 agent 提供统一、共享文件系统的执行环境；它与 Manus 的关系更接近“沙箱底层能力”而非“完整产品替代品”。

## 已确认候选：agentbot-opensource

该项目自称为 **open-source multi-tenant AI agent platform**，强调 **Docker isolation、multi-channel、SIWE auth**，并在仓库结构上呈现出 backend、dashboard、gateway、memory、skills、sdk 等明显的平台化分层。就“多租户”这个维度而言，它是目前搜索结果里少数直接公开声称自己支持多租户的平台型项目。不过其生态偏 OpenClaw/A2A/链上支付等特定方向，与 Manus 的通用工作代理路径并不完全一致；更像一个带多租户层的 agent 平台，而非标准意义的通用个人助理式执行代理。

## 已确认候选：sandboxed.sh

sandboxed.sh 的定位是 **Self-hosted orchestrator for AI autonomous agents**，支持在隔离 Linux 工作区中运行 **Claude Code、OpenCode、Amp**。它强调 mission control、isolated workspaces、skills/tools/rules 的 git-backed library、Telegram 集成、调度自动化与模型路由。它比单纯的沙箱更接近平台层，但仍然偏向“编排多个 coding-agent runtime 的运维/任务平台”，而不是像 Manus 那样从同一个产品界面统一承接广义任务。尽管如此，在开源项目中，它对“OpenCode + 隔离工作区 + 任务编排”的覆盖度很高，值得列入高相关候选。

## 已确认候选：OpenSandbox

OpenSandbox 的定位是 **general-purpose sandbox platform for AI applications**，提供多语言 SDK、统一 sandbox API，以及 Docker/Kubernetes runtime。它覆盖 coding agents、GUI agents、evaluation、code execution、RL training 等场景，并强调强隔离与可替换隔离后端。它不是完整的 Agent 产品，而是非常典型的 **平台执行底座/沙箱平台**，适合作为多租户 Agent 平台的底层执行层。

## 已确认候选：Kubernetes Agent Sandbox

Kubernetes 社区的 Agent Sandbox 明确将自己定位为 **secure, isolated execution layer**，用于在 Kubernetes 上安全部署会生成和运行不可信代码的 autonomous agents。它强调标准化 Kubernetes API、隔离后端解耦，以及对高风险 agentic web browsing、computer use、code interpretation 等场景的支持。它与 Manus 的关系不是“成品替代”，而是偏底层的隔离执行基础设施。
