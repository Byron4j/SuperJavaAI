# Eureka深度解析：服务注册与发现的源码级剖析

**文章标签：** #java #springcloud #eureka #微服务 #服务注册发现 #源码分析 #面试 #netflix

## 目录

- [引言：服务注册与发现的本质](#引言服务注册与发现的本质)
- [理论基础：分布式系统中的服务发现](#理论基础分布式系统中的服务发现)
- [演进史：从Netflix OSS到Spring Cloud生态](#演进史从netflix-oss到spring-cloud生态)
- [源码深度分析：Eureka Server核心机制](#源码深度分析eureka-server核心机制)
- [源码深度分析：Eureka Client核心机制](#源码深度分析eureka-client核心机制)
- [源码深度分析：Peer复制与自我保护](#源码深度分析peer复制与自我保护)
- [实战案例：生产级Eureka集群部署](#实战案例生产级eureka集群部署)
- [对比分析：Eureka vs Consul vs Nacos vs ZooKeeper](#对比分析eureka-vs-consul-vs-nacos-vs-zookeeper)
- [性能分析：注册中心的瓶颈与优化](#性能分析注册中心的瓶颈与优化)
- [常见陷阱与最佳实践](#常见陷阱与最佳实践)
- [面试题与参考答案](#面试题与参考答案)

---

## 引言：服务注册与发现的本质

微服务架构将单体应用拆分为数十甚至上百个独立服务，服务间的网络拓扑呈指数级增长。在这种环境下，**服务注册与发现（Service Registration & Discovery）**不再是可选的基础设施，而是微服务通信的基石。

### 核心问题域

```
┌─────────────────────────────────────────────────────────────┐
│                    微服务通信的核心挑战                        │
├─────────────────────────────────────────────────────────────┤
│ 1. 服务位置动态化：IP和端口随弹性伸缩不断变化                    │
│ 2. 服务健康不透明：实例可能随时宕机或网络分区                    │
│ 3. 调用链路复杂化：N个服务互相调用形成M x N的依赖矩阵           │
│ 4. 环境隔离需求：开发、测试、生产环境需独立管理                  │
└─────────────────────────────────────────────────────────────┘
```

**服务注册与发现的核心语义**：

```
服务注册（Registration）：
   服务实例启动 ──HTTP POST──> 注册中心
   携带信息：{serviceName, instanceId, ip, port, metadata, healthUrl}

服务发现（Discovery）：
   消费方 ──HTTP GET──> 注册中心
   获取：serviceName -> [Instance1, Instance2, Instance3]

服务续约（Renewal）：
   服务实例 ──HTTP PUT──> 注册中心（每30秒）
   目的：证明"我还活着"

服务下线（Deregistration）：
   服务实例 ──HTTP DELETE──> 注册中心
   或：注册中心超时未收到续约则主动剔除
```

### Eureka的定位

Eureka是Netflix开源的**服务注册与发现组件**，采用**AP架构**（可用性优先、最终一致），专为大规模微服务集群设计。在Spring Cloud Netflix生态中，Eureka是默认的服务注册中心实现。

```
┌─────────────────────────────────────────────────────────────┐
│                    Eureka在Spring Cloud中的位置               │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│   服务提供方              Eureka Server           服务消费方    │
│   ┌─────────┐           ┌──────────┐           ┌─────────┐  │
│   │Service A│──注册────>│          │<──发现────│Service B│  │
│   │(Provider)│ 续约     │ Registry │           │(Consumer)│  │
│   └─────────┘           │  Server  │           └─────────┘  │
│        ▲                └──────────┘                ▲       │
│        │                     ▲                      │       │
│        └─────────────────────┘                      │       │
│              注册表同步(Peer Replication)             │       │
│                                                      │       │
│   ┌─────────┐                                       │       │
│   │Service C│───────────────────────────────────────┘       │
│   │(Provider)│              直接调用(Ribbon/Feign)           │
│   └─────────┘                                               │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

**关键认知**：Eureka不是简单的"服务地址簿"，它是一个**具有自我保护能力、支持集群对等复制的分布式协调服务**。理解Eureka需要深入到CAP权衡、租约机制、异步复制模型等分布式系统核心概念。

---

## 理论基础：分布式系统中的服务发现

### 1. CAP定理与服务发现

CAP定理指出分布式系统无法同时满足一致性（Consistency）、可用性（Availability）、分区容错性（Partition Tolerance）。服务注册中心作为分布式系统的协调者，必须在CAP中做出权衡。

```
CAP权衡空间：

                    Consistency
                         ▲
                        /│\
                       / │ \
                      /  │  \
                     /   │   \
                    /    │    \
                   /     │     \
                  /      │      \
         CP系统  /───────┼───────\  AP系统
        (ZooKeeper)     │      (Eureka)
                         │
                  Partition Tolerance
                         │
                         ▼
                    Availability

CP系统（ZooKeeper/etcd/Consul强一致模式）：
- 优点：注册信息强一致，不会读到过期数据
- 缺点：网络分区时牺牲可用性（无法注册/发现）
- 适用：对数据一致性要求极高的场景

AP系统（Eureka/Nacos AP模式）：
- 优点：网络分区时仍可注册和发现，保证可用性
- 缺点：可能读取到过期的服务列表
- 适用：大规模微服务，允许短暂不一致
```

**Eureka的AP选择**：

Eureka选择AP是因为在微服务架构中，**可用性比强一致性更重要**。原因如下：

1. **服务实例的短暂性**：微服务实例频繁扩缩容，强一致带来的延迟不可接受
2. **客户端缓存兜底**：Eureka Client本地缓存服务列表，即使Server全部宕机也能继续调用
3. **心跳机制的自然收敛**：90秒内无心跳则剔除，不一致窗口可控
4. **自我保护机制**：网络分区时不盲目剔除服务，避免雪崩

### 2. 服务发现模式：客户端发现 vs 服务端发现

```
服务端发现模式（Server-Side Discovery）：

Client ──请求──> Load Balancer(Nginx/Envoy) ──转发──> Service Instance
                         ▲
                         │
                    查询注册中心
                         │
                    Consul/etcd

特点：
- 客户端无感知，无需维护服务列表
- 增加网络跳数（LB层）
- 适合外部流量入口

客户端发现模式（Client-Side Discovery）：

Client ──查询──> Registry(Eureka) ──返回列表──> Client
  │                                              │
  └────────────直接调用──────────────────────────> Service Instance

特点：
- 客户端直接连接目标实例，减少网络跳数
- 客户端需实现负载均衡逻辑（Ribbon/Spring Cloud LoadBalancer）
- 本地缓存服务列表，提高可用性
- 适合服务间内部调用
```

Eureka采用**客户端发现模式**，这是Spring Cloud微服务间调用的标准架构。

### 3. 租约机制（Lease Mechanism）

租约是分布式系统中管理资源生命周期的核心机制。Eureka使用租约来跟踪服务实例的健康状态。

```
租约的生命周期：

时间轴 ───────────────────────────────────────────────────────>

    注册          续约        续约        续约       失效剔除
      │            │           │           │            │
      ▼            ▼           ▼           ▼            ▼
   ┌────┐       ┌────┐      ┌────┐      ┌────┐     ┌──────┐
   │注册│       │心跳│      │心跳│      │心跳│     │超时未│
   │    │       │30s │      │30s │      │30s │     │收到  │
   └────┘       └────┘      └────┘      └────┘     └──────┘
      │            │           │           │            │
   租约开始    租约续期    租约续期    租约续期     租约到期
                                                (默认90s)

关键参数：
- lease-renewal-interval-in-seconds: 30 (续约间隔)
- lease-expiration-duration-in-seconds: 90 (租约过期时间)
- eviction-interval-timer-in-ms: 60000 (剔除任务执行间隔，Server端)
```

**租约的数学本质**：

```
设续约间隔为R，过期时间为E，则：
- 正常情况：服务每R秒发送一次心跳
- 容错能力：服务最多可以连续丢失 (E/R - 1) 次心跳
- 对于默认值：最多丢失 (90/30 - 1) = 2 次心跳

这意味着：
- 两次连续心跳丢失不会导致实例被剔除
- 三次心跳丢失（90秒内无心跳）才会触发剔除
- 这种设计容忍了短暂的网络抖动
```

### 4. 最终一致性与Gossip协议

Eureka Server集群采用**异步复制**实现最终一致性，这与Gossip协议有相似之处。

```
Eureka Server对等复制模型：

        ┌─────────────┐
        │ Eureka S1   │
        │ (注册表A)   │
        └──────┬──────┘
               │
     ┌─────────┼─────────┐
     ▼         ▼         ▼
┌────────┐ ┌────────┐ ┌────────┐
│Eureka S2│ │Eureka S3│ │Eureka S4│
└────────┘ └────────┘ └────────┘

复制流程：
1. Client注册到S1
2. S1将增量更新放入ReplicationQueue
3. S1异步向S2、S3、S4发送/replicate请求
4. S2、S3、S4收到后合并到本地注册表
5. 复制延迟通常在毫秒到秒级

一致性模型：
- 读取：任何Server都可读，可能读到稍微过期的数据
- 写入：任何Server都可写，写入后立即返回（异步复制）
- 冲突解决：以时间戳为准（last-write-wins）
```

---

## 演进史：从Netflix OSS到Spring Cloud生态

### 第一阶段：Netflix内部孵化（2012-2014）

Eureka诞生于Netflix大规模微服务化过程中。Netflix在2012年开始将单体应用拆分为微服务，迫切需要服务发现机制。

```
Netflix OSS技术栈的诞生背景：

2012年：Netflix开始全面迁移到AWS云
2013年：发布Eureka 1.0，解决AWS EC2实例动态IP问题
2014年：发布Hystrix、Ribbon、Zuul，形成完整微服务套件

设计约束（AWS环境）：
- EC2实例随时可能故障（Non-deterministic failure）
- 自动扩缩容导致IP频繁变化
- 可用区（AZ）之间的网络延迟
- 没有固定基础设施（No fixed infrastructure）

Eureka的设计哲学：
"在AWS这样的云环境中，故障是常态而非异常。注册中心必须首先保证可用性，
允许短暂的不一致，而不是在分区时拒绝服务。"
```

### 第二阶段：Spring Cloud Netflix集成（2014-2018）

Spring Cloud团队将Netflix OSS组件集成到Spring生态，Eureka成为Spring Cloud默认注册中心。

```
Spring Cloud Netflix Eureka 里程碑：

Spring Cloud Angel (2015)：初步集成Eureka
- @EnableEurekaServer / @EnableEurekaClient 注解
- Spring Boot Starter简化配置

Spring Cloud Brixton (2016)：完善集群支持
- 多区域（Region/Zone） awareness
- 更完善的安全集成

Spring Cloud Dalston (2017)：稳定性增强
- 改进的自我保护机制
- 更好的度量指标支持

Spring Cloud Finchley (2018)：与Spring Boot 2.0兼容
- 响应式编程初步支持
- 改进的健康检查
```

### 第三阶段：维护模式与社区迁移（2018-2020）

Netflix宣布Eureka 2.0开发停止，Eureka 1.x进入维护模式。Spring Cloud社区开始寻找替代方案。

```
关键事件时间线：

2018.07：Netflix宣布Eureka 2.0项目停止开发
          - Eureka 2.0原计划引入更强的一致性保证
          - 由于内部需求变化，项目被取消
          - Eureka 1.x继续维护但不再添加新特性

2018.12：Spring Cloud团队宣布Netflix组件进入维护模式
          - Hystrix停止开发
          - Ribbon停止开发（由Spring Cloud LoadBalancer替代）
          - Zuul 1.x停止开发（由Gateway替代）
          - Eureka继续维护但标记为"maintenance"

2020.04：Spring Cloud 2020.0 (Ilford) 发布
          - 默认移除Ribbon（改为Spring Cloud LoadBalancer）
          - 默认移除Hystrix（改为Resilience4j/Sentinel）
          - Eureka Client继续保留但变为可选
          - 推荐Nacos、Consul作为替代注册中心
```

### 第四阶段：当前状态（2021-2026）

Eureka 1.x仍然是许多存量系统的核心组件。Spring Cloud 2021.x/2022.x/2023.x继续提供Eureka支持，但社区重心已转向Nacos和Kubernetes原生服务发现。

```
当前Eureka的使用场景：

1. 存量系统维护：
   - 已有大量基于Eureka的微服务集群
   - 迁移成本高，继续使用Eureka

2. 学习价值：
   - Eureka的AP设计是理解分布式系统的优秀案例
   - 源码相对简单，适合深入学习

3. 小规模项目：
   - 团队技术栈以Spring Cloud Netflix为主
   - 不需要Nacos的配置中心功能

替代方案选择：
- Nacos：阿里开源，AP/CP可选，自带配置中心，控制台丰富
- Consul：HashiCorp开源，CP默认，多数据中心，服务网格
- Kubernetes DNS：云原生方案，无需额外组件
- etcd：CoreOS开源，强一致，Kubernetes底层存储
```

---

## 源码深度分析：Eureka Server核心机制

### 1. 整体架构与核心类图

```
Eureka Server核心类结构：

┌─────────────────────────────────────────────────────────────┐
│                    REST API Layer                            │
│  ApplicationResource    InstanceResource    PeerReplication  │
│       │                      │                   Resource    │
│       ▼                      ▼                       ▼       │
├─────────────────────────────────────────────────────────────┤
│                  Registry Layer                              │
│  PeerAwareInstanceRegistryImpl ──extends──> AbstractInstance  │
│       │                                          Registry    │
│       ▼                                                      │
│  InstanceRegistry (Spring Cloud封装)                         │
├─────────────────────────────────────────────────────────────┤
│                   Data Layer                                 │
│  ConcurrentHashMap<String, Map<String, Lease<InstanceInfo>>> │
│       │                                                      │
│       ▼                                                      │
│  ResponseCacheImpl (读写缓存分离)                              │
├─────────────────────────────────────────────────────────────┤
│                  Peer Replication Layer                      │
│  PeerEurekaNodes ──manages──> PeerEurekaNode (HTTP Client)   │
└─────────────────────────────────────────────────────────────┘
```

### 2. 注册表数据结构

Eureka Server的核心是一个嵌套的ConcurrentHashMap，这是理解其性能特性的关键。

```java
// AbstractInstanceRegistry.java 核心数据结构

public abstract class AbstractInstanceRegistry implements InstanceRegistry {
    
    // 第一层：服务名称 -> (实例ID -> 租约)
    private final ConcurrentHashMap<String, Map<String, Lease<InstanceInfo>>> registry
            = new ConcurrentHashMap<>();
    
    // 注册表保护锁（按服务名分段）
    protected final Object lock = new Object();
    
    /**
     * 注册实例的核心方法
     */
    public void register(InstanceInfo registrant, int leaseDuration, boolean isReplication) {
        try {
            read.lock(); // 读锁保护（支持并发读）
            
            // 1. 获取该服务名的实例Map
            Map<String, Lease<InstanceInfo>> gMap = registry.get(registrant.getAppName());
            
            // 2. 如果是首个实例，创建新的ConcurrentHashMap
            if (gMap == null) {
                final ConcurrentHashMap<String, Lease<InstanceInfo>> gNewMap = 
                    new ConcurrentHashMap<>();
                gMap = registry.putIfAbsent(registrant.getAppName(), gNewMap);
                if (gMap == null) {
                    gMap = gNewMap;
                }
            }
            
            // 3. 检查是否已有该实例的租约
            Lease<InstanceInfo> existingLease = gMap.get(registrant.getId());
            
            // 4. 保留已存在实例的覆盖规则（last-dirty-timestamp）
            if (existingLease != null && (existingLease.getHolder() != null)) {
                Long existingLastDirtyTimestamp = existingLease.getHolder().getLastDirtyTimestamp();
                Long registrationLastDirtyTimestamp = registrant.getLastDirtyTimestamp();
                
                // 如果已有实例更新，则忽略此次注册（防止旧数据覆盖新数据）
                if (existingLastDirtyTimestamp != null && 
                    registrationLastDirtyTimestamp != null &&
                    existingLastDirtyTimestamp > registrationLastDirtyTimestamp) {
                    registrant = existingLease.getHolder();
                }
            } else {
                // 新实例注册：自我保护计数器增加
                synchronized (lock) {
                    if (this.expectedNumberOfClientsSendingRenews > 0) {
                        this.expectedNumberOfClientsSendingRenews++;
                        updateRenewsPerMinThreshold();
                    }
                }
            }
            
            // 5. 创建或更新租约
            Lease<InstanceInfo> lease = new Lease<>(registrant, leaseDuration);
            if (existingLease != null) {
                // 保留原租约的注册时间和服务上线时间
                lease.setServiceUpTimestamp(existingLease.getServiceUpTimestamp());
            }
            
            // 6. 放入注册表
            gMap.put(registrant.getId(), lease);
            
            // 7. 更新最近修改队列（用于增量同步）
            recentRegisteredQueue.add(new Pair<>(
                System.currentTimeMillis(), 
                registrant.getAppName() + "(" + registrant.getId() + ")"
            ));
            
        } finally {
            read.unlock();
        }
        
        // 8. 使缓存失效（触发ResponseCache刷新）
        invalidateCache(registrant.getAppName());
    }
}
```

**数据结构深度解析**：

```
registry 数据结构可视化：

registry (ConcurrentHashMap)
│
├── "USER-SERVICE" ──> ConcurrentHashMap
│   ├── "user-service:192.168.1.10:8080" ──> Lease<InstanceInfo>
│   │   ├── holder: InstanceInfo
│   │   │   ├── appName: "USER-SERVICE"
│   │   │   ├── instanceId: "user-service:192.168.1.10:8080"
│   │   │   ├── ipAddr: "192.168.1.10"
│   │   │   ├── status: UP
│   │   │   ├── port: 8080
│   │   │   └── metadata: { "version": "v1", "region": "beijing" }
│   │   ├── evictionTimestamp: 0 (未过期)
│   │   ├── registrationTimestamp: 1714982400000
│   │   └── serviceUpTimestamp: 1714982415000
│   │
│   └── "user-service:192.168.1.11:8080" ──> Lease<InstanceInfo>
│       └── ...
│
├── "ORDER-SERVICE" ──> ConcurrentHashMap
│   ├── "order-service:192.168.1.20:8080" ──> Lease<InstanceInfo>
│   └── "order-service:192.168.1.21:8080" ──> Lease<InstanceInfo>
│
└── "INVENTORY-SERVICE" ──> ConcurrentHashMap
    └── ...

关键设计决策：
1. ConcurrentHashMap：保证高并发读写性能
2. 两层Map结构：第一层按服务名分片，第二层按实例ID索引
3. Lease包装器：将实例信息与租约状态（过期时间）解耦
4. last-dirty-timestamp：解决并发写入冲突，保证最终一致性
```

### 3. 租约（Lease）机制实现

```java
// Lease.java - 租约是Eureka健康检查的核心抽象

public class Lease<T> {
    
    // 注册的实体（InstanceInfo）
    private T holder;
    
    // 租约过期时间（毫秒）
    private long evictionTimestamp;
    
    // 注册时间戳
    private long registrationTimestamp;
    
    // 服务上线时间戳（状态变为UP的时间）
    private long serviceUpTimestamp;
    
    // 最后更新时间戳（续约时更新）
    private volatile long lastUpdateTimestamp;
    
    // 租约持续时间（毫秒）
    private long duration;
    
    /**
     * 续约操作：Client发送心跳时调用
     */
    public void renew() {
        lastUpdateTimestamp = System.currentTimeMillis() + duration;
    }
    
    /**
     * 取消租约：服务主动下线时调用
     */
    public void cancel() {
        if (evictionTimestamp <= 0) {
            evictionTimestamp = System.currentTimeMillis();
        }
    }
    
    /**
     * 判断租约是否已过期
     */
    public boolean isExpired() {
        // 已取消（evictionTimestamp > 0） 或 超过过期时间
        return evictionTimestamp > 0 || 
               System.currentTimeMillis() > lastUpdateTimestamp + duration;
    }
    
    /**
     * 判断租约是否已过期（额外增加补偿时间）
     */
    public boolean isExpired(long additionalLeaseMs) {
        return evictionTimestamp > 0 || 
               System.currentTimeMillis() > (lastUpdateTimestamp + duration + 
                                            additionalLeaseMs);
    }
}
```

**租约过期计算原理**：

```
租约续约时间线：

时间(ms)    0        30000      60000      90000      120000
  │         │          │          │          │          │
  ▼         ▼          ▼          ▼          ▼          ▼
注册      第一次     第二次     第三次     第四次     第五次
          续约       续约       续约       续约       续约
            │          │          │          │          │
            ▼          ▼          ▼          ▼          ▼
        lastUpdateTimestamp = currentTime + duration (90000ms)
        
过期判定：
- 正常情况：每次续约将lastUpdateTimestamp延长90秒
- 异常情况：如果连续3次（90秒）未续约
  System.currentTimeMillis() > lastUpdateTimestamp + duration
  即：当前时间 > 上次续约时间 + 90秒
  租约被标记为过期，等待EvictionTask剔除
```

### 4. 服务剔除（Eviction）机制

```java
// AbstractInstanceRegistry.java -  eviction task

public void evict(long additionalLeaseMs) {
    logger.debug("Running the evict task");
    
    // 1. 检查是否开启自我保护模式
    if (!isLeaseExpirationEnabled()) {
        logger.debug("DS: lease expiration is currently disabled.");
        return; // 自我保护模式下不剔除任何服务
    }
    
    // 2. 收集所有过期的租约
    List<Lease<InstanceInfo>> expiredLeases = new ArrayList<>();
    for (Entry<String, Map<String, Lease<InstanceInfo>>> groupEntry : registry.entrySet()) {
        Map<String, Lease<InstanceInfo>> leaseMap = groupEntry.getValue();
        if (leaseMap != null) {
            for (Entry<String, Lease<InstanceInfo>> leaseEntry : leaseMap.entrySet()) {
                Lease<InstanceInfo> lease = leaseEntry.getValue();
                if (lease.isExpired(additionalLeaseMs) && lease.getHolder() != null) {
                    expiredLeases.add(lease);
                }
            }
        }
    }
    
    // 3. 限制每批次剔除数量（防止一次性剔除过多导致服务雪崩）
    int registrySize = (int) getLocalRegistrySize();
    int registrySizeThreshold = (int) (registrySize * serverConfig.getRenewalPercentThreshold());
    int evictionLimit = registrySize - registrySizeThreshold;
    
    int toEvict = Math.min(expiredLeases.size(), evictionLimit);
    
    // 4. 按过期时间排序，优先剔除最早过期的
    if (toEvict > 0) {
        Collections.sort(expiredLeases, new Comparator<Lease<InstanceInfo>>() {
            @Override
            public int compare(Lease<InstanceInfo> l1, Lease<InstanceInfo> l2) {
                return Long.compare(l1.getEvictionTimestamp(), l2.getEvictionTimestamp());
            }
        });
        
        // 5. 执行剔除
        for (int i = 0; i < toEvict; i++) {
            Lease<InstanceInfo> lease = expiredLeases.get(i);
            String appName = lease.getHolder().getAppName();
            String id = lease.getHolder().getId();
            internalCancel(appName, id, false); // false = 非复制操作
        }
    }
}
```

### 5. ResponseCache：读写分离优化

Eureka Server使用多级缓存机制减少注册表读竞争。

```java
// ResponseCacheImpl.java

public class ResponseCacheImpl implements ResponseCache {
    
    // 只读缓存（Guava Cache，支持TTL自动过期）
    private final LoadingCache<Key, Value> readOnlyCacheMap;
    
    // 读写缓存（ConcurrentHashMap，实时更新）
    private final ConcurrentMap<Key, Value> readWriteCacheMap;
    
    // 缓存更新策略
    private final AbstractQueueWorker<Key> cacheUpdateWorker;
    
    /**
     * 读取缓存（优先读只读缓存，未命中则读读写缓存）
     */
    public String get(final Key key) {
        // 1. 先从只读缓存读取
        Value payload = readOnlyCacheMap.get(key);
        
        if (payload == null || payload.getPayload() == null) {
            // 2. 只读缓存未命中，从读写缓存读取
            payload = readWriteCacheMap.get(key);
            
            if (payload != null && payload.getPayload() != null) {
                // 3. 回填只读缓存
                readOnlyCacheMap.put(key, payload);
            }
        }
        
        return payload == null ? null : payload.getPayload();
    }
    
    /**
     * 使缓存失效（注册/续约/下线时调用）
     */
    public void invalidate(Key... keys) {
        for (Key key : keys) {
            readWriteCacheMap.remove(key);
            // 异步更新只读缓存
            cacheUpdateWorker.add(key);
        }
    }
}
```

**缓存机制架构图**：

```
Client读取注册表时的数据流：

Client ──GET /eureka/apps──> Eureka Server
                                  │
                                  ▼
                        ┌─────────────────┐
                        │ readOnlyCacheMap │ (Guava Cache, TTL=30s)
                        │   只读缓存       │
                        └────────┬────────┘
                                 │ 命中？
                    ┌────────────┴────────────┐
                    │ 是                       │ 否
                    ▼                          ▼
              直接返回                    ┌─────────────────┐
                                          │ readWriteCacheMap│ (ConcurrentHashMap)
                                          │   读写缓存       │
                                          └────────┬────────┘
                                                   │ 命中？
                                       ┌───────────┴───────────┐
                                       │ 是                     │ 否
                                       ▼                        ▼
                                 返回并回填              从Registry读取
                                 只读缓存                并更新两级缓存

缓存一致性保证：
- 写操作（注册/续约/下线）会立即清除readWriteCacheMap
- readOnlyCacheMap通过定时任务（默认30秒）从readWriteCacheMap同步
- 这种设计牺牲强一致性，换取极高的读性能
```

---

## 源码深度分析：Eureka Client核心机制

### 1. DiscoveryClient初始化流程

```java
// DiscoveryClient.java - 客户端核心类

@Singleton
public class DiscoveryClient implements EurekaClient {
    
    // 调度器（用于心跳、缓存刷新）
    private final ScheduledExecutorService scheduler;
    
    // 心跳线程池
    private final ThreadPoolExecutor heartbeatExecutor;
    
    // 缓存刷新线程池
    private final ThreadPoolExecutor cacheRefreshExecutor;
    
    // 本地缓存的注册表
    private final AtomicReference<Applications> localRegionApps;
    
    // Eureka Server HTTP客户端
    private final EurekaHttpClient eurekaHttpClient;
    
    /**
     * 构造方法：初始化所有组件
     */
    @Inject
    DiscoveryClient(ApplicationInfoManager applicationInfoManager, 
                    EurekaClientConfig config,
                    AbstractDiscoveryClientOptionalArgs args,
                    Provider<BackupRegistry> backupRegistryProvider) {
        
        // 1. 初始化调度器
        this.scheduler = Executors.newScheduledThreadPool(
            2, 
            new ThreadFactoryBuilder()
                .setNameFormat("DiscoveryClient-%d")
                .setDaemon(true)
                .build()
        );
        
        // 2. 初始化心跳执行器
        this.heartbeatExecutor = new ThreadPoolExecutor(
            1, config.getHeartbeatExecutorThreadPoolSize(), 0, TimeUnit.SECONDS,
            new SynchronousQueue<>(),
            new ThreadFactoryBuilder().setNameFormat("DiscoveryClient-HeartbeatExecutor-%d").build()
        );
        
        // 3. 初始化缓存刷新执行器
        this.cacheRefreshExecutor = new ThreadPoolExecutor(
            1, config.getCacheRefreshExecutorThreadPoolSize(), 0, TimeUnit.SECONDS,
            new SynchronousQueue<>(),
            new ThreadFactoryBuilder().setNameFormat("DiscoveryClient-CacheRefreshExecutor-%d").build()
        );
        
        // 4. 初始化HTTP客户端（支持多Server负载均衡）
        this.eurekaTransport = new EurekaTransport();
        scheduleServerEndpointTask(eurekaTransport, args);
        
        // 5. 初始化本地缓存
        this.localRegionApps = new AtomicReference<>(new Applications());
        
        // 6. 拉取注册表（全量）
        if (clientConfig.shouldFetchRegistry()) {
            boolean success = fetchRegistry(false);
            if (success) {
                this.localRegionApps.set(this.localRegionApps.get());
            }
        }
        
        // 7. 注册到Server（如果是服务提供方）
        if (clientConfig.shouldRegisterWithEureka()) {
            initScheduledTasks();
        }
    }
    
    /**
     * 初始化定时任务：心跳和缓存刷新
     */
    private void initScheduledTasks() {
        // 1. 心跳定时任务（每30秒）
        if (clientConfig.shouldRegisterWithEureka()) {
            int renewalIntervalInSecs = instanceInfo.getLeaseInfo().getRenewalIntervalInSecs();
            int expBackOffBound = clientConfig.getHeartbeatExecutorExponentialBackOffBound();
            
            scheduler.schedule(
                new TimedSupervisorTask(
                    "heartbeat",
                    scheduler,
                    heartbeatExecutor,
                    renewalIntervalInSecs,
                    TimeUnit.SECONDS,
                    expBackOffBound,
                    new HeartbeatThread() // 心跳线程
                ),
                renewalIntervalInSecs, TimeUnit.SECONDS
            );
        }
        
        // 2. 缓存刷新定时任务（每30秒）
        if (clientConfig.shouldFetchRegistry()) {
            int registryFetchIntervalSeconds = clientConfig.getRegistryFetchIntervalSeconds();
            int expBackOffBound = clientConfig.getCacheRefreshExecutorExponentialBackOffBound();
            
            scheduler.schedule(
                new TimedSupervisorTask(
                    "cacheRefresh",
                    scheduler,
                    cacheRefreshExecutor,
                    registryFetchIntervalSeconds,
                    TimeUnit.SECONDS,
                    expBackOffBound,
                    new CacheRefreshThread() // 缓存刷新线程
                ),
                registryFetchIntervalSeconds, TimeUnit.SECONDS
            );
        }
        
        // 3. 实例信息复制器（向Server同步状态变化）
        instanceInfoReplicator = new InstanceInfoReplicator(
            this,
            instanceInfo,
            clientConfig.getInstanceInfoReplicationIntervalSeconds(),
            2 // burst size
        );
        instanceInfoReplicator.start(clientConfig.getInitialInstanceInfoReplicationIntervalSeconds());
    }
}
```

### 2. 心跳线程（HeartbeatThread）

```java
// DiscoveryClient.java 内部类

private class HeartbeatThread implements Runnable {
    
    public void run() {
        if (renew()) {
            lastSuccessfulHeartbeatTimestamp = System.currentTimeMillis();
        }
    }
}

/**
 * 发送续约（心跳）请求
 */
boolean renew() {
    EurekaHttpResponse<InstanceInfo> httpResponse;
    try {
        // 发送PUT请求：/eureka/apps/{appName}/{instanceId}
        httpResponse = eurekaTransport.registrationClient.sendHeartBeat(
            instanceInfo.getAppName(), 
            instanceInfo.getId(), 
            instanceInfo, 
            null // overriddenStatus
        );
        
        logger.debug("DiscoveryClient_{} - Heartbeat status: {}", 
                    appPathIdentifier, httpResponse.getStatusCode());
        
        // 404表示Server没有该实例，需要重新注册
        if (httpResponse.getStatusCode() == Status.NOT_FOUND.getStatusCode()) {
            logger.info("Re-registering apps/{}" , instanceInfo.getAppName());
            long timestamp = instanceInfo.setIsDirtyWithTime();
            boolean success = register();
            if (success) {
                instanceInfo.unsetIsDirty(timestamp);
            }
            return success;
        }
        
        return httpResponse.getStatusCode() == Status.OK.getStatusCode();
        
    } catch (Throwable e) {
        logger.error("DiscoveryClient_{} - was unable to send heartbeat!", appPathIdentifier, e);
        return false;
    }
}
```

### 3. 缓存刷新线程（CacheRefreshThread）

```java
// DiscoveryClient.java 内部类

private class CacheRefreshThread implements Runnable {
    public void run() {
        refreshRegistry();
    }
}

@VisibleForTesting
void refreshRegistry() {
    try {
        boolean isFetchingRemoteRegionRegistries = isFetchingRemoteRegionRegistries();
        
        boolean remoteRegionModified = false;
        if (isFetchingRemoteRegionRegistries) {
            // 如果远程区域配置变更，标记需要全量拉取
            remoteRegionModified = checkRemoteRegions();
        }
        
        // 决定是全量拉取还是增量拉取
        boolean success = fetchRegistry(remoteRegionModified);
        if (success) {
            registrySize = localRegionApps.get().size();
            lastSuccessfulRegistryFetchTimestamp = System.currentTimeMillis();
        }
        
    } catch (Throwable e) {
        logger.error("Cannot fetch registry from server", e);
    }
}

/**
 * 拉取注册表（支持增量和全量）
 */
private boolean fetchRegistry(boolean forceFullRegistryFetch) {
    Stopwatch tracer = FETCH_REGISTRY_TIMER.start();
    
    try {
        Applications applications = getApplications();
        
        if (clientConfig.shouldDisableDelta() 
            || forceFullRegistryFetch
            || (applications == null)
            || (applications.getRegisteredApplications().size() == 0)
            || (applications.getVersion() == -1)) {
            
            // 全量拉取
            logger.info("Disable delta property : {}", clientConfig.shouldDisableDelta());
            logger.info("Single vip registry refresh property : {}", clientConfig.getRegistryRefreshSingleVipAddress());
            logger.info("Force full registry fetch : {}", forceFullRegistryFetch);
            logger.info("Application is null : {}", (applications == null));
            logger.info("Registered Applications size is zero : {}", 
                       (applications.getRegisteredApplications().size() == 0));
            logger.info("Application version is -1: {}", (applications.getVersion() == -1));
            
            getAndStoreFullRegistry();
            
        } else {
            // 增量拉取
            getAndUpdateDelta(applications);
        }
        
        // 重新计算应用哈希（用于和Server校验一致性）
        applications.setAppsHashCode(applications.getReconcileHashCode());
        
        // 记录拉取成功
        logTotalInstances();
        
    } catch (Throwable e) {
        logger.error("DiscoveryClient_{} - was unable to refresh its cache! status = {}",
                    appPathIdentifier, e.getMessage(), e);
        return false;
    } finally {
        tracer.stop();
    }
    
    return true;
}
```

### 4. Eureka HTTP客户端架构

```java
// EurekaHttpClient 装饰器链

/**
 * Eureka HTTP客户端采用装饰器模式，构建了一个处理链：
 * 
 * SessionedEurekaHttpClient
 *   └── RetryableEurekaHttpClient
 *         └── RedirectingEurekaHttpClient
 *               └── MetricsCollectingEurekaHttpClient
 *                     └── JerseyApplicationClient (实际HTTP调用)
 * 
 * 各层职责：
 * 1. SessionedEurekaHttpClient：维护与单个Server的会话
 * 2. RetryableEurekaHttpClient：失败重试，切换到下一个Server
 * 3. RedirectingEurekaHttpClient：处理Server返回的302重定向
 * 4. MetricsCollectingEurekaHttpClient：收集调用指标
 * 5. JerseyApplicationClient：使用Jersey发送HTTP请求
 */

// 装饰器链构建代码
EurekaHttpClient build() {
    EurekaHttpClient delegate = new JerseyApplicationClient(
        jerseyClient, serviceUrl, false
    );
    
    delegate = new MetricsCollectingEurekaHttpClient(
        delegate, transportConfig
    );
    
    delegate = new RedirectingEurekaHttpClient(
        serviceUrl, delegate, transportConfig
    );
    
    delegate = new RetryableEurekaHttpClient(
        serviceUrl, false, delegate, transportConfig
    );
    
    return new SessionedEurekaHttpClient(
        serviceUrl, delegate, transportConfig
    );
}
```

---

## 源码深度分析：Peer复制与自我保护

### 1. 对等复制（Peer Replication）机制

```java
// PeerAwareInstanceRegistryImpl.java

/**
 * 复制实例信息到其他Eureka Server节点
 */
public boolean replicateToPeers(Action action, String appName, String id,
                                InstanceInfo info, InstanceStatus newStatus, 
                                boolean isReplication) {
    
    // 如果当前操作已经是复制操作（来自其他节点），不再转发
    if (isReplication) {
        numberOfReplicationsLastMin.increment();
    }
    
    // 如果只有一个节点，无需复制
    if (peerEurekaNodes == Collections.emptyList()) {
        return false;
    }
    
    // 向所有对等节点发送复制请求
    for (PeerEurekaNode node : peerEurekaNodes.getPeerEurekaNodes()) {
        // 如果是当前节点，跳过
        if (peerEurekaNodes.isThisMyUrl(node.getServiceUrl())) {
            continue;
        }
        
        // 异步复制
        replicateInstanceActionsToPeers(action, appName, id, info, newStatus, node);
    }
    
    return true;
}

/**
 * 具体的复制操作
 */
private void replicateInstanceActionsToPeers(Action action, String appName, 
                                              String id, InstanceInfo info,
                                              InstanceStatus newStatus, 
                                              PeerEurekaNode node) {
    try {
        switch (action) {
            case Cancel:
                node.cancel(appName, id);
                break;
            case Heartbeat:
                node.heartbeat(appName, id, info, newStatus, false);
                break;
            case Register:
                node.register(info);
                break;
            case StatusUpdate:
                node.statusUpdate(appName, id, newStatus);
                break;
            case DeleteStatusOverride:
                node.deleteStatusOverride(appName, id);
                break;
        }
    } catch (Throwable t) {
        logger.error("Cannot replicate information to {} for action {}", 
                    node.getServiceUrl(), action.name(), t);
    }
}
```

### 2. 自我保护机制（Self Preservation）

自我保护是Eureka最著名的特性，用于防止网络分区时的服务雪崩。

```java
// AbstractInstanceRegistry.java

/**
 * 计算预期每分钟续约次数
 */
protected void updateRenewsPerMinThreshold() {
    // 预期续约数 = 注册实例数 * (60 / 续约间隔) * 保护阈值
    this.numberOfRenewsPerMinThreshold = (int) (
        this.expectedNumberOfClientsSendingRenews 
        * (60.0 / serverConfig.getExpectedClientRenewalIntervalSeconds())
        * serverConfig.getRenewalPercentThreshold()
    );
}

/**
 * 判断是否启用租约过期（即：是否关闭自我保护）
 */
public boolean isLeaseExpirationEnabled() {
    // 如果配置了关闭自我保护，直接返回true
    if (!isSelfPreservationModeEnabled()) {
        return true;
    }
    
    // 如果最近一分钟实际续约数 < 预期续约数，进入自我保护模式
    // 自我保护模式下返回false，禁止剔除任何服务
    return numberOfRenewsInLastMin > numberOfRenewsPerMinThreshold;
}
```

**自我保护机制图解**：

```
自我保护机制触发流程：

                      ┌─────────────────┐
                      │  每分钟统计续约数  │
                      │ numberOfRenews   │
                      │   inLastMin      │
                      └────────┬────────┘
                               │
                               ▼
                      ┌─────────────────┐
                      │ 实际续约 < 预期？ │
                      │ renews < threshold│
                      └────────┬────────┘
                               │
                    ┌──────────┴──────────┐
                    │ 是                   │ 否
                    ▼                      ▼
           ┌─────────────┐         ┌─────────────┐
           │ 进入自我保护 │         │ 正常模式     │
           │ Self Pres.  │         │ Normal      │
           └──────┬──────┘         └──────┬──────┘
                  │                      │
                  ▼                      ▼
           ┌─────────────┐         ┌─────────────┐
           │ isLeaseExp. │         │ isLeaseExp. │
           │ 返回 false   │         │ 返回 true   │
           │ (不剔除服务) │         │ (正常剔除)  │
           └─────────────┘         └─────────────┘

数学公式：
预期续约数 = 注册实例数 × (60 / 续约间隔秒数) × 阈值百分比
           = N × 2 × 0.85
           = 1.7N (默认配置下)

触发条件：15分钟内心跳失败比例 > 85%
即：实际续约数 < 预期续约数 × 0.85
```

**自我保护的意义**：

```
场景：网络分区（Network Partition）

        网络分区线
   ┌──────────┬──────────┐
   │          │          │
   │ Zone A   │  Zone B  │
   │          │          │
   │ ┌──────┐ │ ┌──────┐ │
   │ │Server│ │ │Server│ │
   │ └──┬───┘ │ └──┬───┘ │
   │    │     │    │     │
   │ 心跳丢失  │  心跳丢失 │
   │    │     │    │     │
   │ ┌──┴──┐ │ ┌──┴──┐ │
   │ │Client│ │ │Client│ │
   │ └──┬──┘ │ └──┬──┘ │
   │    │     │    │     │
   └────┼─────┴────┼─────┘
        │          │
     服务正常    服务正常
     但心跳不通   但心跳不通

无自我保护时的后果：
- Server认为Client已死，大量剔除服务
- Client本地缓存过期后，无法发现其他服务
- 级联故障导致整个集群瘫痪

自我保护的保护行为：
- 停止剔除任何服务（即使心跳超时）
- 仍然接受新服务注册和查询
- 网络恢复后自动退出自我保护
- 保留过期服务列表总比没有服务好（AP哲学的极致体现）
```

---

## 实战案例：生产级Eureka集群部署

### 案例1：三节点Eureka集群（高可用）

```yaml
# eureka-server-1.yml
server:
  port: 8761

spring:
  application:
    name: eureka-server
  profiles:
    active: peer1

eureka:
  instance:
    hostname: eureka1
    prefer-ip-address: false
    instance-id: ${spring.application.name}:${eureka.instance.hostname}:${server.port}
  client:
    register-with-eureka: true
    fetch-registry: true
    service-url:
      defaultZone: http://eureka2:8762/eureka/,http://eureka3:8763/eureka/
  server:
    enable-self-preservation: true
    renewal-percent-threshold: 0.85
    eviction-interval-timer-in-ms: 60000
    peer-node-read-timeout-ms: 5000
    peer-node-connect-timeout-ms: 2000

---
# eureka-server-2.yml
server:
  port: 8762

spring:
  application:
    name: eureka-server
  profiles:
    active: peer2

eureka:
  instance:
    hostname: eureka2
    prefer-ip-address: false
    instance-id: ${spring.application.name}:${eureka.instance.hostname}:${server.port}
  client:
    register-with-eureka: true
    fetch-registry: true
    service-url:
      defaultZone: http://eureka1:8761/eureka/,http://eureka3:8763/eureka/
  server:
    enable-self-preservation: true
    renewal-percent-threshold: 0.85

---
# eureka-server-3.yml
server:
  port: 8763

spring:
  application:
    name: eureka-server
  profiles:
    active: peer3

eureka:
  instance:
    hostname: eureka3
    prefer-ip-address: false
    instance-id: ${spring.application.name}:${eureka.instance.hostname}:${server.port}
  client:
    register-with-eureka: true
    fetch-registry: true
    service-url:
      defaultZone: http://eureka1:8761/eureka/,http://eureka2:8762/eureka/
  server:
    enable-self-preservation: true
    renewal-percent-threshold: 0.85
```

```java
// 启动类
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}

// 配置类：启用Spring Security保护Eureka端点
@Configuration
@EnableWebSecurity
public class EurekaSecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.csrf().disable()
            .authorizeRequests()
            .anyRequest().authenticated()
            .and()
            .httpBasic();
        return http.build();
    }
    
    @Bean
    public InMemoryUserDetailsManager userDetailsManager() {
        UserDetails user = User.builder()
            .username("eureka")
            .password(passwordEncoder().encode("eureka_password"))
            .roles("EUREKA")
            .build();
        return new InMemoryUserDetailsManager(user);
    }
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

```dockerfile
# Dockerfile for Eureka Server
FROM eclipse-temurin:17-jdk-alpine
VOLUME /tmp
COPY target/eureka-server-*.jar app.jar
ENTRYPOINT ["java", "-Djava.security.egd=file:/dev/./urandom", "-jar", "/app.jar"]
```

```yaml
# docker-compose.yml for 3-node cluster
version: '3.8'

services:
  eureka1:
    build: ./eureka-server
    container_name: eureka1
    hostname: eureka1
    ports:
      - "8761:8761"
    environment:
      - SPRING_PROFILES_ACTIVE=peer1
      - EUREKA_INSTANCE_HOSTNAME=eureka1
    networks:
      - eureka-net

  eureka2:
    build: ./eureka-server
    container_name: eureka2
    hostname: eureka2
    ports:
      - "8762:8762"
    environment:
      - SPRING_PROFILES_ACTIVE=peer2
      - EUREKA_INSTANCE_HOSTNAME=eureka2
    networks:
      - eureka-net

  eureka3:
    build: ./eureka-server
    container_name: eureka3
    hostname: eureka3
    ports:
      - "8763:8763"
    environment:
      - SPRING_PROFILES_ACTIVE=peer3
      - EUREKA_INSTANCE_HOSTNAME=eureka3
    networks:
      - eureka-net

networks:
  eureka-net:
    driver: bridge
```

### 案例2：服务提供方配置（生产级）

```yaml
# application.yml for Service Provider
spring:
  application:
    name: order-service
  cloud:
    inetutils:
      preferred-networks:
        - 192.168.1
      ignored-interfaces:
        - docker0
        - veth.*

eureka:
  client:
    service-url:
      defaultZone: http://eureka1:8761/eureka/,http://eureka2:8762/eureka/,http://eureka3:8763/eureka/
    register-with-eureka: true
    fetch-registry: true
    registry-fetch-interval-seconds: 30
    instance-info-replication-interval-seconds: 30
    initial-instance-info-replication-interval-seconds: 40
    eureka-service-url-poll-interval-seconds: 300
    healthcheck:
      enabled: true
  instance:
    prefer-ip-address: true
    ip-address: ${spring.cloud.client.ip-address}
    instance-id: ${spring.application.name}:${spring.cloud.client.ip-address}:${server.port}
    lease-renewal-interval-in-seconds: 30
    lease-expiration-duration-in-seconds: 90
    status-page-url-path: /actuator/info
    health-check-url-path: /actuator/health
    metadata-map:
      version: v1.2.0
      region: beijing
      zone: az-1
      team: order-team
      description: Order Service v1.2.0
      profile: ${spring.profiles.active:default}
```

```java
// 服务提供方：优雅下线处理
@Component
@Slf4j
public class EurekaGracefulShutdown implements ApplicationListener<ContextClosedEvent> {
    
    @Autowired
    private EurekaClient eurekaClient;
    
    @Autowired
    private DiscoveryClient discoveryClient;
    
    @Override
    public void onApplicationEvent(ContextClosedEvent event) {
        log.info("Application is shutting down, initiating graceful shutdown...");
        
        // 1. 将服务状态标记为DOWN（停止接收新流量）
        ApplicationInfoManager.getInstance().setInstanceStatus(InstanceInfo.InstanceStatus.DOWN);
        
        // 2. 等待现有请求处理完成（给负载均衡器时间刷新缓存）
        try {
            log.info("Waiting for 30 seconds to allow load balancers to refresh...");
            Thread.sleep(30000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        
        // 3. 向Eureka Server发送下线通知
        if (eurekaClient != null) {
            eurekaClient.shutdown();
            log.info("Eureka client shutdown completed.");
        }
        
        // 4. 关闭线程池和连接池
        log.info("Graceful shutdown completed.");
    }
}
```

### 案例3：Kubernetes环境下的Eureka部署

```yaml
# eureka-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: eureka-server
  namespace: microservices
spec:
  serviceName: eureka-headless
  replicas: 3
  selector:
    matchLabels:
      app: eureka-server
  template:
    metadata:
      labels:
        app: eureka-server
    spec:
      containers:
        - name: eureka
          image: registry.internal/eureka-server:1.0.0
          ports:
            - containerPort: 8761
              name: http
          env:
            - name: EUREKA_INSTANCE_HOSTNAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: EUREKA_CLIENT_SERVICE_URL_DEFAULTZONE
              value: "http://eureka-server-0.eureka-headless:8761/eureka/,http://eureka-server-1.eureka-headless:8761/eureka/,http://eureka-server-2.eureka-headless:8761/eureka/"
            - name: EUREKA_INSTANCE_PREFER_IP_ADDRESS
              value: "false"
          livenessProbe:
            httpGet:
              path: /actuator/health
              port: 8761
            initialDelaySeconds: 60
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /actuator/health
              port: 8761
            initialDelaySeconds: 30
            periodSeconds: 5
          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "1000m"
---
apiVersion: v1
kind: Service
metadata:
  name: eureka-headless
  namespace: microservices
spec:
  clusterIP: None
  selector:
    app: eureka-server
  ports:
    - port: 8761
      targetPort: 8761
      name: http
---
apiVersion: v1
kind: Service
metadata:
  name: eureka
  namespace: microservices
spec:
  type: LoadBalancer
  selector:
    app: eureka-server
  ports:
    - port: 8761
      targetPort: 8761
      name: http
```

### 案例4：自定义健康检查与元数据

```java
// 自定义健康检查指示器
@Component
public class DatabaseHealthIndicator implements HealthIndicator {
    
    @Autowired
    private DataSource dataSource;
    
    @Override
    public Health health() {
        try (Connection conn = dataSource.getConnection()) {
            if (conn.isValid(5)) {
                return Health.up()
                    .withDetail("database", "MySQL")
                    .withDetail("validationQuery", "SELECT 1")
                    .build();
            }
        } catch (SQLException e) {
            return Health.down()
                .withDetail("error", e.getMessage())
                .build();
        }
        return Health.down().build();
    }
}

// Eureka事件监听：服务注册/下线通知
@Component
@Slf4j
public class EurekaEventListener {
    
    @EventListener
    public void onInstanceRegistered(EurekaInstanceRegisteredEvent event) {
        InstanceInfo instanceInfo = event.getInstanceInfo();
        log.info("Instance registered: {} - {}:{}", 
                instanceInfo.getAppName(),
                instanceInfo.getIPAddr(),
                instanceInfo.getPort());
        
        // 可以发送企业微信/钉钉通知
        sendNotification("注册", instanceInfo);
    }
    
    @EventListener
    public void onInstanceCancelled(EurekaInstanceCanceledEvent event) {
        log.info("Instance cancelled: {} - {}", 
                event.getAppName(), 
                event.getServerId());
        
        sendNotification("下线", event.getAppName(), event.getServerId());
    }
    
    @EventListener
    public void onInstanceRenewed(EurekaInstanceRenewedEvent event) {
        // 心跳续约事件（通常不处理，避免日志过多）
    }
    
    private void sendNotification(String action, Object... args) {
        // 实现通知逻辑
    }
}
```

---

## 对比分析：Eureka vs Consul vs Nacos vs ZooKeeper

### 核心特性对比

```
┌──────────────────┬─────────────┬─────────────┬─────────────┬─────────────┐
│     特性         │   Eureka    │   Consul    │    Nacos    │ ZooKeeper   │
├──────────────────┼─────────────┼─────────────┼─────────────┼─────────────┤
│ 开发方           │  Netflix    │HashiCorp    │   阿里      │   Apache    │
├──────────────────┼─────────────┼─────────────┼─────────────┼─────────────┤
│ CAP理论          │     AP      │    CP/AP   │    AP/CP    │     CP      │
│ 一致性模型       │ 最终一致    │ 强一致/可调  │ 最终/强一致  │   强一致    │
├──────────────────┼─────────────┼─────────────┼─────────────┼─────────────┤
│ 健康检查         │Client心跳   │Client+Server│Client+Server│ TCP心跳     │
│                 │(被动)        │(主动+被动)   │(主动+被动)   │(会话保持)   │
├──────────────────┼─────────────┼─────────────┼─────────────┼─────────────┤
│ 多数据中心       │   支持      │   原生支持   │    支持     │    支持     │
│ Multi-Region     │(通过配置)    │(WAN gossip)  │(通过集群)    │(Observer)   │
├──────────────────┼─────────────┼─────────────┼─────────────┼─────────────┤
│ 配置中心         │   不支持    │  Key/Value  │   原生支持   │    支持     │
│                 │             │  (简单支持)  │(动态配置)    │(通过Curator)│
├──────────────────┼─────────────┼─────────────┼─────────────┼─────────────┤
│ 负载均衡         │  Ribbon     │  内置LB     │  内置LB     │  需配合     │
│                 │(客户端)      │(服务端+客户端)│(多种策略)    │             │
├──────────────────┼─────────────┼─────────────┼─────────────┼─────────────┤
│ 服务网格         │   不支持    │   支持      │   不支持    │    支持     │
│ Service Mesh     │             │(Consul Connect)│         │(Istio集成)  │
├──────────────────┼─────────────┼─────────────┼─────────────┼─────────────┤
│ 控制台           │   简单      │   丰富      │   非常丰富   │    无       │
│ UI               │(仅注册表)     │(服务/健康/KV)│(服务/配置/流量)│(需第三方)   │
├──────────────────┼─────────────┼─────────────┼─────────────┼─────────────┤
│ 社区活跃度       │  维护模式   │   活跃      │   非常活跃   │    活跃     │
│                 │(Netflix已停更)│            │            │             │
├──────────────────┼─────────────┼─────────────┼─────────────┼─────────────┤
│ Spring Cloud集成 │   原生      │   支持      │   原生支持   │   需配置    │
│                 │(spring-cloud-│(spring-cloud-│(spring-cloud-│(spring-cloud-│
│                 │ netflix)    │ consul)     │ alibaba)    │ starter-zk) │
├──────────────────┼─────────────┼─────────────┼─────────────┼─────────────┤
│ 学习曲线         │   低        │   中        │    低       │     高      │
├──────────────────┼─────────────┼─────────────┼─────────────┼─────────────┤
│ 适用场景         │存量Spring   │多语言/服务  │Spring Cloud │大数据/      │
│                 │Cloud维护    │网格/安全    │新/存量项目   │分布式协调   │
└──────────────────┴─────────────┴─────────────┴─────────────┴─────────────┘
```

### 架构模式对比

```
Eureka架构（客户端发现 + AP）：

Client ──发现──> Eureka Server <──注册── Service
  │                   │
  │                   │ (Peer Replication)
  │                   ▼
  │              Eureka Server
  │                   │
  └──────直接调用─────┘
   (本地缓存服务列表)

特点：
- 客户端持有服务列表缓存
- 网络分区时仍可调用（基于本地缓存）
- 不保证强一致性

Consul架构（服务端发现 + CP/AP）：

Client ──请求──> Consul Agent ──转发──> Service
                    │
                    ▼
              Consul Server (Raft)
                    │
              Consul Server
                    │
              Consul Server

特点：
- 本地Agent代理所有请求
- Server使用Raft保证强一致
- 支持服务端健康检查

Nacos架构（灵活模式）：

Client ──请求──> Nacos Server <──注册── Service
                    │ (Distro/CP协议)
                    ▼
              Nacos Server
                    │
              Nacos Server

特点：
- 默认AP（Distro协议）
- 可切换CP（Raft协议）
- 内置配置中心
- 支持权重和灰度

ZooKeeper架构（强一致协调）：

Client ──监听──> ZK Server (Leader) <──注册── Service
                    │
                    ▼ (ZAB协议)
              ZK Server (Follower)
                    │
              ZK Server (Follower)

特点：
- 强一致性（ZAB协议）
- 临时节点绑定会话
- 会话过期自动删除
- 不适合作为服务注册中心（设计用于协调）
```

### 选型建议

```
选型决策树：

是否需要配置中心？
├── 是 ──> 优先考虑 Nacos（注册+配置一体化）
└── 否 ──> 继续判断

是否使用Spring Cloud？
├── 是 ──> 是否存量Netflix系统？
│          ├── 是 ──> 继续使用 Eureka（维护模式）
│          └── 否 ──> 新项目推荐 Nacos 或 Consul
└── 否 ──> 多语言环境？
           ├── 是 ──> Consul（多语言SDK完善）
           └── 否 ──> Kubernetes环境？
                      ├── 是 ──> 使用K8s DNS/Endpoint（云原生）
                      └── 否 ──> Consul 或 etcd

性能考量：
- 超大规模（>10万实例）：Nacos（水平扩展能力强）
- 高可用优先：Eureka AP模式 或 Nacos AP模式
- 强一致优先：Consul CP模式 或 ZooKeeper
- 低延迟优先：Eureka（本地缓存设计）
```

---

## 性能分析：注册中心的瓶颈与优化

### 1. 性能基准测试

```
Eureka性能基准（典型配置）：

测试环境：
- Server: 3节点，8C16G
- Client: 100个服务，每服务5实例 = 500实例
- 网络: 内网千兆

指标：
┌────────────────────────┬──────────────┬──────────────┐
│        指标            │    Eureka    │     Nacos    │
├────────────────────────┼──────────────┼──────────────┤
│ 注册吞吐量 (TPS)       │    ~500      │   ~1000+     │
│ 查询吞吐量 (QPS)       │   ~5000      │   ~8000+     │
│ 平均注册延迟 (ms)      │    ~20       │    ~10       │
│ 平均查询延迟 (ms)      │    ~5        │    ~3        │
│ 内存占用 (MB/500实例)  │   ~200       │   ~300       │
│ CPU使用率 (%)          │    ~15       │    ~20       │
│ 网络带宽 (KB/s)        │    ~50       │    ~80       │
└────────────────────────┴──────────────┴──────────────┘

注：Eureka查询性能高主要得益于ResponseCache的两级缓存设计
```

### 2. JVM与GC优化

```bash
# Eureka Server生产级JVM参数
JAVA_OPTS="
  -server
  -Xms2g -Xmx2g
  -XX:+UseG1GC
  -XX:MaxGCPauseMillis=200
  -XX:InitiatingHeapOccupancyPercent=45
  -XX:+ParallelRefProcEnabled
  -XX:+AlwaysPreTouch
  -XX:+DisableExplicitGC
  -XX:+HeapDumpOnOutOfMemoryError
  -XX:HeapDumpPath=/logs/heapdump.hprof
  -Xlog:gc*:file=/logs/gc.log:time,uptime:filecount=5,filesize=100m
  -Djava.security.egd=file:/dev/./urandom
"

# 容器环境（Docker/K8s）
JAVA_OPTS="
  -server
  -XX:+UseContainerSupport
  -XX:MaxRAMPercentage=75.0
  -XX:InitialRAMPercentage=50.0
  -XX:+UseG1GC
  -XX:MaxGCPauseMillis=200
"
```

### 3. 网络优化

```yaml
# 优化Eureka Server的网络配置
eureka:
  server:
    # 减少复制批处理大小，降低延迟
    max-elements-in-peer-replication-pool: 10000
    max-elements-in-status-replication-pool: 10000
    
    # 缩短复制间隔（默认500ms，改为100ms）
    peer-node-read-timeout-ms: 5000
    peer-node-connect-timeout-ms: 2000
    peer-node-total-connections: 1000
    peer-node-total-connections-per-host: 500
    
    # 开启压缩（跨可用区部署时尤为重要）
    enable-replicated-request-compression: true
    
  instance:
    # 使用IP减少DNS解析开销
    prefer-ip-address: true
```

### 4. 内存与注册表优化

```java
// 自定义注册表实现（极端性能场景）
@Component
public class OptimizedInstanceRegistry extends PeerAwareInstanceRegistryImpl {
    
    @Override
    public void register(InstanceInfo info, boolean isReplication) {
        // 优化1：批量注册时使用批量锁
        // 优化2：大注册表时使用分片ConcurrentHashMap
        // 优化3：减少recentRegisteredQueue的大小
        super.register(info, isReplication);
    }
    
    @Override
    protected void invalidateCache(String appName, String vipAddress, String secureVipAddress) {
        // 优化：异步批量失效缓存，减少锁竞争
        cacheInvalidationQueue.offer(new CacheKey(appName, vipAddress, secureVipAddress));
    }
}
```

---

## 常见陷阱与最佳实践

### 陷阱1：Eureka Server单点部署

**问题描述**：只部署一个Eureka Server，一旦宕机整个微服务集群的服务发现能力丧失，新服务无法注册，消费方无法更新服务列表。

**根本原因**：忽视了Eureka Server作为基础设施组件的高可用性要求。

**最佳实践**：
```yaml
# 生产环境必须部署至少3个Eureka Server节点
eureka:
  client:
    service-url:
      defaultZone: 
        http://eureka1:8761/eureka/,
        http://eureka2:8762/eureka/,
        http://eureka3:8763/eureka/
```

```
部署拓扑：

可用区A              可用区B              可用区C
┌─────────┐         ┌─────────┐         ┌─────────┐
│Eureka S1│◄───────►│Eureka S2│◄───────►│Eureka S3│
│  8761   │  复制    │  8762   │  复制    │  8763   │
└────┬────┘         └────┬────┘         └────┬────┘
     │                   │                   │
     └───────────────────┴───────────────────┘
                    互相注册
                    对等复制
```

### 陷阱2：服务使用hostname注册导致访问失败

**问题描述**：服务注册的是机器名（如`docker-desktop`或`MacBook-Pro.local`），其他服务无法解析该hostname，导致调用超时。

**根本原因**：Eureka默认使用`eureka.instance.hostname`注册，容器/K8s环境中hostname可能不可解析。

**最佳实践**：
```yaml
eureka:
  instance:
    prefer-ip-address: true  # 强制使用IP地址注册
    ip-address: ${spring.cloud.client.ip-address}
    instance-id: ${spring.application.name}:${spring.cloud.client.ip-address}:${server.port}
    
  client:
    # 忽略Docker虚拟网卡
    healthcheck:
      enabled: true
```

```java
// 容器环境：动态获取容器IP
@Configuration
public class EurekaInstanceConfig {
    
    @Bean
    public EurekaInstanceConfigBean eurekaInstanceConfigBean(InetUtils inetUtils) {
        EurekaInstanceConfigBean config = new EurekaInstanceConfigBean(inetUtils);
        
        // Kubernetes环境下获取Pod IP
        String podIp = System.getenv("POD_IP");
        if (StringUtils.hasText(podIp)) {
            config.setIpAddress(podIp);
            config.setPreferIpAddress(true);
        }
        
        return config;
    }
}
```

### 陷阱3：关闭自我保护后网络抖动导致服务雪崩

**问题描述**：测试环境关闭自我保护（`enable-self-preservation: false`），网络抖动时Server大量剔除服务，消费方本地缓存未及时更新，导致大量调用失败。

**根本原因**：自我保护是Eureka应对网络分区的核心机制，盲目关闭会丧失这一保护能力。

**最佳实践**：
```yaml
eureka:
  server:
    # 生产环境必须开启自我保护
    enable-self-preservation: true
    # 阈值可以适当调整（默认0.85）
    renewal-percent-threshold: 0.85
    
  instance:
    # 合理配置心跳间隔和失效时间
    lease-renewal-interval-in-seconds: 30
    lease-expiration-duration-in-seconds: 90
```

```
不同环境配置建议：

开发环境：
  enable-self-preservation: false（方便调试，快速剔除已停止的服务）
  eviction-interval-timer-in-ms: 5000（5秒检查一次）

测试环境：
  enable-self-preservation: true
  renewal-percent-threshold: 0.5（阈值放低，更敏感）

生产环境：
  enable-self-preservation: true
  renewal-percent-threshold: 0.85（默认值）
  eviction-interval-timer-in-ms: 60000（1分钟检查一次）
```

### 陷阱4：服务下线未通知Eureka

**问题描述**：直接`kill -9`终止服务进程，Eureka需等待90秒（lease-expiration-duration）才剔除，期间调用方仍可能路由到已下线实例，导致调用失败。

**根本原因**：未实现优雅下线（Graceful Shutdown），Eureka Server不知道服务已经停止。

**最佳实践**：
```java
@Component
public class GracefulShutdownListener implements ApplicationListener<ContextClosedEvent> {
    
    @Autowired
    private EurekaClient eurekaClient;
    
    @Override
    public void onApplicationEvent(ContextClosedEvent event) {
        // 1. 发送下线通知
        eurekaClient.shutdown();
        
        // 2. 等待负载均衡器刷新（Eureka Client缓存30秒刷新一次）
        try {
            Thread.sleep(35000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

```yaml
# Kubernetes preStop钩子
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 30"]
      # 给Spring Boot的shutdown hook执行时间
```

### 陷阱5：Eureka Client未开启服务发现

**问题描述**：服务能注册到Eureka但无法调用其他服务，报错`No instances available for user-service`。

**根本原因**：`fetch-registry`配置为`false`，Client没有拉取注册表。

**最佳实践**：
```yaml
eureka:
  client:
    register-with-eureka: true   # 注册自己（提供方必须）
    fetch-registry: true         # 获取注册表（消费方必须）
    registry-fetch-interval-seconds: 30
```

### 陷阱6：Server节点间时钟不同步

**问题描述**：Eureka Server集群节点间系统时间不一致，导致last-dirty-timestamp比较错误，新注册的服务被旧数据覆盖。

**根本原因**：Eureka使用系统时间戳解决写入冲突，时钟偏移会导致冲突解决错误。

**最佳实践**：
```bash
# 所有Eureka Server节点必须配置NTP时间同步
# Linux
sudo timedatectl set-ntp true

# 或者使用chrony
sudo apt-get install chrony
sudo systemctl enable chrony
sudo systemctl start chrony
```

### 陷阱7：注册表过大导致内存溢出

**问题描述**：微服务实例数过万时，Eureka Server内存占用持续上升，最终OOM。

**根本原因**：每个实例的InstanceInfo、Lease对象、最近修改队列等都会占用内存。

**最佳实践**：
```yaml
# 减少不必要的元数据
eureka:
  instance:
    metadata-map:
      # 只保留关键元数据
      version: ${app.version}
      region: ${app.region}
      # 避免存储大量无用信息
```

```bash
# 增加Server内存
-Xms4g -Xmx4g

# 或者考虑分集群部署
# 按业务域拆分多个Eureka集群
```

### 陷阱8：使用默认的Jersey客户端在高并发下性能不足

**问题描述**：Eureka Client默认使用Jersey 1.x作为HTTP客户端，在高并发场景下连接池耗尽，导致注册/续约失败。

**根本原因**：Jersey 1.x的连接池配置不够灵活，默认连接数较少。

**最佳实践**：
```java
// 自定义EurekaHttpClient（使用Apache HttpClient）
@Configuration
public class EurekaHttpClientConfig {
    
    @Bean
    public EurekaHttpClient eurekaHttpClient() {
        // 使用更高效的HTTP客户端替代Jersey
        // 或者调整Jersey连接池参数
        ClientConfig clientConfig = new DefaultClientConfig();
        clientConfig.getProperties().put(
            DefaultApacheHttpClient4Config.PROPERTY_CONNECTION_MANAGER,
            new PoolingClientConnectionManager()
        );
        return new JerseyApplicationClient(
            Client.create(clientConfig), 
            serviceUrl, 
            false
        );
    }
}
```

### 陷阱9：忽略Eureka的指标监控

**问题描述**：Eureka Server运行异常（如大量服务被剔除、自我保护频繁触发）时没有及时发现，导致故障扩大。

**最佳实践**：
```java
// 启用Actuator指标
@Configuration
public class EurekaMetricsConfig {
    
    @Bean
    public MeterRegistryCustomizer<MeterRegistry> eurekaMetrics() {
        return registry -> {
            // 注册自我保护状态指标
            Gauge.builder("eureka.self.preservation.enabled", 
                         () -> isSelfPreservationEnabled() ? 1 : 0)
                 .register(registry);
            
            // 注册注册表大小指标
            Gauge.builder("eureka.registry.size", 
                         () -> getLocalRegistrySize())
                 .register(registry);
            
            // 注册续约率指标
            Gauge.builder("eureka.renews.in.last.minute", 
                         () -> getNumberOfRenewsInLastMin())
                 .register(registry);
        };
    }
}
```

```yaml
# Prometheus监控配置
management:
  endpoints:
    web:
      exposure:
        include: prometheus,health,info,metrics
  metrics:
    export:
      prometheus:
        enabled: true
```

### 陷阱10：Client配置不当导致频繁全量拉取

**问题描述**：Eureka Client频繁全量拉取注册表（而非增量），导致Server负载增高、网络带宽占用大。

**根本原因**：`shouldDisableDelta`配置为true，或者增量同步时校验失败。

**最佳实践**：
```yaml
eureka:
  client:
    # 必须启用增量拉取
    disable-delta: false
    
    # 增加缓存刷新间隔（如果服务列表变化不大）
    registry-fetch-interval-seconds: 30
```

---

## 面试题与参考答案

### Q1：Eureka的架构包含哪些核心角色？它们之间如何通信？

**参考答案**：

Eureka架构包含两个核心角色：

1. **Eureka Server（注册中心）**：
   - 维护服务注册表（`ConcurrentHashMap<String, Map<String, Lease<InstanceInfo>>>`）
   - 处理服务注册、续约、下线请求
   - 支持集群部署，节点间通过异步复制同步注册表
   - 提供REST API供Client查询

2. **Eureka Client（服务实例）**：
   - **服务提供方（Provider）**：启动时向Server注册，定期发送心跳续约
   - **服务消费方（Consumer）**：从Server拉取服务列表，缓存在本地，直接调用目标服务
   - 内置定时任务：HeartbeatThread（心跳，默认30秒）和CacheRefreshThread（缓存刷新，默认30秒）

**通信协议**：
- 注册：`POST /eureka/apps/{appName}`
- 续约（心跳）：`PUT /eureka/apps/{appName}/{instanceId}`
- 下线：`DELETE /eureka/apps/{appName}/{instanceId}`
- 查询全量：`GET /eureka/apps`
- 查询增量：`GET /eureka/apps/delta`
- 复制（Server间）：`POST /peerreplication/batch`

### Q2：Eureka的AP架构是如何实现的？为什么它选择AP而不是CP？

**参考答案**：

Eureka的AP实现：

1. **对等复制（Peer-to-Peer Replication）**：所有Server节点对等，无Master/Slave，任何节点都可读写
2. **异步复制**：Client写入一个节点后立即返回，该节点异步将数据复制到其他节点
3. **冲突解决**：使用`last-dirty-timestamp`（最后修改时间戳），时间戳大的覆盖小的
4. **客户端缓存**：Client本地缓存服务列表，即使所有Server宕机也能继续调用
5. **自我保护**：网络分区时不剔除任何服务，保证可用性

**选择AP的原因**：
- 微服务实例频繁扩缩容，强一致性带来的延迟不可接受
- 服务发现场景允许短暂的不一致（读到已下线的实例只会导致少量调用失败，可通过重试解决）
- 如果注册中心不可用导致无法发现服务，比读到过期数据的影响更严重
- AWS环境的设计理念：故障是常态，可用性优先

### Q3：详细描述Eureka的心跳机制（Lease机制）

**参考答案**：

Eureka使用**租约（Lease）机制**管理实例生命周期：

**核心参数**：
- `lease-renewal-interval-in-seconds`：心跳间隔，默认30秒
- `lease-expiration-duration-in-seconds`：租约过期时间，默认90秒

**流程**：
1. **注册**：Client启动时发送POST请求注册，Server创建`Lease<InstanceInfo>`对象，设置`registrationTimestamp`和`lastUpdateTimestamp`
2. **续约**：Client每30秒发送PUT心跳请求，Server调用`lease.renew()`更新`lastUpdateTimestamp = currentTime + duration`
3. **过期检查**：Server每60秒执行`EvictionTask`，遍历注册表，检查`System.currentTimeMillis() > lastUpdateTimestamp + duration`，满足则标记为过期
4. **剔除**：将过期实例从注册表中移除，并同步到其他Server节点
5. **下线**：Client主动发送DELETE请求，或应用关闭时调用`eurekaClient.shutdown()`

**容错设计**：
- 实例最多可连续丢失2次心跳（90秒内无心跳才剔除）
- 容忍短暂的网络抖动

### Q4：什么是Eureka的自我保护机制？触发条件和行为是什么？

**参考答案**：

**自我保护机制（Self-Preservation）**是Eureka应对网络分区的保护策略。

**触发条件**：
- 计算预期每分钟续约次数：`expectedRenewsPerMin = instanceCount × 2 × 0.85`
- 如果最近一分钟实际续约次数 `actualRenews < expectedRenewsPerMin`，触发自我保护
- 即：15分钟内心跳失败比例超过85%

**保护行为**：
1. **停止剔除任何服务**：即使租约过期也不剔除
2. **继续接受新注册**：新服务可以正常注册
3. **继续提供查询**：Client可以继续获取服务列表
4. **网络恢复后自动退出**：当实际续约数恢复到阈值以上时，自动退出自我保护

**意义**：防止网络分区时Server误认为大量Client宕机而剔除它们，避免服务雪崩。

**配置**：
```yaml
eureka:
  server:
    enable-self-preservation: true
    renewal-percent-threshold: 0.85
```

### Q5：Eureka Server集群之间如何同步数据？有什么特点？

**参考答案**：

**同步机制**：
- 采用**异步复制（Asynchronous Replication）**
- Client注册到一个Server后，该Server通过HTTP向其他节点发送复制请求
- 复制操作类型：Register、Heartbeat、Cancel、StatusUpdate

**特点**：
1. **对等架构（Peer-to-Peer）**：无Master节点，所有节点对等
2. **最终一致性**：复制有延迟（毫秒到秒级），可能出现短暂不一致
3. **无冲突检测**：以`last-dirty-timestamp`为准，后写入的覆盖先写入的
4. **批量复制**：支持批量复制优化性能

**数据流**：
```
Client -> Server A (写入成功，立即返回)
   └─> Server A -> Server B (异步复制)
   └─> Server A -> Server C (异步复制)
```

### Q6：Eureka Client本地缓存是如何工作的？为什么即使Eureka Server全部宕机，服务间仍能调用？

**参考答案**：

**本地缓存机制**：

1. **全量拉取**：Client启动时从Server全量拉取注册表（`GET /eureka/apps`），存储在`AtomicReference<Applications> localRegionApps`中
2. **增量更新**：之后每30秒执行增量拉取（`GET /eureka/apps/delta`），只获取变更部分，合并到本地缓存
3. **定时刷新**：通过`CacheRefreshThread`定时任务执行

**Server宕机后的可用性**：
- Client本地持有完整的服务列表副本
- 服务调用直接从本地缓存获取目标实例地址，不依赖Server
- 只有在新服务注册或服务下线时，才需要Server参与
- 因此即使所有Eureka Server宕机，已缓存的服务间调用仍可继续（直到本地缓存中的实例实际不可用）

这种设计体现了**Eureka的AP哲学**：可用性优先，宁可调用已下线的实例（可通过客户端重试/熔断处理），也不能因为注册中心不可用而完全无法调用。

### Q7：Eureka和Nacos有什么区别？

**参考答案**：

| 特性 | Eureka | Nacos |
|------|--------|-------|
| 开发方 | Netflix（已停止维护） | 阿里巴巴（活跃） |
| CAP | AP（仅支持最终一致） | AP/CP可选（默认AP） |
| 健康检查 | Client心跳（被动） | Client心跳 + Server主动探测 |
| 配置中心 | 不支持 | 原生支持（动态配置） |
| 负载均衡 | 需配合Ribbon | 内置多种策略 |
| 控制台 | 简单（仅注册表查看） | 丰富（服务/配置/流量管理） |
| 性能 | 一般（满足大多数场景） | 更好（支持10万+实例） |
| Spring Cloud集成 | spring-cloud-netflix | spring-cloud-alibaba |
| 元数据 | 简单Map | 丰富元数据 + 权重 + 灰度 |

**选型建议**：
- 新项目优先选择Nacos
- 存量Spring Cloud Netflix系统可继续维护使用Eureka
- 需要配置中心时Nacos更合适（一体化）

### Q8：Eureka的ResponseCache是如何设计的？为什么要用两级缓存？

**参考答案**：

Eureka Server使用**两级缓存**优化读性能：

**缓存结构**：
1. **readWriteCacheMap**（读写缓存）：`ConcurrentHashMap`，写操作（注册/续约/下线）时直接清除对应key
2. **readOnlyCacheMap**（只读缓存）：`Guava LoadingCache`，定时（默认30秒）从readWriteCacheMap同步数据

**读取流程**：
1. Client查询时，优先从readOnlyCacheMap读取
2. 未命中时，从readWriteCacheMap读取，并回填到readOnlyCacheMap
3. 写操作发生时，清除readWriteCacheMap，不立即清除readOnlyCacheMap（异步同步）

**为什么用两级缓存**：
1. **减少锁竞争**：读操作只访问缓存，不直接访问注册表（`registry`是`ConcurrentHashMap`但仍存在竞争）
2. **读写分离**：读走只读缓存，写清除读写缓存，互不阻塞
3. **性能优化**：只读缓存使用Guava Cache，支持TTL自动过期，读性能极高
4. **最终一致性**：允许短暂的不一致（30秒窗口），换取极高的读吞吐量

**缺点**：
- 增加了数据不一致窗口（最长30秒）
- 需要额外的内存存储缓存

### Q9：如何排查Eureka服务注册不上的问题？

**参考答案**：

**排查步骤**：

1. **检查网络连通性**：
   ```bash
   curl http://eureka-server:8761/eureka/apps
   telnet eureka-server 8761
   ```

2. **检查Client配置**：
   ```yaml
   eureka:
     client:
       service-url:
         defaultZone: http://eureka-server:8761/eureka/
       register-with-eureka: true  # 必须为true
       fetch-registry: true        # 消费方必须为true
   ```

3. **检查应用名称**：
   ```yaml
   spring:
     application:
       name: user-service  # 必须有名称，否则注册失败
   ```

4. **查看Client日志**：
   - 搜索`DiscoveryClient_`开头的日志
   - 检查是否有`Registration status: 204`（注册成功）
   - 检查是否有连接超时或拒绝连接异常

5. **查看Server日志**：
   - 搜索`Registering instance`确认收到注册请求
   - 检查是否触发了自我保护模式

6. **检查防火墙和安全组**：
   - 确保8761端口开放
   - 如果使用Spring Security，确保Client携带正确的认证信息

7. **使用Actuator端点**：
   ```bash
   curl http://localhost:8080/actuator/health
   curl http://localhost:8080/actuator/eureka
   ```

### Q10：在Eureka中如何实现服务的灰度发布？

**参考答案**：

Eureka本身不直接支持灰度发布，但可以通过**元数据（Metadata）**和**自定义负载均衡规则**实现：

**方案1：基于元数据过滤**

```yaml
# 服务提供方：标记版本
eureka:
  instance:
    metadata-map:
      version: v2
      gray: true
```

```java
// 自定义Ribbon负载均衡规则
public class GrayReleaseRule extends AbstractLoadBalancerRule {
    
    @Override
    public Server choose(Object key) {
        ILoadBalancer lb = getLoadBalancer();
        List<Server> servers = lb.getReachableServers();
        
        // 从请求上下文中获取目标版本
        String targetVersion = RibbonFilterContextHolder.getCurrentContext().get("version");
        
        // 过滤匹配版本的服务实例
        List<Server> grayServers = servers.stream()
            .filter(server -> {
                if (server instanceof DiscoveryEnabledServer) {
                    DiscoveryEnabledServer dServer = (DiscoveryEnabledServer) server;
                    String version = dServer.getInstanceInfo()
                        .getMetadata().get("version");
                    return targetVersion.equals(version);
                }
                return false;
            })
            .collect(Collectors.toList());
        
        // 如果灰度实例存在，从中选择；否则使用所有实例
        if (!grayServers.isEmpty()) {
            return new RandomRule().choose(lb, key, grayServers);
        }
        
        return new RoundRobinRule().choose(lb, key);
    }
}
```

**方案2：基于区域（Zone）的灰度**

```yaml
eureka:
  instance:
    metadata-map:
      zone: gray-zone
```

**方案3：结合Spring Cloud Gateway实现**

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: user-service-gray
          uri: lb://user-service
          predicates:
            - Path=/api/user/**
            - Header=Gray-Version, v2
          metadata:
            version: v2
```

**注意**：Eureka的灰度能力较弱，更复杂的灰度发布建议：
- 使用Nacos（原生支持权重和灰度）
- 使用Istio/Envoy等服务网格
- 使用Spring Cloud Gateway + 自定义过滤器

---

*此文原创，转载请注明出处。*
