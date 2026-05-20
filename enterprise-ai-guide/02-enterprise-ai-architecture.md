# 第 2 章 · 企业 AI 架构设计

---

> AI 系统的架构不是"把 LLM API 包一层就完事"。一个生产级 AI 架构需要解决：流控、多模型路由、缓存、安全、可观测、成本归因。

---

## 2.1 企业 AI 分层架构

```
┌──────────────────────────────────────────────────────────────────┐
│                      接入层 (Access Layer)                        │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐    │
│  │ Web Chat │  │ API/SDK  │  │ IDE 插件  │  │ 企业内部系统   │    │
│  └──────────┘  └──────────┘  └──────────┘  └──────────────┘    │
└──────────────────────────────┬───────────────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────────────┐
│                   AI Gateway (统一流量管理)                        │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌──────────┐ ┌────────────┐  │
│  │ 认证鉴权│ │ 流量控制│ │ 模型路由│ │ Prompt过滤│ │ 审计日志   │  │
│  │ AuthN/Z│ │RateLimit│ │ Router │ │ Guardrail │ │Audit Log  │  │
│  └────────┘ └────────┘ └────────┘ └──────────┘ └────────────┘  │
└──────────────────────────────┬───────────────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────────────┐
│                     AI 能力层 (Capability Layer)                   │
│  ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌───────────────┐   │
│  │ RAG 引擎   │ │ Agent 引擎 │ │ Fine-tuning│ │ 评估/评测引擎  │   │
│  │ 检索-生成  │ │ 多步推理    │ │ 模型微调   │ │ Eval Engine  │   │
│  └───────────┘ └───────────┘ └───────────┘ └───────────────┘   │
│  ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌───────────────┐   │
│  │ Prompt管理 │ │ 对话记忆   │ │ 结构化输出 │ │ FunctionCall  │   │
│  │ 版本/模板  │ │Memory Mgmt│ │ JSON Mode │ │ Tool Orchest │   │
│  └───────────┘ └───────────┘ └───────────┘ └───────────────┘   │
└──────────────────────────────┬───────────────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────────────┐
│                    模型层 (Model Layer)                            │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    Model Router                          │   │
│  │   根据任务复杂度 / 成本 / 延迟 → 路由到最佳模型          │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────┐       │
│  │云端 API   │ │自建推理   │ │HuggingFace│ │ 边缘推理      │       │
│  │GPT/Claude│ │vLLM等    │ │ 模型库    │ │ 端侧小模型    │       │
│  └──────────┘ └──────────┘ └──────────┘ └──────────────┘       │
└──────────────────────────────┬───────────────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────────────┐
│                    基础设施层 (Infrastructure)                      │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────┐       │
│  │ GPU集群   │ │ 向量数据库│ │ 消息队列  │ │ 对象存储      │       │
│  │ K8s + GPU│ │ Milvus等 │ │ Kafka    │ │ S3/MinIO    │       │
│  └──────────┘ └──────────┘ └──────────┘ └──────────────┘       │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────┐       │
│  │ 监控告警  │ │ 日志收集  │ │ 链路追踪  │ │ CI/CD       │       │
│  │Prometheus│ │  ELK     │ │ Jaeger   │ │ Argo/GitOps │       │
│  └──────────┘ └──────────┘ └──────────┘ └──────────────┘       │
└──────────────────────────────────────────────────────────────────┘
```

---

## 2.2 AI Gateway 设计

AI Gateway 是企业 AI 架构的**核心枢纽**——所有 LLM 请求必须经过它。

```java
/**
 * 企业 AI Gateway
 * 
 * 类比: API Gateway (Kong/APISIX)，但专为 LLM 流量设计
 * 必须功能: 统一鉴权、流量整形、模型路由、安全护栏、成本归因
 */
@Service
public class AIGateway {

    private final AuthService auth;
    private final RateLimiter rateLimiter;
    private final ModelRouter router;
    private final GuardrailEngine guardrail;
    private final CostTracker costTracker;
    private final AuditLogger audit;

    /**
     * 核心请求处理流水线
     */
    public AIResponse processRequest(AIRequest request) {

        // ===== Step 1: 认证 & 鉴权 =====
        auth.authenticate(request.apiKey());
        auth.authorize(request.userId(), request.modelId());

        // ===== Step 2: 流量控制 =====
        if (!rateLimiter.tryAcquire(request)) {
            return AIResponse.tooManyRequests();
        }

        // ===== Step 3: 输入安全护栏 =====
        GuardrailResult inputCheck = guardrail.checkInput(request.prompt());
        if (!inputCheck.passed()) {
            audit.log("INPUT_BLOCKED", request, inputCheck.reason());
            return AIResponse.blocked(inputCheck.reason());
        }

        // ===== Step 4: 缓存查找 =====
        Optional<AIResponse> cached = cache.lookup(request.promptHash());
        if (cached.isPresent()) {
            return cached.get();  // 语义相同的请求直接返回缓存
        }

        // ===== Step 5: 智能路由 =====
        ModelTarget target = router.route(request);
        // 根据: 任务复杂度、成本预算、可用性、延迟要求

        // ===== Step 6: 调用 LLM =====
        AIResponse response = target.invoke(request);

        // ===== Step 7: 输出安全护栏 =====
        GuardrailResult outputCheck = guardrail.checkOutput(response.content());
        if (!outputCheck.passed()) {
            audit.log("OUTPUT_BLOCKED", request, outputCheck.reason());
            return AIResponse.blocked(outputCheck.reason());
        }

        // ===== Step 8: 成本记录 =====
        costTracker.record(request, response, target.cost());

        // ===== Step 9: 审计日志 =====
        audit.log("SUCCESS", request, response);

        // ===== Step 10: 更新缓存 =====
        cache.put(request.promptHash(), response);

        return response;
    }
}

/**
 * 模型路由器——企业 AI 架构的"大脑"
 */
class ModelRouter {

    /**
     * 根据请求特征选择最优模型
     */
    public ModelTarget route(AIRequest request) {
        return decisionTree(request);
    }

    private ModelTarget decisionTree(AIRequest request) {
        // ===== 规则 1: 简单任务 → 小模型 (省钱) =====
        if (request.complexity() == Complexity.LOW) {
            if (request.needsChinese()) {
                return ModelTarget.QWEN_7B;      // 中文小模型
            }
            return ModelTarget.LLAMA_3B;          // 通用小模型
        }

        // ===== 规则 2: 代码任务 → 代码专用模型 =====
        if (request.category() == Category.CODE) {
            if (request.complexity() == Complexity.HIGH) {
                return ModelTarget.CLAUDE_SONNET; // 代码能力最强
            }
            return ModelTarget.DEEPSEEK_CODER;    // 开源代码模型
        }

        // ===== 规则 3: 复杂推理 → 大模型 =====
        if (request.complexity() == Complexity.HIGH) {
            if (request.budget() == Budget.PREMIUM) {
                return ModelTarget.GPT_4;         // 最贵但最强
            }
            return ModelTarget.LLAMA_70B;         // 自建大模型
        }

        // ===== 规则 4: 默认 → 性价比模型 =====
        return ModelTarget.LLAMA_8B;
    }
}
```

---

## 2.3 AI 平台核心模块设计

### 2.3.1 Prompt 管理中心

```java
/**
 * Prompt 版本管理 (类比: Git for Prompts)
 */
@Service
public class PromptManager {

    /**
     * Prompt 模板: 可参数化、可版本化
     */
    @Entity
    public class PromptTemplate {
        private String id;
        private String name;
        private int version;             // 版本号
        private String systemPrompt;     // 系统提示
        private String userPromptTemplate; // 用户提示模板 (支持 {{variable}})
        private List<String> variables;   // 模板变量
        private Map<String, String> metadata; // 作者、更新时间、说明

        /**
         * 渲染 Prompt
         */
        public RenderedPrompt render(Map<String, String> params) {
            String rendered = userPromptTemplate;
            for (var entry : params.entrySet()) {
                rendered = rendered.replace("{{" + entry.getKey() + "}}", entry.getValue());
            }
            return new RenderedPrompt(systemPrompt, rendered);
        }
    }

    /**
     * Prompt 的 CI/CD 流程
     * 
     * draft → review → staging → canary → production
     */
    public void promote(String promptId) {
        PromptTemplate current = getCurrentVersion(promptId);

        // 1. 新版本从 draft 开始
        PromptTemplate draft = current.newDraft();
        draft.setSystemPrompt("增强版 prompt...");

        // 2. 跑评估
        EvalResult eval = evaluator.evaluate(draft);
        if (eval.passRate() < 0.95) {
            throw new EvalFailedException("评估未通过: " + eval.summary());
        }

        // 3. 通过后发布到 staging
        deployToStaging(draft, 10);  // 10% 流量

        // 4. 监控 24h 无异常后全量
        if (monitor.isHealthy(draft, Duration.ofHours(24))) {
            deployToProduction(draft, 100);
        }
    }
}
```

### 2.3.2 多租户与隔离

```java
/**
 * 企业级多租户设计
 * 
 * AI 平台通常需要服务多个业务线 (客服、营销、研发...)
 */
@Service
public class MultiTenantAIPlatform {

    /**
     * 租户隔离的三种级别
     */
    public enum TenantIsolationLevel {
        LOGICAL,      // 逻辑隔离: 共享模型实例，Prompt/数据在应用层隔离
        NAMESPACE,    // 命名空间隔离: 不同租户用不同模型实例
        PHYSICAL      // 物理隔离: 不同租户用独立 GPU/集群 (金融、军工场景)
    }

    @Entity
    public class Tenant {
        private String id;
        private String name;
        private TenantIsolationLevel isolationLevel;

        // ===== 每个租户的限额 =====
        private long maxTokensPerMonth;
        private int maxRequestsPerMinute;
        private int maxConcurrentRequests;
        private List<String> allowedModels;     // 允许使用的模型白名单

        // ===== 每个租户的独立配置 =====
        private PromptTemplate defaultPrompt;   // 租户专属 Prompt
        private String knowledgeBaseId;         // 租户专属知识库
        private Map<String, String> dataPolicy; // 数据保存策略
    }

    /**
     * 租户间成本归因
     */
    @Scheduled(cron = "0 0 1 * * ?")  // 每日凌晨计算
    public void generateDailyCostReport() {
        for (Tenant tenant : tenantService.getAllActive()) {
            DailyCost cost = costTracker.getTenantCost(tenant.id(), LocalDate.now());
            // 按照模型、按用户、按场景做二级归因
            costService.save(cost);
            // 发送日报给租户管理员
            notifyService.sendDailyReport(tenant.adminEmail(), cost);
        }
    }
}
```

---

## 2.4 Agent 架构 (Anthropic 工作流模式)

基于 Anthropic 2024 年 12 月发布的最佳实践，企业 AI Agent 应采用**从简单到复杂**的渐进式架构：

```java
/**
 * Agent 架构的 5 种模式 (按复杂度递增)
 * 
 * 原则: "从最简单的方案开始，只在需要时才增加复杂度"
 */
public class AgenticPatterns {

    // ===== 模式 1: 增强型 LLM (Augmented LLM) =====
    // 最简单的模式：LLM + 检索 + 工具
    public String augmentedLLM(String prompt) {
        var context = vectorDB.search(embed(prompt), 5);
        var toolResult = executeToolsIfNeeded(prompt);
        return llm.generate(prompt, context, toolResult);
    }

    // ===== 模式 2: Prompt Chain (链式调用) =====
    // 将任务分解为固定的子步骤串联
    public Document generateDocument(String topic) {
        var outline = llm.generate("写大纲: " + topic);
        var checked = llm.generate("检查大纲: " + outline);  // gate
        if (!isValid(checked)) return generateDocument(topic);
        return llm.generate("根据大纲写正文: " + outline);
    }

    // ===== 模式 3: Router (路由分发) =====
    // 根据输入类型分派到不同处理链
    public String routeAndProcess(String query) {
        var category = llm.classify(query, categories);
        return switch (category) {
            case "退款" -> refundAgent.process(query);
            case "技术" -> techAgent.process(query);
            case "投诉" -> complaintAgent.process(query);
            default -> generalAgent.process(query);
        };
    }

    // ===== 模式 4: Orchestrator-Worker (编排-执行) =====
    // 中心 Agent 动态分解任务，分配给 Worker
    public Report orchestratorWorker(String complexTask) {
        var subtasks = orchestrator.decompose(complexTask);
        var results = subtasks.parallelStream()
            .map(task -> workerPool.execute(task))
            .toList();
        return orchestrator.synthesize(results);
    }

    // ===== 模式 5: Autonomous Agent (自主 Agent) =====
    // 完全自主：观察 → 决策 → 执行 → 反思 → 循环
    public void autonomousAgent(String goal) {
        while (!goalAchieved && steps < MAX_STEPS) {
            var observation = observe(env);
            var action = llm.decide(goal, observation, history);
            var result = execute(action);
            var reflection = llm.reflect(goal, action, result);
            if (reflection.needsAdjustment) adjustPlan();
            history.record(action, result, reflection);
        }
    }
}
```

---

## 2.5 企业 AI 架构反模式

```java
/**
 * 常见架构错误——请避开
 */
public class ArchitectureAntiPatterns {

    // ❌ 反模式 1: LLM 直连
    // 每个业务系统直接调 OpenAI API
    // 后果: 无流量控制、无安全审查、无法成本归因
    // ✅ 正确: 所有流量经过 AI Gateway

    // ❌ 反模式 2: Prompt 硬编码
    // System Prompt 写在代码里，改一个字要重新部署
    // 后果: 迭代慢、无法 A/B 测试、出问题回滚慢
    // ✅ 正确: Prompt 管理中心 + 版本化

    // ❌ 反模式 3: 一把梭全用 GPT-4
    // 简单分类、情感分析也用最贵的模型
    // 后果: 成本爆炸 (简单任务成本是大模型的 1/50)
    // ✅ 正确: 模型路由器，按复杂度分级

    // ❌ 反模式 4: 没有降级机制
    // 大模型挂了整个业务就挂了
    // 后果: 单点故障影响面极大
    // ✅ 正确: 模型级别的 fallback (GPT挂了切 Claude)、缓存兜底

    // ❌ 反模式 5: 忽略 Hallucination (幻觉)
    // 不验证 LLM 输出的正确性就展示给用户
    // 后果: 严重信任危机，尤其金融/医疗/法律场景
    // ✅ 正确: 对关键输出做事实核查 (RAG + source citation)

    // ❌ 反模式 6: 无成本归因
    // 不知道哪个团队/场景花了多少钱
    // 后果: 月底账单炸裂 → CFO 叫停所有 AI 项目
    // ✅ 正确: 从 Day 1 开始按 tenant/project 归因成本
}
```

---

> **下一章**：[技术选型与模型评估](03-technology-selection-model-evaluation.md)
