---
created: 2026-04-21
updated: 2026-04-21
tags:
  - database
  - redis
  - transaction
  - rollback
  - lua-scripting
  - distributed-system
aliases:
  - Redis 事务
  - Redis rollback
  - Redis MULTI EXEC
  - Redis CAS
source_type: official-doc
source_urls:
  - https://redis.io/docs/latest/develop/using-commands/transactions/
  - https://redis.io/docs/latest/develop/programmability/eval-intro/
status: verified
---

## 是什么

Redis 事务是通过 `MULTI`、`EXEC`、`DISCARD`、`WATCH` 四个命令实现的批量命令执行机制。其核心特性是**将一组命令打包、顺序执行、不被其他客户端打断**，但与传统数据库事务的关键差异在于：**Redis 事务不支持回滚**。

## 为什么重要

| 维度 | 说明 |
|------|------|
| **原子性隔离** | 事务执行期间，其他客户端的命令不会穿插在中间 |
| **简化并发** | 单线程模型下，事务天然保证执行顺序一致性 |
| **乐观锁支持** | `WATCH` 提供类似 CAS（Check-and-Set）的并发控制 |
| **性能优先** | 不支持回滚的设计简化了实现，保持了高性能 |

## Redis 事务机制

### 基本用法

```
MULTI          # 开启事务，进入命令队列模式
SET key1 val1  # 命令入队，返回 QUEUED
SET key2 val2  # 命令入队，返回 QUEUED
EXEC           # 执行所有入队命令，返回各命令结果数组
```

**关键命令**：

| 命令 | 作用 |
|------|------|
| `MULTI` | 标记事务开始，后续命令入队而非立即执行 |
| `EXEC` | 执行队列中所有命令，返回结果数组 |
| `DISCARD` | 放弃事务，清空命令队列 |
| `WATCH key [key...]` | 监视键，若执行前被修改则事务失败 |

### 事务的两个保证

根据 Redis 官方文档，事务提供两个核心保证：

1. **顺序执行与隔离**：事务内所有命令序列化执行，其他客户端请求不会在执行中途被处理
2. **原子执行或完全不执行**：
   - 若调用 `EXEC` 前 connection 断开，所有命令不执行
   - 若调用 `EXEC` 后，所有命令执行（即使部分失败）
   - 使用 AOF 时，Redis 保证用单次 `write(2)` syscall 写入事务

### WATCH 乐观锁

`WATCH` 实现 Check-and-Set（CAS）语义：

```
WATCH balance      # 监视键
val = GET balance
val = val - 100
MULTI
SET balance $val
EXEC               # 若 balance 被其他客户端修改，返回 nil，事务不执行
```

**工作机制**：
- 监视的键在 `WATCH` 到 `EXEC` 期间被修改（包括过期、驱逐），事务自动中止
- `EXEC` 返回 Null Reply 表示失败，需重试
- `EXEC` 或 `DISCARD` 后自动 `UNWATCH`
- Redis 6.0.9 之前，过期键不会触发事务中止（[GitHub PR #7920](https://github.com/redis/redis/pull/7920)）

### Redis 8.4+ 的原子 CAS 命令

Redis 8.4 引入了更简洁的原子 compare-and-set 命令，无需使用 `WATCH` + `MULTI`：

| 命令/选项 | 用途 |
|----------|------|
| `SET key value IFEQ prev_value` | 仅当当前值等于 `prev_value` 时才设置 |
| `SET key value IFNE prev_value` | 仅当当前值不等于 `prev_value` 时才设置 |
| `SET key value IFDEQ prev_value` | 仅当当前值等于时删除并设置 |
| `SET key value IFDNE prev_value` | 仅当当前值不等于时删除并设置 |
| `DELEX key` | 仅当值未变化时删除 |

这些命令比 `WATCH` 模式更高效、更简洁。

## Redis 不支持回滚的原因

### 官方说明

Redis 官方文档明确指出：

> "Redis does not support rollbacks of transactions since supporting rollbacks would have a significant impact on the simplicity and performance of Redis."

### 设计哲学

| 原因 | 说明 |
|------|------|
| **简化实现** | 回滚机制需要记录事务前状态，增加复杂度 |
| **保持性能** | 避免回滚日志的内存和 CPU 开销 |
| **错误类型区分** | Redis 认为命令错误应是编程问题，而非运行时异常 |
| **单线程优势** | 无需复杂的锁和并发控制机制 |

### 错误类型与行为

| 错误类型 | 发生时机 | 行为 |
|----------|----------|------|
| **入队错误** | `EXEC` 前（语法错误、内存不足） | Redis 2.6.5+ 拒绝执行整个事务 |
| **执行错误** | `EXEC` 后（类型错误如对 String 执行 `LPOP`） | **该命令失败，其他命令继续执行** |

**执行错误示例**：

```
MULTI
+OK
SET a abc
+QUEUED
LPOP a        # 类型错误，但入队成功
+QUEUED
EXEC
*2
+OK
-WRONGTYPE Operation against a key holding the wrong kind of value
```

**关键点**：即使某命令执行失败，队列中其他命令仍会执行——这是 Redis 不回滚的直接体现。

## 如何处理 Redis 事务中的"回滚"需求

Redis 事务本身不支持回滚，但可通过以下策略实现类似效果：

### 1. 使用 Lua 脚本替代事务

Lua 脚本在 Redis 中是**原子执行的**，执行期间服务器阻塞，所有操作要么全部完成要么全部不完成。

**优势**：
- 可以读取中间结果并决定后续操作
- 可以实现条件判断和逻辑控制
- 单次网络请求，减少 RTT

**示例**：原子转账

```lua
local from_balance = tonumber(redis.call('GET', KEYS[1]))
local to_balance = tonumber(redis.call('GET', KEYS[2]))
local amount = tonumber(ARGV[1])

if from_balance < amount then
    return {err = "INSUFFICIENT_BALANCE"}
end

redis.call('DECRBY', KEYS[1], amount)
redis.call('INCRBY', KEYS[2], amount)
return {ok = "TRANSFER_SUCCESS"}
```

调用方式：

```
EVAL "local from_balance = tonumber(redis.call('GET', KEYS[1]))..." 2 from_key to_key 100
```

**注意**：
- 脚本执行时间受 `lua-time-limit` 限制（默认 5 秒）
- 脚本中只能调用 Redis 命令，不能进行外部 I/O
- 超时后可用 `SCRIPT KILL` 终止（仅限只读脚本）

### 2. 使用补偿命令模式

对于简单场景，可手动设计补偿逻辑：

```
# 伪代码
try:
    MULTI
    SET key1 val1
    SET key2 val2
    result = EXEC
    if has_error(result):
        # 执行补偿命令
        SET key1 old_val1
        SET key2 old_val2
except:
    # 连接断开时事务自动放弃，无需补偿
```

**适用场景**：
- 操作可逆（如数值加减）
- 操作前后状态可获取

### 3. 使用 WATCH 实现条件执行

`WATCH` 可实现"条件失败时整体放弃"：

```
WATCH key1 key2
current1 = GET key1
current2 = GET key2
# 计算新值
MULTI
SET key1 new_val1
SET key2 new_val2
result = EXEC
if result is nil:
    # 键被修改，事务未执行，重试
    retry_operation()
```

**特点**：事务要么成功执行，要么完全不执行（类似"回滚到未开始"状态）。

### 4. 使用 Redis Functions（7.0+）

Redis 7.0 引入 Functions，提供比 Lua 脚本更结构化的方案：

- 支持函数库、持久化、版本管理
- 通过 `FUNCTION LOAD` 预加载，通过 `FCALL` 调用
- 函数持久化在 RDB/AOF 中，重启后仍可用

## Redis 事务与数据库事务的配合

当业务逻辑涉及 Redis 和数据库（如 MySQL）时，两者的事务机制独立，需要额外设计保证一致性。

### 问题分析

| 场景 | 问题 |
|------|------|
| **数据库先提交，Redis 后写入** | 若 Redis 写入失败，数据库已提交无法回滚 |
| **Redis 先写入，数据库后提交** | 若数据库提交失败，Redis 已修改 |
| **并行操作** | 两边都不保证顺序一致性 |

### 解决方案

#### 1. Outbox 模式（可靠消息投递）

将 Redis 操作作为"消息"持久化到数据库，再异步执行：

```
# 应用层伪代码
db.begin_transaction()
db.update_business_data()
db.insert_outbox_message(redis_operation)  # Redis 操作记录存入 DB
db.commit()

# 异步消费者
message = db.fetch_outbox_message()
redis.execute(message.operation)
db.mark_message_processed(message.id)
```

**优势**：
- 数据库事务保证 Redis 操作记录不丢失
- 异步消费者可重试
- 适合 Redis 作为缓存或消息队列场景

#### 2. Saga 模式（补偿事务）

将操作分解为多个步骤，每个步骤有对应的补偿操作：

```
# 正向流程
step1: db.update_account()
step2: redis.update_cache()

# 若 step2 失败
compensate: db.rollback_account_update()  # 业务层面的"回滚"
```

**适用场景**：
- 分布式事务场景
- 可接受最终一致性

#### 3. Redis 作为数据库操作的副作用

若 Redis 仅作为缓存，可接受 Redis 数据不一致：

```
db.begin_transaction()
db.update_data()
db.commit()
# 即使 Redis 更新失败，下次读取时重新加载
redis.update_cache()  # 可接受失败
```

**前提**：
- Redis 数据可重建
- 不依赖 Redis 作为数据源

#### 4. 两阶段提交（2PC）

严格一致性场景可使用 2PC，但复杂度高：

```
# 阶段 1：准备
redis.prepare_lock()
db.prepare_commit()

# 阶段 2：提交或回滚
if all_prepared:
    redis.commit()
    db.commit()
else:
    redis.abort()
    db.rollback()
```

**注意**：Redis 本身不支持 prepare/commit 语义，需应用层模拟。

### 最佳实践总结

| 场景 | 推荐方案 |
|------|----------|
| Redis 仅作缓存 | 事务后更新 Redis，失败时可接受（下次重建） |
| Redis 数据重要 | Outbox 模式，将 Redis 操作记录持久化到数据库 |
| 需严格一致性 | Saga 补偿模式，或接受性能代价使用 2PC 模拟 |
| 操作可逆 | 补偿命令模式，失败时手动回滚 Redis |

## 常见误区与注意事项

| 误区 | 事实 |
|------|------|
| "Redis 事务支持回滚" | 不支持；命令失败后其他命令继续执行 |
| "`WATCH` 能防止所有并发问题" | 仅防止所监视键被修改；其他键不受保护 |
| "Lua 脚本可以无限执行" | 受 `lua-time-limit` 限制，超时会被终止 |
| "事务中命令出错会阻止执行" | 入队错误阻止执行；执行错误不影响其他命令 |
| "Redis 事务等同于数据库事务" | Redis 事务提供隔离和原子执行，但无 ACID 的完整保证 |

## 相关概念

- **Lua Scripting**：Redis 内嵌 Lua 执行引擎，脚本原子执行，可替代事务实现复杂逻辑
- **WATCH/CAS**：乐观锁机制，检测键变化决定事务是否执行
- **Pipeline**：批量发送命令减少 RTT，但不提供事务隔离
- **Outbox Pattern**：将副作用操作持久化到数据库，异步执行
- **Saga Pattern**：分布式事务补偿模式，每个步骤有对应补偿操作

## 参考资料

- [Redis 官方文档 — Transactions](https://redis.io/docs/latest/develop/using-commands/transactions/)
- [Redis 官方文档 — Scripting with Lua](https://redis.io/docs/latest/develop/programmability/eval-intro/)
- [Redis Lua API Reference](https://redis.io/docs/latest/develop/programmability/lua-api/)
- [Redis Commands — MULTI](https://redis.io/docs/latest/commands/multi/)
- [Redis Commands — WATCH](https://redis.io/docs/latest/commands/watch/)
- [GitHub PR #7920 — WATCH expired key behavior](https://github.com/redis/redis/pull/7920)