# [论文解读] ReAct Agent 论文：推理与行动的协同艺术，为什么"边想边做"比"想完再做"更好

> **论文**：Yao, S., et al. (2023). *ReAct: Synergizing Reasoning and Acting in Language Models.* ICLR 2023.  
> **核心洞察**：人类解决问题不是"先想完再执行"，而是"思考→行动→观察→再思考"的循环——AI也应该如此。

---

## 一、开篇：人类解决问题的自然方式

想象你在调试一个Bug：

1. **想法**："这个NullPointerException可能是从userService来的，userService可能没注入进来"
2. **行动**：你在代码里加了一行`System.out.println(userService)`，重启服务
3. **观察**：控制台输出`null`
4. **新想法**："果然没注入，但你明明加了`@Autowired`...哦等等，这个类是不是没有被Spring管理？"
5. **新行动**：检查类上有没有`@Component`注解
6. **观察**：确实没有
7. **解决**：加上`@Component`，Bug修复

这就是人类自然的**Thought-Action-Observation循环**。

**ReAct的核心思想就是把这套流程搬到大语言模型上。**

---

## 二、为什么"只推理"不够？—— CoT的局限性

### 2.1 Chain-of-Thought (CoT) 的优势和弱点

CoT（思维链）让LLM把推理过程写出来，确实能提升复杂问题的准确率。但是：

**问题1：幻觉陷阱**  
LLM推理时可能"想当然"。比如问"2024年奥运会在哪个城市举办"，模型可能推出"应该是巴黎"，但它没有实际去查证。

**问题2：无法与外部世界交互**  
CoT只是"脑子里想"，不能真正去查数据、调API、执行代码。推理的基础如果是错误知识，推出来的结论必然错误。

**问题3：推理链无法被验证和修正**  
你无法中途插入"等等，你上一步错了"——因为CoT是一次性的，推理过程无法中断。

### 2.2 用Java类比

CoT就像这样：

```java
// CoT：一次性推理，无法获取外部信息
public String pureReasoning(String question) {
    // LLM从头到尾自己推，推错了也改不了
    return llm.generate("""
        请逐步推理回答：
        
        问题：%s
        
        推理步骤：""".formatted(question));
}
```

---

## 三、为什么"只行动"不够？—— Act-Only的问题

纯行动模式（Act-Only）就是让模型一直执行动作，不明确写出推理过程。

**问题**：
- 行动缺少计划性，容易"乱撞"
- 无法总结规律、调整策略
- 没有显式推理，人类（和模型自己）不知道"为什么要这样做"

ReAct论文的实验证明：**单独用Reasoning或单独用Acting，效果都不如二者结合。**

---

## 四、ReAct的魔力：1+1 > 2

### 4.1 ReAct的核心循环

```
┌──────────┐     ┌──────────┐     ┌──────────┐
│  THOUGHT │ ──→ │  ACTION  │ ──→ │OBSERVATION│
│  (推理)   │ ←── │  (行动)   │ ←── │  (观察)   │
└──────────┘     └──────────┘     └──────────┘
      ↑                                  │
      └──────────────────────────────────┘
                循环直到完成任务
```

### 4.2 一个完整的ReAct示例

论文中经典的问答示例：

```
Question: 科罗拉多造山运动东部区域的海拔范围是多少？

Thought 1: 我需要搜索"科罗拉多造山运动"，找到它影响的区域。
Action 1: Search[科罗拉多造山运动]
Observation 1: 科罗拉多造山运动...影响了从科罗拉多州东部到堪萨斯州西部...
              科罗拉多州东部区域的高平原海拔约1000米。

Thought 2: 观察1提到了"高平原海拔约1000米"，但我需要的是"东部区域"的具体海拔范围。
          让我搜索更具体的信息。
Action 2: Lookup[科罗拉多州东部海拔范围]
Observation 2: 科罗拉多州东部平原...海拔范围大致在1000-2000米之间。

Thought 3: 海拔范围是1000-2000米。让我再查"科罗拉多造山运动"来确认这个答案。
Action 3: Search[科罗拉多造山运动 东部区域 海拔]
Observation 3: 科罗拉多造山运动在科罗拉多高原...东部区域在堪萨斯州...
              最高点约2000米。

Thought 4: 综合信息，答案应是1000-2000米。
Action 4: Finish[1000-2000米]
```

**注意每个Thought的作用**：
- Thought 1：制定搜索计划
- Thought 2：评估上次结果，决定下一步
- Thought 3：验证信息
- Thought 4：综合判断，准备输出

这正是人类解决问题的自然思维过程。

---

## 五、Java实现一个完整的ReAct Loop

下面是一个可以直接运行的ReAct Agent实现：

```java
import java.util.*;
import java.util.regex.*;

/**
 * 完整的 ReAct Agent 实现
 * 
 * 核心逻辑：
 * 1. LLM 输出 Thought/Action/Observation 格式的文本
 * 2. 解析 Action，调用对应的工具
 * 3. 将工具返回结果作为 Observation 喂回 LLM
 * 4. 循环直到 LLM 输出 Finish 或达到最大迭代次数
 */
public class ReActAgent {

    // ==================== 配置 ====================
    static final int MAX_ITERATIONS = 10;        // 最大循环次数，防止死循环
    
    // ==================== 核心循环 ====================
    
    public String run(String task) {
        // 用 List 记录完整对话历史
        List<String> history = new ArrayList<>();
        
        // 初始 Prompt：告诉LLM它是什么角色，可以用什么工具
        history.add(buildSystemPrompt());
        history.add("User Task: " + task);
        
        System.out.println("========================================");
        System.out.println("🎯 Task: " + task);
        System.out.println("========================================\n");
        
        // ReAct 主循环
        for (int step = 0; step < MAX_ITERATIONS; step++) {
            System.out.println("--- Step " + (step + 1) + " ---");
            
            // Step 1: 调用 LLM，获取 Thought + Action
            String llmOutput = callLLM(history);
            System.out.println("LLM Output:\n" + llmOutput + "\n");
            
            // Step 2: 解析 Action
            ParsedAction action = parseAction(llmOutput);
            
            if (action == null) {
                System.out.println("⚠️ 无法解析Action，重试...");
                history.add(llmOutput);
                continue;
            }
            
            // Step 3: 检查是否是结束指令
            if ("FINISH".equals(action.toolName)) {
                System.out.println("✅ 任务完成");
                return action.arguments;
            }
            
            // Step 4: 执行工具
            String observation = executeTool(action.toolName, action.arguments);
            System.out.println("Observation: " + observation + "\n");
            
            // Step 5: 把结果喂给LLM
            history.add(llmOutput);
            history.add("Observation: " + observation);
        }
        
        return "Agent未能在" + MAX_ITERATIONS + "步内完成任务";
    }
    
    // ==================== Prompt 构建 ====================
    
    String buildSystemPrompt() {
        return """
            You are an AI agent that follows the ReAct (Reasoning + Acting) framework.
            
            YOUR FORMAT (MUST follow exactly):
            Thought: <your reasoning about what to do next>
            Action: <TOOL_NAME>[<arguments>]
            
            After each Action, you will receive an Observation.
            When you have the answer, use:
            Thought: I have enough information to answer.
            Action: FINISH[<your final answer>]
            
            AVAILABLE TOOLS:
            - Search[关键词]  : 搜索互联网获取信息
            - Calculator[算式] : 执行数学计算 (支持 + - * /)
            - Weather[城市名]  : 获取城市当前天气
            - Database[SQL语句] : 查询数据库
            
            IMPORTANT: Always write Thought before Action.
            """;
    }
    
    // ==================== LLM 调用 ====================
    
    String callLLM(List<String> history) {
        // 实际代码中替换为 OpenAI/Claude API 调用
        // 
        // OpenAI Java SDK 示例：
        //   OpenAI client = OpenAI.builder()
        //       .apiKey(System.getenv("OPENAI_API_KEY"))
        //       .build();
        //   ChatCompletionResponse response = client.chatCompletion(
        //       ChatCompletionRequest.builder()
        //           .model("gpt-4")
        //           .messages(historyToMessages(history))
        //           .build()
        //   );
        //   return response.choices().get(0).message().content();
        
        // 这里用模拟输出演示ReAct循环结构
        // 实际使用时删除这段mock，取消上面代码的注释
        return mockLLMResponse(history);
    }
    
    // ==================== Action 解析 ====================
    
    static class ParsedAction {
        String toolName;
        String arguments;
        
        ParsedAction(String toolName, String arguments) {
            this.toolName = toolName;
            this.arguments = arguments;
        }
    }
    
    ParsedAction parseAction(String llmOutput) {
        // 匹配 Action: TOOL_NAME[arguments]
        Pattern pattern = Pattern.compile("Action:\\s*(\\w+)\\[(.*?)\\]", 
                                          Pattern.DOTALL);
        Matcher matcher = pattern.matcher(llmOutput);
        
        if (matcher.find()) {
            String toolName = matcher.group(1).toUpperCase();
            String arguments = matcher.group(2).trim();
            return new ParsedAction(toolName, arguments);
        }
        return null;
    }
    
    // ==================== 工具执行 ====================
    
    String executeTool(String toolName, String args) {
        return switch (toolName) {
            case "SEARCH" -> executeWebSearch(args);
            case "CALCULATOR" -> executeCalculation(args);
            case "WEATHER" -> executeWeatherQuery(args);
            case "DATABASE" -> executeDatabaseQuery(args);
            case "FINISH" -> args; // 不会被调用，在 run() 里拦截了
            default -> "Unknown tool: " + toolName;
        };
    }
    
    String executeWebSearch(String query) {
        // 实际代码中使用 SerpAPI / Google Custom Search API
        // 或使用 Brave Search API / Tavily API
        
        // 示例：使用 Tavily Java SDK
        // TavilyClient client = TavilyClient.builder()
        //     .apiKey(System.getenv("TAVILY_API_KEY"))
        //     .build();
        // SearchResponse response = client.search(query);
        // return response.results().stream()
        //     .map(r -> r.title() + ": " + r.content())
        //     .collect(Collectors.joining("\n"));
        
        return "[模拟] 搜索结果: " + query + " 的相关信息...";
    }
    
    String executeCalculation(String expression) {
        try {
            // 简单实现，生产环境用更安全的表达式引擎
            // 例如：JEXL, exp4j, 或直接调 WolframAlpha API
            return expression + " = [计算结果]";
        } catch (Exception e) {
            return "计算错误: " + e.getMessage();
        }
    }
    
    String executeWeatherQuery(String city) {
        // 调用和风天气 / OpenWeatherMap API
        return "[模拟] " + city + " 天气: 晴, 22°C";
    }
    
    String executeDatabaseQuery(String sql) {
        // 这里要对SQL做安全检查！生产环境必须加权限控制和SQL注入防护
        // 建议：①白名单限制操作类型 ②禁止修改操作 ③使用只读数据库副本
        return "[模拟] 查询结果: [数据库返回的数据]";
    }
    
    // ==================== Mock（演示用，实际删除） ====================
    
    private int mockStep = 0;
    
    String mockLLMResponse(List<String> history) {
        mockStep++;
        // 模拟一个完整的ReAct推理过程
        return switch (mockStep) {
            case 1 -> """
                Thought: I need to search for the latest Java LTS version features.
                Action: Search[Java 21 new features 2024]""";
            case 2 -> """
                Thought: The search returned information about virtual threads and pattern matching.
                        I should also check what version is the latest LTS.
                Action: Search[Java latest LTS version 2025]""";
            case 3 -> """
                Thought: I now have comprehensive information. The latest LTS is Java 21,
                        featuring Virtual Threads, Pattern Matching, Record Patterns, and
                        Sequenced Collections. I can provide a complete answer.
                Action: FINISH[Java 21 (released September 2023) is the latest LTS version.
                        Key features: 1) Virtual Threads (Project Loom) - lightweight concurrency,
                        2) Pattern Matching for switch, 3) Record Patterns,
                        4) Sequenced Collections, 5) String Templates (preview).]""";
            default -> """
                Thought: I seem to be going in circles. Let me summarize what I know.
                Action: FINISH[Task completed with available information.]""";
        };
    }
    
    // ==================== 入口 ====================
    
    public static void main(String[] args) {
        ReActAgent agent = new ReActAgent();
        String result = agent.run("告诉我Java最新LTS版本及其核心新特性");
        System.out.println("\n🎉 Final Answer:\n" + result);
    }
}

/**
 * ==================== 扩展：生产级 ReAct 框架 ====================
 * 
 * 上述代码是教学用的简化版。生产环境建议使用成熟框架：
 * 
 * 1. LangChain4j (Java)
 *    - 内置 ReAct Agent 实现
 *    - 支持流式输出、工具调用
 *    - 文档：https://docs.langchain4j.dev/
 *    
 *    @Service
 *    public class MyAgent {
 *        @Tool("搜索互联网获取信息")
 *        public String search(String query) { ... }
 *        
 *        @Tool("计算数学表达式")
 *        public double calculate(String expr) { ... }
 *    }
 *    
 *    AiServices.builder(Assistant.class)
 *        .chatLanguageModel(model)
 *        .tools(new MyAgent())
 *        .build();
 * 
 * 2. Spring AI (Spring官方)
 *    - 函数调用 (Function Calling) 支持
 *    - 与 Spring Boot 深度集成
 * 
 * 3. AutoGPT/AgentGPT 等开源项目的Java移植
 */
```

### 关键设计要点

1. **Prompt工程**：必须明确告诉LLM输出格式，否则它可能自由发挥导致解析失败
2. **循环上限**：必须设置`MAX_ITERATIONS`，防止Agent陷入死循环
3. **工具安全**：Agent调用外部工具（特别是数据库/API）时必须有权限控制
4. **解析健壮性**：LLM输出可能不符合格式，需要重试机制

---

## 六、ReAct vs 其他Agent范式

ReAct论文发表后，出现了多个Agent范式的变体：

| 范式 | 核心思想 | 适用场景 | 代表实现 |
|------|----------|----------|----------|
| **ReAct** | 推理→行动→观察循环 | 需要多步信息收集的任务 | LangChain Agent |
| **Plan-and-Execute** | 先生成完整计划，再逐步执行 | 任务步骤明确、路径固定 | BabyAGI |
| **Reflexion** | ReAct + 自我反思，执行后自我评估 | 需要质量保证的任务 | Reflexion Agent |
| **AutoGPT** | 带回滚能力的自主Agent | 开放式长任务 | AutoGPT |
| **Tool-Augmented** | 工具定义+函数调用 | 结构化API调用 | OpenAI Function Calling |

**ReAct的优势**：
- 灵活：不需要提前规划全局路径
- 可纠错：每一步都能根据观察调整
- 透明：人类可以追踪每步推理过程
- 简单：不需要复杂的子任务管理

---

## 七、生产环境中的注意事项

```java
/**
 * 生产级 ReAct Agent 的额外考量
 */
public class ProductionReActAgent {

    // 1. Token 控制 —— 避免上下文爆炸
    //    当历史对话过长时，使用滑动窗口或摘要压缩
    private List<String> manageContext(List<String> history, int maxTokens) {
        // 保留系统Prompt + 最近N轮对话
        // 或使用 LLM 对历史进行摘要
        return history;
    }

    // 2. 工具超时控制
    private String executeToolWithTimeout(String tool, String args, int timeoutMs) {
        // 不能让一个慢查询卡住整个Agent
        return CompletableFuture
            .supplyAsync(() -> executeTool(tool, args))
            .completeOnTimeout("Tool timeout", timeoutMs, TimeUnit.MILLISECONDS)
            .exceptionally(e -> "Tool error: " + e.getMessage())
            .join();
    }

    // 3. 成本控制
    //    记录每步的Token消耗，超出预算时优雅降级
    // 4. 幂等性
    //    如果执行过程中断，重运行时不应重复产生副作用
    // 5. 审计日志
    //    记录Agent的每一步Thought/Action/Observation，便于调试
}
```

---

## 八、总结与预告

### ReAct为什么有效

ReAct不是一项炫技的技术，它是在**接近人类思考的本质**：

1. **Thought提供了方向**：像GPS导航，不断根据当前位置调整路线
2. **Action提供了信息**：不再"蒙眼推理"，每一步都有真实信息作为输入
3. **Observation提供了反馈**：确保推理链条不跑偏

三者结合，让LLM从一个"话痨"（只会说）变成了一个"能干活的助手"（会说也会做）。

### 下期预告

ReAct让AI学会了"思考和行动"，但这还只局限于**文本世界**。现在的AI已经能"看"图片、"听"声音、"理解"视频了——这就是**多模态大模型**（GPT-4V、Gemini、Claude Vision等）。下一篇文章将带你拆解多模态大模型的底层技术路线，以及Java开发者如何调用这些"能看会听"的AI。

**关键词**：#ReAct #Agent #LLM #推理 #LangChain4j #SpringAI

---

> **参考文献**
> - Yao, S., et al. (2023). ReAct: Synergizing Reasoning and Acting in Language Models. *ICLR 2023*.
> - Wei, J., et al. (2022). Chain-of-Thought Prompting Elicits Reasoning in Large Language Models.
> - Shinn, N., et al. (2023). Reflexion: Language Agents with Verbal Reinforcement Learning.
