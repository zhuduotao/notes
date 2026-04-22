---
created: 2026-04-22
updated: 2026-04-22
tags:
  - Spring
  - AOP
  - Java
  - 代理模式
  - 框架原理
aliases:
  - Spring AOP 原理
  - Spring AOP 实现机制
  - Spring AOP Proxy
  - Spring 面向切面编程
source_type: official-doc
source_urls:
  - https://docs.spring.io/spring-framework/reference/core/aop.html
  - https://docs.spring.io/spring-framework/reference/core/aop/introduction-defn.html
  - https://docs.spring.io/spring-framework/reference/core/aop/introduction-proxies.html
  - https://docs.spring.io/spring-framework/reference/core/aop/proxying.html
status: verified
---

## 是什么

Spring AOP（Aspect-Oriented Programming，面向切面编程）是 Spring Framework 的核心模块之一，用于将**横切关注点**（crosscutting concerns，如事务管理、日志、安全校验等）与核心业务逻辑解耦。

**核心结论：Spring AOP 基于动态代理（Proxy-based）实现，在运行时（Runtime）完成织入（Weaving），而非编译期织入。**

## 核心概念

以下术语为 AOP 通用概念，非 Spring 独有：

| 概念 | 说明 |
|------|------|
| **Aspect（切面）** | 横切多个类的关注点的模块化。在 Spring AOP 中，通过普通类或 `@Aspect` 注解类实现 |
| **Join point（连接点）** | 程序执行过程中的一个点，如方法调用、异常处理。**Spring AOP 中，连接点始终代表方法执行** |
| **Advice（通知）** | 切面在特定连接点采取的动作。常见类型：Before、After Returning、After Throwing、After (finally)、Around |
| **Pointcut（切点）** | 匹配连接点的谓词。通知与切点表达式关联，在匹配的连接点上执行。Spring 默认使用 AspectJ 切点表达式语言 |
| **Introduction（引入）** | 为类型声明额外的方法或字段。Spring AOP 允许向被通知对象引入新的接口及其实现 |
| **Target object（目标对象）** | 被一个或多个切面通知的对象，也称"advised object" |
| **AOP proxy（AOP 代理）** | AOP 框架创建的对象，用于实现切面契约。Spring 中使用 **JDK 动态代理** 或 **CGLIB 代理** |
| **Weaving（织入）** | 将切面与其他应用类型或对象链接以创建被通知对象的过程。可在编译期、加载期或运行时进行。**Spring AOP 在运行时织入** |

## 实现机制

### 代理方式选择

Spring AOP 使用两种代理机制，选择规则如下：

```
目标对象实现了至少一个接口 → 使用 JDK 动态代理（默认）
目标对象未实现任何接口   → 使用 CGLIB 代理（运行时生成的子类）
```

**JDK 动态代理**：JDK 内置，基于接口代理，所有目标类型实现的接口都会被代理。

**CGLIB 代理**：开源类定义库（已重新打包到 `spring-core` 中），基于继承实现，生成目标类型的运行时子类。

### 强制使用 CGLIB

某些场景需要强制使用 CGLIB（如需要代理未声明在接口上的方法），配置方式：

**XML 配置：**
```xml
<aop:config proxy-target-class="true">
    <!-- beans -->
</aop:config>

<aop:aspectj-autoproxy proxy-target-class="true"/>
```

**注解配置：**
```java
@EnableAspectJAutoProxy(proxyTargetClass = true)
@EnableTransactionManagement(proxyTargetClass = true)
```

> **注意**：多个 `<aop:config/>` 或注解配置会在运行时合并为统一的自动代理创建器，应用**最强**的代理设置。Spring Framework 7.0 起，`@EnableAsync` 等注解也参与统一的全局默认设置。
>
> Spring Framework 7.0 新增 `@Proxyable` 注解，支持对单个 Bean 强制指定代理类型：`@Proxyable(INTERFACES)` 或 `@Proxyable(TARGET_CLASS)`。

### 代理创建流程

1. Spring 容器启动时，扫描到 `@Aspect` 注解或 `<aop:config>` 配置
2. 注册 `AnnotationAwareAspectJAutoProxyCreator`（或对应的 BeanPostProcessor）
3. 在 Bean 初始化阶段（`postProcessAfterInitialization`），判断 Bean 是否匹配切点表达式
4. 若匹配，根据目标对象是否实现接口选择 JDK 动态代理或 CGLIB 代理
5. 将匹配的 Advice 组装为拦截器链（Interceptor Chain），附加到代理对象
6. 容器返回代理对象，替代原始目标对象

## 通知类型

Spring AOP 支持以下通知类型：

| 通知类型 | 执行时机 | 说明 |
|----------|----------|------|
| **Before** | 连接点执行前 | 无法阻止执行流程继续（除非抛出异常） |
| **After Returning** | 连接点正常完成后 | 方法正常返回时执行（未抛异常） |
| **After Throwing** | 方法抛出异常退出时 | 仅在异常场景执行 |
| **After (finally)** | 连接点退出时（无论正常或异常） | 相当于 finally 块 |
| **Around** | 环绕连接点 | 最强大的通知类型，可控制是否执行目标方法、修改返回值或抛出异常 |

**建议**：使用能实现所需行为的最弱通知类型。例如，只需用方法返回值更新缓存时，使用 After Returning 而非 Around，可简化编程模型并减少出错概率。Around 通知需要手动调用 `proceed()`，若遗漏会导致目标方法不执行。

## 核心用法示例

### @AspectJ 注解方式（推荐）

```java
@Aspect
@Component
public class LoggingAspect {

    @Before("execution(* com.example.service.*.*(..))")
    public void logBefore(JoinPoint joinPoint) {
        System.out.println("Executing: " + joinPoint.getSignature().getName());
    }

    @AfterReturning(pointcut = "execution(* com.example.service.*.*(..))", returning = "result")
    public void logAfterReturning(JoinPoint joinPoint, Object result) {
        System.out.println("Completed: " + joinPoint.getSignature().getName() + ", Result: " + result);
    }

    @Around("execution(* com.example.service.*.*(..))")
    public Object logAround(ProceedingJoinPoint joinPoint) throws Throwable {
        long start = System.currentTimeMillis();
        Object result = joinPoint.proceed();
        long elapsed = System.currentTimeMillis() - start;
        System.out.println("Method " + joinPoint.getSignature().getName() + " took " + elapsed + "ms");
        return result;
    }
}
```

启用 @AspectJ 支持：
```java
@Configuration
@EnableAspectJAutoProxy
public class AopConfig {
}
```

## 限制与注意事项

### 1. 自调用（Self-Invocation）问题

**这是 Spring AOP 最常见的陷阱。**

由于 Spring AOP 基于代理，当被代理对象内部通过 `this` 引用调用自身方法时，调用不会经过代理，因此关联的通知**不会执行**。

```java
public class SimplePojo implements Pojo {
    public void foo() {
        this.bar(); // 直接调用，不经过代理，bar() 上的通知不会执行
    }

    public void bar() {
        // some logic...
    }
}
```

**解决方案：**
- **重构代码**（推荐）：避免自调用
- **自注入**：注入自身 Bean 引用，通过该引用调用方法
- **`AopContext.currentProxy()`**：通过 `AopContext.currentProxy()` 获取当前代理对象调用（需配置 `exposeProxy=true`）

### 2. CGLIB 代理限制

使用 CGLIB 代理时存在以下约束：

- **`final` 类**无法被代理（不能被继承）
- **`final` 方法**无法被通知（不能被重写）
- **`private` 方法**无法被通知（不能被重写）
- **不可见方法**（如不同包中父类的 package-private 方法）无法被通知
- CGLIB 代理实例通过 Objenesis 创建，构造函数不会被调用两次（若 JVM 不允许绕过构造函数，可能出现双重调用）
- **Java Module System 限制**：在模块路径上部署时，无法为 `java.lang` 包中的类创建 CGLIB 代理，需要 JVM 启动参数 `--add-opens=java.base/java.lang=ALL-UNNAMED`

### 3. 仅支持方法级连接点

Spring AOP 的连接点**仅支持方法执行**，不支持字段访问、构造器调用等。如需更全面的 AOP 支持（如字段织入），需使用完整的 AspectJ。

### 4. 仅适用于 Spring 管理的 Bean

Spring AOP 只对 Spring IoC 容器管理的 Bean 生效。通过 `new` 关键字直接创建的对象不会被代理。

### 5. 代理类型与 Spring Boot 默认行为

Spring Framework 核心默认建议使用基于接口的代理（JDK 动态代理），但 **Spring Boot 可能根据配置属性默认启用基于类的代理（CGLIB）**。需注意两者差异。

## 与 AspectJ 的关系

Spring AOP 与完整 AspectJ 是两种不同的 AOP 实现：

| 对比维度 | Spring AOP | 完整 AspectJ |
|----------|------------|-------------|
| 织入时机 | 运行时 | 编译期 / 加载期 / 运行时 |
| 连接点类型 | 仅方法执行 | 方法执行、字段访问、构造器调用等 |
| 代理方式 | JDK 动态代理 / CGLIB | 字节码织入 |
| 依赖 | 无需额外编译器 | 需要 AspectJ 编译器（ajc） |
| 适用场景 | Spring Bean 的横切关注点 | 领域对象、非 Spring 管理的对象 |
| 切点表达式 | 复用 AspectJ 切点表达式语言 | 完整的 AspectJ 切点表达式 |

Spring AOP 使用 AspectJ 的切点表达式语言，但织入机制与 AspectJ 不同。两者可混合使用。

## 常见应用场景

- **声明式事务管理**：`@Transactional` 注解的底层实现
- **日志与审计**：统一记录方法调用、参数、返回值、执行时间
- **安全校验**：方法级别的权限检查
- **性能监控**：方法执行时间统计
- **缓存**：`@Cacheable` 等注解的底层实现
- **异常处理**：统一异常捕获与转换

## 参考资料

- Spring Framework 7.0.7 官方文档 - Aspect Oriented Programming with Spring: https://docs.spring.io/spring-framework/reference/core/aop.html
- AOP Concepts: https://docs.spring.io/spring-framework/reference/core/aop/introduction-defn.html
- AOP Proxies: https://docs.spring.io/spring-framework/reference/core/aop/introduction-proxies.html
- Proxying Mechanisms: https://docs.spring.io/spring-framework/reference/core/aop/proxying.html
- @AspectJ Support: https://docs.spring.io/spring-framework/reference/core/aop/ataspectj.html
- Spring AOP APIs: https://docs.spring.io/spring-framework/reference/core/aop-api.html
