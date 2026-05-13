# LLM 调用的并发控制：Semaphore + RateLimiter 的最佳实践，别让你的AI服务把下游打挂了

> 本系列文章专注 **Java + AI 工程实践**，我将用真实可运行的代码，系统讲解如何用 Java 构建生产级 AI 应用。如果觉得有帮助，欢迎**点赞、收藏、关注**三连，你的支持是我持续创作的动力！

---

## 一、一个真实的生产事故

凌晨2点，报警群炸了。

我们的智能客服系统刚上线两周，突然大面积不可用。日志显示大量 OpenAI API 返回 `429 Too Many Requests`，紧接着是层层超时，整个服务雪崩。

排查后发现根因很简单：**上游业务放量，并发请求全部打到了同一个 API Key 上，瞬间触发了 OpenAI 的 RPM（每分钟请求数）限制。**

那次事故后，我们给 LLM 调用链路加了三层防护：**Semaphore 并发控制 + Guava RateLimiter 本地限流 + Redis 分布式限流**。从那以后，再也没有因为调用频率问题被下游限流过。

今天这篇文章，我将把这套方案完整分享给你。

---

## 二、为什么要控制 LLM API 的并发？

### 2.1 厂商的硬限制

几乎所有大模型厂商都对 API 调用有严格的频率和并发限制：

| 厂商 | RPM 限制（免费/基础版） | TPM 限制 | 并发数限制 |
|------|------------------------|----------|-----------|
| OpenAI | 3 RPM（免费）/500 RPM（Tier 1） | 10K TPM（免费）/200K TPM（Tier 1） | 无硬并发限制，但受 RPM 约束 |
| 智谱（GLM） | 60 RPM | 100K TPM | 不建议超过 50 并发 |
| 百川 | 100 RPM | 300K TPM | 30 并发 |
| DeepSeek | 100 RPM | 500K TPM | 50 并发 |
| 通义千问 | 100 RPM | 200K TPM | 60 并发 |
| Anthropic Claude | 50 RPM（Tier 1） | 50K TPM（Tier 1） | 无硬限制 |

> 注：以上数据时间为 2026 年初，具体限制请以各厂商最新文档为准。

**RPM（Requests Per Minute）**：每分钟最大请求数。
**TPM（Tokens Per Minute）**：每分钟最大 Token 消耗数（输入+输出）。

这两个限制是独立生效的，**只要触及任意一个，就会收到 429 错误**。

### 2.2 不控制的后果

我们来看一个典型的场景：

```java
// 危险！没有任何限流保护
@PostMapping("/chat")
public String chat(@RequestBody ChatRequest request) {
    return openAiClient.call(new Prompt(request.getQuestion())).getResult();
}
```

假设你的服务有 100 个用户同时调用 `/chat` 接口，每个请求都会创建一个与 OpenAI 的 HTTP 连接。如果 `gpt-4o-mini` 的响应时间平均为 2 秒，那么：

- 每分钟并发请求数 = 100 × (60 / 2) = 3000 次
- 但你的 API Key 可能只有 **500 RPM** 的配额

结果就是：前 500 个请求正常，后面 2500 个请求全部返回 429。更糟糕的是，这些失败的请求如果做了重试，会进一步恶化局面，形成恶性循环。

**这就是典型的"上游放量，下游崩溃"问题。**

### 2.3 并发控制的本质

并发控制的本质不是"拒绝请求"，而是 **"以被调方能承受的速度发送请求"**。

这背后涉及三个关键概念：

1. **并发数控制**：同一时刻最多有多少个请求在执行（Semaphore）
2. **频率限流**：单位时间内最多发送多少个请求（RateLimiter）
3. **流量整形**：将突发流量平滑为匀速流量（令牌桶）

接下来，我将逐一讲解这三个方案在 Java 中的实现。

---

## 三、第一道防线：Semaphore 控制并发数

### 3.1 原理

Java 的 `Semaphore`（信号量）是最简单也最可靠的并发控制工具。它的核心思想是：**许可证（Permit）机制**。

- 初始化时设定 N 个许可证
- 每个线程执行前 `acquire()` 获取一个许可证
- 如果许可证都被占用，后续线程必须等待
- 执行完毕后 `release()` 归还许可证

### 3.2 基础实现

```java
import java.util.concurrent.*;

public class SemaphoreLimiter {
    private final Semaphore semaphore;

    public SemaphoreLimiter(int maxConcurrent) {
        this.semaphore = new Semaphore(maxConcurrent);
    }

    public <T> T executeWithLimit(Callable<T> task, long timeout, TimeUnit unit) 
            throws Exception {
        if (!semaphore.tryAcquire(timeout, unit)) {
            throw new RuntimeException("获取许可证超时，当前并发已满");
        }
        try {
            return task.call();
        } finally {
            semaphore.release();
        }
    }
}
```

使用方式：

```java
SemaphoreLimiter limiter = new SemaphoreLimiter(10); // 最多10并发

@GetMapping("/chat")
public Response chat(@RequestParam String question) {
    return limiter.executeWithLimit(() -> {
        return llmClient.chat(question);
    }, 30, TimeUnit.SECONDS); // 等待最多30秒
}
```

### 3.3 进阶：公平信号量

```java
// 使用公平模式，防止某些请求长时间等待（饥饿问题）
private final Semaphore semaphore = new Semaphore(maxConcurrent, true);
```

### 3.4 Semaphore 的局限性

Semaphore 只限制**同时执行**的请求数，但不关心**频率**。如果每个请求耗时 100ms，10 个许可证在 1 分钟内可以完成 10 × (60000/100) = 6000 个请求，远超厂商的 RPM 限制。

**所以还需要第二道防线——频率限流**。

---

## 四、第二道防线：Guava RateLimiter 频率限流

### 4.1 引入依赖

```xml
<dependency>
    <groupId>com.google.guava</groupId>
    <artifactId>guava</artifactId>
    <version>33.0.0-jre</version>
</dependency>
```

### 4.2 Guava RateLimiter 原理

Guava 的 `RateLimiter` 基于**令牌桶算法**，核心参数是 `QPS`（每秒许可数）。

```java
// 每秒生成100个令牌，即每分钟6000个请求
RateLimiter rateLimiter = RateLimiter.create(100);
```

关键特性：
- **平滑突发限流**（SmoothBursty）：冷启动时可以突发使用存储的令牌
- **平滑预热限流**（SmoothWarmingUp）：系统冷启动时逐步增加速率

### 4.3 完整实现

```java
import com.google.common.util.concurrent.RateLimiter;
import java.util.concurrent.*;
import java.util.function.Supplier;

public class CombinedLimiter {
    // 信号量控制并发数
    private final Semaphore semaphore;
    // RateLimiter 控制请求频率
    private final RateLimiter rateLimiter;
    // 超时等待时间
    private final long timeoutSeconds;

    public CombinedLimiter(int maxConcurrent, double maxQPS, long timeoutSeconds) {
        this.semaphore = new Semaphore(maxConcurrent, true);
        this.rateLimiter = RateLimiter.create(maxQPS);
        this.timeoutSeconds = timeoutSeconds;
    }

    /**
     * 带双重限流的执行方法
     * @param supplier 实际要执行的任务
     * @return 执行结果
     */
    public <T> T execute(Supplier<T> supplier) {
        // 第一道防线：频率限流（阻塞直到获取到令牌）
        rateLimiter.acquire();

        // 第二道防线：并发数控制
        if (!semaphore.tryAcquire(timeoutSeconds, TimeUnit.SECONDS)) {
            throw new RateLimitException("服务繁忙，请稍后重试（并发已满）");
        }

        try {
            return supplier.get();
        } finally {
            semaphore.release();
        }
    }

    /**
     * 非阻塞尝试，获取不到令牌时立即返回
     */
    public <T> Optional<T> tryExecute(Supplier<T> supplier) {
        if (!rateLimiter.tryAcquire()) {
            return Optional.empty();
        }
        if (!semaphore.tryAcquire()) {
            rateLimiter.acquire(); // 这里需要退还令牌吗？不需要，令牌不退还
            return Optional.empty();
        }
        try {
            return Optional.ofNullable(supplier.get());
        } finally {
            semaphore.release();
        }
    }

    // 自定义异常
    public static class RateLimitException extends RuntimeException {
        public RateLimitException(String message) {
            super(message);
        }
    }
}
```

### 4.4 实战中的参数配置

```yaml
llm:
  openai:
    rpm-limit: 500       # 每分钟最多500次请求
    tpm-limit: 200000    # 每分钟最多20万token
    max-concurrent: 10   # 最大并发10个请求
    timeout-seconds: 30  # 等待超时
```

```java
@Configuration
public class RateLimitConfig {

    @Bean
    public CombinedLimiter openAiLimiter(
            @Value("${llm.openai.max-concurrent}") int maxConcurrent,
            @Value("${llm.openai.rpm-limit}") int rpmLimit,
            @Value("${llm.openai.timeout-seconds}") long timeoutSeconds) {
        // 将 RPM 转换为 QPS（每秒请求数）
        double maxQPS = (double) rpmLimit / 60.0;
        return new CombinedLimiter(maxConcurrent, maxQPS, timeoutSeconds);
    }
}
```

### 4.5 处理 429 错误的指数退避重试

即使有了限流，偶尔还是可能收到 429。稳妥的做法是加上重试：

```java
import org.springframework.retry.annotation.Backoff;
import org.springframework.retry.annotation.Retryable;
import org.springframework.retry.annotation.Recover;

@Service
public class OpenAiService {

    @Retryable(
        retryFor = {RateLimitException.class, HttpStatusCodeException.class},
        maxAttempts = 3,
        backoff = @Backoff(delay = 1000, multiplier = 2.0) 
        // 1秒、2秒、4秒 逐次递增
    )
    public String chat(String question) {
        return combinedLimiter.execute(() -> {
            ResponseEntity<String> response = restTemplate.postForEntity(
                "https://api.openai.com/v1/chat/completions", 
                buildRequest(question), 
                String.class);
            
            if (response.getStatusCode() == HttpStatus.TOO_MANY_REQUESTS) {
                throw new RateLimitException("下游服务限流");
            }
            return response.getBody();
        });
    }

    @Recover
    public String chatFallback(RateLimitException e, String question) {
        return "服务繁忙，请稍后再试。错误信息：" + e.getMessage();
    }
}
```

---

## 五、第三道防线：Bucket4j 令牌桶高级限流

Guava RateLimiter 简单易用，但有两个不足：
1. **不支持分钟级别的配置**（只支持 QPS）
2. **不支持突发与平均的精细化控制**

`Bucket4j` 是更强大的令牌桶实现，支持复杂的限流策略。

### 5.1 引入依赖

```xml
<dependency>
    <groupId>com.github.vladimir-bukhtoyarov</groupId>
    <artifactId>bucket4j-core</artifactId>
    <version>8.7.0</version>
</dependency>
```

### 5.2 Bucket4j 实战实现

```java
import io.github.bucket4j.Bandwidth;
import io.github.bucket4j.Bucket;
import io.github.bucket4j.Refill;

import java.time.Duration;

public class Bucket4jLimiter {
    
    private final Semaphore semaphore;
    private final Bucket rpmBucket;  // 每分钟请求数
    private final Bucket tpmBucket;  // 每分钟Token数
    private final long timeoutSeconds;

    /**
     * @param maxConcurrent 最大并发数
     * @param rpmLimit 每分钟最大请求数
     * @param tpmLimit 每分钟最大Token数
     */
    public Bucket4jLimiter(int maxConcurrent, long rpmLimit, long tpmLimit, 
                           long timeoutSeconds) {
        this.semaphore = new Semaphore(maxConcurrent, true);
        this.timeoutSeconds = timeoutSeconds;

        // 每分钟补充一次令牌
        Bandwidth rpmBandwidth = Bandwidth.classic(rpmLimit, 
            Refill.intervally(rpmLimit, Duration.ofMinutes(1)));
        this.rpmBucket = Bucket.builder().addLimit(rpmBandwidth).build();

        Bandwidth tpmBandwidth = Bandwidth.classic(tpmLimit,
            Refill.intervally(tpmLimit, Duration.ofMinutes(1)));
        this.tpmBucket = Bucket.builder().addLimit(tpmBandwidth).build();
    }

    /**
     * 执行LLM调用，同时控制RPM和TPM
     * @param estimatedTokens 预估的Token消耗（输入+输出）
     */
    public <T> T execute(Supplier<T> task, long estimatedTokens) {
        // RPM限制
        if (!rpmBucket.tryConsume(1, Duration.ofSeconds(timeoutSeconds))) {
            throw new RateLimitException("RPM限制已达上限");
        }

        // TPM限制
        if (!tpmBucket.tryConsume(estimatedTokens, Duration.ofSeconds(timeoutSeconds))) {
            // 消耗了RPM令牌但TPM不够，返还RPM令牌
            rpmBucket.addTokens(1);
            throw new RateLimitException("TPM限制已达上限");
        }

        // 并发数限制
        if (!semaphore.tryAcquire(timeoutSeconds, TimeUnit.SECONDS)) {
            // 返还两个令牌
            rpmBucket.addTokens(1);
            tpmBucket.addTokens(estimatedTokens);
            throw new RateLimitException("并发已满");
        }

        try {
            return task.get();
        } finally {
            semaphore.release();
        }
    }
}
```

### 5.3 Bucket4j 的优势

```java
// 设置每分钟500个请求，但允许瞬时60个请求的突发
Bandwidth burstyRefill = Bandwidth.classic(500,
    Refill.intervallyAligned(500, Duration.ofMinutes(1)));

// 设置"首次使用后1分钟补充500个"而不是"每1分钟重置为500"
Bandwidth slidingWindow = Bandwidth.simple(500, Duration.ofMinutes(1));
```

---

## 六、背压处理策略

有了限流，必然有请求被拒绝。如何优雅地处理被拒绝的请求？

### 6.1 四种背压策略

```java
public enum BackpressureStrategy {
    /**
     * 快速失败：立即返回错误，由调用方决定重试
     */
    FAIL_FAST,

    /**
     * 阻塞等待：在超时时间内等待，适合同步调用
     */
    BLOCKING,

    /**
     * 排队处理：将请求放入队列，异步消费
     */
    QUEUEING,

    /**
     * 降级兜底：返回缓存结果或默认响应
     */
    FALLBACK
}
```

### 6.2 完整背压实现

```java
import java.util.concurrent.*;
import java.util.function.Function;

public class BackpressureHandler<T> {

    // 队列模式：有界阻塞队列
    private final BlockingQueue<RequestWrapper<T>> queue;
    private final ExecutorService consumer;

    public BackpressureHandler(int queueCapacity, Function<String, T> llmFunction) {
        this.queue = new LinkedBlockingQueue<>(queueCapacity);
        this.consumer = Executors.newFixedThreadPool(5);
        
        // 启动消费者线程
        for (int i = 0; i < 5; i++) {
            consumer.submit(() -> {
                while (!Thread.currentThread().isInterrupted()) {
                    try {
                        RequestWrapper<T> request = queue.poll(5, TimeUnit.SECONDS);
                        if (request != null) {
                            T result = llmFunction.apply(request.getInput());
                            request.getFuture().complete(result);
                        }
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                        break;
                    }
                }
            });
        }
    }

    /**
     * 根据策略处理请求
     */
    public T handleRequest(String input, BackpressureStrategy strategy, T fallbackResult) {
        switch (strategy) {
            case QUEUEING:
                return handleQueueing(input);
            case FALLBACK:
                return handleFallback(input, fallbackResult);
            case FAIL_FAST:
            default:
                throw new RateLimitException("请求被限流");
        }
    }

    private T handleQueueing(String input) {
        CompletableFuture<T> future = new CompletableFuture<>();
        RequestWrapper<T> wrapper = new RequestWrapper<>(input, future);
        
        if (!queue.offer(wrapper)) {
            throw new RateLimitException("排队队列已满，请稍后重试");
        }
        
        try {
            return future.get(60, TimeUnit.SECONDS);
        } catch (TimeoutException e) {
            future.cancel(true);
            throw new RateLimitException("排队等待超时");
        } catch (Exception e) {
            throw new RuntimeException("排队处理异常", e);
        }
    }

    private T handleFallback(String input, T fallbackResult) {
        try {
            // 尝试从缓存获取
            return getFromCache(input);
        } catch (Exception e) {
            return fallbackResult;
        }
    }

    private T getFromCache(String input) {
        // 这里接入缓存层（下一篇文章会详细讲）
        return null;
    }

    static class RequestWrapper<T> {
        private final String input;
        private final CompletableFuture<T> future;

        RequestWrapper(String input, CompletableFuture<T> future) {
            this.input = input;
            this.future = future;
        }

        String getInput() { return input; }
        CompletableFuture<T> getFuture() { return future; }
    }
}
```

---

## 七、分布式限流：Redis + 令牌桶

前面的方案都是**单机限流**。如果你的服务部署了 5 个实例，每个实例限流 100 QPS，那么集群实际会向 LLM API 发送 500 QPS，如果厂商限制是 300 QPS，仍然会超限。

**这就是为什么需要分布式限流。**

### 7.1 方案一：Redis Lua 脚本实现令牌桶

```lua
-- rate_limiter.lua
local key = KEYS[1]                    -- 令牌桶的 Redis Key
local capacity = tonumber(ARGV[1])     -- 桶容量
local rate = tonumber(ARGV[2])         -- 填充速率（令牌/秒）
local requested = tonumber(ARGV[3])    -- 请求的令牌数
local now = tonumber(ARGV[4])          -- 当前时间戳（秒）

-- 获取当前桶的状态
local bucket = redis.call('hmget', key, 'tokens', 'last_refill')
local tokens = tonumber(bucket[1])
local last_refill = tonumber(bucket[2])

-- 首次使用，初始化桶
if tokens == nil then
    tokens = capacity
    last_refill = now
end

-- 计算应该补充的令牌数
local elapsed = math.max(0, now - last_refill)
local refill = math.floor(elapsed * rate)
tokens = math.min(capacity, tokens + refill)
last_refill = now

-- 判断是否有足够的令牌
if tokens >= requested then
    tokens = tokens - requested
    redis.call('hmset', key, 'tokens', tokens, 'last_refill', last_refill)
    redis.call('expire', key, 60)  -- 1分钟过期
    return 1  -- 通过
else
    redis.call('hmset', key, 'tokens', tokens, 'last_refill', last_refill)
    return 0  -- 拒绝
end
```

### 7.2 Java 端实现

```java
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.data.redis.core.script.DefaultRedisScript;

import java.util.Collections;

public class RedisTokenBucketLimiter {

    private final StringRedisTemplate redisTemplate;
    private final DefaultRedisScript<Long> rateLimiterScript;

    public RedisTokenBucketLimiter(StringRedisTemplate redisTemplate) {
        this.redisTemplate = redisTemplate;
        this.rateLimiterScript = new DefaultRedisScript<>();
        rateLimiterScript.setScriptText(LUA_SCRIPT);
        rateLimiterScript.setResultType(Long.class);
    }

    private static final String LUA_SCRIPT = 
        "local key = KEYS[1]\n" +
        "local capacity = tonumber(ARGV[1])\n" +
        "local rate = tonumber(ARGV[2])\n" +
        "local requested = tonumber(ARGV[3])\n" +
        "local now = tonumber(ARGV[4])\n" +
        "\n" +
        "local bucket = redis.call('hmget', key, 'tokens', 'last_refill')\n" +
        "local tokens = tonumber(bucket[1])\n" +
        "local last_refill = tonumber(bucket[2])\n" +
        "\n" +
        "if tokens == nil then\n" +
        "    tokens = capacity\n" +
        "    last_refill = now\n" +
        "end\n" +
        "\n" +
        "local elapsed = math.max(0, now - last_refill)\n" +
        "local refill = math.floor(elapsed * rate)\n" +
        "tokens = math.min(capacity, tokens + refill)\n" +
        "last_refill = now\n" +
        "\n" +
        "if tokens >= requested then\n" +
        "    tokens = tokens - requested\n" +
        "    redis.call('hmset', key, 'tokens', tokens, 'last_refill', last_refill)\n" +
        "    redis.call('expire', key, 60)\n" +
        "    return 1\n" +
        "else\n" +
        "    redis.call('hmset', key, 'tokens', tokens, 'last_refill', last_refill)\n" +
        "    return 0\n" +
        "end";

    /**
     * 尝试获取令牌
     */
    public boolean tryAcquire(String resourceKey, int capacity, double ratePerSecond, int requested) {
        Long result = redisTemplate.execute(
            rateLimiterScript,
            Collections.singletonList("rate:limit:" + resourceKey),
            String.valueOf(capacity),
            String.valueOf(ratePerSecond),
            String.valueOf(requested),
            String.valueOf(System.currentTimeMillis() / 1000)
        );
        return result != null && result == 1L;
    }
}
```

### 7.3 方案二：Redisson RRateLimiter

Redisson 提供了开箱即用的分布式限流组件：

```xml
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson</artifactId>
    <version>3.27.1</version>
</dependency>
```

```java
import org.redisson.api.RRateLimiter;
import org.redisson.api.RateIntervalUnit;
import org.redisson.api.RateType;
import org.redisson.api.RedissonClient;

@Service
public class RedissonRateLimiterService {

    private final RedissonClient redissonClient;

    public RedissonRateLimiterService(RedissonClient redissonClient) {
        this.redissonClient = redissonClient;
    }

    public boolean tryAcquire(String apiKey) {
        RRateLimiter limiter = redissonClient.getRateLimiter("llm:ratelimit:" + apiKey);

        // 初始化：每分钟500次请求
        limiter.trySetRate(RateType.OVERALL, 500, 1, RateIntervalUnit.MINUTES);

        return limiter.tryAcquire();
    }
}
```

---

## 八、终极方案：三层限流架构

将以上所有方案整合到一个生产级的三层架构中：

```java
@Component
public class TieredRateLimiter {

    // L1: 本地信号量控制并发数（最快）
    private final Semaphore semaphore;

    // L2: 本地令牌桶控制频率（快）
    private final RateLimiter rateLimiter;

    // L3: 分布式令牌桶控制集群总量（最准）
    @Autowired
    private RedisTokenBucketLimiter redisLimiter;

    public TieredRateLimiter(
            @Value("${llm.max-concurrent:20}") int maxConcurrent,
            @Value("${llm.local-qps:50}") int localQps) {
        this.semaphore = new Semaphore(maxConcurrent, true);
        this.rateLimiter = RateLimiter.create(localQps);
    }

    public <T> T execute(String apiKey, Supplier<T> task, BackpressureStrategy strategy) {
        // 三层逐级限流
        
        // L1: 分布式限流（提前挡住集群超量请求）
        if (!redisLimiter.tryAcquire(apiKey, 500, 500.0 / 60, 1)) {
            return handleRejection(strategy, "集群请求量已达上限");
        }

        // L2: 本地频率限流
        if (!rateLimiter.tryAcquire()) {
            return handleRejection(strategy, "本机请求频率已达上限");
        }

        // L3: 本地并发数控制
        if (!semaphore.tryAcquire(30, TimeUnit.SECONDS)) {
            return handleRejection(strategy, "本机并发数已达上限");
        }

        try {
            return task.get();
        } finally {
            semaphore.release();
        }
    }

    @SuppressWarnings("unchecked")
    private <T> T handleRejection(BackpressureStrategy strategy, String reason) {
        return switch (strategy) {
            case FAIL_FAST -> throw new RateLimitException(reason);
            case FALLBACK -> (T) "抱歉，当前请求量较大，请稍后再试。";
            case QUEUEING -> throw new UnsupportedOperationException("排队策略需提前配置队列");
            case BLOCKING -> throw new UnsupportedOperationException("已在获取令牌时阻塞");
        };
    }
}
```

---

## 九、监控与告警

没有监控的限流是"盲飞"。推荐使用 Micrometer + Prometheus 打点：

```java
import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;

@Component
public class RateLimitMetrics {

    private final Counter requestTotal;
    private final Counter requestRejected;
    private final Timer requestLatency;
    private final AtomicInteger currentConcurrent;

    public RateLimitMetrics(MeterRegistry registry) {
        this.requestTotal = Counter.builder("llm.request.total")
            .description("LLM总请求数").register(registry);
        this.requestRejected = Counter.builder("llm.request.rejected")
            .description("LLM被限流数").register(registry);
        this.requestLatency = Timer.builder("llm.request.latency")
            .description("LLM请求耗时").register(registry);
        this.currentConcurrent = registry.gauge("llm.request.concurrent", 
            new AtomicInteger(0));
    }

    public void recordSuccess(long latencyMs) {
        requestTotal.increment();
        requestLatency.record(latencyMs, java.util.concurrent.TimeUnit.MILLISECONDS);
    }

    public void recordRejected() {
        requestRejected.increment();
    }

    public void incrementConcurrent() {
        currentConcurrent.incrementAndGet();
    }

    public void decrementConcurrent() {
        currentConcurrent.decrementAndGet();
    }
}
```

Grafana 仪表盘关键指标：
- **限流拒绝率** = `llm.request.rejected / llm.request.total`（超过 10% 需要扩容）
- **P99 延迟** = 请求排队时间 + LLM 响应时间
- **当前并发数** = 是否接近设定的上限

---

## 十、总结

| 方案 | 适用场景 | 优点 | 缺点 |
|------|---------|------|------|
| Semaphore | 并发数控制 | 简单，Java 原生 | 不限制频率 |
| Guava RateLimiter | 单机频率限流 | 平滑，API 友好 | 不支持分钟级，非分布式 |
| Bucket4j | 复杂限流策略 | 功能强大，支持多维度 | 学习成本稍高 |
| Redis 令牌桶 | 分布式集群 | 全局精确限流 | 依赖 Redis，有网络开销 |
| Redisson | 快速落地 | 开箱即用 | 依赖 Redisson |

**生产环境建议组合**：
- 单实例 → Semaphore（并发）+ Bucket4j（频率）
- 多实例 → Semaphore（本地并发）+ Redis Lua 令牌桶（分布式频率）
- 高可用 → 上述 + 背压队列 + 降级兜底

---

**下一篇预告**：限流是为了"保护下游不被我们打挂"，但另一个同等重要的命题是"相同的请求不要重复花钱"。下一篇《LLM 调用的缓存策略：相同问题不重复付费的工程方案》，我将带你实现一个能**省下 30%~50% API 费用**的多级缓存架构。敬请期待！

---

> 如果觉得这篇文章有帮助，欢迎点赞、收藏、关注，感谢支持！

> 作者：深耕 Java 企业级开发多年，专注 AI 工程化落地。有问题欢迎在评论区交流。
