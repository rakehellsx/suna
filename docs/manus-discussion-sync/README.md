# Manus 类 Agent 平台与 Suna 深度研究文档索引

**作者：Manus AI**  
**更新日期：2026-04-19**

本文档用于说明本次同步到仓库 `docs/manus-discussion-sync/` 目录中的研究材料范围、阅读路径与使用方式。这里保存的不是原始对话逐字稿，而是围绕 **Manus 类多租户 Agent 平台架构、Suna 深度调研、OpenCode 集成、商业化改造、沙箱执行层治理与集群部署** 等主题整理而成的结构化文档。

## 一、文档总览

本目录中的文档已经从“概念判断”逐步推进到“代码结构”、“商业化改造”和“沙箱集群部署”层面，因此适合同时服务于架构评估、研发落地和实施规划三类场景。为了便于快速定位，下面先给出一张总表。

| 文件名 | 主题 | 说明 |
| --- | --- | --- |
| `agent_sandbox_architecture_full_summary.md` | 综合架构总览 | 总结 Agent 平台控制面、执行面、任务时序与沙箱复用策略 |
| `sandbox_reuse_strategy_multitenant.md` | 沙箱复用策略 | 分析多租户场景下的分层复用、warm pool 与 TTL 策略 |
| `manus_like_open_source_agent_platforms_research.md` | 开源项目调研 | 调研接近 Manus 的开源 Agent 平台与沙箱方案 |
| `research_agent_platform_candidates.md` | 候选项目线索 | 记录第一轮候选平台与调研起点 |
| `suna_deep_research_report.md` | Suna 深度研究 | 梳理产品定位、架构、沙箱模型与开源边界 |
| `suna_commercialization_recommendations.md` | 商业化改造建议 | 围绕多租户、控制面、计费与执行面治理提出改造方向 |
| `suna_technical_implementation_roadmap.md` | 技术路线图 | 按阶段给出从当前开源形态到商业化平台的实施路径 |
| `suna_database_and_service_split_checklist.md` | 数据库与服务拆分 | 拆解核心域模型、服务边界与迁移顺序 |
| `suna_code_architecture_and_business_logic.md` | 代码架构分析 | 从 monorepo 与后端链路角度梳理 Suna 核心结构 |
| `suna_source_code_reading_checklist.md` | 源码阅读索引 | 按模块、优先级与阅读顺序整理源码入口 |
| `suna_opencode_integration_analysis.md` | OpenCode 集成分析 | 解释 OpenCode 如何内嵌在沙箱镜像中并由网关管理 |
| `suna_research_notes.md` | 研究补充笔记 | 保留若干中间观察与补充事实 |
| `suna_sandbox_cluster_deployment_guide.md` | Suna 沙箱集群结论 | 回答 Suna 是否支持沙箱集群模式，并给出分层部署方案 |
| `daytona_sandbox_cluster_deployment_detailed_guide.md` | Daytona 详细集群部署 | 聚焦 Daytona 作为 Suna 执行层时的 Kubernetes 详细实施步骤 |

## 二、推荐阅读顺序

如果你的目标是先建立全局判断，再逐步进入 Suna 与 Daytona 的实施细节，建议采用由总到分的阅读顺序。

首先阅读 `agent_sandbox_architecture_full_summary.md`，建立对 **控制面 / 执行面分离、沙箱调用时序、多租户复用策略** 的整体认知。接着阅读 `manus_like_open_source_agent_platforms_research.md` 与 `suna_deep_research_report.md`，把 Suna 放回更大的开源 Agent 生态中理解。完成这一层之后，再进入 `suna_commercialization_recommendations.md`、`suna_technical_implementation_roadmap.md` 与 `suna_database_and_service_split_checklist.md`，把“是否可用”推进到“如何落地商业化改造”。

如果你更偏向工程实施，则建议在上述基础上继续阅读 `suna_code_architecture_and_business_logic.md`、`suna_source_code_reading_checklist.md` 与 `suna_opencode_integration_analysis.md`，以便从高层设计下钻到实际代码路径。最后阅读 `suna_sandbox_cluster_deployment_guide.md` 与 `daytona_sandbox_cluster_deployment_detailed_guide.md`，完成从平台控制面到沙箱执行面集群部署的闭环。

## 三、针对不同角色的阅读建议

不同角色关注的切入点并不相同，因此可以按角色快速选读。

| 角色 | 优先阅读 | 目的 |
| --- | --- | --- |
| 架构师 / CTO | `agent_sandbox_architecture_full_summary.md`、`suna_deep_research_report.md`、`suna_commercialization_recommendations.md` | 判断 Suna 是否适合作为 Manus 类平台底座 |
| 后端负责人 | `suna_code_architecture_and_business_logic.md`、`suna_database_and_service_split_checklist.md` | 理解服务边界、配置体系与改造顺序 |
| 平台 / Infra 团队 | `suna_sandbox_cluster_deployment_guide.md`、`daytona_sandbox_cluster_deployment_detailed_guide.md` | 规划平台层与执行层的集群部署 |
| 研发团队 | `suna_source_code_reading_checklist.md`、`suna_opencode_integration_analysis.md` | 快速进入关键模块与调用路径 |
| 产品 / 战略团队 | `manus_like_open_source_agent_platforms_research.md`、`suna_commercialization_recommendations.md` | 比较开源方案与商业化演进空间 |

## 四、本次新增内容说明

本轮新增的重点在于把“沙箱能否集群化”从结论层推进到实施层。

`suna_sandbox_cluster_deployment_guide.md` 主要回答 **Suna 是否支持集群模式** 这一问题，并明确区分 **平台层集群化** 与 **Daytona 执行层集群化**。`daytona_sandbox_cluster_deployment_detailed_guide.md` 则进一步聚焦 Daytona 本身，详细说明如何使用官方 Helm Charts、自定义 Region、外部数据库与 Redis、Wildcard TLS、Proxy、Runner 以及最终与 Suna 对接的配置方式。这两份文档结合起来，已经能够支撑从“技术判断”进入“实施准备”。

## 五、使用建议

这些文档并不是相互割裂的独立报告，而是围绕同一目标逐层递进的研究资产。最适合的使用方式，是把它们同时作为 **架构评审材料**、**研发实施蓝图** 和 **源码阅读索引**。在真正启动商业化改造时，建议把本文档目录视为一套“前期设计输入”，后续再继续沉淀为 ADR、服务拆分任务清单、Helm values 仓库、环境基线与 SRE Runbook。

如果后续仍继续围绕 Suna / Daytona 演进，这个目录建议持续追加三类文档：一类是 **生产环境配置样板**，一类是 **容量规划与压测报告**，另一类是 **计费与审计模型设计**。这样才能从“研究闭环”进一步过渡到“可运行的工程闭环”。
