---
created: 2026-04-16
updated: 2026-04-16
tags:
  - linux
  - io-model
  - system-programming
  - bio
  - nio
  - aio
  - epoll
  - io-uring
  - kernel
aliases:
  - Linux I/O 模型
  - BIO NIO AIO
  - 阻塞IO 非阻塞IO 异步IO
  - Linux I/O Multiplexing
source_type: official-doc
source_urls:
  - https://man7.org/linux/man-pages/man2/select.2.html
  - https://man7.org/linux/man-pages/man7/epoll.7.html
  - https://man7.org/linux/man-pages/man7/aio.7.html
  - https://man7.org/linux/man-pages/man2/io_setup.2.html
  - https://man7.org/linux/man-pages/man7/io_uring.7.html
status: verified
---

## 概述

在 Linux 系统中，I/O 模型决定了应用程序如何与内核交互以完成数据读写。理解这些模型是构建高性能网络服务和存储系统的基础。

Linux 主要支持以下 I/O 模型：

| 模型 | 全称 | 阻塞性 | 触发方式 | 典型系统调用 |
|------|------|--------|----------|-------------|
| BIO | Blocking I/O | 阻塞 | 同步 | `read()`, `write()` |
| NIO（非阻塞） | Non-blocking I/O | 非阻塞 | 轮询 | `read()` + `O_NONBLOCK` |
| I/O 多路复用 | I/O Multiplexing | 阻塞（等待事件） | 事件驱动 | `select()`, `poll()`, `epoll()` |
| 信号驱动 I/O | Signal-Driven I/O | 非阻塞 | 信号通知 | `sigaction()` + `SIGIO` |
| AIO（POSIX） | Asynchronous I/O | 非阻塞 | 异步回调 | `aio_read()`, `aio_write()` |
| AIO（Linux 原生） | Linux Native AIO | 非阻塞 | 异步回调 | `io_setup()`, `io_submit()`, `io_getevents()` |
| io_uring | io_uring | 非阻塞 | 异步回调 | `io_uring_setup()`, `io_uring_enter()` |

> **注意**：Java 生态中的 "NIO" 通常指 I/O 多路复用（`Selector`），而非单纯的非阻塞 I/O。本文以 Linux 系统调用为基准进行说明。

---

## BIO（Blocking I/O，阻塞式 I/O）

### 是什么

BIO 是最基础的 I/O 模型。当应用程序发起 I/O 系统调用（如 `read()`、`write()`）时，如果数据尚未就绪（例如网络数据包未到达、磁盘数据未加载到页缓存），调用线程会被**挂起**，直到 I/O 操作完成或出错。

### 工作流程

1. 用户态调用 `read(fd, buf, len)`
2. 内核检查数据是否就绪
3. 若未就绪，线程进入睡眠状态（`TASK_INTERRUPTIBLE` 或 `TASK_UNINTERRUPTIBLE`）
4. 设备驱动完成数据准备后唤醒线程
5. 数据从内核缓冲区拷贝到用户缓冲区
6. 系统调用返回

### 特点

- **编程简单**：同步顺序模型，代码直观
- **资源消耗大**：每个连接需要一个独立线程，线程上下文切换开销随连接数线性增长
- **无法有效利用 CPU**：线程在等待 I/O 期间完全空闲

### 适用场景

- 连接数少且稳定的服务
- 简单的脚本和工具程序
- 对延迟敏感但不需要高并发的场景

---

## NIO（Non-blocking I/O，非阻塞式 I/O）

### 是什么

通过设置文件描述符的 `O_NONBLOCK` 标志，使 I/O 操作在数据未就绪时**立即返回**而不是阻塞等待，返回值为 `-1` 且 `errno` 设为 `EAGAIN` 或 `EWOULDBLOCK`。

### 设置方式

```c
// 方法一：open 时设置
int fd = open("/dev/file", O_RDONLY | O_NONBLOCK);

// 方法二：fcntl 修改已有 fd
int flags = fcntl(fd, F_GETFL, 0);
fcntl(fd, F_SETFL, flags | O_NONBLOCK);
```

### 工作流程

1. 用户态调用 `read(fd, buf, len)`
2. 内核检查数据是否就绪
3. 若未就绪，**立即返回** `-1`，`errno = EAGAIN`
4. 若已就绪，拷贝数据并返回实际读取字节数

### 轮询模式（Polling）

非阻塞 I/O 本身不提供事件通知机制，应用程序需要**主动轮询**：

```c
while (1) {
    ssize_t n = read(fd, buf, sizeof(buf));
    if (n > 0) {
        // 处理数据
    } else if (n == -1 && errno == EAGAIN) {
        // 数据未就绪，继续轮询（会浪费 CPU）
    } else {
        // 错误或 EOF
    }
}
```

> **问题**：纯轮询会持续消耗 CPU 资源，实际应用中通常与 I/O 多路复用结合使用。

### 特点

- **不阻塞线程**：调用立即返回
- **需要轮询**：单独使用时 CPU 利用率低
- **与多路复用配合**：是 epoll/selector 工作模式的基础

---

## I/O 多路复用（I/O Multiplexing）

### 是什么

I/O 多路复用允许**单个线程同时监控多个文件描述符**，当其中任意一个 fd 变为就绪状态（可读、可写或异常）时，通知应用程序进行实际的 I/O 操作。

多路复用本身**是阻塞的**——调用线程会在 `select`/`poll`/`epoll_wait` 上等待，直到有事件发生。但它与 BIO 的区别在于：一次等待可以监控**成百上千个** fd，而不是每个 fd 需要一个线程。

### select

**引入时间**：4.2BSD，POSIX.1-2001 标准化[^select]

```c
#include <sys/select.h>
int select(int nfds, fd_set *readfds, fd_set *writefds,
           fd_set *exceptfds, struct timeval *timeout);
```

**核心机制**：

- 使用 `fd_set` 位图表示关注的 fd 集合
- 调用时传入三个集合（可读、可写、异常），返回时被修改为就绪的 fd
- `nfds` 为最大 fd 值 + 1

**限制**：

| 限制 | 说明 |
|------|------|
| fd 上限 | `FD_SETSIZE` = 1024，不可扩展[^select] |
| 每次调用需重新初始化 | `fd_set` 是 value-result 参数，返回后被修改，下次调用前必须重新设置[^select] |
| 线性扫描 | 内核和用户态都需要遍历所有 fd，时间复杂度 O(n) |
| 无法直接定位就绪 fd | 需要用 `FD_ISSET` 逐一检查 |

**典型用法**：

```c
fd_set rfds;
FD_ZERO(&rfds);
FD_SET(0, &rfds);  // 监控 stdin

struct timeval tv = { .tv_sec = 5, .tv_usec = 0 };
int retval = select(1, &rfds, NULL, NULL, &tv);
// 注意：返回后 tv 的值在 Linux 上会被修改，不可复用[^select]
```

### poll

**引入时间**：System V

```c
#include <poll.h>
int poll(struct pollfd *fds, nfds_t nfds, int timeout);
```

**与 select 的对比**：

| 特性 | select | poll |
|------|--------|------|
| fd 表示 | `fd_set` 位图 | `struct pollfd` 数组 |
| fd 上限 | 1024（`FD_SETSIZE`） | 无硬性限制 |
| 事件类型 | 读/写/异常三类 | 更细粒度的事件掩码（`POLLIN`, `POLLOUT`, `POLLERR`, `POLLHUP`, `POLLPRI` 等） |
| value-result | 是（需每次重新初始化） | 否（`revents` 字段独立存储结果） |
| 时间复杂度 | O(n) | O(n) |

**核心结构**：

```c
struct pollfd {
    int   fd;         // 文件描述符
    short events;     // 关注的事件（输入）
    short revents;    // 实际发生的事件（输出）
};
```

### epoll

**引入时间**：Linux 2.5.44（2002年），glibc 2.3.2[^epoll]

epoll 是 Linux 特有的 I/O 事件通知设施，专为大规模并发连接设计。

```c
#include <sys/epoll.h>
int epoll_create(int size);         // 已废弃，使用 epoll_create1
int epoll_create1(int flags);       // 创建 epoll 实例
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);  // 管理监控列表
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);  // 等待事件
```

**核心架构**：

epoll 实例在内核中维护两个列表[^epoll]：

1. **Interest List（兴趣列表）**：通过 `epoll_ctl(EPOLL_CTL_ADD)` 注册的 fd 集合
2. **Ready List（就绪列表）**：内核根据 I/O 活动动态维护的就绪 fd 子集

`epoll_wait` 仅从 Ready List 中取数据，无需遍历所有 fd。

**触发模式**：

| 模式 | 标志 | 行为 |
|------|------|------|
| 水平触发（LT） | 默认 | 只要 fd 处于就绪状态，每次 `epoll_wait` 都会通知 |
| 边缘触发（ET） | `EPOLLET` | 仅在 fd 状态**变化**时通知一次 |

**ET 模式使用要点**[^epoll]：

- 必须配合**非阻塞 fd** 使用
- 收到事件后需循环 `read`/`write` 直到返回 `EAGAIN`
- 否则可能遗漏缓冲区中剩余的数据

```c
// ET 模式推荐用法
ev.events = EPOLLIN | EPOLLET;
ev.data.fd = conn_sock;
epoll_ctl(epollfd, EPOLL_CTL_ADD, conn_sock, &ev);

// 处理时需读到 EAGAIN
while (1) {
    ssize_t n = read(fd, buf, sizeof(buf));
    if (n > 0) {
        // 处理数据
    } else if (n == -1 && errno == EAGAIN) {
        break;  // 缓冲区已空
    } else {
        // 错误或关闭
        break;
    }
}
```

**性能优势**：

| 特性 | select/poll | epoll |
|------|-------------|-------|
| 添加 fd | 每次调用传入全量集合 | `epoll_ctl` 一次性注册 |
| 内核扫描 | 每次调用遍历全部 fd | 仅检查活跃的 fd |
| 返回结果 | 需要用户态遍历检查 | 直接返回就绪 fd 数组 |
| 时间复杂度 | O(n) | O(1)（仅与活跃 fd 数相关） |
| 内存拷贝 | 每次调用全量拷贝 fd 集合 | 通过 `mmap` 共享内存 |

**限制**：

- 仅 Linux 可用（FreeBSD 有 `kqueue`，Solaris 有 `/dev/poll`）[^epoll]
- `/proc/sys/fs/epoll/max_user_watches` 限制单个用户可注册的 fd 总数（默认为可用低内存的 4%）[^epoll]

---

## AIO（Asynchronous I/O，异步 I/O）

### POSIX AIO（glibc 用户态实现）

**标准化**：POSIX.1-2001[^aio]

POSIX AIO 提供了一组异步 I/O 函数，允许应用程序发起 I/O 操作后继续执行其他任务，通过信号、线程或轮询方式获取完成通知。

**核心 API**[^aio]：

| 函数 | 说明 |
|------|------|
| `aio_read()` | 发起异步读 |
| `aio_write()` | 发起异步写 |
| `aio_fsync()` | 异步同步（类似 `fsync`） |
| `aio_error()` | 查询 I/O 操作错误状态 |
| `aio_return()` | 获取 I/O 操作返回值 |
| `aio_suspend()` | 阻塞等待指定 I/O 完成 |
| `aio_cancel()` | 尝试取消未完成的 I/O |
| `lio_listio()` | 单次调用发起多个 I/O 请求 |

**控制块结构**[^aio]：

```c
struct aiocb {
    int             aio_fildes;     // 文件描述符
    off_t           aio_offset;     // 文件偏移
    volatile void  *aio_buf;        // 数据缓冲区
    size_t          aio_nbytes;     // 传输长度
    int             aio_reqprio;    // 请求优先级
    struct sigevent aio_sigevent;   // 通知方式（SIGEV_NONE/SIGEV_SIGNAL/SIGEV_THREAD）
    int             aio_lio_opcode; // 操作类型（仅 lio_listio 使用）
};
```

**重要限制**[^aio]：

> Linux 的 POSIX AIO 实现是**用户态**的（由 glibc 提供），内部通过线程池模拟异步行为。这意味着：
> - 维护多个线程执行 I/O 操作开销大、扩展性差
> - 并非真正的内核级异步 I/O

### Linux Native AIO（内核态实现）

**引入时间**：Linux 2.5[^io_setup]

Linux 提供了一组原生内核异步 I/O 系统调用，但**没有 glibc 包装函数**，需通过 `libaio` 库或直接 `syscall()` 调用。

**核心 API**[^io_setup]：

| 系统调用 | 说明 |
|---------|------|
| `io_setup()` | 创建异步 I/O 上下文 |
| `io_submit()` | 提交 I/O 请求 |
| `io_getevents()` | 获取完成的 I/O 事件 |
| `io_cancel()` | 取消指定 I/O 请求 |
| `io_destroy()` | 销毁 I/O 上下文 |

**限制**：

- 仅支持**直接 I/O**（`O_DIRECT`），不适用于常规 buffered I/O
- 主要针对块设备（磁盘）优化，对 socket 支持有限
- 接口较为复杂，未成为主流选择

---

## io_uring

**引入时间**：Linux 5.1（2019年），由 Jens Axboe 开发[^io_uring]

io_uring 是 Linux 最新的异步 I/O 接口，旨在解决传统 AIO 的局限性，提供高性能、通用的异步 I/O 机制。

### 核心设计

io_uring 通过**用户态与内核态共享的环形缓冲区**（ring buffer）进行通信，避免了频繁的系统调用和数据拷贝[^io_uring]：

- **Submission Queue (SQ)**：用户态放入 I/O 请求（SQE），内核态读取
- **Completion Queue (CQ)**：内核态放入完成结果（CQE），用户态读取

```
用户态 ──→ SQ（提交请求）──→ 内核态处理 ──→ CQ（返回结果）──→ 用户态读取
```

### 工作流程[^io_uring]

1. `io_uring_setup()` 创建实例，通过 `mmap()` 映射共享缓冲区
2. 用户填充 `io_uring_sqe` 结构体（指定操作类型、fd、缓冲区等）
3. 将 SQE 添加到 SQ 尾部，更新 tail 指针
4. 调用 `io_uring_enter()` 通知内核处理请求
5. 内核完成后将 `io_uring_cqe` 添加到 CQ 尾部
6. 用户从 CQ 头部读取结果

### SQE 核心字段[^io_uring]：

```c
struct io_uring_sqe {
    __u8    opcode;         // 操作类型（IORING_OP_READ, IORING_OP_WRITE 等）
    __s32   fd;             // 文件描述符
    __u64   off;            // 文件偏移
    __u64   addr;           // 缓冲区地址
    __u32   len;            // 缓冲区长度
    __u64   user_data;      // 用户自定义数据（提交→完成透传）
    __u8    flags;          // IOSQE_ 标志
    // ... 其他字段
};
```

### CQE 结构[^io_uring]：

```c
struct io_uring_cqe {
    __u64   user_data;  // 对应 SQE 的 user_data
    __s32   res;        // 结果（成功时等同于系统调用返回值，失败时为 -errno）
    __u32   flags;      // 标志位
};
```

> **注意**：io_uring 不使用 `errno` 传递错误信息，`res` 字段直接包含负的错误码[^io_uring]。

### 性能优化特性

| 特性 | 说明 |
|------|------|
| 零拷贝通信 | SQ/CQ 通过 `mmap` 共享，避免系统调用间的数据拷贝 |
| 批量提交 | 一次 `io_uring_enter()` 可提交多个请求 |
| SQ Polling | 内核线程轮询 SQ，完全消除 `io_uring_enter()` 系统调用开销 |
| 固定缓冲区 | 通过 `IORING_REGISTER_BUFFERS` 预注册缓冲区，避免每次 pin/unpin |
| 链接操作 | `IOSQE_IO_LINK` 保证多个操作的执行顺序 |
| 多 shot 模式 | 单次注册可接收多次事件（如 `IORING_OP_RECV` 的 `IORING_RECV_MULTISHOT`） |

### 支持的 opcode（部分）

| opcode | 等效系统调用 | 说明 |
|--------|------------|------|
| `IORING_OP_READ` | `read()` | 异步读 |
| `IORING_OP_WRITE` | `write()` | 异步写 |
| `IORING_OP_READV` | `readv()` | 分散读 |
| `IORING_OP_WRITEV` | `writev()` | 集中写 |
| `IORING_OP_ACCEPT` | `accept()` | 接受连接 |
| `IORING_OP_CONNECT` | `connect()` | 发起连接 |
| `IORING_OP_RECV` | `recv()` | 接收数据 |
| `IORING_OP_SEND` | `send()` | 发送数据 |
| `IORING_OP_OPENAT` | `openat()` | 打开文件 |
| `IORING_OP_CLOSE` | `close()` | 关闭 fd |
| `IORING_OP_FSYNC` | `fsync()` | 同步文件 |
| `IORING_OP_TIMEOUT` | — | 超时控制 |
| `IORING_OP_NOP` | — | 空操作（测试用） |

### 版本要求

| 特性 | 最低内核版本 |
|------|------------|
| 基础 io_uring | 5.1 |
| 单次 mmap 映射 SQ+CQ | 5.4（`IORING_FEAT_SINGLE_MMAP`） |
| SQ Polling | 5.1 |
| 链接操作（IO_LINK） | 5.3 |
| 固定缓冲区 | 5.1 |
| 多 shot recv | 5.19+ |

### 与 liburing

由于 glibc 尚未提供 io_uring 的包装函数，官方推荐使用 **liburing** 库（<https://github.com/axboe/liburing>）简化开发[^io_uring]。liburing 提供了友好的 API：

```c
struct io_uring ring;
io_uring_queue_init(32, &ring, 0);

struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_read(sqe, fd, buf, sizeof(buf), offset);
io_uring_submit(&ring);

struct io_uring_cqe *cqe;
io_uring_wait_cqe(&ring, &cqe);
// 处理 cqe->res
io_uring_cqe_seen(&ring, cqe);
```

---

## 各模型对比总结

| 维度 | BIO | NIO（非阻塞） | select/poll | epoll | POSIX AIO | io_uring |
|------|-----|-------------|-------------|-------|-----------|----------|
| 阻塞性 | 阻塞 | 非阻塞 | 阻塞等待 | 阻塞等待 | 非阻塞 | 非阻塞 |
| 并发模型 | 一连接一线程 | 轮询 | 单线程多路 | 单线程多路 | 线程池模拟 | 异步回调 |
| fd 上限 | 无 | 无 | 1024（select）/ 无（poll） | 系统内存限制 | 无 | 无 |
| 时间复杂度 | O(1) | O(1) | O(n) | O(1) | — | O(1) |
| 系统调用开销 | 每次 I/O | 每次轮询 | 每次全量传入 | 注册+等待分离 | 每次提交 | 批量提交 |
| 适用连接数 | 少（<100） | 少 | 中等（<1000） | 高（10万+） | 中等 | 高（10万+） |
| 平台 | 通用 | 通用 | POSIX | Linux only | POSIX | Linux only |
| 典型应用 | 简单脚本 | 底层配合 | 跨平台库 | Nginx, Redis | 较少使用 | 高性能存储/数据库 |

---

## 常见误区

### 误区 1：NIO 就是 epoll

**纠正**：在 Java 语境中，"NIO" 指的是 `java.nio` 包中的 I/O 多路复用能力（底层在 Linux 上使用 epoll），而非单纯的非阻塞 I/O。在 Linux 系统编程中，NIO 特指设置 `O_NONBLOCK` 后的非阻塞行为。

### 误区 2：epoll 一定比 select/poll 快

**纠正**：当监控的 fd 数量较少（如几十个）且大部分都活跃时，epoll 与 select/poll 的性能差异不明显。epoll 的优势在于**大量 fd 中只有少量活跃**的场景。

### 误区 3：ET 模式总是比 LT 模式高效

**纠正**：ET 模式减少了事件通知次数，但要求应用程序必须正确处理所有数据（读到 `EAGAIN`），实现复杂度更高。对于简单场景，LT 模式更安全可靠。

### 误区 4：POSIX AIO 在 Linux 上是真正的异步 I/O

**纠正**：Linux 的 glibc POSIX AIO 实现是用户态线程池模拟，并非内核级异步。真正的内核级异步 I/O 需要通过 `io_setup`/`io_submit` 系列调用或 io_uring。

---

## 相关概念

### I/O 操作的两个阶段

理解 I/O 模型需要区分两个阶段[^select]：

1. **等待数据就绪**：数据从设备到达内核缓冲区（如网络包到达网卡、磁盘数据读入页缓存）
2. **数据拷贝**：数据从内核缓冲区拷贝到用户缓冲区

BIO 在两个阶段都阻塞；NIO 在第一阶段不阻塞；真正的异步 I/O（AIO）在两个阶段都不阻塞，由内核完成后通知用户。

### 文件描述符（fd）与打开文件描述（open file description）

- **fd**：进程级别的整数索引，指向内核中的打开文件描述
- **打开文件描述**：内核级别的数据结构，包含文件偏移、状态标志等
- 通过 `dup()`、`fork()` 可以创建多个 fd 指向同一个打开文件描述
- epoll 监控的是**打开文件描述**，关闭一个 fd 不会立即从 epoll 列表中移除，需要所有引用该打开文件描述的 fd 都被关闭[^epoll]

### 水平触发 vs 边缘触发

| 特性 | 水平触发（LT） | 边缘触发（ET） |
|------|--------------|--------------|
| 通知时机 | 只要条件满足就通知 | 仅在条件**变化**时通知一次 |
| 编程复杂度 | 低 | 高（需处理完所有数据） |
| 遗漏风险 | 无 | 有（未读完数据会丢失通知） |
| 默认行为 | epoll 默认 | 需显式设置 `EPOLLET` |

---

## 参考资料

[^select]: `select(2)` — Linux man-pages 6.16, <https://man7.org/linux/man-pages/man2/select.2.html>
[^epoll]: `epoll(7)` — Linux man-pages 6.16, <https://man7.org/linux/man-pages/man7/epoll.7.html>
[^aio]: `aio(7)` — Linux man-pages 6.16, <https://man7.org/linux/man-pages/man7/aio.7.html>
[^io_setup]: `io_setup(2)` — Linux man-pages 6.16, <https://man7.org/linux/man-pages/man2/io_setup.2.html>
[^io_uring]: `io_uring(7)` — liburing, <https://man7.org/linux/man-pages/man7/io_uring.7.html>
