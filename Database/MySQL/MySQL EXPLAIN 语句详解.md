---
created: 2026-04-16
updated: 2026-04-16
tags:
  - MySQL
  - 数据库
  - 查询优化
  - 执行计划
  - SQL
aliases:
  - EXPLAIN
  - DESCRIBE
  - DESC
  - 执行计划
  - Query Execution Plan
source_type: official-doc
source_urls:
  - https://dev.mysql.com/doc/refman/8.4/en/explain.html
  - https://dev.mysql.com/doc/refman/8.4/en/explain-output.html
status: verified
---

## 是什么

`EXPLAIN` 是 MySQL 中用于查看 SQL 语句**执行计划（Query Execution Plan）**的语句。它告诉开发者 MySQL 优化器将如何处理一条查询——包括表的访问顺序、使用的索引、扫描的行数估算、连接方式等。

`EXPLAIN`、`DESCRIBE`、`DESC` 三者在 MySQL 解析器中完全同义，但约定俗成的用法是：

| 关键字 | 惯用场景 |
|--------|----------|
| `DESCRIBE` / `DESC` | 查看表结构（列名、类型、键） |
| `EXPLAIN` | 查看查询执行计划 |

## 语法

```sql
{EXPLAIN | DESCRIBE | DESC}
    [explain_type] [INTO variable]
    {[schema_spec] explainable_stmt | FOR CONNECTION connection_id}

{EXPLAIN | DESCRIBE | DESC} ANALYZE [FORMAT = TREE] [schema_spec] select_statement

explain_type:
    FORMAT = format_name

format_name:
    TRADITIONAL
  | JSON
  | TREE

explainable_stmt:
    SELECT statement
  | TABLE statement
  | DELETE statement
  | INSERT statement
  | REPLACE statement
  | UPDATE statement

schema_spec:
    FOR {SCHEMA | DATABASE} schema_name
```

### 支持的语句类型

`EXPLAIN` 可用于以下语句的执行计划分析：

- `SELECT`
- `DELETE`
- `INSERT`
- `REPLACE`
- `UPDATE`
- `TABLE`

### 三种输出格式

| 格式 | 说明 | 适用场景 |
|------|------|----------|
| `TRADITIONAL` | 表格输出（默认） | 快速浏览 |
| `JSON` | JSON 格式，字段完整，可用 JSON 函数提取 | 程序化分析、自动化脚本 |
| `TREE` | 树形输出，描述更精确，唯一显示 Hash Join 的格式 | 深度分析复杂查询 |

MySQL 8.4 中默认格式由系统变量 `explain_format` 控制，默认值为 `TRADITIONAL`。

### MySQL 8.4 新增特性

- **`INTO` 子句**：`EXPLAIN FORMAT=JSON INTO @var SELECT ...` 可将 JSON 结果存入用户变量，后续可用 `JSON_EXTRACT()` 等函数处理
- **`FOR SCHEMA` / `FOR DATABASE` 子句**：`EXPLAIN FORMAT=TREE FOR SCHEMA db_name SELECT ...` 可指定在哪个数据库上下文中分析语句
- **JSON 格式版本 2**：通过 `SET @@explain_json_format_version = 2` 可切换到基于 access path 的 JSON 输出格式，面向未来优化器兼容

## 输出列说明

`EXPLAIN` 对查询中使用的每张表返回一行信息，列含义如下：

| 列名 | JSON 名称 | 含义 |
|------|-----------|------|
| `id` | `select_id` | SELECT 标识符，查询中 SELECT 的序号 |
| `select_type` | — | SELECT 类型（见下表） |
| `table` | `table_name` | 输出行对应的表名 |
| `partitions` | `partitions` | 匹配的分区，非分区表为 NULL |
| `type` | `access_type` | 连接类型（访问类型），性能评估最关键指标 |
| `possible_keys` | `possible_keys` | 可能使用的索引 |
| `key` | `key` | 实际使用的索引 |
| `key_len` | `key_length` | 实际使用索引的长度（字节数） |
| `ref` | `ref` | 与索引比较的列或常量 |
| `rows` | `rows` | 估算需要扫描的行数 |
| `filtered` | `filtered` | 按表条件过滤的行百分比，最大值 100 |
| `Extra` | — | 额外信息（见下文） |

### select_type 取值

| 值 | 含义 |
|----|------|
| `SIMPLE` | 简单 SELECT（不使用 UNION 或子查询） |
| `PRIMARY` | 最外层 SELECT |
| `UNION` | UNION 中第二个或后续的 SELECT |
| `DEPENDENT UNION` | 依赖外层查询的 UNION 中的 SELECT |
| `UNION RESULT` | UNION 的结果 |
| `SUBQUERY` | 子查询中的第一个 SELECT |
| `DEPENDENT SUBQUERY` | 依赖外层查询的子查询中的第一个 SELECT |
| `DERIVED` | 派生表（FROM 子句中的子查询） |
| `DEPENDENT DERIVED` | 依赖另一个表的派生表 |
| `MATERIALIZED` | 物化子查询 |
| `UNCACHEABLE SUBQUERY` | 结果无法缓存的子查询，必须对外层每行重新评估 |
| `UNCACHEABLE UNION` | 属于不可缓存子查询的 UNION 中的后续 SELECT |

**`DEPENDENT` 与 `UNCACHEABLE` 的区别**：
- `DEPENDENT SUBQUERY`：对外层上下文中每组不同的变量值仅重新评估一次
- `UNCACHEABLE SUBQUERY`：对外层上下文的**每一行**都重新评估

## 连接类型（type）—— 性能从优到劣

`type` 列是评估查询性能最核心的指标，值从最优到最差排列：

| type | 含义 | 典型场景 |
|------|------|----------|
| `system` | 表仅一行（系统表），const 的特例 | — |
| `const` | 最多一行匹配，通过 PRIMARY KEY 或 UNIQUE 索引与常量比较 | `WHERE id = 1` |
| `eq_ref` | 对前表的每行组合，本表读取一行。使用 PRIMARY KEY 或 UNIQUE NOT NULL 索引的等值连接 | `JOIN ... ON t1.pk = t2.fk` |
| `ref` | 对前表的每行组合，读取本表中所有匹配索引值的行。使用非唯一索引或左前缀 | `WHERE name = 'Alice'` |
| `fulltext` | 使用 FULLTEXT 全文索引 | `MATCH ... AGAINST` |
| `ref_or_null` | 类似 ref，但额外搜索 NULL 值 | `WHERE key = expr OR key IS NULL` |
| `index_merge` | 使用 Index Merge 优化，多个索引合并 | 多个独立索引条件 |
| `unique_subquery` | 替代某些 IN 子查询中的 eq_ref | `val IN (SELECT pk FROM t)` |
| `index_subquery` | 类似 unique_subquery，用于非唯一索引 | `val IN (SELECT key FROM t)` |
| `range` | 使用索引检索给定范围内的行 | `WHERE id BETWEEN 10 AND 20` |
| `index` | 全索引扫描（索引树扫描），比 ALL 快因为索引通常比数据小 | 覆盖索引场景 |
| `ALL` | 全表扫描，通常应避免 | 无索引或索引不可用 |

**关键原则**：至少达到 `range` 级别，理想情况是 `ref` 或更好。`ALL` 出现在非 const 首表时通常意味着性能问题。

## Extra 常见值

`Extra` 列提供查询执行的额外信息，以下是最需要关注的值：

### 需要警惕的值（通常需要优化）

| Extra 值 | 含义 | 优化方向 |
|----------|------|----------|
| `Using filesort` | MySQL 需要额外排序，无法利用索引完成 ORDER BY | 添加合适的索引覆盖排序字段 |
| `Using temporary` | 使用临时表存储中间结果（常见于 GROUP BY、DISTINCT） | 优化查询结构或添加索引 |

### 常见值

| Extra 值 | 含义 |
|----------|------|
| `Using index` | 覆盖索引——所有需要的数据都能从索引中获取，无需回表 |
| `Using where` | 使用了 WHERE 条件过滤 |
| `Using index condition` | 使用了 Index Condition Pushdown（ICP）优化 |
| `Using join buffer (Block Nested Loop)` | 使用连接缓冲区，通常意味着缺少合适的连接索引 |
| `Impossible WHERE` | WHERE 条件永远为假，不会返回任何行 |
| `Impossible HAVING` | HAVING 条件永远为假 |
| `No tables used` | 查询没有 FROM 子句或 FROM DUAL |
| `Distinct` | 找到第一个匹配行后停止搜索 |
| `Not exists` | LEFT JOIN 优化，找到匹配行后不再扫描 |
| `Select tables optimized away` | 优化器已用索引聚合替代表扫描（如 MIN/MAX） |
| `Range checked for each record` | 没有找到好的索引，但对每行组合检查是否可用 range/index_merge |

## EXPLAIN ANALYZE

`EXPLAIN ANALYZE` **实际执行**语句并返回执行计划，同时提供计时信息和迭代器级别的详细数据。

### 输出信息

对每个迭代器提供：

- 估算执行成本
- 估算返回行数
- 返回第一行的时间
- 执行该迭代器耗时（毫秒）
- 实际返回行数
- 循环次数

### 限制

- 仅支持 `SELECT`、多表 `UPDATE`、多表 `DELETE`、`TABLE` 语句
- 始终使用 `TREE` 输出格式
- 不支持 `FOR CONNECTION`
- 可用 `KILL QUERY` 或 `Ctrl+C` 终止

### 示例

```sql
EXPLAIN ANALYZE SELECT * FROM t1 JOIN t2 ON (t1.c1 = t2.c2)\G
```

## 实用技巧

### 1. 发现缺失索引

`EXPLAIN` 最直接的作用是识别应该添加索引的位置：

- `possible_keys` 为 NULL 且 `type` 为 `ALL` → 考虑在 WHERE 列上建索引
- `key` 为 NULL → MySQL 没有找到可用的索引

### 2. 检查连接顺序

`EXPLAIN` 按 MySQL 读取表的顺序输出行。如果连接顺序不是最优，可考虑：

- 使用 `SELECT STRAIGHT_JOIN` 强制按书写顺序连接
- 更新表统计信息：`ANALYZE TABLE tbl_name`

### 3. 强制/忽略索引

```sql
-- 强制使用某索引
SELECT * FROM t FORCE INDEX (idx_name) WHERE ...

-- 忽略某索引
SELECT * FROM t IGNORE INDEX (idx_name) WHERE ...

-- 仅使用某索引
SELECT * FROM t USE INDEX (idx_name) WHERE ...
```

### 4. 查看扩展信息

```sql
EXPLAIN SELECT ...;
SHOW WARNINGS;  -- 显示优化器重写后的语句
```

### 5. 查看特定连接的执行计划

```sql
EXPLAIN FOR CONNECTION <connection_id>;
```

需要 `PROCESS` 权限（如果连接属于其他用户）。

### 6. 更新表统计信息

当索引未被按预期使用时，先尝试：

```sql
ANALYZE TABLE tbl_name;
```

这会更新键的基数（cardinality）等统计信息，影响优化器的选择。

## 与相关概念的关系

| 概念 | 关系 |
|------|------|
| `ANALYZE TABLE` | 更新表统计信息，影响 EXPLAIN 的估算结果 |
| Optimizer Trace | 提供比 EXPLAIN 更详细的优化器决策过程，但格式和内容会随版本变化 |
| `SHOW PROFILE` | 已废弃，推荐使用 Performance Schema 或 EXPLAIN ANALYZE |
| MySQL Workbench Visual Explain | 图形化展示 EXPLAIN 输出 |

## 常见误区

- **`rows` 是估算值**：对 InnoDB 表，`rows` 列是优化器的估算，不保证精确
- **`EXPLAIN` 不执行语句**：普通 `EXPLAIN` 只展示计划，不实际运行（`EXPLAIN ANALYZE` 除外）
- **`possible_keys` 与表输出顺序无关**：`possible_keys` 列出的索引可能因连接顺序而在实际中不可用
- **`key` 可能不在 `possible_keys` 中**：当某个索引覆盖所有查询列时，即使不用于行查找，也可能被选做索引扫描
- **`filtered` 是百分比**：`rows × filtered%` = 与下一表连接的行数

## 参考资料

- MySQL 8.4 Reference Manual — [EXPLAIN Statement](https://dev.mysql.com/doc/refman/8.4/en/explain.html)
- MySQL 8.4 Reference Manual — [EXPLAIN Output Format](https://dev.mysql.com/doc/refman/8.4/en/explain-output.html)
- MySQL 8.4 Reference Manual — [Optimizing Queries with EXPLAIN](https://dev.mysql.com/doc/refman/8.4/en/using-explain.html)
