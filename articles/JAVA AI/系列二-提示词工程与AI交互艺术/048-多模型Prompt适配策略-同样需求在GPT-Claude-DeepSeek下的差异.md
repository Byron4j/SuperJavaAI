# 多模型 Prompt 适配策略：同样需求在 GPT-4 / Claude / DeepSeek 下的差异

> 同一个 Prompt 换模型就"水土不服"？GPT-4、Claude、DeepSeek、Gemini 四大模型对 Prompt 的理解偏好截然不同。本文带你摸清每个模型的"脾气"，并给出多模型路由的 Java 实战方案。

---

## 一、开篇实验：一个 Prompt，四个答案

我设计了一个最简单的实验——用完全相同的 Prompt，让四个主流模型生成同一个 Spring Boot 接口：

**实验 Prompt：**
```
写一个Spring Boot的UserController，实现用户注册接口。
要求：参数校验、密码加密存储、返回JWT Token。
```

**GPT-4o 的生成结果：**
```java
@RestController
@RequestMapping("/api/users")
@RequiredArgsConstructor
public class UserController {

    private final UserService userService;

    @PostMapping("/register")
    public ResponseEntity<ApiResponse<LoginResponse>> register(
            @Valid @RequestBody RegisterRequest request) {

        LoginResponse response = userService.register(request);
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(ApiResponse.success(response));
    }
}
// 同时生成了完整的 RegisterRequest DTO、UserService 接口+实现、
// LoginResponse、ApiResponse 统一返回体、全局异常处理、JwtUtil 工具类
// 完美三层架构，包含 Swagger 注解
```

**Claude 3.5 Sonnet 的生成结果：**
```java
@RestController
@RequestMapping("/api/v1/users")
@RequiredArgsConstructor
@Slf4j
@CrossOrigin(origins = "${app.cors.allowed-origins}")
public class UserController {

    private final UserService userService;
    private final RateLimiter rateLimiter;
    private final AuditService auditService;

    @PostMapping("/register")
    @RateLimit(key = "register", permitsPerSecond = 5)
    public ResponseEntity<Result<LoginResponse>> register(
            @Valid @RequestBody RegisterRequest request,
            HttpServletRequest httpRequest) {

        log.info("User registration attempt: {}", request.maskSensitiveData());
        String clientIp = httpRequest.getRemoteAddr();

        // 做了很多额外的安全检查和审计记录
        // 引入了限流、CORS配置化、敏感数据脱敏等过度设计
    }
}
```

**DeepSeek-V3 的生成结果：**
```java
@RestController
@RequestMapping("/user")
public class UserController {

    @Autowired
    private UserService userService;

    @PostMapping("/register")
    public Result register(@RequestBody RegisterRequest request) {
        if (request.getUsername() == null || request.getUsername().isEmpty()) {
            return Result.error("用户名不能为空");
        }
        // 使用 @Autowired 而非构造器注入
        // 手动校验而非 @Valid 注解
        // 缺少密码长度/复杂度校验
        // 没有全局异常处理
    }
}
```

**Gemini 1.5 Pro 的生成结果：**
```java
@RestController
@RequestMapping("/api/user")
public class UserController extends BaseController {

    @Inject
    private UserService userService;

    @PostMapping("/register")
    public Mono<ServerResponse> register(ServerRequest request) {
        // 使用了 WebFlux 响应式（而没被要求）
        // 用了 @Inject 而非 @Autowired
        // 继承了自定义 BaseController
        // 风格独特，不太遵循常规 Spring Boot 惯例
    }
}
```

**四个模型的差异一目了然：**

| 维度 | GPT-4o | Claude 3.5 | DeepSeek-V3 | Gemini 1.5 Pro |
|------|:------:|:--------:|:---------:|:-----------:|
| 架构完整性 | 三层架构完备 | 过度设计（限流/审计） | 简单但有遗漏 | 风格飘逸 |
| 代码规范性 | 标准 Spring Boot | 过于工程化 | 偏向传统写法 | 引入非标准写法 |
| 防御性编程 | 适度 | 过度 | 不足 | 中等但偏门 |
| 中文理解 | 良好 | 良好 | **最优** | 一般 |
| 指令遵循度 | **最高** | 中高（自己加戏） | 中（会打折） | 低（发挥过多） |

---

## 二、四大模型的"理解偏好"深度分析

### 2.1 GPT-4o：最"听话"的模范生

**核心特征：** 严格遵循指令，结构化输出，缺少意外惊喜但也很少犯错。

**Prompt 适配策略：**

```
适合的 Prompt 风格：
- 精确的、逐条列出的约束
- JSON 格式的配置描述
- 明确的 DO/DON'T 列表
- 英文 Prompt 效果好（Token效率高）

不适合的 Prompt 风格：
- "你觉得怎么好就怎么来"
- 模糊的开放式问题
- 需要"创意发挥"的场景
```

**针对 GPT-4o 优化的 Prompt 示例：**

```java
/**
 * GPT-4o 优化版 Prompt 模板
 * 特点：结构化、精确、约束明确
 */
public class Gpt4oPromptTemplate {

    public static String buildPrompt(String task, List<String> constraints, String outputFormat) {
        return String.format("""
            Task: %s

            Constraints (MUST follow exactly):
            %s

            Output format:
            %s

            DO:
            - Follow standard Java conventions (Google Java Style)
            - Use constructor injection (@RequiredArgsConstructor)
            - Include input validation annotations
            - Generate complete, compilable code

            DON'T:
            - Add features not listed in constraints
            - Use deprecated APIs
            - Skip error handling
            """,
            task,
            constraints.stream().map(c -> "- " + c).collect(Collectors.joining("\n")),
            outputFormat
        );
    }
}
```

### 2.2 Claude 3.5 Sonnet：最"贴心"的完美主义者

**核心特征：** 会主动补充你可能遗漏的设计细节，代码质量极高但容易"过度工程化"。

**Prompt 适配策略：**

```
适合的 Prompt 风格：
- "简洁描述需求 + DON'T列表"组合
- 需要判断力的任务（代码审查、重构建议、架构设计）
- 长篇上下文任务（200K Token窗口）

不适合的 Prompt 风格：
- 不加限制的开放式需求
- 过于简短的指令（会自己脑补）
```

**针对 Claude 优化的 Prompt 模板：**

```java
/**
 * Claude 优化版 Prompt 模板
 * 特点：明确边界 + DON'T 列表 + 简洁指令
 */
public class ClaudePromptTemplate {

    public static String buildPrompt(String task, List<String> dontDoList) {
        return String.format("""
            %s

            Important constraints:
            %s

            Keep it simple. No extra features unless I specifically ask.
            No rate limiting, no audit logs, no AOP unless required.
            Standard Spring Boot conventions only.
            """,
            task,
            dontDoList.stream().map(d -> "DO NOT: " + d).collect(Collectors.joining("\n"))
        );
    }
}
```

**Claude 的"过度设计"典型表现：**

| 需求 | 期望输出 | Claude 实际输出（过度） |
|------|---------|-------------------|
| 注册接口 | @Valid 校验 | 加限流 + 审计 + IP黑名单 |
| 分页查询 | PageHelper | 加缓存 + 布隆过滤器 |
| 登录功能 | JWT Token | 加刷新Token + 设备管理 + 登录历史 |
| 实体类 | @Data + JPA注解 | 加Builder模式 + 值对象封装 |

**解决方案：** 在 Claude 的 Prompt 中加一句 `No over-engineering. Generate exactly what is requested, nothing more.`，可以减少 80% 的冗余设计。

### 2.3 DeepSeek-V3：最"直接"的务实派

**核心特征：** 代码简洁实用，中文理解能力最强，但防御性编程不足（缺少校验、异常处理）。

**Prompt 适配策略：**

```
适合的 Prompt 风格：
- 中文 Prompt（效果显著优于英文）
- 代码示例驱动（Few-shot 效果好）
- 需要明确写出"别忘了加XXX"

不适合的 Prompt 风格：
- 英文 Prompt（Token效率高但效果打折扣）
- 假设"模型会自动补充校验逻辑"
```

**针对 DeepSeek 优化的 Prompt 模板：**

```java
/**
 * DeepSeek 优化版 Prompt 模板
 * 特点：中文 + 代码示例 + 强制约束
 */
public class DeepSeekPromptTemplate {

    public static String buildPrompt(String task, String example) {
        return String.format("""
            %s

            必须遵守以下规范（非常重要，一项都不能少）：
            1. 所有入参必须使用 @Valid + JSR303 注解校验
            2. 所有可能为null的返回值必须用 Optional 包装
            3. 异常统一通过 @RestControllerAdvice 处理
            4. 使用构造器注入，不要用 @Autowired 字段注入
            5. 每个公开方法必须添加 JavaDoc 注释
            6. 密码不得明文存储，必须BCrypt加密
            7. 分页查询统一使用 MyBatis-Plus 的 Page 对象

            参考代码风格：
            ```java
            %s
            ```
            请严格参考上述风格的代码规范，生成完整可运行的代码。
            """,
            task, example
        );
    }
}
```

**DeepSeek 特有的强项和短板：**

| 场景 | DeepSeek 表现 | 建议 |
|------|:---------:|------|
| 中文文档生成 | 最优 | 直接用 |
| 代码注释 | 中文注释质量高 | 直接用 |
| 简单CRUD | 代码简洁能跑 | 需补充校验 |
| 复杂架构设计 | 偏简单 | 加更多架构约束 |
| SQL生成 | 质量高 | 直接用 |
| 英文Prompt | 效果打8折 | 宁可用中文 |

### 2.4 Gemini 1.5 Pro：最"发散"的艺术家

**核心特征：** 创意性强，会引入非标准写法（WebFlux、Kotlin协程风格、函数式编程偏好），代码独特性高但可维护性存疑。

**Prompt 适配策略：**

```
适合的 Prompt 风格：
- 严格的框架/库约束（"必须用XXX"）
- 具体的代码风格示例
- 限定版本号（"Spring Boot 3.2, Java 17"）

不适合的 Prompt 风格：
- "用你认为最好的方式"
- 不给具体的技术栈限制
```

**针对 Gemini 优化的 Prompt 模板：**

```java
/**
 * Gemini 优化版 Prompt 模板
 * 特点：技术栈锁定 + 禁止发散 + 代码示例约束
 */
public class GeminiPromptTemplate {

    public static String buildPrompt(String task, String techStack, String codeStyle) {
        return String.format("""
            Task: %s

            SPECIFIC TECHNOLOGY STACK (use ONLY these, no alternatives):
            %s

            CODE STYLE REFERENCE (follow this exact style):
            ```java
            %s
            ```

            CONSTRAINTS:
            - Use Spring MVC (@RestController), NOT Spring WebFlux
            - Constructor injection ONLY (Lombok @RequiredArgsConstructor)
            - Standard Java patterns, no functional/reactive unless specified
            - Do NOT introduce new libraries or frameworks
            - Follow the EXACT code style shown above
            """,
            task, techStack, codeStyle
        );
    }
}
```

---

## 三、多模型自适应 Prompt 引擎的 Java 实现

### 3.1 模型路由策略

```java
import java.util.*;
import java.util.function.Function;

/**
 * 模型路由器：根据任务复杂度和要求自动选择模型和优化Prompt
 */
public class ModelRouter {

    public enum Model {
        GPT_4O("gpt-4o", 0.0025, 0.0100, false),
        GPT_4O_MINI("gpt-4o-mini", 0.00015, 0.00060, false),
        CLAUDE_SONNET("claude-3.5-sonnet", 0.0030, 0.0150, true),
        CLAUDE_HAIKU("claude-3-haiku", 0.00025, 0.00125, true),
        DEEPSEEK_V3("deepseek-v3", 0.00027, 0.00110, false),
        GEMINI_PRO("gemini-1.5-pro", 0.0035, 0.0105, false);

        public final String modelId;
        public final double inputPricePer1k;
        public final double outputPricePer1k;
        public final boolean tendsToOverEngineer;

        Model(String modelId, double inputPrice, double outputPrice, boolean overEngineer) {
            this.modelId = modelId;
            this.inputPricePer1k = inputPrice;
            this.outputPricePer1k = outputPrice;
            this.tendsToOverEngineer = overEngineer;
        }
    }

    public enum TaskComplexity {
        SIMPLE,    // 简单CRUD、单文件生成
        MODERATE,  // 多文件、带业务逻辑
        COMPLEX,   // 架构设计、复杂业务
        REVIEW     // 代码审查、质量评估
    }

    public enum TaskDomain {
        CODE_GENERATION,
        CODE_REVIEW,
        BUG_FIX,
        ARCHITECTURE_DESIGN,
        DOCUMENTATION,
        SQL_GENERATION
    }

    /**
     * 路由决策：根据任务特征选择最优模型
     */
    public static RouteDecision route(TaskComplexity complexity, TaskDomain domain,
                                       boolean needChineseOptimization, int contextSize) {

        return switch (complexity) {
            case SIMPLE -> switch (domain) {
                case CODE_GENERATION -> contextSize < 4000
                        ? new RouteDecision(Model.CLAUDE_HAIKU, "低复杂度代码生成用Claude Haiku最经济")
                        : new RouteDecision(Model.GPT_4O_MINI, "长上下文用GPT-4o-mini");
                case DOCUMENTATION -> new RouteDecision(
                        needChineseOptimization ? Model.DEEPSEEK_V3 : Model.GPT_4O_MINI,
                        "文档生成" + (needChineseOptimization ? "用DeepSeek中文最佳" : "用GPT-4o-mini"));
                default -> new RouteDecision(Model.CLAUDE_HAIKU, "简单任务默认用Haiku");
            };

            case MODERATE -> switch (domain) {
                case CODE_GENERATION -> new RouteDecision(Model.DEEPSEEK_V3,
                        "中等复杂度用DeepSeek V3性价比最高");
                case SQL_GENERATION -> new RouteDecision(Model.DEEPSEEK_V3,
                        "SQL生成DeepSeek表现优秀");
                case BUG_FIX -> new RouteDecision(Model.CLAUDE_SONNET,
                        "Bug修复需要Claude的分析能力");
                case CODE_REVIEW -> new RouteDecision(Model.GPT_4O,
                        "代码审查用GPT-4o最可靠");
                default -> new RouteDecision(Model.GPT_4O,
                        "中等复杂度默认用GPT-4o");
            };

            case COMPLEX -> switch (domain) {
                case ARCHITECTURE_DESIGN -> new RouteDecision(Model.CLAUDE_SONNET,
                        "架构设计Claude最具洞察力");
                case CODE_REVIEW -> new RouteDecision(Model.CLAUDE_SONNET,
                        "复杂审查用Claude");
                default -> new RouteDecision(Model.CLAUDE_SONNET,
                        "复杂任务用Claude Sonnet综合最优");
            };

            case REVIEW -> new RouteDecision(Model.GPT_4O,
                    "审查评估用GPT-4o最客观");
        };
    }

    /**
     * 根据路由结果优化 Prompt
     */
    public static String optimizePrompt(String rawPrompt, RouteDecision decision) {
        Model model = decision.model;

        return switch (model) {
            case GPT_4O, GPT_4O_MINI -> Gpt4oPromptTemplate.buildPrompt(
                    rawPrompt,
                    List.of("Follow exactly the requirements", "Include proper validation",
                            "Use standard Spring Boot patterns"),
                    "Java code with standard annotations"
            );

            case CLAUDE_SONNET, CLAUDE_HAIKU -> {
                String prompt = ClaudePromptTemplate.buildPrompt(rawPrompt, List.of(
                        "add rate limiting or auditing",
                        "over-engineer with unnecessary patterns",
                        "add features not explicitly requested",
                        "use reactive programming unless specified"
                ));
                yield prompt;
            }

            case DEEPSEEK_V3 -> DeepSeekPromptTemplate.buildPrompt(
                    rawPrompt,
                    "@RequiredArgsConstructor\npublic class ExampleService {\n" +
                    "    private final XxxMapper mapper;\n" +
                    "    @Transactional\n    public Result<XxxVO> method(@Valid XxxDTO dto) { ... }\n}"
            );

            case GEMINI_PRO -> GeminiPromptTemplate.buildPrompt(
                    rawPrompt,
                    "Spring Boot 3.2 + JDK 17 + Spring MVC + MyBatis-Plus 3.5 + Lombok",
                    "@RestController\n@RequestMapping(\"/api/xxx\")\n" +
                    "@RequiredArgsConstructor\npublic class XxxController { ... }"
            );
        };
    }

    public record RouteDecision(Model model, String reason) {}

    // ---- 使用示例 ----
    public static void main(String[] args) {
        String rawPrompt = "写一个订单管理模块，支持创建订单、查询订单列表（分页+状态筛选）、取消订单";

        RouteDecision decision = route(
                TaskComplexity.MODERATE,
                TaskDomain.CODE_GENERATION,
                true,  // 需要中文优化
                2000   // 上下文约2000 Token
        );

        System.out.println("选择模型: " + decision.model.modelId);
        System.out.println("决策理由: " + decision.reason);

        String optimizedPrompt = optimizePrompt(rawPrompt, decision);
        System.out.println("优化后的Prompt:\n" + optimizedPrompt);
    }
}
```

**运行结果：**
```
选择模型: deepseek-v3
决策理由: 中等复杂度用DeepSeek V3性价比最高
优化后的Prompt:
写一个订单管理模块，支持创建订单、查询订单列表（分页+状态筛选）、取消订单

必须遵守以下规范（非常重要，一项都不能少）：
1. 所有入参必须使用 @Valid + JSR303 注解校验
2. 所有可能为null的返回值必须用 Optional 包装
...
```

### 3.2 多模型 Fallback 机制

```java
/**
 * 多模型 Failover 机制
 * 主模型失败/超时/质量不达标时自动切换备用模型
 */
public class ModelFailoverExecutor {

    private final List<ModelEndpoint> endpoints;
    private final QualityGate qualityGate;

    public record ModelEndpoint(Model model, String name, Function<String, String> caller) {}
    public record QualityGate(double minCompileRate, int maxResponseTimeMs) {}

    public ModelFailoverExecutor(List<ModelEndpoint> endpoints, QualityGate qualityGate) {
        this.endpoints = new ArrayList<>(endpoints);
        this.qualityGate = qualityGate;
    }

    /**
     * 执行多模型调用，支持 fallback
     */
    public GenerationResult executeWithFailover(String prompt, int maxRetries) {
        List<AttemptResult> attempts = new ArrayList<>();

        for (int i = 0; i < Math.min(maxRetries, endpoints.size()); i++) {
            ModelEndpoint endpoint = endpoints.get(i);
            long startTime = System.currentTimeMillis();

            try {
                String response = endpoint.caller.apply(prompt);
                long elapsed = System.currentTimeMillis() - startTime;

                // 质量检查
                QualityScore score = evaluateQuality(response, elapsed);

                attempts.add(new AttemptResult(endpoint.name, score, elapsed));

                if (score.overallScore >= 0.7 && elapsed <= qualityGate.maxResponseTimeMs) {
                    return new GenerationResult(response, endpoint.name, attempts, true);
                }

                System.err.println("[" + endpoint.name + "] 质量不达标(评分:" +
                        String.format("%.2f", score.overallScore) + "), 切换到备用模型...");

            } catch (Exception e) {
                long elapsed = System.currentTimeMillis() - startTime;
                attempts.add(new AttemptResult(endpoint.name, null, elapsed));
                System.err.println("[" + endpoint.name + "] 调用失败: " + e.getMessage());
            }
        }

        // 所有模型都失败，返回最后一个非null结果
        String fallback = findLastNonNullResult(attempts, endpoints);
        return new GenerationResult(fallback, "fallback", attempts, false);
    }

    /**
     * 评估代码质量
     */
    private QualityScore evaluateQuality(String code, long responseTimeMs) {
        double compileScore = checkBasicSyntax(code) ? 1.0 : 0.3;
        double structureScore = checkHasAnnotations(code) ? 1.0 : 0.6;
        double completenessScore = checkCompleteness(code);
        double performanceScore = responseTimeMs < 10000 ? 1.0 : 0.5;

        double overall = compileScore * 0.4 + structureScore * 0.2
                + completenessScore * 0.2 + performanceScore * 0.2;

        return new QualityScore(compileScore, structureScore,
                completenessScore, performanceScore, overall);
    }

    private boolean checkBasicSyntax(String code) {
        // 简化检查：匹配基本的类定义
        return code != null && code.contains("class ") && code.contains("{") && code.contains("}");
    }

    private boolean checkHasAnnotations(String code) {
        return code != null && (code.contains("@Service") || code.contains("@RestController")
                || code.contains("@Component") || code.contains("@Repository"));
    }

    private double checkCompleteness(String code) {
        if (code == null) return 0;
        double score = 0.5;
        if (code.contains("import ")) score += 0.1;
        if (code.contains("return ")) score += 0.1;
        if (code.contains("@Override")) score += 0.1;
        if (code.contains("public ")) score += 0.1;
        if (code.contains("private ")) score += 0.1;
        return Math.min(score, 1.0);
    }

    private String findLastNonNullResult(List<AttemptResult> attempts, List<ModelEndpoint> endpoints) {
        for (int i = attempts.size() - 1; i >= 0; i--) {
            AttemptResult ar = attempts.get(i);
            if (ar.score != null) {
                return endpoints.get(i).caller.apply(""); // 重新调用
            }
        }
        return "// All models failed to generate code";
    }

    public record AttemptResult(String modelName, QualityScore score, long elapsedMs) {}
    public record QualityScore(double compile, double structure, double completeness,
                                double performance, double overallScore) {}
    public record GenerationResult(String code, String sourceModel,
                                    List<AttemptResult> attempts, boolean primarySuccess) {}
}
```

---

## 四、成本优化：模型选择决策树

```java
/**
 * 智能模型选择决策树
 */
public class ModelSelectionDecisionTree {

    public static Model selectModel(GenerationRequest request) {

        // 决策1：需要超长上下文？(>128K)
        if (request.contextTokens > 128000) {
            return request.needsEnglishOutput ? Model.CLAUDE_SONNET : Model.GEMINI_PRO;
        }

        // 决策2：任务是否需要中文深度理解？
        if (request.needsChineseNuance) {
            return Model.DEEPSEEK_V3;
        }

        // 决策3：任务是否简单且对延迟敏感？
        if (request.complexity == TaskComplexity.SIMPLE && request.maxLatencyMs < 3000) {
            return Model.GPT_4O_MINI;
        }

        // 决策4：是否需要"过度工程化"的设计建议？
        if (request.domain == TaskDomain.ARCHITECTURE_DESIGN && request.wantThoroughDesign) {
            return Model.CLAUDE_SONNET; // Claude 会多想一步
        }

        // 决策5：是否需要精确的指令执行？
        if (request.requiresStrictFollowing) {
            return Model.GPT_4O;
        }

        // 默认：性价比之选
        return request.contextTokens > 20000 ? Model.CLAUDE_SONNET : Model.DEEPSEEK_V3;
    }

    public static class GenerationRequest {
        public int contextTokens;
        public boolean needsChineseNuance;
        public TaskComplexity complexity;
        public int maxLatencyMs = 30000;
        public TaskDomain domain;
        public boolean wantThoroughDesign;
        public boolean requiresStrictFollowing;
        public boolean needsEnglishOutput;
    }
}
```

---

## 五、实战总结：模型选择速查表

| 场景 | 推荐模型 | 备选模型 | Prompt 策略 |
|------|:------:|:------:|------|
| 简单CRUD生成 | Claude Haiku | GPT-4o-mini | 简洁 + DON'T 列表 |
| 复杂业务代码 | Claude Sonnet | GPT-4o | 精确约束 + 架构说明 |
| 代码审查 | GPT-4o | Claude Sonnet | 结构化评估要点 |
| 中文文档生成 | DeepSeek-V3 | Qwen-Max | 中文直接描述 |
| Bug 修复 | Claude Sonnet | GPT-4o | 提供完整堆栈信息 |
| 架构设计 | Claude Sonnet | GPT-4o | 开放性问题让Claude发挥 |
| SQL 生成 | DeepSeek-V3 | GPT-4o | 提供表结构DDL |
| 单元测试 | GPT-4o | Claude Haiku | 精确指定覆盖场景 |
| API 文档 | GPT-4o | DeepSeek-V3 | 结构化输出格式 |
| 性能优化 | Claude Sonnet | GPT-4o | 提供JVM/数据库指标 |

**核心原则：**
- GPT-4o：精确任务，严格按指令执行
- Claude：需要判断力和综合分析的任务  
- DeepSeek-V3：中文场景、SQL、性价比优先
- Gemini：超长上下文、需要发散思维的场景

---

**下一篇预告：** 《Prompt 模板引擎：使用 Jinja2/Mustache 动态生成 Prompt》—— 一行 Java 代码生成不同场景的 Prompt，告别复制粘贴！我们将构建一个企业级 Prompt 模板引擎，支持模板继承、参数驱动和团队模板库管理。敬请期待！
