# Kubernetes 完全指南：原理、架构、实战与命令手册

> 从容器编排原理到生产落地的系统教程，附 200+ 条常用命令速查
>
> 适用版本：Kubernetes 1.28 及以上（示例在 1.28 / 1.30 / 1.32 验证）
>
> 文档版本：v1.0 ｜ 编写日期：2026-11

# 第1章 认识 Kubernetes

## 1.1 从容器到容器编排

容器技术解决的是**"一个应用如何被一致地打包和隔离运行"**：镜像把代码、依赖、运行时固化成一个不可变产物，容器运行时（containerd、CRI-O）负责把它跑起来，namespace 与 cgroup 负责隔离与限额。这让"在我机器上是好的"这个问题基本消失。

但真实的生产环境不是一台机器，而是几十上百台机器，上面跑着几十个互相调用的服务。容器本身没有回答下面这些问题：

- 这个容器应该跑在哪台机器上？那台机器还剩多少 CPU 和内存？
- 容器挂了谁来拉起？所在机器宕机了谁来重新安排？
- 流量涨了怎么从 3 个副本扩到 30 个？跌了怎么缩回去？
- 新版本怎么做到不停机发布？出问题怎么在 30 秒内回滚？
- A 服务怎么找到 B 服务？B 扩容后 IP 变了怎么办？
- 配置和密钥怎么注入？敏感信息怎么不落盘到镜像里？
- 有状态服务（数据库、消息队列）的数据怎么跟着 Pod 走？

**Kubernetes（简称 K8s，"K"和"s"之间有 8 个字母）就是这些问题的答案。** 它把"机器"抽象成资源池，把"运维动作"抽象成声明式 API，把"人的经验"固化成控制器逻辑。

> 一句话对比：Docker 解决**单机容器的生命周期**，Kubernetes 解决**集群范围的期望状态**。

## 1.2 Kubernetes 的定义与能力边界

Kubernetes 是一个开源的**容器编排平台**，官方定义是"用于自动部署、扩缩和管理容器化应用的生产级平台"。更准确的技术定义是：

> 一个**以 etcd 为唯一状态存储、以声明式 API 为交互界面、以控制器循环为执行引擎**的分布式资源管理系统。

它的核心能力可以归纳为 8 项：

| 能力 | 说明 | 对应机制 |
|---|---|---|
| 服务发现与负载均衡 | Pod 有集群内虚拟 IP 与 DNS 名 | Service、CoreDNS |
| 自动装箱调度 | 按资源需求与约束选择节点 | kube-scheduler |
| 自愈 | 重启失败容器、替换不健康 Pod、节点失联后迁移 | 控制器、kubelet、探针 |
| 水平扩缩 | 按 CPU/内存/自定义指标调整副本数 | HPA、KEDA |
| 滚动发布与回滚 | 分批替换 Pod，可暂停、可回退 | Deployment、Rollout |
| 配置与密钥管理 | 配置与镜像解耦 | ConfigMap、Secret |
| 存储编排 | 自动挂载本地盘、云盘、NFS、对象存储 | PV/PVC、StorageClass、CSI |
| 批量与定时任务 | 一次性、并行、定时任务 | Job、CronJob |

同样重要的是**能力边界**——Kubernetes 不负责这些事：

- **不负责应用的架构**：它不会让你的单体变微服务，也不会帮你选数据库。
- **不提供 CI/CD 流水线**：它只提供部署的"落点"，构建与发布编排需要 Jenkins、GitLab CI、Argo CD 等。
- **不内置可观测性后端**：它暴露指标和日志接口，存储与展示要接 Prometheus、Loki、Jaeger。
- **不保证"高可用"本身**：它提供机制，是否高可用取决于你的副本数、亲和性、PDB、多可用区设计。
- **不是 Serverless 平台**：Pod 仍是常驻进程模型（Knative 等在此之上做了补充）。

## 1.3 Kubernetes 解决了哪四类问题

把纷繁的功能收拢，K8s 对业务的价值集中在四条主线上：

**第一，调度（Scheduling）。** 你只声明"我要 3 个副本，每个要 1 核 2G"，调度器负责在几十台节点中挑选满足条件的节点，并尽量把负载摊平。它还支持把 Pod 打散到不同可用区、不同机架，或者反过来要求"必须和缓存 Pod 在同一台机器上"。

**第二，自愈（Self-healing）。** 容器进程崩溃，kubelet 按重启策略重启；Pod 被删除，ReplicaSet 立刻补一个；节点宕机，Node Controller 打上污点，控制器在别的节点重建 Pod；探针失败，立刻摘流量并重启。

**第三，弹性（Elasticity）。** HPA 根据指标调整副本数，Cluster Autoscaler 根据 Pending Pod 增加节点，VPA 调整单 Pod 的资源配额。让资源成本跟着业务曲线走，而不是按峰值买机器。

**第四，发布（Release）。** Deployment 的滚动更新保证"先起新的、再停旧的"，maxSurge / maxUnavailable 控制节奏；readinessProbe 保证"没准备好的不接流量"；`kubectl rollout undo` 一条命令回到上一个版本；配合 Service 的标签选择器和 Ingress 注解还能做金丝雀与蓝绿。

这四件事过去要靠运维手册 + Shell 脚本 + 人盯屏幕完成，现在变成了 YAML 里的几行字段。

## 1.4 设计哲学：为什么是声明式

这是理解 K8s 的分水岭。**命令式**关心"做什么动作"，**声明式**关心"最终是什么样"。

| 维度 | 命令式（docker run / 脚本） | 声明式（kubectl apply） |
|---|---|---|
| 交互内容 | 执行一条条指令 | 提交一份期望状态 |
| 幂等性 | 差，重复执行会出错 | 好，重复执行结果一致 |
| 故障恢复 | 需要额外的重试逻辑 | 控制器自动收敛 |
| 多人协作 | 状态散落在各人终端 | 状态收敛到集群与 Git |
| 审计与回滚 | 靠日志和记忆 | 靠版本历史（GitOps） |

声明式带来三个直接后果：

1. **控制器循环（Reconciliation Loop）**：每个控制器都在执行 `for { 观察实际状态 → 与期望状态比较 → 执行动作缩小差异 }`。这就是 K8s 的"心脏"。
2. **面向终态（Level-triggered）**：系统关心的是"当前整体状态"，而不是"某条事件有没有被处理"。所以控制器天然幂等、天然可重入。
3. **API 即契约**：所有组件都通过 apiserver 读写同一份对象模型，没有私有通道。这也让扩展变得自然——你新增的 CRD 和内置资源在 API 层完全平等。

## 1.5 发行版与形态：先选一个能跑起来的环境

Kubernetes 是"上游项目"，实际使用中几乎不会自己从源码编译，而是选择一种发行形态：

| 形态 | 代表产品 | 适用场景 | 运维成本 |
|---|---|---|---|
| 本地学习 | kind、minikube、k3d、Docker Desktop | 开发、CI、教程实验 | 极低 |
| 轻量生产 | k3s、k0s、MicroK8s | 边缘、IoT、小规模、资源受限 | 低 |
| 自建通用 | kubeadm + containerd + Calico | 私有云、IDC、可控性要求高 | 高 |
| 云托管 | ACK（阿里云）、TKE（腾讯云）、EKS、GKE、AKS | 绝大多数生产业务 | 低（控制面由云厂商托管） |
| 企业发行版 | Rancher RKE2、OpenShift、SUSE | 需要商业支持与合规 | 中 |

**选择建议：**

- 只想学习和做实验 → **kind**（最快，几分钟起一个多节点集群，且能模拟多节点拓扑）。
- 需要图形界面和常用插件 → **minikube**（`minikube dashboard`、`minikube addons`）。
- 团队真实业务 → **云托管**，把控制面的运维交给云厂商，人力投在业务上。
- 私有化/离线部署 → **kubeadm 或 k3s**，本文第 4 章给出手把手步骤。

## 1.6 学习路径建议

K8s 的知识面很宽，容易陷入"学不完"的焦虑。按下面的顺序推进，每一阶段都以"能独立完成一件事"为验收标准：

| 阶段 | 目标 | 核心内容 | 验收标准 |
|---|---|---|---|
| L1 入门（1 周） | 能跑起来 | 集群搭建、kubectl、Pod、Deployment、Service | 独立部署一个 Web 应用并从集群外访问 |
| L2 会用（2 周） | 能配得对 | ConfigMap/Secret、探针、资源配额、存储、Ingress | 部署一个有状态服务并完成数据持久化 |
| L3 会调（1 个月） | 能排故障 | 调度约束、网络模型、事件与日志、常见 CrashLoop 排查 | 独立定位一次 Pod 启动失败与一次 502 |
| L4 会设计（3 个月） | 能上生产 | RBAC、NetworkPolicy、HPA、PDB、多环境管理、Helm/Kustomize | 设计一套满足滚动发布与回滚的部署方案 |
| L5 精通 | 能改源码 | 控制器原理、Operator 开发、CRI/CNI/CSI、调度框架 | 用 Kubebuilder 写一个有意义的 Operator |

> 读本文的建议：第 2、3 章要慢读，它们是所有命令行的解释器；第 5 章的实验要动手敲，只看不练等于没学；第 6 章当手册用，不必通读。

# 第2章 架构与工作原理

## 2.1 集群全景

一个 Kubernetes 集群由**控制平面（Control Plane）**和**工作节点（Worker Node）**两部分组成。控制平面做决策，工作节点跑负载，etcd 存状态，apiserver 是唯一的通信枢纽。

```text
┌──────────────────────────── Control Plane ────────────────────────────┐
│                                                                        │
│   kubectl / Dashboard / CI ──HTTPS──┐                                  │
│                                     ▼                                  │
│                          ┌──────────────────┐      ┌───────────────┐   │
│                          │  kube-apiserver  │◄────►│     etcd      │   │
│                          │  认证/授权/准入   │      │  (Raft 集群)  │   │
│                          └────────┬─────────┘      └───────────────┘   │
│                                   │ watch/list                         │
│              ┌────────────────────┼────────────────────┐               │
│              ▼                    ▼                    ▼               │
│      ┌───────────────┐   ┌──────────────────┐  ┌─────────────────┐    │
│      │kube-scheduler │   │kube-controller-  │  │cloud-controller-│    │
│      │  调度 Pod     │   │manager（40+ 控制器）│ │manager（可选）  │    │
│      └───────────────┘   └──────────────────┘  └─────────────────┘    │
└────────────────────────────────┬───────────────────────────────────────┘
                                 │ kubelet 主动 watch（长连接，443）
        ┌────────────────────────┼────────────────────────┐
        ▼                        ▼                        ▼
┌───────────────┐        ┌───────────────┐        ┌───────────────┐
│   Node 1      │        │   Node 2      │        │   Node 3      │
│ ┌───────────┐ │        │ ┌───────────┐ │        │ ┌───────────┐ │
│ │  kubelet  │ │        │ │  kubelet  │ │        │ │  kubelet  │ │
│ ├───────────┤ │        │ ├───────────┤ │        │ ├───────────┤ │
│ │ kube-proxy│ │        │ │ kube-proxy│ │        │ │ kube-proxy│ │
│ ├───────────┤ │        │ ├───────────┤ │        │ ├───────────┤ │
│ │ containerd│ │        │ │ containerd│ │        │ │ containerd│ │
│ ├───────────┤ │        │ ├───────────┤ │        │ ├───────────┤ │
│ │  CNI 插件  │ │        │ │  CNI 插件  │ │        │ │  CNI 插件  │ │
│ └───────────┘ │        │ └───────────┘ │        │ └───────────┘ │
│   Pod  Pod    │        │   Pod  Pod    │        │   Pod  Pod    │
└───────────────┘        └───────────────┘        └───────────────┘
```

关键点：**所有组件都只与 apiserver 通信，组件之间不直接对话**。kubelet 不连 scheduler，scheduler 不连 etcd，controller-manager 也不直接操作节点。这种"星型拓扑 + 一切经 API"的设计，是 K8s 可扩展、可观测、可审计的根本原因。

## 2.2 控制平面组件

### 2.2.1 kube-apiserver

集群的**唯一入口**和**唯一的状态读写网关**。它本身是**无状态**的，可以水平扩展多个副本，通过负载均衡对外提供 6443 端口。

它的请求处理链分四段，理解这个链条就理解了大部分"权限问题"：

1. **认证（Authentication）**：你是谁？支持 X.509 客户端证书、ServiceAccount Token（JWT）、OIDC、Webhook、静态 Token。任一方式通过即进入下一步。
2. **授权（Authorization）**：你能做什么？默认 RBAC（也可用 ABAC、Webhook、Node 授权）。判定的是"动词 + 资源 + 范围"，例如 `get pods in namespace dev`。
3. **准入（Admission）**：要不要改写/拒绝？分 Mutating（如注入 sidecar、填充默认值）和 Validating（如校验资源配额、Pod Security）两类，**只有写请求才会走准入**，读请求不走。
4. **持久化**：校验通过后写入 etcd，并递增 `resourceVersion`，随后通过 watch 通知所有关注者。

apiserver 还负责：**API 版本转换**（同一对象在 `v1` 和 `v1beta1` 间转换）、**watch 流**、**字段选择器与标签选择器**的服务端过滤、**限流**（`--max-requests-inflight`）、以及 **Aggregation Layer**（把 `/apis/metrics.k8s.io` 这类请求代理到 metrics-server 等扩展 apiserver）。

> 排障提示：`kubectl` 报 `Unable to connect to the server` 是 apiserver 都没通；报 `forbidden` 是过了认证卡在授权；报 `admission webhook denied` 是卡在准入。

### 2.2.2 etcd

集群的**唯一事实来源（Source of Truth）**。所有对象（包括 Secret、Service、Node 状态）都以 key-value 形式存在这里，key 形如 `/registry/pods/default/nginx-abc`。

技术特征：

- 基于 **Raft** 共识算法，强一致，写需要多数派（quorum）确认。3 节点容忍 1 台故障，5 节点容忍 2 台。**偶数节点没有意义**。
- 支持 **watch**，这是 K8s 能做到"秒级感知"的基础。
- 对**延迟极其敏感**：官方建议 etcd 的 fsync 延迟 P99 < 10ms、WAL fsync P99 < 10ms。用普通机械盘或网络存储跑 etcd 是常见的事故源，生产必须用本地 SSD。
- 默认 2GB 存储上限（`--quota-backend-bytes`），超过后集群变只读。建议按 8GB 配置并监控使用率，定期 `defrag`。
- **必须备份**。etcd 挂了且无备份，等于集群没了（Pod 还在跑，但所有控制器停摆，且无法恢复编排状态）。

### 2.2.3 kube-scheduler

负责决定"**Pod 应该放在哪个节点**"，它只做决策，不执行。工作方式是**两阶段**：

1. **过滤（Filter / Predicates）**：筛掉不满足硬性条件的节点。常见检查包括资源是否够（`NodeResourcesFit`）、节点选择器（`NodeAffinity`）、污点容忍（`TaintToleration`）、端口冲突、PVC 的可用区约束（`VolumeZone`）、Pod 间亲和反亲和（`InterPodAffinity`）。剩下的节点叫 feasible nodes。
2. **打分（Score / Priorities）**：给候选节点 0–100 分并加权求和，选出最高分。常见策略：`LeastAllocated`（资源使用率低的优先，默认均衡）、`BalancedAllocation`、`ImageLocality`（镜像已在本地的优先）、`InterPodAffinity`、`TopologySpread`。同分则随机。

关键行为：

- **没有可用节点时，Pod 处于 `Pending`**，事件里会写 `0/3 nodes are available: 3 Insufficient cpu`。这是排查 Pending 的第一手证据。
- **抢占（Preemption）**：高优先级 Pod 调度失败时，可以驱逐低优先级 Pod 腾位置（`priorityClassName`）。
- 支持 **调度框架（Scheduling Framework）** 插件，在 `PreFilter / Filter / PostFilter / Score / Reserve / Permit / Bind` 等扩展点插入自定义逻辑。
- 可以**指定调度器**（`spec.schedulerName`），用于多调度器场景。

### 2.2.4 kube-controller-manager

一个进程里跑了 40 多个控制器，每个都是一个独立的调谐循环，通过 **leader election** 保证同一时刻只有一个副本在工作（其余待命）。

必须认识的几个：

| 控制器 | 职责 |
|---|---|
| Deployment Controller | 管理 ReplicaSet 的版本与滚动节奏 |
| ReplicaSet Controller | 保证实际 Pod 数等于 `replicas` |
| StatefulSet Controller | 保证 Pod 的序号、稳定网络标识、PVC 绑定 |
| DaemonSet Controller | 保证每个（匹配的）节点有一个 Pod |
| Job / CronJob Controller | 任务完成与定时触发 |
| Node Controller | 监控节点心跳，失联时打 `node.kubernetes.io/unreachable` 污点并驱逐 Pod |
| EndpointSlice Controller | 把 Service 选择器匹配到的 Pod IP 写入 EndpointSlice |
| Namespace Controller | 命名空间删除时清理内部资源 |
| ServiceAccount / Token Controller | 创建 SA、签发 Token |
| PV / PVC Controller | 存储卷的绑定与回收 |
| HPA Controller | 周期性（默认 15s）计算期望副本数 |
| Garbage Collector | 依据 ownerReferences 级联删除 |

### 2.2.5 cloud-controller-manager

把云厂商相关逻辑从核心代码里剥离出来的组件，负责：节点生命周期（云主机被删则删 Node 对象）、路由（云网络路由表）、负载均衡器（`type: LoadBalancer` 的 Service 创建云 LB）。自建集群通常不部署它，用 MetalLB 等替代 LoadBalancer 实现。

## 2.3 工作节点组件

### 2.3.1 kubelet

节点上的"代理 + 工头"，以 systemd 服务形式常驻。它**不是**通过 apiserver 被动接收指令，而是**主动 watch** apiserver 上"绑定到本节点"的 Pod 列表。

核心职责：

- 把 PodSpec 翻译成容器运行时调用（通过 **CRI** gRPC 接口调 containerd/CRI-O）。
- 管理容器的生命周期：创建沙箱（pause 容器）、拉镜像、挂卷、启动容器。
- 执行**探针**：liveness / readiness / startup，失败时按策略重启或摘流量。
- 上报**节点与 Pod 状态**（`Node.Status`、`Pod.Status`），包括资源容量、已分配资源、Condition。
- 通过内置 **cAdvisor** 采集容器资源用量（这是 `kubectl top` 和 metrics-server 的数据源）。
- 挂载存储：调用 CSI 插件的 NodeStage/NodePublish。

kubelet 的关键配置：`--node-ip`、`--max-pods`（默认 110）、`--eviction-hard`（如 `memory.available<100Mi`，触发驱逐）、`--image-gc-high-threshold`（默认 85%，磁盘回收）、`--kube-reserved / --system-reserved`（给系统组件预留资源）。

> 注意：`kubectl top` 的数据来自 metrics-server，而 metrics-server 的数据来自 kubelet 的 Summary API。如果 `kubectl top` 报 `Metrics API not available`，先查 metrics-server 是否就绪，而不是查 kubelet。

### 2.3.2 kube-proxy

实现 **Service 的虚拟 IP 到 Pod IP 的转发**。它 watch Service 与 EndpointSlice，在本机写入转发规则。

三种模式：

| 模式 | 原理 | 特点 |
|---|---|---|
| iptables（默认） | 写 nat 表规则做 DNAT | 稳定通用；规则数随 Service 线性增长，几千个 Service 时同步变慢 |
| IPVS | 内核 LVS，哈希表 | 查找 O(1)，支持多种调度算法（rr、lc、wrr）；需要内核模块 |
| nftables | 新版替代 iptables | 规则集更高效，较新版本可用 |

它**不参与 Pod 之间的直接通信**（那是 CNI 的事），只处理"访问 Service IP / NodePort"这一层。

### 2.3.3 容器运行时（CRI）

K8s 通过 **CRI（Container Runtime Interface）** 与运行时解耦。主流实现：

- **containerd**：当前最主流，kubeadm 默认，Docker 的底层运行时。
- **CRI-O**：为 K8s 定制，OpenShift 默认。
- **Docker Engine**：自 1.24 起 **dockershim 被移除**，不能再直接作为运行时（但 Docker 构建的镜像完全兼容 OCI，仍可正常使用）。

两者的分工：**kubelet 管 Pod 层面，运行时管容器层面**。镜像格式遵循 OCI 标准，所以 `docker build` 的镜像在 containerd 上能直接跑。

### 2.3.4 CNI 网络插件

K8s 只规定网络模型（每个 Pod 一个 IP、Pod 间扁平互通、无需 NAT），具体实现交给 CNI 插件：

| 插件 | 数据面 | 特点 |
|---|---|---|
| Calico | iptables/eBPF | 老牌，BGP 路由，NetworkPolicy 完善，纯三层无 overlay 可选 |
| Cilium | eBPF | 性能好，可观测性强，L7 策略，Hubble，近年主流 |
| Flannel | VXLAN/host-gw | 最简单，只做连通不做策略，适合入门 |
| Antrea | OVS/eBPF | VMware 系，策略能力强 |
| 云厂商 CNI | VPC 原生 | 阿里云 Terway、AWS VPC CNI，Pod 直接用 VPC IP |

选型一句话：**要网络策略就排除 Flannel；追求性能与可观测性选 Cilium；要成熟稳定和三层路由选 Calico。**

## 2.4 一次 `kubectl apply` 的完整旅程

把上面的组件串起来，看一条命令如何变成运行中的容器。这段流程是面试高频题，也是排障的路线图。

```text
1. kubectl 读取 kubeconfig，找到 apiserver 地址、证书、当前 namespace
2. kubectl 把 YAML 序列化为 JSON，POST /apis/apps/v1/namespaces/default/deployments
3. apiserver 完成 认证(TLS 证书) → 授权(RBAC 检查 create deployments)
4. 准入链：MutatingWebhook（如注入 sidecar）→ 内置准入（ResourceQuota、PodSecurity）
   → ValidatingWebhook（如校验镜像仓库白名单）
5. 校验 schema 与默认值填充，写入 etcd，返回 201 + 对象（含 uid、resourceVersion）
6. Deployment Controller watch 到新对象，创建一个 ReplicaSet（ownerReference 指向 Deployment）
7. ReplicaSet Controller watch 到新 RS，按 replicas 创建 N 个 Pod 对象（此时 Pod 无 nodeName）
8. Scheduler watch 到未绑定的 Pod：Filter 过滤节点 → Score 打分 → 选中 node-2
9. Scheduler 通过 apiserver 写入 Pod 的 spec.nodeName（Bind 操作）
10. node-2 上的 kubelet watch 到"属于我且未启动"的 Pod：
    调 CRI 创建 sandbox（pause）→ 分配 Pod IP（调 CNI）→ 拉镜像 → 启动容器
11. kubelet 更新 Pod.Status = Running，上报 PodIP、conditions（Ready=False 直到探针通过）
12. EndpointSlice Controller 发现 Pod Ready，把它的 IP 加入对应 Service 的 EndpointSlice
13. kube-proxy watch 到 EndpointSlice 变化，更新本机 iptables/IPVS 规则
14. 此时访问 Service ClusterIP，流量才会被转发到该 Pod
15. 若配置了 HPA，HPA Controller 每 15s 拉取指标，必要时修改 Deployment 的 replicas，回到第 6 步
```

排障时按这个链条倒推，几乎不会迷路：**对象在不在 → 调度成没成 → 容器起没起 → 探针过没过 → 端点加没加 → 规则写没写**。

## 2.5 控制器模式与 informer 机制

### 2.5.1 期望状态与实际状态的收敛

所有控制器共享同一个骨架：

```go
for {
    desired := 从 API 读取期望状态        // 例如 Deployment.spec.replicas = 3
    actual  := 从 API 读取实际状态        // 例如现有 2 个 Ready 的 Pod
    if desired != actual {
        takeAction()                      // 创建 1 个 Pod
    }
}
```

这段伪代码解释了三件事：

- **为什么控制器天然幂等**：它每次都在比较"整体状态"，重复执行不会多建 Pod（因为会重新计算差量）。
- **为什么最终一定会收敛**：即使中间失败，下一轮循环还会重试。
- **为什么不要手改控制器管理的资源**：你手删一个 ReplicaSet 管理的 Pod，RS 会立刻补回来——这不是 bug，是设计。要改就改上层（Deployment）。

### 2.5.2 list-watch 与 informer

如果每个控制器都轮询 apiserver，集群会被打爆。K8s 的解法是 **list-watch + 本地缓存**：

1. **Reflector** 先 `list` 全量对象（拿到 `resourceVersion`），再 `watch` 该版本之后的增量事件（ADDED / MODIFIED / DELETED）。
2. 事件写入 **DeltaFIFO** 队列，**Indexer** 维护一份本地只读缓存（带索引，可按 namespace/label 快速查询）。
3. 控制器从 **WorkQueue** 取 key（`namespace/name`），从缓存读对象处理。队列自动去重、可限速重试。
4. 处理失败则 `AddRateLimited` 重新入队，指数退避。

这带来两个实践认知：

- **`kubectl get` 默认读的是 apiserver 的实时数据**（`--cache-dir` 除外），而控制器决策基于本地缓存，所以存在**秒级延迟**。刚创建的对象"短暂不一致"是正常的。
- **`resourceVersion` 是乐观锁**。两个客户端同时改同一对象，后提交的会得到 `409 Conflict`，需重试。`kubectl apply` 用 `last-applied-configuration` 注解做三方合并来避免覆盖他人字段。

## 2.6 高可用与规模边界

### 2.6.1 生产级高可用拓扑

| 层次 | 做法 |
|---|---|
| apiserver | ≥ 3 副本，前置 L4 负载均衡（VIP / 云 SLB / HAProxy + keepalived） |
| etcd | 3 或 5 个独立节点，本地 SSD，与 master 分离部署（stacked 模式仅适合小集群） |
| scheduler / controller-manager | ≥ 2 副本，靠 leader election 自动选主 |
| 工作节点 | ≥ 3，跨可用区分布，配合 topologySpreadConstraints |
| 应用 | 多副本 + PDB + 反亲和，避免单节点/单可用区故障全灭 |
| 入口 | Ingress/LB 多副本，跨可用区 |

### 2.6.2 官方规模阈值（v1.30 左右）

| 指标 | 建议上限 |
|---|---|
| 单集群节点数 | 5000 |
| 单集群 Pod 总数 | 150,000 |
| 单集群容器总数 | 300,000 |
| 单节点 Pod 数 | 110（默认 `--max-pods`；官方性能测试按 100 验证） |
| etcd 数据量 | 建议 < 8GB（默认配额 2GB，超限后集群变只读） |

超过这些量级不是"跑不了"，而是"控制面延迟会显著上升"。超大规模通常走向**多集群联邦**（Karmada、Cluster API）或**按业务拆分集群**。

### 2.6.3 版本偏差策略

K8s 每 4 个月发一个次版本（minor），支持最近 3 个次版本的安全更新。组件间允许的版本偏差：

- **kubelet 可以比 apiserver 旧最多 3 个次版本**（例如 apiserver 1.32，kubelet 最低 1.29）。
- **kubelet 不能比 apiserver 新**。
- **kubectl 允许 ±1 个次版本**。
- **controller-manager / scheduler 应与 apiserver 同版本**。

升级顺序：先升控制面，再升 kubelet/kube-proxy，最后处理不兼容的 API（用 `kubectl convert` 或 `pluto` 提前扫描废弃 API）。


# 第3章 资源模型与核心对象

## 3.1 API 对象、GVK 与 YAML 结构

K8s 中一切都是 **API 对象**：你提交的、控制器管理的、etcd 存储的，都是同一套对象模型。每个对象由四部分组成：

```yaml
apiVersion: apps/v1        # ① 组/版本：决定字段 schema
kind: Deployment           # ② 资源类型
metadata:                  # ③ 元数据：名称、命名空间、标签、注解
  name: web
  namespace: default
  labels:
    app: web
spec:                      # ④ 期望状态：由你声明
  replicas: 3
status:                    # ⑤ 实际状态：由控制器写入，你只读
  readyReplicas: 3
```

**GVK（Group / Version / Kind）** 是对象的全局唯一标识。查询某个资源属于哪个组：

```bash
kubectl api-resources | grep deployments
# deployments   deploy   apps/v1   true   Deployment
```

`apiVersion` 的写法是 `组/版本`（核心组 `v1` 省略组名）。同一个 Kind 可能同时存在多个版本（如 `networking.k8s.io/v1` 与已废弃的 `v1beta1`），它们描述**同一个对象的不同表示**，apiserver 负责转换。

> 重要：`spec` 是你写的，`status` 是系统写的。**不要手改 status**，改了也会被控制器覆盖；要影响行为只能改 spec。

## 3.2 命名空间与资源作用域

**Namespace** 是集群内的逻辑分区，用于：隔离团队/环境、配合 RBAC 做权限边界、配合 ResourceQuota 做资源配额。

两大类资源：

| 类型 | 含义 | 例子 |
|---|---|---|
| 命名空间级 | 归属某个 namespace | Pod、Deployment、Service、ConfigMap、PVC、Secret、Ingress |
| 集群级 | 全局唯一，不属于 namespace | Node、PV、StorageClass、ClusterRole、CRD、Namespace 本身 |

查看某个资源的性质：

```bash
kubectl api-resources --namespaced=true   # 命名空间级
kubectl api-resources --namespaced=false  # 集群级
```

系统内置的命名空间：

- `default`：未指定 namespace 时的默认落点，生产建议只做过渡。
- `kube-system`：控制面组件与核心插件（CoreDNS、kube-proxy、metrics-server）。
- `kube-public`：所有人可读，通常放集群信息 ConfigMap。
- `kube-node-lease`：存放各节点的 Lease 对象（心跳），减轻 apiserver 压力。

> **命名空间不提供网络隔离**。`dev` 和 `prod` 的 Pod 默认可以互相访问，要隔离必须用 NetworkPolicy。命名空间也不隔离节点，Pod 仍可能混部在同一台机器上。

## 3.3 标签、选择器、注解与属主引用

### 3.3.1 Labels：用于"选择"的键值对

标签是 K8s 里**唯一**用于分组和筛选的机制。Service 找 Pod、ReplicaSet 管 Pod、NetworkPolicy 圈定范围，全靠标签选择器。

```yaml
metadata:
  labels:
    app.kubernetes.io/name: web
    app.kubernetes.io/component: frontend
    app.kubernetes.io/version: "1.27"
    env: prod
```

两种选择器：

```yaml
# 等值选择器（matchLabels）：多个条件是 AND
selector:
  matchLabels:
    app: web
    env: prod

# 集合选择器（matchExpressions）：支持 In / NotIn / Exists / DoesNotExist
selector:
  matchExpressions:
    - key: env
      operator: In
      values: ["prod", "staging"]
    - key: tier
      operator: Exists
```

命令行筛选：

```bash
kubectl get pods -l app=web,env=prod
kubectl get pods -l 'env in (prod,staging)'
kubectl get pods -l '!canary'                 # 不存在 canary 标签
kubectl get pods --show-labels
```

> **最常见的坑**：Deployment 的 `spec.selector.matchLabels` 与 `spec.template.metadata.labels` 不匹配，apiserver 直接拒绝（`selector does not match template labels`）。另外 selector **创建后不可修改**（apps/v1 的硬约束）。

### 3.3.2 Annotations：用于"传递信息"的键值对

注解不能用于筛选，用于给工具读的元数据：Ingress 控制器配置、Prometheus 抓取开关、GitOps 同步状态、变更原因。

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
    kubernetes.io/change-cause: "upgrade image to 1.27"
```

### 3.3.3 OwnerReference 与 Finalizer

**ownerReferences** 定义对象间的从属关系，是**级联删除**的依据。删 Deployment → 删它的 ReplicaSet → 删它的 Pod，靠的就是这条链。查看：

```bash
kubectl get pod web-xxx -o jsonpath='{.metadata.ownerReferences}'
```

**Finalizer** 是"删除前的拦截器"。对象被删除时，如果 `metadata.finalizers` 非空，对象只会被打上 `deletionTimestamp`（进入 Terminating），直到控制器完成清理并移除 finalizer 才真正消失。

> **经典故障**：Namespace 长时间卡在 `Terminating`，通常是有资源（如 APIService、CRD、PVC）的 finalizer 没人清理。排查见第 9 章。

## 3.4 资源请求与限制：QoS 与调度

### 3.4.1 requests 与 limits

```yaml
resources:
  requests:          # 调度依据：调度器按这个值找节点
    cpu: "250m"      # 0.25 核
    memory: "256Mi"
  limits:            # 运行时上限：cgroup 强制执行
    cpu: "1"
    memory: "512Mi"
```

| | requests | limits |
|---|---|---|
| 作用对象 | 调度器 | 内核 cgroup |
| 不足时 | Pod 调度不上去（Pending） | CPU 被限流（throttle），内存被 OOMKill |
| 可否超卖 | 是（节点上 requests 总和 ≤ 可分配） | 是（limits 总和可超过节点容量） |

**CPU 是可压缩资源**：超过 limit 只会被 throttle，进程变慢但不死。
**内存是不可压缩资源**：超过 limit 立刻 OOMKill，容器重启，`kubectl describe pod` 显示 `Last State: Terminated, Reason: OOMKilled, Exit Code: 137`。

实践中最重要的规则：**requests 必须设，limits 谨慎设**。

- 不设 requests → 调度器认为它不要资源 → 节点超卖 → 雪崩。
- 内存 limit 设得比实际峰值低 → 频繁 OOMKill。
- CPU limit 设得紧 → 延迟毛刺（throttling），对延迟敏感的服务建议只设 requests 不设 CPU limit。

### 3.4.2 QoS 等级

K8s 根据 requests/limits 的组合自动给 Pod 打三个 QoS 等级，**节点资源紧张时按等级从低到高驱逐**：

| QoS | 条件 | 驱逐优先级 |
|---|---|---|
| **Guaranteed** | 每个容器的每个资源都设了 requests 且等于 limits | 最后被驱逐 |
| **Burstable** | 至少一个容器设了 requests，但不满足 Guaranteed | 中间 |
| **BestEffort** | 所有容器都没设任何 requests/limits | 最先被驱逐 |

生产建议：核心服务用 Guaranteed；普通服务用 Burstable 且 requests 接近真实用量；**避免 BestEffort**。

### 3.4.3 单位速记

- CPU：`1` = 1 核 = 1000m。`0.5` 与 `500m` 等价。最小精度 1m。
- 内存：`Mi`/`Gi`（1024 进制）与 `M`/`G`（1000 进制）。**混用会导致容量算错**，统一用 Mi/Gi。
- 临时存储：`ephemeral-storage`，用于限制容器可写的本地磁盘。

## 3.5 Pod：最小调度单元

### 3.5.1 为什么是 Pod 而不是容器

Pod 是**一个或多个共享网络和存储的容器的集合**，是调度的原子单位。同一 Pod 内：

- 共享 **network namespace**：同一个 IP、同一个端口空间，容器间用 `localhost` 互访。
- 共享 **IPC namespace** 与 **UTS namespace**（同一 hostname）。
- 可以共享 **Volume**。
- 默认不共享 PID namespace（除非 `shareProcessNamespace: true`）。

这个抽象的必要性在于：有些进程必须**紧耦合**地共处（日志收集 sidecar、服务网格代理、配置热加载器），它们需要共享 localhost 和文件。Pod 让"这些进程必须在一起、必须同生共死"成为一等公民。

### 3.5.2 Pod 生命周期与相位

```text
Pending ──► Running ──► Succeeded
   │           │
   │           └──► Failed
   └──► Unknown（与节点失联）
```

- **Pending**：已被 apiserver 接受，但还没完成调度，或还在拉镜像。
- **Running**：已绑定节点，至少一个容器在运行或正在启动/重启。
- **Succeeded**：所有容器正常退出（exit 0），且不会重启。Job 的正常终态。
- **Failed**：所有容器都已终止，且至少一个非 0 退出。
- **Unknown**：无法获取 Pod 状态，通常是节点通信故障。

注意 `Running` **不代表可用**。可用的判据是 `Ready` condition 为 True，也就是 readinessProbe 通过。

### 3.5.3 容器类型

| 类型 | 声明位置 | 特点 |
|---|---|---|
| 主容器 | `containers` | 业务进程，可多个（不推荐） |
| 初始化容器 | `initContainers` | **串行**执行，全部成功才启动主容器；适合等依赖、初始化数据、改权限 |
| 边车容器 | `initContainers` + `restartPolicy: Always` | 原生 sidecar（1.29+ beta，1.33 GA），保证在主容器之前启动、之后停止 |
| 临时容器 | `kubectl debug` | 调试用，注入到运行中的 Pod，不参与调度与重启 |

初始化容器的典型用法：

```yaml
initContainers:
  - name: wait-for-db
    image: busybox:1.36
    command: ['sh', '-c', 'until nc -z mysql 3306; do echo waiting; sleep 2; done']
  - name: init-perm
    image: busybox:1.36
    command: ['sh', '-c', 'chown -R 1000:1000 /data']
    volumeMounts:
      - name: data
        mountPath: /data
```

> `initContainer` 失败的 Pod 会处于 `Init:Error` 或 `Init:CrashLoopBackOff`，且**不会**被 Service 摘除（它从未 Ready 过）。排查先看 `kubectl logs pod -c init-container名`。

### 3.5.4 三种探针

这是 K8s 中最容易配错、也最影响可用性的部分。

| 探针 | 回答的问题 | 失败后果 | 建议场景 |
|---|---|---|---|
| **livenessProbe** | 进程还活着吗？ | 重启容器 | 有死锁/假死风险的服务 |
| **readinessProbe** | 能接流量吗？ | 从 Service 摘除端点，不重启 | **所有对外服务都该配** |
| **startupProbe** | 启动完了吗？ | 重启容器；成功前禁用另外两个探针 | 启动慢的应用（Java、大数据） |

三种探测方式：

```yaml
# 1. HTTP GET：返回 200-399 视为成功（最常用）
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
    httpHeaders:
      - name: X-Probe
        value: kubelet
  initialDelaySeconds: 10   # 容器启动后等多久开始探测
  periodSeconds: 10         # 探测间隔
  timeoutSeconds: 3         # 单次超时
  successThreshold: 1       # 成功几次算恢复（liveness 只能为 1）
  failureThreshold: 3       # 连续失败几次判定失败

# 2. TCP 端口：能建连即成功
readinessProbe:
  tcpSocket:
    port: 3306
  periodSeconds: 5

# 3. 执行命令：退出码 0 即成功
readinessProbe:
  exec:
    command: ["sh", "-c", "test -f /tmp/ready"]
```

配置要点：

- **startupProbe 是慢启动应用的救星**。没有它，只能把 liveness 的 `initialDelaySeconds` 调得很大，而这会拖慢故障恢复。
- **liveness 不要探测下游依赖**。数据库挂了不应该重启你的 Pod（重启也没用），应该让 readiness 失败摘流量。这是"级联雪崩"的典型成因。
- **readiness 的 `failureThreshold` 不要太敏感**，瞬时抖动会导致端点频繁上下线，引发 502。
- 探针端点要**轻量**，不要在里面做重查询，否则探针本身成为负载。

### 3.5.5 重启策略与终止流程

`restartPolicy` 对 Pod 内所有容器生效：

- `Always`（默认）：总是重启，适合常驻服务。
- `OnFailure`：非 0 退出才重启，适合 Job。
- `Never`：从不重启。

**Pod 的优雅终止**是一个必须理解的流程（默认 `terminationGracePeriodSeconds: 30`）：

```text
1. 用户/控制器删除 Pod → 写入 deletionTimestamp，Pod 进入 Terminating
2. 【并行】kubelet 发送 SIGTERM 给容器主进程
3. 【并行】EndpointSlice Controller 把 Pod 从 Service 端点摘除
4. kube-proxy 更新规则（有一定传播延迟）
5. 应用收到 SIGTERM，开始优雅关闭：停止接新请求、处理完存量请求、关闭连接池
6. 若超过 grace period 还没退出，kubelet 发送 SIGKILL 强杀
7. 容器退出，kubelet 清理沙箱与卷，Pod 对象被删除
```

由此可得两条实践铁律：

1. **应用必须处理 SIGTERM**。否则会被 SIGKILL，正在处理的请求直接 500 或连接重置。
2. **预留 shutdown 时间**。`preStop` 里 `sleep 5~10` 可以让"摘端点"的规则先传播开，再开始关闭，避免"规则还没更新，流量还在打进来，进程已经退出"导致的 502。这是生产环境最常见的"发布期间少量 502"的根因。

```yaml
lifecycle:
  preStop:
    exec:
      command: ["sh", "-c", "sleep 10"]
terminationGracePeriodSeconds: 45   # 要大于 preStop + 应用自身关闭时间
```

## 3.6 工作负载控制器：选对抽象

不要直接创建裸 Pod（除非调试）。裸 Pod 没有自愈能力，节点故障就永久丢失。用控制器管理：

| 控制器 | 管什么 | Pod 身份 | 典型场景 |
|---|---|---|---|
| **Deployment** | 无状态应用 | 随机后缀，可互换 | Web API、前端、微服务 |
| **ReplicaSet** | 副本数（被 Deployment 管理） | 同上 | 一般不用直接建 |
| **StatefulSet** | 有状态应用 | 固定序号 `-0/-1/-2` + 稳定 DNS | 数据库、ZK、Kafka、ES |
| **DaemonSet** | 每节点一个 | 按节点名 | 日志采集、监控 agent、CNI、kube-proxy |
| **Job** | 一次性任务 | 随机 | 数据迁移、批处理、备份 |
| **CronJob** | 定时任务 | 随机 | 定时报表、清理、备份 |

### 3.6.1 Deployment

最常用的控制器。它管理 ReplicaSet，ReplicaSet 管理 Pod，形成三层结构：

```text
Deployment (web, replicas=3, image=1.27)
   └── ReplicaSet (web-7d9f8b6c5, 3 pods)   ← 当前版本
   └── ReplicaSet (web-5c4b3a2d1, 0 pods)   ← 上一个版本（保留用于回滚）
```

滚动更新的两个关键参数决定"发布多快、可用性多高"：

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 25%          # 允许超出 replicas 的最大数量（先扩容再缩容）
    maxUnavailable: 25%    # 允许不可用的最大数量（0 表示不减少可用副本）
```

- `maxUnavailable: 0` + `maxSurge: 1` → **零不可用**，发布最慢，最安全。生产面向用户的服务推荐。
- 两者都为 0 是非法的。
- `minReadySeconds: 10` 让新 Pod 在 Ready 后继续观察 10 秒才算真正可用，能拦掉"启动就绪但立刻就崩"的版本。

其他关键字段：

```yaml
revisionHistoryLimit: 10        # 保留多少个旧 RS 用于回滚
progressDeadlineSeconds: 600    # 超过这个时间没进展则标记 Failed
```

**Deployment 的硬约束**：`spec.selector` 不可变；`spec.template` 一变就产生新的 ReplicaSet，即一次"发布"。

### 3.6.2 StatefulSet

有状态服务的三个诉求，Deployment 都不满足，StatefulSet 全部满足：

1. **稳定的网络标识**：Pod 名为 `<sts-name>-<序号>`，且配 Headless Service 后有稳定 DNS：`web-0.web.default.svc.cluster.local`。Pod 重建后名字与 DNS 不变。
2. **稳定的存储**：通过 `volumeClaimTemplates` 为每个 Pod 生成独立的 PVC（`data-web-0`），Pod 漂移后仍挂回自己的盘。
3. **有序的启停**：默认 `podManagementPolicy: OrderedReady`，`0` 就绪后才起 `1`，逆序终止。这对主从复制、集群选主至关重要。

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: web-headless       # 必须指向一个 headless Service
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: app
          image: nginx:1.27
          volumeMounts:
            - name: data
              mountPath: /usr/share/nginx/html
  volumeClaimTemplates:           # 每个 Pod 一个 PVC
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: standard
        resources:
          requests:
            storage: 1Gi
```

> 注意：**删除 StatefulSet 不会删除 PVC**（这是有意的数据保护）。要彻底清理必须手动删 PVC。

### 3.6.3 DaemonSet

保证"每个（满足条件的）节点上运行一个 Pod"。用于节点级 agent：

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      hostNetwork: true          # 用宿主机网络
      hostPID: true
      tolerations:               # 容忍 master 的污点，才能在 master 上也跑
        - operator: Exists
      containers:
        - name: exporter
          image: prom/node-exporter:v1.8.2
          ports:
            - containerPort: 9100
          volumeMounts:
            - name: proc
              mountPath: /host/proc
              readOnly: true
      volumes:
        - name: proc
          hostPath:
            path: /proc
```

DaemonSet 的 Pod 会被调度器特殊处理：它**自带对节点 Condition 污点的容忍**（NotReady、Unreachable 等），但**不会**自动容忍你自定义的污点，需要显式写 `tolerations`。

### 3.6.4 Job 与 CronJob

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: migrate
spec:
  completions: 1          # 需要成功完成的 Pod 数
  parallelism: 1          # 同时跑几个
  backoffLimit: 4         # 失败重试次数上限
  activeDeadlineSeconds: 600   # 整体超时，超时强制失败
  ttlSecondsAfterFinished: 300 # 完成后 5 分钟自动清理（很实用）
  template:
    spec:
      restartPolicy: Never     # Job 必须是 Never 或 OnFailure
      containers:
        - name: migrate
          image: busybox:1.36
          command: ["sh", "-c", "echo migrating && sleep 5"]
```

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup
spec:
  schedule: "0 2 * * *"            # 每天 02:00（UTC，注意时区！）
  timeZone: "Asia/Shanghai"        # 1.27+ 支持显式时区，强烈建议设置
  concurrencyPolicy: Forbid        # Allow / Forbid / Replace
  startingDeadlineSeconds: 300     # 错过调度点多久内还补跑
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  suspend: false
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: busybox:1.36
              command: ["sh", "-c", "date && echo backup done"]
```

> CronJob 的两个坑：① 默认用 **UTC**，国内业务必须设 `timeZone`，否则会在早上 10 点跑"凌晨 2 点的任务"；② 调度精度受 controller 轮询影响，可能延迟几十秒，**不要用它做秒级定时**。

## 3.7 Service、DNS 与网络

### 3.7.1 为什么需要 Service

Pod 的 IP 是**临时**的：重建、扩缩、滚动都会变。Service 提供了一个**稳定的虚拟 IP（ClusterIP）和 DNS 名**，并通过标签选择器动态维护后端 Pod 列表。

```text
             Service: web (ClusterIP 10.96.12.34)
                        │   selector: app=web
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
   Pod 10.244.1.5  Pod 10.244.2.7  Pod 10.244.3.9
     (EndpointSlice 中动态维护)
```

### 3.7.2 四种类型

| type | 可访问范围 | 实现 | 场景 |
|---|---|---|---|
| **ClusterIP**（默认） | 仅集群内 | kube-proxy 写 iptables/IPVS | 服务间调用 |
| **NodePort** | 集群外（节点IP:端口） | 在每节点开 30000-32767 端口 | 临时暴露、无 LB 环境 |
| **LoadBalancer** | 集群外 | 云厂商 LB / MetalLB | 生产对外入口 |
| **ExternalName** | — | 返回 CNAME，无代理 | 把外部服务映射成集群内域名 |

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  type: ClusterIP
  selector:
    app: web
  ports:
    - name: http
      port: 80            # Service 自己的端口
      targetPort: 8080    # 容器端口（可用名字引用，如 http）
      protocol: TCP
  sessionAffinity: None   # ClientIP 可开启会话保持
```

访问路径差异：

- ClusterIP：`curl http://web.default.svc.cluster.local:80`
- NodePort：`curl http://<任意节点IP>:30080`
- LoadBalancer：`curl http://<LB公网IP>`

### 3.7.3 Headless Service

`clusterIP: None` 的 Service 不分配虚拟 IP，DNS 查询直接返回所有 Pod IP（A 记录）。这是 StatefulSet 稳定网络标识的基础，也常用于客户端自己做负载均衡（如 gRPC、Kafka 客户端）。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-headless
spec:
  clusterIP: None
  selector:
    app: web
  ports:
    - port: 80
```

此时 `web-0.web-headless.default.svc.cluster.local` 直接解析到 `web-0` 的 Pod IP。

### 3.7.4 Endpoints 与 EndpointSlice

Service 本身只是规则，真正的后端列表存在 **EndpointSlice**（Endpoints 的继任者，为大规模性能而设计）。只有 **Ready 的 Pod** 才会进入 `ready` 端点。

```bash
kubectl get endpointslices -l kubernetes.io/service-name=web
kubectl get endpoints web            # 旧接口，仍可用
```

> 排障金句：**"Service 访问不通，先看端点空不空。"** 端点为空只有一个原因：没有 Pod 同时满足"标签匹配"+"Ready"。这时候要查的是 Pod 的 readinessProbe 和标签，而不是 Service。

### 3.7.5 Kubernetes DNS 命名规则

CoreDNS 为每个 Service 生成如下记录：

```text
<service>.<namespace>.svc.cluster.local
```

在同一个 namespace 内可以简写为 `web`，跨 namespace 用 `web.other-ns` 或全名。Pod 自身的 DNS 也遵循规则：

```text
<pod-ip-dashed>.<namespace>.pod.cluster.local     # 如 10-244-1-5.default.pod.cluster.local
```

StatefulSet Pod 额外有：

```text
<sts>-<ordinal>.<headless-svc>.<namespace>.svc.cluster.local
```

`/etc/resolv.conf` 中的 `search` 域决定了短名的解析顺序，`ndots:5` 意味着"点数少于 5 的名字先尝试补全 search 域"，这也是**跨 namespace 访问建议写 FQDN 并加结尾点**（`web.other.svc.cluster.local.`）的原因——能省掉多次无效查询。

### 3.7.6 Ingress 与 Gateway API

Service 是四层（TCP/UDP），Ingress 是**七层 HTTP 路由**：按 Host 和 Path 把流量分发到不同 Service，并统一处理 TLS。

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  tls:
    - hosts: ["www.example.com"]
      secretName: example-tls
  rules:
    - host: www.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web
                port:
                  number: 80
```

**Ingress 只是一个规范，必须装一个 Ingress Controller 才生效**（nginx-ingress、Traefik、Higress、APISIX、云厂商 ALB Ingress）。只创建 Ingress 对象而没有控制器，什么都不会发生——这是新手最常见的困惑。

`pathType` 三选一：`Prefix`（路径前缀，最常用）、`Exact`（精确匹配）、`ImplementationSpecific`（交给控制器解释）。

**Gateway API** 是 Ingress 的演进方向：把"谁负责监听"（Gateway）与"谁负责路由"（HTTPRoute）拆成不同资源，支持跨 namespace 委派、TCP/UDP/gRPC 路由、更细的匹配与流量切分。需要安装 Gateway API CRD 和具体实现（Cilium、Envoy Gateway、Istio 等）。新项目建议优先评估。

> 生态提醒：社区已宣布 **ingress-nginx 于 2026 年 3 月停止维护**，新项目在做技术选型时应把 Gateway API 实现作为主选项。

### 3.7.7 网络模型与策略

K8s 网络模型三条基本要求：

1. 每个 Pod 有独立 IP，所有 Pod 之间**无需 NAT** 即可互通。
2. 节点上的 Pod 与所有节点上的 Pod 互通。
3. Pod 看到的自己的 IP 就是别人看到的 IP。

**NetworkPolicy** 是 Pod 级防火墙，**默认拒绝需要显式声明**（没被任何策略选中的 Pod 是全通的）：

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-allow-web
  namespace: prod
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes: ["Ingress", "Egress"]
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: web          # 同 namespace 的 web
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
      ports:
        - protocol: TCP
          port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: db
      ports:
        - protocol: TCP
          port: 5432
```

要点：`podSelector` 与 `namespaceSelector` 在**同一个** `from` 元素里是 **AND**，写成两个元素则是 **OR**。这是高频陷阱。

## 3.8 ConfigMap 与 Secret

### 3.8.1 ConfigMap

把配置从镜像里剥出来，做到"一次构建，多环境部署"。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_MODE: production
  LOG_LEVEL: info
  config.yaml: |
    server:
      port: 8080
    features:
      newUI: true
```

三种使用方式：

```yaml
# ① 全部注入为环境变量
envFrom:
  - configMapRef:
      name: app-config

# ② 单个 key 注入
env:
  - name: APP_MODE
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: APP_MODE

# ③ 挂载为文件（推荐放配置文件）
volumeMounts:
  - name: cfg
    mountPath: /etc/app
    readOnly: true
volumes:
  - name: cfg
    configMap:
      name: app-config
```

**关键差异**：以**卷**方式挂载的 ConfigMap 更新后，容器内的文件会在约 1 分钟内自动同步（kubelet 定期刷新软链接）；以**环境变量**方式注入的**永远不会更新**，必须重启 Pod。需要热加载就用卷 + 应用自行监听文件变化，或配合 Reloader 自动重启。

### 3.8.2 Secret

结构同 ConfigMap，但值必须是 **base64 编码**，且有更严格的 RBAC 与（可选的）etcd 静态加密。

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque            # 还有 kubernetes.io/tls、kubernetes.io/dockerconfigjson 等
data:
  username: YWRtaW4=    # echo -n admin | base64
  password: cGFzc3dvcmQ=
```

命令行创建（避免手写 base64 出错）：

```bash
kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password='S3cr3t!' \
  --dry-run=client -o yaml > secret.yaml

kubectl create secret docker-registry regcred \
  --docker-server=registry.cn-hangzhou.aliyuncs.com \
  --docker-username=user --docker-password=pass

kubectl create secret tls example-tls --cert=tls.crt --key=tls.key
```

**必须强调的安全事实**：base64 只是编码不是加密。任何能 `get secret` 的人都能解出明文。生产要求：

- 开启 etcd 静态加密（`EncryptionConfiguration`）；
- 用 RBAC 严格限制 Secret 的读权限（`get/list/watch secrets` 权限等价于拿到全部凭据）；
- 更彻底的方案是接外部密钥管理：Vault + CSI Driver、Sealed Secrets、External Secrets Operator、云厂商 KMS；
- **不要把 Secret 提交到 Git**（明文 YAML 尤其禁止）。

## 3.9 存储：Volume、PV、PVC 与 StorageClass

### 3.9.1 三层模型

```text
StorageClass（存储"类型"：SSD、HDD、NFS……）
     │ 动态供给
     ▼
PV（PersistentVolume，集群级真实存储）
     ▲ 绑定
PVC（PersistentVolumeClaim，命名空间级的"申请单"）
     ▲ 引用
Pod.spec.volumes
```

用户只写 PVC："我要 10Gi、ReadWriteOnce、用 ssd 类"。StorageClass 的 provisioner（CSI 驱动）自动创建对应的云盘/NFS 目录，绑定后 Pod 就能用。

### 3.9.2 生命周期

```text
Provisioning（静态：管理员建 PV；动态：SC 自动建）
      ↓
Binding（PVC 与 PV 一对一绑定，容量/访问模式/SC 需匹配）
      ↓
Using（Pod 挂载）
      ↓
Releasing（PVC 删除，PV 进入 Released）
      ↓
Reclaiming（Retain 保留 / Delete 删除 / Recycle 已废弃）
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: standard     # 留空则用默认 SC
  resources:
    requests:
      storage: 10Gi
```

访问模式：

| 模式 | 缩写 | 含义 |
|---|---|---|
| ReadWriteOnce | RWO | 只能被**单个节点**读写挂载（云盘最常见） |
| ReadOnlyMany | ROX | 多节点只读 |
| ReadWriteMany | RWX | 多节点读写（需要 NFS、CephFS、EFS 等） |
| ReadWriteOncePod | RWOP | 只能被**单个 Pod** 读写（1.22+） |

> 踩坑预警：**RWO 不等于"单 Pod"**，而是"单节点"。同一个节点上的两个 Pod 可以共享同一个 RWO 卷。云盘跨可用区不能挂载，所以带 PVC 的 Pod 会被调度到卷所在可用区（`VolumeZone` 约束）——这也是"Pod Pending，事件说 node(s) had volume node affinity conflict"的原因。

### 3.9.3 常用的 Volume 类型

| 类型 | 持久性 | 场景 |
|---|---|---|
| emptyDir | Pod 生命周期内 | 临时缓存、容器间共享文件 |
| hostPath | 节点级（Pod 迁移就丢） | 访问宿主机文件（agent 类），慎用于业务 |
| configMap / secret | 只读配置 | 配置文件注入 |
| projected | 多种来源合并 | 把 SA token + configmap 挂到同一目录 |
| persistentVolumeClaim | 持久 | 业务数据 |
| nfs / csi | 持久 | 共享存储 |

### 3.9.4 StorageClass 与默认类

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: disk.csi.aliyun.com
parameters:
  type: cloud_ssd
reclaimPolicy: Delete          # 生产关键数据建议 Retain
allowVolumeExpansion: true     # 允许扩容 PVC
volumeBindingMode: WaitForFirstConsumer   # 延迟绑定，避免跨可用区调度失败
```

`volumeBindingMode` 很重要：`Immediate` 会在 PVC 创建时就建盘，可能建在 Pod 调度不到的可用区；`WaitForFirstConsumer` 等 Pod 调度后再建盘，**云环境强烈建议用它**。

扩容：`allowVolumeExpansion: true` 时，直接改 PVC 的 `resources.requests.storage`，文件系统会自动扩容（部分驱动需要重启 Pod）。

## 3.10 调度：把 Pod 放到正确的位置

调度器默认按"资源够 + 均衡"决策。要干预，有六种武器：

### 3.10.1 nodeSelector（最简单）

```yaml
spec:
  nodeSelector:
    disktype: ssd
```
节点需先打标签：`kubectl label node node-1 disktype=ssd`。

### 3.10.2 节点亲和性（表达能力更强）

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:    # 硬约束：不满足就不调度
      nodeSelectorTerms:
        - matchExpressions:
            - key: topology.kubernetes.io/zone
              operator: In
              values: ["cn-hangzhou-a", "cn-hangzhou-b"]
    preferredDuringSchedulingIgnoredDuringExecution:   # 软约束：打分时加分
      - weight: 100
        preference:
          matchExpressions:
            - key: node-type
              operator: In
              values: ["gpu"]
```

名字里的 `IgnoredDuringExecution` 意思是：**调度时考虑，运行中节点标签变了不会驱逐 Pod**。支持的操作符：`In`、`NotIn`、`Exists`、`DoesNotExist`、`Gt`、`Lt`。

### 3.10.3 Pod 亲和与反亲和

控制 Pod 之间"要不要在一起"：

```yaml
affinity:
  podAntiAffinity:      # 反亲和：同一应用的副本不要挤在一个节点
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: web
          topologyKey: kubernetes.io/hostname    # 以"节点"为拓扑域
```

把 `topologyKey` 换成 `topology.kubernetes.io/zone`，就变成"副本分散到不同可用区"——这是生产高可用的标配。

> 注意：`requiredDuringScheduling` 的**反亲和**在大规模下会显著拖慢调度（每个节点都要评估），几百副本以上建议改用 `topologySpreadConstraints`。

### 3.10.4 拓扑分布约束（推荐的打散方式）

```yaml
topologySpreadConstraints:
  - maxSkew: 1                       # 各拓扑域之间数量差不超过 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule # 或 ScheduleAnyway（软约束）
    labelSelector:
      matchLabels:
        app: web
```

它比反亲和更"柔性"：允许一定偏斜，但保证整体均衡，且性能更好。K8s 1.19+ 稳定，是当前推荐的打散方案。

### 3.10.5 污点与容忍（节点主动拒绝）

**污点（Taint）打在节点上**，表示"我有毛病/我有专用用途"；**容忍（Toleration）写在 Pod 上**，表示"我能忍受"。

```bash
# 打污点：key=value:效果
kubectl taint node node-1 dedicated=gpu:NoSchedule
kubectl taint node node-1 dedicated=gpu:NoSchedule-   # 末尾加 - 表示移除
```

三种效果：

| 效果 | 含义 |
|---|---|
| NoSchedule | 新 Pod 不调度上来，已有的不动 |
| PreferNoSchedule | 尽量不调度（软性） |
| NoExecute | 新 Pod 不调度，**已有的不忍耐的 Pod 立即驱逐** |

```yaml
tolerations:
  - key: dedicated
    operator: Equal
    value: gpu
    effect: NoSchedule

  # 容忍所有污点（DaemonSet 常用）
  - operator: Exists
```

系统自动加的污点，必须认识：

| 污点 | 含义 |
|---|---|
| `node.kubernetes.io/not-ready` | 节点未就绪 |
| `node.kubernetes.io/unreachable` | 节点失联 |
| `node.kubernetes.io/memory-pressure` | 内存压力 |
| `node.kubernetes.io/disk-pressure` | 磁盘压力 |
| `node.kubernetes.io/pid-pressure` | PID 耗尽 |
| `node.kubernetes.io/network-unavailable` | 网络未就绪 |
| `node.kubernetes.io/unschedulable` | 被 cordon |

> 排障要点：Pod 默认会自动容忍 `not-ready` 和 `unreachable` 各 300 秒（由 Admission 注入），所以节点宕机后 Pod 大约 5 分钟才被驱逐重建。想更快恢复就调小这个容忍时间，或配置 `pod-eviction-timeout`。

### 3.10.6 优先级与抢占

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
preemptionPolicy: PreemptLowerPriority
```

Pod 引用 `priorityClassName: high-priority`。当高优先级 Pod 调度不上时，调度器会尝试驱逐低优先级的 Pod 腾位置。**这是双刃剑**：配错会让核心服务被批量任务挤掉。

## 3.11 弹性伸缩

| 机制 | 维度 | 依据 | 说明 |
|---|---|---|---|
| **HPA** | 副本数（水平） | CPU/内存/自定义/QPS 指标 | 最常用 |
| **VPA** | 单 Pod 资源（垂直） | 历史用量推荐 | 需重启 Pod 生效，与 HPA 的 CPU 指标冲突 |
| **Cluster Autoscaler** | 节点数 | 是否有 Pending Pod | 云环境扩容节点 |
| **KEDA** | 副本数（事件驱动） | 队列长度、Kafka lag、cron | 可缩到 0 |

HPA 示例：

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60     # 相对 requests 的百分比
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Percent
          value: 100
          periodSeconds: 15
    scaleDown:
      stabilizationWindowSeconds: 300   # 缩容更保守，防止抖动
      policies:
        - type: Pods
          value: 1
          periodSeconds: 60
```

**HPA 生效的前提**：① Deployment 设置了 `resources.requests.cpu`（否则无法算利用率）；② 集群装了 metrics-server；③ 指标确实可用（`kubectl top pods` 有数据）。

## 3.12 安全：四道防线

### 3.12.1 认证与 RBAC

四类资源构成权限模型：

```text
ServiceAccount / User / Group  ──绑定──►  Role（命名空间内）
                                    └──►  ClusterRole（集群级）
                              RoleBinding / ClusterRoleBinding
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev
  name: pod-reader
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "update", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: dev
  name: read-pods
subjects:
  - kind: ServiceAccount
    name: ci-bot
    namespace: dev
  - kind: User
    name: alice@example.com
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

`verbs` 常用值：`get`、`list`、`watch`、`create`、`update`、`patch`、`delete`、`deletecollection`、`*`。子资源用 `/` 表示，如 `pods/log`、`pods/exec`、`deployments/scale`。

**最小权限实践**：ServiceAccount 的 Token 以投射卷形式自动挂载到 Pod（1.24 起不再自动创建长期 Secret Token）；不需要访问 API 的工作负载应显式设置 `automountServiceAccountToken: false`。给 CI 用 Role 而不是 ClusterRole；绝不把 `cluster-admin` 绑给应用。

### 3.12.2 Pod 安全上下文

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  runAsGroup: 3000
  fsGroup: 2000
  seccompProfile:
    type: RuntimeDefault
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
    add: ["NET_BIND_SERVICE"]
```

这组配置是"容器逃逸"的主要防线。**Pod Security Admission（PSA）** 用三个等级（`privileged`、`baseline`、`restricted`）在命名空间粒度强制执行，通过命名空间标签启用：

```bash
kubectl label ns prod \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest \
  pod-security.kubernetes.io/warn=restricted
```

### 3.12.3 镜像与供应链

- 只拉可信仓库，私有仓库用 `imagePullSecrets`；
- **不用 `:latest` 标签**，用 digest 或明确的版本号（`nginx@sha256:...`）；
- 配 `imagePullPolicy: IfNotPresent`（`latest` 会强制 Always，这也是别用它的原因）；
- 扫描镜像漏洞（Trivy、Grype），签名验证（Cosign + admission 策略）；
- 配 `ResourceQuota`/`LimitRange` 防止资源滥用。

### 3.12.4 审计与准入策略

`--audit-policy-file` 记录所有 API 请求（谁、什么时候、对什么、做了什么）。准入策略可用 OPA Gatekeeper、Kyverno 强制"禁止 latest 标签""必须设 resources""必须设探针"等规则。这是把团队规范代码化的关键手段。

## 3.13 扩展机制：CRD 与 Operator

### 3.13.1 CRD（自定义资源定义）

CRD 让你在 apiserver 中注册新的 Kind，从而用 `kubectl get` 管理自定义资源：

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: websites.example.com
spec:
  group: example.com
  scope: Namespaced
  names:
    kind: Website
    plural: websites
    singular: website
    shortNames: ["ws"]
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                image:
                  type: string
                replicas:
                  type: integer
                  minimum: 1
              required: ["image"]
            status:
              type: object
              properties:
                readyReplicas:
                  type: integer
      subresources:
        status: {}      # 启用 status 子资源，控制器写 status 不影响 spec
        scale: {}       # 支持 kubectl scale
```

### 3.13.2 Operator

CRD 只是"数据"，Operator = **CRD + 控制器**，把人类运维知识编码成程序。典型形态：

```text
用户创建:  kind: PostgresCluster, spec: {version: 15, replicas: 3, storage: 100Gi}
Operator 循环:
   → 创建 StatefulSet（3 副本）
   → 创建 Service、Secret
   → 配置主从复制
   → 定期备份到对象存储
   → 主库故障时自动 failover
   → 升级版本时逐个滚动
```

主流 Operator 开发框架：**Kubebuilder**（controller-runtime）、**Operator SDK**（基于前者）。现成 Operator 可在 OperatorHub.io / Artifact Hub 找到（Prometheus、Strimzi Kafka、CloudNativePG 等）。

### 3.13.3 其他扩展点

| 扩展点 | 作用 |
|---|---|
| Admission Webhook | 拦截并改写/拒绝对象（注入 sidecar、校验规范） |
| Aggregation Layer | 注册扩展 apiserver（如 metrics-server、KEDA） |
| CRI / CNI / CSI | 替换容器运行时、网络、存储实现 |
| Scheduler Framework | 自定义调度插件 |
| Device Plugin | 暴露 GPU、FPGA 等异构资源 |
| Cloud Provider | 对接云厂商 API |


# 第4章 环境准备：十分钟起一个集群

## 4.1 四种方案怎么选

| 方案 | 启动速度 | 多节点 | 最接近生产 | 推荐用途 |
|---|---|---|---|---|
| **kind**（K8s in Docker） | 约 60 秒 | 支持（容器当节点） | 中 | 学习、CI、本文实验 |
| **minikube** | 约 2 分钟 | 支持（`--nodes`） | 中 | 学习、需要 dashboard/addons |
| **k3d**（k3s in Docker） | 约 30 秒 | 支持 | 中低 | 边缘模拟、快速验证 |
| **云托管 / kubeadm** | 十几分钟 | 真实节点 | 高 | 生产、性能测试 |

本文实验全部基于 **kind**。原因：一条命令创建、可指定多节点、删除干净、与 CI 集成好。

## 4.2 工具清单

需要三个东西：容器运行时（Docker Desktop 或 containerd）、`kind`（或 minikube）、`kubectl`。

```bash
# macOS
brew install kind kubectl

# Windows（PowerShell，需先装 Docker Desktop 或 WSL2）
winget install Kubernetes.kind
winget install Kubernetes.kubectl

# Linux（二进制安装）
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.24.0/kind-linux-amd64
chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind

curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/
```

验证：

```bash
kubectl version --client
kind version
docker info | head -5
```

## 4.3 用 kind 创建多节点集群

kind 默认只创建一个"节点"（其实是一个容器）。要模拟真实拓扑，用配置文件：

```yaml
# kind-cluster.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: lab
nodes:
  - role: control-plane
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "ingress-ready=true"
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
        protocol: TCP
      - containerPort: 443
        hostPort: 443
        protocol: TCP
  - role: worker
  - role: worker
```

```bash
kind create cluster --config kind-cluster.yaml
kubectl cluster-info --context kind-lab
kubectl get nodes -o wide
```

预期看到 1 个 control-plane + 2 个 worker，全部 `Ready`。

常用操作：

```bash
kind get clusters
kind load docker-image myapp:1.0 --name lab    # 把本地镜像塞进集群（无需推仓库）
kind delete cluster --name lab
```

> **为什么要 `kind load`**：kind 节点是容器，看不到宿主机 Docker 的镜像。要让集群用上本地构建的镜像，必须 `kind load docker-image`，并在 Pod 里写 `imagePullPolicy: IfNotPresent`（否则会去公网拉）。

## 4.4 用 minikube（需要 GUI 或 addons 时）

```bash
minikube start --nodes=3 --cpus=2 --memory=4096 --kubernetes-version=v1.31.0
minikube status
minikube dashboard                   # 打开 Web 控制台
minikube addons enable metrics-server
minikube addons enable ingress
minikube service web --url           # 拿到 NodePort 服务的访问地址
minikube ssh                         # 进入节点
minikube stop / delete
```

## 4.5 kubeconfig 的结构与多集群管理

`kubectl` 通过 `~/.kube/config`（Windows 为 `%USERPROFILE%\.kube\config`）找到集群。结构三要素：

```yaml
apiVersion: v1
kind: Config
clusters:                 # 集群：apiserver 地址 + CA 证书
  - name: kind-lab
    cluster:
      server: https://127.0.0.1:6443
      certificate-authority-data: <base64>
users:                    # 用户：客户端证书 / token
  - name: kind-lab
    user:
      client-certificate-data: <base64>
      client-key-data: <base64>
contexts:                 # 上下文：cluster + user + 默认 namespace
  - name: kind-lab
    context:
      cluster: kind-lab
      user: kind-lab
      namespace: default
current-context: kind-lab
```

常用操作：

```bash
kubectl config get-contexts                       # 列出所有上下文
kubectl config current-context                    # 当前上下文
kubectl config use-context kind-lab               # 切换
kubectl config set-context --current --namespace=dev   # 改默认 namespace（高频）
kubectl config view --minify --flatten            # 查看当前上下文合并后的完整配置
kubectl config rename-context old new
kubectl config delete-context x
```

> 多集群环境下，**强烈建议装 `kubectx` 和 `kubens`**（或 `fzf` 版 `kubectx`），切换上下文和命名空间一秒完成，还能避免"在 prod 上执行了删库命令"这类事故。

## 4.6 装三个必备组件

### 4.6.1 metrics-server（`kubectl top` 与 HPA 的前提）

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# kind / minikube 的自签证书需要跳过校验，否则 metrics-server 起不来
kubectl patch deployment metrics-server -n kube-system --type=json \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'

kubectl -n kube-system rollout status deploy/metrics-server
kubectl top nodes
```

> 生产环境**不要**加 `--kubelet-insecure-tls`，应该正确配置 kubelet 证书。

### 4.6.2 ingress-nginx（做第 5 章实验 6 时用）

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
kubectl -n ingress-nginx wait --for=condition=Ready pod \
  -l app.kubernetes.io/component=controller --timeout=180s
```

### 4.6.3 本地存储类（做实验 5 时用）

kind 自带 `standard` StorageClass（local-path-provisioner）。确认：

```bash
kubectl get storageclass
# NAME                 PROVISIONER             RECLAIMPOLICY
# standard (default)   rancher.io/local-path   Delete
```

有 `(default)` 就说明 PVC 不写 `storageClassName` 也能自动供给。

---

# 第5章 实战教程：15 个递进实验

> 每个实验都遵循"目标 → 操作 → 验证 → 要点"。建议按顺序做，后面的实验会复用到前面的对象。
> 所有实验在 `default` 命名空间完成；想隔离可先 `kubectl create ns lab && kubectl config set-context --current --namespace=lab`。

## 实验 1：第一个 Pod

**目标**：理解 Pod 的最小结构，掌握 `get/describe/logs/exec/delete` 这条主线。

```yaml
# pod-nginx.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      ports:
        - containerPort: 80
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
        limits:
          cpu: 500m
          memory: 256Mi
```

```bash
kubectl apply -f pod-nginx.yaml

kubectl get pod nginx -o wide          # 看 IP、所在节点
kubectl describe pod nginx             # 看事件、镜像、卷、探针
kubectl logs nginx                     # 看日志
kubectl exec -it nginx -- bash         # 进容器
# 容器内：curl -s localhost:80 | head -3 ; exit

kubectl delete pod nginx
```

**验证**：`kubectl get pod` 显示 `1/1 Running`，`describe` 末尾的 Events 里能看到 `Scheduled → Pulled → Created → Started` 四步。

**要点**：

- 直接创建的 Pod **没有自愈**。手动 `kubectl delete pod nginx` 之后它不会回来。
- `describe` 的 Events 是最重要的排障入口，**永远先看它**。
- `kubectl exec` 需要镜像里有 shell。distroless 镜像没有 shell，要用 `kubectl debug`（实验 15）。

## 实验 2：Deployment 滚动更新与回滚

**目标**：理解 ReplicaSet 的三层结构、滚动节奏、版本历史与回滚。

```yaml
# deploy-web.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  revisionHistoryLimit: 5
  selector:
    matchLabels:
      app: web
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  minReadySeconds: 5
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: web
          image: nginx:1.25
          ports:
            - containerPort: 80
          readinessProbe:
            httpGet:
              path: /
              port: 80
            periodSeconds: 5
```

```bash
kubectl apply -f deploy-web.yaml
kubectl rollout status deploy/web --timeout=120s

kubectl get deploy,rs,pod -l app=web          # 观察三层结构
kubectl get rs -l app=web                     # 当前只有 1 个 RS

# 触发滚动更新
kubectl set image deploy/web web=nginx:1.27
kubectl rollout status deploy/web

# 看两个 RS 的副本数变化（旧的 3→0，新的 0→3）
kubectl get rs -l app=web -w

# 版本历史与回滚
kubectl rollout history deploy/web
kubectl rollout undo deploy/web
kubectl rollout undo deploy/web --to-revision=1

# 暂停/恢复（做金丝雀时有用）
kubectl rollout pause deploy/web
kubectl rollout resume deploy/web

# 仅重启（不改镜像）
kubectl rollout restart deploy/web
```

**验证**：更新过程中持续执行 `kubectl get pod -l app=web -w`，应看到新旧 Pod 交替，且**任一时刻 Ready 的数量都 ≥ 3**（因为 `maxUnavailable: 0`）。

**要点**：

- `kubectl set image` 只是改 Deployment 的 spec，本质与 `kubectl apply` 等价，都会产生新 RS。
- `rollout history` 默认只显示版本号，看不到内容。**做发布时务必写变更原因**：
  ```bash
  kubectl annotate deploy/web kubernetes.io/change-cause="upgrade nginx to 1.27"
  ```
- `rollout undo` 是"把旧 RS 的模板复制回来"，会产生一个新 RS（版本号继续增长），而不是简单回退指针。

## 实验 3：Service 的三种暴露方式

**目标**：把实验 2 的 `web` 暴露出来，理解 ClusterIP / NodePort / 端口映射，并能排查"端点为空"。

```bash
# ① ClusterIP（集群内）
kubectl expose deploy web --name=web-cip --port=80 --target-port=80
kubectl get svc web-cip
# 集群内验证（起一个临时 Pod）
kubectl run tmp --rm -it --image=busybox:1.36 --restart=Never -- \
  wget -qO- http://web-cip.default.svc.cluster.local

# ② NodePort（集群外可访问）
kubectl expose deploy web --name=web-np --type=NodePort --port=80 --target-port=80
kubectl get svc web-np
# 输出形如 80:31234/TCP，然后：
curl http://localhost:31234          # kind 需配合 extraPortMappings 或 port-forward

# ③ port-forward（最方便的本地调试）
kubectl port-forward svc/web-cip 8080:80
# 另开终端：curl http://localhost:8080
```

**验证端点**：

```bash
kubectl get endpointslices -l kubernetes.io/service-name=web-cip
kubectl get endpoints web-cip
# ENDPOINTS 列应显示 3 个 10.244.x.x:80
```

**故障演练**（重要，务必做一遍）：

```bash
# 故意把 Service 的 selector 改错
kubectl patch svc web-cip -p '{"spec":{"selector":{"app":"wrong"}}}'
kubectl get endpoints web-cip        # ENDPOINTS 变为 <none>
curl http://localhost:8080           # 现象？

# 改回来
kubectl patch svc web-cip -p '{"spec":{"selector":{"app":"web"}}}'
```

**要点**：

- 访问不通的第一诊断永远是 `kubectl get endpoints <svc>`。**空端点 = 标签不匹配或 Pod 不 Ready**，与网络插件无关。
- `port-forward` 走的是 apiserver 的隧道，**不经过 Service/CNI/kube-proxy**，所以它能通不代表网络没问题，反之亦然。
- NodePort 范围 30000–32767，端口在每台节点上都监听，多节点时不要写死某个节点 IP（用 LB 或 Ingress）。

## 实验 4：ConfigMap 与 Secret 注入

**目标**：掌握配置的三种注入方式，并观察"卷挂载可热更新、环境变量不更新"。

```bash
# 创建 ConfigMap 与 Secret
kubectl create configmap app-config \
  --from-literal=APP_MODE=prod \
  --from-literal=LOG_LEVEL=info \
  --from-literal=index.html='<h1>Hello K8s</h1>'

kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password='P@ssw0rd'

kubectl get configmap app-config -o yaml
kubectl get secret db-secret -o jsonpath='{.data.password}' | base64 -d; echo
```

```yaml
# pod-config.yaml
apiVersion: v1
kind: Pod
metadata:
  name: cfg-demo
spec:
  containers:
    - name: app
      image: nginx:1.27
      env:
        - name: APP_MODE                      # 单个 key
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: APP_MODE
        - name: DB_PASSWORD                   # 来自 Secret
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
      envFrom:
        - configMapRef:
            name: app-config                  # 全部 key 变环境变量
      volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html    # 挂成静态文件
        - name: cfg
          mountPath: /etc/app
      resources:
        requests: {cpu: 50m, memory: 64Mi}
  volumes:
    - name: html
      configMap:
        name: app-config
        items:
          - key: index.html
            path: index.html
    - name: cfg
      configMap:
        name: app-config
```

```bash
kubectl apply -f pod-config.yaml
kubectl exec cfg-demo -- env | grep -E 'APP_MODE|LOG_LEVEL|DB_PASSWORD'
kubectl exec cfg-demo -- cat /usr/share/nginx/html/index.html

# 更新 ConfigMap，观察热更新
kubectl create configmap app-config \
  --from-literal=APP_MODE=prod --from-literal=LOG_LEVEL=debug \
  --from-literal=index.html='<h1>Updated</h1>' --dry-run=client -o yaml | kubectl apply -f -

sleep 60
kubectl exec cfg-demo -- cat /usr/share/nginx/html/index.html   # 已更新
kubectl exec cfg-demo -- printenv LOG_LEVEL                     # 仍是 info（未变）
```

**验证**：文件内容变了（约 1 分钟内），环境变量没变。

**要点**：

- 环境变量是**进程启动时**读入的，之后再怎么改 ConfigMap 都不会变——热更新必须靠卷 + 应用监听文件。
- 挂载 ConfigMap 的目录是**只读**的，且会覆盖挂载点下的内容（想只覆盖单个文件必须用 `items` + `subPath`，但 **subPath 挂载不支持热更新**）。
- Secret 的 base64 不是加密。生产要开 etcd 加密 + 收紧 RBAC，最好用 External Secrets / Vault。

## 实验 5：持久化存储 PV / PVC

**目标**：理解 PVC → StorageClass → PV 的动态供给与绑定，验证数据在 Pod 重建后仍然存在。

```yaml
# pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: standard
  resources:
    requests:
      storage: 1Gi
```

```yaml
# pod-pvc.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pvc-demo
spec:
  containers:
    - name: app
      image: nginx:1.27
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: data
```

```bash
kubectl apply -f pvc.yaml
kubectl get pvc data                   # STATUS 应为 Bound
kubectl get pv                         # 自动创建的 PV

kubectl apply -f pod-pvc.yaml
kubectl exec pvc-demo -- sh -c 'echo hello-persistent > /data/note.txt'

# 删 Pod，重建，数据应仍在
kubectl delete pod pvc-demo
kubectl apply -f pod-pvc.yaml
kubectl exec pvc-demo -- cat /data/note.txt
```

**验证**：输出 `hello-persistent`，说明数据独立于 Pod 生命周期。

**要点**：

- `pvc` 的 `STATUS` 有三个关键值：`Pending`（等供给/等 Pod 调度）、`Bound`（已绑定）、`Lost`（后端卷丢失）。
- 删除 PVC 后 PV 按 `reclaimPolicy` 处理：`Delete` 会真的删掉存储，`Retain` 保留数据但 PV 变 `Released`，需要手动清理后才能复用。
- 把 PVC 挂给多个 Pod 时，若存储只支持 RWO，第二个 Pod 会卡在 `ContainerCreating` 并报 `Multi-Attach error`。

## 实验 6：Ingress 七层路由与灰度

**目标**：用域名 + 路径把流量分发到不同 Service，并用注解做权重灰度。

前置：已完成 4.6.2 的 ingress-nginx 安装。

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  ingressClassName: nginx
  rules:
    - host: demo.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-cip
                port:
                  number: 80
```

```bash
kubectl apply -f ingress.yaml
kubectl get ingress web

# 让 demo.local 指向本地（kind 已把 80 端口映射出来）
# Linux/macOS: echo "127.0.0.1 demo.local" | sudo tee -a /etc/hosts
# Windows: 编辑 C:\Windows\System32\drivers\etc\hosts 追加 127.0.0.1 demo.local

curl -H "Host: demo.local" http://localhost/
```

**灰度演练**：先部署一个 v2 版本，再用注解把 20% 流量导过去。

```bash
kubectl create deploy web-v2 --image=nginx:1.27 --replicas=1
kubectl annotate deploy/web-v2 kubernetes.io/change-cause="canary v2"
kubectl expose deploy web-v2 --name=web-v2 --port=80 --target-port=80

kubectl annotate ingress web \
  nginx.ingress.kubernetes.io/canary="true" \
  nginx.ingress.kubernetes.io/canary-weight="20" --overwrite

# 循环请求 20 次，观察命中分布
for i in $(seq 1 20); do curl -s -H "Host: demo.local" http://localhost/ -o /dev/null -w "%{http_code} "; done; echo
```

**要点**：

- 只创建 Ingress 对象、没装 Controller，是**完全静默无效**的。先 `kubectl get pods -n ingress-nginx` 确认控制器在跑。
- 灰度注解（`canary`、`canary-weight`、`canary-by-header`）是 nginx-ingress 的**实现私有**能力，不是 Ingress 标准。换控制器要重写。
- 生产做灰度更推荐 **Gateway API 的 HTTPRoute 权重**或服务网格（Istio VirtualService），能力更标准化。

## 实验 7：探针与资源限制（含 OOMKilled 演练）

**目标**：亲眼看到三种探针的作用，以及超限内存被 OOMKill 的现象。

```yaml
# pod-probe.yaml
apiVersion: v1
kind: Pod
metadata:
  name: probe-demo
spec:
  containers:
    - name: app
      image: nginx:1.27
      ports:
        - containerPort: 80
      resources:
        requests:
          cpu: 100m
          memory: 64Mi
        limits:
          cpu: 200m
          memory: 128Mi          # 故意设小，稍后压爆
      startupProbe:
        httpGet: {path: /, port: 80}
        failureThreshold: 30
        periodSeconds: 2
      readinessProbe:
        httpGet: {path: /, port: 80}
        periodSeconds: 5
        failureThreshold: 2
      livenessProbe:
        httpGet: {path: /, port: 80}
        periodSeconds: 10
        failureThreshold: 3
```

```bash
kubectl apply -f pod-probe.yaml
kubectl get pod probe-demo -w           # 观察 READY 从 0/1 变 1/1

# 让 readiness 失败：把首页删掉
kubectl exec probe-demo -- rm /usr/share/nginx/html/index.html
kubectl get pod probe-demo -w           # READY 变 0/1，但 RESTARTS 不变
kubectl describe pod probe-demo | grep -A3 "Readiness probe failed"
```

**OOMKilled 演练**：

```bash
kubectl run oom --image=polinux/stress --restart=Never -- \
  stress --vm 1 --vm-bytes 200M --vm-hang 0

kubectl get pod oom -w          # 会看到 Running → OOMKilled → CrashLoopBackOff
kubectl describe pod oom | grep -E "Last State|Reason|Exit Code"
# Reason: OOMKilled    Exit Code: 137
```

**要点**：

- 退出码语义：`137` = 128+9（SIGKILL，通常 OOM 或被强杀）；`143` = 128+15（SIGTERM，优雅终止）；`1` = 应用自身错误。
- `kubectl logs --previous` 可以看**上一个崩溃实例**的日志，这是排查 CrashLoopBackOff 的必备参数。
- 内存 limit 要基于真实压测设定，而不是拍脑袋。JVM 应用还要注意 `-Xmx` 应小于容器 limit（建议 70%–80%），否则会被 cgroup 杀掉而不是抛 Java OOM。

## 实验 8：HPA 自动扩缩容

**目标**：让副本数随 CPU 使用率自动变化。

前置：metrics-server 已就绪（`kubectl top nodes` 有数据）。

```bash
kubectl autoscale deploy web --min=2 --max=10 --cpu-percent=50
kubectl get hpa web -w
```

或者写 YAML（推荐，可控性更强）：

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
```

**制造负载**：

```bash
kubectl run load --rm -it --image=busybox:1.36 --restart=Never -- \
  sh -c 'while true; do wget -q -O- http://web-cip > /dev/null; done'
```

另开终端观察：

```bash
kubectl get hpa web -w
kubectl get deploy web -w
# TARGETS 从 0%/50% 上升到 80%/50%，REPLICAS 从 3 增加到 6、8……
```

停止负载后，等 5 分钟（`stabilizationWindowSeconds`），副本数回落。

**要点**：

- HPA 算的是**相对 requests 的利用率**。若容器没设 `requests.cpu`，HPA 会报 `<unknown>/50%` 并且不工作。
- 扩容是**分钟级**的（默认 15s 采集、需要指标稳定），不适合应对秒级突发。突发流量要靠 `minReplicas` 兜底或预热。
- `scaleDown` 的稳定窗口是防抖的关键，设太小会导致副本数反复震荡。


## 实验 9：StatefulSet + Headless Service

**目标**：观察"稳定网络标识 + 独立存储 + 有序启停"三个特性。

```yaml
# sts.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-headless
spec:
  clusterIP: None
  selector:
    app: sts-web
  ports:
    - port: 80
      name: http
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: sts-web
spec:
  serviceName: web-headless
  replicas: 3
  selector:
    matchLabels:
      app: sts-web
  template:
    metadata:
      labels:
        app: sts-web
    spec:
      containers:
        - name: app
          image: nginx:1.27
          ports:
            - containerPort: 80
          volumeMounts:
            - name: data
              mountPath: /data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: standard
        resources:
          requests:
            storage: 100Mi
```

```bash
kubectl apply -f sts.yaml

# 观察有序创建：0 就绪后才创建 1
kubectl get pod -l app=sts-web -w

kubectl get pod -l app=sts-web
# sts-web-0  sts-web-1  sts-web-2

kubectl get pvc
# data-sts-web-0  data-sts-web-1  data-sts-web-2   ← 每个 Pod 独立

# 验证稳定 DNS
kubectl run dns-test --rm -it --image=busybox:1.36 --restart=Never -- \
  nslookup sts-web-0.web-headless

# 每个 Pod 写入不同内容
for i in 0 1 2; do
  kubectl exec sts-web-$i -- sh -c "echo pod-$i > /data/id.txt"
done
kubectl exec sts-web-1 -- cat /data/id.txt       # pod-1

# 删除一个 Pod，重建后名字、存储都不变
kubectl delete pod sts-web-1
kubectl get pod -l app=sts-web -w
kubectl exec sts-web-1 -- cat /data/id.txt       # 仍是 pod-1
```

**要点**：

- 删除 StatefulSet **不会**删除 PVC。清理时要额外 `kubectl delete pvc -l app=sts-web`。
- `podManagementPolicy: Parallel` 可关闭有序启动（启动更快，但失去顺序保证）。
- 滚动更新按**序号从大到小**逐个进行，且默认会等前一个 Ready 才继续（`updateStrategy: RollingUpdate` + `partition` 可实现灰度）。

## 实验 10：DaemonSet 与节点选择

**目标**：在每个节点跑一个 agent，并验证污点/容忍与节点标签。

```yaml
# ds.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-agent
spec:
  selector:
    matchLabels:
      app: node-agent
  template:
    metadata:
      labels:
        app: node-agent
    spec:
      tolerations:
        - operator: Exists          # 容忍所有污点，master 上也跑
      containers:
        - name: agent
          image: busybox:1.36
          command: ["sh", "-c", "while true; do echo agent on $(hostname); sleep 30; done"]
          resources:
            requests: {cpu: 10m, memory: 16Mi}
```

```bash
kubectl apply -f ds.yaml
kubectl get ds node-agent          # DESIRED 应等于节点数
kubectl get pod -l app=node-agent -o wide

# 给节点打标签并只让 DS 跑在带标签的节点上
kubectl label node kind-lab-worker gpu=true
# 在 spec.template.spec 加 nodeSelector: {gpu: "true"} 后重新 apply
kubectl get pod -l app=node-agent -o wide

# 给节点打污点，验证"不容忍的 DS Pod 会被赶走"
kubectl taint node kind-lab-worker2 maintenance=true:NoExecute
kubectl get pod -l app=node-agent -o wide
kubectl taint node kind-lab-worker2 maintenance=true:NoExecute-
```

**要点**：

- DaemonSet 的"每个节点一个"是**控制器直接指定 nodeName**实现的（绕过调度器的默认打分，但仍会做资源与亲和性检查）。所以它天然"均匀"。
- DaemonSet 会自动容忍内存/磁盘压力、NotReady、Unreachable 等系统污点，但**不自动容忍自定义污点**。
- 节点 `cordon` 后，DaemonSet 的 Pod **仍会保留**（这是期望行为，agent 要一直跑）。

## 实验 11：Job 与 CronJob

**目标**：跑一次性任务与定时任务，观察重试与并发策略。

```yaml
# job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  completions: 3          # 总共要成功 3 次
  parallelism: 2          # 同时跑 2 个
  backoffLimit: 2
  ttlSecondsAfterFinished: 120
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: pi
          image: perl:5.38
          command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(200)"]
          resources:
            requests: {cpu: 100m, memory: 64Mi}
```

```bash
kubectl apply -f job.yaml
kubectl get job pi
kubectl get pod -l job-name=pi -w
kubectl logs job/pi                  # 聚合所有 Pod 日志
kubectl get job pi -o jsonpath='{.status.succeeded}'
```

**故意失败，看重试**：

```bash
kubectl create job fail --image=busybox:1.36 -- sh -c 'echo fail; exit 1'
kubectl get pod -l job-name=fail -w    # 会看到多个 Pod（每次重试一个）
kubectl describe job fail | tail -20
```

**CronJob**：

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
  timeZone: "Asia/Shanghai"
  concurrencyPolicy: Forbid
  startingDeadlineSeconds: 60
  successfulJobsHistoryLimit: 2
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: hello
              image: busybox:1.36
              command: ["sh", "-c", "date; echo hello from cronjob"]
```

```bash
kubectl apply -f cronjob.yaml
kubectl get cj -w                    # LAST SCHEDULE 列会更新
kubectl get jobs
kubectl logs job/<job-name>
kubectl create job manual-run --from=cronjob/hello    # 手动触发一次（很实用）
```

**要点**：

- **`restartPolicy` 必须是 `Never` 或 `OnFailure`**，写 `Always` 会被 apiserver 拒绝。
- `backoffLimit` 是**重试次数**，超过后 Job 变 `Failed`；`activeDeadlineSeconds` 是**整体超时**，两者用途不同。
- CronJob 默认用 UTC，**一定要设 `timeZone`**（1.27+）。`concurrencyPolicy: Forbid` 能避免上一轮没跑完就叠下一轮。

## 实验 12：RBAC 最小权限

**目标**：创建一个只能读取 `lab` 命名空间 Pod 的 ServiceAccount，并验证权限边界。

```bash
kubectl create namespace lab
kubectl create serviceaccount ci-bot -n lab

cat <<'EOF' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: lab
  name: pod-reader
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: lab
  name: ci-bot-read
subjects:
  - kind: ServiceAccount
    name: ci-bot
    namespace: lab
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
EOF
```

**验证**：

```bash
# 能做什么？
kubectl auth can-i --list --as=system:serviceaccount:lab:ci-bot -n lab

# 逐条验证
kubectl auth can-i list pods -n lab --as=system:serviceaccount:lab:ci-bot
# yes
kubectl auth can-i delete pods -n lab --as=system:serviceaccount:lab:ci-bot
# no
kubectl auth can-i list pods -n default --as=system:serviceaccount:lab:ci-bot
# no（Role 只在 lab 生效）

# 用这个 SA 的 Token 实际请求一次
TOKEN=$(kubectl create token ci-bot -n lab --duration=1h)
kubectl --token="$TOKEN" --server=https://127.0.0.1:6443 \
  --certificate-authority=<CA路径> get pods -n lab
```

**要点**：

- 权限判定顺序：**默认拒绝**，只有显式 allow 才通过。没有 deny 规则（除 Node 授权外）。
- Role + RoleBinding 是**命名空间内**；ClusterRole + RoleBinding 可以在某命名空间授予集群级权限；ClusterRole + ClusterRoleBinding 才是全局。
- `kubectl auth can-i --list` 是审计权限最好用的命令，**上线前必跑**。
- `get secrets` 权限 = 拿到所有凭据权限，等价于 cluster-admin 一大半，授予要极其谨慎。

## 实验 13：NetworkPolicy 网络隔离

**目标**：默认全通 → 加策略后只允许指定来源访问。

前置：kind 需要带支持 NetworkPolicy 的 CNI（默认 kindnet **不支持** NetworkPolicy）。改用 Calico：

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.2/manifests/calico.yaml
kubectl -n kube-system rollout status ds/calico-node --timeout=180s
```

```yaml
# netpol.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-allow-web-only
  namespace: lab
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes: ["Ingress"]
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: web
      ports:
        - protocol: TCP
          port: 80
```

```bash
kubectl -n lab create deploy api --image=nginx:1.27
kubectl -n lab expose deploy api --port=80 --target-port=80
kubectl -n lab create deploy web --image=nginx:1.27
kubectl -n lab create deploy other --image=nginx:1.27

# 加策略前：两个客户端都能访问
kubectl -n lab exec deploy/other -- curl -s -m3 -o /dev/null -w "%{http_code}\n" http://api
# 200

kubectl apply -f netpol.yaml

kubectl -n lab exec deploy/other -- curl -s -m3 -o /dev/null -w "%{http_code}\n" http://api
# 000（超时，被策略丢弃）
kubectl -n lab exec deploy/web -- curl -s -m3 -o /dev/null -w "%{http_code}\n" http://api
# 200
```

**要点**：

- NetworkPolicy 是**白名单**模型：一旦某个 Pod 被某条策略的 `podSelector` 选中且 `policyTypes` 含 `Ingress`，它的入向流量就变成"默认拒绝，只放行规则里的"。
- **DNS 必须放行**。一旦写了 `Egress` 策略，别忘了放行到 `kube-system` 的 53 端口，否则所有域名解析失败（现象极具迷惑性：`curl: could not resolve host`）。
- 策略是**命名空间级资源**，且**只对同集群内 Pod 间流量生效**；不作用于宿主机到 Pod 或外部 LB 的流量（取决于 CNI 实现）。

## 实验 14：用 Helm 部署一个真实应用

**目标**：掌握 Helm 的三个核心动作（install / upgrade / rollback）与 values 覆盖。

```bash
# 安装 Helm
# macOS: brew install helm
# Windows: winget install Helm.Helm
# Linux: curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

helm version
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo nginx          # 搜索
```

```bash
# 安装（自定义 values）
cat > values.yaml <<'EOF'
replicaCount: 2
service:
  type: ClusterIP
resources:
  requests: {cpu: 100m, memory: 128Mi}
  limits: {cpu: 500m, memory: 256Mi}
EOF

helm install my-nginx bitnami/nginx -n demo --create-namespace -f values.yaml
helm list -A
kubectl -n demo get deploy,svc,pod

# 查看渲染后的最终 YAML（排错神器）
helm get manifest my-nginx -n demo
helm template my-nginx bitnami/nginx -f values.yaml | head -40

# 升级
helm upgrade my-nginx bitnami/nginx -n demo -f values.yaml --set replicaCount=3

# 历史与回滚
helm history my-nginx -n demo
helm rollback my-nginx 1 -n demo

# 卸载
helm uninstall my-nginx -n demo
```

**自己创建一个 Chart**：

```bash
helm create mychart
tree mychart
# mychart/
#   Chart.yaml          元数据
#   values.yaml         默认值
#   templates/          Go template 渲染的 K8s 清单
#   charts/             子 chart 依赖

helm lint mychart
helm template my-release mychart --debug
helm install my-release mychart --dry-run --debug
```

**要点**：

- Helm 3 用 **Secret** 存 release 状态（不是 ConfigMap），`helm list` 读不到时先查 `kubectl -n <ns> get secret -l owner=helm`。
- `helm upgrade --install` 是幂等部署的常用写法（CI 里标准动作）。
- `helm template` 只渲染不发请求，是排查"模板渲染错误"和"字段写错"的首选。
- 生产更推荐 **GitOps（Argo CD / Flux）**：把 chart 或 manifest 放 Git，控制器负责同步，比"人在终端敲 helm upgrade"更可审计。

## 实验 15：故障演练与排查

**目标**：把六种最常见的故障造出来，然后用正确的命令定位。这一节的价值高于前面所有实验。

### 15.1 镜像拉取失败（ImagePullBackOff）

```bash
kubectl run bad-image --image=nginx:not-exist-tag
kubectl get pod bad-image                       # STATUS: ErrImagePull / ImagePullBackOff
kubectl describe pod bad-image | tail -8
# Failed to pull image "nginx:not-exist-tag": ... manifest unknown
```

排查顺序：镜像名/标签是否存在 → 私有仓库是否配 `imagePullSecrets` → 节点能否访问仓库（DNS/网络/代理）→ 仓库限流。

### 15.2 调度失败（Pending）

```bash
kubectl run too-big --image=nginx:1.27 \
  --overrides='{"spec":{"containers":[{"name":"c","image":"nginx:1.27","resources":{"requests":{"cpu":"64"}}}]}}'

kubectl get pod too-big            # Pending
kubectl describe pod too-big | grep -A5 Events
# 0/3 nodes are available: 3 Insufficient cpu.
```

排查顺序：`describe` 里的 Events 会直接告诉你原因——资源不够、节点选择器不匹配、污点不容忍、PVC 不可用、超出配额。

### 15.3 崩溃循环（CrashLoopBackOff）

```bash
kubectl run crash --image=busybox:1.36 -- sh -c 'echo starting; sleep 1; exit 1'
kubectl get pod crash -w           # CrashLoopBackOff，RESTARTS 递增
kubectl logs crash                 # 当前实例日志
kubectl logs crash --previous      # 上一个实例日志（关键！）
kubectl describe pod crash | grep -E "Last State|Exit Code|Reason"
```

排查顺序：应用自身报错（配置/依赖/权限）→ 启动命令写错 → 探针太激进导致误杀 → 内存超限被 OOMKill。

### 15.4 就绪失败（Running 但 0/1）

```bash
kubectl get pod -o wide            # READY 0/1
kubectl describe pod <name> | grep -A3 "Readiness probe failed"
kubectl get endpoints <svc>        # 端点为空 → 流量进不来
```

这是"服务 502/503"最常见的原因：Pod 活着但没准备好，端点被摘除，Service 无后端。

### 15.5 无 shell 镜像的调试

```bash
# 注入一个临时容器到运行中的 Pod（共享进程命名空间）
kubectl debug -it <pod> --image=busybox:1.36 --target=<container-name>

# 复制一个 Pod 并改命令（不影响原 Pod）
kubectl debug <pod> --copy-to=debug-pod --container=app -- sh

# 直接调试节点（会起一个特权 Pod 挂载宿主根文件系统到 /host）
kubectl debug node/kind-lab-worker -it --image=busybox:1.36
```

### 15.6 集群级异常排查顺序

```text
① kubectl get nodes            → 节点是否 Ready，版本是否一致
② kubectl get pods -A          → 哪些系统组件不健康（kube-system 优先）
③ kubectl get events -A --sort-by=.lastTimestamp | tail -40
④ kubectl -n kube-system logs <组件> --tail=100
⑤ 节点上：systemctl status kubelet / journalctl -u kubelet -n 100
⑥ 容器运行时：crictl ps -a / crictl logs <id>
⑦ 网络：kubectl run tmp --image=busybox --rm -it -- nslookup kubernetes
⑧ etcd：etcdctl endpoint health / endpoint status
```

**要点**：

- **事件（Events）默认只保留 1 小时**，且数量上限 1000 条/命名空间。事后排查一定要有事件持久化（如 event-exporter + Loki）。
- 排查永远从"**最小可验证单元**"开始：先确认 apiserver 通不通，再确认对象在不在，再确认控制器动没动，最后才是应用内部。
- 养成习惯：`kubectl describe` 看 Events → `kubectl logs --previous` 看上次日志 → `kubectl get events` 看全局。这三步能解决 80% 的问题。

---

# 第6章 常用命令手册

> 本章按"任务"而非"资源"组织。建议先看 6.1 的约定，之后当字典查。所有命令在 1.28+ 验证。

## 6.1 使用约定与全局参数

```bash
# 命令结构
kubectl <动词> <资源类型> <资源名> [flags]

# 资源类型支持单数/复数/缩写
kubectl get pod / pods / po
kubectl get deployment / deployments / deploy
kubectl get service / services / svc
kubectl get namespace / namespaces / ns
kubectl get persistentvolumeclaim / pvc
```

**全局参数（几乎每个命令都能用）**：

| 参数 | 作用 | 示例 |
|---|---|---|
| `-n, --namespace` | 指定命名空间 | `-n kube-system` |
| `-A, --all-namespaces` | 所有命名空间 | `kubectl get pods -A` |
| `-o, --output` | 输出格式 | `-o wide / yaml / json / name / jsonpath=...` |
| `-l, --selector` | 标签选择器 | `-l app=web,tier=fe` |
| `--field-selector` | 字段选择器 | `--field-selector status.phase=Running` |
| `--sort-by` | 按字段排序 | `--sort-by=.metadata.creationTimestamp` |
| `--show-labels` | 显示标签列 | `kubectl get pods --show-labels` |
| `--context` | 指定上下文（临时切集群） | `--context=prod` |
| `--kubeconfig` | 指定配置文件 | `--kubeconfig=./prod.yaml` |
| `--dry-run=client` | 只生成不发请求 | 配合 `-o yaml` 生成模板 |
| `-w, --watch` | 持续监听变化 | `kubectl get pods -w` |
| `--all` | 所有对象 | `kubectl delete pods --all` |

## 6.2 集群与上下文

```bash
# 集群信息
kubectl cluster-info                     # apiserver / CoreDNS 地址
kubectl cluster-info dump                # 全量诊断信息（提交 issue 用）
kubectl version                          # 客户端 + 服务端版本
kubectl version --client -o yaml

# 节点
kubectl get nodes                        # 状态、角色、版本
kubectl get nodes -o wide                # 加 IP、OS、内核、运行时
kubectl top nodes                        # CPU/内存使用（需 metrics-server）
kubectl describe node <node>             # 已分配资源、污点、事件

# API 资源发现（不知道资源名时用）
kubectl api-resources                    # 所有资源类型 + 缩写 + 是否命名空间级
kubectl api-resources --namespaced=false
kubectl api-resources --api-group=apps
kubectl api-versions                     # 所有可用 API 组版本

# 字段文档（离线写 YAML 的最佳助手）
kubectl explain deployment
kubectl explain deployment.spec.strategy
kubectl explain pod.spec.containers.resources --recursive   # 递归展示全部字段
kubectl explain ingress.spec.rules.http.paths --api-version=networking.k8s.io/v1

# 上下文（多集群）
kubectl config get-contexts
kubectl config current-context
kubectl config use-context kind-lab
kubectl config set-context --current --namespace=dev
kubectl config view --minify --flatten
kubectl config rename-context old-name new-name
kubectl config delete-context name
kubectl config unset users.<name>
```

## 6.3 查询与输出格式化（最提效的一节）

### 6.3.1 常用输出格式

```bash
kubectl get pods                 # 简洁
kubectl get pods -o wide         # 加 IP、节点、镜像
kubectl get pods -o yaml         # 完整对象（调试必用）
kubectl get pods -o json
kubectl get pods -o name         # 只输出类型/名字，便于管道
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
kubectl get pods --no-headers    # 去掉表头，便于脚本处理
```

### 6.3.2 jsonpath 常用模板

```bash
# 所有 Pod 名
kubectl get pods -o jsonpath='{.items[*].metadata.name}'

# 名字 + 节点 + IP，制表符分隔
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.nodeName}{"\t"}{.status.podIP}{"\n"}{end}'

# 单个字段
kubectl get pod web-xxx -o jsonpath='{.status.podIP}'
kubectl get svc web -o jsonpath='{.spec.clusterIP}'

# 取镜像列表（排查版本漂移）
kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.namespace}{"/"}{.metadata.name}{"\t"}{.spec.containers[*].image}{"\n"}{end}'

# 取 Secret 明文
kubectl get secret db -o jsonpath='{.data.password}' | base64 -d; echo

# 取节点内网 IP
kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}'
```

### 6.3.3 custom-columns（更易读）

```bash
kubectl get pods -o custom-columns=\
'NAME:.metadata.name,NODE:.spec.nodeName,IP:.status.podIP,IMAGE:.spec.containers[0].image'

kubectl get pv -o custom-columns=\
'NAME:.metadata.name,CAP:.spec.capacity.storage,SC:.spec.storageClassName,STATUS:.status.phase'

kubectl get pods -A -o custom-columns=\
'NS:.metadata.namespace,NAME:.metadata.name,RESTARTS:.status.containerStatuses[0].restartCount'
```

### 6.3.4 排序与筛选

```bash
# 按创建时间（最新在最后）
kubectl get pods --sort-by=.metadata.creationTimestamp

# 按重启次数（找不稳定 Pod）
kubectl get pods -A --sort-by='.status.containerStatuses[0].restartCount'

# 按字段筛选
kubectl get pods --field-selector status.phase=Running
kubectl get pods --field-selector status.phase!=Running -A
kubectl get pods --field-selector spec.nodeName=node-1
kubectl get events --field-selector type=Warning

# 按标签
kubectl get pods -l app=web -l '!canary'
kubectl get pods -l 'env in (prod,staging)'

# 组合
kubectl get pods -A -l app=web --field-selector status.phase=Running
```

### 6.3.5 实用组合技

```bash
# 找出所有未就绪的 Pod
kubectl get pods -A -o json | jq -r '
  .items[] | select(.status.phase!="Running" or
  ([.status.conditions[]?|select(.type=="Ready")|.status][0] != "True"))
  | "\(.metadata.namespace)/\(.metadata.name) \(.status.phase)"'

# 统计各节点 Pod 数
kubectl get pods -A -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' | sort | uniq -c

# 找出没有设 resources 的 Pod（治理用）
kubectl get pods -A -o json | jq -r '
  .items[] | select(.spec.containers[]|.resources.limits==null)
  | "\(.metadata.namespace)/\(.metadata.name)"'

# 找出使用 latest 标签的镜像
kubectl get pods -A -o jsonpath='{range .items[*]}{.spec.containers[*].image}{"\n"}{end}' \
  | grep -E ':latest$|^[^:]+$' | sort -u
```

## 6.4 创建、更新、删除

### 6.4.1 声明式（推荐）

```bash
kubectl apply -f app.yaml
kubectl apply -f ./manifests/            # 整个目录
kubectl apply -f https://example.com/x.yaml
kubectl apply -k ./overlays/prod         # Kustomize
kubectl apply -f app.yaml --server-side  # 服务端 apply（避免注解过大）
kubectl diff -f app.yaml                 # 先看会改什么（发布前必跑）
kubectl apply -f app.yaml --prune -l app=web   # 删除清单里已不存在的对象
```

### 6.4.2 命令式生成模板（`--dry-run=client -o yaml`）

这是写 YAML 最快的方式：让 kubectl 生成骨架，再改细节。

```bash
# Pod
kubectl run nginx --image=nginx:1.27 --port=80 --dry-run=client -o yaml > pod.yaml
kubectl run tmp --image=busybox:1.36 --rm -it --restart=Never -- sh   # 临时调试 Pod

# Deployment
kubectl create deploy web --image=nginx:1.27 --replicas=3 --dry-run=client -o yaml > deploy.yaml

# Job / CronJob
kubectl create job pi --image=perl:5.38 --dry-run=client -o yaml -- perl -Mbignum=bpi -wle 'print bpi(100)'
kubectl create cronjob hello --image=busybox:1.36 --schedule="*/5 * * * *" -- date --dry-run=client -o yaml

# Service
kubectl expose deploy web --port=80 --target-port=8080 --dry-run=client -o yaml > svc.yaml

# ConfigMap / Secret
kubectl create configmap app --from-literal=K=V --from-file=./conf --dry-run=client -o yaml
kubectl create secret generic db --from-literal=password=x --dry-run=client -o yaml

# Namespace / Quota
kubectl create ns dev --dry-run=client -o yaml
kubectl create quota q --hard=cpu=10,memory=20Gi,pods=20 --dry-run=client -o yaml

# RBAC
kubectl create role r --verb=get,list --resource=pods --dry-run=client -o yaml
kubectl create clusterrole cr --verb=get,list --resource=nodes --dry-run=client -o yaml
kubectl create rolebinding rb --role=r --serviceaccount=dev:sa --dry-run=client -o yaml
```

### 6.4.3 就地修改

```bash
kubectl edit deploy web                  # 打开编辑器改（保存即生效）
kubectl edit svc web -o yaml --save-config

kubectl patch deploy web -p '{"spec":{"replicas":5}}'                     # 合并补丁
kubectl patch deploy web --type=json \
  -p='[{"op":"replace","path":"/spec/template/spec/containers/0/image","value":"nginx:1.27"}]'
kubectl patch svc web -p '{"spec":{"externalTrafficPolicy":"Local"}}'

kubectl label pod web-xxx tier=frontend
kubectl label pod web-xxx tier-                  # 删标签（末尾加 -）
kubectl annotate deploy web kubernetes.io/change-cause="release 2026-11-10"
kubectl annotate deploy web note- --overwrite    # 覆盖已有注解

kubectl set image deploy/web web=nginx:1.27
kubectl set env deploy/web LOG_LEVEL=debug
kubectl set env deploy/web --from=configmap/app-config
kubectl set env deploy/web LOG_LEVEL-            # 删除环境变量
kubectl set resources deploy/web --limits=cpu=1,memory=1Gi
kubectl set serviceaccount deploy/web ci-bot
```

> `kubectl edit` / `patch` **不会**更新 `last-applied-configuration` 注解，之后 `kubectl apply` 可能把这些改动覆盖掉。生产环境建议统一走 apply（GitOps），避免混用。

### 6.4.4 删除

```bash
kubectl delete -f app.yaml
kubectl delete pod web-xxx
kubectl delete pod web-xxx --grace-period=0 --force    # 强制删（仅当卡在 Terminating）
kubectl delete pods -l app=web
kubectl delete pods --all -n dev
kubectl delete deploy,svc,cm -l app=web                # 多类型一起删
kubectl delete ns dev                                  # 删命名空间（会连带删除所有内容）
```

> `--force --grace-period=0` 只是从 etcd 把对象摘掉，**容器可能还在节点上跑**（成为"孤儿容器"）。用过之后要到节点上 `crictl ps` 确认并手动清理。

## 6.5 发布管理（Deployment 为主）

```bash
# 状态与历史
kubectl rollout status deploy/web --timeout=300s
kubectl rollout status deploy/web -w
kubectl rollout history deploy/web
kubectl rollout history deploy/web --revision=2

# 回滚
kubectl rollout undo deploy/web
kubectl rollout undo deploy/web --to-revision=1

# 暂停/恢复（金丝雀中间态）
kubectl rollout pause deploy/web
kubectl rollout resume deploy/web

# 重启（滚动重启，不改任何配置）
kubectl rollout restart deploy/web
kubectl rollout restart deploy -l app=web

# 扩缩容
kubectl scale deploy/web --replicas=10
kubectl scale deploy/web --replicas=10 --current-replicas=3    # 条件式，安全
kubectl scale sts/web --replicas=5
kubectl scale --replicas=0 deploy/web                           # 缩到 0（临时下线）

# 自动扩缩
kubectl autoscale deploy/web --min=2 --max=10 --cpu-percent=60
kubectl get hpa
kubectl describe hpa web

# 等待就绪（脚本里非常有用）
kubectl wait --for=condition=Available deploy/web --timeout=120s
kubectl wait --for=condition=Ready pod -l app=web --timeout=120s
kubectl wait --for=delete pod/old-pod --timeout=60s
```

**一次标准发布流程**：

```bash
kubectl diff -f deploy.yaml                                   # 1. 看变更
kubectl annotate deploy/web kubernetes.io/change-cause="v1.2.0 修复登录超时" --overwrite
kubectl apply -f deploy.yaml                                  # 2. 应用
kubectl rollout status deploy/web --timeout=300s              # 3. 等收敛
kubectl get pods -l app=web -o wide                           # 4. 抽查
# 出问题：
kubectl rollout undo deploy/web                               # 5. 回滚
```

## 6.6 Pod 运维与调试

```bash
# 日志
kubectl logs web-xxx
kubectl logs web-xxx -c sidecar            # 指定容器
kubectl logs -f web-xxx                    # 实时跟随
kubectl logs web-xxx --tail=100
kubectl logs web-xxx --since=1h
kubectl logs web-xxx --since-time='2026-11-10T02:00:00Z'
kubectl logs web-xxx --previous            # 上一个崩溃实例（CrashLoop 必用）
kubectl logs -l app=web --max-log-requests=10 --prefix    # 多 Pod 聚合
kubectl logs deploy/web                    # 自动选一个 Pod

# 进入容器
kubectl exec -it web-xxx -- sh
kubectl exec -it web-xxx -c app -- bash
kubectl exec web-xxx -- cat /etc/config/app.yaml
kubectl exec web-xxx -- env | sort

# 文件拷贝
kubectl cp web-xxx:/var/log/app.log ./app.log
kubectl cp ./config.yaml web-xxx:/etc/app/config.yaml
kubectl cp web-xxx:/data ./backup -c app

# 端口转发
kubectl port-forward pod/web-xxx 8080:80
kubectl port-forward svc/web 8080:80
kubectl port-forward deploy/web 8080:80
kubectl port-forward svc/web 8080:80 --address=0.0.0.0    # 允许外部访问（注意安全）

# 资源用量
kubectl top pod
kubectl top pod --containers
kubectl top pod -A --sort-by=cpu | head -10
kubectl top pod -l app=web --sum                  # 汇总

# 调试
kubectl describe pod web-xxx
kubectl debug -it web-xxx --image=busybox:1.36 --target=app
kubectl debug web-xxx --copy-to=web-debug --container=app -- sh
kubectl debug node/node-1 -it --image=busybox:1.36
kubectl attach -it web-xxx                        # 附着到主进程 stdin/stdout

# 事件
kubectl get events --sort-by=.lastTimestamp
kubectl get events -A --field-selector type=Warning
kubectl get events --field-selector involvedObject.name=web-xxx
```

## 6.7 Service、Ingress 与网络排查

```bash
# Service
kubectl get svc
kubectl get svc -o wide
kubectl describe svc web
kubectl get endpoints web                 # 端点是否为空（第一诊断）
kubectl get endpointslices -l kubernetes.io/service-name=web
kubectl get endpointslices -o wide

# Ingress
kubectl get ingress
kubectl describe ingress web
kubectl get ingressclass

# 网络连通性测试
kubectl run nettest --image=nicolaka/netshoot --rm -it --restart=Never -- bash
# 容器内：
#   nslookup web.default.svc.cluster.local
#   curl -v http://web:80
#   dig +short web
#   traceroute 10.244.2.7
#   tcpdump -i any port 80 -nn

# 只测 DNS
kubectl run dnstest --image=busybox:1.36 --rm -it --restart=Never -- \
  nslookup kubernetes.default

# 查看 CoreDNS 是否正常
kubectl -n kube-system get pods -l k8s-app=kube-dns
kubectl -n kube-system logs -l k8s-app=kube-dns --tail=50

# 走 apiserver 代理访问（绕过 CNI，验证 apiserver 通路）
kubectl get --raw /api/v1/namespaces/default/services/web/proxy/
```

**Service 不通的四步定位法**：

```text
① kubectl get endpoints <svc>          → 空？查 selector 与 Pod readiness
② kubectl get pod -o wide              → Pod 是否 Running 且 Ready
③ kubectl exec 到同 ns 的 Pod 里 curl  → 集群内能否直连 Pod IP
④ kubectl exec ... curl <svc>:<port>   → 能否通过 ClusterIP 访问
   → ②通③不通：CNI / NetworkPolicy 问题
   → ③通④不通：kube-proxy / Service 规则问题
```

## 6.8 配置与密钥

```bash
# ConfigMap
kubectl create configmap app --from-literal=K=V --from-literal=K2=V2
kubectl create configmap app --from-file=./app.conf
kubectl create configmap app --from-file=key1=./a.txt --from-file=key2=./b.txt
kubectl create configmap app --from-env-file=./.env
kubectl get cm
kubectl get cm app -o yaml
kubectl describe cm app
kubectl edit cm app
kubectl delete cm app

# Secret
kubectl create secret generic db --from-literal=password='x'
kubectl create secret generic db --from-file=./id_rsa
kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com --docker-username=u --docker-password=p
kubectl create secret tls example-tls --cert=tls.crt --key=tls.key
kubectl get secret
kubectl get secret db -o jsonpath='{.data.password}' | base64 -d; echo
kubectl get secret db -o go-template='{{range $k,$v := .data}}{{$k}}={{$v|base64decode}}{{"\n"}}{{end}}'

# 更新后让 Pod 感知（环境变量方式必须重启）
kubectl rollout restart deploy/web
```

## 6.9 存储

```bash
kubectl get pv
kubectl get pvc
kubectl get pvc -A
kubectl describe pvc data
kubectl get storageclass
kubectl get sc
kubectl describe sc standard

# 扩容（需 StorageClass 允许）
kubectl patch pvc data -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'

# 排查 PVC 一直 Pending
kubectl describe pvc data
kubectl get events --field-selector involvedObject.name=data

# 确认 Pod 挂了哪些卷
kubectl get pod web-xxx -o jsonpath='{.spec.volumes}'
```

## 6.10 节点与集群维护

```bash
# 标记不可调度（保留已有 Pod）
kubectl cordon node-1
kubectl uncordon node-1

# 驱逐节点上的 Pod（维护前必做）
kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data
kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data --force --grace-period=60
kubectl drain node-1 --dry-run=client          # 先看会发生什么

# 标签
kubectl label node node-1 disktype=ssd
kubectl label node node-1 disktype-            # 删除
kubectl label node node-1 node-role.kubernetes.io/worker=""

# 污点
kubectl taint node node-1 dedicated=gpu:NoSchedule
kubectl taint node node-1 dedicated=gpu:NoSchedule-       # 移除
kubectl taint node node-1 dedicated-                      # 按 key 移除
kubectl describe node node-1 | grep Taints

# 节点状态
kubectl get nodes -o wide
kubectl describe node node-1
kubectl top node

# 集群事件与组件日志
kubectl get componentstatuses           # 已废弃，用下面的替代
kubectl -n kube-system get pods
kubectl -n kube-system logs kube-apiserver-xxx --tail=100
kubectl -n kube-system logs -l component=kube-scheduler --tail=50
```

> drain 常见报错：`cannot delete Pods with local storage`（加 `--delete-emptydir-data`）、`cannot delete DaemonSet-managed Pods`（加 `--ignore-daemonsets`）、`PodDisruptionBudget` 阻止（说明 PDB 在起作用，需要调整副本数或暂时绕过）。

## 6.11 命名空间与配额

```bash
kubectl get ns
kubectl create ns dev
kubectl describe ns dev
kubectl delete ns dev

# 配额与默认限制
kubectl create quota dev-quota -n dev --hard=cpu=20,memory=40Gi,pods=50,services=20
kubectl get resourcequota -n dev
kubectl describe resourcequota dev-quota -n dev     # 看 Used / Hard

cat <<'EOF' | kubectl apply -n dev -f -
apiVersion: v1
kind: LimitRange
metadata:
  name: defaults
spec:
  limits:
    - type: Container
      default:            {cpu: 500m, memory: 512Mi}
      defaultRequest:     {cpu: 100m, memory: 128Mi}
      max:                {cpu: "2",  memory: 2Gi}
EOF

# 排查"配额超了"
kubectl describe resourcequota -n dev
# 事件里会写: exceeded quota: dev-quota, requested: pods=1, used: pods=50, limited: pods=50
```

## 6.12 权限与证书

```bash
# 我能做什么（最常用）
kubectl auth can-i create pods
kubectl auth can-i '*' '*'
kubectl auth can-i --list
kubectl auth can-i --list -n dev
kubectl auth can-i delete pods --as=system:serviceaccount:dev:ci-bot -n dev
kubectl auth can-i get secrets --as=alice@example.com

# ServiceAccount
kubectl create sa ci-bot -n dev
kubectl get sa -n dev
kubectl create token ci-bot -n dev --duration=24h        # 手动签发短期 Token
kubectl describe sa ci-bot -n dev

# 查看谁能访问某资源（逆向审计）
kubectl get rolebindings,clusterrolebindings -A -o json | jq -r '
  .items[] | select(.roleRef.name=="cluster-admin") | "\(.kind) \(.metadata.namespace // "-") \(.metadata.name)"'

# 证书（kubeadm 集群）
kubeadm certs check-expiration
kubeadm certs renew all
kubectl get csr
kubectl certificate approve <csr-name>
kubectl certificate deny <csr-name>
```

## 6.13 排障组合拳（按现象索引）

| 现象 | 首选命令 | 次选 |
|---|---|---|
| Pod 起不来 | `kubectl describe pod <n>` | `kubectl logs <n> --previous` |
| Pod Pending | `kubectl describe pod <n>` 看 Events | `kubectl get nodes`、`describe node` |
| 镜像拉不下来 | `kubectl describe pod` 看 Failed to pull | 检查 `imagePullSecrets`、节点网络 |
| 反复重启 | `kubectl logs <n> --previous` | `describe` 看 Last State / Exit Code |
| 服务 502/503 | `kubectl get endpoints <svc>` | `describe pod` 看 readiness |
| 服务超时 | `kubectl exec` 里 `curl -v` 对比 | 检查 NetworkPolicy、CNI |
| DNS 解析失败 | `kubectl -n kube-system get pod -l k8s-app=kube-dns` | `nslookup` 测试、看 CoreDNS 日志 |
| 配置没生效 | `kubectl exec -- cat <file>` | `env`、`describe` 看挂载 |
| 存储挂不上 | `kubectl describe pvc` / `pod` | 看 Multi-Attach、VolumeZone 事件 |
| 发布卡住 | `kubectl rollout status` | `describe deploy` 看 Conditions |
| 权限被拒 | `kubectl auth can-i <verb> <res> --as=...` | 检查 RoleBinding 的 subject |
| 节点 NotReady | `kubectl describe node` | 节点上 `systemctl status kubelet` |
| 集群整体慢 | `kubectl top nodes/pods` | etcd 延迟、apiserver 指标 |
| Namespace 卡 Terminating | `kubectl get ns <n> -o json` 看 finalizers | 手动清理残留资源 |
| 无法删除 Pod | `describe` 看 finalizer / 节点状态 | `--force --grace-period=0` |

**三个万能起手式**：

```bash
kubectl get events -A --sort-by=.lastTimestamp | tail -40
kubectl get pods -A -o wide | grep -v -E 'Running|Completed'
kubectl describe <资源> <名字> | sed -n '/Events/,$p'
```

## 6.14 效率工具与插件

```bash
# 官方插件管理器
kubectl krew install ctx ns neat tree df-pv resource-capacity

# 必装清单
kubectx / kubens        # 秒切集群与命名空间
k9s                     # 终端 UI，最推荐的日常工具
stern                   # 多 Pod 日志聚合（比 kubectl logs -l 好用）
kubectl-neat            # 去掉 YAML 里的 status/managedFields 噪音
kubectl-tree            # 展示 ownerReference 树状关系
kubectl-df-pv           # 查看 PV 实际使用率
kubectl-resource-capacity  # 节点资源容量与请求量对比
pluto                   # 扫描即将废弃的 API
kube-no-trouble         # 同上
popeye                  # 集群配置体检（发现常见反模式）
velero                  # 集群备份与迁移
```

```bash
# 示例：看对象归属树
kubectl tree deploy web

# 示例：多 Pod 日志
stern -n prod web --since 10m

# 示例：清理 YAML 噪音
kubectl get deploy web -o yaml | kubectl neat | less
```

## 6.15 别名与自动补全

```bash
# bash
source <(kubectl completion bash)
echo 'source <(kubectl completion bash)' >> ~/.bashrc

alias k=kubectl
alias kgp='kubectl get pods'
alias kgpa='kubectl get pods -A -o wide'
alias kgd='kubectl get deploy'
alias kgs='kubectl get svc'
alias kd='kubectl describe'
alias kl='kubectl logs -f'
alias kx='kubectl exec -it'
alias kaf='kubectl apply -f'
alias kdel='kubectl delete'
complete -F __start_kubectl k

# zsh
source <(kubectl completion zsh)
echo 'source <(kubectl completion zsh)' >> ~/.zshrc

# PowerShell
kubectl completion powershell | Out-String | Invoke-Expression
# 永久生效：追加到 $PROFILE
```

## 6.16 etcd 备份、证书与集群升级

```bash
# 备份（在 etcd 节点上执行）
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-$(date +%F-%H%M).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

etcdctl snapshot status /backup/etcd-xxx.db --write-out=table

# 恢复（先停 kube-apiserver 与 etcd）
etcdctl snapshot restore /backup/etcd-xxx.db \
  --data-dir=/var/lib/etcd-restore \
  --initial-cluster-token=etcd-cluster \
  --initial-advertise-peer-urls=https://10.0.0.1:2380 \
  --initial-cluster=master1=https://10.0.0.1:2380

# 健康检查
etcdctl endpoint health --cluster --write-out=table
etcdctl endpoint status --cluster --write-out=table
etcdctl defrag --cluster                      # 回收碎片空间

# kubeadm 集群操作
kubeadm init --pod-network-cidr=10.244.0.0/16 --control-plane-endpoint=lb.example.com:6443
kubeadm token create --print-join-command
kubeadm token list
kubeadm join <endpoint> --token <token> --discovery-token-ca-cert-hash sha256:<hash>

# 升级（控制面）
kubeadm upgrade plan
kubeadm upgrade apply v1.32.0
# 节点（先 drain，再升级 kubelet）
kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data
apt-mark unhold kubeadm kubelet kubectl
apt-get install -y kubeadm=1.32.0-1.1 kubelet=1.32.0-1.1 kubectl=1.32.0-1.1
kubeadm upgrade node
systemctl daemon-reload && systemctl restart kubelet
kubectl uncordon node-1

# 检查废弃 API（升级前必做）
kubectl api-resources --verbs=list --namespaced -o name | \
  xargs -n1 kubectl get --show-kind --ignore-not-found -A
```

## 6.17 一页速查表

```text
【查】
kubectl get <res> [-A] [-o wide|yaml|json|name|jsonpath=] [-l sel] [--sort-by=] [-w]
kubectl describe <res> <name>
kubectl explain <res>.<field>
kubectl api-resources / api-versions
kubectl top node|pod
kubectl get events -A --sort-by=.lastTimestamp

【建】
kubectl apply -f x.yaml | -k dir
kubectl create <res> ... --dry-run=client -o yaml
kubectl run <n> --image= --rm -it --restart=Never -- sh
kubectl expose deploy <n> --port= --target-port= --type=

【改】
kubectl edit <res> <n>
kubectl patch <res> <n> -p '{...}' [--type=json]
kubectl set image|env|resources|serviceaccount <res> <n> ...
kubectl label|annotate <res> <n> k=v
kubectl scale <res> <n> --replicas=N
kubectl rollout restart|status|undo|history|pause|resume deploy/<n>

【删】
kubectl delete <res> <n> | -f x.yaml | -l sel --all
kubectl delete ns <n>
kubectl delete pod <n> --force --grace-period=0

【调试】
kubectl logs <n> [-c c] [-f] [--previous] [--tail=100] [--since=1h]
kubectl exec -it <n> [-c c] -- sh
kubectl port-forward svc/<n> 8080:80
kubectl cp <n>:<path> <local>
kubectl debug -it <n> --image=busybox --target=<c>
kubectl get endpoints <svc>

【权限】
kubectl auth can-i <verb> <res> [--as=sa] [-n ns]
kubectl auth can-i --list

【节点】
kubectl cordon|uncordon <node>
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
kubectl label node <n> k=v
kubectl taint node <n> k=v:NoSchedule [-]

【上下文】
kubectl config get-contexts|use-context|current-context
kubectl config set-context --current --namespace=<ns>
```


# 第7章 YAML 模板库

> 直接复制修改即可。所有模板都省略了 `namespace`，用 `-n` 或 `kubectl apply -n` 指定。

## 7.1 标准 Pod（带探针与安全上下文）

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
  labels:
    app: app
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      image: nginx:1.27
      imagePullPolicy: IfNotPresent
      ports:
        - name: http
          containerPort: 8080
      env:
        - name: TZ
          value: Asia/Shanghai
      resources:
        requests: {cpu: 100m, memory: 128Mi}
        limits:   {cpu: "1",  memory: 512Mi}
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
      startupProbe:
        httpGet: {path: /healthz, port: http}
        failureThreshold: 30
        periodSeconds: 2
      readinessProbe:
        httpGet: {path: /ready, port: http}
        periodSeconds: 5
        failureThreshold: 2
      livenessProbe:
        httpGet: {path: /healthz, port: http}
        periodSeconds: 10
        failureThreshold: 3
      lifecycle:
        preStop:
          exec:
            command: ["sh", "-c", "sleep 5"]
      volumeMounts:
        - name: tmp
          mountPath: /tmp
  terminationGracePeriodSeconds: 30
  volumes:
    - name: tmp
      emptyDir: {}
```

## 7.2 生产级 Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  labels:
    app.kubernetes.io/name: web
    app.kubernetes.io/version: "1.0.0"
spec:
  replicas: 3
  revisionHistoryLimit: 10
  progressDeadlineSeconds: 600
  minReadySeconds: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app.kubernetes.io/name: web
  template:
    metadata:
      labels:
        app.kubernetes.io/name: web
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
    spec:
      serviceAccountName: web
      automountServiceAccountToken: false
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: web
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: web
      containers:
        - name: web
          image: registry.example.com/web:1.0.0
          ports:
            - name: http
              containerPort: 8080
          envFrom:
            - configMapRef:
                name: web-config
            - secretRef:
                name: web-secret
          resources:
            requests: {cpu: 200m, memory: 256Mi}
            limits:   {memory: 512Mi}
          readinessProbe:
            httpGet: {path: /ready, port: http}
            periodSeconds: 5
          livenessProbe:
            httpGet: {path: /healthz, port: http}
            periodSeconds: 10
          lifecycle:
            preStop:
              exec:
                command: ["sh", "-c", "sleep 5"]
```

> 注意这里只设了内存 limit、没设 CPU limit——对延迟敏感的在线服务，这样可以避免 CPU throttling 造成毛刺。是否这样做取决于你的 SLA 与团队共识。

## 7.3 Service + PDB

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  type: ClusterIP
  selector:
    app.kubernetes.io/name: web
  ports:
    - name: http
      port: 80
      targetPort: http
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web
spec:
  minAvailable: 2          # 或 maxUnavailable: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: web
```

## 7.4 Ingress（TLS + 多路径）

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts: ["www.example.com"]
      secretName: example-tls
  rules:
    - host: www.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service: {name: api, port: {number: 80}}
          - path: /
            pathType: Prefix
            backend:
              service: {name: web, port: {number: 80}}
```

## 7.5 ConfigMap + Secret

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-config
data:
  APP_ENV: production
  LOG_LEVEL: info
  app.yaml: |
    server:
      port: 8080
      timeout: 30s
---
apiVersion: v1
kind: Secret
metadata:
  name: web-secret
type: Opaque
stringData:                 # 直接用明文写，apiserver 自动转 base64
  DB_PASSWORD: "P@ssw0rd"
  API_KEY: "sk-xxxx"
```

> `stringData` 只用于输入，读出来时会被合并进 `data`（base64）。写模板时用它更省事，但**提交到 Git 前务必换成 ExternalSecret / SealedSecret**。

## 7.6 有状态服务（StatefulSet + Headless + PVC）

```yaml
apiVersion: v1
kind: Service
metadata:
  name: db-headless
spec:
  clusterIP: None
  selector: {app: db}
  ports:
    - port: 5432
      name: pg
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: db
spec:
  serviceName: db-headless
  replicas: 3
  podManagementPolicy: OrderedReady
  updateStrategy:
    type: RollingUpdate
  selector:
    matchLabels: {app: db}
  template:
    metadata:
      labels: {app: db}
    spec:
      terminationGracePeriodSeconds: 60
      containers:
        - name: postgres
          image: postgres:16
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef: {name: db-secret, key: DB_PASSWORD}
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          resources:
            requests: {cpu: 500m, memory: 1Gi}
            limits:   {memory: 2Gi}
          readinessProbe:
            exec:
              command: ["pg_isready", "-U", "postgres"]
            periodSeconds: 10
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: ssd
        resources:
          requests:
            storage: 50Gi
```

## 7.7 Job / CronJob

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
spec:
  backoffLimit: 3
  activeDeadlineSeconds: 900
  ttlSecondsAfterFinished: 600
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: registry.example.com/app:1.0.0
          command: ["/app/migrate", "--up"]
          resources:
            requests: {cpu: 100m, memory: 128Mi}
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-backup
spec:
  schedule: "0 3 * * *"
  timeZone: "Asia/Shanghai"
  concurrencyPolicy: Forbid
  startingDeadlineSeconds: 600
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      backoffLimit: 2
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: registry.example.com/backup:1.0
              args: ["--target", "s3://bucket/db"]
```

## 7.8 HPA + VPA

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 3
  maxReplicas: 30
  metrics:
    - type: Resource
      resource:
        name: cpu
        target: {type: Utilization, averageUtilization: 60}
    - type: Resource
      resource:
        name: memory
        target: {type: Utilization, averageUtilization: 75}
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - {type: Percent, value: 100, periodSeconds: 15}
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - {type: Pods, value: 2, periodSeconds: 60}
```

## 7.9 NetworkPolicy（默认拒绝 + 白名单）

```yaml
# 第一步：命名空间内默认拒绝所有入向
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}
  policyTypes: ["Ingress"]
---
# 第二步：显式放行
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-to-api
spec:
  podSelector:
    matchLabels: {app: api}
  policyTypes: ["Ingress", "Egress"]
  ingress:
    - from:
        - podSelector:
            matchLabels: {app: web}
      ports:
        - {protocol: TCP, port: 8080}
  egress:
    - to:                       # 放行 DNS，否则解析全挂
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - {protocol: UDP, port: 53}
        - {protocol: TCP, port: 53}
    - to:
        - podSelector:
            matchLabels: {app: db}
      ports:
        - {protocol: TCP, port: 5432}
```

## 7.10 RBAC 三件套

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: deploy-bot
  namespace: prod
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployer
  namespace: prod
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "update", "patch"]
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: deploy-bot-deployer
  namespace: prod
subjects:
  - kind: ServiceAccount
    name: deploy-bot
    namespace: prod
roleRef:
  kind: Role
  name: deployer
  apiGroup: rbac.authorization.k8s.io
```

## 7.11 命名空间治理（Quota + LimitRange + PSA）

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: prod
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/audit: restricted
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: prod-quota
  namespace: prod
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    pods: "100"
    services: "30"
    persistentvolumeclaims: "20"
    count/deployments.apps: "30"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: prod-defaults
  namespace: prod
spec:
  limits:
    - type: Container
      default:        {cpu: 500m, memory: 512Mi}
      defaultRequest: {cpu: 100m, memory: 128Mi}
      max:            {cpu: "4",  memory: 8Gi}
```

## 7.12 Kustomize（多环境管理，kubectl 内置）

```text
base/
  deployment.yaml
  service.yaml
  kustomization.yaml
overlays/
  dev/
    kustomization.yaml
  prod/
    kustomization.yaml
```

```yaml
# overlays/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: prod
resources:
  - ../../base
images:
  - name: registry.example.com/web
    newTag: 1.2.0
replicas:
  - name: web
    count: 6
patches:
  - target:
      kind: Deployment
      name: web
    patch: |-
      - op: replace
        path: /spec/template/spec/containers/0/resources/limits/memory
        value: 1Gi
configMapGenerator:
  - name: web-config
    literals:
      - APP_ENV=production
```

```bash
kubectl kustomize overlays/prod          # 只渲染
kubectl apply -k overlays/prod           # 渲染并应用
kubectl diff -k overlays/prod
```

# 第8章 生产落地检查清单

上线前逐项打勾。这是本文最有价值的一页，建议做成团队 Code Review 清单。

## 8.1 工作负载

- [ ] 没有裸 Pod，全部由 Deployment/StatefulSet/DaemonSet/Job 管理
- [ ] 全部设置了 `resources.requests`；内存 limit 基于压测设定
- [ ] 有 `readinessProbe`；有假死风险的配 `livenessProbe`；慢启动的配 `startupProbe`
- [ ] `livenessProbe` **不**探测下游依赖
- [ ] 配了 `preStop sleep`，`terminationGracePeriodSeconds` 大于关闭所需时间
- [ ] 应用正确处理 SIGTERM（停止接新请求 → 处理存量 → 退出）
- [ ] `replicas` ≥ 2，且用 `topologySpreadConstraints` 跨节点/可用区打散
- [ ] 关键服务配了 `PodDisruptionBudget`
- [ ] `maxUnavailable: 0`（面向用户的服务），`minReadySeconds` > 0
- [ ] 有 `revisionHistoryLimit`，能回滚

## 8.2 镜像与供应链

- [ ] 镜像带明确版本或 digest，**没有 `:latest`**
- [ ] 镜像来自受信仓库，配了 `imagePullSecrets`
- [ ] 镜像扫描无高危漏洞；基础镜像定期更新
- [ ] 多阶段构建，最终镜像不含编译工具链
- [ ] 以非 root 运行，`readOnlyRootFilesystem: true`，`drop: ["ALL"]`
- [ ] 私有仓库凭据不写在镜像里

## 8.3 配置与密钥

- [ ] 配置全部外置到 ConfigMap/Secret，镜像可直接跨环境复用
- [ ] Secret 不落 Git；用 External Secrets / Sealed Secrets / Vault
- [ ] etcd 开启静态加密
- [ ] Secret 的 RBAC 最小化（谁能 `get secrets` 等于拿到全部凭据）
- [ ] 改配置后知道怎么让应用生效（卷热更新 / 滚动重启）

## 8.4 网络与入口

- [ ] Service 的 selector 与 Pod labels 一致（用 `kubectl get endpoints` 验证）
- [ ] 对外入口用 Ingress/Gateway，不直接暴露 NodePort
- [ ] TLS 证书自动续期（cert-manager）
- [ ] 配了 `NetworkPolicy`（至少默认拒绝 + 白名单），并放行 DNS
- [ ] 了解集群的 Pod CIDR / Service CIDR，不与 IDC 网段冲突
- [ ] 有出口流量方案（NAT 网关 / Egress Gateway），固定公网 IP

## 8.5 存储与数据

- [ ] 有状态服务用 StatefulSet + volumeClaimTemplates
- [ ] 关键数据 `reclaimPolicy: Retain`
- [ ] `volumeBindingMode: WaitForFirstConsumer`（云环境）
- [ ] 有备份策略，且**演练过恢复**（没演练过的备份等于没有备份）
- [ ] PVC 容量与性能（IOPS）满足要求；监控容量水位

## 8.6 可观测性

- [ ] 应用暴露 `/metrics`，接入 Prometheus；配了 ServiceMonitor
- [ ] 日志输出到 stdout/stderr（不要写容器内文件），由 agent 采集
- [ ] 关键指标有告警：错误率、P99 延迟、Pod 重启数、Pending Pod、节点磁盘、证书到期
- [ ] 有链路追踪（OpenTelemetry）
- [ ] 事件（Events）有持久化，否则事后无法回溯
- [ ] 有 Dashboard 与值班手册（Runbook）

## 8.7 弹性与容量

- [ ] HPA 的 `minReplicas` 能兜住日常峰值
- [ ] HPA 依赖的指标确实可用（`kubectl top pods` 有数据）
- [ ] 有集群扩容方案（Cluster Autoscaler / 预留节点池）
- [ ] 命名空间有 `ResourceQuota` 与 `LimitRange`，防止单业务吃满集群
- [ ] 节点预留了 `kube-reserved` / `system-reserved`，避免系统组件被业务挤死

## 8.8 安全与合规

- [ ] 命名空间开启 Pod Security Admission（`restricted` 或至少 `baseline`）
- [ ] 应用用独立 ServiceAccount，`automountServiceAccountToken: false`
- [ ] RBAC 最小化，无 `cluster-admin` 滥用；定期审计
- [ ] 开启审计日志（audit log）并集中存储
- [ ] 有准入策略拦截违规配置（Kyverno / Gatekeeper）
- [ ] 控制面不对公网暴露；kubeconfig 按人分发，不用共享凭据

## 8.9 发布与容灾

- [ ] 发布走 GitOps 或流水线，不手工改生产
- [ ] 有灰度/金丝雀能力与回滚预案
- [ ] 升级前用 pluto 扫描废弃 API
- [ ] etcd 定期备份（每日 + 变更前），备份文件异地存放
- [ ] 演练过"节点宕机""可用区故障""etcd 恢复"
- [ ] 文档化了集群拓扑、证书位置、关键联系人

# 第9章 故障排查手册

## 9.1 排查的思维模型

```text
分层假设 —— 从下往上，逐层证伪
① 基础设施：节点 Ready 吗？磁盘满吗？内存压力？
② 控制面：apiserver 通吗？控制器在工作吗？etcd 健康吗？
③ 调度：Pod 被调度了吗？为什么没被调度？
④ 运行时：镜像拉到了吗？容器起来了吗？退出码是什么？
⑤ 应用：进程活着吗？端口对吗？依赖通吗？配置对吗？
⑥ 流量：端点有了吗？规则写了吗？策略拦了吗？
```

三条纪律：

1. **先看 Events，再看日志**。Events 会直接告诉你"为什么"，日志告诉你"发生了什么"。
2. **对比法**。同一个 Deployment 里有一个 Pod 正常、一个不正常，直接 `diff` 两个 Pod 的 YAML。
3. **改一个变量**。不要一次改三处再重启，那样永远不知道是哪一处生效了。

## 9.2 常见故障速查表

| # | 现象 | 根因 | 定位命令 | 解决 |
|---|---|---|---|---|
| 1 | `ImagePullBackOff` | 镜像名/标签错、私有仓库无凭据、网络不通 | `describe pod` 看 Failed to pull | 修正镜像；配 `imagePullSecrets`；检查节点出网 |
| 2 | `ErrImagePull: x509` | 仓库证书不受信 | `describe pod` | 节点导入 CA，或配 `insecure-registries` |
| 3 | `CrashLoopBackOff` | 应用启动即退出 | `logs --previous`、`describe` 看 Exit Code | 修配置/依赖；Exit 137 查 OOM |
| 4 | `OOMKilled` | 内存超 limit | `describe` 看 Last State | 提高 limit 或修内存泄漏；JVM 调 `-Xmx` |
| 5 | `Pending` | 资源不足/亲和性/污点/PVC | `describe pod` 看 Events | 扩容节点、放宽约束、修 PVC |
| 6 | `Pending: volume node affinity conflict` | PVC 的可用区与节点不符 | `describe pvc` / `pod` | 用 `WaitForFirstConsumer`，或换同区节点 |
| 7 | `ContainerCreating` 卡住 | 挂卷失败、CNI 未就绪、镜像大 | `describe pod` | 看具体事件；查 CSI/CNI Pod |
| 8 | `Multi-Attach error` | RWO 卷被另一节点占用 | `describe pod` | 等旧 Pod 彻底终止；换 RWX 存储 |
| 9 | `Running` 但 `0/1` | readiness 失败 | `describe` 看 probe failed | 修探针路径/端口/阈值 |
| 10 | `Terminating` 卡住 | finalizer 未清理、节点失联 | `get pod -o yaml` 看 finalizers | 清理 finalizer；`--force` |
| 11 | Service 访问不通 | 端点为空 | `get endpoints <svc>` | 修 selector 或让 Pod Ready |
| 12 | 集群内 DNS 失败 | CoreDNS 挂/网络策略拦 53 | `get pod -n kube-system -l k8s-app=kube-dns` | 修 CoreDNS；放行 UDP 53 |
| 13 | 502/504 | 后端未就绪或超时 | `get endpoints`、Ingress 日志 | 调探针与超时；扩副本 |
| 14 | 发布卡住不前进 | 新 Pod 起不来 | `rollout status`、`describe deploy` | 修镜像/探针；`rollout undo` |
| 15 | `forbidden` | RBAC 不足 | `auth can-i <verb> <res> --as=...` | 补 Role/RoleBinding |
| 16 | `exceeded quota` | 命名空间配额满 | `describe resourcequota` | 提配额或清理 |
| 17 | 节点 `NotReady` | kubelet 挂/磁盘满/网络断 | `describe node`、`journalctl -u kubelet` | 修 kubelet；清磁盘；重启运行时 |
| 18 | 节点 `DiskPressure` | 镜像/日志占满 | `df -h`、`crictl rmi --prune` | 清理；调 `image-gc` 阈值 |
| 19 | `Namespace` 卡 `Terminating` | 残留资源 finalizer | `get ns <n> -o json` 看 spec.finalizers | 清理残留 APIService/CRD |
| 20 | apiserver 429 | 客户端限流 | `--v=6` 看返回 | 调 QPS/加副本/减少 list 频率 |
| 21 | etcd `no space` | 数据超配额 | `etcdctl endpoint status` | 扩容配额 + defrag + 清理事件 |
| 22 | 时间漂移导致证书错误 | 节点时钟不同步 | `timedatectl` | 配 NTP/chrony |

## 9.3 四个经典案例复盘

**案例一：滚动发布期间出现少量 502。**

现象：每次发布，监控上 502 会有一个小尖峰（约 10 秒），发布完成后自动消失。

排查：`kubectl get endpoints` 显示端点正常。抓包发现有请求打到已经关闭的 Pod。检查 Pod 配置：没有 `preStop`，应用收到 SIGTERM 立刻 `exit`。

根因：删除 Pod 时，**kubelet 发 SIGTERM** 与 **EndpointSlice 摘除** 是并行的，kube-proxy 更新 iptables 还有传播延迟。应用先死了，规则还在转发。

修复：`preStop: sleep 10`，`terminationGracePeriodSeconds: 45`，应用实现优雅关闭。502 消失。

**案例二：Pod 全部 Pending，事件显示 Insufficient cpu。**

现象：扩容后 20 个新 Pod 全部 Pending。

排查：`kubectl describe node` 看 Allocated resources，发现 requests 已经接近 100%。但 `kubectl top node` 显示实际 CPU 只有 30%。

根因：requests 设置过大（每个 Pod 要 4 核，实际只用 0.5 核）。调度看的是 **requests**，不是实际用量。

修复：按 P99 实际用量重设 requests（如 1 核），配合 HPA。这是"集群看起来很闲但调度不上去"的标准原因。

**案例三：Java 服务频繁被 OOMKilled，但堆内存远未到上限。**

现象：容器 limit 2Gi，`-Xmx1500m`，`describe` 显示 `OOMKilled, exit 137`，但 GC 日志显示堆只用了 1G。

根因：JVM 的非堆内存（Metaspace、线程栈、Direct Buffer、JIT code cache）加上堆，超过了 cgroup limit。JVM 看到的"容器可用内存"如果没被正确识别，还会把 `MaxRAMPercentage` 算错。

修复：`-XX:MaxRAMPercentage=70`（让 JVM 按容器 limit 的 70% 设堆），或显式把 `-Xmx` 设为 limit 的 70%，并留出非堆空间。

**案例四：Namespace 删除后一直 Terminating。**

现象：`kubectl delete ns test` 之后，`get ns` 一直显示 `Terminating`，持续几小时。

排查：

```bash
kubectl get ns test -o json | jq '.status, .spec.finalizers'
kubectl api-resources --verbs=list --namespaced -o name | \
  xargs -n1 kubectl get -n test --ignore-not-found 2>/dev/null
```

发现残留了 `metrics.k8s.io/v1beta1` 的 APIService 指向一个已删除的扩展 apiserver。

修复：删掉无效的 APIService 后，Namespace Controller 完成清理。极端情况需要直接改 etcd（`etcdctl del /registry/namespaces/test`），但那属于最后手段，风险极高。

# 附录 A 术语表

| 术语 | 全称 | 含义 |
|---|---|---|
| Pod | — | 最小调度单元，一组共享网络与存储的容器 |
| ReplicaSet | RS | 保证指定数量的 Pod 副本 |
| Deployment | — | 管理 RS，提供滚动更新与回滚 |
| StatefulSet | STS | 有状态工作负载，稳定标识与存储 |
| DaemonSet | DS | 每节点一个 Pod |
| Job / CronJob | — | 一次性 / 定时任务 |
| Service | SVC | 稳定的虚拟 IP 与负载均衡 |
| EndpointSlice | — | Service 的后端 IP 列表 |
| Ingress | — | 七层 HTTP 路由规则 |
| Gateway API | — | Ingress 的演进标准 |
| ConfigMap | CM | 非敏感配置 |
| Secret | — | 敏感配置（base64，非加密） |
| PV | PersistentVolume | 集群级存储资源 |
| PVC | PersistentVolumeClaim | 存储申请单 |
| StorageClass | SC | 存储类型与动态供给模板 |
| CSI | Container Storage Interface | 存储插件标准接口 |
| CNI | Container Network Interface | 网络插件标准接口 |
| CRI | Container Runtime Interface | 运行时标准接口 |
| HPA | HorizontalPodAutoscaler | 水平扩缩容 |
| VPA | VerticalPodAutoscaler | 垂直扩缩容 |
| PDB | PodDisruptionBudget | 自愿中断时的最小可用保证 |
| RBAC | Role-Based Access Control | 基于角色的权限控制 |
| SA | ServiceAccount | Pod 使用的身份 |
| CRD | CustomResourceDefinition | 自定义资源类型 |
| Operator | — | CRD + 控制器，封装运维知识 |
| PSA | Pod Security Admission | Pod 安全标准准入 |
| QoS | Quality of Service | 资源质量等级（Guaranteed/Burstable/BestEffort） |
| Kubelet | — | 节点代理，管理 Pod 生命周期 |
| kube-proxy | — | 实现 Service 转发的组件 |
| etcd | — | 分布式 KV 存储，集群唯一状态源 |
| CIDR | — | 网段表示法，如 10.244.0.0/16 |
| OCI | Open Container Initiative | 容器镜像与运行时标准 |

# 附录 B 学习资源

**官方**

- 文档：https://kubernetes.io/zh-cn/docs/ （中文版质量很好）
- API 参考：https://kubernetes.io/docs/reference/kubernetes-api/
- 任务教程：https://kubernetes.io/docs/tasks/
- 版本发布说明：https://github.com/kubernetes/kubernetes/releases

**规范与标准**

- Gateway API：https://gateway-api.sigs.k8s.io/
- Operator 开发：https://book.kubebuilder.io/
- CNCF 全景图：https://landscape.cncf.io/

**值得读的书**

- 《Kubernetes in Action》（第二版）— 概念讲解最系统
- 《Kubernetes 权威指南》— 中文体系化教材
- 《Production Kubernetes》— 生产落地视角
- 《Designing Distributed Systems》— 分布式模式与 Operator 思想

**练手项目**

- kind / minikube 起本地集群，把本文 15 个实验跑一遍
- 用 Kubebuilder 写一个 Operator（如"自动为带注解的 Deployment 创建 HPA"）
- 用 Argo CD 把本文的 YAML 管起来，体验 GitOps
- 部署一套完整的可观测栈（Prometheus + Grafana + Loki + Tempo）

# 附录 C API 版本与弃用提示

K8s 的 API 有严格的演进策略：`v1alpha1` → `v1beta1` → `v1`（GA）。**GA 后的 API 承诺长期兼容**，beta 通常 3 个版本后废弃或转正，alpha 随时可能删除。

近年重要的移除（升级前必须处理）：

| 已移除 API | 替代 | 移除版本 |
|---|---|---|
| `extensions/v1beta1` Ingress | `networking.k8s.io/v1` | 1.22 |
| `extensions/v1beta1` Deployment | `apps/v1` | 1.16 |
| `policy/v1beta1` PodDisruptionBudget | `policy/v1` | 1.25 |
| `batch/v1beta1` CronJob | `batch/v1` | 1.25 |
| `autoscaling/v2beta1` / `v2beta2` HPA | `autoscaling/v2` | 1.26 |
| `node.k8s.io/v1beta1` RuntimeClass | `node.k8s.io/v1` | 1.25 |
| `flowcontrol.apiserver.k8s.io/v1beta1/v1beta2` | `v1beta3` / `v1` | 1.26–1.29 |
| `dockershim`（内置 Docker 支持） | containerd / CRI-O | 1.24 |
| `v1.Endpoints`（不删除但逐步弃用） | `discovery.k8s.io/v1 EndpointSlice` | 建议迁移 |

**升级前扫描废弃 API**：

```bash
# 用 pluto（推荐）
pluto detect-all-in-cluster

# 用 kubectl 遍历（慢但无依赖）
kubectl api-resources --verbs=list --namespaced -o name | \
  xargs -n1 kubectl get --show-kind --ignore-not-found -A 2>/dev/null | \
  grep -i deprecated

# 用 kube-no-trouble
kubent
```

**几点版本经验**：

- 不要追最新版本。生产建议用"**当前版本 - 1**"，即新版本发布后观察 2–3 个月再升。
- 小版本升级（如 1.31 → 1.32）通常平滑，但**必须读 CHANGELOG 的 "Urgent Upgrade Notes"**。
- 大跨度升级（如 1.28 → 1.32）不能跳版本，要逐个 minor 升（1.28→1.29→1.30→1.31→1.32），因为 etcd 数据格式与 API 转换有版本约束。
- 升级顺序固定：**etcd → 控制面 → kubelet/kube-proxy → 工作负载**。
- 永远先在测试集群演练，尤其是有 CRD 和 Operator 的集群（第三方控制器的兼容性是最常见的坑）。


