---
created: '2026-04-20'
updated: '2026-04-20'
tags:
  - Docker
  - Compose
  - 容器编排
  - 云原生
  - DevOps
aliases:
  - docker-compose
  - Compose
source_type: official-doc
source_urls:
  - 'https://docs.docker.com/compose/'
  - 'https://docs.docker.com/reference/compose-file/'
  - 'https://docs.docker.com/compose/gettingstarted/'
status: verified
---
Docker Compose 是用于定义和运行多容器应用的工具。通过单一 YAML 配置文件管理服务、网络、卷等资源，一条命令即可启动整个应用栈。

## 用途

- **多容器编排**：在单机上管理多个相互依赖的容器
- **开发环境标准化**：团队成员使用相同的 compose.yaml，环境一致性高
- **CI/CD 流程**：在测试流程中快速启动/销毁完整应用栈
- **原型验证**：快速搭建复杂应用原型

适用场景：本地开发、测试、CI、生产单机部署。不适用：大规模分布式部署（应使用 Kubernetes）。

## 前置条件

- 已安装 Docker Engine（20.10+）或 Docker Desktop
- Docker Compose CLI（v2 已集成到 Docker CLI，无需单独安装）

检查安装：

```bash
docker compose version
```

## 核心用法

### compose.yaml 基本结构

```yaml
services:
  web:
    build: .
    ports:
      - "8000:5000"
    environment:
      - REDIS_HOST=redis
    depends_on:
      redis:
        condition: service_healthy

  redis:
    image: redis:alpine
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

volumes:
  redis-data:
```

顶层元素：
- `services`：定义容器服务（必填）
- `volumes`：命名卷声明
- `networks`：自定义网络声明
- `configs`：配置文件（Swarm 模式）
- `secrets`：敏感数据（Swarm 模式）
- `include`：引入其他 compose 文件

### 常用命令

| 命令 | 说明 |
|------|------|
| `docker compose up` | 创建并启动所有服务 |
| `docker compose up -d` | 后台运行 |
| `docker compose down` | 停止并移除容器、网络 |
| `docker compose down -v` | 同时移除命名卷 |
| `docker compose stop` | 停止服务（保留容器） |
| `docker compose start` | 启动已停止的服务 |
| `docker compose restart` | 重启服务 |
| `docker compose ps` | 查看运行状态 |
| `docker compose logs` | 查看日志 |
| `docker compose logs -f` | 实时跟踪日志 |
| `docker compose exec <service> <cmd>` | 在运行容器中执行命令 |
| `docker compose build` | 构建或重建服务镜像 |
| `docker compose pull` | 拉取服务镜像 |
| `docker compose config` | 验证并查看解析后的配置 |

### 项目与命名

Compose 使用项目名隔离不同应用栈：

- 默认项目名：目录名
- 自定义项目名：`docker compose -p myproject up`
- 或通过环境变量：`COMPOSE_PROJECT_NAME=myproject`

资源命名规则：`<project>_<service>_<index>`（如 `myapp_web_1`）

## 服务配置详解

### 镜像与构建

```yaml
services:
  app:
    image: nginx:alpine        # 使用现有镜像
    
  custom:
    build: .                   # 当前目录 Dockerfile
    build:
      context: ./app           # 构建上下文路径
      dockerfile: Dockerfile.dev  # 指定 Dockerfile
      args:                    # 构建参数
        - VERSION=1.0
```

### 端口映射

```yaml
ports:
  - "8080:80"                  # 短格式：主机端口:容器端口
  - "9090:9090/tcp"            # 指定协议
  - target: 80
    published: 8080
    protocol: tcp
    mode: host                 # 完整格式
```

### 环境变量

```yaml
environment:
  - DEBUG=true
  - DB_HOST=db

# 或使用文件
env_file:
  - .env
  - config.env
```

优先级：命令行 `--env` > compose.yaml `environment` > compose.yaml `env_file` > Dockerfile `ENV`

### 卷挂载

```yaml
volumes:
  - redis-data:/data           # 命名卷
  - ./app:/code                # 绑定挂载（相对路径）
  - /host/path:/container/path # 绑定挂载（绝对路径）
```

命名卷需在顶层 `volumes` 声明；绑定挂载无需声明。

### 服务依赖与健康检查

```yaml
services:
  web:
    depends_on:
      db:
        condition: service_healthy  # 等待 db 健康
      redis:
        condition: service_started  # 等待 redis 启动

  db:
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 30s
```

`condition` 类型：
- `service_started`：容器启动即满足（默认）
- `service_healthy`：健康检查通过
- `service_completed_successfully`：容器执行完毕退出（适用于初始化任务）

### 网络

```yaml
services:
  web:
    networks:
      - frontend
      
  db:
    networks:
      - backend

networks:
  frontend:
  backend:
    driver: bridge
```

默认行为：所有服务加入名为 `default` 的桥接网络，服务间可通过服务名通信。

### 资源限制

```yaml
services:
  app:
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
```

注意：单机部署使用 `deploy.resources`；Swarm 模式还支持副本、更新策略等。

## Compose Watch

实时同步代码变更到容器，适用于开发环境：

```yaml
services:
  web:
    develop:
      watch:
        - action: sync+restart
          path: ./src
          target: /app/src
        - action: rebuild
          path: requirements.txt
```

启动：

```bash
docker compose up --watch
```

`action` 类型：
- `sync`：同步文件，不重启
- `sync+restart`：同步后重启容器
- `rebuild`：重新构建镜像

## 多文件合并

使用 `include` 引入其他 compose 文件：

```yaml
# compose.yaml
include:
  - path: ./infra.yaml

services:
  web:
    build: .
```

```yaml
# infra.yaml
services:
  redis:
    image: redis:alpine
```

Compose 启动时自动合并。适用于：
- 模块化复杂应用
- 不同环境配置覆盖（开发/生产）
- 共享基础设施定义

覆盖语法（不使用 include）：

```bash
docker compose -f compose.yaml -f compose.prod.yaml up
```

后一个文件覆盖前一个文件的相同配置项。

## 环境变量文件

Compose 自动读取 `.env` 文件，用于 YAML 变量插值：

```text
# .env
APP_PORT=8000
REDIS_HOST=redis
```

```yaml
services:
  web:
    ports:
      - "${APP_PORT}:5000"
```

验证解析结果：

```bash
docker compose config
```

## 常见问题

### 服务启动顺序

问题：依赖服务未就绪导致主服务启动失败。

解决：使用 `depends_on` + `healthcheck`，或应用层实现重连逻辑。

### 数据丢失

问题：`docker compose down` 后数据丢失。

解决：使用命名卷持久化；`down -v` 会删除卷，慎用。

### 网络隔离

问题：服务间无法通信。

检查：
- 是否在同一网络
- 是否使用服务名作为 hostname
- 是否有端口冲突

### 变量未替换

问题：`${VAR}` 未被替换。

检查：
- `.env` 文件是否存在且在项目目录
- 变量名是否正确
- 使用 `docker compose config` 验证

## 与 Kubernetes 的对比

| 特性 | Docker Compose | Kubernetes |
|------|---------------|------------|
| 部署范围 | 单机 | 多节点集群 |
| 副本管理 | 有限（Swarm 模式） | 完整 |
| 自动扩缩容 | 无 | HPA/VPA |
| 服务发现 | 内置（服务名） | Service + DNS |
| 滚动更新 | Swarm 支持 | Deployment |
| 配置复杂度 | 低 | 高 |

Compose 适合本地开发、测试、小规模部署；Kubernetes 适合生产级分布式系统。

## 最佳实践

1. **使用 .env 文件**：配置与 YAML 分离，便于环境切换
2. **命名卷持久化**：数据库等数据服务必须使用命名卷
3. **健康检查**：依赖服务必须配置 healthcheck
4. **资源限制**：生产环境必须限制 CPU/内存
5. **模块化**：复杂应用使用 include 分拆文件
6. **使用 .dockerignore**：排除不必要文件，加速构建
7. **版本固定**：镜像使用具体版本标签，避免 `latest`

## 参考资料

- [Docker Compose Overview](https://docs.docker.com/compose/) - 官方概述
- [Compose file reference](https://docs.docker.com/reference/compose-file/) - YAML 配置规范
- [Docker Compose Quickstart](https://docs.docker.com/compose/gettingstarted/) - 快速入门教程
- [Compose Specification](https://github.com/compose-spec/compose-spec) - 开源规范仓库
