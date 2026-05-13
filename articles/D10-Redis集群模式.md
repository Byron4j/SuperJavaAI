# Redis集群模式深度解析：主从、哨兵与Cluster的架构演进与实战

**文章标签：** #redis #集群 #主从复制 #哨兵模式 #cluster #高可用 #分布式存储 #面试

## 目录

- [引言：Redis集群的本质](#引言redis集群的本质)
- [理论基础：为什么需要Redis集群](#理论基础为什么需要redis集群)
- [来龙去脉：Redis集群的发展史](#来龙去脉redis集群的发展史)
- [核心原理深度解析](#核心原理深度解析)
  - [主从复制的底层原理](#主从复制的底层原理)
  - [哨兵模式的Raft选举机制](#哨兵模式的raft选举机制)
  - [Cluster集群的Gossip协议](#cluster集群的gossip协议)
  - [数据分片与Slot映射](#数据分片与slot映射)
  - [故障转移的完整流程](#故障转移的完整流程)
- [模型差异：三种集群模式对比](#模型差异三种集群模式对比)
- [工业级实践案例](#工业级实践案例)
  - [案例1：电商缓存集群部署](#案例1电商缓存集群部署)
  - [案例2：社交网络Feed流存储](#案例2社交网络feed流存储)
  - [案例3：金融交易会话存储](#案例3金融交易会话存储)
- [性能分析与压测数据](#性能分析与压测数据)
- [常见陷阱与最佳实践](#常见陷阱与最佳实践)
- [面试题与参考答案](#面试题与参考答案)

---

## 引言：Redis集群的本质

Redis集群（Redis Cluster）不是"多台Redis放一起"的简单概念，而是一门**在分布式环境下实现数据分片、高可用和水平扩展**的存储工程。

核心认知：

```
单机Redis的本质：单进程、单线程、单节点
  性能瓶颈：CPU（单核）、内存（单机上限）、网络（单网卡）
  可靠性瓶颈：单点故障、数据丢失风险

Redis集群的本质：多节点协同，数据分片，故障自动转移
  扩展性：水平扩展，动态增删节点
  可用性：部分节点故障仍可服务
  一致性：最终一致，权衡CAP

质量差异的根源：
- 差的集群部署：手动切换、数据不一致、脑裂
- 好的集群部署：自动故障转移、数据强一致监控、完善的运维体系
```

**关键洞察**：Redis集群的效果不取决于"有多少个节点"，而取决于**在故障场景下（网络分区、节点宕机、数据迁移）是否能保证数据安全和业务连续性**。

---

## 理论基础：为什么需要Redis集群

### 1. 单机Redis的物理限制

#### 内存限制

```
单机Redis的内存天花板：

┌─────────────────────────────────────────┐
│  物理限制                                │
│  - 64位系统理论上限：数百TB               │
│  - 实际生产建议：单节点 < 64GB            │
│                                         │
│  为什么不能超过64GB？                     │
│  1. RDB持久化时间：内存越大，BGSAVE越慢   │
│     64GB内存 → RDB可能需要10-30分钟       │
│     期间fork操作阻塞主线程                │
│                                         │
│  2. 主从复制时间：全量同步耗时            │
│     64GB → 网络传输可能需要数分钟         │
│     复制缓冲区可能溢出                    │
│                                         │
│  3. 故障恢复时间：重启加载RDB/AOF         │
│     64GB → 启动可能需要5-10分钟           │
│     业务长时间不可用                      │
│                                         │
│  4. 内存碎片：长期使用后碎片率上升        │
│     实际可用内存减少                      │
└─────────────────────────────────────────┘
```

#### CPU限制

```
Redis的单线程模型：

┌─────────────────────────────────────────┐
│  单线程事件循环                          │
│                                         │
│  读取请求 → 解析命令 → 执行命令 → 返回结果 │
│      ↑__________________________________│
│                                         │
│  优势：无锁编程，避免并发问题              │
│  劣势：无法利用多核CPU                    │
│                                         │
│  性能上限：                             │
│  - 简单命令（GET/SET）：10万QPS/核        │
│  - 复杂命令（HGETALL KEYS）：性能骤降     │
│                                         │
│  集群方案：多节点部署，每个节点跑在不同核  │
│           8核机器 → 部署8个Redis实例     │
└─────────────────────────────────────────┘
```

#### 网络限制

```
单机网络带宽：

- 千兆网卡：理论125MB/s，实际80-100MB/s
- 万兆网卡：理论1.25GB/s，实际800MB-1GB/s

场景计算：
- 每个请求1KB，10万QPS → 100MB/s
- 加上主从复制（1主2从）：300MB/s
- 加上持久化（AOF重写）：额外50MB/s
- 总计：350MB/s，接近千兆网卡上限

集群方案：
- 数据分片到多个节点
- 每个节点独立网卡
- 总带宽 = 节点数 × 单节点带宽
```

### 2. 高可用性的数学定义

```
可用性（Availability）= MTBF / (MTBF + MTTR)

MTBF：平均故障间隔时间（Mean Time Between Failures）
MTTR：平均恢复时间（Mean Time To Recovery）

单机Redis：
- MTBF：1年（假设硬件质量良好）
- MTTR：30分钟（人工介入重启）
- 可用性 = 365×24 / (365×24 + 0.5) = 99.98%（两个9）

主从复制：
- MTBF：1年
- MTTR：5分钟（手动切换从节点）
- 可用性 = 365×24 / (365×24 + 0.08) = 99.997%（三个9）

哨兵模式：
- MTBF：1年
- MTTR：30秒（自动故障转移）
- 可用性 = 365×24 / (365×24 + 0.008) = 99.9997%（四个9）

Cluster集群：
- MTBF：1年
- MTTR：30秒（自动故障转移）
- 多节点冗余：部分故障不影响整体
- 可用性：99.999%+（五个9）

关键洞察：
自动故障转移比手动切换提升10倍可用性
多节点冗余进一步提升容错能力
```

### 3. 分布式系统的CAP选择

```
Redis集群的CAP定位：

┌─────────────────────────────────────────┐
│              CAP 三角                    │
│                                         │
│         Consistency（一致性）            │
│                /\                       │
│               /  \                      │
│              /    \                     │
│             /      \                    │
│            /   Redis  \                 │
│           /   Cluster   \               │
│   Availability ─────── Partition      │
│   （可用性）            （分区容错）      │
│                                         │
│  Redis Cluster的选择：CP偏向AP           │
│  - 数据分片保证分区容错性                 │
│  - 主从复制保证可用性                     │
│  - 最终一致性（非强一致）                 │
│                                         │
│  具体表现：                             │
│  - 主从延迟：从节点数据可能落后主节点     │
│  - 故障转移：切换期间可能丢失少量数据     │
│  - 脑裂：网络分区时可能出现双主           │
│                                         │
│  与ZooKeeper的区别：                     │
│  - ZK选择CP：强一致，牺牲部分可用性       │
│  - Redis选择AP：高可用，最终一致          │
└─────────────────────────────────────────┘
```

---

## 来龙去脉：Redis集群的发展史

### 第一阶段：单节点时代（2009-2012）

```
Redis 1.0 - 2.4 时期：

特点：
- 纯内存存储
- 单线程模型
- 支持RDB持久化（快照）
- 支持AOF持久化（日志）
- 发布订阅（Pub/Sub）

局限性：
1. 单点故障：节点宕机，服务完全不可用
2. 容量限制：受单机内存限制
3. 性能瓶颈：单核CPU、单网卡
4. 无自动恢复：需要人工重启

使用场景：
- 小型应用缓存
- 开发测试环境
- 数据量<10GB
- 可用性要求不高
```

### 第二阶段：主从复制时代（2012-2014）

```
Redis 2.6 - 2.8 时期：

核心特性：
- 主从复制（Replication）
- 部分重同步（PSYNC，2.8引入）
- 复制偏移量（Replication Offset）

架构演进：
  主节点（Master）
     │
     ├── 从节点1（Slave）
     ├── 从节点2（Slave）
     └── 从节点3（Slave）

解决的问题：
1. 数据冗余：从节点备份主节点数据
2. 读写分离：读请求分摊到从节点
3. 故障恢复：从节点可以提升为主节点

未解决的问题：
1. 手动故障转移：主节点宕机需要人工切换
2. 写无法扩展：仍然只有主节点写
3. 存储限制：受单节点内存限制
```

### 第三阶段：哨兵模式时代（2014-2015）

```
Redis 2.8+ 引入 Sentinel：

核心特性：
- 自动故障转移（Automatic Failover）
- 监控（Monitoring）
- 通知（Notification）
- 配置提供（Configuration Provider）

架构：
        ┌── Sentinel1
        │
  主节点 ──┼── Sentinel2 ── 监控和自动故障转移
        │
        └── Sentinel3
            │
    ┌───────┼───────┐
 从节点1  从节点2  从节点3

解决的问题：
1. 自动故障转移：无需人工介入
2. 高可用：秒级切换
3. 客户端透明：Sentinel通知客户端新主节点地址

未解决的问题：
1. 写无法扩展：仍然只有一个主节点
2. 存储限制：受单节点内存限制
3. 哨兵本身的高可用：需要部署多个哨兵
```

### 第四阶段：Cluster集群时代（2015至今）

```
Redis 3.0+ 引入 Cluster：

核心特性：
- 数据分片（Sharding）：16384个slot
- 去中心化：无中心节点，Gossip协议通信
- 自动故障转移：从节点自动提升为主节点
- 在线扩容：动态增删节点，迁移slot

架构：
  MasterA(0-5460) ── SlaveA1
     │
  MasterB(5461-10922) ── SlaveB1
     │
  MasterC(10923-16383) ── SlaveC1

解决的问题：
1. 写扩展：多主节点，数据分片
2. 存储扩展：多节点总内存 = 单节点 × 节点数
3. 高可用：部分节点故障仍可服务
4. 自动运维：故障转移、数据迁移自动化

新的复杂性：
1. 客户端路由：需要处理MOVED/ASK重定向
2. 多Key操作限制：keys必须在同一slot
3. 事务限制：事务中的keys必须在同一节点
4. 运维复杂度：集群监控、扩容缩容、数据平衡
```

### 第五阶段：云原生与Proxy时代（2018至今）

```
当前工业标准：

1. 云托管Redis：
   - AWS ElastiCache for Redis
   - Alibaba Cloud Redis
   - Azure Cache for Redis
   - 特点：托管运维，自动扩容，监控告警

2. Proxy代理层：
   - Twemproxy（Twitter开源）
   - Codis（豌豆荚开源）
   - Redis Cluster Proxy（Redis官方）
   - 特点：客户端透明，支持多key操作

3. 混合部署：
   - 核心数据：Redis Cluster（高可用）
   - 缓存数据：Redis Sentinel（简单）
   - 大数据量：Redis Cluster + SSD（Redis on Flash）

4. 2026年新趋势：
   - Redis 7.x Functions：支持Lua函数持久化
   - Redis Raft模块：强一致性选项
   - RedisJSON/RediSearch/RedisGraph模块生态
```

---

## 核心原理深度解析

### 1. 主从复制的底层原理

#### 复制流程详解

```
主从复制的完整流程：

┌─────────────────────────────────────────────────────────┐
│  阶段1：连接建立                                          │
│                                                         │
│  从节点 ──→ 主节点                                       │
│  REPLCONF listening-port 6379                           │
│  REPLCONF capa psync2                                   │
│                                                         │
│  主节点 ──→ 从节点                                       │
│  +OK                                                    │
│                                                         │
├─────────────────────────────────────────────────────────┤
│  阶段2：数据同步                                          │
│                                                         │
│  情况A：首次同步（Full Resynchronization）               │
│                                                         │
│  从节点 ──→ 主节点                                       │
│  PSYNC ? -1  （请求全量同步）                            │
│                                                         │
│  主节点 ──→ 从节点                                       │
│  +FULLRESYNC runid offset                               │
│                                                         │
│  主节点：                                                 │
│  1. fork子进程执行BGSAVE                                 │
│  2. 生成RDB文件                                          │
│  3. 将RDB文件发送给从节点                                 │
│  4. 期间的写命令写入复制缓冲区（replication backlog）      │
│                                                         │
│  从节点：                                                 │
│  1. 接收RDB文件                                          │
│  2. 清空当前数据                                          │
│  3. 加载RDB文件                                          │
│  4. 向主节点确认RDB加载完成                               │
│                                                         │
│  情况B：增量同步（Partial Resynchronization）            │
│                                                         │
│  从节点 ──→ 主节点                                       │
│  PSYNC runid offset  （请求增量同步）                    │
│                                                         │
│  主节点检查：                                             │
│  - runid匹配？                                          │
│  - offset在复制缓冲区内？                                 │
│                                                         │
│  如果都满足：                                             │
│  +CONTINUE                                              │
│  发送复制缓冲区中offset之后的命令                         │
│                                                         │
│  如果不满足（runid不匹配或offset超出范围）：               │
│  +FULLRESYNC runid offset                               │
│  执行全量同步                                             │
│                                                         │
├─────────────────────────────────────────────────────────┤
│  阶段3：命令传播（Command Propagation）                   │
│                                                         │
│  主节点每执行一个写命令：                                  │
│  1. 将命令写入复制缓冲区                                   │
│  2. 将命令发送给所有从节点                                 │
│                                                         │
│  从节点：                                                 │
│  接收命令并执行，保持与主节点数据一致                       │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

#### 复制缓冲区的数据结构

```
复制缓冲区（Replication Backlog）：

┌─────────────────────────────────────────┐
│  数据结构：固定大小的循环缓冲区           │
│  默认大小：1MB（可配置repl-backlog-size） │
│                                         │
│  ┌─────────────────────────────────┐   │
│  │  offset: 1000                   │   │
│  │  ┌─────┬─────┬─────┬─────┬─────┐│   │
│  │  │ cmd1│ cmd2│ cmd3│ cmd4│ cmd5││   │
│  │  └─────┴─────┴─────┴─────┴─────┘│   │
│  │  offset: 1001  1002  1003  1004  1005│
│  └─────────────────────────────────┘   │
│                                         │
│  增量同步条件：                           │
│  从节点的offset在缓冲区内（>= 1001）      │
│  且runid匹配                             │
│                                         │
│  如果offset < 1001（超出缓冲区范围）：     │
│  必须执行全量同步                         │
│                                         │
│  调优建议：                               │
│  - 写入频繁的场景：增大到64MB或更大        │
│  - 减少全量同步频率                       │
│  - 监控：master_repl_offset与slave_repl_offset差│
└─────────────────────────────────────────┘
```

#### 主从复制的配置与优化

```bash
# redis.conf 主节点配置

# 基础配置
bind 0.0.0.0
port 6379
daemonize yes
pidfile /var/run/redis_6379.pid

# 持久化配置（RDB + AOF）
save 900 1
save 300 10
save 60 10000
appendonly yes
appendfsync everysec

# 复制配置
# repl-backlog-size：复制缓冲区大小
repl-backlog-size 64mb

# repl-timeout：复制超时时间
repl-timeout 60

# repl-disable-tcp-nodelay：控制TCP_NODELAY
# no：低延迟（默认）
# yes：高吞吐量（减少包数量）
repl-disable-tcp-nodelay no

# min-replicas-to-write：最少从节点数（写入限制）
# min-replicas-max-lag：从节点最大延迟（秒）
min-replicas-to-write 1
min-replicas-max-lag 10
```

```bash
# redis.conf 从节点配置

# 基础配置
bind 0.0.0.0
port 6380
daemonize yes

# 主从复制配置
replicaof 192.168.1.100 6379
masterauth password123

# 从节点只读（默认yes）
replica-read-only yes

# 从节点是否同步数据到磁盘
# yes：同步（安全但慢）
# no：不同步（快但可能丢数据）
replica-diskless-sync no

# 复制连接断开后是否继续服务
# yes：继续（可能读到旧数据）
# no：停止（保证数据新鲜）
replica-serve-stale-data yes
```

### 2. 哨兵模式的Raft选举机制

#### 哨兵架构与职责

```
哨兵（Sentinel）架构：

┌─────────────────────────────────────────────────────────┐
│                    Sentinel集群                          │
│                                                         │
│   ┌──────────┐   ┌──────────┐   ┌──────────┐          │
│   │ Sentinel1│   │ Sentinel2│   │ Sentinel3│          │
│   │  (Leader)│   │ (Follower│   │ (Follower│          │
│   │          │   │          │   │          │          │
│   │ 职责：    │   │ 职责：    │   │ 职责：    │          │
│   │ - 监控    │   │ - 监控    │   │ - 监控    │          │
│   │ - 故障转移│   │ - 投票    │   │ - 投票    │          │
│   │ - 通知    │   │           │   │           │          │
│   └────┬─────┘   └────┬─────┘   └────┬─────┘          │
│        │              │              │                  │
│        └──────────────┼──────────────┘                  │
│                       │                                 │
│              ┌────────┴────────┐                       │
│              ↓                 ↓                       │
│         主节点(Master)     从节点1(Slave)              │
│              │                 ↑                       │
│              └────────→ 从节点2(Slave)                 │
│                                                         │
└─────────────────────────────────────────────────────────┘

Sentinel的核心职责：
1. 监控（Monitoring）：持续检查主从节点状态
2. 通知（Notification）：通过API通知管理员
3. 自动故障转移（Automatic Failover）：主节点宕机时自动切换
4. 配置提供（Configuration Provider）：客户端询问Sentinel获取主节点地址
```

#### 主观下线与客观下线

```
节点状态判定流程：

┌─────────────────────────────────────────┐
│  Step 1: 主观下线（Subjectively Down）   │
│                                         │
│  每个Sentinel每秒向所有节点发送PING       │
│                                         │
│  如果节点在down-after-milliseconds内     │
│  未回复PONG或返回错误                     │
│                                         │
│  Sentinel将该节点标记为 SDown            │
│  （主观认为该节点不可用）                 │
│                                         │
│  配置：sentinel down-after-milliseconds  │
│        mymaster 5000  （5秒）            │
│                                         │
├─────────────────────────────────────────┤
│  Step 2: 客观下线（Objectively Down）    │
│                                         │
│  当某个Sentinel发现主节点SDown后          │
│  向其他Sentinel询问该主节点状态           │
│                                         │
│  如果同意主节点不可用的Sentinel数量       │
│  >= quorum（法定人数）                   │
│                                         │
│  主节点被标记为 ODown                    │
│  （客观确认该节点不可用）                 │
│                                         │
│  配置：sentinel monitor mymaster        │
│        192.168.1.100 6379 2             │
│        （quorum=2，至少2个Sentinel同意）  │
│                                         │
│  注意：ODown只针对主节点                  │
│        从节点SDown不会触发故障转移        │
└─────────────────────────────────────────┘
```

#### Sentinel Leader选举（Raft算法）

```
Sentinel Leader选举流程（基于Raft算法）：

┌─────────────────────────────────────────┐
│  触发条件：主节点被标记为ODown           │
│                                         │
│  Step 1: 发起选举                        │
│  ┌─────────────────────────────────┐   │
│  │  Sentinel1发现主节点ODown        │   │
│  │  向其他Sentinel发送：            │   │
│  │  "我想成为Leader，请投票给我"     │   │
│  │                                 │   │
│  │  条件：                          │   │
│  │  - 该Sentinel已确认主节点ODown   │   │
│  │  - 该Sentinel未投票给其他Sentinel│   │
│  └─────────────────────────────────┘   │
│                                         │
│  Step 2: 投票                            │
│  ┌─────────────────────────────────┐   │
│  │  其他Sentinel收到投票请求后：     │   │
│  │                                 │   │
│  │  如果：                          │   │
│  │  - 未投票给其他Sentinel          │   │
│  │  - 确认主节点确实ODown           │   │
│  │                                 │   │
│  │  则：回复同意投票                │   │
│  │                                 │   │
│  │  每个Sentinel只能投一票          │   │
│  │  先到先得                        │   │
│  └─────────────────────────────────┘   │
│                                         │
│  Step 3: 当选Leader                      │
│  ┌─────────────────────────────────┐   │
│  │  如果Sentinel1获得半数以上投票   │   │
│  │  （例如3个Sentinel中至少2票）    │   │
│  │                                 │   │
│  │  Sentinel1成为Leader             │   │
│  │  负责执行故障转移                 │   │
│  └─────────────────────────────────┘   │
│                                         │
│  选举超时：                               │
│  - 如果选举失败（平票或网络问题）         │
│  - 随机等待一段时间后重新发起选举         │
│  - 避免活锁（livelock）                  │
│                                         │
│  随机等待时间：                           │
│  - 固定部分：2秒                          │
│  - 随机部分：0-1秒                        │
│  - 总计：2-3秒                            │
└─────────────────────────────────────────┘
```

#### 故障转移的完整流程

```
Sentinel故障转移流程：

┌─────────────────────────────────────────────────────────┐
│  Step 1: 选举新主节点                                     │
│                                                         │
│  Leader Sentinel从从节点中选择新主节点：                   │
│                                                         │
│  选择优先级：                                             │
│  1. replica-priority（配置优先级，越小越优先）            │
│     配置：replica-priority 100                           │
│     特殊值0：表示不参与选举                               │
│                                                         │
│  2. 复制偏移量（Replication Offset）                      │
│     选择数据最新的从节点                                  │
│     偏移量越大，说明同步的数据越多                         │
│                                                         │
│  3. Run ID                                               │
│     如果优先级和偏移量都相同                              │
│     选择Run ID最小的（字典序）                            │
│                                                         │
│  示例：                                                  │
│  从节点1：priority=100, offset=10000                     │
│  从节点2：priority=100, offset=15000  ← 选择这个          │
│  从节点3：priority=200, offset=15000                     │
│                                                         │
├─────────────────────────────────────────────────────────┤
│  Step 2: 提升新主节点                                     │
│                                                         │
│  Leader Sentinel向选中的从节点发送：                       │
│  SLAVEOF NO ONE  （取消从节点身份）                       │
│                                                         │
│  从节点变为主节点                                         │
│                                                         │
├─────────────────────────────────────────────────────────┤
│  Step 3: 切换其他从节点                                   │
│                                                         │
│  Leader Sentinel向其他从节点发送：                         │
│  SLAVEOF new_master_ip new_master_port                   │
│                                                         │
│  其他从节点开始复制新主节点                                │
│                                                         │
├─────────────────────────────────────────────────────────┤
│  Step 4: 更新配置                                         │
│                                                         │
│  Leader Sentinel更新Sentinel配置：                         │
│  - 将旧主节点标记为从节点（如果恢复）                      │
│  - 将新主节点信息写入配置文件                              │
│                                                         │
│  示例：                                                  │
│  sentinel monitor mymaster 192.168.1.101 6379 2          │
│  （新主节点IP：192.168.1.101）                            │
│                                                         │
├─────────────────────────────────────────────────────────┤
│  Step 5: 通知客户端                                       │
│                                                         │
│  Sentinel通过发布订阅通知客户端：                          │
│  频道：+switch-master                                    │
│  消息：mymaster old_ip old_port new_ip new_port          │
│                                                         │
│  订阅了该频道的客户端自动切换到新主节点                     │
│                                                         │
│  或者客户端通过Sentinel API获取新主节点地址：               │
│  SENTINEL get-master-addr-by-name mymaster               │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

#### 哨兵配置详解

```bash
# sentinel.conf 配置示例

# 哨兵端口
port 26379
daemonize yes
pidfile /var/run/redis-sentinel.pid

# 监控主节点
# sentinel monitor <master-name> <ip> <port> <quorum>
sentinel monitor mymaster 192.168.1.100 6379 2

# 主节点密码
sentinel auth-pass mymaster password123

# 主观下线时间（毫秒）
# 如果主节点在5000ms内未响应，标记为SDown
sentinel down-after-milliseconds mymaster 5000

# 故障转移超时时间（毫秒）
# 包括：选举Leader、选择新主节点、提升新主节点
sentinel failover-timeout mymaster 60000

# 并行同步的从节点数
# 故障转移期间，同时向几个从节点同步数据
sentinel parallel-syncs mymaster 1

# 通知脚本（可选）
# 当发生故障转移时执行脚本
sentinel notification-script mymaster /path/to/notify.sh

# 客户端重新配置脚本（可选）
# 当主节点切换时执行脚本
sentinel client-reconfig-script mymaster /path/to/reconfig.sh
```

### 3. Cluster集群的Gossip协议

#### Gossip协议原理

```
Gossip协议（谣言协议）：去中心化的信息传播算法

核心思想：
- 每个节点定期随机选择几个邻居节点交换信息
- 信息像谣言一样在集群中传播
- 最终所有节点都知道全部信息

┌─────────────────────────────────────────┐
│  节点A的信息传播过程：                     │
│                                         │
│  T=0: 节点A知道 [A, B, C]               │
│                                         │
│  T=1: 节点A随机选择节点B交换信息          │
│       A告诉B：[A, C]                     │
│       B告诉A：[B, D]                     │
│       A现在知道 [A, B, C, D]            │
│                                         │
│  T=2: 节点A随机选择节点C交换信息          │
│       A告诉C：[A, B, C, D]              │
│       C告诉A：[C, E]                     │
│       A现在知道 [A, B, C, D, E]         │
│                                         │
│  T=3: 节点A随机选择节点D交换信息          │
│       A告诉D：[A, B, C, D, E]           │
│       D告诉A：[D, F]                     │
│       A现在知道 [A, B, C, D, E, F]      │
│                                         │
│  最终：所有节点都知道全部信息              │
│  时间复杂度：O(log N)                     │
│  消息复杂度：O(N log N)                   │
└─────────────────────────────────────────┘
```

#### Cluster节点通信

```
Redis Cluster节点间通信：

┌─────────────────────────────────────────────────────────┐
│  通信端口：                                             │
│  - 普通端口：6379（客户端连接）                          │
│  - 集群端口：6379 + 10000 = 16379（节点间通信）          │
│                                                         │
│  通信类型：                                             │
│  1. MEET消息：新节点加入集群时发送                       │
│     "你好，我是新节点，请认识我"                         │
│                                                         │
│  2. PING消息：定期心跳（默认每秒）                       │
│     "我还活着，这是我的状态信息"                         │
│     包含：节点ID、IP、端口、角色、slot分配、epoch等      │
│                                                         │
│  3. PONG消息：对PING/MEET的响应                         │
│     "收到，我也还活着"                                   │
│                                                         │
│  4. FAIL消息：节点故障广播                               │
│     "节点X故障了，大家注意"                              │
│                                                         │
│  5. PUBLISH消息：发布订阅消息的集群传播                   │
│                                                         │
│  6. UPDATE消息：slot配置更新                             │
│                                                         │
├─────────────────────────────────────────────────────────┤
│  故障检测：                                             │
│                                                         │
│  节点A向节点B发送PING：                                  │
│  - 如果B在cluster-node-timeout内未回复PONG              │
│  - A标记B为PFail（可能故障）                             │
│                                                         │
│  A通过Gossip传播B的PFail状态：                           │
│  - 如果大多数主节点都认为B是PFail                        │
│  - B被标记为Fail（确认故障）                             │
│  - A广播FAIL消息给所有节点                                │
│                                                         │
│  配置：cluster-node-timeout 15000  （15秒）              │
└─────────────────────────────────────────────────────────┘
```

### 4. 数据分片与Slot映射

#### Slot分配原理

```
Redis Cluster数据分片：

┌─────────────────────────────────────────────────────────┐
│  Slot总数：16384（16K）                                  │
│  为什么是16384？                                         │
│  - 2^14 = 16384，刚好14位可以表示                       │
│  - 心跳消息中需要携带slot分配信息                         │
│  - 16384个bit = 2KB，适合心跳包大小                       │
│  - 太多slot会增加元数据开销                               │
│  - 太少slot会降低分片精度                                 │
│                                                         │
│  Slot计算：                                              │
│  slot = CRC16(key) % 16384                              │
│                                                         │
│  CRC16算法：                                             │
│  - 多项式：x^16 + x^12 + x^5 + 1                        │
│  - 初始值：0x0000                                       │
│  - 结果：16位无符号整数                                  │
│                                                         │
│  示例：                                                  │
│  key = "user:1000"                                      │
│  CRC16("user:1000") = 12345                             │
│  slot = 12345 % 16384 = 12345                           │
│                                                         │
│  Hash Tag（强制同slot）：                                 │
│  key = "{user}:1000"                                    │
│  只计算{}内的内容：CRC16("user")                        │
│  key = "{user}:profile"                                 │
│  同上slot，保证相关数据在同一节点                          │
│                                                         │
├─────────────────────────────────────────────────────────┤
│  3主3从集群的Slot分配：                                   │
│                                                         │
│  MasterA: 0 - 5460    (5461 slots)                      │
│  MasterB: 5461 - 10922 (5462 slots)                     │
│  MasterC: 10923 - 16383 (5461 slots)                    │
│                                                         │
│  每个Master有1个Slave：                                   │
│  MasterA ← SlaveA1                                      │
│  MasterB ← SlaveB1                                      │
│  MasterC ← SlaveC1                                      │
│                                                         │
│  客户端请求路由：                                         │
│  SET user:1000 "value"                                  │
│  CRC16("user:1000") % 16384 = 12345                     │
│  12345 在 [0, 5460] 区间 → 路由到 MasterA               │
│                                                         │
│  GET order:2000                                         │
│  CRC16("order:2000") % 16384 = 6789                     │
│  6789 在 [5461, 10922] 区间 → 路由到 MasterB            │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

#### 客户端路由机制

```
Cluster客户端路由：

┌─────────────────────────────────────────────────────────┐
│  方式1：ASK/MOVED重定向                                  │
│                                                         │
│  客户端发送请求到任意节点：                                │
│  GET user:1000                                          │
│                                                         │
│  如果该节点不负责这个slot：                                │
│  节点回复：MOVED 12345 192.168.1.1:6379                 │
│  （slot 12345 在 192.168.1.1:6379）                      │
│                                                         │
│  客户端行为：                                             │
│  1. 更新slot-node映射表                                  │
│  2. 重新发送请求到正确节点                                 │
│                                                         │
│  如果slot正在迁移：                                       │
│  节点回复：ASK 12345 192.168.1.2:6379                   │
│  （slot 12345 正在迁移到 192.168.1.2:6379）              │
│                                                         │
│  客户端行为：                                             │
│  1. 发送 ASKING 命令到新节点                              │
│  2. 然后发送实际请求                                       │
│  3. 不更新映射表（临时重定向）                             │
│                                                         │
├─────────────────────────────────────────────────────────┤
│  方式2：Smart Client（智能客户端）                        │
│                                                         │
│  JedisCluster/Lettuce的实现：                            │
│                                                         │
│  1. 启动时获取集群拓扑：                                   │
│     CLUSTER SLOTS                                       │
│     返回：slot范围 -> 主节点IP:端口                       │
│                                                         │
│  2. 本地缓存slot-node映射：                                │
│     Map<Integer, HostAndPort> slotCache                  │
│                                                         │
│  3. 请求时直接路由到正确节点：                             │
│     slot = CRC16(key) % 16384                           │
│     node = slotCache.get(slot)                          │
│     sendRequest(node, request)                          │
│                                                         │
│  4. 遇到MOVED/ASK时更新缓存：                              │
│     定期刷新：每60秒执行一次CLUSTER SLOTS                 │
│                                                         │
│  5. 连接池管理：                                          │
│     每个节点维护独立连接池                                  │
│     连接数 = minIdle + (maxIdle - minIdle) / nodeCount   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

#### Java客户端配置

```java
/**
 * JedisCluster配置
 */
@Configuration
public class JedisClusterConfig {
    
    @Bean
    public JedisCluster jedisCluster() {
        // 集群节点配置（只需要配置部分节点，客户端会自动发现全部）
        Set<HostAndPort> nodes = new HashSet<>();
        nodes.add(new HostAndPort("192.168.1.1", 6379));
        nodes.add(new HostAndPort("192.168.1.2", 6379));
        nodes.add(new HostAndPort("192.168.1.3", 6379));
        
        // 连接池配置
        JedisPoolConfig poolConfig = new JedisPoolConfig();
        poolConfig.setMaxTotal(100);           // 最大连接数
        poolConfig.setMaxIdle(50);             // 最大空闲连接
        poolConfig.setMinIdle(10);             // 最小空闲连接
        poolConfig.setMaxWaitMillis(3000);     // 最大等待时间
        poolConfig.setTestOnBorrow(true);      // 借用时测试连接
        
        // JedisCluster配置
        JedisCluster jedisCluster = new JedisCluster(
            nodes,              // 节点集合
            5000,               // connectionTimeout（连接超时）
            3000,               // soTimeout（读取超时）
            3,                  // maxAttempts（最大重试次数）
            "password123",      // password
            poolConfig          // 连接池配置
        );
        
        return jedisCluster;
    }
}

/**
 * LettuceCluster配置（Spring Boot 2.x+推荐）
 */
@Configuration
public class LettuceClusterConfig {
    
    @Bean
    public RedisConnectionFactory redisConnectionFactory() {
        // 集群节点配置
        RedisClusterConfiguration clusterConfig = new RedisClusterConfiguration(
            Arrays.asList(
                "192.168.1.1:6379",
                "192.168.1.2:6379",
                "192.168.1.3:6379"
            )
        );
        clusterConfig.setMaxRedirects(3);        // 最大重定向次数
        clusterConfig.setPassword("password123");
        
        // 客户端配置
        ClientOptions clientOptions = ClientOptions.builder()
            .autoReconnect(true)                  // 自动重连
            .pingBeforeActivateConnection(true)   // 激活连接前PING
            .timeoutOptions(TimeoutOptions.enabled(Duration.ofSeconds(5)))
            .build();
        
        // 连接池配置
        GenericObjectPoolConfig<Object> poolConfig = new GenericObjectPoolConfig<>();
        poolConfig.setMaxTotal(100);
        poolConfig.setMaxIdle(50);
        poolConfig.setMinIdle(10);
        
        LettuceClientConfiguration clientConfig = LettucePoolingClientConfiguration.builder()
            .commandTimeout(Duration.ofSeconds(5))
            .shutdownTimeout(Duration.ZERO)
            .poolConfig(poolConfig)
            .clientOptions(clientOptions)
            .build();
        
        return new LettuceConnectionFactory(clusterConfig, clientConfig);
    }
    
    @Bean
    public StringRedisTemplate stringRedisTemplate(RedisConnectionFactory factory) {
        return new StringRedisTemplate(factory);
    }
}
```

### 5. 故障转移的完整流程

#### Cluster故障检测

```
Cluster故障检测流程：

┌─────────────────────────────────────────────────────────┐
│  Step 1: 节点间心跳检测                                   │
│                                                         │
│  每个节点每秒向随机几个节点发送PING（默认3个）             │
│                                                         │
│  节点A向节点B发送PING：                                   │
│  - 包含A的已知节点状态信息                                │
│  - 包含A负责的slot信息                                    │
│                                                         │
│  节点B回复PONG：                                          │
│  - 包含B的已知节点状态信息                                │
│                                                         │
├─────────────────────────────────────────────────────────┤
│  Step 2: PFail标记                                        │
│                                                         │
│  如果节点A在cluster-node-timeout内（默认15秒）            │
│  未收到节点B的PONG                                       │
│                                                         │
│  节点A将B标记为 PFail（Probably Fail）                   │
│  并通过Gossip传播给其他节点                                │
│                                                         │
├─────────────────────────────────────────────────────────┤
│  Step 3: Fail确认                                         │
│                                                         │
│  当大多数主节点（>= N/2 + 1）都认为B是PFail时             │
│  B被标记为 Fail（确认故障）                               │
│                                                         │
│  节点A广播FAIL消息给所有可达节点：                         │
│  "节点B已确认故障"                                       │
│                                                         │
│  所有收到FAIL消息的节点立即将B标记为Fail                   │
│                                                         │
├─────────────────────────────────────────────────────────┤
│  Step 4: 故障转移触发                                     │
│                                                         │
│  如果B是从节点：无操作（从节点故障不影响服务）              │
│                                                         │
│  如果B是主节点：                                          │
│  - B的从节点发现B为Fail                                   │
│  - 从节点发起故障转移选举                                  │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

#### 从节点选举与提升

```
从节点选举流程：

┌─────────────────────────────────────────────────────────┐
│  Step 1: 资格检查                                         │
│                                                         │
│  从节点C（原B的从节点）检查自己是否有资格：                │
│  - 是否与主节点B断开连接超过一定时间？                     │
│    （避免网络分区时从节点误提升）                          │
│  - 数据是否足够新？（复制偏移量差距不能太大）               │
│                                                         │
│  如果检查通过，进入选举阶段                                │
│                                                         │
├─────────────────────────────────────────────────────────┤
│  Step 2: 发起选举                                         │
│                                                         │
│  从节点C增加自己的currentEpoch（当前纪元）                 │
│  向所有主节点发送：                                        │
│  "我想成为新主节点，currentEpoch = X"                    │
│                                                         │
├─────────────────────────────────────────────────────────┤
│  Step 3: 主节点投票                                         │
│                                                         │
│  每个主节点收到选举请求后：                                │
│  - 检查currentEpoch是否最新                               │
│  - 检查是否已投过票（每个epoch只能投一票）                  │
│  - 检查从节点数据是否足够新                               │
│                                                         │
│  如果同意：回复ACK                                        │
│  如果不同意：忽略                                         │
│                                                         │
├─────────────────────────────────────────────────────────┤
│  Step 4: 当选新主节点                                     │
│                                                         │
│  如果从节点C获得多数主节点（>= N/2 + 1）的投票            │
│                                                         │
│  C成为新主节点：                                          │
│  1. 更新自己的configEpoch为currentEpoch                   │
│  2. 接管原主节点B的slot                                   │
│  3. 向所有节点发送PONG，宣告自己是新主节点                 │
│                                                         │
├─────────────────────────────────────────────────────────┤
│  Step 5: 更新集群状态                                     │
│                                                         │
│  其他节点收到新主节点的PONG后：                            │
│  - 更新slot-node映射                                      │
│  - 将原主节点B标记为从节点（如果恢复）                     │
│  - 开始向新主节点C发送请求                                 │
│                                                         │
│  故障转移完成！                                           │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

#### 故障转移时间分析

```
故障转移时间构成：

┌─────────────────────────────────────────┐
│  检测时间（Detection Time）              │
│  = cluster-node-timeout                 │
│  默认：15秒                              │
│  （可配置为更短，如5秒，但可能误报）       │
│                                         │
│  选举时间（Election Time）               │
│  = 随机延迟 + 投票时间                    │
│  随机延迟：0-1秒（避免多个从节点同时选举）  │
│  投票时间：<1秒（网络往返）                │
│                                         │
│  切换时间（Switchover Time）             │
│  = 从节点提升为主节点 + 更新配置           │
│  < 1秒                                  │
│                                         │
│  总时间：                                │
│  默认配置：~15秒（主要是检测时间）         │
│  优化配置：~5-10秒                       │
│                                         │
│  优化方法：                              │
│  1. 减小cluster-node-timeout            │
│  2. 部署在同机房，减少网络延迟             │
│  3. 使用SSD，加快RDB加载                 │
└─────────────────────────────────────────┘
```

---

## 模型差异：三种集群模式对比

### 1. 架构对比

```
┌─────────────────────────────────────────────────────────┐
│  主从复制（Replication）                                 │
│                                                         │
│     Master（读写）                                       │
│        │                                                │
│    ┌───┴───┐                                            │
│  Slave1  Slave2（只读）                                  │
│                                                         │
│  特点：简单，手动故障转移，适合读多写少                    │
│  最小节点：2（1主1从）                                    │
│                                                         │
├─────────────────────────────────────────────────────────┤
│  哨兵模式（Sentinel）                                    │
│                                                         │
│       ┌── Sentinel1                                     │
│       │                                                 │
│  Master─┼── Sentinel2                                   │
│       │                                                 │
│       └── Sentinel3                                     │
│           │                                             │
│       ┌───┴───┐                                         │
│     Slave1  Slave2                                      │
│                                                         │
│  特点：自动故障转移，高可用，适合生产环境                  │
│  最小节点：4（1主1从+2哨兵，推荐3哨兵）                   │
│                                                         │
├─────────────────────────────────────────────────────────┤
│  Cluster集群                                             │
│                                                         │
│  MasterA(0-5460) ── SlaveA1                             │
│     │                                                   │
│  MasterB(5461-10922) ── SlaveB1                         │
│     │                                                   │
│  MasterC(10923-16383) ── SlaveC1                        │
│                                                         │
│  特点：数据分片，读写扩展，去中心化，适合大数据量           │
│  最小节点：6（3主3从）                                    │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 2. 功能特性对比

```
┌─────────────────┬─────────────────┬─────────────────┬─────────────────┐
│     特性        │   主从复制       │   哨兵模式       │   Cluster集群   │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ 数据复制        │ 主->从          │ 主->从          │ 主->从          │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ 自动故障转移    │ 否              │ 是              │ 是              │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ 读写分离        │ 是              │ 是              │ 是              │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ 写扩展          │ 否              │ 否              │ 是（分片）       │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ 存储扩展        │ 否              │ 否              │ 是（分片）       │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ 最小节点数      │ 2               │ 4（推荐6）       │ 6               │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ 一致性          │ 最终一致         │ 最终一致         │ 最终一致         │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ 复杂度          │ 低              │ 中              │ 高              │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ 运维成本        │ 低              │ 中              │ 高              │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ 客户端复杂度    │ 低              │ 中              │ 高              │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ 多Key操作       │ 支持            │ 支持            │ 限制（同slot）   │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ 事务支持        │ 支持            │ 支持            │ 限制（同slot）   │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ 数据迁移        │ 手动            │ 手动            │ 自动（reshard）  │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ 适用场景        │ 读多写少         │ 高可用要求       │ 大数据量         │
│                 │ 数据量小         │ 数据量小         │ 需要水平扩展     │
└─────────────────┴─────────────────┴─────────────────┴─────────────────┘
```

### 3. 选择决策树

```
Redis集群模式选择决策树：

┌─────────────────────────────────────────┐
│  问题1：数据量是否超过单节点内存？        │
│                                         │
│  是 → Cluster集群（必须分片）            │
│  否 → 继续问题2                         │
│                                         │
├─────────────────────────────────────────┤
│  问题2：是否需要自动故障转移？            │
│                                         │
│  是 → 继续问题3                         │
│  否 → 主从复制（简单，手动切换）         │
│                                         │
├─────────────────────────────────────────┤
│  问题3：写入量是否很大？                  │
│  （单节点写入成为瓶颈）                   │
│                                         │
│  是 → Cluster集群（多主节点写入）        │
│  否 → 哨兵模式（高可用，简单）           │
│                                         │
├─────────────────────────────────────────┤
│  问题4：是否需要复杂的多Key操作/事务？    │
│                                         │
│  是 → 哨兵模式（无slot限制）             │
│  否 → Cluster集群（推荐）                │
│                                         │
├─────────────────────────────────────────┤
│  问题5：运维能力是否有限？                │
│                                         │
│  是 → 哨兵模式（运维简单）               │
│  否 → Cluster集群（功能更强大）          │
│                                         │
└─────────────────────────────────────────┘
```

---

## 工业级实践案例

### 案例1：电商缓存集群部署

**场景**：大型电商平台，日均PV过亿，缓存数据量500GB

**核心挑战**：
- 高并发：峰值QPS 50万+
- 大数据量：商品、库存、用户会话等500GB
- 高可用：99.99%可用性要求
- 读写比例：读:写 = 100:1

**架构设计**：

```
┌─────────────────────────────────────────────────────────┐
│              电商缓存集群架构                             │
│                                                         │
│   客户端（APP/WEB/H5）                                    │
│      ↓                                                  │
│   ┌─────────────────────────────────────┐               │
│   │        CDN + Nginx 反向代理          │               │
│   └─────────────────────────────────────┘               │
│      ↓                                                  │
│   ┌─────────────────────────────────────┐               │
│   │      Spring Gateway（网关层）         │               │
│   │   - 限流（Sentinel）                 │               │
│   │   - 熔断（Hystrix）                  │               │
│   └─────────────────────────────────────┘               │
│      ↓                                                  │
│   ┌─────────────────────────────────────┐               │
│   │      微服务集群（K8s）                │               │
│   │   - 商品服务                          │               │
│   │   - 订单服务                          │               │
│   │   - 库存服务                          │               │
│   │   - 用户服务                          │               │
│   └─────────────────────────────────────┘               │
│      ↓                                                  │
│   ┌─────────────────────────────────────┐               │
│   │      Redis Cluster 缓存层            │               │
│   │                                     │               │
│   │   ┌─────────┐ ┌─────────┐ ┌─────────┐             │
│   │   │MasterA  │ │MasterB  │ │MasterC  │             │
│   │   │0-5460   │ │5461-10922│ │10923-16383│           │
│   │   └────┬────┘ └────┬────┘ └────┬────┘             │
│   │        │           │           │                   │
│   │   ┌────┴────┐ ┌────┴────┐ ┌────┴────┐             │
│   │   │SlaveA1  │ │SlaveB1  │ │SlaveC1  │             │
│   │   │(热备)   │ │(热备)   │ │(热备)   │             │
│   │   └─────────┘ └─────────┘ └─────────┘             │
│   │                                     │               │
│   │   配置：                            │               │
│   │   - 3主3从，每主节点约170GB数据      │               │
│   │   - 每主节点QPS峰值：15万            │               │
│   │   - 集群总QPS：45万（读）+ 1万（写） │               │
│   └─────────────────────────────────────┘               │
│      ↓                                                  │
│   ┌─────────────────────────────────────┐               │
│   │      持久化层                         │               │
│   │   - MySQL（主库）                     │               │
│   │   - MySQL（从库 × 2）                 │               │
│   │   - Elasticsearch（搜索）             │               │
│   └─────────────────────────────────────┘               │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**配置细节**：

```bash
# redis-cluster.conf（每个节点）

# 基础配置
port 6379
cluster-enabled yes
cluster-config-file nodes-6379.conf
cluster-node-timeout 5000
cluster-require-full-coverage no  # 部分slot不可用时不拒绝所有请求
appendonly yes
appendfsync everysec

# 内存配置
maxmemory 64gb
maxmemory-policy allkeys-lru  # 内存不足时淘汰最近最少使用

# 复制配置
repl-backlog-size 256mb
repl-timeout 60
min-replicas-to-write 1
min-replicas-max-lag 10

# 慢查询日志
slowlog-log-slower-than 10000  # 10ms
slowlog-max-len 128

# 连接配置
tcp-keepalive 300
timeout 0
tcp-backlog 511

# 安全配置
requirepass password123
masterauth password123
```

```java
/**
 * 电商缓存服务
 */
@Service
public class EcommerceCacheService {
    
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    @Autowired
    private RedissonClient redissonClient;
    
    private static final String PRODUCT_KEY_PREFIX = "product:";
    private static final String INVENTORY_KEY_PREFIX = "inventory:";
    private static final String USER_SESSION_PREFIX = "session:";
    private static final long PRODUCT_TTL = 3600;  // 1小时
    private static final long SESSION_TTL = 1800;  // 30分钟
    
    /**
     * 获取商品信息（缓存优先）
     */
    public Product getProduct(Long productId) {
        String key = PRODUCT_KEY_PREFIX + productId;
        
        // 1. 查询缓存
        String json = redisTemplate.opsForValue().get(key);
        if (json != null) {
            return JSON.parseObject(json, Product.class);
        }
        
        // 2. 缓存未命中，查询数据库
        Product product = productRepository.findById(productId)
            .orElse(null);
        
        if (product != null) {
            // 3. 写入缓存（设置TTL）
            redisTemplate.opsForValue().set(
                key, 
                JSON.toJSONString(product),
                PRODUCT_TTL,
                TimeUnit.SECONDS
            );
        }
        
        return product;
    }
    
    /**
     * 扣减库存（分布式锁保证原子性）
     */
    public boolean deductInventory(Long skuId, Integer quantity) {
        String lockKey = "lock:inventory:" + skuId;
        String inventoryKey = INVENTORY_KEY_PREFIX + skuId;
        
        RLock lock = redissonClient.getLock(lockKey);
        
        try {
            boolean locked = lock.tryLock(3, 10, TimeUnit.SECONDS);
            if (!locked) {
                return false;
            }
            
            try {
                // Lua脚本原子扣减
                String script = 
                    "local stock = tonumber(redis.call('get', KEYS[1])); " +
                    "if stock == nil then return -1; end; " +
                    "if stock < tonumber(ARGV[1]) then return -2; end; " +
                    "redis.call('decrby', KEYS[1], ARGV[1]); " +
                    "return redis.call('get', KEYS[1]);";
                
                Long result = redisTemplate.execute(
                    new DefaultRedisScript<>(script, Long.class),
                    Collections.singletonList(inventoryKey),
                    String.valueOf(quantity)
                );
                
                return result != null && result >= 0;
            } finally {
                if (lock.isHeldByCurrentThread()) {
                    lock.unlock();
                }
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return false;
        }
    }
    
    /**
     * 用户会话管理
     */
    public void updateSession(String sessionId, UserSession session) {
        String key = USER_SESSION_PREFIX + sessionId;
        redisTemplate.opsForValue().set(
            key,
            JSON.toJSONString(session),
            SESSION_TTL,
            TimeUnit.SECONDS
        );
    }
    
    /**
     * 缓存预热（活动开始前）
     */
    public void warmupCache(List<Long> productIds) {
        for (Long productId : productIds) {
            Product product = productRepository.findById(productId).orElse(null);
            if (product != null) {
                String key = PRODUCT_KEY_PREFIX + productId;
                redisTemplate.opsForValue().set(
                    key,
                    JSON.toJSONString(product),
                    PRODUCT_TTL,
                    TimeUnit.SECONDS
                );
            }
        }
    }
}
```

### 案例2：社交网络Feed流存储

**场景**：社交APP，用户关注关系、Feed流、时间线

**核心挑战**：
- 写放大：一个用户有1000粉丝，发布一条内容需要写入1000个粉丝的时间线
- 读优化：时间线需要按时间排序，支持分页
- 热点用户：大V发布内容，瞬间大量写入

**架构设计**：

```
┌─────────────────────────────────────────────────────────┐
│              社交网络Feed流架构                           │
│                                                         │
│  写入流程（推模式）：                                     │
│                                                         │
│  用户A发布内容 ──→ 内容服务                              │
│                      ↓                                  │
│              ┌───────────────┐                          │
│              │  写入内容库     │                          │
│              │  content:{id}  │                          │
│              └───────────────┘                          │
│                      ↓                                  │
│              获取粉丝列表（1000人）                       │
│                      ↓                                  │
│              异步写入粉丝时间线（Kafka）                   │
│                      ↓                                  │
│              ┌─────────────────────────────┐            │
│              │   Redis Cluster 时间线存储   │            │
│              │                             │            │
│              │   timeline:user:1  → [c10,c9,c8]       │
│              │   timeline:user:2  → [c10,c7,c6]       │
│              │   timeline:user:3  → [c10,c5,c4]       │
│              │   ...                         │            │
│              │                             │            │
│              │   数据结构：Sorted Set（ZSET）│            │
│              │   Score：发布时间戳            │            │
│              │   Member：内容ID               │            │
│              └─────────────────────────────┘            │
│                                                         │
├─────────────────────────────────────────────────────────┤
│  读取流程：                                               │
│                                                         │
│  用户B刷新首页 ──→ 时间线服务                            │
│                      ↓                                  │
│              ZREVRANGE timeline:user:B 0 20             │
│              （获取最新的20条内容）                        │
│                      ↓                                  │
│              批量查询内容详情（Pipeline）                 │
│                      ↓                                  │
│              返回给客户端                                 │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**代码实现**：

```java
/**
 * 社交Feed流服务
 */
@Service
public class FeedStreamService {
    
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    private static final String TIMELINE_PREFIX = "timeline:user:";
    private static final String CONTENT_PREFIX = "content:";
    private static final int TIMELINE_SIZE = 1000;  // 每个用户最多保存1000条
    
    /**
     * 发布内容（推模式）
     */
    public void publishContent(Long userId, Content content) {
        // 1. 保存内容
        String contentKey = CONTENT_PREFIX + content.getId();
        redisTemplate.opsForValue().set(contentKey, JSON.toJSONString(content));
        
        // 2. 发送异步消息，写入粉丝时间线
        FeedMessage message = new FeedMessage(userId, content.getId(), content.getCreateTime());
        kafkaTemplate.send("feed-push-topic", message);
    }
    
    /**
     * 异步消费：写入粉丝时间线
     */
    @KafkaListener(topics = "feed-push-topic", groupId = "feed-push-group")
    public void pushToFollowers(FeedMessage message) {
        // 获取粉丝列表（从Redis或数据库）
        List<Long> followers = getFollowers(message.getUserId());
        
        // 批量写入时间线（使用Pipeline）
        redisTemplate.executePipelined((RedisCallback<Object>) connection -> {
            for (Long followerId : followers) {
                String timelineKey = TIMELINE_PREFIX + followerId;
                byte[] keyBytes = timelineKey.getBytes();
                byte[] memberBytes = String.valueOf(message.getContentId()).getBytes();
                byte[] scoreBytes = String.valueOf(message.getTimestamp()).getBytes();
                
                // ZADD timeline:user:{id} timestamp contentId
                connection.zAdd(keyBytes, message.getTimestamp(), memberBytes);
                
                // 限制时间线大小（只保留最新的1000条）
                connection.zRemRangeByRank(keyBytes, 0, -TIMELINE_SIZE - 1);
            }
            return null;
        });
    }
    
    /**
     * 获取用户时间线
     */
    public List<Content> getTimeline(Long userId, int page, int size) {
        String timelineKey = TIMELINE_PREFIX + userId;
        int start = (page - 1) * size;
        int end = start + size - 1;
        
        // 获取内容ID列表（按时间倒序）
        Set<String> contentIds = redisTemplate.opsForZSet()
            .reverseRange(timelineKey, start, end);
        
        if (contentIds == null || contentIds.isEmpty()) {
            return Collections.emptyList();
        }
        
        // 批量查询内容详情
        List<String> contentKeys = contentIds.stream()
            .map(id -> CONTENT_PREFIX + id)
            .collect(Collectors.toList());
        
        List<String> contentJsonList = redisTemplate.opsForValue()
            .multiGet(contentKeys);
        
        return contentJsonList.stream()
            .filter(Objects::nonNull)
            .map(json -> JSON.parseObject(json, Content.class))
            .collect(Collectors.toList());
    }
    
    /**
     * 获取粉丝列表（简化版）
     */
    private List<Long> getFollowers(Long userId) {
        // 实际实现：从Redis Set或数据库查询
        String followersKey = "followers:user:" + userId;
        Set<String> followers = redisTemplate.opsForSet().members(followersKey);
        
        if (followers == null) {
            return Collections.emptyList();
        }
        
        return followers.stream()
            .map(Long::valueOf)
            .collect(Collectors.toList());
    }
}
```

**优化策略**：

```
1. 大V用户优化（拉模式混合）
   普通用户（<1000粉丝）：推模式（写入粉丝时间线）
   大V用户（>=1000粉丝）：拉模式（粉丝主动拉取大V内容）
   
   混合模式减少写放大：
   - 大V发布内容只写入自己的发件箱
   - 粉丝刷新时，合并自己的收件箱 + 关注大V的发件箱

2. 时间线分片
   - 按时间分片：timeline:user:{id}:2024-01
   - 减少单个key的大小
   - 便于清理历史数据

3. 预热机制
   - 用户登录时预加载最近时间线
   - 使用Pipeline批量获取

4. 本地缓存
   - Caffeine本地缓存热门内容
   - 减少Redis访问压力
```

### 案例3：金融交易会话存储

**场景**：金融交易系统，用户登录会话、交易限频、风控标记

**核心挑战**：
- 强一致性：会话数据不能丢失
- 高可用：99.999%可用性
- 安全性：敏感数据加密存储
- 审计：所有操作可追溯

**架构设计**：

```
┌─────────────────────────────────────────────────────────┐
│              金融交易会话存储架构                         │
│                                                         │
│  客户端（APP/PC/交易终端）                                │
│      ↓                                                  │
│   ┌─────────────────────────────────────┐               │
│   │        API Gateway（Kong/APISIX）    │               │
│   │   - SSL终止                          │               │
│   │   - 限流                             │               │
│   │   - 鉴权                             │               │
│   └─────────────────────────────────────┘               │
│      ↓                                                  │
│   ┌─────────────────────────────────────┐               │
│   │      交易服务集群（K8s）              │               │
│   │   - 登录服务                          │               │
│   │   - 交易服务                          │               │
│   │   - 风控服务                          │               │
│   └─────────────────────────────────────┘               │
│      ↓                                                  │
│   ┌─────────────────────────────────────┐               │
│   │      Redis哨兵集群（会话存储）        │               │
│   │                                     │               │
│   │   选择哨兵而非Cluster的原因：          │               │
│   │   1. 会话数据量不大（<10GB）          │               │
│   │   2. 需要完整的事务支持               │               │
│   │   3. 多Key操作频繁（用户+会话+限频）   │               │
│   │   4. 运维简单，故障转移快             │               │
│   │                                     │               │
│   │   架构：                              │               │
│   │   Master（读写）                      │               │
│   │      │                              │               │
│   │   Slave1（只读，热备）                │               │
│   │   Slave2（只读，热备）                │               │
│   │                                     │               │
│   │   Sentinel × 3（监控+自动切换）       │               │
│   │                                     │               │
│   └─────────────────────────────────────┘               │
│      ↓                                                  │
│   ┌─────────────────────────────────────┐               │
│   │      持久化层                         │               │
│   │   - MySQL（主从）                     │               │
│   │   - 审计日志（ELK）                   │               │
│   └─────────────────────────────────────┘               │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**代码实现**：

```java
/**
 * 金融交易会话服务
 */
@Service
public class TradingSessionService {
    
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    @Autowired
    private RedissonClient redissonClient;
    
    private static final String SESSION_PREFIX = "trading:session:";
    private static final String RATE_LIMIT_PREFIX = "trading:ratelimit:";
    private static final String RISK_MARK_PREFIX = "trading:risk:";
    private static final long SESSION_TTL = 1800;  // 30分钟
    private static final long RATE_LIMIT_WINDOW = 60;  // 60秒窗口
    
    /**
     * 创建交易会话
     */
    public Session createSession(Long userId, String deviceId, String ip) {
        String sessionId = generateSessionId();
        String sessionKey = SESSION_PREFIX + sessionId;
        
        Session session = new Session();
        session.setSessionId(sessionId);
        session.setUserId(userId);
        session.setDeviceId(deviceId);
        session.setIp(ip);
        session.setCreateTime(System.currentTimeMillis());
        session.setExpireTime(System.currentTimeMillis() + SESSION_TTL * 1000);
        
        // 加密存储敏感信息
        String encryptedSession = encryptSession(session);
        
        // 写入Redis（带TTL）
        redisTemplate.opsForValue().set(
            sessionKey,
            encryptedSession,
            SESSION_TTL,
            TimeUnit.SECONDS
        );
        
        // 记录审计日志
        auditLog.info("Session created: userId={}, sessionId={}, ip={}", 
            userId, sessionId, ip);
        
        return session;
    }
    
    /**
     * 验证会话（限频检查）
     */
    public boolean validateSession(String sessionId, String action) {
        String sessionKey = SESSION_PREFIX + sessionId;
        
        // 1. 检查会话是否存在
        String encryptedSession = redisTemplate.opsForValue().get(sessionKey);
        if (encryptedSession == null) {
            return false;
        }
        
        // 2. 解密会话
        Session session = decryptSession(encryptedSession);
        
        // 3. 检查风控标记
        String riskKey = RISK_MARK_PREFIX + session.getUserId();
        Boolean isRisky = redisTemplate.opsForSet().isMember(riskKey, action);
        if (Boolean.TRUE.equals(isRisky)) {
            log.warn("Risky action blocked: userId={}, action={}", 
                session.getUserId(), action);
            return false;
        }
        
        // 4. 限频检查（滑动窗口）
        String rateLimitKey = RATE_LIMIT_PREFIX + session.getUserId() + ":" + action;
        Long currentCount = redisTemplate.opsForValue().increment(rateLimitKey);
        
        if (currentCount != null && currentCount == 1) {
            // 第一次访问，设置窗口过期时间
            redisTemplate.expire(rateLimitKey, RATE_LIMIT_WINDOW, TimeUnit.SECONDS);
        }
        
        if (currentCount != null && currentCount > getRateLimit(action)) {
            log.warn("Rate limit exceeded: userId={}, action={}, count={}", 
                session.getUserId(), action, currentCount);
            return false;
        }
        
        // 5. 刷新会话TTL
        redisTemplate.expire(sessionKey, SESSION_TTL, TimeUnit.SECONDS);
        
        return true;
    }
    
    /**
     * 销毁会话
     */
    public void destroySession(String sessionId) {
        String sessionKey = SESSION_PREFIX + sessionId;
        
        // 获取会话信息（用于审计）
        String encryptedSession = redisTemplate.opsForValue().get(sessionKey);
        if (encryptedSession != null) {
            Session session = decryptSession(encryptedSession);
            auditLog.info("Session destroyed: userId={}, sessionId={}", 
                session.getUserId(), sessionId);
        }
        
        // 删除会话
        redisTemplate.delete(sessionKey);
    }
    
    /**
     * 获取限频阈值
     */
    private int getRateLimit(String action) {
        switch (action) {
            case "login": return 5;      // 登录：5次/分钟
            case "trade": return 10;     // 交易：10次/分钟
            case "transfer": return 3;   // 转账：3次/分钟
            default: return 100;
        }
    }
    
    private String generateSessionId() {
        return UUID.randomUUID().toString().replace("-", "");
    }
    
    private String encryptSession(Session session) {
        // 使用AES加密敏感信息
        String json = JSON.toJSONString(session);
        return AESUtils.encrypt(json, sessionKey);
    }
    
    private Session decryptSession(String encrypted) {
        String json = AESUtils.decrypt(encrypted, sessionKey);
        return JSON.parseObject(json, Session.class);
    }
}
```

**安全与合规**：

```
1. 数据加密
   - 传输加密：TLS 1.3
   - 存储加密：AES-256
   - 密钥管理：KMS/HSM

2. 访问控制
   - Redis密码 + ACL（Redis 6.x+）
   - 网络隔离：VPC/安全组
   - IP白名单

3. 审计日志
   - 所有会话操作记录
   - 保留时间：180天
   - 存储：ELK Stack

4. 备份策略
   - RDB：每小时备份
   - AOF：每秒fsync
   - 跨地域备份

5. 故障演练
   - 每季度模拟主节点故障
   - 验证自动切换时间
   - 验证数据一致性
```

---

## 性能分析与压测数据

### 1. 三种集群模式性能对比

```
压测环境：
- 服务器：AWS r6g.2xlarge（8核64G）
- 网络：10Gbps内网
- 客户端：Java Jedis/Lettuce，1000并发线程
- 数据大小：Value = 1KB

┌─────────────────┬─────────────────┬─────────────────┬─────────────────┐
│     指标        │   主从复制       │   哨兵模式       │   Cluster集群   │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ 读QPS（单节点）  │ 100,000        │ 100,000        │ 100,000        │
│ 读QPS（集群）    │ 300,000        │ 300,000        │ 600,000        │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ 写QPS（单节点）  │ 80,000         │ 80,000         │ 80,000         │
│ 写QPS（集群）    │ 80,000         │ 80,000         │ 240,000        │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ 平均读延迟       │ 0.5ms          │ 0.5ms          │ 0.6ms          │
│ P99读延迟        │ 2ms            │ 2ms            │ 3ms            │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ 平均写延迟       │ 0.8ms          │ 0.8ms          │ 0.9ms          │
│ P99写延迟        │ 3ms            │ 3ms            │ 5ms            │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ 故障转移时间     │ 手动（分钟级）   │ 自动（~30秒）   │ 自动（~15秒）   │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ 数据复制延迟     │ <1ms           │ <1ms           │ <1ms           │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ 内存利用率       │ 100%           │ 100%           │ 60-70%（分片）  │
└─────────────────┴─────────────────┴─────────────────┴─────────────────┘
```

### 2. Cluster集群性能调优

```
调优方向1：连接池配置
┌─────────────────────────────────────┐
│  Lettuce连接池优化：                 │
│  - maxTotal: 100-200（每节点）       │
│  - maxIdle: 50-100                   │
│  - minIdle: 10-20                    │
│  - maxWaitMillis: 3000               │
│                                     │
│  原理：                              │
│  - Cluster有多个节点                  │
│  - 每个节点需要独立连接池              │
│  - 总连接数 = 节点数 × 单节点连接数   │
│  - 避免连接数过多导致Redis拒绝         │
└─────────────────────────────────────┘

调优方向2：Pipeline批量操作
┌─────────────────────────────────────┐
│  使用Pipeline减少网络RTT：            │
│                                     │
│  错误：                              │
│  for (key : keys) {                 │
│      redis.get(key);  // N次网络往返 │
│  }                                  │
│                                     │
│  正确：                              │
│  redis.executePipelined((conn) -> { │
│      for (key : keys) {             │
│          conn.get(key.getBytes());  │
│      }                              │
│      return null;                   │
│  });                                │
│  // 1次网络往返                      │
│                                     │
│  效果：N个命令从N次RTT降为1次         │
│  适合：批量读取、批量写入             │
└─────────────────────────────────────┘

调优方向3：Hash Tag减少跨节点操作
┌─────────────────────────────────────┐
│  使用Hash Tag强制相关Key同节点：      │
│                                     │
│  错误：                              │
│  user:1000:profile                  │
│  user:1000:orders                   │
│  user:1000:settings                 │
│  // 可能分布在不同节点                 │
│  // MGET操作报错：CROSSSLOT          │
│                                     │
│  正确：                              │
│  {user:1000}:profile                │
│  {user:1000}:orders                 │
│  {user:1000}:settings               │
│  // 只计算{}内的内容                  │
│  // CRC16("user:1000") → 同一slot   │
│  // 同一节点，支持MGET/事务            │
│                                     │
│  注意：                              │
│  - 可能导致数据倾斜                   │
│  - 大用户的数据都集中在一个节点        │
│  - 需要监控节点负载                   │
└─────────────────────────────────────┘

调优方向4：读写分离
┌─────────────────────────────────────┐
│  配置读写分离：                       │
│                                     │
│  Lettuce配置：                       │
│  - readFrom: ReadFrom.REPLICA       │
│    读请求优先路由到从节点              │
│  - readFrom: ReadFrom.ANY           │
│    读请求路由到任意节点                │
│                                     │
│  效果：                              │
│  - 读请求分摊到从节点                  │
│  - 主节点专注写操作                    │
│  - 总吞吐量提升50-100%                │
│                                     │
│  注意：                              │
│  - 从节点数据可能有延迟                │
│  - 强一致性读需要路由到主节点          │
│  - 配置：ReadFrom.MASTER             │
└─────────────────────────────────────┘
```

### 3. 扩容缩容操作

```bash
# Cluster扩容（添加新节点）

# Step 1: 启动新节点
redis-server /etc/redis/redis-new.conf
# 配置：
# port 6385
# cluster-enabled yes
# cluster-config-file nodes-6385.conf
# cluster-node-timeout 5000

# Step 2: 添加节点到集群
redis-cli --cluster add-node 192.168.1.4:6385 192.168.1.1:6379
# 第一个IP：新节点
# 第二个IP：集群中任意已有节点

# Step 3: 分配Slot
redis-cli --cluster reshard 192.168.1.1:6379
# 交互式输入：
# - 迁移多少个slot？（例如4096，每个节点平均分配）
# - 接收slot的节点ID？（新节点ID）
# - 从哪些节点迁移？（all表示所有主节点）

# Step 4: 添加从节点（可选）
redis-cli --cluster add-node 192.168.1.4:6386 192.168.1.1:6379 \
  --cluster-slave --cluster-master-id <new-master-id>
```

```bash
# Cluster缩容（移除节点）

# Step 1: 迁移slot（将数据迁出）
redis-cli --cluster reshard 192.168.1.1:6379
# 交互式输入：
# - 迁移多少个slot？（该节点持有的全部slot）
# - 接收slot的节点ID？（其他主节点）
# - 从哪个节点迁移？（要移除的节点ID）

# Step 2: 移除从节点（如果有）
redis-cli --cluster del-node 192.168.1.1:6379 <slave-node-id>

# Step 3: 移除主节点
redis-cli --cluster del-node 192.168.1.1:6379 <master-node-id>
```

---

## 常见陷阱与最佳实践

### 陷阱1：Cluster模式下误用多Key操作

```bash
# 错误：MGET的Key可能在不同Slot
MGET user:1000 order:2000 product:3000
# 报错：(error) CROSSSLOT Keys in request don't hash to the same slot

# 正确：使用Hash Tag强制同Slot
MGET {user:1000}:profile {user:1000}:orders {user:1000}:settings

# 或者使用Redis Cluster Proxy
# 代理层自动将多Key操作拆分为单Key操作
```

**最佳实践：**
- 多Key操作确保在同一Slot（使用Hash Tag）
- 或使用 `redis-cli --cluster call` 分别执行
- 复杂事务考虑用Lua脚本，但需保证Key同节点

### 陷阱2：忽视主从复制延迟

```java
// 错误：写后立即读从节点
redisTemplate.opsForValue().set("key", "value");
// 从节点可能还没同步
redisTemplate.opsForValue().get("key");  // 可能读到旧值

// 正确：读要求强一致性时，强制读主节点
// Lettuce配置：
LettuceClientConfiguration clientConfig = LettucePoolingClientConfiguration.builder()
    .readFrom(ReadFrom.MASTER)  // 只读主节点
    .build();

// 或者使用Sentinel/Cluster的读写分离策略
// 写操作：路由到主节点
// 读操作：根据一致性要求选择主节点或从节点
```

**最佳实践：**
- 读要求强一致性时，强制读主节点
- 监控 `master_repl_offset` 和 `slave_repl_offset` 差异
- 关键业务设置 `min-replicas-to-write 1`

### 陷阱3：哨兵/Cluster脑裂

```
网络分区导致脑裂：

┌─────────────────────────────────────────┐
│  场景：网络分区                           │
│                                         │
│  机房A          X         机房B          │
│  (主节点A)    网络断开    (Sentinel1,2)  │
│                                         │
│  Sentinel1,2认为主节点A宕机              │
│  选举从节点B为新主节点                    │
│                                         │
│  此时：                                   │
│  - 机房A：主节点A仍在服务写请求           │
│  - 机房B：新主节点B也接受写请求           │
│                                         │
│  网络恢复后：                             │
│  - 主节点A变成从节点                      │
│  - A上的写数据丢失（被B的数据覆盖）        │
│                                         │
│  这就是脑裂（Split-Brain）                │
└─────────────────────────────────────────┘

解决方案：
1. 配置min-slaves-to-write 1
   主节点至少要有1个从节点连接，才接受写请求
   
2. 配置min-slaves-max-lag 10
   从节点延迟不能超过10秒
   
3. Cluster配置cluster-node-timeout适当值
   不要太短（避免误报），不要太长（故障恢复慢）
   
4. 部署时哨兵/节点分布在不同网络区域
   避免单点网络故障导致误判
```

### 陷阱4：大Key导致阻塞

```
大Key问题：

场景：
- Hash有100万个field
- ZSET有1000万个member
- List长度100万

危害：
1. DEL大Key：阻塞Redis主线程数秒
2. HGETALL/ZRANGE：返回大量数据，网络拥塞
3. 主从复制：RDB生成和传输耗时
4. 内存分配：导致内存碎片

诊断：
redis-cli --bigkeys
# 扫描所有key，输出每个类型的最大key

redis-cli --memkeys
# 扫描所有key，输出内存占用最大的key

解决方案：
1. 拆分大Key
   user:1000:followers → 拆分为多个小Hash
   user:1000:followers:0-999
   user:1000:followers:1000-1999
   
2. 渐进式删除
   不用DEL，用UNLINK（Redis 4.0+）
   UNLINK在后台线程删除，不阻塞主线程
   
3. 限制返回数量
   HSCAN替代HGETALL
   ZRANGEBYSCORE with LIMIT
   LRANGE with small range
   
4. 监控告警
   设置key大小阈值（如1MB）
   超过阈值告警
```

### 陷阱5：AOF重写导致性能抖动

```
AOF重写（BGREWRITEAOF）问题：

触发条件：
- 手动执行BGREWRITEAOF
- 自动触发：aof_current_size > aof_base_size * auto-aof-rewrite-percentage

过程：
1. fork子进程（类似BGSAVE）
2. 子进程重写AOF文件
3. 期间主进程的写命令同时写入：
   - 原AOF文件
   - AOF重写缓冲区

问题：
1. fork操作阻塞主线程（内存越大阻塞越久）
2. 重写期间内存使用翻倍（Copy-on-Write）
3. 磁盘IO激增

优化方案：
1. 配置合理的重写阈值
   auto-aof-rewrite-percentage 100
   auto-aof-rewrite-min-size 64mb
   
2. 使用SSD磁盘
   HDD磁盘IO成为瓶颈
   
3. 配置no-appendfsync-on-rewrite yes
   重写期间不执行fsync，减少IO压力
   （可能丢失1秒数据）
   
4. 在低峰期手动触发重写
   避免高峰期自动触发
```

### 最佳实践总结

```
1. 选择合适的集群模式
   - 数据量<10GB，读多写少：主从复制
   - 高可用要求，数据量<50GB：哨兵模式
   - 大数据量，需要水平扩展：Cluster集群

2. 合理的节点配置
   - 单节点内存 < 64GB
   - 单节点QPS < 10万（留有余量）
   - 主从比例 1:1 或 1:2

3. 连接池优化
   - 根据节点数调整连接池大小
   - 启用连接检测（testOnBorrow）
   - 配置合理的超时时间

4. 数据设计
   - Key命名规范：{service}:{resource}:{id}
   - Value大小控制：<10KB
   - TTL设置：避免key永久存在
   - Hash Tag使用：谨慎，避免数据倾斜

5. 监控告警
   - 节点存活状态
   - 内存使用率
   - QPS和延迟（P99）
   - 主从复制延迟
   - 慢查询日志
   - 集群slot平衡度

6. 备份策略
   - RDB：每小时备份（低峰期）
   - AOF：每秒fsync（everysec）
   - 跨地域备份（容灾）
   - 定期恢复演练

7. 安全加固
   - 启用密码认证
   - 配置ACL（Redis 6.x+）
   - 网络隔离（VPC/安全组）
   - TLS加密传输
   - 禁用危险命令（FLUSHALL, KEYS等）

8. 故障演练
   - 每季度模拟主节点故障
   - 验证自动切换时间
   - 验证数据一致性
   - 验证客户端重连机制
```

---

## 面试题与参考答案

### Q1：主从复制的原理是什么？全量同步和增量同步有什么区别？

**参考答案：**

```
主从复制的原理：

1. 连接建立：
   从节点向主节点发送SYNC/PSYNC命令
   主节点回复+FULLRESYNC或+CONTINUE

2. 全量同步（Full Resynchronization）：
   - 触发条件：首次连接、复制偏移量超出缓冲区、runid不匹配
   - 过程：
     a. 主节点fork子进程执行BGSAVE，生成RDB文件
     b. 主节点将RDB文件发送给从节点
     c. 从节点清空数据，加载RDB
     d. 主节点将复制缓冲区中的写命令发送给从节点
   - 特点：数据量大，耗时较长，阻塞主线程（fork时）

3. 增量同步（Partial Resynchronization）：
   - 触发条件：从节点断线重连，且复制偏移量在缓冲区内
   - 过程：
     a. 从节点发送PSYNC runid offset
     b. 主节点检查runid和offset
     c. 如果满足条件，发送+CONTINUE
     d. 主节点发送复制缓冲区中offset之后的命令
   - 特点：数据量小，速度快，无阻塞

4. 命令传播（Command Propagation）：
   - 同步完成后，主节点每执行一个写命令，发送给从节点
   - 从节点执行相同命令，保持数据一致
```

### Q2：哨兵如何实现故障转移？主观下线和客观下线的区别？

**参考答案：**

```
哨兵故障转移流程：

1. 监控（Monitoring）：
   Sentinel每秒向主从节点发送PING命令
   检查节点是否存活

2. 主观下线（Subjectively Down, SDown）：
   单个Sentinel认为节点不可用
   条件：在down-after-milliseconds内未收到PONG
   例如：sentinel down-after-milliseconds mymaster 5000

3. 客观下线（Objectively Down, ODown）：
   多个Sentinel达成共识，认为主节点不可用
   条件：同意SDown的Sentinel数量 >= quorum
   例如：sentinel monitor mymaster 192.168.1.100 6379 2
   （quorum=2，至少2个Sentinel同意）

4. 选举Leader Sentinel：
   Sentinel之间通过Raft算法选举Leader
   获得半数以上投票的Sentinel成为Leader
   负责执行故障转移

5. 选择新主节点：
   从从节点中选择：
   - 优先级（replica-priority）越小越优先
   - 复制偏移量越大越优先（数据越新）
   - Run ID越小越优先

6. 故障转移：
   - 提升选中的从节点为新主节点（SLAVEOF NO ONE）
   - 让其他从节点复制新主节点（SLAVEOF new_master）
   - 更新Sentinel配置
   - 通知客户端（发布订阅）

主观下线 vs 客观下线：
- 主观下线：单个Sentinel的判断，可能误判（网络抖动）
- 客观下线：多个Sentinel达成共识，避免误判
- 只有客观下线才会触发故障转移
```

### Q3：Cluster如何分配数据？Slot是什么？

**参考答案：**

```
Cluster数据分配原理：

1. Slot（槽）：
   - Redis Cluster将数据分为16384个slot（0-16383）
   - 为什么是16384？
     * 2^14 = 16384，14位可以表示
     * 心跳消息中需要携带slot分配信息（bitmap）
     * 16384个bit = 2KB，适合心跳包大小
     * 太多slot增加元数据开销，太少降低分片精度

2. Key到Slot的映射：
   slot = CRC16(key) % 16384
   
   CRC16算法：
   - 多项式：x^16 + x^12 + x^5 + 1
   - 结果：16位无符号整数

3. Slot到节点的映射：
   - 每个主节点负责一部分slot
   - 例如3主节点：
     MasterA: 0-5460（5461 slots）
     MasterB: 5461-10922（5462 slots）
     MasterC: 10923-16383（5461 slots）

4. Hash Tag（强制同slot）：
   key = "{user:1000}:profile"
   只计算{}内的内容：CRC16("user:1000")
   保证相关key在同一slot，支持多key操作

5. 客户端路由：
   - 客户端缓存slot-node映射
   - 请求时直接路由到正确节点
   - 节点变化时通过MOVED/ASK重定向更新映射
```

### Q4：Cluster扩容缩容的过程是什么？数据如何迁移？

**参考答案：**

```
Cluster扩容过程：

1. 添加新节点：
   redis-cli --cluster add-node new_ip:new_port existing_ip:existing_port
   
2. 分配Slot：
   redis-cli --cluster reshard existing_ip:existing_port
   - 输入要迁移的slot数量
   - 输入接收slot的节点ID（新节点）
   - 输入源节点ID（或all表示所有节点）

3. 添加从节点（可选）：
   redis-cli --cluster add-node new_slave_ip:port existing_ip:port \
     --cluster-slave --cluster-master-id new_master_id

数据迁移过程（Slot迁移）：

1. 目标节点准备接收slot：
   CLUSTER SETSLOT slot IMPORTING source_node_id

2. 源节点准备发送slot：
   CLUSTER SETSLOT slot MIGRATING target_node_id

3. 迁移数据：
   - 对slot中的每个key：
     a. 在源节点执行DUMP key（序列化）
     b. 在目标节点执行RESTORE key ttl serialized_value
     c. 在源节点删除key
   
   - 迁移期间：
     客户端访问该slot的key：
     源节点回复ASK重定向到目标节点
     客户端发送ASKING后访问目标节点

4. 迁移完成：
   CLUSTER SETSLOT slot NODE target_node_id
   所有节点更新slot分配

Cluster缩容过程：

1. 迁移slot（将数据迁出）：
   redis-cli --cluster reshard existing_ip:existing_port
   - 输入要迁移的slot数量（该节点持有的全部）
   - 输入接收slot的节点ID（其他节点）
   - 输入源节点ID（要移除的节点）

2. 移除从节点（如果有）：
   redis-cli --cluster del-node existing_ip:existing_port slave_node_id

3. 移除主节点：
   redis-cli --cluster del-node existing_ip:existing_port master_node_id
```

### Q5：主从、哨兵、Cluster三种模式如何选择？

**参考答案：**

```
三种模式的选择依据：

┌─────────────────┬─────────────────┬─────────────────┬─────────────────┐
│     维度        │   主从复制       │   哨兵模式       │   Cluster集群   │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ 自动故障转移    │ 否（手动）       │ 是（自动）       │ 是（自动）       │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ 写扩展          │ 否              │ 否              │ 是（多主节点）   │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ 存储扩展        │ 否              │ 否              │ 是（数据分片）   │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ 最小节点数      │ 2               │ 4（推荐6）       │ 6               │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ 多Key操作       │ 支持            │ 支持            │ 限制（同slot）   │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ 事务支持        │ 支持            │ 支持            │ 限制（同slot）   │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ 复杂度          │ 低              │ 中              │ 高              │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ 适用场景        │ 读多写少         │ 高可用要求       │ 大数据量         │
│                 │ 数据量<10GB     │ 数据量<50GB    │ 需要水平扩展     │
└─────────────────┴─────────────────┴─────────────────┴─────────────────┘

选择建议：

1. 读多写少，数据量小（<10GB），可用性要求不高
   → 主从复制（简单，成本低）

2. 读多写少，数据量小（<50GB），高可用要求
   → 哨兵模式（自动故障转移，运维简单）

3. 读写都多，数据量大（>50GB），需要水平扩展
   → Cluster集群（数据分片，读写扩展）

4. 需要复杂的多Key操作/事务
   → 哨兵模式（无slot限制）
   或 Cluster + Hash Tag（强制同slot）

5. 云原生环境（K8s）
   → Cluster集群（与K8s StatefulSet配合）
   或 云托管Redis（AWS ElastiCache等）
```

### Q6：Redis Cluster脑裂问题如何解决？

**参考答案：**

```
脑裂（Split-Brain）问题：

场景：网络分区导致集群分裂为两个独立部分
- 分区A：包含原主节点和部分节点
- 分区B：包含大部分Sentinel/节点

结果：
- 分区B选举新主节点
- 原主节点仍在分区A接受写请求
- 网络恢复后，原主节点变成从节点，数据丢失

解决方案：

1. 配置min-slaves-to-write
   min-slaves-to-write 1
   主节点至少要有1个从节点连接，才接受写请求
   网络分区后，原主节点没有从节点连接，拒绝写请求

2. 配置min-slaves-max-lag
   min-slaves-max-lag 10
   从节点复制延迟不能超过10秒
   延迟过大的从节点不计入min-slaves

3. Cluster配置cluster-node-timeout
   cluster-node-timeout 5000
   不要太短（避免误报），不要太长（故障恢复慢）
   建议：5-15秒

4. 部署策略
   - 哨兵/节点分布在不同网络区域
   - 避免单点网络故障导致误判
   - 使用独立的网络链路

5. 监控告警
   - 监控节点连通性
   - 发现脑裂立即告警
   - 人工介入处理
```

### Q7：Redis Cluster的Gossip协议有什么优缺点？

**参考答案：**

```
Gossip协议的优点：

1. 去中心化
   - 没有单点故障
   - 所有节点对等，无特殊角色（除主从）
   - 任何节点都可以处理请求

2. 可扩展性
   - 新节点加入只需认识一个现有节点
   - 信息自动传播到整个集群
   - 时间复杂度O(log N)，适合大规模集群

3. 容错性
   - 部分节点故障不影响信息传播
   - 信息通过多条路径传播
   - 最终一致性保证

4. 简单高效
   - 实现简单，无需复杂的协调算法
   - 心跳包大小固定（2KB）
   - 网络开销小

Gossip协议的缺点：

1. 收敛速度慢
   - 信息传播需要时间
   - 新节点加入后，需要数秒才能被所有节点知道
   - 节点故障检测延迟（默认15秒）

2. 消息冗余
   - 同一信息可能被多次传播
   - 网络带宽有一定浪费
   - 消息复杂度O(N log N)

3. 一致性弱
   - 各节点视图可能暂时不一致
   - 需要处理冲突（epoch机制）
   - 不适合强一致性场景

优化方案：
- 调整cluster-node-timeout（权衡检测速度和误报）
- 控制集群规模（建议不超过100个节点）
- 结合其他协议（如Redis 7.x的Cluster Bus优化）
```

### Q8：Redis持久化RDB和AOF的区别？如何选择？

**参考答案：**

```
RDB（Redis Database）：

原理：
- 定时生成内存快照（二进制文件）
- fork子进程执行BGSAVE
- 子进程将内存数据写入RDB文件

优点：
- 文件紧凑，体积小
- 恢复速度快（直接加载二进制）
- 适合备份和灾难恢复

缺点：
- 可能丢失最后一次快照后的数据
- fork操作阻塞主线程（内存越大阻塞越久）
- 频繁快照影响性能

配置：
save 900 1      # 900秒内至少1次修改
save 300 10     # 300秒内至少10次修改
save 60 10000   # 60秒内至少10000次修改

AOF（Append Only File）：

原理：
- 记录每个写命令（类似MySQL binlog）
- 命令以Redis协议格式追加到文件
- 重启时重新执行命令恢复数据

优点：
- 数据安全性高（最多丢失1秒数据）
- 可读的日志格式
- 支持增量备份

缺点：
- 文件体积大（记录每个命令）
- 恢复速度慢（需要重新执行命令）
- 重写（rewrite）时影响性能

配置：
appendonly yes
appendfsync everysec  # 每秒fsync（推荐）
# appendfsync always  # 每次写都fsync（最安全，最慢）
# appendfsync no      # 由OS决定（最快，最不安全）

选择建议：

1. 数据安全性要求高（如金融）：
   RDB + AOF（同时启用）
   AOF everysec，RDB每小时

2. 性能要求高（如缓存）：
   只启用RDB
   或 AOF no（让OS决定fsync）

3. 大数据量（>64GB）：
   只启用RDB
   AOF重写太慢，影响性能

4. 云托管Redis：
   通常默认RDB + AOF
   由云厂商管理持久化

混合持久化（Redis 4.0+）：
- RDB文件 + AOF增量日志
- 恢复时先加载RDB，再执行AOF增量
- 兼顾恢复速度和数据安全
```

---

*此文原创，转载请注明出处。*
