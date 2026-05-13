# CrewAI 解析：角色扮演式多 Agent 框架的 Java 化思考，用 Java 实现你的 AI 开发团队

## 一、引言：如果 AI 有了分工

你玩过《足球经理》吗？你不是上场踢球的，你是安排阵容的——前锋负责进球，中场负责组织，后卫负责防守，门将负责扑救。每个人有自己的角色、能力和职责，最终靠团队配合赢得比赛。

CrewAI 就是这个思路的 AI 版。它的核心理念：**不再让一个 Agent 做所有事，而是组建一个 AI 团队，每个 Agent 扮演特定的角色，有专属的工具，各司其职，协同完成复杂任务。**

举个例子，一个软件开发任务可以这样分工：

| 角色 | Agent | 工具 | 职责 |
|------|-------|------|------|
| 产品经理 | PM Agent | 需求文档解析 | 分析需求，写 PRD |
| 架构师 | Architect Agent | UML 生成、数据库设计工具 | 设计系统架构 |
| 开发者 | Dev Agent | 代码生成、代码搜索 | 编写业务代码 |
| 测试 | QA Agent | 单元测试生成、API 测试 | 编写和执行测试 |
| DevOps | Ops Agent | kubectl、docker 命令 | 部署和监控 |

CrewAI 在 Python 社区已经爆火，GitHub 33k+ star。今天我们不谈 Python，我们谈——**如何用 Java 实现 CrewAI 的核心思想**。

> 全文约 5000 字。读完你会理解多 Agent 协作的本质，并拥有一套可用的 Java 实现。

---

## 二、CrewAI 核心概念拆解

CrewAI 有四个核心抽象：

```
┌─────────────────────────────────────────────────────┐
│                     Crew（团队）                       │
│   ┌─────────────────────────────────────────────┐    │
│   │              Process（流程）                   │    │
│   │  sequential / hierarchical / parallel       │    │
│   └─────────────────────────────────────────────┘    │
│                                                       │
│   ┌─────────┐  ┌─────────┐  ┌─────────┐             │
│   │ Agent A │  │ Agent B │  │ Agent C │             │
│   │ (角色A) │  │ (角色B) │  │ (角色C) │             │
│   │ + 工具集 │  │ + 工具集 │  │ + 工具集 │             │
│   └────┬───-┘  └───┬─────┘  └───┬─────┘             │
│        │            │            │                   │
│   ┌────▼────────────▼────────────▼─────┐             │
│   │            Task（任务）              │             │
│   │  描述 + 期望输出 + 分配给哪个 Agent  │             │
│   └───────────────────────────────────┘             │
└─────────────────────────────────────────────────────┘
```

**Agent（智能体）**：有角色（Role）、目标（Goal）、背景故事（Backstory）、工具集（Tools）

**Task（任务）**：有描述（Description）、期望输出（Expected Output）、分配 Agent（Assigned Agent）

**Tool（工具）**：Agent 可以调用的外部功能（搜索、代码执行、API 调用等）

**Crew（团队）**：Agent 集合 + Task 集合 + 执行流程（Process）

---

## 三、Java 实现：核心类设计

### 3.1 Agent 定义

```java
/**
 * AI Agent - 角色扮演式的智能体
 */
@Data
@Builder
public class Agent {
    
    /** 角色名称 */
    private String role;
    
    /** 目标 - 这个 Agent 要完成什么 */
    private String goal;
    
    /** 背景故事 - 给 LLM 的角色设定，影响行为风格 */
    private String backstory;
    
    /** 是否允许委托任务给其他 Agent */
    @Builder.Default
    private boolean allowDelegation = false;
    
    /** 是否运行异步执行 */
    @Builder.Default
    private boolean asyncExecution = false;
    
    /** 是否启用记忆（记住之前的交互） */
    @Builder.Default
    private boolean memory = true;
    
    /** LLM 配置（模型、温度等） */
    @Builder.Default
    private LLMConfig llm = LLMConfig.defaults();
    
    /** 该 Agent 可用的工具集 */
    @Singular
    private List<Tool> tools;
    
    /** 最大重试次数 */
    @Builder.Default
    private int maxRetry = 3;
    
    /** 最大迭代次数（ReAct 循环） */
    @Builder.Default
    private int maxIteration = 10;
    
    /** 是否输出详细日志 */
    @Builder.Default
    private boolean verbose = false;
    
    /**
     * 生成该 Agent 的 System Prompt
     */
    public String buildSystemPrompt() {
        return """
            你现在扮演的角色是：%s
            
            你的目标：%s
            
            你的背景和行事风格：%s
            
            ## 工作规则
            1. 严格按照你的角色职责行事
            2. 使用提供的工具完成任务
            3. 输出前确保结果符合你的角色标准
            4. %s 将任务委派给其他 Agent
            5. 如果工具调用失败，尝试其他方法或报告问题
            
            ## 可用工具
            %s
            
            ## 输出要求
            根据任务描述，按照指定格式输出结果。
            """.formatted(
                role,
                goal,
                backstory,
                allowDelegation ? "可以" : "不可以",
                formatTools()
            );
    }
    
    private String formatTools() {
        if (tools == null || tools.isEmpty()) {
            return "无可用的特殊工具，请基于你的知识和角色经验完成任务。";
        }
        
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < tools.size(); i++) {
            Tool tool = tools.get(i);
            sb.append(i + 1).append(". **").append(tool.getName()).append("**: ")
                    .append(tool.getDescription()).append("\n");
            sb.append("   参数: ").append(tool.getParameterSchema()).append("\n");
        }
        return sb.toString();
    }
    
    /**
     * 执行 ReAct 循环（思考-行动-观察）
     */
    public AgentExecutionResult execute(Task task, AgentContext context) {
        if (verbose) {
            System.out.println("🤖 Agent [" + role + "] 开始执行任务: " + task.getDescription());
        }
        
        StringBuilder thoughtLog = new StringBuilder();
        StringBuilder actionLog = new StringBuilder();
        StringBuilder observationLog = new StringBuilder();
        String finalOutput = null;
        int iterations = 0;
        
        while (iterations < maxIteration) {
            iterations++;
            
            try {
                // ===== Thought: 思考下一步该做什么 =====
                String thought = think(task, context, thoughtLog.toString(), 
                        actionLog.toString(), observationLog.toString());
                
                if (verbose) {
                    System.out.println("  💭 Thought (" + iterations + "): " 
                            + truncate(thought, 100));
                }
                thoughtLog.append("Step ").append(iterations).append(": ")
                        .append(thought).append("\n\n");
                
                // ===== Action: 执行动作 =====
                Action action = decideAction(thought, task, context);
                
                if (action == null || action.getType() == ActionType.FINISH) {
                    finalOutput = thought;
                    break;
                }
                
                if (verbose) {
                    System.out.println("  ⚡ Action: " + action.getType() 
                            + " → " + action.getToolName());
                }
                actionLog.append("Action: ").append(action.getType())
                        .append(" ").append(action.getToolName()).append("\n");
                
                // ===== Observation: 观察结果 =====
                String observation = executeAction(action, task, context);
                
                if (verbose) {
                    System.out.println("  👁️ Observation: " + truncate(observation, 100));
                }
                observationLog.append("Observation: ").append(observation).append("\n\n");
                
                // 检查是否出现致命错误
                if (observation.startsWith("ERROR:")) {
                    break;
                }
                
            } catch (Exception e) {
                if (verbose) {
                    System.out.println("  ❌ Error at iteration " + iterations + ": " 
                            + e.getMessage());
                }
                if (iterations >= maxRetry) {
                    observationLog.append("FATAL: Max retries exceeded - ")
                            .append(e.getMessage()).append("\n");
                    break;
                }
            }
        }
        
        return AgentExecutionResult.builder()
                .agent(this)
                .task(task)
                .output(finalOutput != null ? finalOutput : "未能在最大迭代次数内完成任务")
                .thoughtChain(thoughtLog.toString())
                .iterationCount(iterations)
                .success(finalOutput != null)
                .build();
    }
    
    /**
     * 思考阶段——通过 LLM 决定下一步
     */
    private String think(Task task, AgentContext context, 
                          String thoughtHistory, String actionHistory, 
                          String observationHistory) {
        // 构建 Prompt 让 LLM 进行推理
        String thinkPrompt = buildSystemPrompt() + "\n\n"
                + "## 当前任务\n" + task.getDescription() + "\n"
                + "## 期望输出\n" + task.getExpectedOutput() + "\n\n"
                + "## 历史思考\n" + (thoughtHistory.isEmpty() ? "（首次思考）" : thoughtHistory) + "\n"
                + "## 已执行动作\n" + (actionHistory.isEmpty() ? "（暂无）" : actionHistory) + "\n"
                + "## 观察结果\n" + (observationHistory.isEmpty() ? "（暂无）" : observationHistory) + "\n\n"
                + "请用以下格式输出你的思考：\n"
                + "【当前状态分析】...\n"
                + "【下一步计划】...\n"
                + "【需要使用的工具】...\n"
                + "如果任务已经完成，请输出【任务完成】并给出最终结果。";
        
        return callLLM(thinkPrompt);
    }
    
    /**
     * 决定执行什么动作
     */
    private Action decideAction(String thought, Task task, AgentContext context) {
        if (thought.contains("任务完成") || thought.contains("【任务完成】")) {
            return Action.finish(thought);
        }
        
        // 解析 thought 中的工具调用意图
        for (Tool tool : tools) {
            if (thought.contains(tool.getName())) {
                return Action.useTool(tool, extractToolParams(thought, tool));
            }
        }
        
        // 如果需要委派且允许
        if (thought.contains("委派") && allowDelegation) {
            return Action.delegate(extractDelegateTarget(thought));
        }
        
        return Action.noop("无法确定下一步动作");
    }
    
    /**
     * 执行动作
     */
    private String executeAction(Action action, Task task, AgentContext context) {
        return switch (action.getType()) {
            case USE_TOOL -> executeTool(action.getToolName(), action.getToolParams(), context);
            case DELEGATE -> delegateToAgent(action.getDelegateTarget(), task, context);
            case FINISH -> "任务完成";
            case NOOP -> "INFO: " + action.getMetadata();
        };
    }
    
    private String executeTool(String toolName, Map<String, Object> params, 
                                AgentContext context) {
        Tool tool = tools.stream()
                .filter(t -> t.getName().equals(toolName))
                .findFirst()
                .orElse(null);
        
        if (tool == null) {
            return "ERROR: 未找到工具 " + toolName;
        }
        
        try {
            return tool.execute(params);
        } catch (Exception e) {
            return "ERROR: 工具执行失败 - " + e.getMessage();
        }
    }
    
    private String delegateToAgent(String targetRole, Task task, AgentContext context) {
        Agent targetAgent = context.findAgentByRole(targetRole);
        if (targetAgent == null) {
            return "ERROR: 未找到 Agent: " + targetRole;
        }
        
        Task subTask = Task.builder()
                .description(task.getDescription() + "（由 " + targetRole + " 协助完成）")
                .expectedOutput(task.getExpectedOutput())
                .assignedAgent(targetAgent)
                .build();
        
        AgentExecutionResult result = targetAgent.execute(subTask, context);
        return "委派结果 [" + targetRole + "]: " + result.getOutput();
    }
    
    private Map<String, Object> extractToolParams(String thought, Tool tool) {
        // 简化实现：从 LLM 输出中提取参数
        return Map.of("input", thought);
    }
    
    private String extractDelegateTarget(String thought) {
        // 简化实现：从 LLM 输出中解析委派目标
        return "开发者";
    }
    
    private String callLLM(String prompt) {
        // 调用 OpenAI / 兼容 API
        // 实际项目中注入 ChatClient
        return "LLM response for: " + prompt.substring(0, 50) + "...";
    }
    
    private String truncate(String text, int maxLen) {
        return text != null && text.length() > maxLen 
                ? text.substring(0, maxLen) + "..." : text;
    }
}

@Data
@Builder
class Action {
    private ActionType type;
    private String toolName;
    private Map<String, Object> toolParams;
    private String delegateTarget;
    private String metadata;
    
    static Action useTool(Tool tool, Map<String, Object> params) {
        return Action.builder()
                .type(ActionType.USE_TOOL)
                .toolName(tool.getName())
                .toolParams(params)
                .build();
    }
    
    static Action delegate(String targetRole) {
        return Action.builder()
                .type(ActionType.DELEGATE)
                .delegateTarget(targetRole)
                .build();
    }
    
    static Action finish(String thought) {
        return Action.builder()
                .type(ActionType.FINISH)
                .metadata(thought)
                .build();
    }
    
    static Action noop(String reason) {
        return Action.builder()
                .type(ActionType.NOOP)
                .metadata(reason)
                .build();
    }
}

enum ActionType { USE_TOOL, DELEGATE, FINISH, NOOP }

@Data
@Builder
class LLMConfig {
    private String model;
    private double temperature;
    private int maxTokens;
    
    static LLMConfig defaults() {
        return LLMConfig.builder()
                .model("gpt-4-turbo")
                .temperature(0.3)
                .maxTokens(2000)
                .build();
    }
}

@Data
@Builder
class AgentExecutionResult {
    private Agent agent;
    private Task task;
    private String output;
    private String thoughtChain;
    private int iterationCount;
    private boolean success;
}
```

### 3.2 Task 与 Tool

```java
/**
 * 任务定义
 */
@Data
@Builder
public class Task {
    
    /** 任务描述（告诉 Agent 要做什么） */
    private String description;
    
    /** 期望输出格式（告诉 Agent 输出长什么样） */
    private String expectedOutput;
    
    /** 分配给哪个 Agent 执行 */
    private Agent assignedAgent;
    
    /** 依赖的前置任务（必须先完成） */
    @Singular
    private List<Task> dependencies;
    
    /** 上下文（之前任务的输出，传入当前任务） */
    @Builder.Default
    private Map<String, Object> context = new HashMap<>();
    
    /** 是否异步执行 */
    @Builder.Default
    private boolean async = false;
    
    /** 回调函数——任务完成后执行 */
    private Consumer<AgentExecutionResult> callback;
}

/**
 * 工具抽象
 */
@FunctionalInterface
public interface Tool {
    
    /** 工具名称 */
    String getName();
    
    /** 工具描述 */
    String getDescription();
    
    /** 参数 Schema */
    default String getParameterSchema() {
        return "{\"input\": \"string\"}";
    }
    
    /** 执行工具 */
    String execute(Map<String, Object> params) throws Exception;
    
    /**
     * 创建简单工具
     */
    static Tool of(String name, String description, Function<Map<String, Object>, String> executor) {
        return new Tool() {
            @Override
            public String getName() { return name; }
            @Override
            public String getDescription() { return description; }
            @Override
            public String execute(Map<String, Object> params) throws Exception {
                return executor.apply(params);
            }
        };
    }
    
    /**
     * 预置工具 —— 代码搜索
     */
    static Tool codeSearch() {
        return Tool.of("CodeSearch", "搜索代码库中与输入关键词相关的代码文件",
                params -> {
                    String keyword = (String) params.get("keyword");
                    return "找到 3 个相关文件: UserService.java, UserController.java, UserMapper.java";
                });
    }
    
    /**
     * 预置工具 —— 代码生成
     */
    static Tool codeGenerator() {
        return Tool.of("CodeGenerator", "根据需求描述生成 Java 代码",
                params -> {
                    String requirement = (String) params.get("requirement");
                    return """
                        ```java
                        public class GeneratedService {
                            public String process(String input) {
                                return "Processed: " + input;
                            }
                        }
                        ```""";
                });
    }
    
    /**
     * 预置工具 —— 单元测试生成
     */
    static Tool testGenerator() {
        return Tool.of("TestGenerator", "为给定的 Java 类生成单元测试",
                params -> {
                    String className = (String) params.get("className");
                    return """
                        ```java
                        @Test
                        void testProcess() {
                            GeneratedService service = new GeneratedService();
                            assertEquals("Processed: hello", service.process("hello"));
                        }
                        ```""";
                });
    }
    
    /**
     * 预置工具 —— 数据库查询
     */
    static Tool dbQuery() {
        return Tool.of("DBQuery", "执行数据库查询并返回结果",
                params -> {
                    String sql = (String) params.get("sql");
                    return "[{\"id\": 1, \"name\": \"示例数据\"}]";
                });
    }
}
```

### 3.3 Crew（团队）与 Process（流程）

```java
/**
 * Crew - Agent 团队的控制器
 * 负责调度 Agent 执行任务，管理执行流程
 */
@Slf4j
public class Crew {
    
    private final String name;
    private final List<Agent> agents;
    private final List<Task> tasks;
    private final ProcessType process;
    private final boolean verbose;
    private final boolean memory;
    
    @Builder
    public Crew(String name, List<Agent> agents, List<Task> tasks, 
                ProcessType process, boolean verbose, boolean memory) {
        this.name = name;
        this.agents = agents;
        this.tasks = tasks;
        this.process = process != null ? process : ProcessType.SEQUENTIAL;
        this.verbose = verbose;
        this.memory = memory;
    }
    
    /**
     * 启动团队执行
     */
    public CrewResult kickoff() {
        if (verbose) {
            System.out.println("🚀 Crew [" + name + "] 开始执行");
            System.out.println("  流程: " + process);
            System.out.println("  Agent 数量: " + agents.size());
            System.out.println("  任务数量: " + tasks.size());
        }
        
        long startTime = System.currentTimeMillis();
        
        // 创建执行上下文
        AgentContext context = new AgentContext(agents, memory);
        
        // 根据流程类型执行任务
        Map<Task, AgentExecutionResult> results = switch (process) {
            case SEQUENTIAL -> executeSequential(tasks, context);
            case HIERARCHICAL -> executeHierarchical(tasks, context);
            case PARALLEL -> executeParallel(tasks, context);
        };
        
        long totalTime = System.currentTimeMillis() - startTime;
        
        if (verbose) {
            System.out.println("✅ Crew [" + name + "] 执行完毕，耗时: " + totalTime + "ms");
        }
        
        return CrewResult.builder()
                .crewName(name)
                .results(results)
                .totalTimeMs(totalTime)
                .success(results.values().stream().allMatch(AgentExecutionResult::isSuccess))
                .build();
    }
    
    /**
     * 顺序执行——任务按顺序一个个执行，后一个任务可以使用前一个任务的输出
     */
    private Map<Task, AgentExecutionResult> executeSequential(List<Task> tasks, 
                                                               AgentContext context) {
        Map<Task, AgentExecutionResult> results = new LinkedHashMap<>();
        String previousOutput = "";
        
        for (int i = 0; i < tasks.size(); i++) {
            Task task = tasks.get(i);
            
            // 当前一个任务的输出传递给当前任务
            if (!previousOutput.isEmpty()) {
                task.getContext().put("previous_output", previousOutput);
            }
            
            if (verbose) {
                System.out.println("  📋 Task " + (i + 1) + "/" + tasks.size() + ": " 
                        + task.getDescription());
            }
            
            Agent assignedAgent = task.getAssignedAgent();
            if (assignedAgent == null) {
                assignedAgent = selectBestAgent(task);
            }
            
            AgentExecutionResult result = assignedAgent.execute(task, context);
            results.put(task, result);
            previousOutput = result.getOutput();
            
            if (task.getCallback() != null) {
                task.getCallback().accept(result);
            }
        }
        
        return results;
    }
    
    /**
     * 层级执行——有一个管理者 Agent 分配任务
     */
    private Map<Task, AgentExecutionResult> executeHierarchical(List<Task> tasks, 
                                                                 AgentContext context) {
        Map<Task, AgentExecutionResult> results = new LinkedHashMap<>();
        
        // 找一个管理者 Agent（通常是第一个 Agent）
        Agent manager = agents.get(0);
        
        // 管理者分析所有任务并分配
        for (Task task : tasks) {
            Agent bestAgent = selectBestAgent(task);
            task.setAssignedAgent(bestAgent);
            
            if (verbose) {
                System.out.println("  🎯 Manager [" + manager.getRole() 
                        + "] 分配任务给 [" + bestAgent.getRole() + "]: " 
                        + task.getDescription());
            }
        }
        
        // 执行分配好的任务
        return executeSequential(tasks, context);
    }
    
    /**
     * 并行执行——独立任务并发执行
     */
    private Map<Task, AgentExecutionResult> executeParallel(List<Task> tasks, 
                                                              AgentContext context) {
        Map<Task, AgentExecutionResult> results = new ConcurrentHashMap<>();
        List<CompletableFuture<Void>> futures = new ArrayList<>();
        
        for (Task task : tasks) {
            futures.add(CompletableFuture.runAsync(() -> {
                Agent assignedAgent = task.getAssignedAgent() != null 
                        ? task.getAssignedAgent() : selectBestAgent(task);
                AgentExecutionResult result = assignedAgent.execute(task, context);
                results.put(task, result);
            }));
        }
        
        CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
        return results;
    }
    
    /**
     * 根据任务描述智能选择最匹配的 Agent
     */
    private Agent selectBestAgent(Task task) {
        String taskDesc = task.getDescription().toLowerCase();
        
        return agents.stream()
                .max(Comparator.comparingDouble(agent -> {
                    double score = 0;
                    // 角色匹配
                    if (taskDesc.contains(agent.getRole().toLowerCase())) score += 10;
                    // 目标匹配
                    for (String word : agent.getGoal().split("\\s+")) {
                        if (taskDesc.contains(word.toLowerCase())) score += 2;
                    }
                    // 工具匹配
                    for (Tool tool : agent.getTools()) {
                        if (taskDesc.contains(tool.getName().toLowerCase())) score += 5;
                        if (taskDesc.contains(tool.getDescription().toLowerCase())) score += 3;
                    }
                    return score;
                }))
                .orElse(agents.get(0));
    }
}

enum ProcessType {
    SEQUENTIAL,    // 顺序执行
    HIERARCHICAL,  // 层级执行（管理者分配）
    PARALLEL        // 并行执行
}

@Data
@Builder
class CrewResult {
    private String crewName;
    private Map<Task, AgentExecutionResult> results;
    private long totalTimeMs;
    private boolean success;
    
    public String getFinalOutput() {
        return results.values().stream()
                .filter(AgentExecutionResult::isSuccess)
                .map(AgentExecutionResult::getOutput)
                .reduce((a, b) -> b) // 取最后一个任务的输出
                .orElse("所有任务都失败了");
    }
}

/**
 * Agent 执行上下文——共享状态和 Agent 发现
 */
class AgentContext {
    private final List<Agent> allAgents;
    private final boolean memoryEnabled;
    private final Map<String, List<String>> memory;
    
    AgentContext(List<Agent> allAgents, boolean memoryEnabled) {
        this.allAgents = allAgents;
        this.memoryEnabled = memoryEnabled;
        this.memory = new ConcurrentHashMap<>();
    }
    
    Agent findAgentByRole(String role) {
        return allAgents.stream()
                .filter(a -> a.getRole().equals(role))
                .findFirst()
                .orElse(null);
    }
    
    void remember(String agentRole, String interaction) {
        if (!memoryEnabled) return;
        memory.computeIfAbsent(agentRole, k -> new ArrayList<>()).add(interaction);
    }
}
```

---

## 四、实战：组建一个 AI 开发团队

```java
public class AIDevTeam {
    
    public static void main(String[] args) {
        // ========== 定义工具 ==========
        Tool codeSearch = Tool.codeSearch();
        Tool codeGenerator = Tool.codeGenerator();
        Tool testGenerator = Tool.testGenerator();
        
        // ========== 定义 Agent ==========
        
        // PM Agent：分析需求，拆解任务
        Agent pm = Agent.builder()
                .role("产品经理")
                .goal("将业务需求转化为可执行的技术任务，确保需求清晰、验收标准明确")
                .backstory("你是一位经验丰富的技术产品经理，擅长与技术团队沟通，"
                        + "你关注用户价值，同时理解技术实现的复杂度。")
                .tools(List.of())
                .verbose(true)
                .build();
        
        // 架构师 Agent：设计系统方案
        Agent architect = Agent.builder()
                .role("架构师")
                .goal("设计可扩展、高性能的系统架构方案")
                .backstory("你是一位专注于分布式系统的高级架构师，对微服务、"
                        + "消息队列、缓存策略有深入理解。")
                .tools(List.of(codeSearch))
                .verbose(true)
                .build();
        
        // 开发 Agent：实写代码
        Agent developer = Agent.builder()
                .role("开发工程师")
                .goal("根据需求和架构设计编写高质量、可维护的 Java 代码")
                .backstory("你是一位全栈 Java 开发工程师，精通 Spring Boot、"
                        + "MyBatis、Redis 等技术栈，代码风格简洁规范。")
                .tools(List.of(codeSearch, codeGenerator))
                .verbose(true)
                .build();
        
        // 测试 Agent：编写和执行测试
        Agent qa = Agent.builder()
                .role("测试工程师")
                .goal("确保代码质量，编写全面的单元测试和集成测试，发现潜在 Bug")
                .backstory("你是一位严谨的自动化测试工程师，以破坏别人的代码为乐，"
                        + "擅长边界条件测试和异常场景覆盖。")
                .tools(List.of(testGenerator))
                .verbose(true)
                .build();
        
        // ========== 定义任务 ==========
        
        Task task1 = Task.builder()
                .description("分析用户需求：实现一个用户积分系统，用户完成特定行为后获得积分，"
                        + "积分可以兑换优惠券。需要支持积分获取、消费、查询、过期等功能。")
                .expectedOutput("输出 PRD 文档，包含功能清单、业务流程、数据库表设计建议。")
                .assignedAgent(pm)
                .build();
        
        Task task2 = Task.builder()
                .description("基于 PRD 设计积分系统的技术架构，"
                        + "包括模块划分、接口设计、数据库表结构、缓存策略。")
                .expectedOutput("输出架构设计文档，包含类图、接口定义、数据库 DDL。")
                .assignedAgent(architect)
                .build();
        
        Task task3 = Task.builder()
                .description("根据架构设计实现积分系统核心代码，"
                        + "包括积分获取(earn)、消费(consume)、查询(query)、过期(expire)接口。")
                .expectedOutput("输出完整的 Java 代码，Spring Boot Controller + Service + Repository。")
                .assignedAgent(developer)
                .build();
        
        Task task4 = Task.builder()
                .description("为积分系统编写单元测试和集成测试，"
                        + "覆盖正常流程、边界条件、并发场景。")
                .expectedOutput("输出测试代码和测试报告。")
                .assignedAgent(qa)
                .build();
        
        // ========== 组建团队并启动 ==========
        Crew devCrew = Crew.builder()
                .name("AI积分系统开发小组")
                .agents(List.of(pm, architect, developer, qa))
                .tasks(List.of(task1, task2, task3, task4))
                .process(ProcessType.SEQUENTIAL)
                .verbose(true)
                .memory(true)
                .build();
        
        CrewResult result = devCrew.kickoff();
        
        System.out.println("\n========== 团队执行结果 ==========");
        System.out.println("成功: " + result.isSuccess());
        System.out.println("耗时: " + result.getTotalTimeMs() + "ms");
        System.out.println("最终输出:\n" + result.getFinalOutput());
    }
}
```

---

## 五、CrewAI vs 其他多 Agent 框架对比

| 特性 | CrewAI | AutoGen | MetaGPT | 我们的 Java 实现 |
|------|--------|---------|---------|-----------------|
| 角色扮演 | ✓ 核心特性 | ✗ | ✓ | ✓ |
| 工具系统 | ✓ | ✓ | ✓ | ✓ |
| 流程控制 | Sequential/Hierarchical | Group Chat | SOP | Sequential/Hierarchical/Parallel |
| 记忆能力 | ✓ | ✓ | ✓ | ✓ (可选) |
| 委派机制 | ✓ | ✗ | ✓ | ✓ |
| 语言 | Python | Python | Python | **Java** |
| 生产可用 | ✓ | ✓ | 实验 | 可定制 |

---

## 六、总结

CrewAI 的核心理念可以完美迁移到 Java：

1. **角色扮演**（Agent.role + Agent.backstory）：让 LLM 的行为有方向性
2. **工具赋予**（Tool 接口）：让 Agent 能调用外部能力
3. **流程编排**（Crew.process）：Sequential/Hierarchical/Parallel 三种模式
4. **任务链**（Task.context）：任务的输出作为下一个任务的输入

**CrewAI 模式的适用场景**：
- 复杂业务流程的自动化（需求→设计→开发→测试→部署）
- 内容创作管线（策划→撰稿→审核→发布）
- 研究分析（信息采集→分析→报告→评审）

---

**下篇预告**：《Agent 与 RAG 的协同：让 Agent 学会"查资料+做决策"》——光有思考能力不够，Agent 还需要一个知识库。Agent + RAG 的组合模式如何设计？Agent 如何动态决定什么时候需要检索知识？关注我，下一篇全是干货！
