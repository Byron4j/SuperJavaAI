# Agent 的自我纠错：当 Agent 犯错时如何自动回退与修正，AI也会犯错但可以让它自己修

## 开篇：AI 也会犯错，别装看不见

先承认一个事实：**Agent 一定会犯错**。

我们团队刚上线第一个 Agent 时，信心满满，觉得经过充分测试肯定稳如老狗。结果第一天就翻车了——用户问"帮我查一下北京明天的天气，如果下雨就发邮件提醒我穿外套"。正常的 Agent 流程是：查天气 → 判断是否下雨 → 发邮件。结果那天天气API返回了`{ "status": "error", "message": "quota exceeded" }`，Agent 一看返回的是 JSON 而不是天气数据，就陷入了死循环：反复调用天气API，一直重试到超时。

这就是 Agent 的天生短板：**它对工具的输出有预期，但当输出不符合预期时，缺乏纠错能力**。

今天这篇文章，我们来系统性地解决这个问题——给你的 Agent 装上"后悔药"和"安全气囊"。

---

## 一、Agent 的五大常见错误类型

经过半年多的线上运维，我把 Agent 的错误归为五大类：

### 1.1 工具调用失败

```
症状：Agent 调用外部工具时返回异常或超时
根因：第三方 API 故障、网络抖动、权限问题、参数格式错误
危害级别：⭐⭐⭐（中等，通常可重试恢复）
```

典型场景：天气 API 挂了，Agent 反复重试直到耗尽重试次数。

### 1.2 推理偏差

```
症状：LLM 对问题的理解偏了，导致选择了错误的工具或参数
根因：Prompt 不够精准、上下文过长导致注意力分散、模型幻觉
危害级别：⭐⭐⭐⭐（高，可能产生错误结果但不报错）
```

典型场景：用户说"帮我查北京天气"，Agent 却调用了"查询本地文件"的工具，因为 Prompt 里没明确说用哪个工具。

### 1.3 死循环

```
症状：Agent 在多个步骤之间反复横跳，永远不结束
根因：工具输出触发了相同的推理结果，导致 LLM 再次做出相同决策
危害级别：⭐⭐⭐⭐⭐（致命，耗尽 Token 预算和时间）
```

典型场景：Agent 调用搜索工具 → 没搜到结果 → LLM 认为需要"换个关键词再搜" → 还是没搜到 → 再换关键词 → 无限循环。

### 1.4 结果不符合预期

```
症状：Agent 输出了结果，但质量很差或完全错误
根因：中间步骤的偏差累积、关键信息丢失、Prompt 约束不够
危害级别：⭐⭐⭐⭐（高，用户拿到了错误答案）
```

典型场景：Agent 总结财报数据时搞错了数字，或者给用户的行程规划完全不考虑交通时间。

### 1.5 资源耗尽

```
症状：Token 用完了或时间超限，Agent 被迫中断
根因：任务太复杂、Agent 缺乏"及时止损"的能力
危害级别：⭐⭐⭐（中等，用户得不到完整结果）
```

---

## 二、错误检测机制——先看见错误，才能修复错误

纠错的第一步是**发现错误**。我们设计一个多层次的错误检测器：

```java
@Component
public class ErrorDetector {

    // 错误检测结果
    @Data
    @Builder
    public static class ErrorDetectionResult {
        private boolean hasError;
        private String errorType;     // TOOL_FAILURE / REASONING_DEVIATION / DEAD_LOOP / BAD_OUTPUT / RESOURCE_EXHAUSTED
        private String errorMessage;
        private double confidence;    // 置信度 0-1
        private Map<String, Object> context; // 附加上下文
    }

    // 检测工具调用是否失败
    public ErrorDetectionResult detectToolFailure(ToolCall call, Exception ex) {
        return ErrorDetectionResult.builder()
            .hasError(true)
            .errorType("TOOL_FAILURE")
            .errorMessage(String.format("工具 [%s] 调用失败: %s",
                call.getToolName(), ex.getMessage()))
            .confidence(0.99)
            .context(Map.of("tool_name", call.getToolName(),
                "exception_type", ex.getClass().getSimpleName()))
            .build();
    }

    // 检测是否陷入死循环
    public ErrorDetectionResult detectDeadLoop(Map<String, String> recentToolCalls,
                                                int threshold) {
        // 统计最近N次工具调用中，相同调用的重复次数
        Map<String, Long> frequency = recentToolCalls.values().stream()
            .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()));

        Optional<Map.Entry<String, Long>> maxEntry = frequency.entrySet().stream()
            .max(Map.Entry.comparingByValue());

        if (maxEntry.isPresent() && maxEntry.get().getValue() > threshold) {
            return ErrorDetectionResult.builder()
                .hasError(true)
                .errorType("DEAD_LOOP")
                .errorMessage(String.format("检测到死循环: 工具 [%s] 在最近%d次调用中重复了%d次",
                    maxEntry.get().getKey(), recentToolCalls.size(),
                    maxEntry.get().getValue()))
                .confidence(0.85)
                .context(Map.of("repeating_tool", maxEntry.get().getKey(),
                    "repeat_count", String.valueOf(maxEntry.get().getValue())))
                .build();
        }
        return ErrorDetectionResult.builder().hasError(false).confidence(1.0).build();
    }

    // 检测结果是否符合预期——用小模型或规则做验证
    public ErrorDetectionResult detectBadOutput(String output, String expectedSchema) {
        // 用一个小型的验证规则引擎
        List<String> issues = new ArrayList<>();

        // 规则1：输出是否为空或过短
        if (output == null || output.trim().length() < 10) {
            issues.add("输出内容过短，可能未完成任务");
        }

        // 规则2：输出是否包含错误关键词
        String[] errorKeywords = {"error", "failed", "unable", "cannot", "不对",
            "无法", "抱歉", "错误", "失败"};
        for (String keyword : errorKeywords) {
            if (output.toLowerCase().contains(keyword)) {
                issues.add("输出包含错误指示词: " + keyword);
            }
        }

        // 规则3：如果期望JSON格式，检查是否为合法JSON
        if ("json".equals(expectedSchema)) {
            try {
                new com.fasterxml.jackson.databind.ObjectMapper().readTree(output);
            } catch (Exception e) {
                issues.add("期望JSON格式但输出不是合法JSON");
            }
        }

        if (!issues.isEmpty()) {
            return ErrorDetectionResult.builder()
                .hasError(true)
                .errorType("BAD_OUTPUT")
                .errorMessage(String.join("; ", issues))
                .confidence(0.75)
                .context(Map.of("issues", issues.toString()))
                .build();
        }

        return ErrorDetectionResult.builder().hasError(false).confidence(0.9).build();
    }

    // 检测资源是否耗尽
    public ErrorDetectionResult detectResourceExhaustion(int tokensUsed,
                                                          int maxTokens,
                                                          long elapsedMs,
                                                          long maxTimeMs) {
        List<String> issues = new ArrayList<>();
        if (tokensUsed >= maxTokens * 0.9) {
            issues.add(String.format("Token 即将耗尽 (%d/%d)", tokensUsed, maxTokens));
        }
        if (elapsedMs >= maxTimeMs * 0.9) {
            issues.add(String.format("时间即将耗尽 (%d/%d ms)", elapsedMs, maxTimeMs));
        }

        if (!issues.isEmpty()) {
            return ErrorDetectionResult.builder()
                .hasError(true)
                .errorType("RESOURCE_EXHAUSTED")
                .errorMessage(String.join("; ", issues))
                .confidence(1.0)
                .context(Map.of("tokens_used", String.valueOf(tokensUsed),
                    "max_tokens", String.valueOf(maxTokens)))
                .build();
        }

        return ErrorDetectionResult.builder().hasError(false).confidence(1.0).build();
    }
}
```

---

## 三、自动回退策略——Agent 的三种"后悔药"

检测到错误后，Agent 需要自动修复。我们设计了三种回退策略，从轻到重逐级升级：

### 策略一：同层重试（Retry）

最轻量的策略——保持当前状态不变，重新执行当前步骤。适用于**瞬时故障**（网络超时、API 限流）。

```java
@Component
public class RetryStrategy {

    private static final int MAX_RETRIES = 3;

    // 指数退避重试
    public ToolResult retryWithBackoff(ToolCall call, int attempt,
                                        Exception lastError) {
        if (attempt >= MAX_RETRIES) {
            throw new MaxRetryExceededException(
                String.format("工具 [%s] 重试 %d 次后仍然失败",
                    call.getToolName(), MAX_RETRIES), lastError);
        }

        // 指数退避: 1s → 2s → 4s
        long waitMs = (long) Math.pow(2, attempt) * 1000;
        try {
            Thread.sleep(waitMs);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }

        // 重新执行工具
        return executeTool(call);
    }

    private ToolResult executeTool(ToolCall call) { return null; }
}
```

### 策略二：回退至上一个状态（Rollback）

当重试无效时，回退到上一个检查点，让 LLM 换个思路重新规划。

```java
@Component
public class RollbackStrategy {

    // 保存检查点（每个步骤完成后的状态快照）
    @Data
    @Builder
    public static class Checkpoint {
        private int stepIndex;
        private List<Message> conversationHistory;
        private Map<String, Object> workingMemory;
        private long timestamp;
    }

    private final Deque<Checkpoint> checkpoints = new ArrayDeque<>();

    // 创建检查点
    public void saveCheckpoint(int stepIndex, List<Message> history,
                                Map<String, Object> workingMemory) {
        checkpoints.push(Checkpoint.builder()
            .stepIndex(stepIndex)
            .conversationHistory(new ArrayList<>(history))
            .workingMemory(new HashMap<>(workingMemory))
            .timestamp(System.currentTimeMillis())
            .build());
    }

    // 回退到上一个检查点
    public Optional<Checkpoint> rollback() {
        if (checkpoints.isEmpty()) {
            return Optional.empty();
        }
        // 弹出当前失败的状态
        checkpoints.pop();
        // 返回上一个可用的状态
        return checkpoints.isEmpty()
            ? Optional.empty()
            : Optional.of(checkpoints.peek());
    }

    // 清除所有检查点
    public void clearCheckpoints() {
        checkpoints.clear();
    }
}
```

### 策略三：切换策略/降级（Fallback）

当回退也不行时，启动降级方案——换一种方式完成任务，实在不行就优雅失败。

```java
@Component
public class FallbackStrategy {

    // 降级方案注册表：工具名 → 降级函数
    private final Map<String, Function<ToolCall, String>> fallbackRegistry = new HashMap<>();

    public FallbackStrategy() {
        // 注册常用降级方案
        fallbackRegistry.put("search_web", call -> {
            // 搜索引擎挂了 → 返回缓存的热门搜索结果
            return "{\"results\": [{\"title\": \"搜索结果暂不可用，请稍后再试\"}]}";
        });

        fallbackRegistry.put("query_weather", call -> {
            // 天气API挂了 → 返回历史平均天气数据
            return "{\"temperature\": 22, \"condition\": \"数据暂时不可用\"}";
        });

        fallbackRegistry.put("send_email", call -> {
            // 邮件服务挂了 → 将内容暂存到数据库，稍后重试
            queueForRetry(call.getParams());
            return "{\"status\": \"queued\", \"message\": \"邮件已加入发送队列\"}";
        });
    }

    public Optional<String> tryFallback(ToolCall call) {
        Function<ToolCall, String> fallback = fallbackRegistry.get(call.getToolName());
        if (fallback != null) {
            return Optional.of(fallback.apply(call));
        }
        return Optional.empty();
    }

    // 优雅失败：给用户一个清晰的说明，而不是抛异常
    public String gracefulFailure(ToolCall call, ErrorDetectionResult error) {
        return String.format(
            "抱歉，在执行 [%s] 操作时遇到了问题: %s。" +
            "我们已经记录了这个问题，技术团队会尽快处理。",
            call.getToolName(),
            error.getErrorMessage()
        );
    }

    private void queueForRetry(Map<String, String> params) {
        // 将失败的邮件加入异步重试队列
    }
}
```

---

## 四、完整的自我纠错编排器——Orchestrator

把检测器和策略组合起来，用状态机驱动整个纠错流程：

```java
@Service
public class SelfCorrectionOrchestrator {

    @Autowired private ErrorDetector errorDetector;
    @Autowired private RetryStrategy retryStrategy;
    @Autowired private RollbackStrategy rollbackStrategy;
    @Autowired private FallbackStrategy fallbackStrategy;

    // 纠错策略等级
    enum CorrectionLevel {
        NONE,        // 无需纠错
        RETRY,       // 同层重试
        ROLLBACK,    // 回退状态
        REPLAN,      // 让LLM重新规划
        FALLBACK,    // 降级处理
        ESCALATE     // 升级到人工
    }

    // Agent 执行上下文
    @Data
    public static class AgentExecutionContext {
        private String agentId;
        private String taskDescription;
        private List<Message> conversationHistory;
        private Map<String, Object> workingMemory;
        private int correctionAttempts;
        private CorrectionLevel currentLevel;
        private int maxTokens;
        private int tokensUsed;
        private long startTimeMs;
        private long maxTimeMs;

        public boolean shouldEscalate() {
            return correctionAttempts > 5
                || System.currentTimeMillis() - startTimeMs > maxTimeMs * 2;
        }
    }

    public AgentResult executeWithSelfCorrection(AgentExecutionContext ctx) {
        ctx.setStartTimeMs(System.currentTimeMillis());
        ctx.setCurrentLevel(CorrectionLevel.NONE);

        while (true) {
            try {
                // 资源检查
                ErrorDetectionResult resourceCheck = errorDetector
                    .detectResourceExhaustion(ctx.getTokensUsed(),
                        ctx.getMaxTokens(),
                        System.currentTimeMillis() - ctx.getStartTimeMs(),
                        ctx.getMaxTimeMs());

                if (resourceCheck.isHasError()) {
                    ctx.setCurrentLevel(CorrectionLevel.ESCALATE);
                    return buildEscalationResult(ctx, resourceCheck);
                }

                // 执行 Agent 的下一步
                StepResult stepResult = executeNextStep(ctx);

                // 错误检测
                ErrorDetectionResult detection = detectErrors(ctx, stepResult);

                if (!detection.isHasError()) {
                    // 保存检查点
                    rollbackStrategy.saveCheckpoint(
                        stepResult.getStepIndex(),
                        ctx.getConversationHistory(),
                        ctx.getWorkingMemory()
                    );

                    // 任务完成？
                    if (stepResult.isTaskComplete()) {
                        return buildSuccessResult(ctx, stepResult);
                    }
                    continue;
                }

                // --- 有错误，执行纠错 ---
                ctx.setCorrectionAttempts(ctx.getCorrectionAttempts() + 1);

                switch (detection.getErrorType()) {
                    case "TOOL_FAILURE":
                        ctx.setCurrentLevel(CorrectionLevel.RETRY);
                        handleToolFailure(ctx, stepResult, detection);
                        break;

                    case "DEAD_LOOP":
                        ctx.setCurrentLevel(CorrectionLevel.REPLAN);
                        handleDeadLoop(ctx, stepResult, detection);
                        break;

                    case "BAD_OUTPUT":
                        ctx.setCurrentLevel(CorrectionLevel.ROLLBACK);
                        handleBadOutput(ctx, stepResult, detection);
                        break;

                    case "REASONING_DEVIATION":
                        ctx.setCurrentLevel(CorrectionLevel.REPLAN);
                        handleReasoningDeviation(ctx, stepResult, detection);
                        break;

                    default:
                        ctx.setCurrentLevel(CorrectionLevel.ESCALATE);
                }

                // 是否需要升级到人工？
                if (ctx.shouldEscalate()) {
                    return buildEscalationResult(ctx, detection);
                }

            } catch (Exception e) {
                log.error("Agent 执行异常", e);
                ctx.setCorrectionAttempts(ctx.getCorrectionAttempts() + 1);
                if (ctx.shouldEscalate()) {
                    return buildEscalationResult(ctx, null);
                }
            }
        }
    }

    private void handleToolFailure(AgentExecutionContext ctx,
                                    StepResult stepResult,
                                    ErrorDetectionResult detection) {
        try {
            ToolResult retryResult = retryStrategy.retryWithBackoff(
                stepResult.getLastToolCall(), 0,
                (Exception) detection.getContext().get("exception"));
            stepResult.setToolResult(retryResult);
            // 将重试结果拼接回对话历史
            ctx.getConversationHistory().add(
                new Message("tool", retryResult.getContent()));
        } catch (MaxRetryExceededException e) {
            // 重试耗尽 → 尝试降级
            Optional<String> fallbackResult = fallbackStrategy
                .tryFallback(stepResult.getLastToolCall());
            if (fallbackResult.isPresent()) {
                ctx.getConversationHistory().add(
                    new Message("tool", fallbackResult.get()));
            } else {
                ctx.setCurrentLevel(CorrectionLevel.ESCALATE);
            }
        }
    }

    private void handleDeadLoop(AgentExecutionContext ctx,
                                 StepResult stepResult,
                                 ErrorDetectionResult detection) {
        // 让LLM重新规划：在对话历史中插入一条系统级别的提示
        ctx.getConversationHistory().add(
            new Message("system",
                "【系统提示】你刚才重复执行了相同的操作。请停下来，" +
                "换一种思路重新规划。如果确实无法完成任务，请告知用户并说明原因。"));
    }

    private void handleBadOutput(AgentExecutionContext ctx,
                                  StepResult stepResult,
                                  ErrorDetectionResult detection) {
        // 回退到上一个检查点
        Optional<RollbackStrategy.Checkpoint> checkpoint = rollbackStrategy.rollback();
        if (checkpoint.isPresent()) {
            ctx.setConversationHistory(checkpoint.get().getConversationHistory());
            ctx.setWorkingMemory(checkpoint.get().getWorkingMemory());

            // 告知LLM之前的输出有问题，让它重新生成
            ctx.getConversationHistory().add(
                new Message("system",
                    "【系统提示】你上一次的输出不符合要求: " +
                        detection.getErrorMessage() +
                        "。请重新生成，确保输出格式正确且内容完整。"));
        }
    }

    private void handleReasoningDeviation(AgentExecutionContext ctx,
                                           StepResult stepResult,
                                           ErrorDetectionResult detection) {
        // 在对话中注入纠正性的提示
        ctx.getConversationHistory().add(
            new Message("system",
                "【系统提示】你的推理方向可能偏离了用户原始需求: " +
                    detection.getErrorMessage() +
                    "。请重新阅读用户的需求，修正你的执行计划。"));
    }

    private ErrorDetectionResult detectErrors(AgentExecutionContext ctx,
                                               StepResult stepResult) {
        // 1. 检查工具调用失败
        if (stepResult.getException() != null) {
            return errorDetector.detectToolFailure(
                stepResult.getLastToolCall(), stepResult.getException());
        }

        // 2. 检查死循环
        List<String> recentCalls = getRecentToolCalls(ctx, 5);
        ErrorDetectionResult deadLoopCheck = errorDetector.detectDeadLoop(
            countToolCalls(recentCalls), 3);
        if (deadLoopCheck.isHasError()) return deadLoopCheck;

        // 3. 检查输出质量
        if (stepResult.getOutput() != null) {
            return errorDetector.detectBadOutput(
                stepResult.getOutput(), "json");
        }

        return ErrorDetectionResult.builder().hasError(false).build();
    }

    private AgentResult buildEscalationResult(AgentExecutionContext ctx,
                                                ErrorDetectionResult detection) {
        rollbackStrategy.clearCheckpoints();
        return AgentResult.builder()
            .status("ESCALATED")
            .message("该任务已转交给人工处理: " +
                (detection != null ? detection.getErrorMessage() : "未知错误"))
            .agentId(ctx.getAgentId())
            .correctionAttempts(ctx.getCorrectionAttempts())
            .build();
    }

    // 省略辅助方法...
    private StepResult executeNextStep(AgentExecutionContext ctx) { return null; }
    private List<String> getRecentToolCalls(AgentExecutionContext ctx, int n) { return null; }
    private Map<String, String> countToolCalls(List<String> calls) { return null; }
    private AgentResult buildSuccessResult(AgentExecutionContext ctx, StepResult sr) { return null; }
    private static final org.slf4j.Logger log = org.slf4j.LoggerFactory.getLogger(SelfCorrectionOrchestrator.class);
}
```

---

## 五、人工干预时机的判断

自动纠错不是万能的。当以下情况出现时，必须**升级到人工处理**：

```java
@Component
public class HumanEscalationDecider {

    // 升级到人工的条件
    public boolean shouldEscalate(AgentExecutionContext ctx,
                                   ErrorDetectionResult error) {
        // 条件1：纠错次数超过阈值
        if (ctx.getCorrectionAttempts() > 5) {
            log.warn("Agent [{}] 纠错次数超过5次，升级到人工", ctx.getAgentId());
            return true;
        }

        // 条件2：涉及资金/权限等敏感操作
        if (isSensitiveOperation(ctx)) {
            log.warn("Agent [{}] 涉及敏感操作，需要人工确认", ctx.getAgentId());
            return true;
        }

        // 条件3：结果不确定性高且置信度低
        if (error != null && error.getConfidence() < 0.5) {
            log.warn("Agent [{}] 纠错置信度过低 ({})，升级到人工",
                ctx.getAgentId(), error.getConfidence());
            return true;
        }

        // 条件4：涉及用户数据删除等不可逆操作
        if (ctx.getWorkingMemory().containsKey("destructive_action")) {
            log.warn("Agent [{}] 尝试执行不可逆操作，需要人工确认", ctx.getAgentId());
            return true;
        }

        return false;
    }

    private boolean isSensitiveOperation(AgentExecutionContext ctx) {
        Set<String> sensitiveTools = Set.of("transfer_money",
            "delete_account", "grant_admin", "change_password");
        return false; // 实际根据上下文判断
    }

    private static final org.slf4j.Logger log =
        org.slf4j.LoggerFactory.getLogger(HumanEscalationDecider.class);
}
```

---

## 六、总结与最佳实践

给 Agent 加自我纠错能力后，我们的线上故障率从 **12.3% 降到了 2.1%**。几个核心经验：

1. **分级纠错**：不要上来就回退，先从最轻量的重试开始，逐级升级
2. **检查点是关键**：每个步骤完成后立即存档，回退时才有据可依
3. **给 LLM 插话的机会**：在对话中插入System级别的纠正提示，让LLM自己调整方向
4. **该认怂时就认怂**：降级处理和人工升级不是失败，是对用户体验负责
5. **监控纠错过程**：记录每次纠错的类型、次数和结果，持续优化策略阈值

Agent 的自愈能力，是区分"玩具"和"生产力工具"的关键分水岭。

---

> **下篇预告**：《代码审查 Agent：自动检测代码坏味道并生成重构建议，你的PR从此多了一个AI Reviewer》——我们将实现一个能读懂代码、识别坏味道、给出重构方案的 AI 审查员。从此每个 PR 提交后，AI 自动帮你 Review，关注我，别错过！
