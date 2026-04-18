# Daytona 沙箱集群部署详细步骤

作者：**Manus AI**  
日期：2026-04-19

## 文档目的

本文档聚焦回答一个更具体的问题：如果要把 **Daytona 作为 Suna 的沙箱执行层**，并以 **Kubernetes 集群** 方式自托管部署，应该如何一步步落地。这里不再停留在“支持或不支持”的判断，而是给出一套可执行的 **生产化实施路径**，并补充 Helm Chart、Ingress、TLS、外部依赖、Runner、Region 与 Suna 接入配置之间的关系。[1] [2] [3] [4]

## 一、先说结论：Daytona 的集群化有两条主线

Daytona 官方 Helm Charts 仓库实际上提供了两类不同角色的 Kubernetes 部署能力，这一点非常关键。[1] [2] 一类是 **完整 Daytona 平台**，对应主 Chart `daytona`，用于在 Kubernetes 上部署 API、Proxy、SSH Gateway 以及可选的 PostgreSQL、Redis、Harbor、MinIO 等组件。[2] 另一类是 **自定义 Region**，对应 `daytona-region` Chart，用于把 Proxy 与可选的 Snapshot Manager 部署到你的网络或云环境中，并通过注册流程与 Daytona API 建立关联。[3]

这意味着在生产设计上，你至少有两种可选形态：

| 形态 | 适用场景 | 部署内容 | 与 Suna 的关系 |
| --- | --- | --- | --- |
| **方案 A：完整自托管 Daytona 平台** | 希望把沙箱控制面与执行面都收回自有基础设施 | `daytona` 主 Chart，必要时配外部 DB/Redis/S3/Registry | Suna 直接指向你的 Daytona API |
| **方案 B：Daytona Cloud + 自定义 Region** | 希望保留 Daytona 托管控制面，但把代理、快照流量或数据驻留落在自己网络中 | `daytona-region` Chart | Suna 继续指向 Daytona API，执行流量可落入自有 Region |

如果你的目标是做 **类似 Manus 的商业化多租户 Agent 平台**，最推荐的是 **方案 A**，因为它更利于你掌控执行层扩缩容、安全基线、存储、审计与成本结构。

## 二、完整自托管 Daytona 平台的目标架构

Daytona 主 Chart 的 README 已经明确说明，它会在 Kubernetes 中部署完整 Daytona 平台，并允许你决定数据库、Redis、S3 与身份提供者是使用内置子 Chart，还是改接外部服务。[2] 这是生产设计的核心分水岭。

> Daytona 主 Helm Chart 的说明明确指出，虽然架构图展示 PostgreSQL、Redis、S3 Storage 与 IdP 作为外部组件，但 Helm Chart 也可以通过内置 subcharts 部署 PostgreSQL、Redis 与 Dex；同时，MinIO 默认关闭，你也可以通过 `values.yaml` 改接外部服务。[2]

从生产视角看，我建议将架构拆成三层：

| 层级 | 建议部署内容 | 建议策略 |
| --- | --- | --- |
| **入口层** | Ingress Controller、DNS、TLS 证书 | 独立治理，统一域名与证书 |
| **Daytona 无状态服务层** | API、Proxy、SSH Gateway | Kubernetes Deployment，多副本 + HPA |
| **Daytona 状态依赖层** | PostgreSQL、Redis、S3、Registry | 优先外部托管，避免长期依赖内嵌子 Chart |

## 三、部署前准备

### 1. Kubernetes 与基础工具

根据官方 README，Daytona Helm Chart 需要 **Kubernetes 1.19+** 与 **Helm 3.2.0+**。[2] 在真正部署之前，建议你至少准备以下基础条件：

| 项目 | 建议 |
| --- | --- |
| Kubernetes 版本 | 1.24+ 更稳妥，至少满足官方 1.19+ 要求 [2] |
| Helm | 3.2.0+ [2] |
| Ingress | 建议 NGINX Ingress 或云厂商 ALB Ingress |
| 证书 | cert-manager + DNS 验证，或预置 wildcard 证书 |
| 外部数据库 | 托管 PostgreSQL 优先 |
| 外部缓存 | 托管 Redis 优先 |
| 对象存储 | S3 或兼容存储优先 |
| 镜像仓库 | 企业级 Registry/Harbor 优先 |

### 2. 域名规划

Daytona 主 Chart 中最核心的入口配置是 `baseDomain`，默认值是 `daytona.example.com`。[4] 这个域名并不只是 API 域名，它还关系到 API、Dashboard、Proxy 以及 wildcard 子域名路由。因为 Proxy 会为每个沙箱使用唯一子域名，所以你在证书与 DNS 设计时，必须同时覆盖 **主域名** 与 **通配子域名**。[2] [4]

一个比较清晰的生产规划如下：

| 功能 | 建议域名 |
| --- | --- |
| Daytona API / Dashboard | `daytona.example.com` |
| 沙箱代理入口 | `*.daytona.example.com` |
| SSH Gateway | `ssh.daytona.example.com` |
| Snapshot Manager（如启用） | `snapshots.daytona.example.com` |

### 3. 决定内嵌依赖还是外部依赖

Daytona 主 Chart 允许 PostgreSQL、Redis 等以内嵌子 Chart 的方式一并安装，但这更适合测试环境。[2] 对于多租户生产系统，建议采用下表策略：

| 组件 | 测试环境 | 生产环境建议 |
| --- | --- | --- |
| PostgreSQL | 可用子 Chart | 外部托管 PostgreSQL |
| Redis | 可用子 Chart | 外部托管 Redis |
| S3 / MinIO | 可本地 MinIO | 外部 S3 / 对象存储 |
| Harbor | 可内嵌 | 企业镜像仓库或专用 Harbor |
| Dex / OIDC | 可内嵌 | 接入企业 IdP 或稳定 OIDC |

## 四、安装 Daytona Helm Chart 的详细步骤

### 步骤 1：添加官方 Helm 仓库并确认 Chart

Daytona 官方 Helm Charts 仓库 README 给出了标准命令：[1]

```bash
helm repo add daytonaio https://charts.daytona.io
helm repo update
helm search repo daytonaio
```

如果你本地已经克隆了 Helm 仓库，也可以直接使用本地路径安装；但生产环境最好固定 chart 版本，并将 values 文件纳入 Git 管理。

### 步骤 2：创建专用命名空间

建议将 Daytona 与 Suna 控制面分在不同命名空间，甚至不同集群。一个简单示例如下：

```bash
kubectl create namespace daytona
```

如果你还计划将 Suna 放到另一命名空间，可以分别使用 `suna-platform` 与 `daytona-exec` 两个命名空间，以便隔离权限、网络策略和资源配额。

### 步骤 3：准备生产 values 文件

Daytona 主 Chart 的 `values.yaml` 暴露了非常多生产关键项，其中最先要改的不是副本数，而是 **域名、加密密钥、外部依赖与 TLS**。[4]

下面给出一份适合作为生产起点的示例文件 `values-prod.yaml`。该示例体现的是“**无状态组件跑在 K8s，状态组件尽量外置**”的思路。

```yaml
global:
  storageClass: gp3

baseDomain: "daytona.example.com"

services:
  api:
    replicaCount: 2
    autoscaling:
      enabled: true
      minReplicas: 2
      maxReplicas: 6
      targetCPUUtilizationPercentage: 70
    ingress:
      enabled: true
      className: nginx
      tls: true
      selfSigned: false
      tlsSecretName: daytona-wildcard-tls
    resources:
      requests:
        cpu: 250m
        memory: 512Mi
      limits:
        memory: 1Gi
    env:
      ENVIRONMENT: "production"
      ENCRYPTION_KEY: "REPLACE_WITH_32_CHAR_SECRET_VALUE"
      ENCRYPTION_SALT: "REPLACE_WITH_RANDOM_SALT"
      DEFAULT_SNAPSHOT_IMAGE_NAME: "daytonaio/sandbox:0.5.1-slim"
      DEFAULT_SNAPSHOT_NAME: "default-snapshot"
      RUNNER_MANAGER_API_KEY: "replace-runner-manager-api-key"

  proxy:
    replicaCount: 2
    autoscaling:
      enabled: true
      minReplicas: 2
      maxReplicas: 8
      targetCPUUtilizationPercentage: 70
    ingress:
      enabled: true
      className: nginx
      tls: true
      selfSigned: false
      tlsSecretName: daytona-wildcard-tls
    resources:
      requests:
        cpu: 250m
        memory: 256Mi
      limits:
        memory: 512Mi

  sshGateway:
    enabled: true
    service:
      type: LoadBalancer
      port: 2222

postgresql:
  enabled: false

redis:
  enabled: false

minio:
  enabled: false

externalDatabase:
  host: "postgres.example.internal"
  port: 5432
  name: "daytona"
  user: "daytona"
  existingSecret: "daytona-db-secret"
  enableTLS: true
  allowSelfSignedCert: false

externalRedis:
  host: "redis.example.internal"
  port: 6379
  tls: true
  existingSecret: "daytona-redis-secret"
```

这份 values 文件与官方 Chart 暴露的配置项是一致的：`baseDomain`、`services.api.ingress`、`services.proxy.ingress`、`services.api.autoscaling`、`externalDatabase`、`externalRedis` 等都在 README 和默认值中有明确说明。[2] [4]

### 步骤 4：准备外部 Secret

如果你关闭了内置 PostgreSQL 和 Redis，就要把外部连接秘密以 Secret 的形式交给集群。因为 Chart 对 `existingSecret` 有约定，所以要确保 Secret 键名与 Chart 预期一致。README 中已说明数据库密码 Secret 需要键 `database-password`，Redis 密码 Secret 需要键 `redis-password`。[2]

示例如下：

```bash
kubectl -n daytona create secret generic daytona-db-secret \
  --from-literal=database-password='replace_me'

kubectl -n daytona create secret generic daytona-redis-secret \
  --from-literal=redis-password='replace_me'
```

如果你的 OIDC、SMTP、S3 或外部 Registry 也需要凭据，建议不要把敏感值直接写进 values 文件，而是通过 `extraEnv` 配合 `secretKeyRef` 注入；官方 values 也为 `extraEnv` 预留了这种扩展方式。[4]

### 步骤 5：准备 TLS 证书

Daytona 的主域名和 Proxy wildcard 域名通常共用同一个 TLS Secret。官方 values 文件明确写到，证书必须覆盖 **基础域名** 与 **其通配子域名**，因为 API 与 Proxy ingress 会共享证书覆盖 `daytona.example.com` 与 `*.daytona.example.com`。[4]

如果你使用 cert-manager，可以提前申请 `daytona-wildcard-tls`；如果暂时没有正式证书，也可以把 `selfSigned` 打开，但这只适合测试环境。[2] [4]

### 步骤 6：安装 Daytona Chart

完成上述准备后，即可安装：

```bash
helm upgrade --install daytona daytonaio/daytona \
  -n daytona \
  -f values-prod.yaml
```

如果你是基于本地仓库测试，也可以使用：

```bash
helm upgrade --install daytona ./charts/daytona \
  -n daytona \
  -f values-prod.yaml
```

官方 README 的最小命令是 `helm install daytona ./charts/daytona`，而这里使用 `upgrade --install` 是为了兼容后续迭代发布。[2]

### 步骤 7：检查部署状态

安装完成后，建议按下面顺序验证：

```bash
kubectl get pods -n daytona
kubectl get svc -n daytona
kubectl get ingress -n daytona
kubectl get hpa -n daytona
kubectl logs -n daytona deploy/daytona-api
kubectl logs -n daytona deploy/daytona-proxy
```

你应重点检查以下事项：

| 检查项 | 通过标准 |
| --- | --- |
| API Pod | Ready 且无持续重启 |
| Proxy Pod | Ready 且 ingress 已下发 |
| SSH Gateway | Service 有外部地址或可达入口 |
| Ingress | 主域名和 wildcard 域名规则正确 |
| HPA | 已创建并读取到 requests |
| 数据库连接 | API 日志无 DB 连接失败 |
| Redis 连接 | API 日志无 Redis 认证失败 |

## 五、Runner 与实际沙箱承载节点怎么处理

这是很多人第一次部署 Daytona 时最容易忽略的地方。**Kubernetes 里部署 Daytona API/Proxy，不等于沙箱已经有了承载能力。** 真正运行用户代码、浏览器和任务进程的，是 Runner 层。

在当前官方材料中，Runner 的另一条落地路径是通过官方安装脚本在主机上安装为 systemd 服务。官方 `runner/README.md` 给出的方式是：

```bash
curl -sSL https://download.daytona.io/install.sh | sudo bash
```

安装脚本会提示输入 Daytona API URL、Admin API Key、CPU/内存/磁盘配额、Runner 域名、Runner API URL，以及可选的 proxy URL、region、capacity 和 runner API key；随后自动完成 Docker 安装、Runner 注册、systemd 服务创建与启动。[5]

这意味着从生产设计角度，你至少可以采用两种 Runner 承载方式：

| Runner 承载方式 | 特点 | 建议用途 |
| --- | --- | --- |
| **独立 VM / 裸机 + systemd Runner** | 与 K8s 控制面解耦，适合高风险隔离 | 生产优先推荐 |
| **与平台共集群的容器化方式** | 管理一致，但隔离与性能调优更复杂 | 测试或过渡期 |

对于 Suna 这类多租户 Agent 平台，更建议使用 **独立 Runner 节点池**，不要让 API 控制面与高风险代码执行混跑在同一节点上。

### Runner 节点接入建议流程

#### 1. 准备 Runner 主机

建议为 Runner 使用专门的 VM 或裸机，满足以下最低基线：

| 任务类型 | 最低建议 |
| --- | --- |
| 轻量代码执行 | 4 vCPU / 8 GB RAM |
| 浏览器自动化 | 8 vCPU / 16 GB RAM |
| 高并发混合任务 | 16+ vCPU / 32+ GB RAM |

#### 2. 运行安装脚本并注册

在每个 Runner 节点执行：

```bash
curl -sSL https://download.daytona.io/install.sh | sudo bash
```

根据官方说明，安装时要填写 API URL、管理密钥、资源配额、域名等信息。[5] 这些参数实际上决定了 Runner 在 Daytona 控制面中的注册信息与可承载能力。

#### 3. 把 Runner 分组

虽然官方 README 没有直接给出“节点池分组”这一术语，但从运维实践上，建议你人为划分 Runner 资源池，例如：浏览器任务池、通用代码池、高内存池。然后在 Suna 侧通过不同 `DAYTONA_TARGET`、调度策略或后续扩展标签，实现不同工作负载的路由。

#### 4. 监控 Runner 服务

官方 README 给出了 systemd 运维命令：[5]

```bash
sudo systemctl status daytona-runner
sudo tail -f /var/log/daytona-runner.log
sudo systemctl stop daytona-runner
```

在生产环境中，应将这些日志与主机指标接入统一监控系统，否则排查 Runner 失联、镜像拉取失败、磁盘打满与端口冲突会非常困难。

## 六、关于 Proxy、Ingress 与 Wildcard 证书的重点说明

Daytona 的 Proxy 不是一个普通的固定域名反向代理。官方 README 和 values 都明确说明，Proxy ingress 会自动包含 wildcard host，用于支持“每个 sandbox 一个唯一子域名”的路由模式。[2] [4] 这意味着：

1. 你不能只申请 `daytona.example.com` 的单域名证书；
2. 你必须同时让 `*.daytona.example.com` 正确解析到 Ingress；
3. 你的 TLS Secret 必须覆盖主域名和通配子域名；
4. 反向代理层的超时与 WebSocket 设置必须按长连接场景调整。

如果你使用的是 NGINX Ingress，建议在 Proxy ingress annotations 中补充更长的超时。例如官方 values 中已经给出了 `proxy-read-timeout` 与 `proxy-send-timeout` 的注释示例。[4]

## 七、可选扩展：使用 `daytona-region` Chart 建自定义 Region

如果你并不打算完全自托管 Daytona 平台，而是准备使用 Daytona API，同时把 Proxy 与快照存储能力部署在自己网络里，那么可以使用 `daytona-region` Chart。[3]

其 README 已明确说明，自定义 Region 的工作方式分四步：安装时通过 pre-install hook 使用 `daytonaApiUrl` 和 `daytonaApiKey` 向 Daytona API 注册 Region；API 返回 `proxyApiKey` 等凭据并写入 K8s Secret；随后 Proxy 利用这些凭据与 Daytona API 通信；如启用 Snapshot Manager，则快照落入你自己的 S3 存储。[3]

### Region 模式 values 示例

```yaml
regionName: "ap-sg-private-region"
proxyUrl: "https://proxy.daytona.example.com"
snapshotManagerUrl: "https://snapshots.daytona.example.com"

daytonaApiUrl: "https://api.daytona.io/api"
daytonaApiKey: "dtn_replace_me"

registration:
  enabled: true

services:
  proxy:
    replicaCount: 2
    autoscaling:
      enabled: true
      minReplicas: 2
      maxReplicas: 6
      targetCPUUtilizationPercentage: 70
    ingress:
      enabled: true
      className: nginx
      tls: true
      selfSigned: false

  snapshotManager:
    enabled: true
    ingress:
      enabled: true
      hostname: "snapshots.daytona.example.com"
      tls: true
    storage:
      s3:
        region: "ap-southeast-1"
        bucket: "daytona-snapshots"
        encrypt: true
        secure: true
```

安装命令如下：

```bash
helm upgrade --install my-region daytonaio/daytona-region \
  -n daytona \
  -f region-values.yaml
```

README 也特别指出，若启用 Snapshot Manager，S3 建议优先通过 **IRSA** 等云原生身份方式接入，而不是硬编码 AK/SK。[3]

## 八、Suna 如何接入你部署好的 Daytona 集群

无论你采用完整自托管 Daytona 平台，还是托管 Daytona + 自定义 Region，Suna 侧的接入方式都是一致的。Suna 本地源码已明确规定，只要 `ALLOWED_SANDBOX_PROVIDERS` 中包含 `daytona`，就必须提供 `DAYTONA_API_KEY`、`DAYTONA_SERVER_URL` 与 `DAYTONA_TARGET`；否则配置校验将报错。[6]

一个典型示例如下：

```env
ALLOWED_SANDBOX_PROVIDERS=daytona
DAYTONA_API_KEY=replace_me
DAYTONA_SERVER_URL=https://daytona.example.com
DAYTONA_TARGET=ap-sg-private-region
```

如果你部署的是完整自托管 Daytona 平台，`DAYTONA_SERVER_URL` 就是你自己的 Daytona API 地址；如果你使用 Daytona Cloud + 自定义 Region，`DAYTONA_SERVER_URL` 依然可以是 Daytona API，而 `DAYTONA_TARGET` 则指向你注册的 Region。[3] [6]

## 九、生产环境中的安全与容量建议

### 1. 默认关闭沙箱间网络互通

Daytona 官方开源部署文档明确指出，`INTER_SANDBOX_NETWORK_ENABLED` 用于控制同一 Runner 上多个沙箱是否可以互相通信，且在 Docker Compose 默认配置中它是 `false`；官方还明确建议，在自定义 Runner 部署中也应显式关闭，除非你确实需要沙箱互联。[7]

> 对于多租户 Agent SaaS，这一项应被视为默认安全基线，而不是可有可无的优化项。[7]

### 2. API 与 Runner 分层扩缩容

API、Proxy 与 SSH Gateway 是无状态控制组件，适合通过 HPA 管理；Runner 则应根据活跃沙箱数、CPU 占用、浏览器实例密度与镜像拉取耗时单独扩缩。[2] [5] 这两层不要放进同一个扩缩容策略中。

### 3. 避免长期依赖内嵌状态子 Chart

虽然 Daytona 主 Chart 支持 PostgreSQL、Redis、Harbor、MinIO 子 Chart，但长周期生产更推荐外部托管依赖，以降低升级风险、提升备份恢复能力，并保持状态与无状态服务的治理边界清晰。[2]

### 4. 建立镜像预热与 Runner 池化策略

因为真实任务会频繁拉起浏览器、CLI、代码运行环境，若每次都冷拉基础镜像，会显著拉高启动时延。因此建议对常用 snapshot image 做预热，并让 Runner 节点按工作负载类型划分资源池。

## 十、推荐的落地顺序

如果你准备把 Daytona 作为 Suna 的正式执行层，我建议按以下顺序实施：

| 阶段 | 目标 | 推荐动作 |
| --- | --- | --- |
| **P0** | 验证连通性 | 单节点或测试 K8s 安装 Daytona 主 Chart，打通 Suna 接入 |
| **P1** | 固化入口与 TLS | 规划 `baseDomain`、wildcard DNS、cert-manager 与 Ingress |
| **P2** | 外置状态组件 | 替换内嵌 PostgreSQL / Redis / MinIO |
| **P3** | 建立 Runner 资源池 | 按浏览器、通用、重负载任务分组部署 Runner |
| **P4** | 正式接入 Suna | 配置 `ALLOWED_SANDBOX_PROVIDERS=daytona` 与 Daytona API 参数 |
| **P5** | 观测与容量治理 | 建立 Runner 指标、沙箱密度、镜像命中率与失败率监控 |

## 十一、最终建议

对于你的目标——围绕 Suna 构建类似 Manus 的商业化多租户 Agent 平台——**最合理的 Daytona 路线并不是停留在 Docker Compose 单机，而是尽快进入“Kubernetes 控制面 + 独立 Runner 资源池”的形态**。主 Helm Chart 负责把 Daytona API、Proxy、SSH Gateway 和必要基础服务放进集群；Runner 则建议作为独立高风险执行节点池管理；若你暂时仍依赖 Daytona 官方 API，也可以先用 `daytona-region` Chart 将 Proxy 与快照层前移到自己的网络中。[2] [3] [5]

从 Suna 的视角看，只要 `DAYTONA_SERVER_URL` 与 `DAYTONA_TARGET` 配置正确，它并不关心 Daytona 背后到底是单机、完整自托管 K8s 还是自定义 Region；这也正是它适合作为上层控制面的原因。[6]

## References

[1]: https://github.com/daytonaio/helm-charts "daytonaio/helm-charts"
[2]: https://github.com/daytonaio/helm-charts/blob/main/charts/daytona/README.md "charts/daytona/README.md"
[3]: https://github.com/daytonaio/helm-charts/blob/main/charts/daytona-region/README.md "charts/daytona-region/README.md"
[4]: https://github.com/daytonaio/helm-charts/blob/main/charts/daytona/values.yaml "charts/daytona/values.yaml"
[5]: https://github.com/daytonaio/helm-charts/blob/main/runner/README.md "runner/README.md"
[6]: https://github.com/kortix-ai/suna/blob/main/apps/api/src/config.ts "apps/api/src/config.ts"
[7]: https://www.daytona.io/docs/en/oss-deployment/ "Open Source Deployment | Daytona"
