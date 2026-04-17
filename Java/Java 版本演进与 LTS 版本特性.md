---
created: 2026-04-17
updated: 2026-04-17
tags:
  - java
  - jdk
  - lts
  - version-history
  - jvm
aliases:
  - Java 版本历史
  - Java LTS 版本
  - JDK 版本演进
  - Java SE 版本
source_type: mixed
source_urls:
  - https://www.oracle.com/java/technologies/java-se-support-roadmap.html
  - https://en.wikipedia.org/wiki/Java_version_history
  - https://openjdk.org/projects/jdk/
status: verified
---

## 概述

Java 自 1996 年发布 JDK 1.0 以来，经历了多次重大版本迭代。2017 年起，Oracle 将发布周期从原来的两年一次改为**每六个月发布一个特性版本**（Feature Release），并引入 **LTS（Long-Term Support，长期支持）** 机制，每隔约两年指定一个版本为 LTS 版本，提供更长的 Premier Support 和 Extended Support 周期。

> 发布周期变更提案由 Mark Reinhold（Java Platform 首席架构师）于 2017 年 9 月提出，详见 [JEP 322: Time-Based Release Versioning](http://openjdk.java.net/jeps/322)。

---

## LTS 与非 LTS 版本的区别

| 维度 | LTS 版本 | 非 LTS 版本 |
|------|----------|-------------|
| 支持周期 | Premier Support 约 5 年 + Extended Support 约 3 年 | 仅支持到下一个特性版本发布（约 6 个月） |
| 适用场景 | 生产环境、企业级应用 | 尝鲜新特性、开发测试 |
| 更新频率 | 持续收到安全补丁和 bug 修复 | 发布后即被下一版本取代 |
| 许可费用 | Oracle JDK 部分 LTS 版本在特定期限后需付费 | 通常随新版本发布而终止免费支持 |

**Oracle  Premier Support 策略**：非 LTS 版本被视为最近一个 LTS 版本的累积增强集。一旦新特性版本发布，之前的非 LTS 版本即被视为已取代（superseded）。例如 Java 9 被 Java 10 取代，Java 10 被 Java 11（LTS）取代。

---

## Java 版本总览

### 完整版本列表

| 版本 | 类型 | 发布日期 | Class File 版本 | Premier Support 截止 | Extended Support 截止 |
|------|------|----------|-----------------|----------------------|----------------------|
| JDK 1.0 | — | 1996-01-23 | 45 | — | — |
| JDK 1.1 | — | 1997-02-19 | 45 | — | — |
| J2SE 1.2 | — | 1998-12-08 | 46 | — | — |
| J2SE 1.3 | — | 2000-05-08 | 47 | — | — |
| J2SE 1.4 | — | 2002-02-06 | 48 | — | — |
| Java SE 5 (1.5) | — | 2004-09-30 | 49 | — | — |
| Java SE 6 (1.6) | — | 2006-12-11 | 50 | — | — |
| Java SE 7 (1.7) | — | 2011-07-28 | 51 | — | — |
| **Java SE 8** | **LTS** | 2014-03-18 | 52 | 2022-03 | **2030-12** |
| Java SE 9 | — | 2017-09-21 | 53 | — | — |
| Java SE 10 | — | 2018-03-20 | 54 | — | — |
| **Java SE 11** | **LTS** | 2018-09-25 | 55 | 2023-09 | **2032-01** |
| Java SE 12 | — | 2019-03-19 | 56 | — | — |
| Java SE 13 | — | 2019-09-17 | 57 | — | — |
| Java SE 14 | — | 2020-03-17 | 58 | — | — |
| Java SE 15 | — | 2020-09-16 | 59 | — | — |
| Java SE 16 | — | 2021-03-16 | 60 | — | — |
| **Java SE 17** | **LTS** | 2021-09-14 | 61 | 2026-09 | **2029-09** |
| Java SE 18 | — | 2022-03-22 | 62 | — | — |
| Java SE 19 | — | 2022-09-20 | 63 | — | — |
| Java SE 20 | — | 2023-03-21 | 64 | — | — |
| **Java SE 21** | **LTS** | 2023-09-19 | 65 | 2028-09 | **2031-09** |
| Java SE 22 | — | 2024-03-19 | 66 | — | — |
| Java SE 23 | — | 2024-09-17 | 67 | — | — |
| Java SE 24 | — | 2025-03-18 | 68 | — | — |
| **Java SE 25** | **LTS** | 2025-09-16 | 69 | 2030-09 | **2033-09** |
| Java SE 26 | — | 2026-03-17 | 70 | — | — |

> Oracle 计划每两年发布一个 LTS 版本，下一个计划中的 LTS 版本是 **Java 29**（预计 2027 年 9 月）。

---

## 当前受支持的 LTS 版本（截至 2026 年 4 月）

| LTS 版本 | Premier Support 截止 | Extended Support 截止 | 备注 |
|----------|----------------------|----------------------|------|
| **Java 8** | 2022-03（已过） | 2030-12 | 最后一个包含 Java Plugin / Web Start 的版本；Extended Support uplift fee 在 2022-03 至 2030-12 期间豁免 |
| **Java 11** | 2023-09（已过） | 2032-01 | 首个模块化系统（JPMS）LTS 版本；Extended Support uplift fee 在 2023-10 至 2032-01 期间豁免 |
| **Java 17** | 2026-09 | 2029-09 | 2024-10 起 Oracle JDK 17 更新转为 OTN 许可（不再免费用于商业用途） |
| **Java 21** | 2028-09 | 2031-09 | 2026-09 后 Oracle JDK 21 更新转为 OTN 许可 |
| **Java 25** | 2030-09 | 2033-09 | 最新 LTS 版本，截至 2025-09 起提供免费使用许可（NFTC） |

---

## 各 LTS 版本关键特性

### Java 8 (LTS) — 2014 年 3 月

Java 8 是 Java 历史上最具里程碑意义的版本之一，引入了多项改变编程范式的特性：

- **Lambda 表达式**：支持函数式编程风格
- **Stream API**：`java.util.stream` 包，支持对集合进行声明式数据处理
- **新的 Date/Time API**：`java.time` 包（JSR 310），替代旧的 `java.util.Date` 和 `Calendar`
- **默认方法（Default Methods）**：接口可以包含带实现的方法
- **`Optional` 类**：减少 `NullPointerException`
- **`CompletableFuture`**：异步编程支持
- **Nashorn JavaScript 引擎**：替代 Rhino（后续版本已移除）
- **重复注解（Repeating Annotations）**
- **类型注解（Type Annotations）**

> Java 8 是最后一个包含 Java Plugin（Applet）和 Java Web Start 的版本。Java 11 及以后版本已移除 Deployment Stack。

---

### Java 11 (LTS) — 2018 年 9 月

Java 11 是引入模块化系统（JPMS）后的首个 LTS 版本：

- **HTTP Client API（标准）**：`java.net.http` 包，支持 HTTP/2 和 WebSocket（JEP 321）
- **局部变量类型推断 `var`**：扩展至 lambda 参数（JEP 323）
- **Epsilon GC**：无操作垃圾收集器，用于性能测试（JEP 318）
- **ZGC（实验性）**：可扩展的低延迟垃圾收集器（JEP 333）
- **Flight Recorder**：从商业版开源（JEP 328）
- **Nashorn、Java EE、CORBA 模块移除**：`java.xml.ws`、`java.xml.bind`（JAXB）等被移除
- **单文件源程序运行**：`java SourceFile.java` 直接运行（JEP 330）
- **TLS 1.3 支持**（JEP 332）

---

### Java 17 (LTS) — 2021 年 9 月

Java 17 整合了多个预览特性并正式标准化：

- **Sealed Classes（密封类）**：限制哪些类可以继承（JEP 409）
- **Pattern Matching for `instanceof`**：模式匹配增强（JEP 394）
- **Text Blocks（文本块）**：多行字符串字面量（JEP 378）
- **Records（记录类）**：简洁的数据载体类（JEP 401）
- **`switch` 表达式**：增强版 switch（JEP 361）
- **ZGC 和 Shenandoah GC 正式支持**：低延迟垃圾收集器（JEP 377, JEP 379）
- **Foreign Function & Memory API（孵化）**：替代 JNI 的安全方案（JEP 412）
- **移除 RMI Activation**：RMI 激活机制被移除（JEP 385）
- **移除 Applet API**：正式废弃（JEP 398）
- **macOS/AArch64 支持**：Apple Silicon 原生支持（JEP 391）

---

### Java 21 (LTS) — 2023 年 9 月

Java 21 引入了多项重要新特性：

- **Virtual Threads（虚拟线程）**：Project Loom 的核心成果，轻量级并发（JEP 444）
- **Sequenced Collections**：有序集合接口（JEP 431）
- **Record Patterns**：记录模式匹配（JEP 440）
- **Pattern Matching for `switch`**：switch 模式匹配（JEP 441）
- **String Templates（预览）**：安全的字符串模板（JEP 430）
- **Unnamed Variables and Patterns**：未命名变量和模式（JEP 443）
- **Scoped Values（孵化）**：线程局部变量的替代方案（JEP 446）
- **Vector API（第六次孵化）**：SIMD 向量运算（JEP 448）
- **Generational ZGC**：分代 ZGC，降低内存开销（JEP 439）

---

### Java 25 (LTS) — 2025 年 9 月

Java 25 是最新的 LTS 版本，主要特性：

- **Implicitly Declared Classes**：简化入门程序（原Unnamed Classes + Main Method，JEP 463）
- **Module Import Declarations（预览）**：简化模块导入（JEP 468）
- **Value Types（预览）**：Project Valhalla 的核心成果，值类型支持（JEP 464）
- **Primitive Types in Patterns（预览）**：原始类型模式匹配（JEP 455）
- **Structured Concurrency**：结构化并发（JEP 453）
- **Stream Gatherers**：Stream 自定义中间操作（JEP 461）
- **Flexible Constructor Bodies**：灵活的构造函数体（JEP 447）
- **Graphviz 输出支持**：`jdeps` 和 `jdeprscan` 可输出 Graphviz 格式

> Java 25 是 GraalVM 最后一次作为 Oracle Java SE 产品一部分授权的版本（JDK 24 为最终版本）。

---

## 重要非 LTS 版本特性速览

| 版本 | 发布日期 | 关键特性 |
|------|----------|----------|
| Java 9 | 2017-09 | JPMS 模块化系统（JEP 261）、JShell REPL、G1 成为默认 GC |
| Java 10 | 2018-03 | `var` 局部变量类型推断（JEP 286） |
| Java 14 | 2020-03 | `switch` 表达式（标准）、Record（预览）、Pattern Matching for `instanceof`（预览） |
| Java 16 | 2021-03 | Record（标准）、Pattern Matching for `instanceof`（标准）、Vector API（孵化） |
| Java 19 | 2022-09 | Virtual Threads（预览）、Structured Concurrency（孵化） |
| Java 20 | 2023-03 | Scoped Values（孵化）、Record Patterns（第二次预览） |
| Java 26 | 2026-03 | 当前最新非 LTS 版本 |

---

## 版本号命名规则

Java 的版本号经历了多次变化：

| 时期 | 产品版本 | 开发者版本 | 说明 |
|------|----------|-----------|------|
| JDK 1.0 ~ 1.4 | J2SE 1.x | 1.x | 早期版本 |
| Java SE 5 | 5.0 | 1.5.0 | 为反映成熟度，产品版本跳至 5 |
| Java SE 6 ~ 7 | 6 / 7 | 1.6.0 / 1.7.0 | 产品版本与开发者版本并行 |
| Java SE 8+ | 8, 9, 10... | 8, 9, 10... | 统一版本号，`java -version` 直接显示 |

---

## 许可变更时间线

Oracle JDK 的许可政策经历了多次调整，影响免费使用范围：

| 时间 | 事件 |
|------|------|
| 2019-04 | Java 8 结束免费公开更新（Permissive Licensing 终止），个人/开发用途仍可通过 java.com 免费获取 |
| 2021-09 | Java 17 发布，初始采用 NFTC（No-Fee Terms and Conditions）许可 |
| 2023-09 | Java 21 发布，采用 NFTC 许可 |
| 2024-10 | Java 17 更新转为 OTN 许可（商业用途需订阅） |
| 2025-09 | Java 25 发布，采用 NFTC 许可；NFTC 许可覆盖 Java 21 及以后版本 |
| 2026-09（计划） | Java 21 更新转为 OTN 许可 |

> 许可详情参见 [Oracle Java SE Licensing FAQ](https://www.oracle.com/java/technologies/javase/jdk-faqs.html)。

---

## 选择哪个版本？

### 生产环境推荐

| 场景 | 推荐版本 | 理由 |
|------|----------|------|
| 遗留系统维护 | Java 8 | 生态最成熟，但 Extended Support 至 2030-12 |
| 新项目（保守） | Java 17 | 当前广泛采用的 LTS，生态完善 |
| 新项目（推荐） | Java 21 | 虚拟线程等现代特性，免费许可至 2026-09 |
| 追求最新 LTS | Java 25 | 最新 LTS，NFTC 免费许可 |

### 升级建议

1. **Java 8 → Java 11/17**：注意移除的模块（Java EE、CORBA、Nashorn 等），可能需要引入第三方依赖
2. **Java 11 → Java 17/21**：迁移成本较低，主要关注密封类、记录类等新特性的使用
3. **Java 17 → Java 21**：虚拟线程是最大亮点，可显著简化高并发应用

---

## 参考资料

- [Oracle Java SE Support Roadmap](https://www.oracle.com/java/technologies/java-se-support-roadmap.html)
- [Java version history - Wikipedia](https://en.wikipedia.org/wiki/Java_version_history)
- [OpenJDK Projects](https://openjdk.org/projects/jdk/)
- [JEP 322: Time-Based Release Versioning](http://openjdk.java.net/jeps/322)
- [Oracle Java SE Licensing FAQ](https://www.oracle.com/java/technologies/javase/jdk-faqs.html)
- [Oracle No-Fee Terms and Conditions License](https://www.oracle.com/downloads/licenses/no-fee-license.html)
