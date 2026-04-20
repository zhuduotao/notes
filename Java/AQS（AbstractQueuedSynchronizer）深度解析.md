---
created: '2026-04-20'
updated: '2026-04-20'
tags:
  - Java
  - 并发编程
  - AQS
  - JUC
  - 锁机制
aliases:
  - AbstractQueuedSynchronizer
  - Java AQS
  - AQS 框架
source_type: mixed
source_urls:
  - >-
    https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/locks/AbstractQueuedSynchronizer.html
  - >-
    https://github.com/openjdk/jdk/blob/master/src/java.base/share/classes/java/util/concurrent/locks/AbstractQueuedSynchronizer.java
status: verified
---
## 概述

**AQS（AbstractQueuedSynchronizer）** 是 Java 并发包 `java.util.concurrent.locks` 的核心基础设施，由 Doug Lea 在 JDK 1.5 中引入。它是一个用于构建锁和同步器的**模板框架**，提供了：

- **同步状态管理**：通过 `volatile int state` 表示锁/信号量等的状态
- **线程排队机制**：基于 CLH 队列变体实现等待线程的 FIFO 排队
- **阻塞/唤醒机制**：使用 `LockSupport.park/unpark` 实现线程的挂起与恢复

几乎所有 JUC 中的同步器都基于 AQS 实现：

| 同步器 | AQS 使用方式 | state 语义 |
|--------|-------------|-----------|
| `ReentrantLock` | 独占模式 | 持有次数（0=未锁定，N=重入次数） |
| `ReentrantReadWriteLock` | 共享 + 独占模式 | 高16位=读锁计数，低16位=写锁计数 |
| `Semaphore` | 共享模式 | 可用许可数 |
| `CountDownLatch` | 共享模式 | 待完成事件数（减至0时唤醒所有等待线程） |
| `FutureTask` | 独占模式 | 任务状态（0=未开始，1=完成，2=取消等） |

---

## 核心设计

### 三大支柱

1. **state 同步状态**
2. **CLH 队列（等待队列）**
3. **模板方法模式（钩子方法）**

---

### state 同步状态

```java
private volatile int state;
```

- `volatile` 保证**可见性**：所有线程能看到 state 的最新值
- 通过 `getState()`、`setState(int)`、`compareAndSetState(int, int)` 操作
- `compareAndSetState` 使用 **CAS（Compare-And-Swap）** 保证原子性

不同同步器赋予 state 不同语义：

| 同步器 | state 含义 |
|--------|-----------|
| `ReentrantLock` | 0=未锁定，N=锁被持有且重入N次 |
| `ReentrantReadWriteLock` | 高16位=读锁持有数，低16位=写锁重入数 |
| `Semaphore` | 可用信号量数量 |
| `CountDownLatch` | 计数器值（N→0触发唤醒） |

---

### CLH 队列（等待队列）

AQS 使用 **CLH（Craig, Landin, and Hagersten）队列变体**作为等待队列：

```java
static final class Node {
    volatile int waitStatus;     // 节点状态
    volatile Node prev;          // 前驱节点
    volatile Node next;          // 后继节点
    volatile Thread thread;      // 关联线程
    Node nextWaiter;             // 模式标记（SHARED/EXCLUSIVE）或条件队列后继
}
```

#### 队列结构

```
          +------+  prev  +-----+  prev  +-----+
   HEAD --| Node | <----- | Node| <----- | Node| -- TAIL
          +------+        +-----+        +-----+
             ↓               ↓              ↓
          dummy node      等待线程1       等待线程2
          (空节点)        (park中)        (park中)
```

- **HEAD**：虚拟节点（dummy node），不存储实际线程
- **TAIL**：队列尾部，新节点通过 CAS 追加到此
- 每个 Node 封装一个等待线程

#### waitStatus 状态值

| 值 | 常量名 | 含义 |
|----|--------|------|
| 0 | — | 初始状态 |
| 1 | `CANCELLED` | 线程已取消（因超时或中断） |
| -1 | `SIGNAL` | 后继节点需要被唤醒（当前节点释放时通知后继） |
| -2 | `CONDITION` | 节点在条件队列中等待（`Condition.await()`） |
| -3 | `PROPAGATE` | 共享模式下的传播状态（释放共享锁时向后传播） |

---

### 模板方法模式

AQS 定义了一系列**模板方法**（final 方法），子类通过重写**钩子方法**（protected 方法）实现特定同步语义：

#### 模板方法（AQS 定义，不可重写）

| 方法 | 功能 |
|------|------|
| `acquire(int arg)` | 独占获取锁（不响应中断） |
| `acquireInterruptibly(int arg)` | 独占获取锁（响应中断） |
| `tryAcquireNanos(int arg, long nanos)` | 独占获取锁（响应中断+超时） |
| `release(int arg)` | 独占释放锁 |
| `acquireShared(int arg)` | 共享获取锁 |
| `acquireSharedInterruptibly(int arg)` | 共享获取锁（响应中断） |
| `tryAcquireSharedNanos(int arg, long nanos)` | 共享获取锁（响应中断+超时） |
| `releaseShared(int arg)` | 共享释放锁 |

#### 钩子方法（子类重写）

| 方法 | 说明 | 默认实现 |
|------|------|---------|
| `tryAcquire(int arg)` | 尝试独占获取 | 抛出 `UnsupportedOperationException` |
| `tryRelease(int arg)` | 尝试独占释放 | 抛出 `UnsupportedOperationException` |
| `tryAcquireShared(int arg)` | 尝试共享获取 | 抛出 `UnsupportedOperationException` |
| `tryReleaseShared(int arg)` | 尝试共享释放 | 抛出 `UnsupportedOperationException` |
| `isHeldExclusively()` | 是否被独占持有 | 抛出 `UnsupportedOperationException` |

> 子类只需重写其需要的钩子方法。例如 `ReentrantLock` 只需重写 `tryAcquire/tryRelease/isHeldExclusively`，`Semaphore` 只需重写 `tryAcquireShared/tryReleaseShared`。

---

## 独占模式工作流程

以 `ReentrantLock`（非公平锁）为例：

### 获取锁流程（acquire）

```
┌─────────────────────────────────────────────────────────────────┐
│                      acquire(1) 流程                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. tryAcquire(1)                                               │
│     ├── CAS 尝试将 state 从 0 改为 1                            │
│     ├── 成功 → 设置独占线程为当前线程 → 返回 true → 获取锁完成  │
│     └── 失败 → 判断当前线程是否已持有锁（重入）                  │
│         ├── 是 → state++ → 返回 true                            │
│         └── 否 → 返回 false                                     │
│                                                                 │
│  2. tryAcquire 失败后：                                         │
│     addWaiter(Node.EXCLUSIVE)                                   │
│     ├── 创建 Node 包装当前线程                                  │
│     ├── CAS 将 Node 追加到 CLH 队列尾部                         │
│     └── 返回 Node                                               │
│                                                                 │
│  3. acquireQueued(node, 1)                                      │
│     ├── 死循环：                                                │
│     │   ├── 若前驱是 HEAD → 再次 tryAcquire                     │
│     │   │   ├── 成功 → 设置自己为 HEAD → 返回                   │
│     │   │   └── 失败 → 继续                                     │
│     │   ├── 若前驱不是 HEAD → shouldParkAfterFailedAcquire      │
│     │   │   ├── 将前驱 waitStatus 设为 SIGNAL                   │
│     │   │   └── LockSupport.park() 挂起当前线程                 │
│     │   └── 被唤醒后继续循环                                    │
│     └── 返回中断状态（若在等待期间被中断）                       │
│                                                                 │
│  4. 若 acquireQueued 返回中断状态 → selfInterrupt() 补发中断    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 释放锁流程（release）

```
┌─────────────────────────────────────────────────────────────────┐
│                      release(1) 流程                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. tryRelease(1)                                               │
│     ├── 判断当前线程是否持有锁（非持有者抛异常）                │
│     ├── state--                                                 │
│     ├── 若 state == 0 → 设置独占线程为 null → 返回 true         │
│     └── 若 state > 0 → 返回 false（仍持有锁）                   │
│                                                                 │
│  2. 若 tryRelease 返回 true：                                   │
│     ├── 检查 HEAD 的 waitStatus                                 │
│     ├── 若 waitStatus == SIGNAL → unparkSuccessor(head)        │
│     │   ├── 找到后继节点（skip CANCELLED 节点）                 │
│     │   └── LockSupport.unpark(后继线程)                       │
│     └── 后继线程被唤醒，继续 acquireQueued 中的循环             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 共享模式工作流程

以 `Semaphore` 为例：

### 获取共享锁流程（acquireShared）

```
┌─────────────────────────────────────────────────────────────────┐
│                  acquireShared(1) 流程                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. tryAcquireShared(1)                                         │
│     ├── 尝试获取 1 个许可                                       │
│     ├── 若剩余许可 >= 1 → 返回剩余许可数（正数=成功）           │
│     └── 若剩余许可 < 1 → 返回负数（失败）                       │
│                                                                 │
│  2. 若返回负数（获取失败）：                                    │
│     doAcquireShared(1)                                          │
│     ├── addWaiter(Node.SHARED) 追加到队列                       │
│     ├── 死循环：                                                │
│     │   ├── 若前驱是 HEAD → tryAcquireShared                    │
│     │   │   ├── 返回正数 → setHeadAndPropagate                  │
│     │   │   │   ├── 设置当前 Node 为 HEAD                       │
│     │   │   │   └── propagate（传播唤醒）                       │
│     │   │   │   ├── 若有剩余许可 →唤醒后继共享节点              │
│     │   │   └── 返回负数 → park                                 │
│     │   └── 若前驱不是 HEAD → park                              │
│                                                                 │
│  关键点：setHeadAndPropagate 会向后传播唤醒                     │
│         因为共享锁允许多个线程同时持有                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 释放共享锁流程（releaseShared）

```
┌─────────────────────────────────────────────────────────────────┐
│                  releaseShared(1) 流程                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. tryReleaseShared(1)                                         │
│     ├── state++（归还许可）                                     │
│     ├── 使用 CAS 保证原子性                                     │
│     └── 返回 true（表示可能需要唤醒后继）                       │
│                                                                 │
│  2. 若返回 true：                                               │
│     doReleaseShared()                                           │
│     ├── 检查 HEAD 的 waitStatus                                 │
│     ├── 若是 SIGNAL → CAS 改为 0 → unparkSuccessor             │
│     ├── 若是 0 → CAS 改为 PROPAGATE（传播状态）                 │
│     └── 循环直到不需要唤醒                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Condition 条件队列

AQS 内部实现了 `ConditionObject`，提供类似 `Object.wait/notify` 的功能：

```java
public class ConditionObject implements Condition, java.io.Serializable {
    private transient Node firstWaiter;  // 条件队列头
    private transient Node lastWaiter;   // 条件队列尾
}
```

### 双队列结构

AQS 维护两个队列：

```
┌─────────────────────────────────────────────────────────────────┐
│                       AQS 双队列架构                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   CLH 等待队列（同步队列）                                       │
│   ─────────────────────                                         │
│   HEAD → Node1 → Node2 → Node3 → TAIL                          │
│          (等待获取锁)                                           │
│                                                                 │
│   条件队列（Condition Queue）                                    │
│   ─────────────────────                                         │
│   firstWaiter → NodeA → NodeB → lastWaiter                     │
│                 (等待条件满足)                                   │
│                                                                 │
│   await(): 同步队列 → 条件队列（释放锁 + 入队）                  │
│   signal(): 条件队列 → 同步队列（转移节点等待获取锁）            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### await 流程

```java
public final void await() throws InterruptedException {
    // 1. 检查中断
    if (Thread.interrupted())
        throw new InterruptedException();
    
    // 2. 创建 Node 加入条件队列尾部
    Node node = addConditionWaiter();
    
    // 3. 完全释放锁（savedState = state, state = 0）
    int savedState = fullyRelease(node);
    
    // 4. 挂起线程，等待 signal 或中断唤醒
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        if (checkInterruptWhileWaiting(node) != 0)
            break;
    }
    
    // 5. 被唤醒后，重新获取锁（恢复 savedState）
    acquireQueued(node, savedState);
}
```

### signal 流程

```java
public final void signal() {
    // 1. 检查当前线程是否持有锁
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    
    // 2. 获取条件队列第一个节点
    Node first = firstWaiter;
    if (first != null)
        doSignal(first);  // 转移到同步队列
}

private void doSignal(Node first) {
    do {
        if ((firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
    } while (!transferForSignal(first) &&  // CAS 转移
             (first = firstWaiter) != null);
}
```

---

## ReentrantLock 源码剖析

### 非公平锁实现（NonfairSync）

```java
static final class NonfairSync extends Sync {
    final void lock() {
        // 直接尝试 CAS 抢锁（非公平：不排队，直接抢）
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
            acquire(1);  // 抢失败才走 acquire 流程
    }

    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }
}

final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        // 无锁，CAS 抢占
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    } else if (current == getExclusiveOwnerThread()) {
        // 已持有锁，重入
        int nextc = c + acquires;
        if (nextc < 0)  // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

### 公平锁实现（FairSync）

```java
static final class FairSync extends Sync {
    final void lock() {
        acquire(1);  // 公平锁：不直接抢，先排队
    }

    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            // 关键区别：检查队列是否有等待者
            if (!hasQueuedPredecessors() &&  // ★ 公平性保证
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        } else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
}

// 公平锁核心检查：队列中是否有等待者排在前面
public final boolean hasQueuedPredecessors() {
    Node h, s;
    if ((h = head) != null) {
        if ((s = h.next) == null || s.thread != Thread.currentThread())
            return true;  // 有等待者
    }
    return false;
}
```

---

## ReentrantReadWriteLock 源码剖析

### state 位设计

```
┌─────────────────────────────────────────────────────────────────┐
│              state 位分布（32位 int）                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌─────────────────┬─────────────────┐                        │
│   │   高16位        │   低16位        │                        │
│   │   读锁计数      │   写锁计数      │                        │
│   │   (共享锁)      │   (独占锁)      │                        │
│   └─────────────────┴─────────────────┘                        │
│                                                                 │
│   读锁计数 = state >>> 16                                        │
│   写锁计数 = state & 0xFFFF                                       │
│                                                                 │
│   示例：                                                         │
│   state = 0x00010001 → 读锁=1, 写锁=1（写锁持有且有一个读锁）   │
│   state = 0x00030000 → 读锁=3, 写锁=0（三个读锁共享）           │
│   state = 0x00000002 → 读锁=0, 写锁=2（写锁重入两次）           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 读锁获取

```java
protected final int tryAcquireShared(int unused) {
    Thread current = Thread.currentThread();
    int c = getState();
    
    // 写锁被其他线程持有 → 失败
    if (exclusiveCount(c) != 0 &&
        getExclusiveOwnerThread() != current)
        return -1;
    
    // 读锁计数
    int r = sharedCount(c);
    
    // 公平性检查 + CAS 增加读锁计数
    if (!readerShouldBlock() &&
        r < MAX_COUNT &&  // 65535
        compareAndSetState(c, c + SHARED_UNIT)) {
        // 记录读锁持有者（cachedHoldCounter 等）
        return 1;
    }
    
    // CAS 失败或需要排队 → fullTryAcquireShared
    return fullTryAcquireShared(current);
}
```

### 写锁获取

```java
protected final boolean tryAcquire(int acquires) {
    Thread current = Thread.currentThread();
    int c = getState();
    int w = exclusiveCount(c);
    
    if (c != 0) {
        // 有锁（读或写）
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;  // 读锁被持有 或 写锁被其他线程持有
        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        setState(c + acquires);  // 重入
        return true;
    }
    
    // 无锁，尝试获取
    if (writerShouldBlock() ||
        !compareAndSetState(c, c + acquires))
        return false;
    setExclusiveOwnerThread(current);
    return true;
}
```

---

## 自定义同步器示例

实现一个简单的**一次性闸门**（类似 `CountDownLatch(1)`）：

```java
public class OneShotLatch {
    private final Sync sync = new Sync();
    
    public void signal() {
        sync.releaseShared(1);
    }
    
    public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }
    
    private class Sync extends AbstractQueuedSynchronizer {
        @Override
        protected int tryAcquireShared(int ignored) {
            // state == 1 表示闸门已打开，返回 1（成功）
            // state == 0 表示闸门关闭，返回 -1（失败）
            return (getState() == 1) ? 1 : -1;
        }
        
        @Override
        protected boolean tryReleaseShared(int ignored) {
            // 打开闸门
            setState(1);
            return true;  // 告诉 AQS 需要唤醒等待者
        }
    }
}
```

---

## 关键技术点

### CAS 与 LockSupport

AQS 依赖两个底层机制：

| 机制 | 类 | 作用 |
|------|-----|------|
| **CAS** | `Unsafe.compareAndSwapInt` | 原子更新 state、队列节点指针 |
| **Park/Unpark** | `LockSupport.park/unpark` | 线程挂起与唤醒（不依赖 Monitor） |

#### LockSupport 特点

- `park()` 可以被 `unpark()` 先行调用激活（许可证机制）
- `park()` 响应中断但不抛异常（需检查中断状态）
- 精确唤醒指定线程（不像 `Object.notify` 随机）

### 内存屏障

AQS 的 `volatile` 变量配合 CAS 提供了必要的内存屏障：

- **volatile 读**：`getState()` — 确保看到最新值
- **volatile 写**：`setState(int)` — 确保写入可见
- **CAS**：`compareAndSetState` — 同时具有 volatile 读/写语义

---

## 常见问题

### 1. 为什么用 CLH 队列而非简单链表？

CLH 队列的特点：
- 每个节点只需检查**前驱节点的状态**判断是否需要等待
- 避免了传统队列中的竞争（多线程同时检查队头）
- 前驱释放锁时只需唤醒**一个后继**，减少唤醒风暴

### 2. 公平锁和非公平锁的性能差异？

| 维度 | 公平锁 | 非公平锁 |
|------|--------|---------|
| 吞吐量 | 较低（严格排队） | 较高（允许抢锁） |
| 线程饥饿 | 无 | 可能发生 |
| 上下文切换 | 较多 | 较少 |
| 适用场景 | 需要严格顺序 | 高并发通用场景 |

**非公平锁优势**：刚释放锁的线程大概率立即再次获取（锁的持有时间通常很短），避免不必要的上下文切换。

### 3. AQS 如何处理中断？

| 方法 | 中断处理 |
|------|---------|
| `acquire()` | 不响应中断，但记录中断状态，获取锁后补发 `selfInterrupt()` |
| `acquireInterruptibly()` | 检测中断立即抛 `InterruptedException` |
| `tryAcquireNanos()` | 超时或中断都立即返回 |

### 4. 为什么 HEAD 是虚拟节点？

- HEAD 初始化时 `new Node()`，thread 为 null
- 成功获取锁的线程成为 HEAD，但**不再存储线程对象**
- HEAD 的 `waitStatus` 用于指示后继是否需要唤醒
- 避免了 HEAD 节点被 GC 后无法访问后继的问题

---

## AQS 与 synchronized 对比

| 维度 | synchronized | AQS-based Lock |
|------|-------------|----------------|
| 实现层面 | JVM 内置（Monitor） | JDK 代码实现 |
| 锁释放 | 自动（代码块结束） | 手动（必须在 finally 中 unlock） |
| 公平性 | 仅非公平 | 可配置公平/非公平 |
| 中断响应 | 不支持 | 支持（`lockInterruptibly`） |
| 超时获取 | 不支持 | 支持（`tryLock(timeout)`） |
| 多条件 | 单一 wait set | 多个 Condition 对象 |
| 锁状态查询 | 无 | `isLocked()`, `getQueueLength()` 等 |
| 适用场景 | 简单同步、代码块同步 | 复杂同步需求、需精细控制 |

---

## 总结

AQS 是 JUC 并发包的**基石**，其核心设计思想：

1. **状态驱动**：`volatile int state` 表示同步状态
2. **队列等待**：CLH 队列管理等待线程
3. **模板分离**：模板方法处理公共逻辑，钩子方法由子类实现特定语义
4. **CAS + Park**：无锁原子操作 + 线程阻塞机制

理解 AQS 有助于：
- 深入理解 `ReentrantLock`、`Semaphore`、`CountDownLatch` 等的工作原理
- 构建自定义同步器
- 分析并发问题（如死锁检测、性能优化）

---

## 参考资料

- OpenJDK 源码 — `AbstractQueuedSynchronizer.java`：https://github.com/openjdk/jdk/blob/master/src/java.base/share/classes/java/util/concurrent/locks/AbstractQueuedSynchronizer.java
- Oracle Java SE 21 API — `AbstractQueuedSynchronizer`：https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/locks/AbstractQueuedSynchronizer.html
- Doug Lea — "The java.util.concurrent Synchronizer Framework"（JSR 166）：https://gee.cs.oswego.edu/dl/papers/aqs.pdf
- 《Java Concurrency in Practice》— Brian Goetz 等
