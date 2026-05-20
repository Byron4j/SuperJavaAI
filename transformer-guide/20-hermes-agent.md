# 第 17 章 · Hermes Agent 详解

---

> Hermes 是 Nous Research 推出的专门针对 **Function Calling 和 Agent 场景** 微调的开源模型系列。本章解析其原理、架构与实战集成。

---

## 17.1 Hermes 模型系列

Nous Research 的 Hermes 系列是专门为 **工具使用和 Agent 场景** 微调的 LLM：

| 模型 | 基座 | 参数量 | 特点 |
|---|---|---|---|
| **Hermes 2** | LLaMA 2 / Mistral | 7B, 13B, 70B | 基础 Function Calling |
| **Hermes 2 Pro** | Mistral | 7B | 增强 JSON 输出 + 多工具并行 |
| **Hermes 3** | LLaMA 3.1 | 8B, 70B, 405B | Function Calling + 结构化输出 + Agent 能力 |

**与普通 LLM 的核心差异**：Hermes 在训练数据中加入了大量结构化的工具调用对话，使其天然理解如何将自然语言指令翻译为工具调用 JSON。

---

## 17.2 Hermes 的核心能力

### 17.2.1 结构化 Function Calling

```java
/**
 * Hermes 的对话格式（ChatML 风格）
 * 
 * 与 OpenAI Function Calling 兼容，但模型本身原生支持
 */
public class HermesChatFormat {

    public static final String FUNCTION_CALL_START = "<function_call>";
    public static final String FUNCTION_CALL_END = "</function_call>";
    public static final String TOOL_RESPONSE_START = "<tool_response>";
    public static final String TOOL_RESPONSE_END = "</tool_response>";

    /**
     * 构建 Hermes 格式的系统提示
     */
    public static String buildSystemPrompt(List<ToolDefinition> tools) {
        StringBuilder prompt = new StringBuilder();
        prompt.append("You are a helpful AI assistant with access to the following tools:\n\n");

        for (ToolDefinition tool : tools) {
            prompt.append(formatToolDefinition(tool));
        }

        prompt.append("""
            
            When you need to use a tool, respond with:
            <function_call>
            {"name": "tool_name", "arguments": {"arg1": "value1", "arg2": "value2"}}
            </function_call>
            
            After the tool response, continue your reasoning to answer the user.
            """);

        return prompt.toString();
    }

    /**
     * 解析 Hermes 的工具调用
     */
    public static Optional<ToolCall> parseFunctionCall(String modelOutput) {
        int start = modelOutput.indexOf(FUNCTION_CALL_START);
        int end = modelOutput.indexOf(FUNCTION_CALL_END);

        if (start == -1 || end == -1) {
            return Optional.empty();
        }

        String jsonStr = modelOutput.substring(
            start + FUNCTION_CALL_START.length(), end
        ).trim();

        // 解析 JSON: {"name": "get_weather", "arguments": {"city": "Beijing"}}
        JsonNode root = objectMapper.readTree(jsonStr);
        String name = root.get("name").asText();
        Map<String, Object> args = objectMapper.convertValue(
            root.get("arguments"), new TypeReference<>() {}
        );

        return Optional.of(new ToolCall(name, args));
    }
}
```

---

## 17.3 Hermes Agent 循环

Hermes 的核心指令遵循格式是 **"You are a function calling AI"** + **结构化对话格式**，使其在 Agent 场景中表现优异：

```java
/**
 * Hermes Agent 的核心推理循环
 */
public class HermesAgent {

    private final TransformerDecoder model;  // Hermes 模型
    private final ToolRegistry tools;
    private final int maxIterations = 10;

    /**
     * Hermes Agent 执行的完整流程
     */
    public String execute(String userMessage) {
        List<ChatMessage> conversation = new ArrayList<>();
        conversation.add(new ChatMessage("system", buildSystemPrompt(tools.listAll())));
        conversation.add(new ChatMessage("user", userMessage));

        int iterations = 0;

        while (iterations < maxIterations) {
            // 1. 模型推理
            String modelOutput = model.generate(conversation);

            // 2. 检查是否有工具调用
            Optional<ToolCall> toolCall = HermesChatFormat.parseFunctionCall(modelOutput);

            if (toolCall.isEmpty()) {
                // 没有工具调用 → 这就是最终回答
                return modelOutput;
            }

            // 3. 执行工具
            ToolCall call = toolCall.get();
            System.out.printf("[Agent] Calling: %s(%s)%n", call.name(), call.args());

            Object result = tools.execute(call.name(), call.args());

            // 4. 将工具调用和结果追加到对话
            String callText = modelOutput.substring(0,
                modelOutput.indexOf(HermesChatFormat.TOOL_RESPONSE_START) != -1
                    ? modelOutput.indexOf(HermesChatFormat.TOOL_RESPONSE_START)
                    : modelOutput.length()
            );

            conversation.add(new ChatMessage("assistant", callText));
            conversation.add(new ChatMessage("tool",
                HermesChatFormat.TOOL_RESPONSE_START
                + "\n" + toJson(result) + "\n"
                + HermesChatFormat.TOOL_RESPONSE_END
            ));

            iterations++;
        }

        return "达到最大迭代次数，任务未完成。";
    }
}
```

---

## 17.4 Hermes 的 JSON 结构化输出

Hermes 2 Pro 和 Hermes 3 特别强化了 JSON 输出能力：

```java
/**
 * Hermes 结构化输出 —— 强制模型输出合法 JSON
 * 
 * 使用方式: 在 System Prompt 中指定输出格式
 */
public class HermesStructuredOutput {

    /**
     * 告诉 Hermes 你想要 JSON 格式
     */
    public static String buildJsonPrompt(String task, JsonSchema schema) {
        return """
            你是一个 AI 助手。请严格按以下 JSON Schema 输出。

            JSON Schema:
            %s

            任务: %s

            请只输出 JSON，不要包含任何其他文字。
            JSON:
            """.formatted(schema.toPrettyString(), task);
    }

    /**
     * 解析 + 验证 Hermes 的 JSON 输出
     */
    public static <T> T parseResponse(String modelOutput, Class<T> targetType,
                                       JsonSchema schema) {
        // 1. 提取 JSON 代码块（Hermes 可能会用 ```json 包裹）
        String json = extractJson(modelOutput);

        // 2. 验证 JSON Schema
        if (!schema.validate(json)) {
            throw new IllegalArgumentException("Output does not match schema: " + json);
        }

        // 3. 反序列化
        return objectMapper.readValue(json, targetType);
    }

    /**
     * 从 Hermes 输出中提取 JSON
     */
    private static String extractJson(String output) {
        // 去除可能的 Markdown 代码块包裹
        output = output.replaceAll("```json\\s*", "").replaceAll("```\\s*", "").trim();

        // 找到第一个 { 和最后一个 }
        int start = output.indexOf('{');
        int end = output.lastIndexOf('}');
        if (start == -1 || end == -1) {
            throw new IllegalArgumentException("No JSON object found in output");
        }
        return output.substring(start, end + 1);
    }
}

// 使用示例
record WeatherReport(String city, int temperature, String condition, int humidity) {}

WeatherReport report = HermesStructuredOutput.parseResponse(
    modelOutput,
    WeatherReport.class,
    JsonSchema.builder()
        .property("city", JsonSchemaType.STRING)
        .property("temperature", JsonSchemaType.INTEGER)
        .property("condition", JsonSchemaType.STRING)
        .property("humidity", JsonSchemaType.INTEGER)
        .required("city", "temperature", "condition")
        .build()
);
```

---

## 17.5 Hermes 多工具并行调用

Hermes 2 Pro 的一个重要特性是 **多工具并行调用**：

```java
/**
 * Hermes 2 Pro 的并行工具调用
 * 
 * <function_call>
 * [
 *   {"name": "get_weather", "arguments": {"city": "Beijing"}},
 *   {"name": "get_weather", "arguments": {"city": "Shanghai"}},
 *   {"name": "get_weather", "arguments": {"city": "Guangzhou"}}
 * ]
 * </function_call>
 */
public class HermesParallelToolCall {

    /**
     * 解析并行工具调用
     */
    public static List<ToolCall> parseParallelCalls(String modelOutput) {
        Optional<String> jsonBlock = extractFunctionCallBlock(modelOutput);

        if (jsonBlock.isEmpty()) {
            return List.of();
        }

        String json = jsonBlock.get().trim();

        // 判断是数组还是单个对象
        if (json.startsWith("[")) {
            // 并行调用: [{...}, {...}]
            JsonNode array = objectMapper.readTree(json);
            List<ToolCall> calls = new ArrayList<>();
            for (JsonNode node : array) {
                calls.add(parseSingleCall(node));
            }
            return calls;
        } else {
            // 单个调用: {...}
            return List.of(parseSingleCall(objectMapper.readTree(json)));
        }
    }

    /**
     * 并行执行工具调用
     */
    public static List<ToolResult> executeParallel(
            List<ToolCall> calls, ToolRegistry tools) {
        // 用 CompletableFuture 真正并行执行
        List<CompletableFuture<ToolResult>> futures = calls.stream()
            .map(call -> CompletableFuture.supplyAsync(() -> {
                try {
                    Object result = tools.execute(call.name(), call.args());
                    return ToolResult.success(call.name(), result);
                } catch (Exception e) {
                    return ToolResult.error(call.name(), e.getMessage());
                }
            }))
            .toList();

        return futures.stream()
            .map(CompletableFuture::join)
            .toList();
    }
}
```

---

## 17.6 Hermes 训练原理

Hermes 能学会 Function Calling，关键在于**训练数据格式**：

```java
/**
 * Hermes 训练数据格式
 * 
 * 每条数据是一个完整的工具调用对话
 */
public class HermesTrainingData {

    /**
     * 训练数据的一轮（turn）
     */
    record TrainingTurn(
        String system,        // 系统提示（含工具定义）
        String user,          // 用户输入
        String assistant,     // 模型期望输出（含 <function_call> 或正常文本）
        String toolResponse,  // 如果 assistant 调用了工具，这是工具返回结果
        String finalAnswer    // 最终回答
    ) {}

    /**
     * 完整训练样本示例
     */
    public static TrainingTurn createWeatherExample() {
        return new TrainingTurn(
            """
            You are a helpful assistant with access to functions.
            Use them if required.
            Functions:
            - get_weather(city: string, unit?: string): Get current weather
            """,

            "What's the weather like in Paris?",

            // ===== 模型学习输出这个 =====
            """
            <function_call>
            {"name": "get_weather", "arguments": {"city": "Paris", "unit": "celsius"}}
            </function_call>
            """,

            // ===== 工具返回 =====
            "<tool_response>\n{\"temperature\": 22, \"condition\": \"partly cloudy\"}\n</tool_response>",

            // ===== 模型学习基于工具结果生成这个 =====
            "The current weather in Paris is partly cloudy with a temperature of 22°C."
        );
    }

    /**
     * 训练时如何计算 Loss
     */
    public static float computeTrainingLoss(TrainingTurn turn, TransformerDecoder model) {
        // 拼接完整对话
        String fullConversation = String.format(
            "<|im_start|>system\n%s<|im_end|>\n" +
            "<|im_start|>user\n%s<|im_end|>\n" +
            "<|im_start|>assistant\n%s<|im_end|>\n" +
            "<|im_start|>tool\n%s<|im_end|>\n" +
            "<|im_start|>assistant\n%s<|im_end|>",
            turn.system(), turn.user(), turn.assistant(),
            turn.toolResponse(), turn.finalAnswer()
        );

        int[] tokens = tokenize(fullConversation);

        // 关键: 对 tool_response 部分不计算 loss
        // 因为模型不需要"学习"天气数据本身，只需要学习"如何调用工具"和"如何根据结果回答"
        int[] lossMask = buildLossMask(tokens, turn);
        // lossMask[t] = 0 → 不参与梯度计算（tool_response 部分）
        // lossMask[t] = 1 → 参与梯度计算（system/user/assistant 部分）

        float[][] logits = model.forward(tokens);
        return maskedCrossEntropy(logits, tokens, lossMask);
    }
}
```

---

## 17.7 Hermes Agent 架构总结

```
Hermes Agent 架构全景:

┌─────────────────────────────────────────────────────────┐
│                      Hermes LLM                        │
│   (Nous Research 在 LLaMA 上微调 Function Calling)     │
│                                                        │
│   能力:                                                 │
│   ┌──────────────────────────────────────────────────┐ │
│   │ 1. 意图理解  → "我要查天气"                       │ │
│   │ 2. 工具选择  → 在 N 个工具中选出 get_weather     │ │
│   │ 3. 参数填充  → city="Beijing", unit="celsius"    │ │
│   │ 4. 结果整合  → 把 JSON 结果变成自然语言           │ │
│   │ 5. 错误恢复  → 工具失败时换方案                   │ │
│   │ 6. 多轮推理  → 需要多步工具调用时自主规划         │ │
│   └──────────────────────────────────────────────────┘ │
└───────────────────────────┬─────────────────────────────┘
                            │
    ┌───────────────────────┼───────────────────────┐
    ▼                       ▼                       ▼
┌─────────┐          ┌─────────┐           ┌──────────┐
│ Tool 1  │          │ Tool 2  │           │ MCP      │
│天气查询  │          │搜索     │           │ Server   │
└─────────┘          └─────────┘           └──────────┘
```

---

## 17.8 Function Calling 各方案对比

| 维度 | Hermes | OpenAI GPT-4 | 普通 LLM + Prompt | Grammar-Constrained |
|---|---|---|---|---|
| **准确率** | 高（专为 FC 微调） | 最高 | 中（不稳定） | 中（格式正确但语义可能差） |
| **成本** | 免费（本地部署） | 按 token 付费 | 免费 | 免费 |
| **数据隐私** | 完全本地 | 数据传出 | 完全本地 | 完全本地 |
| **多工具并行** | Hermes 2 Pro+ 原生支持 | 原生支持 | 难以实现 | 难以实现 |
| **JSON 稳定性** | 高（训练中有大量 JSON） | 极高 | 低 | 100%（但语义可能偏差） |
| **适用场景** | 私密数据、高频调用、离线 | 通用、最高质量 | 快速原型 | 对格式有强制要求的场景 |

---

## 17.9 Spring Boot 集成 Hermes

```java
/**
 * 用 ONNX Runtime 或 llama.cpp 加载 Hermes 模型
 */
@Service
public class HermesService {

    private final LLamaModel model;  // 通过 llama.cpp Java 绑定

    public HermesService(@Value("${hermes.model.path}") String modelPath) {
        this.model = new LLamaModel(modelPath, new ModelParams()
            .setNGpuLayers(35)       // GPU 加速
            .setContextSize(4096)    // 上下文窗口
        );
    }

    /**
     * Agent 对话接口
     */
    public String agentChat(String userMessage) {
        HermesAgent agent = new HermesAgent(model, tools, 10);
        return agent.execute(userMessage);
    }

    /**
     * 结构化输出接口
     */
    public <T> T structuredExtract(String input, Class<T> targetType, JsonSchema schema) {
        String prompt = HermesStructuredOutput.buildJsonPrompt(input, schema);
        String output = model.generate(prompt);
        return HermesStructuredOutput.parseResponse(output, targetType, schema);
    }
}
```

---

> **下一章**：[全书总结与展望](README.md) 或返回 [目录](README.md)
