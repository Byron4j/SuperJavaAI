# Prompt 版本管理与 A/B 测试流水线搭建，改了 Prompt 效果变好变差不再凭感觉

## 前言

"你又改 Prompt 了？"

PM 的声音带着一丝不信任。"用户反馈说最近的回答风格变了，比之前啰嗦了很多。你上周是不是调了那个什么 system prompt？"

我心里一惊。上周确实改了一版 Prompt，加了"请详细解释"这个短语。当时觉得让回答更详尽是好事，没想到用户反而觉得"啰嗦"。更要命的是——我没有记录老版本 Prompt，也没做 A/B 测试，改了就全网生效。现在想回滚，只能凭记忆恢复。

这不是个例。90% 的 AI 团队都在"盲改 Prompt"。今天这篇文章，带你建立 Prompt 版本管理体系 + A/B 测试流水线，让 Prompt 的每一次改动都有据可查、有效可证。


## 一、为什么 Prompt 管理是 LLMOps 的核心命题

### 1.1 Prompt = LLM 应用的"源代码"

在传统软件开发中，修改代码 → 跑测试 → 通过才上线。但大多数团队的 Prompt 迭代是这样的：

```bash
# 常见（错误）的 Prompt 修改方式
# 1. 开发直接改数据库里的 Prompt
# 2. 重启服务生效
# 3. 等用户反馈
# 4. 发现问题回滚（如果有备份的话）
# 5. 没有备份 → 重写
```

**Prompt 本质上就是 LLM 应用的"程序代码"**，只是它的语法是自然语言而不是 Java。代码需要 Git，Prompt 同样需要。

### 1.2 Prompt 管理的四个核心需求

| 需求 | 传统做法 | 正确的做法 |
|------|---------|-----------|
| 版本管理 | 复制粘贴到 Word/飞书 | Git-like 版本控制 |
| 效果评估 | 开发者自己试几遍 | 自动化评估 + 人工评测 |
| 上线发布 | 直接改配置 | 灰度发布 + A/B 测试 |
| 回滚 | 祈祷有备份 | 一键回滚到任意历史版本 |


## 二、Prompt 版本管理系统的设计

### 2.1 数据库设计

```sql
-- Prompt 模板表：存储所有版本的 Prompt
CREATE TABLE prompt_templates (
    id              BIGSERIAL PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,     -- Prompt 名称（如 "customer_service"）
    version         VARCHAR(50) NOT NULL,       -- 语义化版本（如 "v1.2.0"）
    version_seq     INT NOT NULL,               -- 递增序列号
    system_prompt   TEXT NOT NULL,              -- System Prompt
    user_template   TEXT,                       -- User Prompt 模板
    model           VARCHAR(100) NOT NULL,      -- 推荐模型
    parameters      JSONB NOT NULL,             -- {temperature, max_tokens, top_p...}
    status          VARCHAR(20) DEFAULT 'draft',-- draft/published/retired
    author          VARCHAR(100),               -- 修改人
    change_log      TEXT,                       -- 变更说明
    created_at      TIMESTAMP DEFAULT NOW(),
    UNIQUE(name, version)
);

-- A/B 测试实验表
CREATE TABLE prompt_experiments (
    id              BIGSERIAL PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,       -- 实验名称
    prompt_name     VARCHAR(255) NOT NULL,       -- 关联的 Prompt
    variant_a       VARCHAR(50) NOT NULL,        -- 对照组版本
    variant_b       VARCHAR(50) NOT NULL,        -- 实验组版本
    a_weight        INT DEFAULT 50,              -- 对照组流量权重
    b_weight        INT DEFAULT 50,              -- 实验组流量权重
    start_time      TIMESTAMP NOT NULL,
    end_time        TIMESTAMP,
    status          VARCHAR(20) DEFAULT 'running',-- running/completed/stopped
    metrics         JSONB,                       -- 评估指标配置
    winner          VARCHAR(20),                 -- 胜出版本 (a/b/tie)
    created_at      TIMESTAMP DEFAULT NOW()
);

-- 评估结果表
CREATE TABLE prompt_evaluations (
    id              BIGSERIAL PRIMARY KEY,
    experiment_id   BIGINT REFERENCES prompt_experiments(id),
    version         VARCHAR(50) NOT NULL,
    metric_name     VARCHAR(100) NOT NULL,       -- 指标名称
    metric_value    DOUBLE PRECISION NOT NULL,   -- 指标值
    sample_size     INT NOT NULL,                -- 样本数
    confidence      DOUBLE PRECISION,            -- 置信度
    evaluated_at    TIMESTAMP DEFAULT NOW()
);

-- 执行记录表：每笔调用的完整记录
CREATE TABLE prompt_execution_logs (
    id              BIGSERIAL PRIMARY KEY,
    prompt_name     VARCHAR(255) NOT NULL,
    prompt_version  VARCHAR(50) NOT NULL,
    user_id         VARCHAR(100),
    session_id      VARCHAR(100),
    input_variables JSONB,                       -- 模板变量值
    rendered_prompt TEXT NOT NULL,                -- 渲染后的完整 Prompt
    completion      TEXT NOT NULL,                -- LLM 回答
    model           VARCHAR(100),
    input_tokens    INT,
    output_tokens   INT,
    latency_ms      INT,
    cost_usd        DOUBLE PRECISION,
    user_rating     INT,                         -- 用户评分 (1-5)
    created_at      TIMESTAMP DEFAULT NOW()
);

-- 索引
CREATE INDEX idx_execution_logs_prompt ON prompt_execution_logs(prompt_name, prompt_version);
CREATE INDEX idx_execution_logs_created ON prompt_execution_logs(created_at);
CREATE INDEX idx_experiments_status ON prompt_experiments(status);
```

### 2.2 Prompt 版本管理服务

```java
@Service
@Slf4j
public class PromptVersionManager {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Autowired
    private StringRedisTemplate redisTemplate;

    private static final String CACHE_KEY_PREFIX = "prompt:template:";

    /**
     * 发布新版本 Prompt
     */
    public PromptTemplate publishNewVersion(String name, String systemPrompt,
                                             String userTemplate, String model,
                                             Map<String, Object> parameters,
                                             String author, String changeLog) {
        // 1. 获取当前最新版本号
        int currentSeq = getCurrentVersionSeq(name);
        int newSeq = currentSeq + 1;
        String version = generateVersion(name, newSeq);

        // 2. 存储到数据库
        String jsonParams = serializeJson(parameters);
        jdbcTemplate.update("""
            INSERT INTO prompt_templates 
            (name, version, version_seq, system_prompt, user_template, 
             model, parameters, status, author, change_log)
            VALUES (?, ?, ?, ?, ?, ?, ?::jsonb, 'published', ?, ?)
            """, name, version, newSeq, systemPrompt, userTemplate,
            model, jsonParams, author, changeLog);

        // 3. 更新 Redis 缓存
        PromptTemplate template = PromptTemplate.builder()
                .name(name)
                .version(version)
                .systemPrompt(systemPrompt)
                .userTemplate(userTemplate)
                .model(model)
                .parameters(parameters)
                .build();
        
        cacheTemplate(template);

        log.info("Published new prompt version: name={}, version={}", 
                name, version);
        return template;
    }

    /**
     * 获取 Prompt 模板（生产环境熔断回退）
     */
    public PromptTemplate getTemplate(String name, String version) {
        // 1. 先查 Redis
        String cacheKey = CACHE_KEY_PREFIX + name + ":" + version;
        String cached = redisTemplate.opsForValue().get(cacheKey);
        if (cached != null) {
            return deserialize(cached);
        }

        // 2. 查数据库
        PromptTemplate template = jdbcTemplate.queryForObject("""
            SELECT name, version, system_prompt, user_template, model, parameters
            FROM prompt_templates
            WHERE name = ? AND version = ?
            """, this::mapToTemplate, name, version);

        if (template == null) {
            throw new PromptNotFoundException(name, version);
        }

        // 3. 回写缓存
        cacheTemplate(template);
        return template;
    }

    /**
     * 获取生产版本（标签为 production 的版本）
     */
    public PromptTemplate getProduction(String name) {
        String cacheKey = CACHE_KEY_PREFIX + name + ":production";
        String version = redisTemplate.opsForValue().get(cacheKey);
        if (version != null) {
            return getTemplate(name, version);
        }

        // 查最新 published 版本
        PromptTemplate template = jdbcTemplate.queryForObject("""
            SELECT name, version, system_prompt, user_template, model, parameters
            FROM prompt_templates
            WHERE name = ? AND status = 'published'
            ORDER BY version_seq DESC
            LIMIT 1
            """, this::mapToTemplate, name);

        if (template != null) {
            redisTemplate.opsForValue().set(cacheKey, template.getVersion(), 
                    Duration.ofHours(24));
            cacheTemplate(template);
        }

        return template;
    }

    /**
     * 标记生产版本（将某版本标签为 production）
     */
    public void markAsProduction(String name, String version) {
        String cacheKey = CACHE_KEY_PREFIX + name + ":production";
        redisTemplate.opsForValue().set(cacheKey, version);
        log.info("Marked prompt {} version {} as production", name, version);
    }

    /**
     * 回滚到指定版本
     */
    public PromptTemplate rollback(String name, String version) {
        PromptTemplate template = getTemplate(name, version);
        if (template == null) {
            throw new PromptNotFoundException(name, version);
        }
        markAsProduction(name, version);
        log.warn("Rolled back prompt {} to version {}", name, version);
        return template;
    }

    /**
     * 渲染 Prompt（替换模板变量）
     */
    public String renderPrompt(PromptTemplate template, 
                                Map<String, String> variables) {
        String content = template.getSystemPrompt();
        if (template.getUserTemplate() != null) {
            content += "\n\n" + template.getUserTemplate();
        }
        
        for (Map.Entry<String, String> entry : variables.entrySet()) {
            content = content.replace("{{" + entry.getKey() + "}}", 
                    entry.getValue());
        }
        
        return content;
    }

    /**
     * 查看版本历史
     */
    public List<PromptVersionSummary> getHistory(String name) {
        return jdbcTemplate.query("""
            SELECT version, version_seq, author, change_log, 
                   status, created_at
            FROM prompt_templates
            WHERE name = ?
            ORDER BY version_seq DESC
            """, (rs, rowNum) -> new PromptVersionSummary(
                rs.getString("version"),
                rs.getInt("version_seq"),
                rs.getString("author"),
                rs.getString("change_log"),
                rs.getString("status"),
                rs.getTimestamp("created_at").toLocalDateTime()
            ), name);
    }

    /**
     * Diff 两个版本的 Prompt
     */
    public DiffResult diff(String name, String versionA, String versionB) {
        PromptTemplate a = getTemplate(name, versionA);
        PromptTemplate b = getTemplate(name, versionB);
        
        return DiffResult.builder()
                .versionA(versionA)
                .versionB(versionB)
                .systemDiff(computeDiff(a.getSystemPrompt(), b.getSystemPrompt()))
                .paramDiff(computeParamDiff(a.getParameters(), b.getParameters()))
                .build();
    }

    private int getCurrentVersionSeq(String name) {
        Integer seq = jdbcTemplate.queryForObject("""
            SELECT MAX(version_seq) FROM prompt_templates WHERE name = ?
            """, Integer.class, name);
        return seq != null ? seq : 0;
    }

    private String generateVersion(String name, int seq) {
        return "v" + (seq / 100) + "." + ((seq / 10) % 10) + "." + (seq % 10);
    }

    private void cacheTemplate(PromptTemplate template) {
        String cacheKey = CACHE_KEY_PREFIX + template.getName() 
                + ":" + template.getVersion();
        redisTemplate.opsForValue().set(cacheKey, 
                serializeJson(template), Duration.ofHours(24));
    }

    // 省略辅助方法...
}
```


## 三、A/B 测试引擎

### 3.1 实验分流器

```java
@Service
@Slf4j
public class ExperimentRouter {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Autowired
    private StringRedisTemplate redisTemplate;

    private static final String ROUTING_KEY = "experiment:routing:";

    /**
     * 为用户分配 Prompt 版本（A/B 分流）
     * 
     * 分流策略：
     * 1. 哈希分桶：保证同一用户始终看到同一版本
     * 2. 权重分配：A组/B组按配置比例分流
     * 3. 动态调整：如果实验已结束，全部路由到胜出版本
     */
    public String route(String userId, String promptName) {
        // 1. 查询正在进行的实验
        PromptExperiment experiment = findActiveExperiment(promptName);
        
        if (experiment == null) {
            // 没有实验，使用生产版本
            return "production";
        }
        
        // 2. 检查该用户是否已经分配过
        String userVariant = getUserAssignedVariant(
                userId, experiment.getId());
        if (userVariant != null) {
            return userVariant;
        }
        
        // 3. 哈希分桶
        int hash = Math.abs(userId.hashCode()) % 100;
        String assigned = hash < experiment.getAWeight() ? "a" : "b";
        
        // 4. 记录分配
        recordAssignment(userId, experiment.getId(), assigned);
        
        log.info("Routed user {} to experiment {}, variant {}", 
                userId, experiment.getId(), assigned);
        
        // 5. 返回对应版本
        return "a".equals(assigned) ? experiment.getVariantA() 
                                    : experiment.getVariantB();
    }

    /**
     * 获取活跃实验
     */
    private PromptExperiment findActiveExperiment(String promptName) {
        return jdbcTemplate.queryForObject("""
            SELECT id, prompt_name, variant_a, variant_b, 
                   a_weight, b_weight, status
            FROM prompt_experiments
            WHERE prompt_name = ? AND status = 'running'
              AND start_time <= NOW() 
              AND (end_time IS NULL OR end_time > NOW())
            ORDER BY start_time DESC
            LIMIT 1
            """, this::mapExperiment, promptName);
    }

    /**
     * 哈希一致性：同一用户始终分配到同一组
     */
    private String getUserAssignedVariant(String userId, Long experimentId) {
        String key = ROUTING_KEY + experimentId + ":" + userId;
        return redisTemplate.opsForValue().get(key);
    }

    private void recordAssignment(String userId, Long experimentId, 
                                   String variant) {
        String key = ROUTING_KEY + experimentId + ":" + userId;
        redisTemplate.opsForValue().set(key, variant, 
                Duration.ofDays(30)); // 30天内稳定
    }

    private PromptExperiment mapExperiment(ResultSet rs, int rowNum) 
            throws SQLException {
        return new PromptExperiment(
            rs.getLong("id"),
            rs.getString("prompt_name"),
            rs.getString("variant_a"),
            rs.getString("variant_b"),
            rs.getInt("a_weight"),
            rs.getInt("b_weight"),
            rs.getString("status")
        );
    }
}
```

### 3.2 Spring 集成：拦截器自动分流

```java
@Aspect
@Component
@Order(0)
public class PromptRoutingAspect {

    @Autowired
    private ExperimentRouter router;

    @Autowired
    private PromptVersionManager versionManager;

    /**
     * 拦截所有 LLM 调用，自动进行 A/B 分流
     */
    @Around("@annotation(llmCall)")
    public Object routePrompt(ProceedingJoinPoint pjp, LLMCall llmCall) 
            throws Throwable {
        
        // 1. 提取用户 ID 和 Prompt 名称
        Object[] args = pjp.getArgs();
        String userId = extractUserId(args);
        String promptName = llmCall.promptName();
        
        // 2. A/B 路由
        String targetVersion = router.route(userId, promptName);
        
        // 3. 加载对应版本的 Prompt
        PromptTemplate template;
        if ("production".equals(targetVersion)) {
            template = versionManager.getProduction(promptName);
        } else {
            template = versionManager.getTemplate(promptName, targetVersion);
        }
        
        // 4. 渲染 Prompt（替换变量）
        Map<String, String> variables = extractVariables(args);
        String renderedPrompt = versionManager.renderPrompt(template, variables);
        
        // 5. 将 Prompt 注入到调用参数中
        injectPrompt(args, renderedPrompt, template);
        
        // 6. 执行调用
        Object result = pjp.proceed(args);
        
        // 7. 记录实验数据
        logExperimentData(promptName, targetVersion, 
                userId, template, result);
        
        return result;
    }

    private void logExperimentData(String promptName, String version,
                                     String userId, PromptTemplate template,
                                     Object result) {
        // 记录执行日志到数据库，供后续统计分析
        // 使用异步方式避免阻塞主流程
        CompletableFuture.runAsync(() -> {
            // INSERT INTO prompt_execution_logs ...
        });
    }

    // 辅助方法省略...
}
```

### 3.3 实验配置 API

```java
@RestController
@RequestMapping("/api/experiments")
public class ExperimentController {

    @Autowired
    private PromptVersionManager versionManager;

    @Autowired
    private JdbcTemplate jdbcTemplate;

    /**
     * 创建 A/B 测试实验
     */
    @PostMapping
    public Map<String, Object> createExperiment(@RequestBody CreateExperimentRequest req) {
        // 1. 验证两个版本都存在
        versionManager.getTemplate(req.getPromptName(), req.getVariantA());
        versionManager.getTemplate(req.getPromptName(), req.getVariantB());
        
        // 2. 创建实验
        jdbcTemplate.update("""
            INSERT INTO prompt_experiments 
            (name, prompt_name, variant_a, variant_b, 
             a_weight, b_weight, start_time, end_time, status, metrics)
            VALUES (?, ?, ?, ?, ?, ?, NOW(), ?, 'running', ?::jsonb)
            """, req.getExperimentName(), req.getPromptName(),
            req.getVariantA(), req.getVariantB(),
            req.getAWeight(), req.getBWeight(),
            req.getEndTime(), serializeJson(req.getMetrics()));
        
        log.info("Created experiment: {} ({} vs {})",
                req.getExperimentName(), req.getVariantA(), req.getVariantB());
        
        return Map.of("status", "ok", 
                "message", "Experiment created and started");
    }

    /**
     * 查看实验结果
     */
    @GetMapping("/{experimentId}/results")
    public ExperimentResult getResults(@PathVariable long experimentId) {
        // 1. 获取实验配置
        PromptExperiment experiment = getExperiment(experimentId);
        
        // 2. 查询各分组的指标数据
        List<PromptEvaluation> evalA = getEvaluations(
                experiment.getPromptName(), experiment.getVariantA());
        List<PromptEvaluation> evalB = getEvaluations(
                experiment.getPromptName(), experiment.getVariantB());
        
        // 3. 统计分析
        return ExperimentResult.builder()
                .experimentName(experiment.getName())
                .variantA(compareVariant("A", evalA))
                .variantB(compareVariant("B", evalB))
                .winner(determineWinner(evalA, evalB))
                .confidence(calculateConfidence(evalA, evalB))
                .recommendation(generateRecommendation(evalA, evalB))
                .build();
    }

    /**
     * 结束实验并选出胜出版本
     */
    @PostMapping("/{experimentId}/complete")
    public Map<String, Object> completeExperiment(
            @PathVariable long experimentId,
            @RequestBody CompleteRequest req) {
        
        jdbcTemplate.update("""
            UPDATE prompt_experiments 
            SET status = 'completed', end_time = NOW(), winner = ?
            WHERE id = ?
            """, req.getWinner(), experimentId);
        
        // 如果选择 B 版本为胜出，标记为 production
        if ("b".equals(req.getWinner())) {
            PromptExperiment exp = getExperiment(experimentId);
            versionManager.markAsProduction(
                    exp.getPromptName(), exp.getVariantB());
        }
        
        return Map.of("status", "ok", "message", "Experiment completed");
    }

    private String determineWinner(List<PromptEvaluation> a, 
                                     List<PromptEvaluation> b) {
        double avgA = a.stream().mapToDouble(PromptEvaluation::getScore)
                .average().orElse(0);
        double avgB = b.stream().mapToDouble(PromptEvaluation::getScore)
                .average().orElse(0);
        
        if (Math.abs(avgA - avgB) < 0.1) return "tie";
        return avgA > avgB ? "a" : "b";
    }

    private PromptExperiment getExperiment(long id) {
        return jdbcTemplate.queryForObject("""
            SELECT id, name, prompt_name, variant_a, variant_b, 
                   a_weight, b_weight, status
            FROM prompt_experiments WHERE id = ?
            """, (rs, rowNum) -> new PromptExperiment(
                rs.getLong("id"), rs.getString("name"),
                rs.getString("prompt_name"),
                rs.getString("variant_a"), rs.getString("variant_b"),
                rs.getInt("a_weight"), rs.getInt("b_weight"),
                rs.getString("status")
            ), id);
    }

    private List<PromptEvaluation> getEvaluations(
            String promptName, String version) {
        return jdbcTemplate.query("""
            SELECT metric_name, metric_value, sample_size
            FROM prompt_evaluations
            WHERE version = ?
            """, (rs, rowNum) -> new PromptEvaluation(
                rs.getString("metric_name"),
                rs.getDouble("metric_value"),
                rs.getInt("sample_size")
            ), promptName + ":" + version);
    }

    // 静态内部类和数据类省略...
}
```


## 四、自动化评估流水线

### 4.1 评估指标定义

```yaml
# evaluation-metrics.yml
metrics:
  # 客观指标（自动评估）
  objective:
    - bleu_score          # BLEU 分数
    - rouge_l_score       # ROUGE-L 分数
    - response_length     # 回答长度（字数）
    - token_count         # Token 消耗
    
  # LLM 自动评估
  llm_eval:
    - accuracy            # 准确性（1-5）
    - relevance           # 相关性（1-5）
    - conciseness         # 简洁性（1-5）
    - helpfulness         # 有帮助性（1-5）
    - toxicity            # 有害性（越低越好）
    - hallucination       # 幻觉程度（越低越好）
    
  # 业务指标
  business:
    - user_satisfaction   # 用户满意度（点赞率）
    - follow_up_rate      # 追问率（越低越好）
    - task_completion     # 任务完成率
    - average_latency     # 平均延迟
    - cost_per_task       # 单任务成本
    
  # 综合评分
  composite:
    formula: "0.3*accuracy + 0.2*relevance + 0.2*helpfulness + 0.15*conciseness + 0.15*user_satisfaction"
```

### 4.2 批量评估执行器

```java
@Service
@Slf4j
public class BatchEvaluationRunner {

    @Autowired
    private PromptVersionManager versionManager;

    @Autowired
    private UnifiedAIService aiService;

    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Value("${evaluation.eval-model:gpt-4o-mini}")
    private String evalModel;

    /**
     * 对 Prompt 版本进行批量评估
     * 
     * @param name       Prompt 名称
     * @param version    待评估版本
     * @param testCases  测试用例（问题 + 参考答案）
     */
    public EvaluationReport evaluate(String name, String version,
                                      List<TestCase> testCases) {
        
        PromptTemplate template = versionManager.getTemplate(name, version);
        List<EvaluationResult> results = new ArrayList<>();
        
        // 并行评估每个测试用例
        ExecutorService executor = Executors.newFixedThreadPool(10);
        List<CompletableFuture<EvaluationResult>> futures = 
                new ArrayList<>();
        
        for (TestCase tc : testCases) {
            futures.add(CompletableFuture.supplyAsync(() -> {
                // 1. 使用该版本 Prompt 生成回答
                String renderedPrompt = versionManager.renderPrompt(
                        template, tc.getVariables());
                AIResponse response = aiService.chat(AIRequest.builder()
                        .model(template.getModel())
                        .message(renderedPrompt)
                        .temperature(
                            ((Number) template.getParameters()
                                .get("temperature")).doubleValue())
                        .build());
                
                // 2. 使用评估模型评分
                Map<String, Double> scores = evaluateWithLLM(
                        tc.getQuestion(), 
                        response.getContent(),
                        tc.getExpectedAnswer());
                
                // 3. 计算客观指标
                scores.put("token_count", 
                        (double) response.getUsage().getTotalTokens());
                scores.put("response_length", 
                        (double) response.getContent().length());
                
                return new EvaluationResult(tc.getId(), scores);
            }, executor));
        }
        
        // 收集结果
        futures.forEach(f -> {
            try {
                results.add(f.get(5, TimeUnit.MINUTES));
            } catch (Exception e) {
                log.error("Evaluation failed for a test case", e);
            }
        });
        
        executor.shutdown();
        
        // 汇总统计
        return compileReport(name, version, results);
    }

    /**
     * 使用 LLM 评估生成质量
     */
    private Map<String, Double> evaluateWithLLM(String question,
                                                  String generated,
                                                  String expected) {
        String evalPrompt = String.format("""
            你是 AI 回答评估专家。请评估以下回答质量。

            问题：%s
            回答：%s
            参考答案：%s

            请输出 JSON 格式（仅输出 JSON，不要其他文字）:
            {
              "accuracy": <1-5>,
              "relevance": <1-5>,
              "conciseness": <1-5>,
              "helpfulness": <1-5>,
              "hallucination_risk": <0-1>
            }
            """, question, generated, expected);

        AIResponse response = aiService.chat(AIRequest.builder()
                .model(evalModel)
                .message(evalPrompt)
                .temperature(0.0)
                .maxTokens(200)
                .build());

        try {
            return new ObjectMapper()
                    .readValue(response.getContent(), new TypeReference<>() {});
        } catch (Exception e) {
            log.error("Failed to parse evaluation result", e);
            return Map.of("accuracy", 3.0);
        }
    }

    /**
     * 生成评估报告
     */
    private EvaluationReport compileReport(String name, String version,
                                            List<EvaluationResult> results) {
        
        DoubleSummaryStatistics accStats = results.stream()
                .mapToDouble(r -> r.getScore("accuracy"))
                .summaryStatistics();
        DoubleSummaryStatistics relStats = results.stream()
                .mapToDouble(r -> r.getScore("relevance"))
                .summaryStatistics();
        
        double avgTokens = results.stream()
                .mapToDouble(r -> r.getScore("token_count"))
                .average().orElse(0);
        
        // 保存到数据库
        saveEvaluation(name, version, "accuracy", 
                accStats.getAverage(), results.size());
        saveEvaluation(name, version, "relevance", 
                relStats.getAverage(), results.size());
        saveEvaluation(name, version, "token_count", 
                avgTokens, results.size());
        
        return EvaluationReport.builder()
                .promptName(name)
                .version(version)
                .sampleSize(results.size())
                .avgAccuracy(accStats.getAverage())
                .avgRelevance(relStats.getAverage())
                .avgTokens(avgTokens)
                .passingRate(accStats.getAverage() >= 3.0 ? 
                    (double) results.stream()
                        .filter(r -> r.getScore("accuracy") >= 3.0)
                        .count() / results.size() : 0)
                .detailedResults(results)
                .build();
    }

    private void saveEvaluation(String name, String version,
                                 String metric, double value, int sampleSize) {
        jdbcTemplate.update("""
            INSERT INTO prompt_evaluations 
            (version, metric_name, metric_value, sample_size)
            VALUES (?, ?, ?, ?)
            """, name + ":" + version, metric, value, sampleSize);
    }

    @Data
    @AllArgsConstructor
    public static class TestCase {
        private String id;
        private String question;
        private String expectedAnswer;
        private Map<String, String> variables;
    }

    @Data
    @AllArgsConstructor
    public static class EvaluationResult {
        private String testCaseId;
        private Map<String, Double> scores;
        
        public double getScore(String key) {
            return scores.getOrDefault(key, 0.0);
        }
    }
}
```


## 五、CI/CD 集成：Prompt 发布流水线

### 5.1 GitHub Actions 工作流

```yaml
# .github/workflows/prompt-cicd.yml
name: Prompt CI/CD Pipeline

on:
  push:
    paths:
      - 'prompts/**/*.yaml'       # Prompt 定义文件
  pull_request:
    paths:
      - 'prompts/**/*.yaml'

jobs:
  # 第一步：验证 Prompt 语法
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Validate Prompt Syntax
        run: |
          python scripts/validate_prompts.py

  # 第二步：自动评估
  evaluate:
    needs: validate
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Auto Evaluation
        env:
          EVAL_MODEL: gpt-4o-mini
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
        run: |
          python scripts/evaluate_prompts.py \
            --test-cases tests/prompt_test_cases.json \
            --threshold 3.5

  # 第三步：人工审批（生产环境需要）
  approval:
    needs: evaluate
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Wait for Approval
        run: echo "Awaiting manual approval..."

  # 第四步：部署到生产
  deploy:
    needs: approval
    runs-on: ubuntu-latest
    steps:
      - name: Deploy Prompt
        env:
          GATEWAY_URL: ${{ secrets.GATEWAY_URL }}
          API_KEY: ${{ secrets.GATEWAY_API_KEY }}
        run: |
          curl -X POST "$GATEWAY_URL/api/prompts/deploy" \
            -H "Authorization: Bearer $API_KEY" \
            -H "Content-Type: application/json" \
            -d @prompts/production.yaml

  # 第五步：创建 A/B 测试（如果配置了）
  ab-test:
    needs: deploy
    if: ${{ github.event.inputs.create_ab_test == 'true' }}
    runs-on: ubuntu-latest
    steps:
      - name: Create A/B Test
        run: |
          curl -X POST "$GATEWAY_URL/api/experiments" \
            -H "Authorization: Bearer $API_KEY" \
            -H "Content-Type: application/json" \
            -d '{
              "experimentName": "prompt-v2-test",
              "promptName": "customer_service",
              "variantA": "'"${{ vars.CURRENT_PRODUCTION }}"'",
              "variantB": "'"${{ vars.NEW_VERSION }}"'",
              "aWeight": 80,
              "bWeight": 20,
              "endTime": "'"$(date -d '+7 days' --iso-8601=seconds)"'"
            }'
```

### 5.2 Prompt 定义文件规范

```yaml
# prompts/customer_service.yaml
name: customer_service
description: "客服对话 Prompt"
tags: ["customer_support", "chinese"]

versions:
  - version: "v1.3.0"
    status: production
    model: "gpt-4o"
    parameters:
      temperature: 0.7
      max_tokens: 1024
      top_p: 0.9
    
    system_prompt: |
      你是 {company_name} 的专业客服助手。
      
      核心原则：
      1. 友好、耐心、专业
      2. 先理解用户问题，再提供解决方案
      3. 如果不确定，明确告知而非编造
      4. 涉及退款/投诉类问题，引导联系人工客服
      
      当前上下文：
      - 用户等级：{user_level}
      - 产品线：{product_line}
    
    test_cases:
      - question: "我的订单为什么还没发货？"
        expected: "帮助用户查询订单状态，给出发货时间"
      - question: "我想退货，怎么操作？"
        expected: "告知退货流程，或引导联系售后"
```

### 5.3 评估脚本

```python
# scripts/evaluate_prompts.py
import json
import os
import sys
import yaml
from openai import OpenAI

def evaluate_prompt(prompt_config, test_cases, threshold):
    """对 Prompt 配置进行批量评估"""
    client = OpenAI(api_key=os.environ["OPENAI_API_KEY"])
    eval_model = os.environ.get("EVAL_MODEL", "gpt-4o-mini")
    
    scores = {"accuracy": [], "relevance": [], "helpfulness": []}
    
    for tc in test_cases:
        # 1. 使用被评估的 Prompt 生成回答
        response = client.chat.completions.create(
            model=prompt_config["model"],
            messages=[
                {"role": "system", "content": prompt_config["system_prompt"]},
                {"role": "user", "content": tc["question"]}
            ],
            temperature=prompt_config.get("parameters", {}).get("temperature", 0.7),
            max_tokens=prompt_config.get("parameters", {}).get("max_tokens", 1024)
        )
        
        answer = response.choices[0].message.content
        
        # 2. 使用评估模型评分
        eval_response = client.chat.completions.create(
            model=eval_model,
            messages=[{
                "role": "user",
                "content": f"""Evaluate this AI response (JSON only):
Question: {tc['question']}
Expected: {tc.get('expected', 'N/A')}
Response: {answer}

Output JSON: {{"accuracy": <1-5>, "relevance": <1-5>, "helpfulness": <1-5>}}"""
            }],
            temperature=0,
            max_tokens=100
        )
        
        try:
            result = json.loads(eval_response.choices[0].message.content)
            for k in scores:
                scores[k].append(result.get(k, 3))
        except json.JSONDecodeError:
            print(f"Failed to parse eval for: {tc['question'][:50]}...")
    
    # 计算平均分
    avg_scores = {k: sum(v)/len(v) for k, v in scores.items() if v}
    overall = sum(avg_scores.values()) / len(avg_scores)
    
    print(f"\n=== Evaluation Results ===")
    for k, v in avg_scores.items():
        print(f"  {k}: {v:.2f}")
    print(f"  Overall: {overall:.2f} (threshold: {threshold})")
    print(f"  {'PASS' if overall >= threshold else 'FAIL'}")
    
    return overall >= threshold


def main():
    if len(sys.argv) < 2:
        print("Usage: evaluate_prompts.py <prompt_file>")
        sys.exit(1)
    
    with open(sys.argv[1]) as f:
        prompt_config = yaml.safe_load(f)
    
    test_cases = prompt_config.get("test_cases", [])
    if not test_cases:
        print("No test cases found, skipping evaluation")
        sys.exit(0)
    
    threshold = 3.5
    success = evaluate_prompt(prompt_config, test_cases, threshold)
    sys.exit(0 if success else 1)


if __name__ == "__main__":
    main()
```


## 六、Prompt 效果看板

```java
@RestController
@RequestMapping("/api/dashboard")
public class PromptDashboardController {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    /**
     * Prompt 版本效果对比表
     */
    @GetMapping("/compare")
    public List<Map<String, Object>> compareVersions(
            @RequestParam String promptName,
            @RequestParam(defaultValue = "7") int days) {
        
        return jdbcTemplate.queryForList("""
            SELECT 
                prompt_version,
                COUNT(*) as call_count,
                AVG(latency_ms) as avg_latency,
                SUM(cost_usd) as total_cost,
                AVG(user_rating) as avg_rating,
                -- 追问率：用户在同 session 内继续问下一个问题
                COUNT(DISTINCT CASE WHEN user_rating <= 2 THEN session_id END) 
                    * 1.0 / COUNT(DISTINCT session_id) as dissatisfaction_rate
            FROM prompt_execution_logs
            WHERE prompt_name = ?
              AND created_at > NOW() - INTERVAL '%d days'
            GROUP BY prompt_version
            ORDER BY prompt_version DESC
            """.formatted(days), promptName);
    }

    /**
     * A/B 测试实时数据
     */
    @GetMapping("/experiment-live")
    public Map<String, Object> getExperimentLiveData(
            @RequestParam long experimentId) {
        
        // 查询各组实时统计
        Map<String, Object> result = new LinkedHashMap<>();
        
        // A 组数据
        Map<String, Object> aData = jdbcTemplate.queryForMap("""
            SELECT 
                COUNT(*) as calls,
                AVG(user_rating) as avg_rating,
                SUM(input_tokens + output_tokens) as total_tokens,
                SUM(cost_usd) as total_cost,
                AVG(latency_ms) as avg_latency
            FROM prompt_execution_logs ple
            JOIN prompt_experiments pe ON ple.prompt_name = pe.prompt_name
            WHERE pe.id = ?
              AND ple.prompt_version = pe.variant_a
            """, experimentId);
        
        // B 组数据
        Map<String, Object> bData = jdbcTemplate.queryForMap("""
            SELECT 
                COUNT(*) as calls,
                AVG(user_rating) as avg_rating,
                SUM(input_tokens + output_tokens) as total_tokens,
                SUM(cost_usd) as total_cost,
                AVG(latency_ms) as avg_latency
            FROM prompt_execution_logs ple
            JOIN prompt_experiments pe ON ple.prompt_name = pe.prompt_name
            WHERE pe.id = ?
              AND ple.prompt_version = pe.variant_b
            """, experimentId);
        
        result.put("variant_a", aData);
        result.put("variant_b", bData);
        result.put("significance", calculateSignificance(aData, bData));
        
        return result;
    }

    private String calculateSignificance(Map<String, Object> a, 
                                          Map<String, Object> b) {
        double ratingA = ((Number) a.get("avg_rating")).doubleValue();
        double ratingB = ((Number) b.get("avg_rating")).doubleValue();
        double diff = Math.abs(ratingA - ratingB);
        
        if (diff < 0.1) return "no_significant_difference";
        if (diff < 0.3) return "moderate_difference";
        return "significant_difference";
    }
}
```


## 总结

Prompt 版本管理与 A/B 测试是 LLMOps 的"基础设施"，不建好这个地基，上面的所有优化都是空中楼阁。核心要点：

1. **Prompt = 代码，需要 Git 级别的版本管理**：版本号、变更日志、Diff、回滚
2. **A/B 测试是唯一的"公正裁判"**：哈希一致性分桶 + 自动化评估 + 置信度计算
3. **评估要自动化**：人工评估不可持续，用 LLM 评估 LLM 是当前最佳实践
4. **CI/CD 流水线**：Prompt 修改 → 自动评估 → 人工审批 → 灰度发布 → 全量上线

不凭感觉改 Prompt，从今天开始。


---

**下篇预告**：Prompt 管理妥了，但还有一个省钱大招你没用——Semantic Cache。下篇《**LLM 缓存层设计：Redis + Semantic Cache 减少重复推理成本**》，教你用一句话省下 30% 的 API 费用。高频相同问题的反复推理，就是在烧钱！
