---
created: 2026-04-17
updated: 2026-04-17
tags:
  - kubernetes
  - k8s
  - service-discovery
  - dns
  - coreDNS
  - endpoint-slice
  - cloud-native
  - networking
aliases:
  - K8s Service Discovery
  - Kubernetes 服务发现
  - K8s DNS 服务发现
  - CoreDNS 服务发现
source_type: official-doc
source_urls:
  - https://kubernetes.io/docs/concepts/services-networking/service/
  - https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/
  - https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/
  - https://kubernetes.io/docs/tasks/administer-cluster/coredns/
status: verified
---

## 是什么

基于 K8s 的服务发现（Service Discovery）是指 Kubernetes 集群内，Pod 之间自动发现和相互通信的机制。它解决了**Pod 生命周期短暂、IP 动态变化**带来的寻址问题，使客户端无需关心后端 Pod 的实际位置。

K8s 提供两层服务发现机制：

| 机制 | 说明 | 推荐程度 |
|------|------|----------|
| **DNS** | 通过 CoreDNS 将 Service 名称解析为 ClusterIP 或 Pod IP 列表 | 推荐，生产首选 |
| **环境变量** | kubelet 在 Pod 启动时注入 `{SVCNAME}_SERVICE_HOST` 和 `{SVCNAME}_SERVICE_PORT` | 仅适用于简单场景 |

## 为什么重要

在传统架构中，服务通过固定 IP 或 DNS 记录定位。但在 K8s 中：

- Pod 随时可能被调度、驱逐或重新创建，每次 IP 都会变化
- 水平扩缩容会动态增减后端实例
- 滚动更新期间新旧版本 Pod 共存

Service 抽象层 + DNS 的组合提供了**稳定的访问入口**，使微服务架构下的服务间调用变得可靠且可维护。

## 核心机制

### Service 抽象层

Service 是一组 Pod 的逻辑集合，通过 **Label Selector** 匹配后端 Pod。它为客户端提供稳定的网络入口，屏蔽了后端 Pod 的动态变化。

#### Service 类型

| 类型 | 说明 | 适用场景 |
|------|------|----------|
| **ClusterIP**（默认） | 分配集群内部虚拟 IP，仅集群内可达 | 服务间内部通信 |
| **NodePort** | 在每个 Node 上开放固定端口（默认范围 30000-32767） | 开发/测试环境外部访问 |
| **LoadBalancer** | 通过云提供商创建外部负载均衡器 | 生产环境对外暴露服务 |
| **ExternalName** | 通过 CNAME 记录映射到外部 DNS 名称 | 访问集群外部服务 |

#### Service 定义示例

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  namespace: default
spec:
  selector:
    app: my-app
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 8080
  type: ClusterIP
```

关键字段说明：
- `selector`：通过标签匹配后端 Pod，控制器持续扫描并更新匹配结果
- `port`：Service 暴露的端口
- `targetPort`：后端 Pod 实际监听的端口，可以引用命名端口（如 `http-web-svc`）
- 多端口 Service 必须为每个端口指定 `name`

#### Headless Service

设置 `.spec.clusterIP: "None"` 的 Service，特点：

- 不分配 ClusterIP，kube-proxy 不处理，不做负载均衡
- **有 selector**：EndpointSlice 控制器创建 EndpointSlice，DNS 返回 A/AAAA 记录直接指向 Pod IP
- **无 selector**：DNS 为 ExternalName 创建 CNAME，或为 endpoint IP 创建 A/AAAA 记录

典型用途：StatefulSet 需要客户端直接连接具体 Pod 实例（如数据库集群、ZooKeeper）。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: headless-service
spec:
  clusterIP: "None"
  selector:
    app: stateful-app
  ports:
    - port: 80
```

### DNS 服务发现

K8s 集群默认使用 **CoreDNS** 作为 DNS 服务。CoreDNS 监听 API Server，自动为 Service 和 Pod 创建 DNS 记录。

#### Service DNS 命名规则

| Service 类型 | A/AAAA 记录格式 | 解析结果 |
|-------------|----------------|----------|
| **普通 Service** | `my-svc.my-namespace.svc.cluster.local` | 解析为 ClusterIP |
| **Headless Service** | `my-svc.my-namespace.svc.cluster.local` | 解析为 Pod IP 集合（客户端自行选择或轮询） |

#### SRV 记录

为**命名端口**的 Service 创建 SRV 记录，格式：

```
_port-name._port-protocol.my-svc.my-namespace.svc.cluster.local
```

- **普通 Service**：解析为端口号 + 域名
- **Headless Service**：返回多个答案（每个 Pod 一条），包含端口号 + Pod 域名

SRV 记录常用于需要同时获取端口和地址的场景，如 gRPC、数据库连接。

#### Pod DNS 记录

| 记录类型 | 格式 | 示例 |
|---------|------|------|
| **A 记录（CoreDNS 服务级）** | `<pod-ipv4-address>.<service-name>.<namespace>.svc.<cluster-domain>` | `172-17-0-3.barista.cafe.svc.cluster.local` |
| **A 记录（传统 Kube-DNS）** | `<pod-ipv4-address>.<namespace>.pod.<cluster-domain>` | `172-17-0-3.default.pod.cluster.local` |

Pod 的 hostname 默认为 `metadata.name`，可通过 `spec.hostname` 覆盖。配合 `spec.subdomain` 可形成完整 FQDN：

```
hostname.subdomain.namespace.svc.cluster.local
```

当存在与 subdomain 同名的 Headless Service 时，DNS 会返回该 Pod FQDN 的 A/AAAA 记录。

> **注意**：Pod 必须处于 Ready 状态才会有 DNS 记录，除非 Service 设置了 `publishNotReadyAddresses: true`。

#### Pod 的 dnsPolicy 选项

| 策略 | 行为 |
|------|------|
| **Default** | 继承所在节点的 DNS 配置 |
| **ClusterFirst**（默认） | 集群域名走 CoreDNS，非集群域名转发上游 DNS |
| **ClusterFirstWithHostNet** | 适用于 `hostNetwork: true` 的 Pod；Windows 不支持 |
| **None** | 忽略 K8s DNS 设置，完全由 `dnsConfig` 指定 |

#### Pod 的 /etc/resolv.conf 示例

```
nameserver 10.32.0.10
search <namespace>.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

- `ndots:5`：名称中点的数量 ≥ 5 时视为 FQDN 直接查询，否则先尝试 search 域拼接
- search 域上限：最多 32 个，总长度不超过 2048 字符（v1.28 稳定）[^1]

#### Pod DNS Config（v1.14 稳定）

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dns-example
spec:
  dnsPolicy: "None"
  dnsConfig:
    nameservers:
      - 8.8.8.8
    searches:
      - ns1.svc.cluster-domain.example
      - my.dns.search.suffix
    options:
      - name: ndots
        value: "2"
      - name: edns0
```

- `dnsConfig` 可与任意 `dnsPolicy` 配合使用；当 `dnsPolicy: None` 时必须提供
- `nameservers` 最多 3 个 IP

### EndpointSlice

EndpointSlice 是 K8s v1.21 稳定的 API，用于跟踪 Service 后端 endpoint（Pod IP）列表，**替代了已废弃的 Endpoints API**（v1.33 废弃）[^2]。

#### 为什么需要 EndpointSlice

旧版 Endpoints API 的局限：
- 不支持双栈（Dual-Stack）
- 缺少新特性所需的信息（如 `trafficDistribution`）
- 单个对象最多容纳 1000 个 endpoint，超出后截断

EndpointSlice 的改进：
- 每个 Slice 默认最多 **100 个 endpoint**（可通过 `--max-endpoints-per-slice` 调整，上限 1000）
- 一个 Service 可以有**多个 EndpointSlice**，无总数限制
- kube-proxy 在每个节点上 watch EndpointSlice，减少大规模更新频率

#### EndpointSlice 示例

```yaml
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: example-abc
  labels:
    kubernetes.io/service-name: example
addressType: IPv4
ports:
  - name: http
    protocol: TCP
    port: 80
endpoints:
  - addresses:
      - "10.1.2.3"
    conditions:
      ready: true
    hostname: pod-1
    nodeName: node-1
    zone: us-west2-a
```

#### Endpoint Conditions（v1.26 稳定）

| 条件 | 含义 |
|------|------|
| **serving** | endpoint 正在提供服务，对应 Pod 的 Ready 条件 |
| **terminating** | endpoint 正在终止（Pod 有删除时间戳）。代理通常忽略，但当所有 endpoint 都在 terminating 时会路由（防止滚动更新期间流量丢失） |
| **ready** | 等价于 "serving 且非 terminating"。当 Service 设置 `publishNotReadyAddresses: true` 时始终为 true |

#### 拓扑信息

EndpointSlice 包含 `nodeName`、`zone` 等字段，用于流量分发策略：

- `PreferSameZone`：优先路由到同 zone 的 endpoint
- `PreferSameNode`：优先路由到同 Node 的 endpoint

> **注意**：endpoint 在过渡期间可能出现在多个 Slice 中，客户端必须**去重**。

## 服务发现的两种方式

### 环境变量方式

kubelet 在 Pod 启动时会为当前已存在的 Service 注入环境变量：

```
MY_SERVICE_SERVICE_HOST=10.32.0.10
MY_SERVICE_SERVICE_PORT=80
MY_SERVICE_PORT=tcp://10.32.0.10:80
MY_SERVICE_PORT_80_TCP=tcp://10.32.0.10:80
MY_SERVICE_PORT_80_TCP_PROTO=tcp
MY_SERVICE_PORT_80_TCP_PORT=80
MY_SERVICE_PORT_80_TCP_ADDR=10.32.0.10
```

**关键限制**：Service 必须在客户端 Pod **之前创建**。如果 Pod 先启动，环境变量不会包含后创建的 Service。因此生产环境推荐使用 DNS 方式。

### DNS 方式（推荐）

CoreDNS 持续监听 API Server 变化，自动更新 DNS 记录。客户端 Pod 只需通过 Service 名称（如 `my-service` 或 `my-service.default.svc.cluster.local`）即可解析到目标地址，不受创建顺序影响。

## 流量策略

### Internal Traffic Policy

`.spec.internalTrafficPolicy` 控制集群内部流量的路由行为：

| 值 | 行为 |
|----|------|
| **Cluster**（默认） | 流量可路由到集群内任意 endpoint |
| **Local** | 仅路由到与客户端同 Node 的 endpoint |

### External Traffic Policy

`.spec.externalTrafficPolicy` 控制外部流量的路由行为：

| 值 | 行为 |
|----|------|
| **Cluster**（默认） | 流量可路由到任意 endpoint，可能产生跨 Node 转发（额外一跳） |
| **Local** | 仅路由到接收流量的 Node 上的 endpoint，保留客户端源 IP |

### Traffic Distribution（v1.35）

`.spec.trafficDistribution` 提供更细粒度的分发偏好：

- `PreferSameZone`：优先同 zone
- `PreferSameNode`：优先同 Node
- `PreferClose`：优先就近（已废弃别名）

## 限制与注意事项

### 命名限制

- Service 名称必须是有效的 **RFC 1035 label**（字母开头，仅含字母数字和连字符）
- v1.34 alpha 引入 `RelaxedServiceNameValidation`，允许数字开头（RFC 1123）[^3]

### Endpoint IP 限制

以下 IP 不允许作为 endpoint：
- 回环地址：`127.0.0.0/8`、`::1/128`
- 链路本地地址：`169.254.0.0/16`、`224.0.0.0/24`、`fe80::/64`
- 其他 Service 的 ClusterIP

### DNS 搜索域限制

- 最多 32 个搜索域，总长度不超过 2048 字符
- 已知兼容性问题：containerd ≤ v1.5.5、CRI-O ≤ v1.21 可能导致 Pod 卡在 Pending 状态[^1]

### Windows 节点 DNS 注意事项

- `ClusterFirstWithHostNet` 不支持 Windows
- Windows 将含 `.` 的名称视为 FQDN，跳过 FQDN 解析
- 仅支持 1 个 DNS 后缀（namespace 后缀），不支持列表
- 可解析 FQDN 和短名称，但**无法解析部分限定名称**（如 `kubernetes.default` 不生效）

### Hostname 长度限制

`setHostnameAsFQDN`（v1.22 稳定）使 kubelet 将 FQDN 写入 Pod hostname。Linux 内核 hostname 限制为 **64 字符**，超长 FQDN 会导致 Pod 无法启动[^4]。

### LoadBalancer 相关

- `.spec.loadBalancerIP` 已于 v1.24 废弃[^5]
- `.spec.allocateLoadBalancerNodePorts`（v1.24 稳定）：设为 `false` 可禁止为 LoadBalancer 分配 NodePort
- `.spec.loadBalancerClass`（v1.24 稳定）：使用非默认负载均衡器实现
- `MixedProtocolLBService`（v1.26 稳定）：允许 LoadBalancer 端口使用不同协议

## 最佳实践

1. **始终使用 DNS 方式**进行服务发现，避免环境变量方式的时序问题
2. **为端口命名**：多端口 Service 必须命名，且命名端口可触发 SRV 记录创建
3. **Headless Service 用于有状态应用**：StatefulSet 配合 Headless Service 实现 Pod 级别的直接访问
4. **合理设置 dnsPolicy**：大多数场景使用默认的 `ClusterFirst`；需要自定义上游 DNS 时使用 `dnsConfig`
5. **监控 EndpointSlice 规模**：大规模集群关注 EndpointSlice 数量和更新频率，必要时调整 `--max-endpoints-per-slice`
6. **使用 Traffic Policy 优化延迟**：跨 zone/跨 Node 场景下设置 `internalTrafficPolicy: Local` 减少网络跳数
7. **注意 ndots 配置**：默认 `ndots:5` 可能导致短名称多次 DNS 查询，高延迟场景可考虑调低

## 相关概念

- **Service**：K8s 网络抽象层，提供稳定的访问入口
- **CoreDNS**：K8s 默认 DNS 服务，负责 Service 和 Pod 的域名解析
- **EndpointSlice**：记录 Service 后端 endpoint 列表的 API 对象
- **kube-proxy**：运行在每个 Node 上的网络代理，基于 iptables 或 IPVS 实现 Service 负载均衡
- **Ingress / Gateway API**：七层流量路由，处理外部到集群的 HTTP/HTTPS 流量

## 参考资料

[^1]: DNS 搜索域限制 — https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#dns-search-domain-list
[^2]: Endpoints API 废弃说明 — https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/
[^3]: Service 名称验证 — https://kubernetes.io/docs/concepts/services-networking/service/#relaxed-service-name-validation
[^4]: Pod hostname 与 FQDN — https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#pod-sethostnameasfqdn-field
[^5]: loadBalancerIP 废弃 — https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer-deprecated
