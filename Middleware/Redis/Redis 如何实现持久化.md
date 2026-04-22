---
created: 2026-04-22
updated: 2026-04-22
tags:
  - database
  - redis
  - persistence
  - RDB
  - AOF
  - durability
  - backup
  - data-safety
aliases:
  - Redis Persistence
  - Redis 持久化机制
  - Redis RDB AOF
  - How Redis persists data
source_type: official-doc
source_urls:
  - https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/
  - https://redis.io/docs/latest/commands/bgsave/
  - https://redis.io/docs/latest/commands/bgrewriteaof/
  - https://redis.io/docs/latest/commands/config-set/
status: verified
---

## 是什么

Redis 持久化（Persistence）指将内存中的数据写入到**耐久存储介质**（如 SSD、HDD）的过程，确保服务器重启或故障后数据可恢复。作为内存数据库，Redis 默认数据全部驻留内存，持久化是可选的，但生产环境中几乎总是需要配置。

Redis 提供三种持久化策略：

| 策略 | 说明 | 适用场景 |
|------|------|----------|
| **RDB**（Redis Database） | 定时生成内存数据集的快照（snapshot），保存为紧凑的二进制文件 | 备份、灾难恢复、可容忍分钟级数据丢失的场景 |
| **AOF**（Append Only File） | 记录每条写命令到日志文件，重启时重放命令重建数据 | 对数据安全性要求较高的场景 |
| **RDB + AOF 混合** | 同时启用两种机制，兼顾恢复速度与数据完整性 | 一般业务推荐配置 |

> **注意**：也可以完全关闭持久化，仅用于缓存场景（数据可从源数据库重建）。

---

## 为什么重要

Redis 是内存数据库，**不持久化意味着进程退出或服务器断电后所有数据丢失**。持久化机制的选择直接影响：

- **数据安全性**：故障时最多可能丢失多少数据
- **恢复速度**：重启后重建数据集需要多长时间
- **运行时性能**：持久化操作对正常读写延迟的影响
- **存储成本**：持久化文件的大小和磁盘占用

---

## RDB 持久化

### 原理

RDB 在指定时间间隔内生成内存数据集的**时间点快照**（point-in-time snapshot），保存为单个二进制文件 `dump.rdb`。

### 触发方式

| 方式 | 命令/配置 | 是否阻塞 |
|------|-----------|----------|
| 手动阻塞 | `SAVE` | 是，主线程阻塞直到快照完成 |
| 手动非阻塞 | `BGSAVE` | 否，fork 子进程在后台执行 |
| 自动触发 | `save <seconds> <changes>` 配置 | 否，内部调用 `BGSAVE` |

**自动触发配置示例**：

```conf
# 900 秒（15 分钟）内至少 1 次修改则触发
save 900 1
# 300 秒（5 分钟）内至少 10 次修改则触发
save 300 10
# 60 秒内至少 10000 次修改则触发
save 60 10000
```

满足任一条件即触发。配置 `save ""` 可禁用自动快照。

### 工作流程（BGSAVE）

```
1. 主进程调用 fork() 创建子进程
2. 子进程遍历内存数据，写入临时 RDB 文件（temp-XXXX.rdb）
3. 写入完成后，原子替换旧 RDB 文件（rename(2) 系统调用）
4. 子进程退出，父进程继续服务
```

**关键技术：Copy-on-Write（COW）**

fork 后父子进程共享相同的物理内存页。只有当父进程修改某页时，操作系统才会复制该页给子进程。这意味着：

- fork 本身很快（仅需复制页表，24 GB 数据集约需 48 MB 页表）
- 写入越频繁，COW 产生的额外内存开销越大
- **必须确保系统有足够可用内存**供 COW 使用

### 优点

- 文件紧凑，单个二进制文件，适合备份和传输
- 恢复速度快（直接加载二进制数据，无需重放命令）
- 对性能影响小（fork 后子进程独立工作，父进程不做磁盘 I/O）
- 在从节点上支持重启/故障转移后的**部分重同步**

### 缺点

- **可能丢失最后一次快照后的数据**（通常几分钟到几小时）
- fork 子进程时，大数据集可能导致主进程阻塞数毫秒到数秒
- 写入频繁时 COW 机制增加内存开销

### RDB 文件格式

RDB 文件是二进制格式，包含：

- 文件头（魔术字 "REDIS" + 版本号）
- 数据库选择标记（`SELECT DB`）
- 键值对数据（带过期时间、类型编码）
- 文件尾校验和

可使用 `redis-check-rdb` 工具检查 RDB 文件完整性，或使用第三方工具（如 `rdbtools`）解析 RDB 文件内容。

---

## AOF 持久化

### 原理

AOF 记录每条**改变数据集的写命令**（如 `SET`、`DEL`、`LPUSH`），使用与 Redis 通信协议（RESP）相同的格式。重启时按顺序重放这些命令重建数据。

### 启用方式

```conf
appendonly yes
```

Redis 7.0+ 使用**多文件 AOF 机制**，原始单文件被拆分为：

| 文件类型 | 说明 | 数量 |
|----------|------|------|
| **Base 文件** | 重写时生成的全量快照（RDB 或 AOF 格式） | 最多 1 个 |
| **Incremental 文件** | 上次重写以来的增量写命令 | 0 个或多个 |
| **Manifest 文件** | 清单文件，记录 base 和 incremental 文件的引用关系 | 1 个 |

所有文件存放在 `appenddirname` 配置的目录中（默认 `appendonlydir`）。

### 刷盘策略（appendfsync）

| 策略 | 行为 | 数据安全性 | 性能影响 |
|------|------|-----------|----------|
| `always` | 每条写命令都执行 `fsync` 到磁盘 | 最多丢失 1 条命令 | 最慢，但支持 group commit |
| `everysec`（**默认**） | 每秒执行一次 `fsync` | 最多丢失 1 秒数据 | 推荐，性能与安全性平衡 |
| `no` | 交给操作系统决定何时刷盘 | 不确定（Linux 通常约 30 秒） | 最快，但最不安全 |

> **`always` 的实际行为**：命令并非逐条 fsync，而是在一批来自多个客户端或 pipeline 的命令执行后，执行一次写操作和一次 fsync。

### AOF 重写（Rewrite）

**为什么需要重写**：AOF 文件会随写操作不断增长。例如对一个计数器执行 100 次 `INCR`，AOF 中会有 100 条记录，但实际只需 1 条 `SET` 即可表示当前状态。

**触发方式**：

- 手动：`BGREWRITEAOF`
- 自动：当 AOF 文件大小超过上次重写后的 `auto-aof-rewrite-percentage`（默认 100%，即翻倍）且至少 `auto-aof-rewrite-min-size`（默认 64 MB）

**Redis 7.0+ 重写流程**：

```
1. 父进程 fork() 创建子进程
2. 父进程打开一个新的 incremental AOF 文件，继续写入新命令
3. 子进程遍历内存数据，生成新的 base AOF 文件（写入临时文件）
4. 子进程完成后，父进程收到信号
5. 父进程用新 base 文件 + 新 incremental 文件构建临时 manifest
6. 原子交换 manifest 文件，使重写结果生效
7. 清理旧的 base 文件和不再使用的 incremental 文件
```

**重写期间的安全保障**：如果重写失败，旧的 base + incremental 文件 + 新打开的 incremental 文件仍然代表完整的数据集，不会丢失数据。

**Redis < 7.0 的旧机制**：父进程在内存中缓冲重写期间的所有写命令，子进程完成后追加到新文件末尾。这导致重写期间的写命令被写入磁盘两次，且重写结束时可能短暂阻塞。

### 优点

- **更高的数据安全性**：默认每秒刷盘，最多丢失 1 秒数据
- AOF 是 append-only 日志，无 seek 操作，断电不会产生 seek 类损坏
- 日志格式易读易解析（RESP 协议），可手动编辑修复
- 即使 AOF 文件末尾被截断，Redis 也能自动加载并丢弃最后一条不完整命令

### 缺点

- AOF 文件通常比同等数据的 RDB 文件更大
- `everysec` 策略下性能略低于 RDB（但仍很高）
- Redis < 7.0 时，重写期间的写命令会占用额外内存缓冲

---

## 混合持久化

### 原理

Redis 4.0+ 引入混合持久化，在 AOF 重写时**先写入 RDB 格式的全量数据，再追加 AOF 格式的增量数据**到同一个 AOF 文件中。

```conf
aof-use-rdb-preamble yes    # Redis 4.0+ 默认开启
```

### 优势

- **快速恢复**：RDB 部分可快速加载，无需重放全量命令
- **数据安全**：AOF 增量部分保证最后一次重写后的数据不丢失
- **文件紧凑**：RDB 格式的全量数据比纯 AOF 命令更节省空间

### 文件格式

混合 AOF 文件结构：

```
[RDB preamble] + [AOF tail]
```

Redis 加载时自动识别：先读取 RDB 部分恢复大部分数据，再重放 AOF 尾部命令恢复增量数据。

---

## 两种机制的交互

| 场景 | 行为 |
|------|------|
| RDB 快照进行中 | 不会触发 AOF 重写，避免两个后台进程同时做重度磁盘 I/O |
| AOF 重写进行中 | 不会触发 `BGSAVE` |
| 用户手动请求 `BGREWRITEAOF` 但快照正在进行 | 返回 OK，重写操作在快照完成后执行 |
| AOF 和 RDB 同时启用，Redis 重启 | **优先使用 AOF** 重建数据（AOF 保证更完整） |

---

## 配置建议

### 按场景推荐

| 场景 | 推荐配置 | 数据丢失容忍 | 理由 |
|------|----------|-------------|------|
| **纯缓存**（数据可重建） | 关闭持久化或仅 RDB | 全部/分钟级 | 性能优先，重启后从源数据库重建 |
| **一般业务** | RDB + AOF（everysec）+ 混合持久化 | 秒级 | 平衡性能与数据安全，推荐默认配置 |
| **金融/关键数据** | AOF（always）+ RDB 定期备份 | 命令级 | 数据安全性优先，性能次之 |
| **大规模数据** | RDB 定期快照 + AOF（everysec） | 秒级 | 避免 AOF 文件过大影响恢复速度 |

### 关键配置项

```conf
# RDB 快照规则
save 900 1
save 300 10
save 60 10000

# RDB 文件名
dbfilename dump.rdb

# AOF 配置
appendonly yes
appendfilename "appendonly.aof"
appenddirname "appendonlydir"
appendfsync everysec

# 混合持久化（Redis 4.0+ 默认开启）
aof-use-rdb-preamble yes

# AOF 自动重写
auto-aof-rewrite-percentage 100    # 增长 100% 时触发
auto-aof-rewrite-min-size 64mb     # 最小触发大小

# AOF 截断加载（Redis 默认开启）
# 当 AOF 文件末尾不完整时，丢弃最后一条不完整命令后继续加载
aof-load-truncated yes

# 重写期间暂停 fsync（降低磁盘压力，可能丢失重写期间的秒级数据）
no-appendfsync-on-rewrite no
```

---

## 备份与灾难恢复

### RDB 备份

RDB 文件在生成过程中使用临时文件名，完成后通过 `rename(2)` 原子替换，因此**可以在 Redis 运行时安全地复制 RDB 文件**。

**推荐备份策略**：

- 每小时快照保留 48 小时，每日快照保留 1-2 个月
- 每天至少一次将 RDB 快照传输到**数据中心外**（如 S3、远端服务器）
- 使用 `gpg -c` 加密后传输

### AOF 备份（Redis 7.0+）

由于 AOF 是多文件结构，备份期间需防止重写导致文件不一致：

```bash
# 1. 禁用自动重写
redis-cli CONFIG SET auto-aof-rewrite-percentage 0

# 2. 确认没有重写正在进行
redis-cli INFO persistence | grep aof_rewrite_in_progress
# 确保值为 0

# 3. 安全复制 appenddirname 目录中的所有文件
cp -r /path/to/appendonlydir /backup/path/

# 4. 恢复自动重写
redis-cli CONFIG SET auto-aof-rewrite-percentage 100
```

**优化技巧**：使用硬链接（`ln`）代替复制，创建硬链接后即可立即恢复重写，然后从硬链接备份。

### 灾难恢复

- 将加密的 RDB/AOF 备份传输到 Amazon S3、远端 VPS 等外部存储
- 至少使用两个不同的存储服务商
- 传输后验证文件大小和 SHA1/SHA256 摘要
- 建立独立的告警系统，确保备份传输失败时能及时通知

---

## 从 RDB 切换到 AOF

**重要**：必须按照正确步骤操作，否则可能导致数据丢失。

```bash
# 1. 备份最新的 dump.rdb
cp dump.rdb /safe/place/

# 2. 在运行中的服务器上启用 AOF
redis-cli CONFIG SET appendonly yes

# 3. 可选：禁用 RDB 自动快照
redis-cli CONFIG SET save ""

# 4. 确认写入正常追加到 AOF 文件

# 5. 持久化配置到 redis.conf（关键步骤！）
redis-cli CONFIG REWRITE

# 6. 等待 AOF 重写完成
redis-cli INFO persistence
# 确认 aof_rewrite_in_progress=0, aof_rewrite_scheduled=0, aof_last_bgrewrite_status=ok

# 7. 重启服务器后验证键数量一致
```

> **不遵循此流程的后果**：如果仅修改 `redis.conf` 然后重启，Redis 会用空的 AOF 文件启动，导致数据丢失。

---

## AOF 文件修复

### 截断（Truncated）

服务器崩溃或磁盘满时，AOF 文件末尾可能被截断。Redis 默认行为：

```
# !!! Warning: short read while loading the AOF file !!!
# !!! Truncating the AOF at offset 439 !!!
# AOF loaded anyway because aof-load-truncated is enabled
```

丢弃最后一条不完整命令后继续加载。可通过 `aof-load-truncated no` 改变此行为。

### 损坏（Corrupted）

AOF 文件中间部分出现无效字节序列时，Redis 启动会中止并报错：

```bash
# 先不使用 --fix 查看问题
redis-check-aof /path/to/appendonlydir/

# 确认后修复
redis-check-aof --fix /path/to/appendonlydir/
```

> **警告**：`--fix` 会丢弃从损坏位置到文件末尾的所有数据。如果损坏发生在文件开头，可能导致大量数据丢失。建议先手动检查并尝试修复。

---

## 限制与注意事项

| 限制 | 说明 |
|------|------|
| **fork 阻塞** | 大数据集 fork 时主进程可能阻塞数毫秒到数秒，24 GB 数据集约需 48 MB 页表复制 |
| **THP 影响** | Linux Transparent Huge Pages 会导致 fork 后 COW 复制几乎全部内存，**必须禁用** |
| **内存预留** | `maxmemory` 应设置为物理内存的 60%-75%，预留空间给 fork 子进程的 COW 开销 |
| **vm.overcommit_memory** | 建议设为 1，允许 fork 时 overcommit，否则可能因内存不足导致 `BGSAVE` 失败 |
| **磁盘 I/O 竞争** | RDB 快照和 AOF 重写不会同时进行，但会与其他磁盘操作竞争 I/O |
| **AOF 文件大小** | 纯 AOF 模式下文件可能远大于 RDB，需监控磁盘空间 |
| **复制期间的持久化** | 从节点首次连接主节点时会接收 RDB 快照，持久化配置不影响此过程 |

---

## 常见误区

| 误区 | 事实 |
|------|------|
| "RDB 比 AOF 安全" | 相反，AOF（everysec）最多丢失 1 秒数据，RDB 可能丢失几分钟到几小时 |
| "AOF always 是最安全的" | 虽然理论上最安全，但实际性能很差；`everysec` 配合混合持久化是更好的平衡 |
| "RDB 和 AOF 同时启用会冲突" | 不会，Redis 会自动协调两者，避免后台进程冲突；重启时优先使用 AOF |
| "关闭持久化就完全没有数据文件" | RDB 文件可能仍存在（上次快照留下的），只是不再更新 |
| "AOF 文件可以直接编辑" | 可以（RESP 格式易读），但需谨慎，编辑后务必用 `redis-check-aof` 验证 |
| "Redis 7.0 的 AOF 还是单文件" | 7.0+ 改为多文件结构（base + incremental + manifest），备份方式有变化 |

---

## 相关概念

- **Copy-on-Write（COW）**：操作系统内存管理机制，RDB 快照和 AOF 重写依赖此机制实现后台持久化
- **fork()**：Unix 系统调用，创建当前进程的副本；Redis 持久化的核心机制
- **fsync**：系统调用，强制将缓冲区数据写入物理磁盘；AOF 刷盘的核心操作
- **RESP**（Redis Serialization Protocol）：Redis 通信协议格式，AOF 文件使用相同格式记录命令
- **Transparent Huge Pages（THP）**：Linux 内核特性，会严重影响 Redis fork 性能，必须禁用
- **redis-check-aof / redis-check-rdb**：Redis 自带的持久化文件检查和修复工具

---

## 参考资料

- [Redis 官方文档 — Persistence](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/)
- [BGSAVE 命令文档](https://redis.io/docs/latest/commands/bgsave/)
- [BGREWRITEAOF 命令文档](https://redis.io/docs/latest/commands/bgrewriteaof/)
- [CONFIG SET 命令文档](https://redis.io/docs/latest/commands/config-set/)
- [fork(2) 手册页](http://linux.die.net/man/2/fork)
- [fsync(2) 手册页](http://linux.die.net/man/2/fsync)
