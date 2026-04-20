---
created: '2026-04-20'
updated: '2026-04-20'
tags:
  - Docker
  - Dockerfile
  - 容器
  - 云原生
  - 镜像构建
aliases:
  - Dockerfile指南
  - Dockerfile语法
  - Dockerfile最佳实践
source_type: official-doc
source_urls:
  - 'https://docs.docker.com/engine/reference/builder/'
  - 'https://docs.docker.com/develop/develop-images/dockerfile_best-practices/'
status: verified
---
Dockerfile 是一个文本文件，包含构建 Docker 镜像所需的所有指令。Docker 通过读取 Dockerfile 中的指令自动构建镜像，每条指令创建一个镜像层。

## 基本格式

```dockerfile
# 注释
INSTRUCTION arguments
```

指令不区分大小写，但约定使用大写以区分指令与参数。Dockerfile **必须以 `FROM` 指令开头**（允许在 `FROM` 前有解析器指令、注释和全局 `ARG`）[^1]。

## 解析器指令

解析器指令位于 Dockerfile 顶部，以 `# directive=value` 格式编写，不产生构建层。

### syntax

指定 Dockerfile 语法版本：

```dockerfile
# syntax=docker/dockerfile:1
```

使用最新稳定语法版本，无需升级 BuildKit 或 Docker Engine 即可获得新特性。

### escape

设置转义字符，默认为 `\`。Windows 环境推荐使用 `` ` ``：

```dockerfile
# escape=`
```

### check

配置构建检查行为[^1]：

```dockerfile
# check=skip=JSONArgsRecommended,StageNameCasing
# check=error=true
# check=skip=all
```

## Shell 与 Exec 形式

`RUN`、`CMD`、`ENTRYPOINT` 支持两种形式[^1]：

| 形式 | 格式 | 特点 |
|------|------|------|
| Exec 形式 | `["executable","param1","param2"]` | JSON 数组，不调用 shell，变量替换需手动处理 |
| Shell 形式 | `command param1 param2` | 自动调用默认 shell，支持变量替换、管道 |

Exec 形式注意事项：
- 必须使用双引号 `"`
- 变量替换需显式调用 shell：`RUN ["sh", "-c", "echo $HOME"]`
- Windows 路径需转义反斜杠：`RUN ["c:\\windows\\system32\\tasklist.exe"]`

Shell 形式支持 heredocs：

```dockerfile
RUN <<EOF
apt-get update
apt-get install -y curl
EOF
```

## 核心指令

### FROM

初始化构建阶段并设置基础镜像[^1]：

```dockerfile
FROM [--platform=<platform>] <image> [AS <name>]
FROM [--platform=<platform>] <image>[:<tag>] [AS <name>]
FROM [--platform=<platform>] <image>[@<digest>] [AS <name>]
```

- `AS <name>`：命名构建阶段，供后续 `FROM`、`COPY --from`、`RUN --mount=from` 引用
- `--platform`：指定多平台镜像的目标平台（如 `linux/amd64`）
- 未指定 `tag` 或 `digest` 时默认使用 `latest`

`FROM` 前只能有 `ARG` 指令：

```dockerfile
ARG VERSION=latest
FROM busybox:$VERSION
ARG VERSION
RUN echo $VERSION > image_version
```

### RUN

执行命令并创建新镜像层[^1]：

```dockerfile
RUN <command>                  # Shell 录式
RUN ["executable", "param1"]   # Exec 录式
```

常用选项：

| 选项 | 说明 |
|------|------|
| `--mount` | 创建文件系统挂载（bind、cache、tmpfs、secret、ssh） |
| `--network` | 网络环境（default、none、host） |
| `--security` | 安全模式（sandbox、insecure） |

**挂载类型**[^1]：

```dockerfile
# 缓存包管理器目录
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    apt-get update && apt-get install -y gcc

# 访问构建密钥
RUN --mount=type=secret,id=aws,target=/root/.aws/credentials \
    aws s3 cp s3://...

# SSH 密钥访问
RUN --mount=type=ssh \
    ssh -q -T git@gitlab.com
```

**管道命令**[^2]：

```dockerfile
# 确保管道任意阶段失败即中断构建
RUN set -o pipefail && wget -O - https://some.site | wc -l > /number
```

### CMD

设置容器启动时的默认命令[^1]：

```dockerfile
CMD ["executable","param1","param2"]   # Exec 录式（推荐）
CMD ["param1","param2"]                # 作为 ENTRYPOINT 的默认参数
CMD command param1 param2              # Shell 录式
```

- 一个 Dockerfile 只能有一个 `CMD`，多个时仅最后一个生效
- `docker run` 的参数会覆盖 `CMD` 但保留 `ENTRYPOINT`
- `CMD` 不在构建时执行，仅指定镜像的默认启动命令

### ENTRYPOINT

设置镜像的主命令[^1]：

```dockerfile
ENTRYPOINT ["executable","param1","param2"]  # Exec 录式（推荐）
ENTRYPOINT command param1 param2             # Shell 录式
```

最佳实践：用 `ENTRYPOINT` 设置主命令，用 `CMD` 设置默认参数[^2]：

```dockerfile
ENTRYPOINT ["s3cmd"]
CMD ["--help"]
```

```console
$ docker run s3cmd              # 显示帮助
$ docker run s3cmd ls s3://...  # 执行 ls 命令
```

### COPY 与 ADD

复制文件到镜像[^1]：

```dockerfile
COPY [--chown=<user>:<group>] <src>... <dest>
COPY [--from=<stage>] <src>... <dest>

ADD [--chown=<user>:<group>] [--checksum=<sum>] <src>... <dest>
```

| 指令 | 特性 |
|------|------|
| `COPY` | 从构建上下文或阶段复制文件 |
| `ADD` | 支持远程 URL、自动解压 tar、Git URL、checksum 验证 |

推荐[^2]：
- 本地文件复制用 `COPY`
- 远程文件下载或 tar 解压用 `ADD`
- 临时文件可用 `RUN --mount=type=bind` 替代 `COPY`（不写入镜像层）

### ENV

设置环境变量[^1]：

```dockerfile
ENV <key>=<value> [<key>=<value>...]
```

环境变量在构建阶段和运行时都有效，支持以下指令中的变量替换[^1]：

`ADD`、`COPY`、`ENV`、`EXPOSE`、`FROM`、`LABEL`、`STOPSIGNAL`、`USER`、`VOLUME`、`WORKDIR`

变量语法：

```dockerfile
ENV FOO=/bar
WORKDIR ${FOO}          # WORKDIR /bar
COPY \$FOO /quux        # 复制字面量 $FOO
```

支持 bash 修饰符：
- `${variable:-word}`：变量未设置或为空时使用 word
- `${variable:+word}`：变量设置且非空时使用 word

**注意**[^2]：`ENV` 创建的层即使后续 unset 变量，值仍可被导出。应在单条 `RUN` 中设置、使用、unset：

```dockerfile
RUN export ADMIN_USER="mark" \
    && echo $ADMIN_USER > ./mark \
    && unset ADMIN_USER
```

### ARG

定义构建时变量[^1]：

```dockerfile
ARG <name>[=<default value>]
```

- `ARG` 变量仅构建时有效，不写入最终镜像
- `docker build --build-arg <name>=<value>` 可覆盖默认值

内置平台 ARG[^1]：

| 变量 | 说明 |
|------|------|
| `TARGETPLATFORM` | 目标平台（如 linux/amd64） |
| `BUILDPLATFORM` | 构建平台 |
| `TARGETARCH` | 目标架构 |
| `BUILDARCH` | 构建架构 |

### WORKDIR

设置工作目录[^1]：

```dockerfile
WORKDIR /path/to/workdir
```

- 未指定时默认 `/`
- 多次使用会累积：`WORKDIR /a` + `WORKDIR b` = `/a/b`
- 支持环境变量：`WORKDIR ${PATH}`

推荐[^2]：使用绝对路径，避免 `RUN cd … && do-something`。

### EXPOSE

声明容器监听端口[^1]：

```dockerfile
EXPOSE <port> [<port>/<protocol>...]
```

- 默认 TCP，可指定 UDP：`EXPOSE 80/udp`
- 不实际发布端口，仅作文档说明
- 运行时用 `-p` 或 `-P` 映射端口

### VOLUME

创建匿名卷挂载点[^1]：

```dockerfile
VOLUME ["/data"]
```

用于持久化数据库存储、配置文件等可变数据[^2]。

### USER

设置运行用户和组[^1]：

```dockerfile
USER <user>[:<group>]
USER <UID>[:<GID>]
```

推荐[^2]：
- 服务无需特权时使用非 root 用户
- 创建用户时指定显式 UID/GID
- 使用 `gosu` 替代 `sudo`
- 避免 `USER` 频繁切换

```dockerfile
RUN groupadd -r postgres && useradd --no-log-init -r -g postgres postgres
USER postgres
```

### HEALTHCHECK

定义健康检查[^1]：

```dockerfile
HEALTHCHECK [OPTIONS] CMD command
HEALTHCHECK NONE
```

选项：
- `--interval=<duration>`：检查间隔（默认 30s）
- `--timeout=<duration>`：超时时间（默认 30s）
- `--start-period=<duration>`：启动等待（默认 0s）
- `--retries=<N>`：连续失败次数（默认 3）

### LABEL

添加镜像元数据[^1]：

```dockerfile
LABEL <key>=<value> [<key>=<value>...]
```

查看标签：

```console
$ docker image inspect --format='{{json .Config.Labels}}' myimage
```

### ONBUILD

设置子镜像触发指令[^1]：

```dockerfile
ONBUILD <INSTRUCTION>
```

当其他镜像 `FROM` 当前镜像时，`ONBUILD` 指令在子镜像构建前执行。适用于语言栈镜像[^2]。

### SHELL

设置默认 shell[^1]：

```dockerfile
SHELL ["executable", "parameters"]
```

Windows 默认 `["cmd", "/S", "/C"]`，Linux 默认 `["/bin/sh", "-c"]`。

### STOPSIGNAL

设置停止信号[^1]：

```dockerfile
STOPSIGNAL <signal>
```

发送给容器以停止的 syscall 信号，如 `SIGTERM`（15）或 `SIGKILL`（9）。

## 多阶段构建

使用多个 `FROM` 创建多个构建阶段[^2]：

```dockerfile
FROM golang:1.21 AS builder
WORKDIR /app
COPY . .
RUN go build -o myapp

FROM alpine:3.21
COPY --from=builder /app/myapp /usr/local/bin/
CMD ["myapp"]
```

优势[^2]：
- 最终镜像仅包含运行所需文件，体积更小
- 构建工具（编译器）不进入生产镜像
- 阶段可并行构建，效率更高

创建可复用基阶段：

```dockerfile
FROM ubuntu AS base
RUN apt-get update && apt-get install -y shared-tooling

FROM base AS dev
RUN apt-get install -y dev-tooling

FROM base AS prod
COPY --from=build /app /app
```

## 构建缓存

Docker 按顺序执行指令，每条指令检查是否可复用缓存[^1]：

- `RUN`、`COPY`、`ADD` 指令的缓存基于指令内容和构建上下文
- 缓存失效：指令内容变化、构建上下文文件变化、父层缓存失效
- `ADD` 和 `COPY` 的缓存基于文件内容和元数据

强制刷新：

```console
$ docker build --no-cache -t myimage .
$ docker build --pull -t myimage .      # 拉取最新基础镜像
```

## .dockerignore

排除构建上下文中的文件[^1]，类似 `.gitignore`：

```plaintext
*.md
.git
node_modules
```

减少构建上下文大小，避免敏感文件进入镜像。

## 最佳实践

### 镜像选择[^2]

- 使用 Docker Official Images 或 Verified Publisher 镜像
- 选择最小基础镜像（如 `alpine`）
- 构建镜像用完整版本，生产镜像用 slim 版本

### 版本固定[^2]

```dockerfile
# Tag 方式（获取最新 patch 版本）
FROM alpine:3.21

# Digest 方式（完全固定）
FROM alpine:3.21@sha256:a8560b36e8b8210634f77d9f7f9efd7ffa463e380b75e2e74aff4511df3ef88c
```

### apt-get 使用[^2]

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends \
    package-bar \
    package-baz \
    package-foo=1.3.* \
    && rm -rf /var/lib/apt/lists/*
```

- `update` 与 `install` 在同一 `RUN` 中执行
- 使用 `--no-install-recommends` 减少安装
- 清理 `/var/lib/apt/lists/*` 减小镜像体积
- 版本固定避免缓存问题

### 减少层数[^2]

合并相关指令：

```dockerfile
RUN apt-get update && apt-get install -y \
    package-bar \
    package-baz \
    && rm -rf /var/lib/apt/lists/*
```

### 参数排序[^2]

多行参数按字母排序，便于维护：

```dockerfile
RUN apt-get install -y --no-install-recommends \
    bzr \
    cvs \
    git \
    mercurial \
    subversion
```

### 临时容器原则[^2]

容器应可随时停止、销毁、重建，配置最小化。遵循 Twelve-Factor App 原则。

### 单一职责[^2]

每个容器只处理一个关注点，便于横向扩展和复用。

## 常见误区

### CMD 与 ENTRYPOINT 混淆

- `CMD`：默认命令，可被 `docker run` 参数完全覆盖
- `ENTRYPOINT`：主命令，`docker run` 参数追加到 `ENTRYPOINT` 后

### COPY 与 ADD 误用

- 不要用 `ADD` 复制本地文件（`ADD` 会自动解压 tar）
- 不要用 `COPY` 下载远程文件（`COPY` 不支持 URL）

### ENV 持久化问题

`ENV` 设置的变量即使后续 unset，仍可通过 `docker inspect` 或 `docker run --env` 查看[^2]。

### 管道命令失败不中断

默认仅检查管道最后命令的退出码[^2]：

```dockerfile
RUN wget -O - https://some.site | wc -l > /number
```

`wget` 失败但 `wc -l` 成功时构建继续。解决方案：

```dockerfile
RUN set -o pipefail && wget -O - https://some.site | wc -l > /number
```

## 参考资料

[^1]: Dockerfile Reference - https://docs.docker.com/engine/reference/builder/
[^2]: Dockerfile Best Practices - https://docs.docker.com/develop/develop-images/dockerfile_best-practices/
