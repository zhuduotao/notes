---
created: '2026-04-16'
updated: '2026-04-16'
tags:
  - kubernetes
  - k8s
  - container-orchestration
  - cloud-native
  - devops
aliases:
  - K8s
  - Kubernetes 架构
  - Kubernetes 组件
  - 容器编排
source_type: official-doc
source_urls:
  - 'https://kubernetes.io/docs/concepts/overview/'
  - 'https://kubernetes.io/docs/concepts/overview/components/'
  - 'https://kubernetes.io/docs/concepts/workloads/controllers/'
  - 'https://kubernetes.io/docs/concepts/services-networking/'
  - 'https://kubernetes.io/docs/concepts/storage/'
status: verified
---

Kubernetes（简称 K8s）是由 Google 开源的容器编排平台，于 2014 年首次发布，目前由 Cloud Native Computing Foundation (CNCF) 维护[^1]。它提供了一套完整的机制来自动化部署、扩展和管理容器化应用。

## 是什么

Kubernetes 是一个**生产级别的容器编排系统**，用于自动化部署、扩展和运维容器化应用集群。它抽象了底层基础设施（物理机、虚拟机、云资源），让开发者只需关注应用本身，而不必关心运行在哪里。

核心设计目标：

- **声明式配置**：用户描述期望的最终状态（Desired State），K8s 自动将实际状态收敛到期望状态
- **自我修复**：自动重启失败容器、替换不健康节点、按需调度
- **水平扩展**：支持手动和自动扩缩容
- **服务发现与负载均衡**：内置 DNS 和服务代理机制
- **滚动更新与回滚**：零停机部署

## 为什么重要

- **标准化交付**：容器镜像 + YAML 声明 = 可复现的部署单元
- **跨云可移植**：同一套编排逻辑可在 AWS、GCP、Azure、私有云运行
- **生态成熟**：CNCF 景观图中超过 80% 项目与 K8s 集成
- **行业事实标准**：Gartner、Forrester 均将其列为云原生基础设施核心

## 核心架构

K8s 集群由 **Control Plane（控制平面）** 和 **Node（工作节点）** 两部分组成。

### Control Plane 组件

| 组件 | 职责 | 关键说明 |
|------|------|----------|
| **kube-apiserver** | 集群统一入口，处理所有 REST 请求 | 唯一与 etcd 交互的组件；无状态，可水平扩展 |
| **etcd** | 分布式键值存储，保存集群全部状态 | 基于 Raft 共识算法；生产环境建议奇数节点（3/5/7） |
| **kube-scheduler** | 决定 Pod 调度到哪个 Node | 考虑资源请求、亲和性、污点等约束 |
| **kube-controller-manager** | 运行各类控制器，维持期望状态 | 包含 Node、Replication、Endpoint、ServiceAccount 等控制器 |
| **cloud-controller-manager** | 与云提供商 API 交互 | 管理 LoadBalancer、存储卷、路由等云资源（可选） |

### Node 组件

| 组件 | 职责 | 关键说明 |
|------|------|----------|
| **kubelet** | 节点代理，确保容器按 PodSpec 运行 | 与 kube-apiserver 通信；每个 Node 一个实例 |
| **kube-proxy** | 维护网络规则，实现 Service 负载均衡 | 基于 iptables 或 IPVS 模式 |
| **Container Runtime** | 运行容器的底层引擎 | 支持 containerd、CRI-O 等（自 v1.24 起移除 dockershim）[^2] |

### 补充：CRI、CNI、CSI 接口

K8s 通过标准化接口实现可扩展性：

- **CRI (Container Runtime Interface)**：定义容器运行时与 kubelet 的交互协议
- **CNI (Container Network Interface)**：定义容器网络插件标准
- **CSI (Container Storage Interface)**：定义存储插件标准，使第三方存储系统可接入 K8s

## 核心工作负载控制器

K8s 通过 **Controller（控制器）** 管理 Pod 的生命周期。以下是生产中最常用的控制器：

### Deployment

- **用途**：管理无状态应用的声明式更新
- **机制**：底层通过 ReplicaSet 管理 Pod 副本
- **典型场景**：Web 服务、API 后端、微服务
- **关键能力**：滚动更新（RollingUpdate）、回滚（Rollback）、暂停/恢复、扩缩容

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
```

### StatefulSet

- **用途**：管理有状态应用
- **与 Deployment 的区别**：
  - Pod 名称固定且有序（如 `web-0`, `web-1`, `web-2`）
  - 每个 Pod 绑定独立的 PersistentVolume
  - 有序部署、有序终止、有序扩缩容
- **典型场景**：数据库（MySQL、PostgreSQL）、消息队列（Kafka、RabbitMQ）、分布式协调服务（ZooKeeper、etcd）

### DaemonSet

- **用途**：确保每个（或符合条件的）Node 上运行一个 Pod 副本
- **典型场景**：日志采集（Fluentd、Filebeat）、监控代理（Node Exporter）、网络插件（Calico、Cilium）

### Job / CronJob

- **Job**：运行一次性任务，直到成功完成指定次数
- **CronJob**：基于 Cron 表达式定时执行 Job
- **典型场景**：数据迁移、批量计算、定时备份、清理任务

### ReplicaSet

- **用途**：维护固定数量的 Pod 副本
- **注意**：通常不直接使用，而是通过 Deployment 间接管理

## 网络组件

### Service

Service 为一组 Pod 提供稳定的网络入口，解决 Pod IP 动态变化的问题。

| Service 类型 | 说明 | 适用场景 |
|-------------|------|----------|
| **ClusterIP** | 集群内部可访问的虚拟 IP（默认） | 内部服务间通信 |
| **NodePort** | 在每个 Node 上开放固定端口 | 开发/测试环境外部访问 |
| **LoadBalancer** | 通过云提供商创建外部负载均衡器 | 生产环境外部访问 |
| **ExternalName** | 通过 CNAME 记录映射到外部 DNS | 访问外部服务 |

### Ingress

- **用途**：七层（HTTP/HTTPS）路由，将外部流量路由到集群内 Service
- **需要 Ingress Controller**：K8s 本身只提供 Ingress 资源定义，实际路由由 Controller 实现
- **常用 Ingress Controller**：
  - **NGINX Ingress Controller**：最广泛使用，基于 NGINX
  - **Traefik**：云原生边缘路由器，支持动态配置
  - **HAProxy Ingress**：基于 HAProxy，高性能
  - **APISIX Ingress**：基于 Apache APISIX，支持插件化

### Gateway API

- K8s SIG-Network 推出的下一代流量管理标准（v1.0 于 2024 年 GA）[^3]
- 相比 Ingress 提供更细粒度的路由控制、多租户支持和角色分离
- 未来可能逐步替代 Ingress

### CNI 插件（网络插件）

| 插件 | 特点 |
|------|------|
| **Calico** | 基于 BGP，支持 NetworkPolicy，性能优秀 |
| **Cilium** | 基于 eBPF，支持 L7 策略、可观测性，云原生趋势 |
| **Flannel** | 简单 Overlay 网络，适合小型集群 |
| **Antrea** | VMware 开源，基于 OVS，功能丰富 |

## 存储组件

### Volume 类型

| 类型 | 生命周期 | 说明 |
|------|----------|------|
| **emptyDir** | 随 Pod 创建/销毁 | 临时存储，Pod 内容器共享 |
| **hostPath** | 与 Node 绑定 | 挂载 Node 文件系统，不推荐生产使用 |
| **configMap / secret** | 与对象绑定 | 注入配置或敏感信息 |
| **persistentVolumeClaim (PVC)** | 独立于 Pod | 声明式存储请求，与 PV 绑定 |

### PersistentVolume (PV) 与 PersistentVolumeClaim (PVC)

- **PV**：集群级别的存储资源，由管理员预配置或动态供给
- **PVC**：用户侧的存储请求，K8s 自动匹配满足条件的 PV
- **StorageClass**：定义动态供给策略（如自动创建云盘）

### CSI 插件（存储插件）

| 插件 | 支持的存储后端 |
|------|----------------|
| **AWS EBS CSI** | Amazon EBS |
| **GCE PD CSI** | Google Persistent Disk |
| **Azure Disk CSI** | Azure Managed Disk |
| **Ceph CSI** | Ceph RBD / CephFS |
| **NFS CSI** | NFS 文件系统 |
| **Longhorn** | 分布式块存储（Rancher 生态） |

## 常用生态组件

### 包管理

| 工具 | 说明 |
|------|------|
| **Helm** | K8s 包管理器，使用 Chart 模板定义应用 |
| **Kustomize** | 原生配置定制工具（已内置于 kubectl） |

### 可观测性

| 组件 | 用途 |
|------|------|
| **Prometheus** | 指标采集与告警（CNCF 毕业项目） |
| **Grafana** | 指标可视化 |
| **kube-state-metrics** | 暴露 K8s 对象状态指标 |
| **metrics-server** | 提供 CPU/内存指标，支撑 HPA |
| **EFK / ELK** | 日志采集（Elasticsearch + Fluentd/Fluent Bit + Kibana） |
| **Loki** | 轻量级日志聚合（Grafana 生态） |
| **Jaeger / Zipkin** | 分布式追踪 |

### 自动扩缩容

| 组件 | 说明 |
|------|------|
| **HPA (Horizontal Pod Autoscaler)** | 基于 CPU/内存/自定义指标水平扩缩 Pod |
| **VPA (Vertical Pod Autoscaler)** | 自动调整 Pod 的 resource requests/limits |
| **KEDA** | 基于事件驱动的扩缩容（如 Kafka 队列长度、HTTP 请求数） |
| **Cluster Autoscaler** | 自动扩缩 Node 节点数量 |

### 安全与策略

| 组件 | 说明 |
|------|------|
| **Pod Security Standards (PSS)** | 内置安全基线（Privileged / Baseline / Restricted）[^4] |
| **OPA Gatekeeper** | 基于 Open Policy Agent 的策略引擎 |
| **Kyverno** | 原生 K8s 策略引擎，使用 YAML 定义策略 |
| **cert-manager** | 自动化证书管理（如 Let's Encrypt） |
| **Vault** | HashiCorp 密钥管理，支持动态 Secret |

### 服务网格

| 组件 | 说明 |
|------|------|
| **Istio** | 最成熟的服务网格，提供 mTLS、流量管理、可观测性 |
| **Linkerd** | 轻量级服务网格，Rust 实现，性能优秀 |

## 限制与注意事项

### 资源限制

- **单集群规模**：官方设计目标为最多 5000 节点、150000 个 Pod、300000 个容器[^5]
- **etcd 性能**：集群规模越大，etcd 写入延迟越高，建议 SSD + 独立部署
- **API Server 限流**：默认 API 优先级与公平性（APF）机制会限制突发请求

### 常见误区

1. **Pod 不是容器**：Pod 是 K8s 最小调度单元，可包含一个或多个容器（共享网络/存储命名空间）
2. **Deployment 不适用于有状态应用**：有状态场景应使用 StatefulSet
3. **Service 不等于负载均衡器**：ClusterIP 类型仅在集群内部可达
4. **PVC 不会自动扩容**：需要 StorageClass 支持 `allowVolumeExpansion: true`
5. **删除 Deployment 不会删除 PVC**：需手动清理

### 版本兼容性

- K8s 维护最近 3 个小版本的补丁更新（如 v1.33、v1.34、v1.35）[^6]
- API 废弃遵循严格策略：废弃后至少保留 1 年（或 1 个次要版本）
- 生产环境建议锁定版本，避免跨大版本升级

## 相关概念

- **Container**：K8s 调度的基础单元，通常使用 OCI 标准镜像
- **Pod**：K8s 最小部署单元，包含一个或多个紧密耦合的容器
- **Namespace**：逻辑隔离，用于多租户或环境隔离
- **Label / Selector**：K8s 核心关联机制，通过标签匹配资源
- **Operator**：基于 CRD + Controller 的自动化运维模式
- **CRD (Custom Resource Definition)**：扩展 K8s API，定义自定义资源类型

## 参考资料

[^1]: Kubernetes 官方概述 — https://kubernetes.io/docs/concepts/overview/
[^2]: dockershim 移除说明 — https://kubernetes.io/blog/2022/02/17/dockershim-faq/
[^3]: Gateway API v1.0 GA — https://kubernetes.io/blog/2024/04/24/gateway-api-ga-v1/
[^4]: Pod Security Standards — https://kubernetes.io/docs/concepts/security/pod-security-standards/
[^5]: 集群规模限制 — https://kubernetes.io/docs/setup/best-practices/cluster-large/
[^6]: K8s 版本策略 — https://kubernetes.io/releases/version-skew-policy/
