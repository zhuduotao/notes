---

created: 2026-04-21
updated: 2026-04-21
tags:
  - Java
  - Virtual Threads
  - Project Loom
  - concurrency
  - JDK 21
  - 协程
aliases:
  - Java 协程
  - Virtual Thread
  - 虚拟线程
  - Fiber
  - Project Loom
source_type: official-doc
source_urls:
  - https://openjdk.org/jeps/444
  - https://openjdk.org/projects/loom/
  - https://docs.oracle.com/en/java/javase/21/core/virtual-threads.html
status: verified

---

## 概述

Java 没有传统意义上的"协程"概念，但 **Virtual Threads（虚拟线程）** 实现了类似协程的轻量级并发模型。这是 **Project Loom** 的核心成果，于 Java 21 正式发布（JEP 444），旨在让"thread-per-request"风格的 server 应用以接近最优的硬件利用率运行。

虚拟线程是 `java.lang.Thread` 的一种轻量级实现，由 JDK 而非操作系统提供，采用 M:N 调度模式——将大量虚拟线程（M）映射到少量 OS 线程（N）上执行。

---

## 核心概念对比

| 特性 | Platform Thread（平台线程） | Virtual Thread（虚拟线程） |
|------|---------------------------|--------------------------|
| 实现方式 | OS 线程的薄包装 | JDK 实现，用户态线程 |
| 资源开销 | 重（栈约 1MB，OS 管理） | 轻（栈在 GC 堆中，动态伸缩） |
| 数量限制 | 有限（受 OS 线程数限制） | 几乎无限（可创建百万级） |
| 调度方式 | OS 调度（1:1） | JDK 调度（M:N），由 ForkJoinPool 执行 |
| 适用场景 | CPU 密集型、长期运行 | I/O 密集型、短期阻塞任务 |
| 线程池 | 通常需要池化 | **不应池化**，每任务一个虚拟线程 |

---

## 为什么引入虚拟线程

### 传统 thread-per-request 的瓶颈

Server 应用常采用"thread-per-request"风格——每个请求由一个线程处理全程。根据 Little's Law：

```
吞吐量 = 并发数 / 平均延迟
```

要提高吞吐量，需要增加并发线程数。但平台线程数量受限于 OS 线程（通常几百到几千），在请求量增长时，线程数往往成为瓶颈，远早于 CPU、网络连接等其他资源耗尽。

### 异步编程的问题

为突破线程瓶颈，开发者转向异步/响应式风格（如 `CompletableFuture`、Reactive Streams），但这带来：

- 代码复杂：需将逻辑拆分成 lambda 阶段，放弃循环、try/catch 等基本语法
- 调试困难：栈追踪无可用上下文，调试器无法步进
- 与平台割裂：应用并发单元（异步管道）≠ 平台并发单元（线程）

### 虚拟线程的解法

虚拟线程保留 thread-per-request 的简洁性，同时实现异步风格的高吞吐：

- 阻塞 I/O 时自动挂起虚拟线程，释放 OS 线程
- I/O 完成后自动恢复虚拟线程继续执行
- 对开发者透明，无需改写阻塞代码

---

## 创建与使用

### 方式一：Thread.Builder API

```java
// 创建并启动虚拟线程
Thread thread = Thread.ofVirtual().name("duke").start(() -> {
    System.out.println("Hello from virtual thread");
});
thread.join();

// 创建 ThreadFactory 批量创建
ThreadFactory factory = Thread.ofVirtual().name("worker-", 0).factory();
Thread t1 = factory.newThread(task);
```

### 方式二：Executors.newVirtualThreadPerTaskExecutor()

```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    IntStream.range(0, 10_000).forEach(i -> {
        executor.submit(() -> {
            Thread.sleep(Duration.ofSeconds(1));
            return i;
        });
    });
} // close() 自动等待所有任务完成
```

这是最常用的方式——**为每个任务创建一个新虚拟线程**，不池化。

### 方式三：Thread.startVirtualThread()

```java
Thread.startVirtualThread(() -> doSomething());
```

简单快捷，适合单次任务。

### 判断是否为虚拟线程

```java
Thread.currentThread().isVirtual(); // true/false
```

---

## 调度与执行原理

### 载体线程（Carrier）

虚拟线程不直接绑定 OS 线程，而是挂载在"载体线程"（平台线程）上执行：

- JDK 虚拟线程调度器是一个 FIFO 模式的 `ForkJoinPool`
- 默认并行度 = `Runtime.availableProcessors()`
- 可调参数：`jdk.virtualThreadScheduler.parallelism`、`jdk.virtualThreadScheduler.maxPoolSize`

### 挂载与卸载

```
挂载（mount）   → 虚拟线程绑定载体线程执行
卸载（unmount） → 阻塞 I/O 时释放载体线程，虚拟线程挂起等待
```

大多数 JDK 阻塞操作（Socket I/O、`BlockingQueue.take()` 等）会触发卸载。载体线程空闲后可挂载其他虚拟线程。

---

## Pinning（钉住）问题

虚拟线程在以下情况**无法卸载**，会"钉住"载体线程：

1. 执行 `synchronized` 块或方法时阻塞
2. 执行 `native` 方法或 Foreign Function 时

### 影响

Pinning 不影响正确性，但频繁长时间钉住会降低并发吞吐，因为载体线程被阻塞。

### 解决方案

将频繁/长时间阻塞的 `synchronized` 改为 `ReentrantLock`：

```java
// ❌ 不推荐：长时间 I/O 在 synchronized 内
synchronized (lock) {
    socket.read(); // 阻塞时钉住载体线程
}

// ✅ 推荐：使用 ReentrantLock
lock.lock();
try {
    socket.read();
} finally {
    lock.unlock();
}
```

**注意**：短时间或低频 `synchronized` 无需替换。

### 检测 Pinning

- JFR 事件 `jdk.VirtualThreadPinned`（默认阈值 20ms）
- 启动参数：`-Djdk.tracePinnedThreads=full` 或 `-Djdk.tracePinnedThreads=short`

---

## 重要限制与注意事项

### 不要池化虚拟线程

虚拟线程廉价且 plentiful，池化反而有害：

- 虚拟线程代表任务本身，不是共享资源
- 每个任务应创建自己的虚拟线程

```java
// ❌ 旧思维：用线程池
ExecutorService pool = Executors.newFixedThreadPool(200);
pool.submit(task1);
pool.submit(task2);

// ✅ 新思维：每任务一虚拟线程
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(task1);
    executor.submit(task2);
}
```

### 限制并发用 Semaphore，不用线程池

```java
// ❌ 用线程池限制并发
ExecutorService pool = Executors.newFixedThreadPool(10);
pool.submit(() -> callLimitedService());

// ✅ 用 Semaphore 限制并发
Semaphore sem = new Semaphore(10);
sem.acquire();
try {
    callLimitedService();
} finally {
    sem.release();
}
```

### ThreadLocal 使用需谨慎

虚拟线程可使用 ThreadLocal，但：

- 百万虚拟线程 = 百万 ThreadLocal 实例 = 高内存开销
- **不要用 ThreadLocal 缓存昂贵对象**（如 `SimpleDateFormat`）

```java
// ❌ 每虚拟线程创建一个昂贵对象
static final ThreadLocal<SimpleDateFormat> formatter = 
    ThreadLocal.withInitial(SimpleDateFormat::new);

// ✅ 使用不可变对象共享
static final DateTimeFormatter formatter = DateTimeFormatter.ISO_DATE;
```

### 不适合 CPU 密集型任务

虚拟线程提高吞吐，不提高速度：

- 计算密集型任务：增加线程数（无论虚拟或平台）无意义
- 线程数超过 CPU 核心数无助于 CPU 任务吞吐

适用条件：

- 并发任务数 > 几千
- 任务非 CPU 密集型（大量 I/O 等待）

---

## 与其他语言的协程对比

| 语言 | 实现名称 | 调度方式 | 特点 |
|------|---------|---------|------|
| Java | Virtual Thread | M:N（JDK 调度） | 透明阻塞，无需 async/await |
| Go | Goroutine | M:N（Go runtime） | 需用 channel/select，显式并发 |
| Kotlin | Coroutine | M:N | 需要 `suspend` 标记函数 |
| Python | asyncio | 单线程事件循环 | 需要 `async/await` |
| Rust | async/await | 可配置 | 需要 `.await`，Future 组合 |

Java Virtual Thread 的优势：**无需改写现有阻塞代码**，API 与平台线程一致，调试/监控体验相同。

---

## 调试与监控

### JDK Flight Recorder（JFR）事件

| 事件 | 含义 | 默认状态 |
|------|------|---------|
| `jdk.VirtualThreadStart` | 虚拟线程启动 | 禁用 |
| `jdk.VirtualThreadEnd` | 虚拟线程结束 | 禁用 |
| `jdk.VirtualThreadPinned` | 钉住超过阈值 | 启用（阈值 20ms） |
| `jdk.VirtualThreadSubmitFailed` | 启动/恢复失败 | 启用 |

查看事件：

```bash
jfr print --events jdk.VirtualThreadPinned recording.jfr
```

### jcmd 线程转储

```bash
# 文本格式
jcmd <PID> Thread.dump_to_file -format=text thread_dump.txt

# JSON 格式（便于工具分析）
jcmd <PID> Thread.dump_to_file -format=json thread_dump.json
```

新格式列出所有线程（含虚拟线程），不含对象地址、锁等传统信息，且不会暂停应用。

### 调试器支持

IDE 调试器可：
- 步进虚拟线程
- 查看调用栈
- 检查栈帧变量

---

## API 变化摘要

### java.lang.Thread

| 方法 | 说明 |
|------|------|
| `Thread.ofVirtual()` | 创建虚拟线程 Builder |
| `Thread.ofPlatform()` | 创建平台线程 Builder |
| `Thread.startVirtualThread(Runnable)` | 快捷创建并启动虚拟线程 |
| `Thread.isVirtual()` | 判断是否虚拟线程 |
| `Thread.getAllStackTraces()` | 现仅返回平台线程 |

### java.util.concurrent

| 方法 | 说明 |
|------|------|
| `Executors.newVirtualThreadPerTaskExecutor()` | 每任务创建虚拟线程的 ExecutorService |
| `Executors.newThreadPerTaskExecutor(ThreadFactory)` | 通用每任务线程创建器 |
| `LockSupport.park/unpark` | 支持虚拟线程挂起/恢复 |

### 虚拟线程特性限制

- 始终是 daemon 线程，`setDaemon(false)` 无效
- 优先级固定为 `NORM_PRIORITY`，`setPriority()` 无效
- 不属于 ThreadGroup，`getThreadGroup()` 返回 `"VirtualThreads"` 占位组
- SecurityManager 下无权限

---

## 版本历史

| 版本 | 状态 | JEP |
|------|------|-----|
| Java 19 | 预览 | JEP 425 |
| Java 20 | 第二次预览 | JEP 436 |
| Java 21 | **正式发布** | JEP 444 |
| Java 25 | 结构化并发正式 | JEP 453 |

---

## 参考资料

- [JEP 444: Virtual Threads](https://openjdk.org/jeps/444) — 官方规范
- [Project Loom Wiki](https://wiki.openjdk.org/display/loom/Main) — 项目主页
- [Oracle: Virtual Threads Guide](https://docs.oracle.com/en/java/javase/21/core/virtual-threads.html) — 官方教程
- [Ron Pressler: Loom Proposal](https://cr.openjdk.org/~rpressler/loom/Loom-Proposal.html) — 设计提案