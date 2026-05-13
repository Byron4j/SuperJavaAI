# 我用Spring Boot + LangChain4j，2周上线了一个AI代码审查SaaS，3个月收获300付费用户

> 2025年底，我在公司内部做了一个AI代码审查工具，本来只是给团队用。没想到被朋友传到外面，几天内有了500个注册用户。我一咬牙把它改成了SaaS产品，挂上付费。3个月后，300个付费用户，MRR（月经常性收入）$3,500。而整个产品的技术栈，简单到令人发指：Spring Boot + LangChain4j + PostgreSQL + Redis。

## 一、产品是什么？

先看一眼产品截图（文字描述）：

```
AI Code Reviewer - 智能代码审查助手

支持的审查维度：
├── 🔒 安全漏洞检测（SQL注入、XSS、敏感信息泄露等）
├── 🐛 Bug风险分析（空指针、资源泄漏、并发问题等）
├── 📐 代码规范检查（命名、注释、复杂度等）
├── ⚡ 性能优化建议（慢查询、不必要循环、内存浪费等）
├── 🏗️ 架构设计审查（分层是否合理、循环依赖等）
└── 🧪 测试覆盖率评估（缺失测试的关键路径）

定价方案：
┌──────────┬──────────┬──────────┬──────────┐
│  Free    │  Pro     │  Team    │ Enterprise│
│  $0/月   │ $9.9/月  │ $29.9/月 │ 定制      │
│  3次/天  │ 50次/天  │ 200次/天 │ 无限      │
│  单仓库  │ 5个仓库  │ 20个仓库 │ 无限      │
│  基础规则│ 高级规则 │ 自定义规则│ 私有化部署 │
└──────────┴──────────┴──────────┴──────────┘
```

## 二、完整技术架构

整个项目2周上线，技术栈如下：

```
┌─────────────────────────────────────────────────────────┐
│                      前端 (React)                         │
│  - GitHub OAuth登录                                      │
│  - 仓库选择器 + 代码Diff展示                              │
│  - 审查结果可视化（问题列表、严重程度分类）                  │
│  - 仪表盘（审查统计、趋势图）                               │
└──────────────────────┬──────────────────────────────────┘
                       │ REST API (JSON)
┌──────────────────────┴──────────────────────────────────┐
│                  后端 (Spring Boot 3.2)                    │
│                                                           │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │ Auth Module  │  │ Core Module │  │ Billing Mod │     │
│  │ - JWT       │  │ - 代码获取   │  │ - Stripe    │     │
│  │ - GitHub    │  │ - AI审查    │  │ - 用量计费   │     │
│  │   OAuth     │  │ - 结果分析   │  │ - 套餐管理   │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
│                                                           │
│  ┌──────────────────────────────────────────────┐       │
│  │          LangChain4j AI 层                     │       │
│  │  - 代码审查Chain（多阶段审查流程）                │       │
│  │  - Prompt模板管理                               │       │
│  │  - Token消耗追踪                                │       │
│  │  - 审查结果结构化输出（JSON Schema约束）           │       │
│  └──────────────────────────────────────────────┘       │
└──────────────────────┬──────────────────────────────────┘
                       │
┌──────────────────────┴──────────────────────────────────┐
│                   基础设施                                 │
│  - PostgreSQL 16 (用户、项目、审查记录)                     │
│  - Redis (会话、缓存、限流)                                │
│  - MinIO (审查报告文件存储)                                │
│  - Docker Compose (一键部署)                              │
└─────────────────────────────────────────────────────────┘
```

## 三、核心代码实现

### 3.1 GitHub集成

```java
/**
 * GitHub仓库集成服务
 * 连接GitHub API获取代码变更
 */
@Service
public class GitHubIntegrationService {
    
    private final RestTemplate restTemplate;
    private final String githubApiBase = "https://api.github.com";
    
    /**
     * 获取Pull Request的文件变更
     */
    public List<FileChange> getPRChanges(String owner, String repo, int prNumber,
                                          String accessToken) {
        String url = String.format(
            "%s/repos/%s/%s/pulls/%d/files", 
            githubApiBase, owner, repo, prNumber
        );
        
        HttpHeaders headers = new HttpHeaders();
        headers.setBearerAuth(accessToken);
        headers.set("Accept", "application/vnd.github.v3+json");
        
        ResponseEntity<List<Map>> response = restTemplate.exchange(
            url, HttpMethod.GET, 
            new HttpEntity<>(headers), 
            new ParameterizedTypeReference<List<Map>>() {}
        );
        
        return response.getBody().stream()
            .filter(file -> isReviewableFile((String) file.get("filename")))
            .map(file -> FileChange.builder()
                .filename((String) file.get("filename"))
                .status((String) file.get("status"))
                .patch((String) file.get("patch"))
                .additions((int) file.get("additions"))
                .deletions((int) file.get("deletions"))
                .build())
            .collect(Collectors.toList());
    }
    
    /**
     * 判断文件是否需要审查（跳过配置文件、node_modules等）
     */
    private boolean isReviewableFile(String filename) {
        return filename.endsWith(".java") || filename.endsWith(".kt")
            || filename.endsWith(".py") || filename.endsWith(".ts")
            || filename.endsWith(".js") || filename.endsWith(".go");
    }
    
    /**
     * 创建审查评论（直接评论到GitHub PR上）
     */
    public void createReviewComment(String owner, String repo, int prNumber,
                                     String commitId, String filePath,
                                     int line, String comment, 
                                     String accessToken) {
        String url = String.format(
            "%s/repos/%s/%s/pulls/%d/reviews",
            githubApiBase, owner, repo, prNumber
        );
        
        Map<String, Object> reviewBody = Map.of(
            "commit_id", commitId,
            "event", "COMMENT",
            "comments", List.of(Map.of(
                "path", filePath,
                "line", line,
                "body", comment
            ))
        );
        
        HttpHeaders headers = new HttpHeaders();
        headers.setBearerAuth(accessToken);
        
        restTemplate.postForEntity(url, 
            new HttpEntity<>(reviewBody, headers), Map.class);
    }
}
```

### 3.2 核心AI审查引擎

```java
/**
 * AI代码审查引擎 - 整个产品的核心
 * 使用LangChain4j实现多阶段审查Chain
 */
@Service
@Slf4j
public class CodeReviewEngine {
    
    @Autowired
    private ChatLanguageModel chatModel;
    
    /**
     * 多阶段审查流程
     * 阶段1：安全检查
     * 阶段2：Bug检测
     * 阶段3：性能分析
     * 阶段4：代码规范
     * 阶段5：架构审查
     */
    public ReviewResult reviewCode(ReviewRequest request) {
        
        ReviewResult result = new ReviewResult();
        result.setReviewId(UUID.randomUUID().toString());
        result.setRepository(request.getRepository());
        result.setBranch(request.getBranch());
        result.setCommitId(request.getCommitId());
        
        List<ReviewIssue> allIssues = new ArrayList<>();
        
        // 阶段1: 安全检查（最重要，优先处理）
        List<ReviewIssue> securityIssues = reviewSecurityAspect(
            request.getFiles(), request.getProjectContext()
        );
        allIssues.addAll(securityIssues);
        
        // 阶段2: Bug检测
        List<ReviewIssue> bugIssues = reviewBugAspect(
            request.getFiles()
        );
        allIssues.addAll(bugIssues);
        
        // 阶段3: 性能分析
        List<ReviewIssue> perfIssues = reviewPerformanceAspect(
            request.getFiles()
        );
        allIssues.addAll(perfIssues);
        
        // 阶段4: 代码规范
        List<ReviewIssue> styleIssues = reviewStyleAspect(
            request.getFiles()
        );
        allIssues.addAll(styleIssues);
        
        // 阶段5: 架构审查（仅对大型变更触发）
        if (request.getFiles().size() > 10 || request.isArchitectureReview()) {
            List<ReviewIssue> archIssues = reviewArchitectureAspect(
                request.getFiles(), request.getProjectContext()
            );
            allIssues.addAll(archIssues);
        }
        
        // 汇总分析
        result.setIssues(allIssues);
        result.setSummary(generateSummary(allIssues));
        result.setScores(calculateScores(allIssues, request.getFiles()));
        
        return result;
    }
    
    /**
     * 安全检查 - 最高优先级
     */
    private List<ReviewIssue> reviewSecurityAspect(
            List<FileChange> files, ProjectContext context) {
        
        List<ReviewIssue> issues = new ArrayList<>();
        
        for (FileChange file : files) {
            if (file.getPatch() == null || file.getPatch().isEmpty()) {
                continue;
            }
            
            String prompt = buildSecurityReviewPrompt(file, context);
            String aiResponse = chatModel.generate(prompt);
            
            issues.addAll(parseStructuredResponse(aiResponse, file.getFilename()));
        }
        
        return issues;
    }
    
    /**
     * 构建安全检查Prompt
     */
    private String buildSecurityReviewPrompt(FileChange file, ProjectContext context) {
        return """
            # 角色
            你是一个资深的安全代码审查专家。你的任务是在给定代码变更中找出所有潜在的安全问题。
            
            # 审查重点
            1. SQL注入风险（字符串拼接SQL、未使用参数化查询）
            2. XSS漏洞（未转义的用户输入）
            3. SSRF风险（用户可控的URL请求）
            4. 敏感信息泄露（日志中打印密码/Token、硬编码密钥）
            5. 权限绕过风险（缺少鉴权检查）
            6. 反序列化漏洞（不安全的反序列化）
            7. 路径遍历（用户输入的文件路径未校验）
            8. 加密问题（使用弱加密算法、硬编码的盐值）
            
            # 项目上下文
            框架：%s
            ORM：%s
            安全框架：%s
            
            # 代码变更
            文件：%s
            变更类型：%s
            
            ```%s
            %s
            ```
            
            # 输出格式
            请以严格JSON数组格式输出，不要添加任何其他文本：
            [
              {
                "severity": "CRITICAL|HIGH|MEDIUM|LOW",
                "category": "SQL_INJECTION|XSS|...",
                "line": 行号,
                "title": "问题简述",
                "description": "详细问题描述（要包含攻击场景）",
                "codeSnippet": "问题代码片段",
                "suggestion": "修复建议（包含代码示例）",
                "cweId": "对应的CWE编号（如果有）"
              }
            ]
            
            如果没有发现问题，返回空数组 []。
            只返回JSON，不要添加任何解释。
            """.formatted(
                context.getFramework(),    // Spring Boot / Quarkus等
                context.getOrm(),          // MyBatis / JPA等
                context.getSecurityFramework(), // Spring Security / Shiro等
                file.getFilename(),
                file.getStatus(),
                detectLanguage(file.getFilename()),
                file.getPatch()
            );
    }
    
    /**
     * 汇总生成审查总结
     */
    private String generateSummary(List<ReviewIssue> issues) {
        if (issues.isEmpty()) {
            return "🎉 未发现任何问题，代码质量良好！";
        }
        
        long criticalCount = issues.stream()
            .filter(i -> i.getSeverity() == Severity.CRITICAL).count();
        long highCount = issues.stream()
            .filter(i -> i.getSeverity() == Severity.HIGH).count();
        long mediumCount = issues.stream()
            .filter(i -> i.getSeverity() == Severity.MEDIUM).count();
        long lowCount = issues.stream()
            .filter(i -> i.getSeverity() == Severity.LOW).count();
        
        Map<String, Long> categoryBreakdown = issues.stream()
            .collect(Collectors.groupingBy(
                ReviewIssue::getCategory, Collectors.counting()
            ));
        
        return """
            📊 审查总结
            
            发现问题：%d 个
            ├── 🔴 严重(Critical): %d 个
            ├── 🟠 高危(High): %d 个
            ├── 🟡 中危(Medium): %d 个
            └── 🟢 低危(Low): %d 个
            
            问题分类：
            %s
            
            💡 建议优先处理所有Critical和High级别的问题。
            """.formatted(
                issues.size(),
                criticalCount, highCount, mediumCount, lowCount,
                categoryBreakdown.entrySet().stream()
                    .map(e -> "  - " + e.getKey() + ": " + e.getValue() + "个")
                    .collect(Collectors.joining("\n"))
            );
    }
    
    /**
     * 解析AI返回的结构化JSON
     */
    private List<ReviewIssue> parseStructuredResponse(
            String aiResponse, String filename) {
        try {
            // 清理markdown代码块标记
            String cleaned = aiResponse
                .replaceAll("```json\\s*", "")
                .replaceAll("```\\s*", "")
                .trim();
            
            // 解析JSON
            ObjectMapper mapper = new ObjectMapper();
            ReviewIssue[] issues = mapper.readValue(cleaned, ReviewIssue[].class);
            
            // 补充文件名信息
            return Arrays.stream(issues)
                .peek(i -> i.setFile(filename))
                .collect(Collectors.toList());
                
        } catch (Exception e) {
            log.error("AI审查结果解析失败: {}", e.getMessage());
            // 降级处理：返回一个解析错误提示
            return List.of(ReviewIssue.builder()
                .severity(Severity.LOW)
                .category("PARSE_ERROR")
                .file(filename)
                .title("AI审查结果解析异常")
                .description("AI返回的结果格式异常，请重试审查")
                .build());
        }
    }
}
```

### 3.3 Token消耗优化

```java
/**
 * AI调用成本控制
 * 这是SaaS产品盈利的关键
 */
@Service
public class TokenOptimizer {
    
    /**
     * 智能文件筛选 - 不是所有文件都送审
     * 只审查有实际变更的代码行
     */
    public List<FileChange> filterReviewableChanges(List<FileChange> files, 
                                                     SubscriptionTier tier) {
        
        return files.stream()
            .filter(f -> {
                // Free套餐：只审查变更行数少于200的文件
                if (tier == SubscriptionTier.FREE && f.getAdditions() > 200) {
                    return false;
                }
                // 跳过纯删除的文件（没有新代码审查意义小）
                if (f.getPatch() == null || f.getPatch().trim().isEmpty()) {
                    return false;
                }
                // 跳过大文件（>500行变更，成本高但审查效果差）
                if (f.getAdditions() + f.getDeletions() > 500) {
                    return false;
                }
                return true;
            })
            .limit(getFileLimit(tier))
            .collect(Collectors.toList());
    }
    
    /**
     * 审查结果缓存
     * 相同commit sha的代码不用重复审查
     */
    @Cacheable(value = "reviewCache", key = "#commitId + '_' + #filePath")
    public ReviewResult getCachedOrReview(String commitId, String filePath, 
                                           ReviewRequest request) {
        // 如果缓存未命中，执行实际审查
        return reviewEngine.reviewCode(request);
    }
    
    /**
     * Token消耗预估
     * 让用户在使用前知道会消耗多少Token
     */
    public TokenEstimate estimateTokenUsage(List<FileChange> files, 
                                             SubscriptionTier tier) {
        long estimatedInputTokens = 0;
        long estimatedOutputTokens = 0;
        
        for (FileChange file : files) {
            // 输入：代码内容 + Prompt模板
            if (file.getPatch() != null) {
                estimatedInputTokens += estimateTokens(file.getPatch()) + 500; // +Prompt
            }
            // 输出：审查结果（通常为输入的30%-50%）
            estimatedOutputTokens += estimateTokens(file.getPatch()) * 0.4;
        }
        
        // 根据套餐限制Token用量
        long maxTokens = switch (tier) {
            case FREE -> 8000;
            case PRO -> 50000;
            case TEAM -> 200000;
            case ENTERPRISE -> Long.MAX_VALUE;
        };
        
        return TokenEstimate.builder()
            .estimatedInputTokens(estimatedInputTokens)
            .estimatedOutputTokens((long) estimatedOutputTokens)
            .totalTokens(estimatedInputTokens + (long) estimatedOutputTokens)
            .estimatedCost(estimateCost(tier, estimatedInputTokens, 
                (long) estimatedOutputTokens))
            .withinLimit(estimatedInputTokens + estimatedOutputTokens <= maxTokens)
            .build();
    }
    
    private int getFileLimit(SubscriptionTier tier) {
        return switch (tier) {
            case FREE -> 5;
            case PRO -> 20;
            case TEAM -> 50;
            case ENTERPRISE -> Integer.MAX_VALUE;
        };
    }
}
```

### 3.4 计费系统

```java
/**
 * SaaS计费系统
 */
@Service
public class BillingService {
    
    @Autowired
    private UsageTracker usageTracker;
    
    @Autowired
    private StripeService stripeService;
    
    /**
     * 用量计费逻辑
     */
    public MonthlyBill calculateBill(String tenantId, LocalDate period) {
        Tenant tenant = tenantService.findById(tenantId);
        Subscription subscription = tenant.getActiveSubscription();
        
        MonthlyBill bill = MonthlyBill.builder()
            .tenantId(tenantId)
            .period(period)
            .build();
        
        // 1. 基础订阅费
        bill.setBaseFee(subscription.getMonthlyPrice());
        
        // 2. 超额使用费
        UsageStats usage = usageTracker.getMonthlyUsage(tenantId, period);
        List<OverageCharge> overageCharges = new ArrayList<>();
        
        // API调用次数超额
        if (usage.getTotalReviews() > subscription.getMaxReviewsPerMonth()) {
            long overage = usage.getTotalReviews() 
                - subscription.getMaxReviewsPerMonth();
            double charge = overage * subscription.getOveragePricePerReview();
            overageCharges.add(new OverageCharge(
                "API调用超额", overage, 
                subscription.getOveragePricePerReview(), charge
            ));
            bill.addOverageCharge(overageCharges.get(overageCharges.size() - 1));
        }
        
        // Token超额
        if (usage.getTotalTokens() > subscription.getMaxTokensPerMonth()) {
            long overageTokens = usage.getTotalTokens() 
                - subscription.getMaxTokensPerMonth();
            double charge = (overageTokens / 1000.0) 
                * subscription.getOveragePricePerKToken();
            overageCharges.add(new OverageCharge(
                "Token超额", overageTokens,
                subscription.getOveragePricePerKToken(), charge
            ));
            bill.addOverageCharge(overageCharges.get(overageCharges.size() - 1));
        }
        
        // 3. 总费用
        double overageTotal = overageCharges.stream()
            .mapToDouble(OverageCharge::getCharge).sum();
        bill.setTotalAmount(bill.getBaseFee() + overageTotal);
        
        return bill;
    }
    
    /**
     * 使用量耗尽时的降级策略
     */
    public ServiceResponse handleQuotaExhausted(String tenantId) {
        Tenant tenant = tenantService.findById(tenantId);
        
        // 通知用户升级
        String message = String.format("""
            ⚠️ 您的本月%使用量已用尽。
            
            当前套餐：%s
            已用审查次数：%d/%d
            已用Token：%d/%d
            
            您可以：
            1. 等待下月重置（%d天后）
            2. 升级套餐以获得更多用量
            3. 购买一次性加量包
            
            查看升级方案：%s
            """,
            tenant.getServiceUsageDescription(),
            tenant.getSubscription().getPlanName(),
            usageTracker.getCurrentMonthReviews(tenantId),
            tenant.getSubscription().getMaxReviewsPerMonth(),
            usageTracker.getCurrentMonthTokens(tenantId),
            tenant.getSubscription().getMaxTokensPerMonth(),
            LocalDate.now().lengthOfMonth() - LocalDate.now().getDayOfMonth() + 1,
            "https://codereview.ai/upgrade"
        );
        
        return ServiceResponse.quotaExhausted(message);
    }
}
```

## 四、冷启动：如何获取前100个付费用户

有了产品只是第一步，怎么让人付费是关键。

### 4.1 我的冷启动策略

```
第1周：内部种子用户（30人）
  - 公司内5个团队试用
  - 收集反馈，快速迭代
  - 最重要的反馈：他们不会用命令行，要Web界面！

第2周：朋友圈+技术群（100+注册）
  - 写了一篇"A Java Developer Built An AI Code Reviewer"发到Reddit的r/java
  - 同步发到V2EX的"分享创造"板块
  - 效果：Reddit获得200+ upvote，V2EX获得80+回复
  - 当天新增注册150人

第3周：Product Hunt上线（500+注册）
  - 精心准备Product Hunt页面
  - 前2小时冲到当日第3名
  - 获得"Product of the Day"银牌
  - 当天付费转化12人

第4-12周：口碑+SEO（稳定增长）
  - GitHub上开源审查规则引擎
  - YouTube发布教程视频
  - 在Stack Overflow回答相关问题时巧妙引导
  - 邀请用户写评测文章换取免费使用
```

### 4.2 付费转化漏斗

```
注册 → 首次使用 → 7日内复用 → 付费

1,200注册用户
  ↓ 85% 转化率
1,020人完成首次代码审查
  ↓ 40% 转化率  
408人在7天内再次使用
  ↓ 15% 转化率（逐步优化到20%+）
61人付费（4个月累计）
  
每月新增付费用户：约30-40人
付费流失率：约8%/月
净增长：约25人/月
```

### 4.3 定价策略的3次调整

**第1版定价（失败）：按审查次数卖点数包**

用户讨厌预付费点数概念，总担心"用完了怎么办"。

**第2版定价（还行）：$9.9/月的单一付费方案**

转化率上来了，但客单价太低，无法覆盖成本。

**第3版定价（最终版，效果最好）：三档订阅制**

```
Free: 免费，但有明确限制 → 让用户先体验价值
Pro: $9.9/月 → 大多数用户的选择
Team: $29.9/月 → 团队用户升级
Enterprise: 定制报价 → 利润大头
```

**定价的三个经验：**
1. 必须有免费版（放长线钓大鱼），但不能太慷慨（要让用户有升级动力）
2. 最大的付费群体在Pro档，但利润最高的在Enterprise
3. 年付打折（$99/年 vs $9.9×12=$118.8/年），锁定用户减少流失

## 五、3个月收入数据

```
月份    注册用户   付费用户   MRR       API成本   净利润
第1月     500         42     $415     $110      $305
第2月     850        120     $1,188   $280      $908
第3月    1200        240     $2,376   $520      $1,856
第4月    1700        300     $3,500   $750      $2,750 ← 当前
```

**关键指标：**
- 月付转化率：稳定的17-22%（高于SaaS行业平均的3-5%）
- 月流失率：8%（SaaS行业平均5-7%，偏高但正常）
- LTV：$45（$9.9/月 ÷ 8%月流失率）
- CAC（获客成本）：约$3（主要靠内容营销和口碑，成本极低）
- **LTV/CAC = 15x（非常健康！）**

## 六、踩过的坑

1. **AI审查结果不稳定**：同一个文件审查两次结果不同。解决方案：降低temperature到0.1，加上结构化输出约束。

2. **大文件审查超时**：OpenAI API超时60秒，大文件来不及返回。解决：文件切片，每个chunk不超过200行。

3. **用户不信任AI审查结果**：解决：每个问题标注严重度和可信度，高可信度自动标绿，低可信度标灰让用户自己判断。

4. **免费用户滥用**：有人注册50个免费账号轮流用。解决：加GitHub OAuth验证，限制GitHub账号只能绑定一个账户。

## 七、写在最后

很多人问我：一个AI代码审查工具，技术壁垒在哪？大厂想做不是分分钟秒杀你？

我的回答是：**技术没有壁垒，但场景有。**

大厂要做也是做通用代码审查，覆盖所有语言和场景。但我做的是垂直的——Java代码审查+Spring Boot项目专项检查+微服务架构审查。我对这个场景的理解，比大厂深10倍。

而且我只有一个人，决策快，用户需求响应快，迭代快。大厂三个月开一次会才能定下来的功能，我3天上线。

**小团队AI创业的优势不在技术，在速度和对垂直场景的理解。**

---

*下期预告：**B02-AI自动生成单元测试：一个让测试团队效率提升10倍的内部工具，如何变成月入5万的付费产品？**——我会分享如何用AI自动生成高质量的Java单元测试用例，包括Mock生成、边界条件覆盖、测试数据构造。以及如何把内部工具变成商业化产品。*

---

> **作者简介**：某大厂Java架构师转AI技术负责人，专注Java+AI工程化落地。AI代码审查SaaS作者，300+付费用户。关注我，每周一篇Java+AI硬核实战。
