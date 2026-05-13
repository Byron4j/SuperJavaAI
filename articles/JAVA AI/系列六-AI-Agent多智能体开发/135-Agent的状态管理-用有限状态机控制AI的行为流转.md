# Agent 的状态管理：用有限状态机控制 AI 的行为流转，别让你的Agent进入"死循环"

## 前言

最近在做企业级 AI Agent 落地项目时，遇到了一个让我头疼的问题：Agent 在某次对话中反复调用同一个工具，就像卡在了某个状态里出不来。排查后发现是因为状态管理缺失 — Agent 根本不知道自己"现在该干什么"。

后来我用有限状态机（Finite State Machine, FSM）重构了 Agent 的核心流程引擎，问题迎刃而解。这篇文章就把我在生产环境中沉淀下来的 Agent 状态管理方案完整分享出来，希望能帮你避开那些坑。

## 一、Agent 为什么需要状态管理？

很多开发者刚开始做 Agent 时，代码是这样的：

```java
public String execute(String userInput) {
    String llmResponse = callLLM(userInput);
    if (llmResponse.contains("function_call")) {
        // 调用函数
        String toolResult = callTool(llmResponse);
        // 再给 LLM
        return callLLM(toolResult);
    }
    return llmResponse;
}
```

这段代码看似能跑，但稍微复杂一点的场景就会出问题：

### 1.1 死循环陷阱

最常见的翻车现场：Agent 调用工具后，LLM 返回的结果又触发了同一个工具调用，然后无限循环。比如：

> 用户：帮我写一个冒泡排序
> Agent：调用 write_code 工具 → LLM 觉得不够好 → 又重新调用 write_code → 又不够好 → 又调用… → 死循环

Token 疯狂消耗，OpenAI 账单比深圳房价涨得还快。

### 1.2 流程无法保证

真实业务场景的 Agent 执行链路往往是多步骤的：

```
接收任务 → 需求分析 → 方案设计 → 代码生成 → 测试验证 → 结果输出
```

没有状态管理，Agent 可能跳过"需求分析"直接写代码，或者"测试验证"失败后直接返回错误结果。这在企业级应用中是不能接受的 — 就像让一个没有流程管理的团队做项目，迟早翻车。

### 1.3 可恢复性的缺失

想象这个场景：Agent 执行到第4步"代码生成"，生成了 80% 的代码，突然因为网络波动导致 LLM 调用超时。没有状态持久化，你只能重启整个流程，之前的 3 步全白干了。

状态管理要解决的核心问题一句话概括：**让 Agent 知道"我是谁、我在哪、我要干什么"。**

## 二、有限状态机（FSM）设计

有限状态机在 Agent 场景下的核心三要素：**状态（State）、转换条件（Transition）、动作（Action）**。

### 2.1 状态定义

以我们项目中的"AI 编程助手" Agent 为例，定义以下状态：

```
IDLE          — 空闲，等待接收任务
ANALYZING     — 需求分析中
DESIGNING     — 方案设计中
CODING        — 代码生成中
TESTING       — 测试验证中
REVIEWING     — 人工审核中
FIXING        — 修复问题中
COMPLETED     — 任务完成
FAILED        — 任务失败
PAUSED        — 暂停等待外部输入
RECOVERING    — 断点恢复中
```

### 2.2 状态转换图

```
IDLE → ANALYZING → DESIGNING → CODING → TESTING → COMPLETED
  ↑        ↓           ↓          ↓         ↓
  └────────┴─── FAILED ←─────────┴─────────┘
                    ↓
                 RECOVERING → (恢复至失败前的状态)
```

关键原则：**每个状态只能向有限的方向转移**，杜绝"想怎么跳就怎么跳"的混乱局面。

### 2.3 动作触发

在状态转移时，可以绑定三种动作：

- **entryAction**：进入状态时触发（如进入 CODING 时创建代码文件）
- **exitAction**：离开状态时触发（如离开 CODING 时保存草稿）
- **duringAction**：状态持续时触发（如 ANALYZING 中持续输出分析进度）

## 三、Java 实现：状态枚举 + 转换表

### 3.1 状态枚举

```java
public enum AgentState {
    IDLE("空闲"),
    ANALYZING("需求分析中"),
    DESIGNING("方案设计中"),
    CODING("代码生成中"),
    TESTING("测试验证中"),
    REVIEWING("人工审核中"),
    FIXING("修复问题中"),
    COMPLETED("任务完成"),
    FAILED("任务失败"),
    PAUSED("暂停中"),
    RECOVERING("断点恢复中");

    private final String description;

    AgentState(String description) {
        this.description = description;
    }

    public String getDescription() {
        return description;
    }

    public boolean isTerminal() {
        return this == COMPLETED || this == FAILED;
    }

    public boolean isInterruptible() {
        return this == ANALYZING || this == DESIGNING
            || this == CODING || this == TESTING;
    }
}
```

### 3.2 状态转换规则定义

```java
public enum AgentEvent {
    START_ANALYSIS,
    ANALYSIS_COMPLETED,
    ANALYSIS_FAILED,
    START_DESIGN,
    DESIGN_COMPLETED,
    DESIGN_FAILED,
    START_CODING,
    CODING_COMPLETED,
    CODING_FAILED,
    START_TESTING,
    TESTING_PASSED,
    TESTING_FAILED,
    NEEDS_REVIEW,
    REVIEW_APPROVED,
    REVIEW_REJECTED,
    FIX_COMPLETED,
    PAUSE,
    RESUME,
    EXTERNAL_INPUT_RECEIVED,
    RECOVER,
    ABORT
}
```

```java
import java.util.*;

public class AgentStateMachine {
    // 状态转换表：当前状态 → (事件 → 目标状态)
    private final Map<AgentState, Map<AgentEvent, AgentState>> transitionTable;
    // 每个状态的 entry/exit 动作
    private final Map<AgentState, List<Runnable>> entryActions;
    private final Map<AgentState, List<Runnable>> exitActions;

    private AgentState currentState;

    public AgentStateMachine(AgentState initialState) {
        this.currentState = initialState;
        this.transitionTable = new EnumMap<>(AgentState.class);
        this.entryActions = new EnumMap<>(AgentState.class);
        this.exitActions = new EnumMap<>(AgentState.class);
        initTransitionTable();
    }

    private void initTransitionTable() {
        // IDLE 状态的转换
        addTransition(AgentState.IDLE, AgentEvent.START_ANALYSIS, AgentState.ANALYZING);
        addTransition(AgentState.IDLE, AgentEvent.RECOVER, AgentState.RECOVERING);
        addTransition(AgentState.IDLE, AgentEvent.ABORT, AgentState.FAILED);

        // ANALYZING 状态的转换
        addTransition(AgentState.ANALYZING, AgentEvent.ANALYSIS_COMPLETED, AgentState.DESIGNING);
        addTransition(AgentState.ANALYZING, AgentEvent.ANALYSIS_FAILED, AgentState.FAILED);
        addTransition(AgentState.ANALYZING, AgentEvent.PAUSE, AgentState.PAUSED);
        addTransition(AgentState.ANALYZING, AgentEvent.ABORT, AgentState.FAILED);

        // DESIGNING 状态的转换
        addTransition(AgentState.DESIGNING, AgentEvent.START_CODING, AgentState.CODING);
        addTransition(AgentState.DESIGNING, AgentEvent.DESIGN_COMPLETED, AgentState.CODING);
        addTransition(AgentState.DESIGNING, AgentEvent.DESIGN_FAILED, AgentState.FAILED);
        addTransition(AgentState.DESIGNING, AgentEvent.PAUSE, AgentState.PAUSED);

        // CODING 状态的转换
        addTransition(AgentState.CODING, AgentEvent.CODING_COMPLETED, AgentState.TESTING);
        addTransition(AgentState.CODING, AgentEvent.CODING_FAILED, AgentState.FAILED);
        addTransition(AgentState.CODING, AgentEvent.NEEDS_REVIEW, AgentState.REVIEWING);
        addTransition(AgentState.CODING, AgentEvent.PAUSE, AgentState.PAUSED);

        // TESTING 状态的转换
        addTransition(AgentState.TESTING, AgentEvent.TESTING_PASSED, AgentState.COMPLETED);
        addTransition(AgentState.TESTING, AgentEvent.TESTING_FAILED, AgentState.FIXING);
        addTransition(AgentState.TESTING, AgentEvent.PAUSE, AgentState.PAUSED);

        // REVIEWING 状态的转换
        addTransition(AgentState.REVIEWING, AgentEvent.REVIEW_APPROVED, AgentState.COMPLETED);
        addTransition(AgentState.REVIEWING, AgentEvent.REVIEW_REJECTED, AgentState.FIXING);

        // FIXING 状态的转换
        addTransition(AgentState.FIXING, AgentEvent.FIX_COMPLETED, AgentState.TESTING);
        addTransition(AgentState.FIXING, AgentEvent.ABORT, AgentState.FAILED);

        // PAUSED 状态的转换
        addTransition(AgentState.PAUSED, AgentEvent.RESUME, null); // 需要外部指定目标状态
        addTransition(AgentState.PAUSED, AgentEvent.ABORT, AgentState.FAILED);

        // RECOVERING 状态的转换
        addTransition(AgentState.RECOVERING, AgentEvent.RESUME, null);

        // FAILED 状态不可再转换（除非重置）
    }

    private void addTransition(AgentState from, AgentEvent event, AgentState to) {
        transitionTable
            .computeIfAbsent(from, k -> new EnumMap<>(AgentEvent.class))
            .put(event, to);
    }

    /**
     * 尝试状态转换，不合法则抛出异常
     */
    public synchronized StateTransitionResult transit(AgentEvent event) {
        Map<AgentEvent, AgentState> validTransitions = transitionTable.get(currentState);
        if (validTransitions == null || !validTransitions.containsKey(event)) {
            throw new InvalidStateTransitionException(
                String.format("非法状态转换: [%s] --%s--> ???",
                    currentState.getDescription(), event)
            );
        }

        AgentState targetState = validTransitions.get(event);
        AgentState fromState = this.currentState;

        // 执行 exit 动作
        executeActions(exitActions.get(currentState));

        // 切换到目标状态
        if (targetState != null) {
            this.currentState = targetState;
        }

        // 执行 entry 动作
        executeActions(entryActions.get(currentState));

        return new StateTransitionResult(fromState, currentState, event);
    }

    /**
     * 带目标状态的转换（用于 RECOVER/RESUME 场景）
     */
    public synchronized StateTransitionResult transitWithTarget(AgentEvent event, AgentState targetState) {
        AgentState fromState = this.currentState;
        executeActions(exitActions.get(currentState));
        this.currentState = targetState;
        executeActions(entryActions.get(currentState));
        return new StateTransitionResult(fromState, currentState, event);
    }

    private void executeActions(List<Runnable> actions) {
        if (actions != null) {
            actions.forEach(Runnable::run);
        }
    }

    public void registerEntryAction(AgentState state, Runnable action) {
        entryActions.computeIfAbsent(state, k -> new ArrayList<>()).add(action);
    }

    public void registerExitAction(AgentState state, Runnable action) {
        exitActions.computeIfAbsent(state, k -> new ArrayList<>()).add(action);
    }

    public AgentState getCurrentState() {
        return currentState;
    }

    public boolean canTransit(AgentEvent event) {
        Map<AgentEvent, AgentState> validTransitions = transitionTable.get(currentState);
        return validTransitions != null && validTransitions.containsKey(event);
    }

    /**
     * 获取当前状态的所有合法事件
     */
    public Set<AgentEvent> getAvailableEvents() {
        Map<AgentEvent, AgentState> transitions = transitionTable.get(currentState);
        return transitions != null ? transitions.keySet() : Collections.emptySet();
    }
}
```

### 3.3 状态转换结果

```java
public class StateTransitionResult {
    private final AgentState fromState;
    private final AgentState toState;
    private final AgentEvent event;
    private final long timestamp;

    public StateTransitionResult(AgentState fromState, AgentState toState, AgentEvent event) {
        this.fromState = fromState;
        this.toState = toState;
        this.event = event;
        this.timestamp = System.currentTimeMillis();
    }

    public AgentState getFromState() { return fromState; }
    public AgentState getToState() { return toState; }
    public AgentEvent getEvent() { return event; }
    public long getTimestamp() { return timestamp; }

    @Override
    public String toString() {
        return String.format("[%tT] %s --(%s)--> %s",
            timestamp, fromState.getDescription(), event, toState.getDescription());
    }
}
```

### 3.4 自定义异常

```java
public class InvalidStateTransitionException extends RuntimeException {
    public InvalidStateTransitionException(String message) {
        super(message);
    }
}
```

## 四、持久化状态：Redis 实现断点恢复

状态机本身只能管"当前进程内"的状态流转。生产环境中 Agent 随时可能重启、扩容、宕机，所以必须把状态持久化到外部存储。

### 4.1 Agent 上下文模型

```java
import java.io.Serializable;
import java.util.List;
import java.util.ArrayList;
import java.time.Instant;

public class AgentContext implements Serializable {
    private static final long serialVersionUID = 1L;

    private String sessionId;
    private AgentState currentState;
    private String currentTask;
    private List<StateTransitionResult> transitionHistory;
    private String checkpointData;   // 断点数据（JSON）
    private long createdAt;
    private long updatedAt;
    private int retryCount;
    private int maxRetries;

    public AgentContext() {
        this.transitionHistory = new ArrayList<>();
        this.createdAt = System.currentTimeMillis();
        this.retryCount = 0;
        this.maxRetries = 3;
    }

    // Getters & Setters
    public String getSessionId() { return sessionId; }
    public void setSessionId(String sessionId) { this.sessionId = sessionId; }

    public AgentState getCurrentState() { return currentState; }
    public void setCurrentState(AgentState currentState) {
        this.currentState = currentState;
        this.updatedAt = System.currentTimeMillis();
    }

    public String getCurrentTask() { return currentTask; }
    public void setCurrentTask(String currentTask) { this.currentTask = currentTask; }

    public List<StateTransitionResult> getTransitionHistory() { return transitionHistory; }

    public String getCheckpointData() { return checkpointData; }
    public void setCheckpointData(String checkpointData) { this.checkpointData = checkpointData; }

    public long getCreatedAt() { return createdAt; }
    public long getUpdatedAt() { return updatedAt; }

    public int getRetryCount() { return retryCount; }
    public void incrementRetry() { this.retryCount++; }

    public boolean canRetry() { return retryCount < maxRetries; }

    /**
     * 获取失败前最后一个可恢复的状态
     */
    public AgentState getLastRecoverableState() {
        for (int i = transitionHistory.size() - 1; i >= 0; i--) {
            AgentState state = transitionHistory.get(i).getToState();
            if (state.isInterruptible()) {
                return state;
            }
        }
        return AgentState.IDLE;
    }
}
```

### 4.2 Redis 状态持久化服务

```java
import redis.clients.jedis.Jedis;
import redis.clients.jedis.JedisPool;
import com.fasterxml.jackson.databind.ObjectMapper;

public class RedisAgentStateStore {
    private static final String KEY_PREFIX = "agent:state:";
    private static final int TTL_SECONDS = 86400; // 24小时

    private final JedisPool jedisPool;
    private final ObjectMapper objectMapper;

    public RedisAgentStateStore(JedisPool jedisPool) {
        this.jedisPool = jedisPool;
        this.objectMapper = new ObjectMapper();
    }

    /**
     * 保存 Agent 状态快照
     */
    public void saveState(AgentContext context) {
        try (Jedis jedis = jedisPool.getResource()) {
            String key = KEY_PREFIX + context.getSessionId();
            String json = objectMapper.writeValueAsString(context);
            jedis.setex(key, TTL_SECONDS, json);
        } catch (Exception e) {
            throw new StatePersistenceException("保存状态失败: " + context.getSessionId(), e);
        }
    }

    /**
     * 加载 Agent 状态快照
     */
    public AgentContext loadState(String sessionId) {
        try (Jedis jedis = jedisPool.getResource()) {
            String key = KEY_PREFIX + sessionId;
            String json = jedis.get(key);
            if (json == null) {
                return null;
            }
            return objectMapper.readValue(json, AgentContext.class);
        } catch (Exception e) {
            throw new StatePersistenceException("加载状态失败: " + sessionId, e);
        }
    }

    /**
     * 保存断点数据（在关键步骤前调用）
     */
    public void saveCheckpoint(String sessionId, AgentState currentState, String checkpointData) {
        try (Jedis jedis = jedisPool.getResource()) {
            String key = KEY_PREFIX + "checkpoint:" + sessionId;
            String data = objectMapper.writeValueAsString(
                Map.of("state", currentState.name(), "data", checkpointData)
            );
            jedis.setex(key, TTL_SECONDS, data);
        } catch (Exception e) {
            throw new StatePersistenceException("保存断点失败: " + sessionId, e);
        }
    }

    /**
     * 加载最近的断点数据
     */
    public Map<String, String> loadCheckpoint(String sessionId) {
        try (Jedis jedis = jedisPool.getResource()) {
            String key = KEY_PREFIX + "checkpoint:" + sessionId;
            String json = jedis.get(key);
            if (json == null) return null;
            return objectMapper.readValue(json, new TypeReference<Map<String, String>>() {});
        } catch (Exception e) {
            throw new StatePersistenceException("加载断点失败: " + sessionId, e);
        }
    }

    /**
     * 删除已完成会话的状态
     */
    public void clearState(String sessionId) {
        try (Jedis jedis = jedisPool.getResource()) {
            jedis.del(KEY_PREFIX + sessionId, KEY_PREFIX + "checkpoint:" + sessionId);
        } catch (Exception e) {
            // 清理失败不影响主流程
            System.err.println("清理状态失败: " + sessionId);
        }
    }
}
```

### 4.3 带持久化的完整 Agent

```java
import java.util.UUID;

public class StatefulAIAgent {
    private final AgentStateMachine stateMachine;
    private final RedisAgentStateStore stateStore;
    private final AgentContext context;
    private final LLMService llmService;

    public StatefulAIAgent(RedisAgentStateStore stateStore, LLMService llmService) {
        this.stateStore = stateStore;
        this.llmService = llmService;
        this.context = new AgentContext();
        this.context.setSessionId(UUID.randomUUID().toString());
        this.stateMachine = new AgentStateMachine(AgentState.IDLE);
        registerStateActions();
    }

    /**
     * 从断点恢复 Agent
     */
    public static StatefulAIAgent recover(String sessionId,
                                          RedisAgentStateStore stateStore,
                                          LLMService llmService) {
        AgentContext savedContext = stateStore.loadState(sessionId);
        if (savedContext == null) {
            throw new IllegalStateException("未找到会话状态: " + sessionId);
        }

        StatefulAIAgent agent = new StatefulAIAgent(stateStore, llmService);
        agent.restoreFromContext(savedContext);
        return agent;
    }

    private void restoreFromContext(AgentContext savedContext) {
        this.context.setSessionId(savedContext.getSessionId());
        this.context.setCheckpointData(savedContext.getCheckpointData());
        this.context.getTransitionHistory().addAll(savedContext.getTransitionHistory());
        this.stateMachine.transitWithTarget(AgentEvent.RECOVER, savedContext.getCurrentState());
    }

    /**
     * 执行 Agent 主流程
     */
    public void execute(String task) {
        context.setCurrentTask(task);

        try {
            // Step 1: 开始分析
            stateMachine.transit(AgentEvent.START_ANALYSIS);
            saveCheckpoint();
            String analysisResult = llmService.analyze(task);
            System.out.println("分析结果: " + analysisResult);

            // Step 2: 设计
            stateMachine.transit(AgentEvent.ANALYSIS_COMPLETED);
            saveCheckpoint();
            String designResult = llmService.design(analysisResult);
            System.out.println("设计方案: " + designResult);

            // Step 3: 编码
            stateMachine.transit(AgentEvent.DESIGN_COMPLETED);
            saveCheckpoint();
            String code = llmService.generateCode(designResult);
            System.out.println("生成代码: " + code);

            // Step 4: 测试
            stateMachine.transit(AgentEvent.CODING_COMPLETED);
            boolean testPassed = llmService.runTests(code);

            if (testPassed) {
                stateMachine.transit(AgentEvent.TESTING_PASSED);
                System.out.println("任务完成");
            } else {
                stateMachine.transit(AgentEvent.TESTING_FAILED);
                // 进入修复循环
                handleFailure(code);
            }

        } catch (Exception e) {
            System.err.println("执行异常: " + e.getMessage());
            context.incrementRetry();
            handleException(e);
        }

        // 最终持久化
        context.setCurrentState(stateMachine.getCurrentState());
        stateStore.saveState(context);

        // 如果任务完成，清理状态
        if (stateMachine.getCurrentState().isTerminal()) {
            stateStore.clearState(context.getSessionId());
        }
    }

    private void handleFailure(String code) {
        while (context.canRetry() &&
               stateMachine.getCurrentState() == AgentState.FIXING) {
            try {
                String fixedCode = llmService.fix(code);
                stateMachine.transit(AgentEvent.FIX_COMPLETED);
                saveCheckpoint();

                boolean retryPassed = llmService.runTests(fixedCode);
                if (retryPassed) {
                    stateMachine.transit(AgentEvent.TESTING_PASSED);
                    return;
                }
            } catch (Exception e) {
                context.incrementRetry();
                System.err.println("修复尝试 " + context.getRetryCount() + " 失败");
            }
        }
    }

    private void handleException(Exception e) {
        if (context.canRetry() && stateMachine.getCurrentState().isInterruptible()) {
            // 从断点恢复
            AgentState recoverableState = context.getLastRecoverableState();
            stateMachine.transitWithTarget(AgentEvent.RECOVER, recoverableState);
            System.out.println("从状态 " + recoverableState.getDescription() + " 恢复执行");
        } else {
            System.err.println("达到最大重试次数，任务终止");
        }
    }

    private void saveCheckpoint() {
        context.setCurrentState(stateMachine.getCurrentState());
        stateStore.saveState(context);
    }

    private void registerStateActions() {
        stateMachine.registerEntryAction(AgentState.CODING,
            () -> System.out.println("[FSM] 进入编码阶段，准备生成代码..."));
        stateMachine.registerExitAction(AgentState.CODING,
            () -> System.out.println("[FSM] 离开编码阶段，保存代码草稿..."));

        stateMachine.registerEntryAction(AgentState.TESTING,
            () -> System.out.println("[FSM] 进入测试阶段，执行测试用例..."));
        stateMachine.registerEntryAction(AgentState.FAILED,
            () -> System.out.println("[FSM] 任务失败，记录错误日志..."));
    }
}
```

### 4.4 模拟 LLM 服务接口

```java
public interface LLMService {
    String analyze(String task);
    String design(String analysisResult);
    String generateCode(String designResult);
    boolean runTests(String code);
    String fix(String code);
}
```

## 五、死循环防护机制

光有状态机还不够，必须加上几道"保险"：

### 5.1 同状态循环检测

```java
public class LoopDetectionGuard {
    private final Map<String, Integer> stateRepeatCounter = new HashMap<>();
    private static final int MAX_REPEAT = 3;

    /**
     * 检测是否在同一状态卡住
     */
    public boolean detectLoop(AgentState state, AgentEvent event) {
        String key = state.name() + ":" + event.name();
        int count = stateRepeatCounter.merge(key, 1, Integer::sum);
        if (count >= MAX_REPEAT) {
            System.err.println("!!! 检测到死循环: " + state.getDescription()
                + " 响应 " + event + " 超过 " + MAX_REPEAT + " 次");
            stateRepeatCounter.clear();
            return true;
        }
        return false;
    }

    public void reset() {
        stateRepeatCounter.clear();
    }
}
```

### 5.2 Token 消耗熔断

```java
public class TokenCircuitBreaker {
    private final long maxTokens;
    private long consumedTokens;

    public TokenCircuitBreaker(long maxTokens) {
        this.maxTokens = maxTokens;
    }

    public void consume(long tokens) {
        consumedTokens += tokens;
        if (consumedTokens > maxTokens) {
            throw new CircuitBreakerException(
                String.format("Token 预算耗尽: %d/%d", consumedTokens, maxTokens));
        }
    }

    public long getRemainingTokens() {
        return maxTokens - consumedTokens;
    }

    public void reset() {
        consumedTokens = 0;
    }
}
```

### 5.3 超时熔断器

```java
import java.util.concurrent.*;

public class TimeoutGuard {
    private final long timeoutMs;

    public TimeoutGuard(long timeoutMs) {
        this.timeoutMs = timeoutMs;
    }

    public <T> T executeWithTimeout(Callable<T> task) throws Exception {
        ExecutorService executor = Executors.newSingleThreadExecutor();
        try {
            Future<T> future = executor.submit(task);
            return future.get(timeoutMs, TimeUnit.MILLISECONDS);
        } catch (TimeoutException e) {
            future.cancel(true);
            System.err.println("!!! 任务执行超时 (" + timeoutMs + "ms)");
            throw new RuntimeException("任务执行超时");
        } finally {
            executor.shutdownNow();
        }
    }
}
```

## 六、完整流程图

```
                    ┌─────────────────────────────────────────────────┐
                    │                 AgentStateMachine               │
                    │                                                 │
    ┌───────┐       │   ┌──────────┐    ┌──────────┐    ┌────────┐   │
    │ IDLE  │───────│──▶│ANALYZING │───▶│DESIGNING │───▶│ CODING │   │
    └───────┘       │   └──────────┘    └──────────┘    └────────┘   │
        │           │        │               │               │        │
        │           │        │               │               │        │
        │           │     ┌──▼──┐         ┌──▼──┐         ┌──▼──┐     │
        │           │     │PAUSE│         │FAIL │         │REVIEW│    │
        │           │     └──┬──┘         └─────┘         └──┬──┘     │
        │           │        │                               │        │
        │           │        │            ┌──────┐          │        │
        └───────────│────────┴────────────│RECOVER│◀────────┘        │
                    │                     └──────┘                    │
                    │                                                 │
                    │          RedisAgentStateStore (持久化)           │
                    │          LoopDetectionGuard  (防死循环)          │
                    │          TokenCircuitBreaker (熔断)              │
                    │          TimeoutGuard        (超时)              │
                    └─────────────────────────────────────────────────┘
```

## 七、总结

这篇文章的核心要点：

1. **Agent 状态管理的三个目标**：避免死循环、保证流程正确、支持断点恢复
2. **有限状态机三要素**：状态定义、转换条件、动作触发
3. **Java 实现**：枚举定义状态/事件，Map 构建转换表，同步方法保证线程安全
4. **Redis 持久化**：关键步骤前保存快照，支持从任意断点恢复
5. **三道保险**：循环检测 + Token 熔断 + 超时熔断

如果你想看生产级代码，我们内部的项目在这个基础上还加了分布式锁（Redis RedLock）、状态机可视化（PlantUML 自动生成状态流转图）、以及 Grafana 大盘监控。但核心思想就是上面这些。

状态机看似是个很"古老"的设计模式，但在 AI Agent 的工程化落地中，它依然是最可靠的选择。搞定了状态管理，你的 Agent 才能真正在生产环境中"可控地跑起来"，而不是一个随时可能失控的"定时炸弹"。

---

**下一篇预告**：《Multi-Agent 多智能体协作：角色分工与消息传递的架构设计》—— 一个 Agent 不够用？让一群 AI 各司其职，用消息总线实现高效协作。需求分析、编码、测试，每个环节都有专属 Agent，单打独斗的时代过去了。
