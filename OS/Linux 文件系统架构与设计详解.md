---
created: '2026-04-17'
updated: '2026-04-17'
tags:
  - linux
  - filesystem
  - vfs
  - kernel
  - os
  - system-design
aliases:
  - Linux 文件系统设计
  - Linux Filesystem Architecture
  - VFS 虚拟文件系统
  - Linux 文件系统架构
source_type: official-doc
source_urls:
  - 'https://www.kernel.org/doc/html/latest/filesystems/vfs.html'
  - 'https://www.kernel.org/doc/html/latest/filesystems/index.html'
status: verified
---

## 概述

Linux 文件系统设计的核心是 **虚拟文件系统（Virtual File System, VFS）**，它是内核中为用户提供文件系统接口的软件层，同时在内核内部提供抽象机制，使不同的文件系统实现能够共存。

VFS 的设计哲学是 **统一接口、多实现共存**：用户空间通过统一的系统调用（如 `open(2)`、`read(2)`、`write(2)`、`stat(2)`、`chmod(2)` 等）访问文件，而 VFS 负责将这些调用路由到具体的文件系统实现（如 ext4、XFS、Btrfs、NFS 等）。

## 核心架构：VFS 四层对象模型

VFS 通过四个核心对象来抽象文件系统的结构和行为：

| 对象 | 对应内核结构体 | 作用 | 生命周期 |
|------|---------------|------|---------|
| **Superblock** | `struct super_block` | 表示一个已挂载的文件系统实例 | 挂载时创建，卸载时销毁 |
| **Inode** | `struct inode` | 表示文件系统中的一个对象（文件、目录、FIFO、套接字等） | 首次访问时加载，无引用时回收 |
| **Dentry** | `struct dentry` | 目录项缓存，用于路径名到 inode 的快速查找 | 仅存在于内存中，不写入磁盘 |
| **File** | `struct file` | 表示进程打开的文件（文件描述符的内核实现） | `open()` 时创建，`close()` 时销毁 |

### 对象之间的关系

```
进程空间
  │
  ├── fd 0 ──→ struct file ──→ struct dentry ──→ struct inode ──→ 磁盘数据
  ├── fd 1 ──→ struct file ──┘                    ↑
  └── fd 2 ──→ struct file ──→ struct dentry ─────┘
                                      │
                            struct super_block（已挂载的文件系统）
```

- 一个 **inode** 可以被多个 **dentry** 指向（硬链接的场景）
- 一个 **file** 结构指向一个 **dentry**，并保持 dentry 和 inode 在使用期间不被回收
- **dentry** 仅存在于内存中，是纯粹的性能优化组件

## 目录项缓存（Dentry Cache / Dcache）

### 作用

Dcache 是 VFS 路径查找的核心加速机制。当用户调用 `open(2)`、`stat(2)`、`chmod(2)` 等系统调用时，VFS 使用传入的路径名在 dentry 缓存中快速查找，将路径名转换为对应的 dentry（进而获取 inode）。

### 关键特性

- **仅存在于内存**：dentry 永远不会写入磁盘，纯粹为性能而存在
- **全局视图**：dentry 缓存旨在成为整个文件空间的视图
- **按需创建**：当路径中的某些 dentry 不在缓存中时，VFS 会沿途创建 dentry 并加载对应的 inode
- **负缓存（Negative Dentry）**：当查找的文件不存在时，VFS 也会缓存这个"不存在"的结果，避免重复查找

### 查找流程

```
路径名 "/home/user/file.txt"
  │
  ├── 查找 "/" 的 dentry → 命中
  ├── 查找 "home" 的 dentry → 命中
  ├── 查找 "user" 的 dentry → 未命中 → 调用父目录 inode 的 lookup() 方法
  ├── 查找 "file.txt" 的 dentry → 未命中 → 调用父目录 inode 的 lookup() 方法
  └── 返回最终 dentry（含 inode 指针）
```

## Inode 对象详解

### 什么是 Inode

Inode（Index Node）是文件系统中表示一个对象的核心数据结构。它可以表示：
- 普通文件（regular file）
- 目录（directory）
- 符号链接（symbolic link）
- 命名管道（FIFO）
- 套接字（socket）
- 字符/块设备（character/block device）

### Inode 的存储位置

| 文件系统类型 | Inode 存储位置 |
|-------------|---------------|
| 块设备文件系统（ext4, XFS 等） | 磁盘上，按需加载到内存 |
| 伪文件系统（procfs, sysfs, tmpfs 等） | 仅在内存中 |

### Inode 的关键操作（`struct inode_operations`）

文件系统通过实现 `inode_operations` 中的方法来响应 VFS 的调用：

| 方法 | 对应系统调用 | 说明 |
|------|-------------|------|
| `create` | `open(2)`, `creat(2)` | 创建普通文件 |
| `lookup` | 路径查找 | 在父目录中查找子项 |
| `link` | `link(2)` | 创建硬链接 |
| `unlink` | `unlink(2)` | 删除文件 |
| `mkdir` | `mkdir(2)` | 创建目录 |
| `rmdir` | `rmdir(2)` | 删除目录 |
| `symlink` | `symlink(2)` | 创建符号链接 |
| `mknod` | `mknod(2)` | 创建设备节点或 FIFO |
| `rename` | `rename(2)` | 重命名/移动文件 |
| `setattr` | `chmod(2)`, `chown(2)` | 修改文件属性 |
| `getattr` | `stat(2)` | 获取文件属性 |
| `permission` | 权限检查 | POSIX 权限验证 |

### 硬链接与 Inode

硬链接的本质是 **多个 dentry 指向同一个 inode**：
- 硬链接不区分"原始文件"和"链接"，所有指向同一 inode 的 dentry 地位平等
- inode 中的 `i_nlink` 字段记录指向它的 dentry 数量
- 当 `i_nlink` 降为 0 且无进程引用时，inode 才会被真正删除

## File 对象详解

### 什么是 File 对象

File 对象（`struct file`）表示一个进程打开的文件，是 **文件描述符（file descriptor）的内核实现**。注意区分：
- **inode**：表示文件系统中的对象（静态概念）
- **file**：表示进程打开文件的状态（动态概念）

同一个 inode 可以被多个 file 对象打开（多个进程或同一进程多次 `open()`）。

### File 的关键操作（`struct file_operations`）

| 方法 | 对应系统调用 | 说明 |
|------|-------------|------|
| `read` / `read_iter` | `read(2)`, `pread(2)` | 读取文件数据 |
| `write` / `write_iter` | `write(2)`, `pwrite(2)` | 写入文件数据 |
| `llseek` | `lseek(2)` | 移动文件读写位置 |
| `mmap` | `mmap(2)` | 内存映射文件 |
| `open` | `open(2)` | 打开文件时的初始化 |
| `release` | `close(2)` | 关闭文件时的清理 |
| `fsync` | `fsync(2)`, `fdatasync(2)` | 将数据刷入持久存储 |
| `poll` | `select(2)`, `poll(2)`, `epoll(2)` | I/O 多路复用 |
| `unlocked_ioctl` | `ioctl(2)` | 设备控制操作 |
| `splice_read` / `splice_write` | `splice(2)` | 零拷贝数据传输 |
| `copy_file_range` | `copy_file_range(2)` | 内核内文件数据拷贝 |
| `fallocate` | `fallocate(2)` | 预分配文件空间 |

### File 对象的生命周期

```
open() → 分配 file 结构 → 初始化 dentry 指针和 file_operations
       → 调用具体文件系统的 open() 方法
       → 放入进程的文件描述符表
       → 用户通过 fd 进行 read/write/seek 等操作
       → close() → 调用 release() → 释放 file 结构
```

**重要**：只要 file 对象存在，它引用的 dentry 和 inode 就不会被回收。

## Address Space 对象

### 作用

Address Space（`struct address_space`）用于管理 **页缓存（Page Cache）**，是 VFS 与内存管理子系统之间的桥梁。主要职责：

- 跟踪文件数据在页缓存中的分布
- 管理脏页（Dirty Page）和回写（Writeback）状态
- 协调内存压力下的页面回收
- 支持内存映射（mmap）

### 关键操作（`struct address_space_operations`）

| 方法 | 说明 |
|------|------|
| `read_folio` | 从存储读取一个 folio 到页缓存 |
| `writepages` | 将脏页写回存储 |
| `dirty_folio` | 标记 folio 为脏 |
| `readahead` | 预读相邻页面 |
| `direct_IO` | 绕过页缓存的直接 I/O |
| `migrate_folio` | 页面迁移（用于内存压缩） |

### Writeback 错误处理

当写回过程中发生错误时，内核的通用写回错误跟踪基础设施会：
- 将错误记录在 address_space 中
- 在后续 `fsync()` 调用时向所有在错误发生时打开的文件描述符报告错误
- 错误报告后，后续 `fsync()` 返回 0（除非有新错误发生）

## 文件系统的注册与挂载

### 注册文件系统

文件系统通过 `register_filesystem()` 向 VFS 注册：

```c
#include <linux/fs.h>

struct file_system_type {
    const char *name;           // 文件系统名称，如 "ext4", "xfs"
    int fs_flags;               // 标志位（如 FS_REQUIRES_DEV）
    int (*init_fs_context)(struct fs_context *);
    const struct fs_parameter_spec *parameters;
    void (*kill_sb)(struct super_block *);
    struct module *owner;
    struct file_system_type *next;
    struct hlist_head fs_supers;
    // ... 锁相关字段
};

extern int register_filesystem(struct file_system_type *);
extern int unregister_filesystem(struct file_system_type *);
```

已注册的文件系统可在 `/proc/filesystems` 中查看。

### 挂载流程

1. 用户调用 `mount()` 系统调用
2. VFS 根据文件系统名称查找对应的 `file_system_type`
3. 调用该文件系统的 `get_tree()` 方法
4. 创建 `super_block` 对象表示挂载实例
5. 将文件系统挂载到命名空间中的指定目录

## 路径查找（Pathname Lookup）

### 基本流程

```
用户传入路径 "/a/b/c"
  │
  ├── 从当前进程的根目录或工作目录开始
  ├── 逐段解析路径组件
  ├── 每段查找：先在 dcache 中查找，未命中则调用目录 inode 的 lookup()
  ├── 处理符号链接：递归解析链接目标
  ├── 处理 ".." 和 "." 特殊目录
  └── 返回最终 dentry/inode
```

### RCU-Walk 优化

Linux 引入了 **RCU-walk** 模式来加速路径查找：
- 在无锁（RCU）模式下进行路径查找，避免获取 inode 锁
- 仅在需要修改或遇到竞争时回退到 ref-walk 模式（获取引用计数和锁）
- 显著提升了高并发场景下的路径查找性能

## 主流文件系统实现

Linux 支持数十种文件系统，以下是常见的几类：

### 本地磁盘文件系统

| 文件系统 | 特点 | 适用场景 |
|---------|------|---------|
| **ext4** | 第四代扩展文件系统，成熟稳定，支持日志 | 通用 Linux 根文件系统 |
| **XFS** | 高性能日志文件系统，擅长大文件和高并发 I/O | 服务器、存储系统 |
| **Btrfs** | 写时复制（CoW），支持快照、子卷、内建 RAID | 需要快照和高级功能的场景 |
| **F2FS** | 闪存友好型文件系统，针对 NAND 优化 | SSD、eMMC 等闪存设备 |

### 内存/伪文件系统

| 文件系统 | 说明 |
|---------|------|
| **tmpfs** | 基于内存的文件系统，可使用 swap |
| **ramfs** | 纯内存文件系统，不使用 swap，无大小限制 |
| **procfs** | 暴露内核和进程信息的虚拟文件系统（`/proc`） |
| **sysfs** | 暴露设备驱动和内核对象的虚拟文件系统（`/sys`） |
| **devtmpfs** | 自动创建设备节点的内存文件系统（`/dev`） |

### 网络文件系统

| 文件系统 | 说明 |
|---------|------|
| **NFS** | Network File System，UNIX /Linux 网络文件共享标准 |
| **CIFS/SMB** | Common Internet File System，Windows 文件共享协议 |
| **9P** | Plan 9 资源共享协议，常用于虚拟化环境 |

### 堆叠文件系统

| 文件系统 | 说明 |
|---------|------|
| **OverlayFS** | 联合挂载，容器技术（Docker）的基础 |
| **eCryptfs** | 堆叠式加密文件系统 |
| **FUSE** | 用户空间文件系统框架 |

## 关键设计原则

### 1. 一切皆文件

Linux 的核心设计哲学：**所有 I/O 资源都抽象为文件**。
- 普通数据文件
- 目录（也是一种文件，包含文件名到 inode 的映射）
- 设备（字符设备、块设备通过 `/dev` 暴露）
- 管道和套接字
- 内核信息（通过 procfs、sysfs 暴露）

这种统一抽象使得相同的系统调用（`read`、`write`、`ioctl`）可以操作不同类型的资源。

### 2. 分层解耦

```
用户空间应用
     │
     ▼
  系统调用接口（VFS 入口）
     │
     ▼
  VFS 抽象层（统一接口）
     │
     ▼
  具体文件系统实现（ext4, XFS, NFS, ...）
     │
     ▼
  块设备层 / 网络层
     │
     ▼
  物理存储设备
```

每一层只与相邻层交互，具体文件系统的变更不影响用户空间程序。

### 3. 缓存优先

VFS 大量使用缓存来优化性能：
- **Dentry Cache**：加速路径查找
- **Inode Cache**：缓存已加载的 inode
- **Page Cache**：缓存文件数据，减少磁盘 I/O
- **Buffer Cache**：缓存块设备数据（现代内核中已与 Page Cache 合并）

### 4. 延迟写入（Lazy Writeback）

为提高写入性能，Linux 采用延迟写入策略：
- 写入操作先修改页缓存中的脏页
- 内核的 writeback 线程定期将脏页刷入磁盘
- 用户可通过 `fsync()`、`fdatasync()`、`sync()` 强制刷盘
- `lazytime` 挂载选项可进一步延迟时间戳（atime/mtime/ctime）的写入

## 限制与注意事项

### 硬链接限制

- 硬链接 **不能跨文件系统**（inode 号仅在单个文件系统内唯一）
- 硬链接 **不能指向目录**（防止文件系统树中出现环）

### 符号链接注意事项

- 符号链接可以跨文件系统、可以指向目录
- 符号链接包含的是 **路径字符串**，而非 inode 号
- 目标文件被删除后，符号链接变为"悬空链接"（dangling symlink）
- 内核限制符号链接的最大递归深度（通常为 40 层）

### 文件描述符限制

- 每个进程有文件描述符数量上限（可通过 `ulimit -n` 查看和修改）
- 系统全局也有文件描述符上限（`/proc/sys/fs/file-max`）
- 文件描述符泄漏是常见的资源泄漏问题

### 并发与锁

- VFS 内部使用多种锁机制保证并发安全：
  - `i_rwsem`（inode 读写锁）
  - `d_lock`（dentry 自旋锁）
  - `s_umount`（superblock 卸载锁）
- 不当的锁顺序可能导致死锁，VFS 有严格的锁层级规则

## 相关概念

| 概念 | 与文件系统的关系 |
|------|----------------|
| **零拷贝（Zero-Copy）** | 利用 `sendfile()`、`splice()` 等系统调用绕过用户空间，直接在页缓存和网络栈之间传输数据 |
| **I/O 多路复用** | 通过 `select()`、`poll()`、`epoll()` 监控多个文件描述符的 I/O 就绪状态 |
| **DMA** | 直接内存访问，允许外设直接与内存交换数据，减少 CPU 参与 |
| **Inotify** | 基于 VFS 的文件变更通知机制，用于监控文件系统事件 |
| **io_uring** | 新一代异步 I/O 接口，提供高性能的提交/完成队列模型 |

## 参考资料

- [Linux Kernel Documentation: Overview of the Virtual File System](https://www.kernel.org/doc/html/latest/filesystems/vfs.html)
- [Linux Kernel Documentation: Filesystems in the Linux kernel](https://www.kernel.org/doc/html/latest/filesystems/index.html)
- [Linux Kernel Documentation: Pathname lookup](https://www.kernel.org/doc/html/latest/filesystems/path-lookup.html)
- [Linux Kernel Documentation: Filesystem Mount API](https://www.kernel.org/doc/html/latest/filesystems/mount_api.html)
- [Linux Kernel Documentation: Locking](https://www.kernel.org/doc/html/latest/filesystems/locking.html)
- [Linux Kernel Documentation: splice and pipes](https://www.kernel.org/doc/html/latest/filesystems/splice.html)
