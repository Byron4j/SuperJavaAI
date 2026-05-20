# 第 13 章 · Function Calling / Tool Use

---

> Function Calling 是 Transformer 从"聊天机器人"蜕变为"AI Agent"的关键一步——让 LLM 不仅能说话，还能**做事**。

---

## 13.1 什么是 Function Calling？

**Function Calling（函数调用 / 工具使用）** 是一种让 LLM 在生成文本过程中**调用外部工具**的能力。模型不是直接输出答案，而是输出一个结构化的函数调用请求（JSON），由外部程序执行后，结果返回给模型继续推理。

```
传统 LLM:
  用户: "北京今天天气怎么样？"
  模型: "抱歉，我无法获取实时数据。"

Function Calling LLM:
  用户: "北京今天天气怎么样？"
  模型: → 输出 get_weather(city="北京") 调用的 JSON
  系统: → 执行 API 调用，返回 {temp: 25, condition: "晴"}
  模型: "北京今天晴，气温 25°C。"
```

---

## 13.2 Transformer 如何处理 Function Calling？

从架构层面看，Function Calling 不改变 Transformer 本身——它改变的是**输入输出的格式和训练数据**：

```java
/**
 * Function Calling 的三阶段流程
 */
public class FunctionCallingPipeline {
    private final TransformerDecoder model;  // LLM (Decoder-Only)
    private final Map<String, Function<ToolArgs, ToolResult>> tools;

    /**
     * 阶段 1: 模型决定是否需要调用工具
     * 
     * Transformer 的 logits 输出中，前几个 token 如果有特殊标记
     * (如 <function_call>)，说明模型决定调用工具
     */
    public GenerationResult generateWithTools(String userMessage, List<ChatMessage> history) {
        // 构建带工具描述的 Prompt
        String systemPrompt = buildToolPrompt(tools);
        List<ChatMessage> messages = prepend(new ChatMessage("system", systemPrompt), history);
        messages.add(new ChatMessage("user", userMessage));

        // 模型正常生成，但如果遇到 <function_call> 就切换到结构化输出
        int[] generated = new int[1024];
        int pos = 0;

        while (pos < 1024) {
            float[] logits = model.forwardStep(generated, pos);
            int nextToken = sample(logits);

            // 检测到工具调用标记
            if (nextToken == FUNCTION_CALL_TOKEN) {
                // 切换到结构化生成模式
                String toolCallJson = generateStructuredJson(model, generated, pos);
                ToolCall call = parseToolCall(toolCallJson);

                // 执行工具
                ToolResult result = executeTool(call);

                // 将工具结果追加到上下文
                messages.add(new ChatMessage("tool", result.toJson(), call.name));
                // 继续生成最终回答（重新开始）
                return generateWithoutTools(messages);
            }

            generated[pos++] = nextToken;
            if (nextToken == EOS_TOKEN) break;
        }
        return new GenerationResult(tokenizer.decode(generated, pos));
    }

    /**
     * 阶段 2: 结构化 JSON 生成（受限解码）
     * 
     * 生成 JSON 时，通过 logit masking 确保输出合法 JSON
     */
    private String generateStructuredJson(TransformerDecoder model, int[] prefix, int pos) {
        StringBuilder json = new StringBuilder();
        int state = 0; // 0=key, 1=value, 2=done
        // 通过 grammar-constrained decoding 保证 JSON 格式
        // ...
        return json.toString();
    }
}
```

---

## 13.3 训练数据格式

要让 Transformer 学会 Function Calling，训练数据需要特殊格式：

```
训练数据示例:
─────────────────────────────────────────
<|system|>
You have access to the following functions:
- get_weather(city: string, unit: string): Returns current weather
- search_web(query: string): Searches the internet

When you need information, call a function by outputting:
<function_call>{"name": "...", "arguments": {...}}</function_call>
<|user|>
What's the weather in Beijing?
<|assistant|>
<function_call>{"name": "get_weather", "arguments": {"city": "Beijing", "unit": "celsius"}}</function_call>
<|tool_response|>
{"temperature": 25, "condition": "sunny", "humidity": 45}
<|assistant|>
The weather in Beijing is sunny with a temperature of 25°C and humidity at 45%.
```

**关键点**：Transformer 学会了一种特殊的"语言"——先输出 `{name, arguments}` 的结构化 JSON，等待外部填入结果，再继续生成自然语言回答。

---

## 13.4 主流实现方案对比

| 方案 | 代表 | 方式 | 优点 | 缺点 |
|---|---|---|---|---|
| **原生 Function Call** | OpenAI GPT-4, Claude | 模型原生支持，API 传入工具定义 | 最准确、最方便 | 闭源、依赖特定模型 |
| **Prompt Engineering** | 任意开源模型 | 在 System Prompt 中描述工具 | 任何模型都可用 | 不如原生准确，不稳定 |
| **Grammar-Constrained** | llama.cpp, outlines | 强制模型输出符合 JSON Schema | 100% 合法 JSON | 需要特殊推理框架 |
| **Fine-Tuned** | Gorilla, Functionary | 专门微调过的模型 | 专为工具调用优化 | 需要训练数据和算力 |

---

## 13.5 Spring Boot 集成 Function Calling

```java
@Service
public class FunctionCallingService {

    private final ChatClient chatClient;
    private final Map<String, FunctionDefinition> tools;

    public FunctionCallingService(ChatClient.Builder builder) {
        this.chatClient = builder.build();
        this.tools = new HashMap<>();
        registerWeatherTool();
        registerSearchTool();
    }

    /**
     * 注册天气查询工具
     */
    private void registerWeatherTool() {
        FunctionDefinition weatherFn = FunctionDefinition.builder()
            .name("get_weather")
            .description("获取指定城市的当前天气信息")
            .parameters(JsonSchemaObject.builder()
                .property("city", JsonSchemaProperty.builder()
                    .type(JsonSchemaType.STRING)
                    .description("城市名称，如 'Beijing'")
                    .build())
                .property("unit", JsonSchemaProperty.builder()
                    .type(JsonSchemaType.STRING)
                    .description("温度单位: 'celsius' 或 'fahrenheit'")
                    .enumValues(List.of("celsius", "fahrenheit"))
                    .build())
                .required(List.of("city"))
                .build())
            .build();

        tools.put("get_weather", weatherFn);
    }

    /**
     * 对话接口——自动处理工具调用
     */
    public String chat(String userMessage) {
        var response = chatClient.prompt()
            .user(userMessage)
            .functions(tools.values())   // 注册可用工具
            .functionCallback("get_weather", args -> {
                String city = args.get("city").asText();
                String unit = args.getOrDefault("unit", new TextNode("celsius")).asText();
                return callWeatherAPI(city, unit);
            })
            .call();

        return response.content();
    }

    /**
     * 实际的天气 API 调用
     */
    private String callWeatherAPI(String city, String unit) {
        // 调用第三方天气 API
        String response = restTemplate.getForObject(
            "https://api.weather.com/v1/current?city={city}&unit={unit}",
            String.class, city, unit
        );
        return response;
    }
}
```

---

## 13.6 Function Calling 的深层原理

### 13.6.1 模型如何"决定"调用工具？

本质上，Transformer 在第 t 步输出的 logits 向量 `[vocabSize]` 中，`<function_call>` 特殊 token 的 logit 最高，说明模型"决定"调用工具。

```java
// 模型内部的决策逻辑（简化为伪代码）
float[] logits = model.forward(last_hidden_state);  // [vocabSize]
float[] probs = softmax(logits);

// 检查 "工具调用" 相关 token 的概率
float toolCallProb = probs[FUNCTION_CALL_TOKEN_ID];
float normalTextProb = probs.mostProbableExcept(FUNCTION_CALL_TOKEN_ID);

if (toolCallProb > threshold) {
    // 模型认为需要调用工具
    return enterToolCallMode();
} else {
    // 正常文本生成
    return normalDecoding();
}
```

### 13.6.2 受限解码（Constrained Decoding）

生成 JSON 时，通过修改 logits 强制输出合法 JSON：

```java
/**
 * 受限解码：保证输出符合 JSON Schema
 * 
 * 在每一步生成前，根据当前 JSON 状态掩码不合法 token 的 logits
 */
public int constrainedSample(float[] logits, JSONState state) {
    Set<Integer> allowedTokens = state.getAllowedNextTokens();

    // 把不允许的 token 的 logits 设为 -∞
    for (int i = 0; i < logits.length; i++) {
        if (!allowedTokens.contains(i)) {
            logits[i] = Float.NEGATIVE_INFINITY;
        }
    }

    return sample(logits);
}
```

### 13.6.3 并行工具调用

现代 LLM（GPT-4 Turbo+）支持在一次回复中调用多个工具：

```
用户: "对比北京和上海的天气"
模型: → 同时输出两个工具调用
  <function_call>{"name": "get_weather", "arguments": {"city": "Beijing"}}</function_call>
  <function_call>{"name": "get_weather", "arguments": {"city": "Shanghai"}}</function_call>
```

```java
// 并行执行多个工具调用
List<CompletableFuture<ToolResult>> futures = toolCalls.stream()
    .map(call -> CompletableFuture.supplyAsync(() -> executeTool(call)))
    .toList();

List<ToolResult> results = futures.stream()
    .map(CompletableFuture::join)
    .toList();
```

---

## 13.7 Function Calling 的训练（Fine-Tuning）

让模型学会使用工具的微调过程：

```java
/**
 * 训练 Function Calling 的迭代过程
 */
public class ToolUseTrainer {

    // 训练数据格式
    record ToolUseExample(
        String systemPrompt,        // 包含工具定义
        List<Message> conversation,  // 多轮对话（含工具调用和结果）
        List<Integer> ignoreMask    // 工具返回结果的 loss 设为 0（不学工具返回的内容）
    ) {}

    /**
     * 训练循环
     */
    public void train(List<ToolUseExample> dataset) {
        for (ToolUseExample example : dataset) {
            // 拼接整个对话（含工具调用和结果）
            int[] tokens = concatenateTokens(example.conversation);

            // 前向传播
            float[][] logits = model.forward(tokens);

            // 计算 Loss，但忽略工具返回结果部分
            float loss = maskedCrossEntropy(logits, tokens, example.ignoreMask);
            // ignoreMask 中，工具返回结果的位置 = 0（不参与 loss）
            // 正常对话和工具调用 JSON 的位置 = 1（参与 loss）

            // 反向传播
            loss.backward();
            optimizer.step();
        }
    }
}
```

---

## 13.8 从 Function Calling 到 Agent

Function Calling 是 Agent 的**基础构建块**。一个完整的 Agent 还需要：

```
Function Calling （伸手做事）
     │
     ├─→ 规划 (Planning)         —— 分解任务为多个步骤
     ├─→ 反思 (Reflection)       —— 检查结果是否正确
     ├─→ 记忆 (Memory)           —— 记住多轮交互的结果  
     ├─→ 多工具编排 (Orchestration) —— 选择和组合多个工具
     └─→ 错误恢复 (Error Recovery) —— 工具失败时的重试策略
```

这些将在后续章节 (MCP, Skills, OpenClaw, Hermes Agent) 中逐层展开。

---

> **下一章**：[MCP 协议详解](17-mcp-protocol.md)
