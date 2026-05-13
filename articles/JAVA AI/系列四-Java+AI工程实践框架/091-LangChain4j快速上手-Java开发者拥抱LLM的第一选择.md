---
title: LangChain4j 快速上手：Java 开发者拥抱 LLM 的第一选择，比 Spring AI 更灵活的 AI 框架
tags: [LangChain4j, Java, LLM, AI, Spring AI, 大模型, Agent, RAG]
---

# LangChain4j 快速上手：Java 开发者拥抱 LLM 的第一选择

> Spring AI 是官方正统，但 LangChain4j 可能是更灵活的选项。本文带你深度对比两大框架，并用 LangChain4j 快速构建你的第一个 AI 应用。

---

## 一、开篇：Java 开发者的 AI 框架选择困境

如果你是一个 Java 后端开发，最近想在项目里接入大模型，一定会遇到这个灵魂拷问：

> **Spring AI 还是 LangChain4j？**

这两个框架是目前 Java 生态中最主流的大模型接入方案。它们的 GitHub Star 数量都超过了 5K，都有活跃的社区和稳定的 API。但选错框架的代价是巨大的——一旦在一个中大型项目里铺开，后期迁移成本高到让人窒息。

先说结论，省去你翻长文的功夫：

| 维度 | Spring AI | LangChain4j |
|------|-----------|-------------|
| **一句话评价** | Spring 生态的官方 AI 方案 | Java 版 LangChain，更灵活 |
| **上手门槛** | 低（Spring Boot Starter 即用） | 中（需要理解一些概念） |
| **模型支持** | ~10 家厂商 | 30+ 厂商（含国产） |
| **高级特性** | ETL、向量存储、输出解析 | RAG、Agent、Tools、记忆 |
| **Spring 集成** | 原生深度集成 | 提供 Spring Boot Starter |
| **灵活性** | 中等，遵循 Spring 风格 | 高，链式调用、可组合 |
| **社区成熟度** | 背靠 Spring 官方 | 社区驱动，更新频率极高 |

**我的建议**：
- 如果你的项目深度绑定 Spring 生态，追求开箱即用 → Spring AI
- 如果你需要更灵活的 AI 编排能力（Agent、RAG、复杂 Chain）→ LangChain4j
- 两个框架可以共存——LangChain4j 也有 Spring Boot Starter

本文聚焦 LangChain4j，带你从零快速上手，并深度对比两者的设计哲学差异。

---

## 二、LangChain4j 是什么？5 句话讲清

1. LangChain4j 是 Java 生态的 LangChain 对标实现，但**不是简单的翻译版**
2. 它提供了**统一的 LLM 抽象层**，一套 API 对接 30+ 语言模型厂商
3. 内置了 **RAG（检索增强生成）、AI Services（声明式 AI）、Agent（智能体）** 等高级能力
4. 支持**同步 + 异步 + 流式**三种调用模式
5. 对 Spring Boot / Quarkus / Micronaut 均提供 Starter 支持

> LangChain4j 的哲学：**不重新发明轮子，但给你组装轮子的最佳方式。**

---

## 三、LangChain4j vs Spring AI：深度架构对比

这可能是全网最详细的 Java AI 框架对比分析。

### 3.1 设计理念的根本差异

**Spring AI 的设计哲学**：用 Spring 一贯的"模板方法+Builder"模式封装 AI 调用。如果你熟悉 `JdbcTemplate` 或 `RestTemplate`，Spring AI 的上手曲线几乎为零。

```java
// Spring AI 风格：Builder + Template
String response = chatClient.prompt()
    .user("Hello")
    .call()
    .content();
```

**LangChain4j 的设计哲学**：借鉴 LangChain 的"链式编排"思想，同时拥抱 Java 的接口驱动风格。它有两种编程模型——**链式 API** 和 **声明式 AI Services**。

```java
// LangChain4j 风格：链式 API
String response = ChatLanguageModel.generate("Hello");

// LangChain4j 风格：声明式 AI Services
interface Assistant {
    @SystemMessage("You are a helpful assistant")
    String chat(@UserMessage String message);
}
```

### 3.2 模型支持对比

LangChain4j 支持的模型厂商数量碾压 Spring AI：

| 厂商 | Spring AI | LangChain4j |
|------|:---:|:---:|
| OpenAI | ✅ | ✅ |
| Azure OpenAI | ✅ | ✅ |
| Anthropic Claude | ✅ | ✅ |
| Google Gemini | ✅ | ✅ |
| Ollama | ✅ | ✅ |
| 通义千问 | ✅ | ✅ |
| 文心一言 | ❌ | ✅ |
| 智谱 GLM | ❌ | ✅ |
| 月之暗面 Moonshot | ❌ | ✅ |
| Mistral | ✅ | ✅ |
| DeepSeek | ❌ | ✅ |
| 零一万物 | ❌ | ✅ |
| AWS Bedrock | ❌ | ✅ |
| HuggingFace | ❌ | ✅ |
| Vertex AI | ✅ | ✅ |
| 本地模型 (llama.cpp) | ❌ | ✅ |

LangChain4j 的模型覆盖几乎是 Spring AI 的两倍，**尤其在国内模型的接入上遥遥领先**。

### 3.3 高级特性对比

| 特性 | Spring AI | LangChain4j |
|------|:---:|:---:|
| 提示词模板 | PromptTemplate | PromptTemplate |
| 输出解析 | BeanOutputConverter | AiServices（自动映射） |
| 向量存储 | ✅ 10+ 种 | ✅ 20+ 种 |
| RAG（检索增强） | ETL 框架 | 完整 RAG 模块 |
| Chat Memory | ✅ | ✅ 更丰富 |
| Agent / Tools | Planned | ✅ 内置 |
| 链式调用 | ❌ | ✅ Chain/Sequential |
| 流式输出 | ✅ | ✅ |
| 多模态 | ✅ | ✅ |
| 嵌入模型 | ✅ | ✅ 更多 |

**最大差异在于 Agent 和 Chain**：LangChain4j 内置了完整的 Agent 框架，支持 Tool Calling，可以自动规划并执行多步任务。Spring AI 目前还是简单的请求->响应模式。

---

## 四、快速上手：5 分钟跑通第一个 LangChain4j 应用

### 4.1 Maven 依赖

```xml
<dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j-open-ai</artifactId>
    <version>0.35.0</version>
</dependency>
<!-- Spring Boot Starter（可选） -->
<dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j-open-ai-spring-boot-starter</artifactId>
    <version>0.35.0</version>
</dependency>
```

> 当前最新版本为 0.35.0，LangChain4j 版本迭代非常快，建议去 [GitHub Release](https://github.com/langchain4j/langchain4j/releases) 确认最新版本。

### 4.2 最简单的 Hello World

```java
import dev.langchain4j.model.openai.OpenAiChatModel;

public class HelloLangChain4j {
    public static void main(String[] args) {
        // 最简单的调用 —— 一行代码
        OpenAiChatModel model = OpenAiChatModel.builder()
                .apiKey(System.getenv("OPENAI_API_KEY"))
                .modelName("gpt-4o")
                .build();

        String answer = model.generate("用Java写一个Hello World");
        System.out.println(answer);
    }
}
```

### 4.3 Spring Boot 集成方式

```yaml
# application.yml
langchain4j:
  open-ai:
    chat-model:
      api-key: ${OPENAI_API_KEY}
      model-name: gpt-4o
      temperature: 0.7
      max-tokens: 2000
      timeout: 60s
      log-requests: true
      log-responses: true
```

```java
@RestController
@RequestMapping("/api/ai")
public class AiController {

    @Autowired
    private ChatLanguageModel chatLanguageModel;

    @PostMapping("/chat")
    public String chat(@RequestBody Map<String, String> request) {
        return chatLanguageModel.generate(request.get("message"));
    }
}
```

> 只要引入 Starter，LangChain4j 会自动注入 `ChatLanguageModel` Bean，无需任何额外配置。

---

## 五、核心模块全景图

LangChain4j 不是一个大一统的 JAR，而是高度模块化的。按需引入，避免依赖膨胀。

### 5.1 模块一览

| 模块 | 坐标 | 说明 |
|------|------|------|
| 核心 | langchain4j | ChatLanguageModel 接口、Memory、Chain |
| OpenAI | langchain4j-open-ai | OpenAI / Azure 对接 |
| Ollama | langchain4j-ollama | 本地模型对接 |
| Google Gemini | langchain4j-google-ai-gemini | Gemini 对接 |
| DashScope | langchain4j-dashscope | 通义千问对接 |
| Qianfan | langchain4j-qianfan | 文心一言对接 |
| Zhipu | langchain4j-zhipu-ai | 智谱 GLM 对接 |
| RAG | langchain4j-rag-core | 检索增强生成 |
| Easy RAG | langchain4j-easy-rag | 开箱即用的 RAG |
| 文档解析 | langchain4j-document-parser-apache-tika | PDF/Word 等解析 |
| 向量存储 | langchain4j-elasticsearch | Elasticsearch 向量存储 |
| 向量存储 | langchain4j-milvus | Milvus 向量存储 |
| 向量存储 | langchain4j-chroma | Chroma 向量存储 |
| Spring Boot | langchain4j-open-ai-spring-boot-starter | Spring Boot 自动配置 |

### 5.2 核心接口：ChatLanguageModel

这是 LangChain4j 最重要的接口，所有对话模型的统一抽象：

```java
public interface ChatLanguageModel {
    // 同步生成
    String generate(String userMessage);

    // 同步生成（多轮对话）
    Response<AiMessage> generate(ChatMessage... messages);
    Response<AiMessage> generate(List<ChatMessage> messages);

    // 异步生成
    CompletableFuture<Response<AiMessage>> generateAsync(ChatMessage... messages);
    CompletableFuture<Response<AiMessage>> generateAsync(List<ChatMessage> messages);

    // 流式生成
    void generate(String userMessage, StreamingResponseHandler<AiMessage> handler);
}
```

### 5.3 消息类型体系

LangChain4j 定义了清晰的消息类型层次：

```java
// SystemMessage - 系统提示词（设定角色/规则）
SystemMessage systemMessage = SystemMessage.from("你是一个精通Java的编程助手");

// UserMessage - 用户消息
UserMessage userMessage = UserMessage.from("什么是Java的虚引用？");

// AiMessage - AI 回复
AiMessage aiMessage = AiMessage.from("虚引用是Java中最弱的引用类型...");

// ToolExecutionResultMessage - 工具调用结果（Agent 场景）
```

完整对话示例：

```java
List<ChatMessage> conversation = List.of(
    SystemMessage.from("你是一个Java面试官"),  // 定义角色
    UserMessage.from("今天面试什么内容？"),      // 用户问
    AiMessage.from("今天考察JVM内存模型"),        // AI答
    UserMessage.from("请出题")                   // 用户追问
);

Response<AiMessage> response = model.generate(conversation);
System.out.println(response.content().text());
```

---

## 六、高级实战：Chain 链式调用

LangChain4j 的精髓在于 **Chain（链式调用）**——你可以把多个 LLM 调用像搭积木一样串联起来。

### 6.1 顺序链：先翻译再总结

```java
import dev.langchain4j.chain.Chain;
import dev.langchain4j.chain.ConversationalChain;
import dev.langchain4j.chain.SequentialChain;

// 链1：翻译
ChatLanguageModel translator = OpenAiChatModel.builder()
        .apiKey(System.getenv("OPENAI_API_KEY"))
        .modelName("gpt-3.5-turbo")  // 翻译用便宜的模型
        .build();

// 链2：总结
ChatLanguageModel summarizer = OpenAiChatModel.builder()
        .apiKey(System.getenv("OPENAI_API_KEY"))
        .modelName("gpt-4o")          // 总结用高级模型
        .build();

// 构建顺序链
SequentialChain chain = new SequentialChain(
    message -> translator.generate("将以下内容翻译成中文：" + message),
    message -> summarizer.generate("用一句话总结以下内容：" + message)
);

String result = chain.execute("""
    The Spring Framework provides a comprehensive programming
    and configuration model for modern Java-based enterprise
    applications - on any kind of deployment platform.
    """);

System.out.println(result);
// 输出：Spring框架为现代Java企业应用提供了全面的编程和配置模型。
```

### 6.2 路由链：根据意图自动路由

```java
public class IntentRouter {

    private final Map<String, ChatLanguageModel> executors;
    private final ChatLanguageModel router;

    public IntentRouter() {
        this.router = OpenAiChatModel.builder()
                .apiKey(System.getenv("OPENAI_API_KEY"))
                .modelName("gpt-3.5-turbo")
                .build();

        this.executors = Map.of(
            "code", OpenAiChatModel.builder().modelName("gpt-4o").build(),
            "translate", OpenAiChatModel.builder().modelName("gpt-3.5-turbo").build(),
            "summary", OpenAiChatModel.builder().modelName("gpt-4o-mini").build()
        );
    }

    public String execute(String userInput) {
        // 第一步：让路由模型判断意图
        String intent = router.generate("""
            判断用户意图，只返回以下关键词之一：
            code（写代码）、translate（翻译）、summary（总结）
            
            用户输入：%s
            """.formatted(userInput)).trim();

        // 第二步：根据意图调用对应的模型
        return executors.getOrDefault(intent, executors.get("code"))
                .generate(userInput);
    }
}
```

---

## 七、对接国产大模型（通义千问为例）

```xml
<dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j-dashscope</artifactId>
    <version>0.35.0</version>
</dependency>
```

```java
import dev.langchain4j.model.dashscope.QwenChatModel;

QwenChatModel model = QwenChatModel.builder()
        .apiKey(System.getenv("DASHSCOPE_API_KEY"))
        .modelName("qwen-max")
        .build();

String response = model.generate("用Java实现一个LRU缓存");
System.out.println(response);
```

对接文心一言：

```xml
<dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j-qianfan</artifactId>
    <version>0.35.0</version>
</dependency>
```

```java
import dev.langchain4j.model.qianfan.QianfanChatModel;

QianfanChatModel model = QianfanChatModel.builder()
        .apiKey(System.getenv("QIANFAN_API_KEY"))
        .secretKey(System.getenv("QIANFAN_SECRET_KEY"))
        .modelName("ERNIE-4.0-8K")
        .build();
```

对接智谱 GLM：

```xml
<dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j-zhipu-ai</artifactId>
    <version>0.35.0</version>
</dependency>
```

```java
import dev.langchain4j.model.zhipu.ZhipuAiChatModel;

ZhipuAiChatModel model = ZhipuAiChatModel.builder()
        .apiKey(System.getenv("ZHIPU_API_KEY"))
        .modelName("glm-4")
        .build();
```

---

## 八、生产环境配置建议

### 8.1 全局超时与重试

```java
OpenAiChatModel model = OpenAiChatModel.builder()
        .apiKey(System.getenv("OPENAI_API_KEY"))
        .modelName("gpt-4o")
        .timeout(Duration.ofSeconds(60))
        .maxRetries(3)
        .logRequests(true)
        .logResponses(true)
        .build();
```

### 8.2 连接池与并发控制

```java
@Configuration
public class LangChain4jConfig {

    @Bean
    public ChatLanguageModel chatLanguageModel() {
        return OpenAiChatModel.builder()
                .apiKey(System.getenv("OPENAI_API_KEY"))
                .modelName("gpt-4o")
                .timeout(Duration.ofSeconds(60))
                .maxRetries(2)
                .build();
    }
}
```

### 8.3 流式输出支持

```java
@PostMapping("/stream-chat")
public SseEmitter streamChat(@RequestBody Map<String, String> request) {
    SseEmitter emitter = new SseEmitter(0L);  // 无超时限制

    StreamingChatLanguageModel streamingModel = OpenAiStreamingChatModel.builder()
            .apiKey(System.getenv("OPENAI_API_KEY"))
            .modelName("gpt-4o")
            .build();

    streamingModel.generate(request.get("message"), new StreamingResponseHandler<>() {
        @Override
        public void onNext(String token) {
            try {
                emitter.send(SseEmitter.event().data(token));
            } catch (IOException e) {
                emitter.completeWithError(e);
            }
        }

        @Override
        public void onComplete(Response<AiMessage> response) {
            emitter.complete();
        }

        @Override
        public void onError(Throwable error) {
            emitter.completeWithError(error);
        }
    });

    return emitter;
}
```

---

## 九、总结

本文带你全面了解了 LangChain4j：

1. **定位**：Java 生态中功能最完整的 LLM 框架
2. **对比**：Spring AI 胜在简洁和 Spring 原生集成，LangChain4j 胜在灵活性和高级特性
3. **上手**：3 行代码跑通，Spring Boot Starter 自动配置
4. **能力**：支持 30+ 模型厂商、Chain 链式调用、流式输出、RAG、Agent
5. **国产模型**：通义千问、文心一言、智谱 GLM、DeepSeek 全支持

**我的选型建议**：
- 新手 / Spring 深度项目 / 简单 AI 需求 → Spring AI
- 复杂 AI 编排 / Agent / 需要多模型 / 非 Spring 项目 → LangChain4j
- 两者可以共存，LangChain4j 也有 Spring Boot Starter

---

**下一篇预告**：《LangChain4j AiServices：接口驱动的 AI 服务声明式开发》—— 定义接口 = 完成 AI 功能。LangChain4j 的 AiServices 用 Java 接口 + 注解的方式实现 AI 服务的声明式开发，彻底消灭模板代码。敬请期待！

---

> 作者：IT 老熊
> 标签：LangChain4j, Java, LLM, Spring AI, 大模型
> 原文首发：CSDN 技术社区
