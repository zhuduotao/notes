---
created: 2026-04-22
updated: 2026-04-22
tags:
  - Java
  - JVM
  - HotSpot
  - 内存模型
  - 并发编程
aliases:
  - Java Object Header
  - Mark Word
  - Klass Pointer
  - 对象头结构
source_type: mixed
source_urls:
  - https://github.com/openjdk/jdk/blob/master/src/hotspot/share/oops/markWord.hpp
  - https://github.com/openjdk/jdk/blob/master/src/hotspot/share/oops/arrayOop.hpp
  - https://github.com/openjdk/jdk/blob/master/src/hotspot/share/oops/oop.hpp
  - https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-2.html#jvms-2.7
  - https://openjdk.org/jeps/450
status: verified
---

## 是什么

Java 对象头（Object Header）是 HotSpot JVM 中每个 Java 对象在堆内存中**起始位置的一段元数据区域**，用于存储对象的运行时信息。它不属于 Java 语言规范定义的范畴，而是 JVM 实现层面的内部结构。

对象头对开发者不可见，但直接影响：
- **锁机制**：`synchronized` 的锁状态、锁升级全部依赖对象头中的 Mark Word
- **垃圾回收**：分代年龄、转发指针、标记状态存储在对象头中
- **类型识别**：JVM 通过对象头中的 Klass Pointer 确定对象的实际类型
- **内存占用**：对象头是每个对象的基础开销，影响整体内存 footprint

---

## 对象头的组成

HotSpot 中对象头由两部分组成：

| 部分 | 说明 | 大小（64 位 JVM） |
|------|------|-------------------|
| **Mark Word** | 存储对象的运行时数据（哈希码、锁状态、GC 信息等） | 8 bytes（64 bits） |
| **Klass Pointer** | 指向方法区中 Class 元数据的指针 | 8 bytes（开启指针压缩后 4 bytes） |

**数组对象**额外包含一个 **Array Length** 字段（4 bytes），记录数组元素个数。

---

## Mark Word 详解

Mark Word 是对象头中最核心的部分，其 64 位空间根据对象状态复用存储不同信息。

### 位布局（64 位 JVM，非紧凑头模式）

```
unused:22 | hash:31 | unused_gap:4 | age:4 | self_fwd:1 | lock:2
```

| 字段 | 位数 | 说明 |
|------|------|------|
| `lock` | 2 bits | 锁状态标志位 |
| `self_fwd` | 1 bit | 自转发标志（Valhalla 项目预留） |
| `age` | 4 bits | 对象分代年龄（0-15），达到 15 后进入老年代 |
| `unused_gap` | 4 bits | Valhalla 项目预留 |
| `hash` | 31 bits | `System.identityHashCode()` 返回的标识哈希值 |
| `unused` | 22 bits | 未使用 |

> 来源：[OpenJDK markWord.hpp](https://github.com/openjdk/jdk/blob/master/src/hotspot/share/oops/markWord.hpp)

### 锁状态与 Mark Word 内容

Mark Word 的最低 2 位 `lock` 决定对象的锁状态：

| lock 值 | 状态 | Mark Word 存储内容（64 位 JVM） |
|---------|------|--------------------------------|
| `01` | 无锁 / 偏向锁（已废弃） | `hash:31 \| unused_gap:4 \| age:4 \| biased_lock:1 \| 01` |
| `00` | 轻量级锁 | 指向线程栈中 Lock Record 的指针 + `00` |
| `10` | 重量级锁（Monitor） | 指向 ObjectMonitor 的指针 + `10` |
| `11` | 标记状态（GC） | 用于标记对象，GC 完成后恢复 |

> 注意：Java 15 起**偏向锁默认禁用**（JEP 374），Java 21 中已完全移除偏向锁相关代码。现代 JVM 中 `01` 状态仅表示无锁。

#### 轻量级锁（`00`）

线程在自身栈帧中创建 **Lock Record**，通过 CAS 将 Mark Word 替换为指向该 Lock Record 的指针。Lock Record 中保存了被替换前的 Mark Word 副本，用于解锁时恢复。

#### 重量级锁（`10`）

Mark Word 存储指向 **ObjectMonitor** 对象的指针。ObjectMonitor 包含：
- `_owner`：当前持有锁的线程
- `_WaitSet`：调用 `wait()` 进入等待的线程队列
- `_EntryList`：等待获取锁的线程队列（类似 CLH 队列）

当 `UseObjectMonitorTable` 启用时（较新 JVM 版本的优化），Monitor 信息不再存储在 Mark Word 中，而是通过全局表管理，此时 `10` 状态仅作为标志位。

> 来源：[OpenJDK markWord.hpp — lock value definitions](https://github.com/openjdk/jdk/blob/master/src/hotspot/share/oops/markWord.hpp)

---

## Klass Pointer

Klass Pointer 指向 JVM 方法区（Metaspace）中的 **Klass 元数据对象**，用于：
- 确定对象的实际类型（`instanceof` 检查）
- 查找方法表（vtable / itable）
- 获取字段布局信息

### 指针压缩（Compressed Oops）

64 位 JVM 默认开启指针压缩（`-XX:+UseCompressedOops`），Klass Pointer 从 8 bytes 压缩为 4 bytes：

```bash
# 查看是否开启
java -XX:+PrintFlagsFinal -version | grep UseCompressedOops
```

开启条件：
- 堆内存 < 32 GB 时默认开启
- 堆内存 ≥ 32 GB 时自动关闭（4 bytes 无法寻址超过 32 GB 的空间）

> 指针压缩不仅影响 Klass Pointer，还影响对象内部的引用字段，可显著降低内存占用。

---

## 数组对象的额外字段

数组对象在 Mark Word 和 Klass Pointer 之后，额外存储一个 **4 bytes 的 length 字段**：

```
[Mark Word (8B)] [Klass Pointer (4/8B)] [length (4B)] [元素数据...]
```

> 来源：[OpenJDK arrayOop.hpp — "The _length field is not declared in C++. It is allocated after the mark-word..."](https://github.com/openjdk/jdk/blob/master/src/hotspot/share/oops/arrayOop.hpp)

---

## 紧凑对象头（Compact Object Headers）

Java 21 引入了实验性的紧凑对象头特性（`-XX:+UseCompactObjectHeaders`），将 Klass Pointer 编码到 Mark Word 内部：

```
klass:22 | hash:31 | unused_gap:4 | age:4 | self_fwd:1 | lock:2
```

开启后对象头从 12-16 bytes 减少到 **8 bytes**，可显著降低内存占用。该特性在 Java 25+ 中进一步成熟。

> 来源：[OpenJDK markWord.hpp — "64 bits (with compact headers)"](https://github.com/openjdk/jdk/blob/master/src/hotspot/share/oops/markWord.hpp)

---

## 对象内存布局完整示意

### 普通对象（64 位 JVM，开启指针压缩）

```
+------------------+ 0x00
|  Mark Word (8B)  |
+------------------+ 0x08
| Klass Ptr (4B)   |
+------------------+ 0x0C
|  Padding (4B)    | ← 8 字节对齐填充
+------------------+ 0x10
|  Instance Data   | ← 实例字段
+------------------+
|  Padding         | ← 尾部对齐填充
+------------------+
```

### 数组对象（64 位 JVM，开启指针压缩）

```
+------------------+ 0x00
|  Mark Word (8B)  |
+------------------+ 0x08
| Klass Ptr (4B)   |
+------------------+ 0x0C
|  length (4B)     |
+------------------+ 0x10
|  Element[0]      |
|  Element[1]      |
|  ...             |
+------------------+
|  Padding         | ← 尾部对齐填充
+------------------+
```

---

## 对象大小估算

### 基础公式

```
对象总大小 = 对象头 + 实例数据 + 对齐填充
```

### 对齐规则

HotSpot 默认按 **8 字节对齐**（`-XX:ObjectAlignmentInBytes=8`），对象总大小必须是 8 的倍数，不足则填充 padding。

### 常见场景估算（64 位 JVM + 指针压缩）

| 对象类型 | 对象头 | 实例数据 | 对齐填充 | 总大小 |
|----------|--------|----------|----------|--------|
| `new Object()` | 12B | 0B | 4B | **16 bytes** |
| `new Integer(42)` | 12B | 4B (int) | 0B | **16 bytes** |
| `new Long(42L)` | 12B | 8B (long) | 4B | **24 bytes** |
| `new String("")`（Java 9+，Compact Strings） | 12B | 24B（coder 1B + hash 4B + value ref 4B + 对齐） | 4B | **40 bytes** |
| `new int[10]` | 16B（含 length） | 40B（10 × 4B） | 0B | **56 bytes** |
| `new String[10]` | 16B（含 length） | 40B（10 × 4B 引用） | 0B | **56 bytes** |

> 注意：`String` 在 Java 9+ 使用 `byte[]` + `coder` 存储，Java 8 及之前使用 `char[]`（每个字符 2 bytes）。

### 估算工具

- **JOL（Java Object Layout）**：官方提供的对象布局分析工具

```xml
<dependency>
    <groupId>org.openjdk.jol</groupId>
    <artifactId>jol-core</artifactId>
    <version>0.17</version>
</dependency>
```

```java
import org.openjdk.jol.info.ClassLayout;

System.out.println(ClassLayout.parseInstance(new Object()).toPrintable());
```

输出示例：
```
java.lang.Object object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000000000000001
  8   4        (object header: klass)    0x000001e8
 12   4        (loss due to alignment)
Instance size: 16 bytes
```

> 来源：[OpenJDK JOL 项目](https://github.com/openjdk/jol)

---

## 为什么重要

### 1. 理解 synchronized 的底层机制

`synchronized` 的锁升级（无锁 → 轻量级锁 → 重量级锁）完全通过修改 Mark Word 实现，不涉及额外的内存分配。理解对象头是理解 Java 锁优化的前提。

### 2. 内存优化与调优

- 对象头是每个对象的**固定开销**（12-16 bytes），大量小对象会显著增加内存占用
- 开启指针压缩可减少 4 bytes/引用
- 紧凑对象头（Java 21+）可进一步减少对象头大小
- 合理设计数据结构（如使用 `int[]` 而非 `List<Integer>`）可避免大量对象头开销

### 3. GC 行为分析

- 分代年龄（4 bits）决定对象何时晋升老年代
- GC 标记阶段通过修改 Mark Word 的 lock 位为 `11` 来标记存活对象
- 对象拷贝/转发时，Mark Word 中存储转发指针

### 4. 哈希码的存储时机

`System.identityHashCode()` 或 `Object.hashCode()` **首次调用时**才计算并写入 Mark Word 的 hash 字段。未调用过 hashCode 的对象，hash 字段为 0。

> 这解释了为什么 `hashCode()` 在多线程下可能出现竞态条件——两个线程同时首次调用时可能计算出不同的哈希值。

---

## 限制与注意事项

### 1. 对象头是 JVM 实现细节

- JVM 规范（JVMS §2.7）明确指出：**不强制规定对象的内部结构**
- 不同 JVM 实现（HotSpot、OpenJ9、Zing）的对象头布局可能不同
- 本文描述仅适用于 HotSpot JVM

> "The Java Virtual Machine does not mandate any particular internal structure for objects."
> — [JVMS §2.7 Representation of Objects](https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-2.html#jvms-2.7)

### 2. 版本差异

| 特性 | Java 8 | Java 15 | Java 21 | Java 25+ |
|------|--------|---------|---------|----------|
| 偏向锁 | 默认开启 | 默认禁用（JEP 374） | 已移除 | 已移除 |
| 指针压缩 | 默认开启（<32GB） | 同左 | 同左 | 同左 |
| 紧凑对象头 | 不支持 | 不支持 | 实验性（JEP 450） | 更成熟 |
| Value Objects | 不支持 | 不支持 | Preview | 可能正式发布 |

### 3. 32 位 JVM 差异

32 位 JVM 中：
- Mark Word 为 32 bits（4 bytes）
- Klass Pointer 为 32 bits（4 bytes）
- 哈希码仅 25 bits
- 对象头总计 8 bytes（普通对象）

### 4. 对象头不可被 Java 代码直接访问

- 无法通过反射或 Unsafe 直接读取/修改对象头
- `sun.misc.Unsafe` 可以读取对象起始地址的内存，但这是未公开 API，且行为依赖具体 JVM 版本
- JOL 工具通过 JVM TI 接口获取对象布局信息，而非直接访问对象头

---

## 相关概念

| 概念 | 关系 |
|------|------|
| **Monitor** | 重量级锁时，Mark Word 指向 ObjectMonitor 对象 |
| **Lock Record** | 轻量级锁时，线程栈中保存 Mark Word 副本的结构 |
| **偏向锁（已废弃）** | 曾是 Mark Word 的一种状态，Java 15+ 默认禁用 |
| **指针压缩** | 影响 Klass Pointer 大小，间接影响对象头总大小 |
| **JOL** | 分析对象布局的官方工具，可验证对象头大小 |
| **Value Objects（JEP 401）** | 未来 Java 中无对象头的值类型，可消除对象头开销 |

---

## 参考资料

- OpenJDK 源码 — `markWord.hpp`：https://github.com/openjdk/jdk/blob/master/src/hotspot/share/oops/markWord.hpp
- OpenJDK 源码 — `arrayOop.hpp`：https://github.com/openjdk/jdk/blob/master/src/hotspot/share/oops/arrayOop.hpp
- Oracle JVM 规范 — Representation of Objects：https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-2.html#jvms-2.7
- JEP 450 — Compact Object Headers：https://openjdk.org/jeps/450
- JEP 374 — Deprecate and Disable Biased Locking：https://openjdk.org/jeps/374
- JEP 401 — Value Classes and Objects (Preview)：https://openjdk.org/jeps/401
- OpenJDK JOL（Java Object Layout）：https://github.com/openjdk/jol
