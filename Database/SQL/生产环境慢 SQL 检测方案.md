---
created: '2026-04-16'
updated: '2026-04-16'
tags:
  - 数据库
  - 慢SQL
  - 性能优化
  - MySQL
  - PostgreSQL
  - 运维
  - 监控
aliases:
  - 生产环境慢 SQL 检测
  - Slow SQL Detection in Production
  - 慢查询检测方案
source_type: mixed
source_urls:
  - 'https://dev.mysql.com/doc/refman/8.4/en/slow-query-log.html'
  - 'https://dev.mysql.com/doc/refman/8.4/en/performance-schema.html'
  - 'https://www.postgresql.org/docs/current/pgstatstatements.html'
  - 'https://www.percona.com/doc/percona-toolkit/LATEST/pt-query-digest.html'
status: verified
---

## 什么是慢 SQL

慢 SQL 指执行时间超过预设阈值的 SQL 语句。在生产环境中，慢 SQL 是导致数据库性能下降、应用响应延迟甚至雪崩的核心原因之一。及时检测和分析慢 SQL 是数据库运维和性能调优的首要步骤。

## 为什么需要专门的检测方案

- **通用查询日志开销过大**：开启后会记录所有 SQL，生产环境不可长期启用
- **慢查询日志有盲区**：仅记录超过阈值的 SQL，无法覆盖"接近阈值"的潜在风险查询
- **日志是事后分析**：无法做到实时告警和自动拦截
- **多数据库环境**：MySQL、PostgreSQL、Oracle 等各自的慢查询机制不同，需要统一方案

## 检测方案总览

| 方案 | 适用数据库 | 实时性 | 性能影响 | 典型用途 |
| :--- | :--- | :--- | :--- | :--- |
| 慢查询日志 | MySQL / PostgreSQL | 事后分析 | 低 | 定位具体慢 SQL |
| Performance Schema | MySQL 5.6+ | 近实时 | 中 | 细粒度性能分析 |
| pg_stat_statements | PostgreSQL | 近实时 | 低 | 聚合 SQL 统计 |
| pt-query-digest | MySQL | 事后分析 | 无（离线分析） | 慢日志汇总分析 |
| Prometheus + Exporter | MySQL / PostgreSQL | 实时 | 低 | 监控告警 |
| APM 工具（SkyWalking 等） | 不限 | 实时 | 低-中 | 全链路追踪 |

---

## 一、MySQL 慢查询日志（基础方案）

### 原理

MySQL 内置的 Slow Query Log 记录执行时间超过 `long_query_time` 阈值的 SQL。这是最基础、最常用的慢 SQL 检测手段。

### 配置方法

```ini
[mysqld]
# 开启慢查询日志
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log

# 慢查询阈值（秒），支持微秒精度
long_query_time = 1

# 记录未使用索引的查询（即使未超阈值）
log_queries_not_using_indexes = 1

# 限制每分钟记录无索引查询的数量，防止日志膨胀
log_throttle_queries_not_using_indexes = 10

# 记录管理语句（ALTER TABLE、ANALYZE TABLE 等）
log_slow_admin_statements = 1

# 记录额外字段（Thread_id、Bytes_sent、临时表信息等）
log_slow_extra = 1
```

运行时动态开启（无需重启）：

```sql
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;
SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';
```

### 日志格式

```
# Time: 2026-04-16T10:30:00.000000Z
# User@Host: root[root] @ localhost []  Thread_id: 10
# Query_time: 12.345678  Lock_time: 0.000123  Rows_sent: 100  Rows_examined: 5000000
SET timestamp=1713265800;
SELECT * FROM large_table WHERE status = 'active';
```

开启 `log_slow_extra` 后额外包含：`Thread_id`、`Errno`、`Bytes_received`、`Bytes_sent`、`Created_tmp_disk_tables`、`Start`、`End` 等。

### 分析工具

```bash
# mysqldumpslow 汇总慢查询日志
mysqldumpslow -s t -t 10 /var/log/mysql/slow.log
# -s t: 按执行时间排序（其他选项：c=调用次数, l=锁定时间, r=返回行数）
# -t 10: 显示前 10 条
```

### 注意事项

- 获取锁的时间**不计入** `Query_time`（`Lock_time` 单独记录）
- 日志在语句执行完成且锁释放后才写入，日志顺序可能与执行顺序不同
- 密码字段会被自动重写，语法错误的语句不会被记录
- 从库默认不记录复制来的慢查询，需开启 `log_slow_replica_statements`
- ROW 格式复制下，`log_slow_replica_statements` 无效（从库收到的是行变更而非 SQL）

> 参考：[MySQL 8.4 - The Slow Query Log](https://dev.mysql.com/doc/refman/8.4/en/slow-query-log.html)

---

## 二、MySQL Performance Schema（进阶方案）

### 原理

Performance Schema 是 MySQL 5.5+ 引入的性能诊断框架，通过 instrument（探针）收集服务器运行时数据。相比慢查询日志，它能提供**更细粒度的实时性能数据**，包括：

- 每条语句的执行时间分布
- 等待事件（锁、I/O、网络等）
- 语句执行阶段的耗时分解

### 核心表

| 表名 | 用途 |
| :--- | :--- |
| `performance_schema.events_statements_current` | 当前正在执行的语句 |
| `performance_schema.events_statements_history` | 最近执行的语句历史（可配置保留条数） |
| `performance_schema.events_statements_summary_by_digest` | 按 SQL 指纹聚合的统计信息 |

### 检测慢 SQL 的典型查询

```sql
-- 查看当前正在执行的慢语句
SELECT
  THREAD_ID,
  EVENT_ID,
  SQL_TEXT,
  TIMER_WAIT / 1000000000 AS exec_time_ms,
  LOCK_TIME / 1000000000 AS lock_time_ms,
  ROWS_EXAMINED,
  ROWS_SENT
FROM performance_schema.events_statements_current
WHERE TIMER_WAIT > 1000000000000  -- 超过 1 秒（单位：皮秒）
ORDER BY TIMER_WAIT DESC;

-- 按 SQL 指纹聚合，找出平均执行时间最高的语句
SELECT
  DIGEST_TEXT AS query_pattern,
  COUNT_STAR AS exec_count,
  AVG_TIMER_WAIT / 1000000000 AS avg_time_ms,
  MAX_TIMER_WAIT / 1000000000 AS max_time_ms,
  SUM_ROWS_EXAMINED AS total_rows_examined,
  SUM_ROWS_SENT AS total_rows_sent
FROM performance_schema.events_statements_summary_by_digest
ORDER BY AVG_TIMER_WAIT DESC
LIMIT 20;
```

### 配置

Performance Schema 默认在 MySQL 5.6+ 中启用。可通过以下变量调整：

```sql
-- 查看是否启用
SHOW VARIABLES LIKE 'performance_schema';

-- 调整历史表保留条数（默认 10）
UPDATE performance_schema.setup_consumers
SET ENABLED = 'YES'
WHERE NAME = 'events_statements_history';

-- 调整聚合表大小
-- 在 my.cnf 中配置：
-- performance_schema_digests_size = 10000
```

### 优势与限制

| 优势 | 限制 |
| :--- | :--- |
| 实时查询，无需等待日志写入 | 有一定性能开销（通常 < 5%） |
| 提供 SQL 指纹聚合，自动归类相似语句 | 服务器重启后历史数据丢失 |
| 可分解语句各阶段耗时 | 需要 SUPER 或 PERFORMANCE_SCHEMA 权限 |

> 参考：[MySQL 8.4 - Performance Schema](https://dev.mysql.com/doc/refman/8.4/en/performance-schema.html)

---

## 三、PostgreSQL pg_stat_statements（等效方案）

### 原理

`pg_stat_statements` 是 PostgreSQL 的官方扩展模块，跟踪所有 SQL 语句的执行统计信息。功能上等同于 MySQL 的 Performance Schema 聚合表 + 慢查询日志的组合。

### 安装与配置

```ini
# postgresql.conf
shared_preload_libraries = 'pg_stat_statements'

compute_query_id = on
pg_stat_statements.max = 10000
pg_stat_statements.track = all        # top=仅顶层语句, all=含嵌套语句
pg_stat_statements.track_planning = off  # 是否跟踪规划时间
pg_stat_statements.save = on          # 重启后保留统计
```

> **注意**：修改 `shared_preload_libraries` 后需要**重启 PostgreSQL 服务**。

```sql
-- 在目标数据库中创建扩展
CREATE EXTENSION pg_stat_statements;
```

### 检测慢 SQL 的典型查询

```sql
-- 按总执行时间排序，找出最耗时的 SQL
SELECT
  query,
  calls,
  total_exec_time,
  mean_exec_time,
  max_exec_time,
  rows,
  100.0 * shared_blks_hit / NULLIF(shared_blks_hit + shared_blks_read, 0) AS hit_percent
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;

-- 按平均执行时间排序，找出单次执行最慢的 SQL
SELECT
  query,
  calls,
  mean_exec_time,
  max_exec_time,
  rows
FROM pg_stat_statements
WHERE calls > 10  -- 排除只执行过几次的偶然慢查询
ORDER BY mean_exec_time DESC
LIMIT 20;

-- 重置某个特定 SQL 的统计
SELECT pg_stat_statements_reset(0, 0, queryid)
FROM pg_stat_statements
WHERE query LIKE '%problematic_table%';

-- 重置所有统计
SELECT pg_stat_statements_reset();
```

### 核心字段说明

| 字段 | 含义 |
| :--- | :--- |
| `calls` | 执行次数 |
| `total_exec_time` | 总执行时间（毫秒） |
| `mean_exec_time` | 平均执行时间（毫秒） |
| `max_exec_time` | 最大执行时间（毫秒） |
| `shared_blks_hit` | 共享缓冲区命中次数 |
| `shared_blks_read` | 从磁盘读取的共享块数 |
| `hit_percent` | 缓存命中率（自行计算） |

### 注意事项

- 仅 superuser 或拥有 `pg_read_all_stats` 角色的用户可查看其他用户的 SQL 文本
- 相似 SQL（仅常量不同）会被归并为一条记录，常量替换为 `$1`、`$2` 等参数符号
- 统计条目上限由 `pg_stat_statements.max` 控制，超出时淘汰执行最少的语句
- `queryid` 在 PostgreSQL 小版本间稳定，但**不保证跨大版本稳定**

> 参考：[PostgreSQL 18 - pg_stat_statements](https://www.postgresql.org/docs/current/pgstatstatements.html)

---

## 四、pt-query-digest（离线分析方案）

### 原理

`pt-query-digest` 是 Percona Toolkit 中的慢查询日志分析工具，能对 MySQL 慢日志进行深度分析，生成按指纹聚合的报告，包含执行时间分布、调用次数、影响行数等统计。

### 安装

```bash
# Ubuntu/Debian
apt-get install percona-toolkit

# CentOS/RHEL
yum install percona-toolkit

# 或直接下载
wget https://www.percona.com/downloads/percona-toolkit/LATEST/percona-toolkit-LATEST.tar.gz
```

### 基本用法

```bash
# 分析慢查询日志文件
pt-query-digest /var/log/mysql/slow.log

# 分析最近 12 小时的慢查询
pt-query-digest --since 12h /var/log/mysql/slow.log

# 分析特定用户的慢查询
pt-query-digest --filter '$event->{User} =~ m/^admin/' /var/log/mysql/slow.log

# 输出到文件
pt-query-digest /var/log/mysql/slow.log > slow_report.txt

# 分析 Performance Schema 中的数据
pt-query-digest --type=pqs --review h=localhost,D=percona,t=query_review
```

### 报告内容示例

```
# Profile
# Rank Query ID           Response time  Calls  R/Call  V/M   Item
# ==== ================== ============== ====== ======= ===== ========
#    1 0x1234567890ABCDEF  120.5000 50.2%   1000  0.1205  0.01 SELECT users
#    2 0xFEDCBA0987654321   80.3000 33.5%    500  0.1606  0.02 SELECT orders
...

# Query 1: 0.1200s avg, 1000 queries
# SELECT * FROM users WHERE email = ?
```

### 优势

- 自动按 SQL 指纹聚合，识别同类查询
- 提供响应时间分布、调用频率、锁等待等详细指标
- 支持多种输入源：慢日志、Performance Schema、tcpdump 抓包
- 可输出到数据库表，便于长期追踪

> 参考：[Percona Toolkit - pt-query-digest](https://www.percona.com/doc/percona-toolkit/LATEST/pt-query-digest.html)

---

## 五、Prometheus + Exporter（实时监控方案）

### 架构

```
MySQL/PostgreSQL → Exporter → Prometheus → Grafana / AlertManager
```

### MySQL 方案

使用 `mysqld_exporter`（Prometheus 官方维护）：

```bash
# 启动 exporter
mysqld_exporter --config.my-cnf /etc/mysql_exporter.cnf

# /etc/mysql_exporter.cnf
[client]
user=exporter
password=xxx
```

关键监控指标：

| 指标 | 含义 | 告警建议 |
| :--- | :--- | :--- |
| `mysql_global_status_slow_queries` | 慢查询累计数 | 增长率突增时告警 |
| `mysql_info_schema_processlist_threads` | 当前连接数 | 超过阈值告警 |
| `mysql_global_status_threads_running` | 活跃线程数 | 持续高位告警 |

### PostgreSQL 方案

使用 `postgres_exporter`：

关键监控指标：

| 指标 | 含义 |
| :--- | :--- |
| `pg_stat_activity_max_tx_duration` | 最长事务持续时间 |
| `pg_stat_statements_calls` | SQL 调用次数 |
| `pg_stat_statements_mean_exec_time` | SQL 平均执行时间 |

### Grafana 面板

推荐使用社区成熟模板：

- MySQL: [Percona Monitoring and Management (PMM)](https://github.com/percona/grafana-dashboards)
- PostgreSQL: [PostgreSQL Overview](https://grafana.com/grafana/dashboards/9628)

---

## 六、APM 全链路追踪（应用层方案）

### 原理

APM（Application Performance Monitoring）工具通过在应用层注入探针，追踪 SQL 从发起到返回的完整链路，能精确定位慢 SQL 发生在哪个环节。

### 主流方案

| 工具 | 语言支持 | 特点 |
| :--- | :--- | :--- |
| **SkyWalking** | Java / Go / Python / Node.js | Apache 开源，支持 SQL 解析和慢查询告警 |
| **Pinpoint** | Java / PHP | 字节码注入，零代码侵入 |
| **Jaeger** | 多语言 | CNCF 项目，专注分布式追踪 |
| **Datadog APM** | 多语言 | 商业产品，开箱即用 |

### SkyWalking 慢 SQL 检测示例

SkyWalking 通过 JDBC 插件自动采集 SQL 执行时间：

1. 在 `agent.config` 中配置慢 SQL 阈值：
   ```properties
   plugin.jdbc.trace_sql_parameters=true
   plugin.jdbc.slow_sql_threshold=1000  # 毫秒
   ```

2. 慢 SQL 会在 SkyWalking UI 的 "Database" 面板中展示：
   - SQL 语句（参数已脱敏）
   - 执行时间分布（P50/P90/P99）
   - 调用拓扑（哪个服务发起的）

### 优势与限制

| 优势 | 限制 |
| :--- | :--- |
| 关联应用层上下文（哪个接口、哪个用户触发） | 需要部署 Agent，有一定侵入性 |
| 支持分布式追踪（跨服务 SQL 调用链） | 商业方案有成本 |
| 自动参数脱敏，安全性好 | 无法覆盖非应用层发起的 SQL（如 DBA 手动执行） |

---

## 七、生产环境推荐方案组合

### 最小可用方案

```
慢查询日志 + mysqldumpslow / pt-query-digest
```

- 适合：小型团队、单数据库实例
- 成本：零额外部署
- 缺点：事后分析，无实时告警

### 标准方案

```
慢查询日志 + Performance Schema / pg_stat_statements + Prometheus + Grafana
```

- 适合：中型团队、多数据库实例
- 能力：实时监控 + 历史分析 + 自动告警
- 成本：需部署监控基础设施

### 完整方案

```
标准方案 + APM（SkyWalking 等）+ 自动化慢 SQL 拦截
```

- 适合：大型团队、微服务架构
- 能力：全链路追踪 + 应用层关联 + 自动拦截
- 成本：较高，需要专门运维团队

---

## 八、常见问题与最佳实践

### 慢查询阈值设多少合适？

- **经验值**：1 秒（`long_query_time = 1`）
- **核心业务**：500ms 或更低
- **批处理任务**：可适当放宽到 5-10 秒
- 建议根据业务的 P99 响应时间反推

### 慢查询日志会不会影响性能？

- 慢查询日志仅在 SQL 执行完成后追加写入，性能影响**极低**（通常 < 1%）
- 真正影响性能的是**通用查询日志**（记录所有 SQL），生产环境不应开启
- 开启 `log_queries_not_using_indexes` 后日志可能快速增长，建议配合 `log_throttle_queries_not_using_indexes` 限流

### 如何避免慢查询日志膨胀？

1. 设置合理的 `long_query_time`，不要设为 0（除非短期调试）
2. 开启 `log_throttle_queries_not_using_indexes` 限制无索引查询的写入频率
3. 定期轮转日志文件（`FLUSH LOGS` 或 logrotate）
4. 监控日志文件大小，设置告警

### 从库的慢 SQL 怎么检测？

- MySQL ROW 格式复制下，从库不会将复制的查询写入慢查询日志
- 如需检测从库自身的慢查询（如从库上的报表查询），在从库上独立开启慢查询日志
- PostgreSQL 的 `pg_stat_statements` 在从库上同样可用（需单独安装扩展）

### 如何自动告警？

```yaml
# Prometheus 告警规则示例
groups:
  - name: slow-sql
    rules:
      - alert: HighSlowQueryRate
        expr: rate(mysql_global_status_slow_queries[5m]) > 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "慢查询速率异常 ({{ $value }}/s)"
```

### 慢 SQL 分析流程

1. **发现**：通过监控告警或日志分析发现慢 SQL
2. **定位**：使用 `EXPLAIN` 分析执行计划
3. **优化**：添加索引、改写 SQL、调整表结构
4. **验证**：在测试环境验证优化效果
5. **上线**：灰度发布，持续监控

---

## 九、相关概念

- [[MySQL 日志类型与作用]]：MySQL 各类日志的完整说明
- [[MySQL 的 Log 类型详解]]：MySQL 日志系统详细解析
- [[数据库事务的隔离级别]]：事务隔离与性能的关系

---

## 参考资料

- [MySQL 8.4 - The Slow Query Log](https://dev.mysql.com/doc/refman/8.4/en/slow-query-log.html)
- [MySQL 8.4 - Performance Schema](https://dev.mysql.com/doc/refman/8.4/en/performance-schema.html)
- [PostgreSQL 18 - pg_stat_statements](https://www.postgresql.org/docs/current/pgstatstatements.html)
- [Percona Toolkit - pt-query-digest](https://www.percona.com/doc/percona-toolkit/LATEST/pt-query-digest.html)
- [Prometheus mysqld_exporter](https://github.com/prometheus/mysqld_exporter)
- [Apache SkyWalking](https://skywalking.apache.org/)
