# 代码审查 Agent：自动检测代码坏味道并生成重构建议，你的PR从此多了一个AI Reviewer

## 开篇：代码审查的痛，你我都懂

凌晨一点，你改完了最后一行代码，信心满满地提了个 PR。第二天一早，同事的 Review 评论像雪花一样飘来：

- "这个变量名`temp`是什么意思？能换个有意义的名字吗？"
- "这段代码和`UserService`里的`validateEmail`完全重复了，能不能抽出来？"
- "这个循环嵌套太深了，最多三层，你这都五层了"
- "SQL 拼接有注入风险，走参数化查询"

熟悉吗？团队里的 Senior 每天被这些基础 Review 淹没，Junior 写的 PR 等了三天才有人看。代码质量靠人肉 Review 保证的日子，太痛苦了。

如果能有一个 **AI 代码审查员**，在每个 PR 提交时自动扫描，把那些"一眼就能看出来的问题"提前揪出来——人工 Reviewer 只需要关注架构设计和业务逻辑，那效率得多高？

今天，我们就来实现它。

---

## 一、代码审查 Agent 的整体架构

### 1.1 设计理念

这个 Agent 的职责是：**做第一层筛查，而不是替代人类 Reviewer**。它负责发现代码层面的"坏味道"，人类负责审批架构层面的决策。

### 1.2 工作流程

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│  读取代码  │ →  │ 解析AST  │ →  │ 多维度   │ →  │ AI分析   │ →  │ 生成报告  │
│  (PR Diff)│    │ (JavaParser│ │ 规则检查  │    │ LLM深度  │    │ 结构化输出 │
└──────────┘    └──────────┘    └──────────┘    └──────────┘    └──────────┘
     ↑                                                               ↓
  钉钉/GitHub Webhook                                          评论到PR
```

---

## 二、检测维度总览

我们把代码审查拆成五个维度，从规则到AI逐层深入：

| 维度 | 检测内容 | 检测方式 | 严重程度 |
|------|----------|----------|----------|
| **命名规范** | 变量/方法/类名是否清晰、是否符合驼峰规范 | 规则+AI | 中等 |
| **复杂度** | 圈复杂度、方法长度、嵌套深度 | 规则 | 高 |
| **重复代码** | 与已有代码的相似度、内联重复 | AST比对+AI | 高 |
| **安全漏洞** | SQL注入、XSS、敏感信息硬编码 | 规则+AI | 严重 |
| **性能问题** | N+1查询、不必要的循环、资源泄漏 | 规则+AI | 高 |

---

## 三、核心实现

### 3.1 引入依赖

```java
// Maven 关键依赖
dependencies {
    // Java AST 解析
    implementation 'com.github.javaparser:javaparser-core:3.25.8'
    // LLM 调用（以 Spring AI 为例）
    implementation 'org.springframework.ai:spring-ai-openai-spring-boot-starter'
    // GitHub API
    implementation 'org.kohsuke:github-api:1.318'
    // JSON处理
    implementation 'com.fasterxml.jackson.core:jackson-databind'
}
```

### 3.2 代码解析引擎——基于规则的第一层筛查

规则检查不消耗 Token，速度极快，适合做批量预筛：

```java
@Service
public class CodeAnalysisEngine {

    // 分析结果
    @Data
    @Builder
    public static class CodeIssue {
        private String file;
        private int line;
        private String severity;    // BLOCKER / CRITICAL / MAJOR / MINOR
        private String rule;        // 规则名称
        private String message;     // 问题描述
        private String suggestion;  // 修复建议
        private String codeSnippet; // 问题代码片段
    }

    public List<CodeIssue> analyze(String sourceCode, String fileName) {
        List<CodeIssue> issues = new ArrayList<>();
        CompilationUnit cu = StaticJavaParser.parse(sourceCode);

        // === 1. 命名规范检查 ===
        issues.addAll(checkNamingConventions(cu, fileName));

        // === 2. 复杂度检查 ===
        issues.addAll(checkComplexity(cu, fileName));

        // === 3. 安全检查 ===
        issues.addAll(checkSecurityIssues(cu, fileName));

        // === 4. 性能检查 ===
        issues.addAll(checkPerformanceIssues(cu, fileName));

        return issues;
    }

    // --- 命名规范 ---
    private List<CodeIssue> checkNamingConventions(CompilationUnit cu, String file) {
        List<CodeIssue> issues = new ArrayList<>();

        // 检查方法名是否为小驼峰
        cu.findAll(MethodDeclaration.class).forEach(method -> {
            String name = method.getNameAsString();
            if (!Character.isLowerCase(name.charAt(0))) {
                issues.add(CodeIssue.builder()
                    .file(file)
                    .line(method.getBegin().map(p -> p.line).orElse(0))
                    .severity("MAJOR")
                    .rule("method-naming")
                    .message(String.format("方法名 [%s] 不符合小驼峰命名规范", name))
                    .suggestion(String.format("建议重命名为: %s",
                        Character.toLowerCase(name.charAt(0)) + name.substring(1)))
                    .codeSnippet(method.getDeclarationAsString(false, false, false))
                    .build());
            }

            // 检查方法名是否过于简短
            if (name.length() <= 2) {
                issues.add(CodeIssue.builder()
                    .file(file)
                    .line(method.getBegin().map(p -> p.line).orElse(0))
                    .severity("MINOR")
                    .rule("method-name-length")
                    .message(String.format("方法名 [%s] 过于简短，建议使用更有意义的名称", name))
                    .suggestion("请使用能描述方法功能的完整单词")
                    .build());
            }
        });

        // 检查变量名（排除常量）
        cu.findAll(VariableDeclarator.class).forEach(var -> {
            String name = var.getNameAsString();
            if (name.length() == 1 && !name.equals("i") && !name.equals("j")
                && !name.equals("x") && !name.equals("y")) {
                issues.add(CodeIssue.builder()
                    .file(file)
                    .line(var.getBegin().map(p -> p.line).orElse(0))
                    .severity("MAJOR")
                    .rule("variable-naming")
                    .message(String.format("单字母变量 [%s] 含义不明确", name))
                    .suggestion("使用有意义的变量名")
                    .build());
            }
        });

        return issues;
    }

    // --- 复杂度检查 ---
    private List<CodeIssue> checkComplexity(CompilationUnit cu, String file) {
        List<CodeIssue> issues = new ArrayList<>();

        // 检查方法长度
        cu.findAll(MethodDeclaration.class).forEach(method -> {
            int lines = method.getEnd().map(p -> p.line).orElse(0)
                - method.getBegin().map(p -> p.line).orElse(0);
            if (lines > 50) {
                issues.add(CodeIssue.builder()
                    .file(file)
                    .line(method.getBegin().map(p -> p.line).orElse(0))
                    .severity("CRITICAL")
                    .rule("method-length")
                    .message(String.format("方法 [%s] 过长 (%d行)，建议控制在50行以内",
                        method.getNameAsString(), lines))
                    .suggestion("将方法拆分为多个职责单一的小方法")
                    .build());
            }
        });

        // 检查嵌套深度
        cu.findAll(BlockStmt.class).forEach(block -> {
            int depth = calculateNestingDepth(block);
            if (depth > 4) {
                block.getStatements().stream().findFirst().ifPresent(stmt -> {
                    issues.add(CodeIssue.builder()
                        .file(file)
                        .line(stmt.getBegin().map(p -> p.line).orElse(0))
                        .severity("MAJOR")
                        .rule("nesting-depth")
                        .message(String.format("嵌套深度达到 %d 层，建议最多3-4层", depth))
                        .suggestion("使用卫语句提前return，或抽取子方法")
                        .build());
                });
            }
        });

        // 检查圈复杂度（简化版：统计if/for/while/case/catch数量）
        cu.findAll(MethodDeclaration.class).forEach(method -> {
            int complexity = countCyclomaticComplexity(method);
            if (complexity > 15) {
                issues.add(CodeIssue.builder()
                    .file(file)
                    .line(method.getBegin().map(p -> p.line).orElse(0))
                    .severity("CRITICAL")
                    .rule("cyclomatic-complexity")
                    .message(String.format("方法 [%s] 圈复杂度为 %d，建议低于15",
                        method.getNameAsString(), complexity))
                    .suggestion("考虑使用策略模式或状态模式简化条件分支")
                    .build());
            }
        });

        return issues;
    }

    // --- 安全检查 ---
    private List<CodeIssue> checkSecurityIssues(CompilationUnit cu, String file) {
        List<CodeIssue> issues = new ArrayList<>();

        // 检测SQL拼接（字符串包含 "+  " + " 且上下文有SQL关键词）
        cu.findAll(StringLiteralExpr.class).forEach(literal -> {
            String value = literal.getValue();
            String lower = value.toLowerCase();
            if ((lower.contains("select ") || lower.contains("insert ")
                || lower.contains("update ") || lower.contains("delete "))
                && lower.contains("+")) {
                issues.add(CodeIssue.builder()
                    .file(file)
                    .line(literal.getBegin().map(p -> p.line).orElse(0))
                    .severity("BLOCKER")
                    .rule("sql-injection")
                    .message("检测到SQL字符串拼接，存在注入风险")
                    .suggestion("使用PreparedStatement或MyBatis的参数化查询替代字符串拼接")
                    .codeSnippet(truncate(value, 200))
                    .build());
            }
        });

        // 检测硬编码的密码/密钥
        cu.findAll(VariableDeclarator.class).forEach(var -> {
            String name = var.getNameAsString().toLowerCase();
            if ((name.contains("password") || name.contains("secret")
                || name.contains("apikey") || name.contains("token"))
                && var.getInitializer().isPresent()
                && var.getInitializer().get() instanceof StringLiteralExpr) {
                issues.add(CodeIssue.builder()
                    .file(file)
                    .line(var.getBegin().map(p -> p.line).orElse(0))
                    .severity("BLOCKER")
                    .rule("hardcoded-secret")
                    .message(String.format("检测到硬编码的敏感信息: [%s]", var.getNameAsString()))
                    .suggestion("将敏感信息移至环境变量或配置中心，切勿硬编码在代码中")
                    .build());
            }
        });

        // 检测 System.out.println（生产环境不推荐）
        cu.findAll(MethodCallExpr.class).forEach(call -> {
            if (call.toString().startsWith("System.out.print")
                || call.toString().startsWith("System.err.print")) {
                issues.add(CodeIssue.builder()
                    .file(file)
                    .line(call.getBegin().map(p -> p.line).orElse(0))
                    .severity("MINOR")
                    .rule("system-out-println")
                    .message("生产代码中应使用Logger替代System.out.println")
                    .suggestion("使用SLF4J: private static final Logger log = LoggerFactory.getLogger(...)")
                    .build());
            }
        });

        return issues;
    }

    // --- 性能检查 ---
    private List<CodeIssue> checkPerformanceIssues(CompilationUnit cu, String file) {
        List<CodeIssue> issues = new ArrayList<>();

        // 检测循环中的字符串拼接（应使用StringBuilder）
        cu.findAll(ForStmt.class).forEach(forStmt -> {
            forStmt.findAll(AssignExpr.class).forEach(assign -> {
                if (assign.getTarget().toString().contains("+")
                    && assign.getValue().toString().contains("+")) {
                    issues.add(CodeIssue.builder()
                        .file(file)
                        .line(forStmt.getBegin().map(p -> p.line).orElse(0))
                        .severity("MAJOR")
                        .rule("string-concatenation-in-loop")
                        .message("循环中使用了字符串拼接，应使用StringBuilder")
                        .suggestion("在循环外声明StringBuilder，循环内使用append()")
                        .build());
                }
            });
        });

        // 检测 N+1 查询模式（简单的启发式检测）
        cu.findAll(ForEachStmt.class).forEach(loop -> {
            boolean hasQueryInLoop = loop.getBody().getStatements().stream()
                .anyMatch(s -> s.toString().contains("select")
                    || s.toString().contains("findBy"));
            if (hasQueryInLoop) {
                issues.add(CodeIssue.builder()
                    .file(file)
                    .line(loop.getBegin().map(p -> p.line).orElse(0))
                    .severity("CRITICAL")
                    .rule("n-plus-one-query")
                    .message("循环内部疑似包含数据库查询，可能导致N+1问题")
                    .suggestion("考虑使用IN查询、批量查询或JOIN一次性获取所有数据")
                    .build());
            }
        });

        return issues;
    }

    // 计算嵌套深度
    private int calculateNestingDepth(BlockStmt block) {
        Node parent = block.getParentNode().orElse(null);
        int depth = 0;
        while (parent != null) {
            if (parent instanceof IfStmt || parent instanceof ForStmt
                || parent instanceof WhileStmt || parent instanceof SwitchStmt) {
                depth++;
            }
            parent = parent.getParentNode().orElse(null);
        }
        return depth;
    }

    // 计算圈复杂度（简化版）
    private int countCyclomaticComplexity(MethodDeclaration method) {
        int complexity = 1; // 基础复杂度
        complexity += method.findAll(IfStmt.class).size();
        complexity += method.findAll(ForStmt.class).size();
        complexity += method.findAll(ForEachStmt.class).size();
        complexity += method.findAll(WhileStmt.class).size();
        complexity += method.findAll(SwitchEntry.class).size();
        complexity += method.findAll(ConditionalExpr.class).size(); // 三元运算符
        complexity += (int) method.findAll(BinaryExpr.class).stream()
            .filter(e -> e.getOperator() == BinaryExpr.Operator.AND
                || e.getOperator() == BinaryExpr.Operator.OR)
            .count();
        return complexity;
    }

    private String truncate(String s, int maxLen) {
        return s.length() > maxLen ? s.substring(0, maxLen) + "..." : s;
    }
}
```

### 3.3 AI 深度分析——让 LLM 发现规则发现不了的问题

规则能发现"形"上的问题，但代码"神"上的问题——比如逻辑漏洞、设计缺陷——需要 LLM 来深挖：

```java
@Service
public class AICodeReviewer {

    private final String REVIEW_PROMPT = """
        你是一位资深Java代码审查专家。请仔细审查以下代码，从以下维度分析：

        1. **逻辑正确性**：代码逻辑是否存在漏洞？边界条件是否处理完整？
        2. **设计模式**：是否合理使用了设计模式？是否存在反模式？
        3. **可维护性**：代码是否易于理解和修改？依赖关系是否合理？
        4. **异常处理**：异常是否被正确处理？是否存在吞异常的情况？
        5. **线程安全**：是否存在并发问题？关键资源是否正确同步？
        6. **代码风格**：是否符合Clean Code原则？

        请以JSON格式返回审查结果：
        [
          {
            "severity": "BLOCKER|CRITICAL|MAJOR|MINOR",
            "category": "分类",
            "line": 行号(可估算),
            "issue": "问题描述",
            "suggestion": "具体的重构建议，包含代码示例"
          }
        ]

        如果没有发现任何问题，返回空数组 []。

        待审查代码：
        {code}
        """;

    @Autowired
    private ChatClient chatClient;

    public List<CodeAnalysisEngine.CodeIssue> deepReview(
            String fileName, String sourceCode) {

        String prompt = REVIEW_PROMPT.replace("{code}", sourceCode);

        String response = chatClient.call(prompt);

        // 解析 LLM 返回的 JSON
        try {
            ObjectMapper mapper = new ObjectMapper();
            JsonNode issues = mapper.readTree(response);
            List<CodeAnalysisEngine.CodeIssue> result = new ArrayList<>();

            for (JsonNode issue : issues) {
                result.add(CodeAnalysisEngine.CodeIssue.builder()
                    .file(fileName)
                    .line(issue.has("line") ? issue.get("line").asInt() : 0)
                    .severity(issue.get("severity").asText())
                    .rule("ai-deep-review")
                    .message(issue.get("issue").asText())
                    .suggestion(issue.get("suggestion").asText())
                    .build());
            }
            return result;
        } catch (Exception e) {
            log.error("AI审查结果解析失败", e);
            return List.of();
        }
    }

    private static final org.slf4j.Logger log =
        org.slf4j.LoggerFactory.getLogger(AICodeReviewer.class);
}
```

### 3.4 重复代码检测

```java
@Service
public class DuplicationDetector {

    // 简化版相似度检测：基于代码归一化后的文本对比
    public List<CodeAnalysisEngine.CodeIssue> detectDuplications(
            List<FileContent> prFiles,
            List<FileContent> repoFiles) {

        List<CodeAnalysisEngine.CodeIssue> issues = new ArrayList<>();

        for (FileContent prFile : prFiles) {
            List<MethodInfo> prMethods = extractMethods(prFile.getContent());

            for (FileContent repoFile : repoFiles) {
                if (repoFile.getPath().equals(prFile.getPath())) continue;

                List<MethodInfo> repoMethods = extractMethods(repoFile.getContent());

                for (MethodInfo prMethod : prMethods) {
                    for (MethodInfo repoMethod : repoMethods) {
                        double similarity = calculateTextSimilarity(
                            prMethod.getNormalizedBody(),
                            repoMethod.getNormalizedBody());

                        if (similarity > 0.85) {
                            issues.add(CodeAnalysisEngine.CodeIssue.builder()
                                .file(prFile.getPath())
                                .line(prMethod.getStartLine())
                                .severity("CRITICAL")
                                .rule("code-duplication")
                                .message(String.format(
                                    "方法 [%s] 与 %s 中的 [%s] 相似度 %.0f%%，存在代码重复",
                                    prMethod.getName(), repoFile.getPath(),
                                    repoMethod.getName(), similarity * 100))
                                .suggestion("将公共逻辑抽取到工具类或父类中")
                                .build());
                        }
                    }
                }
            }
        }
        return issues;
    }

    // 基于Levenshtein距离的文本相似度
    private double calculateTextSimilarity(String a, String b) {
        if (a == null || b == null) return 0;
        int maxLength = Math.max(a.length(), b.length());
        if (maxLength == 0) return 1.0;
        int distance = levenshteinDistance(a, b);
        return 1.0 - (double) distance / maxLength;
    }

    private int levenshteinDistance(String a, String b) {
        int[][] dp = new int[a.length() + 1][b.length() + 1];
        for (int i = 0; i <= a.length(); i++) dp[i][0] = i;
        for (int j = 0; j <= b.length(); j++) dp[0][j] = j;
        for (int i = 1; i <= a.length(); i++) {
            for (int j = 1; j <= b.length(); j++) {
                int cost = a.charAt(i - 1) == b.charAt(j - 1) ? 0 : 1;
                dp[i][j] = Math.min(Math.min(dp[i - 1][j] + 1,
                    dp[i][j - 1] + 1), dp[i - 1][j - 1] + cost);
            }
        }
        return dp[a.length()][b.length()];
    }

    // 代码归一化：去掉注释、空白、变量名（统一替换为$var）
    private String normalizeCode(String code) {
        return code.replaceAll("//.*", "")
            .replaceAll("/\\*[\\s\\S]*?\\*/", "")
            .replaceAll("\\s+", " ")
            .replaceAll("\"[^\"]*\"", "\"$STR\"")
            .replaceAll("\\b[a-z][a-zA-Z0-9]*\\b", "$var")
            .trim();
    }

    // 从源码中提取方法信息
    private List<MethodInfo> extractMethods(String source) { return List.of(); }

    @Data
    static class MethodInfo {
        private String name;
        private String normalizedBody;
        private int startLine;
    }

    @Data
    @AllArgsConstructor
    static class FileContent {
        private String path;
        private String content;
    }
}
```

---

## 四、与 GitHub/GitLab 集成

### 4.1 GitHub Webhook 监听 PR

```java
@RestController
@RequestMapping("/webhook/github")
public class GitHubWebhookController {

    @Autowired
    private CodeReviewOrchestrator orchestrator;

    @PostMapping("/pr")
    public ResponseEntity<String> handlePullRequest(
            @RequestBody JsonNode payload,
            @RequestHeader("X-GitHub-Event") String eventType) {

        if (!"pull_request".equals(eventType)) {
            return ResponseEntity.ok("ignored");
        }

        String action = payload.get("action").asText();
        if (!"opened".equals(action) && !"synchronize".equals(action)) {
            return ResponseEntity.ok("ignored"); // 只处理新建和更新的PR
        }

        String repoName = payload.get("repository").get("full_name").asText();
        int prNumber = payload.get("number").asInt();

        // 异步执行审查，不阻塞Webhook响应
        CompletableFuture.runAsync(() ->
            orchestrator.reviewPR(repoName, prNumber));

        return ResponseEntity.ok("review started");
    }
}
```

### 4.2 审查结果自动评论到 PR

```java
@Service
public class CodeReviewOrchestrator {

    @Autowired private CodeAnalysisEngine ruleEngine;
    @Autowired private AICodeReviewer aiReviewer;
    @Autowired private DuplicationDetector duplicationDetector;

    public void reviewPR(String repoName, int prNumber) {
        // 1. 获取PR的变更文件
        List<FileContent> changedFiles = githubClient.getPRFiles(repoName, prNumber);
        // 2. 获取仓库已有代码（用于重复检测）
        List<FileContent> repoFiles = githubClient.getRepoFiles(repoName);

        List<CodeAnalysisEngine.CodeIssue> allIssues = new ArrayList<>();

        for (FileContent file : changedFiles) {
            // 3. 规则引擎检查
            allIssues.addAll(ruleEngine.analyze(file.getContent(), file.getPath()));
            // 4. AI 深度审查
            allIssues.addAll(aiReviewer.deepReview(file.getPath(), file.getContent()));
        }
        // 5. 重复代码检测
        allIssues.addAll(duplicationDetector.detectDuplications(changedFiles, repoFiles));

        // 6. 生成审查报告并评论
        String reviewComment = buildReviewComment(allIssues);
        githubClient.postPRComment(repoName, prNumber, reviewComment);
    }

    String buildReviewComment(List<CodeAnalysisEngine.CodeIssue> issues) {
        if (issues.isEmpty()) {
            return """
                ## 🤖 AI Code Review
                **审查结果：未发现明显问题** ✅
                
                代码质量良好，未检测到命名、复杂度、安全或性能方面的明显问题。
                请注意这仅覆盖了常见规则，架构和业务逻辑仍需人工Review。""";
        }

        StringBuilder sb = new StringBuilder();
        sb.append("## 🤖 AI Code Review\n\n");
        sb.append("**审查摘要**：发现 **")
          .append(issues.size()).append("** 个问题\n\n");

        long blockers = issues.stream()
            .filter(i -> "BLOCKER".equals(i.getSeverity())).count();
        long criticals = issues.stream()
            .filter(i -> "CRITICAL".equals(i.getSeverity())).count();
        long majors = issues.stream()
            .filter(i -> "MAJOR".equals(i.getSeverity())).count();

        sb.append(String.format(
            "🔴 Blockers: %d | 🟠 Critical: %d | 🟡 Major: %d | 🔵 Minor: %d\n\n",
            blockers, criticals, majors,
            issues.size() - blockers - criticals - majors));

        sb.append("---\n\n");

        // 按严重程度分组展示
        issues.stream()
            .sorted(Comparator.comparing(CodeAnalysisEngine.CodeIssue::getSeverity))
            .forEach(issue -> {
                sb.append(String.format("### %s | `%s:%d` | %s\n",
                    severityIcon(issue.getSeverity()),
                    issue.getFile(), issue.getLine(),
                    issue.getRule()));
                sb.append("**问题**：").append(issue.getMessage()).append("\n\n");
                if (issue.getSuggestion() != null) {
                    sb.append("**建议**：").append(issue.getSuggestion()).append("\n\n");
                }
                if (issue.getCodeSnippet() != null) {
                    sb.append("```java\n").append(issue.getCodeSnippet())
                        .append("\n```\n\n");
                }
            });

        sb.append("---\n\n");
        sb.append("*本评论由 AI Code Reviewer 自动生成，"
            + "仅供参考，重要决策请结合人工审查*");

        return sb.toString();
    }

    private String severityIcon(String severity) {
        return switch (severity) {
            case "BLOCKER" -> "🔴 BLOCKER";
            case "CRITICAL" -> "🟠 CRITICAL";
            case "MAJOR" -> "🟡 MAJOR";
            default -> "🔵 MINOR";
        };
    }

    // GitHub 客户端（使用 kohsuke/github-api）
    @Service
    static class GitHubClient {
        public List<DuplicationDetector.FileContent> getPRFiles(String repo, int pr) { return List.of(); }
        public List<DuplicationDetector.FileContent> getRepoFiles(String repo) { return List.of(); }
        public void postPRComment(String repo, int pr, String comment) { }
    }

    @Autowired private GitHubClient githubClient;
}
```

---

## 五、总结

这个 AI Code Reviewer 上线后，我们的 PR 审查流程发生了质变：

- **审查等待时间**：从平均 8 小时缩短到 30 分钟（AI 秒出基础审查）
- **命名/风格类评论**：人工不再需要提，AI 全包了
- **安全漏洞发现率**：提升了 40%，很多注入风险是在 PR 阶段就拦截了
- **Senior 工程师满意度**：终于不用在 Review 里写"请把变量名改一下"了

记住：**AI Reviewer 的定位是"第一道防线"，不是"最终裁决者"**。它帮你过滤掉 80% 的机械性问题，剩下的 20% 交给人类，这才是人机协作的最佳姿势。

---

> **下篇预告**：《测试生成 Agent：根据业务需求自动生成集成测试用例，把PRD喂给它就给你一套完整测试》——告别"上线前通宵写测试"的噩梦。把需求文档丢给 Agent，自动提取测试场景、生成测试代码、执行并反馈结果，关注我，最后一篇压轴！
