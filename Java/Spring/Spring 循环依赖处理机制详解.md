---
created: '2026-04-23'
updated: '2026-04-23'
tags:
  - Java
  - Spring
  - IoC
  - 依赖注入
  - 循环依赖
  - 三级缓存
aliases:
  - Spring Circular Dependency
  - Spring循环引用
source_type: official-doc
source_urls:
  - >-
    https://docs.spring.io/spring-framework/reference/core/beans/dependencies/factory-collaborators.html
  - >-
    https://github.com/spring-projects/spring-framework/blob/main/spring-beans/src/main/java/org/springframework/beans/factory/support/DefaultSingletonBeanRegistry.java
  - >-
    https://github.com/spring-projects/spring-framework/blob/main/spring-beans/src/main/java/org/springframework/beans/factory/support/AbstractAutowireCapableBeanFactory.java
status: verified
---
## 是什么

循环依赖（Circular Dependency）是指两个或多个 Bean 相互依赖，形成依赖闭环。例如：Bean A 依赖 Bean B，Bean B 又依赖 Bean A。

Spring 通过**三级缓存机制**解决 setter 注入的循环依赖，但**无法解决构造器注入的循环依赖**。

## 为什么重要

循环依赖是依赖注入系统的常见问题。如果不处理，容器在创建 Bean 时会陷入死循环或无法完成初始化。Spring 的三级缓存机制是其 IoC 容器的核心特性之一，理解这一机制有助于：

- 排查 Bean 创建失败问题
- 设计合理的依赖结构
- 理解 Spring AOP 与循环依赖的关系

## Spring 如何处理

### 三级缓存定义

Spring 在 `DefaultSingletonBeanRegistry` 中定义了三个缓存：

| 缓存 | 变量名 | 作用 |
|------|--------|------|
| 一级缓存 | `singletonObjects` | 存储完全初始化好的 Bean（`ConcurrentHashMap`，初始容量 256） |
| 二级缓存 | `earlySingletonObjects` | 存储早期暴露的 Bean 引用（已从工厂获取） |
| 三级缓存 | `singletonFactories` | 存储 `ObjectFactory`，用于动态生成早期引用（支持 AOP 代理） |

源码位置：`DefaultSingletonBeanRegistry.java:52-65`

```java
/** Cache of singleton objects: bean name to bean instance. */
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

/** Cache of early singleton objects: bean name to bean instance. */
private final Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>(16);

/** Cache of singleton factories: bean name to ObjectFactory. */
private final Map<String, ObjectFactory<?>> singletonFactories = new ConcurrentHashMap<>(16);
```

### 解决流程

假设 Bean A 和 Bean B 通过 setter 注入相互依赖：

1. **创建 Bean A**
   - 实例化 A（调用构造器）→ A 对象创建完成（属性未填充）
   - 将 A 的 `ObjectFactory` 加入三级缓存 `singletonFactories`
   - 属性填充时发现需要注入 B

2. **创建 Bean B**
   - 实例化 B → B 对象创建完成
   - 将 B 的 `ObjectFactory` 加入三级缓存
   - 属性填充时发现需要注入 A
   - 调用 `getSingleton("A")` 查找 A：
     - 一级缓存：无
     - 二级缓存：无
     - 三级缓存：有 → 调用 `ObjectFactory.getObject()` 获取早期引用
     - 将 A 从三级缓存升级到二级缓存
   - B 成功注入 A 的早期引用
   - B 初始化完成，加入一级缓存

3. **回到 Bean A**
   - A 成功注入完整的 B
   - A 初始化完成，加入一级缓存

### getSingleton 方法核心逻辑

源码位置：`DefaultSingletonBeanRegistry.java:87-115`

```java
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    Object singletonObject = this.singletonObjects.get(beanName);  // 一级缓存
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        singletonObject = this.earlySingletonObjects.get(beanName);  // 二级缓存
        if (singletonObject == null && allowEarlyReference) {
            ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);  // 三级缓存
            if (singletonFactory != null) {
                singletonObject = singletonFactory.getObject();
                this.singletonFactories.remove(beanName);
                this.earlySingletonObjects.put(beanName, singletonObject);  // 升级到二级缓存
            }
        }
    }
    return singletonObject;
}
```

### doCreateBean 中的早期暴露

源码位置：`AbstractAutowireCapableBeanFactory.java:556-651`

Bean 实例化后、属性填充前，Spring 会判断是否需要暴露早期引用：

```java
boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
        isSingletonCurrentlyInCreation(beanName));
if (earlySingletonExposure) {
    addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
}
```

`getEarlyBeanReference` 方法支持 AOP 代理的提前创建：

```java
protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
    Object exposedObject = bean;
    if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
        for (SmartInstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().smartInstantiationAware) {
            exposedObject = bp.getEarlyBeanReference(exposedObject, beanName);
        }
    }
    return exposedObject;
}
```

### 为什么需要三级缓存

三级缓存使用 `ObjectFactory` 而非直接存储对象，原因：

- **延迟决策**：在实例化阶段不确定是否需要 AOP 代理
- **动态代理**：`getEarlyBeanReference` 可以在获取时动态创建代理
- 如果只有两级缓存，就需要在实例化后立即决定是否代理，但此时可能还不确定

## 限制条件

### 能解决的情况

| 条件 | 说明 |
|------|------|
| Singleton 作用域 | Prototype 作用域 Bean 不缓存，无法解决 |
| Setter 注入 | 构造器注入无法解决 |
| 允许循环引用 | `allowCircularReferences = true`（默认开启） |

### 无法解决的情况

| 情况 | 原因 |
|------|------|
| 构造器注入循环依赖 | 实例化前就需要依赖，无法提前暴露早期引用 |
| Prototype 作用域循环依赖 | 容器不缓存 Prototype Bean |
| `@Async` 标注的 Bean | `AsyncAnnotationBeanPostProcessor` 不实现 `SmartInstantiationAwareBeanPostProcessor`，无法提前代理 |

### 官方文档说明

> If you use predominantly constructor injection, it is possible to create an unresolvable circular dependency scenario.
> 
> For example: Class A requires an instance of class B through constructor injection, and class B requires an instance of class A through constructor injection. If you configure beans for classes A and B to be injected into each other, the Spring IoC container detects this circular reference at runtime, and throws a `BeanCurrentlyInCreationException`.

来源：[Spring 官方文档 - Circular dependencies](https://docs.spring.io/spring-framework/reference/core/beans/dependencies/factory-collaborators.html)

## 常见问题与解决方案

### 构造器注入循环依赖

**错误现象**：启动时抛出 `BeanCurrentlyInCreationException`

**解决方案**：
1. 改用 setter 注入（官方推荐）
2. 使用 `@Lazy` 注解延迟加载依赖

```java
@Component
public class ServiceA {
    private final ServiceB serviceB;
    
    public ServiceA(@Lazy ServiceB serviceB) {
        this.serviceB = serviceB;
    }
}
```

### 禁用循环依赖

Spring Boot 配置：

```yaml
spring:
  main:
    allow-circular-references: false
```

或代码设置：

```java
context.setAllowCircularReferences(false);
```

### AOP 代理与循环依赖

当 Bean 需要被 AOP 代理，且存在循环依赖时：
- 三级缓存的 `ObjectFactory` 会调用 `getEarlyBeanReference` 提前创建代理
- 最终注入的是代理对象而非原始对象

## 相关概念

- **依赖注入（DI）**：对象通过构造器、setter 或字段声明依赖，容器负责注入
- **Bean 生命周期**：实例化 → 属性填充 → 初始化 → 销毁
- **AOP 代理**：Spring 通过 `BeanPostProcessor` 在初始化阶段创建代理
- **SmartInstantiationAwareBeanPostProcessor**：支持提前获取 Bean 早期引用的后处理器

## 参考资料

- [Spring Framework 官方文档 - Dependency Injection](https://docs.spring.io/spring-framework/reference/core/beans/dependencies/factory-collaborators.html)
- [DefaultSingletonBeanRegistry.java 源码](https://github.com/spring-projects/spring-framework/blob/main/spring-beans/src/main/java/org/springframework/beans/factory/support/DefaultSingletonBeanRegistry.java)
- [AbstractAutowireCapableBeanFactory.java 源码](https://github.com/spring-projects/spring-framework/blob/main/spring-beans/src/main/java/org/springframework/beans/factory/support/AbstractAutowireCapableBeanFactory.java)
