---
created: '2026-04-16'
updated: '2026-04-16'
tags:
  - mysql
  - schema-migration
  - online-ddl
  - production
  - database-operations
aliases:
  - MySQL Online DDL
  - MySQL Schema Migration
  - 生产环境 MySQL DDL
  - gh-ost
  - pt-online-schema-change
source_type: mixed
source_urls:
  - 'https://dev.mysql.com/doc/refman/8.4/en/innodb-online-ddl.html'
  - 'https://dev.mysql.com/doc/refman/8.4/en/innodb-online-ddl-operations.html'
  - 'https://dev.mysql.com/doc/refman/8.4/en/innodb-online-ddl-performance.html'
  - 'https://dev.mysql.com/doc/refman/8.4/en/innodb-online-ddl-limitations.html'
  - 'https://github.com/github/gh-ost'
  - >-
    https://www.percona.com/doc/percona-toolkit/LATEST/pt-online-schema-change.html
status: verified
---

## 是什么

在生产环境中调整 MySQL 表的数据结构（Schema Migration / DDL），是指在**不中断或最小化中断**业务服务的前提下，对已有表执行 `ALTER TABLE` 等结构变更操作。常见的变更包括：

- 添加 / 删除 / 重命名列
- 添加 / 删除 / 修改索引
- 修改列的数据类型、默认值、NULL 约束
- 更改表的存储引擎、ROW_FORMAT、字符集
- 添加 / 删除外键约束

传统 MySQL 在执行 `ALTER TABLE` 时会**锁表并重建整个表**，对大表而言可能导致数分钟甚至数小时的不可用。现代 MySQL 提供了多种机制来规避这一问题。

## 为什么重要

- **可用性**：生产环境通常要求 7×24 可用，锁表变更直接违反 SLO。
- **性能影响**：表重建会消耗大量 I/O、CPU 和磁盘空间，影响同一实例上的其他业务。
- **复制延迟**：DDL 在主库执行完毕后才会传播到从库，大表的 DDL 可能导致主从延迟显著增加。
- **回滚成本**：DDL 一旦开始，中途失败的回滚代价很高。

## 核心方案概览

| 方案 | 适用场景 | 是否锁表 | 是否需要触发器 | 是否支持暂停 |
| :--- | :--- | :--- | :--- | :--- |
| MySQL Online DDL（INSTANT） | 加列、删列、改默认值等轻量操作 | 否（仅短暂元数据锁） | 否 | 否 |
| MySQL Online DDL（INPLACE） | 加索引、改 VARCHAR 长度等 | 否（可并发 DML） | 否 | 否 |
| MySQL Online DDL（COPY） | 改列类型等不支持 INPLACE 的操作 | 是（锁表） | 否 | 否 |
| gh-ost | 大表变更，需要精细控制 | 否 | 否（基于 binlog） | 是 |
| pt-online-schema-change | 大表变更，成熟稳定 | 否 | 是 | 是（通过负载阈值） |

## 方案一：MySQL 原生 Online DDL

MySQL 5.6 引入 Online DDL，MySQL 8.0 进一步扩展，**MySQL 8.4 默认使用 `ALGORITHM=INSTANT`**。

### ALGORITHM 三种模式

| 模式 | 行为 | 并发 DML | 磁盘空间开销 |
| :--- | :--- | :--- | :--- |
| `INSTANT` | 仅修改数据字典元数据，瞬时完成 | 允许 | 无 |
| `INPLACE` | 在原表上就地修改，避免完整复制 | 取决于 LOCK 子句 | 少量（日志文件） |
| `COPY` | 创建临时表、复制数据、替换原表 | 不允许 | 约等于原表大小 |

语法示例：

```sql
ALTER TABLE tbl_name ADD COLUMN new_col INT, ALGORITHM=INSTANT;
ALTER TABLE tbl_name ADD INDEX idx_col(col), ALGORITHM=INPLACE, LOCK=NONE;
```

### LOCK 子句控制并发

| LOCK 值 | 允许的并发操作 | 典型场景 |
| :--- | :--- | :--- |
| `NONE` | 允许并发查询和 DML | 客户注册、交易类表 |
| `SHARED` | 允许并发查询，阻塞 DML | 数据仓库表 |
| `DEFAULT` | 尽可能多的并发（省略 LOCK 即等效） | 一般场景 |
| `EXCLUSIVE` | 阻塞所有并发查询和 DML | 追求最快完成速度 |

### 常见操作的 Online DDL 支持情况

**支持 INSTANT（瞬时完成）的操作：**

- 添加列（`ADD COLUMN`）—— MySQL 8.4 默认算法
- 删除列（`DROP COLUMN`）—— MySQL 8.4 默认算法
- 重命名列（`CHANGE`，保持数据类型不变）
- 设置 / 删除列的默认值
- 修改 ENUM / SET 列的定义（在末尾追加成员，且存储大小不变）
- 添加虚拟生成列（`VIRTUAL`，非分区表）
- 删除虚拟生成列
- 重命名表

**支持 INPLACE（不重建表或就地重建）的操作：**

- 添加 / 删除 / 重命名二级索引
- 添加主键（重建表，但比 COPY 高效）
- 删除主键并添加新主键
- 扩展 VARCHAR 列大小（长度字节数不变的前提下）
- 修改列的 NULL / NOT NULL 属性（重建表）
- 添加 / 删除外键约束
- 更改 ROW_FORMAT、KEY_BLOCK_SIZE
- 更改字符集（若新编码不同则重建表）

**仅支持 COPY（必须锁表重建）的操作：**

- 修改列的数据类型（如 INT → BIGINT）
- 缩小 VARCHAR 列大小
- 添加 STORED 生成列
- 从 VARCHAR(255) 扩展到 VARCHAR(256)（长度字节从 1 变为 2）

### INSTANT 算法的限制

使用 `ALGORITHM=INSTANT` 添加或删除列时，以下限制适用：

- 不能与不支持 INSTANT 的其他 ALTER 操作组合在同一语句中
- 不适用于 `ROW_FORMAT=COMPRESSED` 的表
- 不适用于有 FULLTEXT 索引的表
- 不适用于数据字典表空间中的表
- 不适用于临时表（临时表仅支持 `ALGORITHM=COPY`）
- 添加列后行大小不能超过最大允许值（ERROR 4092）
- 表的内部表示列数不能超过 1022（ERROR 4158）
- 不能对系统 schema 表（如 `mysql` 内部表）使用

**行版本限制**：每次 INSTANT 添加或删除列会创建一个新的行版本。最大允许 64 个行版本（MySQL 9.1.0 起为 255）。达到上限后，必须使用 `COPY` 或 `INPLACE` 重建表来重置行版本计数。可通过 `INFORMATION_SCHEMA.INNODB_TABLES.TOTAL_ROW_VERSIONS` 查看。

### Online DDL 的三个阶段

1. **初始化阶段**：确定允许的并发级别，获取共享可升级元数据锁
2. **执行阶段**：准备并执行语句，可能短暂获取排他元数据锁
3. **提交表定义阶段**：升级元数据锁为排他，提交新表定义

> **关键注意**：Online DDL 在执行阶段和提交阶段需要排他元数据锁，因此会等待持有该表元数据锁的长事务提交或回滚。如果存在长时间未提交的事务，DDL 可能超时。

### 监控 DDL 进度

```sql
-- 查看当前进程状态
SHOW FULL PROCESSLIST;

-- 通过 Performance Schema 监控 ALTER TABLE 进度
SELECT EVENT_NAME, WORK_COMPLETED, WORK_ESTIMATED
FROM performance_schema.events_stages_current
WHERE EVENT_NAME LIKE 'stage/innodb/alter%';

-- 查看元数据锁等待情况
SELECT * FROM performance_schema.metadata_locks
WHERE OBJECT_SCHEMA = 'your_db' AND OBJECT_NAME = 'your_table';
```

## 方案二：gh-ost（GitHub Online Schema Transmogrifier）

[gh-ost](https://github.com/github/gh-ost) 是 GitHub 开发的**无触发器**在线表结构迁移工具，通过解析 binlog 来捕获表变更。

### 核心原理

1. 创建一个结构类似的 ghost 表
2. 批量从原表复制数据到 ghost 表
3. 通过 binlog 异步回放原表的增量变更（INSERT / DELETE / UPDATE）
4. 在合适时机原子性地替换原表

### 相比触发器方案的优势

- **无触发器**：避免触发器带来的性能损耗和限制
- **真正暂停**：节流（throttle）时完全停止写入主库
- **动态控制**：迁移过程中可交互式重新配置
- **可审计**：可通过 UNIX socket 或 TCP 查询状态
- **可控的 cut-over**：可推迟表交换时机，避免在不可控时间窗口执行
- **可在从库测试**：先在从库上执行迁移验证，确认无误后再在主库执行

### 基本用法

```bash
# 空跑验证（不实际执行）
gh-ost \
  --user="migrate_user" --password="password" \
  --host=master-host --port=3306 \
  --database="mydb" --table="my_table" \
  --alter="ADD COLUMN new_col INT" \
  --allow-on-master \
  --dry-run

# 实际执行
gh-ost \
  --user="migrate_user" --password="password" \
  --host=master-host --port=3306 \
  --database="mydb" --table="my_table" \
  --alter="ADD COLUMN new_col INT" \
  --allow-on-master \
  --execute \
  --exact-rowcount \
  --postpone-cut-over-flag-file=/tmp/gh-ost-postpone
```

### 关键参数

| 参数 | 说明 |
| :--- | :--- |
| `--alter` | 表结构变更语句（不含 ALTER TABLE 关键字） |
| `--execute` | 实际执行迁移（默认仅做安全检查） |
| `--dry-run` | 空跑验证，不创建触发器、不复制数据 |
| `--allow-on-master` | 允许直接在主库上执行 |
| `--test-on-replica` | 在从库上测试迁移（不替换原表） |
| `--exact-rowcount` | 精确计算行数，进度指示更准确 |
| `--postpone-cut-over-flag-file` | 推迟表交换，直到删除指定文件 |
| `--max-load` | 负载阈值，超过则暂停 |
| `--critical-load` | 临界负载，超过则中止 |

### 要求与限制

- 需要开启 binlog 且格式为 ROW
- 表必须有 PRIMARY KEY 或 UNIQUE INDEX
- 不支持外键（有外键的表需特殊处理）
- 不支持 MyISAM 引擎
- 在 Amazon RDS 上需要额外配置（参考 [gh-ost RDS 文档](https://github.com/github/gh-ost/blob/master/doc/rds.md)）

## 方案三：pt-online-schema-change（Percona Toolkit）

[pt-online-schema-change](https://www.percona.com/doc/percona-toolkit/LATEST/pt-online-schema-change.html) 是 Percona Toolkit 中的在线表结构变更工具，使用**触发器**机制捕获变更。

### 核心原理

1. 创建空的新表（结构与原表相同）
2. 在新表上执行所需的 ALTER 操作
3. 在原表上创建 INSERT / DELETE / UPDATE 触发器，将变更同步到新表
4. 以小批量方式从原表复制数据到新表
5. 使用原子 RENAME TABLE 交换两表
6. 默认删除原表

### 基本用法

```bash
# 空跑验证
pt-online-schema-change \
  --alter "ADD COLUMN new_col INT" \
  D=sakila,t=actor \
  --dry-run

# 实际执行
pt-online-schema-change \
  --alter "ADD COLUMN new_col INT" \
  D=sakila,t=actor \
  --execute \
  --max-lag=1s \
  --max-load="Threads_running=25" \
  --critical-load="Threads_running=50"
```

### 关键参数

| 参数 | 说明 |
| :--- | :--- |
| `--alter` | 表结构变更语句（不含 ALTER TABLE 关键字） |
| `--execute` | 实际执行（默认仅做安全检查） |
| `--dry-run` | 空跑验证 |
| `--max-lag` | 最大允许的从库延迟（默认 1s） |
| `--max-load` | 负载阈值，超过则暂停（默认 Threads_running=25） |
| `--critical-load` | 临界负载，超过则中止（默认 Threads_running=50） |
| `--chunk-time` | 每批数据复制的目标执行时间（默认 0.5s） |
| `--chunk-size` | 每批复制的行数（默认 1000） |
| `--alter-foreign-keys-method` | 处理外键的方法：auto / rebuild_constraints / drop_swap / none |
| `--preserve-triggers` | 保留原表的触发器（MySQL 5.7.2+ 支持） |

### 限制

- **原表不能有触发器**（除非使用 `--preserve-triggers` 且 MySQL ≥ 5.7.2）
- 表必须有 PRIMARY KEY 或 UNIQUE INDEX
- 不支持 MyISAM 表在 Percona XtraDB Cluster 上
- 如果设置了复制过滤器，默认会中止
- 添加 UNIQUE 索引时可能因 `INSERT IGNORE` 导致数据丢失（工具会检测并阻止）

## 方案选择决策树

```
变更类型是什么？
├── 加列 / 删列 / 改默认值 / 重命名列
│   └── 优先使用 MySQL Online DDL（ALGORITHM=INSTANT）
│       └── 检查是否满足 INSTANT 限制条件
│           ├── 满足 → 直接执行
│           └── 不满足 → 考虑 INPLACE 或第三方工具
│
├── 加索引 / 删索引 / 扩 VARCHAR
│   └── 优先使用 MySQL Online DDL（ALGORITHM=INPLACE, LOCK=NONE）
│
├── 改列类型 / 缩 VARCHAR / 其他需要 COPY 的操作
│   └── 表大小？
│       ├── 小表（< 100 万行）→ 可在低峰期直接执行 COPY
│       └── 大表 → 使用 gh-ost 或 pt-online-schema-change
│
└── 大表变更且需要精细控制
    ├── 原表有触发器？
    │   ├── 有 → pt-online-schema-change（--preserve-triggers）或 gh-ost
    │   └── 无 → gh-ost（推荐，无触发器开销）
    └── 需要暂停 / 回滚能力？
        ├── 是 → gh-ost（真正暂停，动态控制）
        └── 否 → pt-online-schema-change（成熟稳定）
```

## 生产环境最佳实践

### 变更前

1. **备份**：变更前确保有完整的备份，并验证可恢复
2. **评估影响**：
   - 检查表大小（行数、数据量）
   - 检查是否有长事务在运行
   - 检查主从复制状态
   - 检查是否有外键依赖
3. **先在测试环境验证**：
   - 使用 `--dry-run` 或 `--test-on-replica` 验证
   - 对比变更前后的表结构和数据
4. **选择低峰期**：即使是不锁表的方案，也会增加额外负载

### 变更中

1. **监控关键指标**：
   - 主从延迟（`Seconds_Behind_Source`）
   - 线程数（`Threads_running`）
   - 磁盘 I/O 使用率
   - 锁等待情况
2. **设置负载阈值**：使用 `--max-load` 和 `--critical-load` 防止过载
3. **控制 cut-over 时机**：使用 `--postpone-cut-over-flag-file` 手动控制表交换
4. **保留原表**：使用 `--no-drop-old-table` 保留原表一段时间以便回滚

### 变更后

1. **验证数据一致性**：对比变更前后的数据
2. **检查索引统计信息**：必要时执行 `ANALYZE TABLE` 更新统计信息
3. **清理资源**：删除不再需要的临时表、触发器
4. **监控业务指标**：确认变更未引入性能问题

### 常见误区

- **误区 1**：Online DDL 完全不锁表。实际上在初始化和提交阶段仍需短暂的排他元数据锁，长事务会阻塞 DDL。
- **误区 2**：INSTANT 加列没有代价。多次 INSTANT 操作会累积行版本，达到上限后必须重建表。
- **误区 3**：第三方工具比 Online DDL 慢。对于需要 COPY 的操作，第三方工具通过分批复制和负载控制，反而对生产影响更小。
- **误区 4**：DDL 在从库上会自动安全执行。从库的 DDL 同样会阻塞复制，且 DML 需等 DDL 完成后才处理。

## 参考资料

- MySQL 8.4 官方文档 — [InnoDB and Online DDL](https://dev.mysql.com/doc/refman/8.4/en/innodb-online-ddl.html)
- MySQL 8.4 官方文档 — [Online DDL Operations](https://dev.mysql.com/doc/refman/8.4/en/innodb-online-ddl-operations.html)
- MySQL 8.4 官方文档 — [Online DDL Performance and Concurrency](https://dev.mysql.com/doc/refman/8.4/en/innodb-online-ddl-performance.html)
- MySQL 8.4 官方文档 — [Online DDL Limitations](https://dev.mysql.com/doc/refman/8.4/en/innodb-online-ddl-limitations.html)
- GitHub — [gh-ost: GitHub's Online Schema-migration Tool for MySQL](https://github.com/github/gh-ost)
- Percona — [pt-online-schema-change 文档](https://www.percona.com/doc/percona-toolkit/LATEST/pt-online-schema-change.html)
