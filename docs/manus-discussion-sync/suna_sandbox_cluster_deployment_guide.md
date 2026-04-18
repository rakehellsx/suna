# Suna 沙箱是否支持集群模式部署：结论与详细实施步骤

作者：**Manus AI**  
日期：2026-04-19

## 结论摘要

**结论是支持，但必须分两层理解。** Suna 本身并不是把所有沙箱直接硬编码在 API 进程内部，而是把**平台控制层**与**沙箱执行层**分离。官方自托管文档已经明确提供了 **AWS ECS** 与 **AWS EKS** 两种生产部署方案，说明 **Suna 平台层本身可以做集群化部署**。[1] 同时，Suna 官方 Sandboxes 文档明确指出其沙箱由 **Daytona** 驱动，并通过 `DAYTONA_API_KEY`、`DAYTONA_SERVER_URL`、`DAYTONA_TARGET` 连接到 Daytona API，这意味着 Suna 沙箱层是否能集群化，本质上取决于 **Daytona 执行后端的部署方式**。[2]

进一步看，Daytona 官方开源部署文档给出了 **Docker Compose 单机自托管**方案，[3] 官方配置文档说明 `api_key`、`api_url`、`target` 是标准接入参数，[4] 而 Daytona 官方还单独维护了 **Helm Charts** 仓库，明确标注为 **Kubernetes Helm Charts**，说明 Daytona 也存在面向 Kubernetes 的官方集群部署路径。[5] 因而，若问题是“**Suna 沙箱是否支持集群模式部署**”，更准确的回答应是：

| 层级 | 是否支持集群化 | 说明 |
| --- | --- | --- |
| **Suna 平台层** | **支持** | 官方文档已提供 ECS 与 EKS 生产部署路径，EKS 方案包含 ALB、Node Group、HPA 与 Cluster Autoscaler。[1] |
| **沙箱执行层（Daytona）** | **支持，但实现方式取决于你选用的 Daytona 方案** | 可选 Daytona Cloud、Daytona OSS Docker Compose、Daytona 官方 Helm/Kubernetes 路径。[3] [5] |
| **最终整体形态** | **支持“平台集群 + 沙箱集群”** | 这才是更接近 Manus 的生产形态：控制面集中化，执行面独立扩缩容。 |

## 为什么不能只回答“支持”或“不支持”

Suna 的官方文档已经给出一个很关键的事实：它把沙箱抽象成独立的基础设施能力，而不是简单地在单机 Docker 里临时起容器。Sandboxes 文档写明，沙箱由 Daytona 提供容器隔离、资源管理、生命周期管理和网络访问能力，同时通过 `DaytonaConfig` 传入 `api_key`、`api_url`、`target` 来完成连接。[2] 这意味着 **Suna 是 Daytona 的上层控制面，而非沙箱执行引擎本身**。

因此，从架构角度看，Suna 的“集群能力”天然是**双层的**。第一层是 **Suna API / Web / Redis / DB 等平台服务** 的集群化；第二层是 **Daytona API / Runner / Proxy / Registry / MinIO / 存储卷** 这一整套沙箱执行基础设施的集群化。若你只把 Suna 平台部署到 EKS，而 Daytona 仍然单机运行，那么平台层具备 HA，但沙箱执行层依旧是单点；反过来，如果 Daytona 做成 K8s 集群而 Suna 平台仍然单机，沙箱扩展性会增强，但控制面仍不具备完整 HA。

## 推荐的三种部署路径

从落地复杂度、可维护性与可控性来看，建议把 Suna 沙箱部署方案分成三档。下面的比较可以直接作为选型依据。

| 方案 | 执行层形态 | 平台层形态 | 复杂度 | 适用阶段 | 核心判断 |
| --- | --- | --- | --- | --- | --- |
| **方案一：Daytona Cloud** | 托管沙箱 | Suna 可单机，也可 ECS/EKS | 低 | POC、早期商业验证 | 最快，运维负担最低 |
| **方案二：Daytona 自托管 Docker Compose** | 单机 Daytona | Suna 可单机，也可 ECS/EKS | 中 | 小规模私有化、内测环境 | 可控但仍有执行层单点 |
| **方案三：Daytona 自托管 Kubernetes / Helm** | 集群 Daytona | Suna 建议 EKS/ECS | 高 | 多租户商业化生产环境 | 最接近 Manus 风格 |

## 方案一：接入 Daytona Cloud（最简单）

如果你的目标是先验证 Suna 的产品能力，而不是第一天就自己维护一套复杂的沙箱集群，那么最简单的方式是直接接入 Daytona 的托管 API。Daytona 官方配置文档说明，如果不显式指定 API URL，其默认 API URL 为 `https://app.daytona.io/api`；同时 `target` 用于指定沙箱创建目标区域或目标执行后端。[4] 这意味着你可以把 Daytona 视作**托管执行层**，而让 Suna 只承担应用控制层职责。

### 步骤 1：准备 Daytona 组织、API Key 与 Target

首先在 Daytona 侧准备组织与 API Key，并确认要使用的 target。Daytona 官方配置文档的示例将 target 展示为 `us` / `eu` 这样的区域标识。[4] 在托管模式下，通常可以直接使用组织默认区域或显式指定区域。

### 步骤 2：在 Suna 中启用 Daytona Provider

Suna 本地代码与官方文档均表明，若开启 Daytona provider，至少需要配置以下变量：

```env
ALLOWED_SANDBOX_PROVIDERS=daytona
DAYTONA_API_KEY=your_daytona_api_key
DAYTONA_SERVER_URL=https://app.daytona.io/api
DAYTONA_TARGET=us
```

如果你还希望固定沙箱基础镜像，可以继续设置：

```env
DAYTONA_SNAPSHOT=kortix-sandbox-v<version>
```

其中最关键的是 `ALLOWED_SANDBOX_PROVIDERS=daytona`。Suna 的配置校验逻辑要求，只要 provider 列表中包含 `daytona`，就必须同时提供 `DAYTONA_API_KEY`、`DAYTONA_SERVER_URL` 与 `DAYTONA_TARGET`，否则后端启动时会报错并拒绝以完整配置运行。

### 步骤 3：部署 Suna 平台层

此时 Suna 平台层可以根据业务阶段采用三种方式。若只是验证，可直接按官方 Docker Compose 路径部署；若需要高可用，可按官方 ECS 或 EKS 路径部署。[1]

| 平台层部署方式 | 何时使用 | 说明 |
| --- | --- | --- |
| Docker Compose | 本地验证、测试环境 | 速度最快，但控制面无 HA |
| ECS | 中等规模生产环境 | 维护成本低于 EKS |
| EKS | 企业级、多租户生产 | 与集群化 Daytona 组合最佳 |

### 步骤 4：验证 Suna 到 Daytona 的连通性

在完成配置后，创建一个简单的 Agent 任务，观察是否成功拉起沙箱、打开浏览器、执行文件操作与命令执行。如果 Suna 侧能正常创建项目级沙箱，并返回公开预览地址或远程执行结果，就说明 Suna 到 Daytona 的控制链路已经打通。

### 方案一的优缺点

这种方式的最大优势是**上线快**。你不需要自己维护 Daytona API、Runner、Registry、MinIO、Redis、PostgreSQL 与 Proxy 等沙箱基础设施。[3] 但代价是，执行层的容量、底层节点策略、镜像缓存、网络隔离深度与成本模型主要受 Daytona 托管能力约束，因此更适合前期验证，而不适合把“执行面控制权”作为核心竞争力的团队。

## 方案二：Daytona 自托管 Docker Compose（单机执行层）

如果你已经希望把沙箱执行层收回到自己的基础设施中，但暂时不需要完整的 K8s 扩缩容体系，那么可以先采用 Daytona 官方开源部署文档给出的 Docker Compose 方案。[3]

### 步骤 1：准备单机主机

建议先准备一台独立的 Linux 主机承载 Daytona 执行层。由于 Daytona 的官方 Compose 方案会同时启动 API、Runner、Proxy、SSH Gateway、PostgreSQL、Redis、Registry、MinIO 等多个组件，[3] 因此不建议与核心业务数据库或其他高负载服务混布。对于小规模验证，建议从 **8 vCPU / 16 GB RAM / SSD 存储** 起步，并为镜像层、卷数据与日志留出充足空间。

### 步骤 2：部署 Daytona OSS

Daytona 官方文档给出的核心命令如下：[3]

```bash
git clone https://github.com/daytonaio/daytona.git
cd daytona
docker compose -f docker/docker-compose.yaml up -d
```

部署完成后，文档给出了默认访问端口，包括 Dashboard `http://localhost:3000`、PgAdmin `http://localhost:5050`、Registry UI `http://localhost:5100` 与 MinIO `http://localhost:9001`。[3] 在生产环境中，你应当通过反向代理、域名与 TLS 暴露这些服务，而不是直接使用默认本地端口。

### 步骤 3：完成 DNS 与代理设置

Daytona 官方文档强调，本地开发场景需要处理 `*.proxy.localhost` 的解析，[3] 而在生产环境中，你则需要把 Proxy 域名、控制面域名与对象存储访问域名整理为正式 DNS 记录。若执行层后续要承载浏览器预览、临时 Web 服务或文件预览功能，Proxy 域名设计必须在前期一次性定型。

### 步骤 4：关闭沙箱间互通

Daytona 官方文档特别指出，`INTER_SANDBOX_NETWORK_ENABLED` 控制同一 Runner 上多个沙箱之间能否互通。在官方 Docker Compose 配置中，这个值默认是 `false`，会建立隔离桥接网络并禁用容器间通信；官方还特别提醒，若你在自定义 Runner 部署中运行 Daytona，除非业务确有需要，否则应显式设置为 `false`。[3] 这对多租户 SaaS 非常关键，因为它直接关系到**横向探测、内网扫描与租户逃逸风险**。

建议你把这条配置视为生产环境安全基线：

```env
INTER_SANDBOX_NETWORK_ENABLED=false
```

### 步骤 5：为 Suna 配置自托管 Daytona 地址

当 Daytona 自托管实例稳定运行后，将 Suna 的环境变量切换为指向你的 Daytona API：

```env
ALLOWED_SANDBOX_PROVIDERS=daytona
DAYTONA_API_KEY=<your-self-hosted-daytona-api-key>
DAYTONA_SERVER_URL=https://daytona-api.yourdomain.com
DAYTONA_TARGET=<your-target>
```

这里的 `DAYTONA_SERVER_URL` 不再是托管地址，而是你自建 Daytona API 的统一入口地址。`DAYTONA_TARGET` 则应与该自托管 Daytona 环境中的目标配置保持一致。[4]

### 步骤 6：把 Suna 平台层与 Daytona 执行层分开部署

即使 Daytona 还是单机，Suna 平台层也最好与其分离。推荐的方式是：Suna API / Web 与 Redis、数据库部署在平台集群或平台主机上，而 Daytona 运行在专用“执行层主机”中。这样做的价值在于把**用户请求处理**和**高风险代码执行**放在不同故障域中，便于后续演进到完整的“中心化控制面 + 独立执行面”。

## 方案三：Daytona 自托管 Kubernetes / Helm（集群执行层）

如果你的目标是做**类似 Manus 的多租户商业化 Agent 平台**，真正推荐的路线是：**Suna 平台层走 EKS/ECS，Daytona 执行层走 Kubernetes / Helm**。这不是因为 Docker Compose 不可用，而是因为商业化生产环境最终需要处理**容量池、节点分组、沙箱镜像预热、GPU/CPU 异构资源、跨可用区、Runner 扩缩容、故障域隔离、成本优化**等问题，而这些都更适合在 K8s 中完成。

Daytona 官方 Helm Charts 仓库 README 明确写着 **Kubernetes Helm Charts**，并给出了 Helm 仓库地址 `https://charts.daytona.io`。[5] 这已经足以说明官方存在面向 Kubernetes 的部署路径。

### 步骤 1：准备 Kubernetes 集群

首先准备承载 Daytona 的 Kubernetes 集群。这里有两种主流方式。第一种是与 Suna 平台层共用同一个大集群，通过命名空间与节点池隔离平台服务和 Runner 工作负载；第二种是为 Daytona 单独准备执行层集群，让控制面与执行面物理分离。若以多租户商业化为目标，第二种更稳妥，因为它更容易控制镜像缓存、节点规格、网络策略和故障爆炸半径。

### 步骤 2：接入 Daytona 官方 Helm 仓库

官方 README 给出的基础命令如下：[5]

```bash
helm repo add daytonaio https://charts.daytona.io
helm search repo daytonaio
```

这一步的目的是先拉取官方 chart 索引，再查看当前可用 chart 名称与版本。由于官方 chart 可能随版本演进而变动，**不建议在设计文档中硬编码 chart 名**，而应在实际部署时通过 `helm search repo daytonaio` 固定版本后再落盘到 Helm values 仓库中。

### 步骤 3：规划 Daytona 关键组件的 K8s 落点

从 Daytona OSS 文档给出的 Compose 组件看，至少应考虑以下模块：[3]

| 组件 | 作用 | 集群化建议 |
| --- | --- | --- |
| API | Daytona 控制入口 | 以 Deployment 运行，多副本 |
| Runner | 实际承载沙箱 | 独立节点池，可按负载扩缩容 |
| Proxy | 对外暴露预览与代理流量 | 建议配合 Ingress/LoadBalancer |
| SSH Gateway | 远程接入入口 | 单独暴露或内网接入 |
| PostgreSQL | 元数据存储 | 使用托管数据库优先 |
| Redis | 缓存与队列 | 使用托管 Redis 优先 |
| Registry | 镜像仓库 | 生产环境建议替换为企业仓库 |
| MinIO / 对象存储 | 工件与对象存储 | 生产环境建议替换为云对象存储 |

在真正的生产环境里，不建议把数据库、对象存储、镜像仓库都直接跟随 Helm chart 一并作为“内嵌组件”长期运行。更合理的做法是：**Daytona 的无状态组件跑在 K8s，状态组件尽量外置为托管服务**。这样更便于扩容、备份、权限隔离与跨环境迁移。

### 步骤 4：设置 Runner 安全基线与资源池

Runner 是真正运行用户代码与浏览器操作的地方，因此应作为独立资源池管理。建议至少做四件事。

第一，要为 Runner 节点池设置明确的 CPU / 内存规格与自动扩缩容策略，避免业务 API 服务和高耗能沙箱混跑。第二，要显式关闭 `INTER_SANDBOX_NETWORK_ENABLED`，除非你做的是受控协同作业环境。[3] 第三，要通过节点标签、污点容忍与 RuntimeClass 等手段，把高风险沙箱与普通平台服务彻底隔离。第四，要为 Runner 节点预热常用沙箱镜像，降低冷启动时的镜像拉取延迟。

### 步骤 5：让 Suna 平台层指向集群化 Daytona

当 Daytona 在 K8s 中稳定运行并对外提供统一 API 域名后，Suna 的配置方式与前述两种方案完全一致：

```env
ALLOWED_SANDBOX_PROVIDERS=daytona
DAYTONA_API_KEY=<api-key>
DAYTONA_SERVER_URL=https://daytona-api.yourdomain.com
DAYTONA_TARGET=<cluster-target-or-region>
```

这一点非常重要，因为它说明 Suna 对 Daytona 的接入接口是**稳定抽象**。无论 Daytona 在云端托管、单机自托管，还是在 Kubernetes 集群里以 Helm 方式运行，Suna 侧接入参数都不需要重新设计，只需要替换 API 地址、密钥和 target 即可。[2] [4]

### 步骤 6：把平台层与执行层分别做扩缩容

这一步是商业化稳定性的关键。Suna 官方 EKS 文档已经说明平台层可以部署为 `suna-api` Deployment，并配置 HPA（4–15 Pods）与 Cluster Autoscaler（2–8 Nodes）。[1] 这意味着你可以把 **Suna 平台层的伸缩** 建立在 API 请求、队列长度或内存使用率之上；而 **Daytona Runner 的伸缩** 则建立在活跃沙箱数量、CPU/内存预留、镜像冷启动压力与 WebSocket 会话数之上。二者不应该混成同一个扩缩容目标，否则会出现“平台请求少但沙箱负载高”或“平台请求高但沙箱空闲”的错误调度。

## Suna 平台层的 EKS 集群部署步骤

为了构成完整的“平台集群 + 沙箱集群”形态，Suna 平台层本身也应按官方 EKS 路径部署。官方 Deployment 文档已经给出明确步骤。[1]

### 第一步：准备 EKS 前置条件

官方要求准备 AWS 账号与 EKS 权限、`kubectl`、Pulumi，以及已构建好的容器镜像。[1] 从基础设施治理角度看，这意味着平台层部署是通过 **Pulumi IaC** 驱动，而不是手工逐个 kubectl apply。

### 第二步：进入生产环境基础设施目录

官方命令如下：[1]

```bash
cd infra/environments/prod
```

这表明 Suna 仓库已经内置了生产环境基础设施定义。

### 第三步：执行 Pulumi 部署

官方给出的核心命令是：

```bash
pulumi up
```

根据官方文档，部署后将创建 `suna-eks` 集群、`suna` 命名空间、`suna-api` Deployment、`suna-api` ClusterIP Service、ALB Ingress 以及 `suna-api` 的 HPA。[1]

### 第四步：配置 kubectl

官方给出的命令如下：[1]

```bash
aws eks update-kubeconfig \
  --region us-west-2 \
  --name suna-eks
```

完成后，即可使用 `kubectl get pods -n suna`、`kubectl top pods -n suna` 等命令进行平台层检查。[1]

### 第五步：对接 Daytona 执行层

Suna 平台层部署完成后，真正决定沙箱走向的是平台环境变量。若你希望使用集群化 Daytona，就必须把 `ALLOWED_SANDBOX_PROVIDERS` 设为 `daytona`，并把 `DAYTONA_SERVER_URL` 指向 Daytona 统一 API 域名，而不是保留默认的 `local_docker` 单机 provider。

## 一套建议的生产环境配置模板

下面给出一份适合“平台集群 + Daytona 执行层”的示意配置。它不是一键可用模板，但足以作为生产配置基线的起点。

```env
# Suna sandbox provider
ALLOWED_SANDBOX_PROVIDERS=daytona
DAYTONA_API_KEY=replace_me
DAYTONA_SERVER_URL=https://daytona-api.example.com
DAYTONA_TARGET=us
DAYTONA_SNAPSHOT=kortix-sandbox-v1

# 建议保留的平台安全与可观测性配置
TUNNEL_ENABLED=true
FRONTEND_URL=https://app.example.com
KORTIX_URL=https://api.example.com

# Daytona Runner 安全基线（在 Daytona 侧）
INTER_SANDBOX_NETWORK_ENABLED=false
```

## 资源规格与容量规划建议

Suna 官方 EKS 文档给出了平台层 `suna-api` Pod 的参考资源请求：`500m CPU / 2Gi` request，`1500m CPU / 3Gi` limit；同时 HPA 设置为 4–15 Pods，Node Group 使用 `c7i.2xlarge`。[1] 这些数据可以直接视为**平台控制面**的初始参考，但它们并不等同于沙箱执行层的资源需求。

真正消耗资源的是 Runner 上的沙箱。浏览器型任务、代码编译、网页抓取、多进程执行与文件转换都可能显著拉高 CPU、内存与 I/O。因此，生产环境最好把 Runner 按任务类型拆成多个资源池，例如轻量任务池、浏览器任务池和高内存任务池。这样不仅便于成本优化，也更利于后续引入差异化计费。

| 资源域 | 建议策略 | 原因 |
| --- | --- | --- |
| **Suna 平台层** | 小规格多副本 | 主要承载 API、调度、状态与回调 |
| **Daytona Runner** | 独立节点池，按工作负载扩缩 | 真正承载浏览器与代码执行 |
| **数据库与 Redis** | 托管服务优先 | 降低状态组件运维复杂度 |
| **对象存储与镜像仓库** | 云服务优先 | 提高稳定性与跨环境迁移能力 |

## 多租户生产环境中的注意事项

对于面向商业化的多租户 Agent 平台，最需要警惕的并不是“能不能跑起来”，而是“跑起来之后能否长期稳定与安全运营”。在这方面，有四条原则应该提前固化。

第一，**平台层和执行层必须分离**。Suna API 集群与 Daytona Runner 集群不应混跑在同一故障域中。第二，**禁止沙箱间横向通信**，除非用户明确购买或启用共享作业空间能力；默认应当以 `INTER_SANDBOX_NETWORK_ENABLED=false` 为基线。[3] 第三，**状态组件外置**，避免未来在 Helm 升级、集群迁移与备份恢复时被单体 chart 绑定。第四，**扩缩容指标分层**，不要把 API Pod 和 Runner 节点放在同一个容量策略下。

从架构演进角度看，这也是 Suna 向 Manus 类产品演进的必经之路：把 Suna 保留为统一的**控制面**，而把 Daytona 或其他沙箱基础设施建设为可独立扩缩、可独立计费、可独立安全治理的**执行面**。

## 最终建议

如果你当前处于产品验证阶段，我建议选择 **“Suna 平台层 + Daytona Cloud”**。如果你已经进入私有化或早期商业试点阶段，可以选择 **“Suna 平台层 + 自托管 Daytona Docker Compose”**。如果你的目标是打造真正的多租户商业化 Agent 平台，那么建议直接规划 **“Suna 平台层 EKS + Daytona 执行层 Kubernetes / Helm”** 的双层集群形态。

> 换言之，Suna 并不是原生自带一个“单体式沙箱集群开关”；它的集群能力来自于其上层平台可集群化部署，以及其下层 Daytona 执行面可替换为托管、单机自托管或 Kubernetes 集群化自托管。真正的生产方案，应当把这两层分别治理。[1] [2] [3] [5]

## References

[1]: https://mintlify.wiki/kortix-ai/suna/self-hosting/deployment "Deployment - Kortix"
[2]: https://mintlify.wiki/kortix-ai/suna/concepts/sandboxes "Sandboxes - Kortix"
[3]: https://www.daytona.io/docs/en/oss-deployment/ "Open Source Deployment | Daytona"
[4]: https://www.daytona.io/docs/en/configuration/ "Environment Configuration | Daytona"
[5]: https://github.com/daytonaio/helm-charts "GitHub - daytonaio/helm-charts: Daytona official Helm charts"
