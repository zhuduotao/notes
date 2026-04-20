---
created: '2026-04-20'
updated: '2026-04-20'
tags:
  - Docker
  - 容器
  - 云原生
  - Linux
  - Namespace
  - Cgroups
  - OverlayFS
aliases:
  - Docker原理
  - 容器原理
source_type: official-doc
source_urls:
  - 'https://docs.docker.com/get-started/overview/'
  - 'https://docs.docker.com/engine/'
  - 'https://docs.docker.com/storage/storagedriver/overlayfs-driver/'
  - 'https://man7.org/linux/man-pages/man7/namespaces.7.html'
  - 'https://www.kernel.org/doc/Documentation/cgroup-v1/cgroups.txt'
status: verified
---

Docker 是一个开源的容器化平台，用于开发、交付和运行应用程序。它通过 Linux 内核特性（Namespace、Cgroups）实现资源隔离与限制，通过 UnionFS（如 OverlayFS）实现高效的镜像分层存储。

## 架构概述

Docker 采用客户端-服务器架构[^1]：

| 组件 | 说明 |
|------|------|
| Docker Daemon (`dockerd`) | 服务端，监听 Docker API 请求，管理镜像、容器、网络、卷等对象 |
| Docker Client (`docker`) | 客户端，通过 REST API 与 daemon 通信，执行 `docker run`、`docker build` 等命令 |
| Docker Registry | 存储 Docker 镜像的仓库，如 Docker Hub、私有仓库 |

通信方式：REST API，通过 UNIX socket 或网络接口。

### Docker Desktop

Docker Desktop 是集成的桌面应用（Mac/Windows/Linux），包含：
- Docker daemon、client、Docker Compose
- Docker Content Trust、Kubernetes、Credential Helper

## 底层技术

Docker 使用 Go 语言编写[^1]，核心依赖两个 Linux 内核特性：

### Namespace（命名空间）

Namespace 将全局系统资源抽象化，使进程认为自己拥有独立的资源实例[^2]。Docker 容器启动时会创建以下 namespace：

| Namespace 类型 | Flag | 隔离内容 |
|----------------|------|----------|
| PID | `CLONE_NEWPID` | 进程 ID |
| Network | `CLONE_NEWNET` | 网络设备、协议栈、端口 |
| Mount | `CLONE_NEWNS` | 挂载点 |
| UTS | `CLONE_NEWUTS` | 主机名、NIS 域名 |
| IPC | `CLONE_NEWIPC` | System V IPC、POSIX 消息队列 |
| User | `CLONE_NEWUSER` | 用户 ID、组 ID |
| Cgroup | `CLONE_NEWCGROUP` | Cgroup 根目录 |
| Time | `CLONE_NEWTIME` | 时钟（Linux 5.6+） |

关键 API：
- `clone(2)`：创建新进程并指定 namespace
- `setns(2)`：加入已有 namespace
- `unshare(2)`：在新 namespace 中运行

进程的 namespace 信息可通过 `/proc/<pid>/ns/` 查看。

### Cgroups（控制组）

Cgroups 用于限制、记录和隔离进程组使用的物理资源（CPU、内存、I/O）[^3]。

核心概念：
- **cgroup**：一组进程及其关联的子系统参数
- **subsystem**（资源控制器）：如 `cpu`、`memory`、`cpuset` 等
- **hierarchy**：cgroup 组成的树状层级结构

Cgroups 通过虚拟文件系统暴露，挂载于 `/sys/fs/cgroup/`。用户可通过读写文件操作：
- `tasks`：列出或添加进程 PID
- `cgroup.procs`：列出或添加线程组
- 子系统特定文件：如 `cpu.cfs_quota_us`、`memory.limit_in_bytes`

## 存储驱动与镜像分层

### UnionFS / OverlayFS

Docker 使用 UnionFS 实现镜像分层存储，默认驱动为 `overlay2`[^4]。

OverlayFS 将多个目录（层）合并为单一视图：
- `lowerdir`：只读的镜像层（可多层）
- `upperdir`：可写的容器层
- `merged`：合并后的统一视图
- `workdir`：OverlayFS 内部工作目录

### 镜像与容器

| 概念 | 说明 |
|------|------|
| Image（镜像） | 只读模板，包含创建容器的指令；通过 Dockerfile 构建，每条指令生成一层 |
| Container（容器） | 镜像的可运行实例；容器层（upperdir）可写，镜像层（lowerdir）只读 |

镜像存储路径：`/var/lib/docker/overlay2/`

### copy_up 操作

容器首次写入文件时，OverlayFS 执行 `copy_up`：将文件从 lowerdir 复制到 upperdir。特点：
- 文件级别操作（非块级别），即使修改小块也复制整个文件
- 仅首次写入触发，后续直接操作 upperdir
- 大文件首次写入可能有性能开销

### whiteout 与 opaque

- 删除文件：在 upperdir 创建 whiteout 文件，遮蔽 lowerdir 中的文件
- 删除目录：在 upperdir 创建 opaque 目录，遮蔽 lowerdir 中的目录

## 容器运行流程

以 `docker run -i -t ubuntu /bin/bash` 为例[^1]：

1. 拉取镜像：若无本地镜像，从 registry 拉取
2. 创建容器：分配读写文件系统（upperdir）
3. 创建网络接口：连接默认网络，分配 IP
4. 启动容器：执行 `/bin/bash`
5. 交互运行：通过 `-i`、`-t` 附加到终端

## 限制与注意事项

### OverlayFS POSIX 兼容性

OverlayFS 仅实现 POSIX 标准子集[^4]：
- 同一文件先后以只读、读写方式打开，两个文件描述符可能指向不同层
- `rename(2)` 对目录不完全支持，跨层重命名返回 `EXDEV`
- 受影响的工具：`yum`（需安装 `yum-plugin-ovl`）

### 性能建议

- 使用 SSD 作为 backing filesystem
- 写密集型工作负载使用 Volume（绕过存储驱动）
- XFS backing filesystem 需要 `d_type=true`（`ftype=1`）

### namespace 限制

- 创建 namespace 通常需要 `CAP_SYS_ADMIN` capability（User namespace 除外）
- `/proc/sys/user/max_*_namespaces` 定义各类 namespace 数量上限

## 与虚拟机的区别

| 特性 | Docker 容器 | 传统虚拟机 |
|------|------------|-----------|
| 隔离级别 | 进程级别（共享内核） | 系统级别（独立内核） |
| 启动速度 | 秒级 | 分钟级 |
| 资源开销 | 低（共享宿主机资源） | 高（需要完整 OS） |
| 性能 | 接近宿主机 | 有虚拟化损耗 |

## 参考资料

[^1]: Docker Overview - https://docs.docker.com/get-started/overview/
[^2]: Linux namespaces(7) - https://man7.org/linux/man-pages/man7/namespaces.7.html
[^3]: Linux Cgroups Documentation - https://www.kernel.org/doc/Documentation/cgroup-v1/cgroups.txt
[^4]: Docker OverlayFS storage driver - https://docs.docker.com/storage/storagedriver/overlayfs-driver/
