---
created: 2026-04-16T00:00:00.000Z
updated: '2026-04-20T00:00:00.000Z'
tags:
  - database
  - redis
  - production
  - operations
  - troubleshooting
  - performance
  - high-availability
  - best-practices
aliases:
  - Redis Production Issues
  - Redis 生产运维
  - Redis 故障排查
  - Redis 最佳实践
  - Redis in Production
source_type: mixed
source_urls:
  - 'https://redis.io/docs/latest/operate/oss_and_stack/management/'
  - 'https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/'
  - >-
    https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/latency/
  - >-
    https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/memory-optimization/
  - >-
    https://redis.io/docs/latest/operate/oss_and_stack/management/troubleshooting/
  - 'https://redis.io/docs/latest/operate/oss_and_stack/management/sentinel/'
  - 'https://redis.io/docs/latest/operate/oss_and_stack/management/scaling/'
  - 'https://redis.io/docs/latest/operate/oss_and_stack/management/security/'
  - 'https://redis.io/docs/latest/commands/hotkeys/'
  - 'https://redis.io/docs/latest/commands/hotkeys-start/'
  - 'https://redis.io/docs/latest/commands/object-freq/'
status: verified
---

## 概述

Redis 在生产环境中面临的问题可归纳为六大类：**内存问题、延迟问题、持久化问题、高可用问题、集群问题、安全与运维问题**。本文围绕每类问题，给出具体表现、根因分析、处理方式与规避方案。

---

## 一、内存问题

### 1.1 内存溢出（OOM）

**表现**：Redis 返回 `-OOM command not allowed when used memory > 'maxmemory'` 错误，所有写入操作失败。

**根因**：
- 未配置 `maxmemory`，Redis 持续消耗系统内存直至被 OOM Killer 终止
- 配置了 `maxmemory` 但驱逐策略（`maxmemory-policy`）为 `noeviction`，达到上限后拒绝写入
- 缓存未设置 TTL，数据只增不减

**处理方式**：
```bash
# 紧急扩容或释放内存
# 方式一：临时提高 maxmemory（需系统有可用内存）
redis-cli CONFIG SET maxmemory 8gb

# 方式二：切换驱逐策略，允许自动淘汰数据
redis-cli CONFIG SET maxmemory-policy allkeys-lru

# 方式三：手动清理大键或过期键
redis-cli --bigkeys    # 定位大键
redis-cli --memkeys    # 定位内存占用最高的键（Redis 7.0+）
```

**规避方案**：
- **必须配置 `maxmemory`**：设置为物理内存的 60%-75%，预留内存给 fork 子进程（RDB/AOF 重写时使用 Copy-on-Write）
- **合理选择驱逐策略**：
  | 场景 | 推荐策略 |
  |------|----------|
  | 纯缓存（数据可重建） | `allkeys-lru` 或 `allkeys-lfu` |
  | 缓存 + 持久数据混合 | `volatile-lru`（仅淘汰带 TTL 的键） |
  | 不允许数据丢失 | `noeviction`（但需配合监控告警） |
- **所有缓存键设置 TTL**：避免"只写不删"导致内存持续增长
- **监控内存使用率**：设置告警阈值（如 70% 预警、85% 紧急）

### 1.2 内存碎片

**表现**：`INFO memory` 中 `mem_fragmentation_ratio` 持续 > 1.5，RSS 远大于 Redis 报告的已用内存。

**根因**：
- Redis 删除或修改键后，底层内存分配器（jemalloc/tcmalloc）无法立即将释放的内存归还给操作系统
- 频繁的键删除和写入操作导致内存碎片化

**处理方式**：
```bash
# 查看碎片率
redis-cli INFO memory | grep mem_fragmentation_ratio

# Redis 4.0+ 可手动触发内存整理
redis-cli CONFIG SET activedefrag yes

# 自动碎片整理配置
redis-cli CONFIG SET active-defrag-enabled yes
redis-cli CONFIG SET active-defrag-threshold-lower 10   # 碎片率 > 10% 开始整理
redis-cli CONFIG SET active-defrag-threshold-upper 100  # 碎片率 > 100% 全力整理
```

**规避方案**：
- 使用 **jemalloc** 作为内存分配器（Linux 上 Redis 默认使用），其碎片率低于 libc malloc
- 开启 **Active Defragmentation**（Redis 4.0+），自动整理碎片
- 避免频繁删除和重建大量小键
- 注意：`mem_fragmentation_ratio` 在内存使用峰值远高于当前使用量时不可靠，因为 RSS 反映的是峰值内存

### 1.3 大键（Big Keys）

**表现**：
- 删除大键时 Redis 阻塞数秒甚至数十秒
- 迁移大键导致网络拥塞和延迟抖动
- RDB 快照和 AOF 重写时内存峰值过高

**根因**：单个键包含大量元素（如百万元素的 Sorted Set、10 MB 的 String）。

**处理方式**：
```bash
# 定位大键
redis-cli --bigkeys          # 扫描并报告每类数据类型的最大键
redis-cli --bigkeys --json   # JSON 格式输出（便于脚本处理）

# 查看单个键的内存占用（Redis 4.0+）
redis-cli MEMORY USAGE mykey

# 删除大键时使用 UNLINK 替代 DEL（异步删除，不阻塞主线程）
redis-cli UNLINK mykey
```

**规避方案**：
- **设计阶段避免大键**：合理拆分数据结构，单个 Hash/Sorted Set 元素数控制在万级以内
- **使用 `UNLINK` 替代 `DEL`**：异步删除，后台线程回收内存
- **拆分大键操作**：使用 `SCAN`/`HSCAN`/`SSCAN`/`ZSCAN` 逐步处理，避免一次性操作全部元素
- **监控键大小**：定期运行 `--bigkeys` 扫描，建立基线

### 1.4 内存不归还操作系统

**表现**：删除大量数据后，`INFO memory` 显示已用内存下降，但 `used_memory_rss`（操作系统视角的内存占用）不下降。

**根因**：大多数 malloc 实现不会轻易将释放的内存归还给 OS，而是保留在进程内供后续分配复用。

**处理方式**：
- 这是正常行为，无需处理。分配器会复用已释放的内存块
- 如果确实需要释放内存，可重启 Redis 实例（需配合高可用方案保证可用性）

**规避方案**：
- **按峰值内存规划容量**：如果 workload 峰值需要 10 GB，即使大部分时间只需 5 GB，也需按 10 GB 配置
- 理解 `mem_fragmentation_ratio = used_memory_rss / used_memory` 的含义：当大量键被删除后，该比值会虚高，不代表真正的碎片问题

### 1.5 热键（Hot Keys）

**定义**：热键是指被高频访问的键，其访问频率显著高于其他键。Redis 8.6+ 的 HOTKEYS 命令基于两个指标定义热键[^3]：
- **CPU 时间占比**：键消耗的 CPU 时间占 Redis 总 CPU 时间的百分比
- **网络字节占比**：键的网络流量（输入+输出）占 Redis 总网络流量的百分比

**表现**：
- 单个键承载过高的访问压力，导致热点问题
- 在 Cluster 环境中，热键所在节点成为瓶颈，其他节点负载不均衡
- 高频读操作可能造成带宽瓶颈
- 高频写操作可能导致延迟抖动

**检测方式**：

| 方法 | 版本要求 | 说明 |
|------|---------|------|
| `HOTKEYS START/GET` | Redis 8.6+ | 官方热键检测命令，基于 CPU 和网络指标 |
| `OBJECT FREQ key` | Redis 4.0+ | 需要 LFU 驱逐策略，返回键的对数访问频率计数器 |
| `redis-cli --hotkeys` | Redis 8.6+ | 命令行工具，执行热键检测扫描 |
| 代理层统计 | 所有版本 | 在客户端或代理层记录访问统计 |
| MONITOR 分析 | 所有版本 | 使用 `MONITOR` 命令（谨慎使用，性能影响大） |

**Redis 8.6+ HOTKEYS 命令工作流**[^4]：

```bash
# 启动热键检测，追踪 CPU 和网络指标，检测 top 10
redis-cli HOTKEYS START METRICS 2 CPU NET COUNT 10 DURATION 60

# 获取检测结果
redis-cli HOTKEYS GET

# 停止追踪（保留数据）
redis-cli HOTKEYS STOP

# 释放追踪资源
redis-cli HOTKEYS RESET
```

**处理方式**：

```bash
# 方式一：拆分热键为多个子键（本地缓存）
# 例如：将 user:1000:profile 拆分为 user:1000:profile:v1/v2/v3
# 应用端随机选择子键读取，减少单键压力

# 方式二：使用本地缓存
# 在应用端缓存热键数据，减少对 Redis 的访问频率

# 方式三：增加 Replica 负载读请求
# 将热键的读请求分散到多个从节点

# 方式四：在 Cluster 中迁移热键到独立节点
# 使用 CLUSTER SETSLOT 将热键迁移到负载较低的节点
```

**规避方案**：
- **设计阶段避免热键**：
  - 数据拆分：将高频访问的数据分散到多个键（如分片计数器、分片排行榜）
  - 本地缓存：对高频读数据在应用端做 LRU 缓存
  - 避免全量数据缓存：如排行榜只缓存 Top N 而非全量
- **Cluster 环境优化**：
  - 监控各节点负载，发现不均衡时检查热键分布
  - 使用 Hash Tag 将相关键集中到同一槽位时要考虑热键风险
- **设置合理 TTL**：避免热键长期占用资源
- **使用读写分离**：将热键的读请求分流到从节点

---

## 二、延迟问题

### 2.1 慢命令阻塞

**表现**：Redis 偶尔出现毫秒到秒级的响应延迟，所有客户端请求排队等待。

**根因**：Redis 主线程单线程处理命令，O(N) 复杂度的慢命令会阻塞后续所有请求。

**常见危险命令**：

| 命令 | 复杂度 | 风险说明 |
|------|--------|----------|
| `KEYS *` | O(N) | 遍历全量键空间，键数量大时阻塞严重 |
| `SMEMBERS` | O(N) | 返回集合全部元素 |
| `HGETALL` | O(N) | 返回哈希全部字段 |
| `LRANGE 0 -1` | O(N) | 返回列表全部元素 |
| `SORT` | O(N+M log M) | 排序操作 |
| `SUNION`/`SINTER` | O(N) | 集合运算 |
| `FLUSHALL`/`FLUSHDB` | O(N) | 清空数据 |

**处理方式**：
```bash
# 查看慢查询日志
redis-cli SLOWLOG GET 10
redis-cli SLOWLOG LEN

# 调整慢查询阈值（微秒），默认 10000（10ms）
redis-cli CONFIG SET slowlog-log-slower-than 5000

# 调整慢查询日志长度
redis-cli CONFIG SET slowlog-max-len 128
```

**规避方案**：
- **生产环境禁用 `KEYS` 命令**：使用 `rename-command KEYS ""` 或在 ACL 中禁止
- **使用 `SCAN` 系列命令替代**：`SCAN`、`HSCAN`、`SSCAN`、`ZSCAN` 增量迭代，每次只返回少量结果
- **限制集合大小**：避免单个集合/哈希/列表包含过多元素
- **将慢查询放到从节点**：使用只读从节点执行分析类查询
- **启用慢查询日志**：设置合理的阈值，定期分析

### 2.2 Fork 操作延迟

**表现**：在执行 `BGSAVE` 或 `BGREWRITEAOF` 时出现毫秒到秒级的延迟尖刺。

**根因**：fork 子进程时需要复制页表，数据量越大 fork 耗时越长。24 GB 的 Redis 实例在 Linux/AMD64 上 fork 约需 48 MB 页表复制。

**不同环境下的 fork 性能**[^1]：

| 环境 | 6 GB RSS fork 耗时 | 每 GB 耗时 |
|------|-------------------|-----------|
| 物理机（Xeon 2.27GHz） | 62 ms | 9 ms/GB |
| VMware 虚拟机 | 77 ms | 12.8 ms/GB |
| EC2 新实例（HVM） | ~10 ms/GB | 接近物理机 |
| EC2 旧实例（Xen） | 1460 ms | 239.3 ms/GB |
| Linode（Xen） | 382 ms | 424 ms/GB |

**规避方案**：
- **禁用 Transparent Huge Pages（THP）**：THP 会导致 fork 后 Copy-on-Write 时复制几乎全部内存
  ```bash
  echo never > /sys/kernel/mm/transparent_hugepage/enabled
  # 持久化：写入 /etc/rc.local 或 systemd service
  ```
- **选择现代虚拟机类型**：EC2 用户使用 HVM 实例（如 m5/m6i 系列），避免旧版 Xen PV 实例
- **控制数据集大小**：单实例数据量不宜过大，通过 Cluster 分片控制单节点内存
- **调整持久化频率**：减少 `BGSAVE` 触发频率，降低 fork 次数
- **配置 `no-appendfsync-on-rewrite yes`**：在 AOF/RDB 重写期间暂停 fsync，降低磁盘压力（可能丢失重写期间的秒级数据）

### 2.3 Swap（操作系统换页）

**表现**：Redis 偶尔出现数十毫秒到秒级的延迟，系统整体响应变慢。

**根因**：操作系统将 Redis 的内存页换出到磁盘（swap），访问这些页时需要从磁盘换回，产生随机 I/O 延迟。

**检测方式**：
```bash
# 查看 Redis 进程 ID
redis-cli INFO | grep process_id

# 检查 Redis 进程的 swap 使用量
cat /proc/<pid>/smaps | grep 'Swap:' | awk '{sum+=$2} END {print sum " kB"}'

# 使用 vmstat 查看系统级 swap 活动
vmstat 1
# 关注 si（swap in）和 so（swap out）列，非零值说明有换页活动
```

**规避方案**：
- **确保物理内存充足**：Redis 使用的内存不应超过可用物理内存
- **不要在 Redis 所在机器运行其他内存密集型进程**
- **降低 swappiness**：`sysctl vm.swappiness=1`（尽量不使用 swap）
- **禁用 swap**（极端场景）：`swapoff -a`（需确保内存绝对充足）

### 2.4 大量键同时过期

**表现**：在某个时间点 Redis 突然阻塞数秒。

**根因**：Redis 每 100ms 执行一次主动过期清理，随机抽样 20 个键，如果超过 25% 的样本已过期则继续循环。如果大量键在同一秒过期，Redis 会持续循环清理直到过期键比例降到 25% 以下。

**规避方案**：
- **避免大量键设置相同的过期时间**：在 TTL 上添加随机偏移（jitter）
  ```python
  # 示例：在基础 TTL 上添加 0-60 秒的随机偏移
  ttl = base_ttl + random.randint(0, 60)
  redis.set(key, value, ex=ttl)
  ```
- **使用 `EXPIREAT` 时分散时间戳**：不要将所有键的过期时间设为同一个 Unix 时间戳

### 2.5 AOF 磁盘 I/O 延迟

**表现**：使用 AOF 持久化时出现延迟尖刺，尤其在 `appendfsync everysec` 或 `always` 模式下。

**根因**：`fdatasync(2)` 系统调用在某些内核和文件系统组合下可能需要数毫秒到数秒才能完成，尤其当其他进程也在进行大量 I/O 时。

**规避方案**：
- **使用 `appendfsync everysec`**：在数据安全和性能之间取得平衡（最多丢失 1 秒数据）
- **设置 `no-appendfsync-on-rewrite yes`**：在 AOF 重写期间暂停 fsync，降低磁盘压力
- **使用 SSD 磁盘**：显著降低 fsync 延迟
- **避免在 Redis 所在机器运行其他 I/O 密集型进程**

### 2.6 网络延迟

**表现**：客户端到 Redis 的 RTT 较高，即使 Redis 本身处理速度很快。

**典型网络延迟**[^2]：
- 1 Gbit/s 网络：约 200 μs
- Unix Domain Socket：约 30 μs
- 虚拟化环境：显著高于物理机

**规避方案**：
- **优先使用物理机**：虚拟化环境存在额外的调度延迟和"邻居噪声"
- **同机部署使用 Unix Domain Socket**：`redis-cli -s /tmp/redis.sock`
- **使用 Pipeline 减少往返次数**：将多个命令打包发送
- **保持长连接**：避免频繁连接/断开
- **使用聚合命令**：`MSET`/`MGET` 优于多个单独的 `SET`/`GET`

---

## 三、持久化问题

### 3.1 数据丢失

**场景与风险**：

| 持久化方式 | 最多可能丢失 | 说明 |
|-----------|------------|------|
| 无持久化 | 全部数据 | 仅适合缓存场景 |
| 仅 RDB | 最后一次快照后的数据 | 通常几分钟到几小时 |
| AOF `everysec`（默认） | 1 秒数据 | 推荐配置 |
| AOF `always` | 1 条命令 | 性能最低 |
| RDB + AOF 混合 | 1 秒数据 | 兼顾恢复速度和数据安全 |

**规避方案**：
- **一般业务**：RDB + AOF（`everysec`）+ 混合持久化（`aof-use-rdb-preamble yes`，Redis 4.0+ 默认开启）
- **纯缓存**：关闭持久化或仅用 RDB 定期快照
- **关键数据**：AOF `always` + RDB 定期备份到异地
- **定期备份到异地**：使用 cron 定时将 RDB 文件传输到 S3、远端服务器等

### 3.2 RDB 快照失败

**表现**：`BGSAVE` 返回错误，日志中出现 `Can't save in background: fork: Cannot allocate memory`。

**根因**：
- 系统内存不足，fork 子进程失败
- `vm.overcommit_memory` 设置为 2 且可用内存不足

**处理方式**：
```bash
# 检查 overcommit 设置
cat /proc/sys/vm/overcommit_memory
# 0: 启发式检查（默认）
# 1: 总是允许 overcommit（推荐用于 Redis）
# 2: 拒绝超过 swap + (ram * overcommit_ratio) 的分配

# 修改为 1
sysctl vm.overcommit_memory=1
# 持久化：写入 /etc/sysctl.conf
echo "vm.overcommit_memory=1" >> /etc/sysctl.conf
```

**规避方案**：
- **设置 `vm.overcommit_memory=1`**：允许 fork 时 overcommit，依赖 Copy-on-Write 机制
- **确保足够的可用内存**：预留至少与 Redis 数据集等量的内存供 fork 使用
- **监控 fork 状态**：通过 `INFO persistence` 中的 `rdb_last_bgsave_status` 检查

### 3.3 AOF 文件损坏或截断

**表现**：Redis 重启时无法加载 AOF 文件，启动失败。

**处理方式**：
```bash
# Redis 会自动检测并处理截断的 AOF（aof-load-truncated 默认开启）
# 如果 AOF 中间部分损坏，使用 redis-check-aof 修复
redis-check-aof --fix /path/to/appendonly.aof

# Redis 7.0+ 的 AOF 是多文件结构，检查整个目录
redis-check-aof --fix /path/to/appendonlydir/
```

**规避方案**：
- **保持 `aof-load-truncated yes`**（默认）：允许加载截断的 AOF 文件
- **定期备份 AOF 文件**：备份期间暂停 AOF 重写
- **使用 RDB + AOF 混合模式**：AOF 损坏时可用 RDB 恢复

---

## 四、高可用问题

### 4.1 主从复制延迟

**表现**：从节点数据落后于主节点，读取从节点可能拿到过期数据。

**根因**：
- 主从异步复制，写入主节点后立即返回客户端，再异步传播到从节点
- 网络带宽不足、从节点负载高、大键传输等都会增加复制延迟

**处理方式**：
```bash
# 查看复制状态
redis-cli INFO replication
# 关注 offset 差值：master_repl_offset - slave_repl_offset

# 等待从节点同步完成（Redis 3.0+）
redis-cli WAIT 1 5000   # 等待至少 1 个从节点确认，超时 5 秒
```

**规避方案**：
- **不要依赖从节点做强一致性读取**：如需强一致性，使用 `WAIT` 命令或直接从主节点读取
- **监控复制偏移量差值**：设置告警，差值过大时排查网络或从节点负载
- **避免在主节点执行大键操作**：大键传输会阻塞复制流
- **使用专用复制网络**：主从之间使用独立的低延迟网络

### 4.2 Sentinel 脑裂（Split-Brain）

**表现**：Sentinel 错误地判定主节点下线，触发不必要的故障转移，导致出现两个主节点。

**根因**：
- 网络分区导致 Sentinel 无法连接到主节点，但客户端仍然可以连接
- Sentinel 数量不足或 quorum 设置过低

**规避方案**：
- **部署至少 3 个 Sentinel 节点**：奇数部署，分布在不同的物理机器上
- **合理设置 quorum**：`sentinel monitor mymaster <ip> <port> <quorum>`，quorum 应 > Sentinel 总数 / 2
- **合理设置 `down-after-milliseconds`**：不要太短（避免误判），也不要太长（故障转移慢），建议 5000-30000 ms
- **配置 `min-replicas-to-write`**：主节点在从节点数量低于阈值时拒绝写入，防止脑裂后数据分叉
  ```
  min-replicas-to-write 1
  min-replicas-max-lag 10
  ```

### 4.3 故障转移时间过长

**表现**：主节点宕机后，从节点提升为新主节点耗时过长（> 10 秒）。

**根因**：
- `down-after-milliseconds` 设置过大
- Sentinel 选举领导者耗时
- 从节点数据同步未完成

**规避方案**：
- **优化 Sentinel 配置**：
  ```
  sentinel down-after-milliseconds mymaster 5000    # 5 秒判定下线
  sentinel failover-timeout mymaster 60000           # 故障转移超时
  sentinel parallel-syncs mymaster 1                 # 同时同步的从节点数
  ```
- **确保从节点数据接近主节点**：减少故障转移时的同步时间
- **监控 Sentinel 日志**：及时发现故障转移异常

---

## 五、集群问题

### 5.1 跨槽位多键操作失败

**表现**：在 Cluster 模式下执行 `MSET`、`SUNION`、`MULTI` 等多键操作时返回 `-CROSSSLOT Keys in request don't hash to the same slot`。

**根因**：Redis Cluster 要求多键操作的所有键必须在同一个哈希槽（16384 个槽之一）。

**处理方式**：
```bash
# 使用 Hash Tag 强制相关键映射到同一槽位
# {user:1000}:name 和 {user:1000}:email 都使用 user:1000 计算槽位
MSET {user:1000}:name "Alice" {user:1000}:email "alice@example.com"
```

**规避方案**：
- **设计键名时使用 Hash Tag**：将需要一起操作的键用 `{}` 包裹相同前缀
- **避免在集群中使用不支持的多键操作**：将逻辑拆分为单键操作或在客户端合并
- **注意**：集群模式仅支持数据库 0，`SELECT` 命令不可用

### 5.2 槽位迁移期间客户端重定向风暴

**表现**：在扩缩容或重新分片期间，客户端收到大量 `-MOVED`/`-ASK` 重定向，延迟升高。

**根因**：传统槽位迁移（`CLUSTER GETKEYSINSLOT` + `MIGRATE`）逐键迁移，迁移期间客户端请求可能落在正在迁移的槽位上。

**规避方案**：
- **升级到 Redis 8.4+ 使用原子槽位迁移（ASM）**：
  - 迁移速度提升 30 倍（~640 槽/秒 vs ~21 槽/秒）
  - 客户端重定向减少 98%（2.1 次/秒 vs 241.6 次/秒）
  - 延迟峰值降低 40%+（<70 ms vs 127 ms）
- **在业务低峰期执行槽位迁移**
- **使用支持 Cluster 的智能客户端**：缓存槽位映射，自动处理重定向
- **监控重定向频率**：异常升高说明可能有迁移在进行或配置问题

### 5.3 网络分区导致部分槽位不可用

**表现**：集群中部分节点因网络分区无法通信，对应槽位停止服务。

**根因**：
- `cluster-require-full-coverage yes`（默认）：任何槽位不可用时，整个集群拒绝写入
- 某个 Master 和其所有 Replica 同时故障

**规避方案**：
- **评估 `cluster-require-full-coverage` 设置**：
  - `yes`（默认）：强一致性，部分槽位不可用时整个集群不可用
  - `no`：可用性优先，部分槽位不可用时其他槽位仍可服务
- **为每个 Master 配置至少一个 Replica**：避免单点故障
- **合理设置 `cluster-node-timeout`**：默认 15000 ms，根据网络状况调整
- **配置 `cluster-allow-reads-when-down yes`**：集群 FAIL 时仍允许读操作

---

## 六、安全与运维问题

### 6.1 未授权访问

**表现**：Redis 实例暴露到公网或内网未授权用户可访问，导致数据泄露或被恶意操作（如 `FLUSHALL`）。

**根因**：
- `bind` 配置为 `0.0.0.0` 或未配置
- 未设置密码（`requirepass`）或未配置 ACL
- 使用默认端口 6379

**规避方案**：
- **绑定内网地址**：`bind 127.0.0.1 10.x.x.x`
- **设置强密码或配置 ACL**：
  ```
  # 简单密码认证
  requirepass your-strong-password
  
  # ACL（Redis 6.0+，推荐）
  user appuser on >password ~app:* +@all -@admin -@dangerous
  ```
- **禁用危险命令**：
  ```
  rename-command FLUSHALL ""
  rename-command CONFIG ""
  rename-command DEBUG ""
  ```
- **启用 TLS**（Redis 6.0+）：加密传输数据
- **配置防火墙规则**：仅允许应用服务器 IP 访问 Redis 端口
- **修改默认端口**：`port 6380`

### 6.2 配置不当

**常见问题**：

| 配置项 | 错误做法 | 推荐做法 |
|--------|---------|---------|
| `maxmemory` | 不设置 | 设置为物理内存的 60%-75% |
| `maxmemory-policy` | 默认 `noeviction`（缓存场景） | 缓存场景使用 `allkeys-lru` |
| `timeout` | 默认 0（连接永不过期） | 设置合理超时，如 `timeout 300` |
| `tcp-keepalive` | 默认 300 | 保持默认或适当降低 |
| `databases` | 使用多个数据库（`SELECT`） | 使用命名空间前缀（如 `user:1000:settings`） |
| `save` 规则 | 过于频繁 | 根据业务容忍的数据丢失量设置 |

**规避方案**：
- **定期审查配置**：对比官方推荐配置和最佳实践
- **使用配置管理工具**：Ansible、Chef 等统一管理 Redis 配置
- **避免运行时 `CONFIG SET` 后忘记持久化**：修改后执行 `CONFIG REWRITE`

### 6.3 缺乏监控

**表现**：问题发生后才被发现，无法提前预警。

**关键监控指标**：

| 指标 | 来源 | 告警阈值建议 |
|------|------|-------------|
| 内存使用率 | `INFO memory` → `used_memory` / `maxmemory` | > 70% 预警，> 85% 紧急 |
| 内存碎片率 | `INFO memory` → `mem_fragmentation_ratio` | > 1.5 持续 10 分钟 |
| 连接数 | `INFO clients` → `connected_clients` | 接近 `maxclients` 的 80% |
| 拒绝连接数 | `INFO stats` → `rejected_connections` | > 0 即告警 |
| 命中率 | `INFO stats` → `keyspace_hits / (hits + misses)` | < 80%（缓存场景） |
| 复制偏移量差值 | `INFO replication` → offset 差值 | > 10 MB 持续 1 分钟 |
| 慢查询数量 | `SLOWLOG LEN` | 新增 > 10 条/小时 |
| fork 耗时 | `INFO persistence` → `latest_fork_usec` | > 500 ms |
| 主从连接状态 | `INFO replication` → `slaveN` 状态 | 从节点断开 |

**规避方案**：
- **集成监控系统**：Prometheus + redis_exporter、Datadog、New Relic 等
- **启用 Redis 内置延迟监控**（Redis 2.8.13+）：
  ```
  CONFIG SET latency-monitor-threshold 100  # 记录 > 100ms 的延迟事件
  LATENCY DOCTOR                             # 查看延迟分析报告
  ```
- **设置告警通知**：通过 PagerDuty、Slack、企业微信等渠道通知

---

## 七、生产环境配置清单

### 最小推荐配置

```conf
# 网络
bind 127.0.0.1 10.0.0.1
port 6380
tcp-backlog 511
timeout 300
tcp-keepalive 300

# 安全
requirepass <strong-password>
rename-command FLUSHALL ""
rename-command CONFIG ""

# 内存
maxmemory 6gb
maxmemory-policy allkeys-lru

# 持久化（一般业务）
save 900 1
save 300 10
save 60 10000
appendonly yes
appendfsync everysec
aof-use-rdb-preamble yes

# 复制
repl-diskless-sync no
repl-backlog-size 64mb

# 其他
hz 10
dynamic-hz yes
vm.overcommit_memory=1    # sysctl
transparent_hugepage=never  # sysctl
```

---

## 八、常见问题速查表

| 问题 | 快速排查命令 | 常见原因 |
|------|------------|---------|
| Redis 拒绝写入 | `INFO memory` | 达到 maxmemory，策略为 noeviction |
| 响应变慢 | `SLOWLOG GET 10` | 慢命令阻塞 |
| 响应偶尔延迟 | `LATENCY DOCTOR` | fork/THP/swap/AOF fsync |
| 内存使用异常高 | `redis-cli --bigkeys` | 大键、内存碎片 |
| 从节点数据不一致 | `INFO replication` | 复制延迟、网络问题 |
| Sentinel 频繁故障转移 | Sentinel 日志 | down-after-milliseconds 过短、网络不稳定 |
| 集群部分不可用 | `CLUSTER INFO` | 槽位覆盖不全、网络分区 |
| BGSAVE 失败 | `INFO persistence` | 内存不足、overcommit 设置 |
| 连接被拒绝 | `INFO clients` | 超过 maxclients、防火墙 |

---

## 九、相关概念

- **Copy-on-Write**：操作系统机制，RDB 快照和 AOF 重写时 fork 子进程依赖此机制，写入频繁时内存开销增加
- **Transparent Huge Pages（THP）**：Linux 内核特性，会导致 Redis fork 后严重延迟，必须禁用
- **jemalloc**：Redis 在 Linux 上的默认内存分配器，碎片率低于 libc malloc
- **Active Defragmentation**：Redis 4.0+ 引入的在线内存碎片整理功能
- **Hash Tag**：Redis Cluster 中强制多个键映射到同一槽位的机制，使用 `{}` 包裹键的一部分

---

## 十、参考资料

- [Redis 官方运维文档](https://redis.io/docs/latest/operate/oss_and_stack/management/)
- [Redis 持久化文档](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/)
- [Diagnosing latency issues — 官方延迟诊断指南](https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/latency/)
- [Memory optimization — 官方内存优化指南](https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/memory-optimization/)
- [Latency monitoring — 官方延迟监控文档](https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/latency-monitor/)
- [Troubleshooting Redis — 官方故障排查指南](https://redis.io/docs/latest/operate/oss_and_stack/management/troubleshooting/)
- [Redis Sentinel 文档](https://redis.io/docs/latest/operate/oss_and_stack/management/sentinel/)
- [Scale with Redis Cluster — 官方集群文档](https://redis.io/docs/latest/operate/oss_and_stack/management/scaling/)
- [Redis Security 文档](https://redis.io/docs/latest/operate/oss_and_stack/management/security/)
- [Redis Configuration 文档](https://redis.io/docs/latest/operate/oss_and_stack/management/config/)

[^1]: Fork 时间数据来源于 [Redis 官方延迟诊断文档](https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/latency/#fork-time-in-different-systems)
[^2]: 网络延迟数据来源于 [Redis 官方延迟诊断文档](https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/latency/#latency-induced-by-network-and-communication)
[^3]: 热键定义来源于 [Redis HOTKEYS 命令文档](https://redis.io/docs/latest/commands/hotkeys/)
[^4]: HOTKEYS 命令参数来源于 [Redis HOTKEYS START 命令文档](https://redis.io/docs/latest/commands/hotkeys-start/)
