# AI SDK 设计模式：Builder / Chain / Observer 在 AI 框架中的应用，读懂源码的设计智慧

> 源码是最好的老师。Spring AI 和 LangChain4j 的代码中藏着大量经典设计模式的精妙运用。理解这些模式，你不仅能用好框架，还能写出同样优雅的 AI SDK。

---

## 零、为什么写这篇文章？

很多同学用 Spring AI 或 LangChain4j 时，只是照着文档"依葫芦画瓢"。你要问他 `ChatClient.builder().build()` 背后的设计原理，他就说不清楚了。

**读框架源码，最值钱的是理解其中的设计模式**。

我花了两个月时间，把 Spring AI 和 LangChain4j 的核心源码翻了一遍，提炼出 6 个贯穿始终的设计模式。理解它们，你就真正读懂了 AI 框架的设计智慧。

---

## 一、Builder 模式：构建复杂 AI 配置的优雅之道

### 1.1 没有 Builder 模式的噩梦

假设我们要创建一个 LLM 客户端，有 20+ 配置参数：

```java
// 如果没有 Builder 模式——构造函数地狱
ChatClient client = new ChatClient(
    "gpt-4o",
    "sk-xxx",
    "https://api.openai.com",
    0.7,     // temperature
    1024,    // maxTokens
    0.9,     // topP
    30,      // timeout
    3,       // maxRetries
    true,    // logRequests
    false,   // logResponses
    Map.of("X-Custom", "value"),
    // ... 还有 15 个参数
    null, null, null, null
);
// 哪些 null 是什么意思？完全不可读！
```

### 1.2 Spring AI 中的 Builder 模式

打开 Spring AI 源码，`ChatClient` 的创建链：

```java
// Spring AI 源码：org.springframework.ai.chat.client.ChatClient
public class ChatClient {

    private final ChatModel chatModel;
    private final String defaultSystem;
    private final Double defaultTemperature;
    private final Integer defaultMaxTokens;
    private final List<Advisor> defaultAdvisors;
    // ... 10+ 个配置字段

    // 私有的全参构造函数，只有 Builder 能调用
    private ChatClient(ChatModel chatModel, Builder builder) {
        this.chatModel = chatModel;
        this.defaultSystem = builder.defaultSystem;
        this.defaultTemperature = builder.defaultTemperature;
        this.defaultMaxTokens = builder.defaultMaxTokens;
        this.defaultAdvisors = Collections.unmodifiableList(builder.defaultAdvisors);
        // ...
    }

    // Builder 内部类
    public static class Builder {
        private final ChatModel chatModel;
        private String defaultSystem;
        private Double defaultTemperature;
        private Integer defaultMaxTokens;
        private final List<Advisor> defaultAdvisors = new ArrayList<>();

        // Builder 构造函数只接受必填参数
        public Builder(ChatModel chatModel) {
            this.chatModel = Objects.requireNonNull(chatModel);
        }

        // 每个配置项都有独立的设置方法，返回 this 实现链式调用
        public Builder defaultSystem(String system) {
            this.defaultSystem = system;
            return this;
        }

        public Builder defaultTemperature(Double temperature) {
            this.defaultTemperature = temperature;
            return this;
        }

        public Builder defaultMaxTokens(Integer maxTokens) {
            this.defaultMaxTokens = maxTokens;
            return this;
        }

        public Builder defaultAdvisors(Advisor... advisors) {
            this.defaultAdvisors.addAll(Arrays.asList(advisors));
            return this;
        }

        // build() 做最终校验
        public ChatClient build() {
            validate();
            return new ChatClient(chatModel, this);
        }

        private void validate() {
            if (defaultTemperature != null && 
                (defaultTemperature < 0 || defaultTemperature > 2)) {
                throw new IllegalArgumentException(
                    "Temperature must be between 0 and 2");
            }
        }
    }
}
```

**设计亮点**：
1. **必填参数在 Builder 构造函数中**：`ChatModel` 是必填的，放在 Builder 构造函数里，编译期强制传入。
2. **可选的链式设置**：每个可选参数都有独立 setter，返回 `this`，实现流畅的链式调用。
3. **延迟校验**：所有校验集中在 `build()` 中，避免"半成品"对象。
4. **不可变对象**：`ChatClient` 的字段都是 `final` 的，构建后不可修改，线程安全。

### 1.3 LangChain4j 中的 Builder 模式

LangChain4j 的 Builder 更进一步——使用了 **Step Builder** 模式，引导你按顺序配置：

```java
// LangChain4j 源码简化版：dev.langchain4j.model.openai.OpenAiChatModel
public class OpenAiChatModel implements ChatLanguageModel {

    // Step Builder：强制按步骤构建
    public static ApiKeyStep builder() {
        return new Builder();
    }

    // 第 1 步：必须先设置 API Key
    public interface ApiKeyStep {
        ModelNameStep apiKey(String apiKey);
    }

    // 第 2 步：必须设置模型名
    public interface ModelNameStep {
        FinalStep modelName(String modelName);
    }

    // 第 3 步：可选配置，最后 build
    public interface FinalStep {
        FinalStep temperature(Double temperature);
        FinalStep maxTokens(Integer maxTokens);
        FinalStep timeout(Duration timeout);
        FinalStep maxRetries(Integer maxRetries);
        FinalStep logRequests(Boolean logRequests);
        // ... 更多可选配置

        OpenAiChatModel build();
    }

    // 内部 Builder 实现所有步骤接口
    private static class Builder implements ApiKeyStep, ModelNameStep, FinalStep {
        private String apiKey;
        private String modelName;
        private Double temperature;
        // ...

        @Override
        public ModelNameStep apiKey(String apiKey) {
            this.apiKey = apiKey;
            return this;
        }

        @Override
        public FinalStep modelName(String modelName) {
            this.modelName = modelName;
            return this;
        }

        @Override
        public FinalStep temperature(Double temperature) {
            this.temperature = temperature;
            return this;
        }

        // ...
    }
}

// 使用：编译期强制顺序
OpenAiChatModel model = OpenAiChatModel.builder()
    .apiKey(System.getenv("OPENAI_API_KEY"))  // 第 1 步，必须
    .modelName("gpt-4o")                      // 第 2 步，必须
    .temperature(0.7)                         // 第 3 步，可选
    .maxTokens(4096)
    .build();
```

**Step Builder 的核心价值**：如果你忘了设置 `apiKey`，代码根本编译不过！这是普通 Builder 做不到的。

### 1.4 Builder 模式总结

| 变体 | 使用场景 | 代表 |
|---|---|---|
| 普通 Builder | 配置项多但无强制顺序 | Spring AI ChatClient |
| Step Builder | 有明确的设置步骤约束 | LangChain4j OpenAiChatModel |
| Fluent Builder | 极致的链式调用体验 | Lombok `@Builder` |

---

## 二、Chain（责任链）模式：请求处理管道

### 2.1 AI 请求需要经过多道"工序"

一个典型的 AI 请求处理流程：

```
用户输入
  → 安全检查（过滤敏感词）
  → Prompt 增强（添加上下文）
  → 调用 LLM
  → 响应过滤（去敏感内容）
  → 日志记录
  → 返回给用户
```

这恰好是**责任链模式**的经典场景。

### 2.2 Spring AI 的 Advisor 链

Spring AI 中没有显式的 "Chain" 类，但 `Advisor` 机制本质上就是责任链：

```java
// Spring AI 源码简化版：
// org.springframework.ai.chat.client.advisor.Advisor
public interface Advisor {
    // 围绕下一个 Advisor 的调用做增强
    AdvisedResponse advise(AdvisedRequest request, Chain chain);

    // 内部接口：描述链中的"下一个"
    interface Chain {
        AdvisedResponse next(AdvisedRequest request);
    }
}

// 默认的链式实现
public class DefaultAdvisorChain implements Advisor.Chain {

    private final Iterator<Advisor> advisors;
    private final Advisor.Chain terminal; // 链的终点——真正调用 LLM

    public DefaultAdvisorChain(List<Advisor> advisors, Advisor.Chain terminal) {
        this.advisors = advisors.iterator();
        this.terminal = terminal;
    }

    @Override
    public AdvisedResponse next(AdvisedRequest request) {
        if (advisors.hasNext()) {
            Advisor nextAdvisor = advisors.next();
            // 递归调用：当前 Advisor 处理完后，调用 chain.next()
            return nextAdvisor.advise(request, this);
        }
        // 链的尽头：真正执行 LLM 调用
        return terminal.next(request);
    }
}
```

**示例：一个日志 Advisor**

```java
// 自定义 Advisor 实现
public class LoggingAdvisor implements Advisor {

    @Override
    public AdvisedResponse advise(AdvisedRequest request, Chain chain) {
        long start = System.currentTimeMillis();

        // 前置处理
        System.out.println("[请求] " + request.userText());

        // 调用链中的下一个
        AdvisedResponse response = chain.next(request);

        // 后置处理
        long elapsed = System.currentTimeMillis() - start;
        System.out.println("[响应] 耗时: " + elapsed + "ms, "
            + "Token: " + response.metadata().get("usage"));

        return response;
    }
}

// 使用：像洋葱一样层层包裹
ChatClient client = ChatClient.builder(model)
    .defaultAdvisors(
        new SecurityAdvisor(),      // 最外层：安全检查
        new ContextEnhanceAdvisor(), // 第二层：增强上下文
        new LoggingAdvisor(),       // 第三层：日志
        new ResponseFilterAdvisor() // 第四层：响应过滤
    )
    .build();
```

调用链路：
```
SecurityAdvisor → ContextEnhanceAdvisor → LoggingAdvisor → ResponseFilterAdvisor → LLM
                                                                                          ↓
SecurityAdvisor ← ContextEnhanceAdvisor ← LoggingAdvisor ← ResponseFilterAdvisor ← (响应)
```

### 2.3 LangChain4j 的 Chain 实现

LangChain4j 的 Chain 更丰富，有 `SequentialChain`（顺序链）、`RouterChain`（路由链）等：

```java
// LangChain4j 源码简化版
public interface Chain<Input, Output> {
    Output execute(Input input);
}

public class SequentialChain implements Chain<String, AiMessage> {

    private final List<ChatLanguageModel> models;

    @Override
    public AiMessage execute(String input) {
        String current = input;
        for (ChatLanguageModel model : models) {
            // 上一个模型的输出，作为下一个模型的输入
            current = model.generate(current).content().text();
        }
        return new AiMessage(current);
    }
}

// 使用：多个模型串联处理
Chain<String, AiMessage> analysisChain = new SequentialChain(
    extractModel,   // 第 1 步：提取关键信息
    analyzeModel,   // 第 2 步：分析
    summarizeModel  // 第 3 步：总结
);

AiMessage result = analysisChain.execute("大段原始文本...");
```

---

## 三、Observer（观察者）模式：流式输出的事件驱动

### 3.1 流式输出的本质

当 LLM 以 Streaming 模式返回时，模型会**逐 token** 推送数据：

```
"我" → "是一" → "一个" → "AI" → "助手" → ...
```

这天然适合 Observer 模式：**模型是发布者，你的回调是订阅者**。

### 3.2 LangChain4j 的 StreamingResponseHandler

```java
// LangChain4j 源码
// dev.langchain4j.model.StreamingResponseHandler
public interface StreamingResponseHandler<T> {

    /**
     * 当收到一个新的 token 时调用
     */
    void onNext(String token);

    /**
     * 当 LLM 发生错误时调用
     */
    void onError(Throwable error);

    /**
     * 当整个响应完成时调用
     */
    default void onComplete(Response<T> response) {
        // 默认空实现
    }
}

// LangChain4j 内部的实现机制（简化）
public class DefaultStreamingChatModel implements StreamingChatLanguageModel {

    private final List<StreamingResponseHandler<ChatResponse>> handlers = new ArrayList<>();

    public void registerHandler(StreamingResponseHandler<ChatResponse> handler) {
        handlers.add(handler);
    }

    @Override
    public void generate(String userMessage, StreamingResponseHandler<ChatResponse> handler) {
        // 发起 SSE 连接
        EventSource eventSource = connectToSSE(userMessage);

        eventSource.onEvent(event -> {
            // 每收到一个 SSE 事件，解析为 token
            String token = parseToken(event.data());

            // 通知所有观察者
            handler.onNext(token);
            for (var h : handlers) {
                h.onNext(token); // 额外的监听器
            }
        });

        eventSource.onComplete(() -> {
            ChatResponse response = buildFinalResponse();
            handler.onComplete(response);
        });

        eventSource.onError(error -> {
            handler.onError(error);
        });
    }
}
```

**使用示例**：

```java
// 实时打印 LLM 的输出
StreamingChatLanguageModel model = OpenAiStreamingChatModel.builder()
        .apiKey(System.getenv("OPENAI_API_KEY"))
        .modelName("gpt-4o")
        .build();

model.generate("请写一首关于春天的诗",
    new StreamingResponseHandler<>() {
        @Override
        public void onNext(String token) {
            System.out.print(token); // 逐字打印，类似打字机效果
        }

        @Override
        public void onComplete(ChatResponse response) {
            System.out.println("\n\n[生成完成，共 "
                + response.tokenUsage().totalTokenCount() + " tokens]");
        }

        @Override
        public void onError(Throwable error) {
            System.err.println("出错了: " + error.getMessage());
        }
    });
```

### 3.3 Spring AI 的 Reactive Streaming

Spring AI 使用了更现代的 **Reactive Streams**（响应式流），本质是 Observer 模式的演进：

```java
// Spring AI 源码：基于 Project Reactor
public interface StreamingChatClient {
    Flux<ChatResponse> stream(Prompt prompt);
}

// 内部实现：将 HTTP SSE 流转换为 Reactor Flux
public class OpenAiStreamingChatModel implements StreamingChatClient {

    @Override
    public Flux<ChatResponse> stream(Prompt prompt) {
        return webClient.post()
                .bodyValue(buildRequest(prompt))
                .accept(MediaType.TEXT_EVENT_STREAM)
                .retrieve()
                .bodyToFlux(String.class)
                .filter(line -> line.startsWith("data: "))
                .map(line -> line.substring(6))
                .filter(data -> !"[DONE]".equals(data))
                .map(this::parseChunk)       // 解析 JSON
                .filter(chunk -> chunk.hasContent())
                .publishOn(Schedulers.boundedElastic());
    }
}

// 使用：完全响应式
chatClient.prompt()
        .user("请介绍 Java 21 的新特性")
        .stream()
        .chatResponse()
        .doOnNext(resp -> System.out.print(resp.getContent()))
        .doOnComplete(() -> System.out.println("\n完成"))
        .doOnError(e -> log.error("流式输出异常", e))
        .subscribe();
```

**对比**：

| 维度 | LangChain4j (回调式) | Spring AI (响应式) |
|---|---|---|
| 编程模型 | Observer (Push) | Reactive Streams (Push + Backpressure) |
| 线程模型 | 被动接收 | 支持背压控制 |
| 学习成本 | 低 | 中（需理解 Reactor） |
| 适用场景 | 简单流式输出 | 复杂异步编排 + 流控 |

---

## 四、Strategy（策略）模式：多模型无缝切换

### 4.1 同样的接口，不同的实现

Strategy 模式是 AI 框架中最基础也最重要的模式：

```
        ┌───────────────┐
        │  ChatModel    │  ← 统一接口
        │  (Interface)  │
        └───────┬───────┘
       ┌────────┼────────┐
       │        │        │
   ┌───┴───┐ ┌──┴───┐ ┌─┴──────┐
   │OpenAI │ │ 智谱  │ │ Ollama │  ← 不同厂商实现
   └───────┘ └──────┘ └────────┘
```

### 4.2 Spring AI 的模型抽象

```java
// Spring AI 源码核心接口
// org.springframework.ai.chat.model.ChatModel
public interface ChatModel extends Model<Prompt, ChatResponse> {

    @Override
    ChatResponse call(Prompt prompt);

    default Flux<ChatResponse> stream(Prompt prompt) {
        throw new UnsupportedOperationException("Streaming not supported");
    }
}

// 不同实现类
public class OpenAiChatModel implements ChatModel { /* 调用 OpenAI API */ }
public class QianFanChatModel implements ChatModel { /* 调用百度千帆 API */ }
public class OllamaChatModel implements ChatModel { /* 调用本地 Ollama */ }
```

### 4.3 LangChain4j 的模型切换

LangChain4j 的设计相同，但接口命名不同：

```java
public interface ChatLanguageModel {
    ChatResponse generate(ChatMessage... messages);
}

// 运行时动态切换
public class ModelSwitcher implements ChatLanguageModel {

    private final Map<String, ChatLanguageModel> models;

    public ModelSwitcher(Map<String, ChatLanguageModel> models) {
        this.models = models;
    }

    public void switchTo(String modelName) {
        this.currentModel.set(models.get(modelName));
    }

    private final AtomicReference<ChatLanguageModel> currentModel = new AtomicReference<>();

    @Override
    public ChatResponse generate(ChatMessage... messages) {
        return currentModel.get().generate(messages);
    }
}
```

### 4.4 更高级的玩法：负载均衡 + 故障转移

将 Strategy 模式与负载均衡结合：

```java
public class LoadBalancedChatModel implements ChatModel {

    private final List<ChatModel> delegates;
    private final AtomicInteger counter = new AtomicInteger(0);

    public LoadBalancedChatModel(List<ChatModel> delegates) {
        this.delegates = delegates;
    }

    @Override
    public ChatResponse call(Prompt prompt) {
        // 轮询策略
        int index = counter.getAndIncrement() % delegates.size();
        ChatModel selected = delegates.get(index);

        try {
            return selected.call(prompt);
        } catch (Exception e) {
            // 故障转移：尝试下一个
            log.warn("Model {} failed, trying next", index, e);
            return delegates.get((index + 1) % delegates.size()).call(prompt);
        }
    }
}
```

---

## 五、Adapter（适配器）模式：统一不同厂商的 API 差异

### 5.1 每个厂商的 API 都不一样

```java
// OpenAI 的请求格式
{
  "model": "gpt-4o",
  "messages": [{"role": "user", "content": "Hello"}],
  "temperature": 0.7
}

// 百度千帆的请求格式（完全不同！）
{
  "messages": [{"role": "user", "content": "Hello"}],
  "temperature": 0.7,
  "system": "You are helpful", // System prompt 的字段名不一样
  "penalty_score": 1.0         // OpenAI 没有的参数
}

// 阿里通义千问的请求格式（又不一样！）
{
  "model": "qwen-max",
  "input": {                    // 用 "input" 而不是 "messages"！
    "messages": [{"role": "user", "content": "Hello"}]
  },
  "parameters": {               // 参数嵌套在 "parameters" 里
    "temperature": 0.7
  }
}
```

Adapter 模式的作用：将这些**千奇百怪的 API 统一为同一个内部接口**。

### 5.2 Spring AI 中的 Adapter 实现

```java
// Spring AI 源码简化
// 统一的内部请求对象
public class ChatModelRequest {
    private final String model;
    private final List<Message> messages;
    private final ChatOptions options;
}

// 适配器接口：将内部请求转换为厂商特定的 HTTP 请求
interface ApiAdapter<T_Request, T_Response> {

    // 将统一请求转为厂商特定请求体
    T_Request createRequest(ChatModelRequest request);

    // 将厂商响应转为统一响应
    ChatResponse toChatResponse(T_Response response);

    // 将厂商流式 chunk 转为统一 chunk
    ChatChunk toChatChunk(String sseData);
}

// OpenAI 适配器
public class OpenAiApiAdapter implements ApiAdapter<OpenAiRequest, OpenAiResponse> {

    @Override
    public OpenAiRequest createRequest(ChatModelRequest request) {
        return new OpenAiRequest(
            request.model(),
            request.messages().stream()
                .map(m -> new OpenAiMessage(m.role(), m.content()))
                .toList(),
            request.options().temperature(),
            request.options().maxTokens()
        );
    }

    @Override
    public ChatResponse toChatResponse(OpenAiResponse response) {
        return new ChatResponse(
            response.choices().stream()
                .map(c -> new Generation(c.message().content()))
                .toList(),
            new ChatResponseMetadata(response.usage())
        );
    }
}

// 千帆适配器（处理不同的 API 格式）
public class QianFanApiAdapter implements ApiAdapter<QianFanRequest, QianFanResponse> {

    @Override
    public QianFanRequest createRequest(ChatModelRequest request) {
        // 北向：Spring AI → 千帆
        var builder = QianFanRequest.builder()
            .messages(convertMessages(request.messages()))
            .temperature(request.options().temperature());

        // 千帆的 System Prompt 是单独的字段
        request.messages().stream()
            .filter(m -> m.role() == MessageType.SYSTEM)
            .findFirst()
            .ifPresent(m -> builder.system(m.content()));

        return builder.build();
    }

    @Override
    public ChatResponse toChatResponse(QianFanResponse response) {
        // 南向：千帆 → Spring AI
        return new ChatResponse(
            List.of(new Generation(response.getResult())),
            new ChatResponseMetadata(convertUsage(response.getUsage()))
        );
    }
}
```

**设计要点**：
1. 每种厂商一个 Adapter 实现类，**新增厂商只需新增一个 Adapter**。
2. Adapter 是**双向转换**：请求时 `createRequest()`（北向），响应时 `toChatResponse()`（南向）。
3. 上层业务代码**完全感知不到**底层厂商差异。

---

## 六、Dynamic Proxy（动态代理）模式：AiServices 的魔法

### 6.1 LangChain4j 的 AiServices：接口即实现

LangChain4j 最让人惊艳的设计是 **AiServices**：你只需要定义一个接口，LangChain4j 自动生成实现。

```java
// 你只需要定义一个接口
interface Assistant {

    @SystemMessage("You are a helpful assistant. Today is {{current_date}}.")
    String chat(@UserMessage @V("message") String message);

    @SystemMessage("Translate the following text to {{target_language}}.")
    String translate(
        @UserMessage @V("text") String text,
        @V("target_language") String language);
}

// 无需实现类！LangChain4j 自动生成
Assistant assistant = AiServices.create(Assistant.class, chatModel);

// 直接调用
String answer = assistant.chat("What is Java 21?");
String translation = assistant.translate("你好世界", "English");
```

**背后的魔法就是 Java 动态代理**。

### 6.2 AiServices 源码剖析

```java
// LangChain4j 源码核心逻辑（简化版）
public class AiServices<T> {

    public static <T> T create(Class<T> interfaceClass, ChatLanguageModel model) {
        // 创建动态代理
        return (T) Proxy.newProxyInstance(
            interfaceClass.getClassLoader(),
            new Class[]{interfaceClass},
            new AiServiceInvocationHandler(model, interfaceClass)
        );
    }

    // 动态代理的 InvocationHandler
    private static class AiServiceInvocationHandler implements InvocationHandler {

        private final ChatLanguageModel model;
        private final Map<Method, MethodMetadata> methodCache = new ConcurrentHashMap<>();

        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            // 1. 解析方法元数据（SystemMessage、UserMessage 等注解）
            MethodMetadata metadata = methodCache.computeIfAbsent(method, this::parseMethod);

            // 2. 构建 System Prompt
            String systemMessage = buildSystemMessage(metadata, args);

            // 3. 构建 User Message（从 @UserMessage 注解和参数中获取）
            String userMessage = buildUserMessage(metadata, args);

            // 4. 组装完整的 Prompt
            List<ChatMessage> messages = new ArrayList<>();
            if (systemMessage != null) {
                messages.add(new SystemMessage(systemMessage));
            }
            messages.add(new UserMessage(userMessage));

            // 5. 调用 LLM
            ChatResponse response = model.generate(
                messages.toArray(new ChatMessage[0]));

            // 6. 提取并返回文本内容
            return response.content().text();
        }

        private MethodMetadata parseMethod(Method method) {
            SystemMessage sysAnno = method.getAnnotation(SystemMessage.class);
            UserMessage userAnno = method.getAnnotation(UserMessage.class);

            // 解析 @V("current_date") 等模板变量
            List<TemplateVariable> variables = extractTemplateVariables(
                sysAnno != null ? sysAnno.value() : "",
                userAnno != null ? userAnno.value() : ""
            );

            return new MethodMetadata(sysAnno, userAnno, variables);
        }

        private String buildSystemMessage(MethodMetadata meta, Object[] args) {
            if (meta.systemAnnotation == null) return null;

            // 获取方法参数名和值的映射
            Map<String, Object> paramMap = buildParamMap(meta.method, args);

            // 用 Caffeine 缓存编译好的模板
            PromptTemplate template = templateCache.get(meta.method, m ->
                PromptTemplate.from(meta.systemAnnotation.value())
            );

            // 替换模板变量 {{variable}} → 实际值
            return template.apply(paramMap).text();
        }
    }
}
```

### 6.3 动态代理的三个关键步骤

#### Step 1：`Proxy.newProxyInstance()` 创建代理对象

```java
// JDK 动态代理要求被代理的是接口
Assistant proxy = (Assistant) Proxy.newProxyInstance(
    Assistant.class.getClassLoader(),   // ClassLoader
    new Class[]{Assistant.class},       // 接口数组
    new AiServiceInvocationHandler(...) // 调用处理器
);
```

#### Step 2：`InvocationHandler.invoke()` 拦截所有方法调用

```java
// 当你调用 assistant.chat("Hello") 时
// JVM 不会执行"真实"的方法
// 而是调用 InvocationHandler.invoke(proxy, method, args)
// method = Assistant.class.getMethod("chat", String.class)
// args   = ["Hello"]
```

#### Step 3：注解解析 + 模板渲染 + LLM 调用

```java
invoke(proxy, method, args) {
    // 1. 反射读取 @SystemMessage 注解
    String template = getSystemMessageTemplate(method);

    // 2. 用参数值替换模板变量
    String systemMsg = renderTemplate(template, method, args);

    // 3. 构建 ChatMessage 并调用 LLM
    ChatResponse response = model.generate(
        SystemMessage.from(systemMsg),
        UserMessage.from(userMessage)
    );

    // 4. 返回结果
    return response.content().text();
}
```

### 6.4 Spring AI 的 `@Tool` 注解也是动态代理

Spring AI 没有 LangChain4j 那样的通用 AI Service 代理，但在 Function Calling 中用了类似机制：

```java
// Spring AI 内部将标注了 @Description 的 @Bean 转为 Tool
@Bean
@Description("Get the weather for a given city")
public Function<WeatherRequest, WeatherResponse> weatherFunction() {
    return request -> new WeatherResponse("Sunny", 25.0);
}

// 内部通过反射解析 @Description，生成 OpenAI 的 functions 定义
// 然后在 Chat Completion 请求中带上 functions 参数
```

### 6.5 动态代理的优势与局限

**优势**：
- 零样板代码：只需接口 + 注解，没有实现类
- 类型安全：编译期检查参数类型和返回值
- 声明式编程：关注"做什么"而非"怎么做"

**局限**：
- 只能代理接口（JDK 动态代理的限制）
- 调试困难：断点打不到"实现"上
- 性能略低于直接调用（实际影响微乎其微）

---

## 七、设计模式全景图

```
┌───────────────────────────────────────────────────┐
│              AI 框架中的设计模式全景图               │
├───────────────────────────────────────────────────┤
│                                                     │
│  Builder          ──→  复杂对象创建                  │
│  (ChatClient.builder())                             │
│                                                     │
│  Chain of Resp    ──→  请求处理管道                  │
│  (Advisor 链)                                       │
│                                                     │
│  Observer         ──→  流式输出监听                  │
│  (StreamingResponseHandler)                         │
│                                                     │
│  Strategy         ──→  多模型切换                    │
│  (ChatModel 多实现)                                  │
│                                                     │
│  Adapter          ──→  统一不同厂商 API              │
│  (ApiAdapter)                                       │
│                                                     │
│  Dynamic Proxy    ──→  接口自动生成实现               │
│  (AiServices)                                       │
│                                                     │
│  Factory Method   ──→  模型工厂创建                  │
│  (ChatModelFactory)                                  │
│                                                     │
│  Singleton        ──→  Bean 单例管理                 │
│  (Spring IoC)                                       │
│                                                     │
│  Template Method  ──→  通用调用流程                   │
│  (AbstractChatModel)                                 │
│                                                     │
└───────────────────────────────────────────────────┘
```

---

## 八、如果你来设计一个 AI SDK...

综合以上模式，一个简洁的 AI SDK 骨架如下：

```java
// 1. 统一接口（Strategy 模式）
public interface LlmClient {
    ChatResponse chat(ChatRequest request);
    Flux<ChatResponse> chatStream(ChatRequest request);
}

// 2. 用 Builder 创建（Builder 模式）
LlmClient client = LlmClient.builder()
    .provider("openai")      // 或 "qwen", "zhipu", "ollama"
    .apiKey("sk-xxx")
    .model("gpt-4o")
    .temperature(0.7)
    .adapter(new OpenAiAdapter()) // Adapter 模式
    .build();

// 3. 动态代理生成 AI 服务（Dynamic Proxy）
@AiService
interface MyAssistant {
    @SystemPrompt("You are a Java expert.")
    @Temperature(0.3)
    String answer(@UserMessage String question);
}

MyAssistant assistant = LlmClient.createService(MyAssistant.class);

// 4. 责任链处理请求（Chain 模式）
client.addInterceptor(new LoggingInterceptor());
client.addInterceptor(new RateLimitInterceptor());
client.addInterceptor(new CachingInterceptor());

// 5. 流式输出（Observer 模式）
client.chatStream(request)
    .doOnNext(token -> emitter.send(token))
    .doOnComplete(() -> emitter.complete())
    .subscribe();
```

---

## 九、总结

Spring AI 和 LangChain4j 的源码之所以优雅，不是因为用了什么高深的技术，而是**恰到好处地运用了经典设计模式**：

| 模式 | 解决的问题 | 关键源码位置 |
|---|---|---|
| Builder | 复杂配置对象的创建 | `ChatClient.Builder` |
| Chain | 请求的多步骤处理 | `Advisor` + `DefaultAdvisorChain` |
| Observer | 流式输出的事件通知 | `StreamingResponseHandler` |
| Strategy | 多模型实现的可替换性 | `ChatModel` 接口 |
| Adapter | 不同厂商 API 的适配 | `ApiAdapter` 接口族 |
| Dynamic Proxy | 接口驱动的声明式 AI 服务 | `AiServices.create()` |

**建议**：下次读任何开源框架源码时，先不要急着看实现细节，而是先识别它用了哪些设计模式。你会发现，顶级框架的代码，本质上就是一本设计模式的最佳实践手册。

---

**下篇预告**：《Java AI 项目实战：从 0 到 1 构建企业级智能客服系统（架构 + 代码全揭秘）》—— 完整呈现一个日处理 10 万对话的智能客服系统从设计到上线的全过程。下一篇见！

---

> 作者：热爱阅读源码的 Java AI 工程师  
> 本文所有源码分析均基于 Spring AI 1.0.0-M2 和 LangChain4j 0.36.0 版本。  
> 框架在不断演进，具体实现可能有所不同，设计思想值得学习。
