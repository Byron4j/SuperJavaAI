# Agent 生命周期的监控与可视化：构建 Agent 执行仪表盘，Agent出问题别等用户投诉才知道

## 开篇：一个真实的故事

上周四凌晨两点，运维群炸了——线上Agent系统全部超时，用户疯狂投诉。我们翻了半天日志才发现：某个Agent在执行"查询天气"工具时，第三方API挂了，导致整个Agent一直重试直到超时。

更扎心的是，这个问题在发生后的**第47分钟**才被发现。如果我们有一个实时监控仪表盘，可能第一分钟就能触发告警，而不是等用户来告诉我们"你们的AI好像坏了"。

Agent 不是普通微服务——它有推理链路、有工具调用、有Token消耗、有不确定性的输出。传统的"QPS+内存+CPU"三板斧监控完全不够用。今天这篇文章，我带你从零搭建一套**Agent专属的监控与可视化体系**，让 Agent 的每一次"呼吸"都尽收眼底。

---

## 一、Agent 监控到底要监控什么？

先想清楚一个问题：**Agent 和普通微服务的本质区别是什么？**

普通微服务：接收请求 → 处理 → 返回结果，链路简单。  
AI Agent：接收请求 → 推理 → 决策 → 调用工具 → 再推理 → 再决策 → 输出，每个环节都可能出错。

所以 Agent 监控至少需要五个维度：

### 1.1 执行时间监控

不只是"总耗时"，要拆到每个阶段：

- **LLM推理耗时**：每次调用大模型花了多少时间
- **工具执行耗时**：每个外部工具的响应时间
- **总体端到端耗时**：从请求进来到输出结果的完整时间

### 1.2 Token 消耗监控

Agent 是烧钱的——Token 就是成本。你需要监控：

- 每次交互的 Token 消耗（输入Token + 输出Token）
- 不同模型的 Token 消耗分布（如果用了多种模型）
- Token 消耗趋势（按小时/天聚合）

### 1.3 工具调用监控

Agent 的能力边界由工具决定，工具挂了 Agent 就废了一半：

- 每个工具的被调用次数
- 工具调用成功率 / 失败率
- 工具响应时间 P50 / P95 / P99

### 1.4 成功率与错误率

- Agent 整体任务成功率
- 分类错误率（工具错误 vs LLM错误 vs 超时 vs 逻辑错误）
- 错误分布（哪个Agent、哪个步骤最容易出错）

### 1.5 并发与吞吐

- 正在执行的 Agent 实例数
- 每分钟完成任务数
- 队列积压情况

---

## 二、技术选型：Micrometer + Prometheus + Grafana

对于 Java 生态，这套组合拳是标准答案：

| 组件 | 角色 |
|------|------|
| **Micrometer** | Java应用内的指标采集门面，类似SLF4J之于日志 |
| **Prometheus** | 时序数据库，拉取并存储指标数据 |
| **Grafana** | 可视化仪表盘，从Prometheus读取数据展示 |

架构很简单：Agent 应用 → Micrometer 暴露 `/actuator/prometheus` → Prometheus 定时拉取 → Grafana 查询展示。

---

## 三、核心代码实现

### 3.1 引入依赖

```java
// pom.xml 关键依赖
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    implementation 'io.micrometer:micrometer-registry-prometheus'
    implementation 'io.micrometer:micrometer-core'
}
```

### 3.2 自定义 Agent 监控指标

Micrometer 的核心抽象是 `MeterRegistry`，我们要定义一套专属指标：

```java
@Component
public class AgentMetrics {

    private final MeterRegistry registry;

    // Agent执行计数器 - 按结果状态分类
    private final Counter agentExecutionTotal;
    private final Counter agentExecutionSuccess;
    private final Counter agentExecutionFailure;

    // Agent执行时间 - 直方图
    private final Timer agentExecutionTimer;
    private final Timer llmCallTimer;
    private final Timer toolCallTimer;

    // Token消耗
    private final Counter tokenInputCounter;
    private final Counter tokenOutputCounter;

    // 工具调用
    private final Counter toolCallTotal;
    private final Counter toolCallFailure;
    private final Timer toolCallDuration;

    // 当前执行中的Agent数
    private final AtomicInteger activeAgents;

    public AgentMetrics(MeterRegistry registry) {
        this.registry = registry;
        this.activeAgents = new AtomicInteger(0);

        // 定义指标名称和标签
        this.agentExecutionTotal = Counter.builder("agent.execution.total")
            .description("Agent 执行总数")
            .tag("app", "agent-platform")
            .register(registry);

        this.agentExecutionSuccess = Counter.builder("agent.execution.success")
            .description("Agent 执行成功数")
            .register(registry);

        this.agentExecutionFailure = Counter.builder("agent.execution.failure")
            .description("Agent 执行失败数")
            .register(registry);

        this.agentExecutionTimer = Timer.builder("agent.execution.duration")
            .description("Agent 端到端执行时间")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);

        this.llmCallTimer = Timer.builder("agent.llm.call.duration")
            .description("LLM 调用耗时")
            .register(registry);

        this.toolCallTimer = Timer.builder("agent.tool.call.duration")
            .description("工具调用耗时")
            .register(registry);

        this.tokenInputCounter = Counter.builder("agent.token.input")
            .description("输入 Token 总数")
            .register(registry);

        this.tokenOutputCounter = Counter.builder("agent.token.output")
            .description("输出 Token 总数")
            .register(registry);

        this.toolCallTotal = Counter.builder("agent.tool.call.total")
            .description("工具调用总次数")
            .register(registry);

        this.toolCallFailure = Counter.builder("agent.tool.call.failure")
            .description("工具调用失败次数")
            .register(registry);

        this.toolCallDuration = Timer.builder("agent.tool.call.duration")
            .description("工具调用耗时")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);
    }

    public void recordAgentExecution(String agentId, String status, long durationMs) {
        Tags tags = Tags.of("agent_id", agentId, "status", status);
        agentExecutionTotal.increment();
        if ("success".equals(status)) {
            agentExecutionSuccess.increment();
        } else {
            agentExecutionFailure.increment();
        }
        agentExecutionTimer.record(durationMs, TimeUnit.MILLISECONDS);
    }

    public void recordLlmCall(String model, long durationMs, int inputTokens, int outputTokens) {
        llmCallTimer.record(durationMs, TimeUnit.MILLISECONDS);
        tokenInputCounter.increment(inputTokens);
        tokenOutputCounter.increment(outputTokens);
    }

    public void recordToolCall(String toolName, boolean success, long durationMs) {
        toolCallTotal.increment();
        if (!success) {
            toolCallFailure.increment();
        }
        toolCallDuration.record(durationMs, TimeUnit.MILLISECONDS);
    }

    public void incrementActiveAgents() {
        activeAgents.incrementAndGet();
        registry.gauge("agent.active.count", activeAgents);
    }

    public void decrementActiveAgents() {
        activeAgents.decrementAndGet();
    }
}
```

### 3.3 在 Agent 执行器中埋点

有了指标定义，接下来在每个关键节点埋点：

```java
@Service
public class AgentExecutor {

    @Autowired
    private AgentMetrics agentMetrics;

    public AgentResult execute(AgentContext context) {
        long startTime = System.currentTimeMillis();
        String agentId = context.getAgentId();
        agentMetrics.incrementActiveAgents();

        try {
            // 第一阶段：LLM 推理
            long llmStart = System.currentTimeMillis();
            LlmResponse llmResponse = callLlm(context.getMessages());
            agentMetrics.recordLlmCall(
                context.getModel(),
                System.currentTimeMillis() - llmStart,
                llmResponse.getInputTokens(),
                llmResponse.getOutputTokens()
            );

            // 第二阶段：工具调用
            List<ToolCall> toolCalls = llmResponse.getToolCalls();
            for (ToolCall toolCall : toolCalls) {
                long toolStart = System.currentTimeMillis();
                try {
                    executeTool(toolCall);
                    agentMetrics.recordToolCall(
                        toolCall.getToolName(),
                        true,
                        System.currentTimeMillis() - toolStart
                    );
                } catch (Exception e) {
                    agentMetrics.recordToolCall(
                        toolCall.getToolName(),
                        false,
                        System.currentTimeMillis() - toolStart
                    );
                    throw e;
                }
            }

            agentMetrics.recordAgentExecution(
                agentId,
                "success",
                System.currentTimeMillis() - startTime
            );
            return buildSuccessResult();

        } catch (Exception e) {
            agentMetrics.recordAgentExecution(
                agentId,
                "failure",
                System.currentTimeMillis() - startTime
            );
            return buildErrorResult(e);
        } finally {
            agentMetrics.decrementActiveAgents();
        }
    }

    private LlmResponse callLlm(List<Message> messages) {
        // 实际调用大模型...
        return null;
    }

    private void executeTool(ToolCall toolCall) {
        // 实际执行工具...
    }

    private AgentResult buildSuccessResult() { return null; }
    private AgentResult buildErrorResult(Exception e) { return null; }
}
```

---

## 四、Agent 执行轨迹追踪——每一步都留痕

监控告诉我们"哪里慢了、哪里错了"，但排查时还需要 **"Agent 到底想了什么"**。这就是执行轨迹追踪的核心价值。

### 4.1 追踪数据结构设计

```java
@Data
@Builder
public class AgentTrace {

    private String traceId;
    private String agentId;
    private String sessionId;

    // 轨迹步骤
    private List<TraceStep> steps;

    // 时间信息
    private long startTime;
    private long endTime;

    // 汇总
    private int totalSteps;
    private int totalTokens;
    private String finalStatus;
}

@Data
@Builder
public class TraceStep {

    private int stepIndex;
    private String stepType;  // LLM_CALL / TOOL_CALL / PLANNING / FINAL_OUTPUT

    // 输入输出
    private String input;     // 当前步骤的输入
    private String output;    // 当前步骤的输出
    private String thinking;  // LLM 的推理过程

    // 工具调用详情
    private String toolName;
    private Map<String, String> toolParams;

    // 性能数据
    private long durationMs;
    private int tokenUsed;

    // 状态
    private String status;   // SUCCESS / FAILED / RETRY
    private String errorMessage;
}
```

### 4.2 轨迹收集器实现

```java
@Component
public class AgentTraceCollector {

    private final ThreadLocal<AgentTrace> currentTrace = new ThreadLocal<>();

    public AgentTrace startTrace(String agentId, String sessionId) {
        AgentTrace trace = AgentTrace.builder()
            .traceId(UUID.randomUUID().toString())
            .agentId(agentId)
            .sessionId(sessionId)
            .startTime(System.currentTimeMillis())
            .steps(new ArrayList<>())
            .build();
        currentTrace.set(trace);
        return trace;
    }

    public void addStep(TraceStep step) {
        AgentTrace trace = currentTrace.get();
        if (trace == null) return;

        step.setStepIndex(trace.getSteps().size() + 1);
        trace.getSteps().add(step);
    }

    public AgentTrace finishTrace(String status) {
        AgentTrace trace = currentTrace.get();
        if (trace == null) return null;

        trace.setEndTime(System.currentTimeMillis());
        trace.setFinalStatus(status);
        trace.setTotalSteps(trace.getSteps().size());
        trace.setTotalTokens(
            trace.getSteps().stream()
                .mapToInt(TraceStep::getTokenUsed)
                .sum()
        );

        // 异步持久化到ES或数据库
        CompletableFuture.runAsync(() -> saveToElasticsearch(trace));

        currentTrace.remove();
        return trace;
    }

    private void saveToElasticsearch(AgentTrace trace) {
        // 存储到 Elasticsearch，方便后续检索和可视化
    }
}
```

### 4.3 在 Agent 执行中插入轨迹记录

```java
public AgentResult executeWithTrace(AgentContext context) {
    AgentTrace trace = traceCollector.startTrace(
        context.getAgentId(),
        context.getSessionId()
    );

    try {
        List<Message> messages = context.getMessages();

        for (int iteration = 0; iteration < context.getMaxIterations(); iteration++) {
            // 第一步：LLM 推理
            long llmStart = System.currentTimeMillis();
            LlmResponse llmResponse = callLlm(messages);

            traceCollector.addStep(TraceStep.builder()
                .stepType("LLM_CALL")
                .input(messages.toString())
                .output(llmResponse.getContent())
                .thinking(llmResponse.getReasoningContent())
                .durationMs(System.currentTimeMillis() - llmStart)
                .tokenUsed(llmResponse.getTotalTokens())
                .status("SUCCESS")
                .build());

            // 第二步：工具调用
            if (!llmResponse.getToolCalls().isEmpty()) {
                for (ToolCall tc : llmResponse.getToolCalls()) {
                    long toolStart = System.currentTimeMillis();
                    try {
                        String result = executeTool(tc);
                        traceCollector.addStep(TraceStep.builder()
                            .stepType("TOOL_CALL")
                            .toolName(tc.getToolName())
                            .toolParams(tc.getParams())
                            .output(result)
                            .durationMs(System.currentTimeMillis() - toolStart)
                            .status("SUCCESS")
                            .build());
                    } catch (Exception e) {
                        traceCollector.addStep(TraceStep.builder()
                            .stepType("TOOL_CALL")
                            .toolName(tc.getToolName())
                            .status("FAILED")
                            .errorMessage(e.getMessage())
                            .build());
                    }
                }
            } else {
                break; // 没有工具调用，任务完成
            }
        }
        return buildResult(traceCollector.finishTrace("SUCCESS"));
    } catch (Exception e) {
        traceCollector.finishTrace("FAILED");
        throw e;
    }
}
```

---

## 五、Grafana 仪表盘设计

有了数据，来设计仪表盘。一个实战级的 Agent Dashboard 应该包含以下几块：

### 面板 1：Agent 概览卡片

```
┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│  总执行次数   │ │  成功率      │ │  平均耗时    │ │  活跃Agent数  │
│   12,847     │ │   94.3%      │ │   3.2 s      │ │     42       │
│   ↑12% vs昨日 │ │  ↓0.5%      │ │  ↑0.3s      │ │   ↓5        │
└──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘
```

### 面板 2：Token 消耗趋势（折线图）

X轴：时间（按小时聚合），Y轴：Token数。两条曲线：输入Token（蓝色）+ 输出Token（绿色）。

### 面板 3：工具调用排行榜（柱状图）

横向柱状图，展示每个工具的被调用次数和失败率。比如：

```
search_database    ████████████████████  8,421 calls  (失败率 0.3%)
query_weather      ████████████          5,210 calls  (失败率 2.1%)
send_email         ██████               2,100 calls  (失败率 0.1%)
```

### 面板 4：错误分布饼图

按错误类型分类：`tool_timeout` (35%)、`llm_rate_limit` (25%)、`tool_exception` (20%)、`validation_error` (15%)、`unknown` (5%)。

### 面板 5：Agent 执行轨迹时间线

这是最酷的面板——直接用 Grafana 的 State Timeline 或者自定义插件展示单个 Agent 的执行步骤：

```
Trace ID: abc-123  Agent: weather-agent  状态: SUCCESS  总耗时: 4.2s

▌ LLM_CALL (1)      ████████████ 1.2s (2,340 tokens)
▌ TOOL:get_location ██████ 0.5s → SUCCESS
▌ LLM_CALL (2)      ██████████ 1.0s (1,800 tokens)
▌ TOOL:query_weather ████████ 0.8s → SUCCESS
▌ FINAL_OUTPUT      ██ 0.7s (890 tokens)
```

### Prometheus 查询示例

```promql
# Agent 成功率
sum(rate(agent_execution_success_total[5m])) 
/ sum(rate(agent_execution_total[5m])) * 100

# P95 延迟
histogram_quantile(0.95, 
  sum(rate(agent_execution_duration_seconds_bucket[5m])) by (le))

# Token 消耗速率
sum(rate(agent_token_input_total[1h])) + sum(rate(agent_token_output_total[1h]))

# 工具失败率 Top 5
topk(5, 
  sum(rate(agent_tool_call_failure_total[5m])) by (tool_name)
  / sum(rate(agent_tool_call_total[5m])) by (tool_name)
)
```

---

## 六、告警规则配置

有了监控，配好告警才能真正做到"别等用户投诉"：

```yaml
# prometheus-rules.yml
groups:
  - name: agent-alerts
    rules:
      - alert: AgentSuccessRateLow
        expr: |
          sum(rate(agent_execution_success_total[5m]))
          / sum(rate(agent_execution_total[5m])) < 0.85
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Agent 成功率低于 85%"
          description: "当前成功率 {{ $value | humanizePercentage }}, 请立即排查"

      - alert: AgentP95LatencyHigh
        expr: |
          histogram_quantile(0.95,
            sum(rate(agent_execution_duration_seconds_bucket[5m])) by (le)) > 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Agent P95 延迟超过 10 秒"

      - alert: ToolFailureRateHigh
        expr: |
          sum(rate(agent_tool_call_failure_total[5m])) by (tool_name)
          / sum(rate(agent_tool_call_total[5m])) by (tool_name) > 0.1
        for: 3m
        labels:
          severity: warning
        annotations:
          summary: "工具 {{ $labels.tool_name }} 失败率超过 10%"

      - alert: TokenCostAnomaly
        expr: |
          sum(rate(agent_token_input_total[1h])) > 1000000
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "Token 消耗异常，1小时内超过100万"
```

---

## 七、总结

把 Agent 监控体系搭好后，你会发现：

1. **出问题早知道**：成功率跌破阈值立马飞书/企微告警，不用等用户投诉
2. **成本可视化**：每天、每周、每月的 Token 消耗一目了然，方便做预算
3. **问题可追溯**：任何一个失败的 Agent 执行，都能通过 traceId 还原完整执行轨迹
4. **优化有方向**：哪个工具慢、哪个步骤Token多，数据说话

记住一句话：**没有监控的 Agent 系统就像开车没有仪表盘——开到哪算哪，出事了才知道。**

---

> **下篇预告**：《Agent 的自我纠错：当 Agent 犯错时如何自动回退与修正，AI也会犯错但可以让它自己修》——我们将深入探讨 Agent 的错误检测机制、自动回退策略，以及什么时候该让人工介入。让你的 Agent 学会"知错能改"，先别急着上线，关注我，下周见！
