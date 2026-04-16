---
created: '2026-04-16'
updated: '2026-04-16'
tags:
  - MySQL
  - 数据库
  - 日志
  - 运维
  - 排查
aliases:
  - MySQL Server Logs
  - MySQL 日志系统
source_type: official-doc
source_urls:
  - 'https://dev.mysql.com/doc/refman/8.4/en/server-logs.html'
  - 'https://dev.mysql.com/doc/refman/8.4/en/error-log.html'
  - 'https://dev.mysql.com/doc/refman/8.4/en/query-log.html'
  - 'https://dev.mysql.com/doc/refman/8.4/en/binary-log.html'
  - 'https://dev.mysql.com/doc/refman/8.4/en/slow-query-log.html'
status: verified
---
## 概述

MySQL Server 提供多种日志用于记录服务器运行状态、客户端行为、数据变更和性能瓶颈。默认情况下，除 Windows 平台上的错误日志外，其余日志均处于关闭状态。

## 日志类型总览

| 日志类型 | 记录内容 | 主要用途 | 默认状态 |
| :--- | :--- | :--- | :--- |
| **Error Log（错误日志）** | 服务器启动、运行、关闭过程中的错误、警告和提示信息 | 排查服务器故障、崩溃、表损坏等问题 | Windows 默认开启 |
| **General Query Log（通用查询日志）** | 客户端连接/断开事件、接收到的所有 SQL 语句 | 调试客户端行为、审计完整 SQL 请求 | 关闭 |
| **Binary Log（二进制日志）** | 数据变更事件（DDL、DML），不记录 SELECT/SHOW 等只读操作 | 主从复制、时间点恢复（Point-in-Time Recovery） | 开启（8.4 起） |
| **Slow Query Log（慢查询日志）** | 执行时间超过 `long_query_time` 阈值的 SQL 语句 | 性能调优、定位慢查询 | 关闭 |
| **Relay Log（中继日志）** | 从复制源服务器接收到的数据变更事件 | 仅用于复制架构中的从节点 | 关闭（仅从节点使用） |
| **DDL Log（DDL 日志）** | 原子 DDL 操作的执行记录 | 追踪 DDL 语句的执行过程 | 关闭 |

---

## Error Log（错误日志）

### 作用

错误日志记录 `mysqld` 的启动和关闭时间，以及运行期间的诊断信息，包括：

- 启动/关闭过程中的错误、警告和提示
- 服务器运行时检测到的异常（如表需要自动检查或修复）
- 非正常退出时的堆栈跟踪（部分操作系统支持）
- `mysqld_safe` 检测到异常退出并重启时写入的 `mysqld restarted` 消息

### 配置

- 错误日志可通过 `log_error` 系统变量配置输出路径
- 支持输出到文件、系统日志（syslog）或 Performance Schema 的 `error_log` 表
- 支持 JSON 格式输出（`log_error_services` 组件配置）
- 支持基于优先级和规则的过滤机制

### 注意事项

- 错误日志是排查 MySQL 崩溃、启动失败等问题的**第一入口**
- 堆栈跟踪可用于定位 `mysqld` 异常退出的具体位置

---

## General Query Log（通用查询日志）

### 作用

通用查询日志是 `mysqld` 行为的完整记录，包含：

- 客户端连接和断开事件（含连接协议类型：TCP/IP、SSL/TLS、Socket 等）
- 从客户端接收到的**每一条 SQL 语句**

适用于怀疑客户端行为异常、需要精确追踪客户端发送内容的场景。

### 配置

```sql
-- 开启/关闭
SET GLOBAL general_log = 'ON';
SET GLOBAL general_log = 'OFF';

-- 指定日志文件路径
SET GLOBAL general_log_file = '/path/to/general.log';
```

- 启动参数：`--general_log[={0|1}]`、`--general_log_file=file_name`
- 输出目标由 `log_output` 控制，可选 `FILE`、`TABLE` 或 `NONE`
- 默认文件名：`host_name.log`，存放于数据目录

### 注意事项

- **性能影响大**：开启后会记录所有 SQL，生产环境不建议长期开启
- 语句按**接收顺序**写入，可能与实际执行顺序不同
- 密码字段会被自动重写为明文不显示（可通过 `--log-raw` 关闭此行为，但生产环境不推荐）
- 语法错误的语句不会被记录（因为无法确认是否包含密码）
- 可通过会话级 `sql_log_off` 变量控制当前会话是否写入通用日志

---

## Binary Log（二进制日志）

### 作用

二进制日志记录所有**修改数据的事件**，包括：

- 表创建等 DDL 操作
- 数据变更（INSERT、UPDATE、DELETE）
- 可能修改数据的语句（如未匹配到行的 DELETE，行格式日志除外）
- 每个更新语句的执行耗时

**两个核心用途：**

1. **主从复制**：主节点将 binlog 事件发送给从节点，从节点重放这些事件实现数据同步
2. **时间点恢复**：备份恢复后，重放 binlog 中备份之后的事件，将数据库恢复到指定时间点

### 配置

```sql
-- 查看 binlog 状态
SHOW BINARY LOGS;
SHOW MASTER STATUS;

-- 查看 binlog 内容（命令行）
mysqlbinlog binlog.000001 | mysql -h server_name
```

- 启动参数：`--log-bin[=base_name]`（8.4 起默认开启）
- 禁用：`--skip-log-bin` 或 `--disable-log-bin`
- 文件格式：`base_name.NNNNNN`（自动递增序号）
- 索引文件：`base_name.index`（记录所有 binlog 文件名）
- 切换条件：服务器重启、执行 `FLUSH LOGS`、文件大小达到 `max_binlog_size`

### 日志格式

| 格式 | 说明 |
| :--- | :--- |
| **ROW（行格式）** | 记录实际的数据行变更，复制最安全，但不记录原始 SQL |
| **STATEMENT（语句格式）** | 记录原始 SQL 语句，体积小但某些函数可能导致主从不一致 |
| **MIXED（混合格式）** | 默认使用 STATEMENT，特定场景自动切换为 ROW |

### 注意事项

- 开启 binlog 会带来轻微性能损耗，但复制和恢复的收益远大于此
- `SELECT` 和 `SHOW` 等只读语句**不会**写入 binlog
- 事务在 `COMMIT` 后才写入 binlog，确保按提交顺序记录
- 非事务表（如 MyISAM）的修改立即写入，无法回滚
- 密码字段会被自动重写，不以明文存储
- 可通过 `SET sql_log_bin=OFF` 临时关闭当前会话的 binlog 记录
- 清理 binlog 推荐使用 `PURGE BINARY LOGS`，不要手动删除文件
- `sync_binlog=1`（默认）确保每次写入都同步到磁盘，是最安全但最慢的配置
- MySQL 8.4 中 InnoDB 的 XA 两阶段提交始终启用，确保 binlog 与 InnoDB 数据文件一致

---

## Slow Query Log（慢查询日志）

### 作用

慢查询日志记录满足以下条件的 SQL 语句：

- 执行时间超过 `long_query_time` 秒（默认 10 秒，最小 0 秒，支持微秒精度）
- 扫描行数达到 `min_examined_row_limit` 阈值

用于定位执行时间长的查询，是 SQL 性能调优的核心工具。

### 配置

```sql
-- 开启/关闭
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL slow_query_log = 'OFF';

-- 指定日志文件路径
SET GLOBAL slow_query_log_file = '/path/to/slow.log';

-- 设置慢查询阈值（秒）
SET GLOBAL long_query_time = 1;
```

- 启动参数：`--slow_query_log[={0|1}]`、`--slow_query_log_file=file_name`、`--long_query_time=N`
- 默认文件名：`host_name-slow.log`

### 可选功能

| 参数 | 作用 |
| :--- | :--- |
| `log_slow_admin_statements` | 记录慢的 DDL 语句（ALTER TABLE、ANALYZE TABLE 等） |
| `log_queries_not_using_indexes` | 记录未使用索引的查询（即使执行时间未超阈值） |
| `log_throttle_queries_not_using_indexes` | 限制未使用索引查询的日志写入频率（每分钟上限） |
| `log_slow_extra` | 写入额外字段（Thread_id、Bytes_sent、Created_tmp_tables 等） |
| `log_slow_replica_statements` | 允许从节点记录复制过来的慢查询 |

### 日志内容格式

每条慢查询前有一行 `#` 开头的元数据：

```
# Query_time: 2.345  Lock_time: 0.001  Rows_sent: 100  Rows_examined: 50000
```

- `Query_time`：语句执行时间（秒）
- `Lock_time`：获取锁的时间（秒）
- `Rows_sent`：返回给客户端的行数
- `Rows_examined`：服务器层扫描的行数

开启 `log_slow_extra` 后还会包含 Thread_id、Errno、Bytes_received/sent、临时表创建数、排序信息等。

### 分析工具

```bash
# 使用 mysqldumpslow 汇总慢查询日志
mysqldumpslow /path/to/slow.log
```

### 注意事项

- 获取锁的时间**不计入**执行时间
- 语句在**执行完成后且所有锁释放后**才写入日志，因此日志顺序可能与执行顺序不同
- 开启 `log_queries_not_using_indexes` 后日志可能快速增长，建议配合 `log_throttle_queries_not_using_indexes` 限制频率
- 从节点默认不记录复制过来的慢查询，需显式开启 `log_slow_replica_statements`
- 行格式复制（`binlog_format=ROW`）下，从节点不会将复制的查询写入慢查询日志
- 密码字段会被自动重写，语法错误的语句不会被记录

---

## Relay Log（中继日志）

### 作用

中继日志仅在复制架构的**从节点**上使用，用于存储从主节点接收到的数据变更事件。从节点的 SQL 线程读取中继日志并重放这些事件，实现与主节点的数据同步。

- 格式与 binlog 相同，可使用 `mysqlbinlog` 工具查看
- 配置和管理详见复制相关文档

---

## DDL Log（DDL 日志）

### 作用

记录原子 DDL 操作的执行过程。MySQL 8.0+ 引入了原子 DDL 特性，确保 DDL 语句要么完全成功，要么完全回滚。

- 可通过相关文档查看 DDL 日志的具体行为

---

## 日志维护

### 日志刷新

执行以下操作可刷新日志（关闭并重新打开）：

```sql
FLUSH LOGS;
```

```bash
mysqladmin flush-logs
mysqldump --flush-logs
```

### 通用查询日志重命名

```bash
mv host_name.log host_name-old.log
mysqladmin flush-logs general
mv host_name-old.log backup-directory
```

或在运行时：

```sql
SET GLOBAL general_log = 'OFF';
-- 外部重命名文件
SET GLOBAL general_log = 'ON';
```

### Binlog 清理

```sql
-- 清理指定日期之前的 binlog
PURGE BINARY LOGS BEFORE '2026-04-01 00:00:00';

-- 清理所有 binlog 和 GTID（谨慎使用）
RESET BINARY LOGS AND GTIDS;
```

> **警告**：删除主节点 binlog 前，必须确认所有从节点已不需要这些日志。

---

## 安全注意事项

- 所有日志中的密码字段默认会被重写，不以明文存储
- `--log-raw` 选项可关闭密码重写，但仅限诊断用途，**不推荐生产环境使用**
- 日志文件应设置适当的访问权限，防止未授权查看
- 详见 [Passwords and Logging](https://dev.mysql.com/doc/refman/8.4/en/password-logging.html)

---

## 常见误区

| 误区 | 说明 |
| :--- | :--- |
| 通用日志和 binlog 是一回事 | 通用日志记录所有 SQL（含 SELECT），binlog 只记录数据变更 |
| 慢查询日志记录所有慢的 SQL | 还需满足 `min_examined_row_limit` 扫描行数阈值 |
| binlog 可以用于审计所有查询 | binlog 不记录 SELECT，审计应使用通用查询日志 |
| 日志顺序 = 执行顺序 | 通用日志按接收顺序记录，慢查询日志按完成顺序记录，只有 binlog 按提交顺序 |
| 关闭日志能大幅提升性能 | 只有通用查询日志对性能影响显著，binlog 和慢查询日志影响相对较小 |

---

## 参考资料

- [MySQL 8.4 Reference Manual - Server Logs](https://dev.mysql.com/doc/refman/8.4/en/server-logs.html)
- [The Error Log](https://dev.mysql.com/doc/refman/8.4/en/error-log.html)
- [The General Query Log](https://dev.mysql.com/doc/refman/8.4/en/query-log.html)
- [The Binary Log](https://dev.mysql.com/doc/refman/8.4/en/binary-log.html)
- [The Slow Query Log](https://dev.mysql.com/doc/refman/8.4/en/slow-query-log.html)
