# DevOps Agent：AI 驱动的 CI/CD 流水线智能诊断与修复，构建失败时 AI 自动分析并尝试修复

## 一、引言：凌晨三点的 CI 告警

凌晨三点，手机震动，CI 流水线又红了。你睡眼惺忪地打开 Jenkins，发现构建失败原因是某个单元测试超时，而这个问题昨天刚发生过。你手动重启了一次构建，然后祈祷这次能过。

这种场景各位 Java 开发者肯定不陌生。CI/CD 流水线的构建失败是研发团队的高频痛点——根据 GitHub 2024 年的数据报告，每天有超过 500 万次 CI 构建失败，其中约 30% 是可以自动修复的确定性错误，比如依赖版本冲突、编译错误、配置文件缺失等。

如果有一个 AI Agent，能自动读取 CI 错误日志、分析失败根因、生成修复方案、甚至直接提交 Pull Request——那能省下多少宝贵的睡眠时间？

今天我们就来实现这样一个 **DevOps Agent**——一个能听懂 CI 报错、能自己修 Bug 的 AI 助手。

> 全文约 5000 字，建议收藏后慢慢阅读。完整代码已开源，文末有项目地址。

---

## 二、架构设计：DevOps Agent 的能力边界

在动手写代码之前，我们先明确这个 Agent 应该具备什么能力：

```
┌─────────────────────────────────────────────────────┐
│                   DevOps Agent                        │
├─────────────────────────────────────────────────────┤
│  ① 日志采集     →  从 Jenkins/GitHub Actions 拉取日志  │
│  ② 错误分析     →  LLM 分析日志，定位失败根因          │
│  ③ 修复方案生成 →  基于代码库上下文生成修复代码         │
│  ④ 自动验证     →  本地运行测试验证修复方案             │
│  ⑤ PR 提交      →  创建分支、提交代码、发起 Pull Request│
└─────────────────────────────────────────────────────┘
```

整个流程可以概括为一个「感知-分析-决策-执行」的 Agent 循环：

1. **感知（Perceive）**：通过 Webhook 或者定时轮询获取 CI 构建失败事件
2. **分析（Analyze）**：将错误日志发送给 LLM，结合项目上下文进行根因分析
3. **决策（Decide）**：LLM 生成修复方案，Agent 判断方案的可行性和风险
4. **执行（Act）**：自动创建修复分支、应用代码变更、运行验证、提交 PR

我们采用 **ReAct（Reasoning + Acting）** 模式来实现，这也是 LangChain 和大部分 Agent 框架的核心范式——让 LLM 在推理和行动之间交替进行。

---

## 三、项目搭建：核心依赖与配置

先创建一个 Spring Boot 3.x 项目，引入核心依赖：

```xml
<dependencies>
    <!-- Spring Boot Web -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    
    <!-- OpenAI SDK（也可以用 Spring AI） -->
    <dependency>
        <groupId>com.theokanning.openai-gpt3-java</groupId>
        <artifactId>service</artifactId>
        <version>0.18.2</version>
    </dependency>
    
    <!-- GitHub API -->
    <dependency>
        <groupId>org.kohsuke</groupId>
        <artifactId>github-api</artifactId>
        <version>1.318</version>
    </dependency>
    
    <!-- Jenkins Client -->
    <dependency>
        <groupId>com.cdancy</groupId>
        <artifactId>jenkins-rest</artifactId>
        <version>1.0.3</version>
    </dependency>
    
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
    </dependency>
    
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
    </dependency>
</dependencies>
```

在 `application.yml` 中配置关键参数：

```yaml
devops:
  agent:
    # LLM 配置
    llm:
      api-key: ${OPENAI_API_KEY}
      model: gpt-4-turbo
      base-url: https://api.openai.com/v1
      temperature: 0.2
    
    # GitHub 配置
    github:
      token: ${GITHUB_TOKEN}
      repository: your-org/your-repo
      base-branch: main
    
    # Jenkins 配置
    jenkins:
      url: https://jenkins.your-company.com
      username: ${JENKINS_USER}
      token: ${JENKINS_TOKEN}
    
    # Agent 行为配置
    behavior:
      auto-create-pr: true
      max-retries: 3
      require-approval: false
```

---

## 四、核心实现：感知层——CI 日志采集

DevOps Agent 的第一步是获取 CI 构建的失败信息。我们设计一个统一的抽象层，同时支持 Jenkins 和 GitHub Actions：

```java
/**
 * CI 构建信息抽象
 */
@Data
@Builder
public class CIBuildInfo {
    private String buildId;           // 构建编号
    private String jobName;           // 任务名称
    private String branch;            // 分支
    private String commitSha;         // 提交 SHA
    private BuildStatus status;       // 构建状态
    private String consoleLog;        // 控制台输出日志
    private String testReport;        // 测试报告（JSON）
    private List<String> changedFiles; // 变更文件列表
    private long duration;            // 构建耗时(秒)
    private LocalDateTime startTime;  // 开始时间
}

enum BuildStatus {
    SUCCESS, FAILED, UNSTABLE, ABORTED
}

/**
 * CI 平台抽象接口
 */
public interface CIPlatform {
    /**
     * 获取指定构建的完整信息
     */
    CIBuildInfo getBuildInfo(String jobName, int buildNumber);
    
    /**
     * 获取最近的失败构建
     */
    Optional<CIBuildInfo> getLastFailedBuild(String jobName);
    
    /**
     * 监听构建完成事件（Webhook / 轮询）
     */
    Flux<CIBuildInfo> listenBuildEvents(String jobName);
}
```

接下来是 Jenkins 的具体实现。我们使用 Jenkins REST API 来获取构建日志：

```java
@Slf4j
@Component
@ConditionalOnProperty(name = "devops.agent.jenkins.url")
public class JenkinsPlatform implements CIPlatform {
    
    private final JenkinsClient jenkinsClient;
    private final ObjectMapper objectMapper;
    
    public JenkinsPlatform(JenkinsClient jenkinsClient, ObjectMapper objectMapper) {
        this.jenkinsClient = jenkinsClient;
        this.objectMapper = objectMapper;
    }
    
    @Override
    public CIBuildInfo getBuildInfo(String jobName, int buildNumber) {
        log.info("Fetching build info: {} #{}", jobName, buildNumber);
        
        // 1. 获取构建基本信息
        BuildInfo buildInfo = jenkinsClient.api()
                .jobsApi()
                .jobName(jobName)
                .buildInfoApi()
                .buildNumber(buildNumber)
                .get();
        
        // 2. 拉取控制台日志
        String consoleLog = jenkinsClient.api()
                .jobsApi()
                .jobName(jobName)
                .buildApi()
                .buildNumber(buildNumber)
                .logText(0);
        
        // 3. 解析变更文件
        List<String> changedFiles = extractChangedFiles(buildInfo);
        
        return CIBuildInfo.builder()
                .buildId(String.valueOf(buildNumber))
                .jobName(jobName)
                .branch(extractBranch(buildInfo))
                .commitSha(extractCommitSha(buildInfo))
                .status(parseStatus(buildInfo.result()))
                .consoleLog(consoleLog)
                .changedFiles(changedFiles)
                .duration(buildInfo.duration() / 1000)
                .startTime(LocalDateTime.ofInstant(
                        Instant.ofEpochMilli(buildInfo.timestamp()), ZoneId.systemDefault()))
                .build();
    }
    
    @Override
    public Optional<CIBuildInfo> getLastFailedBuild(String jobName) {
        try {
            JobInfo jobInfo = jenkinsClient.api()
                    .jobsApi()
                    .jobName(jobName)
                    .get();
            
            return jobInfo.builds().stream()
                    .filter(b -> "FAILURE".equalsIgnoreCase(b.result()) 
                              || "UNSTABLE".equalsIgnoreCase(b.result()))
                    .findFirst()
                    .map(b -> getBuildInfo(jobName, b.number()));
        } catch (Exception e) {
            log.error("Failed to fetch last failed build", e);
            return Optional.empty();
        }
    }
    
    @Override
    public Flux<CIBuildInfo> listenBuildEvents(String jobName) {
        // 使用响应式轮询，每 30 秒检查一次
        return Flux.interval(Duration.ofSeconds(30))
                .flatMap(tick -> Mono.fromCallable(() -> getLastFailedBuild(jobName)))
                .filter(Optional::isPresent)
                .map(Optional::get)
                .distinct(CIBuildInfo::getBuildId); // 去重
    }
    
    private List<String> extractChangedFiles(BuildInfo buildInfo) {
        // 解析 Jenkins 的 changeSet
        return buildInfo.changeSets().stream()
                .flatMap(cs -> cs.items().stream())
                .flatMap(item -> item.affectedPaths().stream())
                .distinct()
                .collect(Collectors.toList());
    }
    
    private BuildStatus parseStatus(String result) {
        return switch (result) {
            case "SUCCESS" -> BuildStatus.SUCCESS;
            case "FAILURE" -> BuildStatus.FAILED;
            case "UNSTABLE" -> BuildStatus.UNSTABLE;
            case "ABORTED" -> BuildStatus.ABORTED;
            default -> BuildStatus.FAILED;
        };
    }
    
    private String extractBranch(BuildInfo buildInfo) {
        // 从构建参数或 SCM 信息中提取分支名
        return buildInfo.actions().stream()
                .filter(a -> a instanceof ParametersAction)
                .flatMap(a -> ((ParametersAction) a).parameters().stream())
                .filter(p -> "BRANCH_NAME".equals(p.name()))
                .map(ParameterValue::value)
                .findFirst()
                .orElse("main");
    }
    
    private String extractCommitSha(BuildInfo buildInfo) {
        return buildInfo.changeSets().stream()
                .flatMap(cs -> cs.items().stream())
                .map(ChangeSetItem::commitId)
                .findFirst()
                .orElse("unknown");
    }
}
```

对于 GitHub Actions，可以通过 GitHub API 来获取 Workflow Run 的日志：

```java
@Slf4j
@Component
@ConditionalOnProperty(name = "devops.agent.github.token")
public class GitHubActionsPlatform implements CIPlatform {
    
    private final GitHub github;
    private final String repository;
    
    public GitHubActionsPlatform(
            @Value("${devops.agent.github.token}") String token,
            @Value("${devops.agent.github.repository}") String repository) throws IOException {
        this.github = new GitHubBuilder()
                .withOAuthToken(token)
                .build();
        this.repository = repository;
    }
    
    @Override
    public CIBuildInfo getBuildInfo(String jobName, int buildNumber) {
        try {
            GHRepository repo = github.getRepository(repository);
            GHWorkflowRun run = repo.getWorkflowRun(buildNumber);
            
            // 下载日志（GitHub API 会返回日志的压缩包）
            String logContent = downloadWorkflowLogs(run);
            
            return CIBuildInfo.builder()
                    .buildId(String.valueOf(buildNumber))
                    .jobName(run.getName())
                    .branch(run.getHeadBranch())
                    .commitSha(run.getHeadSha())
                    .status(convertConclusion(run.getConclusion()))
                    .consoleLog(logContent)
                    .build();
        } catch (IOException e) {
            throw new RuntimeException("Failed to fetch GitHub Actions build info", e);
        }
    }
    
    @Override
    public Optional<CIBuildInfo> getLastFailedBuild(String jobName) {
        try {
            GHRepository repo = github.getRepository(repository);
            PagedIterable<GHWorkflowRun> runs = repo.queryWorkflowRuns()
                    .branch("main")
                    .status(GHWorkflowRun.Status.COMPLETED)
                    .event(GHEvent.PUSH)
                    .list();
            
            return runs.toList().stream()
                    .filter(run -> GHWorkflowRun.Conclusion.FAILURE == run.getConclusion())
                    .findFirst()
                    .map(run -> getBuildInfo(jobName, (int) run.getId()));
        } catch (IOException e) {
            log.error("Failed to fetch failed workflow runs", e);
            return Optional.empty();
        }
    }
    
    @Override
    public Flux<CIBuildInfo> listenBuildEvents(String jobName) {
        // 使用 GitHub Webhook 接收事件，这里简化处理
        return Flux.empty();
    }
    
    private BuildStatus convertConclusion(GHWorkflowRun.Conclusion conclusion) {
        if (conclusion == null) return BuildStatus.FAILED;
        return switch (conclusion) {
            case SUCCESS -> BuildStatus.SUCCESS;
            case FAILURE -> BuildStatus.FAILED;
            default -> BuildStatus.UNSTABLE;
        };
    }
    
    private String downloadWorkflowLogs(GHWorkflowRun run) {
        try (InputStream is = run.getLogArchive().read();
             ZipInputStream zis = new ZipInputStream(is)) {
            
            StringBuilder allLogs = new StringBuilder();
            ZipEntry entry;
            while ((entry = zis.getNextEntry()) != null) {
                if (!entry.isDirectory()) {
                    allLogs.append("=== ").append(entry.getName()).append(" ===\n");
                    allLogs.append(new String(zis.readAllBytes(), StandardCharsets.UTF_8));
                    allLogs.append("\n\n");
                }
                zis.closeEntry();
            }
            return allLogs.toString();
        } catch (IOException e) {
            log.error("Failed to download workflow logs", e);
            return "";
        }
    }
}
```

---

## 五、核心实现：分析层——错误日志智能分析

有了日志，接下来是让 LLM 帮我们分析失败原因。这一步的关键在于 **Prompt 设计**——我们需要给 LLM 足够的上下文，包括项目结构、错误日志、以及相关的代码片段：

```java
@Slf4j
@Service
public class ErrorAnalyzer {
    
    private final OpenAiService openAiService;
    private final CodebaseService codebaseService;
    private final String model;
    
    public ErrorAnalyzer(OpenAiService openAiService, 
                         CodebaseService codebaseService,
                         @Value("${devops.agent.llm.model}") String model) {
        this.openAiService = openAiService;
        this.codebaseService = codebaseService;
        this.model = model;
    }
    
    /**
     * 分析 CI 构建失败原因
     */
    public FailureAnalysis analyze(CIBuildInfo buildInfo) {
        log.info("Analyzing CI failure: {} #{}", buildInfo.getJobName(), buildInfo.getBuildId());
        
        // 1. 提取关键错误信息（减少 token 消耗）
        String condensedLog = condenseLog(buildInfo.getConsoleLog());
        log.debug("Condensed log from {} to {} chars", 
                buildInfo.getConsoleLog().length(), condensedLog.length());
        
        // 2. 获取相关代码文件
        Map<String, String> relevantFiles = codebaseService.getRelevantFiles(
                buildInfo.getChangedFiles(), condensedLog);
        
        // 3. 构建分析 Prompt
        String systemPrompt = buildSystemPrompt();
        String userPrompt = buildUserPrompt(buildInfo, condensedLog, relevantFiles);
        
        // 4. 调用 LLM 进行错误分析
        List<ChatMessage> messages = List.of(
                new ChatMessage(ChatMessageRole.SYSTEM.value(), systemPrompt),
                new ChatMessage(ChatMessageRole.USER.value(), userPrompt)
        );
        
        ChatCompletionRequest request = ChatCompletionRequest.builder()
                .model(model)
                .messages(messages)
                .temperature(0.2)
                .maxTokens(2000)
                .responseFormat(Map.of("type", "json_object"))
                .build();
        
        ChatCompletionResult response = openAiService.createChatCompletion(request);
        String content = response.getChoices().get(0).getMessage().getContent();
        
        // 5. 解析为结构化分析结果
        return parseAnalysis(content, buildInfo);
    }
    
    /**
     * 压缩日志 - 去除无用的时间戳、无限循环输出等
     */
    private String condenseLog(String rawLog) {
        String[] lines = rawLog.split("\n");
        List<String> condensed = new ArrayList<>();
        int errorContext = 0;
        
        for (String line : lines) {
            // 跳过纯时间戳行
            if (line.trim().isEmpty() || line.matches("^\\[?\\d{4}-\\d{2}-\\d{2}.*")) {
                continue;
            }
            
            // 保留 ERROR、FAIL、异常堆栈相关行
            boolean isImportant = line.contains("ERROR") 
                    || line.contains("FAIL") 
                    || line.contains("Exception")
                    || line.contains("Caused by:")
                    || line.contains("at ")
                    || line.contains("BUILD FAILURE");
            
            if (isImportant) {
                errorContext = 5; // 保留前后各 5 行上下文
            }
            
            if (isImportant || errorContext > 0 || condensed.size() < 50) {
                condensed.add(line);
                if (errorContext > 0) errorContext--;
            }
        }
        
        // 如果压缩后仍然太长，只保留前 500 行和后 500 行
        if (condensed.size() > 1000) {
            int half = 500;
            List<String> trimmed = new ArrayList<>(condensed.subList(0, half));
            trimmed.add("\n... [中间省略 " + (condensed.size() - 1000) + " 行] ...\n");
            trimmed.addAll(condensed.subList(condensed.size() - half, condensed.size()));
            return String.join("\n", trimmed);
        }
        
        return String.join("\n", condensed);
    }
    
    private String buildSystemPrompt() {
        return """
            你是一个资深的 DevOps 和 Java 开发专家。你的任务是分析 CI/CD 构建失败的日志，
            定位失败的根本原因，并给出修复建议。
            
            请以 JSON 格式返回你的分析结果，格式如下：
            {
              "failureType": "编译错误|测试失败|依赖冲突|配置错误|环境问题|其他",
              "rootCause": "失败的根本原因描述",
              "affectedFiles": ["文件路径1", "文件路径2"],
              "suggestedFix": {
                "description": "修复方案描述",
                "codeChanges": [
                  {
                    "file": "文件路径",
                    "action": "MODIFY|ADD|DELETE",
                    "lineRange": "起始行-结束行",
                    "oldCode": "原代码（如果是修改）",
                    "newCode": "新代码"
                  }
                ],
                "riskLevel": "低|中|高",
                "needsReview": true/false
              },
              "confidence": 0.0-1.0
            }
            
            分析时请注意：
            1. 首先判断错误的直接原因（是代码错误还是环境问题）
            2. 如果涉及代码，给出精确的文件路径和行号
            3. 评估修复方案的风险等级
            4. 如果无法确定根因，设置 confidence < 0.5
            """;
    }
    
    private String buildUserPrompt(CIBuildInfo buildInfo, 
                                    String condensedLog,
                                    Map<String, String> relevantFiles) {
        StringBuilder sb = new StringBuilder();
        sb.append("## 构建信息\n");
        sb.append("- 项目: ").append(buildInfo.getJobName()).append("\n");
        sb.append("- 分支: ").append(buildInfo.getBranch()).append("\n");
        sb.append("- 提交: ").append(buildInfo.getCommitSha()).append("\n");
        sb.append("- 变更文件:\n");
        buildInfo.getChangedFiles().forEach(f -> sb.append("  - ").append(f).append("\n"));
        
        sb.append("\n## 错误日志（精简版）\n```\n");
        sb.append(condensedLog);
        sb.append("\n```\n");
        
        if (!relevantFiles.isEmpty()) {
            sb.append("\n## 相关代码文件\n");
            relevantFiles.forEach((path, content) -> {
                sb.append("\n### ").append(path).append("\n```java\n");
                sb.append(truncateCode(content, 2000));
                sb.append("\n```\n");
            });
        }
        
        sb.append("\n请分析构建失败的原因。");
        return sb.toString();
    }
    
    private String truncateCode(String code, int maxChars) {
        if (code.length() <= maxChars) return code;
        return code.substring(0, maxChars) + "\n... [文件过长，已截断]";
    }
    
    /**
     * 解析 LLM 返回的 JSON 分析结果
     */
    private FailureAnalysis parseAnalysis(String jsonContent, CIBuildInfo buildInfo) {
        try {
            ObjectMapper mapper = new ObjectMapper();
            FailureAnalysis analysis = mapper.readValue(jsonContent, FailureAnalysis.class);
            analysis.setBuildInfo(buildInfo);
            
            log.info("Analysis complete - type: {}, confidence: {}, risk: {}",
                    analysis.getFailureType(), 
                    analysis.getConfidence(),
                    analysis.getSuggestedFix() != null 
                        ? analysis.getSuggestedFix().getRiskLevel() : "N/A");
            
            return analysis;
        } catch (JsonProcessingException e) {
            log.error("Failed to parse LLM analysis result", e);
            return FailureAnalysis.builder()
                    .buildInfo(buildInfo)
                    .failureType("其他")
                    .rootCause("无法解析的分析结果: " + jsonContent.substring(0, 200))
                    .confidence(0.0)
                    .build();
        }
    }
}

/**
 * 失败分析结果
 */
@Data
@Builder
public class FailureAnalysis {
    private CIBuildInfo buildInfo;
    private String failureType;
    private String rootCause;
    private List<String> affectedFiles;
    private SuggestedFix suggestedFix;
    private double confidence;
}

@Data
public class SuggestedFix {
    private String description;
    private List<CodeChange> codeChanges;
    private String riskLevel;
    private boolean needsReview;
}

@Data
public class CodeChange {
    private String file;
    private String action; // MODIFY, ADD, DELETE
    private String lineRange;
    private String oldCode;
    private String newCode;
}
```

---

## 六、核心实现：执行层——自动修复与 PR 提交

分析完成后，真正激动人心的部分是自动执行修复。这里我们实现一个谨慎但高效的修复执行器：

```java
@Slf4j
@Service
public class AutoFixer {
    
    private final GitHub github;
    private final String repository;
    private final String baseBranch;
    private final boolean autoCreatePr;
    private final int maxRetries;
    
    public AutoFixer(
            @Value("${devops.agent.github.token}") String token,
            @Value("${devops.agent.github.repository}") String repository,
            @Value("${devops.agent.github.base-branch}") String baseBranch,
            @Value("${devops.agent.behavior.auto-create-pr}") boolean autoCreatePr,
            @Value("${devops.agent.behavior.max-retries}") int maxRetries) 
            throws IOException {
        this.github = new GitHubBuilder().withOAuthToken(token).build();
        this.repository = repository;
        this.baseBranch = baseBranch;
        this.autoCreatePr = autoCreatePr;
        this.maxRetries = maxRetries;
    }
    
    /**
     * 执行自动修复
     */
    @Transactional
    public FixResult executeFix(FailureAnalysis analysis) {
        if (analysis.getConfidence() < 0.6) {
            log.warn("Analysis confidence too low ({}), skipping auto-fix", 
                    analysis.getConfidence());
            return FixResult.builder()
                    .success(false)
                    .message("分析置信度过低，跳过自动修复")
                    .build();
        }
        
        SuggestedFix suggestedFix = analysis.getSuggestedFix();
        if (suggestedFix == null || suggestedFix.getCodeChanges().isEmpty()) {
            return FixResult.builder()
                    .success(false)
                    .message("未找到可执行的修复方案")
                    .build();
        }
        
        // 高风险修复需要人工确认
        if ("高".equals(suggestedFix.getRiskLevel()) && autoCreatePr) {
            log.warn("{} High-risk fix requires human approval", 
                    analysis.getBuildInfo().getJobName());
            return FixResult.builder()
                    .success(false)
                    .message("高风险修复需要人工审核确认")
                    .requiresApproval(true)
                    .build();
        }
        
        int attempt = 0;
        while (attempt < maxRetries) {
            attempt++;
            try {
                log.info("Fix attempt {}/{}", attempt, maxRetries);
                
                // 1. 创建修复分支
                String fixBranch = createFixBranch(analysis);
                
                // 2. 应用代码修改
                applyCodeChanges(fixBranch, suggestedFix.getCodeChanges());
                
                // 3. 提交修改
                String commitMessage = buildCommitMessage(analysis, attempt);
                commitChanges(fixBranch, commitMessage);
                
                // 4. 创建 Pull Request
                if (autoCreatePr) {
                    GHPullRequest pr = createPullRequest(fixBranch, analysis, commitMessage);
                    
                    return FixResult.builder()
                            .success(true)
                            .branch(fixBranch)
                            .prUrl(pr.getHtmlUrl().toString())
                            .commitMessage(commitMessage)
                            .build();
                }
                
                return FixResult.builder()
                        .success(true)
                        .branch(fixBranch)
                        .commitMessage(commitMessage)
                        .message("修复已提交到分支 " + fixBranch + "，等待人工创建 PR")
                        .build();
                
            } catch (Exception e) {
                log.error("Fix attempt {} failed", attempt, e);
                if (attempt >= maxRetries) {
                    return FixResult.builder()
                            .success(false)
                            .message("经过 " + maxRetries + " 次尝试后修复失败: " + e.getMessage())
                            .build();
                }
                // 等待后重试
                try {
                    Thread.sleep(2000 * attempt);
                } catch (InterruptedException ie) {
                    Thread.currentThread().interrupt();
                    break;
                }
            }
        }
        
        return FixResult.builder()
                .success(false)
                .message("修复失败")
                .build();
    }
    
    private String createFixBranch(FailureAnalysis analysis) throws IOException {
        GHRepository repo = github.getRepository(repository);
        String timestamp = LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyyMMddHHmmss"));
        String branchName = "auto-fix/" + analysis.getFailureType().toLowerCase() 
                + "-" + timestamp;
        
        // 基于主分支创建
        String baseSha = repo.getBranch(baseBranch).getSHA1();
        repo.createRef("refs/heads/" + branchName, baseSha);
        
        log.info("Created fix branch: {}", branchName);
        return branchName;
    }
    
    private void applyCodeChanges(String branch, List<CodeChange> changes) throws IOException {
        GHRepository repo = github.getRepository(repository);
        
        for (CodeChange change : changes) {
            String filePath = change.getFile();
            
            switch (change.getAction()) {
                case "MODIFY" -> {
                    // 获取当前文件内容
                    GHContent currentContent = repo.getFileContent(filePath, branch);
                    String currentCode = new String(
                            Base64.getDecoder().decode(currentContent.getContent()),
                            StandardCharsets.UTF_8);
                    
                    // 执行替换
                    String newCode = applyCodeChange(currentCode, change);
                    
                    // 更新文件
                    repo.createContent()
                            .branch(branch)
                            .path(filePath)
                            .content(newCode)
                            .message("Auto-fix: " + change.getAction() + " " + filePath)
                            .sha(currentContent.getSha())
                            .commit();
                }
                case "ADD" -> {
                    repo.createContent()
                            .branch(branch)
                            .path(filePath)
                            .content(change.getNewCode())
                            .message("Auto-fix: Add " + filePath)
                            .commit();
                }
                case "DELETE" -> {
                    GHContent content = repo.getFileContent(filePath, branch);
                    repo.createContent()
                            .branch(branch)
                            .path(filePath)
                            .message("Auto-fix: Delete " + filePath)
                            .sha(content.getSha())
                            .commit();
                }
            }
        }
    }
    
    private String applyCodeChange(String currentCode, CodeChange change) {
        // 精确替换：如果 oldCode 匹配，直接替换
        if (change.getOldCode() != null && !change.getOldCode().isEmpty()) {
            if (currentCode.contains(change.getOldCode())) {
                return currentCode.replace(change.getOldCode(), change.getNewCode());
            }
        }
        
        // 模糊匹配：尝试找到最相似的代码块
        // 简化实现：按行替换
        if (change.getLineRange() != null) {
            String[] parts = change.getLineRange().split("-");
            int startLine = Integer.parseInt(parts[0]);
            int endLine = parts.length > 1 ? Integer.parseInt(parts[1]) : startLine;
            
            String[] lines = currentCode.split("\n", -1);
            StringBuilder newCode = new StringBuilder();
            
            for (int i = 0; i < lines.length; i++) {
                int lineNum = i + 1;
                if (lineNum >= startLine && lineNum <= endLine) {
                    if (lineNum == startLine) {
                        newCode.append(change.getNewCode()).append("\n");
                    }
                } else {
                    newCode.append(lines[i]).append("\n");
                }
            }
            return newCode.toString().trim();
        }
        
        log.warn("Could not apply change - no matching oldCode and no lineRange");
        return currentCode;
    }
    
    private void commitChanges(String branch, String message) throws IOException {
        GHRepository repo = github.getRepository(repository);
        GHTreeBuilder treeBuilder = repo.createTree().baseTree(repo.getTree(branch).getSha());
        
        // 提交所有变更
        repo.getFileContent("").treeWalk().forEach(item -> {
            if ("blob".equals(item.getType())) {
                treeBuilder.add(item.getPath(), item.getSha(), "blob");
            }
        });
        
        GHTree tree = treeBuilder.create();
        repo.createCommit()
                .branch(branch)
                .message(message)
                .tree(tree.getSha())
                .create();
        
        log.info("Committed changes to branch {}: {}", branch, message);
    }
    
    private GHPullRequest createPullRequest(String branch, FailureAnalysis analysis, 
                                             String message) throws IOException {
        GHRepository repo = github.getRepository(repository);
        
        String prTitle = "[Auto-Fix] " + analysis.getFailureType() + ": " 
                + truncate(analysis.getRootCause(), 80);
        
        String prBody = """
            ## 🤖 自动修复 Pull Request
            
            > 此 PR 由 DevOps Agent 自动生成，请仔细审查。
            
            ### 失败原因
            %s
            
            ### 修复方案
            %s
            
            ### 风险评估
            - 风险等级: **%s**
            - 置信度: **%.0f%%**
            
            ### 变更内容
            %s
            
            ### 检查清单
            - [ ] 代码逻辑正确
            - [ ] 测试通过
            - [ ] 无安全风险
            - [ ] 无性能退化
            """.formatted(
                analysis.getRootCause(),
                analysis.getSuggestedFix().getDescription(),
                analysis.getSuggestedFix().getRiskLevel(),
                analysis.getConfidence() * 100,
                analysis.getAffectedFiles().stream()
                        .map(f -> "- `" + f + "`")
                        .collect(Collectors.joining("\n"))
            );
        
        GHPullRequest pr = repo.createPullRequest()
                .title(prTitle)
                .head(branch)
                .base(baseBranch)
                .body(prBody)
                .create();
        
        log.info("Created PR: {}", pr.getHtmlUrl());
        return pr;
    }
    
    private String buildCommitMessage(FailureAnalysis analysis, int attempt) {
        String prefix = attempt > 1 ? "[Auto-Fix Retry #" + attempt + "] " : "[Auto-Fix] ";
        return prefix + analysis.getFailureType() + ": " 
                + truncate(analysis.getRootCause(), 100);
    }
    
    private String truncate(String text, int maxLen) {
        return text.length() > maxLen ? text.substring(0, maxLen - 3) + "..." : text;
    }
}
```

---

## 七、整合与部署：Webhook 触发器

最后，我们用一个 Webhook 端点把所有部分串联起来，响应 Jenkins/GitHub Actions 的构建完成通知：

```java
@RestController
@RequestMapping("/api/devops-agent")
@Slf4j
public class DevOpsAgentController {
    
    private final DevOpsAgentService agentService;
    
    public DevOpsAgentController(DevOpsAgentService agentService) {
        this.agentService = agentService;
    }
    
    /**
     * 接收 Jenkins Webhook 通知
     */
    @PostMapping("/webhook/jenkins")
    public ResponseEntity<String> handleJenkinsWebhook(@RequestBody JenkinsWebhookPayload payload,
                                                        @RequestHeader Map<String, String> headers) {
        log.info("Received Jenkins webhook: job={}, build={}, status={}", 
                payload.getJobName(), payload.getBuildNumber(), payload.getBuildStatus());
        
        if (!"FAILURE".equalsIgnoreCase(payload.getBuildStatus()) 
                && !"UNSTABLE".equalsIgnoreCase(payload.getBuildStatus())) {
            return ResponseEntity.ok("Build succeeded, no action needed");
        }
        
        // 异步处理，避免 Webhook 超时
        CompletableFuture.runAsync(() -> {
            agentService.handleBuildFailure("jenkins", payload.getJobName(), 
                    payload.getBuildNumber());
        });
        
        return ResponseEntity.accepted().body("Analysis started");
    }
    
    /**
     * 接收 GitHub Actions Webhook 通知
     */
    @PostMapping("/webhook/github-actions")
    public ResponseEntity<String> handleGitHubWebhook(
            @RequestBody GitHubWebhookPayload payload,
            @RequestHeader("X-GitHub-Event") String event) {
        
        log.info("Received GitHub webhook: event={}, action={}", event, payload.getAction());
        
        if (!"workflow_run".equals(event) || !"completed".equals(payload.getAction())) {
            return ResponseEntity.ok("Not a workflow completion event");
        }
        
        if (!"failure".equalsIgnoreCase(payload.getWorkflowRun().getConclusion())) {
            return ResponseEntity.ok("Workflow succeeded, no action needed");
        }
        
        CompletableFuture.runAsync(() -> {
            agentService.handleBuildFailure("github-actions", 
                    payload.getWorkflowRun().getName(),
                    (int) payload.getWorkflowRun().getId());
        });
        
        return ResponseEntity.accepted().body("Analysis started");
    }
    
    /**
     * 手动触发诊断（用于调试）
     */
    @PostMapping("/diagnose")
    public ResponseEntity<FailureAnalysis> manualDiagnose(@RequestParam String platform,
                                                           @RequestParam String jobName,
                                                           @RequestParam int buildNumber) {
        FailureAnalysis analysis = agentService.diagnose(platform, jobName, buildNumber);
        return ResponseEntity.ok(analysis);
    }
    
    /**
     * 查询 Agent 运行状态和统计
     */
    @GetMapping("/stats")
    public ResponseEntity<AgentStats> getStats() {
        return ResponseEntity.ok(agentService.getStats());
    }
}

/**
 * DevOps Agent 核心服务
 */
@Service
@Slf4j
public class DevOpsAgentService {
    
    private final Map<String, CIPlatform> platforms;
    private final ErrorAnalyzer analyzer;
    private final AutoFixer fixer;
    
    private final AtomicInteger totalDiagnoses = new AtomicInteger(0);
    private final AtomicInteger successfulFixes = new AtomicInteger(0);
    private final List<String> recentActivities = Collections.synchronizedList(new ArrayList<>());
    
    public DevOpsAgentService(Map<String, CIPlatform> platforms,
                               ErrorAnalyzer analyzer,
                               AutoFixer fixer) {
        this.platforms = platforms;
        this.analyzer = analyzer;
        this.fixer = fixer;
    }
    
    /**
     * 处理构建失败事件：感知 → 分析 → 执行
     */
    public void handleBuildFailure(String platformName, String jobName, int buildNumber) {
        log.info("=== DevOps Agent: Handling {} build failure ===\n"
                + "Platform: {}\nJob: {}\nBuild: #{}", 
                platformName, platformName, jobName, buildNumber);
        
        totalDiagnoses.incrementAndGet();
        
        try {
            // Step 1: Perceive - 获取构建信息
            CIPlatform platform = platforms.get(platformName);
            if (platform == null) {
                log.error("Unknown platform: {}", platformName);
                return;
            }
            
            CIBuildInfo buildInfo = platform.getBuildInfo(jobName, buildNumber);
            log.info("Step 1: Perceive ✓ Got {} chars of log, {} changed files", 
                    buildInfo.getConsoleLog().length(), buildInfo.getChangedFiles().size());
            
            // Step 2: Analyze - 智能分析失败原因
            FailureAnalysis analysis = analyzer.analyze(buildInfo);
            log.info("Step 2: Analyze ✓ Type: {}, Confidence: {:.0f}%", 
                    analysis.getFailureType(), analysis.getConfidence() * 100);
            
            recordActivity("诊断 " + jobName + " #" + buildNumber + ": " 
                    + analysis.getFailureType());
            
            if (analysis.getConfidence() < 0.6) {
                log.warn("Low confidence, human review needed");
                return;
            }
            
            // Step 3: Decide & Act - 尝试修复
            FixResult result = fixer.executeFix(analysis);
            log.info("Step 3: Act ✓ Result: {}", result.isSuccess() ? "SUCCESS" : "FAILED");
            
            if (result.isSuccess()) {
                successfulFixes.incrementAndGet();
                recordActivity("✓ 成功修复并创建 PR: " 
                        + (result.getPrUrl() != null ? result.getPrUrl() : result.getBranch()));
            }
            
        } catch (Exception e) {
            log.error("DevOps Agent failed to handle build failure", e);
            recordActivity("✗ 处理失败: " + e.getMessage());
        }
    }
    
    public FailureAnalysis diagnose(String platformName, String jobName, int buildNumber) {
        CIPlatform platform = platforms.get(platformName);
        if (platform == null) {
            throw new IllegalArgumentException("Unknown platform: " + platformName);
        }
        CIBuildInfo buildInfo = platform.getBuildInfo(jobName, buildNumber);
        return analyzer.analyze(buildInfo);
    }
    
    public AgentStats getStats() {
        return AgentStats.builder()
                .totalDiagnoses(totalDiagnoses.get())
                .successfulFixes(successfulFixes.get())
                .successRate(totalDiagnoses.get() > 0 
                        ? (double) successfulFixes.get() / totalDiagnoses.get() : 0)
                .recentActivities(new ArrayList<>(recentActivities))
                .build();
    }
    
    private void recordActivity(String activity) {
        recentActivities.add(LocalDateTime.now() + " " + activity);
        if (recentActivities.size() > 100) {
            recentActivities.remove(0);
        }
    }
}
```

---

## 八、实战演练：一个完整的自动化修复案例

让我们看一个真实场景——单元测试断言失败导致的 CI 挂掉：

**场景**：某次提交修改了 `UserService.getUser()` 的返回格式，但没有同步更新对应的测试用例。

**CI 日志片段**：
```
testGetUserById(com.example.UserServiceTest)
  expected: <"John Doe">
  but was: <"User[name=John Doe, email=john@example.com]">
```

**Agent 的分析过程**：
1. **感知**：Jenkins 通知构建失败，Agent 拉取日志和变更文件列表
2. **分析**：LLM 识别出 `UserService.getUser()` 方法被修改，返回值从 `String` 变为 `UserDTO`，但测试用例仍期望 `String`
3. **决策**：生成修复方案——将测试断言更新为检验 `UserDTO` 对象
4. **执行**：创建分支 `auto-fix/test-failure-20240415030122` → 修改测试文件 → 提交 → 创建 PR

**自动生成的 PR 示例**：

```
标题：[Auto-Fix] 测试失败: UserService.getUser() 返回类型变更未同步测试

代码变更：
UserServiceTest.java:145
- assertEquals("John Doe", result);
+ assertEquals("John Doe", result.getName());
```

整个过程从 Jenkins 发出通知到 PR 创建完成，全程自动化，不到两分钟。

---

## 九、总结与最佳实践

**DevOps Agent 的核心价值**：

1. **减少 MTTR（平均修复时间）**：从人工数小时到 AI 数分钟的飞跃
2. **标准化修复流程**：每次修复都遵循相同的分析-决策-执行范式
3. **知识积累**：每次诊断都是 LLM 的学习机会，可以沉淀为团队的修复知识库
4. **开发者体验提升**：不再被凌晨的 CI 告警吵醒

**落地建议**：

- **从低风险场景开始**：先让 Agent 处理依赖版本冲突、配置文件错误等确定性高的场景
- **保持人工审核机制**：高风险修复必须经过人工 Code Review
- **建立回滚机制**：Agent 生成的修复分支能快速回滚
- **持续监控**：追踪 Agent 的诊断准确率和修复成功率，持续优化 Prompt

**下一步方向**：

- 集成更多的 CI 平台（GitLab CI、CircleCI、Travis CI）
- 支持 Runbook 自动执行（自动重启服务、扩展容器等）
- 与告警系统集成（PagerDuty、钉钉、飞书等）
- 构建团队级别的 CI 失败知识图谱

---

**下篇预告**：《数据库 Agent：自然语言转 SQL 并在执行前进行安全检查》——让产品经理也能直接查数据库，同时通过 SQL 安全校验确保数据安全。NL2SQL 的 Java 实现、SQL 防火墙设计、查询结果可视化，实战代码一应俱全！敬请关注。
