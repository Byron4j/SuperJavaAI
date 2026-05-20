# 第 15 章 · Skills 机制详解

---

> Skills 是"给 AI Agent 预装的专业知识模块"——不只是能调用工具，而是**知道如何组合使用工具完成特定领域的复杂任务**。

---

## 15.1 Skills vs Function Calling vs MCP

```
层级递进关系:

  Function Calling     "我能调用函数"           → 单一原子操作
        ↓
  MCP                  "我用标准协议连接工具"     → 工具的统一接口
        ↓
  Skills               "我知道怎么组合工具做事"    → 领域知识的注入
        ↓
  Agent                "我自主规划并执行任务"     → 端到端的任务完成
```

```java
// 直观对比
// Function Calling: model.call("get_weather", {city: "Beijing"})
//    → 调用一个函数，返回结果

// Skill: "帮我做北京三日游攻略"
//    → 内部调用: 天气查询 + 景点搜索 + 交通查询 + 酒店比价 + 生成行程
//    → 返回: 一份完整的旅行计划（含日程、预算、注意事项）
```

---

## 15.2 Skills 的本质

Skill = **领域特定的 System Prompt + 工具集 + 工作流模板**。

```java
/**
 * Skill 的内部结构
 */
public record Skill(
    String name,                    // 技能名称: "travel_planner"
    String description,             // 描述: 何时触发此技能
    String systemPrompt,            // 领域特定的系统提示
    List<String> requiredTools,     // 需要的工具列表
    List<WorkflowStep> workflow,    // 标准的操作流程
    List<Example> fewShotExamples,  // Few-shot 示例
    Map<String, String> knowledge   // 领域知识库
) {}

/**
 * 一个完整的 Skill 示例: 代码审查
 */
public class CodeReviewSkill {

    public static Skill create() {
        return new Skill(
            "code_reviewer",
            "当用户请求代码审查、PR Review 或代码质量检查时触发",

            // === System Prompt (注入领域知识) ===
            """
            你是一位资深代码审查专家。审查时请关注:
            1. 安全漏洞 (SQL 注入, XSS, 权限绕过, 敏感信息泄露)
            2. 性能问题 (N+1 查询, 不必要的对象创建, 阻塞 I/O)
            3. 代码设计 (SOLID 原则, DRY, 耦合度)
            4. 可读性 (命名, 注释, 函数长度)
            5. 测试覆盖 (边界条件, 异常路径)
            
            对每个问题，请提供:
            - 严重级别: 🔴严重 / 🟡警告 / 🔵建议
            - 问题描述: 一句话
            - 修复建议: 代码示例
            """,

            // === 需要的工具 ===
            List.of("read_file", "search_code", "run_tests", "git_diff"),

            // === 标准工作流 ===
            List.of(
                new WorkflowStep("identify_changes", "用 git_diff 识别变更范围"),
                new WorkflowStep("security_scan", "逐文件检查安全漏洞"),
                new WorkflowStep("performance_check", "识别潜在性能瓶颈"),
                new WorkflowStep("design_review", "评估架构和设计模式"),
                new WorkflowStep("generate_report", "生成结构化审查报告")
            ),

            // === Few-shot 示例 ===
            List.of(
                new Example(
                    "审查这个登录函数",
                    """
                    ### 代码审查报告
                    
                    🔴 **严重: SQL 注入风险**
                    文件: auth/login.java:15
                    问题: 用户输入直接拼接 SQL 查询
                    修复: 使用 PreparedStatement
                    ```java
                    // 修复前:
                    String sql = "SELECT * FROM users WHERE name='" + username + "'";
                    // 修复后:
                    PreparedStatement ps = conn.prepareStatement("SELECT * FROM users WHERE name=?");
                    ps.setString(1, username);
                    ```
                    ..."""
                )
            ),

            // === 领域知识 ===
            Map.of(
                "owasp_top10", "https://owasp.org/www-project-top-ten/",
                "java_best_practices", "Effective Java by Joshua Bloch"
            )
        );
    }
}
```

---

## 15.3 Skills 的运行时机制

```java
/**
 * Skills 引擎 —— 在 LLM 和工具之间插入 Skills 管理层
 */
public class SkillsEngine {

    private final TransformerDecoder llm;
    private final Map<String, Skill> skills;           // 注册的所有技能
    private final Map<String, ToolExecutor> tools;     // 可用的工具执行器

    /**
     * 处理用户请求 —— 自动匹配并激活 Skill
     */
    public GenerationResult process(String userMessage, List<ChatMessage> history) {

        // Phase 1: 技能路由 (Skill Router)
        Skill activatedSkill = routeSkill(userMessage);
        // 通过词向量相似度或小模型判断用户意图匹配哪个技能

        // Phase 2: 注入 Skill 的 System Prompt
        String enhancedSystemPrompt = buildEnhancedPrompt(activatedSkill);

        // Phase 3: 注入工具定义（Skill 需要的工具子集）
        List<ToolDefinition> skillTools = filterTools(activatedSkill.requiredTools());

        // Phase 4: 注入 Few-shot 示例（可选）
        List<Message> augmentedHistory = injectExamples(history, activatedSkill.fewShotExamples());

        // Phase 5: 正常 LLM 推理（带增强上下文和工具）
        return llm.generateWithTools(
            augmentedHistory,
            enhancedSystemPrompt,
            skillTools
        );
    }

    /**
     * 技能路由：匹配用户意图到最合适的 Skill
     */
    private Skill routeSkill(String userMessage) {
        // 方案 1: 向量相似度匹配
        float[] queryEmbedding = llm.embed(userMessage);
        Skill bestMatch = null;
        float bestScore = -1;

        for (Skill skill : skills.values()) {
            float[] skillEmbedding = skillEmbeddings.get(skill.name());
            float score = cosineSimilarity(queryEmbedding, skillEmbedding);
            if (score > bestScore) {
                bestScore = score;
                bestMatch = skill;
            }
        }

        // 方案 2: 小模型快速分类
        // String category = fastClassifier.classify(userMessage);
        // return skills.get(category);

        // 方案 3: 关键词/正则匹配
        // for (Skill s : skills.values()) {
        //     if (matchesKeywords(userMessage, s.keywords())) return s;
        // }

        return bestMatch != null ? bestMatch : skills.get("general_chat");
    }

    /**
     * 构建增强的 System Prompt
     */
    private String buildEnhancedPrompt(Skill skill) {
        return """
            你现在以「%s」的身份工作。
            
            ## 职责
            %s
            
            ## 可用工具
            %s
            
            ## 工作流程
            %s
            
            ## 输出规范
            %s
            """.formatted(
                skill.name(),
                skill.description(),
                formatTools(skill.requiredTools()),
                formatWorkflow(skill.workflow()),
                "使用 Markdown 格式输出"
            );
    }
}
```

---

## 15.4 Skills 的加载与管理

```java
/**
 * Skill 可以通过多种方式定义和加载
 */
public class SkillLoader {

    // ===== 方式 1: Java 代码定义 =====
    public Skill loadFromCode() {
        return CodeReviewSkill.create();
    }

    // ===== 方式 2: YAML 文件 =====
    public Skill loadFromYaml(Path yamlPath) {
        // skills/code-reviewer.yaml:
        //   name: code_reviewer
        //   description: |
        //     当用户请求代码审查时触发
        //   system_prompt: |
        //     你是一位资深代码审查专家...
        //   tools: [read_file, git_diff, run_tests]
        //   workflow:
        //     - identify_changes
        //     - security_scan
        //     - generate_report
        //   examples:
        //     - input: "审查这个类"
        //       output: "### 审查报告\n..."

        Yaml yaml = new Yaml();
        return yaml.loadAs(Files.readString(yamlPath), Skill.class);
    }

    // ===== 方式 3: Markdown 文件（约定优于配置）=====
    public Skill loadFromMarkdown(Path mdPath) {
        // skills/code-reviewer.md:
        //   # 代码审查
        //   ## 触发条件
        //   ## 工作流程
        //   ## 输出格式
        //   ## 示例

        String content = Files.readString(mdPath);
        return parseMarkdownSkill(content);
    }

    // ===== 方式 4: 动态加载 (类似插件系统) =====
    public SkillLoader() {
        // 监控 skills/ 目录变化，热加载新 Skill
        WatchService watcher = FileSystems.getDefault().newWatchService();
        Path skillsDir = Path.of("skills/");
        skillsDir.register(watcher, ENTRY_CREATE, ENTRY_MODIFY, ENTRY_DELETE);

        // 开启后台线程监控
        Thread.startVirtualThread(() -> {
            while (true) {
                WatchKey key = watcher.take();
                for (WatchEvent<?> event : key.pollEvents()) {
                    Path changed = (Path) event.context();
                    reload(changed);
                }
                key.reset();
            }
        });
    }
}
```

---

## 15.5 Skills 组合（Skill Composition）

```java
/**
 * 复杂任务可能需要组合多个 Skill
 * 
 * 例如: "帮我部署一个 Spring Boot 应用"
 *   需要: git_operations + docker_build + k8s_deploy + monitoring_setup
 */
public class SkillComposer {

    public List<Skill> decompose(String complexTask) {
        // Step 1: 让 LLM 分解任务
        String decompositionPrompt = """
            将以下任务分解为子任务，每个子任务对应一个技能:
            
            任务: %s
            
            可用技能: %s
            
            以 JSON 格式返回: [{"skill": "skill_name", "step": "description"}]
            """.formatted(complexTask, listAvailableSkills());

        String result = llm.generate(decompositionPrompt);
        List<SkillStep> steps = parseJson(result);

        // Step 2: 按顺序执行各 Skill
        List<Skill> pipeline = new ArrayList<>();
        String context = "";  // 上一个 Skill 的输出作为下一个的输入

        for (SkillStep step : steps) {
            Skill skill = skills.get(step.skill());
            if (skill != null) {
                skill = skill.withContext(context);  // 注入前置上下文
                pipeline.add(skill);
            }
        }

        return pipeline;
    }
}
```

---

## 15.6 Skills 与 Agent 的关系

```
Skills 是 Agent 的"专业知识库"（类似微服务架构中的 @Service）
Agent 是 Skills 的"调度器"（类似 API Gateway / BPM 引擎）

              ┌─────────────────┐
              │   Agent / BPM   │  ← 决策: 何时调用哪个 Skill
              └────────┬────────┘
         ┌─────────────┼─────────────┐
         ▼             ▼             ▼
   ┌──────────┐ ┌──────────┐ ┌──────────┐
   │ Skill A  │ │ Skill B  │ │ Skill C  │  ← 每个 Skill 封装领域知识
   │代码审查   │ │部署发布   │ │故障排查   │
   └──────────┘ └──────────┘ └──────────┘
         │             │             │
         ▼             ▼             ▼
   ┌──────────────────────────────────────┐
   │          Tools / MCP Servers         │  ← 底层工具
   │   git  docker  kubectl  database ... │
   └──────────────────────────────────────┘
```

---

## 15.7 Skills 开发最佳实践

```java
/**
 * Skills 设计原则（参考 OpenCode/OpenClaw 等开源项目）
 */
public class SkillBestPractices {

    // 1. 单一职责: 一个 Skill 只做一件事
    //    ✅ "生成 Spring Boot 项目结构"
    //    ❌ "Java 全栈开发"

    // 2. 声明式定义: 用 YAML/Markdown 而非硬编码
    //    ✅ skill.yaml 描述触发条件、工具、流程
    //    ❌ 把逻辑写在 Java 代码里

    // 3. 渐进式增强: 基础 Skill → 增强 → 专业化
    //    通用的 "code_review" → 增强为 "java_code_review"

    // 4. 可组合: Skill 之间通过上下文传递数据
    //    "code_review" 的输出作为 "code_fix" 的输入

    // 5. 可测试: 每个 Skill 都能用单元测试验证
    //    @Test void testCodeReviewFindsSqlInjection() { ... }

    // 6. 版本化: Skills 应该有语义化版本
    //    code_reviewer:v1.2.0

    // 7. 有 Fallback: 每个 Skill 都应该有默认行为
    //    if (noToolAvailable) return "抱歉，当前无法执行此操作"
}
```

---

> **下一章**：[OpenClaw 详解](19-openclaw.md)
