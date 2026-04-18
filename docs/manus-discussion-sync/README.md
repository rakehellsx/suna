# 对话成果同步总览

**作者：Manus AI**

本文档用于说明本次从对话中沉淀并同步到仓库的研究与设计文档范围。内容主要围绕 **Manus 类 Agent 平台架构、Suna 深度调研、OpenCode 集成方式、商业化改造建议，以及数据库与服务拆分** 等主题展开。

## 一、同步内容范围

本次同步的内容并不是原始聊天逐字稿，而是基于对话过程持续整理形成的 **结构化研究文档、架构分析文档与实施建议文档**。因此，仓库中保存的是更适合阅读、复用与后续开发落地的材料，而不是未经整理的会话日志。

| 文件名 | 主题 |
|---|---|
| `agent_sandbox_architecture_full_summary.md` | Agent 平台、沙箱架构、任务时序与复用策略综合总结 |
| `sandbox_reuse_strategy_multitenant.md` | 多租户场景下沙箱复用策略优化 |
| `manus_like_open_source_agent_platforms_research.md` | 接近 Manus 的开源 Agent 平台与沙箱方案调研 |
| `research_agent_platform_candidates.md` | 开源候选项目与第一轮调研线索 |
| `suna_deep_research_report.md` | Suna 深度调研报告 |
| `suna_commercialization_recommendations.md` | 基于 Suna 的商业化改造建议 |
| `suna_technical_implementation_roadmap.md` | Suna 商业化改造的技术实施路线图 |
| `suna_database_and_service_split_checklist.md` | Suna 的数据库与服务拆分清单 |
| `suna_code_architecture_and_business_logic.md` | 从代码层面梳理 Suna 的系统架构与业务逻辑 |
| `suna_source_code_reading_checklist.md` | Suna 各模块源码解读清单 |
| `suna_opencode_integration_analysis.md` | Suna 与 OpenCode 的协作与部署关系分析 |
| `suna_research_notes.md` | Suna 调研过程中的补充笔记 |

## 二、推荐阅读顺序

如果你的目标是先理解全局，再落到 Suna 的具体实现，建议先阅读综合性文档，再进入专项分析。一个比较顺畅的顺序是：先看 `agent_sandbox_architecture_full_summary.md`，建立对 Agent 平台、控制面、沙箱执行面和多租户问题的整体认知；再阅读 `manus_like_open_source_agent_platforms_research.md` 与 `suna_deep_research_report.md`，把 Suna 放回到更大的开源生态中理解；最后进入 `suna_code_architecture_and_business_logic.md`、`suna_opencode_integration_analysis.md`、`suna_technical_implementation_roadmap.md` 和 `suna_database_and_service_split_checklist.md`，完成从认知到实施的过渡。

## 三、使用建议

这些文档适合用于三类用途。其一，是作为 **产品与架构评估材料**，帮助判断 Suna 是否适合作为 Manus 类平台的基础；其二，是作为 **商业化改造蓝图**，为后续多租户、控制面、计费和沙箱治理提供方向；其三，是作为 **源码阅读索引**，帮助研发团队从高层架构进入关键代码路径。
