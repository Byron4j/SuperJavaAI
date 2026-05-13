# Nginx深度解析：高性能反向代理与负载均衡的底层原理与工业实践

**文章标签：** #nginx #反向代理 #负载均衡 #高性能 #epoll #事件驱动 #源码分析 #运维 #面试

## 目录

- [引言：Nginx的技术本质](#引言nginx的技术本质)
- [理论基础：为什么Nginx能支撑10万并发](#理论基础为什么nginx能支撑10万并发)
- [来龙去脉：Web服务器的架构演进](#来龙去脉web服务器的架构演进)
- [核心架构深度解析](#核心架构深度解析)
- [负载均衡算法底层原理](#负载均衡算法底层原理)
- [源码深度分析：连接处理全流程](#源码深度分析连接处理全流程)
- [工业级配置实战](#工业级配置实战)
- [Nginx vs Apache vs Caddy：架构对比](#nginx-vs-apache-vs-caddy架构对比)
- [性能分析与调优](#性能分析与调优)
- [常见陷阱与最佳实践](#常见陷阱与最佳实践)
- [面试题与参考答案](#面试题与参考答案)

---

## 引言：Nginx的技术本质

Nginx不是"另一个Web服务器"，而是**基于事件驱动的异步非阻塞I/O架构**的高性能网络中间件。

核心认知：

```
传统Web服务器（Apache Prefork）的本质：
每个连接 = 一个进程/线程
┌─────────┐  ┌─────────┐  ┌─────────┐
│ Client1 │  │ Client2 │  │ Client3 │
└────┬────┘  └────┬────┘  └────┬────┘
     │            │            │
┌────▼────┐  ┌────▼────┐  ┌────▼────┐
│ Process1│  │ Process2│  │ Process3│
│ 阻塞等待 │  │ 阻塞等待 │  │ 阻塞等待 │
└─────────┘  └─────────┘  └─────────┘

Nginx的本质：
一个Worker进程处理数万个连接
┌─────────┐  ┌─────────┐  ┌─────────┐  ...  ┌──────────┐
│ Client1 │  │ Client2 │  │ Client3 │       │ Client10K│
└────┬────┘  └────┬────┘  └────┬────┘       └────┬─────┘
     │            │            │                 │
     └────────────┴────────────┴────────┬────────┘
                                        │
                              ┌─────────▼──────────┐
                              │   Worker进程        │
                              │  ┌──────────────┐  │
                              │  │  事件循环     │  │
                              │  │  epoll_wait  │  │
                              │  └──────────────┘  │
                              └────────────────────┘
```

**关键洞察**：Nginx的高性能不取决于"代码写得好"，而取决于**架构设计彻底摒弃了每个连接一个线程的阻塞模型**，将I/O等待从线程占用转化为事件回调。

---

## 理论基础：为什么Nginx能支撑10万并发

### 1. C10K问题与I/O模型演进

#### 阻塞I/O的困境

```
阻塞I/O模型（Blocking I/O）：

应用进程              内核
   │    read()      │
   │────────────────>│
   │                 │ 数据未就绪
   │    阻塞等待      │
   │<────────────────│ 数据就绪，复制到用户态
   │    返回数据      │
   │                 │

问题：一个线程阻塞在一个连接上，无法处理其他连接
     10K连接 = 10K线程 → 上下文切换开销巨大，内存占用爆炸
```

#### 非阻塞I/O与I/O多路复用

```
I/O多路复用模型（select/poll/epoll）：

应用进程              内核
   │   select(epoll) │
   │────────────────>│
   │                 │ 监控多个fd
   │    阻塞等待      │ 任一fd就绪即返回
   │<────────────────│
   │   返回就绪fd列表 │
   │                 │
   │   read(fd1)     │  非阻塞读取
   │   read(fd2)     │  非阻塞读取
   │                 │

突破：一个线程监控数万个连接，只处理就绪的连接
```

#### select vs poll vs epoll 底层对比

| 特性 | select | poll | epoll |
|------|--------|------|-------|
| 数据结构 | fd_set（位图） | 链表 | 红黑树 + 就绪链表 |
| 最大fd限制 | 1024（默认） | 无限制 | 无限制 |
| 时间复杂度 | O(n) | O(n) | O(1) |
| 触发模式 | 水平触发 | 水平触发 | 支持边缘触发 |
| 数据拷贝 | 每次调用拷贝fd_set | 每次调用拷贝数组 | 首次epoll_ctl拷贝，后续不拷贝 |

```c
// select 的问题：每次调用都要拷贝整个fd_set到内核
fd_set readfds;
FD_ZERO(&readfds);
FD_SET(fd1, &readfds);
FD_SET(fd2, &readfds);
// ... 每次select都要拷贝readfds
select(max_fd + 1, &readfds, NULL, NULL, NULL);

// epoll 的优势：fd只需注册一次
int epoll_fd = epoll_create1(0);
struct epoll_event ev;
ev.events = EPOLLIN;
ev.data.fd = fd1;
epoll_ctl(epoll_fd, EPOLL_CTL_ADD, fd1, &ev);  // 只注册一次

// 等待事件，返回就绪的fd列表（只返回就绪的，不遍历全部）
struct epoll_event events[MAX_EVENTS];
int nfds = epoll_wait(epoll_fd, events, MAX_EVENTS, -1);
```

**关键理解**：
- select/poll 的问题在于**每次都要遍历所有fd**，连接数增加时性能线性下降
- epoll 使用**回调机制**：fd就绪时内核通过回调函数将其加入就绪链表，epoll_wait直接返回就绪列表，时间复杂度O(1)
- Nginx在Linux上使用epoll，在FreeBSD/macOS上使用kqueue（类似epoll）

### 2. 事件驱动异步架构

```
Nginx事件循环（Event Loop）的核心逻辑：

┌─────────────────────────────────────────┐
│            Worker进程启动                │
└─────────────────┬───────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│  1. 初始化：监听端口，注册epoll事件       │
└─────────────────┬───────────────────────┘
                  │
                  ▼
        ┌─────────────────┐
        │   事件循环开始   │◄─────────────────────┐
        └────────┬────────┘                      │
                 │                               │
                 ▼                               │
┌─────────────────────────────────────────┐     │
│  2. epoll_wait(timeout)                 │     │
│     阻塞等待I/O事件或定时器到期          │     │
└─────────────────┬───────────────────────┘     │
                  │                              │
                  ▼                              │
┌─────────────────────────────────────────┐     │
│  3. 分发事件（Event Dispatch）           │     │
│     - 新连接事件 → accept_handler()      │     │
│     - 可读事件   → read_handler()        │     │
│     - 可写事件   → write_handler()       │     │
│     - 定时器事件 → timer_handler()       │     │
└─────────────────┬───────────────────────┘     │
                  │                              │
                  ▼                              │
┌─────────────────────────────────────────┐     │
│  4. 处理事件（非阻塞）                   │     │
│     - 读取请求 → 解析HTTP                │     │
│     - 处理逻辑 → 访问静态文件/反向代理    │     │
│     - 发送响应 → 注册可写事件            │     │
└─────────────────┬───────────────────────┘     │
                  │                              │
                  ▼                              │
        ┌─────────────────┐                    │
        │   继续下一轮     │────────────────────┘
        └─────────────────┘

关键：没有线程阻塞在I/O等待上，所有等待都在epoll_wait中统一处理
```

### 3. 零拷贝（Zero-Copy）技术

```
传统文件发送的数据拷贝路径：

磁盘 → 内核PageCache → 用户缓冲区 → 内核Socket缓冲区 → 网卡
     (DMA拷贝)      (CPU拷贝)      (CPU拷贝)         (DMA拷贝)
     
     4次上下文切换，4次数据拷贝（2次DMA，2次CPU）

Nginx sendfile零拷贝：

磁盘 → 内核PageCache ──────────────> 网卡
     (DMA拷贝)              (DMA拷贝，通过sendfile系统调用)
     
     2次上下文切换，2次DMA拷贝，0次CPU拷贝

┌─────────────────────────────────────────┐
│ 用户态                                  │
│   sendfile(fd_in, fd_out, offset, len)  │
└─────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│ 内核态                                  │
│   1. DMA从磁盘读取到PageCache            │
│   2. 直接将PageCache引用传递给Socket     │
│   3. DMA从Socket缓冲区发送到网卡         │
│                                        │
│   不需要经过用户态缓冲区！               │
└─────────────────────────────────────────┘
```

**Nginx配置**：
```nginx
http {
    # 开启sendfile零拷贝
    sendfile on;
    
    # 配合tcp_nopush，累积数据包减少网络拥塞
    tcp_nopush on;
    
    # 快速响应小数据包
    tcp_nodelay on;
}
```

---

## 来龙去脉：Web服务器的架构演进

### 第一代：进程模型（Apache Prefork）

```
Apache Prefork MPM：

┌──────────────────────────────────────┐
│           Apache主进程                │
│  ┌────────┐ ┌────────┐ ┌────────┐   │
│  │子进程1 │ │子进程2 │ │子进程3 │   │
│  │处理请求│ │处理请求│ │处理请求│   │
│  │阻塞I/O │ │阻塞I/O │ │阻塞I/O │   │
│  └────────┘ └────────┘ └────────┘   │
└──────────────────────────────────────┘

优点：稳定性好，一个进程崩溃不影响其他进程
缺点：进程切换开销大，内存占用高，C10K问题
      每个进程通常占用2-10MB内存，10K连接 = 20-100GB内存
```

### 第二代：线程模型（Apache Worker）

```
Apache Worker MPM：

┌──────────────────────────────────────┐
│           Apache主进程                │
│  ┌────────────────────────────────┐  │
│  │        线程池（每个进程）        │  │
│  │  ┌────┐┌────┐┌────┐┌────┐     │  │
│  │  │线程││线程││线程││线程│     │  │
│  │  └────┘└────┘└────┘└────┘     │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘

优点：线程切换比进程切换快，内存共享
缺点：线程仍阻塞在I/O上，并发能力有限
      多线程编程复杂，锁竞争严重
```

### 第三代：事件驱动模型（Nginx）

```
Nginx Master-Worker模型：

┌──────────────────────────────────────┐
│           Master进程                  │
│  - 读取配置                          │
│  - 管理Worker进程（fork/spawn）       │
│  - 处理信号（reload/stop/quit）       │
│  - 不处理客户端请求                   │
└──────────────┬───────────────────────┘
               │ fork
    ┌──────────┼──────────┐
    ▼          ▼          ▼
┌───────┐ ┌───────┐ ┌───────┐
│Worker1│ │Worker2│ │Worker3│
│epoll  │ │epoll  │ │epoll  │
│事件循环│ │事件循环│ │事件循环│
└───────┘ └───────┘ └───────┘

特点：
1. Worker进程之间相互独立，无共享状态（无锁设计）
2. 每个Worker单线程，通过epoll处理数万连接
3. Master负责管理，Worker负责干活，职责分离
```

### 为什么Nginx选择了多进程单线程而非单进程多线程？

```
多进程单线程的优势：

1. 无锁编程（Lock-Free）
   - 每个Worker独立处理连接，无需锁竞争
   - 避免了多线程的复杂同步问题

2. 容错性
   - 一个Worker崩溃，Master可以立即fork新的Worker
   - 不影响其他Worker和整体服务

3. 利用多核CPU
   - 每个Worker绑定到一个CPU核心
   - 避免进程迁移带来的缓存失效

4. 简单性
   - 单线程代码比多线程代码简单得多
   - 没有死锁、竞态条件、线程安全等问题

┌─────────────────────────────────────┐
│           CPU 0                     │
│  ┌─────────────────────────────┐   │
│  │ Worker 0（绑定CPU0）          │   │
│  │ - 独立内存空间                │   │
│  │ - 独立epoll实例               │   │
│  │ - 无锁，无竞态                │   │
│  └─────────────────────────────┘   │
├─────────────────────────────────────┤
│           CPU 1                     │
│  ┌─────────────────────────────┐   │
│  │ Worker 1（绑定CPU1）          │   │
│  │ - 独立内存空间                │   │
│  │ - 独立epoll实例               │   │
│  │ - 无锁，无竞态                │   │
│  └─────────────────────────────┘   │
└─────────────────────────────────────┘
```

**配置Worker与CPU绑定**：
```nginx
worker_processes auto;  # 自动设置为CPU核心数
worker_cpu_affinity auto;  # 自动绑定CPU
```

---

## 核心架构深度解析

### 1. Nginx进程模型详解

```
Nginx进程家族：

┌──────────────────────────────────────────────┐
│              Master Process (root)            │
│  PID: 1234                                    │
│  ├─ 读取并验证nginx.conf                      │
│  ├─ 创建监听socket（bind/listen）             │
│  ├─ fork Worker进程                           │
│  ├─ 处理信号（HUP, USR1, USR2, WINCH, QUIT）  │
│  └─ 管理Cache Loader/Cache Manager            │
└────────────────────┬─────────────────────────┘
                     │ fork
        ┌────────────┼────────────┐
        ▼            ▼            ▼
   ┌─────────┐ ┌─────────┐ ┌─────────┐
   │ Worker1 │ │ Worker2 │ │ Worker3 │
   │ (nobody)│ │ (nobody)│ │ (nobody)│
   │ PID:1235│ │ PID:1236│ │ PID:1237│
   └─────────┘ └─────────┘ └─────────┘
        │            │            │
        └────────────┴────────────┘
                     │
              监听相同端口
              (SO_REUSEPORT)

信号处理：
- HUP  : 优雅重载配置（reload）
- USR1 : 重新打开日志文件（日志切割）
- USR2 : 热升级（平滑升级到新版Nginx）
- WINCH: 优雅关闭Worker进程
- QUIT : 优雅退出
- TERM : 快速退出
```

### 2. HTTP请求的11个处理阶段

Nginx将HTTP请求处理划分为11个阶段（phase），每个阶段由不同的模块处理：

```
HTTP请求处理阶段（ngx_http_phases）：

┌─────┬────────────────────────┬──────────────────────────────┐
│ 阶段 │ 名称                    │ 典型处理模块                  │
├─────┼────────────────────────┼──────────────────────────────┤
│  1  │ NGX_HTTP_POST_READ      │ realip模块（读取真实IP）      │
├─────┼────────────────────────┼──────────────────────────────┤
│  2  │ NGX_HTTP_SERVER_REWRITE │ rewrite模块（server级别重写） │
├─────┼────────────────────────┼──────────────────────────────┤
│  3  │ NGX_HTTP_FIND_CONFIG    │ 核心：根据URI定位location     │
├─────┼────────────────────────┼──────────────────────────────┤
│  4  │ NGX_HTTP_REWRITE        │ rewrite模块（location级别）   │
├─────┼────────────────────────┼──────────────────────────────┤
│  5  │ NGX_HTTP_POST_REWRITE   │ 检查rewrite次数（防死循环）   │
├─────┼────────────────────────┼──────────────────────────────┤
│  6  │ NGX_HTTP_PREACCESS      │ limit_req, limit_conn（限流） │
├─────┼────────────────────────┼──────────────────────────────┤
│  7  │ NGX_HTTP_ACCESS         │ access, auth_basic（认证）    │
├─────┼────────────────────────┼──────────────────────────────┤
│  8  │ NGX_HTTP_POST_ACCESS    │ 检查访问权限结果              │
├─────┼────────────────────────┼──────────────────────────────┤
│  9  │ NGX_HTTP_TRY_FILES      │ try_files指令                 │
├─────┼────────────────────────┼──────────────────────────────┤
│ 10  │ NGX_HTTP_CONTENT        │ proxy, fastcgi, static（内容）│
├─────┼────────────────────────┼──────────────────────────────┤
│ 11  │ NGX_HTTP_LOG            │ access_log模块（日志）        │
└─────┴────────────────────────┴──────────────────────────────┘

请求处理流程：

Client Request
      │
      ▼
┌──────────────┐
│ POST_READ    │ ──> 获取真实IP（X-Forwarded-For）
└──────┬───────┘
       │
┌──────▼───────┐
│SERVER_REWRITE│ ──> server块rewrite
└──────┬───────┘
       │
┌──────▼───────┐
│FIND_CONFIG   │ ──> location匹配（关键阶段！）
└──────┬───────┘
       │
┌──────▼───────┐
│ REWRITE      │ ──> location块rewrite
└──────┬───────┘
       │
      ...
       │
┌──────▼───────┐
│ CONTENT      │ ──> 生成响应（proxy_pass / static）
└──────┬───────┘
       │
┌──────▼───────┐
│ LOG          │ ──> 记录访问日志
└──────────────┘
```

### 3. 内存池机制（ngx_pool_t）

Nginx使用内存池管理内存，避免频繁的malloc/free：

```
Nginx内存池结构：

┌─────────────────────────────────────────┐
│           ngx_pool_t（内存池）            │
│  ┌─────────────────────────────────┐   │
│  │ 小块内存（<4KB）：链表管理        │   │
│  │ ┌────────┐   ┌────────┐        │   │
│  │ │ 4KB块1 │──>│ 4KB块2 │──> ... │   │
│  │ └────────┘   └────────┘        │   │
│  └─────────────────────────────────┘   │
│  ┌─────────────────────────────────┐   │
│  │ 大块内存（≥4KB）：直接malloc     │   │
│  │ ┌────────┐   ┌────────┐        │   │
│  │ │ 大块1  │──>│ 大块2  │──> ... │   │
│  │ └────────┘   └────────┘        │   │
│  └─────────────────────────────────┘   │
│  ┌─────────────────────────────────┐   │
│  │ 清理回调（cleanup）：释放资源     │   │
│  │ ┌────────┐   ┌────────┐        │   │
│  │ │ fd关闭 │──>│ 内存释放│──> ... │   │
│  │ └────────┘   └────────┘        │   │
│  └─────────────────────────────────┘   │
└─────────────────────────────────────────┘

内存分配策略：
1. 小块内存：从当前block分配，block用完申请新block
2. 大块内存：直接调用malloc，单独管理
3. 内存只在池销毁时统一释放（无单独free）

优势：
- 减少内存碎片
- 避免频繁系统调用
- 请求处理完毕一次性释放，无内存泄漏风险
```

---

## 负载均衡算法底层原理

### 1. 负载均衡算法对比

| 算法 | 适用场景 | 优点 | 缺点 |
|------|---------|------|------|
| 轮询（Round Robin） | 后端性能均等 | 简单均匀 | 不考虑服务器负载 |
| 加权轮询（Weighted RR） | 后端性能不均 | 按能力分配 | 需要人工配置权重 |
| IP哈希（IP Hash） | 会话保持 | 同一用户固定路由 | 后端变化时大量重哈希 |
| 最少连接（Least Conn） | 长连接场景 | 按实际负载分配 | 需要维护连接计数 |
| 一致性哈希（Consistent Hash） | 缓存场景 | 节点变化影响小 | 实现复杂，可能有倾斜 |

### 2. 平滑加权轮询算法（Nginx SW算法）

Nginx使用**平滑加权轮询（Smooth Weighted Round Robin）**算法，避免普通加权轮询的"突发"问题：

```
普通加权轮询的问题：
服务器A权重5，服务器B权重1
分配序列：A A A A A B A A A A A B ...
问题：前5个请求全部打到A，然后才到B，不够平滑

平滑加权轮询（Nginx算法）：

初始化：
  Server A: weight=5, current_weight=0
  Server B: weight=1, current_weight=0

第1轮：
  A: current_weight = 0 + 5 = 5
  B: current_weight = 0 + 1 = 1
  选择A（最大），A.current_weight = 5 - (5+1) = -1

第2轮：
  A: current_weight = -1 + 5 = 4
  B: current_weight = 1 + 1 = 2
  选择A，A.current_weight = 4 - 6 = -2

第3轮：
  A: current_weight = -2 + 5 = 3
  B: current_weight = 2 + 1 = 3
  选择A（相等选第一个），A.current_weight = 3 - 6 = -3

第4轮：
  A: current_weight = -3 + 5 = 2
  B: current_weight = 3 + 1 = 4
  选择B，B.current_weight = 4 - 6 = -2

分配序列：A A A B A A ...
结果：6个请求中A占5个，B占1个，且分布均匀
```

### 3. 一致性哈希（Consistent Hashing）

```
一致性哈希原理：

普通哈希：hash(key) % N
问题：N变化时，几乎所有key都重新映射

一致性哈希：

将hash空间组织成一个环（0 ~ 2^32-1）

      hash环
   0 ┌─────────────────┐ 2^32
     │    服务器A      │
     │      ●          │
     │  (hash=100)     │
     │                 │
     │    服务器B      │
     │      ●          │
     │  (hash=1000)    │
     │                 │
     │    服务器C      │
     │      ●          │
     │  (hash=5000)    │
     └─────────────────┘

Key的分配规则：
  hash(key) 顺时针找到的第一个服务器

  Key1(hash=50)  ──>  服务器A(100)
  Key2(hash=800) ──>  服务器B(1000)
  Key3(hash=3000)──>  服务器C(5000)

优势：服务器B宕机时，只有B到C之间的key受影响
     其他key的映射关系不变！

虚拟节点（解决倾斜问题）：
  每个物理服务器对应多个虚拟节点
  服务器A: A#1, A#2, A#3 ...
  使key分布更均匀
```

**Nginx配置**：
```nginx
upstream backend {
    # 一致性哈希，基于request_uri
    hash $request_uri consistent;
    
    server 192.168.1.1:8080;
    server 192.168.1.2:8080;
    server 192.168.1.3:8080;
}
```

---

## 源码深度分析：连接处理全流程

### 1. 连接建立流程

```
Nginx处理新连接的事件驱动流程：

┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  内核TCP层   │     │ Nginx核心    │     │  HTTP模块    │
└──────┬──────┘     └──────┬──────┘     └──────┬──────┘
       │                   │                   │
       │ 1. SYN包到达       │                   │
       │──────────────────>│                   │
       │                   │                   │
       │ 2. 完成三次握手    │                   │
       │    触发epoll事件   │                   │
       │──────────────────>│                   │
       │                   │                   │
       │                   │ 3. accept新连接    │
       │                   │    ngx_event_accept│
       │                   │                   │
       │                   │ 4. 创建ngx_connection_t
       │                   │    初始化连接结构体 │
       │                   │                   │
       │                   │ 5. 注册可读事件    │
       │                   │    epoll_ctl(ADD) │
       │                   │                   │
       │                   │ 6. 等待HTTP请求    │
       │                   │                   │
       │ 7. 客户端发送请求  │                   │
       │──────────────────>│                   │
       │                   │                   │
       │                   │ 8. 触发read事件    │
       │                   │    调用read_handler│
       │                   │                   │
       │                   │ 9. HTTP状态机解析  │
       │                   │    解析请求行/头部 │
       │                   │                   │
       │                   │ 10. 进入11个phase │
       │                   │    处理请求        │
       │                   │                   │
       │                   │ 11. 生成响应       │
       │                   │    注册可写事件    │
       │                   │                   │
       │ 12. 发送响应       │                   │
       │<──────────────────│                   │
       │                   │                   │
       │ 13. 保持连接或关闭 │                   │
       │                   │                   │

关键数据结构：
- ngx_connection_t：连接对象，封装socket fd
- ngx_event_t：事件对象，关联回调函数
- ngx_http_request_t：HTTP请求对象
```

### 2. HTTP状态机解析

```
Nginx HTTP请求解析状态机：

┌──────────────┐
│  sw_start    │
└──────┬───────┘
       │ 读取第一个字符
       ▼
┌──────────────┐     ┌──────────────┐
│  sw_method   │────>│   sw_spaces  │
│  解析HTTP方法 │     │  跳过空格    │
└──────────────┘     └──────┬───────┘
                            │
                            ▼
                    ┌──────────────┐
                    │   sw_uri     │
                    │  解析URI     │
                    └──────┬───────┘
                           │
                           ▼
                    ┌──────────────┐
                    │ sw_http_H    │
                    │ 等待"HTTP"   │
                    └──────┬───────┘
                           │
                           ▼
                    ┌──────────────┐
                    │ sw_major_digit│
                    │ 解析主版本号  │
                    └──────┬───────┘
                           │
                           ▼
                    ┌──────────────┐
                    │ sw_minor_digit│
                    │ 解析次版本号  │
                    └──────┬───────┘
                           │
                           ▼
                    ┌──────────────┐
                    │   sw_done    │
                    │ 请求行解析完成│
                    └──────────────┘

解析完成后进入HTTP头部解析（类似的状态机）

为什么用状态机而非正则？
- 状态机是O(n)线性扫描，性能最高
- 流式处理，不需要等完整请求到达
- 内存占用固定，不受输入大小影响
```

### 3. Location匹配算法源码级解析

```
Nginx Location匹配流程（ngx_http_core_find_location）：

1. 精确匹配（=）优先检查
   
   location = /exact {
       # 只有URI完全等于/exact才匹配
       # O(1)哈希查找
   }

2. 前缀字符串匹配（无修饰符）
   
   location /prefix {
       # 最长前缀匹配
       # 存储在二叉搜索树中，O(logN)
   }

3. 优先前缀匹配（^~）
   
   location ^~ /static {
       # 如果匹配成功，停止检查正则
       # 用于提高性能（跳过正则匹配）
   }

4. 正则匹配（~ 和 ~*）
   
   location ~ \.php$ {
       # 按配置文件中定义的顺序依次匹配
       # 第一个匹配的正则生效
   }
   
   location ~* \.(jpg|png)$ {
       # ~* 不区分大小写
   }

5. 通用匹配（/）
   
   location / {
       # 最低优先级，兜底匹配
   }

匹配顺序图示：

URI: /api/v1/users

检查顺序：
  1. = /api/v1/users  ?  否
  2. = /api           ?  否
  3. 最长前缀匹配      ?  /api/v1 (长度7)
  4. ^~ /api/v1       ?  如果匹配，停止
  5. ~ /api/.*        ?  按顺序检查正则
  6. ~ /.*            ?  第一个匹配的正则
  7. 无修饰符最长前缀   ->  使用 /api/v1

关键源码逻辑：
static ngx_int_t
ngx_http_core_find_location(ngx_http_request_t *r) {
    // 1. 检查精确匹配（哈希表O(1)）
    rc = ngx_http_core_find_static_location(r, clcf->locations);
    
    if (rc == NGX_OK && r->loc_conf == ...) {
        return NGX_OK;  // 精确匹配直接返回
    }
    
    // 2. 检查前缀匹配（二叉树O(logN)）
    // 3. 检查正则匹配（顺序遍历）
    // 4. 返回最长前缀匹配结果
}
```

**工业级Location配置示例**：
```nginx
server {
    listen 80;
    server_name api.example.com;
    
    # 1. 精确匹配：最高优先级
    location = /health {
        access_log off;
        return 200 "OK\n";
        add_header Content-Type text/plain;
    }
    
    # 2. 优先前缀匹配：匹配后跳过正则（性能优化）
    location ^~ /static/ {
        alias /data/static/;
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
    
    # 3. 正则匹配：区分大小写
    location ~ \.php$ {
        fastcgi_pass 127.0.0.1:9000;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
    
    # 4. 正则匹配：不区分大小写（图片缓存）
    location ~* \.(jpg|jpeg|png|gif|ico|css|js|svg|woff|woff2)$ {
        root /data/static;
        expires 30d;
        access_log off;
    }
    
    # 5. 通用匹配：兜底
    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

---

## 工业级配置实战

### 1. 高可用架构：Nginx + Keepalived

```
高可用架构设计：

                    ┌─────────────┐
                    │   客户端     │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │  虚拟IP(VIP) │  10.0.0.100
                    │  (浮动IP)    │
                    └──────┬──────┘
                           │
              ┌────────────┴────────────┐
              │                         │
       ┌──────▼──────┐         ┌──────▼──────┐
       │  Nginx主节点 │         │  Nginx备节点 │
       │  Keepalived │◄───────►│  Keepalived │
       │  MASTER     │  VRRP   │  BACKUP     │
       │  10.0.0.10  │         │  10.0.0.11  │
       └──────┬──────┘         └─────────────┘
              │
       ┌──────┴──────┐
       │             │
  ┌────▼────┐   ┌────▼────┐
  │Backend1 │   │Backend2 │
  └─────────┘   └─────────┘

Keepalived通过VRRP协议实现VIP漂移：
- 主节点宕机，VIP自动漂移到备节点
- 客户端始终访问VIP，无感知切换
```

**Keepalived配置**（`/etc/keepalived/keepalived.conf`）：
```bash
# 主节点配置
vrrp_script chk_nginx {
    script "/etc/keepalived/check_nginx.sh"  # 检查Nginx是否存活
    interval 2
    weight -20
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1234
    }
    virtual_ipaddress {
        10.0.0.100/24
    }
    track_script {
        chk_nginx
    }
}

# 检查脚本
#!/bin/bash
# /etc/keepalived/check_nginx.sh
curl -f http://localhost/health || exit 1
```

### 2. 微服务网关配置

```nginx
# 微服务API网关完整配置

upstream user_service {
    least_conn;
    server 10.0.1.10:8080 weight=5 max_fails=3 fail_timeout=30s;
    server 10.0.1.11:8080 weight=5 max_fails=3 fail_timeout=30s;
    server 10.0.1.12:8080 backup;
    keepalive 32;  # 长连接数
}

upstream order_service {
    least_conn;
    server 10.0.1.20:8080 weight=5 max_fails=3 fail_timeout=30s;
    server 10.0.1.21:8080 weight=5 max_fails=3 fail_timeout=30s;
    keepalive 32;
}

# 限流区域定义
limit_req_zone $binary_remote_addr zone=req_limit:10m rate=100r/s;
limit_conn_zone $binary_remote_addr zone=conn_limit:10m;

server {
    listen 80;
    server_name api.example.com;
    
    # 日志格式（JSON，便于ELK收集）
    log_format json_analytics escape=json '{'
        '"time":"$time_iso8601",'
        '"remote_addr":"$remote_addr",'
        '"request":"$request",'
        '"status":$status,'
        '"request_time":$request_time,'
        '"upstream_time":"$upstream_response_time",'
        '"body_bytes_sent":$body_bytes_sent'
    '}';
    
    access_log /var/log/nginx/api-access.log json_analytics;
    
    # 全局限流
    limit_req zone=req_limit burst=200 nodelay;
    limit_conn conn_limit 50;
    
    # 错误页面
    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
        root /usr/share/nginx/html;
    }
    
    # 用户服务路由
    location /api/users/ {
        # 重写去掉/api前缀
        rewrite ^/api/users/(.*) /$1 break;
        
        proxy_pass http://user_service;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Request-ID $request_id;
        
        # 超时配置
        proxy_connect_timeout 5s;
        proxy_send_timeout 10s;
        proxy_read_timeout 30s;
        
        # 缓冲区
        proxy_buffering on;
        proxy_buffer_size 4k;
        proxy_buffers 8 4k;
    }
    
    # 订单服务路由
    location /api/orders/ {
        rewrite ^/api/orders/(.*) /$1 break;
        proxy_pass http://order_service;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Request-ID $request_id;
        
        proxy_connect_timeout 5s;
        proxy_send_timeout 10s;
        proxy_read_timeout 60s;  # 订单处理可能较慢
    }
    
    # 静态资源（Swagger UI等）
    location /swagger/ {
        alias /data/swagger/;
        expires 1h;
    }
}
```

### 3. 多级缓存架构

```
多级缓存架构设计：

客户端 ──► CDN（边缘缓存）──► Nginx（本地缓存）──► 应用服务器 ──► 数据库
              │                    │
              │                    │
         静态资源缓存          proxy_cache
         (图片/CSS/JS)       (动态内容缓存)

缓存层级：
1. 浏览器缓存（Cache-Control）
2. CDN缓存（阿里云/腾讯云/CloudFlare）
3. Nginx代理缓存（proxy_cache）
4. 应用本地缓存（Caffeine/Guava Cache）
5. Redis分布式缓存
6. 数据库
```

**Nginx代理缓存配置**：
```nginx
# 定义缓存区域
proxy_cache_path /data/nginx/cache levels=1:2 keys_zone=api_cache:100m 
                 max_size=10g inactive=60m use_temp_path=off;

server {
    location /api/ {
        proxy_cache api_cache;
        proxy_cache_valid 200 302 10m;   # 200和302缓存10分钟
        proxy_cache_valid 404 1m;        # 404缓存1分钟
        proxy_cache_use_stale error timeout updating http_500 http_502;
        
        # 缓存Key定义
        proxy_cache_key "$scheme$request_method$host$request_uri";
        
        # 添加缓存状态头（调试用）
        add_header X-Cache-Status $upstream_cache_status;
        
        proxy_pass http://backend;
        
        # 不缓存的条件
        proxy_cache_bypass $http_cache_control;  # 客户端要求不缓存
        proxy_no_cache $http_pragma;             # Pragma: no-cache
    }
}

# 缓存状态说明：
# HIT      - 缓存命中
# MISS     - 缓存未命中
# EXPIRED  - 缓存过期
# UPDATING - 缓存正在更新
# STALE    - 返回过期缓存（后端异常时）
```

### 4. 安全加固配置

```nginx
server {
    listen 443 ssl http2;
    server_name www.example.com;
    
    # SSL证书
    ssl_certificate /etc/nginx/ssl/example.crt;
    ssl_certificate_key /etc/nginx/ssl/example.key;
    
    # SSL安全优化
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384';
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:50m;
    ssl_session_timeout 1d;
    ssl_session_tickets off;
    
    # HSTS（强制HTTPS）
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
    
    # 安全响应头
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'" always;
    
    # 防止点击劫持
    location / {
        # ...
    }
}

# 隐藏Nginx版本号
http {
    server_tokens off;
}

# 限制请求体大小（防DOS）
client_max_body_size 10m;

# 限制缓冲区（防内存溢出）
client_body_buffer_size 128k;
client_header_buffer_size 1k;
large_client_header_buffers 4 4k;
```

---

## Nginx vs Apache vs Caddy：架构对比

### 1. 架构模型对比

| 维度 | Nginx | Apache | Caddy |
|------|-------|--------|-------|
| **进程模型** | Master-Worker多进程 | Prefork/Worker/Event多进程/多线程 | 单进程多线程 |
| **I/O模型** | 事件驱动（epoll/kqueue） | 阻塞（Prefork）/事件（Event MPM） | 事件驱动（Go netpoll） |
| **并发能力** | 10万+ | 几千（Prefork）/几万（Event） | 几万 |
| **内存占用** | 低（连接不占用线程栈） | 高（每个连接一个线程/进程） | 中等 |
| **配置语法** | 专用DSL | 专用DSL（更复杂） | Caddyfile（简洁） |
| **动态模块** | 需编译 | 支持动态加载 | 静态编译 |
| **自动HTTPS** | 需手动配置 | 需手动配置 | 内置自动HTTPS |
| **适用场景** | 反向代理、静态资源、高并发 | 动态内容（PHP）、.htaccess | 快速部署、自动HTTPS |

### 2. 并发模型深度对比

```
Nginx（事件驱动）：

进程：Worker1 ──► epoll_wait ──► 处理就绪连接 ──► 回到epoll_wait
         │                              │
         └──────── 单线程循环 ──────────┘

并发数 = 内存 / 每个连接开销（约几百字节）
10万并发 ≈ 几百MB内存

Apache Prefork（进程模型）：

进程：Process1 ──► accept ──► read（阻塞）──► 处理 ──► write（阻塞）──► close
进程：Process2 ──► accept ──► read（阻塞）──► 处理 ──► write（阻塞）──► close
...

并发数 = 内存 / 每个进程开销（几MB）
1万并发 ≈ 几十GB内存

Apache Event MPM（混合模型）：

主线程：epoll_wait ──► 新连接 ──► 分配给工作线程
工作线程：read（阻塞）──► 处理 ──► write（阻塞）

并发数比Prefork高，但仍受限于线程数
```

### 3. 性能对比数据

```
基准测试场景：1000并发，持续30秒，请求4KB静态文件

┌────────────┬─────────────┬─────────────┬──────────────┐
│   服务器    │  请求/秒     │  平均延迟    │   内存占用    │
├────────────┼─────────────┼─────────────┼──────────────┤
│ Nginx      │  ~120,000   │   ~8ms      │    ~50MB     │
│ Apache(P)  │   ~8,000    │  ~125ms     │   ~2GB       │
│ Apache(E)  │  ~40,000    │  ~25ms      │   ~500MB     │
│ Caddy      │  ~60,000    │  ~16ms      │   ~200MB     │
└────────────┴─────────────┴─────────────┴──────────────┘

结论：
- Nginx在静态文件和高并发场景优势明显
- Apache适合动态内容（尤其是PHP，mod_php性能极佳）
- Caddy在易用性和自动HTTPS方面最优
```

---

## 性能分析与调优

### 1. 内核参数调优

```bash
# /etc/sysctl.conf

# TCP连接优化
net.ipv4.tcp_max_tw_buckets = 6000          # TIME_WAIT状态连接数上限
net.ipv4.tcp_sack = 1                       # 启用选择性确认
net.ipv4.tcp_window_scaling = 1             # 启用窗口缩放
net.ipv4.tcp_rmem = 4096 87380 4194304      # TCP读缓冲区
net.ipv4.tcp_wmem = 4096 16384 4194304      # TCP写缓冲区
net.core.wmem_default = 8388608             # 默认发送缓冲区
net.core.rmem_default = 8388608             # 默认接收缓冲区
net.core.rmem_max = 16777216                # 最大接收缓冲区
net.core.wmem_max = 16777216                # 最大发送缓冲区

# 连接跟踪优化（如果使用iptables）
net.netfilter.nf_conntrack_max = 655350     # 最大连接跟踪数

# 文件描述符
fs.file-max = 2097152                       # 系统最大文件句柄数
fs.nr_open = 2097152                        # 单进程最大文件句柄数

# 应用后执行
sysctl -p
```

### 2. Nginx性能调优配置

```nginx
worker_processes auto;           # 自动匹配CPU核心数
worker_rlimit_nofile 65535;      # Worker进程最大文件句柄数

events {
    worker_connections 65535;    # 每个Worker最大连接数
    use epoll;                   # Linux使用epoll
    multi_accept on;             # 尽可能接受更多连接
}

http {
    # 高效文件传输
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    
    # 连接保持
    keepalive_timeout 65;
    keepalive_requests 1000;     # 单个连接最大请求数
    
    # 压缩
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css text/xml application/json 
               application/javascript application/rss+xml 
               application/atom+xml image/svg+xml;
    
    # 打开文件缓存（提升静态文件性能）
    open_file_cache max=1000 inactive=20s;
    open_file_cache_valid 30s;
    open_file_cache_min_uses 2;
    open_file_cache_errors on;
    
    # 上游长连接
    upstream backend {
        server 127.0.0.1:8080;
        keepalive 100;           # 保持100个长连接
    }
    
    server {
        location / {
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            proxy_pass http://backend;
        }
    }
}
```

### 3. 压测与性能监控

```bash
# 使用wrk进行压测
wrk -t12 -c400 -d30s http://localhost/index.html

# 参数说明：
# -t12: 12个线程
# -c400: 400个并发连接
# -d30s: 持续30秒

# 输出示例：
# Running 30s test @ http://localhost/index.html
#   12 threads and 400 connections
#   Thread Stats   Avg      Stdev     Max   +/- Stdev
#     Latency     8.23ms   12.45ms 234.56ms   98.12%
#     Req/Sec     4.12k   512.34     5.67k    78.45%
#   1234567 requests in 30.01s, 4.56GB read
# Requests/sec:  41138.12
# Transfer/sec:    155.78MB

# Nginx状态监控模块
server {
    location /nginx_status {
        stub_status on;
        access_log off;
        allow 10.0.0.0/8;
        deny all;
    }
}

# 状态页输出：
# Active connections: 291           # 当前活跃连接数
# server accepts handled requests   # 总接受/处理/请求数
#  16630948 16630948 31070465
# Reading: 6 Writing: 125 Waiting: 160  # 读/写/等待状态连接数
```

---

## 常见陷阱与最佳实践

### 1. Location匹配顺序陷阱

**陷阱**：认为location按书写顺序匹配，导致配置不生效。

**真相**：location匹配有固定优先级，不按书写顺序。

```nginx
# 错误配置示例：
location /api/ {
    proxy_pass http://api_backend;
}

location ~ ^/api/v1/special {
    # 用户期望这个匹配 /api/v1/special
    # 但实际上 /api/v1/special 已经被上面的前缀匹配了！
    proxy_pass http://special_backend;
}

# 正确做法：将更精确的正则放在前面，或使用 ^~
location ~ ^/api/v1/special {
    proxy_pass http://special_backend;
}

location /api/ {
    proxy_pass http://api_backend;
}

# 或者使用 ^~ 阻止正则匹配
location ^~ /api/ {
    proxy_pass http://api_backend;
}
```

### 2. 反向代理丢失客户端IP

**陷阱**：后端服务获取的IP都是Nginx的IP，无法做限流、审计。

**最佳实践**：
```nginx
location / {
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    
    proxy_pass http://backend;
}
```

### 3. 缓冲区配置不当导致性能问题

**陷阱**：proxy_buffering关闭或缓冲区太小，导致大量后端连接占用。

```nginx
# 默认proxy_buffering是on，通常应该保持开启
location / {
    proxy_buffering on;              # 启用缓冲
    proxy_buffer_size 4k;            # 响应头缓冲区
    proxy_buffers 8 4k;              # 8个4KB缓冲区
    proxy_busy_buffers_size 8k;      # 忙缓冲区大小
    proxy_temp_file_write_size 64k;  # 临时文件写入大小
    
    proxy_pass http://backend;
}

# 大数据流场景（如下载、视频）应关闭缓冲
location /download/ {
    proxy_buffering off;             # 关闭缓冲，直接流式传输
    proxy_pass http://file_server;
}
```

### 4. 健康检查缺失

**陷阱**：后端服务宕机后，Nginx仍向故障节点转发请求。

**最佳实践**：
```nginx
upstream backend {
    server 192.168.1.1:8080 max_fails=3 fail_timeout=30s;
    server 192.168.1.2:8080 max_fails=3 fail_timeout=30s;
    server 192.168.1.3:8080 backup;  # 备用服务器
    
    # 主动健康检查（需要nginx_upstream_check_module）
    check interval=3000 rise=2 fall=3 timeout=1000 type=http;
    check_http_send "GET /health HTTP/1.0\r\n\r\n";
    check_http_expect_alive http_2xx http_3xx;
}
```

### 5. 未限制请求体大小导致OOM

**陷阱**：用户上传大文件导致Nginx内存占用过高。

```nginx
# 全局限制
client_max_body_size 50m;        # 最大请求体50MB
client_body_buffer_size 128k;    # 请求体缓冲区
client_header_buffer_size 1k;    # 请求头缓冲区
large_client_header_buffers 4 4k; # 大请求头缓冲区

# 特定location放宽限制
location /upload/ {
    client_max_body_size 500m;
    proxy_pass http://upload_server;
}
```

### 6. SSL证书配置陷阱

```nginx
# 错误：缺少中间证书
ssl_certificate /etc/nginx/ssl/server.crt;  # 只包含服务器证书

# 正确：证书链必须完整
ssl_certificate /etc/nginx/ssl/fullchain.crt;  # 服务器证书 + 中间证书
ssl_certificate_key /etc/nginx/ssl/server.key;

# 错误：允许不安全的SSL协议
ssl_protocols SSLv3 TLSv1 TLSv1.1 TLSv1.2;  # SSLv3和TLSv1有漏洞

# 正确：只启用安全协议
ssl_protocols TLSv1.2 TLSv1.3;

# 错误：弱密码套件
ssl_ciphers 'ALL:!ADH:!EXPORT56:RC4+RSA:+HIGH:+MEDIUM:+LOW:+SSLv2:+EXP';

# 正确：强密码套件
ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256';
ssl_prefer_server_ciphers on;
```

---

## 面试题与参考答案

### Q1：Nginx为什么能支撑10万并发？从操作系统层面分析。

**参考答案**：

```
根本原因：事件驱动的异步非阻塞I/O架构

1. 多路复用epoll：
   - 一个Worker进程通过epoll监控数万个连接
   - 只有就绪的连接才进行读写，没有空轮询
   - epoll时间复杂度O(1)，不受连接数影响

2. 无阻塞设计：
   - 没有线程/进程阻塞在I/O等待上
   - 所有等待都在epoll_wait中统一处理
   - 连接不占用线程栈（一个线程栈通常2-8MB）

3. 零拷贝sendfile：
   - 静态文件传输不经过用户态
   - 减少2次CPU拷贝和2次上下文切换

4. 内存池机制：
   - 减少内存碎片和系统调用
   - 请求结束统一释放，无内存泄漏

5. 多进程单线程：
   - Worker间无共享状态，无锁竞争
   - 利用多核CPU，每个Worker绑定一个核心

对比：Apache Prefork每个连接一个进程（2-10MB）
     10万并发需要200GB-1TB内存
     Nginx 10万并发只需几百MB内存
```

### Q2：Nginx的Master-Worker进程模型有什么优势？为什么选择多进程单线程而非多线程？

**参考答案**：

```
Master-Worker模型的优势：

1. 职责分离：
   - Master：管理配置、信号处理、Worker生命周期
   - Worker：专注处理客户端请求
   - 管理逻辑与业务逻辑解耦

2. 热部署能力：
   - 修改配置后，Master创建新Worker加载新配置
   - 旧Worker处理完现有请求后优雅退出
   - 服务不中断

3. 容错性：
   - 单个Worker崩溃不影响其他Worker
   - Master立即fork新Worker补充

选择多进程单线程的原因：

1. 无锁编程（最重要）：
   - 每个Worker独立处理连接，无需锁竞争
   - 避免了多线程的复杂同步问题

2. 隔离性：
   - 进程有独立地址空间，一个进程崩溃不影响其他进程
   - 线程共享地址空间，一个线程崩溃可能拖垮整个进程

3. 简单性：
   - 单线程代码无需考虑线程安全
   - 没有死锁、竞态条件等问题

4. 多核利用：
   - 每个Worker绑定一个CPU核心
   - 避免线程切换和缓存失效

代价：进程间通信需要通过IPC（如共享内存），比线程间通信复杂
```

### Q3：Nginx的location匹配优先级是什么？如果配置冲突如何解决？

**参考答案**：

```
Location匹配优先级（从高到低）：

1. = 精确匹配：
   location = /exact {
       # 只有URI完全等于/exact才匹配
       # O(1)哈希查找，性能最高
   }

2. ^~ 优先前缀匹配：
   location ^~ /static {
       # 最长前缀匹配成功，停止检查正则
       # 用于性能优化（跳过正则匹配）
   }

3. ~ 区分大小写正则匹配：
   location ~ \.php$ {
       # 按配置文件中的定义顺序匹配
       # 第一个匹配的正则生效
   }

4. ~* 不区分大小写正则匹配：
   location ~* \.(jpg|png)$ {
       # 与~同级，按顺序匹配
   }

5. 无修饰符前缀匹配（最长前缀）：
   location /api {
       # 最低优先级
   }

冲突解决原则：
- 精确匹配 > 优先前缀 > 正则 > 普通前缀
- 同级正则按配置文件书写顺序，第一个匹配的生效
- 建议：将更精确的location放在前面，避免依赖顺序
```

### Q4：Nginx的负载均衡策略有哪些？平滑加权轮询算法是如何工作的？

**参考答案**：

```
Nginx支持的负载均衡策略：

1. 轮询（round_robin）：默认，按顺序分配
2. 加权轮询（weight）：按权重比例分配
3. IP哈希（ip_hash）：同一IP固定路由（会话保持）
4. 最少连接（least_conn）：分配到当前连接数最少的服务器
5. 一致性哈希（hash $key consistent）：缓存友好

平滑加权轮询算法（Nginx SW算法）：

每个服务器维护两个权重：
- weight：配置权重（固定）
- current_weight：当前权重（动态变化）

算法步骤：
1. 每个服务器 current_weight += weight
2. 选择 current_weight 最大的服务器
3. 被选中的服务器 current_weight -= total_weight

示例：
服务器A weight=5，服务器B weight=1，total=6

轮次1: A(0+5=5), B(0+1=1) -> 选A -> A=5-6=-1
轮次2: A(-1+5=4), B(1+1=2) -> 选A -> A=4-6=-2
轮次3: A(-2+5=3), B(2+1=3) -> 选A -> A=3-6=-3
轮次4: A(-3+5=2), B(3+1=4) -> 选B -> B=4-6=-2
轮次5: A(2+5=7), B(-2+1=-1) -> 选A -> A=7-6=1
轮次6: A(1+5=6), B(-1+1=0) -> 选A -> A=6-6=0

结果：6轮中A被选5次，B被选1次，分布均匀
```

### Q5：Nginx的限流是如何实现的？漏桶算法和令牌桶算法有什么区别？

**参考答案**：

```
Nginx限流模块：

1. limit_conn（连接数限制）：
   - 限制单个IP的并发连接数
   - 基于共享内存计数器
   
   limit_conn_zone $binary_remote_addr zone=addr:10m;
   limit_conn addr 10;  # 单个IP最多10个并发连接

2. limit_req（请求速率限制）：
   - 基于漏桶算法（Leaky Bucket）
   - 限制请求处理速率
   
   limit_req_zone $binary_remote_addr zone=one:10m rate=10r/s;
   limit_req zone=one burst=20 nodelay;

漏桶算法 vs 令牌桶算法：

漏桶算法（Nginx limit_req使用）：
- 请求像水一样流入桶，桶以固定速率漏水
- 桶满了，新请求被丢弃（或延迟）
- 特点：强制匀速处理，不允许突发流量

令牌桶算法：
- 系统以固定速率生成令牌放入桶中
- 请求需要获取令牌才能执行
- 桶满了，令牌丢弃（不丢弃请求）
- 特点：允许一定程度的突发流量（桶中有令牌时）

Nginx limit_req参数：
- rate=10r/s：每秒处理10个请求（漏桶漏水速率）
- burst=20：桶容量20，允许突发20个请求排队
- nodelay：突发请求不延迟，立即处理（但会消耗令牌）
```

### Q6：什么是Nginx的HTTP处理阶段（Phase）？Rewrite指令在哪个阶段执行？

**参考答案**：

```
Nginx HTTP处理的11个阶段：

1. POST_READ        - 读取请求头后（realip模块）
2. SERVER_REWRITE   - server块rewrite
3. FIND_CONFIG      - 查找location（核心）
4. REWRITE          - location块rewrite
5. POST_REWRITE     - rewrite后检查（防死循环）
6. PREACCESS        - 访问控制前（limit_req/conn）
7. ACCESS           - 访问控制（auth_basic, access）
8. POST_ACCESS      - 访问控制后检查
9. TRY_FILES        - try_files指令
10. CONTENT         - 生成内容（proxy_pass, static）
11. LOG             - 记录日志

Rewrite指令执行阶段：
- server块中的rewrite：在SERVER_REWRITE阶段（阶段2）
- location块中的rewrite：在REWRITE阶段（阶段4）
- 如果rewrite有break/last标志，可能跳过后续阶段

重要：rewrite可以跳转到其他location，但只在阶段2和4执行
      proxy_pass在CONTENT阶段（阶段10）执行
      因此rewrite先于proxy_pass执行
```

### Q7：Nginx如何实现热部署（平滑升级）？

**参考答案**：

```
Nginx热部署流程（不中断服务升级）：

1. 替换Nginx二进制文件：
   cp nginx-new /usr/local/nginx/sbin/nginx

2. 发送USR2信号给旧Master：
   kill -USR2 $(cat /var/run/nginx.pid)
   
   旧Master响应：
   - 重命名pid文件：nginx.pid -> nginx.pid.oldbin
   - fork出新Master（使用新二进制）
   - 新Master创建新Worker
   - 此时有两个Master+多组Worker在运行

3. 发送WINCH信号给旧Master：
   kill -WINCH $(cat /var/run/nginx.pid.oldbin)
   
   旧Master响应：
   - 优雅关闭旧Worker（不再接受新连接）
   - 旧Worker处理完现有请求后退出
   - 旧Master保留，用于回滚

4. 验证新Master工作正常：
   - 测试新Worker是否正常工作
   - 如果异常，可以回滚（发送HUP给旧Master）

5. 发送QUIT信号给旧Master：
   kill -QUIT $(cat /var/run/nginx.pid.oldbin)
   
   旧Master退出，完成升级

回滚（如果新Master有问题）：
   kill -HUP $(cat /var/run/nginx.pid.oldbin)  # 重新启动旧Worker
   kill -QUIT $(cat /var/run/nginx.pid)         # 关闭新Master

整个过程中，旧Worker处理完现有请求才退出，
新连接由新Worker处理，服务不中断。
```

### Q8：Nginx反向代理时，如何处理后端返回的大响应？proxy_buffering的作用是什么？

**参考答案**：

```
proxy_buffering机制：

当proxy_buffering on（默认）：
1. Nginx接收后端响应，存储在内存缓冲区
2. 如果响应超过缓冲区大小，写入临时文件
3. 响应完全接收后（或缓冲区满），再发送给客户端

优势：
- 后端连接快速释放（不需要等待客户端接收）
- 后端可以处理更多请求
- 支持后端慢、客户端快的场景

劣势：
- 增加内存/磁盘使用
- 客户端看到响应有延迟（等缓冲）

当proxy_buffering off：
- Nginx接收到后端数据立即转发给客户端
- 后端连接保持直到客户端接收完成
- 适合：视频流、大文件下载、实时数据

配置示例：
location / {
    proxy_buffering on;
    proxy_buffer_size 4k;        # 响应头缓冲区
    proxy_buffers 8 4k;          # 8个4KB缓冲区 = 32KB
    proxy_busy_buffers_size 8k;  # 忙缓冲区
    proxy_max_temp_file_size 1024m;  # 临时文件最大1GB
    
    proxy_pass http://backend;
}

location /stream/ {
    proxy_buffering off;         # 流式传输
    proxy_pass http://stream_backend;
}

proxy_cache vs proxy_buffering：
- proxy_buffering：缓解后端压力，不缓存响应
- proxy_cache：缓存响应，供后续请求使用
```

### Q9：Nginx中try_files指令的作用是什么？与rewrite有什么区别？

**参考答案**：

```
try_files指令：
- 按顺序尝试多个文件/URI，直到找到存在的
- 如果都不存在，返回最后一个参数（通常是状态码或URI）

语法：try_files file ... uri 或 try_files file ... =code

示例1（前端History模式路由）：
location / {
    try_files $uri $uri/ /index.html;
    # 1. 尝试 $uri（精确文件）
    # 2. 尝试 $uri/（目录）
    # 3. 返回 /index.html（前端路由处理）
}

示例2（自定义404）：
location /images/ {
    try_files $uri $uri/ /images/default.jpg;
}

示例3（直接返回状态码）：
location /private/ {
    try_files $uri =403;  # 如果文件不存在，返回403
}

try_files vs rewrite区别：

1. 执行阶段：
   - try_files：TRY_FILES阶段（阶段9）
   - rewrite：REWRITE阶段（阶段2/4）

2. 行为：
   - try_files：按顺序尝试文件存在性，内部重定向
   - rewrite：URI重写，可以外部重定向（return 301/302）

3. 性能：
   - try_files：内部重定向，客户端无感知
   - rewrite外部重定向：增加RTT

4. 使用场景：
   - try_files：文件存在性检查、SPA路由、默认页面
   - rewrite：URI规范化、伪静态、跳转
```

### Q10：如何排查Nginx 499错误？什么是499状态码？

**参考答案**：

```
499状态码：
- Nginx自定义状态码，非HTTP标准
- 含义：客户端在服务器返回响应前关闭了连接

常见场景：
1. 客户端超时：
   - 客户端设置的超时时间 < 后端处理时间
   - 客户端主动断开连接

2. 用户取消请求：
   - 浏览器中关闭页面
   - 移动APP切换页面取消请求

3. 负载均衡器超时：
   - LB（如AWS ALB）等待Nginx响应超时
   - LB断开连接，Nginx记录499

排查方法：

1. 检查Nginx日志：
   log_format 包含 $status 和 $request_time
   如果499的请求request_time很大，说明后端处理慢

2. 对比后端日志：
   - 如果后端返回200，但Nginx记录499
   - 说明后端处理太慢，客户端等不及了

3. 检查客户端超时配置：
   - 浏览器fetch/XHR的timeout
   - 移动APP的网络请求超时

4. 调整Nginx超时：
   proxy_read_timeout 60s;  # 增加读取超时
   
   但更好的做法是优化后端性能，而不是延长超时

5. 使用proxy_ignore_client_abort：
   location / {
       proxy_ignore_client_abort on;  # 客户端断开后继续请求后端
       proxy_pass http://backend;
   }
   
   注意：这会导致后端做无用功，谨慎使用

解决方案：
- 优化后端接口性能
- 前端增加loading状态，避免用户重复点击
- 对于非关键请求，客户端增加重试机制
- 使用异步处理（前端轮询或WebSocket推送）
```

---

*此文原创，转载请注明出处。*
