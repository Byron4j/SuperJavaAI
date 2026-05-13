# 构建你的第一个 AI Agent 团队：从需求分析到上线，3 个 Agent 组成一个开发小组

## 一、引言：你的第一个 AI 开发团队

系列六写了这么多篇关于 Agent 的文章，今天我们来点实际的——**从零构建一个完整的、可运行的 AI Agent 团队**。

这个团队由 3 个 Agent 组成：

1. **需求分析师 Agent**：理解用户需求，输出 PRD 文档
2. **开发工程师 Agent**：根据 PRD 编写完整的 Java 代码
3. **测试工程师 Agent**：自动生成测试用例并模拟测试

咱们的项目主题是：**「构建一个简单的用户积分管理系统」**。

你将得到一个完整的、可直接运行的项目——包括代码生成、代码执行、测试验证的全链路。

> 全文约 5000 字。跟着走，一小时内你就能搭建起属于自己的 AI 开发团队。

---

## 二、项目架构

```
ai-dev-team/
├── src/main/java/com/example/aidevteam/
│   ├── agent/
│   │   ├── BaseAgent.java           # Agent 基类
│   │   ├── AnalystAgent.java        # 需求分析师
│   │   ├── DeveloperAgent.java      # 开发工程师
│   │   └── TesterAgent.java         # 测试工程师
│   ├── orhestration/
│   │   ├── DevTeamOrchestrator.java # 团队编排器
│   │   └── DevTeamResult.java       # 执行结果
│   ├── tool/
│   │   ├── Tool.java                # 工具接口
│   │   ├── CodeGeneratorTool.java   # 代码生成工具
│   │   ├── CodeExecutorTool.java    # 代码编译执行工具
│   │   └── TestRunnerTool.java      # 测试运行工具
│   ├── config/
│   │   ├── LLMConfig.java           # LLM 配置
│   │   └── AgentConfig.java         # Agent 配置
│   └── AIDevTeamApplication.java    # 启动类
└── src/main/resources/
    └── application.yml
```

---

## 三、Agent 基类设计

```java
package com.example.aidevteam.agent;

import com.fasterxml.jackson.databind.ObjectMapper;
import java.util.*;
import java.util.function.Function;

/**
 * AI Agent 基类——ReAct 模式的通用实现
 */
public abstract class BaseAgent {
    
    protected final String name;
    protected final String role;
    protected final LLMClient llmClient;
    protected final List<Tool> tools;
    protected final boolean verbose;
    protected final ObjectMapper mapper;
    
    public BaseAgent(String name, String role, LLMClient llmClient, 
                     List<Tool> tools, boolean verbose) {
        this.name = name;
        this.role = role;
        this.llmClient = llmClient;
        this.tools = tools != null ? tools : List.of();
        this.verbose = verbose;
        this.mapper = new ObjectMapper();
    }
    
    /**
     * 每个 Agent 子类实现自己的系统 Prompt
     */
    protected abstract String buildSystemPrompt();
    
    /**
     * 执行 Agent 主循环
     */
    public AgentResult execute(String task, Map<String, Object> context) {
        if (verbose) {
            System.out.println("\n" + "=".repeat(60));
            System.out.println("🤖 " + role + " [" + name + "] 开始执行");
            System.out.println("📋 任务: " + truncate(task, 100));
            System.out.println("=".repeat(60));
        }
        
        long startTime = System.currentTimeMillis();
        List<ChatMessage> messages = new ArrayList<>();
        messages.add(new ChatMessage(ChatMessageRole.SYSTEM.value(), buildSystemPrompt()));
        
        // 添加上下文信息
        if (context != null && !context.isEmpty()) {
            String contextStr = buildContextPrompt(context);
            messages.add(new ChatMessage(ChatMessageRole.USER.value(), 
                    "## 上下文信息\n" + contextStr));
        }
        
        messages.add(new ChatMessage(ChatMessageRole.USER.value(), 
                "## 当前任务\n" + task));
        
        int iteration = 0;
        int maxIterations = 10;
        String finalResult = null;
        List<IterationLog> logs = new ArrayList<>();
        
        while (iteration < maxIterations) {
            iteration++;
            
            if (verbose) {
                System.out.println("\n--- 第 " + iteration + " 轮思考 ---");
            }
            
            // 调用 LLM 获取思考和动作
            ChatCompletionResult completion = llmClient.chat(messages);
            String response = completion.getContent();
            
            // 解析 LLM 响应：是否是工具调用？
            ToolCall toolCall = parseToolCall(response);
            
            if (toolCall == null) {
                // 没有工具调用——Agent 认为任务完成
                finalResult = response;
                logs.add(new IterationLog(iteration, "COMPLETE", response, null));
                
                if (verbose) {
                    System.out.println("✅ 任务完成");
                    System.out.println("📤 输出: " + truncate(response, 200));
                }
                break;
            }
            
            // 有工具调用——执行工具并反馈结果
            if (verbose) {
                System.out.println("🔧 调用工具: " + toolCall.getToolName());
            }
            
            String toolResult;
            try {
                toolResult = executeTool(toolCall);
            } catch (Exception e) {
                toolResult = "工具执行失败: " + e.getMessage();
            }
            
            logs.add(new IterationLog(iteration, "TOOL_CALL", 
                    "调用了 " + toolCall.getToolName(), toolResult));
            
            // 将工具结果反馈给 LLM
            messages.add(new ChatMessage(ChatMessageRole.ASSISTANT.value(), response));
            messages.add(new ChatMessage(ChatMessageRole.USER.value(), 
                    "工具执行结果（" + toolCall.getToolName() + "）:\n```\n" 
                    + toolResult + "\n```\n请继续完成剩余任务。如果任务已完成，直接返回最终结果。"));
        }
        
        long duration = System.currentTimeMillis() - startTime;
        
        if (finalResult == null) {
            finalResult = "未能在 " + maxIterations + " 轮内完成任务。";
        }
        
        return AgentResult.builder()
                .agentName(name)
                .agentRole(role)
                .success(finalResult != null && !finalResult.contains("未能在"))
                .output(finalResult)
                .iterations(iteration)
                .durationMs(duration)
                .logs(logs)
                .build();
    }
    
    /**
     * 解析 LLM 输出中的工具调用
     */
    private ToolCall parseToolCall(String response) {
        // 匹配格式: ```tool\n{"name": "...", "params": {...}}\n```
        Pattern pattern = Pattern.compile(
                "```tool\\s*\\n(\\{.*?\\})\\s*```", Pattern.DOTALL);
        Matcher matcher = pattern.matcher(response);
        
        if (matcher.find()) {
            try {
                JsonNode node = mapper.readTree(matcher.group(1));
                String toolName = node.get("name").asText();
                
                Map<String, Object> params = new HashMap<>();
                JsonNode paramsNode = node.get("params");
                if (paramsNode != null) {
                    paramsNode.fields().forEachRemaining(e -> {
                        params.put(e.getKey(), e.getValue().asText());
                    });
                }
                
                return new ToolCall(toolName, params);
            } catch (Exception e) {
                if (verbose) {
                    System.out.println("⚠️ 工具调用解析失败: " + e.getMessage());
                }
            }
        }
        return null;
    }
    
    /**
     * 执行工具调用
     */
    private String executeTool(ToolCall toolCall) throws Exception {
        Tool targetTool = tools.stream()
                .filter(t -> t.getName().equals(toolCall.getToolName()))
                .findFirst()
                .orElse(null);
        
        if (targetTool == null) {
            return "错误: 未找到工具 '" + toolCall.getToolName() + "'。可用工具: " 
                    + tools.stream().map(Tool::getName).collect(Collectors.joining(", "));
        }
        
        return targetTool.execute(toolCall.getParams());
    }
    
    private String buildContextPrompt(Map<String, Object> context) {
        StringBuilder sb = new StringBuilder();
        for (Map.Entry<String, Object> entry : context.entrySet()) {
            sb.append("- ").append(entry.getKey()).append(": ")
                    .append(truncate(String.valueOf(entry.getValue()), 500))
                    .append("\n");
        }
        return sb.toString();
    }
    
    protected String truncate(String text, int maxLen) {
        return text != null && text.length() > maxLen 
                ? text.substring(0, maxLen) + "..." : text;
    }
    
    private static final Pattern pattern = null;
}
```

---

## 四、需求分析师 Agent

```java
package com.example.aidevteam.agent;

import com.example.aidevteam.tool.Tool;
import java.util.List;

/**
 * 需求分析师 Agent - 理解用户需求，输出标准化 PRD
 */
public class AnalystAgent extends BaseAgent {
    
    public AnalystAgent(LLMClient llmClient, boolean verbose) {
        super("需求分析师", "产品需求分析师", llmClient, List.of(), verbose);
    }
    
    @Override
    protected String buildSystemPrompt() {
        return """
            你是一个资深的产品需求分析师。你的职责是将用户的模糊需求转化为
            清晰、结构化、可执行的产品需求文档（PRD）。
            
            ## 你的工作流程
            1. 理解用户的原始需求
            2. 提炼核心功能点
            3. 定义用户故事和验收标准
            4. 设计数据模型（表结构）
            5. 定义 API 接口
            6. 输出结构化的 PRD 文档
            
            ## PRD 输出格式
            请按照以下 Markdown 格式输出 PRD：
            
            # 产品需求文档
            ## 一、需求概述
            （用 2-3 句话概括整个需求）
            
            ## 二、功能清单
            1. 【功能1名称】：功能描述
               - 验收标准：...
            2. 【功能2名称】：功能描述
               - 验收标准：...
            
            ## 三、数据模型
            ### 表1: table_name
            | 字段 | 类型 | 说明 |
            |------|------|------|
            | id | BIGINT | 主键 |
            
            ## 四、API 接口设计
            ### POST /api/xxx
            - 功能描述
            - 请求参数
            - 响应格式
            
            ## 五、非功能性需求
            （并发、安全、性能等方面的要求）
            
            ## 六、优先级划分
            - P0（必须）
            - P1（应该）
            - P2（可以）
            """;
    }
}
```

---

## 五、开发工程师 Agent

```java
package com.example.aidevteam.agent;

import com.example.aidevteam.tool.CodeGeneratorTool;
import com.example.aidevteam.tool.CodeExecutorTool;
import com.example.aidevteam.tool.Tool;
import java.util.List;

/**
 * 开发工程师 Agent - 根据 PRD 生成可运行的 Java 代码
 */
public class DeveloperAgent extends BaseAgent {
    
    public DeveloperAgent(LLMClient llmClient, CodeGeneratorTool codeGenerator, 
                          CodeExecutorTool codeExecutor, boolean verbose) {
        super("开发工程师", "全栈 Java 开发工程师", llmClient, 
                List.of(codeGenerator, codeExecutor), verbose);
    }
    
    @Override
    protected String buildSystemPrompt() {
        return """
            你是一个资深的 Java 开发工程师，精通 Spring Boot、MyBatis、MySQL。
            
            ## 你的技术栈
            - Spring Boot 3.x
            - MyBatis / MyBatis-Plus
            - MySQL 8.0+
            - Swagger/OpenAPI 3.0
            - Lombok
            - JUnit 5
            
            ## 你的工作流程
            1. 仔细阅读 PRD 文档（在上下文信息中提供）
            2. 按以下顺序生成代码：
               a. 数据库 DDL（建表语句）
               b. 实体类（Entity）
               c. Mapper 接口
               d. Service 接口 + 实现
               e. Controller
               f. 配置文件（application.yml 片段）
            3. 使用 CodeGenerator 工具生成代码文件
            4. 所有代码生成完毕后，输出代码清单和说明
            
            ## 代码规范
            - 所有类使用 Lombok @Data
            - Controller 统一返回 ResponseEntity
            - Service 层方法加 @Transactional
            - 使用统一的异常处理和日志记录
            - 代码要有完整的 JavaDoc 注释
            
            ## 输出格式
            生成完代码后，输出：
            
            # 代码实现说明
            ## 文件清单
            1. src/main/java/com/example/points/entity/XxxEntity.java
            2. ...
            
            ## 关键实现说明
            （简要说明核心逻辑和技术选型原因）
            
            ## 调用 CodeGenerator 工具来生成实际代码文件
            （使用 ```tool 格式调用工具）
            """;
    }
}
```

---

## 六、测试工程师 Agent

```java
package com.example.aidevteam.agent;

import com.example.aidevteam.tool.TestRunnerTool;
import com.example.aidevteam.tool.Tool;
import java.util.List;

/**
 * 测试工程师 Agent - 根据 PRD 和代码生成测试用例
 */
public class TesterAgent extends BaseAgent {
    
    public TesterAgent(LLMClient llmClient, TestRunnerTool testRunner, boolean verbose) {
        super("测试工程师", "高级测试工程师", llmClient, 
                List.of(testRunner), verbose);
    }
    
    @Override
    protected String buildSystemPrompt() {
        return """
            你是一个严谨的高级测试工程师，你的目标是确保代码质量，发现潜在的 Bug。
            
            ## 你的测试策略
            1. 单元测试：每个 Service 方法至少 3 个测试场景
            2. 集成测试：测试完整的 API 调用链路
            3. 边界条件：空值、超长字符串、负数、并发场景
            4. 异常场景：数据库连接失败、参数校验失败
            
            ## 测试用例输出格式
            # 测试报告
            ## 测试概览
            - 测试用例总数：X
            - 通过：Y
            - 失败：Z
            
            ## 测试用例详情
            ### 测试类: XxxServiceTest
            | 用例编号 | 测试场景 | 输入 | 期望输出 | 实际结果 |
            |---------|---------|------|---------|---------|
            | TC-001 | 正常获取积分 | userId=1 | 100 | ✅ PASS |
            
            ## 发现的 Bug
            （如果有的话，列出 Bug 的描述、严重程度和复现步骤）
            
            ## 建议
            （测试覆盖建议、代码改进建议）
            """;
    }
}
```

---

## 七、工具实现

```java
package com.example.aidevteam.tool;

import java.io.*;
import java.nio.file.*;
import java.util.*;
import java.util.concurrent.TimeUnit;

/**
 * Tool 接口
 */
public interface Tool {
    String getName();
    String getDescription();
    String execute(Map<String, Object> params) throws Exception;
    
    static Tool of(String name, String description, 
                   ThrowingFunction<Map<String, Object>, String> executor) {
        return new Tool() {
            @Override public String getName() { return name; }
            @Override public String getDescription() { return description; }
            @Override public String execute(Map<String, Object> params) throws Exception {
                return executor.apply(params);
            }
        };
    }
    
    @FunctionalInterface
    interface ThrowingFunction<T, R> {
        R apply(T t) throws Exception;
    }
}

/**
 * 代码生成工具——创建实际的 .java 文件
 */
public class CodeGeneratorTool implements Tool {
    
    private final String outputDir;
    
    public CodeGeneratorTool(String outputDir) {
        this.outputDir = outputDir;
    }
    
    @Override
    public String getName() { return "CodeGenerator"; }
    
    @Override
    public String getDescription() { 
        return "根据指定的文件路径和内容生成 Java 源代码文件。参数: fileName(文件路径), content(代码内容)"; 
    }
    
    @Override
    public String execute(Map<String, Object> params) throws Exception {
        String fileName = (String) params.get("fileName");
        String content = (String) params.get("content");
        
        if (fileName == null || content == null) {
            return "错误: fileName 和 content 参数不能为空";
        }
        
        Path filePath = Path.of(outputDir, fileName);
        Files.createDirectories(filePath.getParent());
        Files.writeString(filePath, content);
        
        return "文件已创建: " + filePath.toAbsolutePath() + " (" 
                + content.length() + " 字节)";
    }
}

/**
 * 代码执行工具——编译并运行生成的 Java 代码
 */
public class CodeExecutorTool implements Tool {
    
    private final String projectDir;
    private final String mavenHome;
    
    public CodeExecutorTool(String projectDir, String mavenHome) {
        this.projectDir = projectDir;
        this.mavenHome = mavenHome;
    }
    
    @Override
    public String getName() { return "CodeExecutor"; }
    
    @Override
    public String getDescription() { 
        return "使用 Maven 编译项目并运行测试。参数: command(compile|test|package)"; 
    }
    
    @Override
    public String execute(Map<String, Object> params) throws Exception {
        String command = (String) params.getOrDefault("command", "compile");
        
        String mvnCmd = mavenHome + "/bin/mvn";
        String goal = switch (command) {
            case "test" -> "test";
            case "package" -> "package -DskipTests";
            default -> "compile";
        };
        
        ProcessBuilder builder = new ProcessBuilder(
                mvnCmd, goal, "-q", "-f", projectDir + "/pom.xml");
        builder.redirectErrorStream(true);
        
        Process process = builder.start();
        StringBuilder output = new StringBuilder();
        
        try (BufferedReader reader = new BufferedReader(
                new InputStreamReader(process.getInputStream()))) {
            String line;
            while ((line = reader.readLine()) != null) {
                output.append(line).append("\n");
            }
        }
        
        boolean finished = process.waitFor(120, TimeUnit.SECONDS);
        int exitCode = finished ? process.exitValue() : -1;
        
        if (exitCode == 0) {
            return "命令执行成功\n" + output;
        } else {
            return "命令执行失败 (exit code: " + exitCode + ")\n" + output;
        }
    }
}

/**
 * 测试运行工具——运行特定测试类并返回结果
 */
public class TestRunnerTool implements Tool {
    
    @Override
    public String getName() { return "TestRunner"; }
    
    @Override
    public String getDescription() { 
        return "运行指定的测试类。参数: testClass(全限定类名), method(可选，指定测试方法)"; 
    }
    
    @Override
    public String execute(Map<String, Object> params) throws Exception {
        String testClass = (String) params.get("testClass");
        String method = (String) params.get("method");
        
        // 使用 ProcessBuilder 执行 mvn test
        String mvnArgs = "-Dtest=" + testClass;
        if (method != null) {
            mvnArgs += "#" + method;
        }
        
        ProcessBuilder builder = new ProcessBuilder(
                "mvn", "test", mvnArgs, "-q");
        builder.directory(new File(System.getProperty("user.dir")));
        builder.redirectErrorStream(true);
        
        Process process = builder.start();
        StringBuilder output = new StringBuilder();
        
        try (BufferedReader reader = new BufferedReader(
                new InputStreamReader(process.getInputStream()))) {
            String line;
            int lineCount = 0;
            while ((line = reader.readLine()) != null && lineCount < 50) {
                output.append(line).append("\n");
                lineCount++;
            }
        }
        
        process.waitFor(60, TimeUnit.SECONDS);
        
        // 解析测试结果
        String result = output.toString();
        if (result.contains("BUILD SUCCESS")) {
            return "测试全部通过\n" + extractTestSummary(result);
        } else {
            return "测试失败\n" + extractTestSummary(result) + "\n\n失败详情:\n" + result;
        }
    }
    
    private String extractTestSummary(String output) {
        Pattern testsPattern = Pattern.compile("Tests run: (\\d+), Failures: (\\d+), Errors: (\\d+)");
        Matcher matcher = testsPattern.matcher(output);
        if (matcher.find()) {
            return "运行: " + matcher.group(1) + ", 失败: " + matcher.group(2) 
                    + ", 错误: " + matcher.group(3);
        }
        return "无法解析测试结果";
    }
}
```

---

## 八、团队编排器——把一切串联起来

```java
package com.example.aidevteam.orchestration;

import com.example.aidevteam.agent.*;
import java.util.*;
import java.util.concurrent.CompletableFuture;

/**
 * 开发团队编排器——协调需求分析师、开发工程师、测试工程师的工作
 */
public class DevTeamOrchestrator {
    
    private final AnalystAgent analyst;
    private final DeveloperAgent developer;
    private final TesterAgent tester;
    private final boolean verbose;
    
    public DevTeamOrchestrator(AnalystAgent analyst, DeveloperAgent developer, 
                                TesterAgent tester, boolean verbose) {
        this.analyst = analyst;
        this.developer = developer;
        this.tester = tester;
        this.verbose = verbose;
    }
    
    /**
     * 启动开发团队
     */
    public DevTeamResult execute(String userRequirement) {
        long startTime = System.currentTimeMillis();
        
        if (verbose) {
            System.out.println("\n" + "█".repeat(60));
            System.out.println("🚀 AI 开发团队启动");
            System.out.println("📝 用户需求: " + userRequirement);
            System.out.println("█".repeat(60));
        }
        
        // ===== Phase 1: 需求分析 =====
        if (verbose) {
            System.out.println("\n📊 Phase 1: 需求分析师工作");
        }
        
        AgentResult analystResult = analyst.execute(
                "请分析以下用户需求，输出完整的 PRD 文档：\n" + userRequirement, null);
        
        String prd = analystResult.getOutput();
        if (verbose) {
            System.out.println("📄 PRD 已生成 (" + prd.length() + " 字符)");
        }
        
        // ===== Phase 2: 代码开发 =====
        if (verbose) {
            System.out.println("\n💻 Phase 2: 开发工程师工作");
        }
        
        Map<String, Object> devContext = new HashMap<>();
        devContext.put("prd_document", prd);
        devContext.put("user_requirement", userRequirement);
        
        AgentResult developerResult = developer.execute(
                "根据 PRD 文档生成完整的 Java 项目代码。"
                        + "重点实现 P0 优先级的核心功能，确保代码可以编译通过。", 
                devContext);
        
        String codeResult = developerResult.getOutput();
        if (verbose) {
            System.out.println("💾 代码已生成 (" + codeResult.length() + " 字符)");
        }
        
        // ===== Phase 3: 测试验证 =====
        if (verbose) {
            System.out.println("\n🧪 Phase 3: 测试工程师工作");
        }
        
        Map<String, Object> testContext = new HashMap<>();
        testContext.put("prd_document", prd);
        testContext.put("code_generation_result", codeResult);
        
        AgentResult testerResult = tester.execute(
                "根据 PRD 和生成的代码，编写全面的测试用例并进行测试。"
                        + "重点关注核心业务逻辑的单元测试和 API 集成测试。", 
                testContext);
        
        String testReport = testerResult.getOutput();
        if (verbose) {
            System.out.println("📊 测试报告已生成 (" + testReport.length() + " 字符)");
        }
        
        long totalTime = System.currentTimeMillis() - startTime;
        
        boolean overallSuccess = analystResult.isSuccess() 
                && developerResult.isSuccess() 
                && testerResult.isSuccess();
        
        if (verbose) {
            System.out.println("\n" + "█".repeat(60));
            System.out.println(overallSuccess ? "✅ 开发团队任务完成" : "❌ 部分任务失败");
            System.out.println("⏱️ 总耗时: " + totalTime + "ms");
            System.out.println("█".repeat(60));
        }
        
        return DevTeamResult.builder()
                .userRequirement(userRequirement)
                .prd(prd)
                .codeGenerationResult(codeResult)
                .testReport(testReport)
                .analystResult(analystResult)
                .developerResult(developerResult)
                .testerResult(testerResult)
                .totalTimeMs(totalTime)
                .overallSuccess(overallSuccess)
                .build();
    }
}
```

---

## 九、Spring Boot 启动类与完整配置

```java
package com.example.aidevteam;

import com.example.aidevteam.agent.*;
import com.example.aidevteam.tool.*;
import com.example.aidevteam.orchestration.*;
import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;

@SpringBootApplication
public class AIDevTeamApplication {
    
    public static void main(String[] args) {
        SpringApplication.run(AIDevTeamApplication.class, args);
    }
    
    @Bean
    public LLMClient llmClient() {
        return new OpenAIClient(
                System.getenv("OPENAI_API_KEY"),
                "gpt-4-turbo",
                0.3
        );
    }
    
    @Bean
    public CodeGeneratorTool codeGeneratorTool() {
        return new CodeGeneratorTool("./generated-src/main/java/com/example/points");
    }
    
    @Bean
    public CodeExecutorTool codeExecutorTool() {
        return new CodeExecutorTool("./generated-src", 
                System.getenv().getOrDefault("M2_HOME", "/usr/local/maven"));
    }
    
    @Bean
    public TestRunnerTool testRunnerTool() {
        return new TestRunnerTool();
    }
    
    @Bean
    public AnalystAgent analystAgent(LLMClient llmClient) {
        return new AnalystAgent(llmClient, true);
    }
    
    @Bean
    public DeveloperAgent developerAgent(LLMClient llmClient,
                                          CodeGeneratorTool codeGenerator,
                                          CodeExecutorTool codeExecutor) {
        return new DeveloperAgent(llmClient, codeGenerator, codeExecutor, true);
    }
    
    @Bean
    public TesterAgent testerAgent(LLMClient llmClient, TestRunnerTool testRunner) {
        return new TesterAgent(llmClient, testRunner, true);
    }
    
    @Bean
    public DevTeamOrchestrator devTeam(AnalystAgent analyst, 
                                        DeveloperAgent developer,
                                        TesterAgent tester) {
        return new DevTeamOrchestrator(analyst, developer, tester, true);
    }
    
    // ===== 命令行入口（可以直接运行） =====
    
    @Bean
    public CommandLineRunner runDevTeam(DevTeamOrchestrator orchestrator) {
        return args -> {
            String requirement = """
                构建一个用户积分管理系统，支持以下功能：
                1. 用户可以通过完成特定行为获取积分（如签到、购物、评价）
                2. 积分可以兑换优惠券
                3. 用户可查询自己的积分余额和积分变动明细
                4. 积分有过期机制（一年内有效）
                5. 管理员可以查看积分统计报表
                """;
            
            DevTeamResult result = orchestrator.execute(requirement);
            
            System.out.println("\n\n" + "=".repeat(60));
            System.out.println("📋 最终交付物");
            System.out.println("=".repeat(60));
            System.out.println("\n【PRD 文档】\n" + result.getPrd().substring(0, 
                    Math.min(500, result.getPrd().length())) + "...");
            System.out.println("\n【实现结果】\n" 
                    + result.getCodeGenerationResult().substring(0, 
                    Math.min(500, result.getCodeGenerationResult().length())) + "...");
            System.out.println("\n【测试报告】\n" 
                    + result.getTestReport().substring(0, 
                    Math.min(500, result.getTestReport().length())) + "...");
        };
    }
}
```

---

## 十、总结

恭喜你，你已经构建了第一个 AI Agent 团队。回顾一下我们做了什么：

1. **BaseAgent**：通用的 ReAct 循环（思考→行动→观察）
2. **AnalystAgent**：专业的需求分析师，输出结构化 PRD
3. **DeveloperAgent**：Java 开发工程师，生成可运行的代码
4. **TesterAgent**：测试工程师，编写并执行测试用例
5. **DevTeamOrchestrator**：串联三个 Agent 的编排器

**下一步可以优化什么**：
- 加入代码审查 Agent（Code Reviewer）
- 加入部署 Agent（DevOps）
- 增加 Agent 之间的「讨论」机制（类似团队评审）
- 引入 LangGraph 的状态管理（回退/重试/分支）

**关键提示**：在生产环境中使用 AI 生成代码时，务必进行人工 Code Review。Agent 是一个加速器，不是替代者。

---

**系列最终篇预告**：《AI Agent 的 10 个开源项目推荐与实战部署》——站在巨人的肩膀上造 Agent！深度盘点 2024-2025 最值得关注的 10 个开源 Agent 项目，每个项目的 Java 可用性评估和部署建议。系列收官之作，不容错过！
