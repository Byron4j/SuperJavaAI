---
title: Spring AI + Spring Cloud：构建 AI 微服务集群的最佳实践，AI 能力从单体到分布式
tags: [Spring AI, Spring Cloud, 微服务, AI, Java, 分布式, Nacos, Gateway]
---

# Spring AI + Spring Cloud：构建 AI 微服务集群的最佳实践

> 当你的 AI 服务从一个 Demo 成长为日均百万调用的生产系统，单体架构显然跟不上了。本文将手把手教你用 Spring AI + Spring Cloud 全家桶，把 AI 能力从单体拆分为高可用的微服务集群。

---

## 一、开篇：AI 服务的"成长的烦恼"

先看看这个真实的故事：

王工是国内某中型 SaaS 公司的架构师。公司年初上线了一个 AI 助手功能，基于 Spring AI 对接了 GPT-4o，给客户提供智能问答、文档摘要、图片识别等服务。刚开始一切都很美好——一个 Spring Boot 单体应用，几百个 DAU（日活用户），响应时间 2-3 秒，用户反馈不错。

三个月后，随着客户量从几十家增长到上千家，问题接连暴雷：

1. **单点瓶颈**：所有 AI 调用集中在一个实例，高峰期 CPU 飙到 95%，响应时间从 2 秒拉长到 15 秒
2. **模型成本爆炸**：GPT-4o 实在太贵了，简单问答场景完全可以用便宜模型替代，但在单体里没法做策略路由
3. **部署即宕机**：升级 AI Prompt 模板需要重启整个服务，导致所有功能不可用
4. **缺乏容错**：OpenAI API 偶尔抽风，整个 AI 功能直接挂掉，没有任何降级策略

这就是**典型 AI 服务从单体到分布式的演进拐点**。

好在 Spring 生态给了我们天然的解药——**Spring Cloud 全家桶**。本文将用 Spring AI + Spring Cloud 搭建一个生产级的 AI 微服务集群，涵盖服务注册、负载均衡、网关路由、配置中心、熔断降级全链路。

---

## 二、架构全景图

在动手写代码之前，先看清我们要构建的系统长什么样：

```text
                     ┌─────────────┐
                     │   客户端     │
                     └──────┬──────┘
                            │
                     ┌──────▼──────┐
                     │  Gateway    │  网关：限流、鉴权、路由
                     │  (Spring    │
                     │   Cloud     │
                     │   Gateway)  │
                     └──────┬──────┘
                            │
              ┌─────────────┼─────────────┐
              │             │             │
     ┌────────▼───┐  ┌──────▼──────┐  ┌──▼──────────┐
     │ AI-Chat    │  │ AI-Image    │  │ AI-Embedding │
     │ Service    │  │ Service     │  │ Service      │
     │ (对话服务)  │  │ (图片服务)   │  │ (向量化服务)  │
     └────────────┘  └─────────────┘  └──────────────┘
              │             │             │
     ┌────────▼─────────────▼─────────────▼──────────┐
     │              Nacos 注册中心 & 配置中心          │
     └───────────────────────────────────────────────┘
```

**核心服务拆分**：
- **ai-chat-service**：智能对话、Prompt 管理、Token 统计
- **ai-image-service**：图片理解、图片生成、OCR 识别
- **ai-embedding-service**：文本向量化、语义搜索、RAG 检索

为什么这么拆？遵循**单一职责和高内聚**原则：
- 对话服务压力大就扩对话服务，不用带着图片服务一起扩
- 图片服务需要 GPU 资源，单独部署在 GPU 节点
- Embedding 服务是 RAG 的底层基础设施，独立部署便于复用

---

## 三、环境搭建：一张表看清技术选型

| 组件 | 选型 | 版本 | 作用 |
|------|------|------|------|
| 微服务框架 | Spring Cloud | 2023.0.x | 微服务全家桶 |
| 注册中心 | Nacos | 2.3.x | 服务注册与发现 |
| 配置中心 | Nacos Config | 2.3.x | 动态配置管理 |
| 网关 | Spring Cloud Gateway | 4.1.x | 统一入口、限流路由 |
| 负载均衡 | Spring Cloud LoadBalancer | 4.1.x | 客户端负载均衡 |
| 熔断降级 | Resilience4j | 2.2.x | 容错与降级 |
| AI 框架 | Spring AI | 1.0.0-M6 | AI 能力接入 |
| 远程调用 | OpenFeign | 4.1.x | 服务间调用 |

### 3.1 父工程 pom.xml

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.3.0</version>
</parent>

<properties>
    <spring-cloud.version>2023.0.2</spring-cloud.version>
    <spring-ai.version>1.0.0-M6</spring-ai.version>
    <spring-cloud-alibaba.version>2023.0.1.0</spring-cloud-alibaba.version>
</properties>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring-cloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-bom</artifactId>
            <version>${spring-ai.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

---

## 四、服务注册与发现：Nacos 接入

Nacos 是阿里开源的注册中心+配置中心一体化方案，比 Eureka 功能更强，中文社区活跃，是 Spring Cloud Alibaba 的核心组件。

### 4.1 Nacos 服务端快速启动

```bash
# Docker 一键启动
docker run -d --name nacos -e MODE=standalone \
  -p 8848:8848 -p 9848:9848 \
  nacos/nacos-server:v2.3.1
```

启动后访问 `http://localhost:8848/nacos`，默认用户名密码 `nacos/nacos`。

### 4.2 服务提供者：ai-chat-service

每个 AI 微服务都是一个标准的 Spring Boot 项目，通过 `@EnableDiscoveryClient` 注册到 Nacos。

```xml
<!-- ai-chat-service 的 pom.xml -->
<dependencies>
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
```

```yaml
# ai-chat-service 的 application.yml
server:
  port: 8081

spring:
  application:
    name: ai-chat-service
  cloud:
    nacos:
      discovery:
        server-addr: localhost:8848
        namespace: dev
        group: AI_SERVICE_GROUP
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      chat:
        options:
          model: gpt-4o
```

```java
// 启动类
@SpringBootApplication
@EnableDiscoveryClient
public class AiChatServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(AiChatServiceApplication.class, args);
    }
}
```

```java
// 对话服务接口
@RestController
@RequestMapping("/api/chat")
@Slf4j
public class ChatController {

    private final ChatClient chatClient;

    public ChatController(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    @PostMapping("/completions")
    public ResponseEntity<Map<String, String>> chat(@RequestBody @Valid ChatRequest request) {
        String response = chatClient.prompt()
                .user(request.getPrompt())
                .call()
                .content();

        log.info("Chat 请求处理完成，Token 消耗约: {} 字符", response.length());
        return ResponseEntity.ok(Map.of("response", response, "service", "ai-chat-service"));
    }

    @Data
    public static class ChatRequest {
        @NotBlank private String prompt;
    }
}
```

### 4.3 服务消费者：OpenFeign 声明式调用

网关或业务服务通过 OpenFeign 调用 AI 微服务：

```java
@FeignClient(name = "ai-chat-service", path = "/api/chat")
public interface AiChatFeignClient {

    @PostMapping("/completions")
    Map<String, String> chat(@RequestBody Map<String, String> request);
}
```

```java
// 在其他服务中使用
@RestController
@RequestMapping("/api/business")
public class BusinessController {

    @Autowired
    private AiChatFeignClient aiChatClient;

    @PostMapping("/smart-reply")
    public String smartReply(@RequestBody Map<String, String> request) {
        // 声明式调用，无需关心服务地址
        return aiChatClient.chat(request).get("response");
    }
}
```

> 关键点：`@FeignClient(name = "ai-chat-service")` 中的 name 必须与目标服务在 Nacos 中注册的服务名一致。OpenFeign 会自动从 Nacos 拉取实例列表并通过 LoadBalancer 做负载均衡。

---

## 五、负载均衡：智能分配 AI 请求

### 5.1 Spring Cloud LoadBalancer 默认策略

引入 Nacos Discovery 后，Spring Cloud LoadBalancer 自动生效，默认采用**轮询（Round Robin）**策略。对于多实例部署的 AI 服务，请求会被均匀分发：

```yaml
# 启动多个实例
# 实例1：
java -jar ai-chat-service.jar --server.port=8081
# 实例2：
java -jar ai-chat-service.jar --server.port=8082
# 实例3：
java -jar ai-chat-service.jar --server.port=8083
```

### 5.2 自定义负载均衡策略

AI 服务比较特殊——不同实例可能配置了不同的模型（昂贵模型 vs 便宜模型），或者不同实例的 GPU 资源不同。这时需要自定义路由策略：

```java
@Configuration
public class AiLoadBalancerConfig {

    @Bean
    public ReactorLoadBalancer<ServiceInstance> aiServiceLoadBalancer(
            Environment environment,
            LoadBalancerClientFactory loadBalancerClientFactory) {

        String serviceName = environment.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
        return new RoundRobinLoadBalancer(
                loadBalancerClientFactory.getLazyProvider(serviceName, ServiceInstanceListSupplier.class),
                serviceName
        );
    }
}
```

更高级的做法是**基于 Token 消耗的加权负载均衡**——给配置了便宜模型的实例更高权重，让简单请求优先走便宜模型：

```java
// 自定义 Nacos 权重
spring:
  cloud:
    nacos:
      discovery:
        weight: 10  # 便宜模型实例权重 10，昂贵模型实例权重 2
```

---

## 六、统一网关：Spring Cloud Gateway

网关是整个微服务集群的"门面"，负责**统一入口、请求路由、限流、鉴权、日志**。

### 6.1 网关配置

```yaml
# gateway-service 的 application.yml
server:
  port: 8080

spring:
  application:
    name: ai-gateway
  cloud:
    nacos:
      discovery:
        server-addr: localhost:8848
    gateway:
      discovery:
        locator:
          enabled: true          # 自动根据服务名创建路由
          lower-case-service-id: true
      routes:
        # 对话服务路由
        - id: ai-chat-route
          uri: lb://ai-chat-service
          predicates:
            - Path=/api/chat/**
          filters:
            - StripPrefix=0
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10
                redis-rate-limiter.burstCapacity: 20
        # 图片服务路由
        - id: ai-image-route
          uri: lb://ai-image-service
          predicates:
            - Path=/api/multimodal/**
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 5
                redis-rate-limiter.burstCapacity: 10
        # Embedding 服务路由
        - id: ai-embedding-route
          uri: lb://ai-embedding-service
          predicates:
            - Path=/api/embedding/**
```

> `lb://ai-chat-service` 中的 `lb://` 前缀表示启用 LoadBalancer 进行客户端负载均衡。

### 6.2 全局鉴权过滤器

```java
@Component
@Slf4j
public class ApiKeyAuthFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String apiKey = exchange.getRequest().getHeaders().getFirst("X-API-Key");
        if (apiKey == null || !apiKey.equals("your-secret-key")) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }
        log.info("网关请求: {} {}", exchange.getRequest().getMethod(), exchange.getRequest().getPath());
        return chain.filter(exchange);
    }

    @Override
    public int getOrder() {
        return -100; // 最高优先级
    }
}
```

---

## 七、配置中心：AI Prompt 动态热更新

AI 服务的 Prompt 模板、模型参数、API Key 经常需要调整，如果每次改动都要重启服务，生产环境完全不可接受。

### 7.1 Nacos Config 集成

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
</dependency>
```

```yaml
# bootstrap.yml（优先于 application.yml 加载）
spring:
  application:
    name: ai-chat-service
  cloud:
    nacos:
      config:
        server-addr: localhost:8848
        namespace: dev
        group: AI_SERVICE_GROUP
        file-extension: yaml
        refresh-enabled: true   # 开启动态刷新
```

### 7.2 动态配置 AI Prompt

在 Nacos 控制台创建配置 `ai-chat-service-dev.yaml`：

```yaml
ai:
  prompts:
    translation: "你是一个专业的翻译助手，请将以下内容翻译为{targetLang}：{content}"
    summary: "请用不超过100字总结以下内容，保持重点突出：{content}"
    code-review: "你是一个资深Java代码审查专家，请审查以下代码：{code}"
  models:
    simple-task: "gpt-3.5-turbo"     # 简单任务用便宜模型
    complex-task: "gpt-4o"           # 复杂任务用高级模型
```

```java
@Configuration
@RefreshScope   // 支持配置热更新
@ConfigurationProperties(prefix = "ai")
@Data
public class AiPromptConfig {
    private Map<String, String> prompts;
    private Map<String, String> models;
}
```

```java
@Service
public class IntelligentChatService {

    private final ChatClient chatClient;
    private final AiPromptConfig config;

    public IntelligentChatService(ChatClient.Builder builder, AiPromptConfig config) {
        this.chatClient = builder.build();
        this.config = config;
    }

    public String translate(String content, String targetLang) {
        String promptTemplate = config.getPrompts().get("translation");
        String prompt = promptTemplate
                .replace("{targetLang}", targetLang)
                .replace("{content}", content);

        return chatClient.prompt().user(prompt).call().content();
    }
}
```

> 关键：加上 `@RefreshScope` 后，你在 Nacos 控制台修改 Prompt 模板，**无需重启服务，实时生效**。运维同学再也不用来找你"改个 Prompt 要发版吗"。

---

## 八、容错与降级：Resilience4j 为 AI 服务保驾护航

调用外部 AI API 有三个天然的不可靠性：
1. **超时**：GPT-4o 的复杂推理可能超过 30 秒
2. **限流**：API 有 RPM（每分钟请求数）限制
3. **服务异常**：OpenAI 偶尔宕机（真发生过）

### 8.1 熔断器配置

```yaml
resilience4j:
  circuitbreaker:
    instances:
      aiChatService:
        sliding-window-size: 10
        failure-rate-threshold: 50    # 失败率超过 50% 触发熔断
        wait-duration-in-open-state: 30s  # 熔断 30 秒后尝试半开
        permitted-number-of-calls-in-half-open-state: 3
  timelimiter:
    instances:
      aiChatService:
        timeout-duration: 30s         # 30 秒超时
```

### 8.2 降级策略

```java
@Service
@Slf4j
public class ResilientChatService {

    private final ChatClient chatClient;

    public ResilientChatService(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    @CircuitBreaker(name = "aiChatService", fallbackMethod = "chatFallback")
    @TimeLimiter(name = "aiChatService")
    public CompletableFuture<String> chatWithAI(String prompt) {
        return CompletableFuture.supplyAsync(() ->
                chatClient.prompt().user(prompt).call().content()
        );
    }

    /**
     * 降级逻辑：返回预设的友好提示
     */
    public CompletableFuture<String> chatFallback(String prompt, Throwable t) {
        log.error("AI 服务调用失败，执行降级策略。错误: {}", t.getMessage());
        return CompletableFuture.completedFuture(
                "抱歉，AI 服务暂时繁忙，请稍后再试。您的问题已记录，我们会尽快处理。"
        );
    }
}
```

---

## 九、生产部署最佳实践

### 9.1 多模型策略路由

不同场景路由到不同模型，成本可控：

```java
@Service
public class ModelRouter {

    private final Map<String, ChatClient> clients = new HashMap<>();

    public ModelRouter(ChatClient.Builder builder) {
        // 为不同场景创建不同的 ChatClient
        clients.put("simple", builder.clone()
                .defaultOptions(OpenAiChatOptions.builder().withModel("gpt-3.5-turbo").build())
                .build());
        clients.put("complex", builder.clone()
                .defaultOptions(OpenAiChatOptions.builder().withModel("gpt-4o").build())
                .build());
    }

    public String chat(String prompt, AiTaskType taskType) {
        return switch (taskType) {
            case SIMPLE_QA, TRANSLATION -> clients.get("simple").prompt().user(prompt).call().content();
            case CODE_GENERATION, COMPLEX_REASONING -> clients.get("complex").prompt().user(prompt).call().content();
        };
    }
}

enum AiTaskType {
    SIMPLE_QA, TRANSLATION, CODE_GENERATION, COMPLEX_REASONING
}
```

### 9.2 Docker Compose 一键部署

```yaml
version: '3.8'
services:
  nacos:
    image: nacos/nacos-server:v2.3.1
    environment:
      - MODE=standalone
    ports:
      - "8848:8848"

  ai-chat-service-1:
    build: ./ai-chat-service
    ports:
      - "8081:8081"
    environment:
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - NACOS_SERVER_ADDR=nacos:8848

  ai-chat-service-2:
    build: ./ai-chat-service
    ports:
      - "8082:8082"
    environment:
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - NACOS_SERVER_ADDR=nacos:8848

  ai-image-service:
    build: ./ai-image-service
    ports:
      - "8083:8083"
    environment:
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - NACOS_SERVER_ADDR=nacos:8848

  ai-gateway:
    build: ./ai-gateway
    ports:
      - "8080:8080"
    depends_on:
      - nacos
```

---

## 十、总结

本文从单体 AI 服务面临的真实痛点出发，完整演示了如何用 Spring AI + Spring Cloud 构建一个**生产级 AI 微服务集群**：

1. **服务拆分**：按职责拆分对话服务、图片服务、Embedding 服务
2. **服务发现**：Nacos 注册中心，实现服务自动注册与发现
3. **负载均衡**：Spring Cloud LoadBalancer + 自定义权重策略
4. **统一网关**：Spring Cloud Gateway 统一入口 + 限流 + 鉴权
5. **配置中心**：Nacos Config 实现 Prompt 模板动态热更新
6. **熔断降级**：Resilience4j 为 AI API 调用保驾护航
7. **多模型路由**：根据任务复杂度智能分配模型，成本优化 60%+

核心理念：**AI 能力是业务逻辑的一部分，不应该是架构中的特殊公民。用 Spring Cloud 标准范式管理 AI 服务，让 AI 能力和传统微服务统一治理。**

---

**下一篇预告**：《LangChain4j 快速上手：Java 开发者拥抱 LLM 的第一选择》—— Spring AI 虽好，但它的抽象层还不够灵活。LangChain4j 作为 Java 生态对标 LangChain 的框架，提供了更强大的链式调用、Agent、RAG 能力。它为什么可能是 Java 开发者更好的选择？敬请期待！

---

> 作者：IT 老熊
> 标签：Spring AI, Spring Cloud, 微服务, AI, 分布式, Nacos
> 原文首发：CSDN 技术社区
