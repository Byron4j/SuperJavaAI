# Spring AI ChatClient 深度配置：连接池、超时、重试与熔断，把你的AI服务调到生产级

> 能调通 API 只是第一步，上了生产才发现：高并发下连接超时、API 限流后报错、重试风暴导致雪崩……

## 一、开篇：Hello World 到生产级有多远

上篇文章我们 5 分钟接入了 GPT-4，3 个注解就搞定了。但如果你把那段代码直接丢到生产环境，大概率会碰到这些问题：

| 问题场景 | 根因 | 后果 |
|---------|------|------|
| QPS 上到 100 时大面积超时 | 默认连接池太小，连接排队等待 | 用户体验差，线程阻塞 |
| 晚上 8 点高峰期 10% 请求失败 | OpenAI API 限流（429），重试风暴 | 雪崩效应，CPU 打满 |
| 某个接口慢导致整个服务卡死 | 没有超时控制，线程无限等待 | 线程池耗尽，服务不可用 |
| API Key 过期后服务全部报错 | 没有熔断机制，请求直达故障端 | 大面积 5xx 错误 |

这篇文章将逐一解决这些问题，让你的 AI 服务具备生产级的稳定性。

---

## 二、HttpClient 连接池配置

Spring AI 底层使用 `RestClient`（Spring Framework 6.x 的 HTTP 客户端）发送请求。默认配置的连接池很小，高并发下会产生大量 `ConnectionPoolTimeoutException`。

### 2.1 默认行为分析

Spring AI 1.0.0-M5 默认使用 **JDK HttpClient**，底层基于 `java.net.http.HttpClient`，默认配置如下：

- 最大连接数：无限（实际上受系统限制）
- 每个地址最大连接数：5
- 连接超时：未设置（使用系统默认）
- 空闲超时：未设置

这个配置根本无法支撑生产环境。

### 2.2 自定义连接池配置

我们需要显式定义一个 `RestClient.Builder` Bean，覆盖默认行为：

```java
package com.example.springaidemo.config;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.openai.OpenAiChatModel;
import org.springframework.ai.openai.api.OpenAiApi;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.web.client.RestClientCustomizer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.client.JdkClientHttpRequestFactory;
import org.springframework.web.client.RestClient;

import javax.net.ssl.SSLContext;
import java.net.http.HttpClient;
import java.security.KeyManagementException;
import java.security.NoSuchAlgorithmException;
import java.time.Duration;

@Configuration
public class ChatClientConfig {

    @Value("${spring.ai.openai.api-key}")
    private String apiKey;

    @Value("${spring.ai.openai.base-url:https://api.openai.com}")
    private String baseUrl;

    @Bean
    public OpenAiApi openAiApi() {
        return new OpenAiApi(baseUrl, apiKey);
    }

    @Bean
    public OpenAiChatModel openAiChatModel(OpenAiApi openAiApi) {
        return new OpenAiChatModel(openAiApi);
    }

    @Bean
    public ChatClient.Builder chatClientBuilder(OpenAiChatModel chatModel) {
        return ChatClient.builder(chatModel);
    }

    /**
     * 自定义 JDK HttpClient，配置连接池参数
     */
    @Bean
    public RestClient.Builder restClientBuilder() {
        HttpClient httpClient = HttpClient.newBuilder()
                .version(HttpClient.Version.HTTP_2)            // 启用 HTTP/2
                .connectTimeout(Duration.ofSeconds(10))        // 连接建立超时
                .executor(java.util.concurrent.Executors.newVirtualThreadPerTaskExecutor())
                .build();

        return RestClient.builder()
                .requestFactory(new JdkClientHttpRequestFactory(httpClient));
    }

    /**
     * RestClient 全局定制（推荐方式）
     */
    @Bean
    public RestClientCustomizer restClientCustomizer() {
        return restClientBuilder -> {
            HttpClient httpClient = HttpClient.newBuilder()
                    // ===== 关键：连接池配置 =====
                    .connectTimeout(Duration.ofSeconds(10))
                    .build();

            JdkClientHttpRequestFactory factory =
                    new JdkClientHttpRequestFactory(httpClient);
            // 连接超时
            factory.setReadTimeout(Duration.ofSeconds(60));
            factory.setConnectTimeout(Duration.ofSeconds(10));

            restClientBuilder.requestFactory(factory);
        };
    }
}
```

### 2.3 使用 Apache HttpClient 5（更丰富的连接池配置）

如果你的项目重度依赖连接池管理，建议切换到 Apache HttpClient 5：

```xml
<dependency>
    <groupId>org.apache.httpcomponents.client5</groupId>
    <artifactId>httpclient5</artifactId>
</dependency>
```

```java
package com.example.springaidemo.config;

import org.apache.hc.client5.http.classic.HttpClient;
import org.apache.hc.client5.http.config.ConnectionConfig;
import org.apache.hc.client5.http.impl.classic.HttpClientBuilder;
import org.apache.hc.client5.http.impl.io.PoolingHttpClientConnectionManager;
import org.apache.hc.client5.http.io.HttpClientConnectionManager;
import org.apache.hc.core5.util.Timeout;
import org.springframework.boot.web.client.RestClientCustomizer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.client.HttpComponentsClientHttpRequestFactory;

import java.util.concurrent.TimeUnit;

@Configuration
public class HttpClientConfig {

    @Bean
    public RestClientCustomizer restClientCustomizer() {
        return restClientBuilder -> {
            // 创建连接管理器（核心：连接池）
            PoolingHttpClientConnectionManager connectionManager =
                    new PoolingHttpClientConnectionManager();

            // ===== 连接池核心参数 =====
            connectionManager.setMaxTotal(200);              // 总连接数上限
            connectionManager.setDefaultMaxPerRoute(50);     // 每个路由（域名）最大连接数
            connectionManager.setValidateAfterInactivity(
                    TimeValue.ofSeconds(10));                // 空闲连接验证间隔

            // ===== 连接级别超时 =====
            ConnectionConfig connectionConfig = ConnectionConfig.custom()
                    .setConnectTimeout(Timeout.ofSeconds(10))    // 建立 TCP 连接超时
                    .setSocketTimeout(Timeout.ofSeconds(60))     // 等待数据超时（读超时）
                    .setTimeToLive(TimeValue.ofMinutes(5))       // 连接最大存活时间
                    .setValidateAfterInactivity(Timeout.ofSeconds(10))
                    .build();

            connectionManager.setDefaultConnectionConfig(connectionConfig);

            // 构建 HttpClient
            HttpClient httpClient = HttpClientBuilder.create()
                    .setConnectionManager(connectionManager)
                    // ===== 连接保活策略 =====
                    .setKeepAliveStrategy((response, context) -> {
                        // 从响应头读取 Keep-Alive 时长，默认 30 秒
                        return TimeValue.ofSeconds(30);
                    })
                    // ===== 空闲连接驱逐（防止连接泄漏）=====
                    .evictIdleConnections(TimeValue.ofSeconds(30))
                    .build();

            // 注入到 RestClient
            HttpComponentsClientHttpRequestFactory factory =
                    new HttpComponentsClientHttpRequestFactory(httpClient);
            restClientBuilder.requestFactory(factory);
        };
    }
}
```

### 2.4 连接池参数速查表

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| `maxTotal` | 200 | 总连接数上限。按 QPS × 平均响应时间估算 |
| `maxPerRoute` | 50 | 单域名最大连接数。AI API 通常只有一个域名 |
| `connectTimeout` | 5~10s | TCP 握手超时，过长会让请求排队 |
| `socketTimeout` | 30~120s | 读超时。GPT-4 生成慢，建议设 120s |
| `timeToLive` | 5 min | 连接存活时间，定期重建避免对端主动断开 |
| `evictIdleConnections` | 30s | 驱逐空闲连接，防止泄露 |

> **算力公式**：如果你的服务 QPS = 100，平均响应时间 = 3s，则理论连接数 = 100 × 3 = 300。给 1.5 倍冗余即 450。但 OpenAI 本身有并发限制（付费账户每分钟 500 次请求），所以设为 50~100 通常够用。

---

## 三、超时策略：连接超时 vs 读超时 vs 写超时

超时配置往往是"谁调谁知道"，设置不当要么用户体验差，要么资源泄漏。

### 3.1 三种超时图示

```
客户端                              OpenAI API
  |                                    |
  |-------- TCP SYN (建立连接) -------->|
  |                                    |
  |<------- TCP SYN-ACK --------------|
  |                                    |
  |-------- 发送请求 Body ------------->|
  |                                    |
  |         [等待 AI 生成...]           |
  |                                    |
  |<------- 返回响应数据 --------------|
  |                                    |

连接超时：TCP 握手阶段，默认 10s
写超时：  发送请求体阶段，默认不设置
读超时：  等待响应阶段，GPT-4 生成慢时可能需要 60~120s
```

### 3.2 Spring AI 层面的超时配置

除了 HTTP 层面的超时，Spring AI 也提供了应用层超时：

```java
package com.example.springaidemo.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.ai.openai.OpenAiChatOptions;

import java.time.Duration;

@Configuration
public class ChatTimeoutConfig {

    /**
     * 为 ChatClient 设置默认的超时和模型参数
     */
    @Bean
    public OpenAiChatOptions defaultChatOptions() {
        return OpenAiChatOptions.builder()
                .withModel("gpt-4o")
                .withTemperature(0.7)
                .withMaxTokens(2000)
                // ⚠️ 注意：Spring AI 1.0.0-M5 的 Options 暂不支持直接设置超时
                // 超时应该在 HTTP Client 层面控制（见上文）
                .build();
    }
}
```

### 3.3 实战：按场景配置不同超时

```java
package com.example.springaidemo.controller;

import lombok.RequiredArgsConstructor;
import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.openai.OpenAiChatOptions;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequiredArgsConstructor
public class SmartChatController {

    private final ChatClient.Builder chatClientBuilder;

    /**
     * 短回答场景：10s 超时
     * 适用：简单问答、分类、摘要
     */
    @GetMapping("/chat/quick")
    public String quickChat(@RequestParam String prompt) {
        return chatClientBuilder.build()
                .prompt()
                .options(OpenAiChatOptions.builder()
                        .withMaxTokens(200)      // 限制输出长度
                        .withTemperature(0.3)    // 低温度，更快
                        .build())
                .user(prompt)
                .call()
                .content();
    }

    /**
     * 长回答场景：60s 超时
     * 适用：长文生成、代码生成
     */
    @GetMapping("/chat/long")
    public String longChat(@RequestParam String prompt) {
        return chatClientBuilder.build()
                .prompt()
                .options(OpenAiChatOptions.builder()
                        .withMaxTokens(4000)     // 允许长输出
                        .withTemperature(0.7)
                        .build())
                .user(prompt)
                .call()
                .content();
    }
}
```

### 3.4 超时推荐值

| 场景 | 连接超时 | 读超时 | 说明 |
|------|---------|--------|------|
| 简单问答 | 5s | 15s | 短回答，不超过 200 token |
| 常规对话 | 10s | 30s | 聊天场景 |
| 代码生成 | 10s | 60s | 生成代码较长 |
| 长文创作 | 10s | 120s | 4000+ token 输出 |
| Embedding | 5s | 10s | 向量化很快 |

---

## 四、重试策略：指数退避 + 最大重试

AI API 调用失败的原因多种多样：网络抖动、API 限流（429）、服务暂时不可用（503）。盲目的重试会导致"重试风暴"。

### 4.1 Spring Retry 集成

首先添加依赖：

```xml
<dependency>
    <groupId>org.springframework.retry</groupId>
    <artifactId>spring-retry</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-aspects</artifactId>
</dependency>
```

在启动类上启用重试：

```java
package com.example.springaidemo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.retry.annotation.EnableRetry;

@SpringBootApplication
@EnableRetry   // ← 只需要这一个注解
public class SpringAiDemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(SpringAiDemoApplication.class, args);
    }
}
```

### 4.2 带重试的 AI 服务层

```java
package com.example.springaidemo.service;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.ai.chat.client.ChatClient;
import org.springframework.retry.annotation.Backoff;
import org.springframework.retry.annotation.Recover;
import org.springframework.retry.annotation.Retryable;
import org.springframework.stereotype.Service;

@Slf4j
@Service
@RequiredArgsConstructor
public class ResilientChatService {

    private final ChatClient.Builder chatClientBuilder;

    /**
     * 带重试的 AI 调用
     *
     * @Retryable 参数说明：
     * - retryFor: 遇到这些异常时重试
     * - maxAttempts: 最多尝试 4 次（1 次原始 + 3 次重试）
     * - backoff: 指数退避，delay=2s, multiplier=2
     *   即: 2s → 4s → 8s 间隔
     */
    @Retryable(
            retryFor = {org.springframework.web.client.HttpServerErrorException.class,
                        org.springframework.web.client.ResourceAccessException.class},
            maxAttempts = 4,
            backoff = @Backoff(
                    delay = 2000,        // 初始等待 2 秒
                    multiplier = 2.0,     // 每次乘以 2
                    maxDelay = 30000     // 最大等待 30 秒
            )
    )
    public String callWithRetry(String prompt) {
        log.info("正在调用 AI API: {}", prompt.substring(0, Math.min(50, prompt.length())));

        return chatClientBuilder.build()
                .prompt()
                .user(prompt)
                .call()
                .content();
    }

    /**
     * 所有重试都失败后的兜底策略
     * 方法名必须和 @Retryable 标注的方法一致
     */
    @Recover
    public String recover(RuntimeException e, String prompt) {
        log.error("AI API 调用最终失败，已重试 4 次", e);

        // 返回降级文案
        return "抱歉，AI 服务暂时不可用，请稍后重试。"
                + "（原始问题：" + prompt.substring(0, Math.min(50, prompt.length())) + "）";
    }
}
```

### 4.3 更精细的重试：只重试可恢复的错误

并非所有错误都适合重试——4xx 客户端错误（如 401 认证失败、400 参数错误）重试没有意义：

```java
package com.example.springaidemo.service;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.ai.chat.client.ChatClient;
import org.springframework.retry.annotation.Backoff;
import org.springframework.retry.annotation.Retryable;
import org.springframework.stereotype.Service;
import org.springframework.web.client.HttpClientErrorException;
import org.springframework.web.client.HttpServerErrorException;
import org.springframework.web.client.ResourceAccessException;

import java.util.Set;

@Slf4j
@Service
@RequiredArgsConstructor
public class SmartRetryService {

    private final ChatClient.Builder chatClientBuilder;

    /** 不应重试的 HTTP 状态码 */
    private static final Set<Integer> NO_RETRY_STATUS = Set.of(
            400,  // Bad Request（参数错误，重试没用）
            401,  // Unauthorized（API Key 不对）
            402,  // Payment Required（欠费了）
            403,  // Forbidden（没权限）
            404   // Not Found
    );

    /**
     * 智能重试：只重试服务器错误和限流
     */
    @Retryable(
            retryFor = {HttpServerErrorException.class, ResourceAccessException.class},
            notRecoverable = {HttpClientErrorException.TooManyRequests.class},
            maxAttempts = 3,
            backoff = @Backoff(delay = 5000, multiplier = 2.0)
    )
    public String smartCall(String prompt) {
        return chatClientBuilder.build()
                .prompt()
                .user(prompt)
                .call()
                .content();
    }

    /**
     * 处理 429 限流：使用更长的退避时间
     * 从响应头 Retry-After 获取建议等待时间（需要自定义拦截器）
     */
    @Retryable(
            retryFor = {HttpClientErrorException.TooManyRequests.class},
            maxAttempts = 3,
            backoff = @Backoff(
                    delay = 10_000,        // 初始等待 10 秒
                    multiplier = 3.0,       // 每次乘 3
                    maxDelay = 60_000       // 最多等 60 秒
            )
    )
    public String callWithRateLimitHandling(String prompt) {
        return chatClientBuilder.build()
                .prompt()
                .user(prompt)
                .call()
                .content();
    }
}
```

### 4.4 重试参数快速参考

| 场景 | maxAttempts | delay | multiplier | maxDelay |
|------|-------------|-------|------------|----------|
| 网络抖动 | 3 | 1s | 2.0 | 10s |
| 503 服务不可用 | 3 | 2s | 2.0 | 15s |
| 429 限流 | 3 | 10s | 3.0 | 60s |
| 生产级（综合） | 4 | 2s | 2.0 | 30s |

---

## 五、Resilience4j 熔断集成

重试解决的是"偶发失败"，熔断解决的是"持续故障"。当 AI API 长时间不可用时，快速失败（Fail Fast）比一直等待更好。

### 5.1 添加依赖

```xml
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
    <version>2.2.0</version>
</dependency>
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-reactor</artifactId>
    <version>2.2.0</version>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

### 5.2 application.yml 熔断配置

```yaml
resilience4j:
  circuitbreaker:
    configs:
      default:
        # 滑动窗口类型（COUNT_BASED: 按请求数，TIME_BASED: 按时间）
        sliding-window-type: COUNT_BASED
        # 窗口大小：最近 10 次请求
        sliding-window-size: 10
        # 失败率阈值：50% 失败就熔断
        failure-rate-threshold: 50
        # 熔断后等待多久进入半开状态
        wait-duration-in-open-state: 30s
        # 半开状态下允许通过的请求数
        permitted-number-of-calls-in-half-open-state: 3
        # 慢调用阈值（秒）
        slow-call-duration-threshold: 60s
        # 慢调用比例阈值
        slow-call-rate-threshold: 100
        # 自动从半开切换到关闭
        automatic-transition-from-open-to-half-open-enabled: false

    instances:
      openaiChat:
        base-config: default
        sliding-window-size: 10
        failure-rate-threshold: 50
        wait-duration-in-open-state: 30s
        # 记录以下异常为失败
        record-exceptions:
          - org.springframework.web.client.HttpServerErrorException
          - org.springframework.web.client.ResourceAccessException
          - java.util.concurrent.TimeoutException
        # 忽略以下异常（不计入失败统计）
        ignore-exceptions:
          - org.springframework.web.client.HttpClientErrorException.BadRequest
          - org.springframework.web.client.HttpClientErrorException.Unauthorized

  timelimiter:
    configs:
      default:
        timeout-duration: 60s
    instances:
      openaiChat:
        timeout-duration: 60s
```

### 5.3 Java 注解方式使用熔断

```java
package com.example.springaidemo.service;

import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.ai.chat.client.ChatClient;
import org.springframework.stereotype.Service;

@Slf4j
@Service
@RequiredArgsConstructor
public class CircuitBreakerChatService {

    private final ChatClient.Builder chatClientBuilder;

    /**
     * @CircuitBreaker name 对应 yml 中的 instances 名称
     * fallbackMethod 指定降级方法
     */
    @CircuitBreaker(
            name = "openaiChat",
            fallbackMethod = "fallbackChat"
    )
    public String chatWithCircuitBreaker(String prompt) {
        log.info("正在调用 AI API（当前断路器状态：closed/open/half-open）");
        return chatClientBuilder.build()
                .prompt()
                .user(prompt)
                .call()
                .content();
    }

    /**
     * 降级方法
     * 参数必须和原方法一致，最后一个参数可以是 Throwable
     */
    public String fallbackChat(String prompt, Throwable t) {
        log.error("AI 服务已熔断，执行降级逻辑", t);
        return "AI 助手暂时不可用，请稍后再试。当前已启用本地缓存回答。";
    }
}
```

### 5.4 程序化配置（更灵活）

```java
package com.example.springaidemo.config;

import io.github.resilience4j.circuitbreaker.CircuitBreaker;
import io.github.resilience4j.circuitbreaker.CircuitBreakerConfig;
import io.github.resilience4j.circuitbreaker.CircuitBreakerRegistry;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.time.Duration;

@Configuration
public class Resilience4jConfig {

    @Bean
    public CircuitBreaker openaiCircuitBreaker(CircuitBreakerRegistry registry) {
        CircuitBreakerConfig config = CircuitBreakerConfig.custom()
                // 故障率阈值 50%
                .failureRateThreshold(50)
                // 慢调用阈值 60s
                .slowCallDurationThreshold(Duration.ofSeconds(60))
                // 慢调用比例阈值 100%
                .slowCallRateThreshold(100)
                // 熔断持续时间 30s
                .waitDurationInOpenState(Duration.ofSeconds(30))
                // 半开状态允许 3 个请求
                .permittedNumberOfCallsInHalfOpenState(3)
                // 滑动窗口 10 次请求
                .slidingWindowSize(10)
                .slidingWindowType(CircuitBreakerConfig.SlidingWindowType.COUNT_BASED)
                // 记录这些异常
                .recordExceptions(
                        org.springframework.web.client.HttpServerErrorException.class,
                        org.springframework.web.client.ResourceAccessException.class,
                        java.util.concurrent.TimeoutException.class
                )
                // 忽略这些异常
                .ignoreExceptions(
                        org.springframework.web.client.HttpClientErrorException.BadRequest.class,
                        org.springframework.web.client.HttpClientErrorException.Unauthorized.class
                )
                .build();

        return registry.circuitBreaker("openaiChat", config);
    }
}
```

---

## 六、监控指标接入（Micrometer）

生产环境必须知道：AI API 调了多久、失败率多少、QPS 多少。

### 6.1 添加依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus,circuitbreakers,retry
  metrics:
    export:
      prometheus:
        enabled: true
  endpoint:
    health:
      show-details: always
```

### 6.2 自定义 AI 调用指标

```java
package com.example.springaidemo.service;

import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import lombok.RequiredArgsConstructor;
import org.springframework.ai.chat.client.ChatClient;
import org.springframework.stereotype.Service;

import java.util.concurrent.TimeUnit;

@Service
@RequiredArgsConstructor
public class MonitoredChatService {

    private final ChatClient.Builder chatClientBuilder;
    private final MeterRegistry meterRegistry;

    public String chatWithMetrics(String prompt) {
        // 计数器：总请求数
        meterRegistry.counter("ai.chat.requests.total",
                        "model", "gpt-4o")
                .increment();

        // 计时器：记录调用耗时
        Timer.Sample sample = Timer.start(meterRegistry);
        try {
            String result = chatClientBuilder.build()
                    .prompt()
                    .user(prompt)
                    .call()
                    .content();

            // 成功计数
            meterRegistry.counter("ai.chat.requests.success",
                            "model", "gpt-4o")
                    .increment();

            return result;

        } catch (Exception e) {
            // 失败计数（按错误类型分类）
            meterRegistry.counter("ai.chat.requests.failure",
                            "model", "gpt-4o",
                            "error", e.getClass().getSimpleName())
                    .increment();
            throw e;

        } finally {
            sample.stop(Timer.builder("ai.chat.latency")
                    .description("AI API 调用延迟")
                    .tag("model", "gpt-4o")
                    .publishPercentileHistogram(true)
                    .publishPercentiles(0.5, 0.95, 0.99)
                    .register(meterRegistry));
        }
    }
}
```

### 6.3 Prometheus 指标示例

访问 `http://localhost:8080/actuator/prometheus`，你会看到：

```
# HELP ai_chat_requests_total AI 聊天请求总数
# TYPE ai_chat_requests_total counter
ai_chat_requests_total{model="gpt-4o"} 1507

# HELP ai_chat_requests_success AI 聊天成功数
# TYPE ai_chat_requests_success counter
ai_chat_requests_success{model="gpt-4o"} 1482

# HELP ai_chat_requests_failure AI 聊天失败数
# TYPE ai_chat_requests_failure counter
ai_chat_requests_failure{model="gpt-4o",error="HttpServerErrorException"} 25

# HELP ai_chat_latency_seconds AI API 调用延迟
# TYPE ai_chat_latency_seconds histogram
ai_chat_latency_seconds{model="gpt-4o",quantile="0.5"} 2.3
ai_chat_latency_seconds{model="gpt-4o",quantile="0.95"} 8.7
ai_chat_latency_seconds{model="gpt-4o",quantile="0.99"} 15.2

# Resilience4j 自带指标
resilience4j_circuitbreaker_state{name="openaiChat",state="closed"} 1.0
resilience4j_circuitbreaker_failure_rate{name="openaiChat"} 0.03
```

配合 Grafana 接入，告别盲飞。

---

## 七、生产级配置组合拳（完整代码）

把连接池 + 超时 + 重试 + 熔断 + 监控组合在一起，这是你的生产级 ChatService：

```java
package com.example.springaidemo.service;

import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.ai.chat.client.ChatClient;
import org.springframework.retry.annotation.Backoff;
import org.springframework.retry.annotation.Retryable;
import org.springframework.stereotype.Service;

@Slf4j
@Service
@RequiredArgsConstructor
public class ProductionChatService {

    private final ChatClient.Builder chatClientBuilder;
    private final MeterRegistry meterRegistry;

    /**
     * 生产级 AI 调用
     * 层级关系：重试（内层） → 熔断（外层） → 监控（贯穿）
     */
    @CircuitBreaker(name = "openaiChat", fallbackMethod = "fallbackResponse")
    @Retryable(
            retryFor = {org.springframework.web.client.HttpServerErrorException.class,
                        org.springframework.web.client.ResourceAccessException.class},
            maxAttempts = 3,
            backoff = @Backoff(delay = 2000, multiplier = 2.0, maxDelay = 15000)
    )
    public String chat(String prompt) {
        Timer.Sample sample = Timer.start(meterRegistry);

        try {
            meterRegistry.counter("ai.chat.requests").increment();

            String result = chatClientBuilder.build()
                    .prompt()
                    .user(prompt)
                    .call()
                    .content();

            meterRegistry.counter("ai.chat.success").increment();
            log.info("AI 调用成功: {}", prompt.substring(0, Math.min(50, prompt.length())));
            return result;

        } finally {
            sample.stop(meterRegistry.timer("ai.chat.latency"));
        }
    }

    /** 熔断降级 */
    private String fallbackResponse(String prompt, Throwable t) {
        log.error("AI 服务已熔断，使用降级回复", t);
        meterRegistry.counter("ai.chat.circuitbreaker.fallback").increment();
        return "AI 助手正在升级维护中，请稍后再试。";
    }
}
```

**注解的执行顺序**示意图：

```
请求进来
  ↓
@CircuitBreaker（熔断检查） → 如果断路器打开，直接 fallback
  ↓ 断路器关闭/半开，继续
@Retryable（重试）→ 失败则按退避策略重试
  ↓ 成功
实际调用 ChatClient
  ↓
返回结果 + 记录指标
```

---

## 八、总结与下篇预告

本文从连接池、超时、重试、熔断、监控 5 个维度，把你的 AI 服务从"玩具"调到"生产级"：

| 维度 | 核心配置/注解 | 效果 |
|------|-------------|------|
| 连接池 | `PoolingHttpClientConnectionManager` | 高并发不排队 |
| 超时 | `connectTimeout` / `socketTimeout` | 防止线程泄漏 |
| 重试 | `@Retryable` + 指数退避 | 自动恢复临时故障 |
| 熔断 | `@CircuitBreaker` + Resilience4j | 防止雪崩 |
| 监控 | `MeterRegistry` + Prometheus | 可观测性 |

但到目前为止，我们都只是在"调用"AI——把用户输入发给 GPT-4，再把回复返回。真正的 AI 应用远不止于此：

**下一篇**：《Spring AI Embedding：文本向量化与相似度计算的 Java 实现》，我们将解锁 AI 应用的另一个核心能力——把文字变成数学向量，实现语义搜索、智能去重、FAQ 自动匹配。

---

> **系列目录**：
> - 081：[Spring AI 入门]：5 分钟接入 GPT-4
> - 082：ChatClient 深度配置 ← 本文
> - 083：[Embedding 文本向量化]（语义搜索/相似度计算）
> - 084：[VectorStore 向量数据库集成]（Chroma/Milvus/Pgvector）
