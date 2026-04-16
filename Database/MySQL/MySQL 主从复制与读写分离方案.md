---
created: '2026-04-16'
updated: '2026-04-16'
tags:
  - MySQL
  - 数据库
  - 主从复制
  - 读写分离
  - 高可用
  - 架构设计
aliases:
  - MySQL Replication
  - MySQL Read-Write Splitting
  - MySQL Master-Slave Replication
source_type: official-doc
source_urls:
  - 'https://dev.mysql.com/doc/refman/8.4/en/replication.html'
  - 'https://dev.mysql.com/doc/refman/8.4/en/replication-implementation.html'
  - 'https://dev.mysql.com/doc/refman/8.4/en/replication-formats.html'
  - 'https://dev.mysql.com/doc/refman/8.4/en/replication-semisync.html'
  - 'https://dev.mysql.com/doc/refman/8.4/en/replication-threads.html'
  - 'https://dev.mysql.com/doc/refman/8.4/en/replication-problems.html'
status: verified
---

## 概述

MySQL 主从复制（Replication）是将一台 MySQL 服务器（Source / 主库）的数据变更同步到一台或多台 MySQL 服务器（Replica / 从库）的机制。读写分离是在主从复制基础上，将写操作路由到主库、读操作路由到从库的架构模式，用于扩展读性能和实现高可用。

MySQL 8.4 起官方术语已将 Master/Slave 替换为 Source/Replica[^1]。

[^1]: https://dev.mysql.com/doc/refman/8.4/en/replication.html

## 主从复制原理

### 工作机制

主从复制基于主库的 **Binary Log（binlog）** 实现，核心流程如下：

1. **主库** 将所有数据变更（INSERT、UPDATE、DELETE、DDL 等）记录到 binlog
2. **从库** 的 I/O 线程连接主库，请求 binlog 事件（Pull 模式，非 Push）
3. 从库 I/O 线程将收到的事件写入本地的 **Relay Log（中继日志）**
4. 从库的 SQL 线程读取 Relay Log 并重放（Replay）这些事件，使数据与主库一致

### 核心线程

| 线程 | 所在服务器 | 作用 |
|------|-----------|------|
| Binlog Dump Thread | 主库 | 向从库发送 binlog 事件 |
| I/O Receiver Thread | 从库 | 连接主库，接收 binlog 并写入 Relay Log |
| SQL Applier Thread | 从库 | 读取 Relay Log 并执行其中的事务 |

从库 8.0+ 支持多线程复制（MTS），通过设置 `replica_parallel_workers > 0` 可创建多个 Worker 线程并行应用事务，配合 Coordinator 线程进行调度[^2]。

[^2]: https://dev.mysql.com/doc/refman/8.4/en/replication-threads.html

### 复制格式

MySQL 支持三种 binlog 格式，直接影响复制行为：

| 格式 | 说明 | 特点 |
|------|------|------|
| **ROW**（默认） | 记录每一行数据的变更 | 精确、安全，但 binlog 体积较大 |
| **STATEMENT (SBR)** | 记录原始 SQL 语句 | binlog 体积小，但对非确定性函数（如 `NOW()`, `UUID()`）可能产生不一致 |
| **MIXED (MBR)** | 默认 SBR，特定场景自动切换 ROW | 兼顾体积与安全性 |

> **重要**：`binlog_format` 在 MySQL 8.0 起已被标记为废弃（deprecated），未来版本将只保留 ROW 格式[^3]。生产环境建议直接使用 ROW 格式。

[^3]: https://dev.mysql.com/doc/refman/8.4/en/replication-formats.html

## 同步模式

### 异步复制（Asynchronous Replication）

- **默认模式**
- 主库写入 binlog 后立即返回给客户端，不等待从库确认
- **优点**：写入延迟最低，性能最好
- **缺点**：主库宕机时，已提交的事务可能未传送到任何从库，存在数据丢失风险

### 半同步复制（Semisynchronous Replication）

- 主库在事务提交后，等待**至少一个从库**确认收到并写入 Relay Log 后才返回客户端
- 从库的确认时机：事件写入 Relay Log 并刷盘（fsync）后
- **优点**：保证已提交的事务至少存在于两个节点，数据安全性显著提升
- **缺点**：写入延迟增加至少一个 TCP 往返时间（RTT）
- **适用场景**：同城或低延迟网络环境；跨地域高延迟网络不推荐

半同步配置要点：

```ini
# 主库
[mysqld]
plugin-load-add = "semisync_source.so"
rpl_semi_sync_source_enabled = ON
rpl_semi_sync_source_timeout = 10000  # 超时(ms)，超时后降级为异步

# 从库
[mysqld]
plugin-load-add = "semisync_replica.so"
rpl_semi_sync_replica_enabled = ON
```

> **注意**：如果超时发生且无从库确认，主库会自动降级为异步复制；当至少一个半同步从库追赶上来后，自动恢复半同步模式[^4]。

[^4]: https://dev.mysql.com/doc/refman/8.4/en/replication-semisync.html

### 组复制（Group Replication）

- 基于 Paxos 协议的完全同步复制
- 所有节点收到并确认事务后才提交
- 提供自动故障转移和节点自动加入能力
- 适用于需要强一致性和高可用的场景

## GTID 复制

### 什么是 GTID

GTID（Global Transaction Identifier）是全局事务标识符，格式为 `source_uuid:transaction_id`，例如 `3E11FA47-71CA-11E1-9E33-C80AA9429562:23`。

### GTID 复制的优势

| 对比项 | 传统 binlog 位置复制 | GTID 复制 |
|--------|---------------------|-----------|
| 配置方式 | 需要指定 binlog 文件名和位置 | 只需指定主库连接信息，自动定位 |
| 故障转移 | 需手动查找各从库的 binlog 位置 | 自动识别已执行的事务 |
| 一致性保证 | 依赖人工确认 | 只要主库提交的事务在从库都已应用，即保证一致 |

### 开启 GTID

```ini
[mysqld]
gtid_mode = ON
enforce_gtid_consistency = ON
```

> **限制**：GTID 模式下不支持 `CREATE TABLE ... SELECT` 语句、不支持在一个事务中同时操作事务表和非事务表（如 MyISAM）[^5]。

[^5]: https://dev.mysql.com/doc/refman/8.4/en/replication-gtids-restrictions.html

## 读写分离架构

### 架构模式

```
                    ┌──────────┐
   写请求 ─────────▶│  主库     │
                    │ (Source) │
                    └────┬─────┘
                         │ binlog
              ┌──────────┼──────────┐
              ▼          ▼          ▼
         ┌────────┐ ┌────────┐ ┌────────┐
   读请求▶│ 从库 1 │ │ 从库 2 │ │ 从库 3 │
         │(Replica)│ │(Replica)│ │(Replica)│
         └────────┘ └────────┘ └────────┘
```

### 实现方式

#### 1. 应用层路由

在应用代码中根据 SQL 类型选择数据源：

```java
// 伪代码示例
if (isWriteOperation(sql)) {
    connection = writeDataSource.getConnection();
} else {
    connection = readDataSource.getConnection();
}
```

- **优点**：灵活可控，可定制路由策略
- **缺点**：侵入业务代码，维护成本高

#### 2. 中间件代理

| 中间件 | 类型 | 特点 |
|--------|------|------|
| **ProxySQL** | 独立代理 | 高性能，支持查询规则、连接池、故障自动切换 |
| **ShardingSphere-Proxy** | 分布式数据库中间件 | 支持分库分表、读写分离、分布式事务 |
| **MySQL Router** | 官方轻量代理 | 配合 InnoDB Cluster 使用，自动路由 |
| **MaxScale** | MariaDB 官方代理 | 支持读写分离、分片、过滤 |

#### 3. ORM / 框架层

- MyBatis-Plus 动态数据源
- Spring AbstractRoutingDataSource
- Hibernate Multi-Tenancy

### ProxySQL 读写分离示例

```sql
-- 添加主库（hostgroup 10 = 写）
INSERT INTO mysql_servers (hostgroup_id, hostname, port) VALUES (10, 'master', 3306);

-- 添加从库（hostgroup 20 = 读）
INSERT INTO mysql_servers (hostgroup_id, hostname, port) VALUES (20, 'slave1', 3306);
INSERT INTO mysql_servers (hostgroup_id, hostname, port) VALUES (20, 'slave2', 3306);

-- 配置路由规则：SELECT 走读库组，其他走写库组
INSERT INTO mysql_query_rules (rule_id, active, match_pattern, destination_hostgroup, apply)
VALUES (1, 1, '^SELECT', 20, 1);
INSERT INTO mysql_query_rules (rule_id, active, match_pattern, destination_hostgroup, apply)
VALUES (2, 1, '.*', 10, 1);

LOAD MYSQL SERVERS TO RUNTIME;
SAVE MYSQL SERVERS TO DISK;
LOAD MYSQL QUERY RULES TO RUNTIME;
SAVE MYSQL QUERY RULES TO DISK;
```

## 常见问题与解决方案

### 1. 复制延迟（Replication Lag）

**现象**：从库数据落后于主库，`Seconds_Behind_Master` 值较大。

**原因**：
- 主库写入量大，从库单线程回放跟不上（MySQL 5.6 之前）
- 从库硬件配置低于主库
- 大事务（如大批量 DELETE/UPDATE）阻塞回放
- 网络延迟

**解决方案**：
- 启用多线程复制（MTS）：`replica_parallel_workers = 4`（或更高）
- 使用 `slave_parallel_type = LOGICAL_CLOCK`（基于逻辑时钟的并行）
- 从库硬件配置不低于主库（尤其是磁盘 I/O）
- 拆分大事务为多个小事务
- 监控 `SHOW REPLICA STATUS` 中的 `Seconds_Behind_Master` 和 `Relay_Log_Space`

### 2. 主从数据不一致

**现象**：从库查询结果与主库不一致。

**常见原因**：
- 使用了非确定性函数（`NOW()`, `RAND()`, `UUID()`）配合 STATEMENT 格式
- 在从库上执行了写操作（人为误操作）
- 主从 `sql_mode` 不一致导致行为差异

**解决方案**：
- 使用 ROW 格式（默认），避免非确定性问题
- 从库设置 `read_only = ON`，防止误写
- 确保主从 `sql_mode` 一致
- 定期使用 `pt-table-checksum`（Percona Toolkit）校验数据一致性
- 发现不一致时使用 `pt-table-sync` 修复

### 3. 复制中断（SQL Thread Error）

**现象**：`SHOW REPLICA STATUS` 显示 `Last_SQL_Error`，`Replica_SQL_Running = No`。

**常见错误**：
- `Duplicate entry`：主键冲突
- `Error ... on query`：从库缺少对应表或列

**解决方案**：
- 确认从库是否有人为写入，如有需修复数据后重启复制
- 跳过错误事务（谨慎使用）：
  ```sql
  -- 基于 GTID 跳过
  SET SESSION gtid_next = '出错的事务GTID';
  BEGIN; COMMIT;
  SET SESSION gtid_next = 'AUTOMATIC';
  START REPLICA;
  
  -- 非 GTID 模式跳过
  SET GLOBAL sql_replica_skip_counter = 1;
  START REPLICA;
  ```
- 最可靠方式：重新做全量快照 + 重建复制

### 4. 主库宕机后的故障转移

**问题**：主库宕机后，如何选择一个数据最新的从库提升为新主库？

**方案**：
- **手动切换**：检查各从库的 `Executed_Gtid_Set`，选择 GTID 集合最完整的从库提升
- **半同步 + 自动切换**：配合 Orchestrator、MHA 等工具实现自动选主
- **组复制（Group Replication）**：内置自动选主，无需外部工具

> **注意**：半同步复制下，如果主库宕机且发生了 failover，原主库不应再作为复制源重新使用，因为它可能包含未被任何从库确认的事务[^4]。

### 5. binlog 磁盘空间爆满

**现象**：主库磁盘空间不足，binlog 文件持续增长。

**解决方案**：
- 设置合理的过期时间：`binlog_expire_logs_seconds = 604800`（7 天）
- 定期清理：`PURGE BINARY LOGS BEFORE '2026-04-09 00:00:00';`
- 监控 binlog 生成速率，异常增长时排查是否有大事务或异常写入

### 6. 读写分离下的"读己之写"问题

**现象**：用户写入数据后立即查询，由于复制延迟，从库尚未同步，查不到刚写入的数据。

**解决方案**：
- **关键查询路由到主库**：在应用层标记"写后首次查询"强制走主库
- **基于 Session 的粘性路由**：同一 Session 内的读写都路由到主库
- **使用半同步复制**：降低延迟窗口
- **ProxySQL 的 `mysql-replication_lag` 阈值**：延迟超过阈值自动将流量切回主库

## 最佳实践

### 架构设计

- 一主多从是读扩展的标准模式，建议从库数量 2-4 个
- 跨机房部署时，至少一个从库与主库同城低延迟（用于半同步）
- 使用 GTID 复制而非传统的 binlog 位置复制
- 从库设置 `read_only = ON` 和 `super_read_only = ON`

### 监控指标

| 指标 | 获取方式 | 告警阈值建议 |
|------|---------|-------------|
| 复制延迟 | `Seconds_Behind_Master` | > 10s |
| I/O 线程状态 | `Replica_IO_Running` | != Yes |
| SQL 线程状态 | `Replica_SQL_Running` | != Yes |
| Relay Log 积压 | `Relay_Log_Space` | 持续增长 |
| 半同步从库数 | `Rpl_semi_sync_source_clients` | = 0 时告警 |

### 版本兼容性

- MySQL 8.4 的复制协议与 8.0 兼容，但与 5.7 存在部分不兼容
- 升级复制拓扑时，建议先升级从库，最后升级主库[^6]
- `binlog_format` 动态修改在活跃事务期间可能导致复制失败

[^6]: https://dev.mysql.com/doc/refman/8.4/en/replication-upgrade.html

## 适用场景

| 场景 | 推荐方案 |
|------|---------|
| 读多写少，需要水平扩展读性能 | 一主多从 + 读写分离 |
| 数据安全性要求高，不能丢数据 | 半同步复制 或 组复制 |
| 需要跨地域数据分发 | 异步复制 + 延迟容忍 |
| 需要在从库上做分析查询（OLAP） | 专用分析从库 + 延迟复制 |
| 自动化高可用 | 组复制 或 Orchestrator + 半同步 |

## 限制与注意事项

- 复制**不能替代备份**：误删操作也会被复制到从库
- 所有写操作必须在主库执行，从库上的写操作会导致数据不一致
- 复制是**异步或半同步**的，不是强同步（除非使用组复制或 NDB Cluster）
- ROW 格式下，无主键表的复制效率极低（从库需全表扫描定位行），所有表都应定义主键
- 大事务会阻塞从库回放，建议将大批量操作拆分为小批次
- 存储过程和触发器的复制行为与 binlog 格式相关，ROW 格式下更安全

## 参考资料

- MySQL 8.4 Replication 官方文档: https://dev.mysql.com/doc/refman/8.4/en/replication.html
- Replication Implementation: https://dev.mysql.com/doc/refman/8.4/en/replication-implementation.html
- Replication Formats: https://dev.mysql.com/doc/refman/8.4/en/replication-formats.html
- Semisynchronous Replication: https://dev.mysql.com/doc/refman/8.4/en/replication-semisync.html
- Replication Threads: https://dev.mysql.com/doc/refman/8.4/en/replication-threads.html
- Troubleshooting Replication: https://dev.mysql.com/doc/refman/8.4/en/replication-problems.html
- GTID Restrictions: https://dev.mysql.com/doc/refman/8.4/en/replication-gtids-restrictions.html
- Upgrading Replication Topology: https://dev.mysql.com/doc/refman/8.4/en/replication-upgrade.html
