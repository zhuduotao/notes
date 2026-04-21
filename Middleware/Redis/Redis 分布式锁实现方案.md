---
created: '2026-04-21'
updated: '2026-04-21'
tags:
  - redis
  - distributed-lock
  - concurrency
  - distributed-system
  - redlock
aliases:
  - Redis 分布式锁
  - Redlock 算法
  - Redis 分布式锁实现
source_type: mixed
source_urls:
  - 'https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/'
  - 'https://redis.io/commands/set/'
  - 'https://antirez.com/news/77'
  - 'https://antirez.com/news/101'
  - 'http://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html'
status: verified
---
## 概述

基于 Redis 的分布式锁是利用 Redis 的原子操作特性实现跨进程、跨节点互斥访问的一种方案。Redis 官方文档提供了两种实现方式：单实例基础方案和 Redlock 多实例容错方案。

分布式锁的核心用途分为两类：
- **效率优化**：避免重复执行相同工作（如重复计算、重复发送通知），偶尔的锁失效只会造成少量额外开销。
- **正确性保证**：防止并发操作破坏系统状态（如数据损坏、不一致），锁失效会导致严重问题。

## 单实例实现方案

### 基本原理

使用 Redis 的 `SET` 命令配合 `NX` 和 `EX/PX` 参数实现原子加锁：

```redis
SET resource_name my_random_value NX PX 30000
```

- `NX`：仅当 key 不存在时才设置（实现互斥）
- `PX 30000`：设置 30 秒过期时间（防止死锁）
- `my_random_value`：客户端生成的唯一随机值（用于安全释放）

### 安全释放锁

不能直接使用 `DEL` 命令释放锁，因为客户端可能因 GC 停顿等原因超过锁有效期后误删其他客户端的锁。

**Redis 8.4+ 方式**：
```redis
DELEX key IFEQ my_random_value
```

**Redis 8.4 之前**（Lua 脚本）：
```lua
if redis.call("get",KEYS[1]) == ARGV[1] then
    return redis.call("del",KEYS[1])
else
    return 0
end
```

脚本确保只有锁的持有者才能释放锁。

### 随机值生成建议

- 推荐：从 `/dev/urandom` 读取 20 字节
- 可接受：UNIX 时间戳（微秒精度）+ 客户端 ID 组合

### 单实例方案的局限

单实例方案存在单点故障问题。如果使用主从复制架构进行故障转移，由于 Redis 复制是异步的，可能出现：

1. 客户端 A 在主节点获取锁
2. 主节点崩溃，锁尚未同步到副本
3. 剖本晋升为主节点
4. 客户端 B 获取同一资源的锁 → **互斥被破坏**

如果可以容忍故障时短暂的互斥失效，单实例或主从方案仍然可用。

## Redlock 多实例方案

### 设计目标

Redlock（Redis Distributed Lock）旨在提供更高容错能力，满足三个核心属性：

1. **安全性**：任意时刻只有一个客户端持有锁
2. **活性 A**：无死锁，即使持有锁的客户端崩溃或被分区，锁最终可被重新获取
3. **活性 B**：容错性，只要多数 Redis 节点存活，客户端就能获取和释放锁

### 算法流程

假设使用 N=5 个独立的 Redis 主节点（无复制关系）：

1. 获取当前时间戳（毫秒）
2. **并行**向所有 N 个节点尝试加锁，使用相同的 key 和随机值，每个节点的超时时间远小于锁有效期（如 5-50ms）
3. 计算加锁耗时：若成功获取**多数节点**（≥3）的锁，且耗时小于锁有效期，则视为成功
4. 锁的实际有效期 = 初始有效期 - 加锁耗时
5. 若获取失败（未达多数或耗时超限），向所有节点释放锁（包括未成功获取的节点）

### 系统模型假设

Redlock 依赖**半同步系统模型**：
- 不同进程的时钟以大致相同的速率运行（相对误差有界）
- 不要求时钟绝对同步，只需能以可接受的误差"计数"时长

Redis 官方建议使用单调时钟 API（`clock_gettime`）而非 `gettimeofday`，以避免系统时间跳变影响。

### 锁续期

如果客户端执行的操作需要更长时间，可在锁接近过期前续期：

向所有持有锁的节点发送 Lua 脚本，仅在值匹配时延长 TTL。续期同样需要获取多数节点成功才算有效。

### 崩溃恢复与持久化

节点崩溃重启可能破坏安全性：

- 若无持久化：重启后锁丢失，其他客户端可能获取同一锁
- 若启用 AOF 并设置 `fsync=always`：安全性有保证，但性能大幅下降
- **延迟重启**（推荐）：崩溃后延迟重启超过最大锁有效期，使所有可能存在的锁自然过期后再恢复服务

## 学术争议

### Martin Kleppmann 的批评

Martin Kleppmann 于 2016 年发表文章质疑 Redlock 的安全性，主要论点：

1. **缺少 Fencing Token**
   
   分布式锁仅保证时间窗口内的互斥。如果客户端因 GC 停顿或网络延迟超过有效期后仍操作共享资源，会与新锁持有者并发访问。
   
   解决方案是使用 **fencing token**（单调递增数字）：每次获取锁时返回递增的 token，共享资源在执行操作前检查 token，拒绝低于已记录 token 的请求。
   
   Redlock 使用随机值而非单调 token，无法提供此保护。

2. **时间假设不可靠**
   
   Redlock 依赖有界的网络延迟、进程停顿和时钟误差。但现实中：
   - GC 停顿可达数分钟
   - 网络延迟可达 90 秒（GitHub 案例）
   - 系统时钟可能跳变（NTP 调整或人工修改）
   
   这些情况下 Redlock 可能违反互斥保证。

### antirez 的回应

Redis 作者 antirez 对批评作出反驳：

1. **关于 Fencing Token**
   - 如果共享资源本身能解决并发冲突，则根本不需要强保证的分布式锁
   - Redlock 的随机值可用于 CAS（Check-and-Set）模式实现类似保护
   - 多数场景下锁保护的资源不支持事务性检查

2. **关于时间假设**
   - 网络延迟在锁获取流程中有时间检查（步骤 1 和 3），超时会判定锁无效
   - 应使用单调时钟 API 避免系统时间跳变
   - 正确配置 NTP（仅 slewing 而非 stepping）可避免大幅跳变

### 共识与建议

双方各有道理。实际建议：

- **仅用于效率优化**：单实例方案足够，复杂度低
- **用于正确性保证**：
  - 若可接受极小概率的互斥失效，Redlock 可用
  - 若要求严格正确性，应使用 ZooKeeper 等共识系统并配合 fencing token

## 使用建议

### 适用场景

| 场景 | 推荐方案 |
|------|----------|
| 防止重复任务执行（效率） | 单实例方案 |
| 高并发但偶发冲突可接受 | 单实例方案 |
| 需要高容错但允许极小风险 | Redlock |
| 严格要求互斥（如金融、医疗） | ZooKeeper + fencing token |

### 最佳实践

1. **锁有效期设置**：应大于预期操作耗时 + 网络延迟 + 安全裕度
2. **释放锁务必使用脚本**：禁止直接 `DEL`
3. **获取失败后立即释放**：避免等待过期才能重试
4. **使用单调时钟**：避免系统时间调整影响
5. **延迟重启节点**：崩溃后等待超过最大锁 TTL
6. **实现续期机制**：长操作应主动续期

### 常见误区

- 误用 `SETNX + EXPIRE`（非原子）：应使用 `SET NX EX/PX`
- 直接 `DEL` 释放锁：可能误删其他客户端的锁
- 未考虑 GC 停顿：锁有效期应足够长
- 在主从架构上直接使用：异步复制可能破坏互斥

## 相关概念

- **分布式锁 vs 本地锁**：本地锁（如 mutex）依赖共享内存，分布式锁依赖外部协调服务
- **Lease（租约）**：带过期时间的锁，防止死锁
- **Fencing Token**：单调递增的锁标识，用于资源端验证
- **ZooKeeper 锁**：使用 znode 版本号作为 fencing token，提供更强保证
- **CAS（Compare-and-Set）**：条件更新操作，可配合随机值实现类似 fencing 功能

## 客户端实现

主流客户端库已内置分布式锁实现：

| 语言 | 库 |
|------|-----|
| Java | Redisson |
| Go | Redsync |
| Python | redlock-ng, pottery |
| Node.js | node-redlock |
| Ruby | redlock-rb |
| C#/.NET | RedLock.net |

Redisson（Java）是功能最完善的实现，支持可重入锁、读写锁、公平锁等多种模式。

## 参考资料

- [Redis 官方文档：Distributed Locks with Redis](https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/)
- [Redis SET 命令文档](https://redis.io/commands/set/)
- [antirez: A proposal for more reliable locks using Redis](https://antirez.com/news/77)
- [Martin Kleppmann: How to do distributed locking](http://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)
- [antirez: Is Redlock safe?](https://antirez.com/news/101)
- [Gray & Cheriton: Leases: An Efficient Fault-Tolerant Mechanism for Distributed File Cache Consistency (SOSP 1989)](https://dl.acm.org/citation.cfm?id=74870)
