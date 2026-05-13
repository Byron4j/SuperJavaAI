# 设计一个 AI-Native 的权限系统：Attribute-Based Access Control，AI 调用的最低权限原则

## 开场白：月均 $30000 的 OpenAI 账单是怎么来的？

某创业公司的后台管理系统接入了 GPT-4。三个月后，CTO 收到账单：$32000。排查发现，一个测试账号通过前端界面反复调用 GPT-4 生成了 50 万条"hello world"回复，API Key 没有做任何权限限制。

更可怕的是：一个实习生把包含客户 PII 数据的 Excel 文件粘贴到 ChatGPT 中生成分析报告——15000 名客户的身份信息、信用卡后四位、家庭住址全部暴露给了 OpenAI。

这不是虚构。这是 AI 时代每天都在发生的安全事故。

传统 RBAC（基于角色的访问控制）无法应对 AI 调用场景的复杂性。本文带你设计一个 **AI-Native 的权限系统**，基于 ABAC（基于属性的访问控制）模型，实现 AI 调用的精确控制。

## 一、为什么 RBAC 不够用了？

### 1.1 RBAC 的典型设计

```
用户 → 角色(Role) → 权限(Permission)

例如：
用户张三 → ROLE_ADMIN → 可以调用 GPT-4
用户李四 → ROLE_USER  → 可以调用 GPT-3.5
```

这种设计的致命缺陷：

```
┌────────────────────────────────────────────────────┐
│  RBAC 在 AI 场景下的 "五不" 原则                    │
├────────────────────────────────────────────────────┤
│  1. 不能限制调用次数      ADMIN角色无限调用         │
│  2. 不能限制 Token 消耗  一次调用可花费$10          │
│  3. 不能限制数据内容     什么数据都能发给AI         │
│  4. 不能限制模型使用      ADMIN可以用任何模型       │
│  5. 不能限制时间窗口     凌晨3点也可以大量调用      │
└────────────────────────────────────────────────────┘
```

### 1.2 AI 调用的复杂性

AI 调用权限需要考虑的维度远超传统 CRUD：

```
          ┌──── 用户身份（谁？）
          │
          │     ┌──── 资源/模型（用哪个模型？）
          │     │
          │     │     ┌──── 操作/目的（干什么？）
          │     │     │
          ▼     ▼     ▼
   ┌─────────────────────────┐
   │    AI 调用权限决策       │
   └────▲────▲────▲──────────┘
               │    │
               │    └──── 环境上下文（什么时候？在哪里？）
               │
               └──── 数据敏感度（传了什么数据？）
```

## 二、ABAC 核心模型

### 2.1 ABAC 四大要素

```
ABAC 决策函数：
f(Subject, Resource, Action, Environment) → {PERMIT, DENY}

Subject（主体）:     用户、应用、服务账号
Resource（资源）:    模型、API、数据
Action（操作）:      调用、训练、微调、导出
Environment（环境）: 时间、IP、设备、风险评分
```

### 2.2 完整的 AI 调用策略模型

```java
/**
 * ABAC 策略评估核心
 */
public record AIPolicyContext(
    // Subject 主体属性
    String userId,
    String userDepartment,
    String userRole,
    int userSecurityLevel,      // 安全等级 1-5

    // Resource 资源属性
    String aiModel,              // gpt-4 / gpt-3.5-turbo / claude-3
    String modelTier,            // premium / standard / basic
    boolean allowsTraining,      // 是否允许用于训练

    // Action 操作属性
    String action,               // COMPLETION / EMBEDDING / FINE_TUNE
    int maxTokensPerCall,        // 单次最大 Token
    int maxTokensPerDay,         // 每日最大 Token
    DataClassification dataClassification, // 数据分类等级

    // Environment 环境属性
    String sourceIp,
    String timeOfDay,            // BUSINESS_HOURS / OFF_HOURS
    RiskScore riskScore,         // 实时风险评分
    String approvedPurpose       // 审批过的使用目的
) {}

public enum DataClassification {
    PUBLIC,        // 公开数据
    INTERNAL,      // 内部数据
    CONFIDENTIAL,  // 机密数据
    PII,           // 个人身份信息
    RESTRICTED     // 限制级数据（不能离开内网）
}
```

## 三、实战：ABAC 权限引擎实现

### 3.1 策略定义语言

```yaml
# ai-access-policies.yml
policies:
  - id: "POLICY-001"
    name: "限制每日Token消耗"
    description: "普通用户每日最多消耗100K Token"
    priority: 100
    target:
      subject:
        role: ["USER"]
        department: ["ENGINEERING", "PRODUCT"]
    condition:
      anyOf:
        - predicate: "context.action == 'COMPLETION'"
        - predicate: "context.action == 'EMBEDDING'"
    obligation:
      maxTokensPerDay: 100000
      maxTokensPerCall: 4096
    effect: PERMIT_WITH_LIMITS

  - id: "POLICY-002"
    name: "禁止PII数据发送到外部模型"
    description: "含个人敏感信息的数据不得调用外部API"
    priority: 200
    target:
      resource:
        modelTier: ["external"]  # GPT/Claude 等外部模型
    condition:
      anyOf:
        - predicate: "context.dataClassification == 'PII'"
        - predicate: "context.dataClassification == 'RESTRICTED'"
    effect: DENY

  - id: "POLICY-003"
    name: "VIP用户不受Token限制"
    priority: 300
    target:
      subject:
        role: ["VIP", "ADMIN"]
    condition:
      predicate: "context.action == 'COMPLETION'"
    effect: PERMIT

  - id: "POLICY-004"
    name: "非工作时间需二次审批"
    priority: 150
    condition:
      predicate: "context.timeOfDay == 'OFF_HOURS'"
    effect: PERMIT_WITH_APPROVAL
    approval:
      required: true
      approvers: ["manager", "security-team"]

  - id: "POLICY-005"
    name: "高风险评分强制拦截"
    priority: 999
    condition:
      predicate: "context.riskScore == 'HIGH'"
    effect: DENY
```

### 3.2 策略评估引擎

```java
/**
 * ABAC 策略决策引擎
 */
@Component
public class ABACPolicyEngine {

    private final List<Policy> policies;
    private final RiskAssessmentService riskService;
    private final AuditLogger auditLogger;

    public ABACPolicyEngine(RiskAssessmentService riskService,
                             AuditLogger auditLogger) {
        this.riskService = riskService;
        this.auditLogger = auditLogger;
        this.policies = loadPolicies();
    }

    /**
     * 评估 AI 调用请求是否允许
     */
    public Decision evaluate(AIPolicyContext context) {
        // Step 1: 实时风险评估
        RiskScore riskScore = riskService.assess(context);
        AIPolicyContext enrichedContext = context.withRiskScore(riskScore);

        // Step 2: 按优先级排序策略
        List<Policy> sortedPolicies = policies.stream()
            .sorted(Comparator.comparingInt(Policy::getPriority).reversed())
            .toList();

        // Step 3: 逐个评估策略
        Decision finalDecision = Decision.PERMIT;
        List<Policy> matchedPolicies = new ArrayList<>();
        List<String> obligations = new ArrayList<>();

        for (Policy policy : sortedPolicies) {
            if (!policy.matches(enrichedContext)) {
                continue;
            }

            matchedPolicies.add(policy);

            switch (policy.getEffect()) {
                case DENY -> {
                    // DENY 是最高优先级，直接返回
                    auditLogger.log(context, policy, Decision.DENY);
                    return new Decision(Decision.Result.DENY,
                        policy.getDescription(), matchedPolicies);
                }
                case PERMIT_WITH_APPROVAL -> {
                    finalDecision = Decision.PERMIT_WITH_APPROVAL;
                }
                case PERMIT_WITH_LIMITS -> {
                    obligations.addAll(policy.getObligations());
                }
                case PERMIT -> {
                    // 继续评估其他策略
                }
            }
        }

        auditLogger.log(context, null, finalDecision);
        return new Decision(finalDecision.getResult(),
            "Evaluated " + matchedPolicies.size() + " policies",
            matchedPolicies, obligations);
    }
}
```

### 3.3 策略评估的 SpEL 表达式引擎

```java
/**
 * 使用 Spring Expression Language (SpEL) 评估策略条件
 */
public class PolicyEvaluator {

    private final ExpressionParser parser = new SpelExpressionParser();

    public boolean evaluateCondition(String predicate, AIPolicyContext context) {
        if (predicate == null || predicate.isEmpty()) {
            return true;
        }

        StandardEvaluationContext evalContext = new StandardEvaluationContext();
        evalContext.setVariable("context", context);

        Expression exp = parser.parseExpression(predicate);
        Boolean result = exp.getValue(evalContext, Boolean.class);
        return result != null && result;
    }
}

/**
 * 策略匹配逻辑
 */
public class Policy {

    private String id;
    private String name;
    private String description;
    private int priority;
    private PolicyTarget target;
    private PolicyCondition condition;
    private Effect effect;
    private List<String> obligations;

    public boolean matches(AIPolicyContext context) {
        return targetMatches(context) && conditionMatches(context);
    }

    private boolean targetMatches(AIPolicyContext context) {
        if (target == null) return true;

        // 主体匹配
        if (target.getSubject() != null) {
            SubjectConstraint subj = target.getSubject();
            if (subj.getRole() != null && !subj.getRole().contains(context.userRole())) {
                return false;
            }
            if (subj.getDepartment() != null
                    && !subj.getDepartment().contains(context.userDepartment())) {
                return false;
            }
        }

        // 资源匹配
        if (target.getResource() != null) {
            ResourceConstraint res = target.getResource();
            if (res.getModelTier() != null
                    && !res.getModelTier().contains(context.modelTier())) {
                return false;
            }
        }

        return true;
    }

    private boolean conditionMatches(AIPolicyContext context) {
        if (condition == null) return true;

        if (condition.getPredicate() != null) {
            return evaluateSpEL(condition.getPredicate(), context);
        }

        if (condition.getAllOf() != null) {
            return condition.getAllOf().stream()
                .allMatch(c -> evaluateSpEL(c.getPredicate(), context));
        }

        if (condition.getAnyOf() != null) {
            return condition.getAnyOf().stream()
                .anyMatch(c -> evaluateSpEL(c.getPredicate(), context));
        }

        return true;
    }

    private boolean evaluateSpEL(String predicate, AIPolicyContext context) {
        ExpressionParser parser = new SpelExpressionParser();
        StandardEvaluationContext evalCtx = new StandardEvaluationContext();
        evalCtx.setVariable("context", context);

        try {
            return Boolean.TRUE.equals(
                parser.parseExpression(predicate).getValue(evalCtx, Boolean.class));
        } catch (Exception e) {
            return false;
        }
    }
}
```

### 3.4 AI 调用网关中的拦截器

```java
/**
 * AI 调用的权限拦截切面
 */
@Aspect
@Component
public class AIAccessControlAspect {

    private final ABACPolicyEngine policyEngine;
    private final TokenUsageTracker tokenTracker;
    private final DataClassifer dataClassifier;

    @Around("@annotation(aiAccess)")
    public Object enforceAIAccess(ProceedingJoinPoint joinPoint,
                                   AIAccess aiAccess) throws Throwable {
        // 获取方法参数
        Object[] args = joinPoint.getArgs();
        String prompt = extractPrompt(args);

        // 构建 ABAC 上下文
        AIPolicyContext context = AIPolicyContext.builder()
            .userId(SecurityContext.getCurrentUserId())
            .userRole(SecurityContext.getCurrentUserRole())
            .userDepartment(SecurityContext.getCurrentDepartment())
            .aiModel(aiAccess.model())
            .modelTier(determineModelTier(aiAccess.model()))
            .action(aiAccess.action())
            .maxTokensPerCall(aiAccess.maxTokens())
            .maxTokensPerDay(tokenTracker.getTodayUsage(
                SecurityContext.getCurrentUserId()))
            .dataClassification(dataClassifier.classify(prompt))
            .sourceIp(RequestContext.getClientIp())
            .timeOfDay(isBusinessHours() ? "BUSINESS_HOURS" : "OFF_HOURS")
            .build();

        // 评估权限
        Decision decision = policyEngine.evaluate(context);

        if (decision.getResult() == Decision.Result.DENY) {
            throw new AIAccessDeniedException(
                "AI access denied: " + decision.getReason());
        }

        if (decision.getResult() == Decision.Result.PERMIT_WITH_APPROVAL) {
            // 检查是否有有效审批
            if (!hasValidApproval(context)) {
                throw new ApprovalRequiredException(
                    "This AI call requires manager approval");
            }
        }

        // 执行 AI 调用
        Object result = joinPoint.proceed();

        // 记录 Token 使用量
        tokenTracker.recordUsage(context, result);

        return result;
    }
}
```

### 3.5 服务层使用

```java
@Service
public class SecureAIService {

    @AIAccess(
        model = "gpt-4",
        action = "COMPLETION",
        maxTokens = 4096
    )
    public String generateContent(String prompt) {
        // ABAC 拦截器自动介入
        return openAIClient.chat(prompt);
    }

    @AIAccess(
        model = "gpt-3.5-turbo",
        action = "EMBEDDING",
        maxTokens = 8192
    )
    public List<Float> embed(String text) {
        return embeddingClient.embed(text);
    }
}
```

## 四、数据分类与敏感信息拦截

### 4.1 自动数据分类器

```java
/**
 * AI 驱动的数据分类器
 * 在数据发送到外部 API 之前自动检测敏感信息
 */
@Component
public class DataClassifier {

    private final AIClient aiClient;
    private final Pattern[] sensitivePatterns = {
        Pattern.compile("\\b\\d{4}[- ]?\\d{4}[- ]?\\d{4}[- ]?\\d{4}\\b"), // 信用卡
        Pattern.compile("\\b\\d{3}-\\d{2}-\\d{4}\\b"),                      // SSN
        Pattern.compile("\\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Z]{2,}\\b"), // 邮箱
        Pattern.compile("\\b(?:\\+?86)?1[3-9]\\d{9}\\b"),                    // 手机号
        Pattern.compile("\\b\\d{6}(?:19|20)\\d{2}(?:0[1-9]|1[0-2])(?:0[1-9]|[12]\\d|3[01])\\d{3}[\\dXx]\\b") // 身份证
    };

    // 使用本地规则 + AI 双重检测
    public DataClassification classify(String content) {
        // 第一层：正则快速检测
        for (Pattern pattern : sensitivePatterns) {
            if (pattern.matcher(content).find()) {
                return DataClassification.PII;
            }
        }

        // 第二层：AI 语义检测（检测上下文中的隐式敏感信息）
        if (containsConfidentialKeywords(content)) {
            String aiClassification = aiClient.chat("""
                判断以下文本的数据分类等级，仅回复 PUBLIC/INTERNAL/CONFIDENTIAL：
                文本：%s
                """.formatted(content));
            return DataClassification.valueOf(aiClassification.trim());
        }

        return DataClassification.PUBLIC;
    }
}
```

### 4.2 Token 使用追踪

```java
/**
 * 实时 Token 使用追踪和配额管理
 */
@Component
public class TokenUsageTracker {

    private final LoadingCache<String, DailyUsage> usageCache = CacheBuilder.newBuilder()
        .expireAfterWrite(25, TimeUnit.HOURS)
        .build(new CacheLoader<>() {
            @Override
            public DailyUsage load(String userId) {
                return new DailyUsage(userId, LocalDate.now(), 0);
            }
        });

    public long getTodayUsage(String userId) {
        return usageCache.getUnchecked(userId).getTokensUsed();
    }

    public void recordUsage(AIPolicyContext context, Object aiResult) {
        String userId = context.userId();
        int tokensUsed = extractTokensFromResponse(aiResult);

        DailyUsage usage = usageCache.getUnchecked(userId);
        usage.addTokens(tokensUsed);

        // 接近配额时告警
        int maxTokens = context.maxTokensPerDay();
        if (usage.getTokensUsed() > maxTokens * 0.8) {
            alertService.sendWarning(userId,
                "AI Token usage at 80%% of daily limit (%d/%d)"
                    .formatted(usage.getTokensUsed(), maxTokens));
        }
    }
}
```

## 五、完整权限检查流程

```
API 请求到达
    │
    ▼
┌──────────────────┐
│ 1. 提取调用上下文  │  用户ID、角色、IP、时间
└────────┬─────────┘
         ▼
┌──────────────────┐
│ 2. 数据分类       │  检测 PII/机密数据
└────────┬─────────┘
         ▼
┌──────────────────┐
│ 3. 风险评估       │  行为分析、异常检测
└────────┬─────────┘
         ▼
┌──────────────────┐
│ 4. ABAC 策略评估  │  多策略链式评估
└────────┬─────────┘
         ▼
    ┌────┴────┐
    │ DECISION │
    └────┬────┘
         │
    ┌────┼────┬──────────────┐
    ▼    ▼    ▼              ▼
  DENY PERMIT APPROVAL  PERMIT_WITH_LIMITS
    │    │    │              │
    ▼    ▼    ▼              ▼
  403  放行  等待审批   限制MaxTokens/速率
```

## 六、独特观点：AI 权限的"零信任"原则

在 AI 时代，**零信任（Zero Trust）不应只用于网络安全，也应用于 AI 调用**：

1. **永不信任数据**：所有发送给 AI 的数据都应经过敏感信息扫描
2. **永不信任用户**：即使是 CEO 也应该有 Token 配额限制
3. **永不信任模型**：每个模型包装独立的 API Key，最小化密钥暴露范围
4. **持续验证**：不是鉴权一次就永久放行，每次调用都重新评估

最低权限原则（Least Privilege）在 AI 场景下的具象化：

```
不是问"用户能用 GPT-4 吗？"
而是问"用户能在工作时间、从公司VPN、用已审批的目的、
      发送非敏感数据、调用不超过4096 Token的GPT-4吗？"
```

## 七、总结

AI-Native 权限系统的核心设计原则：

- **ABAC 替代 RBAC**：主体、资源、操作、环境四维决策
- **数据分类前置**：发送数据前先分类，PII 数据不得离开内网
- **配额即权限**：Token 消耗是新的权限维度
- **实时风险评估**：行为分析 + 异常检测，而非静态规则
- **声明式策略**：YAML/注解驱动，策略即代码，Git 版本管理

---

**本文是「Java + AI 编程从入门到精通」310 篇系列的第 291 篇。全系列覆盖：**

- 基础篇（1-50）：Java 基础 + AI 概念入门
- 框架篇（51-120）：Spring AI、LangChain4j、Semantic Kernel
- 进阶篇（121-200）：RAG、Agent、多模态、微调
- 实战篇（201-250）：企业级 AI 应用实战
- 延伸篇（251-310）：专项技术突破、性能优化、架构设计

**系列全部文章持续更新中，关注获取最新内容。**
