---
created: '2026-04-21'
updated: '2026-04-21'
tags:
  - Java
  - 并发编程
  - 线程池
  - ThreadPoolExecutor
aliases:
  - ThreadPoolExecutor
  - ExecutorService
  - Java线程池
source_type: official-doc
source_urls:
  - >-
    https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ThreadPoolExecutor.html
  - >-
    https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Executors.html
status: verified
---
线程池（ThreadPoolExecutor）是 Java 并发编程的核心组件，通过复用线程、统一管理任务队列，降低线程创建销毁开销，同时提供资源边界控制机制。

## 核心价值

线程池解决两类问题[^1]：

- **性能优化**：减少每任务线程创建/销毁开销，适用于大量异步任务场景
- **资源管理**：限制并发线程数量，防止资源耗尽，提供任务队列缓冲

## 核心参数

ThreadPoolExecutor 的核心构造参数（7 个参数）：

```java
public ThreadPoolExecutor(
    int corePoolSize,           // 核心线程数
    int maximumPoolSize,        // 最大线程数
    long keepAliveTime,         // 非核心线程存活时间
    TimeUnit unit,              // 时间单位
    BlockingQueue<Runnable> workQueue,  // 任务队列
    ThreadFactory threadFactory,        // 线程工厂（可选）
    RejectedExecutionHandler handler    // 拒绝策略（可选）
)
```

### 参数详解

| 参数 | 说明 | 生产建议 |
|------|------|----------|
| `corePoolSize` | 核心线程数，即使空闲也保留（除非 `allowCoreThreadTimeOut=true`）[^1] | CPU 密集型：N+1；IO 密集型：2N 或更多 |
| `maximumPoolSize` | 最大线程数，队列满时才扩展到此值[^1] | 根据系统资源上限设定，避免过多线程导致上下文切换开销 |
| `keepAliveTime` | 非核心线程空闲存活时间[^1] | 通常 60s，避免频繁创建销毁 |
| `workQueue` | 任务队列，缓存待执行任务 | 有界队列（防止 OOM）|
| `threadFactory` | 线程创建工厂，可自定义线程名、优先级、守护线程属性 | 自定义命名便于排查问题 |
| `handler` | 拒绝策略，队列满且线程数达上限时触发 | 根据业务选择，避免默认 AbortPolicy 丢失任务 |

## 任务执行流程

任务提交（`execute()`）后的执行顺序[^1]：

1. **线程数 < corePoolSize** → 创建新线程执行任务（即使有空闲线程）
2. **线程数 ≥ corePoolSize** → 任务入队列
3. **队列满 + 线程数 < maximumPoolSize** → 创建新线程
4. **队列满 + 线程数 = maximumPoolSize** → 执行拒绝策略

> 注意：线程池优先复用核心线程，队列满后才扩展非核心线程。这意味着 `maximumPoolSize` 在无界队列场景下永远不会生效。

## 三种队列策略

官方文档定义的三种队列策略[^1]：

### 1. 直接交接（SynchronousQueue）

```java
new ThreadPoolExecutor(n, Integer.MAX_VALUE, 60L, TimeUnit.SECONDS,
                       new SynchronousQueue<Runnable>());
```

- 任务直接传递给线程，不缓存
- 无可用线程时立即创建新线程
- 需要 `maximumPoolSize` 无界，否则任务会被拒绝
- 适用场景：任务间有依赖关系，避免死锁

### 2. 无界队列（LinkedBlockingQueue 无容量上限）

```java
new ThreadPoolExecutor(n, n, 0L, TimeUnit.MILLISECONDS,
                       new LinkedBlockingQueue<Runnable>());
```

- 核心线程满后任务入队列，不再创建新线程
- `maximumPoolSize` 参数无效
- **风险**：任务堆积可能导致 OOM
- 适用场景：独立任务，瞬时高峰缓冲

### 3. 有界队列（ArrayBlockingQueue / LinkedBlockingQueue 指定容量）

```java
new ThreadPoolExecutor(n, m, 60L, TimeUnit.SECONDS,
                       new ArrayBlockingQueue<Runnable>(100));
```

- 队列满后扩展线程至 maximumPoolSize
- 达上限后执行拒绝策略
- **推荐生产使用**，防止资源耗尽
- 调优权衡：大队列+小线程池 → 低 CPU 开销但可能低吞吐；小队列+大线程池 → 高吞吐但高调度开销

## 四种拒绝策略

| 策略 | 行为 | 适用场景 |
|------|------|----------|
| `AbortPolicy`（默认） | 抛出 `RejectedExecutionException`[^1] | 不允许任务丢失，需业务层捕获处理 |
| `CallerRunsPolicy` | 调用者线程执行任务[^1] | 降速反馈控制，保证不丢任务 |
| `DiscardPolicy` | 静默丢弃任务[^1] | 允许丢弃，如日志采集等非关键任务 |
| `DiscardOldestPolicy` | 丢弃队列头部任务并重试[^1] | 优先执行新任务 |

> 生产建议：自定义拒绝策略，结合日志记录、降级处理、任务持久化等。

## Executors 工厂方法的问题

`Executors` 提供便捷工厂方法[^2]，但生产环境需谨慎：

| 方法 | 问题 | 不推荐原因 |
|------|------|------------|
| `newFixedThreadPool(n)` | 使用 `LinkedBlockingQueue`（无界）[^2] | 任务堆积 → OOM |
| `newSingleThreadExecutor()` | 同上[^2] | 同上 |
| `newCachedThreadPool()` | `maximumPoolSize = Integer.MAX_VALUE`[^2] | 线程数无上限 → 资源耗尽 |
| `newScheduledThreadPool(n)` | 同上 | 同上 |

> 阿里巴巴 Java 开发手册强制要求：线程池必须通过 ThreadPoolExecutor 手动创建，禁止使用 Executors 工厂方法[^3]。

## 生产环境最佳实践

### 线程数配置公式（参考）

```
CPU 密集型：N_cpu + 1
IO 密集型：2 * N_cpu 或 N_cpu / (1 - 阻塞系数)
```

其中 `N_cpu = Runtime.getRuntime().availableProcessors()`。

> 公式仅为参考起点，实际需结合压测、监控调整。

### 推荐配置模板

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    Runtime.getRuntime().availableProcessors() + 1,
    Runtime.getRuntime().availableProcessors() * 2,
    60L, TimeUnit.SECONDS,
    new ArrayBlockingQueue<>(200),
    new NamedThreadFactory("biz-pool"),
    new LogAndFallbackHandler()
);

executor.allowCoreThreadTimeOut(true);  // 允许核心线程超时回收
executor.prestartAllCoreThreads();       // 预启动核心线程（启动时队列非空场景）
```

### ThreadFactory 自定义

```java
public class NamedThreadFactory implements ThreadFactory {
    private final AtomicInteger counter = new AtomicInteger(1);
    private final String prefix;
    
    public NamedThreadFactory(String prefix) {
        this.prefix = prefix;
    }
    
    @Override
    public Thread newThread(Runnable r) {
        Thread t = new Thread(r, prefix + "-" + counter.getAndIncrement());
        t.setDaemon(false);
        t.setPriority(Thread.NORM_PRIORITY);
        return t;
    }
}
```

- 自定义线程名便于日志排查、监控识别
- 可设置异常处理器 `t.setUncaughtExceptionHandler()`

### 优雅关闭

```java
executor.shutdown();  // 不再接受新任务，等待队列任务完成
if (!executor.awaitTermination(60, TimeUnit.SECONDS)) {
    executor.shutdownNow();  // 强制终止，返回未执行任务列表
}
```

- `shutdown()`：平滑关闭，等待已提交任务完成[^1]
- `shutdownNow()`：立即终止，中断执行中任务，返回队列任务[^1]
- 应用退出时必须调用，避免线程泄漏

### 监控指标

| 方法 | 说明 |
|------|------|
| `getPoolSize()` | 当前线程数[^1] |
| `getActiveCount()` | 正在执行任务的线程数[^1] |
| `getQueue().size()` | 队列待处理任务数 |
| `getCompletedTaskCount()` | 已完成任务数[^1] |
| `getLargestPoolSize()` | 历史最大线程数[^1] |

建议接入监控系统（如 Micrometer、Prometheus），定期采集指标告警。

## 常见误区

### 误区 1：maximumPoolSize 会立即生效

> **事实**：只有队列满后才会扩展到 maximumPoolSize。无界队列场景该参数永远不生效。

### 误区 2：Executors.newFixedThreadPool 是固定线程数池

> **事实**：核心线程数固定，但队列无界，可能 OOM。真正的固定线程池应使用有界队列 + corePoolSize = maximumPoolSize。

### 误区 3：线程越多性能越好

> **事实**：过多线程增加上下文切换开销、内存占用，反而降低吞吐。需根据任务类型（CPU/IO）和压测结果调优。

### 误区 4：不关闭线程池没事

> **事实**：线程池线程默认非守护线程，即使主线程结束也不会自动终止，导致 JVM 无法退出（除非调用 `shutdown()` 或 `allowCoreThreadTimeOut(true)`）[^1]。

## Java 8+ 新特性

### WorkStealingPool（Java 8）

```java
Executors.newWorkStealingPool();  // 使用 ForkJoinPool 实现
```

- 基于工作窃取算法，适合分治任务
- 并行度默认为 `Runtime.availableProcessors()`[^2]
- 不保证任务执行顺序

## 相关概念

- [[Java 锁机制深度讲解]]：线程池内部使用锁控制任务队列访问
- [[Java Collections Framework：实现与线程安全]]：BlockingQueue 是线程池任务队列的基础
- [[Java 死锁与活锁：原理与 Demo 示例]]：线程池任务间依赖可能导致死锁

## 参考资料

[^1]: Oracle, "ThreadPoolExecutor (Java Platform SE 8)", https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ThreadPoolExecutor.html
[^2]: Oracle, "Executors (Java Platform SE 8)", https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Executors.html
[^3]: 阿里巴巴 Java 开发手册（嵩山版），强制规约「线程池必须通过 ThreadPoolExecutor 手动创建」
