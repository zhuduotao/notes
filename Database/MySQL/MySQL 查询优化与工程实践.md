---
created: 2026-04-16
updated: 2026-04-16
tags:
  - mysql
  - 查询优化
  - 数据库
  - 性能调优
  - 索引
  - EXPLAIN
  - 工程实践
aliases:
  - MySQL Query Optimization
  - MySQL 性能优化
  - MySQL 查询调优
source_type: official-doc
source_urls:
  - https://dev.mysql.com/doc/refman/8.4/en/optimization.html
  - https://dev.mysql.com/doc/refman/8.4/en/explain-output.html
  - https://dev.mysql.com/doc/refman/8.4/en/optimization-indexes.html
  - https://dev.mysql.com/doc/refman/8.4/en/order-by-optimization.html
  - https://dev.mysql.com/doc/refman/8.4/en/optimizer-hints.html
status: verified
---

## 概述

MySQL 查询优化是一个多层次的工作，涉及 SQL 语句级别、索引设计、数据库结构、存储引擎配置以及服务器参数调优。本文基于 MySQL 8.4 官方文档，系统梳理查询优化的核心机制与工程实践。

> **适用版本**：本文内容主要针对 MySQL 8.0 及以上版本，部分特性（如 Hash Join、Descending Indexes、Optimizer Hints）在 8.0 引入或增强。

---

## 一、使用 EXPLAIN 分析执行计划

`EXPLAIN` 是查询优化的首要工具，用于查看 MySQL 如何执行一条语句。它适用于 `SELECT`、`DELETE`、`INSERT`、`REPLACE` 和 `UPDATE`。

### 1.1 EXPLAIN 输出列

| 列名 | 含义 |
|------|------|
| `id` | SELECT 标识符，查询中 SELECT 的顺序编号 |
| `select_type` | SELECT 类型（SIMPLE、PRIMARY、UNION、SUBQUERY、DERIVED 等） |
| `table` | 输出行所引用的表名 |
| `partitions` | 匹配的分区（非分区表为 NULL） |
| `type` | 连接类型（访问方式），**从最优到最差排列** |
| `possible_keys` | 可能使用的索引列表 |
| `key` | 实际选择的索引 |
| `key_len` | 所选索引的长度（多列索引中实际使用的部分） |
| `ref` | 与索引进行比较的列或常量 |
| `rows` | 估计需要扫描的行数（InnoDB 为估算值） |
| `filtered` | 被表条件过滤的行百分比，100 表示无过滤 |
| `Extra` | 额外信息（Using filesort、Using temporary、Using index 等） |

### 1.2 连接类型（type）—— 从最优到最差

| 类型 | 说明 |
|------|------|
| `system` | 表只有一行（system 表），const 的特例 |
| `const` | 最多一行匹配，通过 PRIMARY KEY 或 UNIQUE 索引与常量比较 |
| `eq_ref` | 对前一表的每一行组合，从本表读取一行。用于 PRIMARY KEY 或 UNIQUE NOT NULL 索引的等值连接 |
| `ref` | 非唯一索引的等值查找，或使用索引的最左前缀 |
| `fulltext` | 使用 FULLTEXT 索引 |
| `ref_or_null` | 类似 ref，但额外搜索 NULL 值 |
| `index_merge` | 使用了 Index Merge 优化 |
| `unique_subquery` | IN 子查询中替代 eq_ref 的索引查找 |
| `index_subquery` | 类似 unique_subquery，但用于非唯一索引 |
| `range` | 索引范围扫描，用于 `=`、`<>`、`>`、`>=`、`<`、`<=`、`IS NULL`、`BETWEEN`、`LIKE`、`IN()` 等 |
| `index` | 全索引扫描（仅扫描索引树），若 Extra 显示 `Using index` 则为覆盖索引 |
| `ALL` | 全表扫描，**应尽量避免**（除非表很小或查询需要访问大部分行） |

### 1.3 关键 Extra 信息

- **`Using index`**：覆盖索引，查询所需的所有列都在索引中，无需回表
- **`Using filesort`**：需要额外的排序操作，**应关注并尽量消除**
- **`Using temporary`**：使用了临时表，**应关注并尽量消除**
- **`Using where`**：使用了 WHERE 条件过滤
- **`Impossible WHERE`**：WHERE 条件永远为假
- **`Not exists`**：LEFT JOIN 优化，找到匹配行后不再扫描

### 1.4 使用方式

```sql
-- 基本用法
EXPLAIN SELECT * FROM t1 WHERE col1 = 'value';

-- JSON 格式（更详细的结构化信息）
EXPLAIN FORMAT=JSON SELECT * FROM t1 WHERE col1 = 'value';

-- TREE 格式（MySQL 8.0+，可读的执行树）
EXPLAIN FORMAT=TREE SELECT * FROM t1 WHERE col1 = 'value';
```

---

## 二、索引优化

### 2.1 索引的基本原理

- 大多数 MySQL 索引（PRIMARY KEY、UNIQUE、INDEX、FULLTEXT）使用 **B-tree** 存储
- 空间数据类型使用 R-tree；MEMORY 表支持 hash 索引；InnoDB 的 FULLTEXT 使用倒排列表
- 索引的本质是"指针"，指向表中的行，使查询能快速定位满足条件的行

### 2.2 最左前缀原则

对于多列索引 `(col1, col2, col3)`，优化器可以使用以下前缀：
- `(col1)`
- `(col1, col2)`
- `(col1, col2, col3)`

但**不能**跳过前缀列直接使用 `(col2)` 或 `(col3)`，也不能跳过中间列使用 `(col1, col3)`。

```sql
-- 假设索引为 (col1, col2, col3)
-- 可以使用索引
SELECT * FROM t WHERE col1 = 1;
SELECT * FROM t WHERE col1 = 1 AND col2 = 2;
SELECT * FROM t WHERE col1 = 1 AND col2 = 2 AND col3 = 3;

-- 不能使用该索引（跳过最左前缀）
SELECT * FROM t WHERE col2 = 2;
SELECT * FROM t WHERE col3 = 3;

-- 只能使用部分索引（跳过中间列）
SELECT * FROM t WHERE col1 = 1 AND col3 = 3;  -- 仅使用 col1 部分
```

### 2.3 覆盖索引（Covering Index）

当查询所需的所有列都包含在索引中时，MySQL 可以直接从索引树获取数据，无需回表查询数据行。

```sql
-- 假设索引为 (key_part1, key_part2, key_part3)
SELECT key_part3 FROM tbl_name WHERE key_part1 = 1;
-- Extra 显示 "Using index"，无需回表
```

对于 InnoDB 表，由于二级索引隐式包含主键值，因此即使查询选择了主键列，二级索引也可能成为覆盖索引。

### 2.4 索引的代价

- 索引占用额外存储空间
- 每次 `INSERT`、`UPDATE`、`DELETE` 都需要更新索引
- 过多的索引会增加优化器选择执行计划的时间
- **需要在查询性能和写入性能之间找到平衡**

### 2.5 特殊索引类型

| 类型 | 说明 | 版本要求 |
|------|------|----------|
| 不可见索引（Invisible Indexes） | 优化器忽略该索引，但索引本身仍被维护。用于安全地测试删除索引的影响 | 8.0+ |
| 降序索引（Descending Indexes） | 索引按降序存储，优化混合 ASC/DESC 的 ORDER BY | 8.0+ |
| 函数索引（Functional Indexes） | 基于表达式或函数创建索引 | 8.0+ |
| 前缀索引 | 仅索引字符串列的前 N 个字符，节省空间但可能降低选择性 | 所有版本 |

```sql
-- 不可见索引
ALTER TABLE t ALTER INDEX idx_name INVISIBLE;
ALTER TABLE t ALTER INDEX idx_name VISIBLE;

-- 降序索引
CREATE INDEX idx_desc ON t (col1 DESC, col2 ASC);

-- 函数索引
CREATE INDEX idx_func ON t ((LOWER(col1)));

-- 前缀索引
CREATE INDEX idx_prefix ON t (col1(10));
```

---

## 三、WHERE 子句优化

### 3.1 索引失效的常见场景

| 场景 | 示例 | 原因 |
|------|------|------|
| 对索引列使用函数 | `WHERE YEAR(date_col) = 2024` | 函数导致索引列无法直接比较 |
| 对索引列进行运算 | `WHERE col1 + 1 = 10` | 表达式改变了列值 |
| 隐式类型转换 | `WHERE varchar_col = 123` | 字符串列与数值比较触发转换 |
| LIKE 前缀通配符 | `WHERE col LIKE '%abc'` | 前缀 `%` 导致无法使用索引 |
| OR 条件中部分列无索引 | `WHERE indexed_col = 1 OR unindexed_col = 2` | 整体可能退化为全表扫描 |
| 字符集不匹配 | `WHERE utf8mb4_col = latin1_col` | 不同字符集比较无法使用索引 |

### 3.2 优化建议

```sql
-- 错误：函数导致索引失效
SELECT * FROM t WHERE YEAR(create_time) = 2024;

-- 正确：使用范围查询
SELECT * FROM t WHERE create_time >= '2024-01-01' AND create_time < '2025-01-01';

-- 错误：隐式类型转换
SELECT * FROM t WHERE phone = 13800138000;  -- phone 是 VARCHAR

-- 正确：使用字符串字面量
SELECT * FROM t WHERE phone = '13800138000';

-- 错误：前缀通配符
SELECT * FROM t WHERE name LIKE '%张%';

-- 正确：前缀匹配可使用索引
SELECT * FROM t WHERE name LIKE '张%';
```

### 3.3 范围查询优化

- `BETWEEN`、`IN()`、`>`、`<`、`>=`、`<=` 可以使用范围扫描
- 多列索引中，只有最左前缀的等值条件之后的列才能使用范围扫描
- 单个查询中，每个表**只能使用一个索引**进行范围扫描（除非触发 Index Merge）

---

## 四、ORDER BY 优化

### 4.1 可以使用索引排序的场景

当 ORDER BY 满足以下条件时，MySQL 可以使用索引避免 filesort：

- ORDER BY 的列与索引的列顺序一致
- ORDER BY 中所有列的排序方向（ASC/DESC）与索引一致，或索引为降序索引
- WHERE 条件中使用了索引的最左前缀作为常量

```sql
-- 假设索引为 (key_part1, key_part2)
-- 可以使用索引排序
SELECT * FROM t ORDER BY key_part1, key_part2;
SELECT * FROM t WHERE key_part1 = 1 ORDER BY key_part2;
SELECT * FROM t ORDER BY key_part1 DESC, key_part2 DESC;

-- 不能使用索引排序
SELECT * FROM t ORDER BY key_part2;              -- 跳过最左前缀
SELECT * FROM t ORDER BY key_part1, key_part3;   -- 不连续的索引列
SELECT * FROM t ORDER BY ABS(key_part1);         -- 表达式
```

### 4.2 filesort 机制

当无法使用索引排序时，MySQL 执行 filesort 操作：

- 优化器分配内存缓冲区，最大为 `sort_buffer_size`
- 如果结果集过大，会使用临时磁盘文件进行外部排序
- 可通过 `Sort_merge_passes` 状态变量监控磁盘合并次数

**优化 filesort 的策略**：
- 增大 `sort_buffer_size`（使排序能在内存中完成）
- 增大 `read_rnd_buffer_size`（加速排序后的行读取）
- 将 `tmpdir` 指向独立的物理磁盘

### 4.3 MySQL 8.4 的重要变化

MySQL 8.4 移除了 `GROUP BY` 的隐式排序行为。在 8.3 及更早版本中，`GROUP BY` 会隐式对结果排序；8.4 中不再如此。如需排序，必须显式使用 `ORDER BY`。

---

## 五、JOIN 优化

### 5.1 嵌套循环连接（Nested-Loop Join）

MySQL 默认使用嵌套循环算法处理 JOIN：
- 从第一个表读取一行
- 在第二个表中查找匹配行
- 依次处理后续表
- 所有表处理完后输出结果，然后回溯

### 5.2 Hash Join（MySQL 8.0+）

- MySQL 8.0 引入了 Hash Join，在**等值连接且无可用索引**时通常比 Block Nested-Loop 更快
- 在 MySQL 8.4 中，Hash Join 已成为默认行为
- 使用 `BNL`/`NO_BNL` Optimizer Hint 控制

### 5.3 JOIN 优化要点

- **连接列的类型和大小必须一致**：`VARCHAR(10)` 和 `CHAR(10)` 可以，但 `VARCHAR(10)` 和 `CHAR(15)` 不行
- **非二进制字符串列必须使用相同字符集**：`utf8mb4` 和 `latin1` 比较会导致索引失效
- 小表驱动大表：让结果集较小的表作为驱动表
- 确保连接列有索引

---

## 六、子查询优化

### 6.1 子查询优化策略

| 策略 | 说明 |
|------|------|
| 半连接（Semijoin） | 将 `IN (SELECT ...)` 转换为 JOIN 形式，避免重复扫描 |
| 物化（Materialization） | 将子查询结果存入临时表，避免重复执行 |
| EXISTS 策略 | 将子查询转换为 EXISTS 相关子查询 |
| 反连接（Antijoin） | 将 `NOT IN` / `NOT EXISTS` 转换为反连接 |

### 6.2 子查询优化建议

- 优先使用 JOIN 替代相关子查询
- `IN (SELECT ...)` 在 MySQL 8.0+ 中通常会被优化为半连接
- 对于大结果集的子查询，物化通常比重复执行更高效
- 使用 `EXPLAIN` 检查子查询是否被物化（`select_type` 显示为 `MATERIALIZED`）

---

## 七、Optimizer Hints（优化器提示）

MySQL 8.0+ 支持在语句级别使用 Optimizer Hints 控制优化器行为，比 `optimizer_switch` 系统变量更精细。

### 7.1 语法

```sql
SELECT /*+ hint_name([table_list]) */ ...
```

### 7.2 常用 Hint

| Hint | 作用域 | 说明 |
|------|--------|------|
| `JOIN_ORDER(t1, t2)` | Query block | 指定表的连接顺序 |
| `JOIN_PREFIX(t1, t2)` | Query block | 指定连接计划的前几个表 |
| `JOIN_SUFFIX(t1, t2)` | Query block | 指定连接计划的后几个表 |
| `INDEX(t, idx_name)` | Index | 强制使用指定索引 |
| `NO_INDEX(t, idx_name)` | Index | 强制忽略指定索引 |
| `BKA(t1)`, `NO_BKA(t1)` | Table | 启用/禁用 Batched Key Access |
| `BNL(t1)`, `NO_BNL(t1)` | Table | 启用/禁用 Hash Join |
| `NO_ICP(t, idx)` | Table/Index | 禁用 Index Condition Pushdown |
| `NO_RANGE_OPTIMIZATION(t, idx)` | Table/Index | 禁用范围优化 |
| `MERGE(dt)`, `NO_MERGE(dt)` | Table | 派生表合并/物化 |
| `MAX_EXECUTION_TIME(N)` | Global | 限制语句执行时间（毫秒） |
| `SET_VAR(var=value)` | Global | 在语句执行期间设置变量 |

### 7.3 使用示例

```sql
-- 强制使用特定索引
SELECT /*+ INDEX(t1 idx_col1) */ * FROM t1 WHERE col1 = 1;

-- 禁用范围优化（当范围过多时）
SELECT /*+ NO_RANGE_OPTIMIZATION(t3 PRIMARY, f2_idx) */ f1
  FROM t3 WHERE f1 > 30 AND f1 < 33;

-- 控制派生表处理方式
SELECT /*+ MERGE(dt) */ * FROM (SELECT * FROM t1) AS dt;

-- 限制执行时间
SELECT /*+ MAX_EXECUTION_TIME(5000) */ * FROM large_table;

-- 临时设置变量
INSERT /*+ SET_VAR(foreign_key_checks=OFF) */ INTO t2 VALUES(2);
```

---

## 八、InnoDB 特定优化

### 8.1 Buffer Pool

- InnoDB Buffer Pool 是最重要的性能参数
- 建议设置为物理内存的 50%-80%（专用数据库服务器）
- 通过 `innodb_buffer_pool_size` 配置

### 8.2 主键设计

- InnoDB 表的数据按主键（聚簇索引）组织
- 主键应尽可能**短且递增**（如 AUTO_INCREMENT）
- 避免使用 UUID 作为主键（导致页分裂和随机 I/O）
- 所有二级索引都隐式包含主键值

### 8.3 事务优化

- 只读事务使用 `START TRANSACTION READ ONLY`（MySQL 8.0+ 可省略 UNDO log）
- 保持事务尽可能短，减少锁持有时间
- 批量操作时适当增大 `innodb_log_file_size`

---

## 九、数据类型优化

### 9.1 基本原则

- 使用**最小**的能正确存储数据的类型
- 更简单的数据类型通常处理更快
- `NOT NULL` 比 `NULL` 更高效（除非确实需要 NULL 语义）

### 9.2 数值类型

| 类型 | 存储 | 范围 | 建议 |
|------|------|------|------|
| TINYINT | 1 字节 | -128~127 | 状态、标志位 |
| SMALLINT | 2 字节 | -32768~32767 | 较小范围的计数 |
| INT | 4 字节 | -2^31~2^31-1 | 最常用的整数类型 |
| BIGINT | 8 字节 | -2^63~2^63-1 | 仅在需要时使用 |

### 9.3 字符串类型

- `CHAR`：定长，适合长度固定的数据（如 MD5、国家代码）
- `VARCHAR`：变长，适合长度不固定的数据
- `TEXT`/`BLOB`：大文本/二进制数据，使用额外的溢出页存储
- 选择正确的字符集：`utf8mb4` 支持完整 Unicode（包括 emoji），`latin1` 仅支持西欧字符

---

## 十、工程实践清单

### 10.1 日常优化流程

1. **慢查询日志**：开启 `slow_query_log`，设置合理的 `long_query_time`（如 1 秒）
2. **EXPLAIN 分析**：对慢查询逐一使用 EXPLAIN 分析执行计划
3. **关注 type 列**：确保至少达到 `range` 级别，避免 `ALL`
4. **关注 Extra 列**：消除 `Using filesort` 和 `Using temporary`
5. **检查索引使用**：确认 `key` 列使用了预期索引
6. **验证行数**：`rows` 列的估算值是否合理

### 10.2 索引设计规范

- 为 WHERE、JOIN、ORDER BY、GROUP BY 中频繁使用的列创建索引
- 复合索引将**高选择性**的列放在前面
- 避免在更新频繁的列上创建过多索引
- 定期使用 `SHOW INDEX FROM table` 检查索引使用情况
- 利用不可见索引（Invisible Indexes）安全测试索引删除的影响

### 10.3 常见反模式

| 反模式 | 问题 | 替代方案 |
|--------|------|----------|
| `SELECT *` | 无法使用覆盖索引，传输多余数据 | 只选择需要的列 |
| 在索引列上使用函数 | 索引失效 | 改写为范围查询 |
| 过度使用 OR | 可能导致全表扫描 | 使用 UNION 或 IN |
| 大事务 | 锁持有时间长，undo log 膨胀 | 拆分为小事务 |
| N+1 查询 | 循环中逐条查询 | 使用 JOIN 或 IN 批量查询 |
| 隐式类型转换 | 索引失效 | 确保比较双方类型一致 |

### 10.4 监控与维护

```sql
-- 查看慢查询配置
SHOW VARIABLES LIKE 'slow_query%';
SHOW VARIABLES LIKE 'long_query_time';

-- 查看当前正在执行的查询
SHOW PROCESSLIST;

-- 查看索引使用情况
SELECT * FROM sys.schema_unused_indexes;

-- 更新索引统计信息（帮助优化器做出更好决策）
ANALYZE TABLE table_name;

-- 查看表状态
SHOW TABLE STATUS LIKE 'table_name';
```

---

## 十一、相关概念

- **聚簇索引（Clustered Index）**：InnoDB 中数据按主键顺序存储，主键即聚簇索引
- **二级索引（Secondary Index）**：非主键索引，叶子节点存储主键值
- **覆盖索引（Covering Index）**：包含查询所需所有列的索引
- **回表（Bookmark Lookup）**：通过二级索引找到主键值后，再查聚簇索引获取完整行
- **Index Condition Pushdown（ICP）**：将 WHERE 条件下推到存储引擎层过滤
- **Multi-Range Read（MRR）**：将随机 I/O 转换为顺序 I/O 的优化

---

## 参考资料

- MySQL 8.4 Reference Manual - Optimization: <https://dev.mysql.com/doc/refman/8.4/en/optimization.html>
- EXPLAIN Output Format: <https://dev.mysql.com/doc/refman/8.4/en/explain-output.html>
- Optimization and Indexes: <https://dev.mysql.com/doc/refman/8.4/en/optimization-indexes.html>
- How MySQL Uses Indexes: <https://dev.mysql.com/doc/refman/8.4/en/mysql-indexes.html>
- ORDER BY Optimization: <https://dev.mysql.com/doc/refman/8.4/en/order-by-optimization.html>
- Optimizer Hints: <https://dev.mysql.com/doc/refman/8.4/en/optimizer-hints.html>
