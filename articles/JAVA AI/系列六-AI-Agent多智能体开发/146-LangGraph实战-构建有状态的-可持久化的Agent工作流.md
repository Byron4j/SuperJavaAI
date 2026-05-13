# LangGraph 实战：构建有状态的、可持久化的 Agent 工作流，Java 也能实现 LangGraph 的核心思想

## 一、引言：为什么需要 "有状态" 的 Agent？

传统 Agent 模式有一个致命伤：**无状态**。

你和一个 AI Agent 聊了两句，建了个分支，执行了一个操作。然后网络断了。等重连回来，Agent 已经忘了刚才聊到哪了——所有上下文丢失，你得重来。

如果你是 Agent 开发者，还有一个更尴尬的问题：**多步骤任务的断点恢复**。假设 Agent 需要执行「分析代码 → 生成修复 → 提交 PR → 等待 CI 通过 → 合并」这五步，如果第三步执行到一半进程挂了，怎么办？从头再来？

LangGraph 就是为了解决这些问题而生的。它是 LangChain 团队推出的**有状态工作流框架**，核心创新在于：

- **StateGraph**：有状态的图结构，每一步的状态都会保存
- **Checkpoint**：任意节点的状态快照，支持断点恢复
- **条件分支**：根据状态动态决定下一步走向

虽然 LangGraph 是 Python 的，但它的核心思想完全是语言无关的。今天我们要在 Java 中实现它的精髓。

> 全文约 5000 字。关注我，用 Java 思维理解前沿 AI 框架。

---

## 二、核心概念：StateGraph + Checkpoint + 条件分支

在 LangGraph 的世界里，有三个核心概念：

```
┌──────────────────────────────────────────────────────────┐
│                     StateGraph                             │
│  ┌──────────┐       ┌──────────┐       ┌──────────┐     │
│  │  Node A   │ ────→ │  Node B   │ ────→ │  Node C   │     │
│  │ (分析)    │       │ (生成)    │       │ (验证)    │     │
│  └──────────┘       └──────────┘       └──────────┘     │
│       │                    │                    │        │
│       ▼                    ▼                    ▼        │
│  ┌──────────────────────────────────────────────────┐    │
│  │              Checkpoint 存储                      │    │
│  │  ┌───────┐  ┌───────┐  ┌───────┐  ┌───────┐    │    │
│  │  │CP #1  │  │CP #2  │  │CP #3  │  │CP #4  │    │    │
│  │  │状态A  │  │状态B  │  │状态C  │  │最终   │    │    │
│  │  └───────┘  └───────┘  └───────┘  └───────┘    │    │
│  └──────────────────────────────────────────────────┘    │
│                                                          │
│       条件分支：如果 验证失败 → 回到 Node B               │
│                   如果 验证通过 → 结束                    │
└──────────────────────────────────────────────────────────┘
```

用一句话概括：

- **StateGraph** = 一个状态机，每个节点代表一个执行步骤
- **State** = 在节点之间流转的共享数据对象
- **Checkpoint** = 在节点转换时自动保存的状态快照
- **条件分支** = 根据当前状态动态选择下一个节点

---

## 三、Java 实现：核心类设计

### 3.1 StateGraph 定义

```java
/**
 * 有状态图 - LangGraph 的核心抽象
 * T 是状态类型（在节点间流转的上下文）
 */
public class StateGraph<T extends Serializable> {
    
    private final Map<String, GraphNode<T>> nodes;
    private final Map<String, List<Edge<T>>> edges;
    private final String entryPoint;
    private String finishPoint;
    
    public StateGraph(String entryPoint) {
        this.nodes = new LinkedHashMap<>();
        this.edges = new HashMap<>();
        this.entryPoint = entryPoint;
    }
    
    /**
     * 添加一个节点
     * @param name 节点名称
     * @param handler 节点处理函数，接收当前状态，返回新状态
     */
    public StateGraph<T> addNode(String name, NodeHandler<T> handler) {
        nodes.put(name, new GraphNode<>(name, handler));
        return this;
    }
    
    /**
     * 添加一个边（固定路由）
     * 执行完 source 节点后，无条件转到 target 节点
     */
    public StateGraph<T> addEdge(String source, String target) {
        edges.computeIfAbsent(source, k -> new ArrayList<>())
                .add(Edge.fixed(target));
        return this;
    }
    
    /**
     * 添加条件边（动态路由）
     * 执行完 source 节点后，根据 condition 判断结果转到对应节点
     */
    public StateGraph<T> addConditionalEdge(String source, 
                                             ConditionHandler<T> condition,
                                             Map<String, String> pathMap) {
        edges.computeIfAbsent(source, k -> new ArrayList<>())
                .add(Edge.conditional(condition, pathMap));
        return this;
    }
    
    /**
     * 设置结束节点
     */
    public StateGraph<T> setFinishPoint(String nodeName) {
        // 如果该节点不是起始点，也需要在图中注册
        if (!nodes.containsKey(nodeName)) {
            throw new IllegalArgumentException("Finish node must be added first: " + nodeName);
        }
        this.finishPoint = nodeName;
        return this;
    }
    
    /**
     * 编译图——返回一个可以执行的工作流
     */
    public CompiledGraph<T> compile() {
        return compile(null);
    }
    
    /**
     * 编译图——带 Checkpoint 存储
     */
    public CompiledGraph<T> compile(CheckpointStore<T> checkpointStore) {
        validate();
        return new CompiledGraph<>(this, checkpointStore);
    }
    
    private void validate() {
        if (!nodes.containsKey(entryPoint)) {
            throw new IllegalStateException("Entry point node not found: " + entryPoint);
        }
        if (finishPoint == null) {
            // 如果没有设置结束点，自动使用最后一个节点
            List<String> nodeNames = new ArrayList<>(nodes.keySet());
            this.finishPoint = nodeNames.get(nodeNames.size() - 1);
        }
    }
    
    // ========== 内部类 ==========
    
    @Data
    @AllArgsConstructor
    static class GraphNode<T> {
        private String name;
        private NodeHandler<T> handler;
    }
    
    @Data
    static class Edge<T> {
        private boolean conditional;
        private String targetNode;                        // 固定路由的目标
        private ConditionHandler<T> condition;            // 条件判断函数
        private Map<String, String> pathMap;              // 条件结果 → 目标节点的映射
        
        static <T> Edge<T> fixed(String target) {
            Edge<T> edge = new Edge<>();
            edge.conditional = false;
            edge.targetNode = target;
            return edge;
        }
        
        static <T> Edge<T> conditional(ConditionHandler<T> condition, 
                                        Map<String, String> pathMap) {
            Edge<T> edge = new Edge<>();
            edge.conditional = true;
            edge.condition = condition;
            edge.pathMap = pathMap;
            return edge;
        }
    }
}

/**
 * 节点处理函数 - 接收当前状态，返回修改后的新状态
 */
@FunctionalInterface
public interface NodeHandler<T> {
    T apply(T state) throws Exception;
}

/**
 * 条件判断函数 - 返回一个字符串路由 key
 */
@FunctionalInterface
public interface ConditionHandler<T> {
    String evaluate(T state);
}
```

### 3.2 Checkpoint 持久化

```java
/**
 * Checkpoint - 状态快照
 */
@Data
@AllArgsConstructor
@NoArgsConstructor
@Builder
public class Checkpoint<T> implements Serializable {
    private String id;                      // Checkpoint ID（UUID）
    private String threadId;                // 线程/会话ID
    private String nodeName;                // 当前所在节点
    private T state;                        // 状态对象快照
    private Map<String, Object> metadata;   // 元数据（时间戳、耗时等）
    private LocalDateTime createdAt;        // 创建时间
    private String parentCheckpointId;      // 父 checkpoint ID（形成链）
    private int step;                       // 第几步
}

/**
 * Checkpoint 存储接口
 */
public interface CheckpointStore<T extends Serializable> {
    
    /** 保存 checkpoint */
    void save(Checkpoint<T> checkpoint);
    
    /** 根据 ID 获取 checkpoint */
    Optional<Checkpoint<T>> get(String checkpointId);
    
    /** 获取指定线程的最新 checkpoint */
    Optional<Checkpoint<T>> getLatest(String threadId);
    
    /** 获取指定线程的所有 checkpoints（按时间排序） */
    List<Checkpoint<T>> list(String threadId);
    
    /** 删除指定线程的所有 checkpoints */
    void deleteByThreadId(String threadId);
}

/**
 * 基于 Redis 的 Checkpoint 存储实现
 */
@Slf4j
public class RedisCheckpointStore<T extends Serializable> implements CheckpointStore<T> {
    
    private final RedisTemplate<String, String> redisTemplate;
    private final ObjectMapper objectMapper;
    private final Duration ttl;
    private static final String KEY_PREFIX = "checkpoint:";
    private static final String THREAD_PREFIX = "checkpoint:thread:";
    
    public RedisCheckpointStore(RedisTemplate<String, String> redisTemplate,
                                 ObjectMapper objectMapper,
                                 Duration ttl) {
        this.redisTemplate = redisTemplate;
        this.objectMapper = objectMapper;
        this.ttl = ttl;
    }
    
    @Override
    public void save(Checkpoint<T> checkpoint) {
        try {
            String key = KEY_PREFIX + checkpoint.getId();
            String value = objectMapper.writeValueAsString(checkpoint);
            
            redisTemplate.opsForValue().set(key, value, ttl);
            // 更新线程最新的 checkpoint 引用
            redisTemplate.opsForValue().set(
                    THREAD_PREFIX + checkpoint.getThreadId(), 
                    checkpoint.getId(), ttl);
            
            // 维护线程的 checkpoint 列表（最近 N 个）
            redisTemplate.opsForList().leftPush(
                    KEY_PREFIX + "history:" + checkpoint.getThreadId(), checkpoint.getId());
            redisTemplate.opsForList().trim(
                    KEY_PREFIX + "history:" + checkpoint.getThreadId(), 0, 99);
            redisTemplate.expire(
                    KEY_PREFIX + "history:" + checkpoint.getThreadId(), ttl);
            
        } catch (JsonProcessingException e) {
            log.error("Failed to serialize checkpoint", e);
            throw new RuntimeException("Checkpoint serialization failed", e);
        }
    }
    
    @Override
    public Optional<Checkpoint<T>> get(String checkpointId) {
        try {
            String json = redisTemplate.opsForValue().get(KEY_PREFIX + checkpointId);
            if (json == null) return Optional.empty();
            
            Checkpoint<T> checkpoint = objectMapper.readValue(json, 
                    new TypeReference<Checkpoint<T>>() {});
            return Optional.of(checkpoint);
        } catch (Exception e) {
            log.error("Failed to deserialize checkpoint", e);
            return Optional.empty();
        }
    }
    
    @Override
    public Optional<Checkpoint<T>> getLatest(String threadId) {
        String checkpointId = redisTemplate.opsForValue().get(THREAD_PREFIX + threadId);
        if (checkpointId == null) return Optional.empty();
        return get(checkpointId);
    }
    
    @Override
    public List<Checkpoint<T>> list(String threadId) {
        List<String> ids = redisTemplate.opsForList()
                .range(KEY_PREFIX + "history:" + threadId, 0, -1);
        if (ids == null || ids.isEmpty()) return List.of();
        
        return ids.stream()
                .map(this::get)
                .filter(Optional::isPresent)
                .map(Optional::get)
                .collect(Collectors.toList());
    }
    
    @Override
    public void deleteByThreadId(String threadId) {
        String key = KEY_PREFIX + "history:" + threadId;
        List<String> ids = redisTemplate.opsForList().range(key, 0, -1);
        if (ids != null) {
            ids.forEach(id -> redisTemplate.delete(KEY_PREFIX + id));
        }
        redisTemplate.delete(key);
        redisTemplate.delete(THREAD_PREFIX + threadId);
    }
}

/**
 * 基于内存的 Checkpoint 存储（用于开发/测试）
 */
public class InMemoryCheckpointStore<T extends Serializable> implements CheckpointStore<T> {
    
    private final Map<String, Checkpoint<T>> checkpoints = new ConcurrentHashMap<>();
    private final Map<String, String> latestPerThread = new ConcurrentHashMap<>();
    private final Map<String, List<String>> historyPerThread = new ConcurrentHashMap<>();
    
    @Override
    public void save(Checkpoint<T> checkpoint) {
        checkpoints.put(checkpoint.getId(), checkpoint);
        latestPerThread.put(checkpoint.getThreadId(), checkpoint.getId());
        
        historyPerThread.computeIfAbsent(checkpoint.getThreadId(), 
                        k -> Collections.synchronizedList(new ArrayList<>()))
                .add(0, checkpoint.getId());
    }
    
    @Override
    public Optional<Checkpoint<T>> get(String checkpointId) {
        return Optional.ofNullable(checkpoints.get(checkpointId));
    }
    
    @Override
    public Optional<Checkpoint<T>> getLatest(String threadId) {
        String id = latestPerThread.get(threadId);
        return id != null ? get(id) : Optional.empty();
    }
    
    @Override
    public List<Checkpoint<T>> list(String threadId) {
        List<String> ids = historyPerThread.getOrDefault(threadId, List.of());
        return ids.stream()
                .map(this::get)
                .filter(Optional::isPresent)
                .map(Optional::get)
                .collect(Collectors.toList());
    }
    
    @Override
    public void deleteByThreadId(String threadId) {
        List<String> ids = historyPerThread.remove(threadId);
        if (ids != null) {
            ids.forEach(checkpoints::remove);
        }
        latestPerThread.remove(threadId);
    }
}
```

### 3.3 CompiledGraph——可执行的工作流

```java
/**
 * 编译后的可执行图
 */
@Slf4j
public class CompiledGraph<T extends Serializable> {
    
    private final StateGraph<T> graph;
    private final CheckpointStore<T> checkpointStore;
    private final ObjectMapper objectMapper;
    
    CompiledGraph(StateGraph<T> graph, CheckpointStore<T> checkpointStore) {
        this.graph = graph;
        this.checkpointStore = checkpointStore;
        this.objectMapper = new ObjectMapper();
    }
    
    /**
     * 执行图——从入口开始
     */
    public GraphResult<T> invoke(T initialState, String threadId) {
        String currentNode = graph.entryPoint;
        T state = initialState;
        int step = 0;
        String parentCheckpointId = null;
        
        log.info("Graph execution started: threadId={}, entry={}", threadId, currentNode);
        long graphStartTime = System.currentTimeMillis();
        
        while (true) {
            step++;
            
            // 1. 获取当前节点
            StateGraph.GraphNode<T> node = graph.nodes.get(currentNode);
            if (node == null) {
                return GraphResult.error("Node not found: " + currentNode);
            }
            
            long nodeStartTime = System.currentTimeMillis();
            
            // 2. 执行节点
            T newState;
            try {
                newState = node.getHandler().apply(state);
            } catch (Exception e) {
                log.error("Node {} failed at step {}", currentNode, step, e);
                // 异常时也保存 checkpoint（用于恢复）
                Checkpoint<T> errorCheckpoint = createCheckpoint(
                        threadId, currentNode, state, parentCheckpointId, step);
                errorCheckpoint.getMetadata().put("error", e.getMessage());
                if (checkpointStore != null) checkpointStore.save(errorCheckpoint);
                
                return GraphResult.error("Node execution failed: " + e.getMessage(), state);
            }
            
            long nodeDuration = System.currentTimeMillis() - nodeStartTime;
            log.info("Step {}: {} completed in {}ms", step, currentNode, nodeDuration);
            
            state = newState;
            
            // 3. 保存 Checkpoint
            Checkpoint<T> checkpoint = createCheckpoint(
                    threadId, currentNode, state, parentCheckpointId, step);
            checkpoint.getMetadata().put("duration", nodeDuration);
            if (checkpointStore != null) checkpointStore.save(checkpoint);
            
            parentCheckpointId = checkpoint.getId();
            
            // 4. 检查是否到达终点
            if (currentNode.equals(graph.finishPoint)) {
                log.info("Graph execution completed in {} steps", step);
                long totalDuration = System.currentTimeMillis() - graphStartTime;
                return GraphResult.success(state, step, totalDuration, parentCheckpointId);
            }
            
            // 5. 确定下一个节点
            String nextNode = getNextNode(currentNode, state);
            if (nextNode == null) {
                return GraphResult.error("No next node found from: " + currentNode, state);
            }
            
            currentNode = nextNode;
        }
    }
    
    /**
     * 从指定 Checkpoint 恢复执行
     */
    public GraphResult<T> resume(String checkpointId) {
        if (checkpointStore == null) {
            return GraphResult.error("No checkpoint store configured, cannot resume");
        }
        
        Optional<Checkpoint<T>> checkpointOpt = checkpointStore.get(checkpointId);
        if (checkpointOpt.isEmpty()) {
            return GraphResult.error("Checkpoint not found: " + checkpointId);
        }
        
        Checkpoint<T> checkpoint = checkpointOpt.get();
        String currentNode = checkpoint.getNodeName();
        T state = checkpoint.getState();
        int step = checkpoint.getStep();
        
        log.info("Resuming from checkpoint {} (step {}, node {})", 
                checkpointId, step, currentNode);
        
        // 继续执行——从当前 Checkpoint 所在节点的下一个节点开始
        // 注意：这里我们需要重新确定下一个节点，因为状态可能已经更新
        
        String nextNode = getNextNode(currentNode, state);
        if (nextNode == null) {
            return GraphResult.error("No next node from resumed point");
        }
        
        // 继续执行后续节点（复用 invoke 逻辑的循环）
        currentNode = nextNode;
        String parentCheckpointId = checkpointId;
        
        while (true) {
            step++;
            StateGraph.GraphNode<T> node = graph.nodes.get(currentNode);
            
            T newState;
            try {
                newState = node.getHandler().apply(state);
            } catch (Exception e) {
                log.error("Node {} failed at step {}", currentNode, step, e);
                return GraphResult.error("Resumed execution failed: " + e.getMessage(), state);
            }
            
            state = newState;
            Checkpoint<T> cp = createCheckpoint(
                    checkpoint.getThreadId(), currentNode, state, parentCheckpointId, step);
            if (checkpointStore != null) checkpointStore.save(cp);
            parentCheckpointId = cp.getId();
            
            if (currentNode.equals(graph.finishPoint)) {
                return GraphResult.success(state, step, 0, parentCheckpointId);
            }
            
            nextNode = getNextNode(currentNode, state);
            if (nextNode == null) {
                return GraphResult.error("No next node found after resume");
            }
            currentNode = nextNode;
        }
    }
    
    /**
     * 获取下一个节点——先检查条件边，再查找固定边
     */
    private String getNextNode(String currentNode, T state) {
        List<StateGraph.Edge<T>> outgoingEdges = graph.edges.get(currentNode);
        if (outgoingEdges == null || outgoingEdges.isEmpty()) {
            return null;
        }
        
        for (StateGraph.Edge<T> edge : outgoingEdges) {
            if (edge.isConditional()) {
                // 条件边：根据条件判断结果选择目标
                String route = edge.getCondition().evaluate(state);
                if (route == null) continue;
                
                String target = edge.getPathMap().get(route);
                if (target != null) {
                    log.debug("Conditional route: {} → {} → {}", currentNode, route, target);
                    return target;
                }
            } else {
                return edge.getTargetNode();
            }
        }
        
        return null;
    }
    
    private Checkpoint<T> createCheckpoint(String threadId, String nodeName, 
                                             T state, String parentId, int step) {
        return Checkpoint.<T>builder()
                .id(UUID.randomUUID().toString())
                .threadId(threadId)
                .nodeName(nodeName)
                .state(deepCopy(state))
                .parentCheckpointId(parentId)
                .step(step)
                .createdAt(LocalDateTime.now())
                .metadata(new HashMap<>())
                .build();
    }
    
    @SuppressWarnings("unchecked")
    private T deepCopy(T state) {
        try {
            String json = objectMapper.writeValueAsString(state);
            return objectMapper.readValue(json, (Class<T>) state.getClass());
        } catch (JsonProcessingException e) {
            log.warn("Deep copy failed, using shallow copy", e);
            return state;
        }
    }
}

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
class GraphResult<T> {
    private boolean success;
    private T finalState;
    private int steps;
    private long durationMs;
    private String lastCheckpointId;
    private String error;
    
    public static <T> GraphResult<T> success(T state, int steps, long duration, String checkpointId) {
        return GraphResult.<T>builder()
                .success(true)
                .finalState(state)
                .steps(steps)
                .durationMs(duration)
                .lastCheckpointId(checkpointId)
                .build();
    }
    
    public static <T> GraphResult<T> error(String error) {
        return GraphResult.<T>builder().success(false).error(error).build();
    }
    
    public static <T> GraphResult<T> error(String error, T state) {
        return GraphResult.<T>builder().success(false).error(error).finalState(state).build();
    }
}
```

---

## 四、实战案例：代码审查工作流

让我们用 Java LangGraph 构建一个完整的代码审查 Agent：

```java
/**
 * 代码审查工作流状态
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
class CodeReviewState implements Serializable {
    private String prUrl;
    private String codeContent;
    private List<String> issues;
    private String reviewSummary;
    private String riskLevel;       // low, medium, high
    private boolean autoMerge;
    private String mergeResult;
}

public class CodeReviewWorkflow {
    
    public static void main(String[] args) {
        // 构建 StateGraph
        StateGraph<CodeReviewState> graph = new StateGraph<>("fetch_code");
        
        // 节点 1：拉取代码
        graph.addNode("fetch_code", state -> {
            System.out.println("📥 Fetching code from PR...");
            state.setCodeContent(simulateFetchCode(state.getPrUrl()));
            state.setIssues(new ArrayList<>());
            return state;
        });
        
        // 节点 2：AI 代码审查
        graph.addNode("review_code", state -> {
            System.out.println("🔍 Reviewing code...");
            String review = simulateAIReview(state.getCodeContent());
            state.setReviewSummary(review);
            state.setRiskLevel(estimateRisk(review));
            state.getIssues().addAll(extractIssues(review));
            return state;
        });
        
        // 节点 3：静态分析检查
        graph.addNode("static_analysis", state -> {
            System.out.println("📊 Running static analysis...");
            List<String> saIssues = simulateStaticAnalysis(state.getCodeContent());
            state.getIssues().addAll(saIssues);
            return state;
        });
        
        // 节点 4：合并决策（只有低风险才自动合并）
        graph.addNode("merge_decision", state -> {
            System.out.println("🤔 Making merge decision...");
            boolean shouldAutoMerge = "low".equals(state.getRiskLevel())
                    && state.getIssues().size() < 3;
            state.setAutoMerge(shouldAutoMerge);
            
            if (shouldAutoMerge) {
                state.setMergeResult("AUTO_MERGED");
            } else if ("high".equals(state.getRiskLevel())) {
                state.setMergeResult("BLOCKED");
            } else {
                state.setMergeResult("NEEDS_REVIEW");
            }
            return state;
        });
        
        // 节点 5：通知
        graph.addNode("notify", state -> {
            System.out.println("📬 Notifying team...");
            String channel = "AUTO_MERGED".equals(state.getMergeResult()) 
                    ? "slack" : "email";
            System.out.println("  Sent via " + channel + ": PR review complete - " 
                    + state.getMergeResult());
            return state;
        });
        
        // 定义流转边
        graph.addEdge("fetch_code", "review_code");
        graph.addEdge("review_code", "static_analysis");
        
        // 条件分支：根据风险等级决定后续流程
        graph.addConditionalEdge("static_analysis", state -> state.getRiskLevel(),
                Map.of(
                        "low", "merge_decision",
                        "medium", "merge_decision",
                        "high", "notify"        // 高风险直接通知
                ));
        
        graph.addEdge("merge_decision", "notify");
        graph.setFinishPoint("notify");
        
        // 编译并执行
        CheckpointStore<CodeReviewState> checkpointStore = 
                new InMemoryCheckpointStore<>();
        CompiledGraph<CodeReviewState> compiled = graph.compile(checkpointStore);
        
        CodeReviewState initialState = CodeReviewState.builder()
                .prUrl("https://github.com/company/repo/pull/1234")
                .build();
        
        GraphResult<CodeReviewState> result = compiled.invoke(initialState, "thread-001");
        
        System.out.println("\n=== Execution Result ===");
        System.out.println("Success: " + result.isSuccess());
        System.out.println("Steps: " + result.getSteps());
        System.out.println("Duration: " + result.getDurationMs() + "ms");
        System.out.println("Issues found: " + result.getFinalState().getIssues().size());
        System.out.println("Merge Result: " + result.getFinalState().getMergeResult());
        System.out.println("Last Checkpoint: " + result.getLastCheckpointId());
        
        // 演示断点恢复
        System.out.println("\n=== Simulating Resume ===");
        Optional<Checkpoint<CodeReviewState>> checkpointOpt = 
                checkpointStore.getLatest("thread-001");
        if (checkpointOpt.isPresent()) {
            Checkpoint<CodeReviewState> cp = checkpointOpt.get();
            System.out.println("Resuming from checkpoint: " + cp.getId());
            System.out.println("  Node: " + cp.getNodeName());
            System.out.println("  Step: " + cp.getStep());
        }
    }
    
    // ========== 模拟函数 ==========
    
    private static String simulateFetchCode(String prUrl) {
        return "public class UserService {\n"
                + "    public User getUser(Long id) {\n"
                + "        return userRepo.findById(id).orElse(null);\n"
                + "    }\n"
                + "    public void deleteUser(Long id) {\n"
                + "        userRepo.deleteById(id);\n"
                + "        // TODO: add audit log\n"
                + "    }\n"
                + "}";
    }
    
    private static String simulateAIReview(String code) {
        return "REVIEW: \n"
                + "- [CRITICAL] deleteUser 方法缺少权限检查\n"
                + "- [WARNING] getUser 返回 null 建议改为 Optional\n"
                + "- [INFO] TODO 未完成\n"
                + "RISK: medium";
    }
    
    private static String estimateRisk(String review) {
        if (review.contains("CRITICAL")) return "high";
        if (review.contains("WARNING")) return "medium";
        return "low";
    }
    
    private static List<String> extractIssues(String review) {
        return Arrays.stream(review.split("\n"))
                .filter(l -> l.startsWith("- ["))
                .collect(Collectors.toList());
    }
    
    private static List<String> simulateStaticAnalysis(String code) {
        return List.of("[WARNING] Line 6: 缺少 @Transactional 注解");
    }
}
```

---

## 五、进阶：与 Spring AI 集成

```java
@Configuration
public class LangGraphConfig {
    
    @Bean
    public StateGraph<ReviewState> reviewWorkflow(ChatClient chatClient) {
        StateGraph<ReviewState> graph = new StateGraph<>("analyze");
        
        graph.addNode("analyze", state -> {
            String prompt = "Analyze this code and return issues as JSON:\n" 
                    + state.getCodeContent();
            String response = chatClient.call(prompt);
            state.setAnalysis(response);
            return state;
        });
        
        graph.addConditionalEdge("analyze", 
            state -> state.getAnalysis().contains("CRITICAL") ? "block" : "approve",
            Map.of("block", "reject", "approve", "merge"));
        
        graph.addNode("reject", state -> { 
            state.setStatus("REJECTED"); 
            return state; 
        });
        graph.addNode("merge", state -> { 
            state.setStatus("APPROVED"); 
            return state; 
        });
        
        graph.setFinishPoint("merge");
        // 隐式：reject 也是结束点
        graph.setFinishPoint("reject");
        
        return graph;
    }
    
    @Bean
    public CompiledGraph<ReviewState> compiledReviewWorkflow(
            StateGraph<ReviewState> graph, 
            RedisCheckpointStore<ReviewState> checkpointStore) {
        return graph.compile(checkpointStore);
    }
}
```

---

## 六、总结

LangGraph 的核心理念在 Java 中完全可行：

1. **StateGraph**：有向图 + 状态共享 + 节点处理函数
2. **Checkpoint**：每个节点执行后的状态快照，支持 Redis/DB/内存存储
3. **条件分支**：`ConditionHandler` 接口 + `pathMap` 实现动态路由
4. **断点恢复**：基于 checkpoint 的 `resume()` 方法

**何时使用 LangGraph 模式**：
- 多步骤工作流（代码审查、CI/CD、审批流程）
- 需要断点恢复的长流程（执行超过 10 秒的任务）
- 需要回溯/回放的场景（调试 AI 决策过程）
- 人机协作的场景（等待人工审批的步骤）

---

**下篇预告**：《CrewAI 解析：角色扮演式多 Agent 框架的 Java 化思考》——CrewAI 是当前最火的多 Agent 框架，它的核心理念是什么？角色、工具、任务、协作流程如何在 Java 中优雅实现？带你用 Java 实现一个 AI 开发团队！
