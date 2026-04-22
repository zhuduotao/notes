---
created: '2026-04-20'
updated: '2026-04-20'
tags:
  - Java
  - Collections
  - 线程安全
  - 并发
  - 数据结构
  - JDK
aliases:
  - Java集合框架
  - Collections Framework
  - Java集合线程安全
source_type: official-doc
source_urls:
  - >-
    https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/package-summary.html
  - >-
    https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/package-summary.html
  - >-
    https://github.com/openjdk/jdk/tree/master/src/java.base/share/classes/java/util
  - >-
    https://github.com/openjdk/jdk/tree/master/src/java.base/share/classes/java/util/concurrent
status: verified
---

## 概述

Java Collections Framework 是 JDK 1.2 引入的一套统一架构，用于表示和操作集合（一组对象）。框架定义了核心接口、多种实现类、算法工具，以及并发版本，为开发者提供了高效、可复用的数据结构解决方案[^1]。

## 接口层次结构

### 核心接口

```
Iterable (java.lang)
    │
    └─ Collection
        ├─ List          有序、允许重复、支持索引访问
        ├─ Set           无序、不允许重复
        │   ├─ SortedSet/NavigableSet  排序 Set
        │   └─ SequencedSet (JDK 21)   可逆序访问的 Set
        ├─ Queue         队列，FIFO 或优先级
        │   ├─ Deque     双端队列
        │   └─ SequencedCollection (JDK 21)
        └─ Map           键值映射，键不允许重复
            ├─ SortedMap/NavigableMap  排序 Map
            └─ SequencedMap (JDK 21)
```

**关键点**[^1]：
- `Collection` 继承 `Iterable`，所有集合均可使用 `for-each` 循环
- `Map` 不继承 `Collection`，是独立接口体系
- JDK 21 引入 `SequencedCollection`、`SequencedSet`、`SequencedMap`，统一了首尾元素访问 API[^2]

## 核心实现类

### List 实现

| 实现类 | 底层结构 | 特性 | 适用场景 |
|--------|----------|------|----------|
| `ArrayList` | 动态数组 | 随机访问 O(1)，尾部添加 O(1)，内存紧凑 | 大多数场景首选 |
| `LinkedList` | 双向链表 | 头部操作 O(1)，实现 Deque | 频繁头部插入/队列用途 |
| `Vector` | 动态数组（同步） | 方法同步，已废弃 | 不推荐，用 `Collections.synchronizedList` 或并发集合替代 |
| `Stack` | Vector 子类（同步） | LIFO栈，已废弃 | 不推荐，用 `Deque`（如 `ArrayDeque`）替代 |

详细对比见《ArrayList与LinkedList的区别与Java版本演进》。

### Set 实现

| 实现类 | 底层结构 | 特性 | 适用场景 |
|--------|----------|------|----------|
| `HashSet` | HashMap（仅存 key） | O(1) 增删查，无序 | 通用去重场景 |
| `LinkedHashSet` | LinkedHashMap | 保持插入顺序 | 需保留插入顺序的去重 |
| `TreeSet` | TreeMap（红黑树） | 自然排序或定制排序，O(log n) | 需排序遍历 |
| `EnumSet` | 位向量 | 专为 enum 设计，极高效 | enum 元素集合 |
| `CopyOnWriteArraySet` | CopyOnWriteArrayList | 线程安全，写时复制 | 低频写入、高频遍历的并发场景 |

**内部实现**[^3]：
- `HashSet` 内部持有一个 `HashMap`，元素作为 key，value 固定为 `PRESENT`（dummy object）
- `LinkedHashSet` 使用 `LinkedHashMap`，维护双向链表记录插入/访问顺序
- `TreeSet` 使用 `TreeMap`，依赖 `Comparable` 或 `Comparator`

### Map 实现

| 实现类 | 底层结构 | 特性 | 适用场景 |
|--------|----------|------|----------|
| `HashMap` | 数组 + 链表/红黑树 | O(1) 平均，非线程安全 | 通用键值存储首选 |
| `LinkedHashMap` | HashMap + 双向链表 | 保持插入/访问顺序 | 需有序遍历、LRU 缓存 |
| `TreeMap` | 黑树 | 排序遍历，O(log n) | 需按 key 排序 |
| `Hashtable` | 数组 + 链表（同步） | 方法同步，已废弃 | 不推荐 |
| `IdentityHashMap` | 数组 | 用 `==` 而非 `equals` 比较 key | 特定场景（如序列化） |
| `WeakHashMap` | 数组 + WeakReference | key 可被 GC 回收 | 缓存、临时映射 |
| `EnumMap` | 数组 | 专为 enum key 设计，极高效 | enum 作为 key |

**HashMap 内部机制**[^4]：
- JDK 8 前：数组 + 链表，链表过长时性能退化
- JDK 8+：当链表长度 ≥ 8 且数组容量 ≥ 64 时，链表转红黑树，避免极端 hash 冲突导致的 O(n) 查询
- 扩容：负载因子默认 0.75，容量翻倍
- 初始容量：JDK 7+ 支持延迟初始化（首次 put 时才分配）

### Queue/Deque 实现

| 实现类 | 底层结构 | 特性 | 适用场景 |
|--------|----------|------|----------|
| `ArrayDeque` | 循环数组 | 头尾操作均 O(1)，无容量限制 | 栈/队列/双端队列首选 |
| `LinkedList` | 双向链表 | 实现 Deque | 已有链表时复用 |
| `PriorityQueue` | 最小堆 | 按 priority 排序出队 | 任务调度、优先级处理 |
| `DelayQueue` | PriorityQueue | 延迟元素，到期才能取出 | 定时任务、过期清理 |

**ArrayDeque vs LinkedList**[^5]：
- `ArrayDeque` 内存更紧凑，无节点对象开销
- `ArrayDeque` 的数组扩容策略：容量翻倍 + 1
- 两者均实现 `Deque`，栈操作推荐 `ArrayDeque`（`Stack` 类已废弃）

## 线程安全

### 非线程安全的集合

以下集合类均**非线程安全**[^1]：
- `ArrayList`、`LinkedList`、`HashMap`、`LinkedHashMap`、`TreeMap`
- `HashSet`、`LinkedHashSet`、`TreeSet`
- `ArrayDeque`、`PriorityQueue`

**官方文档说明**：若多线程并发访问且至少一个线程进行结构性修改（add/remove/clear），必须外部同步，否则行为不可预测。

### 同步包装器（Synchronized Wrappers）

`Collections` 工具类提供同步包装方法[^6]：

```java
List<E> syncList = Collections.synchronizedList(new ArrayList<>());
Set<E> syncSet = Collections.synchronizedSet(new HashSet<>());
Map<K,V> syncMap = Collections.synchronizedMap(new HashMap<>());
```

**实现机制**：
- 包装类在每个方法上使用 `synchronized`（monitor 锁）
- 返回的集合内部持有一个 mutex 对象（默认为集合自身）

**局限性**：
- 粗粒度锁：每次操作锁定整个集合，并发吞吐量低
- 迭代需手动同步：
  ```java
  synchronized (syncList) {
      for (E e : syncList) { ... }
  }
  ```
- 复合操作（如 `putIfAbsent`）仍需外部同步

### 并发集合（java.util.concurrent）

JDK 5 引入 `java.util.concurrent` 包，提供高性能并发集合[^7]：

#### ConcurrentHashMap

**核心特性**：
- 线程安全，高并发读写
- 读操作无锁（volatile + CAS）
- 写操作细粒度锁（JDK 8+：锁单个 bucket 的 Node 或 TreeBin）

**版本演进**：
- JDK 5-7：分段锁（Segment），16 个段各自独立锁
- JDK 8+：取消 Segment，直接锁 Node/head of chain + CAS + synchronized
  - 减少锁粒度，降低内存占用
  - 在链表头或红黑树 root 上加锁

**关键 API**：
- `putIfAbsent(key, value)`：原子性"不存在则插入"
- `replace(key, oldValue, newValue)`：原子性"值匹配则替换"
- `computeIfAbsent(key, mappingFunction)`：原子性计算并插入

**一致性语义**[^8]：
- 读操作：可见最后一次完成的写操作
- 迭代：弱一致性（Weakly consistent），不抛 `ConcurrentModificationException`，可能反映迭代开始时或迭代过程中的状态
- size/isEmpty：近似值，非精确快照

#### CopyOnWriteArrayList / CopyOnWriteArraySet

**实现机制**：
- 内部持有一个 volatile 数组
- 写操作（add/set/remove）创建新数组副本，修改后替换引用
- 读操作直接访问当前数组，无锁

**特点**[^9]：
- 读操作极快，完全无锁
- 写操作开销大（复制整个数组）
- 迭代时不会抛 `ConcurrentModificationException`，迭代器基于创建时的数组快照

**适用场景**：
- 读多写极少（如事件监听器列表、配置列表）
- 需遍历期间保证一致性（不允许并发修改导致迭代失败）

**不适用场景**：
- 写操作频繁、数据量大（内存和性能开销显著）

#### ConcurrentLinkedQueue / ConcurrentLinkedDeque

**实现机制**[^10]：
- 基于 Michael-Scott 非阻塞并发队列算法（无锁算法）
- 使用 CAS 实现无阻塞并发入队/出队
- 节点通过 volatile next 指针链接

**特点**：
- 无阻塞，高吞吐
- size() 是 O(n) 遍历，非精确（官方建议仅用于监控）
- 无容量限制（无界）

**适用场景**：高并发生产者-消费者队列

#### BlockingQueue 实现类

用于生产者-消费者模式，提供阻塞操作[^11]：

| 实现类 | 底层结构 | 特性 |
|--------|----------|------|
| `ArrayBlockingQueue` | 循环数组 | 有界，需指定容量，公平/非公平可选 |
| `LinkedBlockingQueue` | 链表 | 有界（默认 Integer.MAX_VALUE），分离的 take/put 锁 |
| `PriorityBlockingQueue` | 最小堆 | 无界，按 priority 出队 |
| `DelayQueue` | PriorityQueue | 无界，元素需实现 Delayed，到期才能取出 |
| `SynchronousQueue` | 无存储 | 每个插入必须等待对应移除，直接传递 |
| `LinkedTransferQueue` | 链表 | 无界，支持 transfer 操作（阻塞等待消费者） |

**阻塞方法**：
- `put(e)`：队列满时阻塞
- `take()`：队列空时阻塞
- `offer(e, timeout, unit)`：限时插入
- `poll(timeout, unit)`：限时取出

#### ConcurrentSkipListMap / ConcurrentSkipListSet

**实现机制**[^12]：
- 跳表（Skip List），基于 William Pugh 论文
- 无锁并发访问，CAS 实现节点插入/删除
- 维持有序性（自然排序或 Comparator）

**特点**：
- 线程安全 + 排序
- 平均 O(log n) 操作
- 实现 `NavigableMap`/`NavigableSet`（支持范围查询、逆序遍历）

**适用场景**：需要并发 + 排序（如并发优先级映射、定时任务调度）

### 线程安全对比总结

| 需求 | 推荐方案 |
|------|----------|
| 通用并发 Map | `ConcurrentHashMap` |
| 读多写少 List/Set | `CopyOnWriteArrayList`/`CopyOnWriteArraySet` |
| 高并发队列 | `ConcurrentLinkedQueue` |
| 生产者-消费者 | `ArrayBlockingQueue`/`LinkedBlockingQueue` |
| 并发 + 排序 | `ConcurrentSkipListMap`/`ConcurrentSkipListSet` |
| 简单同步（低并发） | `Collections.synchronizedXxx` |

## 常见误区与注意事项

### 误区：Hashtable 是推荐选择

**实际**：Hashtable 自 JDK 1.0 就存在，方法全部 synchronized，已废弃。官方文档建议使用 `ConcurrentHashMap` 或 `Collections.synchronizedMap`[^1]。

### 误区：Collections.synchronizedXxx 能解决所有并发问题

**实际**：
- 只保证单个方法的原子性
- 复合操作（如"检查后执行"）仍需外部同步
- 迭代需手动加锁
- 高并发下性能不如并发集合

### 误区：ConcurrentHashMap 完全强一致

**实际**：
- 读操作弱一致（可见最近完成的写）
- size/isEmpty 是近似值
- 迭代可能不反映最新状态
- 批量操作（如 clear）不保证原子性

### 注意：迭代期间的并发修改

| 集合类型 | 迭代期间修改行为 |
|----------|------------------|
| 非线程安全集合 | 抛 `ConcurrentModificationException`（fail-fast） |
| synchronized 包装 | 需手动同步，否则可能抛异常或行为异常 |
| ConcurrentHashMap | 弱一致，不抛异常，迭代基于某个时间点状态 |
| CopyOnWrite | 不抛异常，迭代基于创建时快照 |

### 注意：equals/hashCode 对 Set/Map 的影响

存入 `HashSet`/`HashMap` 的对象必须正确实现 `equals` 和 `hashCode`[^3]：
- 相等对象必须返回相同 hashCode
- hashCode 相等的对象，equals 可能不等
- 若对象作为 key 存入后修改了影响 hashCode 的字段，将无法正确检索/删除

### 注意：Comparable/Comparator 对 TreeSet/TreeMap 的要求

`TreeSet`/`TreeMap` 要求元素/key 实现 `Comparable`，或构造时提供 `Comparator`[^13]。未满足时抛 `ClassCastException`。

## JDK 版本演进

### JDK 1.2：Collections Framework 引入

引入核心接口和实现类：`Collection`、`List`、`Set`、`Map`、`ArrayList`、`LinkedList`、`HashSet`、`TreeSet`、`HashMap`、`TreeMap`。

### JDK 5：并发集合

引入 `java.util.concurrent` 包：`ConcurrentHashMap`、`CopyOnWriteArrayList`、`BlockingQueue` 系列等。

### JDK 7：HashMap 延迟初始化

`new HashMap()` 不立即分配数组，首次 put 时扩容到默认容量（类似 ArrayList）。

### JDK 8：HashMap 链表转红黑树

当链表长度 ≥ 8 且数组容量 ≥ 64 时，链表转红黑树，改善极端 hash 冲突性能[^4]。

### JDK 9：不可变集合工厂方法

`List.of()`、`Set.of()`、`Map.of()` 创建不可变集合（JEP 269）[^14]。

### JDK 21：SequencedCollection

引入 `SequencedCollection`、`SequencedSet`、`SequencedMap`，统一首尾访问 API[^2]。

## 参考资料

[^1]: [java.util Package Summary (Java SE 21)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/package-summary.html)
[^2]: [JEP 431: Sequenced Collections](https://openjdk.org/jeps/431)
[^3]: [HashSet API (Java SE 21)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/HashSet.html)
[^4]: [HashMap API (Java SE 21)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/HashMap.html)
[^5]: [ArrayDeque API (Java SE 21)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/ArrayDeque.html)
[^6]: [Collections API - Synchronized Wrappers](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/Collections.html)
[^7]: [java.util.concurrent Package Summary (Java SE 21)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/package-summary.html)
[^8]: [ConcurrentHashMap API (Java SE 21)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/ConcurrentHashMap.html)
[^9]: [CopyOnWriteArrayList API (Java SE 21)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/CopyOnWriteArrayList.html)
[^10]: [ConcurrentLinkedQueue API (Java SE 21)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/ConcurrentLinkedQueue.html)
[^11]: [BlockingQueue API (Java SE 21)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/BlockingQueue.html)
[^12]: [ConcurrentSkipListMap API (Java SE 21)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/ConcurrentSkipListMap.html)
[^13]: [TreeMap API (Java SE 21)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/TreeMap.html)
[^14]: [JEP 269: Convenience Factory Methods for Collections](https://openjdk.org/jeps/269)
