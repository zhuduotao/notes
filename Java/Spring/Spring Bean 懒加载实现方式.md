---
created: '2026-04-23'
updated: '2026-04-23'
tags:
  - Spring
  - Java
  - IoC
  - Bean
  - lazy-initialization
aliases:
  - Spring懒加载
  - '@Lazy注解'
  - lazy-init
source_type: official-doc
source_urls:
  - >-
    https://docs.spring.io/spring-framework/reference/core/beans/dependencies/factory-lazy-init.html
  - >-
    https://docs.spring.io/spring-framework/docs/7.0.7/javadoc-api/org/springframework/context/annotation/Lazy.html
status: verified
---
## 是什么

Spring Bean 懒加载（Lazy Initialization）是一种延迟 Bean 实例化的机制。默认情况下，`ApplicationContext` 在启动时会立即创建并配置所有 singleton Bean（预实例化）。通过懒加载配置，可以告诉 IoC 容器在 Bean 首次被请求时才创建实例，而非启动时立即创建。

## 为什么重要

- **启动性能优化**：避免启动时加载所有 Bean，缩短应用启动时间
- **资源节省**：对于创建成本高昂的 Bean，仅在真正需要时才初始化
- **错误延迟发现**：配置错误不再在启动时立即暴露，而是在首次使用时发现（需权衡利弊）

## 实现方式

### 1. `@Lazy` 注解（推荐）

可用于 `@Component`、`@Bean` 方法、`@Configuration` 类。

```java
@Bean
@Lazy
ExpensiveToCreateBean lazyBean() {
    return new ExpensiveToCreateBean();
}
```

在 `@Component` 类上使用：

```java
@Component
@Lazy
public class HeavyService {
    // 首次使用时才初始化
}
```

在 `@Configuration` 类上使用（所有 `@Bean` 方法均懒加载）：

```java
@Configuration
@Lazy
public class LazyConfiguration {
    // 该类中所有 @Bean 方法均懒加载
}
```

在 `@Configuration` 类全局懒加载时，可对特定 Bean 覆盖为立即加载：

```java
@Configuration
@Lazy
public class LazyConfiguration {
    
    @Bean
    @Lazy(false)  // 覆盖为立即加载
    public AnotherBean eagerBean() {
        return new AnotherBean();
    }
}
```

### 2. XML 配置 `lazy-init` 属性

单 Bean 配置：

```xml
<bean id="lazyBean" class="com.example.ExpensiveToCreateBean" lazy-init="true"/>
```

全局配置（所有 Bean 默认懒加载）：

```xml
<beans default-lazy-init="true">
    <!-- 所有 Bean 懒加载 -->
</beans>
```

### 3. 注入点使用 `@Lazy`

在依赖注入点标注 `@Lazy`，会创建懒解析代理，首次访问时才获取真实 Bean：

```java
@Component
public class ClientService {
    
    @Autowired
    @Lazy
    private HeavyService heavyService;  // 注入代理，首次调用方法时才初始化
}
```

这是使用 `ObjectFactory` 或 `Provider` 的替代方案。

## 行为规则

| 配置方式 | 行为 |
|---------|------|
| 单 Bean `@Lazy` 或 `lazy-init="true"` | 首次请求时创建 |
| `@Configuration` 类 `@Lazy` | 类内所有 `@Bean` 懒加载 |
| `default-lazy-init="true"` | 所有 Bean 默认懒加载 |
| 注入点 `@Lazy` | 创建懒解析代理 |

### 依赖传递规则

**重要**：如果懒加载 Bean 是某个**非懒加载 singleton Bean 的依赖**，该懒加载 Bean 会在启动时被创建，以满足 singleton 的依赖需求。

```java
@Bean
@Lazy
public LazyBean lazyBean() { ... }

@Bean  // 非懒加载
public SingletonBean singletonBean() {
    return new SingletonBean(lazyBean());  // lazyBean 会在启动时被创建
}
```

## 适用场景

- Bean 初始化成本高昂（如需要加载大量数据、建立复杂连接）
- Bean 可能不被使用（如条件性功能模块）
- 开发环境快速启动优化
- 特定 Bean 的依赖关系复杂，需要延迟初始化

## 限制与注意事项

1. **依赖传播问题**：懒加载 Bean 被非懒 singleton 依赖时，仍会在启动时初始化
2. **错误延迟发现**：配置错误、依赖缺失等问题在启动时不会暴露，仅在使用时才发现
3. **代理行为**：注入点 `@Lazy` 总是注入代理；若目标依赖不存在，只能在调用时抛出异常
4. **可选依赖**：对于可选依赖，注入点 `@Lazy` 行为不够直观，推荐使用 `ObjectProvider` 替代

```java
@Autowired
private ObjectProvider<OptionalService> optionalService;  // 更优雅的可选依赖处理
```

5. **AOT 编译限制**：Spring AOT（Ahead-of-Time）编译时，懒加载可能受影响，需查阅具体版本文档

## 与 `ObjectProvider`/`ObjectFactory` 的区别

| 特性 | `@Lazy` 注入点 | `ObjectProvider` |
|------|---------------|------------------|
| 获取时机 | 首次方法调用时 | 显式调用 `getObject()` |
| 可选依赖支持 | 不直观（调用时抛异常） | 支持 `getIfAvailable()` |
| 重复获取 | singleton 缓存，prototype 每次重解析 | 灵活控制 |

## 参考资料

- [Spring Framework Reference - Lazy-initialized Beans](https://docs.spring.io/spring-framework/reference/core/beans/dependencies/factory-lazy-init.html)
- [`@Lazy` Annotation JavaDoc (Spring 7.0.7)](https://docs.spring.io/spring-framework/docs/7.0.7/javadoc-api/org/springframework/context/annotation/Lazy.html)
