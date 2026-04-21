---
created: 2026-04-21
updated: 2026-04-21
tags:
  - Java
  - 并发编程
  - 死锁
  - 活锁
  - 线程安全
  - JUC
aliases:
  - Java deadlock
  - Java livelock
  - Java 死锁示例
  - Java 活锁示例
source_type: official-doc
source_urls:
  - https://docs.oracle.com/javase/tutorial/essential/concurrency/deadlock.html
  - https://docs.oracle.com/javase/specs/jls/se21/html/jls-17.html
  - https://openjdk.org/
status: verified
---

## 是什么

**死锁（Deadlock）**：多个线程相互等待对方释放锁，且都不主动释放自己持有的锁，导致所有线程永久阻塞。

**活锁（Livelock）**：线程持续改变状态、响应事件，但始终无法完成有效工作。线程未阻塞，但反复执行无效操作。

## Coffman 四个必要条件

死锁必须同时满足以下四个条件：

| 条件 | 含义 | Java 中的体现 |
|------|------|---------------|
| **互斥** | 资源不能被多线程同时占用 | `synchronized`、`ReentrantLock` 保证独占 |
| **持有并等待** | 线程持有锁同时等待其他锁 | 在第一个锁内尝试获取第二个锁 |
| **不可抢占** | 已持有的锁不能被强制回收 | Java 锁不支持强制释放（除 `ReentrantLock.tryLock` 超时） |
| **循环等待** | 存在等待环：A 等 B，B 等 A | 不同线程以不同顺序获取同一组锁 |

破坏任一条件即可预防死锁。

---

## 死锁 Demo

### 示例 1：嵌套 synchronized 锁顺序不当

```java
public class DeadlockDemo1 {
    private static final Object lockA = new Object();
    private static final Object lockB = new Object();

    public static void main(String[] args) {
        Thread t1 = new Thread(() -> {
            synchronized (lockA) {
                System.out.println("Thread-1: 持有 lockA，等待 lockB");
                sleep(100); // 让 Thread-2 有时间获取 lockB
                synchronized (lockB) {
                    System.out.println("Thread-1: 同时持有 lockA 和 lockB");
                }
            }
        });

        Thread t2 = new Thread(() -> {
            synchronized (lockB) {
                System.out.println("Thread-2: 持有 lockB，等待 lockA");
                sleep(100); // 让 Thread-1 有时间获取 lockA
                synchronized (lockA) {
                    System.out.println("Thread-2: 同时持有 lockB 和 lockA");
                }
            }
        });

        t1.start();
        t2.start();

        // 程序将卡住，两条打印语句永远不会出现
    }

    private static void sleep(long ms) {
        try { Thread.sleep(ms); } catch (InterruptedException e) {}
    }
}
```

**运行结果**：
```
Thread-1: 持有 lockA，等待 lockB
Thread-2: 持有 lockB，等待 lockA
// 程序卡住，无后续输出
```

**死锁形成过程**：
1. Thread-1 获取 lockA
2. Thread-2 获取 lockB
3. Thread-1 尝试获取 lockB → 阻塞（lockB 被 Thread-2 持有）
4. Thread-2 尝试获取 lockA → 阻塞（lockA 被 Thread-1 持有）
5. 两个线程相互等待，永久阻塞

---

### 示例 2：ReentrantLock 死锁

```java
import java.util.concurrent.locks.ReentrantLock;

public class DeadlockDemo2 {
    private static final ReentrantLock lock1 = new ReentrantLock();
    private static final ReentrantLock lock2 = new ReentrantLock();

    public static void main(String[] args) {
        Thread t1 = new Thread(() -> {
            lock1.lock();
            try {
                System.out.println("Thread-1: 持有 lock1，等待 lock2");
                sleep(100);
                lock2.lock();
                try {
                    System.out.println("Thread-1: 同时持有 lock1 和 lock2");
                } finally {
                    lock2.unlock();
                }
            } finally {
                lock1.unlock();
            }
        });

        Thread t2 = new Thread(() -> {
            lock2.lock();
            try {
                System.out.println("Thread-2: 持有 lock2，等待 lock1");
                sleep(100);
                lock1.lock();
                try {
                    System.out.println("Thread-2: 同时持有 lock2 和 lock1");
                } finally {
                    lock1.unlock();
                }
            } finally {
                lock2.unlock();
            }
        });

        t1.start();
        t2.start();
    }

    private static void sleep(long ms) {
        try { Thread.sleep(ms); } catch (InterruptedException e) {}
    }
}
```

---

### 示例 3：转账场景死锁（经典案例）

```java
public class DeadlockDemo3 {
    static class Account {
        private final int id;
        private int balance;

        Account(int id, int balance) {
            this.id = id;
            this.balance = balance;
        }

        void debit(int amount) { balance -= amount; }
        void credit(int amount) { balance += amount; }
        int getId() { return id; }
    }

    public static void transfer(Account from, Account to, int amount) {
        synchronized (from) {
            System.out.println(Thread.currentThread().getName() + 
                " 锁住账户 " + from.getId() + "，等待账户 " + to.getId());
            sleep(50);
            synchronized (to) {
                from.debit(amount);
                to.credit(amount);
                System.out.println(Thread.currentThread().getName() + 
                    " 完成转账: " + from.getId() + " -> " + to.getId());
            }
        }
    }

    public static void main(String[] args) {
        Account a1 = new Account(1, 1000);
        Account a2 = new Account(2, 1000);

        Thread t1 = new Thread(() -> transfer(a1, a2, 100), "Thread-A");
        Thread t2 = new Thread(() -> transfer(a2, a1, 100), "Thread-B");

        t1.start();
        t2.start();
    }

    private static void sleep(long ms) {
        try { Thread.sleep(ms); } catch (InterruptedException e) {}
    }
}
```

---

## 死锁检测方法

### 1. ThreadMXBean 检测（程序内）

```java
import java.lang.management.ManagementFactory;
import java.lang.management.ThreadMXBean;
import java.lang.management.ThreadInfo;

public class DeadlockDetector {
    public static void detect() {
        ThreadMXBean threadMXBean = ManagementFactory.getThreadMXBean();
        long[] deadlockedThreads = threadMXBean.findDeadlockedThreads();
        
        if (deadlockedThreads != null) {
            System.out.println("检测到死锁！涉及线程：");
            ThreadInfo[] threadInfos = threadMXBean.getThreadInfo(deadlockedThreads);
            for (ThreadInfo info : threadInfos) {
                System.out.println("线程 ID: " + info.getThreadId() + 
                    ", 名称: " + info.getThreadName() +
                    ", 状态: " + info.getThreadState() +
                    ", 等待锁: " + info.getLockName());
            }
        } else {
            System.out.println("未检测到死锁");
        }
    }
}
```

**使用方式**：在死锁 Demo 中定期调用检测：

```java
// 在 DeadlockDemo1 中添加检测线程
Thread detector = new Thread(() -> {
    while (true) {
        sleep(1000);
        DeadlockDetector.detect();
    }
});
detector.setDaemon(true);
detector.start();
```

### 2. jstack 命令（外部工具）

```bash
# 获取 Java 进程 PID
jps -l

# 打印线程栈，检测死锁
jstack <pid>
```

**输出示例**：
```
Found one Java-level deadlock:
=============================
"Thread-1":
  waiting to lock monitor 0x00007f8b2800b608 (object 0x000000076ac8b0d0, a java.lang.Object),
  which is held by "Thread-2"
"Thread-2":
  waiting to lock monitor 0x00007f8b2800b5c8 (object 0x000000076ac8b0c0, a java.lang.Object),
  which is held by "Thread-1"

Java stack information for the threads listed above:
===================================================
"Thread-1":
  at DeadlockDemo1.lambda$main$0(DeadlockDemo1.java:12)
  - waiting to lock <0x000000076ac8b0d0> (a java.lang.Object)
  - locked <0x000000076ac8b0c0> (a java.lang.Object)
"Thread-2":
  at DeadlockDemo1.lambda$main$1(DeadlockDemo1.java:18)
  - waiting to lock <0x000000076ac8b0c0> (a java.lang.Object)
  - locked <0x000000076ac8b0d0> (a java.lang.Object)
```

### 3. VisualVM / JConsole

- 打开 VisualVM，连接到目标进程
- Threads 标签页中查看线程状态
- 死锁线程会标记为红色 "Blocked"

---

## 活锁 Demo

### 示例：ReentrantLock 竞争退避活锁

两个线程竞争同一组锁，失败后立即重试，形成活锁：

```java
import java.util.concurrent.locks.ReentrantLock;

public class LivelockDemo {
    private static final ReentrantLock lockA = new ReentrantLock();
    private static final ReentrantLock lockB = new ReentrantLock();

    public static void main(String[] args) {
        Thread t1 = new Thread(() -> {
            while (true) {
                if (lockA.tryLock()) {
                    System.out.println("Thread-1: 获取 lockA，尝试获取 lockB");
                    if (lockB.tryLock()) {
                        try {
                            System.out.println("Thread-1: 同时持有两把锁，完成任务");
                            break; // 成功完成
                        } finally {
                            lockB.unlock();
                        }
                    } else {
                        System.out.println("Thread-1: lockB 获取失败，释放 lockA 并重试");
                        lockA.unlock();
                    }
                }
                // 不休眠，立即重试 → 可能形成活锁
            }
        }, "Thread-1");

        Thread t2 = new Thread(() -> {
            while (true) {
                if (lockB.tryLock()) {
                    System.out.println("Thread-2: 获取 lockB，尝试获取 lockA");
                    if (lockA.tryLock()) {
                        try {
                            System.out.println("Thread-2: 同时持有两把锁，完成任务");
                            break; // 成功完成
                        } finally {
                            lockA.unlock();
                        }
                    } else {
                        System.out.println("Thread-2: lockA 获取失败，释放 lockB 并重试");
                        lockB.unlock();
                    }
                }
                // 不休眠，立即重试 → 可能形成活锁
            }
        }, "Thread-2");

        t1.start();
        t2.start();
    }
}
```

**可能的输出（活锁）**：
```
Thread-1: 获取 lockA，尝试获取 lockB
Thread-2: 获取 lockB，尝试获取 lockA
Thread-1: lockB 获取失败，释放 lockA 并重试
Thread-2: lockA 获取失败，释放 lockB 并重试
Thread-1: 获取 lockA，尝试获取 lockB
Thread-2: 获取 lockB，尝试获取 lockA
Thread-1: lockB 获取失败，释放 lockA 并重试
Thread-2: lockA 获取失败，释放 lockB 并重试
... (无限循环)
```

**关键点**：
- 线程持续执行（非阻塞）
- CPU 占用高
- 始终无法完成任务

---

### 解决活锁：引入随机退避

```java
import java.util.concurrent.locks.ReentrantLock;

public class LivelockFixedDemo {
    private static final ReentrantLock lockA = new ReentrantLock();
    private static final ReentrantLock lockB = new ReentrantLock();

    public static void main(String[] args) {
        Thread t1 = new Thread(() -> {
            while (true) {
                if (lockA.tryLock()) {
                    try {
                        System.out.println("Thread-1: 获取 lockA，尝试获取 lockB");
                        if (lockB.tryLock()) {
                            try {
                                System.out.println("Thread-1: 同时持有两把锁，完成任务");
                                return;
                            } finally {
                                lockB.unlock();
                            }
                        }
                    } finally {
                        lockA.unlock();
                    }
                }
                // 随机退避，打破同步重试
                sleepRandom(10, 100);
            }
        }, "Thread-1");

        Thread t2 = new Thread(() -> {
            while (true) {
                if (lockB.tryLock()) {
                    try {
                        System.out.println("Thread-2: 获取 lockB，尝试获取 lockA");
                        if (lockA.tryLock()) {
                            try {
                                System.out.println("Thread-2: 同时持有两把锁，完成任务");
                                return;
                            } finally {
                                lockA.unlock();
                            }
                        }
                    } finally {
                        lockB.unlock();
                    }
                }
                sleepRandom(10, 100);
            }
        }, "Thread-2");

        t1.start();
        t2.start();
    }

    private static void sleepRandom(int minMs, int maxMs) {
        try {
            Thread.sleep(minMs + (int)(Math.random() * (maxMs - minMs)));
        } catch (InterruptedException e) {}
    }
}
```

**效果**：随机退避打破两个线程的同步重试模式，最终会有线程成功获取两把锁。

---

## 死锁预防方案

### 1. 统一锁顺序（破坏循环等待）

```java
public class DeadlockFixedDemo {
    static class Account {
        private final int id;
        private int balance;
        Account(int id, int balance) { this.id = id; this.balance = balance; }
        void debit(int amount) { balance -= amount; }
        void credit(int amount) { balance += amount; }
        int getId() { return id; }
    }

    public static void transfer(Account from, Account to, int amount) {
        // 按账户 ID 顺序获取锁，破坏循环等待
        Account first = from.getId() < to.getId() ? from : to;
        Account second = from.getId() < to.getId() ? to : from;

        synchronized (first) {
            synchronized (second) {
                from.debit(amount);
                to.credit(amount);
                System.out.println("转账完成: " + from.getId() + " -> " + to.getId());
            }
        }
    }

    public static void main(String[] args) {
        Account a1 = new Account(1, 1000);
        Account a2 = new Account(2, 1000);

        // 无论谁先启动，锁顺序一致
        new Thread(() -> transfer(a1, a2, 100)).start();
        new Thread(() -> transfer(a2, a1, 100)).start();
    }
}
```

### 2. 使用 tryLock 超时（破坏持有并等待）

```java
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantLock;

public class DeadlockTimeoutDemo {
    private static final ReentrantLock lockA = new ReentrantLock();
    private static final ReentrantLock lockB = new ReentrantLock();

    public static void main(String[] args) {
        Thread t1 = new Thread(() -> {
            try {
                if (lockA.tryLock(100, TimeUnit.MILLISECONDS)) {
                    try {
                        if (lockB.tryLock(100, TimeUnit.MILLISECONDS)) {
                            try {
                                System.out.println("Thread-1: 成功获取两把锁");
                            } finally {
                                lockB.unlock();
                            }
                        } else {
                            System.out.println("Thread-1: 获取 lockB 超时，放弃");
                        }
                    } finally {
                        lockA.unlock();
                    }
                } else {
                    System.out.println("Thread-1: 获取 lockA 超时，放弃");
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        Thread t2 = new Thread(() -> {
            // 类似逻辑...
        });

        t1.start();
        t2.start();
    }
}
```

### 3. 单一全局锁（简化设计）

```java
public class GlobalLockDemo {
    private static final Object globalLock = new Object();

    public static void transfer(Account from, Account to, int amount) {
        synchronized (globalLock) {  // 所有转账操作用同一把锁
            from.debit(amount);
            to.credit(amount);
        }
    }
}
```

**缺点**：降低并发度。

### 4. 使用并发工具类

优先使用 `ConcurrentHashMap`、`AtomicInteger`、`LongAdder` 等无锁或高效并发工具，避免手动加锁。

---

## 死锁与活锁对比

| 维度 | 死锁 | 活锁 |
|------|------|------|
| 线程状态 | BLOCKED / WAITING | RUNNABLE |
| CPU 占用 | 低（线程阻塞） | 高（持续执行无效操作） |
| 检测难度 | 较易（ThreadMXBean、jstack） | 较难（需识别无效进展） |
| 成因 | 循环等待资源 | 退避策略同步、对称竞争 |
| 解决方案 | 统一锁顺序、tryLock 超时 | 随机退避、优先级机制 |

---

## 常见误区

| 误区 | 事实 |
|------|------|
| "可重入锁不会死锁" | 可重入解决同一线程重入问题，但多线程仍可能死锁 |
| "volatile 能防止死锁" | volatile 仅保证可见性，不涉及锁 |
| "活锁就是'活的死锁'" | 活锁是另一种阻塞形式，线程活跃但无进展 |
| "tryLock 一定能避免死锁" | tryLock 超时失败后需合理处理（放弃或重试），否则可能活锁 |

---

## 相关概念

- **Coffman 条件**：死锁四个必要条件，源自 E. G. Coffman 1971 年论文
- **AQS**：`ReentrantLock` 底层实现，通过 CLH 队列管理等待线程
- **ThreadMXBean**：JMX 接口，提供线程监控和死锁检测
- **jstack**：JDK 工具，打印线程栈用于分析死锁

---

## 参考资料

- [Oracle Java Tutorial — Deadlock](https://docs.oracle.com/javase/tutorial/essential/concurrency/deadlock.html)
- [Oracle JLS — Synchronization](https://docs.oracle.com/javase/specs/jls/se21/html/jls-17.html)
- [OpenJDK — ThreadMXBean](https://docs.oracle.com/en/java/javase/21/docs/api/java.management/java/lang/management/ThreadMXBean.html)
- Coffman, E. G., et al. "System Deadlocks" (1971)