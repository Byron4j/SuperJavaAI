# Plan-and-Execute Agent：先制定计划再逐步执行的分步式智能体，复杂任务这样分解AI才靠谱

> 上篇我们实现了ReAct Agent，它的问题是"走一步看一步"——遇到复杂任务容易跑偏。Plan-and-Execute Agent引入了"先规划、再执行"的策略，让AI像项目经理一样思考。

---

## 一、ReAct的困境：为什么"走一步看一步"会翻车？

回顾上一篇文章的ReAct Agent，它的工作方式是这样的：

```
思考→行动→观察→思考→行动→观察→思考→行动→观察→完成
```

这种**一步一回头**的模式在简单任务中游刃有余，但在复杂任务场景下会暴露出致命缺陷：

### 场景：生成一份竞品分析报告

用户问："帮我生成一份关于字节跳动AI产品的竞品分析报告，包含产品对比、价格分析、优劣势和未来趋势。"

ReAct Agent可能会这样跑：

```
第1轮: 搜索"字节跳动AI产品"           → 得到豆包、扣子、火山引擎...
第2轮: 搜索"豆包产品信息"              → 得到豆包的基本介绍
第3轮: 搜索"扣子产品信息"              → 得到扣子的基本介绍
第4轮: 搜索"豆包定价"                  → 获取价格信息
第5轮: ...（写到第8轮可能还没开始写报告）
```

问题出在哪？

1. **缺乏全局视角**：Agent不知道整体该搜索什么、搜完数据后还要写报告——它只关注"当下这步"
2. **效率低下**：实际上第1-3轮可以并行搜索，ReAct却只能串行
3. **容易遗忘目标**：搜索了好几轮后，Agent可能忘了自己的最终目的是"写报告"，开始漫无目的地搜下去
4. **Token浪费严重**：每一步都携带全部历史，10轮下来Token消耗惊人

**Plan-and-Execute模式就是为解决这些痛点而生的。**

---

## 二、Plan-and-Execute的核心思想：先谋后动

Plan-and-Execute（规划-执行）把Agent的工作拆成两个大阶段：

```
┌─────────────────────────────────────────────┐
│          Plan-and-Execute Agent             │
│                                             │
│  Phase 1: Planning（规划阶段）              │
│  ┌─────────────────────────────────────┐    │
│  │ LLM 分析任务 → 拆分子步骤 →         │    │
│  │ 确定依赖关系 → 生成执行计划         │    │
│  └─────────────────────────────────────┘    │
│               ↓                             │
│  Phase 2: Execution（执行阶段）             │
│  ┌─────────────────────────────────────┐    │
│  │ 按计划逐步执行 → 每步验证结果 →    │    │
│  │ 计划不合理时动态修正 → 最终汇总    │    │
│  └─────────────────────────────────────┘    │
│               ↓                             │
│  Phase 3: Verification（验证阶段）          │
│  ┌─────────────────────────────────────┐    │
│  │ 检查执行结果是否达标 →             │    │
│  │ 生成最终输出                        │    │
│  └─────────────────────────────────────┘    │
└─────────────────────────────────────────────┘
```

打个比方：

- **ReAct** 像是你走进一个陌生的商场找洗手间——每走几步就看看指示牌，决定下一段往哪走。
- **Plan-and-Execute** 像是你先看了一眼商场导览图，规划好路线，然后按计划走过去。

在简单场景下两者差别不大。但在复杂场景下，**花时间做计划的性价比远高于盲目行动**。

---

## 三、完整Java实现

下面我们完整实现一个Plan-and-Execute Agent，包含计划生成、分步执行、中间验证、动态修正四个核心能力。

### 3.1 数据结构定义

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.stream.Collectors;

/**
 * 原子步骤——计划中的最小执行单元
 */
class PlanStep {
    int stepNumber;               // 步骤编号
    String description;           // 自然语言描述：这一步要做什么
    String actionType;            // 动作类型：SEARCH / CALCULATE / GENERATE / VERIFY
    List<Integer> dependencies;   // 依赖的步骤编号（必须等这些步骤完成才能执行）
    StepStatus status;            // 状态：PENDING / RUNNING / COMPLETED / FAILED
    String result;                // 执行结果
    long startTime;               // 开始时间
    long endTime;                 // 结束时间

    enum StepStatus { PENDING, RUNNING, COMPLETED, FAILED }

    public PlanStep(int stepNumber, String description, String actionType,
                    List<Integer> dependencies) {
        this.stepNumber = stepNumber;
        this.description = description;
        this.actionType = actionType;
        this.dependencies = dependencies != null ? dependencies : List.of();
        this.status = StepStatus.PENDING;
    }
}

/**
 * 完整的执行计划
 */
class ExecutionPlan {
    String taskSummary;           // 任务摘要
    List<PlanStep> steps;         // 步骤列表（按依赖关系偏序排列）
    int maxParallelism;           // 最大并行度
    boolean needsRevision;        // 是否需要修正计划

    public ExecutionPlan(String taskSummary, List<PlanStep> steps) {
        this.taskSummary = taskSummary;
        this.steps = steps;
        this.needsRevision = false;
    }

    /** 获取所有依赖已满足、可以执行的步骤 */
    public List<PlanStep> getExecutableSteps() {
        return steps.stream()
            .filter(s -> s.status == PlanStep.StepStatus.PENDING)
            .filter(this::dependenciesMet)
            .collect(Collectors.toList());
    }

    private boolean dependenciesMet(PlanStep step) {
        return step.dependencies.stream()
            .allMatch(depId -> steps.stream()
                .filter(s -> s.stepNumber == depId)
                .allMatch(s -> s.status == PlanStep.StepStatus.COMPLETED));
    }

    /** 是否所有步骤都已完成 */
    public boolean isComplete() {
        return steps.stream()
            .allMatch(s -> s.status == PlanStep.StepStatus.COMPLETED);
    }

    /** 是否存在失败步骤（且无替代方案） */
    public boolean hasBlockingFailure() {
        return steps.stream()
            .anyMatch(s -> s.status == PlanStep.StepStatus.FAILED);
    }
}
```

### 3.2 核心引擎：Planner + Executor + Verifier

```java
/**
 * Plan-and-Execute Agent 完整实现
 *
 * 架构：Planner(计划者) → Executor(执行者) → Verifier(验证者)
 */
public class PlanExecuteAgent {

    private final LLMClient llm;
    private final ToolRegistry toolRegistry;
    private final ExecutorService parallelPool;

    // 配置参数
    private final int maxPlanSteps = 20;
    private final int maxParallelism = 5;
    private final boolean allowPlanRevision = true;

    public PlanExecuteAgent(LLMClient llm, ToolRegistry toolRegistry) {
        this.llm = llm;
        this.toolRegistry = toolRegistry;
        this.parallelPool = Executors.newFixedThreadPool(maxParallelism);
    }

    /**
     * 任务入口——完整的 Plan → Execute → Verify 三阶段
     */
    public String execute(String userTask) {
        System.out.println("══════ Plan-and-Execute Agent ══════");
        System.out.println("📋 任务: " + userTask + "\n");

        // ═══ Phase 1: PLAN（规划） ═══
        System.out.println("─── Phase 1: 生成执行计划 ───");
        ExecutionPlan plan = generatePlan(userTask);
        printPlan(plan);

        // ═══ Phase 2: EXECUTE（执行） ═══
        System.out.println("\n─── Phase 2: 按计划执行 ───");
        plan = executePlan(plan, userTask);

        // ═══ Phase 3: VERIFY（验证） ═══
        System.out.println("\n─── Phase 3: 验证与汇总 ───");
        String finalResult = verifyAndSummarize(plan, userTask);

        System.out.println("\n✅ 任务完成！");
        return finalResult;
    }

    // ═══════════════ Phase 1: 规划 ═══════════════

    /**
     * 让LLM生成执行计划
     */
    private ExecutionPlan generatePlan(String task) {
        String planPrompt = """
            你是一个任务规划专家。请分析以下任务，将其拆解为可执行的子步骤。

            任务: %s

            可用工具:
            %s

            请生成一个详细的执行计划，包含：
            1. 任务简要描述
            2. 步骤列表（每个步骤包含：步骤编号、描述、动作类型、依赖的前置步骤）

            动作类型分类：
            - SEARCH: 需要搜索/查询信息
            - CALCULATE: 需要计算/分析
            - GENERATE: 需要生成内容/代码
            - VERIFY: 需要验证结果

            输出格式：
            PLAN_SUMMARY: <任务摘要>
            STEP 1: <描述> | 类型: SEARCH | 依赖: 无
            STEP 2: <描述> | 类型: CALCULATE | 依赖: 1
            STEP 3: <描述> | 类型: GENERATE | 依赖: 1,2
            ...
            """.formatted(task, toolRegistry.getAllToolDescriptions());

        String llmResponse = llm.chat(planPrompt);
        return parsePlan(llmResponse);
    }

    /**
     * 解析LLM生成的计划文本
     */
    private ExecutionPlan parsePlan(String llmResponse) {
        String[] lines = llmResponse.split("\n");
        String summary = "";
        List<PlanStep> steps = new ArrayList<>();

        for (String line : lines) {
            line = line.trim();
            if (line.startsWith("PLAN_SUMMARY:")) {
                summary = line.substring("PLAN_SUMMARY:".length()).trim();
            } else if (line.startsWith("STEP")) {
                PlanStep step = parseStep(line);
                if (step != null) steps.add(step);
            }
        }

        if (steps.isEmpty()) {
            // 兜底：如果LLM没按格式输出，生成一个默认的单步计划
            steps.add(new PlanStep(1, "执行任务: " + summary, "GENERATE", List.of()));
        }

        return new ExecutionPlan(summary, steps);
    }

    private PlanStep parseStep(String line) {
        try {
            // 格式: STEP 1: 描述 | 类型: SEARCH | 依赖: 1,2
            int colonIdx = line.indexOf(":");
            String prefix = line.substring(0, colonIdx);
            int stepNum = Integer.parseInt(prefix.replace("STEP", "").trim());

            String body = line.substring(colonIdx + 1).trim();
            String description = body;
            String actionType = "GENERATE";
            List<Integer> deps = List.of();

            if (body.contains("|")) {
                String[] parts = body.split("\\|");
                description = parts[0].trim();
                for (int i = 1; i < parts.length; i++) {
                    String part = parts[i].trim();
                    if (part.startsWith("类型:") || part.startsWith("动作:")) {
                        actionType = part.substring(part.indexOf(":") + 1).trim();
                    } else if (part.startsWith("依赖:")) {
                        String depStr = part.substring(part.indexOf(":") + 1).trim();
                        if (!"无".equals(depStr) && !depStr.isEmpty()) {
                            deps = Arrays.stream(depStr.split(","))
                                .map(String::trim)
                                .map(Integer::parseInt)
                                .collect(Collectors.toList());
                        }
                    }
                }
            }

            return new PlanStep(stepNum, description, actionType, deps);
        } catch (Exception e) {
            return null; // 解析失败，跳过这一行
        }
    }

    // ═══════════════ Phase 2: 执行 ═══════════════

    /**
     * 按计划执行——支持并行 + 动态修正
     */
    private ExecutionPlan executePlan(ExecutionPlan plan, String originalTask) {
        int iteration = 0;
        int maxIterations = 50; // 防止无限循环的安全上限

        while (!plan.isComplete() && iteration < maxIterations) {
            iteration++;

            // 获取当前可以并行执行的步骤
            List<PlanStep> readySteps = plan.getExecutableSteps();
            if (readySteps.isEmpty() && !plan.isComplete()) {
                // 死锁检测：有未完成步骤但无就绪步骤
                System.out.println("⚠️ 检测到计划死锁！可能是依赖关系有问题。");
                if (allowPlanRevision) {
                    plan = revisePlan(plan, "死锁: 部分步骤的依赖无法满足");
                } else {
                    break;
                }
                continue;
            }

            // 并行或串行执行（超过1个就绪步骤且无共享依赖时并行）
            if (readySteps.size() > 1 && canParallelize(readySteps)) {
                System.out.println("⚡ 并行执行 " + readySteps.size() + " 个步骤...");
                executeParallel(readySteps);
            } else {
                for (PlanStep step : readySteps) {
                    executeStep(step, plan);
                }
            }
        }

        return plan;
    }

    /**
     * 执行单个步骤
     */
    private void executeStep(PlanStep step, ExecutionPlan plan) {
        step.status = PlanStep.StepStatus.RUNNING;
        step.startTime = System.currentTimeMillis();
        System.out.println("  ▶ STEP " + step.stepNumber + ": " + step.description);

        try {
            // 构建执行上下文：包含依赖步骤的结果
            String context = buildStepContext(step, plan);
            String prompt = buildStepPrompt(step, context);

            String result = llm.chat(prompt);
            step.result = result;
            step.status = PlanStep.StepStatus.COMPLETED;
            step.endTime = System.currentTimeMillis();

            System.out.println("    ✓ 完成 (耗时" + (step.endTime - step.startTime) + "ms)");
            System.out.println("    结果摘要: " + truncate(result, 100));
        } catch (Exception e) {
            step.status = PlanStep.StepStatus.FAILED;
            step.result = "执行失败: " + e.getMessage();
            System.out.println("    ✗ 失败: " + e.getMessage());

            // 【计划修正】步骤失败时，判断是否需要修正计划
            if (allowPlanRevision && isCriticalStep(step)) {
                plan = revisePlan(plan,
                    "步骤 " + step.stepNumber + " 执行失败: " + e.getMessage());
            }
        }
    }

    /**
     * 并行执行多个步骤
     */
    private void executeParallel(List<PlanStep> steps) {
        List<CompletableFuture<Void>> futures = steps.stream()
            .map(step -> CompletableFuture.runAsync(() -> {
                // 注意：这里不能用 executeStep 因为 plan 是共享的，需要线程安全
                // 实际项目中应该用 ConcurrentHashMap 或对 plan 加锁
                executeStep(step, null); // plan在并行场景下需额外处理
            }, parallelPool))
            .collect(Collectors.toList());

        try {
            CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
                .get(120, TimeUnit.SECONDS); // 总超时2分钟
        } catch (TimeoutException e) {
            System.out.println("⚠️ 并行执行超时，部分步骤可能未完成");
        } catch (Exception e) {
            System.out.println("⚠️ 并行执行异常: " + e.getMessage());
        }
    }

    /**
     * 判断步骤是否可并行（简单启发式）
     */
    private boolean canParallelize(List<PlanStep> steps) {
        // 如果所有步骤都没有互相依赖（它们的依赖都已经满足且不相互依赖）
        Set<Integer> stepNums = steps.stream().map(s -> s.stepNumber).collect(Collectors.toSet());
        long sharedDeps = steps.stream()
            .flatMap(s -> s.dependencies.stream())
            .filter(dep -> stepNums.contains(dep))
            .count();
        return sharedDeps == 0;
    }

    /**
     * 构建当前步骤的执行上下文（包含依赖步骤的结果）
     */
    private String buildStepContext(PlanStep step, ExecutionPlan plan) {
        if (plan == null || step.dependencies.isEmpty()) return "";
        StringBuilder ctx = new StringBuilder("前置步骤的执行结果：\n");
        for (int depId : step.dependencies) {
            plan.steps.stream()
                .filter(s -> s.stepNumber == depId)
                .findFirst()
                .ifPresent(s -> ctx.append("  [步骤").append(depId).append("]: ")
                    .append(s.result).append("\n"));
        }
        return ctx.toString();
    }

    /**
     * 构建步骤执行的提示词
     */
    private String buildStepPrompt(PlanStep step, String context) {
        return """
            请执行以下子任务：

            任务描述: %s
            动作类型: %s

            %s

            可使用的工具:
            %s

            请给出该步骤的执行结果。如果涉及工具调用，请明确指明。
            """.formatted(step.description, step.actionType, context,
                         toolRegistry.getAllToolDescriptions());
    }

    // ═══════════════ 计划修正 ═══════════════

    /**
     * 动态修正执行计划——当执行中发现计划不合理时
     */
    private ExecutionPlan revisePlan(ExecutionPlan plan, String reason) {
        System.out.println("\n🔄 计划修正触发: " + reason);

        // 收集已完成步骤的信息
        StringBuilder completedInfo = new StringBuilder();
        for (PlanStep s : plan.steps) {
            if (s.status == PlanStep.StepStatus.COMPLETED) {
                completedInfo.append("  [已完成] STEP ").append(s.stepNumber)
                    .append(": ").append(s.description)
                    .append(" → 结果: ").append(truncate(s.result, 100)).append("\n");
            } else if (s.status == PlanStep.StepStatus.FAILED) {
                completedInfo.append("  [已失败] STEP ").append(s.stepNumber)
                    .append(": ").append(s.description)
                    .append(" → 原因: ").append(s.result).append("\n");
            }
        }

        String revisePrompt = """
            执行计划遇到了问题，需要修正。

            原始计划摘要: %s
            当前进展:
            %s
            问题: %s

            请针对未完成的步骤，给出修正后的执行计划（只输出需要修改/新增的步骤）。
            输出格式:
            PLAN_SUMMARY: <更新后的任务摘要>
            STEP X: <描述> | 类型: <类型> | 依赖: <依赖步骤>
            ...
            """.formatted(plan.taskSummary, completedInfo.toString(), reason);

        String revisedPlanText = llm.chat(revisePrompt);

        // 合并原计划和新计划
        ExecutionPlan revised = parsePlan(revisedPlanText);

        // 将已完成的步骤保留，未完成的用新计划替换
        List<PlanStep> mergedSteps = new ArrayList<>();
        for (PlanStep s : plan.steps) {
            if (s.status == PlanStep.StepStatus.COMPLETED) {
                mergedSteps.add(s);
            }
        }
        // 加入新步骤，编号偏移
        int maxCompleted = mergedSteps.stream().mapToInt(s -> s.stepNumber).max().orElse(0);
        for (PlanStep s : revised.steps) {
            s.stepNumber += maxCompleted;
            s.dependencies = s.dependencies.stream()
                .map(d -> d + maxCompleted)
                .collect(Collectors.toList());
            mergedSteps.add(s);
        }

        plan = new ExecutionPlan(revised.taskSummary, mergedSteps);
        plan.needsRevision = false;
        System.out.println("  ✓ 计划已修正，剩余 " +
            mergedSteps.stream().filter(s -> s.status != PlanStep.StepStatus.COMPLETED).count()
            + " 个步骤\n");

        return plan;
    }

    private boolean isCriticalStep(PlanStep step) {
        // 如果有后续步骤依赖当前失败步骤，则是关键步骤
        // 这里简化处理，实际应检查plan
        return true;
    }

    // ═══════════════ Phase 3: 验证与汇总 ═══════════════

    private String verifyAndSummarize(ExecutionPlan plan, String originalTask) {
        // 收集所有步骤的结果
        StringBuilder allResults = new StringBuilder();
        for (PlanStep step : plan.steps) {
            allResults.append("【步骤").append(step.stepNumber).append(": ")
                       .append(step.description).append("】\n")
                       .append(step.result != null ? step.result : "未完成")
                       .append("\n\n");
        }

        String finalPrompt = """
            请基于以下各步骤的执行结果，生成最终的回答。

            原始任务: %s

            各步骤执行结果:
            %s

            请生成一份完整、结构化的最终输出。
            如果某些步骤未完成或失败，请说明并尽量基于已有信息给出最佳答案。
            """.formatted(originalTask, allResults.toString());

        return llm.chat(finalPrompt);
    }

    // ═══════════════ 工具方法 ═══════════════

    private void printPlan(ExecutionPlan plan) {
        System.out.println("📋 计划摘要: " + plan.taskSummary);
        for (PlanStep step : plan.steps) {
            String depStr = step.dependencies.isEmpty() ? "无" :
                step.dependencies.stream().map(String::valueOf).collect(Collectors.joining(","));
            System.out.printf("  STEP %d: %s [%s] (依赖: %s)%n",
                step.stepNumber, step.description, step.actionType, depStr);
        }
    }

    private String truncate(String text, int maxLen) {
        if (text == null) return "null";
        return text.length() > maxLen ? text.substring(0, maxLen) + "..." : text;
    }
}
```

### 3.3 LLM客户端接口（适配层）

```java
/**
 * LLM客户端抽象接口——解耦具体的LLM实现
 * 你可以对接OpenAI、DeepSeek、通义千问、智谱等任意模型
 */
public interface LLMClient {
    String chat(String prompt);

    /** 工厂方法：创建OpenAI兼容的客户端 */
    static LLMClient openAI(String apiKey, String model, String baseUrl) {
        return new OpenAILLMClient(apiKey, model,
            baseUrl != null ? baseUrl : "https://api.deepseek.com/v1");
    }
}

/** OpenAI兼容的LLM客户端实现 */
class OpenAILLMClient implements LLMClient {
    private final String apiKey;
    private final String model;
    private final String baseUrl;
    private final OkHttpClient httpClient;
    private final ObjectMapper objectMapper;

    public OpenAILLMClient(String apiKey, String model, String baseUrl) {
        this.apiKey = apiKey;
        this.model = model;
        this.baseUrl = baseUrl;
        this.httpClient = new OkHttpClient();
        this.objectMapper = new ObjectMapper();
    }

    @Override
    @SuppressWarnings("unchecked")
    public String chat(String prompt) {
        try {
            Map<String, Object> body = new LinkedHashMap<>();
            body.put("model", model);
            body.put("messages", List.of(
                Map.of("role", "user", "content", prompt)
            ));
            body.put("temperature", 0.0);

            String json = objectMapper.writeValueAsString(body);
            Request request = new Request.Builder()
                .url(baseUrl + "/chat/completions")
                .header("Authorization", "Bearer " + apiKey)
                .header("Content-Type", "application/json")
                .post(RequestBody.create(json, MediaType.parse("application/json")))
                .build();

            try (Response response = httpClient.newCall(request).execute()) {
                String respBody = response.body().string();
                Map<String, Object> respMap = objectMapper.readValue(respBody, Map.class);
                List<Map<String, Object>> choices =
                    (List<Map<String, Object>>) respMap.get("choices");
                if (choices != null && !choices.isEmpty()) {
                    Map<String, Object> msg =
                        (Map<String, Object>) choices.get(0).get("message");
                    return (String) msg.get("content");
                }
                return "LLM调用失败: " + respBody;
            }
        } catch (Exception e) {
            return "LLM调用异常: " + e.getMessage();
        }
    }
}
```

### 3.4 运行演示

```java
public class PlanExecuteAgentDemo {

    public static void main(String[] args) {
        // ① 创建LLM客户端
        LLMClient llm = LLMClient.openAI(
            System.getenv("DEEPSEEK_API_KEY"),
            "deepseek-chat",
            "https://api.deepseek.com"
        );

        // ② 注册工具
        ToolRegistry registry = new ToolRegistry()
            .register(new Tool("web_search", "互联网搜索",
                Map.of("query", "搜索关键词"),
                p -> "搜索结果: AI Agent是2026技术热点..."))
            .register(new Tool("get_weather", "天气查询",
                Map.of("city", "城市名"),
                p -> "北京晴32°C"))
            .register(new Tool("calculate", "数学计算",
                Map.of("expr", "表达式"),
                p -> "计算结果: 42"))
            .register(new Tool("file_writer", "文件写入",
                Map.of("path", "文件路径", "content", "文件内容"),
                p -> "文件已写入: " + p.get("path")));

        // ③ 创建Agent并执行
        PlanExecuteAgent agent = new PlanExecuteAgent(llm, registry);

        String result = agent.execute("""
            帮我做一份关于2026年AI Agent技术趋势的研究报告，
            包含：当前技术现状、主要框架对比、Java生态支持情况、
            以及未来半年趋势预测。最后将结果保存为 markdown 文件。
            """);

        System.out.println("\n══════ 最终输出 ══════");
        System.out.println(result);
    }
}
```

**运行效果（模拟）**：

```
══════ Plan-and-Execute Agent ══════
📋 任务: 帮我做一份关于2026年AI Agent技术趋势的研究报告...

─── Phase 1: 生成执行计划 ───
📋 计划摘要: 生成2026年AI Agent技术趋势研究报告
  STEP 1: 搜索AI Agent当前技术现状 [SEARCH] (依赖: 无)
  STEP 2: 搜索主流Agent框架对比信息 [SEARCH] (依赖: 无)
  STEP 3: 搜索Java生态Agent支持情况 [SEARCH] (依赖: 无)
  STEP 4: 搜索AI Agent未来趋势预测 [SEARCH] (依赖: 无)
  STEP 5: 综合分析收集到的信息 [CALCULATE] (依赖: 1,2,3,4)
  STEP 6: 生成研究报告markdown内容 [GENERATE] (依赖: 5)
  STEP 7: 保存报告到文件 [GENERATE] (依赖: 6)
  STEP 8: 验证报告完整性和质量 [VERIFY] (依赖: 7)

─── Phase 2: 按计划执行 ───
⚡ 并行执行 4 个步骤...
  ▶ STEP 1: 搜索AI Agent当前技术现状
    ✓ 完成 (耗时1523ms)
  ▶ STEP 2: 搜索主流Agent框架对比信息
    ✓ 完成 (耗时1301ms)
  ▶ STEP 3: 搜索Java生态Agent支持情况
    ✓ 完成 (耗时1412ms)
  ▶ STEP 4: 搜索AI Agent未来趋势预测
    ✓ 完成 (耗时1287ms)
  ▶ STEP 5: 综合分析收集到的信息
    ✓ 完成 (耗时2198ms)
  ▶ STEP 6: 生成研究报告markdown内容
    ✓ 完成 (耗时3567ms)
  ▶ STEP 7: 保存报告到文件
    ✗ 失败: 文件写入权限不足

🔄 计划修正触发: 步骤 7 执行失败: 文件写入权限不足
  ✓ 计划已修正，剩余 3 个步骤

  ▶ STEP 8: 尝试修改文件路径为临时目录 [GENERATE]
    ✓ 完成 (耗时467ms)
  ▶ STEP 9: 保存报告到临时目录 [GENERATE]
    ✓ 完成 (耗时312ms)
  ▶ STEP 10: 验证最终输出 [VERIFY]
    ✓ 完成 (耗时1567ms)

─── Phase 3: 验证与汇总 ───
✅ 任务完成！

══════ 最终输出 ══════
# 2026年AI Agent技术趋势研究报告
...
```

关键点：步骤1-4**并行执行**（都无依赖），步骤7失败后**自动修正计划**（切换路径重试），验证阶段**汇总所有步骤结果**生成最终报告。

---

## 四、计划修正：让Agent不死在半路上

Plan-and-Execute最大的优势之一是"计划可修正"。前面代码已经展示了基本机制，这里补充几个修正策略：

### 修正策略1：失败重试

```java
private static final int MAX_RETRIES = 3;

private void executeStepWithRetry(PlanStep step, ExecutionPlan plan) {
    for (int retry = 0; retry < MAX_RETRIES; retry++) {
        try {
            executeStep(step, plan);
            if (step.status == PlanStep.StepStatus.COMPLETED) return;
        } catch (Exception e) {
            System.out.println("    ↻ 重试 " + (retry + 1) + "/" + MAX_RETRIES);
            // 重试时可以微调步骤参数
            if (retry == 1) {
                step.description += " (请尝试不同的方法)";
            }
        }
    }
    step.status = PlanStep.StepStatus.FAILED;
}
```

### 修正策略2：替代方案生成

```java
/**
 * 当某个步骤连续失败时，让LLM生成替代方案
 */
private PlanStep generateAlternative(PlanStep failedStep) {
    String prompt = """
        原计划步骤执行失败：
        步骤: %s
        失败原因: %s

        请提供一个替代方案来完成相同的目标（可能是不同的工具、不同的方法、或拆分为多个更小的步骤）。
        输出格式: STEP: <替代方案描述>
        """.formatted(failedStep.description, failedStep.result);

    String alternative = llm.chat(prompt);
    return new PlanStep(failedStep.stepNumber + 100, // 新编号避免冲突
        "替代方案: " + alternative, failedStep.actionType, failedStep.dependencies);
}
```

### 修正策略3：目标降级

```java
/**
 * 当完全无法完成原始任务时，优雅降级
 */
private String gracefulDegradation(ExecutionPlan plan, String originalTask) {
    return llm.chat("""
        你是一个诚实且尽力的助手。原始任务是: %s

        虽然部分步骤未能完成，但已收集了以下信息:
        %s

        请基于已有信息给出你能给出的最佳回答。
        同时诚实说明哪些部分未能完成、为什么。
        """.formatted(originalTask, collectCompletedResults(plan)));
}
```

---

## 五、Plan-and-Execute vs ReAct：场景选型指南

两种模式没有绝对优劣，只有场景适配：

| 维度 | ReAct | Plan-and-Execute |
|------|-------|------------------|
| **适合任务** | 探索性强、答案不明确、需灵活应变 | 结构化强、目标明确、步骤可预判 |
| **典型场景** | 开放式问答、Debug、创意生成 | 报告生成、数据分析、代码项目管理 |
| **步骤可预测性** | 低（下一步取决于上一步结果） | 高（大部分步骤可预先规划） |
| **并行能力** | 弱（天然串行） | 强（无依赖步骤可并行） |
| **Token效率** | 较低（每轮携带全量历史） | 较高（计划复用、上下文精简） |
| **自我纠错** | 强（每轮观察后即时调整） | 中（需走修正机制） |
| **实现复杂度** | 简单 | 中等 |

**简单决策规则**：

- 如果你不知道这个任务需要几步、每步要做什么 → **ReAct**
- 如果你能大致列出任务的子步骤 → **Plan-and-Execute**
- 如果任务包含明显的可并行子任务 → **Plan-and-Execute**
- 如果你的预算紧张 → **Plan-and-Execute**（更省Token）

---

## 六、生产化建议

在生产中使用Plan-and-Execute Agent，建议考虑以下增强：

1. **计划缓存**：相似任务复用已有计划模板
2. **人工审批节点**：关键步骤执行前暂停让人类确认（Human-in-the-Loop）
3. **执行沙箱**：避免Agent的"行动"直接操作生产环境
4. **成本监控**：记录每步的Token消耗和执行时间
5. **结果审计**：保留完整的计划+执行轨迹用于事后分析

---

## 七、总结

Plan-and-Execute Agent通过"先规划、再执行"的策略，解决了ReAct在处理复杂任务时全局能力不足、并行性差的问题：

- **Planner生成结构化执行计划**（步骤拆分 + 依赖关系）
- **Executor按依赖顺序执行**（支持并行 + 动态修正）
- **Verifier汇总验证生成最终结果**

这套模式特别适合"目标明确、步骤可预判"的任务场景。

但Plan-and-Execute也有它的代价：规划阶段本身就消耗Token，且如果任务过程中变化太大（比如每一步都不可预判），提前规划反而成为浪费。

有没有一种方案，既能保持ReAct的推理质量，又能减少Token消耗和执行时间？有——**ReWOO**。

---

**下一篇预告**：《ReWOO Agent：将推理与执行解耦的更高效 Agent 架构，比ReAct更快更省Token》——ReWOO的核心思路是"提前把所有工具调用规划好，再批量执行"，将推理和执行解耦，大幅降低Token消耗。

---

> **关于作者**：Java资深研发，专注AI工程化落地。关注我，获取更多Java+AI的实战干货。
