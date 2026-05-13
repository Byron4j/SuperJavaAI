# Gateway深度解析：Reactive网关的核心原理与源码剖析

**文章标签：** #java #springcloud #gateway #网关 #webflux #响应式编程 #源码分析 #面试 #reactor

## 目录

- [引言：API网关的本质与演进](#引言api网关的本质与演进)
- [理论基础：响应式编程与Netty架构](#理论基础响应式编程与netty架构)
- [演进史：从Zuul到Spring Cloud Gateway](#演进史从zuul到spring-cloud-gateway)
- [源码深度分析：路由与断言机制](#源码深度分析路由与断言机制)
- [源码深度分析：过滤器链与执行流程](#源码深度分析过滤器链与执行流程)
- [源码深度分析：动态路由与限流熔断](#源码深度分析动态路由与限流熔断)
- [实战案例：生产级网关配置](#实战案例生产级网关配置)
- [对比分析：Gateway vs Zuul vs Kong vs Nginx](#对比分析gateway-vs-zuul-vs-kong-vs-nginx)
- [性能分析：WebFlux vs Servlet的吞吐量对比](#性能分析webflux-vs-servlet的吞吐量对比)
- [常见陷阱与最佳实践](#常见陷阱与最佳实践)
- [面试题与参考答案](#面试题与参考答案)

---

## 引言：API网关的本质与演进

在微服务架构中，客户端直接调用多个后端服务会导致**客户端复杂、认证重复、跨域问题、协议转换困难**等诸多问题。API网关作为**统一入口**，将所有客户端请求路由到相应的后端服务，并提供横切关注点（cross-cutting concerns）的统一处理。

### 核心问题域

```
┌─────────────────────────────────────────────────────────────┐
│                微服务架构中的网关需求                          │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│   客户端         统一入口         后端服务                      │
│  (Web/App)       API Gateway     (Microservices)             │
│     │                │                │                       │
│     │  1.请求        │  2.路由        │                       │
│     │───────────────>│───────────────>│ 用户服务              │
│     │                │                │                       │
│     │                │  3.鉴权        │                       │
│     │                │  4.限流        │ 订单服务              │
│     │                │  5.日志        │                       │
│     │                │                │ 库存服务              │
│     │                │                │                       │
│     │                │  6.聚合        │ 支付服务              │
│     │<───────────────│<───────────────│                       │
│                                                               │
│ 网关解决的横切关注点：                                          │
│ - 统一认证授权（JWT/OAuth2）                                    │
│ - 请求/响应转换（协议转换、格式转换）                            │
│ - 流量控制（限流、熔断、降级）                                   │
│ - 请求路由（路径、Header、参数匹配）                             │
│ - 灰度发布（A/B测试、金丝雀）                                    │
│ - 日志与监控（请求追踪、指标收集）                               │
│ - 缓存（响应缓存、请求去重）                                     │
└─────────────────────────────────────────────────────────────┘
```

### Spring Cloud Gateway的定位

Spring Cloud Gateway是Spring官方推出的**新一代API网关**，基于Spring 5、Project Reactor和Spring Boot 2构建，使用Netty处理请求，目标是替代Netflix Zuul 1.x。

```
Spring Cloud Gateway核心架构：

┌─────────────────────────────────────────────────────────────┐
│                    Gateway核心组件                             │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  Route（路由）：网关的基本映射单元                              │
│  ├── id: 路由唯一标识                                          │
│  ├── uri: 目标地址（lb://service-name 或 http://host:port）    │
│  ├── predicates: 断言集合（匹配条件）                           │
│  └── filters: 过滤器集合（请求/响应处理）                        │
│                                                               │
│  Predicate（断言）：决定请求是否匹配该路由                       │
│  ├── Path=/api/user/**                                        │
│  ├── Method=GET                                               │
│  ├── Header=X-Request-Id, \d+                                 │
│  └── Query=version, v1                                        │
│                                                               │
│  Filter（过滤器）：对请求或响应进行处理                          │
│  ├── GatewayFilter: 作用于单个路由                              │
│  │   ├── StripPrefix=1  （去掉路径前缀）                        │
│  │   ├── AddRequestHeader=X-User-Id,123                       │
│  │   ├── RewritePath=/old/(?<segment>.*), /new/${segment}     │
│  │   └── RequestRateLimiter=...  （限流）                       │
│  │                                                            │
│  └── GlobalFilter: 作用于所有路由                               │
│      ├── 鉴权过滤器（JWT验证）                                   │
│      ├── 日志过滤器（请求日志）                                  │
│      └── 跨域过滤器（CORS处理）                                  │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

**关键认知**：Gateway不是简单的"HTTP反向代理"，它是一个**基于响应式编程模型、支持函数式路由定义、内置丰富过滤器生态的编程式网关**。理解Gateway需要掌握Reactor、Netty、路由匹配算法、过滤器链等核心概念。

---

## 理论基础：响应式编程与Netty架构

### 1. 响应式编程（Reactive Programming）

```
响应式编程核心概念：

传统编程（阻塞式）：
  请求1 ──> 处理 ──> 响应
  请求2 ──> 等待请求1完成 ──> 处理 ──> 响应
  
  特点：一个线程处理一个请求，线程数随并发增加

响应式编程（非阻塞）：
  请求1 ──> 提交任务 ──> 线程返回EventLoop
  请求2 ──> 提交任务 ──> 线程返回EventLoop
  请求3 ──> 提交任务 ───> 线程返回EventLoop
  
  任务完成 ──> EventLoop回调 ──> 发送响应
  
  特点：少量线程处理大量请求（事件驱动）

Reactive Streams规范：
  Publisher（发布者）──subscribe──> Subscriber（订阅者）
         │                              │
         │  onSubscribe(Subscription)   │
         │  onNext(T)                   │
         │  onError(Throwable)          │
         │  onComplete()                │
         │                              │
         │◄──────request(n)─────────────│
```

### 2. Project Reactor核心组件

```java
// Mono: 0或1个元素的异步序列
Mono<String> mono = Mono.just("hello");  // 同步创建
Mono<String> mono2 = Mono.fromCallable(() -> fetchData()); // 异步创建

// Flux: 0到N个元素的异步序列
Flux<Integer> flux = Flux.range(1, 10);  // 1到10
Flux<String> flux2 = Flux.fromIterable(list); // 从集合创建

// 操作符
Mono<String> result = mono
    .map(String::toUpperCase)       // 转换
    .flatMap(this::fetchDetails)    // 异步转换
    .filter(s -> s.length() > 5)    // 过滤
    .doOnNext(s -> log.info(s))     // 副作用
    .timeout(Duration.ofSeconds(5)) // 超时
    .onErrorReturn("default");      // 错误处理
```

### 3. Netty架构

```
Netty核心架构：

┌─────────────────────────────────────────────────────────────┐
│                     Netty Server                             │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  EventLoopGroup (Boss)          EventLoopGroup (Worker)      │
│  ┌─────────────┐                ┌─────────────┐              │
│  │ EventLoop   │                │ EventLoop   │              │
│  │  (1个线程)   │                │  (N个线程)   │              │
│  │             │                │             │              │
│  │ 监听端口     │──分配连接──────>│ 处理I/O     │              │
│  │ 接收连接     │                │ 读写数据     │              │
│  └─────────────┘                └─────────────┘              │
│                                                               │
│  ChannelPipeline（每个连接一个）                                │
│  ┌─────────────────────────────────────────────────────┐     │
│  │ HeadContext ──> HttpServerCodec ──> HttpObject    │     │
│  │ Decoder ──> GatewayFilterChain ──> NettyRouting    │     │
│  │ Filter ──> NettyWriteResponseFilter ──> TailContext │     │
│  └─────────────────────────────────────────────────────┘     │
│                                                               │
│  特点：                                                        │
│  - Boss线程只负责接收连接                                       │
│  - Worker线程负责I/O读写（默认CPU核数*2）                        │
│  - ChannelPipeline处理请求编解码和业务逻辑                        │
│  - 所有I/O操作都是异步非阻塞的                                   │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

### 4. WebFlux请求处理模型

```
Spring WebFlux请求处理流程：

Client Request
     │
     ▼
Netty Server (EventLoop)
     │
     ▼
HttpServerCodec (解码HTTP请求)
     │
     ▼
HttpWebHandlerAdapter (适配为ServerWebExchange)
     │
     ▼
DispatcherHandler (分发请求)
     │
     ├──> HandlerMapping (路由匹配)
     │       ├──> RoutePredicateHandlerMapping (Gateway路由)
     │       └──> ...
     │
     ▼
HandlerAdapter (调用处理器)
     │
     ▼
FilteringWebHandler (执行过滤器链)
     │
     ├──> GlobalFilter 1 (Ordered)
     ├──> GlobalFilter 2
     ├──> GatewayFilter (路由特定)
     │
     ▼
NettyRoutingFilter (发送代理请求)
     │
     ▼
NettyWriteResponseFilter (写回响应)
     │
     ▼
Client Response
```

---

## 演进史：从Zuul到Spring Cloud Gateway

### 第一阶段：Netflix Zuul 1.x（2013-2018）

```
Zuul 1.x架构：

Client ──> Zuul Servlet ──> ZuulFilter Chain ──> Ribbon ──> Service
                  │
                  └── 基于Servlet API（阻塞IO）

核心特性：
- 基于Servlet 3.0（同步阻塞）
- 使用Tomcat/Jetty容器
- 过滤器链（pre/route/post/error类型）
- 与Ribbon/Eureka深度集成

性能瓶颈：
- 每个请求占用一个线程
- 线程上下文切换开销大
- 不支持长连接（WebSocket）
```

### 第二阶段：Netflix Zuul 2.x（2016-2018）

```
Zuul 2.x架构：

Client ──> Netty Server ──> Inbound Filters ──> Outbound Filters ──> Service
                  │
                  └── 基于Netty（非阻塞IO）

改进：
- 使用Netty替代Servlet容器
- 异步非阻塞处理
- 支持长连接（WebSocket）
- 更好的性能

问题：
- 基于Groovy语言，与Spring生态兼容性差
- 配置复杂
- Netflix未将其集成到Spring Cloud
```

### 第三阶段：Spring Cloud Gateway诞生（2018-至今）

```
Spring Cloud Gateway里程碑：

Spring Cloud Finchley (2018.06)：
- Gateway首个正式发布版
- 基于Spring 5 + Project Reactor + Netty
- 提供路由、断言、过滤器三大核心概念
- 支持动态路由

Spring Cloud Greenwich (2019.01)：
- 支持Kotlin协程
- 改进的指标监控
- 更好的错误处理

Spring Cloud Hoxton (2019.11)：
- 支持RSocket
- 改进的熔断集成
- 性能优化

Spring Cloud 2020.0 (2020.12)：
- 默认使用Spring Cloud LoadBalancer（替代Ribbon）
- 支持Spring Boot 2.4+
- 改进的Kubernetes集成

Spring Cloud 2021.x/2022.x/2023.x：
- 更好的原生镜像支持（GraalVM）
- 支持HTTP/3（实验性）
- 性能持续优化
```

### 第四阶段：当前状态（2024-2026）

```
Gateway当前生态：

核心项目：
- spring-cloud-gateway-server：核心网关功能
- spring-cloud-gateway-webflux：WebFlux集成
- spring-cloud-gateway-mvc：Spring MVC支持（新）

竞争产品：
- Kong：基于OpenResty（Nginx + Lua），插件丰富
- Nginx：高性能反向代理，需配合Lua模块
- Envoy：CNCF项目，服务网格标准数据面
- Traefik：云原生反向代理，自动服务发现

Gateway的优势：
- 与Spring生态深度集成
- 基于Java，团队技术栈统一
- 编程式路由配置，灵活性高
- 支持响应式编程

Gateway的劣势：
- 性能不如Nginx/Envoy（差距约20-30%）
- 高并发下内存占用较高
- 插件生态不如Kong丰富
```

---

## 源码深度分析：路由与断言机制

### 1. 路由定义与数据结构

```java
// Route.java - 路由定义

public class Route implements Ordered {
    
    // 路由ID（唯一标识）
    private final String id;
    
    // 路由顺序（越小优先级越高）
    private final int order;
    
    // 目标URI（如 lb://user-service）
    private final URI uri;
    
    // 断言集合（所有断言都必须匹配）
    private final List<Predicate<ServerWebExchange>> predicates;
    
    // 过滤器集合
    private final List<GatewayFilter> gatewayFilters;
    
    // 元数据
    private final Map<String, Object> metadata;
    
    public Route(String id, int order, URI uri, 
                 List<Predicate<ServerWebExchange>> predicates,
                 List<GatewayFilter> gatewayFilters,
                 Map<String, Object> metadata) {
        this.id = id;
        this.order = order;
        this.uri = uri;
        this.predicates = predicates;
        this.gatewayFilters = gatewayFilters;
        this.metadata = metadata;
    }
    
    /**
     * 判断请求是否匹配该路由
     */
    public boolean matches(ServerWebExchange exchange) {
        // 所有断言都必须返回true
        return predicates.stream().allMatch(predicate -> predicate.test(exchange));
    }
}
```

### 2. 路由定位器（RouteLocator）

```java
// RouteLocator.java - 路由定位器接口

public interface RouteLocator {
    /**
     * 获取所有路由
     */
    Flux<Route> getRoutes();
}

// CachingRouteLocator.java - 缓存路由定位器

public class CachingRouteLocator implements RouteLocator {
    
    // 缓存的路由列表
    private final AtomicReference<List<Route>> routes = new AtomicReference<>();
    
    // 委托的路由定位器
    private final RouteLocator delegate;
    
    // 应用事件发布器（用于刷新路由）
    private final ApplicationEventPublisher applicationEventPublisher;
    
    public CachingRouteLocator(RouteLocator delegate) {
        this.delegate = delegate;
        this.routes.set(Collections.emptyList());
    }
    
    @Override
    public Flux<Route> getRoutes() {
        // 从缓存返回路由列表
        return Flux.fromIterable(routes.get());
    }
    
    /**
     * 刷新路由缓存（触发点：配置变更、动态路由更新）
     */
    private void fetch() {
        // 从delegate获取最新路由
        List<Route> routes = this.delegate.getRoutes().collectList().block();
        
        // 应用顺序排序
        routes.sort(Comparator.comparingInt(Route::getOrder));
        
        // 更新缓存
        this.routes.set(routes);
        
        // 发布路由刷新事件
        applicationEventPublisher.publishEvent(new RefreshRoutesEvent(this));
    }
    
    @EventListener(RefreshRoutesEvent.class)
    public void handleRefresh(RefreshRoutesEvent event) {
        // 收到刷新事件，重新加载路由
        fetch();
    }
}
```

### 3. 断言（Predicate）机制

```java
// RoutePredicateFactory.java - 断言工厂接口

public interface RoutePredicateFactory<C> extends ShortcutConfigurable, Configurable<C> {
    
    /**
     * 创建断言
     */
    Predicate<ServerWebExchange> apply(C config);
    
    /**
     * 获取断言名称
     */
    default String name() {
        return NameUtils.normalizeRoutePredicateName(getClass());
    }
}

// PathRoutePredicateFactory.java - 路径断言工厂

@Component
public class PathRoutePredicateFactory 
        extends AbstractRoutePredicateFactory<PathRoutePredicateFactory.Config> {
    
    public PathRoutePredicateFactory() {
        super(Config.class);
    }
    
    @Override
    public Predicate<ServerWebExchange> apply(Config config) {
        // 创建路径匹配器
        final PathPatternParser pathPatternParser = new PathPatternParser();
        pathPatternParser.setMatchOptionalTrailingSeparator(
            config.isMatchOptionalTrailingSeparator()
        );
        
        // 解析路径模式
        final List<PathPattern> patterns = config.getPatterns().stream()
            .map(pathPatternParser::parse)
            .collect(Collectors.toList());
        
        return exchange -> {
            // 获取请求路径
            PathContainer path = parsePath(
                exchange.getRequest().getURI().getRawPath()
            );
            
            // 匹配路径模式
            Optional<PathPattern> optionalPathPattern = patterns.stream()
                .filter(pattern -> pattern.matches(path))
                .findFirst();
            
            if (optionalPathPattern.isPresent()) {
                // 保存匹配的路径信息到exchange属性中
                PathPattern pathPattern = optionalPathPattern.get();
                PathPattern.PathMatchInfo pathMatchInfo = pathPattern.matchAndExtract(path);
                
                putUriTemplateVariables(exchange, pathMatchInfo.getUriVariables());
                return true;
            }
            
            return false;
        };
    }
    
    @Validated
    public static class Config {
        private List<String> patterns = new ArrayList<>();
        private boolean matchOptionalTrailingSeparator = true;
        
        // getters and setters
    }
}
```

### 4. 常用断言详解

```
断言类型详解：

1. Path断言：
   predicates:
     - Path=/api/user/**
   
   匹配：/api/user/123, /api/user/list
   不匹配：/api/order/123
   
   原理：使用Spring的PathPattern解析，支持Ant风格通配符

2. Method断言：
   predicates:
     - Method=GET,POST
   
   匹配：GET /api/user/123
   不匹配：DELETE /api/user/123

3. Header断言：
   predicates:
     - Header=X-Request-Id, \d+  # 正则匹配
   
   匹配：Header包含X-Request-Id: 12345
   不匹配：X-Request-Id: abc

4. Query断言：
   predicates:
     - Query=version, v1         # 必须包含version=v1
     - Query=debug               # 只需要包含debug参数

5. Cookie断言：
   predicates:
     - Cookie=sessionId, ch.p

6. Host断言：
   predicates:
     - Host=**.example.com

7. After/Before/Between（时间断言）：
   predicates:
     - After=2024-01-01T00:00:00+08:00[Asia/Shanghai]

8. RemoteAddr断言：
   predicates:
     - RemoteAddr=192.168.1.0/24

9. Weight断言（权重分组）：
   predicates:
     - Weight=group1, 8  # 80%流量
   
   配合：
     - Weight=group1, 2  # 20%流量
```

---

## 源码深度分析：过滤器链与执行流程

### 1. 过滤器接口与类型

```java
// GatewayFilter.java - 网关过滤器接口

public interface GatewayFilter {
    
    /**
     * 执行过滤逻辑
     * @param exchange 请求/响应上下文
     * @param chain 过滤器链（用于传递给下一个过滤器）
     */
    Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain);
}

// GlobalFilter.java - 全局过滤器接口

public interface GlobalFilter {
    
    /**
     * 执行全局过滤逻辑
     */
    Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain);
}

// Ordered.java - 顺序接口（数值越小优先级越高）

public interface Ordered {
    int HIGHEST_PRECEDENCE = Integer.MIN_VALUE;
    int LOWEST_PRECEDENCE = Integer.MAX_VALUE;
    
    int getOrder();
}
```

### 2. 过滤器链执行流程

```java
// FilteringWebHandler.java - 过滤器链处理器

public class FilteringWebHandler implements WebHandler {
    
    // 全局过滤器列表（已排序）
    private final List<GatewayFilter> globalFilters;
    
    public FilteringWebHandler(List<GlobalFilter> globalFilters) {
        // 将GlobalFilter适配为GatewayFilter，并按order排序
        this.globalFilters = loadFilters(globalFilters);
    }
    
    private static List<GatewayFilter> loadFilters(List<GlobalFilter> filters) {
        return filters.stream()
            .map(filter -> {
                // 将GlobalFilter包装为GatewayFilterAdapter
                GatewayFilterAdapter gatewayFilter = new GatewayFilterAdapter(filter);
                
                // 如果实现了Ordered接口，包装为OrderedGatewayFilter
                if (filter instanceof Ordered) {
                    int order = ((Ordered) filter).getOrder();
                    return new OrderedGatewayFilter(gatewayFilter, order);
                }
                return gatewayFilter;
            })
            .sorted(Comparator.comparingInt(Ordered::getOrder))
            .collect(Collectors.toList());
    }
    
    @Override
    public Mono<Void> handle(ServerWebExchange exchange) {
        // 获取匹配的路由
        Route route = exchange.getRequiredAttribute(GATEWAY_ROUTE_ATTR);
        
        // 合并全局过滤器和路由特定过滤器
        List<GatewayFilter> gatewayFilters = route.getFilters();
        List<GatewayFilter> combined = new ArrayList<>(this.globalFilters);
        combined.addAll(gatewayFilters);
        
        // 按order排序
        combined.sort(Comparator.comparingInt(Ordered::getOrder));
        
        // 创建过滤器链并执行
        return new DefaultGatewayFilterChain(combined).filter(exchange);
    }
    
    /**
     * 默认过滤器链实现
     */
    private static class DefaultGatewayFilterChain implements GatewayFilterChain {
        
        private final int index;
        private final List<GatewayFilter> filters;
        
        public DefaultGatewayFilterChain(List<GatewayFilter> filters) {
            this.filters = filters;
            this.index = 0;
        }
        
        private DefaultGatewayFilterChain(DefaultGatewayFilterChain parent, int index) {
            this.filters = parent.filters;
            this.index = index;
        }
        
        @Override
        public Mono<Void> filter(ServerWebExchange exchange) {
            // 递归执行过滤器链
            if (index < filters.size()) {
                GatewayFilter filter = filters.get(index);
                // 创建下一个过滤器链（index+1）
                DefaultGatewayFilterChain chain = new DefaultGatewayFilterChain(this, index + 1);
                // 执行当前过滤器，传入下一个链
                return filter.filter(exchange, chain);
            }
            // 所有过滤器执行完毕
            return Mono.empty();
        }
    }
}
```

### 3. 核心全局过滤器源码

```java
// RouteToRequestUrlFilter.java - 路由到请求URL过滤器

@Component
public class RouteToRequestUrlFilter implements GlobalFilter, Ordered {
    
    public static final int ROUTE_TO_URL_FILTER_ORDER = 10000;
    
    @Override
    public int getOrder() {
        return ROUTE_TO_URL_FILTER_ORDER;
    }
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // 获取路由
        Route route = exchange.getAttribute(GATEWAY_ROUTE_ATTR);
        if (route == null) {
            return chain.filter(exchange);
        }
        
        // 获取原始请求URI
        URI uri = exchange.getRequest().getURI();
        
        // 获取路由目标URI
        URI routeUri = route.getUri();
        
        // 如果目标URI是lb://开头（负载均衡），保留scheme
        if ("lb".equalsIgnoreCase(routeUri.getScheme()) && 
            routeUri.getHost() != null) {
            
            // 添加负载均衡属性
            exchange.getAttributes().put(GATEWAY_REQUEST_URL_ATTR, routeUri);
            
        } else {
            // 直接路由：替换URI的scheme和host
            URI requestUrl = UriComponentsBuilder.fromUri(uri)
                .scheme(routeUri.getScheme())
                .host(routeUri.getHost())
                .port(routeUri.getPort())
                .build(encoded)
                .toUri();
            
            exchange.getAttributes().put(GATEWAY_REQUEST_URL_ATTR, requestUrl);
        }
        
        return chain.filter(exchange);
    }
}

// LoadBalancerClientFilter.java - 负载均衡客户端过滤器

@Component
public class LoadBalancerClientFilter implements GlobalFilter, Ordered {
    
    public static final int LOAD_BALANCER_CLIENT_FILTER_ORDER = 10100;
    
    private final LoadBalancerClient loadBalancer;
    
    @Override
    public int getOrder() {
        return LOAD_BALANCER_CLIENT_FILTER_ORDER;
    }
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        URI url = exchange.getAttribute(GATEWAY_REQUEST_URL_ATTR);
        
        // 检查是否需要负载均衡
        String schemePrefix = exchange.getAttribute(GATEWAY_SCHEME_PREFIX_ATTR);
        if (url == null || (!"lb".equals(url.getScheme()) && !"lb".equals(schemePrefix))) {
            return chain.filter(exchange);
        }
        
        // 保留原始URI
        addOriginalRequestUrl(exchange, url);
        
        // 使用LoadBalancer选择实例
        final ServiceInstance instance = loadBalancer.choose(url.getHost());
        
        if (instance == null) {
            throw NotFoundException.create(true, "Unable to find instance for " + url.getHost());
        }
        
        // 构建新的URI（替换为实例的IP:Port）
        URI requestUrl = loadBalancer.reconstructURI(instance, url);
        
        exchange.getAttributes().put(GATEWAY_REQUEST_URL_ATTR, requestUrl);
        
        return chain.filter(exchange);
    }
}

// NettyRoutingFilter.java - Netty路由过滤器

@Component
public class NettyRoutingFilter implements GlobalFilter, Ordered {
    
    private final HttpClient httpClient;
    
    @Override
    public int getOrder() {
        return Ordered.LOWEST_PRECEDENCE - 1;
    }
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        URI requestUrl = exchange.getRequiredAttribute(GATEWAY_REQUEST_URL_ATTR);
        
        // 获取HTTP方法
        HttpMethod httpMethod = HttpMethod.valueOf(exchange.getRequest().getMethodValue());
        
        // 构建Netty HTTP请求
        NettyOutbound outbound = this.httpClient
            .request(httpMethod)
            .uri(requestUrl.toString())
            .send((req, nettyOutbound) -> {
                // 发送请求体
                nettyOutbound.send(exchange.getRequest().getBody()
                    .map(dataBuffer -> 
                        ((NettyDataBuffer) dataBuffer).getNativeBuffer()
                    )
                );
                return nettyOutbound;
            })
            .responseConnection((res, connection) -> {
                // 处理响应
                ServerHttpResponse response = exchange.getResponse();
                
                // 设置响应状态码
                response.setStatusCode(HttpStatus.valueOf(res.status().code()));
                
                // 设置响应头
                res.responseHeaders().forEach(entry -> 
                    response.getHeaders().add(entry.getKey(), entry.getValue())
                );
                
                // 写回响应体
                return response.writeWith(connection.inbound().body()
                    .map(bytes -> exchange.getResponse().bufferFactory().wrap(bytes))
                );
            });
        
        return outbound.then(chain.filter(exchange));
    }
}

// NettyWriteResponseFilter.java - Netty响应写入过滤器

@Component
public class NettyWriteResponseFilter implements GlobalFilter, Ordered {
    
    public static final int WRITE_RESPONSE_FILTER_ORDER = -1;
    
    @Override
    public int getOrder() {
        return WRITE_RESPONSE_FILTER_ORDER;
    }
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // 这是最后一个过滤器，负责将后端响应写回客户端
        return chain.filter(exchange)
            .then(Mono.fromRunnable(() -> {
                // 清理资源
                Object connection = exchange.getAttributes()
                    .get(CLIENT_RESPONSE_CONN_ATTR);
                if (connection instanceof Connection) {
                    ((Connection) connection).dispose();
                }
            }));
    }
}
```

### 4. 过滤器执行顺序

```
Gateway过滤器执行顺序（按Order排序）：

Order    过滤器名称                         职责
─────────────────────────────────────────────────────────────
-3       RemoveCachedBodyFilter            清除缓存的请求体
-2       AdaptCachedBodyGlobalFilter       适配缓存的请求体
-1       NettyWriteResponseFilter          写回响应（最后执行）

10000    RouteToRequestUrlFilter           构建目标URL
10001    LoadBalancerClientFilter          负载均衡选择实例

─────────────────────────────────────────────────────────────
20000+   自定义GlobalFilter                自定义逻辑
         （如鉴权、日志、限流等）

LOWEST   NettyRoutingFilter                发送HTTP请求（最优先）

注意：
- Order数值越小，优先级越高（越先执行）
- 请求阶段：从高Order到低Order执行
- 响应阶段：从低Order到高Order执行（类似Servlet Filter的doFilter）
```

---

## 源码深度分析：动态路由与限流熔断

### 1. 动态路由实现

```java
// RouteDefinitionWriter.java - 路由定义写入接口

public interface RouteDefinitionWriter {
    
    /**
     * 添加路由定义
     */
    Mono<Void> save(Mono<RouteDefinition> route);
    
    /**
     * 删除路由定义
     */
    Mono<Void> delete(Mono<String> routeId);
}

// InMemoryRouteDefinitionRepository.java - 内存路由存储

public class InMemoryRouteDefinitionRepository 
        implements RouteDefinitionRepository {
    
    // 内存中存储路由定义
    private final Map<String, RouteDefinition> routes = synchronizedMap(
        new LinkedHashMap<>()
    );
    
    @Override
    public Mono<Void> save(Mono<RouteDefinition> route) {
        return route.flatMap(r -> {
            if (StringUtils.isEmpty(r.getId())) {
                return Mono.error(new IllegalArgumentException("id may not be empty"));
            }
            routes.put(r.getId(), r);
            return Mono.empty();
        });
    }
    
    @Override
    public Mono<Void> delete(Mono<String> routeId) {
        return routeId.flatMap(id -> {
            if (routes.containsKey(id)) {
                routes.remove(id);
                return Mono.empty();
            }
            return Mono.defer(() -> Mono.error(
                new NotFoundException("RouteDefinition not found: " + id)
            ));
        });
    }
    
    @Override
    public Flux<RouteDefinition> getRouteDefinitions() {
        return Flux.fromIterable(routes.values());
    }
}

// 动态路由服务（业务层封装）
@Service
public class DynamicRouteService {
    
    @Autowired
    private RouteDefinitionWriter routeDefinitionWriter;
    
    @Autowired
    private ApplicationEventPublisher publisher;
    
    /**
     * 添加路由
     */
    public String addRoute(RouteDefinition definition) {
        routeDefinitionWriter.save(Mono.just(definition)).subscribe();
        publisher.publishEvent(new RefreshRoutesEvent(this));
        return "success";
    }
    
    /**
     * 更新路由
     */
    public String updateRoute(RouteDefinition definition) {
        routeDefinitionWriter.delete(Mono.just(definition.getId())).subscribe();
        routeDefinitionWriter.save(Mono.just(definition)).subscribe();
        publisher.publishEvent(new RefreshRoutesEvent(this));
        return "success";
    }
    
    /**
     * 删除路由
     */
    public String deleteRoute(String id) {
        routeDefinitionWriter.delete(Mono.just(id)).subscribe();
        publisher.publishEvent(new RefreshRoutesEvent(this));
        return "success";
    }
}
```

### 2. 基于Redis的分布式限流

```java
// RequestRateLimiterGatewayFilterFactory.java

@Component
public class RequestRateLimiterGatewayFilterFactory 
        extends AbstractGatewayFilterFactory<RequestRateLimiterGatewayFilterFactory.Config> {
    
    // KeyResolver：用于生成限流键（如IP、用户ID）
    private final KeyResolver defaultKeyResolver;
    
    // RateLimiter：限流算法实现（默认RedisRateLimiter）
    private final RateLimiter defaultRateLimiter;
    
    @Override
    public GatewayFilter apply(Config config) {
        KeyResolver resolver = config.getKeyResolver() != null ? 
            config.getKeyResolver() : defaultKeyResolver;
        RateLimiter limiter = config.getRateLimiter() != null ? 
            config.getRateLimiter() : defaultRateLimiter;
        
        return (exchange, chain) -> {
            // 生成限流键
            Mono<String> keyResolver = resolver.resolve(exchange);
            
            return keyResolver.flatMap(key -> {
                // 获取路由ID
                String routeId = config.getRouteId() != null ? 
                    config.getRouteId() : exchange.getAttribute(GATEWAY_ROUTE_ATTR).getId();
                
                // 执行限流检查
                return limiter.isAllowed(routeId, key)
                    .flatMap(response -> {
                        // 添加限流响应头
                        exchange.getResponse().getHeaders().add(
                            "X-RateLimit-Remaining", 
                            String.valueOf(response.getHeaders().get("X-RateLimit-Remaining"))
                        );
                        
                        if (response.isAllowed()) {
                            // 允许通过
                            return chain.filter(exchange);
                        }
                        
                        // 拒绝请求（429 Too Many Requests）
                        exchange.getResponse().setStatusCode(HttpStatus.TOO_MANY_REQUESTS);
                        return exchange.getResponse().setComplete();
                    });
            });
        };
    }
    
    @Validated
    public static class Config {
        private KeyResolver keyResolver;
        private RateLimiter rateLimiter;
        private Integer burstCapacity;
        private Integer replenishRate;
        private String routeId;
        
        // getters and setters
    }
}

// RedisRateLimiter.java - 基于Redis的令牌桶限流

public class RedisRateLimiter extends AbstractRateLimiter<RedisRateLimiter.Config> {
    
    /**
     * 使用Lua脚本实现原子令牌桶算法
     */
    public Mono<Response> isAllowed(String routeId, String id) {
        if (!isInitialized()) {
            throw new IllegalStateException("RedisRateLimiter is not initialized");
        }
        
        // 配置
        Config routeConfig = getConfig().getOrDefault(routeId, defaultConfig);
        int replenishRate = routeConfig.getReplenishRate();
        int burstCapacity = routeConfig.getBurstCapacity();
        int requestedTokens = routeConfig.getRequestedTokens();
        
        // Redis key
        String key = "request_rate_limiter." + id + "." + routeId;
        List<String> keys = Arrays.asList(key + ".tokens", key + ".timestamp");
        
        // Lua脚本参数
        List<String> scriptArgs = Arrays.asList(
            replenishRate + "",      // 每秒填充速率
            burstCapacity + "",      // 桶容量
            Instant.now().getEpochSecond() + "", // 当前时间戳
            requestedTokens + ""     // 请求令牌数
        );
        
        // 执行Lua脚本（原子操作）
        Flux<List<Long>> flux = this.redisTemplate.execute(
            this.script,        // Lua脚本
            keys,               // Redis keys
            scriptArgs.toArray(new String[0]) // 脚本参数
        );
        
        return flux.onErrorResume(throwable -> Flux.just(Arrays.asList(1L, -1L)))
            .reduce(new ArrayList<Long>(), (longs, l) -> {
                longs.addAll(l);
                return longs;
            })
            .map(results -> {
                boolean allowed = results.get(0) == 1L;
                Long tokensLeft = results.get(1);
                
                Response response = new Response(allowed, getHeaders(routeConfig, tokensLeft));
                
                if (log.isDebugEnabled()) {
                    log.debug("response: {}", response);
                }
                return response;
            });
    }
}
```

**Redis限流Lua脚本**：

```lua
-- 令牌桶限流Lua脚本
-- KEYS[1]: tokens_key (当前令牌数)
-- KEYS[2]: timestamp_key (上次更新时间)
-- ARGV[1]: rate (每秒填充速率)
-- ARGV[2]: capacity (桶容量)
-- ARGV[3]: now (当前时间戳)
-- ARGV[4]: requested (请求令牌数)

local tokens_key = KEYS[1]
local timestamp_key = KEYS[2]

local rate = tonumber(ARGV[1])
local capacity = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
local requested = tonumber(ARGV[4])

-- 获取当前令牌数（如果不存在，设为桶容量）
local last_tokens = redis.call('get', tokens_key)
if last_tokens == false then
    last_tokens = capacity
end

-- 获取上次更新时间
local last_updated = redis.call('get', timestamp_key)
if last_updated == false then
    last_updated = 0
end

-- 计算时间差
local delta = math.max(0, now - last_updated)

-- 计算当前令牌数 = min(桶容量, 上次令牌数 + 时间差 * 填充速率)
local filled_tokens = math.min(capacity, last_tokens + (delta * rate))

-- 判断是否允许请求
local allowed = filled_tokens >= requested
local new_tokens = filled_tokens

if allowed then
    -- 扣减令牌
    new_tokens = filled_tokens - requested
end

-- 更新Redis
redis.call('setex', tokens_key, capacity, new_tokens)
redis.call('setex', timestamp_key, capacity, now)

-- 返回结果：是否允许 + 剩余令牌数
return { allowed and 1 or 0, new_tokens }
```

### 3. 熔断器集成（Resilience4j）

```java
// CircuitBreakerGatewayFilterFactory.java

@Component
public class CircuitBreakerGatewayFilterFactory 
        extends AbstractGatewayFilterFactory<CircuitBreakerGatewayFilterFactory.Config> {
    
    private final CircuitBreakerRegistry circuitBreakerRegistry;
    
    @Override
    public GatewayFilter apply(Config config) {
        CircuitBreaker circuitBreaker = circuitBreakerRegistry
            .circuitBreaker(config.getName(), config.getCircuitBreakerConfig());
        
        return (exchange, chain) -> {
            URI uri = exchange.getAttribute(GATEWAY_REQUEST_URL_ATTR);
            
            // 使用CircuitBreaker包装请求
            return circuitBreaker.executeSupplier(() -> 
                chain.filter(exchange)
                    .doOnSuccess(aVoid -> {
                        // 请求成功，记录成功
                    })
                    .doOnError(throwable -> {
                        // 请求失败，记录失败
                        throw new CircuitBreakerException(throwable);
                    })
            );
        };
    }
    
    @Validated
    public static class Config {
        private String name;
        private CircuitBreakerConfig circuitBreakerConfig;
        private URI fallbackUri;
        
        // getters and setters
    }
}
```

```yaml
# 熔断配置
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/api/user/**
          filters:
            - name: CircuitBreaker
              args:
                name: userServiceCircuitBreaker
                fallbackUri: forward:/fallback/user
                circuitBreakerConfig:
                  failureRateThreshold: 50        # 失败率阈值（%）
                  slowCallRateThreshold: 80       # 慢调用阈值（%）
                  slowCallDurationThreshold: 2s   # 慢调用时间阈值
                  slidingWindowSize: 100          # 滑动窗口大小
                  minimumNumberOfCalls: 10        # 最小调用数
                  permittedNumberOfCallsInHalfOpenState: 10
                  waitDurationInOpenState: 10s    # Open状态持续时间
```

---

## 实战案例：生产级网关配置

### 案例1：完整的Gateway配置文件

```yaml
server:
  port: 8080

spring:
  application:
    name: api-gateway
  
  cloud:
    gateway:
      # 默认过滤器（作用于所有路由）
      default-filters:
        - AddRequestHeader=X-Gateway-Source, SpringCloudGateway
        - AddResponseHeader=X-Gateway-Version, 1.0.0
        - DedupeResponseHeader=Access-Control-Allow-Credentials Access-Control-Allow-Origin
      
      # 全局CORS配置
      globalcors:
        cors-configurations:
          '[/**]':
            allowedOrigins: "https://example.com"
            allowedMethods: "*"
            allowedHeaders: "*"
            allowCredentials: true
            maxAge: 3600
      
      # 路由配置
      routes:
        # 用户服务路由
        - id: user-service
          uri: lb://user-service
          order: 100
          predicates:
            - Path=/api/user/**
            - Method=GET,POST,PUT,DELETE
          filters:
            - StripPrefix=1              # 去掉/api前缀
            - AddRequestHeader=X-User-Service, true
            - RequestRateLimiter=        # 限流
                redis-rate-limiter.replenishRate: 100
                redis-rate-limiter.burstCapacity: 200
                key-resolver: "#{@ipKeyResolver}"
            - CircuitBreaker=            # 熔断
                name: userServiceCB
                fallbackUri: forward:/fallback/user
            - Retry=3                    # 重试3次
        
        # 订单服务路由
        - id: order-service
          uri: lb://order-service
          order: 200
          predicates:
            - Path=/api/order/**
            - Method=GET,POST
          filters:
            - StripPrefix=1
            - RequestSize=5242880        # 限制请求大小5MB
            - RequestRateLimiter=
                redis-rate-limiter.replenishRate: 50
                redis-rate-limiter.burstCapacity: 100
                key-resolver: "#{@userKeyResolver}"
        
        # 支付服务路由（高安全级别）
        - id: payment-service
          uri: lb://payment-service
          order: 300
          predicates:
            - Path=/api/payment/**
            - Method=POST
            - Header=X-Request-Source, INTERNAL
          filters:
            - StripPrefix=1
            - RequestRateLimiter=
                redis-rate-limiter.replenishRate: 10
                redis-rate-limiter.burstCapacity: 20
            - Hystrix=paymentFallback      # Hystrix熔断
        
        # 静态资源路由（直接路由到Nginx）
        - id: static-resources
          uri: http://static-server:80
          order: 1000
          predicates:
            - Path=/static/**
          filters:
            - StripPrefix=1
            - CacheRequestBody           # 缓存请求体

# 日志配置
logging:
  level:
    org.springframework.cloud.gateway: DEBUG
    reactor.netty: INFO

# Actuator端点
management:
  endpoints:
    web:
      exposure:
        include: gateway,health,info,metrics,prometheus
  endpoint:
    gateway:
      enabled: true
```

### 案例2：JWT统一鉴权

```java
@Component
@Slf4j
public class JwtAuthGlobalFilter implements GlobalFilter, Ordered {
    
    @Autowired
    private JwtUtil jwtUtil;
    
    // 白名单路径
    private static final List<String> WHITE_LIST = Arrays.asList(
        "/api/auth/login",
        "/api/auth/register",
        "/actuator/health",
        "/actuator/info"
    );
    
    @Override
    public int getOrder() {
        return -100; // 高优先级（最先执行）
    }
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        String path = request.getURI().getPath();
        
        // 1. 白名单放行
        if (isWhiteList(path)) {
            return chain.filter(exchange);
        }
        
        // 2. 获取Token
        String token = request.getHeaders().getFirst("Authorization");
        if (StringUtils.isEmpty(token) || !token.startsWith("Bearer ")) {
            return unauthorized(exchange, "Missing or invalid Authorization header");
        }
        
        token = token.substring(7);
        
        // 3. 验证Token
        try {
            if (!jwtUtil.validateToken(token)) {
                return unauthorized(exchange, "Invalid or expired token");
            }
            
            // 4. 提取用户信息
            String userId = jwtUtil.getUserId(token);
            String userRole = jwtUtil.getUserRole(token);
            
            // 5. 将用户信息添加到请求头（传递给下游服务）
            ServerHttpRequest mutatedRequest = request.mutate()
                .header("X-User-Id", userId)
                .header("X-User-Role", userRole)
                .header("X-Auth-Time", String.valueOf(System.currentTimeMillis()))
                .build();
            
            return chain.filter(exchange.mutate().request(mutatedRequest).build());
            
        } catch (Exception e) {
            log.error("JWT validation failed", e);
            return unauthorized(exchange, "Token validation failed");
        }
    }
    
    private Mono<Void> unauthorized(ServerWebExchange exchange, String message) {
        ServerHttpResponse response = exchange.getResponse();
        response.setStatusCode(HttpStatus.UNAUTHORIZED);
        response.getHeaders().setContentType(MediaType.APPLICATION_JSON);
        
        String body = String.format("{\"code\":401,\"message\":\"%s\"}", message);
        DataBuffer buffer = response.bufferFactory().wrap(body.getBytes(StandardCharsets.UTF_8));
        
        return response.writeWith(Mono.just(buffer));
    }
    
    private boolean isWhiteList(String path) {
        return WHITE_LIST.stream().anyMatch(path::startsWith);
    }
}
```

### 案例3：请求日志与链路追踪

```java
@Component
@Slf4j
public class LoggingGlobalFilter implements GlobalFilter, Ordered {
    
    @Override
    public int getOrder() {
        return Ordered.LOWEST_PRECEDENCE - 2;
    }
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        long start = System.currentTimeMillis();
        String traceId = exchange.getRequest().getHeaders().getFirst("X-Trace-Id");
        
        // 如果没有traceId，生成一个
        if (StringUtils.isEmpty(traceId)) {
            traceId = UUID.randomUUID().toString().replace("-", "");
            ServerHttpRequest mutatedRequest = exchange.getRequest().mutate()
                .header("X-Trace-Id", traceId)
                .build();
            exchange = exchange.mutate().request(mutatedRequest).build();
        }
        
        final String finalTraceId = traceId;
        
        // 记录请求信息
        logRequest(exchange, finalTraceId);
        
        return chain.filter(exchange)
            .doFinally(signalType -> {
                // 记录响应信息
                long duration = System.currentTimeMillis() - start;
                logResponse(exchange, finalTraceId, duration);
            })
            .doOnError(error -> {
                log.error("[TraceId={}] Request failed: {}", finalTraceId, error.getMessage());
            });
    }
    
    private void logRequest(ServerWebExchange exchange, String traceId) {
        ServerHttpRequest request = exchange.getRequest();
        log.info("[TraceId={}] Request: {} {} from {}",
                traceId,
                request.getMethodValue(),
                request.getURI().getPath(),
                request.getRemoteAddress()
        );
    }
    
    private void logResponse(ServerWebExchange exchange, String traceId, long duration) {
        ServerHttpResponse response = exchange.getResponse();
        log.info("[TraceId={}] Response: {} {}ms",
                traceId,
                response.getStatusCode(),
                duration
        );
    }
}
```

### 案例4：灰度发布（基于版本号）

```java
@Component
public class GrayReleaseGlobalFilter implements GlobalFilter, Ordered {
    
    @Autowired
    private LoadBalancerClient loadBalancer;
    
    @Override
    public int getOrder() {
        return 10050; // 在LoadBalancerClientFilter之后
    }
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // 从请求头获取目标版本
        String targetVersion = exchange.getRequest().getHeaders().getFirst("X-Target-Version");
        
        if (StringUtils.isEmpty(targetVersion)) {
            // 没有指定版本，使用默认路由
            return chain.filter(exchange);
        }
        
        // 获取服务名
        Route route = exchange.getAttribute(ServerWebExchangeUtils.GATEWAY_ROUTE_ATTR);
        String serviceId = route.getUri().getHost();
        
        // 获取所有实例
        List<ServiceInstance> instances = ((DiscoveryClient) loadBalancer)
            .getInstances(serviceId);
        
        // 过滤指定版本的实例
        List<ServiceInstance> grayInstances = instances.stream()
            .filter(instance -> targetVersion.equals(
                instance.getMetadata().get("version")))
            .collect(Collectors.toList());
        
        if (!grayInstances.isEmpty()) {
            // 选择灰度实例
            ServiceInstance instance = grayInstances.get(
                ThreadLocalRandom.current().nextInt(grayInstances.size())
            );
            
            // 重建URI
            URI uri = exchange.getAttribute(ServerWebExchangeUtils.GATEWAY_REQUEST_URL_ATTR);
            URI newUri = loadBalancer.reconstructURI(instance, uri);
            
            exchange.getAttributes().put(
                ServerWebExchangeUtils.GATEWAY_REQUEST_URL_ATTR, 
                newUri
            );
        }
        
        return chain.filter(exchange);
    }
}
```

### 案例5：多租户网关

```java
@Component
public class MultiTenantGlobalFilter implements GlobalFilter, Ordered {
    
    @Override
    public int getOrder() {
        return -50; // 高优先级
    }
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        
        // 从子域名或Header中获取租户ID
        String tenantId = extractTenantId(request);
        
        if (StringUtils.isEmpty(tenantId)) {
            return badRequest(exchange, "Tenant ID is required");
        }
        
        // 验证租户ID合法性
        if (!isValidTenant(tenantId)) {
            return badRequest(exchange, "Invalid Tenant ID");
        }
        
        // 将租户ID添加到请求头和路由元数据
        ServerHttpRequest mutatedRequest = request.mutate()
            .header("X-Tenant-Id", tenantId)
            .build();
        
        // 修改路由目标（指向租户特定的服务实例）
        Route route = exchange.getAttribute(ServerWebExchangeUtils.GATEWAY_ROUTE_ATTR);
        if (route != null) {
            URI originalUri = route.getUri();
            String tenantServiceId = originalUri.getHost() + "-" + tenantId;
            
            Route mutatedRoute = Route.async()
                .asyncPredicate(route.getPredicate())
                .filters(route.getFilters())
                .id(route.getId())
                .order(route.getOrder())
                .uri(URI.create("lb://" + tenantServiceId))
                .build();
            
            exchange.getAttributes().put(
                ServerWebExchangeUtils.GATEWAY_ROUTE_ATTR, 
                mutatedRoute
            );
        }
        
        return chain.filter(exchange.mutate().request(mutatedRequest).build());
    }
    
    private String extractTenantId(ServerHttpRequest request) {
        // 优先从Header获取
        String tenantId = request.getHeaders().getFirst("X-Tenant-Id");
        if (!StringUtils.isEmpty(tenantId)) {
            return tenantId;
        }
        
        // 从子域名获取（如 tenant1.example.com）
        String host = request.getURI().getHost();
        if (host != null && host.contains(".")) {
            return host.substring(0, host.indexOf("."));
        }
        
        return null;
    }
    
    private boolean isValidTenant(String tenantId) {
        // 查询租户数据库或缓存
        return TenantCache.contains(tenantId);
    }
    
    private Mono<Void> badRequest(ServerWebExchange exchange, String message) {
        exchange.getResponse().setStatusCode(HttpStatus.BAD_REQUEST);
        return exchange.getResponse().setComplete();
    }
}
```

---

## 对比分析：Gateway vs Zuul vs Kong vs Nginx

### 核心特性对比

```
┌────────────────────┬─────────────┬─────────────┬─────────────┬─────────────┐
│      特性          │   Gateway   │   Zuul 1.x  │    Kong     │    Nginx    │
├────────────────────┼─────────────┼─────────────┼─────────────┼─────────────┤
│ 底层框架           │   Netty     │   Servlet   │  OpenResty  │   Nginx     │
│ IO模型             │  非阻塞(NIO)│  阻塞(BIO)  │  非阻塞(NIO)│  非阻塞(NIO) │
├────────────────────┼─────────────┼─────────────┼─────────────┼─────────────┤
│ 编程模型           │  响应式     │  同步       │  Lua脚本    │  C/Lua模块  │
│ 开发语言           │  Java       │  Java       │  Lua        │  C          │
├────────────────────┼─────────────┼─────────────┼─────────────┼─────────────┤
│ 性能               │   高        │   中        │   很高      │   很高      │
│ 吞吐量(单机)       │  ~20k QPS   │  ~10k QPS   │  ~50k QPS   │  ~100k QPS  │
│ 延迟               │   低        │   中        │   低        │   很低      │
├────────────────────┼─────────────┼─────────────┼─────────────┼─────────────┤
│ 长连接支持         │   支持      │   不支持    │   支持      │   支持      │
│ WebSocket          │   原生支持  │   不支持    │   支持      │   支持      │
├────────────────────┼─────────────┼─────────────┼─────────────┼─────────────┤
│ 动态路由           │   原生支持  │   支持      │   支持      │   需重载    │
│ 配置热更新         │   支持      │   支持      │   支持      │   需reload  │
├────────────────────┼─────────────┼─────────────┼─────────────┼─────────────┤
│ 限流               │   内置      │   需集成    │   插件丰富  │   需模块    │
│ 熔断               │   集成      │   集成      │   插件      │   需模块    │
├────────────────────┼─────────────┼─────────────┼─────────────┼─────────────┤
│ 鉴权               │   自定义    │   自定义    │   插件丰富  │   需模块    │
│ OAuth2/JWT         │   需开发    │   需开发    │   内置插件  │   需开发    │
├────────────────────┼─────────────┼─────────────┼─────────────┼─────────────┤
│ 插件生态           │   较少      │   较少      │   非常丰富  │   中等      │
│ 扩展性             │   高(Java)  │   高(Java)  │   高(Lua)   │   中(C/Lua) │
├────────────────────┼─────────────┼─────────────┼─────────────┼─────────────┤
│ 学习曲线           │   低        │   低        │   中        │   高        │
│ 团队要求           │   Java团队  │   Java团队  │   DevOps    │   运维团队  │
├────────────────────┼─────────────┼─────────────┼─────────────┼─────────────┤
│ Spring Cloud集成   │   原生      │   原生      │   需适配    │   需适配    │
│ 维护状态           │   活跃      │   停止维护  │   活跃      │   活跃      │
└────────────────────┴─────────────┴─────────────┴─────────────┴─────────────┘
```

### 架构模式对比

```
Spring Cloud Gateway架构：

Client ──> Netty Server ──> RoutePredicateHandlerMapping 
                                    │
                                    ├──> Predicate匹配
                                    │
                                    └──> FilteringWebHandler
                                              │
                                              ├──> GlobalFilter Chain
                                              │       ├──> Auth
                                              │       ├──> RateLimit
                                              │       └──> Log
                                              │
                                              ├──> GatewayFilter
                                              │       └──> StripPrefix
                                              │
                                              └──> NettyRoutingFilter
                                                      │
                                                      └──> HTTP Client
                                                              │
                                                              └──> Service

特点：
- 基于Reactor和Netty，全链路异步
- 过滤器链Ordered执行
- 与Spring生态深度集成

Kong架构：

Client ──> Nginx (OpenResty) ──> Lua Plugins
                                      │
                                      ├──> Authentication
                                      ├──> Rate Limiting
                                      ├──> Logging
                                      └──> Proxy
                                              │
                                              └──> Service

特点：
- 基于Nginx，性能极高
- 插件化架构，功能丰富
- 支持数据库（PostgreSQL/Cassandra）存储配置
```

---

## 性能分析：WebFlux vs Servlet的吞吐量对比

### 1. 基准测试数据

```
Gateway vs Zuul 性能对比：

测试环境：
- Server: 8C16G
- 后端服务: 简单Echo服务（延迟10ms）
- 并发连接: 1000

测试结果：
┌────────────────────────┬─────────────┬─────────────┬─────────────┐
│        指标            │   Gateway   │   Zuul 1.x  │    Nginx    │
├────────────────────────┼─────────────┼─────────────┼─────────────┤
│ 吞吐量 (QPS)           │   ~20000    │   ~10000    │   ~50000    │
│ 平均延迟 (ms)          │    ~15      │    ~35      │    ~8       │
│ P99延迟 (ms)           │    ~45      │    ~120     │    ~20      │
│ CPU使用率 (%)          │    ~60      │    ~90      │    ~30      │
│ 内存占用 (MB)          │    ~512     │    ~1024    │    ~128     │
│ 线程数                 │   16 (Netty)│   1000 (Tomcat)│  8       │
└────────────────────────┴─────────────┴─────────────┴─────────────┘

分析：
- Gateway吞吐量是Zuul的2倍（非阻塞IO优势）
- Gateway延迟比Zuul低50%以上
- Gateway内存占用更低（线程少）
- Nginx性能最优，但功能较少
```

### 2. WebFlux性能优势来源

```
WebFlux vs Servlet性能差异分析：

Servlet模型（Tomcat）：
  请求1 ──> 线程1 ──> 处理（阻塞10ms）──> 响应
  请求2 ──> 线程2 ──> 处理（阻塞10ms）──> 响应
  ...
  请求1000 ──> 线程1000 ──> 处理（阻塞10ms）──> 响应
  
  瓶颈：
  - 线程上下文切换开销
  - 线程内存占用（1MB/线程）
  - 线程池大小限制并发数

WebFlux模型（Netty）：
  EventLoop（4线程）
  ├── 请求1 ──> 提交任务 ──> 继续处理请求2
  ├── 请求2 ──> 提交任务 ──> 继续处理请求3
  ├── 请求3 ──> 提交任务 ──> 继续处理请求4
  ...
  ├── 请求1000 ──> 提交任务
  │
  └── 任务完成回调 ──> 发送响应（不占用线程等待）
  
  优势：
  - 少量线程处理大量请求
  - 无阻塞等待，CPU利用率高
  - 内存占用低
  - 适合I/O密集型场景（网关、API代理）
```

### 3. Gateway性能优化

```yaml
# Gateway性能优化配置
spring:
  cloud:
    gateway:
      # 禁用不需要的功能
      discovery:
        locator:
          enabled: false  # 禁用服务发现自动路由（减少路由数量）
      
      httpclient:
        # Netty连接池配置
        pool:
          type: elastic     # 弹性连接池
          max-size: 500     # 最大连接数
          max-idle-time: 10s # 最大空闲时间
        
        # 超时配置
        connect-timeout: 1000   # 连接超时1秒
        response-timeout: 5s    # 响应超时5秒
        
        # 压缩
        compression: true
      
      # 本地缓存响应
      cache:
        routes:
          - id: static-resources
            uri: http://static-server
            predicates:
              - Path=/static/**
            filters:
              - name: CacheRequestBody
              - name: SaveSession

server:
  netty:
    # Netty服务器配置
    connection-timeout: 2s
    # 启用epoll（Linux）或kqueue（Mac）
    # Spring Boot自动检测

# JVM优化
# -Xms2g -Xmx2g -XX:+UseG1GC -XX:MaxGCPauseMillis=200
```

```java
// 自定义连接池配置
@Configuration
public class GatewayHttpClientConfig {
    
    @Bean
    public HttpClient gatewayHttpClient() {
        return HttpClient.create()
            .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 1000)
            .responseTimeout(Duration.ofSeconds(5))
            .doOnConnected(conn -> 
                conn.addHandlerLast(new ReadTimeoutHandler(5, TimeUnit.SECONDS))
                    .addHandlerLast(new WriteTimeoutHandler(5, TimeUnit.SECONDS))
            )
            .pool(PoolProvider.builder()
                .maxConnections(500)
                .maxIdleTime(Duration.ofSeconds(10))
                .build()
            );
    }
}
```

---

## 常见陷阱与最佳实践

### 陷阱1：过滤器顺序配置错误导致逻辑异常

**问题描述**：鉴权过滤器在日志过滤器之后执行，导致未鉴权的请求被记录了日志；或修改请求的过滤器顺序不对导致修改未生效。

**根本原因**：不了解Gateway过滤器的Ordered执行机制。

**最佳实践**：
```
常用过滤器顺序规范：

Order       过滤器类型              说明
─────────────────────────────────────────────────
-100        鉴权过滤器              最先执行，拒绝未授权请求
-50         租户/灰度过滤器          在鉴权后，路由前
0           请求转换过滤器           修改请求路径/参数
10000       RouteToRequestUrlFilter  Spring内置
10100       LoadBalancerClientFilter Spring内置
20000+      业务自定义过滤器          日志、统计等
LOWEST      NettyRoutingFilter       发送HTTP请求

关键原则：
1. 鉴权、限流、安全类过滤器优先（Order小）
2. 日志、统计类过滤器靠后（Order大）
3. 修改请求的过滤器在路由前执行
4. 修改响应的过滤器在路由后执行
```

```java
// 正确的过滤器顺序配置
@Component
public class AuthGlobalFilter implements GlobalFilter, Ordered {
    @Override
    public int getOrder() {
        return -100; // 高优先级
    }
}

@Component
public class LoggingGlobalFilter implements GlobalFilter, Ordered {
    @Override
    public int getOrder() {
        return Ordered.LOWEST_PRECEDENCE - 2; // 低优先级
    }
}
```

### 陷阱2：在GlobalFilter中使用阻塞IO

**问题描述**：Gateway基于WebFlux和Netty，如果在过滤器中执行阻塞操作（如JDBC查询、Thread.sleep），会阻塞EventLoop线程，导致吞吐量急剧下降。

**根本原因**：不了解响应式编程模型，混用了阻塞和非阻塞代码。

**最佳实践**：
```java
// 错误示例：阻塞EventLoop
@Component
public class BadFilter implements GlobalFilter {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // 错误！阻塞了EventLoop线程
        User user = userRepository.findById(userId); // JDBC阻塞！
        
        return chain.filter(exchange);
    }
}

// 正确示例：使用响应式API
@Component
public class GoodFilter implements GlobalFilter {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        return Mono.fromCallable(() -> userRepository.findById(userId))
            .subscribeOn(Schedulers.boundedElastic()) // 放到弹性线程池
            .flatMap(user -> {
                // 处理user
                return chain.filter(exchange);
            });
    }
}

// 正确示例：使用R2DBC（响应式数据库）
@Component
public class R2dbcFilter implements GlobalFilter {
    @Autowired
    private DatabaseClient databaseClient;
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        return databaseClient.sql("SELECT * FROM users WHERE id = :id")
            .bind("id", userId)
            .fetch()
            .first()
            .flatMap(user -> {
                return chain.filter(exchange);
            });
    }
}
```

### 陷阱3：路由配置错误导致404

**问题描述**：使用`StripPrefix`或`RewritePath`后路径不正确，下游服务报404。

**根本原因**：不理解路径转换规则。

**最佳实践**：
```yaml
# 场景1：请求 /api/user/123，转发到 /user/123
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/api/user/**
          filters:
            - StripPrefix=1  # 去掉/api

# 场景2：请求 /v1/users/123，转发到 /users/123
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/v1/**
          filters:
            - RewritePath=/v1/(?<segment>.*), /${segment}

# 场景3：请求 /api/user/123，转发到 /api/user/123（保留前缀）
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/api/user/**
          # 不需要StripPrefix，直接转发

# 调试技巧：开启DEBUG日志查看实际转发URL
logging:
  level:
    org.springframework.cloud.gateway: DEBUG
```

### 陷阱4：限流配置不当导致误杀正常请求

**问题描述**：限流阈值设置过低，或KeyResolver配置错误（如返回null），导致所有请求被限流。

**根本原因**：未经过压测验证阈值，或KeyResolver实现有bug。

**最佳实践**：
```java
// 正确的KeyResolver实现
@Component
public class RateLimiterConfig {
    
    @Bean
    public KeyResolver ipKeyResolver() {
        return exchange -> Mono.just(
            exchange.getRequest().getRemoteAddress().getAddress().getHostAddress()
        );
    }
    
    @Bean
    public KeyResolver userKeyResolver() {
        return exchange -> {
            String userId = exchange.getRequest().getHeaders().getFirst("X-User-Id");
            if (StringUtils.isEmpty(userId)) {
                // 必须返回非空值，否则限流不生效
                return Mono.just("anonymous");
            }
            return Mono.just(userId);
        };
    }
    
    @Bean
    public KeyResolver apiKeyResolver() {
        return exchange -> Mono.just(
            exchange.getRequest().getPath().value()
        );
    }
}
```

```yaml
# 限流配置最佳实践
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/api/user/**
          filters:
            - name: RequestRateLimiter
              args:
                # 每秒填充速率（QPS）
                redis-rate-limiter.replenishRate: 100
                # 突发容量（应对流量尖峰）
                redis-rate-limiter.burstCapacity: 200
                # KeyResolver
                key-resolver: "#{@ipKeyResolver}"

# 压测确定阈值：
# 1. 逐步增加并发，观察服务响应时间和错误率
# 2. 找到服务的最大承载QPS（如500 QPS）
# 3. 设置限流阈值为最大承载的80%（如400 QPS）
# 4. burstCapacity设置为replenishRate的2倍（应对突发）
```

### 陷阱5：全局过滤器未排除健康检查端点

**问题描述**：鉴权、日志等GlobalFilter拦截了/actuator/health，导致健康检查失败，Kubernetes不断重启Pod。

**根本原因**：GlobalFilter默认作用于所有路由，未对白名单路径做排除。

**最佳实践**：
```java
@Component
public class AuthGlobalFilter implements GlobalFilter, Ordered {
    
    // 白名单路径
    private static final List<String> WHITE_LIST = Arrays.asList(
        "/actuator/health",
        "/actuator/info",
        "/actuator/prometheus",
        "/fallback",
        "/error"
    );
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String path = exchange.getRequest().getURI().getPath();
        
        // 白名单放行
        if (isWhiteList(path)) {
            return chain.filter(exchange);
        }
        
        // 鉴权逻辑...
    }
    
    private boolean isWhiteList(String path) {
        return WHITE_LIST.stream().anyMatch(path::startsWith);
    }
}
```

### 陷阱6：未处理响应式异常导致连接泄漏

**问题描述**：过滤器中发生异常但未正确处理，导致Netty连接未关闭，最终连接池耗尽。

**根本原因**：响应式链中异常处理不完整。

**最佳实践**：
```java
@Component
public class SafeFilter implements GlobalFilter {
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        return chain.filter(exchange)
            .doOnError(error -> {
                log.error("Filter error: {}", error.getMessage());
            })
            .onErrorResume(error -> {
                // 返回错误响应，确保连接被正确关闭
                exchange.getResponse().setStatusCode(HttpStatus.INTERNAL_SERVER_ERROR);
                return exchange.getResponse().setComplete();
            })
            .doFinally(signal -> {
                // 清理资源
                log.debug("Request completed with signal: {}", signal);
            });
    }
}
```

### 陷阱7：动态路由更新未触发刷新

**问题描述**：通过API添加路由后，新路由未生效。

**根本原因**：添加路由后未发布`RefreshRoutesEvent`事件。

**最佳实践**：
```java
@Service
public class DynamicRouteService {
    
    @Autowired
    private RouteDefinitionWriter routeDefinitionWriter;
    
    @Autowired
    private ApplicationEventPublisher publisher;
    
    public void addRoute(RouteDefinition definition) {
        routeDefinitionWriter.save(Mono.just(definition)).subscribe();
        
        // 必须发布刷新事件
        publisher.publishEvent(new RefreshRoutesEvent(this));
    }
}
```

### 陷阱8：忽略Gateway的内存泄漏风险

**问题描述**：Gateway运行一段时间后内存持续增长，最终OOM。

**根本原因**：缓存了请求体/响应体但未清理，或Netty的ByteBuf未释放。

**最佳实践**：
```java
// 使用DataBufferUtils释放资源
import org.springframework.core.io.buffer.DataBufferUtils;

@Component
public class SafeBodyFilter implements GlobalFilter {
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        return exchange.getRequest().getBody()
            .map(dataBuffer -> {
                // 处理数据
                byte[] bytes = new byte[dataBuffer.readableByteCount()];
                dataBuffer.read(bytes);
                
                // 释放DataBuffer
                DataBufferUtils.release(dataBuffer);
                
                return bytes;
            })
            .collectList()
            .flatMap(body -> {
                // 处理请求体
                return chain.filter(exchange);
            });
    }
}
```

```yaml
# 限制请求体大小
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://user-service
          filters:
            - name: RequestSize
              args:
                maxSize: 5MB
```

### 陷阱9：跨域配置重复导致响应头异常

**问题描述**：Gateway和下游服务都配置了CORS，导致响应头重复（如多个Access-Control-Allow-Origin）。

**根本原因**：Gateway和下游服务的CORS配置冲突。

**最佳实践**：
```yaml
# 方案1：仅在Gateway配置CORS，下游服务禁用
spring:
  cloud:
    gateway:
      globalcors:
        cors-configurations:
          '[/**]':
            allowedOrigins: "https://example.com"
            allowedMethods: "*"
            allowedHeaders: "*"
            allowCredentials: true
            maxAge: 3600
      default-filters:
        # 去重响应头
        - DedupeResponseHeader=Access-Control-Allow-Credentials Access-Control-Allow-Origin

# 下游服务application.yml
cors:
  enabled: false  # 禁用下游CORS
```

### 陷阱10：未配置优雅关闭导致请求中断

**问题描述**：Gateway重启或缩容时，正在处理的请求被中断。

**根本原因**：未配置优雅关闭（Graceful Shutdown）。

**最佳实践**：
```yaml
# 配置优雅关闭
server:
  shutdown: graceful  # Spring Boot 2.3+支持

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s  # 等待30秒让请求完成
```

```java
// Kubernetes preStop钩子
@Component
public class GracefulShutdown implements ApplicationListener<ContextClosedEvent> {
    
    @Override
    public void onApplicationEvent(ContextClosedEvent event) {
        log.info("Gateway is shutting down, waiting for active requests to complete...");
        
        // 等待活跃请求完成
        try {
            Thread.sleep(10000); // 等待10秒
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        
        log.info("Gateway shutdown completed.");
    }
}
```

```yaml
# Kubernetes deployment配置
spec:
  template:
    spec:
      containers:
        - name: gateway
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 15"]
```

---

## 面试题与参考答案

### Q1：Spring Cloud Gateway和Zuul 1.x有什么区别？

**参考答案**：

| 特性 | Zuul 1.x | Gateway |
|------|---------|---------|
| 底层框架 | Servlet（Tomcat/Jetty） | Netty |
| IO模型 | 阻塞IO（BIO） | 非阻塞IO（NIO） |
| 编程模型 | 同步 | 异步（Reactive） |
| 长连接支持 | 不支持 | 支持WebSocket |
| 性能 | 一般（~10k QPS） | 高（~20k QPS） |
| 线程模型 | 一线程一请求 | EventLoop多路复用 |
| 维护状态 | 停止维护 | 活跃 |
| Spring集成 | spring-cloud-netflix | spring-cloud-gateway |

**核心差异**：
- Gateway基于Spring 5、Project Reactor和Netty，使用响应式编程模型
- Zuul 1.x基于Servlet 3.0，同步阻塞处理请求
- Gateway性能更优，支持长连接，是Spring官方推荐方案

### Q2：Gateway的核心概念有哪些？

**参考答案**：

Gateway有三个核心概念：

1. **Route（路由）**：
   - 网关的基本映射单元
   - 包含ID、目标URI、Predicate集合和Filter集合
   - 示例：将`/api/user/**`路由到`lb://user-service`

2. **Predicate（断言）**：
   - 匹配条件，决定请求是否走该路由
   - 内置断言：Path、Method、Header、Query、Cookie、After/Before/Between、RemoteAddr等
   - 多个断言可以组合使用（AND关系）

3. **Filter（过滤器）**：
   - 对请求或响应进行处理
   - **GatewayFilter**：作用于单个路由，如StripPrefix、AddRequestHeader、RewritePath、RequestRateLimiter等
   - **GlobalFilter**：作用于所有路由，如鉴权、日志、负载均衡等
   - 过滤器通过Ordered接口控制执行顺序

### Q3：Gateway的请求处理流程是怎样的？

**参考答案**：

Gateway的请求处理流程：

1. **接收请求**：Netty Server接收HTTP请求
2. **路由匹配**：`RoutePredicateHandlerMapping`遍历所有路由，使用Predicate匹配请求
3. **构建过滤器链**：`FilteringWebHandler`将GlobalFilter和GatewayFilter合并排序
4. **执行过滤器链**：按Order从小到大执行
   - 高Order过滤器（如鉴权、限流）
   - 路由相关过滤器（如StripPrefix、RewritePath）
   - `LoadBalancerClientFilter`（负载均衡选择实例）
   - `NettyRoutingFilter`（发送HTTP请求到后端服务）
5. **处理响应**：后端响应后，从低Order到高Order执行响应阶段逻辑
6. **返回客户端**：`NettyWriteResponseFilter`将响应写回客户端

### Q4：Gateway如何实现动态路由？

**参考答案**：

动态路由指不重启网关即可更新路由配置。

**实现方式**：

1. **基于配置中心（Nacos/Apollo）**：
   - 监听配置变更事件
   - 收到变更后调用`RouteDefinitionWriter.save()`写入新路由
   - 发布`RefreshRoutesEvent`事件刷新路由表

2. **基于API**：
   ```java
   @Autowired
   private RouteDefinitionWriter routeDefinitionWriter;
   
   @Autowired
   private ApplicationEventPublisher publisher;
   
   public void addRoute(RouteDefinition definition) {
       routeDefinitionWriter.save(Mono.just(definition)).subscribe();
       publisher.publishEvent(new RefreshRoutesEvent(this));
   }
   ```

3. **基于数据库**：
   - 从数据库加载路由定义
   - 定时刷新或监听数据库变更

**核心类**：
- `RouteDefinitionWriter`：路由定义写入接口
- `CachingRouteLocator`：缓存路由定位器（自动处理刷新）
- `RefreshRoutesEvent`：路由刷新事件

### Q5：Gateway如何实现限流和熔断？

**参考答案**：

**限流**：

Gateway使用`RequestRateLimiterGatewayFilterFactory`实现限流，默认基于Redis的令牌桶算法：

```yaml
filters:
  - name: RequestRateLimiter
    args:
      redis-rate-limiter.replenishRate: 100  # 每秒100个令牌
      redis-rate-limiter.burstCapacity: 200  # 桶容量200
      key-resolver: "#{@ipKeyResolver}"       # 按IP限流
```

**Lua脚本实现**：使用Redis+Lua保证原子性，实现令牌桶算法。

**熔断**：

Gateway集成Resilience4j或Hystrix实现熔断：

```yaml
filters:
  - name: CircuitBreaker
    args:
      name: userServiceCB
      fallbackUri: forward:/fallback/user
      circuitBreakerConfig:
        failureRateThreshold: 50
        waitDurationInOpenState: 10s
```

**工作原理**：
- 记录请求成功/失败次数
- 失败率超过阈值时打开熔断器，直接返回fallback
- 经过休眠时间后进入半开状态，允许部分请求试探
- 试探成功则关闭熔断器，失败则重新打开

### Q6：Gateway如何实现统一鉴权？

**参考答案**：

通过自定义`GlobalFilter`实现JWT统一鉴权：

```java
@Component
public class JwtAuthFilter implements GlobalFilter, Ordered {
    
    @Override
    public int getOrder() {
        return -100; // 高优先级
    }
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String path = exchange.getRequest().getURI().getPath();
        
        // 1. 白名单放行（/actuator/health等）
        if (isWhiteList(path)) {
            return chain.filter(exchange);
        }
        
        // 2. 获取Token
        String token = exchange.getRequest().getHeaders().getFirst("Authorization");
        if (token == null || !token.startsWith("Bearer ")) {
            return unauthorized(exchange);
        }
        
        // 3. 验证Token
        token = token.substring(7);
        if (!jwtUtil.validateToken(token)) {
            return unauthorized(exchange);
        }
        
        // 4. 传递用户信息到下游服务
        String userId = jwtUtil.getUserId(token);
        ServerHttpRequest mutatedRequest = exchange.getRequest().mutate()
            .header("X-User-Id", userId)
            .build();
        
        return chain.filter(exchange.mutate().request(mutatedRequest).build());
    }
}
```

### Q7：GlobalFilter和GatewayFilter有什么区别？

**参考答案**：

**GatewayFilter**：
- 作用于单个路由
- 通过配置文件或Java代码绑定到特定Route
- 内置过滤器：StripPrefix、AddRequestHeader、RewritePath、RequestRateLimiter等
- 示例：`- StripPrefix=1`只作用于当前路由

**GlobalFilter**：
- 作用于所有路由
- 无需配置，自动生效
- 适合实现全局功能：鉴权、日志、跨域、负载均衡等
- 通过实现`GlobalFilter`和`Ordered`接口定义

**执行顺序**：
- 两者都通过`Ordered`接口控制顺序
- 数值越小越先执行
- GatewayFilter优先于同优先级的GlobalFilter
- 实际执行时，所有过滤器按Order混合排序后执行

### Q8：Gateway的过滤器链是如何工作的？

**参考答案**：

Gateway使用**责任链模式**实现过滤器链：

```java
// DefaultGatewayFilterChain
public Mono<Void> filter(ServerWebExchange exchange) {
    if (index < filters.size()) {
        GatewayFilter filter = filters.get(index);
        // 创建下一个链（index+1）
        DefaultGatewayFilterChain chain = new DefaultGatewayFilterChain(this, index + 1);
        // 执行当前过滤器，传入下一个链
        return filter.filter(exchange, chain);
    }
    return Mono.empty(); // 所有过滤器执行完毕
}
```

**执行特点**：
1. 请求阶段：按Order从小到大执行（从高优先级到低优先级）
2. 响应阶段：从当前过滤器回溯执行（类似递归的回调）
3. 每个过滤器可以：
   - 前置处理请求（在`chain.filter(exchange)`之前）
   - 后置处理响应（在`chain.filter(exchange)`之后使用`.then()`）
   - 中断链（直接返回响应，不调用`chain.filter`）

### Q9：Gateway性能优化的手段有哪些？

**参考答案**：

**1. 连接池优化**：
```yaml
spring:
  cloud:
    gateway:
      httpclient:
        pool:
          type: elastic
          max-size: 500
```

**2. 超时配置**：
```yaml
spring:
  cloud:
    gateway:
      httpclient:
        connect-timeout: 1000
        response-timeout: 5s
```

**3. 启用压缩**：
```yaml
spring:
  cloud:
    gateway:
      httpclient:
        compression: true
```

**4. 禁用不需要的功能**：
```yaml
spring:
  cloud:
    gateway:
      discovery:
        locator:
          enabled: false  # 禁用自动路由
```

**5. JVM优化**：
```bash
-Xms2g -Xmx2g -XX:+UseG1GC -XX:MaxGCPauseMillis=200
```

**6. 使用直接内存（Netty）**：
Netty默认使用堆外内存，减少GC压力。

### Q10：Gateway和Nginx如何选择？

**参考答案**：

**选择Gateway的场景**：
- 团队技术栈以Java/Spring为主
- 需要与Spring Cloud生态深度集成（服务发现、配置中心）
- 需要编程式路由配置（动态路由、复杂断言）
- 需要自定义业务逻辑（鉴权、灰度、多租户）
- 已有Spring Cloud微服务架构

**选择Nginx的场景**：
- 极高的性能要求（Gateway比Nginx低20-30%）
- 静态资源服务、文件上传下载
- 简单的反向代理和负载均衡
- 运维团队熟悉Nginx配置

**混合架构**：
```
Client ──> Nginx（SSL终止、静态资源）──> Gateway（动态路由、鉴权）──> Services

优势：
- Nginx处理静态请求和SSL，减轻Gateway压力
- Gateway专注于动态路由和业务逻辑
- 两层网关提供更高的可用性
```

---

*此文原创，转载请注明出处。*
