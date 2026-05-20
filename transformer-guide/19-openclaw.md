# 第 16 章 · OpenClaw 详解

---

> OpenClaw 是一个开源的 AI Agent 框架，将前几章讲到的 Function Calling、MCP、Skills 整合为完整可用的 Agent 系统。本章深入其架构设计。

---

## 16.1 OpenClaw 是什么？

OpenClaw 是一个**通用 AI Agent 框架**——把 LLM 从"只会聊天的模型"变成"能操作电脑、执行任务的智能体"。它整合了：

```
OpenClaw = LLM + MCP Tools + Skills + Memory + Planning
```

```java
/**
 * OpenClaw 的核心架构（Java 概念映射）
 */
public class OpenClawArchitecture {

    // ===== 输入层 =====
    record UserRequest(
        String text,                      // 用户自然语言指令
        List<Attachment> attachments,     // 图片、文件等附件
        ConversationContext context        // 前序对话
    ) {}

    // ===== 能力层 =====
    interface Capabilities {
        // 1. 理解 (Understanding)
        void parseIntent(UserRequest request);

        // 2. 规划 (Planning)
        List<ActionPlan> decomposeToSteps(Intent intent);

        // 3. 执行 (Execution)
        ActionResult executeStep(ActionPlan step);

        // 4. 观察 (Observation)
        ObservationResult observe(ActionResult result);

        // 5. 反思 (Reflection)
        void evaluateOutcome(ObservationResult observation);

        // 6. 修正 (Correction)
        ActionPlan adjustPlanIfNeeded(ActionPlan plan, ObservationResult obs);
    }

    // ===== 记忆层 =====
    interface Memory {
        WorkingMemory shortTerm;     // 当前任务上下文
        EpisodicMemory mediumTerm;   // 近期对话历史
        SemanticMemory longTerm;     // 持久化知识和偏好
    }
}
```

---

## 16.2 OpenClaw 核心循环

OpenClaw 使用了经典的 **OODA 循环**（Observe-Orient-Decide-Act），结合了 Agent 领域的 **ReAct 模式**（Reasoning + Acting）：

```java
/**
 * OpenClaw Agent 的核心执行循环
 * 
 * 类比: Spring 的 @Scheduled 定时任务 + 状态机
 */
public class OpenClawAgentLoop {

    private final TransformerDecoder llm;
    private final ToolRegistry tools;
    private final SkillRegistry skills;
    private final MemorySystem memory;

    /**
     * 主循环: OODA + ReAct 融合
     */
    public AgentResult execute(UserRequest request) {
        // 初始化工作记忆
        WorkingMemory wm = new WorkingMemory(request);
        wm.addContext(memory.loadRelevant(request));

        // ===== 主循环 =====
        while (!wm.isComplete() && wm.stepCount() < MAX_STEPS) {

            // 1. OBSERVE: 收集当前状态
            Observation obs = observe(wm, tools, skills);
            // obs = {工具执行结果, 文件内容, 错误信息, ...}

            // 2. ORIENT: 分析状态，形成理解
            Situation sit = orient(wm, obs, llm);
            // sit = "任务完成了一半, 下一步应该..."

            // 3. DECIDE: 决策下一步行动
            Action action = decide(sit, llm, wm.getPlan());
            // action = callTool("git_commit", ...) 或 "任务已完成"

            // 4. ACT: 执行行动
            ActionResult result = act(action, tools);
            wm.record(action, result);

            // 5. REFLECT: 反思是否偏离目标
            if (shouldReflect(wm)) {
                Reflection ref = reflect(wm, llm);
                if (ref.needsAdjustment) {
                    adjustPlan(wm, ref.adjustment);
                }
            }
        }

        // 生成最终结果
        return synthesizeResult(wm, llm);
    }

    // ==================== 各阶段详解 ====================

    /**
     * 1. OBSERVE: 观察当前状态
     */
    private Observation observe(WorkingMemory wm, ToolRegistry tools, SkillRegistry skills) {
        // 收集状态信息
        Map<String, Object> state = new HashMap<>();

        // 已完成步骤的结果
        state.put("action_history", wm.getActionHistory());

        // 可用的工具列表
        state.put("available_tools", tools.listAll());

        // 激活的 Skill
        state.put("active_skill", wm.getActiveSkill());

        // 文件系统状态（如果操作文件）
        state.put("workspace_state", filesystem.snapshot());

        return new Observation(state);
    }

    /**
     * 2. ORIENT: 理解当前位置
     */
    private Situation orient(WorkingMemory wm, Observation obs, TransformerDecoder llm) {
        String prompt = """
            分析当前状态:
            
            原始任务: %s
            已完成步骤: %s
            当前观察: %s
            
            请回答:
            1. 任务完成了百分之几?
            2. 是否有任何错误需要注意?
            3. 下一步合理的行动是什么?
            """.formatted(wm.getOriginalRequest(), wm.getActionHistory(), obs);

        String analysis = llm.generate(prompt);
        return parseSituation(analysis);
    }

    /**
     * 3. DECIDE: 决定下一步行动
     */
    private Action decide(Situation sit, TransformerDecoder llm, Plan plan) {
        if (sit.isComplete()) {
            return Action.done();
        }

        // 让 LLM 选择下一个工具和参数
        String prompt = """
            基于当前理解，决定下一步行动:
            
            任务进度: %s
            计划: %s
            可用工具: %s
            
            返回 JSON: {"tool": "tool_name", "args": {...}}
            """.formatted(sit.progress(), plan, tools.listAll());

        String decision = llm.generate(prompt);
        return parseAction(decision);
    }

    /**
     * 4. ACT: 执行
     */
    private ActionResult act(Action action, ToolRegistry tools) {
        if (action.isDone()) return ActionResult.DONE;

        try {
            Tool tool = tools.get(action.toolName());
            Object rawResult = tool.execute(action.args());
            return ActionResult.success(rawResult);
        } catch (ToolException e) {
            return ActionResult.error(e.getMessage());
        }
    }

    /**
     * 5. REFLECT: 自我反思（周期性触发）
     */
    private Reflection reflect(WorkingMemory wm, TransformerDecoder llm) {
        String prompt = """
            反思当前任务执行情况:
            
            原始目标: %s
            已执行步骤: %s
            最近结果: %s
            
            检查:
            1. 是否偏离了目标?
            2. 是否有更高效的方式?
            3. 是否需要调整计划?
            """.formatted(wm.getOriginalRequest(), wm.getRecentActions(), wm.getRecentResults());

        String reflection = llm.generate(prompt);
        return parseReflection(reflection);
    }
}
```

---

## 16.3 OpenClaw 的工具系统

```java
/**
 * OpenClaw 工具注册表
 * 
 * 工具来源:
 * 1. 内置工具 (文件操作、Shell、网络...)
 * 2. MCP Server 工具 (通过 MCP 协议接入)
 * 3. 自定义工具 (用户注册的函数)
 */
@Service
public class ToolRegistry {

    private final Map<String, Tool> tools = new ConcurrentHashMap<>();

    @PostConstruct
    public void init() {
        // ===== 内置工具 =====
        register("read_file", new ReadFileTool());
        register("write_file", new WriteFileTool());
        register("edit_file", new EditFileTool());
        register("execute_command", new ShellTool());
        register("search_code", new GrepSearchTool());
        register("web_fetch", new WebFetchTool());

        // ===== MCP 工具自动接入 =====
        List<McpClient> mcpClients = discoverMcpServers();
        for (McpClient client : mcpClients) {
            List<ToolDefinition> mcpTools = client.listTools().tools();
            for (ToolDefinition td : mcpTools) {
                register(td.name(), new McpToolAdapter(client, td));
            }
        }
    }

    /**
     * 工具执行（带超时和重试）
     */
    public Object execute(String toolName, Map<String, Object> args, int timeoutSecs) {
        Tool tool = tools.get(toolName);
        if (tool == null) {
            return Map.of("error", "Tool not found: " + toolName);
        }

        return CompletableFuture.supplyAsync(() -> {
            try {
                return tool.execute(args);
            } catch (Exception e) {
                // 重试一次
                try {
                    Thread.sleep(1000);
                    return tool.execute(args);
                } catch (Exception e2) {
                    return Map.of("error", e2.getMessage());
                }
            }
        }).get(timeoutSecs, TimeUnit.SECONDS);
    }
}

/**
 * 工具接口定义
 */
interface Tool {
    String name();
    String description();
    JsonSchema inputSchema();

    @Nullable
    Object execute(Map<String, Object> args);
}

/**
 * 示例工具: 文件读取
 */
class ReadFileTool implements Tool {
    @Override
    public String name() { return "read_file"; }

    @Override
    public String description() { return "读取文件内容"; }

    @Override
    public JsonSchema inputSchema() {
        return JsonSchema.builder()
            .property("path", JsonSchemaType.STRING, "文件绝对路径")
            .property("offset", JsonSchemaType.INTEGER, "起始行号")
            .property("limit", JsonSchemaType.INTEGER, "最大行数")
            .required("path")
            .build();
    }

    @Override
    public Object execute(Map<String, Object> args) {
        String path = (String) args.get("path");
        return Files.readString(Path.of(path));
    }
}
```

---

## 16.4 OpenClaw 的 Skills 集成

```java
/**
 * OpenClaw 将 Skills 作为"预置行为模式"注入 Agent
 */
@Service
public class OpenClawSkillsManager {

    private final SkillRegistry skills;
    private final SkillsEngine engine;

    /**
     * 加载 Skill 库
     */
    @PostConstruct
    public void loadSkills() {
        // 从 skills/ 目录自动加载
        Path skillsDir = Path.of("skills/");
        try (var paths = Files.walk(skillsDir)) {
            paths.filter(p -> p.toString().endsWith(".yaml"))
                 .forEach(this::loadSkill);
        }

        // 内置的核心 Skills
        skills.register("code_explorer", "浏览和理解代码库");
        skills.register("bug_fixer", "定位和修复 Bug");
        skills.register("refactor_engineer", "重构代码");
        skills.register("test_writer", "编写单元测试");
        skills.register("doc_writer", "编写文档");
    }

    /**
     * Skill 路由 —— 自动匹配最适合的 Skill
     */
    public Skill matchSkill(UserRequest request) {
        return engine.routeSkill(request.text());
    }
}
```

---

## 16.5 OpenClaw 的记忆系统

```java
/**
 * 三层记忆架构
 * 
 * 类比:
 *   WorkingMemory = ThreadLocal (请求作用域)
 *   EpisodicMemory = HTTP Session (会话作用域) 
 *   SemanticMemory = Database (持久化)
 */
public class MemorySystem {

    // ===== 层 1: 工作记忆（当前任务）=====
    class WorkingMemory {
        private final UserRequest request;            // 原始请求
        private final List<ActionStep> steps;         // 执行步骤历史
        private final Deque<Observation> obsQueue;     // 观察队列
        private Plan currentPlan;                     // 当前计划
        private int stepCount;
        private static final int MAX_STEPS = 50;

        // 类似 ThreadLocal——每个任务一份
    }

    // ===== 层 2: 情景记忆（对话历史）=====
    class EpisodicMemory {
        private final Map<String, List<Conversation>> sessions;

        // 保存完整对话
        void save(String sessionId, Conversation conv) { ... }
        Conversation load(String sessionId) { ... }

        // 类似 HttpSession——每个会话一份
    }

    // ===== 层 3: 语义记忆（向量数据库存储）=====
    class SemanticMemory {
        private final VectorDatabase vectorDB;

        // 存储长期知识
        void remember(String key, String knowledge, float[] embedding) {
            vectorDB.insert(key, embedding, knowledge);
        }

        // 检索相关知识
        List<String> recall(String query, int topK) {
            float[] queryEmbedding = llm.embed(query);
            return vectorDB.search(queryEmbedding, topK);
        }

        // 类似持久化数据库——跨会话存储
        // 如: 用户偏好、项目结构、常用模式
    }
}
```

---

## 16.6 OpenClaw 的安全沙箱

```java
/**
 * 安全沙箱 —— Agent 必然需要执行代码/Shell 命令，必须隔离
 */
public class SecuritySandbox {

    /**
     * 命令执行前的安全检查
     */
    public CommandValidationResult validateCommand(String command) {
        // 1. 黑名单检查
        List<String> blocked = List.of(
            "rm -rf /", "sudo", "chmod 777", "DROP TABLE",
            "shutdown", "reboot", "mkfs", "dd if="
        );
        for (String b : blocked) {
            if (command.contains(b)) {
                return CommandValidationResult.blocked("危险命令: " + b);
            }
        }

        // 2. 白名单模式（更安全）
        List<String> allowed = List.of("ls", "cat", "grep", "find",
            "git", "npm", "mvn", "gradle", "python", "node");
        String cmdName = command.split("\\s+")[0];
        if (!allowed.contains(cmdName)) {
            return CommandValidationResult.needsApproval("命令不在白名单: " + cmdName);
        }

        // 3. 路径限制
        if (command.contains("..") || command.contains("/etc/")
            || command.contains("/var/") || command.contains("~")) {
            return CommandValidationResult.needsApproval("路径敏感: " + command);
        }

        return CommandValidationResult.allowed();
    }

    /**
     * 权限分级
     */
    enum PermissionLevel {
        READ_ONLY,      // 只能读文件
        READ_WRITE,     // 可以读写工作区
        SHELL_SAFE,     // 可执行白名单命令
        SHELL_FULL,     // 可执行任意命令（需要人工审批）
        NETWORK         // 可以访问网络
    }
}
```

---

## 16.7 OpenClaw 总结

```
OpenClaw = 
    LLM (大脑)
    + Tools / MCP (手)
    + Skills (经验/知识)
    + Memory (记忆)
    + Planning (规划)
    + Reflection (反思)
    + Safety (安全)
    = 一个能自主完成复杂任务的 AI Agent

架构类比:
  LLM        = CPU (中央处理器)
  Tools/MCP  = I/O (输入输出设备)
  Skills     = 函数库 / DLL
  Memory     = RAM + Disk
  Planning   = 操作系统调度器
  Safety     = 防火墙 / SELinux
```

---

> **下一章**：[Hermes Agent 详解](20-hermes-agent.md)
