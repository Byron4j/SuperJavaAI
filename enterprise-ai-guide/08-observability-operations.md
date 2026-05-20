# 第 8 章 · 可观测性与运维

---

> LLM 应用上线后，你不知道它在说什么，就不知道它有没有出事。可观测性不是 nice-to-have——是 AI 系统的生命线。

---

## 8.1 三大支柱 + AI 专属维度

```java
/**
 * AI 可观测性的四大支柱
 * 
 * 传统 Observability: Logs + Metrics + Traces
 * AI Observability:   Logs + Metrics + Traces + QUALITY
 */
public class AIObservability {

    public enum Pillars {
        LOGS,       // 日志: 每次 LLM 调用的完整输入输出
        METRICS,    // 指标: 延迟、吞吐、Token消耗、错误率
        TRACES,     // 链路: 一次用户请求经过的所有服务/工具
        QUALITY     // AI质量: 幻觉率、相关度、用户满意度 (AI 专属!)
    }
}
```

---

## 8.2 LLM 全链路追踪

```java
/**
 * LLM 请求的完整追踪
 * 
 * 一次 RAG 问答的完整链路:
 * 
 * User → Gateway → Embedding(查向量库) → LLM(生成) → Guardrail(审核) → User
 *                    ↓                     ↓
 *                  VectorDB Search      Tool Calls (if Agent)
 */
@Component
public class LLMTracer {

    /**
     * 一次 LLM 调用的完整 Span
     */
    @Entity
    public class LLMCallSpan {
        // ===== 请求上下文 =====
        private String traceId;          // 全链路 ID
        private String spanId;           // 本次调用 ID
        private String parentSpanId;     // 上游 Span ID

        // ===== 调用信息 =====
        private String modelId;          // gpt-4, claude-sonnet, ...
        private String provider;         // openai, anthropic, self-hosted
        private String promptTemplateId; // 使用的 Prompt 版本
        private String scenario;         // customer_service, code_review, ...

        // ===== 输入 =====
        private String systemPrompt;
        private String userPrompt;
        private int inputTokens;

        // ===== 输出 =====
        private String completion;
        private int outputTokens;
        private String finishReason;     // stop, length, content_filter

        // ===== 性能 =====
        private long ttfbMs;             // Time To First Byte (首Token延迟)
        private long totalMs;            // 总耗时
        private int tokensPerSecond;     // 生成速度

        // ===== 质量 =====
        private double relevanceScore;   // 相关性评分
        private double hallucinationScore;// 幻觉评分
        private String userRating;       // 👍 / 👎 / null

        // ===== 成本 =====
        private double costUSD;          // 本次调用费用

        // ===== 工具调用 (如果 Agent) =====
        private List<ToolCallSpan> toolCalls;  // 嵌套的工具调用
    }

    /**
     * 工具调用 Span
     */
    @Entity
    public class ToolCallSpan {
        private String toolName;
        private String inputArgs;    // JSON
        private String outputResult; // 截断后的结果
        private long durationMs;
        private boolean success;
        private String errorMessage;
    }
}
```

---

## 8.3 核心监控指标

```java
/**
 * AI 系统监控指标体系
 */
@Component
public class AIMetricsCollector {

    private final MeterRegistry registry;

    // ===== 服务级指标 (RED: Rate, Error, Duration) =====

    public void recordRequest(String modelId) {
        registry.counter("ai.request.total",
            "model", modelId
        ).increment();
    }

    public void recordLatency(String modelId, long ttfbMs, long totalMs) {
        registry.timer("ai.request.ttfb",
            "model", modelId
        ).record(ttfbMs, TimeUnit.MILLISECONDS);

        registry.timer("ai.request.total_time",
            "model", modelId
        ).record(totalMs, TimeUnit.MILLISECONDS);
    }

    public void recordError(String modelId, String errorType) {
        registry.counter("ai.request.error",
            "model", modelId,
            "error", errorType
        ).increment();
    }

    // ===== Token 指标 =====

    public void recordTokens(String modelId, int inputTokens, int outputTokens) {
        registry.counter("ai.token.input",
            "model", modelId
        ).increment(inputTokens);

        registry.counter("ai.token.output",
            "model", modelId
        ).increment(outputTokens);
    }

    // ===== 质量指标 (AI 专属!) =====

    public void recordRelevanceScore(String scenario, double score) {
        registry.summary("ai.quality.relevance",
            "scenario", scenario
        ).record(score);
    }

    public void recordHallucinationScore(String scenario, double score) {
        registry.summary("ai.quality.hallucination",
            "scenario", scenario
        ).record(score);
    }

    public void recordUserRating(String scenario, String rating) {
        registry.counter("ai.quality.user_rating",
            "scenario", scenario,
            "rating", rating
        ).increment();
    }

    // ===== 缓存指标 =====

    public void recordCacheHit(String scenario) {
        registry.counter("ai.cache.hit",
            "scenario", scenario
        ).increment();
    }

    public void recordCacheMiss(String scenario) {
        registry.counter("ai.cache.miss",
            "scenario", scenario
        ).increment();
    }

    // ===== 护栏指标 =====

    public void recordGuardrailBlock(String guardType, String reason) {
        registry.counter("ai.guardrail.block",
            "type", guardType,
            "reason", reason
        ).increment();
    }
}
```

---

## 8.4 SLO (服务水平目标)

```java
/**
 * AI 系统 SLO 定义
 * 
 * SLO = Service Level Objective (服务水平目标)
 * SLA = Service Level Agreement  (服务水平协议 — 有赔偿)
 */
public class AISLO {

    /**
     * 推荐的企业 AI SLO
     */
    public static class RecommendedSLOs {

        // 可用性
        public static final double UPTIME = 0.999;  // 99.9% (~8小时/年)

        // 延迟
        public static final long P50_TTFB_MS = 500;   // 50% 请求 < 500ms 首Token
        public static final long P95_TTFB_MS = 2000;  // 95% 请求 < 2s 首Token
        public static final long P99_TTFB_MS = 5000;  // 99% 请求 < 5s 首Token

        // 正确率 (Eval Pass Rate)
        public static final double CORRECTNESS = 0.95; // 95% 回答功能正确

        // 安全
        public static final double SAFETY_PASS_RATE = 0.999; // 99.9% 输出安全

        // 幻觉率
        public static final double MAX_HALLUCINATION_RATE = 0.05; // 幻觉率 < 5
    }

    /**
     * SLO Burn Rate 告警
     * 
     * Burn Rate = 实际错误率 / SLO 允许错误率
     * 
     * Burn Rate > 1 → 消耗错误预算过快 → 需要告警
     */
    public class SLOBurnRateAlert {

        public void checkBurnRate(double currentErrorRate, double sloErrorBudget) {
            double burnRate = currentErrorRate / sloErrorBudget;

            if (burnRate > 14.4) {
                // 1 小时消耗了 2% 的月度错误预算 → 紧急告警
                alertService.critical("SLO Burn Rate 极高: %.1fx", burnRate);
            } else if (burnRate > 6) {
                // 1 小时消耗了 1% 的月度错误预算 → 警告
                alertService.warning("SLO Burn Rate 高: %.1fx", burnRate);
            } else if (burnRate > 2) {
                // 正常范围的偏高 → 记录
                log.warn("SLO Burn Rate 偏高: %.1fx", burnRate);
            }
        }
    }
}
```

---

## 8.5 告警策略

```java
/**
 * AI 系统的告警规则
 */
public class AlertingRules {

    /**
     * 分级告警策略
     */
    public enum AlertSeverity { CRITICAL, WARNING, INFO }

    public List<AlertRule> defineRules() {
        return List.of(

            // 🔴 紧急告警 (立即处理)
            AlertRule.of("AI 服务不可用", CRITICAL,
                "ai.request.total[5m] == 0 && ai.request.error[5m] > 0"),

            AlertRule.of("Guardrail 拦截剧增", CRITICAL,
                "rate(ai.guardrail.block[5m]) > 10"),

            AlertRule.of("幻觉率飙升", CRITICAL,
                "ai.quality.hallucination.avg[15m] > 0.10"),

            AlertRule.of("P99 延迟暴增", CRITICAL,
                "histogram_quantile(0.99, ai.request.ttfb[5m]) > 10000"),

            // 🟡 警告 (24 小时内处理)
            AlertRule.of("Token 浪费", WARNING,
                "rate(ai.token.output[1h]) / rate(ai.token.input[1h]) > 5.0"),
            // 输出 Token 是输入的 5 倍 → 模型废话太多

            AlertRule.of("缓存命中率下降", WARNING,
                "ai.cache.hit_rate[1h] < 0.30"),

            AlertRule.of("用户差评增加", WARNING,
                "rate(ai.quality.user_rating{rating='👎'}[1h]) > "
                + "rate(ai.quality.user_rating{rating='👍'}[1h]) * 0.3"),

            AlertRule.of("特定场景质量下降", WARNING,
                "ai.quality.relevance{scenario!=''}[1h] < 0.70"),

            // 🔵 通知 (无需立即行动)
            AlertRule.of("月 Token 消耗接近上限", INFO,
                "ai.token.total_month / monthly_quota > 0.85"),

            AlertRule.of("新模型流量占比异常", INFO,
                "ai.request.total{model='new_model'}[1h] < 100")
        );
    }
}
```

---

## 8.6 运维 Runbook

```yaml
AI 系统常见故障处理手册:

故障 1: LLM 返回空响应或超时
  排查步骤:
    1. 检查模型服务健康状态 (API status page / self-hosted metrics)
    2. 检查网络连接 (curl api.openai.com/v1/models)
    3. 检查 API Key 是否过期或余额不足
  应急处理:
    - 自动切换到备用模型 (fallback)
    - 返回缓存中的相似答案
    - 显示友好降级消息: "AI 服务暂时不可用，请稍后重试"

故障 2: 回答质量突然下降
  排查步骤:
    1. 最近的 Prompt 是否有变更？→ 回滚 Prompt 版本
    2. RAG 知识库是否被污染？→ 检查最近入库的文档
    3. 模型 API 是否升级？(OpenAI 静默升级)
  应急处理:
    - 回滚 Prompt 到上一个稳定版本
    - 如果是文档污染 → 移除问题文档并重建索引

故障 3: Token 消耗暴涨
  排查步骤:
    1. 哪个场景/团队的消耗暴涨？(查成本归因)
    2. 是否有恶意用户或爬虫？
    3. 是否有代码 Bug 导致重复调用？
  应急处理:
    - 对异常来源做速率限制
    - 临时降低该来源的模型级别 (GPT-4 → LLaMA-8B)
    - 检查并修复调用循环

故障 4: Guardrail 大量拦截
  排查步骤:
    1. 是 Prompt 注入攻击还是正常内容被误拦？
    2. 拦截规则的阈值是否太敏感？
  应急处理:
    - 查看拦截样本，判断攻击 vs 误拦
    - 攻击: 封禁来源 IP / API Key
    - 误拦: 调整规则阈值或加白名单

故障 5: 向量检索返回无关结果
  排查步骤:
    1. Embedding 模型是否切换？（不同模型的向量空间不兼容）
    2. 知识库文档质量是否下降？
    3. 用户的查询是否包含检索关键词不匹配？
  应急处理:
    - 检查 Embedding 模型版本
    - 重建索引 (如果模型变了)
    - 增加混合检索权重 (BM25 + 向量)
```

---

> **下一章**：[团队建设与组织变革](09-team-building-organization.md)
