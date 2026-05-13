# LangChain4j 深度源码解读：Java 版 LangChain 的设计哲学，为什么 Java 开发者应该选它而不是自己调 API

---

## 一、开篇：Python 有 LangChain，Java 有什么？

如果你关注 AI 应用开发，一定听说过 **LangChain**。它是 Python 生态中最主流的 LLM 应用框架，GitHub 100K+ Star。但当你兴奋地想把它引入公司的 Java 项目时，却遭遇了「语言壁垒」——团队全是 Java 技术栈，不可能为了接 AI 把整个后端改成 Python。

有人会说：「那直接用 HTTP 调 OpenAI API 不就行了？」确实可以，但当你需要 RAG（检索增强生成）、Agent、工具调用、对话记忆管理这些高级功能时，裸调 API 的工作量会呈指数级增长。你会在 500 行代码后才意识到——**自己正在手动实现一个劣化版的 LangChain**。

**LangChain4j 就是 Java 生态对这个问题的终极答案。** 它是 LangChain 的 Java 移植，但不是简单的 API 照搬，而是从 Java 的设计哲学出发，重新思考了 AI 应用框架的架构。

作为一个看过它全部核心源码的 Java 程序员，我的结论是：**LangChain4j 的设计比 Python 版更「Java」——接口驱动、模块化、强类型、Builder 模式**。接下来，我带你深入源码，看看它到底怎么设计的。

---

## 二、核心架构：四大抽象

LangChain4j 的整体架构可以概括为 **「核心抽象 + 模块化适配器」** 的两层结构：

```
┌──────────────────────────────────────────────────────────────────┐
│                         LangChain4j                              │
├──────────────────────────────────────────────────────────────────┤
│  应用层                                                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │ AiServices   │  │ Chain.of()  │  │ Agent       │              │
│  │ (声明式AI)    │  │ (链式调用)   │  │ (工具调用)   │              │
│  └──────┬───────┘  └──────┬──────┘  └──────┬──────┘              │
├─────────┼─────────────────┼─────────────────┼────────────────────┤
│  核心层  │                  │                  │                  │
│  ┌───────▼─────────────────▼──────────────────▼───────────────┐  │
│  │  ChatLanguageModel / EmbeddingModel / ChatMemory / Tool   │  │
│  │  StreamingChatLanguageModel / ImageModel / ModerationModel │  │
│  └───────────────────────────────────────────────────────────┘  │
├──────────────────────────────────────────────────────────────────┤
│  适配层（各模型提供商实现）                                           │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐           │
│  │ OpenAI    │ │  Azure    │ │ Ollama   │ │ Gemini   │           │
│  │ Adapter   │ │ OpenAl    │ │ Adapter  │ │ Adapter  │  ...      │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘           │
└──────────────────────────────────────────────────────────────────┘
```

四大核心接口是整个框架的基石：

### 2.1 ChatLanguageModel —— 对话的核心

```java
// langchain4j-core/src/main/java/dev/langchain4j/model/chat/ChatLanguageModel.java
public interface ChatLanguageModel {

    /**
     * 发送聊天请求并返回 AI 的响应
     */
    Response<AiMessage> generate(List<ChatMessage> messages);

    /**
     * 发送聊天请求，接收工具调用规范
     */
    Response<AiMessage> generate(List<ChatMessage> messages, List<ToolSpecification> toolSpecifications);

    /**
     * 同上，但直接获取工具的名字
     */
    Response<AiMessage> generate(List<ChatMessage> messages, ToolSpecification toolSpecification);

    default Response<AiMessage> generate(ChatMessage... messages) {
        return generate(asList(messages));
    }
}
```

这个接口的设计非常克制——只定义了一个 `generate` 方法（多个重载），不定义任何实现细节。**为什么不用抽象类？** 因为不同的模型提供商（OpenAI、Azure、Ollama）底层 HTTP 协议完全不同，没有共通的实现逻辑可以抽取。用接口可以保证最大的灵活性。

### 2.2 EmbeddingModel —— 向量化的基石

```java
public interface EmbeddingModel {

    // 对文本片段进行向量化（RAG 的核心步骤）
    Response<List<Embedding>> embedAll(List<TextSegment> textSegments);

    // 对单条文本向量化
    default Response<Embedding> embed(String text) {
        return embed(TextSegment.from(text));
    }

    default Response<Embedding> embed(TextSegment textSegment) {
        Response<List<Embedding>> response = embedAll(singletonList(textSegment));
        return Response.from(embeddingFrom(response), response.tokenUsage(), response.finishReason());
    }
}
```

RAG 的核心就是把文档切成小段，每段转成向量存起来。`EmbeddingModel` 就是这一步骤的抽象。这里有一个设计细节值得注意：`embedAll` 支持批量向量化，因为很多 Embedding API（如 OpenAI）按调用次数计费，分批发送可以减少 HTTP 往返次数。

### 2.3 ChatMemory —— 对话记忆管理

```java
public interface ChatMemory {

    Object id();

    void add(ChatMessage message);

    List<ChatMessage> messages();

    default boolean isEmpty() {
        return messages().isEmpty();
    }

    void clear();
}
```

这个接口看似简单，但实现类 `MessageWindowChatMemory` 和 `TokenWindowChatMemory` 分别代表了两种记忆管理策略：

```java
// 按消息数量限制历史对话
ChatMemory memory = MessageWindowChatMemory.withMaxMessages(10);

// 按 Token 数量限制（更精确，避免超出模型上下文窗口）
ChatMemory memory = TokenWindowChatMemory.withMaxTokens(
    3000, new OpenAiTokenizer(GPT_4_O_MINI));
```

**Token 窗口记忆的实现思路：** 它内部维护一个 `LinkedList`，每次 `add` 追加新消息，然后用 `Tokenizer` 估算总 Token 数。超过阈值时，从队列头部逐个移除最早的消息，直到 Token 总数回到安全水位线。

### 2.4 Tool —— 赋予 LLM「调用代码」的能力

```java
public interface Tool {

    ToolSpecification toolSpecification();

    ToolExecutionResult execute(ToolExecutionRequest request, Object memoryId);
}
```

`ToolSpecification` 描述了工具的名称、参数 Schema（JSON Schema 格式），`execute` 负责实际执行。当一个带 `@Tool` 注解的 Java 方法被注册后，框架会在 `generate` 时将其规格传给 LLM；LLM 返回一个 `ToolExecutionRequest`（JSON 格式），框架解析它并调用 `execute`，再把结果作为新的上下文传给 LLM 形成 Agent Loop。

---

## 三、AiServices：LangChain4j 最惊艳的设计

如果说四大核心接口是骨架，那 **AiServices** 就是 LangChain4j 的灵魂。这是 Python 版 LangChain 完全没有的设计，也是我认为 LangChain4j 最值得研究的部分。

**传统方式调用 LLM：**

```java
ChatLanguageModel model = OpenAiChatModel.builder()
    .apiKey("sk-xxx")
    .modelName("gpt-4o")
    .build();

String response = model.generate(
    SystemMessage.from("你是Java专家"),
    UserMessage.from("解释Spring的IoC")
).content().text();
```

每次都要手动拼接 messages，手动解析响应，手动管理记忆，非常繁琐。

**AiServices 方式：**

```java
// 1. 定义一个接口
interface Assistant {
    @SystemMessage("你是Java专家，用中文回答，代码示例使用Java17+")
    String chat(String userMessage);
}

// 2. 创建代理实例
ChatLanguageModel model = OpenAiChatModel.builder()
    .apiKey(System.getenv("OPENAI_API_KEY"))
    .modelName("gpt-4o")
    .build();

Assistant assistant = AiServices.create(Assistant.class, model);

// 3. 像调用普通接口一样使用
String answer = assistant.chat("解释一下Spring的IoC容器");
```

**一步到位！** 不需要手动拼接 Prompt，不需要解析 Response，不需要管理 Token——一切都由代理对象在幕后完成。

### 3.1 源码剖析：动态代理的实现

`AiServices` 的工作原理是 **Java 动态代理**。核心在 `AiServiceFactory.create()` 方法：

```java
// AiServiceFactory.java (核心逻辑简化)
public static <T> T create(Class<T> aiServiceClass, ChatLanguageModel model) {

    AiServiceContext context = new AiServiceContext(aiServiceClass);
    context.chatModel = model;

    // 关键：解析接口上的注解
    SystemMessage systemMsg = parseSystemMessage(aiServiceClass); // ①
    List<ToolSpecification> toolSpecs = parseToolAnnotations(aiServiceClass); // ②
    ChatMemoryProvider memoryProvider = parseMemoryProvider(aiServiceClass); // ③

    // 动态代理
    @SuppressWarnings("unchecked")
    T proxyInstance = (T) Proxy.newProxyInstance(
        aiServiceClass.getClassLoader(),
        new Class<?>[] { aiServiceClass },
        new AiServiceInvocationHandler(context)  // ④
    );

    return proxyInstance;
}
```

当调用 `assistant.chat("解释Spring的IoC")` 时，执行流程是：

```
assistant.chat("Spring IoC")
    │
    ▼
InvocationHandler.invoke()
    │
    ├── ① 从 @SystemMessage 获取系统提示词
    ├── ② 从 ChatMemoryProvider 获取对话历史
    ├── ③ 构建完整的 List<ChatMessage>
    ├── ④ 将 Tool 规格附加到请求中（如果有 @Tool 方法）
    ├── ⑤ 调用 ChatLanguageModel.generate()
    ├── ⑥ 检查响应是否为 ToolExecutionRequest
    │     ├── 是 → 调用 Tool.execute() → 返回结果给 LLM → 回到⑤
    │     └── 否 → 继续
    └── ⑦ 将响应保存到 ChatMemory
                     │
                     ▼
           返回 String 给调用方
```

### 3.2 SystemMessage 解析

```java
private static SystemMessage prepareSystemMessage(
    AiServiceContext context, Object memoryId) {

    // 从接口注解获取
    SystemMessage annotationMsg = getSystemMessageAnnotation(context);

    // 从记忆中的 SystemMessage 获取（如果支持）
    Optional<SystemMessage> memoryMsg = getSystemMessageFromMemory(context, memoryId);

    return memoryMsg.orElse(annotationMsg);
}
```

这里的巧妙之处在于：如果你通过 `ChatMemory` 的 `memoryId` 为不同用户保存了不同的系统提示词，框架会优先使用记忆中的版本——这意味着你可以为每个用户定制不同的 AI 人格。

### 3.3 Tool 的自动发现

```java
@Tool("获取指定城市的天气信息")
String getWeather(
    @P("城市名称，如上海、北京") String city);

@Tool("计算两个数的乘积")
double multiply(
    @P("第一个数") double a,
    @P("第二个数") double b);
```

`AiServiceFactory` 会通过反射扫描接口中的 `@Tool` 注解方法，对每个方法：
1. 提取方法名和描述 → 生成 `ToolSpecification.name()` 和 `description()`
2. 提取参数名和 `@P` 描述 → 生成 JSON Schema（参数类型和描述）
3. 包装 `Method` 对象 → 生成 `Tool.execute()` 的实现

这样，当用户问「上海今天天气怎么样」，LLM 会返回一个 `ToolExecutionRequest(name="getWeather", arguments={"city":"上海"})`，代理自动调用 `getWeather("上海")`，拿到结果后再发回给 LLM 生成最终回复。

---

## 四、链式调用 vs Builder 模式：Java 版的 API 设计哲学

Python 的 LangChain 使用 **链式调用（Chain of Responsibility）**：

```python
# Python LangChain 风格
chain = (
    prompt
    | llm
    | output_parser
)
result = chain.invoke({"input": "Hello"})
```

这很 Pythonic，利用了 Python 的 `|` 运算符重载。但 Java 没有运算符重载，LangChain4j 选择了 **Builder 模式 + 流式 API**：

```java
// LangChain4j 的链式风格
AiServices<String> service = AiServices.builder(Assistant.class)
    .chatLanguageModel(model)
    .chatMemory(MessageWindowChatMemory.withMaxMessages(10))
    .tools(new Calculator())
    .build();

// 或者使用 Chain 传统风格（更接近 Python 版）
ConversationalChain chain = ConversationalChain.builder()
    .chatLanguageModel(model)
    .chatMemory(MessageWindowChatMemory.withMaxMessages(20))
    .build();

String answer = chain.execute("你好");
```

Builder 模式的优势：
- **类型安全**：编译期就能发现配置缺失（比如忘记设置 `chatLanguageModel` 会在 `build()` 时抛异常）
- **IDE 友好**：自动补全能列出所有可配置项
- **不可变对象**：Builder 生成的实例是线程安全的，可以在多线程环境中共享

```java
// OpenAiChatModel 的 Builder 模式
OpenAiChatModel model = OpenAiChatModel.builder()
    .apiKey("sk-xxx")
    .modelName("gpt-4o")
    .temperature(0.7)
    .topP(0.9)
    .maxTokens(4096)
    .timeout(Duration.ofSeconds(60))
    .maxRetries(3)
    .logRequests(true)
    .logResponses(true)
    .build();
```

**这种 Builder 模式是 LangChain4j 最突出的 Java 特色**——它让代码既有声明式的美感，又不失 Java 的严谨。

---

## 五、模块化设计：core + 适配器的优雅解耦

LangChain4j 的模块设计非常清晰：

```
langchain4j-core          ← 核心抽象（接口、消息、模型定义）
langchain4j-open-ai       ← OpenAI 适配器
langchain4j-azure-open-ai ← Azure OpenAI 适配器
langchain4j-ollama        ← Ollama 适配器
langchain4j-google-ai-gemini ← Gemini 适配器
langchain4j-vertex-ai     ← Vertex AI 适配器
langchain4j-anthropic     ← Claude 适配器
langchain4j-hugging-face  ← HuggingFace 适配器
langchain4j-qianfan       ← 百度千帆适配器
langchain4j-dashscope     ← 阿里灵积适配器
langchain4j-zhipu-ai      ← 智谱AI适配器
langchain4j-elasticsearch ← ES 向量数据库适配器
langchain4j-pinecone      ← Pinecone 向量数据库适配器
langchain4j-milvus        ← Milvus 向量数据库适配器
langchain4j-chroma        ← Chroma 向量数据库适配器
langchain4j-weaviate      ← Weaviate 向量数据库适配器
langchain4j-redis         ← Redis 向量数据库适配器
langchain4j-qdrant        ← Qdrant 向量数据库适配器
...
```

每个适配器模块都只依赖 `langchain4j-core`，互不依赖。这意味着你引入 `langchain4j-ollama` 时不会被拖入一整套 OpenAI SDK。

**以 OpenAI 适配器为例：**

```java
// OpenAiChatModel 的完整实现结构
public class OpenAiChatModel implements ChatLanguageModel {

    private final String apiKey;
    private final String modelName;
    private final Double temperature;
    private final Double topP;
    private final List<String> stop;
    private final Integer maxTokens;
    private final Duration timeout;
    private final Integer maxRetries;
    private final boolean logRequests;
    private final boolean logResponses;
    private final OpenAiClient client;  // 封装 HTTP 调用

    @Override
    public Response<AiMessage> generate(List<ChatMessage> messages) {
        // 1. 将 ChatMessage 列表转换为 OpenAI 的 JSON 请求体
        OpenAiChatRequest request = OpenAiChatRequest.builder()
            .model(modelName)
            .messages(InternalOpenAiHelper.toOpenAiMessages(messages))
            .temperature(temperature)
            .maxTokens(maxTokens)
            .build();

        // 2. 调用 HTTP API
        OpenAiChatResponse response = client.chatCompletion(request);

        // 3. 将 OpenAI 响应转换为框架标准格式
        return Response.from(
            aiMessageFrom(response),
            tokenUsageFrom(response),
            finishReasonFrom(response)
        );
    }
}
```

每个模型适配器的 `generate` 方法做的事都一样：**翻译 → 调用 → 翻译回来**。这是典型的适配器模式（Adapter Pattern）。

---

## 六、InMemoryEmbeddingStore 的存储结构

RAG 的检索部分依赖向量数据库，而 `InMemoryEmbeddingStore` 是 LangChain4j 内置的内存实现，非常适合原型开发和单元测试。

```java
// 核心数据结构
public class InMemoryEmbeddingStore<Embedded> implements EmbeddingStore<Embedded> {

    private final List<Entry<Embedded>> entries = new CopyOnWriteArrayList<>();

    // 每个 Entry 存储：ID、Embedding 向量、原始 Embedded 对象（通常是 TextSegment）
    private static class Entry<Embedded> {
        private final String id;
        private final float[] vector;    // 向量（浮点数组）
        private final Embedded embedded; // 原始数据
    }

    @Override
    public List<EmbeddingMatch<Embedded>> findRelevant(
        Embedding referenceEmbedding, int maxResults, double minScore) {

        // 余弦相似度计算
        return entries.stream()
            .map(entry -> {
                double score = cosineSimilarity(
                    referenceEmbedding.vector(), entry.vector);
                return new EmbeddingMatch<>(score, entry.id, entry.embedded);
            })
            .filter(match -> match.score() >= minScore)
            .sorted(Comparator.comparingDouble(EmbeddingMatch::score).reversed())
            .limit(maxResults)
            .collect(toList());
    }

    // 余弦相似度 = A·B / (|A| * |B|)
    private static double cosineSimilarity(float[] a, float[] b) {
        double dotProduct = 0;
        double normA = 0;
        double normB = 0;
        for (int i = 0; i < a.length; i++) {
            dotProduct += a[i] * b[i];
            normA += a[i] * a[i];
            normB += b[i] * b[i];
        }
        return dotProduct / (Math.sqrt(normA) * Math.sqrt(normB));
    }
}
```

**注意：** `InMemoryEmbeddingStore` 不适合生产环境——它使用全量遍历做相似度搜索，时间复杂度 O(n)，百万级向量会直接撑爆内存。生产环境建议用 `PgvectorEmbeddingStore` 或 `MilvusEmbeddingStore`，它们底层使用 HNSW 索引，O(log n) 的检索速度。

---

## 七、LangChain4j vs Python LangChain 设计差异总结

| 维度 | Python LangChain | LangChain4j |
|------|-----------------|-------------|
| **API 风格** | 链式调用（`pipe` 运算符） | Builder 模式 + 流式 API |
| **AI 服务定义** | 手动构建 Chain 对象 | 接口 + 注解声明式定义（AiServices） |
| **类型系统** | 动态类型（Duck Typing） | 强类型，编译期检查 |
| **函数调用** | 装饰器定义工具 | `@Tool` 注解 + 反射调用 |
| **模块设计** | 单仓库 + 社区适配器散落在不同包 | 明确的一级模块划分，每个适配器独立 |
| **记忆管理** | 多种 Memory 类，API 不统一 | 统一 `ChatMemory` 接口 + Provider |
| **流式处理** | 基于 `AsyncIterator` | 基于 `TokenStream` + `CompletableFuture` |
| **可观测性** | 需自行集成 | 内置 `logRequests`/`logResponses` 开关 |

**核心差异一句话总结：** Python LangChain 追求「快速实验、灵活组合」，LangChain4j 追求「类型安全、企业级工程化」。

---

## 八、小结与预告

LangChain4j 绝不是 Python LangChain 的简单翻译，而是一次面向 Java 语言的深度重设计。它抓住了 Java 生态的几个核心优势——接口抽象、强类型、Builder 模式、动态代理——构建了一套比 Python 版更适合企业级项目的 AI 开发框架。

**为什么你应该选 LangChain4j 而不是自己调 API？**
1. 你不会想手动实现 RAG 管道（文档解析 → 切割 → 向量化 → 存储 → 检索 → 重排序）
2. 你不会想手动实现 Agent 循环（LLM 输出工具调用 → 执行工具 → 回传结果 → 再问 LLM）
3. 你不会想手动管理多轮对话的 Token 窗口
4. 你不会想为每个模型提供商写一套不同的 HTTP 调用逻辑

LangChain4j 把这些脏活累活都封装好了，你只需要定义一个接口。

---

**下一篇预告：** 我们将转向 **Spring AI**——Spring 官方出品的 AI 框架。它和 LangChain4j 是什么关系？Competing or Complementary？Spring AI 的分层架构、ChatClient 的 Builder 哲学、与 Spring Boot AutoConfiguration 的深度集成，我们一一揭开面纱。
