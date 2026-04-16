---
created: '2026-04-15'
updated: '2026-04-15'
tags:
  - Java
  - 并发编程
  - 锁机制
  - JUC
  - JVM
aliases:
  - Java Locks
  - Java 并发锁
  - Java Lock Mechanism
source_type: mixed
source_urls:
  - >-
    https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/locks/package-summary.html
  - 'https://openjdk.org/'
  - 'https://docs.oracle.com/javase/specs/jls/se21/html/index.html'
  - 'https://docs.oracle.com/javase/specs/jvms/se21/html/index.html'
status: verified
---

## 概述

Java 中的锁是**多线程并发控制**的核心机制，用于保证共享资源在并发访问时的**原子性、可见性和有序性**。Java 锁体系从语言层面的 `synchronized` 到 `java.util.concurrent.locks`（简称 JUC locks）包下的显式锁，形成了完整的并发控制工具链。

---

## 锁的分类维度

### 按获取策略

| 分类 | 定义 | 典型代表 |
|------|------|----------|
| **悲观锁** | 假设冲突必然发生，访问前直接加锁 | `synchronized`、`ReentrantLock` |
| **乐观锁** | 假设冲突很少发生，通过 CAS + 版本号实现，失败则重试 | `AtomicInteger`、`StampedLock` 的乐观读 |

### 按是否可重入

| 分类 | 定义 | 说明 |
|------|------|------|
| **可重入锁** | 同一线程可多次获取同一把锁，不会死锁 | `synchronized`、`ReentrantLock` 内部维护持有计数器 |
| **不可重入锁** | 同一线程第二次获取会阻塞或失败 | Java 标准库中较少直接使用 |

### 按公平性

| 分类 | 定义 | 特点 |
|------|------|------|
| **公平锁** | 按线程请求锁的顺序分配 | 避免线程饥饿，但吞吐量较低 |
| **非公平锁** | 允许插队，新线程可能先于等待线程获取锁 | 吞吐量高，但可能导致部分线程长期等待 |

> `ReentrantLock` 默认使用**非公平锁**，可通过构造函数参数 `new ReentrantLock(true)` 切换为公平锁。`synchronized` 始终是非公平的。

### 按锁的状态升级（JVM 层面）

JVM 对 `synchronized` 做了大量优化，锁会随竞争程度逐步升级：

```
无锁 → 偏向锁 → 轻量级锁 → 重量级锁
```

升级过程**不可逆**（Java 15 起偏向锁默认禁用，见下方说明）。

---

## synchronized 关键字

### 基本用法

```java
// 实例方法锁：锁当前实例对象 (this)
public synchronized void method() { }

// 静态方法锁：锁 Class 对象
public static synchronized void staticMethod() { }

// 代码块锁：锁指定对象
public void method() {
    synchronized (this) {
        // 临界区
    }
}
```

### 底层实现原理

`synchronized` 的底层依赖 **JVM 的 Monitor 对象** 和 **字节码指令**：

- **同步代码块**：通过 `monitorenter` 和 `monitorexit` 字节码指令实现
- **同步方法**：通过 `ACC_SYNCHRONIZED` 方法标志位实现，JVM 自动处理

每个 Java 对象头（Object Header）中包含 **Mark Word**，存储锁状态信息：

| 锁状态 | Mark Word 存储内容（64 位 JVM） |
|--------|--------------------------------|
| 无锁 | hashCode、分代年龄（4 bit）、偏向锁标志（0）、锁标志位（01） |
| 偏向锁 | 线程 ID、Epoch、分代年龄、偏向锁标志（1）、锁标志位（01） |
| 轻量级锁 | 指向线程栈中 Lock Record 的指针、锁标志位（00） |
| 重量级锁 | 指向 Monitor 对象的指针、锁标志位（10） |

### 锁升级机制

1. **偏向锁（Biased Locking）**
   - 目标：消除无竞争情况下的同步开销
   - 机制：Mark Word 记录首次获取锁的线程 ID，该线程再次进入时无需 CAS
   - 撤销条件：其他线程尝试获取锁时触发撤销，回到轻量级锁
   - **Java 15 起默认禁用**（`-XX:+UseBiasedLocking` 不再默认开启），因为现代应用中偏向锁的撤销开销往往大于收益

2. **轻量级锁（Thin Lock）**
   - 目标：在竞争不激烈时避免重量级锁的系统调用
   - 机制：线程在栈帧中创建 Lock Record，通过 CAS 将 Mark Word 替换为指向 Lock Record 的指针
   - 失败后：自旋一定次数，仍失败则膨胀为重量级锁

3. **重量级锁（Fat Lock / Monitor）**
   - 目标：处理高竞争场景
   - 机制：依赖操作系统的 Mutex Lock，未获取锁的线程进入阻塞状态（`Object.wait()` 语义）
   - 缺点：线程上下文切换开销大

### 注意事项

- `synchronized` **不可中断**：等待锁的线程无法响应 `Thread.interrupt()`
- `synchronized` **不支持超时**：无法设置获取锁的等待时间上限
- `synchronized` **锁粒度固定**：只能锁对象或 Class，无法实现读写分离

---

## ReentrantLock

`ReentrantLock` 是 `java.util.concurrent.locks.Lock` 接口的标准实现，基于 **AQS（AbstractQueuedSynchronizer）** 构建。

### 核心特性

| 特性 | 说明 |
|------|------|
| 可重入 | 同一线程可多次 `lock()`，需对应次数 `unlock()` |
| 公平/非公平可选 | 构造函数 `ReentrantLock(boolean fair)` 控制 |
| 可中断 | `lockInterruptibly()` 响应中断 |
| 超时获取 | `tryLock(long timeout, TimeUnit unit)` |
| 多 Condition | 可创建多个 `Condition` 对象，实现精细的线程等待/通知 |

### 基本用法

```java
ReentrantLock lock = new ReentrantLock();
try {
    lock.lock();
    // 临界区
} finally {
    lock.unlock(); // 必须在 finally 中释放
}
```

### 与 synchronized 对比

| 维度 | synchronized | ReentrantLock |
|------|-------------|---------------|
| 实现层面 | JVM 内置（字节码 + Monitor） | JDK 代码实现（AQS） |
| 锁释放 | 自动释放（方法结束或异常） | 必须手动 `unlock()` |
| 公平性 | 仅非公平 | 可选公平/非公平 |
| 可中断 | 不支持 | 支持（`lockInterruptibly()`） |
| 超时 | 不支持 | 支持（`tryLock(timeout)`） |
| Condition | 单一（`wait/notify`） | 多个 Condition 对象 |
| 性能 | Java 6 后优化显著，差异不大 | 高竞争下略优 |

---

## AQS（AbstractQueuedSynchronizer）

AQS 是 JUC 并发包的**核心基础设施**，`ReentrantLock`、`CountDownLatch`、`Semaphore`、`ReentrantReadWriteLock` 等均基于 AQS 实现。

### 核心设计

- **state 变量**：`volatile int` 表示同步状态（锁的持有次数、信号量计数等）
- **CLH 队列**：双向链表，存储等待获取锁的线程
- **模板方法模式**：子类通过重写 `tryAcquire`、`tryRelease`、`tryAcquireShared`、`tryReleaseShared` 实现自定义同步语义

### ReentrantLock 中的 AQS 工作流（非公平锁）

```
1. 线程调用 lock()
2. CAS 尝试将 state 从 0 改为 1
3. 成功 → 获取锁，记录独占线程
4. 失败 → 判断是否为当前线程（重入）
   - 是 → state++，返回
   - 否 → 封装 Node 加入 CLH 队列尾部，挂起线程
5. 前驱节点释放锁时唤醒后继节点
```

---

## ReadWriteLock 与 ReentrantReadWriteLock

`ReadWriteLock` 维护**一对锁**：读锁（共享）和写锁（独占）。

### 锁规则

| 场景 | 是否允许并发 |
|------|-------------|
| 读 + 读 | ✅ 允许（共享锁） |
| 读 + 写 | ❌ 互斥 |
| 写 + 写 | ❌ 互斥 |

### 适用场景

- **读多写少**：缓存系统、配置中心
- 读操作频繁且耗时较长

### 基本用法

```java
ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();

// 读操作
rwLock.readLock().lock();
try {
    // 读取共享数据
} finally {
    rwLock.readLock().unlock();
}

// 写操作
rwLock.writeLock().lock();
try {
    // 修改共享数据
} finally {
    rwLock.writeLock().unlock();
}
```

### 公平性策略

- **公平模式**：按请求顺序分配，写线程不会饥饿
- **非公平模式（默认）**：允许读锁插队，可能写线程饥饿

### 锁降级

`ReentrantReadWriteLock` 支持**锁降级**（写锁 → 读锁），但**不支持锁升级**（读锁 → 写锁）：

```java
// 锁降级：合法
rwLock.writeLock().lock();
try {
    // 修改数据
    rwLock.readLock().lock(); // 获取读锁
} finally {
    rwLock.writeLock().unlock(); // 释放写锁，此时仍持有读锁
}
// 使用读锁读取数据...
rwLock.readLock().unlock();
```

> 锁升级会导致死锁：线程持有读锁时尝试获取写锁，会阻塞自己（因为写锁需要等待所有读锁释放）。

---

## StampedLock（Java 8 引入）

`StampedLock` 是 Java 8 引入的更灵活的读写锁，性能优于 `ReentrantReadWriteLock`。

### 三种模式

| 模式 | 方法 | 特点 |
|------|------|------|
| **写锁** | `writeLock()` / `unlockWrite(stamp)` | 独占锁，与读锁互斥 |
| **悲观读锁** | `readLock()` / `unlockRead(stamp)` | 共享锁，与写锁互斥 |
| **乐观读** | `tryOptimisticRead()` / `validate(stamp)` | **无锁读取**，通过版本号验证一致性 |

### 乐观读模式

```java
long stamp = lock.tryOptimisticRead();
// 读取数据（无锁）
Data data = readData();
if (!lock.validate(stamp)) {
    // 数据可能被修改过，升级为悲观读锁重试
    stamp = lock.readLock();
    try {
        data = readData();
    } finally {
        lock.unlockRead(stamp);
    }
}
```

### 与 ReadWriteLock 的关键区别

- `StampedLock` **不可重入**
- 乐观读模式下**不阻塞写操作**，仅在 `validate()` 时检测冲突
- 返回的 `stamp`（long 类型）是锁的凭证，解锁时必须传入

---

## 常见锁相关工具类

| 类 | 用途 | 底层 |
|----|------|------|
| `CountDownLatch` | 等待 N 个事件完成 | AQS（共享模式） |
| `CyclicBarrier` | 线程到达屏障后统一放行 | `ReentrantLock` + `Condition` |
| `Semaphore` | 控制并发访问的资源数量 | AQS（共享模式） |
| `ReentrantReadWriteLock` | 读写分离锁 | AQS |
| `StampedLock` | 高性能读写锁（Java 8+） | 自定义实现 |

---

## 常见误区

### 1. 锁住 String 常量

```java
// ❌ 危险：String 常量池导致不同代码块可能锁住同一对象
synchronized ("lock") { }
```

**正确做法**：使用 `private static final Object` 作为锁对象。

### 2. 忘记在 finally 中释放锁

```java
// ❌ 异常时锁不会释放
lock.lock();
doSomething(); // 抛出异常
lock.unlock(); // 不会执行

// ✅ 正确
lock.lock();
try {
    doSomething();
} finally {
    lock.unlock();
}
```

### 3. 误以为 volatile 能替代锁

`volatile` 仅保证**可见性**和**禁止指令重排序**，**不保证原子性**：

```java
volatile int count = 0;
// count++ 不是原子操作（读-改-写三步），多线程下仍不安全
```

### 4. 锁粒度过大或过小

- **过大**：降低并发度（如锁住整个方法但只有一行需要同步）
- **过小**：无法保证复合操作的原子性（如分别锁住 get 和 set）

---

## 最佳实践

1. **优先使用 JUC 工具类**：如 `ConcurrentHashMap`、`AtomicInteger`，避免手动加锁
2. **缩小锁范围**：仅对真正需要同步的代码块加锁
3. **避免锁嵌套**：多把锁交叉使用易导致死锁
4. **使用 tryLock 避免死锁**：设置超时时间，获取失败时回退
5. **读写分离场景优先 ReadWriteLock / StampedLock**
6. **记录锁的持有者**：调试死锁时，`ReentrantLock` 的 `getOwner()` 和 `getQueuedThreads()` 很有帮助

---

## 参考资料

- Oracle Java SE 21 API — `java.util.concurrent.locks` 包文档：https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/locks/package-summary.html
- Oracle Java 语言规范（JLS）— Synchronization：https://docs.oracle.com/javase/specs/jls/se21/html/jls-17.html
- Oracle JVM 规范（JVMS）— `monitorenter` / `monitorexit`：https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-6.html
- OpenJDK 源码 — `AbstractQueuedSynchronizer`：https://github.com/openjdk/jdk/blob/master/src/java.base/share/classes/java/util/concurrent/locks/AbstractQueuedSynchronizer.java
- JEP 374 — Deprecate and Disable Biased Locking（Java 15）：https://openjdk.org/jeps/374
