# ReWOO Agent：将推理与执行解耦的更高效 Agent 架构，比ReAct更快更省Token

> 前两篇我们分别实现了ReAct和Plan-and-Execute。ReAct本质是"边想边做"，Plan-and-Execute是"先想好再做"。今天我们来玩一个更极致的架构——ReWOO，它将推理和执行彻底分离，省Token还更快。

---

## 一、ReAct的代价：你在为"交头接耳"付Token钱

回顾ReAct Agent的工作方式：

```
第1轮: Thought → Action → Observation (→ 携带全部历史进入第2轮)
第2轮: Thought → Action → Observation (→ 携带全部历史进入第3轮)
第3轮: Thought → Action → Observation (→ 携带全部历史进入第4轮)
...
```

每一轮LLM调用时，消息列表都在膨胀：

```
[系统提示词] (500 tokens)
[用户请求]   (100 tokens)
[第1轮思考]  (150 tokens)
[观察1]      (200 tokens)
[第2轮思考]  (180 tokens)
[观察2]      (350 tokens)
[第3轮思考]  (200 tokens)
[观察3]      (400 tokens)
─────────────────────────
总共已累积: 2080 tokens，而且还在涨！
```

**这就像一个开会老是翻旧账的同事**——每次发言前都要把前面所有人说过的话复述一遍。不光慢，而且贵。

计算结果：假设ReAct平均需要5轮，每轮平均500 token（思考+观察），加上系统提示词500 token。总Token消耗约为：

```
系统提示词: 500
第1轮: 1000 (500提示词+100请求+400 LLM输出)
第2轮: 500 (LLM输出) + 500 (输入包含第1轮) = 500
第3轮: 500 + 700 = 700
第4轮: 500 + 900 = 900
第5轮: 500 + 1100 = 1100
────────────────────────
总输入Token ≈ 4700
总输出Token ≈ 2500
总计 ≈ 7200 tokens
```

而如果有一种架构，只用**3次LLM调用**就能完成同样的事，且每次调用的上下文都保持精简……你觉得能省多少？

这就是ReWOO的切入点。

---

## 二、ReWOO的核心思想：把"想"和"做"彻底分开

ReWOO全称是 **Reasoning WithOut Observation**。出自2023年的论文《ReWOO: Decoupling Reasoning from Observations for Efficient Augmented Language Models》。

核心设计就一句话：

> **不等工具结果就规划好所有步骤，一次收集所有结果，最后批量求解。**

```
┌─────────────────────────────────────────────────────────┐
│                     ReWOO 三阶段架构                     │
│                                                         │
│  Phase 1: Planner（规划器）                              │
│  ┌───────────────────────────────────────────────┐      │
│  │  用户问: "北京和上海哪个更热？"                │      │
│  │                                                │      │
│  │  LLM 输出:                                     │      │
│  │  Plan: 查北京气温 → 查上海气温 → 对比         │      │
│  │  #E1 = get_weather[北京]                       │      │
│  │  #E2 = get_weather[上海]                       │      │
│  │  #E3 = compare[#E1, #E2]                       │      │
│  └───────────────────────────────────────────────┘      │
│               ↓                                         │
│  Phase 2: Worker（执行器）———— 无LLM，纯工具调用        │
│  ┌───────────────────────────────────────────────┐      │
│  │  #E1 → 执行 get_weather("北京") → "32°C"      │      │
│  │  #E2 → 执行 get_weather("上海") → "28°C"      │      │
│  └───────────────────────────────────────────────┘      │
│               ↓                                         │
│  Phase 3: Solver（求解器）                                │
│  ┌───────────────────────────────────────────────┐      │
│  │  输入: Plan + 所有结果                         │      │
│  │  #E1=32°C, #E2=28°C                           │      │
│  │  LLM输出: "北京更热，高4°C......"              │      │
│  └───────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────┘
```

和ReAct的本质差异：

| | ReAct | ReWOO |
|------|-------|-------|
| **LLM调用次数** | N次（每轮1次） | 2次（Plan + Solve） |
| **何时调工具** | 推理之后立刻调 | 全部规划完再批量调 |
| **LLM是否等待工具结果** | 是（每轮都要等） | 否（只在最后求解时看结果） |
| **上下文累积** | 线性增长（每轮增加） | 固定（Plan阶段不包含工具输出） |
| **并行度** | 天然串行 | Worker阶段可全并行 |

用一句话总结ReWOO的哲学：**不要让LLM等工具结果。让LLM专心推理，工具专心执行。**

---

## 三、完整Java实现

### 3.1 核心数据结构

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.function.Function;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

/**
 * ReWOO Agent 完整实现
 * 三阶段: Planner → Worker → Solver
 */
public class ReWOOAgent {

    private final LLMClient llm;
    private final ToolRegistry toolRegistry;
    private final int workerTimeoutSeconds;

    public ReWOOAgent(LLMClient llm, ToolRegistry toolRegistry) {
        this(llm, toolRegistry, 120);
    }

    public ReWOOAgent(LLMClient llm, ToolRegistry toolRegistry, int workerTimeoutSeconds) {
        this.llm = llm;
        this.toolRegistry = toolRegistry;
        this.workerTimeoutSeconds = workerTimeoutSeconds;
    }

    /**
     * 任务主入口
     */
    public String execute(String userQuery) {
        System.out.println("══════ ReWOO Agent ══════");
        System.out.println("👤 用户: " + userQuery + "\n");

        // ═══ Phase 1: Planner ═══
        System.out.println("─── Phase 1: Planner ───");
        Plan plan = generatePlan(userQuery);
        printPlan(plan);

        // ═══ Phase 2: Worker ═══
        System.out.println("\n─── Phase 2: Worker ───");
        Map<String, String> evidence = executePlan(plan);

        // ═══ Phase 3: Solver ═══
        System.out.println("\n─── Phase 3: Solver ───");
        String answer = solve(plan, evidence, userQuery);

        System.out.println("\n✅ 完成！");
        return answer;
    }

    // ═══════════════ Phase 1: Planner ═══════════════

    /**
     * 让LLM生成一个"推理链+工具调用计划"，但不实际执行任何工具。
     * 这是ReWOO最核心的区别——Planner阶段完全不调用工具。
     */
    private Plan generatePlan(String query) {
        String plannerPrompt = """
            你是一个推理规划器。你的任务是分析用户问题，生成一个推理与执行的计划。

            可用工具:
            %s

            输出格式要求（严格遵循）:
            Plan: <用简短自然语言描述解决思路>
            #E1 = <工具名>[参数1, 参数2, ...]
            #E2 = <工具名>[参数]
            #E3 = <工具名>[#E1, 参数]   ← 可以用 #E1 引用前面的结果
            ...

            规则:
            1. 以 Plan: 开头给出整体思路摘要
            2. 每个工具调用一行，格式: #E<序号> = <工具名>[参数]
            3. 参数用逗号分隔，如果参数依赖前面步骤的输出，用 #E<序号> 表示
            4. 考虑步骤间的依赖关系——能并行的尽量并行
            5. 最后一步通常是汇总和分析（用LLM而非工具完成）

            用户问题: %s
            """.formatted(toolRegistry.getAllToolDescriptions(), query);

        String plannerOutput = llm.chat(plannerPrompt);
        return parsePlan(plannerOutput);
    }

    /**
     * 解析Planner输出
     */
    private Plan parsePlan(String plannerOutput) {
        String planSummary = "";
        List<EvidenceCall> calls = new ArrayList<>();

        for (String line : plannerOutput.split("\n")) {
            line = line.trim();
            if (!line.isEmpty() && Character.isDigit(line.charAt(0))) {
                // 序列号包裹的纯文本，跳过
                continue;
            }

            if (line.startsWith("Plan:")) {
                planSummary = line.substring(5).trim();
            } else if (line.startsWith("#E")) {
                EvidenceCall call = parseEvidenceCall(line);
                if (call != null) {
                    calls.add(call);
                }
            }
        }

        return new Plan(planSummary, calls);
    }

    private EvidenceCall parseEvidenceCall(String line) {
        // 格式: #E1 = get_weather[北京, today]
        try {
            Pattern p = Pattern.compile("#E(\\d+)\\s*=\\s*(\\w+)\\[(.*)\\]");
            Matcher m = p.matcher(line.trim());
            if (m.matches()) {
                int id = Integer.parseInt(m.group(1));
                String toolName = m.group(2).trim();
                String rawArgs = m.group(3).trim();

                // 解析参数（简单按逗号分割，支持 #E<id> 引用）
                List<String> args = new ArrayList<>();
                if (!rawArgs.isEmpty()) {
                    // 简单逗号分割（实际项目中需要更robust的解析器）
                    for (String arg : rawArgs.split(",")) {
                        args.add(arg.trim());
                    }
                }

                return new EvidenceCall(id, toolName, args, null);
            }
        } catch (Exception e) {
            System.err.println("解析证据调用失败: " + line + " - " + e.getMessage());
        }
        return null;
    }

    // ═══════════════ Phase 2: Worker ═══════════════

    /**
     * 批量执行所有工具调用（可并行，因为Planner已经把依赖关系声明清楚）
     */
    private Map<String, String> executePlan(Plan plan) {
        Map<String, String> evidenceMap = new ConcurrentHashMap<>();
        ExecutorService pool = Executors.newFixedThreadPool(
            Math.min(plan.calls.size(), 10));

        // 构建依赖图——按依赖关系排序执行
        List<EvidenceCall> remaining = new ArrayList<>(plan.calls);
        int maxRounds = plan.calls.size() + 1; // 防止死锁
        int round = 0;

        while (!remaining.isEmpty() && round < maxRounds) {
            round++;
            List<EvidenceCall> ready = new ArrayList<>();

            // 找出所有依赖已满足的调用
            for (EvidenceCall call : remaining) {
                if (dependenciesResolved(call, evidenceMap)) {
                    ready.add(call);
                }
            }

            if (ready.isEmpty() && !remaining.isEmpty()) {
                System.err.println("⚠️ ReWOO Worker 死锁: 剩余调用存在循环依赖");
                break;
            }

            // 并行执行所有就绪的调用
            List<CompletableFuture<Void>> futures = new ArrayList<>();
            for (EvidenceCall call : ready) {
                futures.add(CompletableFuture.runAsync(() -> {
                    try {
                        String result = executeSingleCall(call, evidenceMap);
                        evidenceMap.put("#E" + call.id, result);
                        call.result = result;
                        System.out.println("  ✓ #E" + call.id + " = " +
                            call.toolName + " → " + truncate(result, 80));
                    } catch (Exception e) {
                        evidenceMap.put("#E" + call.id,
                            "Error: " + e.getMessage());
                        call.result = "Error: " + e.getMessage();
                        System.out.println("  ✗ #E" + call.id + " = " +
                            call.toolName + " → " + e.getMessage());
                    }
                }, pool));

                remaining.remove(call);
            }

            // 等待这一批完成
            try {
                CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
                    .get(workerTimeoutSeconds, TimeUnit.SECONDS);
            } catch (TimeoutException e) {
                System.err.println("⚠️ Worker 超时，部分调用可能未完成");
            } catch (Exception e) {
                System.err.println("⚠️ Worker 异常: " + e.getMessage());
            }
        }

        pool.shutdown();
        return evidenceMap;
    }

    /**
     * 检查某个调用的依赖是否都已解决
     */
    private boolean dependenciesResolved(EvidenceCall call, Map<String, String> evidence) {
        for (String arg : call.args) {
            if (arg.startsWith("#E")) {
                if (!evidence.containsKey(arg)) {
                    return false; // 依赖的 #E 还没有结果
                }
            }
        }
        return true;
    }

    /**
     * 执行一次工具调用，将 #E 引用替换为实际值
     */
    private String executeSingleCall(EvidenceCall call, Map<String, String> evidence) {
        Tool tool = toolRegistry.get(call.toolName);
        if (tool == null) {
            return "Error: 未找到工具 [" + call.toolName + "]";
        }

        // 将参数中的 #E<id> 替换为实际值
        Map<String, Object> resolvedArgs = new LinkedHashMap<>();
        int argIndex = 0;
        String[] paramNames = tool.parameters.keySet().toArray(new String[0]);

        for (String arg : call.args) {
            String resolvedArg = resolveRefs(arg, evidence);
            String paramName = argIndex < paramNames.length
                ? paramNames[argIndex] : "arg" + argIndex;
            resolvedArgs.put(paramName, resolvedArg);
            argIndex++;
        }

        return tool.execute(resolvedArgs);
    }

    /**
     * 递归解析参数中的引用（#E1 → 实际值）
     */
    private String resolveRefs(String input, Map<String, String> evidence) {
        String result = input;
        Pattern refPattern = Pattern.compile("#E\\d+");
        Matcher m = refPattern.matcher(result);
        while (m.find()) {
            String ref = m.group();
            if (evidence.containsKey(ref)) {
                result = result.replace(ref, evidence.get(ref));
            }
        }
        return result;
    }

    // ═══════════════ Phase 3: Solver ═══════════════

    /**
     * 把所有证据交给LLM，生成最终答案
     */
    private String solve(Plan plan, Map<String, String> evidence, String originalQuery) {
        StringBuilder evidenceBlock = new StringBuilder();
        for (EvidenceCall call : plan.calls) {
            evidenceBlock.append("#E").append(call.id)
                .append(" = ").append(call.toolName)
                .append(": ").append(call.result != null ? call.result : "未执行")
                .append("\n");
        }

        String solverPrompt = """
            你是一个问题求解器。你收到了一个规划方案和对应的执行证据。
            请基于这些信息回答原始问题。

            原始问题: %s

            规划思路: %s

            证据（工具执行结果）:
            %s

            请给出完整、准确、结构清晰的最终答案。
            如果某些证据缺失或出错，诚实说明并在已有信息基础上尽力回答。
            """.formatted(originalQuery, plan.summary, evidenceBlock.toString());

        return llm.chat(solverPrompt);
    }

    // ═══════════════ 辅助方法 ═══════════════

    private void printPlan(Plan plan) {
        System.out.println("📋 规划思路: " + plan.summary);
        for (EvidenceCall call : plan.calls) {
            System.out.printf("  #E%d = %s(%s)%n",
                call.id, call.toolName, String.join(", ", call.args));
        }
    }

    private String truncate(String text, int maxLen) {
        if (text == null) return "null";
        return text.length() > maxLen ? text.substring(0, maxLen) + "..." : text;
    }

    // ═══════════════ 内部数据类 ═══════════════

    static class Plan {
        String summary;
        List<EvidenceCall> calls;

        Plan(String summary, List<EvidenceCall> calls) {
            this.summary = summary;
            this.calls = calls;
        }
    }

    static class EvidenceCall {
        int id;
        String toolName;
        List<String> args;       // 可能包含 #E引用
        String result;           // 执行后填充

        EvidenceCall(int id, String toolName, List<String> args, String result) {
            this.id = id;
            this.toolName = toolName;
            this.args = args;
            this.result = result;
        }
    }
}
```

### 3.2 运行演示

```java
public class ReWOOAgentDemo {

    public static void main(String[] args) {
        // 创建LLM客户端
        LLMClient llm = LLMClient.openAI(
            System.getenv("DEEPSEEK_API_KEY"),
            "deepseek-chat",
            "https://api.deepseek.com"
        );

        // 注册工具
        ToolRegistry registry = new ToolRegistry()
            .register(new Tool("get_weather", "查询城市天气",
                Map.of("city", "城市名称"),
                p -> Map.of("北京", "32°C 晴", "上海", "28°C 阴",
                            "广州", "35°C 多云")
                    .getOrDefault(p.get("city"), "未知城市")))
            .register(new Tool("web_search", "网络搜索",
                Map.of("query", "搜索关键词"),
                p -> "搜索结果: 相关文章3篇..."))
            .register(new Tool("calculate", "数学计算",
                Map.of("expression", "数学表达式"),
                p -> "计算结果: 42"))
            .register(new Tool("get_population", "查询城市人口",
                Map.of("city", "城市名称"),
                p -> Map.of("北京", "2188万", "上海", "2487万")
                    .getOrDefault(p.get("city"), "未知")));

        // 创建Agent
        ReWOOAgent agent = new ReWOOAgent(llm, registry, 60);

        // 执行复杂查询
        String result = agent.execute("""
            比较北京和上海：哪个城市更热？哪个城市人口更多？
            帮我给出对比结论。
            """);

        System.out.println("\n══════ 最终答案 ══════");
        System.out.println(result);
    }
}
```

**运行效果（模拟）**：

```
══════ ReWOO Agent ══════
👤 用户: 比较北京和上海：哪个城市更热？哪个城市人口更多？

─── Phase 1: Planner ───
📋 规划思路: 分别查询北京和上海的温度和人口，然后对比分析
  #E1 = get_weather[北京]
  #E2 = get_weather[上海]
  #E3 = get_population[北京]
  #E4 = get_population[上海]

─── Phase 2: Worker ───
  ✓ #E1 = get_weather → 32°C 晴
  ✓ #E2 = get_weather → 28°C 阴
  ✓ #E3 = get_population → 2188万
  ✓ #E4 = get_population → 2487万

─── Phase 3: Solver ───

✅ 完成！

══════ 最终答案 ══════
对比结论：

温度方面：北京(32°C) > 上海(28°C)，北京比上海高4°C。
人口方面：上海(2487万) > 北京(2188万)，上海比北京多约299万人。

总结：北京更热，上海人更多。
```

**关键观察**：整个过程中LLM只被调用了**2次**（Planner 1次 + Solver 1次），工具调用4次全部是Worker阶段并行完成——没有LLM参与。而如果是ReAct做同样的事，至少需要4-5轮LLM调用（每轮一次工具调用）。

---

## 四、ReWOO vs ReAct vs Plan-and-Execute：量化对比

我们用一个真实场景做对比测试：**"帮我分析一下A公司、B公司、C公司三家AI创业公司的技术栈对比"**。

| 指标 | ReAct | Plan-and-Execute | ReWOO |
|------|-------|------------------|-------|
| **LLM调用次数** | 8次 | 5次 | 2次 |
| **总输入Token** | ~9,500 | ~5,200 | ~2,100 |
| **总输出Token** | ~4,800 | ~3,600 | ~1,400 |
| **总Token** | ~14,300 | ~8,800 | ~3,500 |
| **执行时间** | ~22s | ~18s | ~11s |
| **工具调用并行度** | 8次串行 | 3次并行 + 2次串行 | 3次全并行 |
| **准确率** | 高 | 高 | 中高 |
| **自我纠错能力** | 强 | 中 | 弱 |

> 数据为模拟估算，基于DeepSeek推理+10s/次网络调用假设。

**ReWOO在效率上碾压**，但注意最后一行的"自我纠错能力"——ReWOO是"弱"的，因为Plan阶段一旦出错，Worker照做不误，Solver拿到的可能就是垃圾数据。

---

## 五、适用场景分析：什么时候用ReWOO？

### 适合ReWOO的场景

1. **信息检索型任务**
   - 从多个数据源收集信息后汇总分析
   - 例："整理三家竞品的产品信息生成对比表"——先规划要查什么，查完统一汇总

2. **管道型任务**（前一步输出是后一步输入，但无分支决策）
   - 明确的线性依赖链
   - 例："查用户ID → 查下单记录 → 查商品详情 → 生成推荐"

3. **对成本敏感的场景**
   - 需要控制Token预算的生产系统
   - ReWOO比ReAct省40%-70%的Token

4. **实时性要求高的场景**
   - Worker阶段无LLM延迟（纯工具调用并行）
   - 适合做了很多工具调用的场景

### 适合ReAct的场景

1. **探索性任务**
   - 每一步的结果不可预测，必须"走一步看一步"
   - 例：Debug、代码审查、自由搜索

2. **需要自我纠错的场景**
   - 工具可能失败、需要重试或换方案
   - 例：操作外部系统（订票、发邮件）

3. **对话式交互**
   - 需要根据用户反馈实时调整
   - ReWOO的一次性规划不适合

### 决策树

```
任务是否可预判所有步骤？
├── 是 → 步骤间是否有大量分支决策？
│        ├── 否 → 【ReWOO】（效率最优）
│        └── 是 → 【Plan-and-Execute】（兼顾效率与灵活）
└── 否 → 【ReAct】（探索性强，灵活应变）
```

---

## 六、ReWOO的改进方向

ReWOO虽然效率高，但"Plan一旦出错、后面全错"是硬伤。以下是实际项目中的常见改进：

### 改进1：带验证的ReWOO+

```java
/**
 * ReWOO Plus：在Worker和Solver之间加入验证步骤
 */
public String executeWithVerification(String query) {
    // Phase 1: Planner
    Plan plan = generatePlan(query);

    // Phase 2: Worker
    Map<String, String> evidence = executePlan(plan);

    // Phase 2.5: 验证Worker结果（新增）
    boolean valid = verifyEvidence(plan, evidence, query);
    if (!valid) {
        // 回退到ReAct模式
        System.out.println("⚠️ ReWOO结果不可靠，切换ReAct...");
        return fallbackToReAct(query);
    }

    // Phase 3: Solver
    return solve(plan, evidence, query);
}

private boolean verifyEvidence(Plan plan, Map<String, String> evidence, String query) {
    // 快速检查：证据是否充足？是否有明显的错误？
    if (evidence.isEmpty()) return false;

    long errorCount = evidence.values().stream()
        .filter(v -> v.startsWith("Error:"))
        .count();
    if (errorCount > plan.calls.size() / 2) return false; // 超过半数出错

    return true;
}
```

### 改进2：增量式ReWOO

```java
/**
 * 增量ReWOO：分批规划，每N步验证一次
 */
public String executeIncremental(String query, int batchSize) {
    String currentQuery = query;
    List<Map<String, String>> allEvidence = new ArrayList<>();

    while (true) {
        // 为当前子任务生成计划（限制步数）
        Plan subPlan = generatePlan("完成以下子目标: " + currentQuery);

        // 执行
        Map<String, String> batchEvidence = executePlan(subPlan);
        allEvidence.add(batchEvidence);

        // 求解当前批次
        String partialResult = solve(subPlan, batchEvidence, currentQuery);

        // 让LLM判断是否完成
        if (llm.chat("基于以下结果，判断任务是否已完成。回答 YES 或 NO:\n" + partialResult)
                .contains("YES")) {
            return partialResult;
        }

        // 未完成：以当前结果为起点继续
        currentQuery = "继续任务。当前进展: " + partialResult;
    }
}
```

### 改进3：混合模式

```java
/**
 * 智能路由：Agent自动决定使用哪种模式
 */
public String executeSmart(String query) {
    // 让一个快速LLM判断任务类型
    String classification = llm.chat("""
        分类以下任务（只输出一个词）:
        - STRUCTURED: 步骤明确、可预判
        - EXPLORATORY: 步骤不明确、需探索
        任务: %s
        """.formatted(query));

    if (classification.contains("STRUCTURED")) {
        System.out.println("📊 检测到结构化任务 → 使用 ReWOO");
        return execute(query); // ReWOO
    } else {
        System.out.println("🔍 检测到探索性任务 → 使用 ReAct");
        return fallbackToReAct(query);
    }
}
```

---

## 七、总结与系列收官

这篇文章我们讨论了ReWOO（Reasoning WithOut Observation）这种极致高效的Agent架构：

- **三阶段架构**：Planner（推理规划）→ Worker（批量执行）→ Solver（最终求解）
- **核心洞见**：不要让LLM等待工具结果——推理和执行解耦
- **Token消耗极大降低**：2-3次LLM调用 vs ReAct的5-10次
- **适用场景**：信息检索、管道型任务、对成本/速度敏感的场景

至此，我们的AI Agent系列六完整收官。从L1的Function Calling，到L2的ReAct，再到L2-L3的Plan-and-Execute，最后以极致效率的ReWOO收尾。你应该已经对AI Agent有了一个完整的技术图景。

技术选型没有银弹。生产环境中，最好的Agent架构往往是**混合的**——根据任务特征动态选择ReAct、Plan-and-Execute或ReWOO。这正是下一阶段Multi-Agent架构要解决的命题。

---

**下一篇预告**：《Multi-Agent 多智能体实战：让多个Agent像团队一样协作，实现1+1>2的AI编排》——一个Agent的能力有上限，但一群Agent可以。我们将实现一个Orchestrator+多Worker的Multi-Agent系统，让Agent们各司其职、高效协作。

---

> **关于作者**：Java资深研发，专注AI工程化落地。关注我，获取更多Java+AI的实战干货。
