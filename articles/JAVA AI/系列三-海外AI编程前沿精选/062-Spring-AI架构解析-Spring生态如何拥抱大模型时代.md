# Spring AI 架构解析：Spring 生态如何拥抱大模型时代，看完源码你就懂了

---

## 一、开篇：Spring AI 不是一个 API 封装库

很多人第一次接触 Spring AI 时会想：「这不就是把 OpenAI 的 HTTP API 做了一层 Java 封装吗？」确实，`spring-ai-openai-spring-boot-starter` 引入后，两行 yaml 配好 API Key，就能 `@Autowired` 一个 `ChatClient` 开始对话了。但如果研究过 Spring AI 的完整源码，你会发现——**它远不止于此**。

Spring AI 的野心，是将 AI 能力**原生地融入 Spring 生态的每一个角落**：
- 和 Spring Boot AutoConfiguration 无缝集成
- 用 `@Tool` 注解让任意 Spring Bean 变成 AI 的工具
- 把 VectorStore 设计成 `spring-ai-commons` 的标准接口，让 MongoDB、Elasticsearch、Redis、Cassandra 都能扩展
- 将 RAG 管道组件化，像 Spring Integration 的 EIP 模式那样组装 AI 数据流

V1.0 正式版发布后，它与 Spring Boot 3.x、Spring Cloud、Spring Batch 的整合越来越成熟。本文从源码视角带你拆解 Spring AI 的核心架构。

---

## 二、分层架构：三大层次清晰解耦

Spring AI 的架构分三层，从上到下依次是：

```
┌──────────────────────────────────────────────────────────┐
│              Spring 生态层（Spring Boot / Cloud）          │
│  ┌───────────────────┐  ┌────────────┐ ┌──────────────┐ │
│  │ AutoConfiguration  │  │ Observability│ │ Spring Batch │ │
│  │ (零配置自动装配)     │  │ (Micrometer) │ │ (数据ETL)    │ │
│  └───────────────────┘  └────────────┘ └──────────────┘ │
├──────────────────────────────────────────────────────────┤
│                    Core 层（spring-ai-core）              │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────────┐ │
│  │ ChatClient│ │VectorStore│ │Document  │ │ToolCallback│ │
│  │ 统一聊天API│ │ 向量存储_   │ │Reader/Writer│ │ 工具调用   │ │
│  └──────────┘ └──────────┘ └──────────┘ └────────────┘ │
├──────────────────────────────────────────────────────────┤
│               Adapters 层（各模型/存储适配器）              │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────────┐ │
│  │ OpenAI   │ │  Azure   │ │  Ollama  │ │  Anthropic │ │
│  │ Adapter  │ │  OpenAI  │ │ Adapter  │ │  Adapter   │ │
│  └──────────┘ └──────────┘ └──────────┘ └────────────┘ │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────────┐ │
│  │ Chroma   │ │ Pinecone │ │ Milvus   │ │ MongoDB    │ │
│  │ Store    │ │  Store   │ │  Store   │ │  Store     │ │
│  └──────────┘ └──────────┘ └──────────┘ └────────────┘ │
└──────────────────────────────────────────────────────────┘
```

**Adapters 层**：每个模型提供商一个独立模块，只依赖 Core 层。比如 `spring-ai-openai` 把 OpenAI 的 REST API 封装为 `OpenAiChatModel`、`OpenAiEmbeddingModel`。更换模型只需换 Maven 依赖和配置文件。

**Core 层**：翻译层 + 抽象层。它将 Adapters 层的模型特定响应翻译为统一的 Spring AI 消息模型（`ChatResponse`/`ChatGeneration`），提供 `ChatClient`、`VectorStore`、`ToolCallingManager` 等高级抽象。

**Spring 生态层**：AutoConfiguration、Actuator 健康检查、Micrometer 指标、Spring Cloud 服务发现……这些都是原生的 Spring 设施，无需额外配置。

---

## 三、ChatClient：Builder 哲学的极致体现

`ChatClient` 是 Spring AI 最常用的入口，它的设计完整诠释了 Spring 一贯的 **Builder + 不可变配置** 哲学：

```java
@RestController
class AiController {

    private final ChatClient chatClient;

    // 通过 Builder 注入，支持多种配置组合
    public AiController(ChatClient.Builder builder) {
        this.chatClient = builder
            .defaultSystem("你是Java技术专家，请用中文回答")
            .defaultAdvisors(new SimpleLoggerAdvisor())
            .build();
    }

    @GetMapping("/chat")
    public String chat(@RequestParam String message) {
        return this.chatClient.prompt()
            .user(message)
            .call()
            .content();
    }

    // 多模型动态切换
    @GetMapping("/chat-with")
    public String chatWith(@RequestParam String model,
                           @RequestParam String message) {

        // prompt() 每次调用都创建新的 Prompt 实例（不可变）
        return this.chatClient.prompt()
            .user(message)
            .options(OpenAiChatOptions.builder()
                .withModel(model)  // 动态切换模型
                .withTemperature(0.7)
                .build())
            .call()
            .content();
    }
}
```

从源码看，`ChatClient` 的实现是典型的 **非侵入式不可变设计**：

```java
// ChatClient 接口（Spring AI 核心接口）
public interface ChatClient {

    ChatClientRequestSpec prompt();     // 入口：创建请求规范
    ChatClientRequestSpec prompt(Prompt prompt);  // 使用预构建 Prompt

    interface ChatClientRequestSpec {
        ChatClientRequestSpec user(String text);
        ChatClientRequestSpec user(Resource resource); // 图片、音频等多模态
        ChatClientRequestSpec system(String text);
        ChatClientRequestSpec messages(ChatMessage... messages);
        ChatClientRequestSpec advisors(RequestResponseAdvisor... advisors);
        ChatClientRequestSpec options(ChatOptions options);
        ChatClientRequestSpec functions(String... functionNames); // 注册工具
        ChatClientRequestSpec toolContext(Map<String, Object> context);

        ChatClientCallResponseSpec call();    // 同步调用
        Flux<ChatClientResponseSpec> stream(); // 流式调用
    }

    interface Builder {
        Builder defaultSystem(String systemText);
        Builder defaultOptions(ChatOptions options);
        Builder defaultAdvisors(RequestResponseAdvisor... advisors);
        Builder defaultUser(String userText);
        Builder defaultFunctions(String... functionNames);
        ChatClient build();
    }
}
```

**设计要点：**
- `Builder.build()` 返回的 `ChatClient` 是 **不可变的**（immutable），可以在多线程中安全共享
- `prompt()` 每次创建新的 `ChatClientRequestSpec` 实例，不会污染 Builder 中的默认配置
- `defaultAdvisors` 是实现 AOP 风格横切关注点的关键机制（日志、安全、缓存等）

**Advisor 链的处理流程：** 每个 Advisor 实现了 `RequestResponseAdvisor` 接口，形成一条类似 Spring AOP 的拦截器链：

```java
public interface RequestResponseAdvisor {

    // 在请求发送前处理 Prompt
    Prompt adviseRequest(Prompt prompt, Map<String, Object> context);

    // 在收到响应后处理 ChatResponse
    ChatResponse adviseResponse(ChatResponse response, Map<String, Object> context);
}
```

请求链的实际执行流是：

```
ChatClient.prompt().call()
    │
    ├── Advisor1.adviseRequest()  → 修改 Prompt
    ├── Advisor2.adviseRequest()  → 注入 RAG 上下文
    │
    ├── ChatModel.call(prompt)
    │
    ├── Advisor2.adviseResponse() → 后处理响应
    └── Advisor1.adviseResponse() → 后处理响应
```

---

## 四、VectorStore 抽象：为什么选 Interface 而不是 Abstract Class

这是 Spring AI 架构评审中最常被讨论的设计决策之一。我们来对比两种思路：

```java
// Spring AI 的设计：纯接口
public interface VectorStore {

    void add(List<Document> documents);

    Optional<Boolean> delete(List<String> idList);

    List<Document> similaritySearch(SearchRequest request);

    default List<Document> similaritySearch(String query) {
        return this.similaritySearch(SearchRequest.query(query));
    }
}
```

**为什么是 Interface？**

1. **不同向量数据库的实现逻辑完全不同。** MongoDB Atlas 的向量搜索基于 Mongoose 查询 + `$vectorSearch` 聚合管道，Elasticsearch 基于 `dense_vector` + `script_score` 或 kNN 查询，Redis 基于 `FT.SEARCH` 的向量相似度运算。它们在「添加文档」和「相似度搜索」这两个操作上没有一行相同的实现代码。强行用抽象类抽取「共通逻辑」只会变成空壳。

2. **Spring 的核心理念是「面向接口编程」。** 从 `DataSource` 到 `JdbcTemplate`，从 `RestTemplate` 到 `WebClient`，Spring 生态大量使用接口作为核心抽象。VectorStore 延续了这一传统。

3. **default 方法提供了灵活性。** `similaritySearch(String query)` 是 default 方法，实现类只需要实现 `similaritySearch(SearchRequest)` 即可。未来如果新增统一方法（比如 `hybridSearch`），可以通过 default 方法提供默认实现，不破坏现有适配器。

```java
// SearchRequest 的设计——用 Builder 模式支持丰富的检索参数
SearchRequest request = SearchRequest.query("什么是Spring AI？")
    .withTopK(5)
    .withSimilarityThreshold(0.7)
    .withFilterExpression("metadata['source'] == 'official-doc'");
```

---

## 五、Spring Boot AutoConfiguration 的集成魔法

Spring AI 与 Spring Boot 的对接，是通过标准 `AutoConfiguration` 机制实现的，核心在 `spring-ai-spring-boot-autoconfigure` 模块。

```java
@AutoConfiguration
@ConditionalOnClass(ChatModel.class)
@EnableConfigurationProperties(SpringAiProperties.class)
public class SpringAiChatAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public ChatClient chatClient(ChatClient.Builder chatClientBuilder) {
        return chatClientBuilder.build();
    }

    @Bean
    @ConditionalOnMissingBean
    public ChatClient.Builder chatClientBuilder(ChatModel chatModel) {
        return ChatClient.builder(chatModel);
    }
}
```

**自动装配的关键步骤：**

1. Spring Boot 启动时，`spring-boot-autoconfigure` 扫描 classpath 中的 `spring.factories` 或 `org.springframework.boot.autoconfigure.AutoConfiguration.imports` 文件
2. 发现 classpath 中有 `ChatModel`（说明引入了 Spring AI 依赖），触发 `SpringAiChatAutoConfiguration`
3. 根据配置文件 `spring.ai.openai.*` 或 `spring.ai.ollama.*` 的配置前缀，实例化对应的 ChatModel Bean
4. 将 ChatModel 注入 Builder，再构建默认的 ChatClient

```yaml
# application.yml 中的多模型配置
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      chat:
        options:
          model: gpt-4o
          temperature: 0.7
    ollama:
      base-url: http://localhost:11434
      chat:
        model: llama3.1:8b
```

**一个 AutoConfiguration 对应一个 Starter 模块，这是 Spring 的惯例：**

```
spring-ai-openai-spring-boot-starter
    ├── spring-ai-openai          (适配器)
    ├── spring-ai-openai-spring-boot-autoconfigure  (自动装配)
    └── spring-ai-spring-boot-autoconfigure   (通用自动装配)
```

---

## 六、Function Calling 的 Java 实现：@Tool 注解的处理流程

Function Calling 是 AI 应用的核心能力——让 LLM 能够调用外部 API、查询数据库、执行计算。Spring AI 通过 `@Tool` 注解实现了非常优雅的方案。

### 6.1 定义工具

```java
@Component
public class WeatherTools {

    @Tool(description = "获取指定城市的实时天气信息")
    public String getWeather(
        @ToolParam(description = "城市名称，如北京、Shanghai") String city) {

        // 实际项目中这里调用天气 API
        return "城市：" + city + "，温度：22°C，天气：晴，湿度：55%";
    }

    @Tool(description = "查询指定城市的未来天气预报")
    public String getForecast(
        @ToolParam(description = "城市名称") String city,
        @ToolParam(description = "预报天数，1-7") int days) {

        return city + "未来" + days + "天天气预报：多云转晴，温度18-26°C";
    }
}
```

### 6.2 注册工具

```java
@Configuration
public class AiConfig {

    @Bean
    public ChatClient chatClient(ChatClient.Builder builder,
                                  WeatherTools weatherTools) {
        return builder
            .defaultSystem("你是天气助手，帮助用户查询天气")
            .defaultTools(weatherTools)  // 注册工具
            .build();
    }
}
```

### 6.3 源码追踪：Tool 注解的处理流程

```java
// 1. Spring AI 扫描 @Tool 注解的方法
public class ToolCallbackResolver {

    public List<ToolCallback> resolve(Object beanWithTools) {
        List<ToolCallback> callbacks = new ArrayList<>();

        for (Method method : beanWithTools.getClass().getDeclaredMethods()) {
            Tool toolAnnotation = method.getAnnotation(Tool.class);
            if (toolAnnotation != null) {

                // 2. 构建工具定义（函数名、描述、参数Schema）
                ToolDefinition definition = ToolDefinition.builder()
                    .name(method.getName())
                    .description(toolAnnotation.description())
                    .inputSchema(buildJsonSchema(method)) // 从 @ToolParam 生成 JSON Schema
                    .build();

                // 3. 包装为 ToolCallback
                callbacks.add(new MethodToolCallback(definition, method, beanWithTools));
            }
        }
        return callbacks;
    }

    // 将 Java 方法参数转换为 OpenAI Function Calling 的 JSON Schema
    private String buildJsonSchema(Method method) {
        // ... 反射获取参数类型和 @ToolParam 描述，生成 JSON Schema
    }
}
```

**完整调用链路（一次包含 Function Calling 的对话）：**

```
用户: "北京明天天气怎么样？"
    │
    ▼
ChatClient.prompt().user("北京明天天气怎么样？").call()
    │
    ├── 1. Prompt 构建 + Tool 定义的 JSON Schema 附加
    │
    ├── 2. 调用 ChatModel.call(prompt)
    │       │
    │       ▼
    │   OpenAI API 返回:
    │   {
    │     "tool_calls": [{
    │       "function": {
    │         "name": "getWeather",
    │         "arguments": "{\"city\":\"北京\"}"
    │       }
    │     }]
    │   }
    │
    ├── 3. ToolCallingManager 解析 tool_calls
    │       │
    │       ├── 查找对应 ToolCallback
    │       ├── 解析 arguments JSON → 构造方法参数
    │       ├── 通过反射调用 getWeather("北京")
    │       └── 获取结果: "城市：北京，温度：22°C..."
    │
    ├── 4. 将工具执行结果作为新的 ChatMessage 追加到对话历史
    │
    ├── 5. 再次调用 ChatModel.call()（Agent Loop）
    │
    └── 6. LLM 基于工具结果生成自然语言回复
          "北京的天气是晴天，温度22°C，湿度55%，很适合出行。"
```

这个流程被称为 **Agent Loop**——LLM 和工具之间形成一个「提议-执行-观察」的循环。Spring AI 的 `ToolCallingManager` 封装了整个循环，开发者不需要手动管理。

---

## 七、一次 Chat 请求的完整调用链路（源码走读）

把整个链路串起来，从 Controller 到 OpenAI API 响应，每一个环节的源码都经过阅读和标注：

```
Controller.chat()
    │
    ▼
ChatClient.DefaultChatClient.call()
    │ (将 Prompt 传给 Advisor 链)
    ├── SimpleLoggerAdvisor.adviseRequest()
    │     记录请求日志
    │
    ▼
DelegatingChatModel.call(prompt)
    │ (根据 prompt 中的 ChatOptions 选择具体模型实例)
    │
    ▼
OpenAiChatModel.call(prompt)
    │
    ├── 1. 将 Spring AI 的 Prompt → OpenAI ChatCompletionRequest
    │      messages: [{role:"system", content:"..."}, {role:"user", content:"..."}]
    │      model: "gpt-4o", temperature: 0.7, tools: [...]
    │
    ├── 2. HTTP POST https://api.openai.com/v1/chat/completions
    │       Header: Authorization: Bearer sk-xxx
    │       Body: { ... }
    │
    ├── 3. 接收 OpenAI 响应 → 反序列化为 OpenAiApi.ChatCompletion
    │      {
    │        choices: [{
    │          message: {role:"assistant", content:"Spring AI是..."},
    │          finish_reason: "stop"
    │        }],
    │        usage: {prompt_tokens: 150, completion_tokens: 200, total_tokens: 350}
    │      }
    │
    ├── 4. 将 OpenAI 响应 → Spring AI ChatResponse
    │      new ChatResponse(
    │          List.of(new Generation(new AssistantMessage("Spring AI是..."))),
    │          new ChatResponseMetadata(...)
    │      )
    │
    └── 5. 返回 ChatResponse
    │
    ▼
ChatClient.DefaultChatClient.call() → 继续
    │
    ├── Advisor 链 adviseResponse()（如果有）
    │
    ▼
ChatClientCallResponseSpec.content()
    │ (从 ChatResponse 提取第一个 Generation 的文本内容)
    │
    ▼
返回 String 给 Controller
```

---

## 八、Spring AI vs LangChain4j：设计理念大对比

这两个框架常常被拿来比较。我的看法是：**它们不是竞争关系，而是互补关系**。

| 维度 | Spring AI | LangChain4j |
|------|-----------|-------------|
| **出身** | Spring 官方出品 | 社区驱动（独立项目） |
| **核心哲学** | Spring 生态原生集成 | Java 语言特性最大利用 |
| **与 Spring Boot 的集成** | 原生 AutoConfiguration，深度嵌入 | 需要手动适配，或可选 Starter |
| **API 风格** | Builder + Advisor 链 | Builder + AiServices 动态代理 |
| **AI 服务定义** | `ChatClient` 编程式 + `@Tool` 函数式 | `AiServices` 声明式接口 + `@Tool` |
| **RAG 支持** | 原生 ETL 管道（DocumentReader → Transformer → Writer） | 链式 API + Adapter 模式 |
| **可观测性** | 原生 Micrometer / Actuator 集成 | 内置 logRequests/logResponses |
| **模型支持广度** | 10+ 模型提供商 | 20+ 模型提供商（包括国产模型） |
| **向量数据库支持** | 通过接口可插拔 | 通过接口可插拔 |
| **云原生** | 与 Spring Cloud 天然集成 | 需要自行搭建 |
| **生态位置** | Spring 全家桶的 AI 板块 | Java AI 框架的独立选择 |

**选型建议：**
- 如果你的项目已经是 Spring Boot 生态，且需要一个「能干活」的 AI 集成——选 **Spring AI**
- 如果你需要一个完全框架无关的 Java AI 工具包，且想用 `AiServices` 那种声明式接口——选 **LangChain4j**
- 实际上，两者可以混用——LangChain4j 的某些适配器（如对国产模型的支持）比 Spring AI 更全，可以搭配使用

---

## 九、总结与预告

Spring AI 的设计哲学，一言以蔽之：**「把 AI 变成 Spring 生态中的一等公民」**。它不是简单地封装 API，而是从 Spring 的体系出发——分层架构、接口抽象、Builder 模式、AutoConfiguration——让 AI 能力像数据库访问、消息队列、安全控制一样，成为 Spring 开发生态中一个自然、标准化的组件。

你不需要学一套新的框架范式，只需要用你已经熟悉的 Spring 方式——定义 Bean、注入依赖、配置 YAML——就能构建功能完整的 AI 应用。

---

**下一篇预告：** 我们将飞出 Java 生态，深入剖析国内最成功的开源 LLM 应用开发平台——**Dify**。GitHub 60K+ Star、Python Flask + React、DAG 工作流引擎、多模型适配层……Dify 的技术架构中蕴藏着哪些 Java 开发者值得借鉴的智慧？敬请期待！
