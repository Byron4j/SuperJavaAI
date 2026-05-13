# Agent 分层架构：Master Agent + Sub Agent 的任务分配策略，让AI学会"管理"其他AI

## 前言

前面三篇文章，我们分别聊了单个 Agent 的状态管理、多 Agent 的消息协作、辩论模式的质量提升。但如果你要做一个真正复杂的企业级 AI 应用——比如一个能自动完成整个 Sprint 开发任务的 AI 团队——你会发现，光有消息总线还不够。

你需要一个"管理者"：谁来把一个大任务拆成小任务？谁来决定哪个 Agent 做什么？谁来监控执行进度？Agent 执行失败了谁来兜底？

答案就是**分层架构**：Master Agent（管理 Agent）+ Sub Agent（执行 Agent）。这篇文章带你从零实现一个完整的 Agent 分层管理框架，让 AI 学会像技术主管一样管理其他 AI。

## 一、分层架构设计

### 1.1 为什么需要分层？

想象你有一个复杂的业务需求："给用户管理系统增加基于角色的权限控制（RBAC）"。直接丢给一个 Agent，它可能：
- 直接写一堆 if-else（没有抽象）
- 遗漏了缓存权限、动态权限等细节
- 代码和测试混在一起

但如果交给 Master Agent：
```
Master Agent 拆解任务:
  ├── SubTask1: 数据模型设计（User、Role、Permission 表结构）
  ├── SubTask2: RBAC 核心逻辑实现（权限检查引擎）
  ├── SubTask3: API 接口开发（绑定/解绑角色接口）
  └── SubTask4: 集成测试（权限回归测试）

Master Agent 分配任务:
  ├── SubTask1 → DBAgent（擅长数据建模）
  ├── SubTask2 → CoderAgent（擅长业务逻辑）
  ├── SubTask3 → CoderAgent
  └── SubTask4 → TesterAgent（擅长测试）

Master Agent 监控 & 汇总:
  ├── SubTask1 完成 → 检查输出格式 → 通过 ✓
  ├── SubTask2 完成 → 单元测试覆盖 → 通过 ✓
  ├── SubTask3 完成 → 接口格式正确 → 通过 ✓
  └── SubTask4 完成 → 所有用例通过 → 通过 ✓
  
最终: 汇总输出，交付用户
```

这就是分层架构的核心价值：**把"怎么做"和"谁来做"从 Agent 身上剥离出来，交给 Master Agent 统一调度。**

### 1.2 架构全景图

```
                         ┌──────────────────────┐
                         │      用户请求          │
                         └──────────┬───────────┘
                                    │
                         ┌──────────▼───────────┐
                         │    Master Agent      │
                         │  ┌────────────────┐  │
                         │  │ 1. 任务分解器   │  │
                         │  │ 2. 任务调度器   │  │
                         │  │ 3. 进度监控器   │  │
                         │  │ 4. 质量检查器   │  │
                         │  │ 5. 错误恢复器   │  │
                         │  └────────────────┘  │
                         └──┬───────┬───────┬───┘
                            │       │       │
              ┌─────────────┼───────┼───────┼─────────────┐
              │             │       │       │             │
         ┌────▼────┐  ┌────▼────┐ ┌▼──────┐ ┌────▼────┐
         │ DBAgent │  │CoderAgent│ │Tester │ │DocAgent │
         │ (SQL)   │  │ (Java)  │ │Agent  │ │ (MD)    │
         └─────────┘  └─────────┘ └───────┘ └─────────┘
```

Master Agent 不执行具体业务逻辑，它只做四件事：
1. **分解**：把大任务拆成小任务（子任务拆分）
2. **分配**：把子任务发给最合适的 Sub Agent（负载均衡 + 能力匹配）
3. **监控**：实时跟踪每个子任务的执行进度（进度条 + 超时处理）
4. **质控**：检查 Sub Agent 的输出是否达标（格式检查 + 内容验证 + 重新分配）

## 二、任务分配策略

### 2.1 三种分配策略

Master Agent 有三种核心的任务分配策略：

**策略1：轮询分配（Round Robin）**
> 适用于 Sub Agent 能力相同的场景。例如 3 个 CoderAgent 实例，轮流分配任务。

```java
public class RoundRobinStrategy implements TaskAssignStrategy {
    private final Map<String, AtomicInteger> counters = new ConcurrentHashMap<>();

    @Override
    public SubAgent selectAgent(List<SubAgent> candidates, Task task) {
        String capability = task.getRequiredCapability();
        List<SubAgent> eligible = filterByCapability(candidates, capability);

        if (eligible.isEmpty()) return null;

        AtomicInteger counter = counters.computeIfAbsent(capability, k -> new AtomicInteger(0));
        int index = counter.getAndIncrement() % eligible.size();
        return eligible.get(index);
    }
}
```

**策略2：能力匹配（Capability Match）**
> 根据 Sub Agent 的技能标签和任务要求进行匹配。例如 SQL 任务优先给 DBAgent。

```java
public class CapabilityMatchStrategy implements TaskAssignStrategy {

    @Override
    public SubAgent selectAgent(List<SubAgent> candidates, Task task) {
        return candidates.stream()
            .filter(a -> a.isOnline() && a.canHandle(task))
            .max(Comparator.comparingInt(a -> a.getCapabilityMatchScore(task)))
            .orElse(null);
    }
}
```

**策略3：负载均衡（Load Balance）**
> 把任务分配给当前负载最低的 Sub Agent。

```java
public class LoadBalanceStrategy implements TaskAssignStrategy {

    @Override
    public SubAgent selectAgent(List<SubAgent> candidates, Task task) {
        return candidates.stream()
            .filter(a -> a.isOnline() && a.canHandle(task))
            .filter(a -> a.getCurrentLoad() < a.getMaxCapacity())
            .min(Comparator.comparingInt(SubAgent::getCurrentLoad))
            .orElse(null);
    }
}
```

### 2.2 任务分配策略接口

```java
public interface TaskAssignStrategy {
    /**
     * 从候选 Agent 中选择最合适的一个来执行任务
     * @param candidates 候选 Agent 列表
     * @param task 待分配的任务
     * @return 选中的 Agent，如果没有合适的返回 null
     */
    SubAgent selectAgent(List<SubAgent> candidates, Task task);
}
```

### 2.3 自适应混合策略

实际生产环境不会只用一种策略，而是混合使用：

```java
public class AdaptiveAssignStrategy implements TaskAssignStrategy {
    private final RoundRobinStrategy roundRobin = new RoundRobinStrategy();
    private final CapabilityMatchStrategy capabilityMatch = new CapabilityMatchStrategy();
    private final LoadBalanceStrategy loadBalance = new LoadBalanceStrategy();

    @Override
    public SubAgent selectAgent(List<SubAgent> candidates, Task task) {
        // 优先能力匹配
        SubAgent agent = capabilityMatch.selectAgent(candidates, task);
        if (agent == null) return null;

        // 如果有多于一个匹配的，再用负载均衡
        List<SubAgent> matched = candidates.stream()
            .filter(a -> a.canHandle(task))
            .collect(Collectors.toList());

        if (matched.size() > 1) {
            agent = loadBalance.selectAgent(matched, task);
        }

        return agent;
    }
}
```

## 三、核心数据模型

### 3.1 Task（任务）

```java
import java.util.*;
import java.time.Instant;

public class Task {
    public enum TaskStatus {
        PENDING,        // 等待分配
        ASSIGNED,       // 已分配
        RUNNING,        // 执行中
        COMPLETED,      // 已完成
        FAILED,         // 已失败
        TIMEOUT,        // 超时
        RETRYING        // 重试中
    }

    public enum TaskPriority {
        LOW(0), NORMAL(1), HIGH(2), CRITICAL(3);
        final int level;
        TaskPriority(int level) { this.level = level; }
    }

    private String taskId;
    private String parentTaskId;       // 父任务ID（null 表示顶层任务）
    private String title;
    private String description;
    private String requiredCapability;  // 需要的 Agent 能力标签
    private TaskPriority priority;
    private TaskStatus status;
    private String assignedAgentId;    // 被分配的 Agent
    private String output;             // 任务输出
    private long createdAt;
    private long deadlineMs;           // 截止时间
    private int retryCount;
    private int maxRetries;
    private List<Task> subTasks;       // 子任务列表
    private Map<String, Object> metadata;

    public Task(String title, String description, String requiredCapability) {
        this.taskId = UUID.randomUUID().toString();
        this.title = title;
        this.description = description;
        this.requiredCapability = requiredCapability;
        this.priority = TaskPriority.NORMAL;
        this.status = TaskStatus.PENDING;
        this.createdAt = System.currentTimeMillis();
        this.retryCount = 0;
        this.maxRetries = 2;
        this.subTasks = new ArrayList<>();
        this.metadata = new HashMap<>();
    }

    public boolean canRetry() { return retryCount < maxRetries; }
    public void incrementRetry() { this.retryCount++; }

    // Getters & Setters
    public String getTaskId() { return taskId; }
    public String getParentTaskId() { return parentTaskId; }
    public void setParentTaskId(String parentTaskId) { this.parentTaskId = parentTaskId; }
    public String getTitle() { return title; }
    public String getDescription() { return description; }
    public String getRequiredCapability() { return requiredCapability; }
    public TaskPriority getPriority() { return priority; }
    public void setPriority(TaskPriority priority) { this.priority = priority; }
    public TaskStatus getStatus() { return status; }
    public void setStatus(TaskStatus status) { this.status = status; }
    public String getAssignedAgentId() { return assignedAgentId; }
    public void setAssignedAgentId(String assignedAgentId) { this.assignedAgentId = assignedAgentId; }
    public String getOutput() { return output; }
    public void setOutput(String output) { this.output = output; }
    public long getCreatedAt() { return createdAt; }
    public long getDeadlineMs() { return deadlineMs; }
    public void setDeadlineMs(long deadlineMs) { this.deadlineMs = deadlineMs; }
    public int getRetryCount() { return retryCount; }
    public int getMaxRetries() { return maxRetries; }
    public void setMaxRetries(int maxRetries) { this.maxRetries = maxRetries; }
    public List<Task> getSubTasks() { return subTasks; }
    public void addSubTask(Task task) {
        task.setParentTaskId(this.taskId);
        this.subTasks.add(task);
    }
    public Map<String, Object> getMetadata() { return metadata; }
    public void addMetadata(String key, Object value) { metadata.put(key, value); }

    public boolean isCompleted() { return status == TaskStatus.COMPLETED; }
    public boolean isFailed() {
        return status == TaskStatus.FAILED || status == TaskStatus.TIMEOUT;
    }
}
```

### 3.2 SubAgent（执行 Agent）

```java
public class SubAgent {
    private String agentId;
    private String name;
    private String role;
    private Set<String> capabilities;       // 能力标签: ["java", "sql", "testing"]
    private int currentLoad;                // 当前负载
    private int maxCapacity;                // 最大容量
    private boolean online;
    private double successRate;             // 历史成功率
    private AgentStateMachine stateMachine; // 复用之前的状态机

    public SubAgent(String agentId, String name, String role) {
        this.agentId = agentId;
        this.name = name;
        this.role = role;
        this.capabilities = new HashSet<>();
        this.currentLoad = 0;
        this.maxCapacity = 5;
        this.online = true;
        this.successRate = 1.0;
    }

    /**
     * 判断是否能处理这个任务
     */
    public boolean canHandle(Task task) {
        return online
            && currentLoad < maxCapacity
            && capabilities.contains(task.getRequiredCapability());
    }

    /**
     * 能力匹配度评分
     */
    public int getCapabilityMatchScore(Task task) {
        int score = 0;
        if (capabilities.contains(task.getRequiredCapability())) {
            score += 10;
        }
        // 检查任务描述中的关键词和能力标签的匹配
        for (String cap : capabilities) {
            if (task.getDescription().toLowerCase().contains(cap.toLowerCase())) {
                score += 5;
            }
        }
        // 成功率加成
        score += (int)(successRate * 10);
        return score;
    }

    // Getters & Setters
    public String getAgentId() { return agentId; }
    public String getName() { return name; }
    public String getRole() { return role; }
    public Set<String> getCapabilities() { return capabilities; }
    public void addCapability(String capability) { capabilities.add(capability); }
    public int getCurrentLoad() { return currentLoad; }
    public void incrementLoad() { this.currentLoad++; }
    public void decrementLoad() { this.currentLoad = Math.max(0, currentLoad - 1); }
    public int getMaxCapacity() { return maxCapacity; }
    public boolean isOnline() { return online; }
    public void setOnline(boolean online) { this.online = online; }
    public double getSuccessRate() { return successRate; }
    public void updateSuccessRate(boolean succeeded) {
        this.successRate = (successRate * 0.9) + (succeeded ? 0.1 : 0);
    }
}
```

## 四、Master Agent 完整实现

### 4.1 MasterAgent 核心类

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.stream.Collectors;

public class MasterAgent {
    private final String masterId;
    private final LLMService llmService;
    private final TaskAssignStrategy assignStrategy;
    private final Map<String, SubAgent> subAgentPool;
    private final Map<String, CompletableFuture<String>> pendingTasks;

    // 监督配置
    private final long defaultTimeoutMs = 120_000;
    private final long progressCheckIntervalMs = 10_000;

    // 统计
    private long totalTasksCompleted = 0;
    private long totalTasksFailed = 0;

    public MasterAgent(LLMService llmService) {
        this.masterId = "master-" + UUID.randomUUID().toString().substring(0, 6);
        this.llmService = llmService;
        this.assignStrategy = new AdaptiveAssignStrategy();
        this.subAgentPool = new ConcurrentHashMap<>();
        this.pendingTasks = new ConcurrentHashMap<>();
    }

    /**
     * 注册 Sub Agent
     */
    public void registerSubAgent(SubAgent agent) {
        subAgentPool.put(agent.getAgentId(), agent);
        System.out.println("[Master] 注册 SubAgent: " + agent.getName()
            + " (" + agent.getRole() + ")");
    }

    /**
     * 执行任务（入口方法）
     */
    public String execute(String userRequest) {
        long startTime = System.currentTimeMillis();
        System.out.println("\n" + "=".repeat(60));
        System.out.println("[Master " + masterId + "] 收到请求: " + userRequest);
        System.out.println("=".repeat(60));

        // Step 1: 任务分解
        System.out.println("\n[Step 1] 任务分解...");
        Task rootTask = decomposeTask(userRequest);
        System.out.println("   分解出 " + countAllTasks(rootTask) + " 个子任务");

        // Step 2: 编排执行
        System.out.println("\n[Step 2] 开始编排执行...");
        executeWithDependencyCheck(rootTask);

        // Step 3: 质量检查
        System.out.println("\n[Step 3] 质量检查...");
        QualityReport qualityReport = checkQuality(rootTask);

        // Step 4: 结果汇总
        String finalResult = summarizeResults(rootTask, qualityReport,
            System.currentTimeMillis() - startTime);

        System.out.println("\n[Master] 任务完成。成功: " + totalTasksCompleted
            + ", 失败: " + totalTasksFailed);

        return finalResult;
    }

    /**
     * 使用 LLM 分解任务
     */
    private Task decomposeTask(String userRequest) {
        String prompt = String.format(
            "你是一个技术项目经理，需要将用户需求分解为可独立执行的子任务。\n\n" +
            "用户需求: %s\n\n" +
            "可用的 Agent 类型和它们的能力:\n" +
            "- DBAgent: 数据库设计、SQL编写、表结构设计\n" +
            "- CoderAgent: Java代码编写、业务逻辑实现\n" +
            "- TesterAgent: 单元测试、集成测试、测试用例编写\n" +
            "- DocAgent: 文档编写、API文档生成\n\n" +
            "请按以下格式将任务分解为子任务:\n" +
            "## 子任务名称\n" +
            "- 描述: 具体要做什么\n" +
            "- 依赖: 依赖哪个子任务（填编号或无）\n" +
            "- 角色: 需要哪个 Agent（DBAgent/CoderAgent/TesterAgent/DocAgent）\n" +
            "- 优先级: HIGH/NORMAL/LOW\n\n" +
            "注意：子任务之间尽量独立，减少依赖。最多拆分 6 个子任务。",
            userRequest
        );

        String llmResponse = llmService.analyze(prompt);
        return parseDecompositionResult(llmResponse);
    }

    /**
     * 解析 LLM 的分解结果
     */
    private Task parseDecompositionResult(String llmResponse) {
        Task rootTask = new Task("根任务", llmResponse, "orchestration");
        String[] sections = llmResponse.split("##\\s+");

        Map<Integer, Task> taskMap = new LinkedHashMap<>();
        int taskIndex = 1;

        for (int i = 1; i < sections.length; i++) {
            String section = sections[i].trim();
            if (section.isEmpty()) continue;

            String[] lines = section.split("\n");
            String title = lines[0].trim();
            String description = "";
            String role = "CoderAgent";
            String dependsOn = "";

            for (int j = 1; j < lines.length; j++) {
                String line = lines[j].trim();
                if (line.startsWith("- 描述:")) {
                    description = line.substring(4).trim();
                } else if (line.startsWith("- 角色:")) {
                    role = line.substring(4).trim();
                } else if (line.startsWith("- 依赖:")) {
                    dependsOn = line.substring(4).trim();
                }
            }

            String capability = mapRoleToCapability(role);
            Task subTask = new Task(title, description, capability);
            taskMap.put(taskIndex, subTask);
            taskIndex++;
        }

        // 处理依赖关系
        for (Map.Entry<Integer, Task> entry : taskMap.entrySet()) {
            rootTask.addSubTask(entry.getValue());
        }

        return rootTask;
    }

    private String mapRoleToCapability(String role) {
        switch (role.toLowerCase()) {
            case "dbagent": return "database";
            case "coderagent": return "coding";
            case "testeragent": return "testing";
            case "docagent": return "documentation";
            default: return "coding";
        }
    }

    /**
     * 按依赖关系执行任务树
     */
    private void executeWithDependencyCheck(Task task) {
        List<Task> subTasks = task.getSubTasks();
        if (subTasks.isEmpty()) {
            // 叶子节点：分配并执行
            assignAndExecute(task);
            return;
        }

        // 递归执行所有子任务
        for (Task subTask : subTasks) {
            executeWithDependencyCheck(subTask);
        }

        // 检查所有子任务是否完成
        boolean allDone = subTasks.stream().allMatch(Task::isCompleted);
        if (allDone) {
            task.setStatus(Task.TaskStatus.COMPLETED);
            // 汇总子任务输出
            String summary = subTasks.stream()
                .map(t -> "## " + t.getTitle() + "\n" + t.getOutput())
                .collect(Collectors.joining("\n\n"));
            task.setOutput(summary);
        }
    }

    /**
     * 分配并执行单个任务
     */
    private void assignAndExecute(Task task) {
        // 1. 选择 Agent
        List<SubAgent> candidates = new ArrayList<>(subAgentPool.values());
        SubAgent selected = assignStrategy.selectAgent(candidates, task);

        if (selected == null) {
            System.err.println("[Master] 没有可用的 Agent 处理任务: " + task.getTitle());
            task.setStatus(Task.TaskStatus.FAILED);
            totalTasksFailed++;
            return;
        }

        // 2. 分配任务
        task.setAssignedAgentId(selected.getAgentId());
        task.setStatus(Task.TaskStatus.ASSIGNED);
        selected.incrementLoad();
        System.out.println("  分配任务 ["
            + task.getTitle() + "] → " + selected.getName());

        // 3. 执行任务（带超时控制）
        CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
            task.setStatus(Task.TaskStatus.RUNNING);
            try {
                // 模拟 Agent 执行（实际替换为 LLM 调用）
                String result = executeSubAgentTask(selected, task);
                task.setOutput(result);
                task.setStatus(Task.TaskStatus.COMPLETED);
                selected.updateSuccessRate(true);
                totalTasksCompleted++;
                System.out.println("  ✓ 任务完成 [" + task.getTitle() + "]");
                return result;
            } catch (Exception e) {
                task.setStatus(Task.TaskStatus.FAILED);
                selected.updateSuccessRate(false);
                System.err.println("  ✗ 任务失败 [" + task.getTitle() + "]: " + e.getMessage());

                // 重试机制
                if (task.canRetry()) {
                    retryTask(task);
                } else {
                    totalTasksFailed++;
                }
                return null;
            } finally {
                selected.decrementLoad();
            }
        });

        pendingTasks.put(task.getTaskId(), future);

        try {
            future.get(task.getDeadlineMs() > 0 ? task.getDeadlineMs() : defaultTimeoutMs,
                       TimeUnit.MILLISECONDS);
        } catch (TimeoutException e) {
            task.setStatus(Task.TaskStatus.TIMEOUT);
            System.err.println("  ⏱ 任务超时 [" + task.getTitle() + "]");
            future.cancel(true);
        } catch (InterruptedException | ExecutionException e) {
            task.setStatus(Task.TaskStatus.FAILED);
            System.err.println("  ✗ 任务异常 [" + task.getTitle() + "]: " + e.getMessage());
        } finally {
            pendingTasks.remove(task.getTaskId());
        }
    }

    /**
     * Sub Agent 执行任务（调用 LLM）
     */
    private String executeSubAgentTask(SubAgent agent, Task task) {
        String prompt = String.format(
            "你是一个%s，你的任务是:%s\n\n任务详情:%s\n\n请直接输出结果，不要解释。",
            agent.getRole(), task.getTitle(), task.getDescription()
        );
        return llmService.generate(prompt);
    }

    /**
     * 重试失败任务
     */
    private void retryTask(Task task) {
        task.incrementRetry();
        task.setStatus(Task.TaskStatus.RETRYING);
        System.out.println("  ↻ 重试任务 (第" + task.getRetryCount()
            + "次) [" + task.getTitle() + "]");

        // 等待一下再重试
        try { Thread.sleep(2000); } catch (InterruptedException ignored) {}

        assignAndExecute(task);
    }

    /**
     * 质量检查
     */
    private QualityReport checkQuality(Task task) {
        QualityReport report = new QualityReport();

        checkAllTasks(task, report);

        System.out.println("  检查了 " + report.totalTasks + " 个任务");
        System.out.println("  通过: " + report.passed + ", 警告: " + report.warnings);

        return report;
    }

    private void checkAllTasks(Task task, QualityReport report) {
        List<Task> subTasks = task.getSubTasks();
        if (subTasks.isEmpty() && task.getOutput() != null) {
            report.totalTasks++;

            boolean passed = validateTaskOutput(task);
            if (passed) {
                report.passed++;
            } else {
                report.warnings++;
                System.out.println("  ⚠ 质量警告: " + task.getTitle());
            }
        }

        for (Task st : subTasks) {
            checkAllTasks(st, report);
        }
    }

    private boolean validateTaskOutput(Task task) {
        String output = task.getOutput();
        if (output == null || output.trim().isEmpty()) {
            return false;
        }
        // 检查输出长度是否合理（至少 30 字符）
        if (output.trim().length() < 30) {
            return false;
        }
        // 检查是否包含明显的错误标记
        String lower = output.toLowerCase();
        if (lower.contains("i cannot") || lower.contains("unable to")
            || lower.contains("i'm sorry") || lower.contains("as an ai")) {
            return false;
        }
        return true;
    }

    /**
     * 结果汇总
     */
    private String summarizeResults(Task rootTask, QualityReport qualityReport, long durationMs) {
        StringBuilder sb = new StringBuilder();
        sb.append("# 任务执行报告\n\n");
        sb.append("## 执行概况\n");
        sb.append("- 总任务数: ").append(countAllTasks(rootTask)).append("\n");
        sb.append("- 成功: ").append(totalTasksCompleted).append("\n");
        sb.append("- 失败: ").append(totalTasksFailed).append("\n");
        sb.append("- 耗时: ").append(durationMs / 1000.0).append("秒\n\n");

        sb.append("## 质量报告\n");
        sb.append("- 检查项: ").append(qualityReport.totalTasks).append("\n");
        sb.append("- 通过: ").append(qualityReport.passed).append("\n");
        sb.append("- 警告: ").append(qualityReport.warnings).append("\n\n");

        sb.append("## 执行详情\n");
        appendTaskDetails(sb, rootTask, 0);

        return sb.toString();
    }

    private void appendTaskDetails(StringBuilder sb, Task task, int depth) {
        String indent = "  ".repeat(depth);
        String statusIcon = switch (task.getStatus()) {
            case COMPLETED -> "✓";
            case FAILED, TIMEOUT -> "✗";
            case RUNNING -> "◎";
            case RETRYING -> "↻";
            default -> "○";
        };
        sb.append(indent).append("- ").append(statusIcon).append(" ")
            .append(task.getTitle()).append("\n");

        for (Task sub : task.getSubTasks()) {
            appendTaskDetails(sb, sub, depth + 1);
        }
    }

    private int countAllTasks(Task task) {
        int count = task.getSubTasks().isEmpty() ? 1 : 0;
        for (Task sub : task.getSubTasks()) {
            count += countAllTasks(sub);
        }
        return count;
    }

    // 内部类
    private static class QualityReport {
        int totalTasks = 0;
        int passed = 0;
        int warnings = 0;
    }
}
```

## 五、Master Agent 的监督机制

### 5.1 进度监控

```java
public class ProgressMonitor {
    private final Map<String, TaskProgress> taskProgressMap = new ConcurrentHashMap<>();
    private final ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);

    public void startMonitoring(Task task, long intervalMs) {
        TaskProgress progress = new TaskProgress(task);
        taskProgressMap.put(task.getTaskId(), progress);

        scheduler.scheduleAtFixedRate(() -> {
            progress.updateElapsed();
            if (progress.getElapsedMs() > progress.getEstimatedMs() * 1.5) {
                System.out.println("[Monitor] ⚠ 任务进度滞后: " + task.getTitle()
                    + " (已耗时 " + progress.getElapsedMs() + "ms, 预估 "
                    + progress.getEstimatedMs() + "ms)");
            }
            if (task.getStatus() == Task.TaskStatus.COMPLETED
                || task.getStatus() == Task.TaskStatus.FAILED) {
                taskProgressMap.remove(task.getTaskId());
                throw new RuntimeException("STOP"); // 停止定时任务
            }
        }, intervalMs, intervalMs, TimeUnit.MILLISECONDS);
    }

    private static class TaskProgress {
        private final Task task;
        private long startTime;
        private long estimatedMs;

        TaskProgress(Task task) {
            this.task = task;
            this.startTime = System.currentTimeMillis();
            this.estimatedMs = 60_000; // 默认预估 1 分钟
        }

        void updateElapsed() {}
        long getElapsedMs() { return System.currentTimeMillis() - startTime; }
        long getEstimatedMs() { return estimatedMs; }
    }
}
```

### 5.2 超时处理

```java
public class TimeoutHandler {
    private final Map<String, Long> deadlines = new ConcurrentHashMap<>();

    /**
     * 注册超时回调
     */
    public void registerTimeout(String taskId, long timeoutMs, Runnable onTimeout) {
        deadlines.put(taskId, System.currentTimeMillis() + timeoutMs);

        new Thread(() -> {
            try {
                Thread.sleep(timeoutMs);
                Long deadline = deadlines.get(taskId);
                if (deadline != null && System.currentTimeMillis() >= deadline) {
                    System.out.println("[Timeout] 任务超时: " + taskId);
                    onTimeout.run();
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }).start();
    }

    public void cancelTimeout(String taskId) {
        deadlines.remove(taskId);
    }
}
```

### 5.3 降级策略

当某个 Sub Agent 不可用时，Master Agent 需要降级：

```java
public class MasterDegradeStrategy {

    /**
     * Agent 不可用时的降级处理
     */
    public SubAgent degradeAgent(Task task, List<SubAgent> available) {
        // 策略1: 找次优 Agent
        SubAgent fallback = findFallbackAgent(task, available);
        if (fallback != null) return fallback;

        // 策略2: 降低任务要求
        Task simplifiedTask = simplifyTask(task);
        return findFallbackAgent(simplifiedTask, available);

        // 策略3: 返回 null，让 Master 标记失败
    }

    private SubAgent findFallbackAgent(Task task, List<SubAgent> available) {
        return available.stream()
            .filter(SubAgent::isOnline)
            .findFirst()
            .orElse(null);
    }

    private Task simplifyTask(Task task) {
        // 降低期望
        task.setMaxRetries(1);
        return task;
    }
}
```

## 六、完整运行示例

```java
public class MasterAgentDemo {
    public static void main(String[] args) {
        // 初始化 LLM
        LLMService llmService = new OpenAILLMService("your-api-key");

        // 创建 Master Agent
        MasterAgent master = new MasterAgent(llmService);

        // 注册 Sub Agent 团队
        SubAgent dbAgent = new SubAgent("db-1", "数据库专家", "DBAgent");
        dbAgent.addCapability("database");

        SubAgent coder1 = new SubAgent("code-1", "后端开发1号", "CoderAgent");
        coder1.addCapability("coding");

        SubAgent coder2 = new SubAgent("code-2", "后端开发2号", "CoderAgent");
        coder2.addCapability("coding");

        SubAgent tester = new SubAgent("test-1", "测试工程师", "TesterAgent");
        tester.addCapability("testing");

        SubAgent docWriter = new SubAgent("doc-1", "文档工程师", "DocAgent");
        docWriter.addCapability("documentation");

        master.registerSubAgent(dbAgent);
        master.registerSubAgent(coder1);
        master.registerSubAgent(coder2);
        master.registerSubAgent(tester);
        master.registerSubAgent(docWriter);

        // 执行复杂任务
        String result = master.execute(
            "为电商系统设计并实现一个优惠券模块，支持满减券、折扣券和运费券，" +
            "包含券的创建、发放、使用、过期逻辑，需要有完整的单元测试"
        );

        System.out.println(result);
    }
}
```

## 七、总结

| 维度 | 要点 |
|------|------|
| **架构** | Master 负责管理，Sub Agent 负责执行 |
| **策略** | 轮询/能力匹配/负载均衡，自适应混合策略最实用 |
| **监督** | 进度监控 + 超时处理 + 质量检查，三层保障 |
| **容错** | 重试 + 降级 + 超时保护，让系统更鲁棒 |
| **本质** | 把"人管人"的管理经验平移给 AI 管 AI |

分层架构的核心理念一句话：**不要指望一个 AI 能做好所有事，让它学会像管理者一样，把任务拆解、分配、监督其他 AI 完成。** 这其实就是 AI 时代的"技术主管"。你的 AI 团队差一个 Master Agent，现在可以补上了。

---

**系列六总结**：到这里，AI Agent 多智能体开发系列四篇文章就全部完成了：
- 第135篇：Agent 状态管理（FSM 控制行为流转）
- 第136篇：Multi-Agent 协作（消息总线 + 角色分工）
- 第137篇：Agent 辩论模式（对抗性思维提升质量）
- 第138篇：Agent 分层架构（Master + Sub Agent 管理策略）

从单个 Agent 的可控性，到多 Agent 的协作效率，再到辩论模式的质量提升，最后到分层架构的管理能力——这四篇文章构成了一个完整的"AI 团队管理"体系。把这些都用上，你的 Agent 系统就不再是一个 demo，而是能真正跑在生产环境的企业级方案了。

---

**下一篇预告**：《系列七：AI Agent 与 RAG 深度整合》—— 如何让 Agent 拥有"企业知识"，从简单的 LLM 对话升级为能理解你公司业务的智能助手。RAG + Agent 的深度结合实战，敬请期待。
