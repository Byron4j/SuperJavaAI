> **系列专栏**：[Java + AI 工程实践框架](#)  
> **本文收录**：系列四·Java+AI 工程实践框架  
> **阅读时间**：约 25 分钟

---

## 一、痛点：你可能正在被厂商“绑架”

小李是某创业公司的后端负责人，年初他们用了 OpenAI 的 GPT-4 做智能客服。代码里到处散落着 `OpenAiApi`、`ChatCompletionRequest`，甚至连 Prompt 模板都和 OpenAI 的 Message 结构强耦合。

结果呢？公司预算收紧，老板说：“换个便宜的模型吧，听说 DeepSeek 不错。”

小李打开代码仓库一看，整个人都不好了——**23 个 Service 类里都有直接引用 OpenAI SDK 的代码**。这不是改一处的问题，是要动全身。他估算了一下，至少需要两周。

这不是小李一个人的问题。**在 AI 应用爆发的今天，厂商 API 的差异性正在成为企业的技术债**：

| 厂商 | 认证方式 | 消息结构 | 请求/响应格式 | Streaming 方式 |
|------|----------|----------|--------------|----------------|
| OpenAI | `Authorization: Bearer sk-xxx` | `{"role":"user","content":"hello"}` | JSON | SSE（`data: [DONE]`） |
| Anthropic | `x-api-key: sk-ant-xxx` | 同上但有差异 | JSON 差异大 | SSE（`event: message_stop`） |
| DeepSeek | 兼容 OpenAI | 兼容 OpenAI | 兼容 OpenAI | 兼容 OpenAI |
| 通义千问 | `Authorization: Bearer sk-xxx` | `{"role":"user","content":[{text...}]}` | 有差异 | SSE |
| 智谱 GLM | JWT Token | 基本兼容 OpenAI | 基本兼容 | SSE |
| Ollama | 无认证 | `{"role":"user","content":"hello"}` | JSON | 基础 |
| Google Gemini | API Key（URL 参数） | 自定义结构 | 差异较大 | SSE |

看起来 OpenAI 格式是事实标准，但每个厂商总在细节上“插一脚”。**当你的代码被厂商 API 渗透到各个角落，换模型就像换心脏**。

那有没有办法，让切换模型**只改一行配置**呢？

---

## 二、答案：一个设计良好的多模型统一接入层

我们需要一个**中间层**，它向上提供统一的 Java 接口，向下适配各个厂商的差异化 API。核心思想是经典的 **Adapter 模式 + Strategy 模式**：

```
┌──────────────────────────────────────────────────────────┐
│                    业务层 (Service)                       │
│              只依赖 UnifiedLlmClient                      │
├──────────────────────────────────────────────────────────┤
│                  UnifiedLlmClient（门面）                  │
│         ┌────────────────┬───────────────┐               │
│         ▼                ▼               ▼               │
│   ChatStrategy     EmbeddingStrategy  RerankStrategy     │
│         │                │               │               │
│   ┌─────┼─────┐     ┌────┼────┐     ┌───┼───┐           │
│   ▼     ▼     ▼     ▼    ▼    ▼     ▼   ▼   ▼           │
│ OpenAI DeepSeek  ...                                      │
│ Adapter Adapter Adapter                                  │
└──────────────────────────────────────────────────────────┘
```

- **Strategy** 决定“用什么策略处理请求”（调用、流式、重试等）
- **Adapter** 负责“把一个厂商的 API 转成统一格式”

业务层只看到 `UnifiedLlmClient`，底下是谁完全无感。

---

## 三、核心抽象：统一模型和统一接口

### 3.1 统一的消息结构

不管哪个厂商，对话的本质都是 `role + content`。我们定义统一的消息对象：

```java
// 消息角色枚举
public enum MessageRole {
    SYSTEM, USER, ASSISTANT, TOOL
}

// 统一消息
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class UnifiedMessage {
    private MessageRole role;
    private String content;
    private String name;           // 工具名（Function Calling 时用）
    private String toolCallId;     // 工具调用 ID
    private List<ToolCall> toolCalls; // 模型发起的工具调用列表
}

// 工具调用
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ToolCall {
    private String id;
    private String type;          // "function"
    private String functionName;
    private String arguments;     // JSON 字符串
}
```

### 3.2 统一的请求体

```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class UnifiedChatRequest {
    private String model;                          // 模型名称（用户看到的别名）
    private List<UnifiedMessage> messages;         // 对话历史
    private Double temperature;                   // 温度
    private Integer maxTokens;                     // 最大 Token 数
    private Double topP;                           // Top-P 采样
    private List<String> stop;                     // 停止词
    private boolean stream;                        // 是否流式
    private List<UnifiedToolDefinition> tools;     // 工具定义
    private String user;                           // 用户标识（用于计费/审计）
    
    // 厂商特有的扩展参数，Adapter 自己认得就行
    private Map<String, Object> providerOptions;
}
```

### 3.3 统一的工具定义

```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class UnifiedToolDefinition {
    private String name;
    private String description;
    private Map<String, Object> parameters; // JSON Schema
}
```

### 3.4 统一的响应体

```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class UnifiedChatResponse {
    private String id;                           // 请求 ID
    private String model;                        // 实际模型
    private UnifiedMessage message;              // 响应消息
    private Usage usage;                         // Token 用量
    
    @Data
    @Builder
    @NoArgsConstructor
    @AllArgsConstructor
    public static class Usage {
        private int promptTokens;
        private int completionTokens;
        private int totalTokens;
    }
}
```

### 3.5 核心接口：LlmProviderAdapter

这是整个架构的灵魂：

```java
/**
 * LLM 厂商适配器接口（Adapter 模式的核心抽象）。
 * 每个厂商实现此接口，封装自己的 API 细节。
 */
public interface LlmProviderAdapter {

    /** 厂商标识，如 "openai"、"deepseek"、"qwen"、"zhipu"、"ollama" */
    String providerName();

    /** 该厂商支持的模型列表 */
    Set<String> supportedModels();

    /** 是否支持流式调用 */
    boolean supportsStreaming();

    /** 同步聊天补全 */
    UnifiedChatResponse chatCompletion(UnifiedChatRequest request);

    /** 流式聊天补全 */
    Flux<UnifiedChatResponse> chatCompletionStream(UnifiedChatRequest request);

    /** 将统一模型名映射为厂商实际模型名（如 "gpt-4o" -> "gpt-4o-2024-08-06"） */
    default String mapModelName(String unifiedModelName) {
        return unifiedModelName;
    }
}
```

OK，抽象层定义完毕。下面我们来实现几个具体的 Adapter。

---

## 四、实现各厂商 Adapter

### 4.1 OpenAI Adapter

OpenAI 是“标准制定者”，我们就用它作为参考实现：

```java
@Slf4j
@Component
public class OpenAiAdapter implements LlmProviderAdapter {

    private final OpenAiApi openAiApi;
    private final ObjectMapper objectMapper;

    public OpenAiAdapter(@Value("${llm.providers.openai.api-key}") String apiKey,
                         @Value("${llm.providers.openai.base-url:https://api.openai.com}") String baseUrl,
                         ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
        RestClient.Builder builder = RestClient.builder();
        this.openAiApi = OpenAiApi.builder()
                .apiKey(apiKey)
                .restClientBuilder(builder)
                .completionsPath(baseUrl + "/v1/chat/completions")
                .build();
    }

    @Override
    public String providerName() {
        return "openai";
    }

    @Override
    public Set<String> supportedModels() {
        return Set.of("gpt-4o", "gpt-4o-mini", "gpt-4-turbo", "gpt-3.5-turbo",
                "o1", "o1-mini", "o3-mini");
    }

    @Override
    public boolean supportsStreaming() {
        return true;
    }

    @Override
    public UnifiedChatResponse chatCompletion(UnifiedChatRequest request) {
        ChatCompletionResponse response = openAiApi.call(toSpringAiRequest(request));
        return toUnifiedResponse(response, request.getModel());
    }

    @Override
    public Flux<UnifiedChatResponse> chatCompletionStream(UnifiedChatRequest request) {
        return openAiApi.stream(toSpringAiRequest(request))
                .filter(chunk -> chunk.getChoices() != null
                        && !chunk.getChoices().isEmpty()
                        && chunk.getChoices().get(0).getMessage() != null)
                .map(chunk -> toUnifiedResponseFromChunk(chunk, request.getModel()));
    }

    // ===== 转换方法 =====
    private ChatCompletionRequest toSpringAiRequest(UnifiedChatRequest req) {
        List<ChatCompletionMessage> messages = req.getMessages().stream()
                .map(this::convertMessage)
                .toList();

        ChatCompletionRequest.Builder builder = ChatCompletionRequest.builder()
                .withModel(this.mapModelName(req.getModel()))
                .withMessages(messages);

        if (req.getTemperature() != null) builder.withTemperature(req.getTemperature());
        if (req.getMaxTokens() != null) builder.withMaxTokens(req.getMaxTokens());
        if (req.getTopP() != null) builder.withTopP(req.getTopP());
        if (req.getStop() != null) builder.withStop(req.getStop());

        // 工具
        if (req.getTools() != null && !req.getTools().isEmpty()) {
            builder.withTools(toSpringAiTools(req.getTools()));
        }

        return builder.build();
    }

    private ChatCompletionMessage convertMessage(UnifiedMessage msg) {
        return switch (msg.getRole()) {
            case SYSTEM -> new SystemMessage(msg.getContent());
            case USER -> new UserMessage(msg.getContent());
            case ASSISTANT -> new AssistantMessage(msg.getContent());
            case TOOL -> new ToolResponseMessage(msg.getContent());
        };
    }

    private List<FunctionTool> toSpringAiTools(List<UnifiedToolDefinition> unifiedTools) {
        return unifiedTools.stream().map(tool -> {
            FunctionTool.Function function = FunctionTool.Function.builder()
                    .name(tool.getName())
                    .description(tool.getDescription())
                    .parameters(objectMapper.convertValue(tool.getParameters(), java.util.Map.class))
                    .build();
            return new FunctionTool(function);
        }).toList();
    }

    private UnifiedChatResponse toUnifiedResponse(ChatCompletionResponse response, String model) {
        return UnifiedChatResponse.builder()
                .id(response.getId())
                .model(model)
                .message(toUnifiedMessage(response.getResult().getOutput()))
                .usage(UnifiedChatResponse.Usage.builder()
                        .promptTokens(response.getUsage().getPromptTokens())
                        .completionTokens(response.getUsage().getGenerationTokens())
                        .totalTokens(response.getUsage().getTotalTokens())
                        .build())
                .build();
    }

    private UnifiedMessage toUnifiedMessage(AssistantMessage message) {
        List<ToolCall> toolCalls = null;
        if (message.getToolCalls() != null) {
            toolCalls = message.getToolCalls().stream()
                    .map(tc -> ToolCall.builder()
                            .id(tc.getId())
                            .type(tc.getType())
                            .functionName(tc.getName())
                            .arguments(tc.getArguments())
                            .build())
                    .toList();
        }

        return UnifiedMessage.builder()
                .role(MessageRole.ASSISTANT)
                .content(message.getContent())
                .toolCalls(toolCalls)
                .build();
    }

    private UnifiedChatResponse toUnifiedResponseFromChunk(ChatCompletionChunk chunk, String model) {
        AssistantMessage msg = chunk.getChoices().get(0).getMessage();
        return UnifiedChatResponse.builder()
                .id(chunk.getId())
                .model(model)
                .message(toUnifiedMessage(msg))
                .build();
    }
}
```

### 4.2 DeepSeek Adapter（兼容 OpenAI 格式）

DeepSeek 的 API 完全兼容 OpenAI，我们可以**继承** OpenAiAdapter 只改配置：

```java
@Slf4j
@Component
public class DeepSeekAdapter extends OpenAiAdapter {

    public DeepSeekAdapter(@Value("${llm.providers.deepseek.api-key}") String apiKey,
                           @Value("${llm.providers.deepseek.base-url:https://api.deepseek.com}") String baseUrl,
                           ObjectMapper objectMapper) {
        super(apiKey, baseUrl, objectMapper);
    }

    @Override
    public String providerName() {
        return "deepseek";
    }

    @Override
    public Set<String> supportedModels() {
        return Set.of("deepseek-chat", "deepseek-reasoner", "deepseek-v3");
    }
}
```

### 4.3 通义千问 Adapter（DashScope SDK）

通义千问使用阿里云 DashScope SDK，结构略有不同。为了简洁，我们使用 RestClient 直连 HTTP：

```java
@Slf4j
@Component
public class QwenAdapter implements LlmProviderAdapter {

    private final RestClient restClient;
    private final ObjectMapper objectMapper;

    public QwenAdapter(@Value("${llm.providers.qwen.api-key}") String apiKey,
                       @Value("${llm.providers.qwen.base-url:https://dashscope.aliyuncs.com}") String baseUrl,
                       ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
        this.restClient = RestClient.builder()
                .baseUrl(baseUrl)
                .defaultHeader("Authorization", "Bearer " + apiKey)
                .defaultHeader("Content-Type", "application/json")
                .build();
    }

    @Override
    public String providerName() {
        return "qwen";
    }

    @Override
    public Set<String> supportedModels() {
        return Set.of("qwen-turbo", "qwen-plus", "qwen-max",
                "qwen-max-longcontext", "qwen2.5-72b-instruct");
    }

    @Override
    public boolean supportsStreaming() {
        return true;
    }

    @Override
    public UnifiedChatResponse chatCompletion(UnifiedChatRequest request) {
        Map<String, Object> body = buildRequestBody(request, false);
        Map<String, Object> response = restClient.post()
                .uri("/compatible-mode/v1/chat/completions")
                .body(body)
                .retrieve()
                .body(Map.class);
        return parseResponse(response, request.getModel());
    }

    @Override
    public Flux<UnifiedChatResponse> chatCompletionStream(UnifiedChatRequest request) {
        Map<String, Object> body = buildRequestBody(request, true);
        return restClient.post()
                .uri("/compatible-mode/v1/chat/completions")
                .body(body)
                .accept(MediaType.TEXT_EVENT_STREAM)
                .retrieve()
                .bodyToFlux(String.class)
                .filter(line -> !line.isBlank() && line.startsWith("data:"))
                .map(line -> line.substring(5).trim())
                .filter(data -> !"[DONE]".equals(data))
                .map(data -> parseChunk(data, request.getModel()))
                .onErrorContinue((e, o) -> log.warn("Stream error: {}", e.getMessage()));
    }

    private Map<String, Object> buildRequestBody(UnifiedChatRequest req, boolean stream) {
        Map<String, Object> body = new LinkedHashMap<>();
        body.put("model", this.mapModelName(req.getModel()));
        body.put("stream", stream);

        List<Map<String, String>> messages = req.getMessages().stream().map(msg -> {
            Map<String, String> m = new LinkedHashMap<>();
            m.put("role", msg.getRole().name().toLowerCase());
            m.put("content", msg.getContent());
            return m;
        }).toList();
        body.put("messages", messages);

        if (req.getTemperature() != null) body.put("temperature", req.getTemperature());
        if (req.getMaxTokens() != null) body.put("max_tokens", req.getMaxTokens());
        if (req.getTopP() != null) body.put("top_p", req.getTopP());

        // 通义千问特有的参数
        body.put("enable_search", false);

        return body;
    }

    private UnifiedChatResponse parseResponse(Map<String, Object> response, String model) {
        List<Map<String, Object>> choices = (List<Map<String, Object>>) response.get("choices");
        Map<String, Object> choice = choices.get(0);
        Map<String, Object> message = (Map<String, Object>) choice.get("message");

        Map<String, Object> usage = (Map<String, Object>) response.get("usage");
        return UnifiedChatResponse.builder()
                .id((String) response.get("id"))
                .model(model)
                .message(UnifiedMessage.builder()
                        .role(MessageRole.ASSISTANT)
                        .content((String) message.get("content"))
                        .build())
                .usage(UnifiedChatResponse.Usage.builder()
                        .promptTokens((Integer) usage.get("prompt_tokens"))
                        .completionTokens((Integer) usage.get("completion_tokens"))
                        .totalTokens((Integer) usage.get("total_tokens"))
                        .build())
                .build();
    }

    private UnifiedChatResponse parseChunk(String json, String model) {
        try {
            Map<String, Object> chunk = objectMapper.readValue(json, Map.class);
            List<Map<String, Object>> choices = (List<Map<String, Object>>) chunk.get("choices");
            if (choices == null || choices.isEmpty()) return null;
            Map<String, Object> delta = (Map<String, Object>) choices.get(0).get("delta");
            String content = delta != null ? (String) delta.get("content") : "";
            return UnifiedChatResponse.builder()
                    .id((String) chunk.get("id"))
                    .model(model)
                    .message(UnifiedMessage.builder()
                            .role(MessageRole.ASSISTANT)
                            .content(content)
                            .build())
                    .build();
        } catch (Exception e) {
            log.error("Failed to parse chunk: {}", json, e);
            return null;
        }
    }
}
```

### 4.4 Ollama Adapter（本地模型）

本地模型的最大特点是 API 简单、零成本，但功能可能不完整（比如不支持 Function Calling）：

```java
@Slf4j
@Component
public class OllamaAdapter implements LlmProviderAdapter {

    private final RestClient restClient;
    private final ObjectMapper objectMapper;
    private final Set<String> modelCache = new ConcurrentSkipListSet<>();

    public OllamaAdapter(@Value("${llm.providers.ollama.base-url:http://localhost:11434}") String baseUrl,
                         ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
        this.restClient = RestClient.builder()
                .baseUrl(baseUrl)
                .defaultHeader("Content-Type", "application/json")
                .build();
        refreshModelList();
    }

    private void refreshModelList() {
        try {
            Map<String, Object> resp = restClient.get()
                    .uri("/api/tags")
                    .retrieve()
                    .body(Map.class);
            List<Map<String, Object>> models = (List<Map<String, Object>>) resp.get("models");
            if (models != null) {
                models.forEach(m -> modelCache.add((String) m.get("name")));
            }
        } catch (Exception e) {
            log.warn("Failed to fetch Ollama models: {}", e.getMessage());
        }
    }

    @Override
    public String providerName() {
        return "ollama";
    }

    @Override
    public Set<String> supportedModels() {
        return Collections.unmodifiableSet(modelCache);
    }

    @Override
    public boolean supportsStreaming() {
        return true;
    }

    @Override
    public UnifiedChatResponse chatCompletion(UnifiedChatRequest request) {
        Map<String, Object> body = buildOllamaRequest(request, false);
        Map<String, Object> response = restClient.post()
                .uri("/api/chat")
                .body(body)
                .retrieve()
                .body(Map.class);

        Map<String, Object> message = (Map<String, Object>) response.get("message");
        return UnifiedChatResponse.builder()
                .model(request.getModel())
                .message(UnifiedMessage.builder()
                        .role(MessageRole.ASSISTANT)
                        .content((String) message.get("content"))
                        .build())
                .usage(calculateOllamaUsage(response))
                .build();
    }

    @Override
    public Flux<UnifiedChatResponse> chatCompletionStream(UnifiedChatRequest request) {
        Map<String, Object> body = buildOllamaRequest(request, true);
        return restClient.post()
                .uri("/api/chat")
                .body(body)
                .accept(MediaType.APPLICATION_NDJSON)
                .retrieve()
                .bodyToFlux(String.class)
                .filter(line -> !line.isBlank())
                .map(line -> {
                    try {
                        Map<String, Object> chunk = objectMapper.readValue(line, Map.class);
                        Map<String, Object> msg = (Map<String, Object>) chunk.get("message");
                        return UnifiedChatResponse.builder()
                                .model(request.getModel())
                                .message(UnifiedMessage.builder()
                                        .role(MessageRole.ASSISTANT)
                                        .content(msg != null ? (String) msg.get("content") : "")
                                        .build())
                                .build();
                    } catch (Exception e) {
                        return null;
                    }
                })
                .filter(Objects::nonNull);
    }

    private Map<String, Object> buildOllamaRequest(UnifiedChatRequest req, boolean stream) {
        Map<String, Object> body = new LinkedHashMap<>();
        body.put("model", req.getModel());
        body.put("stream", stream);

        List<Map<String, String>> messages = req.getMessages().stream().map(msg -> {
            Map<String, String> m = new LinkedHashMap<>();
            m.put("role", msg.getRole().name().toLowerCase());
            m.put("content", msg.getContent());
            return m;
        }).toList();
        body.put("messages", messages);

        Map<String, Object> options = new LinkedHashMap<>();
        if (req.getTemperature() != null) options.put("temperature", req.getTemperature());
        if (req.getMaxTokens() != null) options.put("num_predict", req.getMaxTokens());
        if (req.getTopP() != null) options.put("top_p", req.getTopP());
        if (!options.isEmpty()) body.put("options", options);

        return body;
    }

    private UnifiedChatResponse.Usage calculateOllamaUsage(Map<String, Object> response) {
        int promptEval = ((Number) response.getOrDefault("prompt_eval_count", 0)).intValue();
        int eval = ((Number) response.getOrDefault("eval_count", 0)).intValue();
        return UnifiedChatResponse.Usage.builder()
                .promptTokens(promptEval)
                .completionTokens(eval)
                .totalTokens(promptEval + eval)
                .build();
    }
}
```

---

## 五、Strategy 模式：调用策略层

Adapter 解决了“怎么调”，Strategy 解决“怎么选”。比如，主模型挂了自动切备选模型：

```java
/**
 * 聊天调用策略接口
 */
public interface ChatStrategy {

    /** 策略名称 */
    String name();

    /** 执行聊天请求 */
    UnifiedChatResponse execute(UnifiedChatRequest request, List<LlmProviderAdapter> adapters);

    /** 执行流式聊天请求 */
    Flux<UnifiedChatResponse> executeStream(UnifiedChatRequest request,
                                             List<LlmProviderAdapter> adapters);
}
```

### 5.1 默认策略：单一模型调用

```java
@Slf4j
@Component
public class SingleModelStrategy implements ChatStrategy {

    @Override
    public String name() {
        return "single";
    }

    @Override
    public UnifiedChatResponse execute(UnifiedChatRequest request,
                                        List<LlmProviderAdapter> adapters) {
        String model = request.getModel();
        LlmProviderAdapter adapter = findAdapter(model, adapters);
        log.debug("Calling model {} via provider {}", model, adapter.providerName());
        return adapter.chatCompletion(request);
    }

    @Override
    public Flux<UnifiedChatResponse> executeStream(UnifiedChatRequest request,
                                                    List<LlmProviderAdapter> adapters) {
        String model = request.getModel();
        LlmProviderAdapter adapter = findAdapter(model, adapters);
        return adapter.chatCompletionStream(request);
    }

    private LlmProviderAdapter findAdapter(String model, List<LlmProviderAdapter> adapters) {
        return adapters.stream()
                .filter(a -> a.supportedModels().contains(model))
                .findFirst()
                .orElseThrow(() -> new IllegalArgumentException(
                        "No adapter found for model: " + model));
    }
}
```

### 5.2 高可用策略：故障转移（Fallover）

这是生产环境最核心的策略——**主模型挂了自动切备选**：

```java
@Slf4j
@Component
public class FalloverChatStrategy implements ChatStrategy {

    // 模型优先级配置：排在前的优先使用
    // 配置示例：llm.fallover.groups.chat = deepseek-chat -> qwen-plus -> gpt-4o-mini
    @Value("#{${llm.fallover.groups.chat}}")
    private Map<String, List<String>> falloverGroups;

    @Override
    public String name() {
        return "fallover";
    }

    @Override
    public UnifiedChatResponse execute(UnifiedChatRequest request,
                                        List<LlmProviderAdapter> adapters) {
        List<String> fallbackChain = falloverGroups.getOrDefault(
                request.getModel(),
                List.of(request.getModel())); // 没配置降级链就用原模型

        Exception lastException = null;
        for (String model : fallbackChain) {
            try {
                LlmProviderAdapter adapter = findAdapter(model, adapters);
                log.info("Trying model: {} via provider: {}", model, adapter.providerName());

                // 用备选模型名创建新请求
                UnifiedChatRequest retryRequest = UnifiedChatRequest.builder()
                        .model(model)
                        .messages(request.getMessages())
                        .temperature(request.getTemperature())
                        .maxTokens(request.getMaxTokens())
                        .topP(request.getTopP())
                        .stop(request.getStop())
                        .stream(request.isStream())
                        .tools(request.getTools())
                        .user(request.getUser())
                        .providerOptions(request.getProviderOptions())
                        .build();

                return adapter.chatCompletion(retryRequest);
            } catch (Exception e) {
                lastException = e;
                log.warn("Model {} failed, trying next fallback. Error: {}",
                        model, e.getMessage());
            }
        }

        throw new RuntimeException("All fallback models exhausted", lastException);
    }

    @Override
    public Flux<UnifiedChatResponse> executeStream(UnifiedChatRequest request,
                                                    List<LlmProviderAdapter> adapters) {
        // 流式场景的降级：直接尝试主模型，失败后切换到备选模型的非流式调用
        List<String> fallbackChain = falloverGroups.getOrDefault(
                request.getModel(), List.of(request.getModel()));

        String primaryModel = fallbackChain.get(0);
        LlmProviderAdapter primaryAdapter = findAdapter(primaryModel, adapters);

        return primaryAdapter.chatCompletionStream(request)
                .onErrorResume(e -> {
                    log.warn("Stream failed for model {}, falling back to non-stream",
                            primaryModel, e);
                    // 降级为非流式调用备选模型
                    for (int i = 1; i < fallbackChain.size(); i++) {
                        try {
                            LlmProviderAdapter fallbackAdapter =
                                    findAdapter(fallbackChain.get(i), adapters);
                            UnifiedChatRequest fallbackReq = toBuilder(request)
                                    .model(fallbackChain.get(i))
                                    .stream(false)
                                    .build();
                            UnifiedChatResponse resp = fallbackAdapter
                                    .chatCompletion(fallbackReq);
                            return Flux.just(resp);
                        } catch (Exception ex) {
                            log.warn("Fallback model {} also failed",
                                    fallbackChain.get(i), ex);
                        }
                    }
                    return Flux.error(new RuntimeException(
                            "All models exhausted after stream failure", e));
                });
    }

    private LlmProviderAdapter findAdapter(String model, List<LlmProviderAdapter> adapters) {
        return adapters.stream()
                .filter(a -> a.supportedModels().contains(model))
                .findFirst()
                .orElseThrow(() -> new IllegalArgumentException(
                        "No adapter found for model: " + model));
    }

    private UnifiedChatRequest.UnifiedChatRequestBuilder toBuilder(
            UnifiedChatRequest req) {
        return UnifiedChatRequest.builder()
                .model(req.getModel())
                .messages(req.getMessages())
                .temperature(req.getTemperature())
                .maxTokens(req.getMaxTokens())
                .topP(req.getTopP())
                .stop(req.getStop())
                .stream(req.isStream())
                .tools(req.getTools())
                .user(req.getUser())
                .providerOptions(req.getProviderOptions());
    }
}
```

---

## 六、统一门面：UnifiedLlmClient

现在把 Adapter 和 Strategy 组装起来：

```java
@Slf4j
@Service
public class UnifiedLlmClient {

    private final List<LlmProviderAdapter> adapters;
    private final Map<String, ChatStrategy> strategyMap;

    public UnifiedLlmClient(List<LlmProviderAdapter> adapters,
                             List<ChatStrategy> strategies) {
        this.adapters = adapters;
        // Spring 自动注入所有 ChatStrategy 实现
        this.strategyMap = strategies.stream()
                .collect(Collectors.toMap(ChatStrategy::name, Function.identity()));

        log.info("UnifiedLlmClient initialized with {} providers: {}",
                adapters.size(),
                adapters.stream().map(LlmProviderAdapter::providerName).toList());
        log.info("Available strategies: {}", strategyMap.keySet());
    }

    /**
     * 同步聊天（使用默认策略）
     */
    public UnifiedChatResponse chat(String model, String userMessage) {
        return chat(model, List.of(
                UnifiedMessage.builder()
                        .role(MessageRole.USER)
                        .content(userMessage)
                        .build()));
    }

    /**
     * 同步聊天（完整的消息历史）
     */
    public UnifiedChatResponse chat(String model, List<UnifiedMessage> messages) {
        return chat(UnifiedChatRequest.builder()
                .model(model)
                .messages(messages)
                .build(), "single");
    }

    /**
     * 同步聊天（指定策略）
     */
    public UnifiedChatResponse chat(UnifiedChatRequest request, String strategyName) {
        ChatStrategy strategy = strategyMap.getOrDefault(strategyName,
                strategyMap.get("single"));
        return strategy.execute(request, adapters);
    }

    /**
     * 流式聊天
     */
    public Flux<String> chatStream(String model, String userMessage) {
        return chatStream(UnifiedChatRequest.builder()
                .model(model)
                .messages(List.of(UnifiedMessage.builder()
                        .role(MessageRole.USER)
                        .content(userMessage)
                        .build()))
                .stream(true)
                .build(), "single");
    }

    /**
     * 流式聊天（指定策略）
     */
    public Flux<String> chatStream(UnifiedChatRequest request, String strategyName) {
        ChatStrategy strategy = strategyMap.getOrDefault(strategyName,
                strategyMap.get("single"));
        return strategy.executeStream(request, adapters)
                .map(resp -> resp.getMessage().getContent())
                .filter(Objects::nonNull);
    }

    /**
     * 获取所有可用模型
     */
    public Set<String> listModels() {
        return adapters.stream()
                .flatMap(a -> a.supportedModels().stream())
                .collect(Collectors.toSet());
    }

    /**
     * 按厂商获取模型
     */
    public Map<String, Set<String>> listModelsByProvider() {
        return adapters.stream()
                .collect(Collectors.toMap(
                        LlmProviderAdapter::providerName,
                        LlmProviderAdapter::supportedModels));
    }

    public List<LlmProviderAdapter> getAdapters() {
        return Collections.unmodifiableList(adapters);
    }
}
```

---

## 七、配置：切换模型只需改一行

### application.yml

```yaml
llm:
  # 默认策略
  default-strategy: single
  
  # 厂商配置
  providers:
    openai:
      api-key: ${OPENAI_API_KEY:}
      base-url: https://api.openai.com
    deepseek:
      api-key: ${DEEPSEEK_API_KEY:}
      base-url: https://api.deepseek.com
    qwen:
      api-key: ${QWEN_API_KEY:}
      base-url: https://dashscope.aliyuncs.com
    ollama:
      base-url: http://localhost:11434
  
  # 模型映射（统一别名 -> 实际模型名）
  model-mapping:
    gpt-4: gpt-4o
    cheap: deepseek-chat
    premium: gpt-4o
    local: qwen2.5:7b
  
  # 降级链配置
  fallover:
    groups:
      # 对话模型降级链：DeepSeek 挂了切通义，通义挂了切 OpenAI 便宜模型
      chat:
        deepseek-chat: 
          - deepseek-chat
          - qwen-plus
          - gpt-4o-mini

# Spring 自动配置
spring:
  autoconfigure:
    exclude: org.springframework.ai.autoconfigure.openai.OpenAiAutoConfiguration
```

现在切换模型有多简单？在业务代码中：

```java
// 之前：硬编码 OpenAI
String answer = openAiService.call("你好");

// 现在：一行配置随便换，代码不动
String answer = unifiedLlmClient.chat("deepseek-chat", "你好");
// 改成 GPT-4o，只改模型名
String answer = unifiedLlmClient.chat("gpt-4o", "你好");
// 改成本地 Ollama
String answer = unifiedLlmClient.chat("qwen2.5:7b", "你好");
```

甚至你可以在运行时动态切换：

```java
// 根据用户等级选模型
String model = user.isVip() ? "gpt-4o" : "deepseek-chat";
String answer = unifiedLlmClient.chat(model, userMessage);
```

---

## 八、进阶：扩展新厂商只需三步

假如明天出了一个新厂商 "MoonShot"，你要接入它，只需要：

**第1步**：实现 `LlmProviderAdapter` 接口

```java
@Component
public class MoonshotAdapter implements LlmProviderAdapter {
    @Override
    public String providerName() { return "moonshot"; }
    
    @Override
    public Set<String> supportedModels() { 
        return Set.of("moonshot-v1-8k", "moonshot-v1-32k", "moonshot-v1-128k");
    }
    
    // ... 实现其他方法
}
```

**第2步**：在 yml 中添加配置

```yaml
llm:
  providers:
    moonshot:
      api-key: ${MOONSHOT_API_KEY:}
      base-url: https://api.moonshot.cn
```

**第3步**：代码零修改，直接使用

```java
unifiedLlmClient.chat("moonshot-v1-32k", "请总结这篇长文档：...");
```

Spring 会自动发现并注入新的 Adapter。这就是**开闭原则**的完美实践——对扩展开放，对修改关闭。

---

## 九、整体架构图

```
                        ┌─────────────────┐
                        │   业务 Service    │
                        │  (YourService)   │
                        └────────┬────────┘
                                 │ 依赖
                        ┌────────▼────────┐
                        │ UnifiedLlmClient │  ← 统一门面
                        └────────┬────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              │                  │                  │
     ┌────────▼────────┐ ┌───────▼───────┐ ┌───────▼───────┐
     │ SingleModel     │ │ Fallover      │ │ (Custom)      │
     │ Strategy        │ │ Strategy      │ │ Strategy      │  ← 策略层
     └────────┬────────┘ └───────┬───────┘ └───────┬───────┘
              │                  │                  │
              └──────────────────┼──────────────────┘
                                 │
     ┌───────────────────────────┼───────────────────────────┐
     │                           │                           │
┌────▼─────┐  ┌──────────┐  ┌───▼──────┐  ┌──────────┐  ┌──▼───────┐
│ OpenAI   │  │ DeepSeek │  │ Qwen     │  │ Zhipu    │  │ Ollama   │
│ Adapter  │  │ Adapter  │  │ Adapter  │  │ Adapter  │  │ Adapter  │  ← 适配层
└────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘
     │              │              │              │              │
┌────▼─────┐  ┌─────▼────┐  ┌─────▼────┐  ┌─────▼────┐  ┌─────▼────┐
│ OpenAI   │  │ DeepSeek │  │ 阿里云   │  │ 智谱     │  │ 本地GPU  │
│ API      │  │ API      │  │ DashScope│  │ API      │  │ 服务器   │
└──────────┘  └──────────┘  └──────────┘  └──────────┘  └──────────┘
```

---

## 十、总结

一个好的多模型统一接入层，核心价值在于**解耦**：

1. **接口层统一**：`UnifiedChatRequest` / `UnifiedChatResponse` 抹平厂商差异
2. **Adapter 封装**：每个厂商一个 Adapter，互不影响，可独立维护
3. **Strategy 编排**：把“怎么调用”和“怎么选模型”解耦，策略可插拔
4. **配置化驱动**：切换模型、配置降级链，改 yml 就行，业务代码零改动
5. **开闭原则**：新增厂商只需一个 Adapter 类 + 一行配置，零侵入

在生产环境中，这个架构还要加上：

- **连接池管理**：每个厂商的 HttpClient 独立配置连接池大小
- **熔断降级**：集成 Resilience4j 做短路保护
- **计费埋点**：每次调用记录 Token 消耗（下一篇文章会讲）
- **监控告警**：接入 Prometheus + Grafana 监控各厂商可用性

**当你有了这层抽象，AI 厂商就不再是“供应商”，而是可以随意替换的“零件”。**

---

**下一篇预告**：

> **《AI 服务网关设计：统一认证、限流、降级、计费的 Java 实现，有了它 API Key 再也不会被盗刷》**
>
> 基于 Spring Cloud Gateway 设计生产级 AI 服务网关，涵盖 API Key 管理、JWT 认证、按用户/按模型限流、主备模型自动切换、Token 计数与成本分摊、审计日志落库等完整实现。敬请期待！

---

> 如果本文对你有帮助，欢迎**点赞、收藏、关注**三连。  
> 有任何问题欢迎在评论区留言讨论。
