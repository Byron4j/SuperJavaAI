# ReAct Agent 实战：Java 实现"思考-行动-观察"循环，一个会自我纠错的AI程序

> 上篇我们搞懂了Agent的"感知→思考→行动→观察"循环。这一篇，我们把它变成可运行的代码——一个会自我纠错、能用工具、能在Java里跑起来的ReAct Agent。

---

## 一、为什么ReAct是Agent的"Hello World"？

2022年，Google Brain的论文《ReAct: Synergizing Reasoning and Acting in Language Models》提出了一种让LLM交替进行"推理"和"行动"的模式。这个看似简单的设计，却成了现代Agent的基石。

ReAct的核心设计哲学只有一句话：

> **思考指导行动，行动反馈思考。**

用伪代码表达就是：

```
while (任务未完成) {
    thought = LLM.think(上下文 + 历史);   // 思考：我该干什么？
    if (thought == "任务完成") break;
    action = parseAction(thought);         // 解析：调用什么工具？
    result = toolExecutor.execute(action); // 行动：执行工具
    context += "观察: " + result;           // 观察：把结果喂回LLM
}
```

上一篇文章我们写了一个最简Agent，但它用的是规则判断代替LLM。这篇文章，我们要动真格的——**接入真实LLM，实现完整的ReAct循环**。

---

## 二、ReAct的原理：为什么这个循环能自我纠错？

### 2.1 推理与行动的交替

ReAct最巧妙的地方在于，它把LLM的输出格式设计成交替出现的**Thought（思考）** 和**Action（行动）**：

```
Thought: 我需要先获取北京的气温数据
Action: get_weather(city="北京")
Observation: 北京当前气温32°C，湿度65%

Thought: 当前气温很高，我需要查询历史同期数据做对比
Action: query_history(city="北京", date="去年同期")
Observation: 去年今日气温为28°C，湿度50%

Thought: 今年确实比去年热不少。可以将结果整理告知用户
Finish: 北京当前气温32°C，比去年同期高出4°C，湿度也增加了15%。建议注意防暑降温。
```

每一次Observation都可能**改变后续的推理方向**。如果天气API返回了错误，LLM在下一轮的Thought中会自动调整——它可能选择重试、换一个数据源、或直接告诉用户API出错了。

### 2.2 自我纠错是怎么发生的？

这是ReAct最让人兴奋的特性——不需要任何特殊的"错误处理"代码，Agent自己会纠错：

```
Thought: 我需要获取北京的气温数据
Action: get_weather("北京")
Observation: Error: 查询失败，城市代码无效

Thought: 查询失败，可能是城市名需要转换为标准格式。让我先获取北京的城市代码。
Action: get_city_code("北京")
Observation: 城市代码: 101010100

Thought: 现在用城市代码重新查询天气
Action: get_weather_by_code("101010100")
Observation: 北京当前气温32°C

Thought: 成功获取数据，可以返回结果了
Finish: 北京当前气温32°C。
```

**没有一行纠错代码，但Agent自己完成了"重试→分析原因→换方案→成功"的全过程。** 这就是LLM推理能力+工具执行循环的威力。

---

## 三、完整实现：从零手写ReActAgent

下面是一个**可直接运行的完整Java实现**。唯一的依赖是HTTP客户端（这里用OkHttp），你需要一个兼容OpenAI API的端点（国内的DeepSeek、通义千问、智谱都可以）。

### 3.1 工具集定义

首先定义Agent的工具系统——这是Agent的"手"：

```java
import java.util.*;
import java.util.concurrent.CompletableFuture;
import java.util.function.Function;

/**
 * 工具定义——描述Agent能调用的每一个工具
 */
public class Tool {
    String name;           // 工具名（LLM决策时使用）
    String description;    // 工具描述（给LLM看，用来决定何时调用）
    Map<String, String> parameters; // 参数定义
    Function<Map<String, Object>, String> executor; // 实际执行的函数

    public Tool(String name, String description,
                Map<String, String> parameters,
                Function<Map<String, Object>, String> executor) {
        this.name = name;
        this.description = description;
        this.parameters = parameters;
        this.executor = executor;
    }

    public String execute(Map<String, Object> args) {
        return executor.apply(args);
    }

    // 生成给LLM看的工具描述（JSON Schema格式）
    public String toPromptDescription() {
        StringBuilder sb = new StringBuilder();
        sb.append("- ").append(name).append(": ").append(description).append("\n");
        sb.append("  参数: ").append(parameters).append("\n");
        return sb.toString();
    }
}

/**
 * 工具注册中心
 */
class ToolRegistry {
    private final Map<String, Tool> tools = new LinkedHashMap<>();

    public ToolRegistry register(Tool tool) {
        tools.put(tool.name, tool);
        return this;
    }

    public Tool get(String name) {
        return tools.get(name);
    }

    public String getAllToolDescriptions() {
        StringBuilder sb = new StringBuilder();
        for (Tool t : tools.values()) {
            sb.append(t.toPromptDescription());
        }
        return sb.toString();
    }

    public boolean contains(String name) {
        return tools.containsKey(name);
    }
}
```

### 3.2 完整的ReAct Agent核心循环

```java
import com.fasterxml.jackson.databind.ObjectMapper;
import okhttp3.*;

import java.util.*;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

/**
 * ReAct Agent 完整实现
 * 核心流程：Thought → Action → Observation → Next Thought → ...
 */
public class ReActAgent {

    private final String apiKey;
    private final String apiUrl;
    private final String model;
    private final ToolRegistry registry;
    private final int maxIterations;
    private final OkHttpClient httpClient;
    private final ObjectMapper objectMapper;
    private final List<StepRecord> history;

    // 构建器模式
    public static class Builder {
        private String apiKey;
        private String apiUrl = "https://api.deepseek.com/v1/chat/completions";
        private String model = "deepseek-chat";
        private ToolRegistry registry = new ToolRegistry();
        private int maxIterations = 10;

        public Builder apiKey(String key) { this.apiKey = key; return this; }
        public Builder apiUrl(String url) { this.apiUrl = url; return this; }
        public Builder model(String m) { this.model = m; return this; }
        public Builder registerTool(Tool tool) { this.registry.register(tool); return this; }
        public Builder maxIterations(int n) { this.maxIterations = n; return this; }

        public ReActAgent build() {
            return new ReActAgent(this);
        }
    }

    private ReActAgent(Builder builder) {
        this.apiKey = builder.apiKey;
        this.apiUrl = builder.apiUrl;
        this.model = builder.model;
        this.registry = builder.registry;
        this.maxIterations = builder.maxIterations;
        this.httpClient = new OkHttpClient();
        this.objectMapper = new ObjectMapper();
        this.history = new ArrayList<>();
    }

    /**
     * Agent核心执行循环
     */
    public String run(String userQuery) {
        System.out.println("══════ ReAct Agent 启动 ══════");
        System.out.println("👤 用户请求: " + userQuery + "\n");

        // 构建系统提示词：告诉LLM它现在是一个ReAct Agent
        String systemPrompt = buildSystemPrompt();
        List<Map<String, String>> messages = new ArrayList<>();
        messages.add(Map.of("role", "system", "content", systemPrompt));
        messages.add(Map.of("role", "user", "content", userQuery));

        int iteration = 0;
        Set<String> actionHistory = new HashSet<>(); // 死循环检测

        while (iteration < maxIterations) {
            iteration++;
            System.out.println("─── 第 " + iteration + " 轮 ───");

            // 【Step 1: 思考 + 行动决策】调用LLM
            String llmResponse = callLLM(messages);
            System.out.println("📤 LLM输出:\n" + llmResponse + "\n");

            // 【Step 2: 解析LLM输出】提取 Action 或 Finish
            ParsedResponse parsed = parseResponse(llmResponse);

            if (parsed.isFinished) {
                System.out.println("✅ Agent完成任务！");
                return parsed.finalAnswer;
            }

            // 【Step 3: 死循环检测】
            String actionKey = parsed.actionName + "(" + parsed.actionArgs + ")";
            if (actionHistory.contains(actionKey)) {
                System.out.println("⚠️ 检测到重复调用，Agent可能陷入死循环，强制终止！");
                messages.add(Map.of("role", "user", "content",
                    "你刚才调用了一次完全相同的操作。请改变策略或直接给出你能给出的最佳答案。"));
                continue;
            }
            actionHistory.add(actionKey);

            // 【Step 4: 执行工具 → 获得观察结果】
            String observation = executeAction(parsed.actionName, parsed.actionArgs);
            System.out.println("👁 观察: " + observation + "\n");

            // 【Step 5: 记录历史 + 将观察反馈给LLM】
            history.add(new StepRecord(iteration, parsed.thought, parsed.actionName,
                                       parsed.actionArgs, observation));
            messages.add(Map.of("role", "assistant", "content", llmResponse));
            messages.add(Map.of("role", "user", "content", "Observation: " + observation));
        }

        return "达到最大迭代次数(" + maxIterations + ")，Agent未能在限定步骤内完成任务。";
    }

    /**
     * 构建系统提示词——这是整个Agent的灵魂
     */
    private String buildSystemPrompt() {
        return """
            你是一个具备工具调用能力的AI助手。你需要通过"思考→行动→观察"的循环来完成任务。

            请严格按照以下格式回复：

            Thought: <你当前的推理和思考过程>
            Action: <工具名>(参数1=值1, 参数2=值2)
            ---- 或 ----
            Thought: <你认为任务已完成的思考>
            Finish: [最终答案]

            可用工具：
            %s

            规则：
            1. 每次只能调用一个工具
            2. 收到 Observation 后，根据结果决定下一步
            3. 如果工具调用失败，分析原因并尝试其他方案
            4. 当信息足够时，用 Finish 给出答案
            5. 不要重复完全相同的工具调用
            """
            .formatted(registry.getAllToolDescriptions());
    }

    /**
     * 调用LLM（兼容 OpenAI API 格式）
     */
    private String callLLM(List<Map<String, String>> messages) {
        try {
            // 将messages转为OpenAI API请求格式
            List<Map<String, String>> msgList = new ArrayList<>();
            for (var msg : messages) {
                msgList.add(Map.of("role", msg.get("role"), "content", msg.get("content")));
            }

            Map<String, Object> requestBody = new LinkedHashMap<>();
            requestBody.put("model", model);
            requestBody.put("messages", msgList);
            requestBody.put("temperature", 0.0); // 低温度保证输出稳定
            requestBody.put("max_tokens", 2048);

            String json = objectMapper.writeValueAsString(requestBody);
            Request request = new Request.Builder()
                    .url(apiUrl)
                    .header("Authorization", "Bearer " + apiKey)
                    .header("Content-Type", "application/json")
                    .post(RequestBody.create(json, MediaType.parse("application/json")))
                    .build();

            try (Response response = httpClient.newCall(request).execute()) {
                String responseBody = response.body().string();
                // 从响应中提取 content
                return extractContentFromResponse(responseBody);
            }
        } catch (Exception e) {
            return "Thought: LLM调用失败\nFinish: [抱歉，AI服务目前不可用: " + e.getMessage() + "]";
        }
    }

    /**
     * 从OpenAI格式的响应中提取文本内容
     */
    @SuppressWarnings("unchecked")
    private String extractContentFromResponse(String responseBody) throws Exception {
        Map<String, Object> responseMap = objectMapper.readValue(responseBody, Map.class);
        List<Map<String, Object>> choices = (List<Map<String, Object>>) responseMap.get("choices");
        if (choices != null && !choices.isEmpty()) {
            Map<String, Object> message = (Map<String, Object>) choices.get(0).get("message");
            return (String) message.get("content");
        }
        throw new RuntimeException("无法解析LLM响应: " + responseBody);
    }

    /**
     * 解析LLM的响应：提取 Thought/Action/Finish
     */
    private ParsedResponse parseResponse(String llmText) {
        // 匹配 Thought
        Pattern thoughtPattern = Pattern.compile("Thought:\\s*(.+?)(?=Action:|Finish:|$)",
                                                 Pattern.DOTALL);
        Matcher thoughtMatcher = thoughtPattern.matcher(llmText);
        String thought = thoughtMatcher.find() ? thoughtMatcher.group(1).trim() : "";

        // 匹配 Action: tool_name(param1=val1, param2=val2)
        Pattern actionPattern = Pattern.compile("Action:\\s*(\\w+)\\(([^)]*)\\)");
        Matcher actionMatcher = actionPattern.matcher(llmText);
        if (actionMatcher.find()) {
            String actionName = actionMatcher.group(1).trim();
            String rawArgs = actionMatcher.group(2).trim();

            // 解析参数
            Map<String, Object> args = new LinkedHashMap<>();
            if (!rawArgs.isEmpty()) {
                for (String pair : rawArgs.split(",")) {
                    String[] kv = pair.split("=", 2);
                    if (kv.length == 2) {
                        args.put(kv[0].trim(), kv[1].trim());
                    }
                }
            }

            System.out.println("🧠 Thought: " + thought);
            System.out.println("🔧 Action: " + actionName);
            return new ParsedResponse(thought, actionName, args);
        }

        // 匹配 Finish: [答案]
        Pattern finishPattern = Pattern.compile("Finish:\\s*\\[(.+?)\\]", Pattern.DOTALL);
        Matcher finishMatcher = finishPattern.matcher(llmText);
        if (finishMatcher.find()) {
            System.out.println("🧠 Thought: " + thought);
            return new ParsedResponse(true, finishMatcher.group(1).trim());
        }

        // 如果什么都没匹配到，默认认为完成了
        return new ParsedResponse(true, llmText.trim());
    }

    /**
     * 执行工具
     */
    private String executeAction(String toolName, Map<String, Object> args) {
        if (!registry.contains(toolName)) {
            return "Error: 未找到工具 [" + toolName + "]。请检查工具名是否正确。可用的工具有：" +
                   String.join(", ", getToolNames());
        }
        try {
            Tool tool = registry.get(toolName);
            return tool.execute(args);
        } catch (Exception e) {
            return "Error: 工具执行失败 - " + e.getMessage();
        }
    }

    private List<String> getToolNames() {
        List<String> names = new ArrayList<>();
        registry.getAllToolDescriptions(); // 简化，实际可用内部names
        return names;
    }

    // ═══════════ 内部类 ═══════════

    record ParsedResponse(String thought, String actionName, Map<String, Object> actionArgs,
                          boolean isFinished, String finalAnswer) {
        ParsedResponse(String thought, String actionName, Map<String, Object> actionArgs) {
            this(thought, actionName, actionArgs, false, null);
        }
        ParsedResponse(boolean isFinished, String finalAnswer) {
            this(null, null, null, isFinished, finalAnswer);
        }
    }

    record StepRecord(int iteration, String thought, String actionName,
                      Map<String, Object> args, String observation) {}
}
```

### 3.3 组装并运行

```java
public class ReActAgentDemo {

    public static void main(String[] args) {
        // ========== 1. 创建工具 ==========

        // 天气查询工具
        Tool weatherTool = new Tool(
            "get_weather",
            "查询指定城市的实时天气信息，返回温度和天气状况",
            Map.of("city", "城市名称（支持中文）"),
            params -> {
                String city = (String) params.get("city");
                // 实际场景中调天气API
                return "城市: " + city + " | 温度: 32°C | 天气: 晴 | 湿度: 65%";
            }
        );

        // 计算工具
        Tool calculatorTool = new Tool(
            "calculate",
            "执行数学计算，支持加减乘除和复杂表达式",
            Map.of("expression", "数学表达式，如 2+3*4"),
            params -> {
                String expr = (String) params.get("expression");
                // 实际场景中用JS引擎或Math库
                return "计算结果: 42 (模拟)";
            }
        );

        // 搜索工具（RAG集成点）
        Tool searchTool = new Tool(
            "web_search",
            "在互联网上搜索信息，返回相关结果摘要",
            Map.of("query", "搜索关键词"),
            params -> {
                String query = (String) params.get("query");
                return "搜索 '" + query + "' 的结果: \n" +
                       "1. AI Agent 是2026年热门技术方向\n" +
                       "2. ReAct 框架能显著提升LLM的任务完成率";
            }
        );

        // 代码执行工具
        Tool codeExecutionTool = new Tool(
            "execute_code",
            "在沙箱中执行Python/JavaScript代码，返回执行结果",
            Map.of("code", "要执行的代码"),
            params -> {
                String code = (String) params.get("code");
                return "代码执行成功，输出: Hello from sandbox!";
            }
        );

        // ========== 2. 创建Agent ==========
        ReActAgent agent = new ReActAgent.Builder()
            .apiKey(System.getenv("OPENAI_API_KEY"))  // 从环境变量读取
            .apiUrl("https://api.deepseek.com/v1/chat/completions")
            .model("deepseek-chat")
            .registerTool(weatherTool)
            .registerTool(calculatorTool)
            .registerTool(searchTool)
            .registerTool(codeExecutionTool)
            .maxIterations(8)  // 最多循环8轮
            .build();

        // ========== 3. 运行 ==========
        String result = agent.run("北京今天热不热？如果气温超过30度，帮我搜索一下防暑降温的方法。");
        System.out.println("\n══════ 最终结果 ══════");
        System.out.println(result);
    }
}
```

**运行效果（模拟）**：

```
══════ ReAct Agent 启动 ══════
👤 用户请求: 北京今天热不热？如果超过30度，搜索防暑降温方法。

─── 第 1 轮 ───
📤 LLM输出:
Thought: 用户想知道北京今天的温度。我需要先查询天气。
Action: get_weather(city=北京)

🔧 Action: get_weather
👁 观察: 城市: 北京 | 温度: 32°C | 天气: 晴

─── 第 2 轮 ───
📤 LLM输出:
Thought: 气温32°C，超过30度了。按照用户要求，我需要搜索防暑降温方法。
Action: web_search(query=防暑降温方法)

🔧 Action: web_search
👁 观察: 搜索 '防暑降温方法' 的结果: 1.多喝水 2.避免高温时段外出...

─── 第 3 轮 ───
📤 LLM输出:
Thought: 已经获取了天气和防暑信息，可以给用户完整答复了。
Finish: [北京当前32°C，确实很热！防暑建议：多喝水、避免高温时段外出、穿浅色衣物、使用空调时温度控制在26°C左右。]

✅ Agent完成任务！

══════ 最终结果 ══════
北京当前32°C，确实很热！防暑建议：多喝水、避免高温时段外出、穿浅色衣物...
```

---

## 四、ReAct + RAG：知识增强的Agent

前面我们提到，Agent可以调用工具。那么RAG（检索增强生成）其实也可以被包装成Agent的一个工具。这样Agent在推理过程中能主动地"去知识库里翻资料"。

### 4.1 方案一：RAG作为工具

这是最简单的集成方式——把向量搜索包装成一个普通的Tool：

```java
/**
 * 将向量知识库包装成Agent工具
 */
public class RAGTool {

    public static Tool create(EmbeddingStore embeddingStore, LLM llm) {
        return new Tool(
            "search_knowledge_base",
            "搜索企业内部知识库。当需要查找公司内部文档、规章制度、技术规范时使用此工具。",
            Map.of("query", "搜索查询，使用自然语言描述你想查找的内容"),
            params -> {
                String query = (String) params.get("query");

                // Step1: 向量检索
                List<TextSegment> relevantDocs = embeddingStore.search(query, 5);

                // Step2: 让LLM总结检索到的内容
                String context = relevantDocs.stream()
                    .map(TextSegment::text)
                    .collect(Collectors.joining("\n---\n"));

                String summary = llm.chat("""
                    根据以下文档内容回答问题：
                    文档：%s
                    问题：%s
                    请给出最相关的信息摘要。
                    """.formatted(context, query));

                return summary;
            }
        );
    }
}
```

### 4.2 方案二：Agent增强的RAG

反过来，Agent的推理能力也可以增强RAG的质量：

```java
/**
 * Agent增强的检索——让Agent决定"怎么搜"而不仅是"搜什么"
 */
public class AgentEnhancedRAG {

    public String queryWithAgent(String userQuestion, ReActAgent agent) {
        // 让Agent先分析：这个问题需要从哪些角度搜索？
        String strategyPrompt = """
            用户想了解：%s
            请分析这个问题，确定需要从哪些角度进行知识库检索（最多5个检索方向）。
            格式：用逗号分隔的检索方向列表
            """.formatted(userQuestion);

        String searchAngles = agent.run(strategyPrompt);
        String[] queries = searchAngles.split(",");

        // 多轮检索
        List<String> allResults = new ArrayList<>();
        for (String query : queries) {
            allResults.add(agent.run("search_knowledge_base(" + query.trim() + ")"));
        }

        // 汇总
        return agent.run("综合以上所有检索结果，回答：%s\n\n检索结果：%s"
            .formatted(userQuestion, String.join("\n", allResults)));
    }
}
```

这种方式让Agent先在"如何检索"上花一步推理，再利用多个检索角度获取更全面的上下文，最后综合回答。这比传统的单次RAG检索质量高出一个档次。

---

## 五、Agent循环控制：防止失控的三道防线

生产环境中，什么都不怕，就怕Agent放飞自我。以下是三道防线：

### 5.1 第一道防线：最大迭代次数

```java
private static final int MAX_ITERATIONS = 10;

while (iteration < MAX_ITERATIONS) {
    // ... Agent循环
}
```

这是最基础的兜底机制。设置一个合理的上限（通常5-15），能防止Agent在复杂任务中无限循环，消耗大量Token。

### 5.2 第二道防线：收敛判断

怎么判断Agent"卡住了"？看两条：

```java
// ① 完全相同的工具调用——明显是死循环
String actionKey = actionName + "(" + args + ")";
if (actionHistory.contains(actionKey)) {
    warnAndAdjust("你刚才调用了完全相同的操作，请改变策略。");
}

// ② 最近N轮都没有实质性进展
if (recentObservations.size() >= 3) {
    boolean stuck = recentObservations.stream()
        .allMatch(o -> o.contains("Error") || o.contains("失败"));
    if (stuck) {
        warnAndAdjust("连续多次操作失败。请换一个思路或直接基于现有信息给出最佳答案。");
    }
}
```

### 5.3 第三道防线：Token预算控制

ReAct循环中每一轮都在消耗Token（系统提示词+历史消息），很容易把上下文窗口打爆：

```java
private String callLLMWithBudgetControl(List<Map<String, String>> messages) {
    int estimatedTokens = estimateTokens(messages);
    if (estimatedTokens > MAX_CONTEXT_TOKENS * 0.8) {
        // 超过80%窗口时，压缩历史消息
        System.out.println("⚠️ Token预算告警，压缩历史...");
        messages = compressHistory(messages);
    }
    return callLLM(messages);
}

private List<Map<String, String>> compressHistory(List<Map<String, String>> messages) {
    // 保留系统提示词 + 最近5轮 + 用户的原始问题
    // 中间的轮次用"摘要"代替
    List<Map<String, String>> compressed = new ArrayList<>();
    compressed.add(messages.get(0)); // 系统提示词
    compressed.add(messages.get(1)); // 用户原始问题

    // 对中间轮次做摘要
    if (messages.size() > 6) {
        String summary = summarizeMiddleRounds(messages.subList(2, messages.size() - 5));
        compressed.add(Map.of("user", "之前的操作摘要: " + summary));
    }

    // 保留最近5轮
    int start = Math.max(2, messages.size() - 5);
    compressed.addAll(messages.subList(start, messages.size()));

    return compressed;
}
```

---

## 六、ReAct的局限与对策

ReAct虽然强大，但不是银弹：

| 局限 | 表现 | 对策 |
|------|------|------|
| Token消耗大 | 每轮都要携带完整历史，10轮可能消耗10K+ token | Token预算控制、历史压缩 |
| 推理速度慢 | 串行执行，每一步都要等LLM返回 | 对于可并行的工具调用，换用Plan-Execute（下篇讲） |
| 容易跑偏 | 连续错误后可能离题万里 | 定期检查与原始目标的语义相似度 |
| 输出格式不稳定 | 非JSON格式下LLM有时不按套路出牌 | 增加格式校验+重试机制 |

这些局限也正是后续架构（Plan-and-Execute、ReWOO）要解决的问题。

---

## 七、总结

这篇文章我们从零实现了一个完整的ReAct Agent，核心要点：

- **ReAct = 推理(Reasoning) + 行动(Acting)的交替循环**
- **自我纠错来自"Observation指导下一轮Thought"的反馈机制**
- **三道防线保生产**：最大迭代+收敛检测+Token预算
- **RAG可以包装成Agent的工具，Agent的推理能力也能增强RAG**

源码可以直接拿去用，替换API Key和工具实现即可。

---

**下一篇预告**：《Plan-and-Execute Agent：先制定计划再逐步执行的分步式智能体，复杂任务这样分解AI才靠谱》——ReAct每次只规划一步，遇到复杂任务容易跑偏。Plan-and-Execute Agent先做全局计划，再按计划逐步执行，天然适合多步骤、有依赖关系的复杂任务。

---

> **关于作者**：Java资深研发，专注AI工程化落地。关注我，获取更多Java+AI的实战干货。
