# Prompt 版本管理与 A/B 测试：像管理代码一样管理你的提示词

> 别改了 Prompt 也不知道效果变好还是变差。

---

## 一、开篇：一个熟悉的场景

周一早上，你花了一上午反复打磨 Prompt，自测了几条 case 觉得效果不错，信心满满地替换了生产环境的 Prompt。

周三下午，同事在 Code Review 时随口说了一句："最近 AI 生成的代码质量好像下降了，很多边界条件都没处理好。"

你心里一紧，打开日志一行一行对比。半小时后，你发现问题出在——周一那版新 Prompt 上。那个"优化"让 AI 在某些场景下漏掉了关键的校验逻辑。

更扎心的是：**你没有保留旧版 Prompt，也没有做对比测试，甚至无法量化"效果变差了"到底差了多少。**

如果你也经历过这种时刻，这篇文章就是为你准备的。

---

## 二、为什么 Prompt 需要像代码一样管理？

我们先对齐一个认知：**Prompt 不是一段随意编写的文本，它是 AI 应用的核心"源代码"。**

| 对比维度 | 传统代码 | Prompt |
|---------|---------|--------|
| 版本管理 | Git 成熟生态 | 大多还在复制粘贴 |
| 测试验证 | 单元测试/集成测试 | 靠人眼"感觉" |
| 发布流程 | CI/CD 自动化 | 手动改配置 |
| 回滚能力 | 一键回滚 | 找不到旧版 |
| 效果度量 | 性能指标/错误率 | "好像还行" |

当你的应用每天调用 LLM 成千上万次，Prompt 的质量波动会直接影响业务指标。是时候用工程化的方式管理 Prompt 了。

---

## 三、方案一：Git 管理 Prompt 文件（轻量入门）

最简单也最可控的方案——把 Prompt 当代码文件用 Git 管理。

### 3.1 目录结构设计

```bash
prompts/
├── code-generation/
│   ├── java-controller.prompt.md        # Java Controller 生成
│   ├── java-service.prompt.md           # Java Service 生成
│   └── sql-generator.prompt.md          # SQL 生成
├── code-review/
│   ├── java-review.prompt.md            # Java 代码审查
│   └── security-check.prompt.md         # 安全检查
├── documentation/
│   └── api-doc.prompt.md                # API 文档生成
└── CHANGELOG.md                         # Prompt 变更日志
```

### 3.2 `.prompt.md` 文件命名规范

每个 Prompt 文件采用 Markdown 格式，头部包含 YAML Front Matter 记录元信息：

```markdown
---
name: java-controller
version: 2.1.0
author: zhangsan
created: 2026-01-15
updated: 2026-05-03
tags: [java, controller, spring-boot]
target_model: [gpt-4o, deepseek-v3]
expected_output: Java Spring Boot Controller
metrics:
  code_correctness: 0.87
  compile_pass_rate: 0.94
  avg_tokens: 2450
---

# Java Controller 生成 Prompt v2.1.0

## System Prompt

你是一个资深的 Java Spring Boot 开发专家，精通 RESTful API 设计...

## User Prompt Template

请根据以下需求生成一个 Spring Boot Controller：

### 需求
{requirement}

### 技术要求
- Spring Boot 3.x
- 参数校验使用 @Valid
- 统一返回 Result<T> 包装
- 异常统一由 GlobalExceptionHandler 处理

### 输出要求
1. 仅输出 Java 代码，不输出解释
2. 包含必要的 import
3. 每个方法添加 JavaDoc
```

### 3.3 Git 工作流

```bash
# 创建新功能分支
git checkout -b prompt/optimize-java-controller

# 修改 Prompt
vim prompts/code-generation/java-controller.prompt.md

# 更新版本号和 CHANGELOG
# version: 2.1.0 -> 2.2.0

# 提交
git add prompts/
git commit -m "feat(prompt): java-controller v2.2.0 - 增加事务处理提示"

# 发起 Prompt PR（后面讲）
```

**优点**：零依赖，开发团队无需学习新工具；Git 历史即完整变更记录。

**缺点**：缺少专用 UI，对比 Prompt 差异不直观；无内置的 A/B 测试和效果追踪。

---

## 四、方案二：YAML/JSON 管理 Prompt 模板（带变量替换）

当 Prompt 中包含大量变量时，纯 Markdown 文件不够灵活。我们可以用 YAML 管理 Prompt 模板。

### 4.1 Prompt 模板 YAML 结构

```yaml
# prompts/code-generation/java-controller.prompt.yaml
meta:
  name: java-controller
  version: 2.1.0
  author: zhangsan
  created: 2026-01-15
  target_models:
    - gpt-4o
    - deepseek-v3
  tags:
    - java
    - controller
    - spring-boot

system_prompt: |
  你是一个资深的 Java Spring Boot 开发专家，精通 RESTful API 设计。
  
  你需要遵循以下规则：
  1. 使用 Spring Boot 3.x + Java 17
  2. 参数校验使用 @Valid 注解
  3. 统一返回 {{response_wrapper}} 包装
  4. 异常由全局异常处理器统一处理
  5. 遵循阿里巴巴 Java 开发手册

user_prompt_template: |
  请根据以下需求生成一个 Spring Boot Controller：

  ## 需求
  {{requirement}}

  ## 接口路径
  {{api_path}}

  ## 请求方式
  {{http_method}}

  ## 请求参数
  {{request_params}}

  ## 响应数据
  {{response_data}}

  ## 技术要求
  {{tech_requirements}}

  ## 输出要求
  - 仅输出 Java 代码，不输出解释
  - 包含所有必要的 import 语句
  - 每个 public 方法添加 JavaDoc

examples:
  - input:
      requirement: "用户登录接口"
      api_path: "/api/user/login"
      http_method: "POST"
      request_params: "username(String), password(String)"
      response_data: "token(String), userId(Long)"
      tech_requirements: "密码使用 BCrypt 加密，返回 JWT Token"
    expected_output: |
      @RestController
      @RequestMapping("/api/user")
      public class UserController {
          // ...预期代码
      }

version_history:
  - version: 2.1.0
    date: 2026-05-03
    changes:
      - 增加 JWT Token 生成逻辑的提示
      - 补充异常场景处理说明
    author: zhangsan
  - version: 2.0.0
    date: 2026-04-20
    changes:
      - 重构 Prompt 结构，引入 few-shot examples
      - 升级 target_model 到 gpt-4o
    author: zhangsan
```

### 4.2 Java 端变量替换引擎

```java
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;
import java.util.Map;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

public class PromptTemplateEngine {

    private static final Pattern VARIABLE_PATTERN = Pattern.compile("\\{\\{(\\w+)\\}\\}");
    private static final ObjectMapper YAML_MAPPER = new ObjectMapper(new YAMLFactory());

    /**
     * 加载 YAML 格式的 Prompt 模板
     */
    public PromptTemplate load(String templatePath) {
        try {
            return YAML_MAPPER.readValue(
                getClass().getClassLoader().getResourceAsStream(templatePath),
                PromptTemplate.class
            );
        } catch (Exception e) {
            throw new PromptLoadException("Failed to load prompt template: " + templatePath, e);
        }
    }

    /**
     * 渲染 Prompt：将模板中的 {{变量}} 替换为实际值
     */
    public RenderedPrompt render(PromptTemplate template, Map<String, String> variables) {
        String systemPrompt = replaceVariables(template.getSystemPrompt(), variables);
        String userPrompt = replaceVariables(template.getUserPromptTemplate(), variables);

        return RenderedPrompt.builder()
                .systemPrompt(systemPrompt)
                .userPrompt(userPrompt)
                .templateName(template.getMeta().getName())
                .templateVersion(template.getMeta().getVersion())
                .build();
    }

    /**
     * 替换模板中的 {{variable}} 占位符
     */
    private String replaceVariables(String template, Map<String, String> variables) {
        Matcher matcher = VARIABLE_PATTERN.matcher(template);
        StringBuilder result = new StringBuilder();

        while (matcher.find()) {
            String variableName = matcher.group(1);
            String value = variables.getOrDefault(variableName, "");
            if (value.isEmpty()) {
                throw new IllegalArgumentException(
                    "Variable '" + variableName + "' not provided for template rendering"
                );
            }
            matcher.appendReplacement(result, Matcher.quoteReplacement(value));
        }
        matcher.appendTail(result);

        return result.toString();
    }
}
```

```java
import lombok.Builder;
import lombok.Data;
import java.util.List;

@Data
public class PromptTemplate {
    private MetaInfo meta;
    private String systemPrompt;
    private String userPromptTemplate;
    private List<Example> examples;
    private List<VersionEntry> versionHistory;

    @Data
    public static class MetaInfo {
        private String name;
        private String version;
        private String author;
        private List<String> targetModels;
    }

    @Data
    public static class Example {
        private Map<String, String> input;
        private String expectedOutput;
    }

    @Data
    public static class VersionEntry {
        private String version;
        private String date;
        private List<String> changes;
        private String author;
    }
}

@Data
@Builder
public class RenderedPrompt {
    private String systemPrompt;
    private String userPrompt;
    private String templateName;
    private String templateVersion;
}
```

### 4.3 使用示例

```java
// 加载模板
PromptTemplateEngine engine = new PromptTemplateEngine();
PromptTemplate template = engine.load("prompts/code-generation/java-controller.prompt.yaml");

// 准备变量
Map<String, String> variables = Map.of(
    "requirement", "用户登录接口，支持用户名密码登录",
    "api_path", "/api/user/login",
    "http_method", "POST",
    "request_params", "登录请求，包含 username 和 password 字段",
    "response_data", "登录响应，包含 JWT token 和用户基本信息",
    "tech_requirements", "使用 Spring Security + BCrypt，返回 JWT Token",
    "response_wrapper", "Result<T>"
);

// 渲染
RenderedPrompt rendered = engine.render(template, variables);

// 发送给 LLM
String response = llmClient.chat(rendered.getSystemPrompt(), rendered.getUserPrompt());

// 记录使用的 Prompt 版本（方便追溯）
log.info("Used prompt: {} v{}", rendered.getTemplateName(), rendered.getTemplateVersion());
```

---

## 五、方案三：专业平台 Prompt 管理

如果你的团队对 Prompt 管理有更高要求，可以考虑专业平台。

### Prompty

**Prompty** 是微软推出的 Prompt 资产标准化格式，核心思想是把 Prompt 的元信息（模型、参数、示例）和 Prompt 内容放在同一个文件中，让 Prompt 成为可测试、可追踪的资产。

```yaml
---
name: Java Code Generator
description: Generates Java Spring Boot code from requirements.
authors:
  - zhangsan
model:
  api: chat
  configuration:
    type: azure_openai
    azure_deployment: gpt-4o
    max_tokens: 4096
    temperature: 0.3
  parameters:
    - name: requirement
      type: string
      default: "Generate a REST API endpoint"
sample:
  requirement: "Generate a User CRUD endpoint"
---

system:
你是一个资深的 Java Spring Boot 开发专家。

user:
根据以下需求生成代码：{{requirement}}
```

配合 **Prompt flow**（微软的 LLM 应用开发工具），可以实现 Prompt 变体管理、批量测试、评估对比。

### LangFuse

**LangFuse** 是一个开源的 LLM 工程平台，核心能力包括：

- **Prompt 管理**：Web UI 创建和编辑 Prompt，自动版本管理
- **追踪 Tracing**：记录每次 LLM 调用的完整链路
- **评估**：人工标注 + 自动评分
- ** playground**：Prompt 调试沙箱

```java
// LangFuse Java SDK 集成示例
import com.langfuse.client.Langfuse;

Langfuse langfuse = Langfuse.builder()
        .publicKey(System.getenv("LANGFUSE_PUBLIC_KEY"))
        .secretKey(System.getenv("LANGFUSE_SECRET_KEY"))
        .host("https://cloud.langfuse.com")
        .build();

// 获取最新版本的 Prompt
String prompt = langfuse.getPrompt("java-controller")
        .version("latest")
        .compile(variables);

// 调用 LLM 并自动追踪
langfuse.trace()
        .name("code-generation")
        .input(variables)
        .output(generatedCode)
        .userId("user-123")
        .metadata(Map.of("promptVersion", "v2.1.0"))
        .capture();
```

### PromptLayer

**PromptLayer** 是另一个 Prompt 管理平台，专注在 Prompt 版本控制、A/B 测试和效果追踪。提供了 Python 和 JavaScript SDK。

### 三方案对比

| 维度 | Git + Markdown | YAML 模板引擎 | 专业平台 |
|------|---------------|--------------|---------|
| 上手成本 | 极低 | 低 | 中 |
| 变量替换 | 手动 | 内置 | 内置 |
| 可视化 | 无 | 无 | Web UI |
| A/B 测试 | 自行实现 | 自行实现 | 内置 |
| 效果追踪 | 自行统计 | 自行统计 | 自动 |
| 成本 | 免费 | 免费 | 有免费额度 |
| 适合团队 | 个人/小团队 | 中型团队 | 大型团队 |

---

## 六、Prompt A/B 测试框架（Java 实现）

选好了管理方案，接下来是关键一步：**如何量化 Prompt 改动到底是优化还是劣化？**

答案就是 A/B 测试。

### 6.1 整体架构

```
请求进来 → A/B 分流器 → [Prompt A (对照组)] / [Prompt B (实验组)]
                ↓                          ↓
           LLM 调用                    LLM 调用
                ↓                          ↓
           效果评分                    效果评分
                ↓                          ↓
           统计对比 → 结论（哪个 Prompt 更好）
```

### 6.2 核心代码实现

#### PromptVariant — Prompt 变体定义

```java
import lombok.Builder;
import lombok.Data;

@Data
@Builder
public class PromptVariant {
    /** 变体唯一标识 */
    private String id;
    /** 变体名称：A/B/C */
    private String name;
    /** Prompt 版本号 */
    private String version;
    /** 系统提示词 */
    private String systemPrompt;
    /** 用户提示词模板 */
    private String userPromptTemplate;
    /** 是否为对照组 */
    private boolean isControl;
    /** 流量权重，总和为 100 */
    private int trafficWeight;
}
```

#### AbTestConfig — A/B 测试配置

```java
import lombok.Builder;
import lombok.Data;
import java.time.LocalDateTime;
import java.util.List;

@Data
@Builder
public class AbTestConfig {
    /** 测试 ID */
    private String testId;
    /** 测试名称 */
    private String testName;
    /** Prompt 变体列表 */
    private List<PromptVariant> variants;
    /** 最小样本数，达到后开始评估 */
    private int minSampleSize;
    /** 测试开始时间 */
    private LocalDateTime startTime;
    /** 测试结束时间（可选） */
    private LocalDateTime endTime;
    /** 判断胜出的置信度阈值，默认 0.95 */
    @Builder.Default
    private double confidenceThreshold = 0.95;
}
```

#### PromptAbTestEngine — A/B 测试引擎

```java
import org.springframework.stereotype.Component;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ThreadLocalRandom;

@Component
public class PromptAbTestEngine {

    private final Map<String, AbTestConfig> activeTests = new ConcurrentHashMap<>();
    private final Map<String, List<PromptVariant>> weightedVariantsCache = new ConcurrentHashMap<>();

    /**
     * 注册一个 A/B 测试
     */
    public void registerTest(AbTestConfig config) {
        activeTests.put(config.getTestId(), config);
        buildWeightedList(config);
        log.info("Registered AB test: {} with {} variants",
            config.getTestName(), config.getVariants().size());
    }

    /**
     * 根据测试 ID 进行流量分流，返回被选中的 Prompt 变体
     */
    public PromptVariant selectVariant(String testId) {
        AbTestConfig config = activeTests.get(testId);
        if (config == null) {
            throw new IllegalArgumentException("AB test not found: " + testId);
        }

        List<PromptVariant> weightedList = weightedVariantsCache.get(testId);
        if (weightedList == null || weightedList.isEmpty()) {
            // 回退：返回对照组
            return config.getVariants().stream()
                    .filter(PromptVariant::isControl)
                    .findFirst()
                    .orElseThrow();
        }

        // 带权重的随机分流
        int index = ThreadLocalRandom.current().nextInt(weightedList.size());
        return weightedList.get(index);
    }

    /**
     * 构建带权重的变体列表
     * 例如：A 权重 50，B 权重 50
     * 列表中将有 50 个 A 和 50 个 B，随机取一个实现按比例分流
     */
    private void buildWeightedList(AbTestConfig config) {
        List<PromptVariant> weightedList = new ArrayList<>();
        for (PromptVariant variant : config.getVariants()) {
            for (int i = 0; i < variant.getTrafficWeight(); i++) {
                weightedList.add(variant);
            }
        }
        weightedVariantsCache.put(config.getTestId(), weightedList);
    }

    /**
     * 停止 A/B 测试
     */
    public void stopTest(String testId) {
        activeTests.remove(testId);
        weightedVariantsCache.remove(testId);
        log.info("Stopped AB test: {}", testId);
    }

    /**
     * 获取所有活跃测试
     */
    public Collection<AbTestConfig> getActiveTests() {
        return Collections.unmodifiableCollection(activeTests.values());
    }
}
```

#### EvaluationResult — 单次评估结果

```java
import lombok.Builder;
import lombok.Data;
import java.time.LocalDateTime;
import java.util.Map;

@Data
@Builder
public class EvaluationResult {
    /** 关联的测试 ID */
    private String testId;
    /** 使用的变体 ID */
    private String variantId;
    /** 请求 ID */
    private String requestId;
    /** 生成的代码 */
    private String generatedCode;
    /** 代码是否正确（人工标注或自动判定） */
    private Boolean codeCorrect;
    /** 编译是否通过 */
    private Boolean compilePassed;
    /** 单元测试是否通过 */
    private Boolean testPassed;
    /** 消耗的 Token 数 */
    private Integer tokensUsed;
    /** 响应时间（毫秒） */
    private Long responseTimeMs;
    /** 人工评分（1-5，可选） */
    private Integer humanScore;
    /** 额外指标 */
    private Map<String, Double> customMetrics;
    /** 创建时间 */
    @Builder.Default
    private LocalDateTime createdAt = LocalDateTime.now();
}
```

#### AbTestEvaluator — 效果评估器

```java
import org.springframework.stereotype.Component;
import java.util.*;
import java.util.stream.Collectors;

@Component
public class AbTestEvaluator {

    /**
     * 统计各变体的核心指标
     */
    public Map<String, VariantMetrics> calculateMetrics(
            AbTestConfig config,
            List<EvaluationResult> results) {

        Map<String, List<EvaluationResult>> grouped = results.stream()
                .collect(Collectors.groupingBy(EvaluationResult::getVariantId));

        Map<String, VariantMetrics> metricsMap = new LinkedHashMap<>();

        for (PromptVariant variant : config.getVariants()) {
            List<EvaluationResult> variantResults = grouped.getOrDefault(
                    variant.getId(), Collections.emptyList());

            if (variantResults.isEmpty()) {
                metricsMap.put(variant.getId(), VariantMetrics.empty(variant.getName()));
                continue;
            }

            int total = variantResults.size();
            long correctCount = variantResults.stream().filter(r -> Boolean.TRUE.equals(r.getCodeCorrect())).count();
            long compilePassCount = variantResults.stream().filter(r -> Boolean.TRUE.equals(r.getCompilePassed())).count();
            long testPassCount = variantResults.stream().filter(r -> Boolean.TRUE.equals(r.getTestPassed())).count();

            double avgTokens = variantResults.stream()
                    .mapToInt(EvaluationResult::getTokensUsed)
                    .average().orElse(0);

            double avgResponseTime = variantResults.stream()
                    .mapToLong(EvaluationResult::getResponseTimeMs)
                    .average().orElse(0);

            double avgHumanScore = variantResults.stream()
                    .filter(r -> r.getHumanScore() != null)
                    .mapToInt(EvaluationResult::getHumanScore)
                    .average().orElse(0);

            VariantMetrics metrics = VariantMetrics.builder()
                    .variantName(variant.getName())
                    .variantVersion(variant.getVersion())
                    .sampleSize(total)
                    .codeCorrectRate((double) correctCount / total)
                    .compilePassRate((double) compilePassCount / total)
                    .testPassRate((double) testPassCount / total)
                    .avgTokensUsed(avgTokens)
                    .avgResponseTimeMs(avgResponseTime)
                    .avgHumanScore(avgHumanScore)
                    .build();

            metricsMap.put(variant.getId(), metrics);
        }

        return metricsMap;
    }

    /**
     * 判定哪个变体胜出，使用 Z 检验比较正确率
     * 注意：这里只是一个简化示例，生产环境建议使用更严谨的统计检验
     */
    public AbTestResult evaluate(AbTestConfig config, List<EvaluationResult> results) {
        Map<String, VariantMetrics> metricsMap = calculateMetrics(config, results);

        // 找到对照组
        PromptVariant controlVariant = config.getVariants().stream()
                .filter(PromptVariant::isControl)
                .findFirst()
                .orElse(null);

        if (controlVariant == null) {
            return AbTestResult.builder()
                    .conclusion("无法判定：缺少对照组")
                    .metrics(metricsMap)
                    .build();
        }

        VariantMetrics controlMetrics = metricsMap.get(controlVariant.getId());

        // 样本量检查
        if (controlMetrics.getSampleSize() < config.getMinSampleSize()) {
            return AbTestResult.builder()
                    .conclusion("样本量不足，当前 " + controlMetrics.getSampleSize()
                            + "，需要 " + config.getMinSampleSize())
                    .metrics(metricsMap)
                    .build();
        }

        // 对比每个实验组与对照组
        StringBuilder conclusion = new StringBuilder();
        String winnerVariantName = controlVariant.getName();
        double bestScore = controlMetrics.getOverallScore();

        for (PromptVariant variant : config.getVariants()) {
            if (variant.isControl()) continue;

            VariantMetrics expMetrics = metricsMap.get(variant.getId());
            if (expMetrics == null || expMetrics.getSampleSize() < config.getMinSampleSize()) {
                conclusion.append(String.format("变体 %s 样本量不足；", variant.getName()));
                continue;
            }

            ComparisonResult comparison = compare(controlMetrics, expMetrics, config.getConfidenceThreshold());

            conclusion.append(String.format(
                "【%s vs %s】正确率: %.1f%% vs %.1f%% (%s), 编译通过率: %.1f%% vs %.1f%%, Token: %.0f vs %.0f; ",
                controlVariant.getName(), variant.getName(),
                controlMetrics.getCodeCorrectRate() * 100,
                expMetrics.getCodeCorrectRate() * 100,
                comparison.getCorrectRateVerdict(),
                controlMetrics.getCompilePassRate() * 100,
                expMetrics.getCompilePassRate() * 100,
                controlMetrics.getAvgTokensUsed(),
                expMetrics.getAvgTokensUsed()
            ));

            if (expMetrics.getOverallScore() > bestScore) {
                bestScore = expMetrics.getOverallScore();
                winnerVariantName = variant.getName();
            }
        }

        return AbTestResult.builder()
                .winner(winnerVariantName)
                .conclusion(conclusion.toString())
                .metrics(metricsMap)
                .build();
    }

    /**
     * 简化版对比：比较核心指标
     */
    private ComparisonResult compare(VariantMetrics control, VariantMetrics experiment,
                                      double confidenceThreshold) {
        double diff = experiment.getCodeCorrectRate() - control.getCodeCorrectRate();

        String verdict;
        if (diff > 0.05) {
            verdict = "实验组显著更好 ✓";
        } else if (diff < -0.05) {
            verdict = "实验组显著更差 ✗";
        } else {
            verdict = "差异不显著 ≈";
        }

        return ComparisonResult.builder()
                .correctRateDiff(diff)
                .correctRateVerdict(verdict)
                .build();
    }
}
```

#### 数据模型

```java
import lombok.Builder;
import lombok.Data;

@Data
@Builder
public class VariantMetrics {
    private String variantName;
    private String variantVersion;
    private int sampleSize;
    private double codeCorrectRate;
    private double compilePassRate;
    private double testPassRate;
    private double avgTokensUsed;
    private double avgResponseTimeMs;
    private double avgHumanScore;

    /**
     * 综合评分（可自定义权重）
     */
    public double getOverallScore() {
        return codeCorrectRate * 0.4
             + compilePassRate * 0.2
             + testPassRate * 0.25
             + (avgHumanScore / 5.0) * 0.15;
    }

    public static VariantMetrics empty(String name) {
        return VariantMetrics.builder().variantName(name).build();
    }
}

@Data
@Builder
public class AbTestResult {
    private String winner;
    private String conclusion;
    private Map<String, VariantMetrics> metrics;
}

@Data
@Builder
class ComparisonResult {
    private double correctRateDiff;
    private String correctRateVerdict;
}
```

### 6.3 在 Spring Boot 服务中集成

```java
import org.springframework.stereotype.Service;
import org.springframework.beans.factory.annotation.Autowired;

@Service
public class CodeGenerationService {

    @Autowired
    private PromptAbTestEngine abTestEngine;

    @Autowired
    private AbTestEvaluator abTestEvaluator;

    @Autowired
    private LlmClient llmClient;

    private static final String AB_TEST_ID = "java-controller-prompt-v2-vs-v3";

    /**
     * 生成代码（同时支持 A/B 测试）
     */
    public GenerationResponse generateCode(GenerationRequest request) {
        // 1. A/B 分流
        PromptVariant variant = abTestEngine.selectVariant(AB_TEST_ID);

        // 2. 渲染 Prompt
        RenderedPrompt rendered = renderPrompt(variant, request);

        // 3. 记录开始时间
        long startTime = System.currentTimeMillis();

        // 4. 调用 LLM
        LlmResponse llmResponse = llmClient.chat(
                rendered.getSystemPrompt(),
                rendered.getUserPrompt()
        );

        long responseTime = System.currentTimeMillis() - startTime;

        // 5. 记录评估结果
        EvaluationResult evaluation = EvaluationResult.builder()
                .testId(AB_TEST_ID)
                .variantId(variant.getId())
                .requestId(request.getRequestId())
                .generatedCode(llmResponse.getContent())
                .tokensUsed(llmResponse.getTokensUsed())
                .responseTimeMs(responseTime)
                .build();

        evaluationResultRepository.save(evaluation);

        // 6. 返回
        return GenerationResponse.builder()
                .code(llmResponse.getContent())
                .promptVersion(variant.getVersion())
                .tokensUsed(llmResponse.getTokensUsed())
                .build();
    }
}
```

### 6.4 评估指标说明

| 指标 | 说明 | 自动化程度 |
|------|------|-----------|
| **代码正确率** | 人工判断或由更强的模型判定 | 半自动 |
| **编译通过率** | 直接执行 `javac` 编译 | 全自动 |
| **测试通过率** | 运行生成的测试或已有测试 | 全自动 |
| **Token 消耗** | 从 LLM 响应中获取 | 全自动 |
| **响应时间** | 客户端计时 | 全自动 |
| **人工评分** | 团队成员打分 1-5 | 人工 |

---

## 七、A/B 测试结果仪表盘

为了方便团队查看，我们可以在日志中输出对比结果，也可以用简单的控制台表格打印：

```java
@Component
public class AbTestDashboard {

    @Autowired
    private AbTestEvaluator evaluator;

    @Autowired
    private EvaluationResultRepository repository;

    /**
     * 打印 A/B 测试结果仪表盘
     */
    public void printDashboard(String testId) {
        AbTestConfig config = /* 从注册表获取 */;
        List<EvaluationResult> results = repository.findByTestId(testId);
        AbTestResult abResult = evaluator.evaluate(config, results);

        System.out.println("\n" + "=".repeat(70));
        System.out.println("   Prompt A/B 测试报告 — " + config.getTestName());
        System.out.println("=".repeat(70));
        System.out.printf("%-12s %-12s %-10s %-12s %-12s %-10s %-10s%n",
                "变体", "版本", "样本量", "正确率", "编译通过率", "Token", "耗时(ms)");
        System.out.println("-".repeat(70));

        for (Map.Entry<String, VariantMetrics> entry : abResult.getMetrics().entrySet()) {
            VariantMetrics m = entry.getValue();
            System.out.printf("%-12s %-12s %-10d %-12.1f%% %-13.1f%% %-10.0f %-10.0f%n",
                    m.getVariantName(),
                    m.getVariantVersion(),
                    m.getSampleSize(),
                    m.getCodeCorrectRate() * 100,
                    m.getCompilePassRate() * 100,
                    m.getAvgTokensUsed(),
                    m.getAvgResponseTimeMs()
            );
        }

        System.out.println("-".repeat(70));
        System.out.println("结论: " + abResult.getConclusion());
        if (abResult.getWinner() != null) {
            System.out.println("胜出: " + abResult.getWinner() + " ✓");
        }
        System.out.println("=".repeat(70) + "\n");
    }
}
```

运行效果示例：

```
======================================================================
   Prompt A/B 测试报告 — Java Controller Prompt 优化实验
======================================================================
变体         版本         样本量    正确率      编译通过率     Token      耗时(ms)
----------------------------------------------------------------------
A(v2.1)     v2.1.0      127       87.0%       94.3%         2450       3200
B(v2.2)     v2.2.0      131       89.8%       96.1%         2520       3150
----------------------------------------------------------------------
结论: 【A vs B】正确率: 87.0% vs 89.8% (实验组显著更好 ✓), ...
胜出: B ✓
======================================================================
```

---

## 八、Prompt 版本 Tag 命名规范

借鉴**语义化版本（SemVer）**，为 Prompt 版本建立命名规范：

```
v<major>.<minor>.<patch>
```

| 版本号 | 变更类型 | 示例 |
|--------|---------|------|
| **Major** | 不兼容的重写，输出格式或风格根本性改变 | v1.0.0 → v2.0.0：从生成 REST 切换到生成 GraphQL |
| **Minor** | 功能增强，向后兼容，增加新规则或场景覆盖 | v1.0.0 → v1.1.0：新增事务处理提示 |
| **Patch** | 修复，措辞优化，Bug 修复 | v1.0.0 → v1.0.1：修复变量名歧义 |

### Git Tag 示例

```bash
# 为 Prompt 打上语义化版本 Tag
git tag -a "prompt/java-controller/v2.1.0" -m "Java Controller Prompt v2.1.0

## Changes
- 增加 JWT Token 生成逻辑提示
- 补充事务处理场景说明
- 优化错误处理提示

## A/B Test Result
- 代码正确率: 87.0% → 89.8% (+2.8%)
- 编译通过率: 94.3% → 96.1% (+1.8%)"

# 推送 Tag
git push origin "prompt/java-controller/v2.1.0"

# 查看所有 Prompt Tag
git tag -l "prompt/*"
```

---

## 九、团队 Prompt 协作流程

将 Prompt 纳入团队的标准研发流程：

```
需求分析 → Prompt 设计 → 分支开发 → A/B 测试 → PR Review → 合并 → 发布 → 监控
```

### 9.1 完整协作流程

```
┌─────────────────────────────────────────────────────────────────┐
│  1. 创建 Prompt Issue/Branch                                     │
│     feat/prompt-optimize-java-service                            │
├─────────────────────────────────────────────────────────────────┤
│  2. 本地开发和自测                                               │
│     - 修改 .prompt.md 或 .prompt.yaml                            │
│     - 在本地 playground 测试 10+ 条 case                         │
│     - 更新 version_history                                       │
├─────────────────────────────────────────────────────────────────┤
│  3. 发起 A/B 测试（可选但推荐）                                   │
│     - 注册 ABTestConfig                                          │
│     - 设置对照组(旧)和实验组(新)                                  │
│     - 运行至满足最小样本量                                        │
│     - 打印仪表盘，确认实验组胜出                                  │
├─────────────────────────────────────────────────────────────────┤
│  4. 提交 Prompt PR                                               │
│     - PR 模板包含：变更说明、测试结果、AB 测试数据               │
│     - @相关同事 Review                                           │
├─────────────────────────────────────────────────────────────────┤
│  5. Code/Prompt Review                                           │
│     - 检查措辞是否清晰                                            │
│     - 检查边界场景是否覆盖                                        │
│     - 检查 token 消耗是否合理                                     │
│     - 审批通过                                                    │
├─────────────────────────────────────────────────────────────────┤
│  6. 合并并打 Tag                                                 │
│     - 合并到 main 分支                                           │
│     - git tag prompt/java-controller/v2.2.0                      │
├─────────────────────────────────────────────────────────────────┤
│  7. 发布上线                                                     │
│     - 部署新 Prompt 到生产                                        │
│     - 灰度发布或直接全量                                          │
├─────────────────────────────────────────────────────────────────┤
│  8. 监控和观察                                                   │
│     - 关注代码正确率是否下降                                      │
│     - 关注异常/错误日志                                           │
│     - 关注 Token 消耗变化                                         │
│     - 如有问题立即回滚到上一个版本                                │
└─────────────────────────────────────────────────────────────────┘
```

### 9.2 Prompt PR 模板

```markdown
## Prompt 变更说明

**Prompt 名称**: java-controller
**当前版本**: v2.1.0
**目标版本**: v2.2.0
**变更类型**: Minor（功能增强）

## 变更内容

- [ ] 增加事务处理提示：要求涉及多表操作的接口添加 @Transactional
- [ ] 补充参数校验规则：String 类型参数增加 @NotBlank
- [ ] 优化异常处理措辞

## A/B 测试结果

| 指标 | v2.1.0 (A) | v2.2.0 (B) | 变化 |
|------|-----------|-----------|------|
| 样本量 | 127 | 131 | - |
| 代码正确率 | 87.0% | 89.8% | +2.8% |
| 编译通过率 | 94.3% | 96.1% | +1.8% |
| 平均 Token | 2450 | 2520 | +2.9% |

**结论**: B 组正确率显著提升，Token 消耗微增可接受，建议合并。

## Review 重点

- [ ] 事务处理提示是否准确
- [ ] 是否有遗漏的场景
- [ ] Token 消耗增加是否可接受
```

---

## 十、最佳实践总结

1. **永远保留旧版本** — Git 历史是最廉价的保险
2. **改 Prompt 必做 A/B 测试** — 数据驱动，而非感觉驱动
3. **语义化版本** — 一眼看懂变更的影响范围
4. **Prompt 也要 Code Review** — 多人 Review 能发现盲区
5. **监控线上指标** — 发布不等于结束，持续观察效果
6. **建立 Prompt 库** — 沉淀团队最佳实践，避免重复踩坑
7. **Few-shot Examples 也纳入版本管理** — 示例的质量直接影响输出

---

## 十一、总结

Prompt Engineering 已经从"玄学调参"进入"工程化时代"。本文介绍的三种管理方案可以按团队规模逐步演进：

- **个人项目**：Git + Markdown 足矣
- **小团队**：YAML 模板引擎 + Git Tag 版本管理
- **大团队/产品化**：LangFuse 等专业平台 + 完整 A/B 测试 + 监控告警

核心原则只有一个：**像管理代码一样管理你的 Prompt。**

下次再有人问你"这个 Prompt 改了效果到底好不好？"，你可以自信地打开 A/B 测试仪表盘，用数据说话。

---

## 下一篇预告

**《20 个高频 Prompt 模板》**

整理了 Java 开发中最常用的 20 个 Prompt 模板，涵盖代码生成、代码审查、单元测试、SQL 优化、异常处理、API 文档等场景。拿来即用，复制粘贴就能提效。

敬请期待！

---

> 如果这篇文章对你有帮助，欢迎点赞、收藏、转发。你的支持是我持续更新的动力。

---

*本文于 2026-05-05 发布*
*作者：一个致力于用工程化思维做 Prompt Engineering 的 Java 程序员*
