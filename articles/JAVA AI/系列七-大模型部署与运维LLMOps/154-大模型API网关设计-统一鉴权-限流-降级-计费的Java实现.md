# 大模型 API 网关设计：统一鉴权、限流、降级、计费的 Java 实现，生产必备

## 前言

"CTO 让我紧急关停 AI 问答功能。"凌晨两点，运维小哥的消息让我从床上弹了起来。

"为什么？" "运营那边搞了个活动，AI 调用量翻了 40 倍，这个月 Token 预算 3000 美金两天就干完了。而且还没做鉴权，有人直接用 Postman 调我们的接口，API Key 都被抄走了。"

这是绝大多数 AI 团队在"接入大模型"阶段最容易踩的坑——**只关心能不能调通，不关心怎么调得对**。当你的应用从 demo 变成产品的那一天，一个统一的 API 网关是绕不过去的坎。

今天这篇文章，我用纯 Java + Spring Cloud Gateway 实现一个生产级的 LLM API 网关，包含**统一鉴权、多级限流、熔断降级、Token 计费、审计日志**五大能力。代码可以直接复制到你的项目。


## 一、为什么 LLM 需要专用网关？

### 1.1 通用网关 vs LLM 网关

| 需求 | 通用 API 网关 | LLM 专用网关 |
|------|-------------|-------------|
| 鉴权 | API Key / JWT | + 用户级别的模型配额 |
| 限流 | QPS 限流 | + Token 额度限流（每日/每月） |
| 熔断 | 连接超时 | + Token 消耗陡增熔断 |
| 计费 | 调用次数 | + 按 Token 实时扣费 |
| 路由 | 实例维度 | + 模型维度（GPT4/Claude/自有模型） |
| 安全 | SQL 注入 | + Prompt 注入 + 有害内容 |

### 1.2 网关架构总览

```yaml
# LLM API Gateway 架构
architecture:
  ingress:                    # 入口层
    - nginx                   # 反向代理 + SSL
    - rate_limiter: "令牌桶"   # 全局限流
    
  gateway:                    # 网关核心
    - spring_cloud_gateway    # 路由分发
    - auth_filter:            # 鉴权过滤器
        - api_key_validation
        - user_quota_check    # 用户配额检查
        - model_permission    # 模型权限验证
    - rate_limiter_filter:    # 限流过滤器
        - user_level_limiter
        - model_level_limiter
        - ip_level_limiter
    - cost_filter:            # 计费过滤器
        - token_counting
        - cost_recording
        - budget_check
    
  backend:                    # 后端服务
    - vllm_cluster            # 自部署模型
    - openai_api              # SaaS 模型
    - fallback_service        # 降级服务
    
  monitoring:                 # 监控
    - prometheus              # 指标采集
    - grafana                 # 可视化面板
    - alert_manager           # 告警通知
```


## 二、项目搭建：Spring Cloud Gateway 基座

### 2.1 Maven 依赖

```xml
<dependencies>
    <!-- Spring Cloud Gateway -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-gateway</artifactId>
    </dependency>
    
    <!-- Redis 限流 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis-reactive</artifactId>
    </dependency>
    
    <!-- Resilience4j 熔断 -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-circuitbreaker-reactor-resilience4j</artifactId>
    </dependency>
    
    <!-- 监控 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-registry-prometheus</artifactId>
    </dependency>
    
    <!-- JWT -->
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-api</artifactId>
        <version>0.12.5</version>
    </dependency>
</dependencies>
```

### 2.2 主配置

```yaml
# application.yml
server:
  port: 8080

spring:
  cloud:
    gateway:
      routes:
        # OpenAI 官方 API 路由
        - id: openai-route
          uri: https://api.openai.com
          predicates:
            - Path=/api/openai/**
          filters:
            - StripPrefix=1
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 50
                redis-rate-limiter.burstCapacity: 100
            - name: CircuitBreaker
              args:
                name: openai-cb
                fallbackUri: forward:/fallback/model

        # vLLM 自部署集群
        - id: vllm-route
          uri: lb://vllm-cluster
          predicates:
            - Path=/api/vllm/**
          filters:
            - StripPrefix=1
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 200
                redis-rate-limiter.burstCapacity: 400

        # Ollama 多实例（负载均衡）
        - id: ollama-route
          uri: lb://ollama-cluster
          predicates:
            - Path=/api/ollama/**
          filters:
            - StripPrefix=1

      default-filters:
        - name: LLMAuthFilter
        - name: CostTrackingFilter

  data:
    redis:
      host: localhost
      port: 6379
      password: ${REDIS_PASSWORD}
      lettuce:
        pool:
          max-active: 20
          max-idle: 10

# 模型定价（USD per 1K tokens）
llm:
  pricing:
    gpt-4o:
      input: 0.005
      output: 0.015
    gpt-4o-mini:
      input: 0.00015
      output: 0.0006
    qwen-max:
      input: 0.0028
      output: 0.0112
    self-deployed:
      input: 0.00001
      output: 0.00003

  # 限流配置
  rate-limit:
    default:
      requests-per-second: 20
      tokens-per-day: 1000000
    premium:
      requests-per-second: 100
      tokens-per-day: 10000000

management:
  endpoints:
    web:
      exposure:
        include: health,metrics,prometheus
```


## 三、统一鉴权过滤器

### 3.1 多级鉴权体系

```java
/**
 * 统一鉴权过滤器
 * 
 * 鉴权层级（从外向内）：
 * 1. API Key 验证：应用级认证
 * 2. 用户身份验证：JWT Token 解析
 * 3. 模型权限检查：该用户是否允许使用该模型
 * 4. 配额检查：该用户/API Key 的剩余配额
 */
@Component
public class LLMAuthFilter implements GlobalFilter, Ordered {

    @Autowired
    private ApiKeyService apiKeyService;
    
    @Autowired
    private UserQuotaService quotaService;
    
    @Autowired
    private ModelPermissionService permissionService;

    private static final List<String> PUBLIC_PATHS = 
            List.of("/health", "/actuator", "/fallback");

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String path = exchange.getRequest().getURI().getPath();
        
        // 跳过公开路径
        if (PUBLIC_PATHS.stream().anyMatch(path::startsWith)) {
            return chain.filter(exchange);
        }
        
        // 1. 提取 API Key
        String apiKey = extractApiKey(exchange);
        if (apiKey == null) {
            return unauthorized(exchange, "Missing API Key");
        }
        
        // 2. 验证 API Key（查 Redis/DB）
        return apiKeyService.validate(apiKey)
            .flatMap(isValid -> {
                if (!isValid) {
                    return unauthorized(exchange, "Invalid API Key");
                }
                
                // 3. 解析用户身份
                String userId = extractUserId(exchange);
                
                // 4. 提取目标模型
                String model = extractModel(exchange);
                
                // 5. 检查模型权限
                return permissionService.canAccess(userId, model)
                    .flatMap(canAccess -> {
                        if (!canAccess) {
                            return forbidden(exchange, 
                                "No permission for model: " + model);
                        }
                        
                        // 6. 注入身份信息到 Header（传给下游）
                        exchange.getRequest().mutate()
                            .header("X-User-Id", userId)
                            .header("X-Api-Key-Id", apiKey)
                            .header("X-Model", model)
                            .build();
                        
                        return chain.filter(exchange);
                    });
            });
    }

    private String extractApiKey(ServerWebExchange exchange) {
        // 优先从 Header 取
        List<String> auth = exchange.getRequest()
                .getHeaders().get("Authorization");
        if (auth != null && !auth.isEmpty()) {
            String bearer = auth.get(0);
            if (bearer.startsWith("Bearer ")) {
                return bearer.substring(7);
            }
        }
        // 也支持 x-api-key
        List<String> apiKeyHeader = exchange.getRequest()
                .getHeaders().get("x-api-key");
        return apiKeyHeader != null && !apiKeyHeader.isEmpty() 
                ? apiKeyHeader.get(0) : null;
    }

    private String extractModel(ServerWebExchange exchange) {
        return exchange.getRequest().getHeaders()
                .getOrDefault("X-Model", List.of("gpt-4o-mini")).get(0);
    }

    private String extractUserId(ServerWebExchange exchange) {
        return exchange.getRequest().getHeaders()
                .getOrDefault("X-User-Id", List.of("anonymous")).get(0);
    }

    private <T> Mono<Void> unauthorized(ServerWebExchange exchange, String msg) {
        exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
        return writeJsonResponse(exchange, Map.of("error", msg));
    }

    private <T> Mono<Void> forbidden(ServerWebExchange exchange, String msg) {
        exchange.getResponse().setStatusCode(HttpStatus.FORBIDDEN);
        return writeJsonResponse(exchange, Map.of("error", msg));
    }

    private <T> Mono<Void> writeJsonResponse(
            ServerWebExchange exchange, Object body) {
        try {
            byte[] bytes = new ObjectMapper().writeValueAsBytes(body);
            exchange.getResponse().getHeaders()
                    .setContentType(MediaType.APPLICATION_JSON);
            return exchange.getResponse()
                    .writeWith(Mono.just(exchange.getResponse()
                        .bufferFactory().wrap(bytes)));
        } catch (Exception e) {
            return Mono.error(e);
        }
    }

    @Override
    public int getOrder() {
        return -100; // 最高优先级
    }
}
```

### 3.2 API Key 管理服务

```java
@Service
public class ApiKeyService {

    @Autowired
    private StringRedisTemplate redisTemplate;
    
    private static final String API_KEY_PREFIX = "llm:apikey:";
    private static final String API_KEY_HASH_ALIAS = "llm:apikey:alias";

    /**
     * 验证 API Key 是否有效
     */
    public Mono<Boolean> validate(String apiKey) {
        String sha256 = DigestUtils.sha256Hex(apiKey);
        return redisTemplate.hasKey(API_KEY_PREFIX + sha256);
    }

    /**
     * 创建 API Key（管理员用）
     */
    public String createApiKey(String userId, String tier, 
                                long dailyTokenLimit) {
        String rawKey = "llm-" + UUID.randomUUID().toString().replace("-", "");
        String sha256 = DigestUtils.sha256Hex(rawKey);
        
        Map<String, String> metadata = Map.of(
            "user_id", userId,
            "tier", tier,
            "daily_token_limit", String.valueOf(dailyTokenLimit),
            "created_at", String.valueOf(System.currentTimeMillis())
        );
        
        // 存储 Key 元数据
        redisTemplate.opsForHash().putAll(
                API_KEY_PREFIX + sha256, metadata);
        
        // 设置过期时间（90天）
        redisTemplate.expire(API_KEY_PREFIX + sha256, 
                Duration.ofDays(90));
        
        return rawKey;
    }

    /**
     * 吊销 API Key
     */
    public void revokeApiKey(String rawKey) {
        String sha256 = DigestUtils.sha256Hex(rawKey);
        redisTemplate.delete(API_KEY_PREFIX + sha256);
    }
}
```


## 四、多级限流体系

### 4.1 三层限流模型

```java
/**
 * 三层限流：
 * Layer 1: 全局限流（Nginx/Redis 令牌桶）
 * Layer 2: API Key 级别限流（QPS + 日Token额度）
 * Layer 3: 模型级别限流（保护后端模型不被打垮）
 */
@Component
public class MultiLevelRateLimiter {

    @Autowired
    private StringRedisTemplate redisTemplate;
    
    private static final String RATE_KEY_PREFIX = "llm:ratelimit:";

    /**
     * 检查是否超过限流阈值
     * 
     * @param apiKey   API Key
     * @param model    目标模型
     * @param qpsLimit QPS 上限
     * @return 是否允许通过
     */
    public Mono<Boolean> isAllowed(String apiKey, String model, 
                                    int qpsLimit) {
        
        // 滑动窗口限流（1秒窗口）
        String key = RATE_KEY_PREFIX + apiKey + ":" + model;
        long now = System.currentTimeMillis() / 1000;
        long windowStart = now;
        
        // 使用 sorted set 实现滑动窗口
        return redisTemplate.opsForZSet()
                .removeRangeByScore(key, 0, windowStart - 1)
            .then(redisTemplate.opsForZSet()
                .zCard(key))
            .flatMap(currentCount -> {
                if (currentCount >= qpsLimit) {
                    return Mono.just(false);
                }
                return redisTemplate.opsForZSet()
                        .add(key, String.valueOf(now + "-" + UUID.randomUUID()), now)
                    .then(redisTemplate.expire(key, Duration.ofSeconds(2)))
                    .thenReturn(true);
            });
    }

    /**
     * Token 额度限流（每日额度）
     */
    public Mono<TokenQuota> checkQuota(String apiKey) {
        String quotaKey = "llm:quota:daily:" + apiKey;
        String todayKey = LocalDate.now().toString();
        
        return redisTemplate.opsForHash()
                .get(quotaKey, todayKey)
                .defaultIfEmpty("0")
                .map(value -> {
                    long used = Long.parseLong(value.toString());
                    long limit = getDailyTokenLimit(apiKey);
                    return new TokenQuota(used, limit, limit - used);
                });
    }

    private long getDailyTokenLimit(String apiKey) {
        // 从数据库/配置中读取该 API Key 的日 Token 限额
        return 1_000_000; // 默认 100 万 Token/天
    }

    @Data
    @AllArgsConstructor
    public static class TokenQuota {
        private long used;
        private long limit;
        private long remaining;
    }
}
```

### 4.2 限流过滤器

```java
@Component
public class RateLimitFilter implements GlobalFilter, Ordered {

    @Autowired
    private MultiLevelRateLimiter rateLimiter;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, 
                              GatewayFilterChain chain) {
        
        String apiKey = extractHeader(exchange, "X-Api-Key-Id");
        String model = extractHeader(exchange, "X-Model");
        
        if (apiKey == null || model == null) {
            return chain.filter(exchange);
        }
        
        // 1. QPS 限流
        int qpsLimit = getQpsLimit(apiKey);
        return rateLimiter.isAllowed(apiKey, model, qpsLimit)
            .flatMap(allowed -> {
                if (!allowed) {
                    return rateLimited(exchange, 
                        "QPS limit exceeded: " + qpsLimit);
                }
                
                // 2. Token 额度检查
                return rateLimiter.checkQuota(apiKey)
                    .flatMap(quota -> {
                        if (quota.getRemaining() <= 0) {
                            return rateLimited(exchange,
                                "Daily token quota exhausted: " 
                                + quota.getLimit());
                        }
                        
                        // 3. 注入配额信息
                        exchange.getRequest().mutate()
                            .header("X-Token-Remaining", 
                                String.valueOf(quota.getRemaining()))
                            .build();
                        
                        return chain.filter(exchange);
                    });
            });
    }

    private int getQpsLimit(String apiKey) {
        // 根据 API Key 的 tier 返回不同限流值
        // Premium: 100, Standard: 20, Free: 5
        return 20;
    }

    private Mono<Void> rateLimited(ServerWebExchange exchange, String msg) {
        exchange.getResponse().setStatusCode(HttpStatus.TOO_MANY_REQUESTS);
        exchange.getResponse().getHeaders()
                .add("Retry-After", "1");
        // 返回 JSON 错误
        exchange.getResponse().getHeaders()
                .setContentType(MediaType.APPLICATION_JSON);
        String body = String.format(
            "{\"error\":{\"type\":\"rate_limit\",\"message\":\"%s\"}}", msg);
        return exchange.getResponse()
                .writeWith(Mono.just(exchange.getResponse()
                    .bufferFactory().wrap(body.getBytes())));
    }

    private String extractHeader(ServerWebExchange exchange, String name) {
        List<String> values = exchange.getRequest().getHeaders().get(name);
        return values != null && !values.isEmpty() ? values.get(0) : null;
    }

    @Override
    public int getOrder() {
        return -50;
    }
}
```


## 五、熔断降级机制

### 5.1 多级降级策略

```java
@Configuration
public class CircuitBreakerConfig {

    /**
     * 自定义熔断配置
     * 
     * 特别注意：LLM 服务的熔断与普通微服务不同
     * 1. Token 消耗陡增也算一种"故障"（成本故障）
     * 2. 输出质量下降需要触发降级
     * 3. 降级方案可以是缓存/更便宜的模型
     */
    @Bean
    public ReactiveCircuitBreakerFactory modelCircuitBreaker() {
        ReactiveResilience4JCircuitBreakerFactory factory = 
                new ReactiveResilience4JCircuitBreakerFactory();
        
        factory.configureDefault(id -> new Resilience4JConfigBuilder(id)
                .circuitBreakerConfig(CircuitBreakerConfig.custom()
                    .slidingWindowSize(10)           // 滑动窗口大小
                    .failureRateThreshold(50)         // 失败率阈值 50%
                    .waitDurationInOpenState(Duration.ofSeconds(30))
                    .permittedNumberOfCallsInHalfOpenState(3)
                    .recordExceptions(IOException.class, 
                                      TimeoutException.class,
                                      CostOverrunException.class) // 成本超限也是异常
                    .build())
                .timeLimiterConfig(TimeLimiterConfig.custom()
                    .timeoutDuration(Duration.ofSeconds(60)) // LLM 调用可设长超时
                    .build())
                .build());
        
        return factory;
    }
}

/**
 * 成本超限异常 —— 自定义的"业务故障"
 */
public class CostOverrunException extends RuntimeException {
    private final double actualCost;
    private final double budget;
    
    public CostOverrunException(double actualCost, double budget) {
        super(String.format("Cost overrun: $%.4f > $%.4f", actualCost, budget));
        this.actualCost = actualCost;
        this.budget = budget;
    }
}
```

### 5.2 降级控制器

```java
@RestController
public class FallbackController {

    @Autowired
    private CacheManager cacheManager;

    @Autowired
    private ModelRouter modelRouter;

    /**
     * 熔断/降级处理
     * 
     * 降级策略优先级：
     * 1. 缓存命中 → 返回上次相同问题的答案
     * 2. 切换到便宜模型 → GPT-4 降级到 GPT-4o-mini
     * 3. 返回静态兜底回答
     */
    @PostMapping("/fallback/model")
    public Mono<Map<String, Object>> fallback(ServerWebExchange exchange) {
        
        String originalModel = extractHeader(exchange, "X-Model");
        String userMessage = extractBody(exchange);
        
        // 策略 1：语义缓存
        if (cacheManager != null && userMessage != null) {
            String cached = cacheManager.semanticSearch(userMessage);
            if (cached != null) {
                log.info("Fallback: cache hit for model={}", originalModel);
                return Mono.just(Map.of(
                    "content", cached,
                    "source", "cache",
                    "original_model", originalModel
                ));
            }
        }
        
        // 策略 2：切换到便宜模型
        try {
            String cheapModel = modelRouter.getFallbackModel(originalModel);
            String answer = modelRouter.chat(cheapModel, userMessage);
            log.info("Fallback: routed to {} from {}", cheapModel, originalModel);
            return Mono.just(Map.of(
                "content", answer,
                "source", "fallback_model:" + cheapModel,
                "original_model", originalModel
            ));
        } catch (Exception e) {
            // 策略 3：静态兜底
            log.error("All fallback strategies failed", e);
            return Mono.just(Map.of(
                "content", "抱歉，AI 服务暂时不可用，请稍后再试。",
                "source", "static"
            ));
        }
    }

    private String extractHeader(ServerWebExchange exchange, String name) {
        return exchange.getRequest().getHeaders()
                .getOrDefault(name, List.of("unknown")).get(0);
    }

    private String extractBody(ServerWebExchange exchange) {
        // 从请求体中提取用户消息
        return "fallback query";
    }
}
```


## 六、Token 计费与成本控制

### 6.1 计费过滤器

```java
@Component
public class CostTrackingFilter implements GlobalFilter, Ordered {

    @Autowired
    private CostTracker costTracker;

    @Autowired
    private BudgetGuard budgetGuard;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, 
                              GatewayFilterChain chain) {
        
        long startTime = System.currentTimeMillis();
        
        // 缓存请求体（后面要读 Token 数）
        return cacheRequestBody(exchange)
            .then(chain.filter(exchange))
            .then(Mono.fromRunnable(() -> {
                // 在响应返回后记录成本
                recordCost(exchange, startTime);
            }));
    }

    private Mono<Void> cacheRequestBody(ServerWebExchange exchange) {
        // 通过 ModifyingRequestBody 缓存请求体
        return DataBufferUtils.join(exchange.getRequest().getBody())
            .flatMap(dataBuffer -> {
                byte[] bytes = new byte[dataBuffer.readableByteCount()];
                dataBuffer.read(bytes);
                DataBufferUtils.release(dataBuffer);
                
                // 解析 Token 用量（可以在路由前做预算检查）
                String body = new String(bytes, StandardCharsets.UTF_8);
                exchange.getAttributes().put("request_body", body);
                
                // 重新设置请求体
                DefaultDataBufferFactory factory = 
                        new DefaultDataBufferFactory();
                return Mono.just(exchange)
                    .doOnNext(ex -> ex.getRequest().mutate()
                        .body(Mono.just(factory.wrap(bytes)))
                        .build());
            })
            .then();
    }

    private void recordCost(ServerWebExchange exchange, long startTime) {
        String apiKey = getHeader(exchange, "X-Api-Key-Id");
        String userId = getHeader(exchange, "X-User-Id");
        String model = getHeader(exchange, "X-Model");
        int status = exchange.getResponse().getStatusCode() != null 
                ? exchange.getResponse().getStatusCode().value() : 500;
        long latency = System.currentTimeMillis() - startTime;

        // 从 response body 中提取 token 数
        // 实际实现需要在 ModifyResponseBody 中读取
        
        costTracker.record(apiKey, userId, model, status, latency);
        
        // 成本拦截：单次调用超过阈值时告警
        budgetGuard.checkAndAlert(apiKey, model);
    }

    @Override
    public int getOrder() {
        return -10;
    }
}
```

### 6.2 成本追踪服务

```java
@Service
@Slf4j
public class CostTracker {

    @Autowired
    private MeterRegistry meterRegistry;

    @Autowired
    private StringRedisTemplate redisTemplate;

    private static final Map<String, Double> INPUT_PRICING = Map.of(
        "gpt-4o", 0.005,
        "gpt-4o-mini", 0.00015,
        "claude-3-opus", 0.015,
        "qwen-max", 0.0028
    );

    private static final Map<String, Double> OUTPUT_PRICING = Map.of(
        "gpt-4o", 0.015,
        "gpt-4o-mini", 0.0006,
        "claude-3-opus", 0.075,
        "qwen-max", 0.0112
    );

    public void recordUsage(String apiKey, String model, 
                             int inputTokens, int outputTokens) {
        double cost = calculateCost(model, inputTokens, outputTokens);
        
        // 1. Prometheus 指标
        meterRegistry.counter("llm.tokens.input", 
                "model", model).increment(inputTokens);
        meterRegistry.counter("llm.tokens.output", 
                "model", model).increment(outputTokens);
        meterRegistry.counter("llm.cost.total", 
                "model", model).increment(cost);
        
        // 2. Redis 实时成本
        String dateKey = LocalDate.now().toString();
        String hashKey = "llm:daily_cost:" + dateKey;
        redisTemplate.opsForHash().increment(hashKey, apiKey, cost);
        redisTemplate.expire(hashKey, Duration.ofDays(7));
        
        // 3. 记录详细日志
        log.info("LLM cost: model={}, in={}, out={}, cost=${}", 
                model, inputTokens, outputTokens, 
                String.format("%.6f", cost));
    }

    public double calculateCost(String model, int inputTokens, 
                                 int outputTokens) {
        double inputPrice = INPUT_PRICING.getOrDefault(model, 0.001);
        double outputPrice = OUTPUT_PRICING.getOrDefault(model, 0.002);
        return (inputTokens * inputPrice + outputTokens * outputPrice) 
                / 1000.0;
    }

    public Mono<Double> getTodayCost(String apiKey) {
        String dateKey = LocalDate.now().toString();
        String hashKey = "llm:daily_cost:" + dateKey;
        return redisTemplate.opsForHash()
                .get(hashKey, apiKey)
                .map(v -> Double.parseDouble(v.toString()))
                .defaultIfEmpty(0.0);
    }
}
```

### 6.3 预算熔断

```java
@Component
@Slf4j
public class BudgetGuard {

    @Autowired
    private CostTracker costTracker;

    @Autowired
    private AlertService alertService;

    private static final double DAILY_BUDGET = 100.0; // 日预算 $100
    private static final double SINGLE_CALL_BUDGET = 5.0; // 单次调用上限 $5

    /**
     * 每日预算检查：超过预算时触发告警
     */
    public void checkAndAlert(String apiKey, String model) {
        costTracker.getTodayCost(apiKey)
            .subscribe(dailyCost -> {
                if (dailyCost > DAILY_BUDGET * 0.8) {
                    log.warn("API Key {} 已使用每日预算的 80%: ${}", 
                            apiKey, String.format("%.2f", dailyCost));
                }
                if (dailyCost > DAILY_BUDGET) {
                    log.error("API Key {} 超过每日预算: ${}", 
                            apiKey, String.format("%.2f", dailyCost));
                    alertService.sendAlert(String.format(
                        "预算熔断：%s 日费用 $%.2f 超过预算 $%.2f",
                        apiKey, dailyCost, DAILY_BUDGET));
                }
            });
    }

    /**
     * 单次调用成本预估：在调用前拦截大请求
     */
    public boolean approveRequest(String model, int estimatedTokens) {
        double estimatedCost = costTracker.calculateCost(
                model, estimatedTokens, estimatedTokens);
        if (estimatedCost > SINGLE_CALL_BUDGET) {
            log.warn("Rejected oversized request: model={}, " 
                    + "estTokens={}, estCost=${}", 
                    model, estimatedTokens, 
                    String.format("%.4f", estimatedCost));
            return false;
        }
        return true;
    }
}
```


## 七、安全防护

### 7.1 Prompt 注入检测

```java
@Component
public class PromptInjectionDetector {

    // 已知的注入模式
    private static final List<String> INJECTION_PATTERNS = List.of(
        "ignore (all |previous )?(instructions|rules|guidelines)",
        "forget (all |previous )?(instructions|rules|guidelines)",
        "you are now (DAN|STAN|a different )",
        "system prompt:",
        "<<SYS>>",
        "\\[INST\\]",
        "new instructions:",
        "pretend (you are|to be)",
        "you must (ignore|forget|disregard)"
    );

    private final List<Pattern> compiledPatterns = INJECTION_PATTERNS.stream()
            .map(p -> Pattern.compile(p, Pattern.CASE_INSENSITIVE))
            .toList();

    /**
     * 检测用户输入中是否包含 Prompt 注入
     * 
     * @return 风险评分 0-100，>50 建议拦截
     */
    public int detectRisk(String userInput) {
        if (userInput == null || userInput.isBlank()) {
            return 0;
        }
        
        int riskScore = 0;
        
        // 1. 匹配已知注入模式
        for (Pattern pattern : compiledPatterns) {
            if (pattern.matcher(userInput).find()) {
                riskScore += 30;
            }
        }
        
        // 2. 检测系统指令类内容（用户输入中包含大量指令性语言）
        long instructionKeywords = countMatches(userInput,
                "must|should|required|always|never|if you|as a|act as");
        riskScore += instructionKeywords * 5;
        
        // 3. 检测分隔符注入（尝试闭合 Prompt）
        if (userInput.contains("---") || userInput.contains("```")) {
            riskScore += 20;
        }
        
        // 4. 超长输入可能是注入
        if (userInput.length() > 4096) {
            riskScore += 15;
        }
        
        return Math.min(100, riskScore);
    }

    private long countMatches(String text, String regex) {
        return Pattern.compile(regex, Pattern.CASE_INSENSITIVE)
                .matcher(text).results().count();
    }
}
```

### 7.2 安全过滤器

```java
@Component
public class SecurityFilter implements GlobalFilter, Ordered {

    @Autowired
    private PromptInjectionDetector injectionDetector;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, 
                              GatewayFilterChain chain) {
        
        // 从请求体中提取用户消息
        return extractUserMessage(exchange)
            .flatMap(userMessage -> {
                int risk = injectionDetector.detectRisk(userMessage);
                
                if (risk > 80) {
                    // 高风险：直接拒绝
                    return rejectRequest(exchange, "Suspected prompt injection", risk);
                } else if (risk > 50) {
                    // 中风险：标记并放行（后续可以人工审核）
                    log.warn("Potential injection attempt, risk={}, message={}", 
                            risk, userMessage.substring(0, Math.min(100, userMessage.length())));
                    exchange.getRequest().mutate()
                        .header("X-Security-Risk", String.valueOf(risk));
                }
                
                return chain.filter(exchange);
            });
    }

    private Mono<String> extractUserMessage(ServerWebExchange exchange) {
        String body = exchange.getAttribute("request_body");
        if (body != null) {
            try {
                JsonNode node = new ObjectMapper().readTree(body);
                if (node.has("messages")) {
                    JsonNode lastMessage = node.get("messages")
                            .get(node.get("messages").size() - 1);
                    return Mono.just(lastMessage.get("content").asText());
                }
                if (node.has("prompt")) {
                    return Mono.just(node.get("prompt").asText());
                }
            } catch (Exception e) {
                return Mono.just(body);
            }
        }
        return Mono.just("");
    }

    private Mono<Void> rejectRequest(ServerWebExchange exchange, 
                                      String reason, int risk) {
        log.warn("Request rejected: reason={}, risk={}", reason, risk);
        exchange.getResponse().setStatusCode(HttpStatus.FORBIDDEN);
        exchange.getResponse().getHeaders()
                .setContentType(MediaType.APPLICATION_JSON);
        String body = String.format(
            "{\"error\":{\"type\":\"security\",\"risk\":%d,\"message\":\"%s\"}}",
            risk, reason);
        return exchange.getResponse()
                .writeWith(Mono.just(exchange.getResponse()
                        .bufferFactory().wrap(body.getBytes())));
    }

    @Override
    public int getOrder() {
        return -80;
    }
}
```


## 八、监控大盘

```java
@Component
public class LLMGatewayMetrics {

    @Autowired
    private MeterRegistry meterRegistry;

    /**
     * 注册所有网关指标
     */
    @PostConstruct
    public void registerMetrics() {
        // 请求计数器
        Counter.builder("llm.gateway.requests.total")
                .tag("type", "gateway")
                .description("Total LLM gateway requests")
                .register(meterRegistry);

        // 延迟直方图
        Timer.builder("llm.gateway.request.duration")
                .description("Request duration in milliseconds")
                .publishPercentiles(0.5, 0.90, 0.95, 0.99)
                .register(meterRegistry);

        // 限流拒绝计数器
        Counter.builder("llm.gateway.rate_limited")
                .description("Requests denied by rate limiter")
                .register(meterRegistry);

        // 安全拦截计数器
        Counter.builder("llm.gateway.security_blocks")
                .description("Requests blocked by security filter")
                .register(meterRegistry);

        // 熔断状态
        Gauge.builder("llm.gateway.circuit_breaker.state", 
                      this, Metrics::getCircuitBreakerState)
                .description("Circuit breaker state (0=closed, 1=open, 2=half-open)")
                .register(meterRegistry);
    }

    public void recordRequest(String model, String status, long latencyMs) {
        meterRegistry.counter("llm.gateway.requests.total",
                "model", model,
                "status", status).increment();
        
        Timer.builder("llm.gateway.request.duration")
                .tag("model", model)
                .register(meterRegistry)
                .record(latencyMs, TimeUnit.MILLISECONDS);
    }
}
```


## 总结

搭建 LLM API 网关不是过度设计，而是生产环境的必需品。核心要点：

1. **鉴权不止于 API Key**：用户模型权限、配额检查一个不能少
2. **限流要分层**：全局 → API Key → 模型，逐层兜底
3. **熔断不只是连接失败**：成本超限、质量下降都是触发条件
4. **计费要实时**：明天看账单今天已经破产
5. **安全是底线**：Prompt 注入检测必须内置到网关

完成这套网关，你的 AI 服务就有了"防火墙"，放心大胆对外提供服务吧。


---

**下篇预告**：网关建好了，流量进来了。但你知道每一次 LLM 调用的具体耗时吗？哪个环节是瓶颈？下篇文章《**LLM 可观测性：基于 LangFuse 的调用链追踪与成本监控**》，带你用 LangFuse + OpenTelemetry 把每一分钱都算得清清楚楚。花了几百美金买 Token，至少要知道花在哪了吧？
