# Prompt 效果评估框架：建立自动化评测流水线

> "你优化了一版 Prompt，感觉效果变好了——是真的变好了还是心理作用？" 没有量化评估的优化都是自嗨。本文构建一套完整的 Prompt 自动化评测体系，让你的每一次 Prompt 改进都有数据支撑。

---

## 一、开篇：你的 Prompt 优化，真的有效吗？

来看一个真实的场景：

小王花了一下午优化了团队的"Java 代码生成 Prompt"。他把角色描述改得更详细了，加了一些约束条件，还贴了两个代码示例。改完之后，他感觉 AI 生成的代码"看起来更规范了"。

PM 问他："优化后效果提升了多少？"

小王："……感觉好多了。"

PM："有数据吗？编译通过率、测试覆盖率、代码审查扣分项有下降吗？"

小王沉默了。

**这就是 Prompt 优化的尴尬——缺乏量化评估。** "感觉变好了"在工程团队里是不可接受的。我们需要像对待代码性能优化一样，用数据说话。

**本文将构建一套完整的 Prompt 自动化评测流水线，让你精确知道每次 Prompt 改动到底带来了多少提升。**

---

## 二、评估指标体系设计

针对 **Java 代码生成场景**，我们设计五维评估体系：

### 维度一：代码正确性（权重 30%）

| 指标 | 说明 | 评分方式 |
|------|------|---------|
| 编译通过率 | 生成的代码能否直接通过 `javac` 编译 | 通过/不通过 |
| 测试通过率 | 用预编写的单元测试验证生成代码 | 通过数/总数 |
| 语法正确性 | Java 语法是否合法 | 基于 AST 解析 |

### 维度二：代码质量（权重 25%）

| 指标 | 说明 | 工具 |
|------|------|------|
| Checkstyle 违规数 | 代码风格问题 | Checkstyle（Google Style） |
| PMD 违规数 | 潜在 Bug 和坏味道 | PMD 7.x |
| 圈复杂度 | 方法级别的复杂度 | JaCoCo/自定义分析 |
| 重复代码率 | 是否存在明显的复制粘贴 | CPD（PMD内置） |

### 维度三：功能完整性（权重 20%）

| 指标 | 说明 |
|------|------|
| 需求覆盖率 | 需求列表中有多少项被实现 |
| 接口完整性 | 指定的 API 端点是否全部生成 |
| 边界情况处理 | 对 null、空集合、异常输入的判断 |

### 维度四：效率指标（权重 15%）

| 指标 | 说明 |
|------|------|
| 响应时间 | Prompt 发送到收到完整响应的时间 |
| Token 消耗 | 输入 + 输出 Token 总量 |
| 代码行数合理性 | 行数是否在合理范围内（不过多不过少） |

### 维度五：安全性（权重 10%）

| 指标 | 说明 | 检测工具 |
|------|------|---------|
| SQL 注入风险 | 是否存在字符串拼接 SQL | 正则+规则 |
| XSS 风险 | 输出是否转义 | 规则检测 |
| 敏感信息泄露 | 是否硬编码密码/密钥 | 正则检测 |
| OWASP Top 10 | 常见安全漏洞 | SpotBugs + 插件 |

---

## 三、完整 Java 评估框架实现

### 3.1 核心评测引擎：PromptBenchmark

```java
import java.io.*;
import java.nio.file.*;
import java.util.*;
import java.util.concurrent.*;
import java.util.function.Consumer;
import java.util.stream.Collectors;

/**
 * Prompt 效果评测引擎
 * 输入Prompt + 测试用例 → 输出多维度评分
 */
public class PromptBenchmark {

    private final LLMClient llmClient;
    private final CodeEvaluator codeEvaluator;
    private final List<TestCase> testCases;
    private final int repeatCount;
    private final ExecutorService executor;

    public PromptBenchmark(LLMClient llmClient, CodeEvaluator codeEvaluator,
                           List<TestCase> testCases, int repeatCount) {
        this.llmClient = llmClient;
        this.codeEvaluator = codeEvaluator;
        this.testCases = new ArrayList<>(testCases);
        this.repeatCount = repeatCount;
        this.executor = Executors.newFixedThreadPool(4);
    }

    /**
     * 对单个 Prompt 进行完整评测
     */
    public BenchmarkResult evaluate(String promptName, String prompt,
                                     String model, Consumer<ProgressEvent> progressCallback) {
        long startTime = System.currentTimeMillis();
        List<TestCaseResult> caseResults = Collections.synchronizedList(new ArrayList<>());

        // 并发执行所有测试用例
        List<CompletableFuture<Void>> futures = new ArrayList<>();

        for (TestCase testCase : testCases) {
            for (int i = 0; i < repeatCount; i++) {
                final int runIndex = i;
                CompletableFuture<Void> future = CompletableFuture.runAsync(() -> {
                    try {
                        TestCaseResult result = executeSingleTest(
                                promptName, prompt, testCase, model, runIndex);
                        caseResults.add(result);

                        if (progressCallback != null) {
                            progressCallback.accept(new ProgressEvent(
                                    caseResults.size(),
                                    testCases.size() * repeatCount,
                                    result));
                        }
                    } catch (Exception e) {
                        System.err.println("Test execution failed: " + e.getMessage());
                    }
                }, executor);
                futures.add(future);
            }
        }

        // 等待所有测试完成
        CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();

        long totalTime = System.currentTimeMillis() - startTime;

        // 聚合结果
        return aggregateResults(promptName, prompt, model, caseResults, totalTime);
    }

    /**
     * 执行单次测试
     */
    private TestCaseResult executeSingleTest(String promptName, String prompt,
                                              TestCase testCase, String model, int runIndex) {
        long genStart = System.nanoTime();

        // 1. 调用 LLM 生成代码
        String finalPrompt = testCase.applyToPrompt(prompt);
        LLMResponse llmResponse = llmClient.generate(finalPrompt, model);
        String generatedCode = llmResponse.content;
        long genTime = (System.nanoTime() - genStart) / 1_000_000; // ms

        // 2. 运行所有评估器
        Map<String, ScoreDetail> scores = new HashMap<>();

        // 2.1 代码正确性评估
        CodeCorrectnessResult correctness = codeEvaluator.evaluateCorrectness(
                generatedCode, testCase.getExpectedFiles());
        scores.put("correctness", correctness.toScoreDetail());

        // 2.2 代码质量评估
        CodeQualityResult quality = codeEvaluator.evaluateQuality(
                generatedCode, testCase.getQualityRules());
        scores.put("quality", quality.toScoreDetail());

        // 2.3 功能完整性评估
        CompletenessResult completeness = codeEvaluator.evaluateCompleteness(
                generatedCode, testCase.getRequirements());
        scores.put("completeness", completeness.toScoreDetail());

        // 2.4 效率评估
        EfficiencyResult efficiency = new EfficiencyResult(
                genTime,
                llmResponse.inputTokens,
                llmResponse.outputTokens,
                countLines(generatedCode)
        );
        scores.put("efficiency", efficiency.toScoreDetail());

        // 2.5 安全性评估
        SecurityResult security = codeEvaluator.evaluateSecurity(generatedCode);
        scores.put("security", security.toScoreDetail());

        // 3. 计算综合评分
        double overallScore = calculateOverallScore(scores);

        return new TestCaseResult(
                testCase.getName(), runIndex, generatedCode,
                scores, overallScore, genTime,
                llmResponse.inputTokens, llmResponse.outputTokens
        );
    }

    /**
     * 聚合所有测试结果
     */
    private BenchmarkResult aggregateResults(String promptName, String prompt,
                                              String model, List<TestCaseResult> results,
                                              long totalTimeMs) {
        // 按测试用例分组
        Map<String, List<TestCaseResult>> grouped = results.stream()
                .collect(Collectors.groupingBy(TestCaseResult::testCaseName));

        List<AggregatedTestCaseResult> aggregatedResults = new ArrayList<>();

        for (Map.Entry<String, List<TestCaseResult>> entry : grouped.entrySet()) {
            List<TestCaseResult> caseResults = entry.getValue();

            double avgOverall = caseResults.stream()
                    .mapToDouble(r -> r.overallScore).average().orElse(0);
            double avgCompileRate = caseResults.stream()
                    .filter(r -> r.scores.get("correctness") != null
                            && r.scores.get("correctness").score >= 0.8)
                    .count() * 100.0 / caseResults.size();
            double avgTestPassRate = caseResults.stream()
                    .filter(r -> r.scores.get("correctness") != null
                            && r.scores.get("correctness").score >= 0.9)
                    .count() * 100.0 / caseResults.size();
            double avgQuality = caseResults.stream()
                    .mapToDouble(r -> r.scores.get("quality") != null
                            ? r.scores.get("quality").score : 0)
                    .average().orElse(0);
            double avgSecurity = caseResults.stream()
                    .mapToDouble(r -> r.scores.get("security") != null
                            ? r.scores.get("security").score : 0)
                    .average().orElse(0);
            long avgGenTime = (long) caseResults.stream()
                    .mapToLong(r -> r.generationTimeMs).average().orElse(0);
            long avgInputTokens = (long) caseResults.stream()
                    .mapToLong(r -> r.inputTokens).average().orElse(0);
            long avgOutputTokens = (long) caseResults.stream()
                    .mapToLong(r -> r.outputTokens).average().orElse(0);

            aggregatedResults.add(new AggregatedTestCaseResult(
                    entry.getKey(), avgOverall, avgCompileRate, avgTestPassRate,
                    avgQuality, avgSecurity, avgGenTime,
                    avgInputTokens, avgOutputTokens, caseResults.size()
            ));
        }

        // 全局聚合
        double globalOverall = aggregatedResults.stream()
                .mapToDouble(r -> r.avgOverallScore).average().orElse(0);
        double globalCompileRate = aggregatedResults.stream()
                .mapToDouble(r -> r.avgCompileRate).average().orElse(0);
        double globalTestPassRate = aggregatedResults.stream()
                .mapToDouble(r -> r.avgTestPassRate).average().orElse(0);
        double globalQuality = aggregatedResults.stream()
                .mapToDouble(r -> r.avgQualityScore).average().orElse(0);
        double globalSecurity = aggregatedResults.stream()
                .mapToDouble(r -> r.avgSecurityScore).average().orElse(0);
        long globalAvgTime = (long) aggregatedResults.stream()
                .mapToLong(r -> r.avgGenerationTimeMs).average().orElse(0);
        long totalInputTokens = aggregatedResults.stream()
                .mapToLong(r -> r.avgInputTokens).sum();
        long totalOutputTokens = aggregatedResults.stream()
                .mapToLong(r -> r.avgOutputTokens).sum();
        double estimatedCost = estimateCost(model, totalInputTokens, totalOutputTokens);

        return new BenchmarkResult(
                promptName, prompt, model, totalTimeMs,
                globalOverall, globalCompileRate, globalTestPassRate,
                globalQuality, globalSecurity, globalAvgTime,
                totalInputTokens, totalOutputTokens, estimatedCost,
                results.size(), aggregatedResults
        );
    }

    private double calculateOverallScore(Map<String, ScoreDetail> scores) {
        double correctness = getScoreWeight(scores, "correctness", 0.30);
        double quality = getScoreWeight(scores, "quality", 0.25);
        double completeness = getScoreWeight(scores, "completeness", 0.20);
        double efficiency = getScoreWeight(scores, "efficiency", 0.15);
        double security = getScoreWeight(scores, "security", 0.10);

        return correctness + quality + completeness + efficiency + security;
    }

    private double getScoreWeight(Map<String, ScoreDetail> scores, String key, double weight) {
        ScoreDetail detail = scores.get(key);
        if (detail == null) return 0;
        return Math.min(detail.score, 1.0) * weight;
    }

    private int countLines(String code) {
        if (code == null) return 0;
        return (int) code.lines().filter(l -> !l.trim().isEmpty()).count();
    }

    private double estimateCost(String model, long inputTokens, long outputTokens) {
        // 使用前面 TokenCounter 中的 pricing
        return TokenCounter.calculateCost((int) inputTokens, model, true)
                + TokenCounter.calculateCost((int) outputTokens, model, false);
    }

    public void shutdown() {
        executor.shutdown();
    }

    // ---- 数据类 ----

    public record TestCase(String name, String description,
                            List<String> requirements,
                            Map<String, String> expectedFiles,
                            QualityRules qualityRules) {
        public String applyToPrompt(String prompt) {
            return prompt + "\n\nTest Case: " + name + "\n" + description;
        }
    }

    public record QualityRules(String checkstyleConfig, String pmdRuleset,
                                 int maxCyclomaticComplexity, int maxMethodLines) {}

    public record ScoreDetail(double score, String description, Map<String, Object> details) {}

    public record TestCaseResult(String testCaseName, int runIndex, String generatedCode,
                                  Map<String, ScoreDetail> scores, double overallScore,
                                  long generationTimeMs, long inputTokens, long outputTokens) {}

    public record AggregatedTestCaseResult(String testCaseName, double avgOverallScore,
                                            double avgCompileRate, double avgTestPassRate,
                                            double avgQualityScore, double avgSecurityScore,
                                            long avgGenerationTimeMs, long avgInputTokens,
                                            long avgOutputTokens, int sampleCount) {
        public String toSummary() {
            return String.format(
                    "%-30s | 综合:%.2f | 编译率:%.0f%% | 测试率:%.0f%% | 质量:%.2f | 安全:%.2f | 耗时:%dms | Token:%d→%d",
                    testCaseName, avgOverallScore, avgCompileRate, avgTestPassRate,
                    avgQualityScore, avgSecurityScore, avgGenerationTimeMs,
                    avgInputTokens, avgOutputTokens
            );
        }
    }

    public record BenchmarkResult(String promptName, String prompt, String model,
                                   long totalTimeMs, double overallScore,
                                   double avgCompileRate, double avgTestPassRate,
                                   double avgQualityScore, double avgSecurityScore,
                                   long avgGenerationTimeMs, long totalInputTokens,
                                   long totalOutputTokens, double estimatedCost,
                                   int totalTests, List<AggregatedTestCaseResult> details) {
        public String toReport() {
            StringBuilder sb = new StringBuilder();
            sb.append("=".repeat(80)).append("\n");
            sb.append("  Prompt Benchmark Report: ").append(promptName).append("\n");
            sb.append("=".repeat(80)).append("\n");
            sb.append(String.format("  Model: %s | 测试次数: %d | 总耗时: %.1fs\n",
                    model, totalTests, totalTimeMs / 1000.0));
            sb.append(String.format("  Token: 输入%d + 输出%d | 估计成本: $%.4f\n",
                    totalInputTokens, totalOutputTokens, estimatedCost));
            sb.append("-".repeat(80)).append("\n");
            sb.append(String.format("  综合评分: %.2f/1.00 | 编译通过率: %.1f%% | 测试通过率: %.1f%%\n",
                    overallScore, avgCompileRate, avgTestPassRate));
            sb.append(String.format("  代码质量: %.2f/1.00 | 安全评分: %.2f/1.00 | 平均生成时间: %dms\n",
                    avgQualityScore, avgSecurityScore, avgGenerationTimeMs));
            sb.append("-".repeat(80)).append("\n");
            sb.append("  各用例详情:\n");
            for (AggregatedTestCaseResult detail : details) {
                sb.append("    ").append(detail.toSummary()).append("\n");
            }
            sb.append("=".repeat(80)).append("\n");
            return sb.toString();
        }
    }

    public record ProgressEvent(int completed, int total, TestCaseResult lastResult) {}

    // ---- 接口定义 ----

    public interface LLMClient {
        LLMResponse generate(String prompt, String model);
    }

    public record LLMResponse(String content, long inputTokens, long outputTokens) {}

    public interface CodeEvaluator {
        CodeCorrectnessResult evaluateCorrectness(String code, Map<String, String> expectedFiles);
        CodeQualityResult evaluateQuality(String code, QualityRules rules);
        CompletenessResult evaluateCompleteness(String code, List<String> requirements);
        SecurityResult evaluateSecurity(String code);
    }

    public record CodeCorrectnessResult(boolean compiles, double testPassRate, String errorMessage) {
        public ScoreDetail toScoreDetail() {
            double score = (compiles ? 0.6 : 0) + (testPassRate * 0.4);
            return new ScoreDetail(score, "正确性评估",
                    Map.of("compiles", compiles, "testPassRate", testPassRate));
        }
    }

    public record CodeQualityResult(int checkstyleViolations, int pmdViolations,
                                      double duplicationRate, int avgCyclomaticComplexity) {
        public ScoreDetail toScoreDetail() {
            double score = Math.max(0, 1.0 - (checkstyleViolations * 0.02)
                    - (pmdViolations * 0.05) - (duplicationRate * 0.5)
                    - (Math.max(0, avgCyclomaticComplexity - 10) * 0.02));
            return new ScoreDetail(Math.min(1.0, score), "质量评估",
                    Map.of("checkstyleViolations", checkstyleViolations,
                            "pmdViolations", pmdViolations,
                            "duplicationRate", duplicationRate,
                            "avgCyclomaticComplexity", avgCyclomaticComplexity));
        }
    }

    public record CompletenessResult(double requirementCoverage, int missingCount,
                                       List<String> missingItems) {
        public ScoreDetail toScoreDetail() {
            return new ScoreDetail(requirementCoverage, "完整性评估",
                    Map.of("missingCount", missingCount, "missingItems", missingItems));
        }
    }

    public record EfficiencyResult(long generationTimeMs, long inputTokens,
                                     long outputTokens, int codeLines) {
        public ScoreDetail toScoreDetail() {
            double timeScore = generationTimeMs < 5000 ? 1.0 :
                    generationTimeMs < 15000 ? 0.7 : 0.3;
            double tokenScore = (inputTokens + outputTokens) < 2000 ? 1.0 :
                    (inputTokens + outputTokens) < 5000 ? 0.7 : 0.4;
            double score = timeScore * 0.5 + tokenScore * 0.5;
            return new ScoreDetail(score, "效率评估",
                    Map.of("generationTimeMs", generationTimeMs,
                            "inputTokens", inputTokens,
                            "outputTokens", outputTokens,
                            "codeLines", codeLines));
        }
    }

    public record SecurityResult(int vulnerabilitiesFound, int sqlInjectionRisks,
                                   int xssRisks, int hardcodedSecrets) {
        public ScoreDetail toScoreDetail() {
            int totalRisks = vulnerabilitiesFound + sqlInjectionRisks + xssRisks + hardcodedSecrets;
            double score = totalRisks == 0 ? 1.0 : Math.max(0, 1.0 - totalRisks * 0.25);
            return new ScoreDetail(score, "安全评估",
                    Map.of("vulnerabilitiesFound", vulnerabilitiesFound,
                            "sqlInjectionRisks", sqlInjectionRisks,
                            "xssRisks", xssRisks,
                            "hardcodedSecrets", hardcodedSecrets));
        }
    }
}
```

### 3.2 代码质量评估器实现

```java
import java.io.*;
import java.nio.file.*;
import java.util.*;
import java.util.regex.*;

/**
 * Java 代码质量评估器
 * 集成 Checkstyle、PMD 等工具进行评估
 */
public class JavaCodeEvaluator implements PromptBenchmark.CodeEvaluator {

    private final Path workspaceDir;
    private final Pattern SQL_INJECTION_PATTERN;
    private final Pattern HARDCODED_SECRET_PATTERN;
    private final Pattern CLASS_PATTERN;

    public JavaCodeEvaluator(String workspaceDir) {
        this.workspaceDir = Paths.get(workspaceDir);
        this.SQL_INJECTION_PATTERN = Pattern.compile(
                "\"(SELECT|INSERT|UPDATE|DELETE|DROP|CREATE).*?\\+.*?\"" +
                "|\"SELECT.*?\"\\s*\\+|\\+\\s*\".*?\"",
                Pattern.CASE_INSENSITIVE);
        this.HARDCODED_SECRET_PATTERN = Pattern.compile(
                "(password|secret|apiKey|token|api_key)\\s*=\\s*\"[^\"]+\"" +
                "|(password|secret|apiKey|token|api_key)\\s*=\\s*'[^']+'",
                Pattern.CASE_INSENSITIVE);
        this.CLASS_PATTERN = Pattern.compile(
                "class\\s+(\\w+)",
                Pattern.MULTILINE);
    }

    @Override
    public PromptBenchmark.CodeCorrectnessResult evaluateCorrectness(
            String code, Map<String, String> expectedFiles) {

        boolean compiles = false;
        String errorMessage = null;

        try {
            // 1. 将代码写入临时目录
            Path tmpDir = Files.createTempDirectory(workspaceDir, "eval_");
            writeCodeToFiles(code, tmpDir);

            // 2. 尝试编译
            ProcessBuilder pb = new ProcessBuilder(
                    "javac", "-d", tmpDir.resolve("classes").toString(),
                    "-sourcepath", tmpDir.toString()
            );
            pb.directory(tmpDir.toFile());

            // 查找所有 .java 文件
            List<Path> javaFiles = Files.walk(tmpDir)
                    .filter(p -> p.toString().endsWith(".java"))
                    .toList();
            for (Path f : javaFiles) {
                pb.command().add(f.toString());
            }

            Process process = pb.start();
            int exitCode = process.waitFor();
            compiles = (exitCode == 0);

            if (!compiles) {
                errorMessage = new String(process.getErrorStream().readAllBytes());
            }

            // 3. 计算测试通过率
            double testPassRate = calculateTestPassRate(code, expectedFiles);

            // 4. 清理临时文件
            deleteRecursively(tmpDir);

            return new PromptBenchmark.CodeCorrectnessResult(compiles, testPassRate, errorMessage);

        } catch (Exception e) {
            return new PromptBenchmark.CodeCorrectnessResult(false, 0, e.getMessage());
        }
    }

    @Override
    public PromptBenchmark.CodeQualityResult evaluateQuality(
            String code, PromptBenchmark.QualityRules rules) {

        int checkstyleViolations = 0;
        int pmdViolations = 0;
        double duplicationRate = 0.0;
        int avgCyclomaticComplexity = 0;

        try {
            Path tmpDir = Files.createTempDirectory(workspaceDir, "quality_");
            writeCodeToFiles(code, tmpDir);

            // 1. 运行 Checkstyle
            checkstyleViolations = runCheckstyle(tmpDir);

            // 2. 运行 PMD
            pmdViolations = runPMD(tmpDir);

            // 3. 分析圈复杂度
            avgCyclomaticComplexity = analyzeCyclomaticComplexity(code);

            // 4. 检测重复代码
            duplicationRate = detectDuplication(code);

            deleteRecursively(tmpDir);

        } catch (Exception e) {
            System.err.println("Quality evaluation error: " + e.getMessage());
        }

        return new PromptBenchmark.CodeQualityResult(
                checkstyleViolations, pmdViolations, duplicationRate, avgCyclomaticComplexity);
    }

    @Override
    public PromptBenchmark.CompletenessResult evaluateCompleteness(
            String code, List<String> requirements) {

        List<String> missingItems = new ArrayList<>();
        int totalRequirements = requirements.size();
        int covered = 0;

        for (String requirement : requirements) {
            String[] keywords = extractKeywords(requirement);
            boolean found = false;

            for (String keyword : keywords) {
                if (code.toLowerCase().contains(keyword.toLowerCase())) {
                    found = true;
                    break;
                }
            }

            if (found) {
                covered++;
            } else {
                missingItems.add(requirement);
            }
        }

        double coverage = totalRequirements > 0 ? (double) covered / totalRequirements : 1.0;
        return new PromptBenchmark.CompletenessResult(coverage, missingItems.size(), missingItems);
    }

    @Override
    public PromptBenchmark.SecurityResult evaluateSecurity(String code) {
        int sqlInjectionRisks = countMatches(SQL_INJECTION_PATTERN, code);
        int hardcodedSecrets = countMatches(HARDCODED_SECRET_PATTERN, code);

        int xssRisks = 0;
        if (code.contains("document.write") || code.contains("innerHTML")
                || code.contains("eval(")) {
            xssRisks++;
        }

        int totalVulnerabilities = sqlInjectionRisks + xssRisks + hardcodedSecrets;
        return new PromptBenchmark.SecurityResult(
                totalVulnerabilities, sqlInjectionRisks, xssRisks, hardcodedSecrets);
    }

    // ---- 辅助方法 ----

    private void writeCodeToFiles(String code, Path dir) throws IOException {
        // 按 "// === FILENAME:" 分割为多个文件
        String[] parts = code.split("//\\s*===\\s*FILENAME:");
        if (parts.length <= 1) {
            // 单文件模式
            Files.writeString(dir.resolve("Generated.java"), code);
            return;
        }

        for (int i = 1; i < parts.length; i++) {
            String part = parts[i];
            int newlineIdx = part.indexOf('\n');
            if (newlineIdx > 0) {
                String fileName = part.substring(0, newlineIdx).trim();
                String fileContent = part.substring(newlineIdx + 1).trim();
                Path filePath = dir.resolve(fileName);
                Files.createDirectories(filePath.getParent());
                Files.writeString(filePath, fileContent);
            }
        }
    }

    private double calculateTestPassRate(String code, Map<String, String> expectedFiles) {
        // 简化实现：检查代码中是否包含预期的关键方法/注解
        if (expectedFiles == null || expectedFiles.isEmpty()) return 1.0;

        int totalChecks = 0;
        int passedChecks = 0;

        for (Map.Entry<String, String> entry : expectedFiles.entrySet()) {
            String expectedContent = entry.getValue();
            String[] checks = expectedContent.split("\\n");

            for (String check : checks) {
                check = check.trim();
                if (check.isEmpty()) continue;
                totalChecks++;

                if (check.startsWith("!")) {
                    // 否定检查：不包含
                    if (!code.contains(check.substring(1))) passedChecks++;
                } else {
                    // 正向检查：包含
                    if (code.contains(check)) passedChecks++;
                }
            }
        }

        return totalChecks > 0 ? (double) passedChecks / totalChecks : 1.0;
    }

    private int runCheckstyle(Path dir) throws IOException, InterruptedException {
        // 简化版：用正则检查基本规范
        int violations = 0;
        List<Path> javaFiles = Files.walk(dir)
                .filter(p -> p.toString().endsWith(".java"))
                .toList();

        for (Path file : javaFiles) {
            String content = Files.readString(file);

            // 检查缩进（Tab vs 空格）
            if (content.contains("\t")) violations++;

            // 检查行长度 > 120
            for (String line : content.split("\n")) {
                if (line.length() > 120) violations++;
            }

            // 检查未使用的 import
            String[] imports = content.split("import ");
            for (int i = 1; i < imports.length; i++) {
                String importClass = imports[i].split(";")[0].trim();
                String simpleName = importClass.substring(importClass.lastIndexOf('.') + 1);
                if (!content.substring(content.indexOf(importClass) + importClass.length())
                        .contains(simpleName)) {
                    violations++;
                }
            }
        }

        return violations;
    }

    private int runPMD(Path dir) throws IOException {
        int violations = 0;
        List<Path> javaFiles = Files.walk(dir)
                .filter(p -> p.toString().endsWith(".java"))
                .toList();

        for (Path file : javaFiles) {
            String content = Files.readString(file);

            // 空 catch 块
            if (Pattern.compile("catch\\s*\\([^)]*\\)\\s*\\{\\s*\\}")
                    .matcher(content).find()) {
                violations++;
            }

            // System.out.println 在非 main 方法中
            if (content.contains("System.out.println")
                    && !content.contains("public static void main")) {
                violations++;
            }

            // 方法超过 50 行
            Matcher methodMatcher = Pattern.compile(
                    "(public|private|protected)\\s+[^{]+\\{").matcher(content);
            while (methodMatcher.find()) {
                int start = methodMatcher.end();
                int braceCount = 1;
                int end = start;
                int lines = 0;
                while (braceCount > 0 && end < content.length()) {
                    if (content.charAt(end) == '{') braceCount++;
                    if (content.charAt(end) == '}') braceCount--;
                    if (content.charAt(end) == '\n') lines++;
                    end++;
                }
                if (lines > 50) violations++;
            }
        }

        return violations;
    }

    private int analyzeCyclomaticComplexity(String code) {
        int totalComplexity = 0;
        int methodCount = 0;

        String[] methods = code.split("(public|private|protected)\\s+\\w+\\s+\\w+\\s*\\(");
        for (int i = 1; i < methods.length; i++) {
            String methodBody = methods[i];
            int complexity = 1;

            complexity += countOccurrences(methodBody, "if ");
            complexity += countOccurrences(methodBody, "else if");
            complexity += countOccurrences(methodBody, "for ");
            complexity += countOccurrences(methodBody, "while ");
            complexity += countOccurrences(methodBody, "case ");
            complexity += countOccurrences(methodBody, "&&");
            complexity += countOccurrences(methodBody, "||");
            complexity += countOccurrences(methodBody, "?");
            complexity += countOccurrences(methodBody, "catch ");

            totalComplexity += complexity;
            methodCount++;
        }

        return methodCount > 0 ? totalComplexity / methodCount : 0;
    }

    private double detectDuplication(String code) {
        String[] lines = code.split("\n");
        Set<String> uniqueLines = new HashSet<>();
        int duplicateLines = 0;

        for (String line : lines) {
            String trimmed = line.trim();
            if (trimmed.length() > 10) {
                if (!uniqueLines.add(trimmed)) {
                    duplicateLines++;
                }
            }
        }

        return lines.length > 0 ? (double) duplicateLines / lines.length : 0;
    }

    private String[] extractKeywords(String requirement) {
        return requirement.split("[，,、\\s]+");
    }

    private int countMatches(Pattern pattern, String text) {
        Matcher matcher = pattern.matcher(text);
        int count = 0;
        while (matcher.find()) count++;
        return count;
    }

    private int countOccurrences(String text, String pattern) {
        int count = 0;
        int idx = 0;
        while ((idx = text.indexOf(pattern, idx)) != -1) {
            count++;
            idx += pattern.length();
        }
        return count;
    }

    private void deleteRecursively(Path path) throws IOException {
        if (Files.exists(path)) {
            Files.walk(path)
                    .sorted(Comparator.reverseOrder())
                    .forEach(p -> {
                        try { Files.deleteIfExists(p); } catch (IOException ignored) {}
                    });
        }
    }
}
```

---

## 四、批量评测脚本：对比 Prompt 版本

```java
import java.io.*;
import java.util.*;
import java.util.stream.*;

/**
 * Prompt 批量对比评测工具
 * 用同一组标准测试用例跑不同 Prompt 版本，生成对比报告
 */
public class PromptVersionComparator {

    private final PromptBenchmark benchmark;
    private final List<PromptVersion> promptVersions;

    public record PromptVersion(String name, String prompt, String model,
                                 String description, String author) {}

    public PromptVersionComparator(PromptBenchmark benchmark) {
        this.benchmark = benchmark;
        this.promptVersions = new ArrayList<>();
    }

    public void addVersion(PromptVersion version) {
        this.promptVersions.add(version);
    }

    /**
     * 运行所有版本的对比评测
     */
    public ComparisonReport runComparison() {
        List<PromptBenchmark.BenchmarkResult> results = new ArrayList<>();

        System.out.println("开始评测 " + promptVersions.size() + " 个 Prompt 版本...");

        for (int i = 0; i < promptVersions.size(); i++) {
            PromptVersion version = promptVersions.get(i);
            System.out.println("[" + (i + 1) + "/" + promptVersions.size() + "] "
                    + "评测: " + version.name);

            PromptBenchmark.BenchmarkResult result = benchmark.evaluate(
                    version.name, version.prompt, version.model,
                    event -> {
                        if (event.completed % 5 == 0) {
                            System.out.print(".");
                        }
                    }
            );
            results.add(result);
            System.out.println(" 完成! 综合评分: " + String.format("%.2f", result.overallScore));
        }

        return buildComparisonReport(results);
    }

    /**
     * 构建对比分析报告
     */
    private ComparisonReport buildComparisonReport(
            List<PromptBenchmark.BenchmarkResult> results) {

        // 找出最佳版本（按综合评分）
        PromptBenchmark.BenchmarkResult best = results.stream()
                .max(Comparator.comparingDouble(r -> r.overallScore))
                .orElse(null);

        // 找出性价比最高的版本（评分/成本）
        PromptBenchmark.BenchmarkResult bestValue = results.stream()
                .filter(r -> r.estimatedCost > 0)
                .max(Comparator.comparingDouble(r -> r.overallScore / r.estimatedCost))
                .orElse(null);

        // 找出编译通过率最高的版本
        PromptBenchmark.BenchmarkResult mostReliable = results.stream()
                .max(Comparator.comparingDouble(r -> r.avgCompileRate))
                .orElse(null);

        // 生成详细对比
        List<VersionComparisonRow> rows = new ArrayList<>();
        PromptBenchmark.BenchmarkResult baseline = results.isEmpty() ? null : results.get(0);

        for (PromptBenchmark.BenchmarkResult result : results) {
            double scoreDiff = baseline != null
                    ? result.overallScore - baseline.overallScore : 0;
            double costDiff = baseline != null
                    ? result.estimatedCost - baseline.estimatedCost : 0;
            double compileDiff = baseline != null
                    ? result.avgCompileRate - baseline.avgCompileRate : 0;

            String verdict;
            if (scoreDiff > 0.05) verdict = "显著提升";
            else if (scoreDiff > 0.02) verdict = "略有提升";
            else if (scoreDiff > -0.02) verdict = "无显著变化";
            else if (scoreDiff > -0.05) verdict = "略有下降";
            else verdict = "显著下降";

            rows.add(new VersionComparisonRow(
                    result.promptName, result.overallScore, scoreDiff,
                    result.avgCompileRate, compileDiff,
                    result.avgQualityScore, result.avgSecurityScore,
                    result.estimatedCost, costDiff,
                    result.avgGenerationTimeMs, verdict
            ));
        }

        return new ComparisonReport(best, bestValue, mostReliable, rows);
    }

    public record VersionComparisonRow(String versionName, double overallScore,
                                        double scoreDiffVsBaseline, double compileRate,
                                        double compileRateDiff, double qualityScore,
                                        double securityScore, double estimatedCost,
                                        double costDiff, long avgGenTime, String verdict) {}

    public record ComparisonReport(PromptBenchmark.BenchmarkResult bestOverall,
                                    PromptBenchmark.BenchmarkResult bestValue,
                                    PromptBenchmark.BenchmarkResult mostReliable,
                                    List<VersionComparisonRow> rows) {

        public String toMarkdownReport() {
            StringBuilder sb = new StringBuilder();
            sb.append("# Prompt 版本对比评测报告\n\n");

            sb.append("## 获奖版本\n\n");
            sb.append("| 奖项 | 版本 | 评分/指标 |\n");
            sb.append("|------|------|----------|\n");
            sb.append(String.format("| 综合最佳 | %s | %.2f |\n",
                    bestOverall.promptName, bestOverall.overallScore));
            sb.append(String.format("| 性价比最高 | %s | %.2f/$ |\n",
                    bestValue.promptName,
                    bestValue.estimatedCost > 0
                            ? bestValue.overallScore / bestValue.estimatedCost : 0));
            sb.append(String.format("| 最稳定 | %s | %.1f%% 编译率 |\n\n",
                    mostReliable.promptName, mostReliable.avgCompileRate));

            sb.append("## 详细对比（以第一个版本为基线）\n\n");
            sb.append("| 版本 | 综合评分 | Δ评分 | 编译率 | Δ编译 | 质量 | 安全 | 成本 | Δ成本 | 耗时 | 结论 |\n");
            sb.append("|------|:------:|:----:|:----:|:---:|:---:|:---:|:---:|:---:|:---:|------|\n");

            for (VersionComparisonRow row : rows) {
                sb.append(String.format("| %s | %.3f | %+.3f | %.1f%% | %+.1f%% | %.2f | %.2f | $%.4f | %+.4f | %dms | %s |\n",
                        row.versionName, row.overallScore, row.scoreDiffVsBaseline,
                        row.compileRate, row.compileRateDiff,
                        row.qualityScore, row.securityScore,
                        row.estimatedCost, row.costDiff,
                        row.avgGenTime, row.verdict));
            }

            return sb.toString();
        }
    }

    // ---- 使用示例 ----
    public static void main(String[] args) {
        // 构建测试用例
        List<PromptBenchmark.TestCase> testCases = Arrays.asList(
                new PromptBenchmark.TestCase(
                        "User CRUD",
                        "Generate CRUD for User entity with fields: id, username, email, age",
                        Arrays.asList("@RestController", "@Valid", "分页", "异常处理", "Swagger"),
                        Map.of("UserController.java", "@RestController\n@Valid\nPage"),
                        new PromptBenchmark.QualityRules("google_checks.xml", "rules.xml", 10, 50)
                ),
                new PromptBenchmark.TestCase(
                        "Order Service",
                        "Generate OrderService with create, cancel, and list methods",
                        Arrays.asList("@Service", "@Transactional", "Optional", "日志"),
                        Map.of("OrderService.java", "@Service\n@Transactional\nlog"),
                        new PromptBenchmark.QualityRules("google_checks.xml", "rules.xml", 8, 40)
                ),
                new PromptBenchmark.TestCase(
                        "Unit Test",
                        "Generate JUnit 5 test for UserService",
                        Arrays.asList("@Test", "@Mock", "Mockito", "assert"),
                        Map.of("UserServiceTest.java", "@Test\n@Mock\nassert"),
                        new PromptBenchmark.QualityRules("google_checks.xml", "rules.xml", 5, 30)
                )
        );

        // 创建评测引擎
        JavaCodeEvaluator evaluator = new JavaCodeEvaluator("/tmp/prompt_benchmark");
        PromptBenchmark benchmark = new PromptBenchmark(
                new MockLLMClient(),
                evaluator,
                testCases,
                3  // 每个用例重复3次取平均
        );

        PromptVersionComparator comparator = new PromptVersionComparator(benchmark);

        // 添加不同版本的 Prompt
        comparator.addVersion(new PromptVersionComparator.PromptVersion(
                "v1-基础版", "Generate Java CRUD for {{entity}}. Use Spring Boot.", "gpt-4o-mini",
                "最简版本，仅描述基本需求", "Zhang San"
        ));

        comparator.addVersion(new PromptVersionComparator.PromptVersion(
                "v2-详细版", """
                    You are a Java expert. Generate CRUD for {{entity}}.
                    Requirements: @Valid validation, @RestControllerAdvice,
                    constructor injection, pagination, Swagger annotations.
                    """, "gpt-4o-mini",
                "增加了详细约束和角色定义", "Zhang San"
        ));

        comparator.addVersion(new PromptVersionComparator.PromptVersion(
                "v3-优化版", """
                    Generate Spring Boot CRUD for {{entity}}.
                    MUST include: @Valid, @RestControllerAdvice, @RequiredArgsConstructor.
                    DON'T add: rate limiting, audit, AOP.
                    Output: Entity, Mapper, Service+Impl, Controller, DTO, VO.
                    """, "gpt-4o-mini",
                "精简英文版 + DO/DON'T 约束 + 明确输出文件列表", "Zhang San"
        ));

        // 运行对比
        ComparisonReport report = comparator.runComparison();
        System.out.println("\n" + report.toMarkdownReport());

        benchmark.shutdown();
    }
}
```

---

## 五、持续优化循环：评测 → 分析 → 改进 → 再评测

```java
/**
 * Prompt 持续优化循环管理器
 * 自动运行 "评测 → 分析 → 改进建议 → 再评测" 的循环
 */
public class PromptOptimizationLoop {

    private final PromptBenchmark benchmark;
    private final PromptVersionComparator comparator;
    private final List<OptimizationResult> history;
    private final int maxIterations;
    private final double targetScore;

    public PromptOptimizationLoop(PromptBenchmark benchmark,
                                   int maxIterations, double targetScore) {
        this.benchmark = benchmark;
        this.comparator = new PromptVersionComparator(benchmark);
        this.history = new ArrayList<>();
        this.maxIterations = maxIterations;
        this.targetScore = targetScore;
    }

    /**
     * 运行优化循环
     */
    public OptimizationResult optimize(PromptVersionComparator.PromptVersion baseVersion) {
        PromptVersionComparator.PromptVersion currentVersion = baseVersion;
        int iteration = 0;

        System.out.println("=== Prompt 优化循环启动 ===");
        System.out.println("目标评分: " + targetScore);
        System.out.println("最大迭代: " + maxIterations + "\n");

        while (iteration < maxIterations) {
            iteration++;
            System.out.println("--- 第 " + iteration + " 轮 ---");

            // Step 1: 评测当前版本
            comparator.addVersion(currentVersion);
            PromptVersionComparator.ComparisonReport report = comparator.runComparison();

            PromptVersionComparator.VersionComparisonRow currentRow = report.rows.get(
                    report.rows.size() - 1);

            // Step 2: 检查是否达标
            if (currentRow.overallScore >= targetScore) {
                System.out.println("达标! 综合评分: " + String.format("%.2f", currentRow.overallScore));
                OptimizationResult result = new OptimizationResult(
                        currentVersion, iteration, true, history);
                history.add(result);
                return result;
            }

            // Step 3: 分析短板并生成改进建议
            List<String> suggestions = analyzeWeaknesses(currentRow);
            System.out.println("改进建议:");
            suggestions.forEach(s -> System.out.println("  - " + s));

            // Step 4: 应用改进建议，生成新版本
            String improvedPrompt = applySuggestions(currentVersion.prompt, suggestions);
            currentVersion = new PromptVersionComparator.PromptVersion(
                    currentVersion.name + "-opt" + iteration,
                    improvedPrompt,
                    currentVersion.model,
                    "自动优化第" + iteration + "轮: " + String.join(", ", suggestions),
                    "AutoOptimizer"
            );

            history.add(new OptimizationResult(
                    currentVersion, iteration, false, List.of()));
        }

        System.out.println("达到最大迭代次数，未达标。");
        return new OptimizationResult(currentVersion, iteration, false, history);
    }

    /**
     * 分析评分短板
     */
    private List<String> analyzeWeaknesses(
            PromptVersionComparator.VersionComparisonRow result) {
        List<String> suggestions = new ArrayList<>();

        if (result.compileRate < 80) {
            suggestions.add("编译通过率低(" + String.format("%.1f%%", result.compileRate)
                    + ")，建议添加更明确的代码规范约束和导入声明要求");
        }
        if (result.qualityScore < 0.6) {
            suggestions.add("代码质量不足(" + String.format("%.2f", result.qualityScore)
                    + ")，建议添加Checkstyle/PMD相关的代码风格约束");
        }
        if (result.securityScore < 0.7) {
            suggestions.add("安全评分低(" + String.format("%.2f", result.securityScore)
                    + ")，建议添加安全约束（参数校验、SQL注入防护、敏感信息处理）");
        }
        if (result.avgGenTime > 15000) {
            suggestions.add("响应时间过长(" + result.avgGenTime
                    + "ms)，考虑精简Prompt或切换到更快的模型");
        }
        if (result.estimatedCost > 0.01 && result.overallScore < 0.85) {
            suggestions.add("性价比不足，尝试用英文写Prompt降低Token消耗");
        }

        if (suggestions.isEmpty()) {
            suggestions.add("各维度表现均衡，尝试增加Few-shot示例提升稳定性");
        }

        return suggestions;
    }

    /**
     * 应用改进建议到 Prompt
     */
    private String applySuggestions(String prompt, List<String> suggestions) {
        StringBuilder enhanced = new StringBuilder(prompt.trim());

        for (String suggestion : suggestions) {
            if (suggestion.contains("编译通过率")) {
                enhanced.append("\n\nIMPORTANT: Ensure the generated code compiles. ")
                        .append("Include all necessary imports. ")
                        .append("Use standard Java 17+ syntax only.");
            }
            if (suggestion.contains("代码质量")) {
                enhanced.append("\n\nCode quality requirements: ")
                        .append("Follow Google Java Style Guide. ")
                        .append("Methods < 50 lines. Max cyclomatic complexity: 10.");
            }
            if (suggestion.contains("安全")) {
                enhanced.append("\n\nSecurity constraints: ")
                        .append("Use parameterized queries. ")
                        .append("Validate all inputs with @Valid. ")
                        .append("Never hardcode secrets or passwords.");
            }
            if (suggestion.contains("响应时间") || suggestion.contains("Token")) {
                enhanced = new StringBuilder(
                        minimizedPrompt(enhanced.toString()));
            }
        }

        return enhanced.toString();
    }

    private String minimizedPrompt(String prompt) {
        // 去掉冗余词，压缩 Token
        return prompt
                .replace("please ", "")
                .replace("Please ", "")
                .replace("thank you", "")
                .replace("make sure to ", "")
                .replace("IMPORTANT: ", "MUST: ");
    }

    public record OptimizationResult(PromptVersionComparator.PromptVersion finalVersion,
                                      int iterations, boolean targetReached,
                                      List<OptimizationResult> history) {}
}
```

---

## 六、评测报告可视化

```java
/**
 * 评测报告生成器
 * 支持控制台表格输出和 HTML 报告
 */
public class BenchmarkReportGenerator {

    public static String generateConsoleReport(
            PromptBenchmark.BenchmarkResult result) {
        return result.toReport();
    }

    public static String generateComparisonTable(
            List<PromptBenchmark.BenchmarkResult> results) {
        StringBuilder sb = new StringBuilder();
        sb.append("┌" + "─".repeat(100) + "┐\n");
        sb.append(String.format("│ %-20s │ %8s │ %8s │ %8s │ %8s │ %8s │ %8s │ %8s │\n",
                "Prompt版本", "综合评分", "编译率", "测试率", "质量分", "安全分", "成本($)", "耗时(ms)"));
        sb.append("├" + "─".repeat(100) + "┤\n");

        for (PromptBenchmark.BenchmarkResult r : results) {
            sb.append(String.format("│ %-20s │ %8.3f │ %7.1f%% │ %7.1f%% │ %8.3f │ %8.3f │ %8.4f │ %8d │\n",
                    truncate(r.promptName, 20),
                    r.overallScore,
                    r.avgCompileRate,
                    r.avgTestPassRate,
                    r.avgQualityScore,
                    r.avgSecurityScore,
                    r.estimatedCost,
                    r.avgGenerationTimeMs));
        }
        sb.append("└" + "─".repeat(100) + "┘\n");
        return sb.toString();
    }

    public static String generateHTMLReport(
            PromptVersionComparator.ComparisonReport report) {
        return String.format("""
            <!DOCTYPE html>
            <html>
            <head>
                <meta charset="UTF-8">
                <title>Prompt Benchmark Report</title>
                <style>
                    body { font-family: monospace; margin: 40px; background: #1e1e1e; color: #d4d4d4; }
                    table { border-collapse: collapse; width: 100%%; margin: 20px 0; }
                    th, td { border: 1px solid #444; padding: 10px; text-align: center; }
                    th { background: #333; }
                    .best { background: #1a3a1a; color: #4ec94e; }
                    .warning { background: #3a3a1a; color: #c9c94e; }
                    .danger { background: #3a1a1a; color: #c94e4e; }
                </style>
            </head>
            <body>
                <h1>Prompt Benchmark Report</h1>
                %s
            </body>
            </html>
            """, report.toMarkdownReport()
                    .replace("| ", "<tr><td>")
                    .replace(" |", "</td></tr>")
                    .replace("|", "</td><td>"));
    }

    private static String truncate(String s, int maxLen) {
        return s.length() > maxLen ? s.substring(0, maxLen - 3) + "..." : s;
    }
}
```

---

## 七、总结

构建 Prompt 评估框架不是一次性的工作，而是一个持续优化的基础设施。

**核心流程：**

```
编写/修改 Prompt → 运行 Benchmark → 查看评分报告
                                        ↓
                                   短板分析
                                        ↓
                                   改进 Prompt
                                        ↓
                                   再次 Benchmark（循环）
```

**关键收益：**

1. **量化决策：** 不再凭"感觉"判断 Prompt 好坏
2. **回归检测：** 每次 Prompt 修改后自动检测是否退化
3. **团队共识：** 用数据说话，减少主观争论
4. **持续改进：** 明确短板，知道下一步该优化什么
5. **成本控制：** 评分 + 成本的综合视角，找到性价比最优解

**推广建议：** 先构建一小组核心测试用例（5-10个），覆盖团队最常见的代码生成场景。跑通评测流程后，再逐步扩展测试用例库和评估指标。

---

**下一篇预告：** 《Prompt 持续优化：从能用→好用→复用》—— 我们将探讨如何将 Prompt 优化融入日常开发流程，建立团队 Prompt 资产的持续演进机制。敬请期待！
