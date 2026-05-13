# Multi-Agent 多智能体协作：角色分工与消息传递的架构设计，一个AI不够用就让一群AI协作

## 前言

上篇我们聊了单 Agent 的状态管理，但真实业务场景下，单打独斗的 Agent 太容易"翻车"了。让一个 Agent 同时负责需求分析、架构设计、代码编写、测试验证，就像让一个程序员独自扛起整个项目 — 不是不能做，是太容易出问题。

我在公司内部落地 AI 开发助手的时候，前两个月用的就是单 Agent 模式。结果发现：需求分析不充分就写代码 → 写出来的代码和需求对不上 → 测试阶段才发现问题 → 回退重写。整个流程效率极低，Token 浪费严重。

后来我拆分成 Multiple Agent 模式，效果立竿见影：需求分析的 Agent 专职拆解需求，编码 Agent 专注写代码，测试 Agent 严格把关。各司其职，互相制约，最后再汇总。这篇文章就把这套 Multi-Agent 协作架构完整分享出来。

## 一、Multi-Agent 的核心概念

### 1.1 为什么要多 Agent？

单 Agent 的痛点很明确：

- **上下文窗口限制**：一个 Agent 的 System Prompt 既要包含需求分析指令、又要包含编码指令、还要包含测试指令，Prompt 越长效果越差
- **角色冲突**：同一个 Agent 既要做"分析师"又要做"程序员"，思考模式完全不同，很难在一个对话中自由切换
- **不可控性**：把所有推理放在一个 Agent 里，中间过程黑盒化，出了问题不知道是哪一步错了
- **无法并行**：代码生成和测试编写完全可以并行，单 Agent 只能串行

Multi-Agent 的本质是 **"分而治之"**：用多个「术业有专攻」的 Agent 拆解复杂任务。

### 1.2 核心角色设计

一个典型的 Multi-Agent 系统包含以下角色：

```
┌──────────────────────────────────────────────────────┐
│                   Orchestrator                       │
│                   (协调器/调度中心)                    │
└───────┬──────────────┬──────────────┬────────────────┘
        │              │              │
   ┌────▼────┐    ┌────▼────┐    ┌────▼────┐
   │ Analyst │    │  Coder  │    │ Tester  │
   │ 分析Agent│    │ 编码Agent│    │ 测试Agent│
   └─────────┘    └─────────┘    └─────────┘
        │                               │
   ┌────▼────┐                     ┌────▼────┐
   │Reviewer │                     │ DocAgent│
   │ 审查Agent│                     │ 文档Agent│
   └─────────┘                     └─────────┘
```

每个 Agent 的核心定义包括：

1. **角色（Role）**：我在团队里是干什么的
2. **能力（Capability）**：我能处理什么类型的任务
3. **输入输出协议（Protocol）**：我接收什么格式的输入，产出什么格式的输出
4. **约束条件（Constraint）**：我的行为边界在哪里

## 二、消息传递机制设计

多 Agent 协作的本质是**消息传递**。消息传递的方式决定了整个系统的灵活性和可扩展性。

### 2.1 三种消息模式

#### 点对点（Point-to-Point）

适用于上下游依赖关系，比如 Analyst → Coder → Tester。

```java
// 点对点消息
AgentMessage msg = AgentMessage.builder()
    .from("analyst-1")
    .to("coder-1")
    .type(MessageType.TASK_ASSIGN)
    .payload(analysisResult)
    .build();
messageBus.send(msg);
```

#### 广播（Broadcast）

适用于全局通知，比如"任务中断""所有 Agent 暂停"。

```java
// 广播消息
AgentMessage msg = AgentMessage.builder()
    .from("orchestrator")
    .toAll()
    .type(MessageType.SYSTEM_PAUSE)
    .priority(Priority.HIGH)
    .build();
messageBus.broadcast(msg);
```

#### 发布订阅（Pub-Sub）

适用于解耦场景，比如"Coder 完成了某模块" → 对该模块感兴趣的 Tester 和 DocAgent 都能收到通知。

```java
// 发布订阅
messageBus.subscribe("topic:module-A-completed", testerAgent);
messageBus.subscribe("topic:module-A-completed", docAgent);
messageBus.publish("topic:module-A-completed", moduleResult);
```

### 2.2 消息协议定义

```java
import java.io.Serializable;
import java.util.Map;
import java.util.UUID;

public class AgentMessage implements Serializable {
    private static final long serialVersionUID = 1L;

    private String messageId;
    private String fromAgentId;
    private String toAgentId;       // null 表示广播
    private MessageType type;
    private String topic;           // 用于 Pub-Sub
    private Map<String, Object> payload;
    private Priority priority;
    private long timestamp;
    private String correlationId;   // 关联会话ID
    private String replyTo;         // 期望回复的目标

    public enum MessageType {
        TASK_ASSIGN,      // 任务分配
        TASK_COMPLETE,    // 任务完成
        TASK_FAILED,      // 任务失败
        REQUEST_HELP,     // 请求协助
        PROVIDE_RESULT,   // 提供结果
        SYSTEM_PAUSE,     // 系统暂停
        SYSTEM_RESUME,    // 系统恢复
        HEARTBEAT,        // 心跳
        QUERY,            // 查询
        RESPONSE          // 响应
    }

    public enum Priority {
        LOW, NORMAL, HIGH, CRITICAL
    }

    public AgentMessage() {
        this.messageId = UUID.randomUUID().toString();
        this.timestamp = System.currentTimeMillis();
        this.priority = Priority.NORMAL;
    }

    // Builder 模式
    public static Builder builder() {
        return new Builder();
    }

    public static class Builder {
        private AgentMessage msg = new AgentMessage();

        public Builder from(String agentId) { msg.fromAgentId = agentId; return this; }
        public Builder to(String agentId) { msg.toAgentId = agentId; return this; }
        public Builder toAll() { msg.toAgentId = null; return this; }
        public Builder type(MessageType type) { msg.type = type; return this; }
        public Builder topic(String topic) { msg.topic = topic; return this; }
        public Builder payload(Map<String, Object> payload) { msg.payload = payload; return this; }
        public Builder priority(Priority p) { msg.priority = p; return this; }
        public Builder correlationId(String id) { msg.correlationId = id; return this; }
        public Builder replyTo(String agentId) { msg.replyTo = agentId; return this; }
        public AgentMessage build() { return msg; }
    }

    // Getters
    public String getMessageId() { return messageId; }
    public String getFromAgentId() { return fromAgentId; }
    public String getToAgentId() { return toAgentId; }
    public MessageType getType() { return type; }
    public String getTopic() { return topic; }
    public Map<String, Object> getPayload() { return payload; }
    public Priority getPriority() { return priority; }
    public long getTimestamp() { return timestamp; }
    public String getCorrelationId() { return correlationId; }
    public String getReplyTo() { return replyTo; }
}
```

### 2.3 消息总线实现

```java
import java.util.concurrent.*;
import java.util.*;

public class AgentMessageBus {
    // 点对点：AgentID → 消息队列
    private final Map<String, BlockingQueue<AgentMessage>> agentQueues;
    // 发布订阅：Topic → 订阅者列表
    private final Map<String, List<String>> topicSubscribers;
    // 广播：所有注册的 Agent ID
    private final Set<String> registeredAgents;
    // 线程池用于异步消息分发
    private final ExecutorService dispatcherPool;

    public AgentMessageBus() {
        this.agentQueues = new ConcurrentHashMap<>();
        this.topicSubscribers = new ConcurrentHashMap<>();
        this.registeredAgents = ConcurrentHashMap.newKeySet();
        this.dispatcherPool = Executors.newFixedThreadPool(4);
    }

    /**
     * 注册 Agent
     */
    public void registerAgent(String agentId) {
        registeredAgents.add(agentId);
        agentQueues.putIfAbsent(agentId, new LinkedBlockingQueue<>());
        System.out.println("[MessageBus] Agent 注册: " + agentId);
    }

    /**
     * 注销 Agent
     */
    public void unregisterAgent(String agentId) {
        registeredAgents.remove(agentId);
        agentQueues.remove(agentId);
        topicSubscribers.values().forEach(list -> list.remove(agentId));
        System.out.println("[MessageBus] Agent 注销: " + agentId);
    }

    /**
     * 点对点发送
     */
    public void send(AgentMessage message) {
        if (message.getType() == null) {
            throw new IllegalArgumentException("消息类型不能为空");
        }

        BlockingQueue<AgentMessage> queue = agentQueues.get(message.getToAgentId());
        if (queue == null) {
            throw new AgentNotFoundException("Agent 未注册: " + message.getToAgentId());
        }

        if (!queue.offer(message)) {
            System.err.println("[MessageBus] 消息队列已满，丢弃: " + message.getMessageId());
        }
    }

    /**
     * 广播发送
     */
    public void broadcast(AgentMessage message) {
        for (String agentId : registeredAgents) {
            // 不发送给自己
            if (!agentId.equals(message.getFromAgentId())) {
                BlockingQueue<AgentMessage> queue = agentQueues.get(agentId);
                if (queue != null) {
                    queue.offer(message);
                }
            }
        }
    }

    /**
     * 发布到 Topic
     */
    public void publish(String topic, AgentMessage message) {
        message.topic = topic;
        List<String> subscribers = topicSubscribers.get(topic);
        if (subscribers != null) {
            for (String agentId : subscribers) {
                BlockingQueue<AgentMessage> queue = agentQueues.get(agentId);
                if (queue != null) {
                    queue.offer(message);
                }
            }
        }
    }

    /**
     * 订阅 Topic
     */
    public void subscribe(String topic, String agentId) {
        topicSubscribers
            .computeIfAbsent(topic, k -> new CopyOnWriteArrayList<>())
            .add(agentId);
        System.out.println("[MessageBus] " + agentId + " 订阅了: " + topic);
    }

    /**
     * 接收消息（阻塞，带超时）
     */
    public AgentMessage receive(String agentId, long timeoutMs)
            throws InterruptedException {
        BlockingQueue<AgentMessage> queue = agentQueues.get(agentId);
        if (queue == null) {
            throw new AgentNotFoundException("Agent 未注册: " + agentId);
        }
        return queue.poll(timeoutMs, TimeUnit.MILLISECONDS);
    }

    /**
     * 批量接收消息
     */
    public List<AgentMessage> receiveAll(String agentId, long timeoutMs)
            throws InterruptedException {
        List<AgentMessage> messages = new ArrayList<>();
        BlockingQueue<AgentMessage> queue = agentQueues.get(agentId);
        if (queue == null) return messages;

        AgentMessage msg = queue.poll(timeoutMs, TimeUnit.MILLISECONDS);
        while (msg != null) {
            messages.add(msg);
            // 继续尝试非阻塞地取消息
            msg = queue.poll();
        }
        return messages;
    }

    public Set<String> getRegisteredAgents() {
        return Collections.unmodifiableSet(registeredAgents);
    }
}
```

## 三、Agent 基类设计

```java
import java.util.concurrent.atomic.AtomicBoolean;
import java.util.Map;
import java.util.HashMap;

public abstract class BaseAgent implements Runnable {
    protected final String agentId;
    protected final String role;
    protected final AgentMessageBus messageBus;
    protected final LLMService llmService;
    protected final AtomicBoolean running;
    private final Map<String, Object> context;

    public BaseAgent(String agentId, String role,
                     AgentMessageBus messageBus, LLMService llmService) {
        this.agentId = agentId;
        this.role = role;
        this.messageBus = messageBus;
        this.llmService = llmService;
        this.running = new AtomicBoolean(true);
        this.context = new ConcurrentHashMap<>();
    }

    @Override
    public void run() {
        messageBus.registerAgent(agentId);
        System.out.println("[" + role + "] Agent 启动: " + agentId);

        try {
            while (running.get()) {
                AgentMessage msg = messageBus.receive(agentId, 1000);
                if (msg != null) {
                    handleMessage(msg);
                }
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            System.out.println("[" + role + "] Agent 被中断: " + agentId);
        } catch (Exception e) {
            System.err.println("[" + role + "] Agent 异常: " + e.getMessage());
        } finally {
            messageBus.unregisterAgent(agentId);
            System.out.println("[" + role + "] Agent 终止: " + agentId);
        }
    }

    /**
     * 子类实现具体的消息处理逻辑
     */
    protected abstract void handleMessage(AgentMessage msg);

    /**
     * 发送回复消息
     */
    protected void reply(AgentMessage original, Map<String, Object> result) {
        AgentMessage response = AgentMessage.builder()
            .from(agentId)
            .to(original.getFromAgentId())
            .type(AgentMessage.MessageType.PROVIDE_RESULT)
            .correlationId(original.getCorrelationId() != null
                ? original.getCorrelationId() : original.getMessageId())
            .payload(result)
            .build();
        messageBus.send(response);
    }

    /**
     * 完成任务通知
     */
    protected void notifyComplete(String taskId, Map<String, Object> result) {
        AgentMessage msg = AgentMessage.builder()
            .from(agentId)
            .to("orchestrator")
            .type(AgentMessage.MessageType.TASK_COMPLETE)
            .correlationId(taskId)
            .payload(result)
            .build();
        messageBus.send(msg);
    }

    /**
     * 请求其他 Agent 协助
     */
    protected void requestHelp(String targetAgentId, String taskDescription,
                               Map<String, Object> context) {
        Map<String, Object> payload = new HashMap<>(context);
        payload.put("task", taskDescription);

        AgentMessage msg = AgentMessage.builder()
            .from(agentId)
            .to(targetAgentId)
            .type(AgentMessage.MessageType.REQUEST_HELP)
            .payload(payload)
            .build();
        messageBus.send(msg);
    }

    public void stop() {
        running.set(false);
    }

    public String getAgentId() { return agentId; }
    public String getRole() { return role; }

    protected Object getContextValue(String key) { return context.get(key); }
    protected void setContextValue(String key, Object value) { context.put(key, value); }
}
```

## 四、协调器（Orchestrator）设计

协调器是整个 Multi-Agent 系统的大脑，负责：接收用户任务 → 分解子任务 → 分配给对应 Agent → 收集结果 → 汇总返回。

```java
import java.util.concurrent.*;

public class AgentOrchestrator {
    private final AgentMessageBus messageBus;
    private final Map<String, BaseAgent> agents;
    private final ExecutorService agentPool;
    private final Map<String, CompletableFuture<Map<String, Object>>> pendingTasks;
    private final Map<String, String> agentRoleMap; // AgentID → Role

    public AgentOrchestrator(AgentMessageBus messageBus) {
        this.messageBus = messageBus;
        this.agents = new ConcurrentHashMap<>();
        this.agentPool = Executors.newCachedThreadPool();
        this.pendingTasks = new ConcurrentHashMap<>();
        this.agentRoleMap = new ConcurrentHashMap<>();
    }

    /**
     * 注册并启动 Agent
     */
    public void registerAndStart(BaseAgent agent) {
        agents.put(agent.getAgentId(), agent);
        agentRoleMap.put(agent.getAgentId(), agent.getRole());
        agentPool.submit(agent);
    }

    /**
     * 提交任务并等待结果
     */
    public Map<String, Object> executeTask(String taskDescription, long timeoutMs)
            throws InterruptedException, ExecutionException, TimeoutException {

        String taskId = UUID.randomUUID().toString();
        System.out.println("\n========== 任务开始: " + taskId + " ==========");
        System.out.println("用户需求: " + taskDescription);

        // Step 1: 需求分析
        System.out.println("\n--- 阶段1: 需求分析 ---");
        Map<String, Object> analysisResult = assignTaskAndWait(
            findAgentByRole("Analyst"), taskDescription, taskId, timeoutMs);
        System.out.println("需求分析完成: " + analysisResult.get("analysis"));

        // Step 2: 编码
        System.out.println("\n--- 阶段2: 代码生成 ---");
        Map<String, Object> codingResult = assignTaskAndWait(
            findAgentByRole("Coder"), analysisResult, taskId, timeoutMs);
        String code = (String) codingResult.get("code");

        // Step 3: 并行执行测试和文档
        System.out.println("\n--- 阶段3: 并行测试+文档 ---");
        CompletableFuture<Map<String, Object>> testFuture = CompletableFuture.supplyAsync(() -> {
            try {
                Map<String, Object> params = new HashMap<>();
                params.put("code", code);
                params.put("task", taskDescription);
                return assignTaskAndWait(findAgentByRole("Tester"), params, taskId, timeoutMs);
            } catch (Exception e) {
                Map<String, Object> err = new HashMap<>();
                err.put("test_pass", false);
                err.put("error", e.getMessage());
                return err;
            }
        });

        CompletableFuture<Map<String, Object>> docFuture = CompletableFuture.supplyAsync(() -> {
            try {
                Map<String, Object> params = new HashMap<>();
                params.put("code", code);
                params.put("task", taskDescription);
                return assignTaskAndWait(findAgentByRole("DocWriter"), params, taskId, timeoutMs);
            } catch (Exception e) {
                Map<String, Object> err = new HashMap<>();
                err.put("doc", "文档生成失败");
                return err;
            }
        });

        // 等待并行结果
        CompletableFuture.allOf(testFuture, docFuture).get(timeoutMs, TimeUnit.MILLISECONDS);
        Map<String, Object> testResult = testFuture.get();
        Map<String, Object> docResult = docFuture.get();

        // Step 4: 审查结果
        System.out.println("\n--- 阶段4: 代码审查 ---");
        Map<String, Object> reviewParams = new HashMap<>();
        reviewParams.put("code", code);
        reviewParams.put("test_result", testResult);

        String reviewerId = findAgentByRole("Reviewer");
        if (reviewerId != null) {
            Map<String, Object> reviewResult = assignTaskAndWait(
                reviewerId, reviewParams, taskId, timeoutMs);
            Boolean approved = (Boolean) reviewResult.get("approved");
            if (approved != null && !approved) {
                System.out.println("审查未通过: " + reviewResult.get("comments"));
            }
        }

        // 汇总结果
        Map<String, Object> finalResult = new LinkedHashMap<>();
        finalResult.put("task_id", taskId);
        finalResult.put("analysis", analysisResult);
        finalResult.put("code", code);
        finalResult.put("test_result", testResult);
        finalResult.put("documentation", docResult);
        finalResult.put("status", "completed");

        System.out.println("\n========== 任务完成: " + taskId + " ==========");
        return finalResult;
    }

    /**
     * 分配任务给指定 Agent 并等待结果
     */
    private Map<String, Object> assignTaskAndWait(String agentId, Object taskPayload,
                                                   String taskId, long timeoutMs)
            throws InterruptedException, ExecutionException, TimeoutException {

        if (agentId == null) {
            System.err.println("未找到对应角色的 Agent");
            return Map.of("error", "Agent not found");
        }

        Map<String, Object> payload = new HashMap<>();
        if (taskPayload instanceof Map) {
            payload.putAll((Map<String, Object>) taskPayload);
        } else if (taskPayload instanceof String) {
            payload.put("task", taskPayload);
        }
        payload.put("task_id", taskId);

        AgentMessage taskMsg = AgentMessage.builder()
            .from("orchestrator")
            .to(agentId)
            .type(AgentMessage.MessageType.TASK_ASSIGN)
            .correlationId(taskId)
            .payload(payload)
            .build();

        CompletableFuture<Map<String, Object>> resultFuture = new CompletableFuture<>();
        pendingTasks.put(taskId, resultFuture);

        messageBus.send(taskMsg);

        try {
            return resultFuture.get(timeoutMs, TimeUnit.MILLISECONDS);
        } catch (TimeoutException e) {
            pendingTasks.remove(taskId);
            throw e;
        }
    }

    /**
     * 根据角色查找 Agent
     */
    private String findAgentByRole(String role) {
        return agentRoleMap.entrySet().stream()
            .filter(e -> e.getValue().equals(role))
            .map(Map.Entry::getKey)
            .findFirst()
            .orElse(null);
    }

    /**
     * 监听消息，处理 Agent 返回的结果
     */
    public void startListening() {
        Thread listener = new Thread(() -> {
            while (!Thread.currentThread().isInterrupted()) {
                try {
                    // Orchestrator 自己也注册到消息总线
                    List<AgentMessage> messages = messageBus.receiveAll("orchestrator", 500);
                    for (AgentMessage msg : messages) {
                        if (msg.getType() == AgentMessage.MessageType.TASK_COMPLETE) {
                            CompletableFuture<Map<String, Object>> future =
                                pendingTasks.get(msg.getCorrelationId());
                            if (future != null) {
                                future.complete(msg.getPayload());
                                pendingTasks.remove(msg.getCorrelationId());
                            }
                        } else if (msg.getType() == AgentMessage.MessageType.TASK_FAILED) {
                            CompletableFuture<Map<String, Object>> future =
                                pendingTasks.get(msg.getCorrelationId());
                            if (future != null) {
                                future.completeExceptionally(
                                    new RuntimeException((String) msg.getPayload().get("error")));
                                pendingTasks.remove(msg.getCorrelationId());
                            }
                        }
                    }
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    break;
                }
            }
        }, "Orchestrator-Listener");
        listener.setDaemon(true);
        listener.start();

        // 注册 orchestrator 到消息总线
        messageBus.registerAgent("orchestrator");
    }

    public void shutdown() {
        agents.values().forEach(BaseAgent::stop);
        agentPool.shutdown();
    }
}
```

## 五、实战：三 Agent 协作开发流程

### 5.1 需求分析 Agent

```java
public class AnalystAgent extends BaseAgent {

    public AnalystAgent(String agentId, AgentMessageBus messageBus, LLMService llmService) {
        super(agentId, "Analyst", messageBus, llmService);
    }

    @Override
    protected void handleMessage(AgentMessage msg) {
        if (msg.getType() == AgentMessage.MessageType.TASK_ASSIGN) {
            String task = (String) msg.getPayload().get("task");
            String taskId = (String) msg.getPayload().get("task_id");

            System.out.println("[Analyst] 收到分析任务: " + task);

            // 调用 LLM 进行需求分析
            String analysisPrompt = String.format(
                "你是一名资深需求分析师，请分析以下需求，拆解为可执行的开发任务：\n\n%s\n\n" +
                "请按以下格式输出：\n" +
                "1. 需求概述\n" +
                "2. 功能点列表\n" +
                "3. 技术要点\n" +
                "4. 潜在风险", task
            );

            String analysisResult = llmService.analyze(analysisPrompt);

            Map<String, Object> result = new HashMap<>();
            result.put("analysis", analysisResult);
            result.put("task_id", taskId);
            result.put("agent", agentId);

            notifyComplete(taskId, result);
        }
    }
}
```

### 5.2 编码 Agent

```java
public class CoderAgent extends BaseAgent {

    public CoderAgent(String agentId, AgentMessageBus messageBus, LLMService llmService) {
        super(agentId, "Coder", messageBus, llmService);
    }

    @Override
    protected void handleMessage(AgentMessage msg) {
        if (msg.getType() == AgentMessage.MessageType.TASK_ASSIGN) {
            String analysis = (String) msg.getPayload().getOrDefault(
                "analysis", msg.getPayload().get("task"));
            String taskId = (String) msg.getPayload().get("task_id");

            System.out.println("[Coder] 收到编码任务: " + taskId);

            String codePrompt = String.format(
                "你是一名资深 Java 开发工程师，根据以下需求分析生成代码：\n\n%s\n\n" +
                "要求：\n" +
                "1. 代码结构清晰，遵循 SOLID 原则\n" +
                "2. 包含必要的异常处理\n" +
                "3. 添加关键方法的 JavaDoc 注释\n" +
                "4. 仅输出代码，不要解释", analysis
            );

            String code = llmService.generateCode(codePrompt);

            Map<String, Object> result = new HashMap<>();
            result.put("code", code);
            result.put("task_id", taskId);

            // 发布消息：代码模块已完成
            AgentMessage pubMsg = AgentMessage.builder()
                .from(agentId)
                .topic("topic:code-completed")
                .type(AgentMessage.MessageType.PROVIDE_RESULT)
                .payload(Map.of("task_id", taskId, "code_length", code.length()))
                .build();
            messageBus.publish("topic:code-completed", pubMsg);

            notifyComplete(taskId, result);
        }
    }
}
```

### 5.3 测试 Agent

```java
public class TesterAgent extends BaseAgent {

    public TesterAgent(String agentId, AgentMessageBus messageBus, LLMService llmService) {
        super(agentId, "Tester", messageBus, llmService);
    }

    @Override
    protected void handleMessage(AgentMessage msg) {
        if (msg.getType() == AgentMessage.MessageType.TASK_ASSIGN) {
            String code = (String) msg.getPayload().get("code");
            String taskDesc = (String) msg.getPayload().getOrDefault("task", "");
            String taskId = (String) msg.getPayload().get("task_id");

            System.out.println("[Tester] 收到测试任务: " + taskId);

            // 生成测试用例
            String testPrompt = String.format(
                "你是一名资深测试工程师，请为以下代码生成测试用例并执行测试评估：\n\n" +
                "原始需求: %s\n\n" +
                "代码: \n```java\n%s\n```\n\n" +
                "请分析：\n" +
                "1. 代码是否满足需求\n" +
                "2. 边界条件是否覆盖\n" +
                "3. 潜在Bug\n" +
                "4. 综合评价(通过/不通过)", taskDesc, code
            );

            String testResult = llmService.runTests(testPrompt);

            boolean passed = !testResult.toLowerCase().contains("不通过")
                          && !testResult.toLowerCase().contains("bug");

            Map<String, Object> result = new HashMap<>();
            result.put("test_pass", passed);
            result.put("test_result", testResult);
            result.put("task_id", taskId);

            notifyComplete(taskId, result);
        }
    }
}
```

### 5.4 组装运行

```java
public class MultiAgentDemo {
    public static void main(String[] args) throws Exception {
        // 初始化消息总线
        AgentMessageBus messageBus = new AgentMessageBus();

        // 初始化 LLM 服务（替换为真实的 LLM 实现）
        LLMService llmService = new OpenAILLMService("your-api-key");

        // 创建 Orchestrator
        AgentOrchestrator orchestrator = new AgentOrchestrator(messageBus);
        orchestrator.startListening();

        // 创建并注册各 Agent
        AnalystAgent analyst = new AnalystAgent("analyst-1", messageBus, llmService);
        CoderAgent coder = new CoderAgent("coder-1", messageBus, llmService);
        TesterAgent tester = new TesterAgent("tester-1", messageBus, llmService);

        orchestrator.registerAndStart(analyst);
        orchestrator.registerAndStart(coder);
        orchestrator.registerAndStart(tester);

        // 等一下让所有 Agent 启动完毕
        Thread.sleep(500);

        // 提交任务
        String task = "实现一个线程安全的 LRU 缓存，支持泛型，包含 get/put/remove/size 方法，" +
                      "最大容量可配置，使用 LinkedHashMap 实现";

        Map<String, Object> result = orchestrator.executeTask(task, 120_000);

        System.out.println("\n======= 最终结果 =======");
        System.out.println("需求分析: " + result.get("analysis"));
        System.out.println("代码: \n" + result.get("code"));
        System.out.println("测试结果: " + result.get("test_result"));

        orchestrator.shutdown();
    }
}
```

## 六、消息传递的可靠性保障

生产环境中，消息传递不能只是"发了就算"，必须考虑可靠性。

### 6.1 消息确认机制

```java
public void sendWithAck(AgentMessage message, long ackTimeoutMs)
        throws TimeoutException, InterruptedException {
    String correlationId = message.getCorrelationId() != null
        ? message.getCorrelationId() : message.getMessageId();

    CompletableFuture<Boolean> ackFuture = new CompletableFuture<>();
    pendingAcks.put(correlationId, ackFuture);

    messageBus.send(message);

    try {
        Boolean acknowledged = ackFuture.get(ackTimeoutMs, TimeUnit.MILLISECONDS);
        if (!acknowledged) {
            throw new RuntimeException("消息未被确认: " + correlationId);
        }
    } catch (TimeoutException e) {
        // 超时重试
        System.err.println("消息超时，准备重试: " + correlationId);
        retrySend(message);
    }
}
```

### 6.2 消息去重

```java
public class MessageDeduplication {
    private final Set<String> processedMessageIds = Collections.newSetFromMap(
        new LinkedHashMap<>() {
            @Override
            protected boolean removeEldestEntry(Map.Entry<String, Boolean> eldest) {
                return size() > 10000; // 只保留最近 10000 条
            }
        }
    );

    public boolean isDuplicate(String messageId) {
        return !processedMessageIds.add(messageId);
    }
}
```

## 七、总结与最佳实践

Multi-Agent 架构落地的关键经验：

1. **角色粒度要合适**：太粗（一个 Agent 干所有事）没有意义，太细（一个功能点一个 Agent）管理成本太高。一般 4-6 个角色足够覆盖大多数场景

2. **消息协议要统一**：所有 Agent 使用相同的消息格式，新 Agent 加入时零成本对接

3. **协调器要有容错能力**：某个 Agent 挂了不能拖垮整个流程，要有超时和降级策略

4. **并行化是关键收益**：把无依赖的任务并行执行，效率提升非常明显。我们在测试 + 文档场景下，并行 vs 串行快了 40%

5. **监控不能少**：每个 Agent 的消息处理速度、队列积压、任务成功率都要监控，否则出了问题你都不知道是哪个 Agent 崩了

Multi-Agent 不是银弹，但它确实能解决单 Agent 上下文窗口限制和角色冲突的痛点。如果你的场景明确可以拆成多个步骤，强烈建议试试这个架构。

---

**下一篇预告**：《Agent 辩论模式：让两个 AI 互相质疑提升输出质量》— 你以为一个 AI 给出的方案就足够好了？让两个 AI 互相"吵架"，一个生成方案、一个锐评挑刺，最后交给裁判 Agent 裁决。辩论模式让你的 AI 输出质量再上一个台阶。
