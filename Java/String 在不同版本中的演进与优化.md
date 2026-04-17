
---
title: String 在不同版本中的演进与优化
created: 2026-04-17
updated: 2026-04-17
tags:
  - Java
  - String
  - JVM
  - 性能优化
  - JEP
  - 版本演进
aliases:
  - Java String 版本差异
  - Java String 优化
  - Compact Strings
  - Text Blocks
  - String Templates
source_type: mixed
source_urls:
  - https://openjdk.org/jeps/254
  - https://openjdk.org/jeps/378
  - https://openjdk.org/jeps/430
  - https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/lang/String.html
status: verified
---

## 概述

`java.lang.String` 是 Java 中最基础、使用最频繁的类之一。从 Java 1.0 到 Java 21+，String 的底层实现、API 能力和语法支持经历了多次重大演进。理解这些变化对内存优化、性能调优和代码可读性提升有直接帮助。

---

## Java 6 及之前：`char[]` 实现与 `substring` 内存泄漏

### 底层实现

- 内部使用 `private final char[] value` 存储字符，每个字符固定占用 **2 字节**（UTF-16）
- 所有字符串操作都基于 `char` 数组

### `substring` 的共享数组问题

**行为**：`substring(int beginIndex, int endIndex)` 返回的新 String 与原 String **共享同一个 `char[]` 数组**，仅通过 offset 和 count 区分。

**问题**：当从一个很大的字符串中截取很小的一部分时，原大数组无法被 GC 回收，导致内存泄漏。

```java
// 问题示例
String large = loadHugeString(); // 10MB
String small = large.substring(0, 10); // 仅取 10 个字符
// large 被 GC 后，small 仍然持有整个 10MB 的 char[]
```

**影响**：在日志解析、大文件处理等场景中容易引发 OOM。

---

## Java 7u6：修复 `substring` 内存泄漏

### 关键变更

- `substring` 改为 **复制数组**，不再共享原 `char[]`
- 新增 `coder` 字段（为后续 Compact Strings 做准备）

**修复方式**：

```java
// Java 7u6+ 的 substring 实现
public String substring(int beginIndex, int endIndex) {
    // ... 参数检查 ...
    return new String(value, beginIndex, subLen); // 复制数组
}
```

**影响**：
- 消除了内存泄漏风险
- 代价：频繁截取小字符串时产生额外内存分配和拷贝开销

---

## Java 8：字符串拼接优化与 String Deduplication

### 字符串拼接优化

- 编译器对 `+` 拼接的优化从 `StringBuffer` 转向 `StringBuilder`
- 为后续 `invokedynamic` 拼接优化奠定基础

### G1 GC 的 String Deduplication（JEP 192）

- **引入版本**：Java 8u20（G1 GC 实验性特性）
- **机制**：自动检测堆中内容相同的 String 对象，让它们的 `value` 数组指向同一份数据
- **开启方式**：`-XX:+UseStringDeduplication`（需配合 `-XX:+UseG1GC`）
- **效果**：可减少 10%~30% 的 String 内存占用（取决于应用特征）

**限制**：
- 仅对 G1 GC 生效
- 仅在 Young GC 或 Mixed GC 时触发，Full GC 不执行去重
- 去重操作有额外 CPU 开销

---

## Java 9：Compact Strings（JEP 254）— 重大内存优化

### 核心变更

将 String 的底层存储从 `char[]`（每字符 2 字节）改为 `byte[]` + 编码标志位：

| 编码 | 每字符字节数 | 适用字符范围 |
|------|-------------|-------------|
| Latin-1 (ISO-8859-1) | 1 字节 | U+0000 ~ U+00FF |
| UTF-16 | 2 字节 | 全部 Unicode 字符 |

### 工作机制

- 新增 `private final byte coder` 字段标识编码方式
- 构造 String 时自动检测内容：如果所有字符都在 Latin-1 范围内，使用 1 字节编码；否则使用 UTF-16
- 对开发者完全透明，无需修改代码

### 内存收益

- 对于以英文/数字为主的字符串，内存占用 **减少约 50%**
- 堆中 String 对象占比高的应用（如 Web 服务、JSON 处理）收益显著
- 减少 GC 压力，间接提升吞吐量

### 性能影响

- 大多数场景性能持平或略有提升（GC 减少）
- 涉及 Latin-1 与 UTF-16 混合操作时（如拼接不同编码的字符串），有轻微性能损耗
- HotSpot 对常用 String 操作做了 intrinsic 优化

### 相关类同步更新

- `StringBuilder`、`StringBuffer`、`AbstractStringBuilder` 同样改用 `byte[]` 存储
- 所有公开 API 保持不变，属于纯实现层变更

### 参考资料

- [JEP 254: Compact Strings](https://openjdk.org/jeps/254)

---

## Java 11：新增 String API

Java 11 为 String 类新增了多个实用方法：

| 方法 | 说明 | 示例 |
|------|------|------|
| `isBlank()` | 判断字符串是否为空或仅含空白字符 | `"".isBlank()` → `true` |
| `strip()` | 去除首尾空白字符（Unicode 感知） | `"  hello  ".strip()` → `"hello"` |
| `stripLeading()` | 去除首部空白字符 | |
| `stripTrailing()` | 去除尾部空白字符 | |
| `lines()` | 按行分割，返回 `Stream<String>` | `"a\nb\nc".lines()` → Stream |
| `repeat(int count)` | 重复字符串指定次数 | `"ab".repeat(3)` → `"ababab"` |

### `strip()` vs `trim()` 的区别

| 特性 | `trim()` | `strip()` |
|------|----------|-----------|
| 空白字符定义 | codepoint ≤ U+0020 | `Character.isWhitespace()`（Unicode 标准） |
| 全角空格 | 不去除 | 去除 |
| 引入版本 | Java 1.0 | Java 11 |

```java
// 全角空格示例
String s = "\u3000hello\u3000"; // 全角空格
s.trim();   // "\u3000hello\u3000" — 不去除
s.strip();  // "hello" — 正确去除
```

### `lines()` 使用示例

```java
String text = "line1\nline2\r\nline3";
text.lines()
    .filter(line -> !line.isBlank())
    .forEach(System.out::println);
// 输出: line1, line2, line3
```

### 参考资料

- [Java 11 String API 文档](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/lang/String.html)

---

## Java 12-14：`indent()` 与 `transform()`

### `indent(int n)`（Java 12）

调整字符串每行的缩进级别：

- `n > 0`：每行前添加 n 个空格
- `n < 0`：每行移除最多 |n| 个前导空格
- `n = 0`：规范化行终止符，不改变缩进

```java
String s = "foo\nbar\nbar2";
System.out.println(s.indent(4));
//     foo
//     bar
//     bar2
```

### `transform(Function<String, R> f)`（Java 12）

对字符串应用转换函数，支持链式操作：

```java
int result = "foo".transform(String::length)
              .transform(len -> len * 2); // 6
```

---

## Java 15：Text Blocks 正式特性（JEP 378）

### 什么是 Text Blocks

多行字符串字面量，使用 `"""` 作为定界符，避免大量转义和拼接：

```java
// Java 15 之前
String html = "<html>\n" +
              "    <body>\n" +
              "        <p>Hello, world</p>\n" +
              "    </body>\n" +
              "</html>\n";

// Java 15+ Text Blocks
String html = """
              <html>
                  <body>
                      <p>Hello, world</p>
                  </body>
              </html>
              """;
```

### 编译时处理三步

1. **行终止符规范化**：CR 和 CRLF 统一转为 LF
2. **去除偶然空白**：根据闭合 `"""` 的位置自动去除前导缩进
3. **转义序列解析**：最后处理 `\n`、`\t` 等转义

### 新增转义序列

| 转义 | 含义 | 适用场景 |
|------|------|---------|
| `\<line-terminator>` | 显式抑制换行符 | 长行拆分 |
| `\s` | 单个空格（U+0020） | 保留尾部空格 |

```java
// 抑制换行
String s = """
           Lorem ipsum dolor sit amet, \
           consectetur adipiscing elit""";

// 保留尾部空格
String colors = """
    red  \s
    green\s
    blue \s
    """;
```

### 新增配套方法

| 方法 | 说明 |
|------|------|
| `stripIndent()` | 去除偶然空白（Text Blocks 的缩进算法） |
| `translateEscapes()` | 手动处理转义序列 |
| `formatted(Object... args)` | 值替换（类似 `String.format`） |

### 演进历史

| 版本 | 状态 | JEP |
|------|------|-----|
| Java 13 | 预览 | JEP 355 |
| Java 14 | 第二次预览（新增 `\s` 和 `\<line-terminator>`） | JEP 368 |
| Java 15 | 正式特性 | JEP 378 |

### 参考资料

- [JEP 378: Text Blocks](https://openjdk.org/jeps/378)

---

## Java 21：String Templates 预览（JEP 430）

### 核心概念

模板表达式 = 模板处理器 + 模板（含嵌入表达式）：

```java
String name = "Joan";
String info = STR."My name is \{name}";
// 结果: "My name is Joan"
```

### 内置模板处理器

| 处理器 | 功能 | 返回类型 |
|--------|------|---------|
| `STR` | 简单字符串插值 | `String` |
| `FMT` | 带格式说明符的插值 | `String` |
| `RAW` | 生成未处理的 `StringTemplate` 对象 | `StringTemplate` |

### `FMT` 格式化示例

```java
double x = 10.0, y = 3.0;
String s = FMT."%.2f\{x} / %.2f\{y} = %.2f\{x/y}";
// "10.00 / 3.00 = 3.33"
```

### 多行模板表达式

```java
String title = "My Page";
String html = STR."""
    <html>
      <head><title>\{title}</title></head>
      <body><p>Hello</p></body>
    </html>
    """;
```

### 安全设计

模板表达式**不能直接**从字符串字面量生成插值结果，必须经过模板处理器。这防止了 SQL 注入等安全问题：

```java
// 编译错误：缺少处理器
String query = "SELECT * FROM users WHERE name = '\{name}'";

// 正确：使用自定义处理器生成 PreparedStatement
var DB = new QueryBuilder(conn);
PreparedStatement ps = DB."SELECT * FROM users WHERE name = \{name}";
```

### 自定义模板处理器

```java
// 生成 JSONObject 的模板处理器
var JSON = StringTemplate.Processor.of(
    (StringTemplate st) -> new JSONObject(st.interpolate())
);

JSONObject doc = JSON."""
    {
        "name": "\{name}",
        "age": \{age}
    }
    """;
```

### 演进状态

| 版本 | 状态 | JEP |
|------|------|-----|
| Java 21 | 第一次预览 | JEP 430 |
| Java 22 | 第二次预览 | JEP 459 |
| Java 23 | 第三次预览 | JEP 467 |

### 参考资料

- [JEP 430: String Templates (Preview)](https://openjdk.org/jeps/430)

---

## Java 24：String 构造器弃用（JEP 466 相关）

### 弃用的构造器

以下构造器被标记为废弃（`@Deprecated(forRemoval = true)`）：

- `String(byte[])`
- `String(byte[], int, int)`
- `String(byte[], int, int, int)`（已废弃更早）

**原因**：使用平台默认字符集进行字节到字符的转换是不可预测的，应显式指定 `Charset`。

**替代方案**：

```java
// 废弃
String s = new String(bytes);

// 推荐
String s = new String(bytes, StandardCharsets.UTF_8);
```

---

## 版本演进总结

| 版本 | 关键变化 | 影响领域 |
|------|---------|---------|
| Java 1.0-6 | `char[]` 存储，`substring` 共享数组 | 内存泄漏风险 |
| Java 7u6 | `substring` 改为复制数组 | 修复内存泄漏 |
| Java 8 | G1 String Deduplication | 内存优化 |
| Java 9 | Compact Strings（`byte[]` + coder） | 内存减少 ~50% |
| Java 11 | `isBlank`、`strip`、`lines`、`repeat` 等新 API | API 增强 |
| Java 12 | `indent()`、`transform()` | API 增强 |
| Java 15 | Text Blocks 正式特性 | 可读性提升 |
| Java 21 | String Templates 预览 | 安全插值 |

---

## 最佳实践

### 1. 优先使用字符串字面量

```java
// 推荐
String s = "hello";

// 不推荐（无意义的对象创建）
String s = new String("hello");
```

### 2. 大量拼接使用 `StringBuilder`

```java
// 循环内拼接
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 1000; i++) {
    sb.append(i);
}
```

> 注：Java 9+ 编译器对单行 `+` 拼接使用 `invokedynamic` 优化（JEP 280），但循环内仍需手动使用 `StringBuilder`。

### 3. 使用 `strip()` 替代 `trim()`

```java
// 推荐（Unicode 感知）
String cleaned = input.strip();

// 不推荐（仅处理 ASCII 空白）
String cleaned = input.trim();
```

### 4. 字节转换时显式指定编码

```java
// 推荐
byte[] bytes = str.getBytes(StandardCharsets.UTF_8);
String s = new String(bytes, StandardCharsets.UTF_8);

// 不推荐（依赖平台默认编码）
byte[] bytes = str.getBytes();
String s = new String(bytes);
```

### 5. 利用 Compact Strings 特性

- Java 9+ 已自动优化，无需手动干预
- 对于纯 ASCII/Latin-1 场景，内存自动减半
- 混合编码场景注意轻微性能损耗

### 6. 使用 Text Blocks 提升可读性

```java
// 推荐
String sql = """
    SELECT id, name, email
    FROM users
    WHERE status = 'ACTIVE'
    ORDER BY name
    """;

// 不推荐
String sql = "SELECT id, name, email\n" +
             "FROM users\n" +
             "WHERE status = 'ACTIVE'\n" +
             "ORDER BY name\n";
```

---

## 常见误区

| 误区 | 事实 |
|------|------|
| `substring` 仍然共享数组 | Java 7u6+ 已改为复制 |
| `trim()` 能去除所有空白 | 仅处理 codepoint ≤ U+0020，`strip()` 才是 Unicode 感知的 |
| Compact Strings 需要手动开启 | Java 9+ 默认启用，无需配置 |
| `+` 拼接总是低效的 | Java 9+ 单行拼接使用 `invokedynamic` 优化，性能与 `StringBuilder` 相当 |
| Text Blocks 是新的字符串类型 | Text Blocks 编译后仍是 `String`，只是语法糖 |
| String Templates 已正式发布 | Java 21~23 仍为预览特性，需 `--enable-preview` |

---

## 相关概念

- **String Pool（字符串常量池）**：JVM 维护的 String 实例缓存，字面量和 `intern()` 结果存放于此
- **String Deduplication**：G1 GC 的去重机制，与 String Pool 不同，作用于堆中所有 String 对象
- **JEP 280: Indify String Concatenation**：Java 9 引入，使用 `invokedynamic` 优化字符串拼接
- **JEP 192: String Deduplication in G1**：Java 8u20 引入的 G1 GC 字符串去重
