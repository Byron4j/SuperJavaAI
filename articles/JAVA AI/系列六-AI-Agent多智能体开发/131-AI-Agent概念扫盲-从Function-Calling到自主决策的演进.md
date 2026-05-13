# AI Agent 概念扫盲：从 Function Calling 到自主决策的演进，你的AI助手正在进化成"AI员工"

> 2026年的技术热词，从RAG变成了Agent。但Agent到底是什么？为什么说它是AI应用的下一阶段？这篇文章一次给你讲透。

---

## 一、开篇：你家门口的AI革命

2025年，所有人都在聊RAG（检索增强生成），我们学会了让AI"翻书"回答问题。到了2026年，风口变成了Agent。你可能已经听说了——Devin能写代码、Manus能订机票、OpenAI的Operator能操作浏览器。

但你有没有想过：这些所谓的"Agent"到底是什么？和我们之前做的Function Calling有什么本质区别？

打个比方：

- **ChatGPT（纯大模型）** 就像一个知识渊博但手无寸铁的书生——什么都知道，但什么都做不了。
- **Function Calling 的AI** 像是一个会打电话的秘书——你说"查天气"，它帮你调用天气API；你说"发邮件"，它帮你调邮件接口。
- **AI Agent** 则像一个真正的员工——你给它一个目标，它自己想办法、做计划、调工具、看结果、调整策略、直到完成任务。

这中间的跨越，比你想象的要大得多。我们今天就从零开始，把AI Agent这件事彻底讲清楚。

---

## 二、Agent的核心定义：感知·思考·行动·观察

Agent这个概念并不是AI领域的原创，它来自人工智能的经典定义。1995年，Russell和Norvig在《Artificial Intelligence: A Modern Approach》中给出过一个经受住时间考验的定义：

> **An agent is anything that can be viewed as perceiving its environment through sensors and acting upon that environment through actuators.**

翻译成人话：**Agent 就是能在环境中感知、思考、行动的智能体。**

放在LLM的语境下，这个循环变成了四个步骤：

```
┌──────────────────────────────────────────────────────┐
│                   AI Agent 循环                       │
│                                                      │
│  ① 感知(Perceive)    →  接收用户输入 + 环境反馈      │
│         ↓                                            │
│  ② 思考(Think)       →  LLM 推理下一步该做什么       │
│         ↓                                            │
│  ③ 行动(Act)         →  调用工具/执行操作            │
│         ↓                                            │
│  ④ 观察(Observe)     →  获取工具返回结果             │
│         ↓                                            │
│     回到②，直到任务完成                              │
└──────────────────────────────────────────────────────┘
```

注意，这和我们以前做的聊一句回一句完全不是一回事。传统的Chat场景是一次性的——用户问，AI答，结束。而Agent的场景是**持续性的**——AI需要在这个循环中转很多圈才能完成一个任务。

举个真实场景：用户说"帮我分析一下竞争对手的定价策略"。

- **传统Chat**：AI凭训练数据里的记忆给你一段泛泛的分析。
- **Agent**：先搜索竞品官网→抓取价格信息→对比分析→生成报告→保存到知识库→通知用户。一步接一步，自己决定下一步做什么。

---

## 三、Agent的四个能力等级：你在哪个Level？

业界对Agent的能力有一个模糊但实用的分级。我根据自己的实践经验，把Agent分成四个等级：

### L1：Tool Calling（工具调用级）

这是Agent的入门阶段，也是目前**95%的"AI应用"实际在做的**。

**能力特征**：
- LLM能识别用户的意图，决定调用哪个函数
- 每次调用都是"用户指令→LLM决策→单次工具调用→返回结果"
- 没有多步规划，没有自我纠错

**典型代码**（Spring AI / LangChain4j 都能轻松实现）：

```java
// L1级别的Tool Calling——这是绝大多数"AI产品"的真实状态
@Tool(name = "get_weather", description = "获取指定城市的天气信息")
public String getWeather(String city) {
    return weatherService.query(city); // 调第三方API返回天气数据
}

// LLM收到用户消息"北京今天天气怎么样？"
// → 自动识别需要调用get_weather("北京")
// → 拿到结果返回给用户
// 整个过程1次思考 + 1次工具调用。这就是L1。
```

**局限性**：遇到需要"先查A，再根据A的结果决定查B"的任务，L1就抓瞎了。你得在业务代码里手动编排流程，Agent自己不会。

### L2：Planning（规划执行级）

L2的Agent开始具备**自主规划和多步执行**的能力。

**能力特征**：
- 能将复杂任务拆解成多个子步骤
- 按顺序执行，并根据中间结果调整后续步骤
- 有了"观察→反思→再行动"的循环（这就是ReAct模式的核心）
- 能自我纠错——执行失败时自动重试或换方案

**举个例子**：
用户说"帮我找一家朝阳区评分最高的川菜馆，订个今晚7点的位子"。

L2 Agent的思考过程：
1. 先搜索"朝阳区川菜馆"→ 拿到列表
2. 从列表中筛选评分最高的 → 确定"蜀大侠"
3. 查询"蜀大侠今晚7点"是否有位 → 有位
4. 调用订座API → 成功
5. 返回结果给用户

整个过程Agent自主决策，不需要人类在每一步"点头"。

### L3：Multi-Agent（多智能体协作级）

L3是当前学术界和工业界的前沿。多个Agent各司其职，像一个微型团队。

**能力特征**：
- 多个Agent分工协作（如：规划Agent + 执行Agent + 审查Agent）
- Agent之间可以通信、协商、传递任务
- 能处理单个Agent解决不了的复杂场景

**典型架构**：

```
          ┌──────────────┐
          │  用户输入     │
          └──────┬───────┘
                 ↓
        ┌────────────────┐
        │  Orchestrator  │ ← 总调度Agent，负责任务拆分和分配
        │   (调度Agent)   │
        └───┬────┬────┬──┘
            ↓    ↓    ↓
      ┌─────┐ ┌───┐ ┌───┐
      │搜索  │ │分析│ │执行│  ← 各专业Agent各司其职
      │Agent │ │Agent│ │Agent│
      └─────┘ └───┘ └───┘
            ↓    ↓    ↓
        ┌────────────────┐
        │   审查Agent    │ ← 最终质量把关
        └────────────────┘
```

### L4：Self-Evolving（自我进化级）

这是"AGI级别"的理想状态。Agent能从经验中学习，自动改进自己的策略。

**能力特征**：
- 能从成功/失败的经验中总结规律
- 自动优化提示词和执行策略
- 长期记忆——记住历史任务中的最佳方案

说实话，L4目前还主要存在于论文里。现实中的Agent产品大多在L2和L3之间徘徊。但这不妨碍我们理解整个演进方向——**Agent的能力等级，本质上是从"被动响应"到"主动执行"的跨越**。

---

## 四、Java生态Agent支持现状

作为Java程序员，你可能会问：Agent这么火，Java生态跟上了吗？

好消息：**跟上了，而且很实用。**

### 主流框架对比

| 框架 | 支持程度 | 特点 |
|------|---------|------|
| **LangChain4j** | ⭐⭐⭐⭐⭐ | 最全面，ReAct/Plan-Execute/多Agent都内置 |
| **Spring AI** | ⭐⭐⭐⭐ | Spring生态集成好，核心抽象清晰 |
| **AutoGen Java** | ⭐⭐⭐ | 微软的Multi-Agent框架，Java版在追赶 |
| **GraphRAG-SDK** | ⭐⭐⭐ | 专注知识图谱+Agent的结合场景 |

我个人推荐 **LangChain4j**，原因是：
1. API设计最符合Java习惯（Builder模式、流畅接口）
2. 内置了ReActAgent、ToolExecutionRequest等核心抽象
3. 社区活跃，文档齐全（Java生态里少见的文档质量）

当然，如果你想从头理解Agent的原理，本文会带你**不依赖任何框架**手写一个Agent。理解了底层，用什么框架都是降维打击。

---

## 五、一个最简Java Agent Demo

废话不多说，直接上代码。这是一个麻雀虽小五脏俱全的Agent——能感知输入、思考、调用工具、观察结果、循环执行。

```java
import java.util.*;
import java.util.function.Function;

/**
 * 最简Agent实现——让你直观感受Agent的"感知→思考→行动→观察"循环
 */
public class SimpleAgent {

    // ① 工具注册表：Agent能调用的所有工具
    private final Map<String, Function<String, String>> tools = new LinkedHashMap<>();

    // ② 注册工具
    public SimpleAgent registerTool(String name, String description, Function<String, String> handler) {
        tools.put(name, handler);
        return this;
    }

    // ③ 核心循环：思考 → 行动 → 观察
    public String execute(String userInput, int maxTurns) {
        System.out.println("👤 用户: " + userInput);
        String currentContext = userInput; // 当前上下文（包含历史信息）

        for (int turn = 1; turn <= maxTurns; turn++) {
            System.out.println("\n══════ 第" + turn + "轮 ══════");

            // 【思考】LLM决定下一步做什么
            String thought = think(currentContext);
            System.out.println("🧠 思考: " + thought);

            if (thought.startsWith("ANSWER:")) {
                // 任务完成，返回最终答案
                String answer = thought.substring(7).trim();
                System.out.println("✅ 最终答案: " + answer);
                return answer;
            }

            // 【行动】解析并调用工具
            ToolCall call = parseToolCall(thought);
            System.out.println("🔧 调用工具: " + call.toolName + "(" + call.input + ")");

            // 【观察】获取执行结果
            String observation = invokeTool(call);
            System.out.println("👁 观察结果: " + observation);

            // 将观察结果追加到上下文中，进入下一轮循环
            currentContext += "\n[工具 " + call.toolName + " 返回]: " + observation;
        }

        return "任务超时，未能完成。";
    }

    // 【思考】——实际项目中这里调用LLM，Demo中用规则模拟
    private String think(String context) {
        // 真实场景：调用 OpenAI/通义千问 等LLM，让模型根据上下文决定下一步
        // 这里简化：根据当前上下文做规则判断
        if (context.contains("[工具")) {
            // 已经有工具结果了，判断是否完成
            return "ANSWER: 已根据工具返回的结果完成了任务。";
        }
        // 首次进入，决定调用哪个工具
        if (context.contains("天气")) {
            return "TOOL: get_weather | 北京";
        } else if (context.contains("计算")) {
            return "TOOL: calculate | " + extractExpression(context);
        } else if (context.contains("搜索")) {
            return "TOOL: web_search | " + extractQuery(context);
        } else {
            return "ANSWER: 我的知识范围内，这个问题的答案是……";
        }
    }

    // 【解析工具调用】——从Thought中提取工具名和参数
    private ToolCall parseToolCall(String thought) {
        // 格式: "TOOL: 工具名 | 参数"
        String body = thought.substring(5).trim();
        int sep = body.indexOf("|");
        String toolName = sep > 0 ? body.substring(0, sep).trim() : body.trim();
        String input = sep > 0 ? body.substring(sep + 1).trim() : "";
        return new ToolCall(toolName, input);
    }

    // 【执行工具】——根据工具名调用对应的函数
    private String invokeTool(ToolCall call) {
        Function<String, String> tool = tools.get(call.toolName);
        if (tool == null) {
            return "错误：未找到工具 [" + call.toolName + "]";
        }
        try {
            return tool.apply(call.input);
        } catch (Exception e) {
            return "工具执行异常: " + e.getMessage();
        }
    }

    // 辅助方法
    private String extractExpression(String context) {
        return "1+1"; // 示例
    }

    private String extractQuery(String context) {
        return "default query"; // 示例
    }

    // 内部类：工具调用封装
    record ToolCall(String toolName, String input) {}

    // ═════════════════ 测试入口 ═════════════════
    public static void main(String[] args) {
        SimpleAgent agent = new SimpleAgent();

        // 注册工具：模拟天气查询
        agent.registerTool("get_weather", "获取城市天气",
                city -> city + "今天晴，25°C，适合出门！");

        // 注册工具：模拟计算器
        agent.registerTool("calculate", "数学计算",
                expr -> {
                    if ("1+1".equals(expr.trim())) return "2";
                    return "计算结果未知";
                });

        // 注册工具：模拟网络搜索
        agent.registerTool("web_search", "网络搜索",
                query -> "搜索到相关结果：AI Agent 是当前热门技术方向。");

        // 运行Agent
        agent.execute("北京天气怎么样？", 5);
    }
}
```

**运行结果**：

```
👤 用户: 北京天气怎么样？

══════ 第1轮 ══════
🧠 思考: TOOL: get_weather | 北京
🔧 调用工具: get_weather(北京)
👁 观察结果: 北京今天晴，25°C，适合出门！

══════ 第2轮 ══════
🧠 思考: ANSWER: 已根据工具返回的结果完成了任务。
✅ 最终答案: 已根据工具返回的结果完成了任务。
```

你可能会说："这不就是个函数调用吗？有什么高级的？"

别急，这个Demo只是让你看到**循环结构**。真正的Agent，每一步的"思考"是由LLM完成的——它自己决定调哪个工具、传什么参数、什么时候停下来。而在L1的Function Calling中，这些决策的"自由度"是受限的。

---

## 六、从Function Calling到Agent：一条光谱，而非一道墙

很多文章喜欢把Function Calling和Agent对立起来。但其实，它们不是两个东西，而是一条**能力光谱上的不同位置**：

```
Function Calling ────→ ReAct Agent ────→ Plan-Execute Agent ────→ Multi-Agent
   (L1)                  (L2)                (L2~L3)                (L3)
```

- **Function Calling**：LLM决定调哪个函数，然后人类决定下一步。**单步决策。**
- **ReAct Agent**：LLM自己循环，每一步观察结果后决定下一步。**多步自主决策。**
- **Plan-Execute Agent**：LLM先做计划再执行。**先规划后行动。**
- **Multi-Agent**：多个Agent协作。**团队作战。**

关键的区别不在于"能不能调工具"，而在于：

> **决策权在谁手里？**
>
> L1的工具调用，流程是程序员写的，LLM只是帮你"填空"。
> L2的Agent，流程是LLM自己决定的，程序员只提供工具和环境。

这个区别看起来微妙，但在实际场景中天差地别。你想，如果你的机器人能自主决定"先调这个工具、再调那个工具、发现不对换一个方案、最后把结果发邮件"，那它就不再是你的"工具"，而是你的"同事"了。

---

## 七、总结与系列预告

这篇文章我们建立了Agent的核心认知框架：

- **Agent = 感知 → 思考 → 行动 → 观察的循环**
- **四个能力等级**：Tool Calling(L1) → Planning(L2) → Multi-Agent(L3) → Self-Evolving(L4)
- **Java生态已就绪**：LangChain4j / Spring AI 都能胜任Agent开发
- **最简单的Agent只需一个while循环**，真正的难点在于"LLM如何做出正确的决策"

**但真正的挑战在于**：如何让LLM在每一步都做出正确的决策？如何防止Agent陷入死循环？如何让Agent自己纠错？

这就是下一篇的内容。

---

**下一篇预告**：《ReAct Agent 实战：Java 实现"思考-行动-观察"循环，一个会自我纠错的AI程序》——我们将完整实现一个ReAct Agent，包含工具集定义、循环控制、死循环检测，以及ReAct+RAG的组合方案。代码可直接运行，拿来即用！

---

> **关于作者**：Java资深研发，专注AI工程化落地。关注我，获取更多Java+AI的实战干货。
