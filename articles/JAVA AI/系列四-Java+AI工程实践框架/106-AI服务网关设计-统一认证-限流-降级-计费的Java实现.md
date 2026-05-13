> **系列专栏**：[Java + AI 工程实践框架](#)  
> **本文收录**：系列四·Java+AI 工程实践框架  
> **阅读时间**：约 30 分钟

---

## 一、一个价值 20 万的 API Key 泄漏事故

去年某创业公司发生了一起“惨案”：他们在 GitHub 上开源了一个内部工具，忘了把 `application.yml` 里的 OpenAI API Key 删掉。第二天早上，CTO 收到了 AWS（他们通过 AWS  Marketplace 用 OpenAI）的账单提醒——**一夜之间被刷了 20 万人民币**。

查日志发现，这个 Key 被某个爬虫扫到，在暗网上分发后，被全球上百个 IP 疯狂调用 GPT-4。更惨的是，他们没有任何限流、没有告警、没有审计日志，连是谁刷的都不知道。

这不是孤例。AI API 的滥用正在成为新的安全重灾区：

- **API Key 泄露**：硬编码、配置文件误提交、前端暴露
- **盗刷攻击**：Key 被窃取后大规模并发调用
- **Token 浪费**：无限循环的重试、没有限制的上下文
- **成本失控**：没人知道哪个部门花了多少钱

答案只有一个：**你需要一个 AI 服务网关**。

---

## 二、整体架构

一个生产级 AI 服务网关的分层架构如下：

```
                        ┌──────────────────────┐
                        │    客户端/前端        │
                        └──────────┬───────────┘
                                   │ HTTPS
                        ┌──────────▼───────────┐
                        │   AI Service Gateway   │  ← Spring Cloud Gateway
                        │  ┌──────────────────┐ │
                        │  │ ① 认证层          │ │  API Key + JWT
                        │  │ ② 限流层          │ │  Token Bucket
                        │  │ ③ 降级层          │ │  Circuit Breaker
                        │  │ ④ 审计层          │ │  日志落库
                        │  │ ⑤ 路由层          │ │  模型路由
                        │  └──────────────────┘ │
                        └──────────┬───────────┘
                                   │
              ┌────────────────────┼────────────────────┐
     ┌────────▼────────┐  ┌───────▼───────┐  ┌─────────▼────────┐
     │  LLM Service A  │  │ LLM Service B │  │  Billing Service │
     │  (上篇文章的     │  │ (备选服务)     │  │  (计费服务)       │
     │  UnifiedClient) │  │               │  │                  │
     └─────────────────┘  └───────────────┘  └──────────────────┘
```

Spring Cloud Gateway 是整个架构的核心，它作为所有 AI 请求的**唯一入口**，在每个过滤器层完成对应职责。

---

## 三、第一层：统一认证

### 3.1 认证方案设计

我们需要支持两种认证方式，且可以混合使用：

| 认证方式 | 适用场景 | 格式 |
|----------|----------|------|
| API Key | 服务间调用、内部工具 | `Authorization: Bearer sk-xxx` |
| JWT | 带用户身份的 Web/App 调用 | `Authorization: Bearer eyJ...` |

核心认证逻辑：

```java
/**
 * 认证过滤器 —— Gateway 全局过滤器
 */
@Slf4j
@Component
@Order(-100) // 最高优先级，最先执行
public class AuthGatewayFilter implements GlobalFilter {

    private final ApiKeyService apiKeyService;
    private final JwtService jwtService;
    private final List<String> publicPaths; // 无需认证的路径，如 /health, /actuator

    public AuthGatewayFilter(ApiKeyService apiKeyService,
                             JwtService jwtService,
                             @Value("${gateway.auth.public-paths}") List<String> publicPaths) {
        this.apiKeyService = apiKeyService;
        this.jwtService = jwtService;
        this.publicPaths = publicPaths;
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String path = exchange.getRequest().getURI().getPath();

        // 白名单放行
        if (publicPaths.stream().anyMatch(path::startsWith)) {
            return chain.filter(exchange);
        }

        String authHeader = exchange.getRequest().getHeaders().getFirst("Authorization");
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            return unauthorized(exchange, "Missing or invalid Authorization header");
        }

        String token = authHeader.substring(7);

        // 判断是 API Key 还是 JWT
        if (token.startsWith("sk-")) {
            return authenticateByApiKey(exchange, chain, token);
        } else if (token.startsWith("eyJ")) {
            return authenticateByJwt(exchange, chain, token);
        } else {
            return unauthorized(exchange, "Unsupported token format");
        }
    }

    private Mono<Void> authenticateByApiKey(ServerWebExchange exchange,
                                             GatewayFilterChain chain,
                                             String apiKey) {
        return apiKeyService.validate(apiKey)
                .flatMap(keyInfo -> {
                    if (!keyInfo.isActive()) {
                        return unauthorized(exchange, "API Key is disabled");
                    }

                    // 将认证信息注入请求头，下游服务可以直接取
                    ServerHttpRequest mutatedRequest = exchange.getRequest().mutate()
                            .header("X-Auth-Type", "API_KEY")
                            .header("X-Auth-Id", keyInfo.getId())
                            .header("X-Auth-Name", keyInfo.getName())
                            .header("X-Auth-Tier", keyInfo.getTier().name())
                            .header("X-Auth-AppId", keyInfo.getAppId())
                            .build();

                    return chain.filter(exchange.mutate().request(mutatedRequest).build());
                })
                .switchIfEmpty(unauthorized(exchange, "Invalid API Key"));
    }

    private Mono<Void> authenticateByJwt(ServerWebExchange exchange,
                                          GatewayFilterChain chain,
                                          String jwtToken) {
        return jwtService.validateAndParse(jwtToken)
                .flatMap(claims -> {
                    ServerHttpRequest mutatedRequest = exchange.getRequest().mutate()
                            .header("X-Auth-Type", "JWT")
                            .header("X-Auth-Id", claims.getSubject())
                            .header("X-Auth-Name", claims.get("name", String.class))
                            .header("X-Auth-Tier", claims.get("tier", String.class))
                            .header("X-Auth-TenantId", claims.get("tenant_id", String.class))
                            .build();

                    return chain.filter(exchange.mutate().request(mutatedRequest).build());
                })
                .onErrorResume(e -> unauthorized(exchange, "Invalid JWT: " + e.getMessage()));
    }

    private Mono<Void> unauthorized(ServerWebExchange exchange, String message) {
        log.warn("Auth failed for path {}: {}",
                exchange.getRequest().getURI().getPath(), message);

        exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
        exchange.getResponse().getHeaders().setContentType(MediaType.APPLICATION_JSON);

        byte[] body = String.format(
                "{\"error\":\"unauthorized\",\"message\":\"%s\"}", message).getBytes();
        DataBuffer buffer = exchange.getResponse().bufferFactory().wrap(body);

        return exchange.getResponse().writeWith(Mono.just(buffer));
    }
}
```

### 3.2 API Key 管理服务

```java
@Slf4j
@Service
public class ApiKeyService {

    private final ApiKeyRepository repository;
    // 本地缓存，避免每次都查 DB
    private final Cache<String, ApiKeyInfo> localCache =
            Caffeine.newBuilder()
                    .maximumSize(10000)
                    .expireAfterWrite(5, TimeUnit.MINUTES)
                    .build();

    public ApiKeyService(ApiKeyRepository repository) {
        this.repository = repository;
    }

    /**
     * 验证并加载 API Key 信息
     */
    public Mono<ApiKeyInfo> validate(String rawKey) {
        // 第一步：hash 后查询（绝不清存原始 Key）
        String hashedKey = hashApiKey(rawKey);

        // 先查本地缓存
        ApiKeyInfo cached = localCache.getIfPresent(hashedKey);
        if (cached != null) {
            return Mono.just(cached);
        }

        // 查数据库
        return repository.findByHashedKey(hashedKey)
                .flatMap(entity -> {
                    ApiKeyInfo info = toInfo(entity);
                    localCache.put(hashedKey, info);
                    return Mono.just(info);
                });
    }

    /**
     * 创建新的 API Key
     */
    public Mono<String> createKey(String name, String appId, Tier tier,
                                   Long quotaLimit, Long rateLimit) {
        String rawKey = "sk-" + UUID.randomUUID().toString().replace("-", "");
        String hashedKey = hashApiKey(rawKey);

        ApiKeyEntity entity = ApiKeyEntity.builder()
                .hashedKey(hashedKey)
                .name(name)
                .appId(appId)
                .tier(tier)
                .quotaLimit(quotaLimit)     // 总额度（Token 数）
                .rateLimit(rateLimit)        // QPS 限制
                .active(true)
                .createdAt(Instant.now())
                .lastUsedAt(null)
                .build();

        return repository.save(entity)
                .thenReturn(rawKey); // 只在创建时返回原始 Key
    }

    /**
     * 禁用 Key
     */
    public Mono<Void> revoke(String rawKey) {
        String hashedKey = hashApiKey(rawKey);
        localCache.invalidate(hashedKey);
        return repository.deactivate(hashedKey);
    }

    /**
     * 更新最后使用时间
     */
    public Mono<Void> touch(String hashedKey) {
        return repository.updateLastUsed(hashedKey, Instant.now());
    }

    // API Key 的哈希存储（SHA-256 + 固定 Salt）
    private String hashApiKey(String rawKey) {
        try {
            MessageDigest digest = MessageDigest.getInstance("SHA-256");
            String salted = rawKey + "your-fixed-salt-here";
            byte[] hash = digest.digest(salted.getBytes(StandardCharsets.UTF_8));
            return HexFormat.of().formatHex(hash);
        } catch (NoSuchAlgorithmException e) {
            throw new RuntimeException(e);
        }
    }
}

// ===== 数据模型 =====
@Data
@Builder
public class ApiKeyInfo {
    private String id;
    private String name;
    private String appId;
    private Tier tier;       // FREE / STANDARD / PRO / ENTERPRISE
    private Long quotaLimit; // Token 总额度
    private Long rateLimit;  // 每分钟最大请求数
    private boolean active;

    public enum Tier {
        FREE(100_000, 10),
        STANDARD(1_000_000, 60),
        PRO(10_000_000, 300),
        ENTERPRISE(100_000_000, 1000);

        public final long defaultQuota;
        public final long defaultRpm;

        Tier(long defaultQuota, long defaultRpm) {
            this.defaultQuota = defaultQuota;
            this.defaultRpm = defaultRpm;
        }
    }
}
```

---

## 四、第二层：精细粒度限流

AI API 的限流远比普通 REST API 复杂——**不同模型成本差距可达 100 倍**。GPT-4 一个请求可能花 $1，GPT-3.5 可能只花 $0.01。所以限流必须**按用户 + 按模型**双重控制。

### 4.1 限流过滤器

```java
@Slf4j
@Component
@Order(-90) // 在认证之后
public class RateLimitGatewayFilter implements GlobalFilter {

    private final RateLimiterService rateLimiterService;

    public RateLimitGatewayFilter(RateLimiterService rateLimiterService) {
        this.rateLimiterService = rateLimiterService;
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String authId = exchange.getRequest().getHeaders().getFirst("X-Auth-Id");
        String authTier = exchange.getRequest().getHeaders().getFirst("X-Auth-Tier");

        if (authId == null) {
            return chain.filter(exchange); // 未认证的已在 AuthFilter 拦截
        }

        // 从请求体或参数获取模型名
        String model = extractModel(exchange);
        String limitKey = authId + ":" + (model != null ? model : "default");

        return rateLimiterService.tryAcquire(limitKey, authTier)
                .flatMap(allowed -> {
                    if (allowed) {
                        return chain.filter(exchange);
                    }
                    return rateLimited(exchange);
                });
    }

    private String extractModel(ServerWebExchange exchange) {
        // 优先从 Header 取
        String model = exchange.getRequest().getHeaders().getFirst("X-Model");
        if (model != null) return model;

        // 从 Query 参数取
        model = exchange.getRequest().getQueryParams().getFirst("model");
        if (model != null) return model;

        // 从路径提取（如 /v1/chat/gpt-4o）
        String path = exchange.getRequest().getURI().getPath();
        String[] segments = path.split("/");
        if (segments.length >= 4) {
            return segments[segments.length - 1];
        }

        return "default";
    }

    private Mono<Void> rateLimited(ServerWebExchange exchange) {
        exchange.getResponse().setStatusCode(HttpStatus.TOO_MANY_REQUESTS);
        exchange.getResponse().getHeaders().setContentType(MediaType.APPLICATION_JSON);
        exchange.getResponse().getHeaders().set("Retry-After", "60");

        byte[] body = """
                {
                    "error": "rate_limit_exceeded",
                    "message": "请求过于频繁，请稍后重试",
                    "retry_after_seconds": 60
                }
                """.getBytes();

        DataBuffer buffer = exchange.getResponse().bufferFactory().wrap(body);
        return exchange.getResponse().writeWith(Mono.just(buffer));
    }
}
```

### 4.2 Token Bucket 限流实现

```java
@Slf4j
@Service
public class RateLimiterService {

    // 内存限流（适合单机，分布式改为 Redis Lua 脚本）
    private final ConcurrentHashMap<String, TokenBucket> buckets = new ConcurrentHashMap<>();

    // 清理过期桶的定时任务
    @Scheduled(fixedRate = 300_000) // 每 5 分钟
    public void cleanup() {
        long now = System.currentTimeMillis();
        buckets.entrySet().removeIf(entry ->
                now - entry.getValue().getLastAccess() > 600_000); // 10 分钟未访问
    }

    public Mono<Boolean> tryAcquire(String key, String tier) {
        TokenBucket bucket = buckets.computeIfAbsent(key,
                k -> createBucket(tier));
        bucket.touch();
        return Mono.just(bucket.tryAcquire());
    }

    private TokenBucket createBucket(String tier) {
        return switch (tier.toUpperCase()) {
            case "ENTERPRISE" -> new TokenBucket(1000, 1000); // 1000 req/min
            case "PRO" -> new TokenBucket(300, 300);
            case "STANDARD" -> new TokenBucket(60, 60);
            case "FREE" -> new TokenBucket(10, 10);
            default -> new TokenBucket(30, 30);
        };
    }

    /**
     * 令牌桶算法实现
     */
    static class TokenBucket {
        private final long capacity;      // 桶容量
        private final long refillRate;    // 每分钟填充速率
        private volatile double tokens;
        private volatile long lastRefillTime;
        private volatile long lastAccess;

        TokenBucket(long capacity, long refillRate) {
            this.capacity = capacity;
            this.refillRate = refillRate;
            this.tokens = capacity;
            this.lastRefillTime = System.currentTimeMillis();
            this.lastAccess = lastRefillTime;
        }

        synchronized boolean tryAcquire() {
            refill();
            if (tokens >= 1) {
                tokens -= 1;
                return true;
            }
            return false;
        }

        private void refill() {
            long now = System.currentTimeMillis();
            long elapsed = now - lastRefillTime;
            double newTokens = (double) elapsed / 60_000 * refillRate;
            tokens = Math.min(capacity, tokens + newTokens);
            lastRefillTime = now;
        }

        void touch() {
            this.lastAccess = System.currentTimeMillis();
        }

        long getLastAccess() {
            return lastAccess;
        }
    }
}
```

### 4.3 分布式限流（Redis + Lua）

单机版适合开发测试，生产环境必须用 Redis 保证精确性：

```java
@Slf4j
@Component
@Profile("production")
public class RedisRateLimiter implements RateLimitStrategy {

    private final ReactiveRedisTemplate<String, String> redisTemplate;

    // Lua 脚本：原子性地检查并扣减令牌
    private static final String RATE_LIMIT_SCRIPT = """
            local key = KEYS[1]
            local capacity = tonumber(ARGV[1])
            local refill_rate = tonumber(ARGV[2])
            local now = tonumber(ARGV[3])
            
            local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
            local tokens = tonumber(bucket[1]) or capacity
            local last_refill = tonumber(bucket[2]) or now
            
            -- 计算新令牌
            local elapsed = math.max(0, now - last_refill)
            local new_tokens = elapsed / 60000 * refill_rate
            tokens = math.min(capacity, tokens + new_tokens)
            
            if tokens >= 1 then
                tokens = tokens - 1
                redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
                redis.call('EXPIRE', key, 120)
                return 1
            else
                redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
                redis.call('EXPIRE', key, 120)
                return 0
            end
            """;

    private final RedisScript<Long> script;

    public RedisRateLimiter(ReactiveRedisTemplate<String, String> redisTemplate) {
        this.redisTemplate = redisTemplate;
        this.script = RedisScript.of(RATE_LIMIT_SCRIPT, Long.class);
    }

    @Override
    public Mono<Boolean> tryAcquire(String key, long capacity, long refillRate) {
        String redisKey = "rate_limit:" + key;
        return redisTemplate.execute(script,
                List.of(redisKey),
                String.valueOf(capacity),
                String.valueOf(refillRate),
                String.valueOf(System.currentTimeMillis())
        ).map(result -> result == 1L);
    }
}
```

---

## 五、第三层：智能降级与熔断

AI 服务的 SLA 永远不是 100%。OpenAI 历史上多次宕机，通义千问也曾数小时不可用。网关必须能**自动感知故障并切换备选**。

### 5.1 集成 Resilience4j

```yaml
# application.yml
resilience4j:
  circuitbreaker:
    configs:
      default:
        sliding-window-size: 20          # 滑动窗口大小
        minimum-number-of-calls: 10      # 最少调用次数才计算
        failure-rate-threshold: 50       # 失败率阈值（%）
        wait-duration-in-open-state: 30s # 半开等待时间
        permitted-number-of-calls-in-half-open-state: 3
        automatic-transition-from-open-to-half-open-enabled: true
    instances:
      openai-gpt-4o:
        failure-rate-threshold: 30      # GPT-4 成本高，更敏感
        sliding-window-size: 10
        wait-duration-in-open-state: 60s
      deepseek-chat:
        failure-rate-threshold: 50
        wait-duration-in-open-state: 15s
```

### 5.2 降级过滤器

```java
@Slf4j
@Component
@Order(-80)
public class CircuitBreakerGatewayFilter implements GlobalFilter {

    private final LlmRoutingService routingService;

    // 模型降级链配置
    @Value("#{${gateway.fallback.chains}}")
    private Map<String, List<String>> fallbackChains;

    public CircuitBreakerGatewayFilter(LlmRoutingService routingService) {
        this.routingService = routingService;
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String model = exchange.getRequest().getHeaders().getFirst("X-Model");
        if (model == null) {
            model = "default";
        }

        // 检查熔断状态
        String finalModel = model;
        return routingService.getCircuitBreakerState(model)
                .flatMap(state -> {
                    if (state == CircuitBreakerState.OPEN) {
                        // 主模型熔断，尝试降级
                        return handleFallback(exchange, chain, finalModel);
                    }
                    return chain.filter(exchange);
                });
    }

    private Mono<Void> handleFallback(ServerWebExchange exchange,
                                       GatewayFilterChain chain,
                                       String failedModel) {
        List<String> fallbacks = fallbackChains.getOrDefault(failedModel, List.of());
        if (fallbacks.isEmpty()) {
            // 没有降级方案，返回 503
            exchange.getResponse().setStatusCode(HttpStatus.SERVICE_UNAVAILABLE);
            byte[] body = """
                    {
                        "error": "service_unavailable",
                        "message": "AI 服务暂时不可用，请稍后重试"
                    }
                    """.getBytes();
            DataBuffer buffer = exchange.getResponse().bufferFactory().wrap(body);
            return exchange.getResponse().writeWith(Mono.just(buffer));
        }

        // 选择第一个可用的备选模型
        String fallbackModel = fallbacks.get(0);
        log.warn("Model {} is down, falling back to {}", failedModel, fallbackModel);

        // 修改请求头中的模型名
        ServerHttpRequest mutatedRequest = exchange.getRequest().mutate()
                .header("X-Model", fallbackModel)
                .header("X-Fallback-Reason", "CIRCUIT_OPEN:" + failedModel)
                .build();

        return chain.filter(exchange.mutate().request(mutatedRequest).build());
    }
}
```

### 5.3 熔断状态服务

```java
@Slf4j
@Service
public class LlmRoutingService {

    private final CircuitBreakerRegistry circuitBreakerRegistry;
    private final Map<String, CircuitBreaker> breakers = new ConcurrentHashMap<>();

    // 灰度发布：模型流量权重
    private final Map<String, Map<String, Integer>> modelWeights = new ConcurrentHashMap<>();

    public LlmRoutingService(CircuitBreakerRegistry circuitBreakerRegistry) {
        this.circuitBreakerRegistry = circuitBreakerRegistry;
    }

    /**
     * 获取模型的熔断状态
     */
    public Mono<CircuitBreakerState> getCircuitBreakerState(String model) {
        Optional<CircuitBreaker> breakerOpt =
                circuitBreakerRegistry.find(model.replace('-', '_'));

        if (breakerOpt.isEmpty()) {
            return Mono.just(CircuitBreakerState.CLOSED);
        }

        return Mono.just(breakerOpt.get().getState());
    }

    /**
     * 获取带熔断保护的调用函数
     */
    public <T> Mono<T> executeWithBreaker(String model,
                                           Supplier<Mono<T>> supplier,
                                           Function<Throwable, Mono<T>> fallback) {
        CircuitBreaker breaker = breakers.computeIfAbsent(model, m ->
                circuitBreakerRegistry.circuitBreaker(m.replace('-', '_')));

        return Mono.defer(supplier)
                .transformDeferred(CircuitBreakerOperator.of(breaker))
                .onErrorResume(CallNotPermittedException.class, ex -> {
                    log.warn("Circuit breaker OPEN for model: {}", model);
                    return fallback.apply(ex);
                })
                .onErrorResume(ex -> {
                    log.error("Model {} call failed: {}", model, ex.getMessage());
                    return fallback.apply(ex);
                });
    }

    /**
     * 设置灰度权重（用于平滑切换模型）
     */
    public void setModelWeight(String modelGroup, String model, int weight) {
        modelWeights.computeIfAbsent(modelGroup, k -> new ConcurrentHashMap<>())
                .put(model, weight);
    }

    /**
     * 根据权重选择一个模型
     */
    public String selectModelByWeight(String modelGroup) {
        Map<String, Integer> weights = modelWeights.get(modelGroup);
        if (weights == null || weights.isEmpty()) {
            return modelGroup;
        }

        int totalWeight = weights.values().stream().mapToInt(Integer::intValue).sum();
        int random = ThreadLocalRandom.current().nextInt(totalWeight);
        int cumulative = 0;

        for (Map.Entry<String, Integer> entry : weights.entrySet()) {
            cumulative += entry.getValue();
            if (random < cumulative) {
                return entry.getKey();
            }
        }

        return weights.keySet().iterator().next();
    }
}
```

---

## 六、第四层：Token 计数与成本分摊

这是 AI 网关的**杀手级功能**——能回答两个核心问题：

1. **我的钱到底花哪了？**
2. **哪个业务方花得最多？**

### 6.1 响应拦截器：统计 Token

```java
@Slf4j
@Component
@Order(100) // 在响应阶段执行
public class TokenCountingFilter implements GlobalFilter {

    private final TokenUsageService tokenUsageService;
    private final MeterRegistry meterRegistry; // Micrometer 指标

    public TokenCountingFilter(TokenUsageService tokenUsageService,
                                MeterRegistry meterRegistry) {
        this.tokenUsageService = tokenUsageService;
        this.meterRegistry = meterRegistry;
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        long startTime = System.currentTimeMillis();

        return chain.filter(exchange)
                .then(Mono.defer(() -> {
                    long duration = System.currentTimeMillis() - startTime;
                    TokenUsage usage = extractTokenUsage(exchange);

                    if (usage != null) {
                        return recordUsage(exchange, usage, duration);
                    }
                    return Mono.empty();
                }));
    }

    /**
     * 从响应体中提取 Token 用量（缓存 body 并解析）
     */
    private TokenUsage extractTokenUsage(ServerWebExchange exchange) {
        // 优先从响应头获取（如果下游服务注入了 X-Token-Usage）
        String tokenHeader = exchange.getResponse()
                .getHeaders().getFirst("X-Token-Usage");
        if (tokenHeader != null) {
            String[] parts = tokenHeader.split(",");
            return new TokenUsage(
                    Integer.parseInt(parts[0]),  // prompt
                    Integer.parseInt(parts[1]),  // completion
                    Integer.parseInt(parts[2])   // total
            );
        }

        // 或者从响应体内解析（缓存的 body）
        byte[] cachedBody = exchange.getAttribute("cachedResponseBody");
        if (cachedBody != null) {
            try {
                JsonNode root = new ObjectMapper()
                        .readTree(new String(cachedBody));
                JsonNode usageNode = root.path("usage");
                if (!usageNode.isMissingNode()) {
                    return new TokenUsage(
                            usageNode.path("prompt_tokens").asInt(),
                            usageNode.path("completion_tokens").asInt(),
                            usageNode.path("total_tokens").asInt()
                    );
                }
            } catch (Exception e) {
                log.debug("Failed to parse token usage from response body");
            }
        }

        return null;
    }

    private Mono<Void> recordUsage(ServerWebExchange exchange,
                                    TokenUsage usage,
                                    long durationMs) {
        String authId = exchange.getRequest().getHeaders().getFirst("X-Auth-Id");
        String model = exchange.getRequest().getHeaders().getFirst("X-Model");
        String appId = exchange.getRequest().getHeaders().getFirst("X-Auth-AppId");

        // 计算成本（不同模型不同单价）
        BigDecimal cost = PricingService.calculate(model,
                usage.promptTokens,
                usage.completionTokens);

        // 写入数据库
        TokenUsageRecord record = TokenUsageRecord.builder()
                .authId(authId)
                .appId(appId)
                .model(model)
                .promptTokens(usage.promptTokens)
                .completionTokens(usage.completionTokens)
                .totalTokens(usage.totalTokens)
                .cost(cost)
                .durationMs(durationMs)
                .timestamp(Instant.now())
                .build();

        // 记录 Prometheus 指标
        meterRegistry.counter("ai.token.total", "model", model).increment(usage.totalTokens);
        meterRegistry.counter("ai.cost.total", "model", model).increment(cost.doubleValue());
        meterRegistry.timer("ai.request.duration", "model", model)
                .record(durationMs, java.util.concurrent.TimeUnit.MILLISECONDS);

        return tokenUsageService.save(record).then();
    }

    record TokenUsage(int promptTokens, int completionTokens, int totalTokens) {}
}
```

### 6.2 定价服务

```java
@Slf4j
@Component
public class PricingService {

    /**
     * 模型定价表（每 1000 Token 的价格，单位：美元）
     * 数据来源：各厂商官方定价
     */
    private static final Map<String, Price> PRICING = Map.ofEntries(
        // OpenAI
        Map.entry("gpt-4o", new Price(new BigDecimal("0.005"), new BigDecimal("0.015"))),
        Map.entry("gpt-4o-mini", new Price(new BigDecimal("0.00015"), new BigDecimal("0.0006"))),
        Map.entry("gpt-4-turbo", new Price(new BigDecimal("0.01"), new BigDecimal("0.03"))),
        // DeepSeek
        Map.entry("deepseek-chat", new Price(
                new BigDecimal("0.00014"), new BigDecimal("0.00028"))),
        Map.entry("deepseek-reasoner", new Price(
                new BigDecimal("0.00055"), new BigDecimal("0.00219"))),
        // Claude
        Map.entry("claude-3-opus", new Price(
                new BigDecimal("0.015"), new BigDecimal("0.075"))),
        Map.entry("claude-3-sonnet", new Price(
                new BigDecimal("0.003"), new BigDecimal("0.015"))),
        Map.entry("claude-3-haiku", new Price(
                new BigDecimal("0.00025"), new BigDecimal("0.00125"))),
        // 通义千问
        Map.entry("qwen-turbo", new Price(
                new BigDecimal("0.0003"), new BigDecimal("0.0006"))),
        Map.entry("qwen-plus", new Price(
                new BigDecimal("0.002"), new BigDecimal("0.006"))),
        Map.entry("qwen-max", new Price(
                new BigDecimal("0.04"), new BigDecimal("0.12"))),
        // Ollama 本地模型
        Map.entry("local", new Price(BigDecimal.ZERO, BigDecimal.ZERO))
    );

    /**
     * 计算单次调用的成本
     */
    public static BigDecimal calculate(String model,
                                        int promptTokens,
                                        int completionTokens) {
        Price price = PRICING.getOrDefault(model,
                // 未识别的模型，按中等价格估算
                new Price(new BigDecimal("0.001"), new BigDecimal("0.005")));

        BigDecimal promptCost = price.promptPrice
                .multiply(BigDecimal.valueOf(promptTokens))
                .divide(BigDecimal.valueOf(1000), 6, RoundingMode.HALF_UP);

        BigDecimal completionCost = price.completionPrice
                .multiply(BigDecimal.valueOf(completionTokens))
                .divide(BigDecimal.valueOf(1000), 6, RoundingMode.HALF_UP);

        return promptCost.add(completionCost);
    }

    record Price(BigDecimal promptPrice, BigDecimal completionPrice) {}
}
```

### 6.3 使用量查询 API

```java
@Slf4j
@RestController
@RequestMapping("/api/admin/billing")
public class BillingController {

    private final TokenUsageRepository repository;

    @GetMapping("/summary")
    public Mono<BillingSummary> getSummary(
            @RequestParam(required = false) String appId,
            @RequestParam(defaultValue = "day") String granularity) {
        return repository.summarize(appId, granularity);
    }

    @GetMapping("/by-model")
    public Flux<ModelCost> getCostByModel(
            @RequestParam(defaultValue = "7") int days) {
        return repository.costByModel(days);
    }

    @GetMapping("/by-user")
    public Flux<UserCost> getCostByUser(
            @RequestParam(defaultValue = "7") int days,
            @RequestParam(defaultValue = "10") int limit) {
        return repository.topUsers(days, limit);
    }
}

@Data
@Builder
class BillingSummary {
    private long totalRequests;
    private long totalTokens;
    private BigDecimal totalCost;
    private String currency;
}

@Data
@Builder
class ModelCost {
    private String model;
    private long tokens;
    private BigDecimal cost;
}

@Data
@Builder
class UserCost {
    private String userId;
    private long tokens;
    private BigDecimal cost;
}
```

---

## 七、第五层：审计日志

```java
@Slf4j
@Component
@Order(200) // 最后执行
public class AuditLogFilter implements GlobalFilter {

    private final AuditLogService auditLogService;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String requestId = exchange.getRequest().getId();
        String path = exchange.getRequest().getURI().getPath();
        String method = exchange.getRequest().getMethod().name();
        String clientIp = getClientIp(exchange);
        String authId = exchange.getRequest().getHeaders().getFirst("X-Auth-Id");
        String model = exchange.getRequest().getHeaders().getFirst("X-Model");
        long startTime = System.currentTimeMillis();

        // 异步记录请求日志
        AuditEntry entry = AuditEntry.builder()
                .requestId(requestId)
                .path(path)
                .method(method)
                .clientIp(clientIp)
                .authId(authId)
                .model(model)
                .startTime(Instant.now())
                .build();

        return chain.filter(exchange)
                .doOnSuccess(v -> completeLog(entry, exchange))
                .doOnError(e -> completeErrorLog(entry, exchange, e));
    }

    private void completeLog(AuditEntry entry, ServerWebExchange exchange) {
        HttpStatus status = (HttpStatus) exchange.getResponse().getStatusCode();
        entry.setEndTime(Instant.now());
        entry.setDurationMs(
                Duration.between(entry.getStartTime(), entry.getEndTime()).toMillis());
        entry.setStatusCode(status != null ? status.value() : 200);
        entry.setSuccess(status != null && status.is2xxSuccessful());

        auditLogService.save(entry).subscribe();
    }

    private void completeErrorLog(AuditEntry entry, ServerWebExchange exchange,
                                   Throwable error) {
        entry.setEndTime(Instant.now());
        entry.setStatusCode(500);
        entry.setSuccess(false);
        entry.setErrorMessage(error.getMessage());

        auditLogService.save(entry).subscribe();
    }

    private String getClientIp(ServerWebExchange exchange) {
        // 优先取代理转发的真实 IP
        String forwarded = exchange.getRequest()
                .getHeaders().getFirst("X-Forwarded-For");
        if (forwarded != null) {
            return forwarded.split(",")[0].trim();
        }
        return exchange.getRequest().getRemoteAddress() != null
                ? exchange.getRequest().getRemoteAddress().getAddress().getHostAddress()
                : "unknown";
    }
}

@Data
@Builder
class AuditEntry {
    private String requestId;
    private String path;
    private String method;
    private String clientIp;
    private String authId;
    private String model;
    private Instant startTime;
    private Instant endTime;
    private long durationMs;
    private int statusCode;
    private boolean success;
    private String errorMessage;
}
```

---

## 八、完整网关配置（application.yml）

```yaml
server:
  port: 8080

spring:
  application:
    name: ai-service-gateway
  cloud:
    gateway:
      routes:
        - id: llm-chat
          uri: http://localhost:8081  # 上篇文章的 UnifiedLlmClient
          predicates:
            - Path=/api/v1/chat/**
          filters:
            - name: RequestSize
              args:
                maxSize: 10MB
        - id: llm-embedding
          uri: http://localhost:8082
          predicates:
            - Path=/api/v1/embedding/**
      default-filters:
        - DedupeResponseHeader=Access-Control-Allow-Origin
  # Redis 配置（分布式限流用）
  data:
    redis:
      host: ${REDIS_HOST:localhost}
      port: ${REDIS_PORT:6379}
      timeout: 2000ms

gateway:
  auth:
    public-paths:
      - /health
      - /actuator
      - /api/public
  rate-limit:
    enabled: true
    default-rpm: 60
    burst-multiplier: 1.5  # 突发系数
  fallback:
    chains:
      gpt-4: [gpt-4o, gpt-4o-mini, deepseek-chat]
      gpt-4o: [gpt-4o-mini, deepseek-chat, qwen-plus]
      deepseek-chat: [qwen-plus, gpt-4o-mini]
      claude-3-opus: [claude-3-sonnet, gpt-4o]
      default: [deepseek-chat, qwen-plus, gpt-4o-mini]

resilience4j:
  circuitbreaker:
    instances:
      gpt-4o:
        sliding-window-size: 20
        failure-rate-threshold: 30
        wait-duration-in-open-state: 60s
      deepseek-chat:
        sliding-window-size: 20
        failure-rate-threshold: 50
        wait-duration-in-open-state: 15s
  timelimiter:
    instances:
      default:
        timeout-duration: 30s

# 审计日志存储
audit:
  storage: elasticsearch  # 或 mysql / mongodb
  retention-days: 90
  batch-size: 100

# 计费
billing:
  currency: CNY
  exchange-rate: 7.25  # USD to CNY
  alert-threshold-daily: 100  # 每日成本超过 100 元告警
```

---

## 九、监控与告警

### 9.1 Prometheus 指标导出

```java
@Configuration
public class MetricsConfig {

    @Bean
    public MeterRegistryCustomizer<MeterRegistry> metricsCommonTags() {
        return registry -> registry.config()
                .commonTags("application", "ai-gateway");
    }

    @Bean
    public TimedAspect timedAspect(MeterRegistry registry) {
        return new TimedAspect(registry);
    }
}
```

关键指标：

```java
// 在 TokenCountingFilter 中已经暴露了以下指标：
// ai.token.total{model="gpt-4o"} - 各模型 Token 消耗
// ai.cost.total{model="gpt-4o"}   - 各模型成本
// ai.request.duration{model="gpt-4o"} - 各模型响应时间

// 还可添加：
// ai.request.errors{model, error_type} - 错误计数
// ai.circuit_breaker.state{model}       - 熔断状态
```

### 9.2 成本告警

```java
@Component
public class CostAlertService {

    private final BillingRepository billingRepository;
    private final AlertNotifier alertNotifier;

    @Value("${billing.alert-threshold-daily:100}")
    private BigDecimal alertThreshold;

    @Scheduled(fixedRate = 300_000) // 每 5 分钟检查
    public void checkCostAlert() {
        BigDecimal todayCost = billingRepository.getTodayCost();
        if (todayCost.compareTo(alertThreshold) > 0) {
            alertNotifier.send(
                "成本告警",
                String.format("今日 AI API 调用成本已达 ¥%s，超过阈值 ¥%s",
                        todayCost, alertThreshold)
            );
        }
    }
}
```

---

## 十、总结

一个生产级 AI 服务网关，核心是**五层安全架构**：

| 层级 | 职责 | 核心实现 |
|------|------|----------|
| 认证层 | 验证调用者身份 | API Key SHA-256 + JWT，本地缓存 |
| 限流层 | 防止滥用 | 令牌桶（单机）/ Redis Lua（分布式），按用户+模型 |
| 降级层 | 保障可用性 | Resilience4j Circuit Breaker + 降级链 |
| 计费层 | 成本可视化 | 响应拦截解析 Token，模型定价表，实时汇总 |
| 审计层 | 问题追溯 | 异步日志，记录 IP/模型/耗时/状态 |

有了这套网关，你再也不用担心：
- ❌ API Key 被盗刷到天亮
- ❌ 某个用户把 GPT-4 当免费的无限用
- ❌ OpenAI 宕机后 500 刷屏
- ❌ 月底对账时一脸茫然

---

**下一篇预告**：

> **《Spring AI 项目从开发到生产的 10 个避坑指南，每个坑都能让你加班到凌晨》**
>
> 10 个真实踩坑案例全面复盘：API Key 泄漏到 GitHub、Streaming 超时配置错误、Embedding 维度不匹配、向量数据库连接池耗尽、Function Calling 无限循环、Token 超限未处理、提示词注入漏洞、SDK 版本不兼容、并发下对话历史错乱、监控缺失导致成本爆炸。每个坑都有事故描述 + 根因 + 解决方案 + 预防措施。敬请期待！

---

> 如果本文对你有帮助，欢迎**点赞、收藏、关注**三连。  
> 有任何问题欢迎在评论区留言讨论。
