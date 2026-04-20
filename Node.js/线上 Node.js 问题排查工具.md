---
created: '2026-04-18'
updated: '2026-04-18'
tags:
  - nodejs
  - debugging
  - profiling
  - diagnostics
  - production
  - troubleshooting
aliases:
  - Node.js 生产环境排查工具
  - Node.js Performance Debugging Tools
  - Node.js Diagnostics
source_type: mixed
source_urls:
  - 'https://nodejs.org/en/learn/getting-started/debugging'
  - 'https://nodejs.org/en/learn/diagnostics/memory'
  - 'https://nodejs.org/en/learn/diagnostics/poor-performance'
  - 'https://nodejs.org/en/learn/diagnostics/flame-graphs'
  - 'https://nodejs.org/api/cli.html'
  - 'https://nodejs.org/api/process.html#processreport'
  - 'https://nodejs.org/api/report.html'
  - 'https://github.com/bnoordhuis/node-heapdump'
  - 'https://github.com/clinicjs/node-clinic'
  - 'https://github.com/davidmarkclements/0x'
status: verified
---

---
created: 2026-04-18
updated: 2026-04-18
tags:
  - nodejs
  - debugging
  - profiling
  - diagnostics
  - production
  - troubleshooting
aliases:
  - Node.js 生产环境排查工具
  - Node.js Performance Debugging Tools
  - Node.js Diagnostics
source_type: mixed
source_urls:
  - 'https://nodejs.org/en/learn/getting-started/debugging'
  - 'https://nodejs.org/en/learn/diagnostics/memory'
  - 'https://nodejs.org/en/learn/diagnostics/poor-performance'
  - 'https://nodejs.org/en/learn/diagnostics/flame-graphs'
  - 'https://nodejs.org/api/cli.html'
  - 'https://nodejs.org/api/process.html#processreport'
  - 'https://nodejs.org/api/report.html'
  - 'https://github.com/bnoordhuis/node-heapdump'
  - 'https://github.com/clinicjs/node-clinic'
  - 'https://github.com/davidmarkclements/0x'
status: verified
---

## 概述

线上 Node.js 问题排查是指在生产或类生产环境中，对运行中的 Node.js 进程进行诊断、分析和定位问题的过程。由于线上环境通常无法直接附加交互式调试器，排查工具需要满足以下要求：

- **低侵入性**：不能显著影响业务性能或稳定性
- **可远程触发**：通过信号、HTTP 接口或预置探针获取诊断数据
- **事后分析能力**：支持生成快照/报告后离线分析

常见线上问题类型包括：内存泄漏/OOM、CPU 占用过高/响应延迟、事件循环阻塞、未捕获异常/未处理 Promise Rejection、异步资源泄漏等。

---

## 一、内置诊断工具

Node.js 自身提供了丰富的诊断能力，无需安装第三方模块即可使用。

### 1.1 Inspector 调试协议（`--inspect`）

启动时添加 `--inspect` 参数，Node.js 会在 `127.0.0.1:9229` 开启 Inspector 协议端口，支持 Chrome DevTools、VS Code 等客户端连接。

| 参数 | 行为 |
|------|------|
| `--inspect` | 开启 Inspector，监听 `127.0.0.1:9229` |
| `--inspect=[host:port]` | 自定义监听地址 |
| `--inspect-brk` | 开启 Inspector 并在用户代码第一行断点 |
| `--inspect-wait` | 开启 Inspector 并等待调试器附加后再执行（Node.js 较新版本） |

**线上使用方式**：通过 SSH 隧道将远程 9229 端口转发到本地，再用 Chrome DevTools 连接：

```bash
# 在远程服务器上（默认只绑定 localhost，安全）
node --inspect server.js

# 在本地机器上建立 SSH 隧道
ssh -L 9221:localhost:9229 user@remote.example.com

# 本地 Chrome 访问 chrome://inspect，连接 localhost:9221
```

> **安全警告**：永远不要将 Inspector 绑定到公网 IP（如 `0.0.0.0`）。Inspector 拥有对进程执行环境的完全访问权限，暴露公网等同于允许远程代码执行。

参考资料：[Debugging Node.js](https://nodejs.org/en/learn/getting-started/debugging)、[--inspect](https://nodejs.org/api/cli.html#inspecthostport)

### 1.2 诊断报告（Diagnostic Report）

Diagnostic Report 是 Node.js 内置的诊断数据导出功能，可生成包含 JavaScript/Native 堆栈、堆信息、事件循环状态、资源使用情况等的 JSON 报告。

**触发方式**：

| 方式 | 说明 |
|------|------|
| `process.report.writeReport()` | 代码中主动调用，可选传入文件名和 Error 对象 |
| `process.report.getReport()` | 获取报告对象但不写入文件 |
| `kill -SIGUSR2 <pid>` | 发送信号触发（需配置 `reportOnSignal`） |
| 崩溃时自动生成 | 配置 `reportOnFatalError` 或 `reportOnUncaughtException` |

**常用配置**（通过 `process.report` 对象或启动参数）：

```js
// 代码中配置
process.report.reportOnUncaughtException = true;
process.report.reportOnSignal = true;
process.report.signal = 'SIGUSR2';  // 默认即为 SIGUSR2
process.report.directory = '/var/log/node-reports';
process.report.filename = 'report-%Y%m%d-%H%M%S-%p.ljson';
process.report.compact = true;       // 紧凑格式（单行 JSON）
process.report.excludeEnv = true;    // 排除环境变量（避免泄露敏感信息）
```

```bash
# 启动参数方式
node --report-on-uncaught-exception \
     --report-on-signal \
     --report-signal=SIGUSR2 \
     --report-directory=/var/log/node-reports \
     server.js
```

**报告内容包含**：header（进程信息、触发原因）、javascriptStack、nativeStack、javascriptHeap、libuv（handles 和 requests）、environmentVariables、resourceUsage 等。

参考资料：[process.report](https://nodejs.org/api/process.html#processreport)、[Report API](https://nodejs.org/api/report.html)

### 1.3 信号触发的 Heap Snapshot（`--heapsnapshot-signal`）

Node.js 12+ 支持通过信号触发 V8 Heap Snapshot，无需引入第三方模块。

```bash
# 启动时指定信号（常用 SIGUSR2）
node --heapsnapshot-signal=SIGUSR2 server.js

# 触发快照
kill -SIGUSR2 <pid>
```

快照文件默认生成在当前工作目录，命名格式为 `Heap.<timestamp>.<pid>.heapsnapshot`。可用 Chrome DevTools Memory 面板加载分析。

相关参数：
- `--heapsnapshot-near-heap-limit=max_count`：当堆内存接近 limit 时自动拍摄快照，最多拍摄 `max_count` 次（Node.js 16.14+）
- `--heap-snapshot-on-oom`：OOM 时自动生成快照（V8 参数，通过 `--v8-options` 或 `NODE_OPTIONS` 传递）

参考资料：[--heapsnapshot-signal](https://nodejs.org/api/cli.html#heapsnapshot-signalsignal)、[--heapsnapshot-near-heap-limit](https://nodejs.org/api/cli.html#heapsnapshot-near-heap-limitmax_count)

### 1.4 CPU Profiling（`--cpu-prof`）

Node.js 16+ 内置 V8 Sampling Profiler，可通过启动参数直接开启 CPU 分析。

```bash
node --cpu-prof server.js
```

进程退出时会自动生成 `.cpuprofile` 文件。可通过以下参数自定义行为：

| 参数 | 说明 |
|------|------|
| `--cpu-prof` | 开启 CPU profiling |
| `--cpu-prof-dir` | 输出目录（默认当前目录） |
| `--cpu-prof-name` | 文件名 |
| `--cpu-prof-interval` | 采样间隔（微秒，默认 1000） |

`.cpuprofile` 文件可直接在 Chrome DevTools Performance 面板加载。

参考资料：[--cpu-prof](https://nodejs.org/api/cli.html#cpu-prof)

### 1.5 Heap Profiling（`--heap-prof`）

与 CPU profiling 类似，`--heap-prof` 开启 V8 Heap Profiler，记录堆内存分配信息。

```bash
node --heap-prof server.js
```

| 参数 | 说明 |
|------|------|
| `--heap-prof` | 开启 Heap profiling |
| `--heap-prof-dir` | 输出目录 |
| `--heap-prof-name` | 文件名 |
| `--heap-prof-interval` | 采样间隔（字节，默认 512 * 1024） |

生成的 `.heapprofile` 文件可在 Chrome DevTools Memory 面板中分析。

参考资料：[--heap-prof](https://nodejs.org/api/cli.html#heap-prof)

### 1.6 Trace Events（`--trace-events-enabled`）

Trace Events 提供细粒度的事件追踪能力，可用于分析事件循环、异步操作、GC 等。

```bash
node --trace-events-enabled \
     --trace-event-categories v8,node,node.async_hooks server.js
```

常用 categories：
- `v8`：V8 引擎相关事件
- `node`：Node.js 核心事件
- `node.async_hooks`：异步资源生命周期
- `node.perf`：性能钩子

生成的 trace 文件可通过 [Chrome Tracing](chrome://tracing) 可视化。

参考资料：[Trace Events API](https://nodejs.org/api/tracing.html)、[--trace-events-enabled](https://nodejs.org/api/cli.html#trace-events-enabled)

### 1.7 Diagnostics Channel

`node:diagnostics_channel` 模块（Node.js 15.1+ / 14.17+）提供发布/订阅式的诊断消息通道，常用于框架和库之间的可观测性集成。

```js
import diagnostics_channel from 'node:diagnostics_channel';

const ch = diagnostics_channel.channel('myapp:request');
ch.subscribe((message) => {
  console.log('Request:', message);
});
```

参考资料：[Diagnostics Channel API](https://nodejs.org/api/diagnostics_channel.html)

### 1.8 其他有用的启动参数

| 参数 | 用途 |
|------|------|
| `--trace-sync-io` | 每次同步 I/O 操作时输出堆栈警告，用于排查阻塞事件循环的代码 |
| `--trace-exit` | 进程退出时输出调用堆栈，排查非预期退出 |
| `--trace-warnings` | 输出警告时包含完整堆栈 |
| `--trace-deprecation` | 输出弃用警告时包含完整堆栈 |
| `--abort-on-uncaught-exception` | 未捕获异常时 abort（生成 core dump），用于 post-mortem 分析 |
| `--unhandled-rejections=strict` | 未处理 Promise rejection 时抛出未捕获异常（Node.js 15+ 默认行为） |
| `--enable-source-maps` | 启用 source map 支持，使堆栈跟踪指向原始源码 |
| `NODE_DEBUG=module[,…]` | 开启指定模块的调试输出（如 `NODE_DEBUG=http,net`） |
| `NODE_DEBUG_NATIVE=module[,…]` | 开启原生模块的调试输出 |

参考资料：[Command-line API](https://nodejs.org/api/cli.html)、[Environment Variables](https://nodejs.org/api/cli.html#environment-variables-1)

---

## 二、内存问题排查

### 2.1 症状识别

- 内存使用量持续增长（可能跨越数小时到数天），最终进程崩溃或被进程管理器重启
- 重启导致部分请求失败（负载均衡器返回 502）
- GC 活动频繁，CPU 占用升高，响应时间变长
- 内存交换（swapping）导致性能进一步下降

### 2.2 `process.memoryUsage()`

最基础的内存监控 API，返回以下字段：

| 字段 | 含义 |
|------|------|
| `rss` | Resident Set Size，进程在内存中的总大小（含堆、栈、代码段等） |
| `heapTotal` | V8 堆的总大小（字节） |
| `heapUsed` | V8 堆已使用大小（字节） |
| `external` | C++ 对象绑定的内存（如 Buffer） |
| `arrayBuffers` | ArrayBuffer 和 SharedArrayBuffer 占用 |

```js
setInterval(() => {
  const mem = process.memoryUsage();
  console.log(`RSS: ${Math.round(mem.rss / 1024 / 1024)}MB, ` +
              `Heap: ${Math.round(mem.heapUsed / 1024 / 1024)}MB / ` +
              `${Math.round(mem.heapTotal / 1024 / 1024)}MB`);
}, 10000);
```

参考资料：[process.memoryUsage()](https://nodejs.org/api/process.html#processmemoryusage)

### 2.3 Heap Snapshot 分析

Heap Snapshot 是 V8 堆的完整快照，包含所有 JS 对象及其引用关系。分析步骤：

1. **获取快照**：
   - 内置：`--heapsnapshot-signal=SIGUSR2` + `kill -USR2 <pid>`
   - 第三方：`heapdump` 模块（支持 `SIGUSR2` 信号和 API 调用）
   - Inspector：通过 Chrome DevTools Memory 面板拍摄

2. **加载分析**：Chrome DevTools → Memory → Load → 选择 `.heapsnapshot` 文件

3. **对比分析**：拍摄多个时间点的快照，对比 Retainers 树，找出持续增长的对象类型及其引用链

> **注意**：在 UNIX 系统上，生成 Heap Snapshot 需要约堆大小两倍的可用内存。如果堆很大，可能触发 OOM Killer 导致快照文件为空或不完整。检查 `dmesg` 确认是否被 OOM Killer 终止。

参考资料：[Using Heap Snapshot](https://nodejs.org/en/learn/diagnostics/memory/using-heap-snapshot/)、[node-heapdump](https://github.com/bnoordhuis/node-heapdump)

### 2.4 Heap Profiler

与 Heap Snapshot 不同，Heap Profiler 持续记录堆分配信息而非拍摄全量快照，开销更低，适合长时间运行。

- 内置：`--heap-prof` 启动参数
- 生成 `.heapprofile` 文件，可在 Chrome DevTools 中分析

### 2.5 GC Traces

通过 V8 参数开启 GC 追踪输出，观察 GC 频率和耗时：

```bash
node --trace-gc server.js
```

输出示例：
```
[12345:0x12345678]    12345 ms: Scavenge 100.5 (135.2) -> 80.3 (135.2) MB, 2.1 / 0.0 ms  (average mu = 0.123, current mu = 0.056) allocation failure
```

可结合 `--trace-gc-ignore-skipped-tasks` 减少输出量。

参考资料：[GC Traces](https://nodejs.org/en/learn/diagnostics/memory/using-gc-traces/)

---

## 三、CPU / 性能问题排查

### 3.1 症状识别

- 应用延迟升高，已排除数据库和下游服务瓶颈
- CPU 占用持续偏高
- 希望了解代码中哪些部分消耗最多 CPU 周期以进行优化

### 3.2 V8 Sampling Profiler

Node.js 内置基于采样的 CPU Profiler：

```bash
# 方式一：--cpu-prof 参数（Node.js 16+）
node --cpu-prof server.js

# 方式二：--prof 参数（生成 v8.log，需后续处理）
node --prof server.js
node --prof-process isolate-*.log
```

`--prof-process` 会输出文本格式的性能报告，显示各函数的 tick 分布。

参考资料：[Profiling Node.js Applications](https://nodejs.org/en/learn/getting-started/profiling)、[--prof](https://nodejs.org/api/cli.html#prof)

### 3.3 Flame Graph（火焰图）

火焰图是可视化 CPU 时间分布的有效方式。

**方式一：使用 0x 工具**（推荐，一键生成）

```bash
npx 0x server.js
# 或用 --on-port 自动触发 benchmark
npx 0x --on-port 'autocannon localhost:$PORT' -- node server.js
```

生成结果包含交互式火焰图 HTML 文件。

参考资料：[0x](https://github.com/davidmarkclements/0x)

**方式二：使用 Linux perf + FlameGraph**

```bash
# 1. 安装 perf（通常在 linux-tools-common 包中）
# 2. 运行 Node.js 并开启 perf 支持
perf record -e cycles:u -g -- node --perf-basic-prof --interpreted-frames-native-stack app.js

# 3. 导出 perf 数据
perf script > perfs.out

# 4. 可选：过滤掉 Node.js 内部函数
sed -i -r \
  -e "/( __libc_start| LazyCompile | v8::internal::| Builtin:| Stub:| LoadIC:|\[unknown\]| LoadPolymorphicIC:)/d" \
  -e 's/ LazyCompile:[*~]?/ /' \
  perfs.out

# 5. 生成火焰图
# 克隆 FlameGraph 工具：https://github.com/brendangregg/FlameGraph
cat perfs.out | ./FlameGraph/stackcollapse-perf.pl | ./FlameGraph/flamegraph.pl --colors=js > profile.svg
```

也可以将 `perfs.out` 上传到 [flamegraph.com](https://flamegraph.com) 在线可视化。

**对运行中的进程采样**：

```bash
# 对已有 Node.js 进程采样 3 秒
perf record -F99 -p $(pgrep -n node) -g -- sleep 3
```

> **Node.js 版本注意**：Node.js 8+ 引入 TurboFan 编译管线，可能导致 perf 无法正确获取函数名（显示为 `ByteCodeHandler:`）。Node.js 10+ 通过 `--interpreted-frames-native-stack` 参数解决此问题。

参考资料：[Flame Graphs](https://nodejs.org/en/learn/diagnostics/flame-graphs)、[FlameGraph](https://github.com/brendangregg/FlameGraph)

### 3.4 Clinic.js

Clinic.js 是一套 Node.js 性能诊断工具套件，包含：

| 工具 | 用途 |
|------|------|
| `clinic doctor` | 自动检测性能问题类型并给出建议（CPU、内存、事件循环、I/O 等） |
| `clinic flame` | 生成火焰图 |
| `clinic heapprofiler` | 堆内存分析 |
| `clinic bubbleprof` | 异步 I/O 延迟分析（可视化异步调用链中的耗时） |

```bash
npm install -g clinic

# Doctor 模式
clinic doctor -- node server.js
# 然后用 wrk/autocannon 压测，最后 Ctrl+C 停止，自动生成分析报告

# 火焰图
clinic flame -- node server.js

# 堆分析
clinic heapprofiler -- node server.js

# 异步延迟分析
clinic bubbleprof -- node server.js
```

> **注意**：Clinic.js 目前未积极维护，依赖 Node.js 内部实现细节，在新版本上可能不工作或结果不准确。支持 Node.js >= 16。

参考资料：[Clinic.js](https://github.com/clinicjs/node-clinic)、[Clinic.js 官网](https://clinicjs.org/)

---

## 四、异步 / 事件循环问题排查

### 4.1 事件循环延迟检测

Node.js 单线程模型意味着同步阻塞操作会延迟整个事件循环，影响所有后续请求。

**使用 `perf_hooks` 监控事件循环延迟**：

```js
import { monitorEventLoopDelay } from 'node:perf_hooks';

const eld = monitorEventLoopDelay({ resolution: 10 });
eld.enable();

setInterval(() => {
  console.log(`Event Loop Delay: min=${eld.min}ns, max=${eld.max}ns, mean=${eld.mean}ns, p99=${eld.percentile(99)}ns`);
  eld.reset();
}, 5000);
```

参考资料：[Performance Hooks](https://nodejs.org/api/perf_hooks.html)

### 4.2 Async Hooks

`node:async_hooks` 模块可追踪异步资源的生命周期（init、before、after、destroy、promiseResolve）。

```js
import async_hooks from 'node:async_hooks';

const hook = async_hooks.createHook({
  init(asyncId, type, triggerAsyncId) {
    console.log(`[${asyncId}] ${type} created, triggered by ${triggerAsyncId}`);
  },
  destroy(asyncId) {
    console.log(`[${asyncId}] destroyed`);
  }
});
hook.enable();
```

> **注意**：Async Hooks 有明显性能开销，不建议在生产环境长期开启，仅用于排查阶段。

参考资料：[Async Hooks API](https://nodejs.org/api/async_hooks.html)

### 4.3 `--trace-sync-io`

检测同步 I/O 调用（会阻塞事件循环）：

```bash
node --trace-sync-io server.js
```

每次同步 I/O 操作都会输出警告及堆栈，帮助定位阻塞代码。

---

## 五、异常与崩溃排查

### 5.1 未捕获异常（`uncaughtException`）

```js
process.on('uncaughtException', (err, origin) => {
  // 仅做同步清理（关闭文件描述符、连接等）
  console.error(`Uncaught exception: ${err.message}\nOrigin: ${origin}`);
  // 不要试图在此之后恢复正常操作
  process.exit(1);
});
```

> **重要**：`uncaughtException` 是最后的防线，不是 `On Error Resume Next`。异常发生后应用处于未定义状态，正确的做法是同步清理后退出，由外部进程管理器（如 systemd、pm2、Kubernetes）重启。

参考资料：[Event: 'uncaughtException'](https://nodejs.org/api/process.html#event-uncaughtexception)

### 5.2 未处理 Promise Rejection（`unhandledRejection`）

```js
process.on('unhandledRejection', (reason, promise) => {
  console.error('Unhandled Rejection at:', promise, 'reason:', reason);
});
```

Node.js 15+ 默认行为为 `--unhandled-rejections=strict`，未处理的 rejection 会作为未捕获异常抛出并导致进程退出。

参考资料：[Event: 'unhandledRejection'](https://nodejs.org/api/process.html#event-unhandledrejection)、[--unhandled-rejections](https://nodejs.org/api/cli.html#--unhandled-rejectionsmode)

### 5.3 使用 `--abort-on-uncaught-exception` 生成 Core Dump

```bash
node --abort-on-uncaught-exception server.js
```

崩溃时生成 core dump 文件，可用 `lldb`、`gdb`、`mdb` 进行 post-mortem 分析：

```bash
lldb -c core.node.* node
# 或
gdb node core
```

### 5.4 诊断报告自动触发

配置 `reportOnUncaughtException` 和 `reportOnFatalError`，在异常/崩溃时自动生成诊断报告：

```bash
node --report-on-uncaught-exception --report-on-fatalerror server.js
```

---

## 六、线上排查最佳实践

### 6.1 安全注意事项

| 原则 | 说明 |
|------|------|
| Inspector 不暴露公网 | `--inspect` 默认绑定 `127.0.0.1`，不要改为 `0.0.0.0` |
| 使用 SSH 隧道远程调试 | `ssh -L 9221:localhost:9229 user@host` |
| 诊断报告排除敏感信息 | 设置 `report.excludeEnv = true`，避免环境变量泄露密钥 |
| 诊断数据及时清理 | Heap Snapshot 和诊断报告可能包含敏感业务数据 |

### 6.2 性能影响参考

| 操作 | 性能影响 |
|------|----------|
| `--inspect`（无客户端连接） | 极低 |
| `--cpu-prof` | 低（采样式） |
| `--heap-prof` | 低~中 |
| Heap Snapshot | 高（全量快照期间会暂停 JS 执行） |
| `--trace-sync-io` | 低 |
| Async Hooks | 中~高（取决于回调数量） |
| `--trace-events-enabled` | 低~中（取决于 categories 数量） |

### 6.3 排查流程建议

1. **先确认问题类型**：内存持续增长 → 内存问题；CPU 持续 100% → CPU 问题；响应延迟但 CPU 不高 → 事件循环或 I/O 问题
2. **收集基础指标**：`process.memoryUsage()`、`process.cpuUsage()`、事件循环延迟
3. **生成诊断数据**：Heap Snapshot、CPU Profile、Diagnostic Report
4. **离线分析**：在本地用 Chrome DevTools 或专用工具分析
5. **验证修复**：在测试环境复现并验证修复效果

### 6.4 常用环境变量

| 变量 | 用途 |
|------|------|
| `NODE_OPTIONS` | 预设启动参数，如 `NODE_OPTIONS="--inspect --cpu-prof"` |
| `NODE_DEBUG=http,net` | 开启 http/net 模块调试输出 |
| `NODE_DEBUG_NATIVE=uv` | 开启 libuv 调试输出 |
| `NODE_HEAPDUMP_OPTIONS=nosignal` | 禁用 heapdump 的 SIGUSR2 信号处理 |

---

## 七、工具速查表

| 问题类型 | 推荐工具 | 是否可线上使用 |
|----------|----------|:--------------:|
| 内存泄漏 | `--heapsnapshot-signal` + Chrome DevTools | ✅ |
| 内存泄漏 | `heapdump` 模块 | ✅ |
| 内存泄漏 | `--heap-prof` | ✅ |
| CPU 瓶颈 | `--cpu-prof` + Chrome DevTools | ✅ |
| CPU 瓶颈 | `0x` / `clinic flame` | ✅ |
| CPU 瓶颈 | Linux `perf` + FlameGraph | ✅（仅 Linux） |
| 异步延迟 | `clinic bubbleprof` | ✅ |
| 异步资源追踪 | `async_hooks` | ⚠️ 有性能开销 |
| 事件循环延迟 | `perf_hooks.monitorEventLoopDelay()` | ✅ |
| 崩溃分析 | `--report-on-fatalerror` | ✅ |
| 崩溃分析 | `--abort-on-uncaught-exception` + core dump | ✅ |
| 综合诊断 | `clinic doctor` | ✅ |
| 同步 I/O 阻塞 | `--trace-sync-io` | ✅ |
| 未捕获异常 | `process.report.writeReport()` | ✅ |

---

## 八、相关概念

- [[调试 V8 执行]]：V8 引擎级别的调试方法
- [[编译独立可调试 V8]]：编译可调试的 V8 构建
- [[React 性能优化思路与常用工具]]：前端性能排查工具
- [[全链路tracing方案(skywalking+kafka+clickhouse+grafana)]]：分布式追踪方案
