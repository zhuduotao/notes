---
created: '2026-04-20'
updated: '2026-04-20'
tags:
  - docker-compose
  - kubernetes
  - kompose
  - deployment
  - cloud-native
  - devops
  - container-orchestration
aliases:
  - Compose 转 K8s
  - Docker Compose to Kubernetes
  - Kompose 实践
source_type: official-doc
source_urls:
  - 'https://kompose.io/'
  - 'https://github.com/kubernetes/kompose'
  - 'https://docs.docker.com/desktop/kubernetes/'
  - 'https://compose-spec.io/'
  - >-
    https://kubernetes.io/docs/tasks/configure-pod-container/translate-compose-kubernetes/
status: verified
---
将 Docker Compose 定义的多容器应用快速迁移或部署到 Kubernetes 集群的实践方法。核心场景：开发环境使用 Docker Compose 快速迭代，生产环境使用 Kubernetes 进行规模化运维。

## 适用场景

| 场景 | 说明 |
|------|------|
| **开发到生产迁移** | 已有 Docker Compose 配置，需快速部署到 K8s 集群 |
| **本地 K8s 开发** | Docker Desktop 内置 K8s，实现开发环境与生产环境一致性 |
| **原型验证** | 快速将 compose.yaml 转换为 K8s YAML 进行测试 |
| **学习过渡** | 从 Compose 语法平滑过渡到 K8s 资源定义 |

## 核心工具

### Kompose

Kompose 是 Kubernetes 官方项目，用于将 Compose Specification 文件转换为 Kubernetes 资源清单[^1][^2]。

特点：
- 支持 Compose v1/v2/v3 格式（底层使用 compose-go 解析）
- 输出 Deployment、Service、PVC、ConfigMap、Secret 等 K8s 资源
- 通过 Labels 扩展实现 1:1 精确转换
- 支持 OpenShift provider

安装：

```bash
# macOS (Homebrew)
brew install kompose

# Linux
curl -L https://github.com/kubernetes/kompose/releases/download/v1.38.0/kompose-linux-amd64 -o kompose
chmod +x kompose
sudo mv kompose /usr/local/bin/kompose

# Windows
# 从 GitHub Releases 下载 kompose-windows-amd64.exe
```

基本用法：

```bash
# 转换 compose.yaml 为 K8s YAML
kompose convert -f compose.yaml

# 直接部署到集群
kompose convert -f compose.yaml && kubectl apply -f .

# 指定输出目录
kompose convert -f compose.yaml -o ./k8s-manifests

# 使用不同控制器
kompose convert --controller daemonSet
kompose convert --controller statefulset
```

### Docker Desktop Kubernetes

Docker Desktop 内置独立 Kubernetes 服务器，支持本地开发与测试[^3]。

特性：
- 支持两种集群模式：`kubeadm`（单节点）和 `kind`（多节点）
- 自动安装 `kubectl` 到 `/usr/local/bin/kubectl`
- 与 Docker CLI 集成，共享镜像仓库

启用方式：

1. 打开 Docker Desktop Dashboard → Kubernetes view
2. 选择 Create cluster
3. 选择集群类型（kubeadm 或 kind）
4. 点击 Create

验证：

```bash
kubectl get nodes
kubectl config use-context docker-desktop
```

`kind` 与 `kubeadm` 对比：

| 特性 | kubeadm | kind |
|------|---------|------|
| 多节点集群 | 不支持 | 支持 |
| K8s 版本选择 | 固定 | 可选 |
| 启动速度 | ~1 分钟 | ~30 秒 |
| ECI 支持 | 不支持 | 支持 |

## 实践流程

### 流程一：Kompose 转换部署

适用于已有 compose.yaml，需快速生成 K8s YAML。

```bash
# 1. 准备 compose.yaml
# 2. 转换为 K8s 资源
kompose convert -f compose.yaml

# 3. 检查生成的 YAML（必要时手动调整）
ls *.yaml

# 4. 部署到集群
kubectl apply -f .

# 5. 验证
kubectl get pods,svc
```

### 流程二：Docker Desktop 本地开发

适用于本地开发环境与生产 K8s 环境保持一致。

```bash
# 1. 启用 Docker Desktop K8s
# 2. 构建镜像
docker compose build

# 3. 使用 Kompose 转换
kompose convert -f compose.yaml

# 4. 部署到本地 K8s
kubectl apply -f .

# 5. 测试验证
kubectl get pods
kubectl port-forward svc/web 8080:80
```

### 流程三：CI/CD 集成

在流水线中自动转换并部署。

```yaml
# GitLab CI 示例
deploy:
  stage: deploy
  script:
    - kompose convert -f compose.yaml -o ./k8s
    - kubectl apply -f ./k8s
```

## 转换对照表

Kompose 将 compose.yaml 字段映射为 K8s 资源字段[^4]。

### 支持的转换

| Compose 字段 | K8s 资源 | 说明 |
|-------------|----------|------|
| `image` | `Deployment.Spec.Containers.Image` | 直接映射 |
| `ports` | `Service.Spec.Ports` | 端口映射 |
| `environment` | `Container.Env` | 环境变量 |
| `volumes` | `PersistentVolumeClaim` | 需集群已有 PV 或 StorageClass |
| `command` | `Container.Args` | 容器命令 |
| `entrypoint` | `Container.Command` | 入口点 |
| `cap_add` / `cap_drop` | `SecurityContext.Capabilities` | capabilities |
| `deploy.resources` | `Containers.Resources.Limits` | CPU/内存限制 |
| `deploy.replicas` | `Deployment.Spec.Replicas` | 副本数 |
| `hostname` | `Pod.Spec.Hostname` | 主机名 |
| `domainname` | `Pod.Spec.Subdomain` | 子域名 |
| `stop_grace_period` | `TerminationGracePeriodSeconds` | 终止宽限期 |
| `tmpfs` | `EmptyDir` (medium=Memory) | 临时文件系统 |
| `secrets` | `Secret` | K8s Secret（外部 Secret 不支持） |
| `configs` | `ConfigMap` | 配置映射 |

### 无 1:1 转换的字段

| Compose 字段 | 原因 |
|-------------|------|
| `depends_on` | K8s 使用 Service DNS 发现，不显式依赖 |
| `links` | K8s 集群扁平网络，服务通过 DNS 通信 |
| `network_mode` | K8s 使用自身集群网络（CNI） |
| `external_links` | K8s 无对应概念 |
| `devices` | K8s 不支持设备映射[^5] |
| `dns` / `dns_search` | K8s 使用内置 DNS 服务 |
| `logging` | K8s 节点级别日志处理 |
| `ulimits` | K8s 不支持[^6] |
| `cgroup_parent` | K8s 不支持[^7] |
| `build` | 转换后需手动构建并推送镜像 |

## Labels 扩展

通过 Kompose 特定 Labels 在 compose.yaml 中声明 K8s 特有配置[^4]。

### 控制器类型

```yaml
services:
  web:
    image: nginx
    labels:
      kompose.controller.type: statefulset
```

可选值：`deployment`、`daemonset`、`replicationcontroller`、`statefulset`

### Service 类型

```yaml
services:
  web:
    image: nginx
    ports:
      - "80:80"
    labels:
      kompose.service.type: loadbalancer
```

可选值：`nodeport`、`clusterip`、`loadbalancer`、`headless`

### Ingress 配置

```yaml
services:
  web:
    image: nginx
    ports:
      - "80:80"
    labels:
      kompose.service.expose: "example.com"
      kompose.service.expose.ingress-class-name: "nginx"
      kompose.service.expose.tls-secret: "my-tls-secret"
```

### HPA 自动扩缩容

```yaml
services:
  api:
    image: my-api
    labels:
      kompose.hpa.cpu: "80"
      kompose.hpa.replicas.min: "2"
      kompose.hpa.replicas.max: "10"
```

### 健康检查

```yaml
services:
  web:
    image: nginx
    labels:
      kompose.service.healthcheck.liveness.http_get_path: /health
      kompose.service.healthcheck.liveness.http_get_port: "8080"
      kompose.service.healthcheck.readiness.http_get_path: /ready
      kompose.service.healthcheck.readiness.interval: 10s
```

### 存储配置

```yaml
services:
  db:
    image: postgres
    volumes:
      - db-data:/var/lib/postgresql/data
    labels:
      kompose.volume.size: 1Gi
      kompose.volume.storage-class-name: standard
      kompose.volume.type: persistentVolumeClaim
```

### 多容器 Pod（Sidecar）

```yaml
services:
  nginx:
    image: nginx
    labels:
      kompose.service.group: sidecar
  logs:
    image: busybox
    command: ["tail", "-f", "/var/log/nginx/access.log"]
    labels:
      kompose.service.group: sidecar
```

相同 `kompose.service.group` 的服务会被合并到同一 Pod。

## 重启策略映射

Compose `restart` 字段决定生成的 K8s 对象类型[^4]：

| `restart` 值 | K8s 对象 | `restartPolicy` |
|-------------|----------|-----------------|
| `""`（空） | Deployment/Controller | `Always` |
| `always` | Deployment/Controller | `Always` |
| `unless-stopped` | Deployment/Controller | `Always` |
| `on-failure` | Pod / CronJob | `OnFailure` |
| `no` | Pod / CronJob | `Never` |

一次性任务示例：

```yaml
services:
  pival:
    image: perl
    command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"]
    restart: "on-failure"
```

生成 Pod 而非 Deployment。

CronJob 示例：

```yaml
services:
  pival:
    image: perl
    command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"]
    restart: "no"
    labels:
      kompose.cronjob.schedule: "* * * * *"
      kompose.cronjob.concurrency_policy: "Forbid"
```

## 构建与推送镜像

Compose 文件中的 `build` 字段需额外处理[^4]：

```bash
# 手动构建并推送
docker compose build
docker tag myapp:latest myregistry/myapp:v1
docker push myregistry/myapp:v1

# 使用 Kompose 自动构建推送（需 --build 和 --push-image）
kompose convert -f compose.yaml --build --push-image=true

# 推送到自定义 registry
kompose convert --push-image-registry=myregistry.com
```

认证方式：Kompose 使用 `$DOCKER_CONFIG/config.json` 或 `$HOME/.docker/config.json` 中的 Docker 认证。

macOS 注意事项：若 `credsStore` 使用 `osxkeychain`，推送可能失败。需禁用后重新 `docker login`[^4]。

## 限制与注意事项

### 转换不完整性

- 转换非 1:1，约覆盖 99% 场景[^1]
- `depends_on` 依赖需改用 Service DNS 或 Init Container
- `build` 需手动处理镜像构建与推送
- 带卷的服务会自动使用 `Recreate` 策略而非 `RollingUpdate`

### 命名约束

服务名含 `_` 或 `.` 会被替换为 `-`（如 `web_service` → `web-service`），因为 K8s 不允许对象名含 `_`[^4]。

### 存储前提

PVC 生成需集群已有 PV 或支持动态供给的 StorageClass，否则 Pod 无法启动。

### 网络模型差异

Compose 网络模型与 K8s CNI 不同：
- Compose：每个服务独立网络命名空间，通过服务名访问
- K8s：扁平网络，Pod 通过 Service DNS 访问

### 版本兼容

Kompose 使用 compose-go 库解析，支持最新 Compose Specification[^4]。若需未支持特性，可提交 PR。

### 生产建议

- 转换后的 YAML 需人工审核，补充 K8s 特有配置（如 Resource Requests、SecurityContext）
- 复杂应用建议直接编写 K8s YAML 或使用 Helm Chart
- 多环境部署使用 Kustomize 进行差异化配置

## 常见问题

### 转换后 Service 无法访问

检查：
1. Service 类型是否正确（ClusterIP 仅集群内部可达）
2. 端口映射是否正确
3. Pod 是否正常运行

### PVC Pending 状态

原因：集群无可用 PV 或 StorageClass。

解决：
```bash
kubectl get storageclass
kubectl get pv
```

若无 StorageClass，需创建或使用 `hostPath`（不推荐生产）。

### 依赖服务启动顺序

Compose `depends_on` 不转换。替代方案：

1. Init Container 等待依赖就绪
2. 应用层实现重连逻辑
3. 使用 `kompose.service.healthcheck.readiness.*` Labels

## 相关概念

- [[Docker Compose 完整指南]] — Compose 文件结构与命令详解
- [[Kubernetes (K8s) 介绍与常用组件]] — K8s 核心架构与资源类型
- [[基于 K8s 的服务发现]] — Service DNS 发现机制
- **Compose Specification** — 云原生应用定义标准[^8]
- **Helm** — K8s 包管理器，适合复杂应用
- **Kustomize** — 原生配置定制工具，适合多环境部署

## 参考资料

[^1]: Kompose 官方网站 — https://kompose.io/
[^2]: Kubernetes Kompose GitHub 仓库 — https://github.com/kubernetes/kompose
[^3]: Docker Desktop Kubernetes 文档 — https://docs.docker.com/desktop/kubernetes/
[^4]: Kompose User Guide — https://kompose.io/user-guide/
[^5]: K8s 设备映射不支持 — https://github.com/kubernetes/kubernetes/issues/5607
[^6]: K8s ulimits 不支持 — https://github.com/kubernetes/kubernetes/issues/3595
[^7]: K8s cgroup_parent 不支持 — https://github.com/kubernetes/kubernetes/issues/11986
[^8]: Compose Specification — https://compose-spec.io/
