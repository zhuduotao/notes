---
created: '2026-04-23'
updated: '2026-04-23'
tags:
  - Spring
  - Java
  - IoC
  - Bean
  - singleton
  - scope
  - 依赖注入
aliases:
  - Spring Bean 单例
  - Spring Bean Scope
  - Spring 作用域
source_type: official-doc
source_urls:
  - >-
    https://docs.spring.io/spring-framework/reference/core/beans/factory-scopes.html
status: verified
---
## 核心结论

**Spring Bean 默认是单例**，但这里的"单例"概念与传统 GoF 单例模式有本质区别：

- **Spring 单例**：每个 Spring IoC 容器中，每个 Bean 定义对应唯一一个实例
- **GoF 单例**：每个 ClassLoader 中，某个类有且仅有一个实例（通过代码硬编码控制）

Spring 通过配置控制作用域，而非在类级别硬编码，灵活性更高。

---

## Spring 支持的六种作用域

| 作用域 | 说明 | 适用环境 |
|--------|------|----------|
| **singleton** |（默认）每个容器每个 Bean 定义对应唯一实例 | 所有环境 |
| **prototype** | 每次请求该 Bean 都创建新实例 | 所有环境 |
| **request** | 每个 HTTP 请求对应一个实例 | Web-aware ApplicationContext |
| **session** | 每个 HTTP Session 对应一个实例 | Web-aware ApplicationContext |
| **application** | 每个 ServletContext 对应一个实例 | Web-aware ApplicationContext |
| **websocket** | 每个 WebSocket 对应一个实例 | Web-aware ApplicationContext |

> 注：`request`、`session`、`application`、`websocket` 仅在 Web-aware Spring `ApplicationContext`（如 `XmlWebApplicationContext`）中有效。若在普通 IoC 容器（如 `ClassPathXmlApplicationContext`）中使用，会抛出 `IllegalStateException`。

---

## Singleton 作用域详解

### 行为机制

- Spring 容器创建 Bean 定义对应的唯一实例
- 该实例被缓存在 singleton beans 缓存中
- 后续所有对该 Bean 的请求和引用均返回缓存对象

### 配置方式

```xml
<!-- 默认就是 singleton，以下写法等价 -->
<bean id="accountService" class="com.something.DefaultAccountService"/>
<bean id="accountService" class="com.something.DefaultAccountService" scope="singleton"/>
```

Java Config：

```java
@Bean
@Scope("singleton") // 可省略，默认值
public AccountService accountService() {
    return new DefaultAccountService();
}
```

### 与 GoF 单例模式对比

| 维度 | Spring Singleton | GoF Singleton |
|------|------------------|---------------|
| 控制方式 | 容器配置 | 类内部硬编码 |
| 实例范围 | per-container, per-bean | per-ClassLoader |
| 灵活性 | 高（可随时切换作用域） | 低（需修改代码） |
| 测试友好度 | 高（不同容器可隔离） | 低（全局状态难以隔离） |

---

## Prototype 作用域详解

### 行为机制

- 每次对该 Bean 的请求（通过 `getBean()` 或依赖注入）都创建新实例
- Spring 不管理 Prototype Bean 的完整生命周期
- **初始化回调会执行，但销毁回调不执行**

> 原因：容器创建 Prototype Bean 后交给客户端，不再持有引用，无法触发销毁逻辑。客户端需自行清理资源和释放昂贵对象。

### 配置方式

```xml
<bean id="accountService" class="com.something.DefaultAccountService" scope="prototype"/>
```

Java Config：

```java
@Bean
@Scope("prototype")
public AccountService accountService() {
    return new DefaultAccountService();
}
```

### 使用建议

- **有状态 Bean**：使用 prototype
- **无状态 Bean**：使用 singleton（如 DAO、Service）

---

## 单例 Bean 依赖原型 Bean 的问题

### 问题描述

若 Singleton Bean 依赖 Prototype Bean，依赖在 Singleton 初始化时就已解析完成，导致该 Prototype 实例被"固定"在 Singleton 中。后续 Singleton 每次使用时，永远是同一个 Prototype 实例，而非新实例。

### 解决方案：方法注入

使用 Spring 的 **Method Injection**（Lookup Method Injection）：

```java
@Component
public class SingletonService {
    
    // 每次调用都返回新的 Prototype Bean 实例
    @Lookup
    protected PrototypeBean getPrototypeBean() {
        return null; // Spring 会覆盖此方法实现
    }
    
    public void doWork() {
        PrototypeBean bean = getPrototypeBean(); // 每次都是新实例
        bean.process();
    }
}
```

XML 配置方式：

```xml
<bean id="singletonService" class="com.example.SingletonService">
    <lookup-method name="getPrototypeBean" bean="prototypeBean"/>
</bean>

<bean id="prototypeBean" class="com.example.PrototypeBean" scope="prototype"/>
```

---

## Web 作用域配置前提

使用 `request`、`session`、`application`、`websocket` 作用域前，需完成 Web 配置：

### Spring MVC 配置

若使用 Spring MVC，在 `DispatcherServlet` 上下文中配置：

```xml
<listener>
    <listener-class>org.springframework.web.context.request.RequestContextListener</listener-class>
</listener>
```

或通过 `RequestContextFilter`：

```xml
<filter>
    <filter-name>requestContextFilter</filter-name>
    <filter-class>org.springframework.web.filter.RequestContextFilter</filter-class>
</filter>
<filter-mapping>
    <filter-name>requestContextFilter</filter-name>
    <url-pattern>/*</url-pattern>
</filter-mapping>
```

---

## 自定义作用域

Spring 支持自定义作用域，需实现 `Scope` 接口并注册：

```java
public class CustomScope implements Scope {
    // 实现 get(), remove(), registerDestructionCallback() 等方法
}
```

注册方式（通过 `CustomScopeConfigurer`）：

```xml
<bean class="org.springframework.beans.factory.config.CustomScopeConfigurer">
    <property name="scopes">
        <map>
            <entry key="customScope">
                <bean class="com.example.CustomScope"/>
            </entry>
        </map>
    </property>
</bean>
```

---

## 常见误区

| 误区 | 正确理解 |
|------|----------|
| "Spring 单例 = GoF 单例" | Spring 单例是容器级别的，不同容器可有不同实例 |
| "单例 Bean 无线程安全问题" | 单例 Bean 若包含可变状态，仍需考虑线程安全 |
| "Prototype Bean 会被 Spring 销毁" | Prototype Bean 的销毁回调不会被 Spring 调用 |
| "每次注入都创建新 Prototype" | Singleton 依赖 Prototype 时，只在 Singleton 初始化时注入一次 |

---

## 相关概念

- [[Spring AOP 实现原理详解]]：AOP Proxy 对作用域的影响
- [[Java 线程池详解：原理与生产环境最佳实践]]：有状态对象与线程池的配合
- [[Java volatile 关键字实现原理]]：单例模式 DCL（Double-Checked Locking）中的 volatile 用途

---

## 参考资料

- [Spring Framework Docs: Bean Scopes](https://docs.spring.io/spring-framework/reference/core/beans/factory-scopes.html)
- [Spring Framework Docs: Method Injection](https://docs.spring.io/spring-framework/reference/core/beans/dependencies/factory-method-injection.html)
- [Spring Framework Javadoc: Scope Interface](https://docs.spring.io/spring-framework/docs/7.0.7/javadoc-api/org/springframework/config/Scope.html)
