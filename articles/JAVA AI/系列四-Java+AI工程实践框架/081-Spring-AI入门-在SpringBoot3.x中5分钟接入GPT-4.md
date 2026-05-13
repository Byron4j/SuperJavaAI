# Spring AI 入门：在 Spring Boot 3.x 中 5 分钟接入 GPT-4，只需3个注解

> Java 开发者等了这么久，Spring 官方终于出手了——Spring AI 让你用写 Spring Boot 的方式接入 AI。

## 一、为什么是 Spring AI

2023年以来，AI 生态几乎被 Python 垄断。LangChain、LlamaIndex、Semantic Kernel 这些框架一个比一个火，但 Java 开发者只能站在门外干瞪眼——直到 Spring AI 出现。

Spring AI 是 Spring 官方推出的 AI 集成框架，核心理念和 Spring Boot 一脉相承：**约定优于配置，自动装配开箱即用**。你不用关心 OpenAI SDK 的底层 HTTP 调用，不用手写 JSON 解析，甚至不需要理解 SSE（Server-Sent Events）协议——一个 `ChatClient` Bean 搞定一切。

当前版本 Spring AI 1.0.0-M5 已经支持：

| 能力 | 支持范围 |
|------|---------|
| 模型供应商 | OpenAI / Azure OpenAI / Ollama / Anthropic / 智谱 / 通义千问 |
| 对话模式 | 同步 / 异步 / 流式 |
| Embedding | OpenAI / Ollama / Transformers |
| 向量存储 | Chroma / Milvus / Pgvector / Redis / Neo4j |
| 高级特性 | Prompt Template / RAG / Function Calling / ETL Pipeline |

本文带你**5 分钟**完成第一个 Spring AI 项目，代码量不超过 50 行。

---

## 二、环境准备

| 工具 | 版本要求 |
|------|---------|
| JDK | 17+ |
| Spring Boot | 3.2.x |
| Maven | 3.8+ 或 Gradle 8.x |
| OpenAI API Key | 需要科学上网且账户有余额 |

> **注意**：如果你没有 OpenAI API Key，可以用[智谱 AI](https://open.bigmodel.cn/)（国内可用，注册就送 token）替代，后面会给出配置方式。

---

## 三、5 分钟极速接入

### 3.1 创建 Spring Boot 项目

用 Spring Initializr（https://start.spring.io）创建项目，勾选以下依赖：

- **Spring Web**（提供 REST 接口）
- **Lombok**（减少样板代码）

或者直接复制下面的配置。

### 3.2 Maven 依赖配置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.5</version>
        <relativePath/>
    </parent>

    <groupId>com.example</groupId>
    <artifactId>spring-ai-demo</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>spring-ai-demo</name>

    <properties>
        <java.version>17</java.version>
        <spring-ai.version>1.0.0-M5</spring-ai.version>
    </properties>

    <dependencies>
        <!-- Spring Web -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- Spring AI OpenAI Starter -->
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
        </dependency>

        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
    </dependencies>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.ai</groupId>
                <artifactId>spring-ai-bom</artifactId>
                <version>${spring-ai.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <!-- Spring AI Milestone 仓库 -->
    <repositories>
        <repository>
            <id>spring-milestones</id>
            <name>Spring Milestones</name>
            <url>https://repo.spring.io/milestone</url>
            <snapshots>
                <enabled>false</enabled>
            </snapshots>
        </repository>
    </repositories>
</project>
```

如果是 Gradle 用户：

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.ai:spring-ai-openai-spring-boot-starter'
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
}

dependencyManagement {
    imports {
        mavenBom "org.springframework.ai:spring-ai-bom:1.0.0-M5"
    }
}

repositories {
    mavenCentral()
    maven { url 'https://repo.spring.io/milestone' }
}
```

### 3.3 application.yml 配置

```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY:sk-your-key-here}
      chat:
        options:
          model: gpt-4o
          temperature: 0.7
          max-tokens: 2000

server:
  port: 8080
```

> **安全提醒**：不要把 API Key 硬编码在配置文件中！生产环境请使用环境变量或 Vault。

**使用智谱 AI（国内可用）替代方案**：

```yaml
spring:
  ai:
    openai:
      api-key: ${ZHIPU_API_KEY:your-zhipu-key}
      base-url: https://open.bigmodel.cn/api/paas/v4
      chat:
        options:
          model: glm-4-flash
          temperature: 0.7
```

智谱 API 完全兼容 OpenAI 接口格式，Spring AI 无缝支持。

### 3.4 启动类

```java
package com.example.springaidemo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class SpringAiDemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(SpringAiDemoApplication.class, args);
    }
}
```

就这一个注解——跟普通 Spring Boot 项目完全一样，Spring AI 的自动装配已经在幕后完成了。

---

## 四、ChatClient 的 3 种使用方式

这是本文的核心。Spring AI 提供了 3 种调用方式，覆盖所有业务场景。

### 4.1 方式一：同步调用（最简单）

```java
package com.example.springaidemo.controller;

import lombok.RequiredArgsConstructor;
import org.springframework.ai.chat.client.ChatClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequiredArgsConstructor
public class ChatController {

    private final ChatClient.Builder chatClientBuilder;

    @GetMapping("/chat/sync")
    public String chatSync(@RequestParam(defaultValue = "用一句话介绍Java") String prompt) {
        ChatClient chatClient = chatClientBuilder.build();

        return chatClient.prompt()
                .user(prompt)
                .call()
                .content();
    }
}
```

启动项目，浏览器访问：

```
http://localhost:8080/chat/sync?prompt=用三句话介绍Spring Boot
```

返回结果：

```
Spring Boot 是一个基于 Spring 框架的快速应用开发工具，它通过"约定优于配置"的理念，
大幅简化了 Spring 应用的搭建和配置过程。内嵌了 Tomcat、Jetty 等服务器，让开发者
无需部署 WAR 包就能直接运行应用。同时还提供了 Starter 依赖管理、Actuator 监控、
自动配置等特性，让微服务开发效率显著提升。
```

**核心流程**：`prompt().user(prompt).call().content()` 链式调用，语义清晰。

### 4.2 方式二：异步调用（推荐用于 Web 场景）

同步调用会阻塞线程，在高并发场景下浪费资源。Spring AI 原生支持异步：

```java
@GetMapping("/chat/async")
public CompletableFuture<String> chatAsync(
        @RequestParam(defaultValue = "用一句话介绍Java") String prompt) {
    ChatClient chatClient = chatClientBuilder.build();

    return chatClient.prompt()
            .user(prompt)
            .call()
            .content()
            .thenApply(content -> {
                System.out.println("异步调用完成: " + Thread.currentThread().getName());
                return content;
            })
            .exceptionally(e -> "调用失败: " + e.getMessage());
}
```

Spring MVC 会自动将 `CompletableFuture` 转为异步响应——Tomcat 工作线程立即释放，不会阻塞。

### 4.3 方式三：流式返回（类 ChatGPT 逐字输出效果）

这是最有"AI 味儿"的调用方式——让用户看到 AI 逐字输出的过程：

```java
@GetMapping(value = "/chat/stream", produces = "text/event-stream")
public Flux<String> chatStream(
        @RequestParam(defaultValue = "用一句话介绍Java") String prompt) {
    ChatClient chatClient = chatClientBuilder.build();

    return chatClient.prompt()
            .user(prompt)
            .stream()
            .content();
}
```

`produces = "text/event-stream"` 告诉浏览器这是 SSE 流，`Flux<String>` 是 Reactor 的响应式类型，Spring WebFlux 自动将每个 token 逐段推送给客户端。

**前端调用示例**（原生 JavaScript）：

```javascript
const eventSource = new EventSource(
    'http://localhost:8080/chat/stream?prompt=用三句话介绍Java'
);

eventSource.onmessage = (event) => {
    // 每个 event.data 是一个 token 片段
    document.getElementById('output').textContent += event.data;
};
```

---

## 五、第一个完整的 AI 接口

把上面 3 种方式整合到一个 `ChatController` 中：

```java
package com.example.springaidemo.controller;

import lombok.RequiredArgsConstructor;
import org.springframework.ai.chat.client.ChatClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;
import reactor.core.publisher.Flux;

import java.util.concurrent.CompletableFuture;

@RestController
@RequiredArgsConstructor
public class ChatController {

    private final ChatClient.Builder chatClientBuilder;

    /** 同步调用 */
    @GetMapping("/chat/sync")
    public String chatSync(
            @RequestParam(defaultValue = "用一句话介绍Java") String prompt) {
        return chatClientBuilder.build()
                .prompt()
                .user(prompt)
                .call()
                .content();
    }

    /** 异步调用 */
    @GetMapping("/chat/async")
    public CompletableFuture<String> chatAsync(
            @RequestParam(defaultValue = "用一句话介绍Java") String prompt) {
        return chatClientBuilder.build()
                .prompt()
                .user(prompt)
                .call()
                .content()
                .toFuture();
    }

    /** 流式返回 */
    @GetMapping(value = "/chat/stream", produces = "text/event-stream")
    public Flux<String> chatStream(
            @RequestParam(defaultValue = "用一句话介绍Java") String prompt) {
        return chatClientBuilder.build()
                .prompt()
                .user(prompt)
                .stream()
                .content();
    }
}
```

**项目完整结构**：

```
spring-ai-demo/
├── pom.xml
├── src/main/java/com/example/springaidemo/
│   ├── SpringAiDemoApplication.java
│   └── controller/
│       └── ChatController.java
└── src/main/resources/
    └── application.yml
```

一共 **3 个文件，约 50 行代码**，就完成了 GPT-4 的接入。

---

## 六、ChatClient 的进阶玩法（预告）

上面的示例只用了 `ChatClient` 最基础的 API，实际上它支持：

### 6.1 系统提示词

```java
chatClient.prompt()
    .system("你是一个资深Java面试官，回答要专业且深入")
    .user("Spring的IoC容器是如何工作的？")
    .call()
    .content();
```

### 6.2 携带历史对话

```java
chatClient.prompt()
    .user("Java是什么？")        // 第1轮
    .call().content();

chatClient.prompt()
    .user("它是解释型还是编译型语言？")  // 第2轮（AI自动记住上文）
    .call().content();
```

> **注意**：这里的"自动记忆"取决于你如何管理对话历史。Spring AI 提供了 `InMemoryChatMemory` 和 `JdbcChatMemory` 两种方式，下篇文章会详细讲。

### 6.3 自定义模型参数

```java
chatClient.prompt()
    .user(prompt)
    .options(OpenAiChatOptions.builder()
        .withModel("gpt-4-turbo")
        .withTemperature(0.3)
        .withMaxTokens(500)
        .withTopP(0.8)
        .build())
    .call()
    .content();
```

---

## 七、常见问题排错

### 7.1 启动报 "OpenAiApi Key must not be null"

**原因**：没有配置 API Key。

**解决**：确认 `application.yml` 中 `spring.ai.openai.api-key` 已设置，或环境变量 `OPENAI_API_KEY` 已配置。

### 7.2 返回 429 Too Many Requests

**原因**：OpenAI API 限流，免费账户每分钟只有 3 次请求。

**解决**：降低调用频率，或升级付费账户；也可以改用智谱 AI 等国内服务。

### 7.3 流式接口前端收不到数据

**原因**：前端没有正确处理 SSE 协议。

**解决**：确认后端设置了 `produces = "text/event-stream"`，前端使用 `EventSource` 或 `fetch` + `ReadableStream` 消费。

### 7.4 返回 "Insufficient quota"

**原因**：OpenAI 账户余额不足或免费额度用完。

**解决**：充值或切换为其他 Provider。

### 7.5 国内网络访问不通

**原因**：OpenAI API 在国内被墙。

**解决方案有 3 种**：

1. **使用代理**：
```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      # Spring AI 1.0.0-M5 未直接支持代理配置，需要在 RestClient 层面配置
```

2. **使用国内兼容供应商**（推荐）：智谱、阿里通义千问、百度文心等都支持 OpenAI 兼容接口。

3. **本地部署 Ollama**：
```yaml
spring:
  ai:
    ollama:
      base-url: http://localhost:11434
      chat:
        model: llama3
```

### 7.6 环境变量配置不生效

在 macOS/Linux 上：

```bash
export OPENAI_API_KEY="sk-xxxxx"
```

在 Windows 上：

```cmd
set OPENAI_API_KEY=sk-xxxxx
```

或者在 IntelliJ IDEA 的 Run Configuration 中配置 Environment Variables。

---

## 八、ChatClient 的自动装配原理（面试加分）

你可能会好奇：`ChatClient.Builder` 这个 Bean 是从哪来的？答案是 Spring Boot 的自动装配机制。

Spring AI 的 `spring-ai-openai-spring-boot-starter` 包含 `spring.factories` 文件，指向 `OpenAiAutoConfiguration` 类。该类在启动时自动创建：

1. **`OpenAiApi`**：封装 HTTP 调用，读取 `spring.ai.openai.*` 配置
2. **`OpenAiChatModel`**：实现 Spring AI 的 `ChatModel` 接口
3. **`ChatClient.Builder`**：基于上述 Bean 构建 `ChatClient` 实例

了解这些对你排查问题、深度定制非常重要。

---

## 九、总结与下篇预告

本文带你从 0 到 1 完成了 Spring AI 的接入，核心就 3 步：

1. **添加 Maven 依赖**（spring-ai-openai-spring-boot-starter）
2. **配置 application.yml**（API Key + Model）
3. **注入 ChatClient.Builder 并调用**（同步/异步/流式）

但把 AI 服务真正用到生产环境，你还会遇到这些挑战：
- 高并发下连接池如何配置？
- 请求超时怎么合理设置？
- API 调用失败了要不要重试？重试几次？
- 下游服务挂了如何快速熔断？

**下一篇**：《Spring AI ChatClient 深度配置：连接池、超时、重试与熔断》，教你把这些"能跑"的代码变成"生产级"的服务。

---

> **本文代码仓库**：[GitHub - spring-ai-demo](https://github.com/example/spring-ai-demo)（占位）
>
> **系列目录**：
> - 081：Spring AI 入门 ← 本文
> - 082：[ChatClient 深度配置]（连接池/超时/重试/熔断）
> - 083：[Embedding 文本向量化]（语义搜索/相似度计算）
> - 084：[VectorStore 向量数据库集成]（Chroma/Milvus/Pgvector）
>
> **相关资源**：
> - [Spring AI 官方文档](https://docs.spring.io/spring-ai/reference/)
> - [OpenAI API 文档](https://platform.openai.com/docs)
> - [智谱 AI 开放平台](https://open.bigmodel.cn/)
