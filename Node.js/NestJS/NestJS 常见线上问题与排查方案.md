---
created: 2026-04-22
updated: 2026-04-22
tags:
  - nestjs
  - node.js
  - production
  - troubleshooting
  - debugging
  - error-handling
aliases:
  - NestJS 线上问题
  - NestJS 排查方案
  - NestJS Production Issues
  - NestJS Troubleshooting
source_type: mixed
source_urls:
  - https://docs.nestjs.com/techniques/exception-filters
  - https://docs.nestjs.com/fundamentals/lifecycle-events
  - https://docs.nestjs.com/fundamentals/error-handling
  - https://github.com/nestjs/nest
status: verified
---

## 概述

NestJS 作为基于 Node.js 的企业级框架，线上问题既包含 Node.js 层面的通用问题（内存泄漏、事件循环阻塞），也包含框架层面的特有问题（依赖注入失败、模块配置错误、生命周期钩子异常）。本文按问题类型分类整理常见线上问题及排查方案。

## 一、依赖注入相关问题

### 1.1 依赖注入失败：`Nest can't resolve dependencies of X`

**现象**：应用启动时抛出类似错误：

```
Nest can't resolve dependencies of the UserService (?). Please make sure that the argument UserRepository at index [0] is available in the AppModule context.
```

**常见原因**：

- 目标依赖所在的模块未在当前模块的 `imports` 中声明
- 依赖的 Provider 未在当前模块或导入模块的 `providers` 中注册
- 使用了 `@Inject()` 自定义 token 但 token 不匹配

**排查步骤**：

1. 检查报错信息中指明的缺失依赖名称和索引位置
2. 确认该依赖所在模块是否已 `export` 该 Provider
3. 确认当前模块是否已通过 `imports` 导入该模块
4. 如果使用了自定义 Injection Token（如 `@Inject('CUSTOM_TOKEN')`），确认 Token 字符串/符号完全一致

**最佳实践**：

- 使用 TypeScript 类型注入时，确保类型是类（class）而非接口（interface），因为接口在编译后不存在
- 跨模块共享 Provider 时，必须同时 `export` 和 `import`

### 1.2 循环依赖问题

**现象**：启动时抛出 `Nest cannot resolve dependencies` 或出现 `undefined` 注入。

**解决方案**：使用 `forwardRef()` 打破循环。

```typescript
import { forwardRef, Inject, Injectable } from '@nestjs/common';

@Injectable()
export class ServiceA {
  constructor(
    @Inject(forwardRef(() => ServiceB))
    private serviceB: ServiceB,
  ) {}
}
```

> 详细说明参见：[[NestJS 依赖注入机制与 forwardRef 解决循环依赖]]

### 1.3 请求作用域（Request-scoped）导致的性能问题

**现象**：高并发下应用响应变慢，CPU 使用率异常升高。

**原因**：请求作用域的 Provider（`scope: Scope.REQUEST`）会在每个请求到来时创建新实例，导致：

- 频繁的对象创建和垃圾回收
- 依赖链上的所有 Provider 也会被实例化为请求作用域

**排查方法**：

```typescript
// 检查是否有不必要的请求作用域声明
@Injectable({ scope: Scope.REQUEST }) // 仅在确实需要时使用
```

**建议**：

- 默认使用单例（Singleton）作用域
- 仅在需要隔离请求状态（如事务上下文、请求级缓存）时使用请求作用域
- 使用 `@nestjs/terminus` 的 `TerminusModule` 配合健康检查监控实例数量

## 二、异常处理相关问题

### 2.1 未捕获异常导致进程退出

**现象**：Node.js 进程因未处理的异常或 Promise rejection 意外退出。

**NestJS 内置保护**：

NestJS 默认通过 `ExceptionsHandler` 捕获路由处理器中的异常，但以下场景仍可能导致进程崩溃：

- 定时器回调中的异常（`setTimeout`、`setInterval`）
- 事件监听器中的异常（`EventEmitter`）
- 未 `await` 的 Promise rejection

**排查与防护**：

```typescript
// main.ts - 添加全局未捕获异常处理
async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  process.on('uncaughtException', (error) => {
    console.error('Uncaught Exception:', error);
    // 记录日志后优雅关闭
    app.close();
    process.exit(1);
  });

  process.on('unhandledRejection', (reason) => {
    console.error('Unhandled Rejection:', reason);
  });

  await app.listen(3000);
}
bootstrap();
```

### 2.2 异常过滤器（Exception Filter）未生效

**现象**：自定义 Exception Filter 未捕获预期异常，返回默认 500 响应。

**常见原因**：

- Filter 的作用域配置不正确（全局 vs 控制器级 vs 方法级）
- Filter 捕获的异常类型不匹配
- Filter 注册顺序导致被其他 Filter 覆盖

**排查步骤**：

1. 确认 Filter 是否正确继承 `BaseExceptionFilter` 或实现 `ExceptionFilter` 接口
2. 检查 `@Catch()` 装饰器指定的异常类型是否正确
3. 全局 Filter 注册方式：

```typescript
// main.ts
app.useGlobalFilters(new HttpExceptionFilter());
```

4. 控制器级注册：

```typescript
@UseFilters(HttpExceptionFilter)
@Controller('users')
export class UsersController {}
```

**注意事项**：

- `useGlobalFilters()` 注册的 Filter 不会作用于微服务或 WebSocket gateway
- 微服务场景需在 `@MessagePattern` 或 `@EventPattern` 上单独注册 Filter

### 2.3 自定义异常未正确序列化

**现象**：返回给客户端的异常响应缺少预期字段。

**原因**：NestJS 默认只序列化 `HttpException` 的 `response` 和 `status` 属性。自定义异常需继承 `HttpException` 或实现 `ExceptionFilter` 自定义序列化逻辑。

```typescript
import { HttpException, HttpStatus } from '@nestjs/common';

export class CustomException extends HttpException {
  constructor(message: string, code: string) {
    super({ message, code, timestamp: new Date().toISOString() }, HttpStatus.BAD_REQUEST);
  }
}
```

## 三、生命周期相关问题

### 3.1 应用启动缓慢

**现象**：`npm run start:prod` 后应用需要数十秒才能响应请求。

**排查方向**：

| 可能原因 | 排查方法 |
|----------|----------|
| 数据库连接池初始化慢 | 在 `onModuleInit` 中记录耗时，检查连接配置 |
| 大量模块同步初始化 | 使用 `APP_INITIALIZER` 延迟非必要初始化 |
| 同步读取大配置文件 | 改为异步读取或缓存 |
| TypeORM/Prisma schema 同步 | 生产环境关闭 `synchronize: true` |

**诊断代码**：

```typescript
@Injectable()
export class StartupLogger implements OnModuleInit {
  async onModuleInit() {
    const start = Date.now();
    // 初始化逻辑
    console.log(`Module initialized in ${Date.now() - start}ms`);
  }
}
```

### 3.2 优雅关闭未生效

**现象**：`SIGTERM` 信号后进程立即退出，正在处理的请求被中断。

**正确配置**：

```typescript
// main.ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // 启用优雅关闭（NestJS 10+ 默认启用）
  app.enableShutdownHooks();

  await app.listen(3000);
}
```

```typescript
// 在 Service 中处理关闭信号
@Injectable()
export class TasksService implements OnApplicationShutdown {
  onApplicationShutdown(signal: string) {
    console.log(`Application shutting down with signal: ${signal}`);
    // 清理资源：关闭连接、完成进行中的任务等
  }
}
```

**注意事项**：

- Docker/K8s 发送 `SIGTERM` 后默认等待 30 秒（`terminationGracePeriodSeconds`），需确保清理逻辑在此时间内完成
- `enableShutdownHooks()` 会注册 `SIGINT`、`SIGTERM`、`SIGQUIT`、`SIGHUP` 信号监听
- Windows 环境下信号支持有限，开发时可能表现不一致

### 3.3 生命周期钩子执行顺序异常

**现象**：`onModuleInit` 中依赖的服务尚未初始化。

**NestJS 生命周期钩子执行顺序**：

1. `onModuleInit()` — 所有模块实例化后调用
2. `onApplicationBootstrap()` — 应用启动完成后调用
3. `onModuleDestroy()` — 应用关闭时调用
4. `beforeApplicationShutdown()` — 关闭前调用
5. `onApplicationShutdown()` — 关闭完成后调用

**排查建议**：

- 如果需要在所有模块初始化后执行逻辑，使用 `onApplicationBootstrap()` 而非 `onModuleInit()`
- 模块间的初始化顺序由依赖图决定，可通过模块 `imports` 顺序间接控制

## 四、性能与内存问题

### 4.1 内存泄漏

**常见场景**：

| 场景 | 原因 | 排查方法 |
|------|------|----------|
| 事件监听器未移除 | `EventEmitter.on()` 注册后未 `removeListener()` | 使用 `--inspect` 启动，Chrome DevTools 查看 Listener 数量 |
| 缓存无上限 | `Map`/`Object` 持续写入无淘汰策略 | 使用 `cache-manager` 设置 TTL 和 maxSize |
| 请求作用域滥用 | 每个请求创建新实例且未被 GC | 检查 `Scope.REQUEST` 使用范围 |
| 数据库连接未释放 | 连接池配置不当或事务未提交/回滚 | 监控连接池活跃连接数 |

**诊断命令**：

```bash
# 启动时开启内存诊断
node --inspect --max-old-space-size=4096 dist/main.js

# 或使用 Node.js 诊断 API
node --heapsnapshot-signal=SIGUSR2 dist/main.js
# 发送信号生成堆快照：kill -SIGUSR2 <pid>
```

### 4.2 事件循环阻塞

**现象**：请求响应时间忽高忽低，CPU 使用率高但吞吐量低。

**常见原因**：

- 同步执行 CPU 密集型操作（大 JSON 解析、复杂计算、同步正则表达式）
- 大型同步循环
- 使用 `fs.readFileSync` 等阻塞 API

**排查方法**：

```typescript
// 使用 clinic.js 诊断
npx clinic doctor -- node dist/main.js

// 或使用内置诊断
import { monitorEventLoopDelay } from 'perf_hooks';
const histogram = monitorEventLoopDelay({ resolution: 10 });
histogram.enable();
setInterval(() => {
  console.log(`p99 latency: ${histogram.percentile(99) / 1e6}ms`);
}, 5000);
```

**解决方案**：

- CPU 密集型操作使用 Worker Threads：`import { Worker } from 'worker_threads'`
- 大文件处理使用 Stream API
- 复杂计算考虑拆分为微服务或使用消息队列异步处理

### 4.3 数据库连接池耗尽

**现象**：请求超时，日志中出现 `Connection pool exhausted` 或 `timeout exceeded`。

**排查方向**：

- TypeORM 默认连接池大小为 100，检查 `maxConnections` 配置
- 确认是否有未关闭的事务或长查询占用连接
- 检查是否有连接泄漏（获取连接后未释放）

```typescript
// TypeORM 配置示例
TypeOrmModule.forRoot({
  // ...
  extra: {
    max: 50,          // 最大连接数
    min: 5,           // 最小空闲连接
    idleTimeoutMillis: 30000,
  },
}),
```

## 五、请求处理相关问题

### 5.1 请求超时

**现象**：部分请求超过客户端超时时间未返回。

**排查步骤**：

1. 确认是否由下游服务/数据库调用慢导致
2. 检查是否有 Interceptor 或 Guard 中的同步阻塞逻辑
3. 确认 NestJS 全局超时配置（Express 默认无超时）

**设置请求超时**：

```typescript
// main.ts - Express 适配器
import { NestFactory } from '@nestjs/core';
import { ExpressAdapter } from '@nestjs/platform-express';
import * as express from 'express';

const server = express();
server.timeout = 30000; // 30 秒超时

const app = await NestFactory.create(new ExpressAdapter(server));
```

```typescript
// 或使用 TimeoutInterceptor
@Injectable()
export class TimeoutInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(timeout(30000));
  }
}
```

### 5.2 请求体解析失败

**现象**：POST 请求返回 400 Bad Request，但请求体格式正确。

**常见原因**：

- 请求体大小超过默认限制（Express 默认 100kb）
- `Content-Type` 与解析器不匹配
- 嵌套对象深度超过限制

**解决方案**：

```typescript
// main.ts
app.use(bodyParser.json({ limit: '10mb' }));
app.use(bodyParser.urlencoded({ limit: '10mb', extended: true }));
```

### 5.3 中间件执行顺序问题

**现象**：某些中间件未按预期执行或执行顺序错误。

**NestJS 请求处理管道顺序**：

```
Incoming Request
  → Middleware（按注册顺序）
    → Guards（全局 → 控制器 → 方法）
      → Interceptors（Before 阶段，全局 → 控制器 → 方法）
        → Pipes（全局 → 参数 → 方法）
          → Controller Handler
        → Interceptors（After 阶段，逆序）
      → Guards（逆序）
    → Middleware（逆序）
  → Response
```

**排查建议**：

- 使用 `console.log` 或 Logger 在各层标记执行顺序
- 全局中间件通过 `app.use()` 注册，在 NestJS 中间件之前执行
- NestJS 中间件通过 `configure(consumer)` 注册，按 `apply().forRoutes()` 顺序执行

## 六、微服务与消息队列问题

### 6.1 微服务连接断开

**现象**：微服务间通信间歇性失败。

**排查方向**：

- TCP/Redis/RMQ 传输层的连接配置
- 心跳/保活机制是否启用
- 网络策略（防火墙、安全组）是否允许端口通信

```typescript
// TCP 微服务配置
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.TCP,
  options: {
    host: '0.0.0.0',
    port: 3001,
    retryAttempts: 5,
    retryDelay: 1000,
  },
});
```

### 6.2 消息重复消费

**现象**：同一条消息被多次处理。

**常见原因**：

- 消费者未正确发送 ACK（RabbitMQ）
- 消费者处理超时导致消息重新入队
- 多个消费者实例订阅同一队列且未正确分配

**排查建议**：

- 确认消息处理逻辑的幂等性
- 检查 ACK/NACK 时机，确保处理完成后才 ACK
- 使用 Redis 或数据库记录已处理消息 ID 去重

## 七、日志与可观测性

### 7.1 日志格式不统一

**问题**：不同模块使用不同日志格式，难以集中分析。

**推荐方案**：使用 `@nestjs/platform-express` 配合 `pino` 或 `winston`。

```typescript
// main.ts - 使用 pino
import { Logger } from 'nestjs-pino';

async function bootstrap() {
  const app = await NestFactory.create(AppModule, {
    bufferLogs: true,
  });
  app.useLogger(app.get(Logger));
  await app.listen(3000);
}
```

### 7.2 缺少请求追踪

**问题**：分布式环境中无法关联同一请求的日志。

**解决方案**：使用 `cls-hooked` 或 `AsyncLocalStorage` 实现请求级上下文。

```typescript
// 使用 AsyncLocalStorage 的请求 ID 中间件
import { AsyncLocalStorage } from 'async_hooks';
import { Injectable, NestMiddleware } from '@nestjs/common';
import { v4 as uuidv4 } from 'uuid';

export const requestContext = new AsyncLocalStorage<Map<string, string>>();

@Injectable()
export class RequestContextMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: Function) {
    const store = new Map();
    store.set('requestId', req.headers['x-request-id'] || uuidv4());
    requestContext.run(store, next);
  }
}
```

## 八、热重载与开发环境问题

### 8.1 热重载不生效

**现象**：修改代码后服务未自动重启。

**排查方向**：

- 确认使用 `nest start --watch` 而非 `node dist/main.js`
- 检查 `tsconfig.json` 中 `outDir` 与 `nest-cli.json` 配置一致
- Webpack 模式下检查 `webpack-hmr` 配置

```typescript
// main.ts - HMR 配置
declare const module: any;

if (module.hot) {
  module.hot.accept();
  module.hot.dispose(() => app.close());
}
```

### 8.2 生产构建体积过大

**排查方向**：

- 使用 `npm run build` 而非 `tsc` 直接编译（nest-cli 会优化）
- 检查 `tsconfig.build.json` 中 `exclude` 配置，排除测试文件
- 使用 `webpack` 打包并启用 Tree Shaking

## 排查工具汇总

| 工具 | 用途 | 使用方式 |
|------|------|----------|
| `--inspect` | Node.js 调试器 | `node --inspect dist/main.js` |
| `--heapsnapshot-signal` | 堆快照 | `node --heapsnapshot-signal=SIGUSR2 dist/main.js` |
| `clinic.js` | 性能诊断 | `npx clinic doctor -- node dist/main.js` |
| `0x` | CPU Flame Graph | `npx 0x dist/main.js` |
| `--trace-warnings` | 追踪警告来源 | `node --trace-warnings dist/main.js` |
| `--trace-uncaught` | 追踪未捕获异常 | `node --trace-uncaught dist/main.js` |
| NestJS Devtools | 模块依赖可视化 | `@nestjs/devtools-integration` |

## 参考资料

- NestJS 官方文档 — Exception Filters: https://docs.nestjs.com/techniques/exception-filters
- NestJS 官方文档 — Lifecycle Events: https://docs.nestjs.com/fundamentals/lifecycle-events
- NestJS 官方文档 — Error Handling: https://docs.nestjs.com/fundamentals/error-handling
- NestJS GitHub 仓库: https://github.com/nestjs/nest
- Node.js 诊断文档: https://nodejs.org/api/diagnostics.html
