---
created: '2026-04-16'
updated: '2026-04-16'
tags:
  - database
  - mysql
  - logging
  - 运维
  - 故障排查
aliases:
  - MySQL日志类型
  - MySQL Server Logs
  - MySQL log types
source_type: official-doc
source_urls:
  - 'https://dev.mysql.com/doc/refman/8.4/en/server-logs.html'
  - 'https://dev.mysql.com/doc/refman/8.4/en/error-log.html'
  - 'https://dev.mysql.com/doc/refman/8.4/en/query-log.html'
  - 'https://dev.mysql.com/doc/refman/8.4/en/binary-log.html'
  - 'https://dev.mysql.com/doc/refman/8.4/en/slow-query-log.html'
  - 'https://dev.mysql.com/doc/refman/8.4/en/innodb-redo-log.html'
  - 'https://dev.mysql.com/doc/refman/8.4/en/innodb-undo-logs.html'
status: verified
---

## 概述

MySQL Server 维护多种日志，用于记录服务器运行状态、数据变更、客户端行为和诊断信息。这些日志在**故障排查、数据恢复、复制、性能优化**等场景中发挥关键作用。

> **注意**：除 Error Log 在 Windows 上默认开启外，**默认情况下所有日志均未启用**。需要手动通过启动参数或运行时变量开启。（[来源](https://dev.mysql.com/doc/refman/8.4/en/server-logs.html)）

## 日志类型总览

| 日志类型 | 记录内容 | 主要用途 | 存储引擎/层级 |
|---|---|---|---|
| **Error Log** | 服务器启动、运行、关闭时的错误/警告/通知 | 故障诊断 | Server 层 |
| **General Query Log** | 客户端连接/断开、所有收到的 SQL 语句 | 调试、审计 | Server 层 |
| **Binary Log** | 数据变更事件（DDL/DML） | 复制、时间点恢复 | Server 层 |
| **Relay Log** | 从主库接收到的数据变更 | 复制（仅 Replica 端） | Server 层 |
| **Slow Query Log** | 执行时间超过阈值的 SQL | 性能优化 | Server 层 |
| **Redo Log** | InnoDB 事务的物理修改记录 | 崩溃恢复、持久化 | InnoDB 引擎层 |
| **Undo Log** | 事务修改前的旧值（回滚信息） | 事务回滚、MVCC | InnoDB 引擎层 |
| **DDL Log** | 原子 DDL 操作的元数据日志 | DDL 崩溃恢复 | Server 层 |

---

## 一、Error Log（错误日志）

### 是什么

Error Log 记录 `mysqld` 启动和关闭的时间点，以及运行过程中产生的**错误（Error）、警告（Warning）和通知（Note）**级别诊断消息。

### 记录内容

- 服务器启动/关闭事件
- 运行时错误（如表需要自动检查或修复）
- 异常退出时的堆栈跟踪（部分操作系统）
- `mysqld_safe` 重启 `mysqld` 时的通知

### 配置方式

```ini
[mysqld]
# 指定错误日志文件路径（默认为主机名.err，位于数据目录）
log_error = /var/log/mysql/error.log

# 日志输出格式：stderr（默认）、file、syslog、eventlog（Windows）
log_error_services = log_sink_internal

# 日志级别过滤：System, Error, Warning, Note
log_error_verbosity = 3
```

### 关键特性

- **默认启用**（Windows 上默认开启，其他平台也通常默认写入）
- 支持 **JSON 格式输出**（`log_error_services = log_sink_json`）
- 可通过 `log_error_suppression_list` 过滤特定错误码
- 消息可写入 Performance Schema 的 `error_log` 表，支持 SQL 查询

### 注意事项

- Error Log 不会自动轮转，需配合 `FLUSH ERROR LOG` 或外部日志管理工具（如 logrotate）
- 生产环境建议设置 `log_error_verbosity = 3` 以获取完整诊断信息

> 参考：[MySQL 8.4 - The Error Log](https://dev.mysql.com/doc/refman/8.4/en/error-log.html)

---

## 二、General Query Log（通用查询日志）

### 是什么

General Query Log 是 `mysqld` 的**完整活动记录**，记录所有客户端连接/断开事件以及收到的每一条 SQL 语句。

### 记录内容

- 客户端连接/断开（包含连接协议：TCP/IP、SSL/TLS、Socket 等）
- 所有收到的 SQL 语句（按接收顺序，非执行顺序）
- 包含只读查询（SELECT、SHOW 等），这点与 Binary Log 不同

### 配置方式

```ini
[mysqld]
general_log = 1
general_log_file = /var/log/mysql/general.log

# 输出目标：FILE（文件）、TABLE（mysql.general_log 表）、NONE
log_output = FILE
```

运行时动态控制：

```sql
SET GLOBAL general_log = 'ON';
SET GLOBAL general_log_file = '/var/log/mysql/general.log';
```

### 关键特性

- 语句中的**密码会被自动重写**，不以明文形式记录
- 可通过 `--log-raw` 关闭密码重写（仅用于诊断，不推荐生产使用）
- 无法解析的语句（如语法错误）默认不写入（除非使用 `--log-raw`）
- 可通过会话级 `sql_log_off` 变量关闭当前会话的日志记录

### 注意事项

- **性能开销极大**：每条语句都记录，生产环境通常不建议开启
- 日志文件不会自动轮转，需手动重命名后 `FLUSH LOGS`
- 适用于短期调试，定位客户端发送的具体 SQL

> 参考：[MySQL 8.4 - The General Query Log](https://dev.mysql.com/doc/refman/8.4/en/query-log.html)

---

## 三、Binary Log（二进制日志）

### 是什么

Binary Log 包含描述**数据库变更**的"事件"（event），如表创建操作、数据修改等。它是 MySQL 中最重要的日志之一。

### 两大核心用途

1. **复制（Replication）**：主库将 Binary Log 中的事件发送给从库，从库重放这些事件实现数据同步
2. **时间点恢复（Point-in-Time Recovery）**：备份恢复后，通过重放 Binary Log 中的事件将数据库恢复到指定时间点

### 记录内容

- 所有修改数据的语句（DDL、DML）
- 不记录只读语句（SELECT、SHOW）
- 包含每条数据变更语句的执行耗时
- 即使语句未影响任何行（如 DELETE 匹配 0 行），也会被记录（ROW 格式除外）

### 配置方式

```ini
[mysqld]
# MySQL 8.4 默认启用 Binary Log
log_bin = /var/log/mysql/binlog
server_id = 1

# 日志格式：ROW（默认）、STATEMENT、MIXED
binlog_format = ROW

# 每个 binlog 文件最大大小（默认 1GB）
max_binlog_size = 1073748576

# 每次写入后同步到磁盘（最安全，默认值为 1）
sync_binlog = 1

# binlog 保留天数（0 表示不过期）
binlog_expire_logs_seconds = 2592000  # 30 天
```

### 三种日志格式

| 格式 | 说明 | 优点 | 缺点 |
|---|---|---|---|
| **ROW** | 记录每行数据的变更 | 精确、可重放、安全 | 日志体积大（大批量操作时） |
| **STATEMENT** | 记录原始 SQL 语句 | 日志体积小 | 某些函数（NOW()、RAND()）可能导致主从不一致 |
| **MIXED** | 自动在 ROW 和 STATEMENT 间切换 | 兼顾两者 | 行为较复杂 |

> MySQL 8.4 默认使用 **ROW** 格式。

### 文件结构

- 二进制日志文件：`binlog.000001`、`binlog.000002`...（按序号递增）
- 索引文件：`binlog.index`（记录所有 binlog 文件名）
- 触发新文件的事件：服务器重启、`FLUSH LOGS`、达到 `max_binlog_size`

### 查看与管理

```sql
-- 查看 binlog 状态
SHOW BINARY LOGS;
SHOW MASTER STATUS;

-- 查看当前 binlog 事件
SHOW BINLOG EVENTS IN 'binlog.000001';

-- 清理过期 binlog
PURGE BINARY LOGS BEFORE '2026-04-01 00:00:00';

-- 使用 mysqlbinlog 工具查看
-- shell> mysqlbinlog binlog.000001
```

### 关键特性

- **写入时机**：语句/事务完成后、锁释放前写入，保证按提交顺序记录
- **事务缓存**：事务性表的修改先缓存在 `binlog_cache_size` 中，COMMIT 时一次性写入
- **非事务性表**（如 MyISAM）的修改立即写入，不受事务回滚影响
- **密码保护**：写入 binlog 的密码会被重写，不以明文存储
- **加密支持**：可通过 `binlog_encryption = ON` 加密 binlog 文件
- **崩溃恢复**：`sync_binlog = 1` + InnoDB 两阶段提交（XA）确保 binlog 与数据文件一致

### 注意事项

- 开启 binlog 会带来轻微性能损耗，但复制和恢复的收益远大于此
- 删除 binlog 前必须确认所有从库已应用完毕
- 大事务可能导致单个 binlog 文件超过 `max_binlog_size`（事务不会跨文件分割）
- 可通过 `SET sql_log_bin = OFF` 临时关闭当前会话的 binlog 记录

> 参考：[MySQL 8.4 - The Binary Log](https://dev.mysql.com/doc/refman/8.4/en/binary-log.html)

---

## 四、Relay Log（中继日志）

### 是什么

Relay Log 仅在**复制架构的从库（Replica）**上存在，用于存储从主库（Source）接收到的数据变更事件。

### 工作机制

1. 从库的 I/O 线程从主库读取 Binary Log 事件
2. 将事件写入本地的 Relay Log
3. 从库的 SQL 线程读取 Relay Log 并重放事件，实现数据同步

### 配置方式

```ini
[mysqld]
# Relay Log 文件路径（默认自动生成）
relay_log = /var/log/mysql/relay-log

# 从库更新也写入自己的 Binary Log（级联复制需要）
log_replica_updates = 1
```

### 关键特性

- Relay Log 与 Binary Log 使用**相同的文件格式**
- 可通过 `mysqlbinlog` 工具查看 Relay Log 内容
- Relay Log 也会被自动清理（通过 `relay_log_purge` 控制）

> 参考：[MySQL 8.4 - The Relay Log](https://dev.mysql.com/doc/refman/8.4/en/replica-logs-relaylog.html)

---

## 五、Slow Query Log（慢查询日志）

### 是什么

Slow Query Log 记录执行时间超过指定阈值的 SQL 语句，是**性能调优的核心工具**。

### 记录条件

一条查询被记录需同时满足以下条件：

1. 执行时间 ≥ `long_query_time`（默认 10 秒）
2. 扫描行数 ≥ `min_examined_row_limit`（默认 0）
3. 非管理语句（除非开启 `log_slow_admin_statements`）
4. 未被 `log_throttle_queries_not_using_indexes` 限流

### 配置方式

```ini
[mysqld]
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log

# 慢查询阈值（秒），支持微秒精度
long_query_time = 1

# 记录未使用索引的查询
log_queries_not_using_indexes = 1

# 限制每分钟记录无索引查询的数量（0 = 不限制）
log_throttle_queries_not_using_indexes = 0

# 记录管理语句（ALTER TABLE、OPTIMIZE TABLE 等）
log_slow_admin_statements = 1

# 记录额外字段（Thread_id、Bytes_sent 等）
log_slow_extra = 1
```

### 日志内容示例

```
# Time: 2026-04-16T10:30:00.000000Z
# User@Host: root[root] @ localhost []  Thread_id: 10
# Query_time: 12.345678  Lock_time: 0.000123  Rows_sent: 100  Rows_examined: 5000000
SET timestamp=1713265800;
SELECT * FROM large_table WHERE status = 'active';
```

启用 `log_slow_extra` 后额外字段包括：`Thread_id`、`Errno`、`Bytes_received`、`Bytes_sent`、`Read_first`、`Read_key`、`Created_tmp_disk_tables`、`Start`、`End` 等。

### 分析工具

```bash
# mysqldumpslow 汇总慢查询日志
mysqldumpslow -s t -t 10 /var/log/mysql/slow.log
# -s t: 按执行时间排序
# -t 10: 显示前 10 条
```

### 注意事项

- 获取锁的时间**不计入**执行时间（`Query_time` 不包含 `Lock_time`）
- 日志写入发生在语句执行完成且锁释放之后，因此日志顺序可能与执行顺序不同
- 密码会被重写，不以明文记录
- 从库默认不记录复制来的慢查询，需开启 `log_slow_replica_statements`
- 在 ROW 格式复制下，`log_slow_replica_statements` 无效（因为从库收到的是行变更而非 SQL）

> 参考：[MySQL 8.4 - The Slow Query Log](https://dev.mysql.com/doc/refman/8.4/en/slow-query-log.html)

---

## 六、Redo Log（重做日志）

### 是什么

Redo Log 是 InnoDB 存储引擎的**磁盘级数据结构**，用于崩溃恢复时重放未完成事务的修改，确保数据持久性（ACID 中的 **D**urability）。

### 工作机制

1. 事务修改数据页时，先写入 **Redo Log Buffer**（内存）
2. 事务提交时，Redo Log 刷盘（WAL：Write-Ahead Logging）
3. 数据页延迟刷盘（脏页机制），提高写入性能
4. 崩溃重启时，通过 Redo Log 重放未持久化的修改

### 物理结构

- 文件位置：数据目录下的 `#innodb_redo/` 目录
- 文件命名：`#ib_redo<N>`（活跃文件）、`#ib_redo<N>_tmp`（备用文件）
- InnoDB 维护 **32 个 redo log 文件**（活跃 + 备用）
- 通过 LSN（Log Sequence Number）追踪写入位置

### 配置方式

```ini
[mysqld]
# Redo Log 总容量（MySQL 8.0.30+ 推荐用法）
innodb_redo_log_capacity = 1G

# 旧版配置方式（仍兼容）
# innodb_log_file_size = 64M
# innodb_log_files_in_group = 2

# Redo Log 目录（默认在数据目录）
# innodb_log_group_home_dir = /path/to/redo
```

MySQL 8.0.30+ 引入 `innodb_redo_log_capacity`，支持**运行时动态调整**：

```sql
SET GLOBAL innodb_redo_log_capacity = 8589934592;  -- 8GB
```

### 关键特性

- **WAL 机制**：先写日志再写数据，崩溃恢复的基础
- **循环写入**：Redo Log 是循环使用的，checkpoint 之后的旧数据会被截断
- **Checkpoint**：InnoDB 定期将脏页刷盘，并记录 checkpoint LSN
- **崩溃恢复**：从最新 checkpoint LSN 开始重放 Redo Log

### Redo Log Archiving（备份归档）

备份工具（如 MySQL Enterprise Backup）可激活 Redo Log 归档，避免备份期间 redo 被覆盖：

```sql
-- 配置归档目录
SET GLOBAL innodb_redo_log_archive_dirs = 'backup:/path/to/archive';

-- 启动归档
SELECT innodb_redo_log_archive_start('backup', 'subdir');

-- 停止归档
SELECT innodb_redo_log_archive_stop();
```

### 禁用 Redo Log（仅限数据导入场景）

```sql
ALTER INSTANCE DISABLE INNODB REDO_LOG;
-- 执行大量数据导入...
ALTER INSTANCE ENABLE INNODB REDO_LOG;
```

> **警告**：禁用期间若发生意外停机，会导致**数据丢失和实例损坏**。仅限新实例数据导入场景使用。

### 监控

```sql
-- 查看 redo log 文件状态
SELECT FILE_NAME, START_LSN, END_LSN FROM performance_schema.innodb_redo_log_files;

-- 查看相关状态变量
SHOW STATUS LIKE 'Innodb_redo_log%';
```

> 参考：[MySQL 8.4 - Redo Log](https://dev.mysql.com/doc/refman/8.4/en/innodb-redo-log.html)

---

## 七、Undo Log（回滚日志）

### 是什么

Undo Log 记录事务修改前的**旧值**，用于：

1. **事务回滚**：ROLLBACK 时恢复原始数据
2. **MVCC（多版本并发控制）**：提供一致性非锁定读（快照读）

### 工作机制

- 修改数据前，将旧值写入 Undo Log
- 每个事务关联一个 Undo Log 段
- Undo Log 存储在 **Undo Tablespace** 中（默认 `innodb_undo` 表空间）
- 事务提交后，Undo Log 不会被立即删除，而是由 **Purge 线程**异步清理

### 配置方式

```ini
[mysqld]
# Undo Tablespace 数量
innodb_undo_tablespaces = 2

# Undo Tablespace 目录
innodb_undo_directory = /var/lib/mysql/undo/

# Undo Log 截断阈值（页数量）
innodb_max_undo_log_size = 1G

# Purge 线程数量
innodb_purge_threads = 4
```

### 关键特性

- Undo Log 支持**自动截断**：当 Undo Tablespace 超过 `innodb_max_undo_log_size` 时，标记为可截断并重建
- 长事务会阻止 Undo Log 清理，可能导致 Undo Tablespace 膨胀
- MVCC 读（一致性读）依赖 Undo Log 构建历史版本

### 与 Redo Log 的关系

| 特性 | Redo Log | Undo Log |
|---|---|---|
| 记录内容 | 修改后的新值（物理修改） | 修改前的旧值（逻辑回滚） |
| 主要用途 | 崩溃恢复（前滚） | 事务回滚、MVCC（回退） |
| 存储位置 | `#innodb_redo/` | Undo Tablespace |
| 生命周期 | 循环覆盖（checkpoint 后） | Purge 线程清理 |
| 所属层级 | InnoDB 引擎层 | InnoDB 引擎层 |

> 参考：[MySQL 8.4 - Undo Logs](https://dev.mysql.com/doc/refman/8.4/en/innodb-undo-logs.html)

---

## 八、DDL Log（DDL 日志）

### 是什么

DDL Log 记录**原子 DDL 操作**的元数据信息，用于 DDL 语句在执行过程中发生崩溃时的恢复。

### 关键特性

- 仅记录 DDL 操作（CREATE、ALTER、DROP 等）
- MySQL 8.0+ 支持原子 DDL，DDL Log 是实现原子性的关键组件
- 崩溃恢复时，根据 DDL Log 判断 DDL 操作是否完成，决定回滚或继续
- 默认行为由服务器自动控制，通常无需手动配置

> 参考：[MySQL 8.4 - Atomic DDL](https://dev.mysql.com/doc/refman/8.4/en/atomic-ddl.html)

---

## 日志对比与关联

### Server 层 vs 存储引擎层

```
┌─────────────────────────────────────────────────┐
│                   MySQL Server                   │
│  ┌──────────┐  ┌────────────┐  ┌──────────────┐ │
│  │Error Log │  │General Log │  │  Binary Log  │ │
│  └──────────┘  └────────────┘  └──────────────┘ │
│  ┌──────────────┐  ┌──────────────────────────┐ │
│  │Slow Query Log│  │        DDL Log           │ │
│  └──────────────┘  └──────────────────────────┘ │
├─────────────────────────────────────────────────┤
│              Storage Engine (InnoDB)             │
│  ┌──────────────┐  ┌──────────────────────────┐ │
│  │  Redo Log    │  │        Undo Log          │ │
│  └──────────────┘  └──────────────────────────┘ │
└─────────────────────────────────────────────────┘
```

### 一次事务的日志写入流程

```
1. 事务开始
2. 修改数据 → 写入 Redo Log Buffer（内存）
3. 修改数据 → 写入 Undo Log
4. 事务提交
   a. Redo Log 刷盘（WAL）
   b. 写入 Binary Log
   c. InnoDB 提交事务（两阶段提交）
5. 数据页延迟刷盘（Checkpoint 机制）
```

### 常用查询命令

```sql
-- 查看日志状态
SHOW VARIABLES LIKE 'log_%';
SHOW VARIABLES LIKE 'binlog_%';
SHOW VARIABLES LIKE 'innodb_redo%';
SHOW VARIABLES LIKE 'slow_query%';
SHOW VARIABLES LIKE 'general_log%';

-- 查看 Binary Log 状态
SHOW BINARY LOGS;
SHOW MASTER STATUS;

-- 查看 Redo Log 状态
SHOW STATUS LIKE 'Innodb_redo_log%';
```

---

## 最佳实践

1. **生产环境必开**：Error Log、Binary Log
2. **复制架构必开**：Binary Log（主库）、Relay Log（从库，自动开启）
3. **按需开启**：Slow Query Log（性能调优时）、General Query Log（短期调试）
4. **Redo Log 容量**：根据写入负载调整，写密集型应用建议 2GB+
5. **日志安全**：密码会被自动重写，但 General Query Log 仍可能泄露敏感数据，注意权限控制
6. **日志轮转**：使用 `FLUSH LOGS` 或外部工具管理日志文件大小
7. **binlog 清理**：使用 `PURGE BINARY LOGS` 而非直接删除文件，确保索引文件同步更新
8. **sync_binlog = 1**：最高安全性，确保崩溃时不丢失 binlog 数据

---

## 参考资料

- [MySQL 8.4 Reference Manual - Server Logs](https://dev.mysql.com/doc/refman/8.4/en/server-logs.html)
- [MySQL 8.4 - The Error Log](https://dev.mysql.com/doc/refman/8.4/en/error-log.html)
- [MySQL 8.4 - The General Query Log](https://dev.mysql.com/doc/refman/8.4/en/query-log.html)
- [MySQL 8.4 - The Binary Log](https://dev.mysql.com/doc/refman/8.4/en/binary-log.html)
- [MySQL 8.4 - The Slow Query Log](https://dev.mysql.com/doc/refman/8.4/en/slow-query-log.html)
- [MySQL 8.4 - InnoDB Redo Log](https://dev.mysql.com/doc/refman/8.4/en/innodb-redo-log.html)
- [MySQL 8.4 - InnoDB Undo Logs](https://dev.mysql.com/doc/refman/8.4/en/innodb-undo-logs.html)
- [MySQL 8.4 - Atomic DDL](https://dev.mysql.com/doc/refman/8.4/en/atomic-ddl.html)
