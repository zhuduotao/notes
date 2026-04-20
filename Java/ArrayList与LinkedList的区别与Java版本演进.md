---
created: '2026-04-20'
updated: '2026-04-20'
tags:
  - Java
  - Collections
  - List
  - ArrayList
  - LinkedList
  - 数据结构
  - JDK版本演进
aliases:
  - ArrayList vs LinkedList
  - ArrayList LinkedList 对比
  - Java List 实现对比
source_type: official-doc
source_urls:
  - >-
    https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/ArrayList.html
  - >-
    https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/LinkedList.html
  - >-
    https://github.com/openjdk/jdk/blob/master/src/java.base/share/classes/java/util/ArrayList.java
  - >-
    https://github.com/openjdk/jdk/blob/master/src/java.base/share/classes/java/util/LinkedList.java
  - 'https://openjdk.org/jeps/269'
status: verified
---
## 核心区别

### 底层数据结构

| 特性 | ArrayList | LinkedList |
|------|-----------|------------|
| 数据结构 | 动态数组（Object[]） | 双向链表（Node 节点） |
| 内存布局 | 连续内存，紧凑存储 | 非连续，每个元素需额外存储前后指针 |
| 内存开销 | 低（仅数组本身） | 高（每个元素约 24 字节额外开销） |

### 时间复杂度对比

| 操作 | ArrayList | LinkedList |
|------|-----------|------------|
| `get(int index)` | O(1) | O(n)，需遍历链表 |
| `set(int index, E e)` | O(1) | O(n)，需先遍历定位 |
| `add(E e)`（尾部添加） | O(1) 平均（扩容时 O(n)） | O(1) |
| `add(int index, E e)` | O(n)，需移动后续元素 | O(n)，需遍历定位 + O(1) 插入 |
| `remove(int index)` | O(n)，需移动后续元素 | O(n)，需遍历定位 + O(1) 删除 |
| `contains(Object o)` | O(n) | O(n) |
| `indexOf(Object o)` | O(n) | O(n) |

> **关键结论**：ArrayList 实现了 `RandomAccess` 标记接口，表明其支持快速随机访问；LinkedList 未实现此接口。根据官方文档，ArrayList 的 `size`、`isEmpty`、`get`、`set`、`iterator`、`listIterator` 操作为常数时间，`add` 为**摊还常数时间**（amortized constant time）[^1]。

## 实现接口差异

| 接口 | ArrayList | LinkedList |
|------|-----------|------------|
| `List<E>` | ✓ | ✓ |
| `RandomAccess` | ✓ | ❌ |
| `Cloneable` | ✓ | ✓ |
| `Serializable` | ✓ | ✓ |
| `SequencedCollection<E>`（JDK 21+） | ✓ | ✓ |
| `Deque<E>` | ❌ | ✓ |
| `Queue<E>` | ❌ | ✓ |

LinkedList 同时实现了 `Deque` 和 `Queue` 接口，可直接用作队列、双端队列或栈（`push`/`pop` 方法）。ArrayList 不支持这些队列操作。

## Java 版本演进中的实现变化

### JDK 1.2：首次引入

ArrayList 与 LinkedList 同于 JDK 1.2 引入，作为 Java Collections Framework 的核心组件[^1][^2]。

### JDK 7：ArrayList 延迟初始化优化

JDK 7 对 ArrayList 的默认构造器做了重要优化：

- **JDK 6 及之前**：`new ArrayList()` 直接创建容量为 10 的数组
- **JDK 7+**：`new ArrayList()` 创建空数组（`EMPTY_ELEMENTDATA`），首次添加元素时才扩容到 10

**目的**：减少空 ArrayList 的内存占用。大量空 ArrayList 在未使用前不分配数组空间。

相关源码（JDK 7）：
```java
private static final Object[] EMPTY_ELEMENTDATA = {};
private transient Object[] elementData = EMPTY_ELEMENTDATA;
```

### JDK 8：无重大结构性变化

ArrayList 与 LinkedList 的核心实现保持稳定，主要配合 Lambda 和 Stream API 的引入进行了兼容性调整。

### JDK 9：便利工厂方法（JEP 269）

JDK 9 通过 [JEP 269](https://openjdk.org/jeps/269) 引入了 `List.of()` 静态工厂方法，用于创建**不可变**的小型列表实例：

```java
List<String> list = List.of("a", "b", "c");  // 返回不可变列表
```

**特点**：
- 返回的列表是不可变的（不支持 add/remove/set）
- 禁止 null 元素
- 内部实现是紧凑的、专门优化的小型列表（非 ArrayList）
- 实现了 `RandomAccess` 接口[^3]

> 注意：`List.of()` 返回的不是 ArrayList，而是专门的不可变实现类（如 `List12`、`ListN`）。若需可变 ArrayList，仍需使用 `new ArrayList<>(List.of(...))`。

### JDK 21：SequencedCollection 接口

JDK 21 引入了 `SequencedCollection<E>` 接口，ArrayList 和 LinkedList 均实现了该接口，新增以下方法[^1][^2]：

| 方法 | ArrayList 行为 | LinkedList 行为 |
|------|----------------|-----------------|
| `getFirst()` | O(1)，访问 index 0 | O(1)，直接访问 head |
| `getLast()` | O(1)，访问 index size-1 | O(1)，直接访问 tail |
| `addFirst(E)` | O(n)，需移动所有元素 | O(1)，修改 head 指针 |
| `addLast(E)` | O(1) 平均，尾部添加 | O(1)，修改 tail 指针 |
| `removeFirst()` | O(n)，需移动所有元素 | O(1)，移除 head |
| `removeLast()` | O(1)，尾部移除 | O(1)，移除 tail |

**结论**：LinkedList 在头部操作上有显著优势；ArrayList 的尾部操作效率高，但头部操作需移动全部元素。

## 扩容机制（ArrayList）

ArrayList 使用动态数组，当容量不足时自动扩容：

- **扩容策略**：`int newCapacity = oldCapacity + (oldCapacity >> 1)`，即增长约 50%（1.5 倍）
- **最大容量**：`Integer.MAX_VALUE - 8`（预留 8 字节用于 JVM header）

可通过 `ensureCapacity(int minCapacity)` 预先扩容，避免多次增量扩容带来的性能损耗。可通过 `trimToSize()` 将容量缩减至实际大小，释放多余内存[^1]。

## 线程安全性

ArrayList 和 LinkedList 均**非线程安全**。官方文档明确指出：若多线程并发访问且至少一个线程进行结构性修改（add/remove），必须外部同步[^1][^2]。

推荐同步方式：
```java
List<E> syncList = Collections.synchronizedList(new ArrayList<>());
List<E> syncList = Collections.synchronizedList(new LinkedList<>());
```

迭代器的 `fail-fast` 行为：若迭代过程中列表被结构性修改（非通过迭代器自身的 remove/add），迭代器抛出 `ConcurrentModificationException`。此行为仅用于检测 bug，不能作为并发控制依赖[^1][^2]。

## 适用场景

### 优先使用 ArrayList

- 需要频繁随机访问（`get`/`set`）
- 主要在尾部添加/删除元素
- 内存空间有限，追求紧凑存储
- 需与数组进行转换（`toArray`）

### 优先使用 LinkedList

- 需频繁在头部插入/删除元素
- 需作为队列、双端队列或栈使用
- 元素数量变化剧烈，不关心内存开销
- 已持有目标节点引用时进行插入/删除（通过 `ListIterator`）

### 需避免的场景

- **LinkedList 随机访问**：每次 `get(index)` 都需遍历，性能极差
- **ArrayList 头部频繁插入**：每次插入都需移动全部元素

## 常见误区

### 误区：LinkedList 插入删除总是更快

**实际情况**：
- 若按索引插入/删除（如 `add(index, e)`），LinkedList 需先遍历定位节点（O(n)），定位后插入本身为 O(1)。整体仍为 O(n)。
- ArrayList 的插入虽需移动元素，但数组移动在内存中是连续操作，现代 CPU 缓存友好。对于小规模数据，ArrayList 可能更快。

**何时 LinkedList 更快**：已通过迭代器定位到目标位置后，直接调用 `iterator.add()` 或 `iterator.remove()`。

### 误区：LinkedList 内存更省

**实际情况**：LinkedList 每个元素需存储 Node 对象（包含 item、next、prev 三字段），额外内存开销约 24 字节/元素。ArrayList 的数组仅存储引用，无额外节点开销。

### 误区：ArrayList 扩容导致性能差

**实际情况**：`add` 操作为摊还常数时间。扩容虽偶尔发生，但每次扩容预留 50% 余量，后续大量添加无需频繁扩容。若已知数据量，调用 `ensureCapacity` 或使用带初始容量的构造器即可完全避免扩容。

## 相关概念

- **List 接口**：定义了有序集合的核心操作
- **RandomAccess 接口**：标记接口，表明实现类支持高效随机访问，算法可据此选择最优遍历策略
- **Deque 接口**：双端队列接口，LinkedList 的额外能力
- **Fail-fast 迭代器**：快速失败机制，用于检测并发修改
- **Iterator vs ListIterator**：ListIterator 支持双向遍历、修改、插入

## 参考资料

[^1]: [ArrayList 官方 API 文档 (Java SE 21)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/ArrayList.html)
[^2]: [LinkedList 官方 API 文档 (Java SE 21)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/LinkedList.html)
[^3]: [JEP 269: Convenience Factory Methods for Collections](https://openjdk.org/jeps/269)
[^4]: [OpenJDK ArrayList 源码](https://github.com/openjdk/jdk/blob/master/src/java.base/share/classes/java/util/ArrayList.java)
[^5]: [OpenJDK LinkedList 源码](https://github.com/openjdk/jdk/blob/master/src/java.base/share/classes/java/util/LinkedList.java)
