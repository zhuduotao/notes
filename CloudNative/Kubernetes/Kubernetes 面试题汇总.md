---
created: "2026-04-20"
updated: "2026-04-20"
tags:
  - Kubernetes
  - K8s
  - 云原生
  - 面试
  - 容器编排
aliases:
  - K8s 面试题
  - Kubernetes 面试
source_type: official-doc
source_urls:
  - https://kubernetes.io/docs/concepts/overview/what-is-kubernetes/
  - https://kubernetes.io/docs/concepts/architecture/
  - https://kubernetes.io/docs/concepts/workloads/pods/
  - https://kubernetes.io/docs/concepts/services-networking/service/
  - https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
  - https://kubernetes.io/docs/concepts/overview/components/
status: verified
---

本文汇总 Kubernetes 高频面试问题，涵盖核心概念、架构原理、常用组件、调度机制、网络模型、存储方案、安全机制等考点。

## 一、核心概念与架构

### 1. Kubernetes 是什么？解决了什么问题？

Kubernetes（K8s）是 Google 开源的容器编排平台，用于自动化容器化应用的部署、扩展和管理。

**解决的核心问题**（来源：[官方文档](https://kubernetes.io/docs/concepts/overview/what-is-kubernetes/)）：
- **服务发现与负载均衡**：自动发现服务并分发流量
- **存储编排**：自动挂载所需存储系统
- **自动部署和回滚**：声明式更新，支持自动回滚
- **自动装箱**：根据资源需求和约束调度容器
- **自修复**：重启失败容器、替换节点、健康检查
- **配置与密钥管理**：管理配置和敏感信息

### 2. Kubernetes 架构组件有哪些？

Kubernetes 采用 Master-Worker 架构，分为 Control Plane（控制平面）和 Worker Node（工作节点）。

**Control Plane 组件**（来源：[官方文档](https://kubernetes.io/docs/concepts/overview/components/)）：

| 组件 | 功能 |
|------|------|
| kube-apiserver | API 入口，所有操作唯一入口，认证、授权、准入控制 |
| etcd | 分布式键值存储，保存集群所有状态数据 |
| kube-scheduler | 调度器，根据资源需求、策略、约束将 Pod 分配到节点 |
| kube-controller-manager | 运行控制器循环，维护集群期望状态 |
| cloud-controller-manager | 云平台特定控制器（可选） |

**Worker Node 组件**：

| 组件 | 功能 |
|------|------|
| kubelet | 节点代理，确保 Pod 容器运行，与 API Server 通信 |
| kube-proxy | 网络代理，维护 Service 规则，实现负载均衡 |
| Container Runtime | 容器运行时（containerd、CRI-O 等），执行容器 |

### 3. etcd 在 Kubernetes 中的作用是什么？

etcd 是 Kubernetes 的核心数据存储：

- 存储所有集群状态和配置数据
- 提供强一致性保证（Raft 协议）
- 支持 watch 机制，控制器可监听状态变化
- 生产环境建议 3-5 节点集群，确保高可用

**注意事项**：etcd 数据丢失会导致集群状态丢失，需定期备份。

### 4. kube-apiserver 的作用？

kube-apiserver 是 Kubernetes 的统一入口：

- 提供 RESTful API
- 处理认证、授权、准入控制
- 是唯一直接与 etcd 交互的组件
- 其他组件通过 API Server 间接访问数据

---

## 二、Pod 相关问题

### 5. Pod 是什么？为什么需要 Pod？

Pod 是 Kubernetes 最小的可部署单元，包含一个或多个容器。

**设计原因**（来源：[官方文档](https://kubernetes.io/docs/concepts/workloads/pods/)）：
- **共享网络**：Pod 内容器共享网络栈，可通过 localhost 通信
- **共享存储**：Pod 内容器可共享 Volume
- **协同调度**：紧密协作的容器必须同时调度到同一节点
- **统一生命周期**：Pod 内容器同时创建、销毁

### 6. Pod 的生命周期阶段有哪些？

| Phase | 说明 |
|-------|------|
| Pending | Pod 已创建，等待调度或镜像拉取 |
| Running | Pod 已调度，至少一个容器运行 |
| Succeeded | 所有容器成功终止（Job 类 Pod） |
| Failed | 所有容器终止，至少一个失败 |
| Unknown | 状态无法获取（通常网络问题） |

### 7. 什么是 Init Container？与普通容器有什么区别？

Init Container（初始化容器）在 Pod 主容器启动前执行，用于初始化任务。

**特点**：
- 必须成功完成才能启动主容器
- 按顺序执行，一个失败则 Pod 失败
- 不支持 readiness/liveness probe
- 可访问主容器无法访问的 Secret/Volume

**用途**：
- 等待依赖服务就绪
- 数据库迁移、配置初始化
- 拉取配置文件到共享 Volume

### 8. Pod 的 QoS 类别有哪些？

来源：[官方文档](https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/)

| QoS Class | 条件 | 说明 |
|-----------|------|------|
| Guaranteed | 所有容器设置 limits 和 requests 且相等 | 最高优先级，资源保证 |
| Burstable | 至少一个容器设置 requests 或 limits | 中等优先级 |
| BestEffort | 无 requests/limits 设置 | 最低优先级，资源压力大时优先驱逐 |

---

## 三、控制器与工作负载

### 9. Deployment、ReplicaSet、Pod 三者的关系？

```
Deployment → ReplicaSet → Pod
```

- **Deployment**：声明式管理，支持滚动更新、回滚
- **ReplicaSet**：维护 Pod 副本数量
- **Pod**：实际运行的容器实例

**Deployment 更新流程**（来源：[官方文档](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)）：
1. Deployment 创建 ReplicaSet
2. ReplicaSet 根据 selector 创建/删除 Pod
3. 更新 Deployment 时，创建新 ReplicaSet，逐步替换旧 Pod

### 10. Deployment 的滚动更新策略？

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # 最多可超出期望副本数
      maxUnavailable: 1  # 最多不可用副本数
```

**原理**：逐步用新 Pod 替换旧 Pod，确保服务持续可用。

### 11. StatefulSet 与 Deployment 的区别？

来源：[官方文档](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)

| 特性 | Deployment | StatefulSet |
|------|-----------|-------------|
| Pod 名称 | 随机哈希 | 固定序号（web-0, web-1） |
| 存储 | 共享 PVC | 每个 Pod 独立 PVC |
| 启动顺序 | 并行 | 有序（0→1→2） |
| 网络标识 | Service 负载均衡 | Headless Service + 固定域名 |
| 适用场景 | 无状态应用 | 有状态应用（数据库、集群） |

### 12. DaemonSet 的用途？

DaemonSet 确保每个节点运行一个 Pod 实例。

**典型用途**：
- 日志采集
- 监控代理
- 网络插件（CNI）
- 存储插件

### 13. Job 和 CronJob 的区别？

- **Job**：一次性任务，执行完成后终止
- **CronJob**：定时任务，按 Cron 表达式周期执行

```yaml
# Job 示例
spec:
  completions: 1    # 完成 1 次
  parallelism: 1    # 并行 1 个 Pod
```

---

## 四、Service 与网络

### 14. Service 的四种类型？

来源：[官方文档](https://kubernetes.io/docs/concepts/services-networking/service/)

| 类型 | 说明 | 适用场景 |
|------|------|---------|
| ClusterIP | 内部虚拟 IP，仅集群内可访问 | 内部服务 |
| NodePort | 在节点端口暴露服务 | 外部临时访问 |
| LoadBalancer | 云厂商负载均衡器 | 生产环境对外暴露 |
| ExternalName | 映射到外部 DNS 名称 | 外部服务引用 |

### 15. Service 如何实现负载均衡？

- **kube-proxy**：在每个节点运行，维护 Service 规则
- **iptables/IPVS 模式**：设置转发规则，将流量分发到后端 Pod
- **轮询策略**：默认轮询分发请求

### 16. 什么是 Headless Service？

Headless Service（`clusterIP: None`）不分配 ClusterIP，直接返回 Pod IP。

**用途**：
- StatefulSet 中为每个 Pod 提供固定域名
- 客户端直接连接特定 Pod
- 服务发现场景

### 17. Ingress 是什么？与 Service 的关系？

Ingress 是七层负载均衡，基于域名和路径路由到不同 Service。

```yaml
# Ingress 示例
rules:
- host: api.example.com
  http:
    paths:
    - path: /v1
      backend:
        service:
          name: api-v1
          port: 80
```

**对比 Service**：
- Service：四层负载均衡（TCP/UDP）
- Ingress：七层负载均衡（HTTP/HTTPS），支持域名、路径、TLS

### 18. Kubernetes 网络模型要求？

来源：[官方文档](https://kubernetes.io/docs/concepts/cluster-administration/networking/)

**三大原则**：
1. 所有 Pod 不使用 NAT 可直接通信
2. 所有 Node 不使用 NAT 可与所有 Pod 通信
3. Pod 看到的 IP 与其他 Pod/Node 看到的 IP 相同

**实现方案**：CNI 插件（Calico、Flannel、Cilium 等）。

---

## 五、存储机制

### 19. PV、PVC、StorageClass 的关系？

```
StorageClass → PV → PVC → Pod
```

| 对象 | 作用 |
|------|------|
| PV（PersistentVolume） | 存储资源，集群级别 |
| PVC（PersistentVolumeClaim） | 存储需求声明，命名空间级别 |
| StorageClass | 存储类型定义，支持动态 provisioning |

### 20. PV 的访问模式？

| Access Mode | 说明 |
|-------------|------|
| ReadWriteOnce (RWO) | 单节点读写 |
| ReadOnlyMany (ROX) | 多节点只读 |
| ReadWriteMany (RWX) | 多节点读写（需 NFS 等支持） |
| ReadWriteOncePod (RWOP) | 单 Pod 读写 |

### 21. PV 的回收策略？

| Reclaim Policy | 说明 |
|----------------|------|
| Retain | 保留数据，需手动清理 |
| Delete | 删除 PV 和底层存储 |
| Recycle | 已废弃，清空数据后重新可用 |

### 22. 什么是动态 provisioning？

通过 StorageClass 自动创建 PV：

1. 用户创建 PVC，指定 StorageClass
2. Provisioner 检测 PVC 请求
3. 自动创建底层存储和 PV
4. PV 与 PVC 绑定

---

## 六、调度机制

### 23. kube-scheduler 的调度流程？

1. **过滤**：排除不满足条件的节点
2. **评分**：对剩余节点打分
3. **绑定**：选择最高分节点，绑定 Pod

**过滤条件**：
- 资源充足（CPU、内存）
- NodeSelector/NodeAffinity 匹配
- Taint/Toleration 兼容
- Volume 绑定限制

### 24. Node Affinity 与 NodeSelector 的区别？

| 特性 | NodeSelector | NodeAffinity |
|------|-------------|--------------|
| 匹配方式 | 简单 label 匹配 | 支持 In/NotIn/Exists 等操作符 |
| 约束强度 | 硬约束 | 支持 required（硬）和 preferred（软） |
| 功能 | 基础 | 更灵活 |

### 25. Taint 和 Toleration 的作用？

- **Taint（污点）**：标记节点，排斥特定 Pod
- **Toleration（容忍）**：Pod 配置，允许调度到有污点的节点

**用途**：
- 专用节点（GPU 节点）
- 节点维护
- 节点故障隔离

```yaml
# 节点添加污点
kubectl taint nodes node1 key=value:NoSchedule
```

**污点效果**：
- `NoSchedule`：不调度新 Pod
- `NoExecute`：驱逐已有 Pod
- `PreferNoSchedule`：尽量不调度

### 26. Pod Priority 和 Preemption？

高优先级 Pod 资源不足时，可抢占低优先级 Pod：

- PriorityClass 定义优先级数值
- Preemption 策略决定抢占行为

---

## 七、安全与认证

### 27. Kubernetes 认证方式有哪些？

来源：[官方文档](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)

| 方式 | 说明 |
|------|------|
| X509 证书 | 客户端证书认证 |
| Static Token File | 静态 token 文件 |
| Bootstrap Tokens | 节点加入集群引导 |
| Service Account | Pod 内部访问 API |
| OIDC | 外部身份提供商集成 |
| Webhook | 外部认证服务 |

### 28. RBAC 是什么？

RBAC（Role-Based Access Control）基于角色的权限控制：

- **Role/ClusterRole**：定义权限规则
- **RoleBinding/ClusterRoleBinding**：绑定角色到用户/组/ServiceAccount

```yaml
# Role 示例
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
```

### 29. Service Account 与 User Account 的区别？

| 特性 | User Account | Service Account |
|------|--------------|-----------------|
| 对象 | 外部用户 | Pod 内部进程 |
| 范围 | 全集群 | 命名空间 |
| 用途 | 人类操作 | 自动化操作 |
| 创建方式 | 外部管理 | kubectl/API |

### 30. Pod Security Standards 是什么？

来源：[官方文档](https://kubernetes.io/docs/concepts/security/pod-security-standards/)

三个策略级别：
- **Privileged**：无限制，特权 Pod
- **Baseline**：最小限制，禁止危险配置
- **Restricted**：严格限制，安全最佳实践

**替代 PSP**：Pod Security Policy 已在 v1.25 弃用，改为 Pod Security Admission。

---

## 八、配置与管理

### 31. ConfigMap 和 Secret 的区别？

| 对象 | 用途 | 存储 |
|------|------|------|
| ConfigMap | 配置数据（明文） | 配置文件、环境变量 |
| Secret | 敏感数据（编码） | 密码、证书、Token |

**Secret 类型**：
- `Opaque`：通用密钥
- `kubernetes.io/dockerconfigjson`：镜像仓库凭证
- `kubernetes.io/service-account-token`：SA token
- `kubernetes.io/tls`：TLS 证书

### 32. Probe（探针）类型有哪些？

来源：[官方文档](https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/)

| Probe | 作用 |
|-------|------|
| livenessProbe | 容器存活检查，失败则重启 |
| readinessProbe | 就绪检查，失败则从 Service 移除 |
| startupProbe | 启动检查，成功后才进行 liveness/readiness |

**探针方式**：
- `exec`：执行命令
- `httpGet`：HTTP 请求
- `tcpSocket`：TCP 连接
- `grpc`：GRPC 健康检查

### 33. Namespace 的作用？

- 资源隔离（逻辑分区）
- 权限边界（RBAC 作用域）
- 资配限制（ResourceQuota、LimitRange）

### 34. ResourceQuota 和 LimitRange 的区别？

| 对象 | 作用 |
|------|------|
| ResourceQuota | 限制命名空间资源总量 |
| LimitRange | 限制单个 Pod/Container 资源范围，设置默认值 |

---

## 九、运维与调试

### 35. 如何排查 Pod 一直 Pending？

排查步骤：
1. `kubectl describe pod <pod-name>` 查看事件
2. 检查资源是否充足
3. 检查 NodeSelector/Affinity 是否匹配
4. 检查 Taint/Toleration 是否兼容
5. 检查 PVC 是否绑定

### 36. Pod 状态 CrashLoopBackOff 原因？

- 容器进程启动后立即退出
- livenessProbe 配置错误
- 应用依赖未就绪
- 资源不足（OOM）

**排查**：查看容器日志 `kubectl logs <pod> --previous`。

### 37. 如何安全驱逐节点 Pod？

```bash
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
```

- DaemonSet Pod 默认忽略
- emptyDir 数据需确认删除
- PodDisruptionBudget 控制最小可用副本

### 38. kubectl 常用命令？

```bash
# 资源查看
kubectl get pods/deployments/services -n <namespace>
kubectl describe pod <pod-name>

# 日志查看
kubectl logs <pod> [-c container] [--previous]

# 资源操作
kubectl apply -f manifest.yaml
kubectl delete -f manifest.yaml
kubectl edit deployment <name>

# 调试
kubectl exec -it <pod> -- sh
kubectl port-forward svc/<service> 8080:80
```

---

## 十、高级话题

### 39. Operator 是什么？

Operator 是使用 CRD（Custom Resource Definition）封装运维知识的控制器：

- 定义自定义资源
- 实现控制器逻辑
- 自动化管理复杂应用（如数据库集群）

### 40. 什么是 CRI、CNI、CSI？

| 接口 | 作用 |
|------|------|
| CRI (Container Runtime Interface) | 容器运行时接口 |
| CNI (Container Network Interface) | 网络插件接口 |
| CSI (Container Storage Interface) | 存储插件接口 |

**意义**：解耦 Kubernetes 核心与具体实现，支持多种插件。

### 41. HPA 和 VPA 的区别？

| 方式 | 说明 |
|------|------|
| HPA (Horizontal Pod Autoscaler) | 根据 CPU/内存等指标增减 Pod 副本数 |
| VPA (Vertical Pod Autoscaler) | 调整 Pod requests/limits |

### 42. 如何实现集群高可用？

- **Control Plane HA**：多 Master 节点，etcd 集群
- **Worker HA**：多工作节点，Pod 多副本
- **负载均衡**：API Server 前置 LB
- **备份策略**：定期备份 etcd

---

## 参考资料

- [Kubernetes 官方文档](https://kubernetes.io/docs/home/)
- [Kubernetes Components](https://kubernetes.io/docs/concepts/overview/components/)
- [Pods](https://kubernetes.io/docs/concepts/workloads/pods/)
- [Service](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)