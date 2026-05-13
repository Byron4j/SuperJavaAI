# Redis分布式锁深度解析：从SETNX到RedLock的工业级实践

**文章标签：** #redis #分布式锁 #redisson #redlock #分布式系统 #java #面试

## 目录

- [引言：分布式锁的本质](#引言分布式锁的本质)
- [理论基础：为什么需要分布式锁](#理论基础为什么需要分布式锁)
- [来龙去脉：分布式锁的发展史](#来龙去脉分布式锁的发展史)
- [核心原理深度解析](#核心原理深度解析)
  - [SETNX 的底层原理与缺陷](#setnx-的底层原理与缺陷)
  - [Redisson 可重入锁的实现](#redisson-可重入锁的实现)
  - [看门狗机制的设计哲学](#看门狗机制的设计哲学)
  - [RedLock 算法的数学推导](#redlock-算法的数学推导)
- [模型差异：不同分布式锁方案对比](#模型差异不同分布式锁方案对比)
- [工业级实践案例](#工业级实践案例)
  - [案例1：电商库存扣减系统](#案例1电商库存扣减系统)
  - [案例2：分布式任务调度](#案例2分布式任务调度)
  - [案例3：金融交易幂等控制](#案例3金融交易幂等控制)
- [性能分析与压测数据](#性能分析与压测数据)
- [常见陷阱与最佳实践](#常见陷阱与最佳实践)
- [面试题与参考答案](#面试题与参考答案)

---

## 引言：分布式锁的本质

分布式锁（Distributed Lock）不是"在多台机器上加锁"的简单概念，而是一门**在分布式环境下保证互斥访问共享资源**的共识工程。

核心认知：

```
单机锁的本质：同一JVM内的线程互斥
  synchronized(lock) { /* 临界区 */ }

分布式锁的本质：跨进程、跨机器、跨网络的互斥
  distributedLock.lock() { /* 分布式临界区 */ }

质量差异的根源：
- 差的分布式锁：仅实现基本互斥，无容错、无续期、无监控
- 好的分布式锁：可重入、自动续期、故障转移、可观测
```

**关键洞察**：分布式锁的效果不取决于"能不能锁住"，而取决于**在故障场景下（网络分区、节点宕机、GC暂停）是否仍能保证安全性**。

---

## 理论基础：为什么需要分布式锁

### 1. 分布式系统的CAP困境

#### 一致性与可用性的权衡

```
CAP定理在分布式锁中的体现：

┌─────────────────────────────────────────┐
│              CAP 三角                    │
│                                         │
│         Consistency（一致性）             │
│                /\                       │
│               /  \                      │
│              /    \                     │
│             /      \                    │
│            /   LOCK  \                  │
│           /            \                │
│   Availability ─────── Partition        │
│   （可用性）            （分区容错）       │
│                                         │
│ 分布式锁必须在 CP 和 AP 之间做选择：       │
│ - CP方案：ZooKeeper/ETCD（强一致，牺牲可用性）│
│ - AP方案：Redis（高可用，最终一致）         │
└─────────────────────────────────────────┘
```

**关键理解**：
- Redis分布式锁是**AP系统**上的**CP模拟**
- 通过过期时间、随机value、Lua原子脚本，在最终一致系统上实现"近似强一致"
- 这与ZooKeeper基于ZAB协议的天然强一致有本质区别

### 2. 分布式锁的四个核心属性

```
分布式锁必须满足的四个条件（Redlock论文定义）：

1. 互斥性（Mutual Exclusion）
   任何时刻只有一个客户端能持有锁

2. 无死锁（Deadlock Free）
   即使持有锁的客户端崩溃，锁也能被释放

3. 可重入（Reentrancy）
   同一线程可以多次获取同一把锁

4. 容错性（Fault Tolerance）
   部分Redis节点故障时，锁机制仍能工作
```

**工程启示**：
- SETNX方案满足1和2，不满足3和4
- Redisson方案满足1、2、3，部分满足4（主从切换有窗口期）
- RedLock方案试图满足全部4个，但有争议
- ZooKeeper方案天然满足全部4个

### 3. 时钟问题与分布式锁

```
分布式系统中的时钟假设：

同步系统假设（不现实）：
- 所有节点时钟完全一致
- 消息延迟有上限
- 分布式锁设计简单

异步系统假设（现实）：
- 节点时钟可能漂移
- 消息延迟无上限
- 需要处理时钟跳跃、GC暂停、网络延迟

对Redis分布式锁的影响：
- 锁的过期时间基于本地时钟
- 时钟向前跳跃 → 锁提前过期
- 时钟向后跳跃 → 锁延迟过期（死锁风险）
```

---

## 来龙去脉：分布式锁的发展史

### 第一阶段：数据库分布式锁（2010-2013）

```
早期方案：基于数据库的乐观锁/悲观锁

乐观锁（版本号机制）：
UPDATE inventory 
SET count = count - 1, version = version + 1 
WHERE id = 1 AND version = 5;

悲观锁（SELECT FOR UPDATE）：
BEGIN;
SELECT * FROM inventory WHERE id = 1 FOR UPDATE;
-- 执行业务逻辑
UPDATE inventory SET count = count - 1 WHERE id = 1;
COMMIT;

局限性：
1. 性能差：数据库连接数有限，高并发下成为瓶颈
2. 单点问题：数据库宕机，锁服务不可用
3. 无自动释放：事务超时或应用崩溃可能导致死锁
4. 不可重入：需要额外字段记录线程标识
```

### 第二阶段：Memcached分布式锁（2012-2014）

```
Memcached的add命令（原子性）：

add lock_key 0 30 1  # key, flags, expire(秒), length
1                    # value

# 如果key已存在，add返回失败（NOT_STORED）
# 如果key不存在，add成功存储（STORED）

解锁：
delete lock_key

局限性：
1. 不支持过期时间自动释放（早期版本）
2. 不支持可重入
3. Memcached集群数据不持久化，重启丢失
4. 无高可用机制
```

### 第三阶段：Redis SETNX时代（2014-2016）

```
Redis 2.6.12 引入 SET key value NX EX seconds（原子操作）

# 之前（非原子，存在竞态条件）：
SETNX lock_key value
EXPIRE lock_key 30  # 如果这里客户端崩溃，锁永不过期

# 之后（原子操作）：
SET lock_key value NX EX 30

SET命令参数：
- NX：Only if Not eXists（不存在才设置）
- XX：Only if eXists（存在才设置）
- EX seconds：设置过期时间为秒
- PX milliseconds：设置过期时间为毫秒
- KEEPTTL：保留原有TTL

里程碑：2014年，Redis分布式锁进入原子操作时代，但仍有三个核心问题未解决：
1. 不可重入
2. 无自动续期
3. 主从切换可能丢锁
```

### 第四阶段：Redisson框架时代（2016-2018）

```
Redisson的诞生：让Redis分布式锁达到工业级标准

核心贡献：
1. 可重入锁：基于Hash结构实现线程级重入计数
2. 看门狗机制：自动续期，防止业务执行时间长导致锁过期
3. 红锁（RedLock）：多节点部署提升可靠性
4. 完善的Java API：lock/unlock/tryLock/lockInterruptibly

关键版本演进：
- Redisson 2.x：基础分布式锁实现
- Redisson 3.x：引入看门狗、红锁、读写锁、信号量等
- Redisson 3.15+：支持Redis 6.x ACL、SSL连接
- Redisson 3.20+：支持Redis 7.x Functions
```

### 第五阶段：RedLock争议与替代方案（2016-2020）

```
争议时间线：

2016.02：Redis作者Antirez发布RedLock算法
         声称在多主节点上实现安全的分布式锁

2016.02：Martin Kleppmann发布《How to do distributed locking》
         提出三点质疑：时钟跳跃、GC暂停、网络延迟
         结论：RedLock不能提供强一致性保证

2016.02：Antirez回应《Is Redlock safe?》
         辩护RedLock的设计，但承认有边界条件

2016-2020：社区持续争论
         - 支持者：RedLock在工程实践中足够安全
         - 反对者：任何基于时钟的锁都不可靠

实际影响：
- 大多数公司仍使用Redisson单节点+看门狗
- 强一致性场景转向ZooKeeper/ETCD
- Redis 7.x引入Functions，为分布式锁提供新可能性
```

### 第六阶段：2026年现状

```
当前工业标准：

1. 一般业务场景：
   Redisson单节点/哨兵 + 看门狗（AP方案，性能优先）
   - 锁丢失概率极低（主从切换窗口期<1秒）
   - 性能：QPS 10万+

2. 高可用场景：
   Redisson Cluster + 看门狗
   - 自动故障转移
   - 数据分片降低单节点压力

3. 强一致性场景：
   ZooKeeper Curator / ETCD Client
   - CP方案，牺牲部分性能换取安全
   - 性能：QPS 数千

4. 云原生场景：
   Kubernetes Lease API / AWS DynamoDB Lock
   - 与基础设施集成
   - 自动扩容和故障转移

5. 混合架构：
   分布式锁服务（如美团的DTLock、阿里的TLock）
   - 底层支持多种存储（Redis/ZK/ETCD）
   - 上层统一API，根据场景自动路由
```

---

## 核心原理深度解析

### 1. SETNX 的底层原理与缺陷

#### Redis SET命令的原子性保证

```
Redis单线程模型保证了命令的原子性：

┌─────────────────────────────────────┐
│          Redis 单线程事件循环         │
│                                     │
│  客户端请求 → 读取命令 → 执行命令      │
│                ↓                    │
│            单线程执行                 │
│                ↓                    │
│         返回结果给客户端               │
│                                     │
│  关键：所有命令串行执行，无并发竞争      │
└─────────────────────────────────────┘

SET key value NX EX 30 的原子性：
1. 检查key是否存在（NX条件）
2. 如果不存在，设置key的值
3. 设置过期时间为30秒

这三步在Redis内部是原子操作，不会被其他命令插入。
```

**为什么需要随机value？**

```java
// 错误示范：不加value或固定value
SET lock:order:123 "" NX EX 30
// 客户端A获取锁
// 客户端A执行耗时操作，锁过期
// 客户端B获取锁
// 客户端A完成后，DEL lock:order:123
// 结果：客户端A删除了客户端B的锁！

// 正确做法：使用随机value（UUID+线程ID）
String requestId = UUID.randomUUID().toString() + ":" + Thread.currentThread().getId();
SET lock:order:123 requestId NX EX 30

// 释放锁时，先检查value是否匹配
if (redis.call('get', KEYS[1]) == ARGV[1]) then
    return redis.call('del', KEYS[1])
else
    return 0
end
```

#### Lua脚本的原子性

```
Redis Lua脚本的原子性保证：

┌─────────────────────────────────────┐
│  1. 收到 EVAL 命令                   │
│  2. 编译 Lua 脚本（或从缓存取）       │
│  3. 在单线程中执行整个脚本           │
│  4. 期间不处理其他客户端命令          │
│  5. 返回脚本执行结果                 │
└─────────────────────────────────────┘

关键：脚本执行期间，Redis不会响应其他命令
      因此脚本内的多个Redis操作是原子的

注意：脚本执行时间过长会阻塞Redis
      建议脚本执行时间 < 10ms
```

#### SETNX方案的完整实现与问题

```java
/**
 * SETNX分布式锁的基础实现
 * 问题：不可重入、无看门狗、主从延迟
 */
@Component
public class SimpleRedisLock {
    
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    private static final String LOCK_PREFIX = "lock:";
    private static final long DEFAULT_EXPIRE = 30;
    
    /**
     * 尝试获取锁
     */
    public boolean tryLock(String lockKey, long expireSeconds) {
        String requestId = generateRequestId();
        Boolean success = redisTemplate.opsForValue()
            .setIfAbsent(LOCK_PREFIX + lockKey, requestId, expireSeconds, TimeUnit.SECONDS);
        return Boolean.TRUE.equals(success);
    }
    
    /**
     * 释放锁（使用Lua保证原子性）
     */
    public void unlock(String lockKey, String requestId) {
        String script = 
            "if redis.call('get', KEYS[1]) == ARGV[1] then " +
            "    return redis.call('del', KEYS[1]) " +
            "else " +
            "    return 0 " +
            "end";
        
        redisTemplate.execute(
            new DefaultRedisScript<>(script, Long.class),
            Collections.singletonList(LOCK_PREFIX + lockKey),
            requestId
        );
    }
    
    private String generateRequestId() {
        return UUID.randomUUID().toString() + ":" + Thread.currentThread().getId();
    }
}
```

**SETNX方案的三大缺陷**：

```
缺陷1：不可重入
┌─────────────────────────────────────┐
│  线程A获取锁 lock:order:123         │
│  ↓                                  │
│  线程A调用另一个需要同锁的方法        │
│  ↓                                  │
│  线程A再次尝试获取 lock:order:123   │
│  ↓                                  │
│  失败！锁已被自己持有，但无法重入     │
│                                     │
│  后果：死锁（如果代码有重入需求）      │
└─────────────────────────────────────┘

缺陷2：无看门狗（Watch Dog）
┌─────────────────────────────────────┐
│  线程A获取锁，过期时间30秒            │
│  ↓                                  │
│  业务逻辑执行耗时35秒（GC暂停/慢查询） │
│  ↓                                  │
│  锁在30秒时自动过期                  │
│  ↓                                  │
│  线程B获取到锁                       │
│  ↓                                  │
│  线程A业务完成，执行DEL（删除了B的锁） │
│  ↓                                  │
│  线程C也能获取锁                     │
│                                     │
│  后果：多个线程同时进入临界区          │
└─────────────────────────────────────┘

缺陷3：主从延迟导致锁丢失
┌─────────────────────────────────────┐
│  主节点写入锁                        │
│  ↓                                  │
│  主节点宕机（锁尚未同步到从节点）      │
│  ↓                                  │
│  从节点提升为主节点                   │
│  ↓                                  │
│  从节点上没有锁的信息                 │
│  ↓                                  │
│  客户端B可以在新主节点上获取锁        │
│  ↓                                  │
│  客户端A和B同时持有锁                │
│                                     │
│  后果：脑裂，多个客户端同时持有锁      │
└─────────────────────────────────────┘
```

### 2. Redisson 可重入锁的实现

#### Hash数据结构的设计哲学

```
Redisson使用Hash结构存储锁信息：

┌─────────────────────────────────────┐
│  Key: lock:order:123                │
│  Type: Hash                         │
│                                     │
│  ┌─────────────────────────────┐   │
│  │ Field          │ Value      │   │
│  ├─────────────────────────────┤   │
│  │ 58c8a...:1     │ 2          │   │  <-- threadId: 重入次数
│  │ 58c8a...:1     │ 表示：      │   │
│  │   UUID前8位    │ 线程1重入2次 │   │
│  │   :线程ID      │             │   │
│  └─────────────────────────────┘   │
│                                     │
│  TTL: 30000ms（看门狗续期）          │
└─────────────────────────────────────┘

为什么用Hash而不是String？
1. 需要存储重入计数（一个field记录一个线程）
2. 需要支持多线程同时等待（ RedissonLockEntry ）
3. Hash的HINCRBY是原子操作，适合重入计数
```

#### 加锁Lua脚本深度解析

```java
// RedissonLock.tryLockInnerAsync() 的核心Lua脚本

<T> RFuture<T> tryLockInnerAsync(long waitTime, long leaseTime, TimeUnit unit,
                                 long threadId, RedisStrictCommand<T> command) {
    return evalWriteAsync(getRawName(), LongCodec.INSTANCE, command,
        // 参数说明：
        // KEYS[1] = lock:order:123（锁的key）
        // ARGV[1] = 30000（leaseTime，毫秒）
        // ARGV[2] = 58c8a...:1（threadId，格式：UUID前8位:线程ID）
        
        // 第一段：锁不存在，直接获取
        "if (redis.call('exists', KEYS[1]) == 0) then " +
        "    redis.call('hincrby', KEYS[1], ARGV[2], 1); " +  // 重入次数设为1
        "    redis.call('pexpire', KEYS[1], ARGV[1]); " +      // 设置过期时间
        "    return nil; " +
        "end; " +
        
        // 第二段：锁存在且是当前线程的，重入
        "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
        "    redis.call('hincrby', KEYS[1], ARGV[2], 1); " +  // 重入次数+1
        "    redis.call('pexpire', KEYS[1], ARGV[1]); " +      // 刷新过期时间
        "    return nil; " +
        "end; " +
        
        // 第三段：锁被其他线程占用，返回剩余时间
        "return redis.call('pttl', KEYS[1]);",
        Collections.singletonList(getRawName()),
        unit.toMillis(leaseTime), getLockName(threadId));
}
```

**脚本执行的三种情况**：

```
情况1：锁不存在（首次获取）
┌─────────────────────────────────────┐
│ exists lock:order:123 → 0           │
│ ↓                                   │
│ hincrby lock:order:123 58c8a...:1 1 │
│ ↓                                   │
│ pexpire lock:order:123 30000        │
│ ↓                                   │
│ return nil（获取成功）               │
└─────────────────────────────────────┘

情况2：锁存在且是当前线程（重入）
┌─────────────────────────────────────┐
│ exists lock:order:123 → 1           │
│ ↓                                   │
│ hexists lock:order:123 58c8a...:1 → 1│
│ ↓                                   │
│ hincrby lock:order:123 58c8a...:1 1 │
│ 结果：重入次数从1变为2               │
│ ↓                                   │
│ pexpire lock:order:123 30000        │
│ ↓                                   │
│ return nil（重入成功）               │
└─────────────────────────────────────┘

情况3：锁被其他线程占用（获取失败）
┌─────────────────────────────────────┐
│ exists lock:order:123 → 1           │
│ ↓                                   │
│ hexists lock:order:123 58c8a...:1 → 0│
│ 说明：锁存在，但不是当前线程的        │
│ ↓                                   │
│ return pttl(lock:order:123)         │
│ 例如：return 15000（还有15秒过期）    │
│                                     │
│ 客户端行为：根据waitTime决定是否等待   │
└─────────────────────────────────────┘
```

#### 解锁Lua脚本深度解析

```java
// RedissonLock.unlockInnerAsync() 的核心Lua脚本

protected RFuture<Boolean> unlockInnerAsync(long threadId) {
    return evalWriteAsync(getRawName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN,
        // KEYS[1] = lock:order:123
        // KEYS[2] = redisson_lock__channel:{lock:order:123}（发布订阅频道）
        // ARGV[1] = 0（解锁消息）
        // ARGV[2] = 30000（看门狗超时时间）
        // ARGV[3] = 58c8a...:1（threadId）
        
        // 第一段：检查锁是否属于当前线程
        "if (redis.call('hexists', KEYS[1], ARGV[3]) == 0) then " +
        "    return nil;" +  // 不属于当前线程，返回nil
        "end; " +
        
        // 第二段：重入次数-1
        "local counter = redis.call('hincrby', KEYS[1], ARGV[3], -1); " +
        
        // 第三段：判断是否需要完全释放
        "if (counter > 0) then " +
        "    redis.call('pexpire', KEYS[1], ARGV[2]); " +  // 仍持有锁，续期
        "    return 0; " +  // 返回0表示未完全释放
        "else " +
        "    redis.call('del', KEYS[1]); " +              // 完全释放，删除锁
        "    redis.call('publish', KEYS[2], ARGV[1]); " + // 通知等待的线程
        "    return 1; " +  // 返回1表示完全释放
        "end; " +
        "return nil;",
        Arrays.asList(getRawName(), getChannelName()),
        LockPubSub.UNLOCK_MESSAGE, internalLockLeaseTime, getLockName(threadId));
}
```

**解锁的三种情况**：

```
情况1：当前线程未持有锁
┌─────────────────────────────────────┐
│ hexists lock:order:123 58c8a...:1 → 0│
│ 说明：锁不存在或不属于当前线程        │
│ ↓                                   │
│ return nil                           │
│ 客户端：抛出IllegalMonitorStateException │
└─────────────────────────────────────┘

情况2：重入次数减1后仍大于0（部分释放）
┌─────────────────────────────────────┐
│ hincrby lock:order:123 58c8a...:1 -1│
│ 原值：2，减1后：counter = 1          │
│ ↓                                   │
│ counter > 0 → true                  │
│ ↓                                   │
│ pexpire lock:order:123 30000        │
│ ↓                                   │
│ return 0（未完全释放）               │
└─────────────────────────────────────┘

情况3：重入次数减1后等于0（完全释放）
┌─────────────────────────────────────┐
│ hincrby lock:order:123 58c8a...:1 -1│
│ 原值：1，减1后：counter = 0          │
│ ↓                                   │
│ counter > 0 → false                 │
│ ↓                                   │
│ del lock:order:123                  │
│ ↓                                   │
│ publish redisson_lock__channel:{...} 0│
│ 通知等待的线程：锁已释放              │
│ ↓                                   │
│ return 1（完全释放）                 │
└─────────────────────────────────────┘
```

### 3. 看门狗机制的设计哲学

#### 为什么需要看门狗？

```
业务执行时间不确定的现实：

┌─────────────────────────────────────┐
│  锁获取（TTL=30秒）                  │
│  ↓                                  │
│  业务逻辑开始                         │
│  ├─ 数据库查询（200ms）              │
│  ├─ RPC调用（500ms）                 │
│  ├─ 复杂计算（1秒）                  │
│  ├─ 外部HTTP请求（3秒，超时）        │
│  ├─ 重试...（再3秒）                 │
│  ├─ GC暂停（5秒）                    │
│  ├─ 数据库批量插入（10秒）           │
│  ↓                                  │
│  业务逻辑完成（共22.7秒）            │
│                                     │
│  问题：如果TTL=20秒，锁已过期！       │
│  如果TTL=60秒，崩溃后死锁60秒        │
│                                     │
│  看门狗的解决方案：动态续期            │
│  - 业务执行时自动续期                 │
│  - 业务完成停止续期                   │
│  - 崩溃后不再续期，锁自然过期         │
└─────────────────────────────────────┘
```

#### 看门狗的实现机制

```java
/**
 * Redisson看门狗核心逻辑
 */
public class WatchDog {
    
    // 默认配置
    private static final long DEFAULT_WATCH_DOG_TIMEOUT = 30000L;  // 30秒
    private static final long DEFAULT_RENEWAL_INTERVAL = 10000L;   // 10秒（1/3的timeout）
    
    private final Timer timer = new HashedWheelTimer();
    private final ConcurrentHashMap<String, Timeout> renewalTasks = new ConcurrentHashMap<>();
    
    /**
     * 启动看门狗
     */
    public void startRenewal(String lockKey, String threadId, long leaseTime) {
        // 计算续期间隔：通常为leaseTime / 3
        long renewalInterval = leaseTime / 3;
        
        Timeout task = timer.newTimeout(timeout -> {
            // 执行续期Lua脚本
            renewExpiration(lockKey, threadId, leaseTime);
            
            // 如果锁仍被持有，递归续期
            if (isHeldByCurrentThread(lockKey, threadId)) {
                startRenewal(lockKey, threadId, leaseTime);
            }
        }, renewalInterval, TimeUnit.MILLISECONDS);
        
        renewalTasks.put(lockKey + ":" + threadId, task);
    }
    
    /**
     * 续期Lua脚本
     */
    private void renewExpiration(String lockKey, String threadId, long leaseTime) {
        String script = 
            "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
            "    redis.call('pexpire', KEYS[1], ARGV[1]); " +
            "    return 1; " +
            "end; " +
            "return 0;";
        
        Long result = redisTemplate.execute(
            new DefaultRedisScript<>(script, Long.class),
            Collections.singletonList(lockKey),
            leaseTime, threadId
        );
        
        if (result != null && result == 1) {
            log.debug("Renewed lock: {}, threadId: {}", lockKey, threadId);
        }
    }
    
    /**
     * 停止看门狗（业务完成时调用）
     */
    public void stopRenewal(String lockKey, String threadId) {
        Timeout task = renewalTasks.remove(lockKey + ":" + threadId);
        if (task != null) {
            task.cancel();
        }
    }
}
```

**看门狗时间线**：

```
T=0s:    线程A获取锁，TTL=30秒，启动看门狗
         看门狗第一次检查时间：T=10秒

T=10s:   看门狗检查：线程A仍持有锁
         执行pexpire，TTL重置为30秒（过期时间=T+40秒）
         下次检查时间：T=20秒

T=20s:   看门狗检查：线程A仍持有锁
         执行pexpire，TTL重置为30秒（过期时间=T+50秒）
         下次检查时间：T=30秒

T=25s:   线程A完成业务，解锁
         看门狗被取消

T=30s:   看门狗不会执行（已取消）
         锁在T=40秒时自然过期（如果未释放）

崩溃场景：
T=0s:    线程A获取锁，启动看门狗
T=15s:   应用崩溃（OOM/Kill -9）
T=20s:   看门狗未执行（应用已死）
T=30s:   锁自然过期，其他线程可以获取
         无死锁！
```

#### 看门狗的配置与调优

```yaml
# Redisson看门狗配置（application.yml）
spring:
  redis:
    redisson:
      config: |
        {
          "singleServerConfig": {
            "address": "redis://127.0.0.1:6379",
            "password": null,
            "database": 0
          },
          "locks": {
            # 看门狗超时时间（默认30秒）
            "watchDogTimeout": 30000,
            # 是否自动启用看门狗
            "autoRenewal": true
          }
        }
```

```java
// Java代码中的看门狗控制

// 方式1：不指定leaseTime，自动启用看门狗（默认30秒）
RLock lock = redissonClient.getLock("order:123");
lock.lock();  // 看门狗自动续期

// 方式2：指定leaseTime，不启用看门狗
lock.lock(10, TimeUnit.SECONDS);  // 10秒后自动释放，无看门狗

// 方式3：tryLock，指定等待时间和leaseTime
boolean locked = lock.tryLock(5, 30, TimeUnit.SECONDS);
// 等待最多5秒，获取后30秒自动释放
// 如果leaseTime=-1或不指定，启用看门狗

// 方式4：全局配置看门狗超时
Config config = new Config();
config.setLockWatchdogTimeout(60000L);  // 60秒
RedissonClient client = Redisson.create(config);
```

**看门狗的调优原则**：

```
1. 看门狗超时时间 > 业务最大执行时间
   建议：业务最大执行时间 × 2
   
2. 续期间隔 = 看门狗超时时间 / 3
   原理：确保在TTL耗尽前，至少还有2次续期机会
   
3. 避免过长的超时时间
   - 崩溃后死锁时间长
   - 建议最大值：5分钟
   
4. 监控指标：
   - 锁平均持有时间
   - 看门狗续期次数
   - 锁竞争等待时间
```

### 4. RedLock 算法的数学推导

#### 算法原理

```
RedLock解决的问题：单节点Redis故障导致锁丢失

核心思想：在N个独立Redis节点上同时加锁
         如果成功加锁的节点数 >= N/2 + 1，则获取锁成功

为什么需要N/2 + 1？
- 这是多数派（Quorum）原则
- 确保任何时刻只有一个客户端能获得多数派
- 即使部分节点故障，仍能保证互斥性
```

#### 算法步骤详解

```
RedLock算法（N=5个独立Redis节点）：

步骤1：获取当前时间戳 T1（毫秒）

步骤2：向5个Redis节点请求加锁
       ┌─────────────────────────────────────┐
       │  节点1: SET lock_key random_value   │
       │         NX PX 30000                  │
       │         → OK（成功）                 │
       ├─────────────────────────────────────┤
       │  节点2: SET lock_key random_value   │
       │         NX PX 30000                  │
       │         → OK（成功）                 │
       ├─────────────────────────────────────┤
       │  节点3: SET lock_key random_value   │
       │         NX PX 30000                  │
       │         → OK（成功）                 │
       ├─────────────────────────────────────┤
       │  节点4: SET lock_key random_value   │
       │         NX PX 30000                  │
       │         → (null)（失败，已被占用）   │
       ├─────────────────────────────────────┤
       │  节点5: SET lock_key random_value   │
       │         NX PX 30000                  │
       │         → 超时（网络延迟）           │
       └─────────────────────────────────────┘

步骤3：计算总耗时 T2 = now - T1
       假设 T2 = 150ms

步骤4：判断锁是否获取成功
       成功节点数 = 3（节点1、2、3）
       需要节点数 = 5/2 + 1 = 3
       3 >= 3 ✓
       
       锁的有效时间 = lockValidityTime - T2
                    = 30000ms - 150ms
                    = 29850ms
       
       获取锁成功！

步骤5：如果获取失败，向所有节点发送解锁请求
       （包括未成功加锁的节点，防止残留）

步骤6：业务执行完成后，向所有节点发送解锁请求
```

#### Redisson的RedLock实现

```java
/**
 * Redisson RedLock使用示例
 */
@Configuration
public class RedLockConfig {
    
    @Bean
    public RedissonRedLock redLock() {
        // 创建3个独立的Redisson客户端（连接不同Redis节点）
        Config config1 = new Config();
        config1.useSingleServer().setAddress("redis://192.168.0.1:6379");
        RedissonClient client1 = Redisson.create(config1);
        
        Config config2 = new Config();
        config2.useSingleServer().setAddress("redis://192.168.0.2:6379");
        RedissonClient client2 = Redisson.create(config2);
        
        Config config3 = new Config();
        config3.useSingleServer().setAddress("redis://192.168.0.3:6379");
        RedissonClient client3 = Redisson.create(config3);
        
        // 创建3个锁
        RLock lock1 = client1.getLock("businessLock");
        RLock lock2 = client2.getLock("businessLock");
        RLock lock3 = client3.getLock("businessLock");
        
        // 组合成RedLock
        return new RedissonRedLock(lock1, lock2, lock3);
    }
}

@Service
public class OrderService {
    
    @Autowired
    private RedissonRedLock redLock;
    
    public void createOrder(Long orderId) {
        try {
            // 尝试获取RedLock
            // 参数：waitTime=100ms, leaseTime=10秒
            boolean locked = redLock.tryLock(100, 10, TimeUnit.SECONDS);
            
            if (locked) {
                try {
                    // 执行业务逻辑
                    processOrder(orderId);
                } finally {
                    redLock.unlock();
                }
            } else {
                throw new RuntimeException("获取分布式锁失败");
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException("获取锁被中断", e);
        }
    }
}
```

#### RedLock的争议与数学分析

```
Martin Kleppmann的三点质疑：

质疑1：时钟跳跃（Clock Skew）
┌─────────────────────────────────────┐
│  假设：Redis节点A的系统时间被NTP调快5秒 │
│  ↓                                  │
│  客户端在T=0s获取锁，TTL=30秒        │
│  预期过期时间：T=30s                 │
│  ↓                                  │
│  节点A时钟跳跃：实际时间T=0s，节点认为T=5s│
│  ↓                                  │
│  锁在节点A上的实际过期时间：25秒（提前5秒）│
│  ↓                                  │
│  客户端B可以在T=25s获取节点A的锁      │
│  但客户端A仍认为持有锁                │
│  ↓                                  │
│  两个客户端同时持有锁！               │
│                                     │
│  数学分析：                           │
│  设时钟最大漂移为δ，锁TTL为T         │
│  有效锁时间 = T - δ                  │
│  如果δ > T，锁完全失效               │
│  即使δ < T，也缩短了锁的安全窗口      │
└─────────────────────────────────────┘

质疑2：客户端GC暂停（GC Pause）
┌─────────────────────────────────────┐
│  T=0s：客户端获取RedLock成功          │
│  T=5s：JVM触发Full GC，暂停15秒       │
│  T=20s：GC结束，客户端继续执行业务     │
│  ↓                                  │
│  问题：在T=5s到T=20s期间              │
│  - Redis中的锁已过期（TTL=10秒）      │
│  - 客户端B获取到锁                    │
│  - T=20s后，客户端A继续修改共享资源   │
│  ↓                                  │
│  两个客户端同时操作！                 │
│                                     │
│  数学分析：                           │
│  设GC暂停时间为G，锁TTL为T           │
│  安全条件：G < T                     │
│  如果G >= T，锁在GC期间丢失          │
│  且客户端不知道锁已丢失               │
└─────────────────────────────────────┘

质疑3：网络延迟（Network Delay）
┌─────────────────────────────────────┐
│  T=0s：客户端发送加锁请求             │
│  T=0.5s：请求到达Redis（网络延迟500ms）│
│  ↓                                  │
│  Redis执行SET命令，设置TTL=10秒       │
│  锁的过期时间：T=10.5s（绝对时间）     │
│  ↓                                  │
│  客户端在T=10s认为锁已过期            │
│  但Redis直到T=10.5s才真正释放        │
│  ↓                                  │
│  在T=10s到T=10.5s之间                │
│  客户端可能重复获取锁                 │
│                                     │
│  数学分析：                           │
│  设网络往返延迟为R，锁TTL为T         │
│  有效锁时间 = T - R                  │
│  如果R接近T，锁的实际可用时间很短     │
└─────────────────────────────────────┘
```

**RedLock的安全边界**：

```
RedLock在什么条件下是安全的？

条件1：时钟漂移可控
  δ << TTL（建议δ < TTL/10）
  使用monotonic clock（单调时钟）而非wall clock

条件2：GC暂停可控
  G < TTL/2
  使用低延迟GC（ZGC/Shenandoah）
  设置合理的JVM堆大小

条件3：网络延迟可控
  R < TTL/10
  同机房部署，延迟<1ms
  避免跨机房、跨地域部署

如果以上条件不满足：
- 使用fencing token（防护令牌）
- 或改用ZooKeeper/ETCD（基于共识算法，不依赖时钟）
```

#### Fencing Token 防护方案

```
Fencing Token解决RedLock的边界条件问题：

核心思想：获取锁时获得递增的token，操作资源时携带token
         资源服务只接受最大token的请求

┌─────────────────────────────────────┐
│  Redis返回递增token                  │
│  ↓                                  │
│  客户端A获取锁，token=100            │
│  客户端A开始执行业务（GC暂停）        │
│  ↓                                  │
│  锁过期，客户端B获取锁，token=101    │
│  ↓                                  │
│  客户端A GC恢复，携带token=100操作资源 │
│  资源服务检查：100 < 当前最大token(101)│
│  拒绝客户端A的请求！                  │
│  ↓                                  │
│  客户端A知道锁已丢失，回滚业务        │
└─────────────────────────────────────┘
```

```java
/**
 * Fencing Token实现示例
 */
@Service
public class FencingTokenService {
    
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    /**
     * 获取带Fencing Token的锁
     */
    public LockResult acquireLock(String lockKey, long expireSeconds) {
        String requestId = UUID.randomUUID().toString();
        
        // 原子性获取锁和token
        String script = 
            "if redis.call('setnx', KEYS[1], ARGV[1]) == 1 then " +
            "    redis.call('expire', KEYS[1], ARGV[2]) " +
            "    local token = redis.call('incr', 'fencing:token:seq') " +
            "    return token " +
            "else " +
            "    return 0 " +
            "end";
        
        Long token = redisTemplate.execute(
            new DefaultRedisScript<>(script, Long.class),
            Collections.singletonList(lockKey),
            requestId, String.valueOf(expireSeconds)
        );
        
        if (token != null && token > 0) {
            return new LockResult(true, requestId, token);
        }
        return new LockResult(false, null, null);
    }
    
    /**
     * 使用Fencing Token操作数据库
     */
    public void updateResource(Long resourceId, LockResult lockResult, String newValue) {
        // 数据库表中需要记录当前最大token
        int updated = jdbcTemplate.update(
            "UPDATE resource SET value = ?, fencing_token = ? " +
            "WHERE id = ? AND fencing_token < ?",
            newValue, lockResult.getToken(), resourceId, lockResult.getToken()
        );
        
        if (updated == 0) {
            throw new LockLostException("锁已丢失，操作被拒绝");
        }
    }
}
```

---

## 模型差异：不同分布式锁方案对比

### 1. Redis vs ZooKeeper vs ETCD 深度对比

```
┌─────────────────┬─────────────────┬─────────────────┬─────────────────┐
│     特性        │   Redis         │  ZooKeeper      │     ETCD        │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ 一致性模型      │ 最终一致（AP）   │ 强一致（CP）     │ 强一致（CP）     │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ 共识算法        │ 无（主从复制）   │ ZAB协议         │ Raft协议        │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ 性能（QPS）     │ 10万+           │ 数千            │ 数万            │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ 延迟            │ <1ms            │ <10ms           │ <5ms            │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ 可靠性          │ 主从切换可能丢锁 │ 不丢锁          │ 不丢锁          │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ 可重入          │ 需自己实现      │ Curator支持     │ 需自己实现      │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ 自动续期        │ Redisson支持    │ 临时节点自动释放 │ 租约（Lease）   │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ 会话保持        │ 无              │ 有（心跳检测）   │ 有（租约续期）   │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ 复杂度          │ 低              │ 中              │ 中              │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ 运维成本        │ 低              │ 高              │ 中              │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ 适用场景        │ 一般业务锁      │ 强一致性要求     │ 云原生/K8s      │
└─────────────────┴─────────────────┴─────────────────┴─────────────────┘
```

### 2. 各方案的最佳实践场景

```
场景1：电商库存扣减（高并发，最终一致可接受）
┌─────────────────────────────────────┐
│ 推荐：Redisson + Redis Cluster      │
│                                     │
│ 原因：                               │
│ - 性能要求极高（秒杀场景QPS 10万+）   │
│ - 库存扣减允许短暂不一致（可回滚）    │
│ - 需要自动续期（防止超卖）            │
│                                     │
│ 配置：                               │
│ - 看门狗超时：30秒                   │
│ - 锁粒度：sku级别（lock:sku:10086）  │
│ - 降级策略：Redis故障时降级为数据库锁 │
└─────────────────────────────────────┘

场景2：金融交易（强一致性，不可丢锁）
┌─────────────────────────────────────┐
│ 推荐：ZooKeeper Curator             │
│                                     │
│ 原因：                               │
│ - 资金安全，不能出现双花              │
│ - 需要强一致性保证                    │
│ - 并发量不高（金融交易QPS < 1000）    │
│                                     │
│ 配置：                               │
│ - 会话超时：30秒                     │
│ - 连接超时：10秒                     │
│ - 重试策略：指数退避                  │
└─────────────────────────────────────┘

场景3：分布式任务调度（K8s环境）
┌─────────────────────────────────────┐
│ 推荐：ETCD + Kubernetes Lease       │
│                                     │
│ 原因：                               │
│ - 与K8s生态集成                      │
│ - 支持租约自动续期                    │
│ - 原生支持Leader选举                  │
│                                     │
│ 配置：                               │
│ - 租约TTL：15秒                      │
│ - 续期间隔：5秒                      │
│ - 故障转移时间：< 15秒               │
└─────────────────────────────────────┘

场景4：分布式配置管理
┌─────────────────────────────────────┐
│ 推荐：ETCD                          │
│                                     │
│ 原因：                               │
│ - Watch机制实时推送变更              │
│ - 强一致保证所有节点配置一致          │
│ - 支持事务（Txn）                     │
└─────────────────────────────────────┘
```

---

## 工业级实践案例

### 案例1：电商库存扣减系统

**场景**：双十一秒杀，库存扣减高并发

**核心挑战**：
- 瞬间流量：10万QPS
- 超卖风险：库存不能为负
- 性能要求：RT < 50ms

**架构设计**：

```
┌─────────────────────────────────────────────────────────┐
│                    秒杀活动架构                          │
│                                                         │
│   用户请求 → Nginx → Gateway → 库存服务集群              │
│                              (10台机器)                  │
│                                ↓                        │
│                    ┌─────────────────────┐              │
│                    │   Redis Cluster     │              │
│                    │  (6主6从，分片)      │              │
│                    │                     │              │
│                    │  MasterA(0-5460)    │              │
│                    │  MasterB(5461-10922)│              │
│                    │  MasterC(10923-16383)│             │
│                    └─────────────────────┘              │
│                                ↓                        │
│                    ┌─────────────────────┐              │
│                    │   MySQL (最终持久化)  │              │
│                    └─────────────────────┘              │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**代码实现**：

```java
/**
 * 秒杀库存服务
 */
@Service
public class SeckillStockService {
    
    @Autowired
    private RedissonClient redissonClient;
    
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    @Autowired
    private StockRepository stockRepository;
    
    private static final String STOCK_KEY_PREFIX = "seckill:stock:";
    private static final String LOCK_KEY_PREFIX = "lock:seckill:";
    
    /**
     * 初始化库存（活动开始前预热）
     */
    public void initStock(Long activityId, Long skuId, Integer stock) {
        String stockKey = STOCK_KEY_PREFIX + activityId + ":" + skuId;
        redisTemplate.opsForValue().set(stockKey, String.valueOf(stock));
    }
    
    /**
     * 扣减库存（核心逻辑）
     */
    public boolean deductStock(Long activityId, Long skuId, Integer quantity) {
        String lockKey = LOCK_KEY_PREFIX + activityId + ":" + skuId;
        String stockKey = STOCK_KEY_PREFIX + activityId + ":" + skuId;
        
        RLock lock = redissonClient.getLock(lockKey);
        
        try {
            // 尝试获取锁，等待3秒，30秒后自动释放
            boolean locked = lock.tryLock(3, 30, TimeUnit.SECONDS);
            if (!locked) {
                log.warn("获取库存锁失败，activityId={}, skuId={}", activityId, skuId);
                return false;
            }
            
            try {
                // 1. 检查库存（Lua原子操作）
                String script = 
                    "local stock = tonumber(redis.call('get', KEYS[1])); " +
                    "if stock == nil then return -1; end; " +  // 库存不存在
                    "if stock < tonumber(ARGV[1]) then return -2; end; " +  // 库存不足
                    "redis.call('decrby', KEYS[1], ARGV[1]); " +
                    "return redis.call('get', KEYS[1]);";
                
                Long result = redisTemplate.execute(
                    new DefaultRedisScript<>(script, Long.class),
                    Collections.singletonList(stockKey),
                    String.valueOf(quantity)
                );
                
                if (result == null || result == -1) {
                    log.error("库存未初始化，stockKey={}", stockKey);
                    return false;
                }
                
                if (result == -2) {
                    log.warn("库存不足，stockKey={}", stockKey);
                    return false;
                }
                
                // 2. 发送异步消息，同步到数据库（最终一致性）
                StockDeductMessage message = new StockDeductMessage(
                    activityId, skuId, quantity, result
                );
                kafkaTemplate.send("stock-deduct-topic", message);
                
                log.info("扣减库存成功，activityId={}, skuId={}, remain={}", 
                    activityId, skuId, result);
                return true;
                
            } finally {
                if (lock.isHeldByCurrentThread()) {
                    lock.unlock();
                }
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            log.error("扣减库存被中断", e);
            return false;
        }
    }
    
    /**
     * 异步同步库存到数据库（消费者）
     */
    @KafkaListener(topics = "stock-deduct-topic", groupId = "stock-sync")
    public void syncStockToDatabase(StockDeductMessage message) {
        stockRepository.decrement(
            message.getActivityId(), 
            message.getSkuId(), 
            message.getQuantity()
        );
    }
    
    /**
     * 库存回滚（订单取消时）
     */
    public boolean rollbackStock(Long activityId, Long skuId, Integer quantity) {
        String lockKey = LOCK_KEY_PREFIX + activityId + ":" + skuId;
        String stockKey = STOCK_KEY_PREFIX + activityId + ":" + skuId;
        
        RLock lock = redissonClient.getLock(lockKey);
        
        try {
            boolean locked = lock.tryLock(3, 30, TimeUnit.SECONDS);
            if (!locked) return false;
            
            try {
                // 原子性增加库存
                Long remain = redisTemplate.opsForValue().increment(stockKey, quantity);
                
                // 发送回滚消息
                StockRollbackMessage message = new StockRollbackMessage(
                    activityId, skuId, quantity, remain
                );
                kafkaTemplate.send("stock-rollback-topic", message);
                
                return true;
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
}
```

**优化策略**：

```
1. 锁粒度优化
   错误：lock:seckill:activity  （活动级别，并发度极低）
   正确：lock:seckill:activity:skuId（SKU级别，并发度高）

2. 减少锁持有时间
   - 库存检查使用Lua脚本（原子操作）
   - 数据库操作异步化（Kafka消息）
   - 锁内只执行Redis操作

3. 限流保护
   - Gateway层限流（Sentinel/RateLimiter）
   - 库存预扣减失败后快速失败
   - 避免无效请求进入锁竞争

4. 缓存预热
   - 活动开始前5分钟预热库存到Redis
   - 避免缓存击穿（使用互斥锁或逻辑过期）
```

### 案例2：分布式任务调度

**场景**：定时任务在多台机器上执行，需要保证同一时刻只有一台执行

**架构设计**：

```
┌─────────────────────────────────────────────────────────┐
│                  分布式任务调度架构                       │
│                                                         │
│   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐  │
│   │  Scheduler1 │   │  Scheduler2 │   │  Scheduler3 │  │
│   │  (主节点)    │   │  (备用)     │   │  (备用)     │  │
│   │             │   │             │   │             │  │
│   │  ┌───────┐  │   │  ┌───────┐  │   │  ┌───────┐  │  │
│   │  │ Task1 │  │   │  │ Task1 │  │   │  │ Task1 │  │  │
│   │  │ Task2 │  │   │  │ Task2 │  │   │  │ Task2 │  │  │
│   │  └───────┘  │   │  └───────┘  │   │  └───────┘  │  │
│   └──────┬──────┘   └──────┬──────┘   └──────┬──────┘  │
│          │                 │                 │          │
│          └─────────────────┼─────────────────┘          │
│                            ↓                            │
│                   ┌─────────────────┐                   │
│                   │   Redisson Lock   │                   │
│                   │  lock:task:Task1  │                   │
│                   │  lock:task:Task2  │                   │
│                   └─────────────────┘                   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**代码实现**：

```java
/**
 * 分布式定时任务（基于Redisson）
 */
@Component
public class DistributedJob {
    
    @Autowired
    private RedissonClient redissonClient;
    
    /**
     * 每5分钟执行一次数据清理任务
     */
    @Scheduled(cron = "0 0/5 * * * ?")
    public void dataCleanupJob() {
        String lockKey = "lock:job:dataCleanup";
        RLock lock = redissonClient.getLock(lockKey);
        
        try {
            // 尝试获取锁，不等待（立即返回）
            // 使用看门狗自动续期
            boolean locked = lock.tryLock(0, -1, TimeUnit.SECONDS);
            
            if (!locked) {
                log.info("数据清理任务已在其他节点执行，跳过");
                return;
            }
            
            try {
                log.info("开始执行数据清理任务");
                
                // 1. 清理过期日志（7天前）
                cleanupExpiredLogs(7);
                
                // 2. 清理临时文件
                cleanupTempFiles();
                
                // 3. 归档历史订单（3个月前）
                archiveHistoricalOrders(90);
                
                log.info("数据清理任务执行完成");
            } finally {
                if (lock.isHeldByCurrentThread()) {
                    lock.unlock();
                }
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            log.error("任务被中断", e);
        }
    }
    
    /**
     * 每小时执行一次报表生成任务
     */
    @Scheduled(cron = "0 0 * * * ?")
    public void reportGenerationJob() {
        String lockKey = "lock:job:hourlyReport";
        RLock lock = redissonClient.getLock(lockKey);
        
        try {
            // 等待最多10秒，获取后使用看门狗续期
            boolean locked = lock.tryLock(10, -1, TimeUnit.SECONDS);
            
            if (!locked) {
                log.warn("报表生成任务获取锁失败");
                return;
            }
            
            try {
                log.info("开始生成小时报表");
                
                // 生成报表（可能耗时较长，看门狗会自动续期）
                Report report = generateHourlyReport();
                
                // 保存报表
                saveReport(report);
                
                log.info("小时报表生成完成");
            } finally {
                if (lock.isHeldByCurrentThread()) {
                    lock.unlock();
                }
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            log.error("报表任务被中断", e);
        }
    }
    
    private void cleanupExpiredLogs(int days) {
        // 实现...
    }
    
    private void cleanupTempFiles() {
        // 实现...
    }
    
    private void archiveHistoricalOrders(int days) {
        // 实现...
    }
    
    private Report generateHourlyReport() {
        // 实现...
        return new Report();
    }
    
    private void saveReport(Report report) {
        // 实现...
    }
}
```

**任务调度最佳实践**：

```
1. 锁命名规范
   lock:job:{jobName}:{param}
   例如：
   - lock:job:dataCleanup
   - lock:job:dailyReport:2024-01-01
   - lock:job:syncUser:shard0

2. 看门狗配置
   - 定时任务通常执行时间较长（分钟级）
   - 建议看门狗超时：5分钟
   - 续期间隔：1分40秒（300秒/3）

3. 失败重试策略
   - 任务失败不立即重试（避免雪崩）
   - 使用指数退避：1分钟、2分钟、4分钟...
   - 最大重试次数：3次
   - 超过重试次数，发送告警通知

4. 任务监控
   - 记录任务开始时间、结束时间、执行时长
   - 监控锁竞争情况（等待时间、获取成功率）
   - 任务执行超时告警
```

### 案例3：金融交易幂等控制

**场景**：支付接口需要保证同一笔订单只扣款一次

**核心挑战**：
- 网络超时后客户端重试
- 重复请求可能导致重复扣款
- 需要强一致性保证

**架构设计**：

```
┌─────────────────────────────────────────────────────────┐
│                   金融交易幂等控制                        │
│                                                         │
│   客户端 → Gateway → 幂等校验 → 分布式锁 → 业务处理      │
│              ↑                                      │
│              └────────── 数据库（已处理记录）            │
│                                                         │
│   分布式锁选择：                                         │
│   - 方案A：Redis Redisson（性能优先，最终一致）         │
│   - 方案B：ZooKeeper Curator（强一致，推荐）            │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**代码实现（ZooKeeper方案）**：

```java
/**
 * 基于ZooKeeper的幂等控制
 */
@Service
public class IdempotentPaymentService {
    
    @Autowired
    private CuratorFramework curatorFramework;
    
    @Autowired
    private PaymentRepository paymentRepository;
    
    private static final String LOCK_PATH = "/locks/payment/";
    private static final String PROCESSED_PATH = "/processed/payment/";
    
    /**
     * 处理支付请求（幂等）
     */
    public PaymentResult processPayment(PaymentRequest request) {
        String orderId = request.getOrderId();
        String lockPath = LOCK_PATH + orderId;
        String processedPath = PROCESSED_PATH + orderId;
        
        // 1. 检查是否已处理（幂等键）
        if (isAlreadyProcessed(processedPath)) {
            log.info("订单已处理，返回缓存结果，orderId={}", orderId);
            return getCachedResult(orderId);
        }
        
        // 2. 获取分布式锁
        InterProcessMutex lock = new InterProcessMutex(curatorFramework, lockPath);
        
        try {
            // 等待最多5秒获取锁
            boolean locked = lock.acquire(5, TimeUnit.SECONDS);
            if (!locked) {
                throw new PaymentException("系统繁忙，请稍后重试");
            }
            
            try {
                // 双重检查（获取锁后再次检查）
                if (isAlreadyProcessed(processedPath)) {
                    return getCachedResult(orderId);
                }
                
                // 3. 执行业务逻辑
                PaymentResult result = executePayment(request);
                
                // 4. 标记为已处理
                markAsProcessed(processedPath, result);
                
                // 5. 缓存结果
                cacheResult(orderId, result);
                
                return result;
                
            } finally {
                lock.release();
            }
        } catch (Exception e) {
            log.error("支付处理异常，orderId={}", orderId, e);
            throw new PaymentException("支付处理失败", e);
        }
    }
    
    private boolean isAlreadyProcessed(String path) {
        try {
            return curatorFramework.checkExists().forPath(path) != null;
        } catch (Exception e) {
            log.error("检查处理状态失败，path={}", path, e);
            return false;
        }
    }
    
    private void markAsProcessed(String path, PaymentResult result) throws Exception {
        byte[] data = JsonUtils.toJson(result).getBytes();
        curatorFramework.create()
            .creatingParentsIfNeeded()
            .withMode(CreateMode.PERSISTENT)
            .forPath(path, data);
    }
    
    private PaymentResult executePayment(PaymentRequest request) {
        // 1. 参数校验
        validateRequest(request);
        
        // 2. 风控检查
        riskCheck(request);
        
        // 3. 扣款（调用支付渠道）
        DeductResult deductResult = callPaymentChannel(request);
        
        // 4. 记录交易流水
        Transaction transaction = recordTransaction(request, deductResult);
        
        // 5. 更新订单状态
        updateOrderStatus(request.getOrderId(), OrderStatus.PAID);
        
        return new PaymentResult(
            transaction.getId(),
            transaction.getStatus(),
            transaction.getAmount(),
            transaction.getChannelResponse()
        );
    }
    
    // 其他辅助方法...
}
```

**幂等控制的关键设计**：

```
1. 幂等键设计
   - 业务唯一标识：orderId + userId + amount
   - 或客户端生成的幂等键：Idempotency-Key header
   - 存储方式：Redis（TTL=24小时）或数据库（永久）

2. 双重检查（Double Check）
   第一次检查：减少锁竞争
   第二次检查：防止并发获取锁后的重复处理

3. 锁粒度
   - 按订单粒度：lock:payment:{orderId}
   - 避免全局锁导致性能瓶颈

4. 结果缓存
   - 已处理的请求返回缓存结果
   - 缓存TTL：24小时（匹配幂等键有效期）

5. 异常处理
   - 锁获取失败：返回"系统繁忙"
   - 业务异常：不标记为已处理，允许重试
   - 网络超时：客户端重试，服务端幂等控制保证安全
```

---

## 性能分析与压测数据

### 1. 不同分布式锁方案性能对比

```
压测环境：
- Redis：3主3从 Cluster，AWS r6g.2xlarge
- ZooKeeper：3节点集群，AWS m6g.xlarge
- ETCD：3节点集群，AWS m6g.xlarge
- 客户端：Java 17，8核16G

压测工具：JMeter 1000并发线程

┌─────────────────┬──────────┬──────────┬──────────┬──────────┐
│     指标        │ SETNX    │ Redisson │ RedLock  │ ZooKeeper│
├─────────────────┼──────────┼──────────┼──────────┼──────────┤
│ 平均延迟        │ 0.5ms    │ 1.2ms    │ 3.5ms    │ 8.5ms    │
│ P99延迟         │ 1.2ms    │ 3.5ms    │ 12ms     │ 25ms     │
│ QPS（单机）     │ 50,000   │ 25,000   │ 8,000    │ 3,000    │
│ QPS（集群）     │ 300,000  │ 150,000  │ 40,000   │ 8,000    │
│ CPU占用         │ 20%      │ 35%      │ 60%      │ 45%      │
│ 内存占用        │ 50MB     │ 150MB    │ 300MB    │ 200MB    │
│ 网络带宽        │ 5MB/s    │ 15MB/s   │ 45MB/s   │ 10MB/s   │
└─────────────────┴──────────┴──────────┴──────────┴──────────┘
```

### 2. Redisson锁性能调优

```
调优方向1：连接池配置
┌─────────────────────────────────────┐
│ 参数：                               │
│ - connectionMinimumIdleSize: 24      │
│ - connectionPoolSize: 64             │
│ - subscriptionConnectionMinimumIdleSize: 1│
│ - subscriptionConnectionPoolSize: 50 │
│                                     │
│ 原理：                                │
│ - 锁操作是高频短连接                  │
│ - 需要足够的连接避免等待              │
│ - 订阅连接用于看门狗和解锁通知         │
└─────────────────────────────────────┘

调优方向2：序列化方式
┌─────────────────────────────────────┐
│ 默认：JsonJacksonCodec               │
│ 优化：KryoCodec / FSTCodec           │
│                                     │
│ 效果：                                │
│ - 序列化体积减少50%                  │
│ - 序列化时间减少60%                  │
│ - 网络传输更快                       │
└─────────────────────────────────────┘

调优方向3：看门狗超时
┌─────────────────────────────────────┐
│ 默认：30秒                           │
│ 优化：根据业务最大执行时间设置         │
│                                     │
│ 场景：                                │
│ - 短任务（<1秒）：10秒               │
│ - 中任务（1-10秒）：30秒             │
│ - 长任务（>10秒）：60-300秒          │
│                                     │
│ 注意：超时时间越短，崩溃后恢复越快     │
│       但续期频率越高，Redis压力越大   │
└─────────────────────────────────────┘

调优方向4：锁粒度
┌─────────────────────────────────────┐
│ 错误：lock:order                     │
│ 正确：lock:order:{orderId}           │
│                                     │
│ 效果：                                │
│ - 细粒度锁减少竞争                   │
│ - 并发度提升10-100倍                 │
│ - 但Redis key数量增加                │
│ - 需要监控key数量，避免内存溢出        │
└─────────────────────────────────────┘
```

### 3. 锁竞争监控与告警

```java
/**
 * 分布式锁监控指标
 */
@Component
public class LockMetrics {
    
    private final MeterRegistry meterRegistry;
    private final ConcurrentHashMap<String, Timer> lockTimers = new ConcurrentHashMap<>();
    
    /**
     * 记录锁获取时间
     */
    public void recordLockAcquire(String lockName, long timeMs, boolean success) {
        Timer timer = lockTimers.computeIfAbsent(lockName, 
            name -> Timer.builder("redisson.lock.acquire")
                .tag("lockName", name)
                .register(meterRegistry));
        
        timer.record(timeMs, TimeUnit.MILLISECONDS);
        
        // 计数器
        meterRegistry.counter("redisson.lock.acquire.count",
            "lockName", lockName,
            "success", String.valueOf(success)
        ).increment();
    }
    
    /**
     * 记录锁持有时间
     */
    public void recordLockHoldTime(String lockName, long timeMs) {
        meterRegistry.timer("redisson.lock.hold.time",
            "lockName", lockName
        ).record(timeMs, TimeUnit.MILLISECONDS);
    }
    
    /**
     * 记录看门狗续期次数
     */
    public void recordRenewal(String lockName) {
        meterRegistry.counter("redisson.lock.renewal",
            "lockName", lockName
        ).increment();
    }
    
    /**
     * 告警规则
     */
    @EventListener
    public void onMetrics(MetricEvent event) {
        // 锁获取成功率 < 95%，告警
        // 锁平均获取时间 > 100ms，告警
        // 锁平均持有时间 > 看门狗超时时间的50%，告警
        // 看门狗续期次数 > 10次，告警（业务执行时间过长）
    }
}
```

---

## 常见陷阱与最佳实践

### 陷阱1：锁未释放导致死锁

```java
// 错误示例：异常后未释放锁
public void badPractice() {
    RLock lock = redissonClient.getLock("myLock");
    lock.lock();
    
    // 业务代码抛出异常...
    doSomething();  // 可能抛出RuntimeException
    
    lock.unlock();  // 如果异常，这行不会执行！
}

// 正确做法：try-finally
public void goodPractice() {
    RLock lock = redissonClient.getLock("myLock");
    lock.lock();
    
    try {
        doSomething();
    } finally {
        // 确保锁一定会被释放
        if (lock.isHeldByCurrentThread()) {
            lock.unlock();
        }
    }
}
```

### 陷阱2：锁粒度过粗

```java
// 错误：锁整个方法，并发度极低
public void processAllOrders() {
    RLock lock = redissonClient.getLock("order");  // 所有订单共享一把锁
    lock.lock();
    try {
        // 处理所有订单...
    } finally {
        lock.unlock();
    }
}

// 正确：锁具体订单
public void processOrder(Long orderId) {
    RLock lock = redissonClient.getLock("order:" + orderId);  // 每个订单独立锁
    lock.lock();
    try {
        // 处理单个订单...
    } finally {
        lock.unlock();
    }
}
```

### 陷阱3：在锁内执行耗时操作

```java
// 错误：锁内执行RPC调用、HTTP请求
public void badPractice() {
    RLock lock = redissonClient.getLock("order:" + orderId);
    lock.lock();
    try {
        // 锁内执行RPC（可能超时、重试）
        rpcService.callExternalSystem();  // 可能耗时5-10秒
        
        // 锁内执行HTTP请求
        httpClient.post(slowUrl);  // 可能耗时3-5秒
        
        // 锁内执行数据库批量操作
        jdbcTemplate.batchUpdate(largeBatch);  // 可能耗时10秒+
    } finally {
        lock.unlock();
    }
}

// 正确：锁内只执行必要的原子操作
public void goodPractice() {
    // 1. 在锁外准备数据
    Data data = prepareData();
    
    RLock lock = redissonClient.getLock("order:" + orderId);
    lock.lock();
    try {
        // 2. 锁内只执行状态检查和快速更新
        Order order = orderRepository.findById(orderId);
        if (order.getStatus() != Status.PENDING) {
            throw new IllegalStateException("订单状态不正确");
        }
        order.setStatus(Status.PROCESSING);
        orderRepository.save(order);
    } finally {
        lock.unlock();
    }
    
    // 3. 在锁外执行耗时操作
    try {
        rpcService.callExternalSystem();
        httpClient.post(slowUrl);
    } catch (Exception e) {
        // 补偿逻辑：回滚状态
        compensateOrderStatus(orderId);
    }
}
```

### 陷阱4：主从延迟导致锁丢失

```
问题场景：
┌─────────────────────────────────────┐
│  主节点写入锁                        │
│  ↓                                  │
│  主节点宕机（锁尚未同步到从节点）      │
│  ↓                                  │
│  从节点提升为主节点                   │
│  ↓                                  │
│  从节点上没有锁的信息                 │
│  ↓                                  │
│  其他客户端获取到同一把锁             │
│  ↓                                  │
│  两个客户端同时持有锁                │
└─────────────────────────────────────┘

解决方案：
1. 强一致性场景使用ZooKeeper/ETCD
2. 使用RedLock（有争议，但比单节点安全）
3. 使用Fencing Token防护
4. 配置min-slaves-to-write 1（Redis 2.8+）
```

### 陷阱5：看门狗配置不当

```java
// 错误：指定leaseTime，关闭看门狗，但业务执行时间不确定
public void badPractice() {
    RLock lock = redissonClient.getLock("myLock");
    lock.lock(10, TimeUnit.SECONDS);  // 10秒后自动释放
    
    try {
        // 业务逻辑可能执行15秒
        unpredictableBusinessLogic();
    } finally {
        lock.unlock();
    }
}

// 正确：不指定leaseTime，使用看门狗自动续期
public void goodPractice() {
    RLock lock = redissonClient.getLock("myLock");
    lock.lock();  // 启用看门狗
    
    try {
        unpredictableBusinessLogic();
    } finally {
        lock.unlock();
    }
}

// 如果必须指定leaseTime，确保大于业务最大执行时间
public void alternative() {
    RLock lock = redissonClient.getLock("myLock");
    lock.lock(60, TimeUnit.SECONDS);  // 确保60秒足够
    
    try {
        // 添加超时控制，确保在60秒内完成
        businessLogicWithTimeout(55, TimeUnit.SECONDS);
    } finally {
        lock.unlock();
    }
}
```

### 陷阱6：锁竞争导致性能瓶颈

```
症状：
- 系统吞吐量突然下降
- 响应时间飙升
- Redis CPU使用率100%
- 大量线程BLOCKED

诊断：
1. 检查锁粒度：是否使用了全局锁？
2. 检查锁持有时间：是否过长？
3. 检查锁竞争频率：是否过高？
4. 检查Redis慢查询：是否有Lua脚本执行慢？

优化方案：
1. 细化锁粒度（用户级别、订单级别）
2. 减少锁持有时间（锁内只做必要操作）
3. 使用读写锁（RedissonReadWriteLock）
4. 批量操作替代循环加锁
5. 本地锁+分布式锁双层锁（先本地，后分布式）
```

### 最佳实践总结

```
1. 永远使用try-finally释放锁
   RLock lock = redissonClient.getLock(key);
   lock.lock();
   try {
       // 业务逻辑
   } finally {
       if (lock.isHeldByCurrentThread()) {
           lock.unlock();
       }
   }

2. 优先使用看门狗（不指定leaseTime）
   lock.lock();  // 启用看门狗
   
   而非：
   lock.lock(10, TimeUnit.SECONDS);  // 关闭看门狗

3. 细化锁粒度
   lock:order:{orderId} 而非 lock:order
   lock:user:{userId}    而非 lock:user

4. 锁内只做原子操作
   - 状态检查
   - 快速更新
   - 发送消息（异步处理）
   
   不做：
   - RPC调用
   - HTTP请求
   - 复杂计算
   - 大数据库查询

5. 设置合理的看门狗超时
   - 默认值30秒适合大多数场景
   - 长任务可以设置为60-300秒
   - 但不要太长（崩溃后恢复慢）

6. 监控锁指标
   - 锁获取成功率
   - 锁平均获取时间
   - 锁平均持有时间
   - 看门狗续期次数

7. 选择合适的分布式锁方案
   - 一般业务：Redisson + Redis
   - 强一致：ZooKeeper / ETCD
   - 云原生：ETCD / K8s Lease

8. 降级策略
   - Redis故障时降级为数据库锁
   - 限流保护，避免雪崩
   - 熔断机制，快速失败
```

---

## 面试题与参考答案

### Q1：SETNX 实现分布式锁有哪些问题？如何解决？

**参考答案：**

```
SETNX实现分布式锁的四大问题：

1. 不可重入
   问题：同一线程无法重复获取锁
   解决：使用Hash结构记录线程ID和重入次数
   Redisson实现：hincrby lock_key threadId 1

2. 无看门狗（自动续期）
   问题：业务执行时间长，锁过期后被其他线程获取
   解决：启动定时任务，定期续期
   Redisson实现：每10秒检查，续期到30秒

3. 主从延迟导致锁丢失
   问题：主节点宕机，从节点提升为主，锁可能丢失
   解决：
   - 方案A：使用RedLock（多节点）
   - 方案B：使用ZooKeeper/ETCD（强一致）
   - 方案C：使用Fencing Token防护

4. 释放锁不安全
   问题：客户端A的锁过期后，客户端B获取锁，客户端A完成后删除B的锁
   解决：释放锁时使用Lua脚本，先判断value是否匹配，再删除
   
   Lua脚本：
   if redis.call('get', KEYS[1]) == ARGV[1] then
       return redis.call('del', KEYS[1])
   else
       return 0
   end
```

### Q2：Redisson 如何实现可重入锁？

**参考答案：**

```
Redisson可重入锁的实现原理：

1. 数据结构：Hash
   Key: lock:order:123
   Type: Hash
   Field: 58c8a...:1（格式：UUID前8位:线程ID）
   Value: 重入次数（Integer）

2. 加锁Lua脚本逻辑：
   - 如果锁不存在（exists=0）：创建Hash，重入次数=1，设置过期时间
   - 如果锁存在且是当前线程（hexists=1）：重入次数+1，刷新过期时间
   - 如果锁被其他线程占用：返回剩余时间（pttl）

3. 解锁Lua脚本逻辑：
   - 如果锁不属于当前线程：返回nil（抛出异常）
   - 重入次数-1（hincrby -1）
   - 如果结果>0：刷新过期时间，返回0（未完全释放）
   - 如果结果=0：删除锁，发布解锁消息，返回1（完全释放）

4. 可重入的好处：
   - 同一线程可以多次获取同一把锁
   - 避免死锁（递归调用、嵌套方法）
   - 与Java的ReentrantLock语义一致
```

### Q3：看门狗机制的原理是什么？

**参考答案：**

```
Redisson看门狗（Watch Dog）机制：

1. 触发条件：
   - 调用lock()或tryLock()不指定leaseTime时自动启用
   - 调用lock(10, TimeUnit.SECONDS)时不会启用

2. 工作流程：
   - 获取锁后，启动定时任务（Netty的HashedWheelTimer）
   - 定时任务间隔：leaseTime / 3（默认30秒/3=10秒）
   - 每次检查：如果锁仍被当前线程持有，执行pexpire续期
   - 业务完成解锁后，取消定时任务

3. 续期Lua脚本：
   if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then
       redis.call('pexpire', KEYS[1], ARGV[1]);
       return 1;
   end;
   return 0;

4. 崩溃安全：
   - 应用崩溃（OOM/Kill -9）：看门狗停止续期，锁自然过期
   - 网络断开：看门狗无法续期，锁自然过期
   - JVM GC暂停：如果暂停时间<TTL，看门狗恢复后续期；如果>TTL，锁丢失

5. 配置：
   Config config = new Config();
   config.setLockWatchdogTimeout(30000L);  // 默认30秒
```

### Q4：RedLock 为什么有争议？Martin Kleppmann 的质疑是什么？

**参考答案：**

```
RedLock的争议来源：

Martin Kleppmann在《How to do distributed locking》中提出三点质疑：

1. 时钟跳跃（Clock Skew）
   - Redis节点使用系统时钟计算TTL
   - 如果NTP同步导致时钟向前跳跃，锁会提前过期
   - 两个客户端可能同时持有锁
   - 即使使用monotonic clock，也无法完全避免（虚拟机迁移、时间调整）

2. 客户端GC暂停（GC Pause）
   - 客户端获取锁后，JVM触发Full GC暂停15秒
   - 锁在Redis中已过期（TTL=10秒）
   - GC恢复后，客户端继续操作，但锁已被其他客户端获取
   - 两个客户端同时操作共享资源

3. 网络延迟（Network Delay）
   - 客户端发送加锁请求，网络延迟500ms
   - 请求到达Redis时，锁已过期
   - 客户端以为自己持有锁10秒，实际只有9.5秒
   - 安全窗口被缩短

Martin的结论：
- RedLock不能提供强一致性保证
- 任何基于时钟的分布式锁都有边界条件
- 建议：强一致性场景使用ZooKeeper/ETCD（基于共识算法，不依赖时钟）

工程界的实际选择：
- 一般业务：Redisson单节点+看门狗（AP方案，性能优先）
- 高可用：Redisson Cluster+看门狗（故障自动转移）
- 强一致：ZooKeeper/ETCD（CP方案，安全优先）
```

### Q5：Redis 分布式锁和 ZooKeeper 分布式锁如何选择？

**参考答案：**

```
选择依据：CAP定理和业务场景

┌─────────────────┬─────────────────┬─────────────────┐
│     维度        │   Redis         │  ZooKeeper      │
├─────────────────┼─────────────────┼─────────────────┤
│ 一致性          │ 最终一致（AP）   │ 强一致（CP）     │
├─────────────────┼─────────────────┼─────────────────┤
│ 性能            │ 高（10万QPS）    │ 中（数千QPS）    │
├─────────────────┼─────────────────┼─────────────────┤
│ 可靠性          │ 主从可能丢锁     │ 不丢锁          │
├─────────────────┼─────────────────┼─────────────────┤
│ 复杂度          │ 低              │ 高              │
├─────────────────┼─────────────────┼─────────────────┤
│ 运维成本        │ 低              │ 高              │
├─────────────────┼─────────────────┼─────────────────┤
│ 自动续期        │ 看门狗          │ 临时节点自动释放 │
├─────────────────┼─────────────────┼─────────────────┤
│ 适用场景        │ 一般业务锁       │ 强一致性要求     │
└─────────────────┴─────────────────┴─────────────────┘

具体场景建议：

1. 电商库存扣减（高并发，允许短暂不一致）
   → Redis + Redisson
   原因：性能要求极高，库存可回滚

2. 金融交易（强一致性，不可丢锁）
   → ZooKeeper / ETCD
   原因：资金安全，不能双花

3. 分布式任务调度（定时任务去重）
   → Redis + Redisson
   原因：性能要求不高，但Redis运维简单

4. 分布式配置管理（配置一致性）
   → ETCD
   原因：Watch机制实时推送，强一致

5. K8s环境下的Leader选举
   → ETCD / K8s Lease API
   原因：与K8s生态集成
```

### Q6：如何避免分布式锁导致的性能瓶颈？

**参考答案：**

```
避免分布式锁性能瓶颈的六大策略：

1. 细化锁粒度
   - 全局锁 → 用户级锁 → 订单级锁
   - 示例：lock:order:{orderId} 替代 lock:order
   - 效果：并发度提升10-100倍

2. 减少锁持有时间
   - 锁内只执行必要的原子操作
   - RPC、HTTP、复杂计算移到锁外
   - 目标：锁持有时间 < 100ms

3. 使用读写锁（ReadWriteLock）
   - 读多写少的场景
   - RedissonReadWriteLock
   - 读锁共享，写锁互斥

4. 本地锁 + 分布式锁双层锁
   - 先获取本地锁（synchronized）
   - 再获取分布式锁
   - 减少分布式锁竞争（同JVM内先过滤）

5. 批量操作替代循环加锁
   - 错误：for (id : ids) { lock(id); process(); unlock(); }
   - 正确：lock(batch); processBatch(); unlock();

6. 异步化处理
   - 锁内发送消息，异步执行耗时操作
   - 减少锁持有时间，提升吞吐量

监控指标：
- 锁获取成功率（目标>99%）
- 锁平均获取时间（目标<10ms）
- 锁竞争等待时间（目标<50ms）
- 锁持有时间分布（P99<1秒）
```

### Q7：分布式锁在微服务架构中的最佳实践？

**参考答案：**

```
微服务架构中的分布式锁实践：

1. 锁服务抽象
   ┌─────────────────────────────────────┐
   │  业务服务                            │
   │  ↓                                  │
   │  LockService（抽象接口）             │
   │  ├─ RedisLockService（默认）         │
   │  ├─ ZkLockService（强一致）          │
   │  └─ EtcdLockService（云原生）        │
   │  ↓                                  │
   │  存储层（Redis/ZK/ETCD）             │
   └─────────────────────────────────────┘

2. 锁命名规范
   {service}:{resource}:{id}
   例如：
   - order:lock:order:10086
   - inventory:lock:sku:20010
   - payment:lock:transaction:30001

3. 锁超时与降级
   - 获取锁超时：快速失败，返回"系统繁忙"
   - Redis故障：降级为数据库乐观锁
   - 熔断机制：连续失败10次，熔断30秒

4. 分布式事务配合
   - 锁保证单服务内的互斥
   - Saga/TCC保证跨服务的事务一致性
   - 锁在事务开始前获取，事务提交后释放

5. 上下文传递
   - 锁信息通过Tracing上下文传递
   - 便于排查锁竞争问题
   - 记录锁的获取、释放、续期日志

6. 配置中心管理
   - 看门狗超时时间：配置中心动态调整
   - 锁策略切换：Redis故障时自动切换ZK
   - 灰度发布：新锁策略灰度验证
```

---

*此文原创，转载请注明出处。*
