---
created: '2026-04-22'
updated: '2026-04-22'
tags:
  - Node.js
  - NestJS
  - 依赖注入
  - IoC
  - forwardRef
  - 循环依赖
aliases:
  - NestJS DI
  - NestJS Circular Dependency
source_type: mixed
source_urls:
  - 'https://docs.nestjs.com/fundamentals/dependency-injection'
  - 'https://docs.nestjs.com/fundamentals/circular-dependency'
  - 'https://docs.nestjs.com/fundamentals/module-reference'
  - 'https://deepwiki.com/nestjs/nest/3.1-dependency-injection-system'
status: verified
---
NestJS 的依赖注入（Dependency Injection，DI）是基于 Inversion of Control（IoC）容器实现的。IoC 容器负责管理组件的创建、依赖解析和实例生命周期，开发者只需通过装饰器声明依赖，框架自动完成注入。循环依赖是 DI 系统的常见问题，NestJS 提供 `forwardRef` 前向引用和 `ModuleRef` 动态引用两种解决方案。

## 依赖注入基础概念

### IoC 容器

IoC 容器是一个 Key-Value 注册表：
- **Key（Token）**：标识依赖的类型，通常是类名、字符串或 Symbol
- **Value**：描述如何创建或获取依赖实例的配置

NestJS 通过 `@Injectable()` 装饰器将类注册为 Provider，IoC 容器在应用启动时解析依赖图并实例化所有 Provider^[1]。

### Provider

Provider 是 NestJS DI 系统的核心单元，可以是：
- **类 Provider**：使用 `@Injectable()` 装饰的类，默认 singleton 作用域
- **值 Provider**：直接提供预创建的值 `{ provide: 'CONFIG', useValue: config }`
- **工厂 Provider**：通过工厂函数动态创建 `{ provide: 'DB', useFactory: () => createConnection() }`
- **别名 Provider**：将现有 Provider 映射到新 Token `{ provide: 'Logger', useClass: ConsoleLogger }`

### 注入方式

依赖通过构造函数注入：

```typescript
@Injectable()
export class UserService {
  constructor(private readonly userRepository: UserRepository) {}
}
```

NestJS 利用 TypeScript 的 emitDecoratorMetadata 编译选项，通过反射机制读取构造函数参数类型，自动注入对应 Provider^[1]。

## IoC 容器实现机制

### 元数据收集

NestJS 使用 TypeScript 反射 API（`reflect-metadata`）收集依赖信息：

1. `@Injectable()` 装饰器标记类为 Provider
2. `@Module()` 装饰器的 `providers` 数组注册 Provider 到模块
3. `@Controller()` 装饰器标记控制器，控制器自动注册为消费者
4. 编译时 `emitDecoratorMetadata: true` 生成元数据，记录构造函数参数类型

元数据存储在类的 `__metadata__` 属性中，IoC 容器通过反射读取。

### 依赖图构建

应用启动时，NestJS 执行以下流程^[3]：

1. **模块扫描**：遍历所有模块，收集 providers、controllers、imports、exports
2. **依赖图构建**：为每个 Provider 创建节点，节点间建立依赖关系边
3. **拓扑排序**：按依赖关系排序实例化顺序，确保依赖先于消费者实例化
4. **实例化**：按拓扑顺序创建 Provider 实例，注入依赖

依赖图是有向无环图（DAG），若存在循环依赖则拓扑排序失败，抛出错误。

### 实例缓存

IoC 容器维护实例缓存：
- **Singleton 作用域**：整个应用生命周期内唯一实例
- **Request 作用域**：每个请求创建新实例
- **Transient 作用域**：每次注入创建新实例

容器在创建实例前检查缓存，避免重复实例化。

## 循环依赖的成因

循环依赖指两个或多个类相互依赖：
- Service A 依赖 Service B
- Service B 依赖 Service A

在依赖图视角，A → B → A 形成环，拓扑排序无法确定实例化顺序：创建 A 需要 B，但 B 需要 A，两者都无法先完成实例化^[2]。

### 循环依赖类型

| 类型 | 描述 | 示例 |
|------|------|------|
| Provider 循环 | 同一模块内两个 Service 相互依赖 | UserService ↔ AuthService |
| Module 循环 | 两个模块相互导入彼此的 Provider | ModuleA imports ModuleB，ModuleB imports ModuleA |
| 跨模块循环 | Service A（在 ModuleA）依赖 Service B（在 ModuleB），B 又依赖 A | 需 forwardRef 和 Module exports 配合 |

## forwardRef 解决循环依赖

### 前向引用原理

`forwardRef` 是 NestJS 提供的工具函数，允许引用尚未定义的类。其原理：

1. **延迟解析**：`forwardRef(() => TargetClass)` 不立即返回类引用，而是返回包装函数
2. **延迟实例化**：IoC 容器在解析依赖图时识别 forwardRef，暂缓实例化该依赖
3. **后续注入**：先实例化循环中的一方，另一方在首次使用时才获取实例（通过 Lazy Reference）

### Provider 循环依赖示例

```typescript
@Injectable()
export class UserService {
  constructor(
    @Inject(forwardRef(() => AuthService))
    private authService: AuthService,
  ) {}
}

@Injectable()
export class AuthService {
  constructor(
    @Inject(forwardRef(() => UserService))
    private userService: UserService,
  ) {}
}
```

关键点：
- 两侧都使用 `@Inject(forwardRef(() => OtherService))`
- `forwardRef` 包装返回类的函数，而非直接类引用
- IoC 容器先创建一方实例，另一方在运行时获取^[2]

### Module 循环依赖示例

```typescript
@Module({
  imports: [forwardRef(() => ModuleB)],
  exports: [ServiceA],
})
export class ModuleA {}

@Module({
  imports: [forwardRef(() => ModuleA)],
  exports: [ServiceB],
})
export class ModuleB {}
```

关键点：
- 两模块相互导入时，使用 `forwardRef` 包装模块引用
- 导入的模块必须导出对应的 Provider（通过 `exports`）
- forwardRef 打破模块导入顺序的依赖^[2]

### forwardRef 的实现机制

`forwardRef` 内部实现：
- 返回一个 Lazy Reference 对象，持有函数引用
- IoC 容器在解析阶段标记 Lazy Reference
- 实例化阶段，容器先实例化非 Lazy 依赖，再处理 Lazy Reference
- Lazy Reference 在首次访问时执行函数，返回实际类引用，触发实例化

## ModuleRef 替代方案

### ModuleRef 简介

`ModuleRef` 是 NestJS 提供的动态依赖注入工具，允许在运行时按需获取 Provider 实例，而非通过构造函数静态注入^[2][^4]。

### 解决循环依赖

使用 `ModuleRef` 在循环依赖的一侧动态获取另一侧实例：

```typescript
@Injectable()
export class UserService {
  constructor(private moduleRef: ModuleRef) {}

  private authService: AuthService;

  onModuleInit() {
    this.authService = this.moduleRef.get(AuthService, { strict: false });
  }
}

@Injectable()
export class AuthService {
  constructor(private userService: UserService) {}
}
```

关键点：
- UserService 通过 `ModuleRef.get()` 在模块初始化后获取 AuthService
- AuthService 正常通过构造函数注入 UserService
- 打破构造函数层面的循环依赖

### ModuleRef API

| 方法 | 描述 | 参数 |
|------|------|------|
| `get<T>(token, options)` | 获取 Provider 实例 | `strict: true` 限制当前模块 |
| `resolve<T>(token, options)` | 解析 Provider（支持 Request 作用域） | 返回 Promise |
| `create<T>(type)` | 动态创建实例（不注册到容器） | 用于非 Provider 类 |
| `registerRequestByContextId<T>(request, contextId)` | 注册请求上下文 | Request 作用域场景 |

### get vs resolve

- **get**：同步获取，适用于 Singleton 作用域 Provider
- **resolve**：异步解析，支持 Request/Transient 作用域，返回 Promise

## 最佳实践与限制

### 优先重构而非 forwardRef

循环依赖通常是架构设计问题的信号：
- **提取公共服务**：将循环依赖的公共逻辑提取到独立 Service
- **引入事件/消息**：通过 EventEmitter 或消息队列解耦
- **使用 Facade**：创建统一接口层，打破直接依赖

forwardRef 和 ModuleRef 是临时方案，重构是根本解决^[2]。

### forwardRef 的限制

| 限制 | 说明 |
|------|------|
| 实例化顺序不确定 | 不应依赖哪一方先实例化，避免在构造函数中访问循环依赖 |
| Request 作用域冲突 | 循环依赖涉及 Request 作用域 Provider 可能导致 undefined 依赖^[2] |
| 两侧都需要 forwardRef | Provider 循环依赖需两侧都用 forwardRef，否则无法解析 |

### ModuleRef 的限制

| 限制 | 说明 |
|------|------|
| 严格模式限制 | `strict: true` 时只能获取当前模块内的 Provider |
| 需手动管理生命周期 | 动态创建的实例不在 IoC 容器管理范围内 |
| 测试难度增加 | 动态依赖难以 Mock，增加单元测试复杂度 |

### 避免在构造函数中访问循环依赖

```typescript
@Injectable()
export class UserService {
  constructor(
    @Inject(forwardRef(() => AuthService))
    private authService: AuthService,
  ) {
    // 错误：此时 authService 可能尚未实例化
    this.authService.doSomething();
  }

  onModuleInit() {
    // 正确：模块初始化后，authService 已可用
    this.authService.doSomething();
  }
}
```

原因是 forwardRef 延迟实例化，构造函数执行时另一侧可能未完成实例化。

## 深层原理：依赖图与拓扑排序

### 依赖图结构

NestJS 将所有 Provider 组织为依赖图：
- 节点：每个 Provider
- 边：依赖关系，A 依赖 B 则存在 A → B 的边

依赖图是 DAG 时，可拓扑排序；存在环时，拓扑排序失败。

### 拓扑排序算法

正常 DI 解析流程：
1. 构建依赖图
2. 计算每个节点的入度（被依赖次数）
3. 入度为 0 的节点先实例化
4. 实例化后，减少其依赖节点的入度
5. 重复直到所有节点实例化

循环依赖破坏此流程：环中节点入度永不为 0。

### forwardRef 如何打破环

forwardRef 将依赖边标记为 Lazy：
- Lazy 边不参与拓扑排序
- 实例化时，Lazy 边在首次访问时才触发另一侧实例化
- 依赖图变为 DAG，拓扑排序可执行

## 调试循环依赖

### NestJS Devtools

NestJS Devtools 提供 UI 可视化模块和 Provider 依赖关系：
- 显示模块间 imports/exports
- 显示 Provider 依赖关系
- 识别循环依赖路径

安装：`npm install @nestjs/devtools`

### NestJS Spelunker

第三方工具 nestjs-spelunker 输出依赖树文本：

```typescript
import { Spelunker } from 'nestjs-spelunker';

const spelunker = new Spelunker(app);
spelunker.modules.forEach(module => {
  console.log(module.name);
  module.providers.forEach(provider => {
    console.log(`  - ${provider.name}`);
  });
});
```

### 错误排查

循环依赖错误信息：
- `Nest can't resolve dependencies of the ServiceA (?)`
- `Circular dependency detected between ServiceA and ServiceB`

排查步骤：
1. 检查错误信息中涉及的 Service
2. 使用 Devtools 或 Spelunker 绘制依赖图
3. 确定循环路径
4. 选择 forwardRef、ModuleRef 或重构方案

## 参考资料

1. NestJS 官方文档 - Custom providers | Dependency Injection: https://docs.nestjs.com/fundamentals/dependency-injection
2. NestJS 官方文档 - Circular dependency: https://docs.nestjs.com/fundamentals/circular-dependency
3. NestJS 官方文档 - Module reference: https://docs.nestjs.com/fundamentals/module-reference
4. DeepWiki - NestJS Dependency Injection System: https://deepwiki.com/nestjs/nest/3.1-dependency-injection-system
