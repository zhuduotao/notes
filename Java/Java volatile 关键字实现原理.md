---
created: '2026-04-21'
updated: '2026-04-21'
tags:
  - Java
  - 并发编程
  - JMM
  - volatile
  - 内存模型
aliases:
  - Java volatile
  - volatile 关键字
  - 内存可见性
source_type: spec
source_urls:
  - 'https://docs.oracle.com/javase/specs/jls/se21/html/jls-17.html'
  - 'https://www.cs.umd.edu/~pugh/java/memoryModel/jsr-133-faq.html'
  - 'https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-2.html'
  - 'https://gee.cs.oswego.edu/dl/jmm/cookbook.html'
status: verified
---
## 概述

`volatile` 是 Java 提供的**轻量级同步机制**，用于保证多线程环境下共享变量的**可见性**和**禁止指令重排序**。与 `synchronized` 不同，`volatile` **不保证原子性**，适用于"一写多读"或状态标记等特定场景。

---

## JMM 内存模型背景

### 为什么需要内存模型

现代处理器普遍使用多级缓存来提升性能，这带来一个问题：多个线程可能同时访问同一内存位置，但各自看到不同的值（缓存不一致）。此外，编译器、JIT、处理器都可能对指令进行重排序优化。

Java Memory Model (JMM) 定义了多线程程序中变量读写的**合法行为**，确保正确同步的程序在任意硬件架构上都能正确运行。

### 核心概念

| 术语 | 定义 |
|------|------|
| **主内存** | 所有线程共享的内存区域 |
| **工作内存** | 每个线程私有的缓存区域（寄存器、CPU 缓存） |
| **同步动作** | volatile 读/写、lock/unlock 等能影响线程间可见性的操作 |

---

## volatile 的三大语义

### 1. 可见性（Visibility）

- **写入**：立即刷新到主内存，其他线程能立即看到
- **读取**：强制从主内存读取，不使用本地缓存

> JLS §17.4.4：对 volatile 变量 `v` 的写入 **synchronizes-with** 所有后续对 `v` 的读取（"后续"按同步顺序定义）。

### 2. 禁止重排序（Ordering）

`volatile` 变量的访问不能与普通变量访问重排序。具体规则：

| 操作类型 | 限制 |
|----------|------|
| volatile 写之前 | 不能重排序到 volatile 写之后 |
| volatile 读之后 | 不能重排序到 volatile 读之前 |
| volatile 写之间 | 不能互相重排序 |
| volatile 读之间 | 不能互相重排序 |

> JSR-133 FAQ：volatile 写具有与 monitor release 相同的内存效果，volatile 读具有与 monitor acquire 相同的内存效果。

### 3. 不保证原子性（No Atomicity）

`volatile` 无法保证复合操作的原子性：

```java
volatile int count = 0;
count++;  // 不安全！包含 read-modify-write 三步
```

---

## happens-before 规则

JMM 通过 **happens-before** 关系定义操作间的可见性顺序。volatile 相关规则：

| 规则 | 说明 |
|------|------|
| 程序顺序规则 | 同一线程中，前面的操作 happens-before 后面的操作 |
| volatile 规则 | volatile 写 **happens-before** 后续的 volatile 读 |
| 传递性 | 若 A happens-before B，且 B happens-before C，则 A happens-before C |

**示例**：

```java
int x = 0;
volatile boolean ready = false;

// Thread 1 (writer)
x = 42;          // (1)
ready = true;    // (2) volatile 写

// Thread 2 (reader)
if (ready) {     // (3) volatile 读
    int r = x;   // (4) 必定看到 42
}
```

执行顺序：
- (1) happens-before (2)（程序顺序）
- (2) happens-before (3)（volatile 规则）
- (3) happens-before (4)（程序顺序）
- 由传递性：(1) happens-before (4)，因此线程 2 必定看到 x = 42

---

## 底层实现：内存屏障

JVM 通过插入**内存屏障（Memory Barrier / Fence）**来实现 volatile 语义。屏障类型：

| 屏障类型 | 作用 | 插入位置 |
|----------|------|----------|
| **LoadLoad** | 禁止 Load1 与 Load2 重排序 | volatile 读后 |
| **LoadStore** | 禁止 Load 与 Store 重排序 | volatile 读后 |
| **StoreStore** | 禁止 Store1 与 Store2 重排序 | volatile 写前 |
| **StoreLoad** | 禁止 Store 与 Load 重排序 | volatile 写后（开销最大） |

### volatile 写的屏障插入策略

```
StoreStore 屏障  ← 禁止前面的普通写与 volatile 写重排序
volatile 写
StoreLoad 屏障   ← 禁止 volatile 写与后面的 volatile 读/普通读重排序
```

### volatile 读的屏障插入策略

```
volatile 读
LoadLoad 屏障    ← 禁止 volatile 读与后面的普通读重排序
LoadStore 屏障   ← 禁止 volatile 读与后面的普通写重排序
```

> 参考：Doug Lea 的《JMM Cookbook》 — https://gee.cs.oswego.edu/dl/jmm/cookbook.html

### 不同硬件架构的实现差异

| 架构 | 特点 |
|------|------|
| x86/x64 | 强内存模型，仅需 StoreLoad 屏障（mfence 指令） |
| ARM/PowerPC | 弱内存模型，可能需要全部四种屏障 |

---

## volatile 与 synchronized 对比

| 维度 | volatile | synchronized |
|------|----------|--------------|
| 保证可见性 | ✅ | ✅ |
| 保证原子性 | ❌ | ✅ |
| 禁止重排序 | ✅（仅 volatile 变量相关） | ✅（锁内所有操作） |
| 阻塞线程 | ❌ | ✅ |
| 适用场景 | 状态标记、一写多读 | 复合操作、读写混合 |
| 性能开销 | 较低（无阻塞） | 较高（可能涉及上下文切换） |

**关键差异**：

- `volatile` 是"半同步"：仅建立读写间的 happens-before 关系
- `synchronized` 是"完整同步"：建立 lock/unlock 间的 happens-before，并保证互斥访问

---

## 适用场景

### ✅ 掁合场景

1. **状态标记位**：一写多读的布尔标志
2. **单例模式的 DCL 优化**：配合 volatile 防止指令重排序（见下方）
3. **读多写少的计数器**：仅由一个线程更新，其他线程读取

### ❌ 不适用场景

1. **多线程并发更新**：如 `count++` 等复合操作
2. **依赖当前值的新值**：如 `x = x + 1`（需要先读再写）
3. **需要互斥访问的资源**：必须使用锁

---

## 常见误区

### 误区 1：volatile 能替代锁

`volatile` 仅保证单次读写的可见性，无法保证复合操作的原子性。正确做法：

```java
// ❌ 错误：volatile 无法保证原子性
volatile int counter = 0;
counter++;

// ✅ 正确：使用 AtomicInteger 或锁
AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet();
```

### 误区 2：volatile 变量对所有变量可见

`volatile` 仅建立对同一 volatile 变量的 happens-before 关系。不同 volatile 变量间无保证：

```java
volatile int a = 0;
volatile int b = 0;

// Thread 1
a = 1;
b = 2;

// Thread 2
int r1 = b;  // 可能看到 2
int r2 = a;  // 可能仍看到 0（非同一 volatile）
```

### 误区 3：volatile 能保证对象内部字段的可见性

`volatile` 仅保证引用本身的可见性，不保证引用对象内部字段：

```java
volatile MutableObject obj = new MutableObject();
// 其他线程能看到 obj 引用的最新值
// 但 obj 内部的非 volatile 字段仍可能不可见
```

---

## 双重检查锁定（DCL）的正确实现

DCL 在旧 JMM 下有缺陷（对象可能未完全初始化）。JSR-133 后，配合 volatile 可修复：

```java
public class Singleton {
    private static volatile Singleton instance;
    
    public static Singleton getInstance() {
        if (instance == null) {             // 第一次检查（无锁）
            synchronized (Singleton.class) {
                if (instance == null) {     // 第二次检查（有锁）
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

**volatile 的作用**：防止 `instance = new Singleton()` 的指令重排序（对象初始化与引用赋值可能被重排，导致其他线程看到未完全初始化的对象）。

**替代方案**：使用静态内部类（更简洁且无需 volatile）：

```java
public class Singleton {
    private static class Holder {
        static final Singleton INSTANCE = new Singleton();
    }
    
    public static Singleton getInstance() {
        return Holder.INSTANCE;
    }
}
```

---

## 最佳实践

1. **仅用于状态标记**：如 `volatile boolean running` 作为线程停止标志
2. **配合原子类使用**：复杂场景优先选择 `AtomicInteger`、`AtomicReference`
3. **避免依赖 volatile 做复杂同步**：需要原子性时使用 `synchronized` 或 JUC 锁
4. **注意可见性范围**：volatile 仅保证引用可见，内部字段需单独处理
5. **优先使用更高层抽象**：如 `ConcurrentHashMap`、`BlockingQueue` 等已内置正确同步

---

## 参考资料

- Oracle Java 语言规范 §17 — Threads and Locks：https://docs.oracle.com/javase/specs/jls/se21/html/jls-17.html
- JSR-133 FAQ（Java Memory Model）：https://www.cs.umd.edu/~pugh/java/memoryModel/jsr-133-faq.html
- Oracle JVM 规范 §2 — The Structure of the Java Virtual Machine：https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-2.html
- Doug Lea — JMM Cookbook：https://gee.cs.oswego.edu/dl/jmm/cookbook.html
- JSR-133：https://jcp.org/en/jsr/detail?id=133
