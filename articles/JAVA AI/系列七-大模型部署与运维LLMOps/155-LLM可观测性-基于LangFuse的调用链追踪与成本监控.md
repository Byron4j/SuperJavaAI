# LLM 可观测性：基于 LangFuse 的调用链追踪与成本监控，每一分钱花在哪都清清楚楚

## 前言

上个月，老板在管理会上甩出一个灵魂拷问："咱们上季度 AI 花了 1.2 万美金，效果提升在哪？"

我打开账单一页一页翻，GPT-4 的调用占了 80% 的费用，但到底哪些请求是 GPT-4，哪些是 GPT-4o-mini，哪些是高价值客户用的，哪些是内部测试打的，一概说不清楚。老板的脸色可以用 `#FF0000` 来形容。

这就是缺乏 LLM 可观测性的代价——**花钱如流水，复盘一脸懵**。

今天这篇文章，我用 LangFuse + OpenTelemetry + Prometheus + Grafana 搭建一套完整的 LLM 可观测性体系，让你对每一次调用的延迟、成本、质量都了如指掌。


## 一、LLM 可观测性的独特挑战

### 1.1 传统可观测性 vs LLM 可观测性

| 维度 | 传统微服务监控 | LLM 监控 |
|------|--------------|----------|
| 追踪对象 | HTTP 调用链 | Prompt → LLM → Response 调用链 |
| 延迟关注点 | 端到端延迟 | TTFT（首Token延迟）+ TPOT（每Token延迟） |
| 成本 | 忽略（服务器固定成本） | 核心（每次调用都在花钱） |
| 质量 | 正确/错误 | 相关性、幻觉、有害性 |
| 调用链嵌套 | 下游服务 | + Tool Call、RAG 检索、Agent 循环 |
| 输入/输出 | 结构化数据 | 非结构化长文本 |

### 1.2 可观测性三支柱

```yaml
# LLM 可观测性的三大支柱
pillars:
  tracing:                    # 链路追踪
    - langfuse_spans          # 完整的 LLM 调用链
    - prompt_version          # Prompt 版本关联
    - user_feedback           # 用户反馈关联
    - model_parameters        # 模型参数快照
    
  metrics:                    # 指标监控
    - latency_p50_p95_p99     # 延迟分位数
    - tokens_per_second       # 吞吐量
    - cost_per_request        # 单次成本
    - error_rate              # 错误率
    - cache_hit_rate          # 缓存命中率
    
  logging:                    # 日志分析
    - structured_logs         # 结构化日志
    - full_prompt_log         # 完整 Prompt 日志
    - full_response_log       # 完整 Response 日志
    - error_traces            # 错误追踪
```


## 二、LangFuse 部署与集成

### 2.1 LangFuse 是什么

LangFuse 是专为 LLM 应用设计的开源可观测性平台，它的核心价值：

- **LLM 原生 Tracing**：自动追踪 Prompt → LLM → Response 的完整链路
- **成本追踪**：自动计算每次调用的 Token 消耗和费用
- **用户反馈收集**：内置评分和标注功能
- **评估功能**：支持数据集评估和人工评测
- **自托管**：数据不出域，完全可控

### 2.2 自托管部署

```yaml
# docker-compose.yml
version: '3.8'

services:
  langfuse-server:
    image: ghcr.io/langfuse/langfuse:latest
    container_name: langfuse-server
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgresql://langfuse:langfuse@postgres:5432/langfuse
      - NEXTAUTH_SECRET=${NEXTAUTH_SECRET}
      - NEXTAUTH_URL=http://localhost:3000
      - SALT=${SALT}
      - TELEMETRY_ENABLED=false
    depends_on:
      postgres:
        condition: service_healthy
    restart: unless-stopped

  postgres:
    image: postgres:15-alpine
    container_name: langfuse-postgres
    environment:
      - POSTGRES_USER=langfuse
      - POSTGRES_PASSWORD=langfuse
      - POSTGRES_DB=langfuse
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U langfuse"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
```

```bash
# 生成密钥
export NEXTAUTH_SECRET=$(openssl rand -base64 32)
export SALT=$(openssl rand -base64 32)

# 启动
docker compose up -d

# 访问 http://localhost:3000
# 注册账户，创建 API Key
```

### 2.3 Java 集成 LangFuse

```xml
<!-- pom.xml -->
<dependency>
    <groupId>com.langfuse</groupId>
    <artifactId>langfuse-core</artifactId>
    <version>3.37.0</version>
</dependency>
```

```java
@Configuration
public class LangFuseConfig {

    @Value("${langfuse.public-key}")
    private String publicKey;

    @Value("${langfuse.secret-key}")
    private String secretKey;

    @Value("${langfuse.host}")
    private String host;

    @Bean
    public Langfuse langfuse() {
        return Langfuse.builder()
            .publicKey(publicKey)
            .secretKey(secretKey)
            .host(host)
            .build();
    }
}

// application.yml
langfuse:
  public-key: "pk-lf-xxx"
  secret-key: "sk-lf-xxx"
  host: "http://localhost:3000"
```

```java
/**
 * 基于 LangFuse 的 LLM 调用追踪 AOP
 * 
 * 自动追踪：
 * 1. 调用创建时间与耗时
 * 2. 模型参数（温度、maxTokens等）
 * 3. Prompt 内容与版本
 * 4. 输入/输出 Token 计数
 * 5. 成本计算
 * 6. 用户反馈
 */
@Service
@Slf4j
public class LangFuseTracingService {

    @Autowired
    private Langfuse langfuse;

    /**
     * 创建一个完整的 LLM 调用追踪
     * 
     * @param userId      用户 ID
     * @param traceName   追踪名称（如 "customer-support-chat"）
     * @param model       模型名称
     * @param input       用户输入
     * @param output      LLM 输出
     * @param usage       Token 用量
     * @param metadata    额外元数据
     */
    public void traceCompletion(String userId, String traceName,
                                 String model, List<ChatMessage> input,
                                 String output, TokenUsage usage,
                                 Map<String, Object> metadata) {
        
        // 1. 创建 Trace（一次完整交互的顶层容器）
        LangfuseTrace trace = langfuse.newTrace()
            .name(traceName)
            .userId(userId)
            .metadata(metadata)
            .build();
        
        // 2. 创建 Generation（一次 LLM 调用）
        LangfuseGeneration generation = langfuse.newGeneration()
            .trace(trace)
            .name("chat-completion")
            .model(model)
            .input(serialize(input))
            .output(output)
            .usage(new OpenAiUsage()
                .setPromptTokens(usage.getInputTokens())
                .setCompletionTokens(usage.getOutputTokens())
                .setTotalTokens(usage.getTotalTokens()))
            .metadata(Map.of(
                "prompt_version", metadata.getOrDefault("promptVersion", "v1"),
                "temperature", metadata.getOrDefault("temperature", 0.7),
                "max_tokens", metadata.getOrDefault("maxTokens", 2048)
            ))
            .build();
        
        // 3. 应用评分（可选）
        if (metadata.containsKey("quality_score")) {
            generation.score(Scores.quality(
                (Double) metadata.get("quality_score"),
                "auto_eval"));
        }
        
        log.info("Traced completion: traceName={}, model={}, tokens={}/{}",
                traceName, model, usage.getInputTokens(), usage.getOutputTokens());
    }

    /**
     * 追踪 RAG 流程：检索 + 生成
     */
    public void traceRAG(String userId, String query,
                          List<Document> retrievedDocs,
                          String generatedAnswer,
                          TokenUsage usage) {
        
        LangfuseTrace trace = langfuse.newTrace()
            .name("rag-query")
            .userId(userId)
            .input(query)
            .output(generatedAnswer)
            .build();
        
        // Span 1: 向量检索
        LangfuseSpan retrievalSpan = langfuse.newSpan()
            .trace(trace)
            .name("vector-retrieval")
            .input(query)
            .output(serialize(retrievedDocs))
            .metadata(Map.of(
                "num_docs", retrievedDocs.size(),
                "vector_db", "milvus",
                "top_k", 5
            ))
            .build();
        
        // Span 2: LLM 生成
        LangfuseGeneration generation = langfuse.newGeneration()
            .trace(trace)
            .name("rag-generation")
            .parentSpan(retrievalSpan)
            .model("gpt-4o")
            .output(generatedAnswer)
            .usage(new OpenAiUsage()
                .setPromptTokens(usage.getInputTokens())
                .setCompletionTokens(usage.getOutputTokens())
                .setTotalTokens(usage.getTotalTokens()))
            .build();
        
        log.info("Traced RAG: query={}, docs={}, tokens={}",
                query.substring(0, 50), retrievedDocs.size(), usage.getTotalTokens());
    }

    /**
     * 追踪 Agent 循环：多次 LLM 调用 + Tool Call
     */
    public void traceAgentCycle(String userId, String task,
                                 List<AgentStep> steps) {
        
        LangfuseTrace trace = langfuse.newTrace()
            .name("agent-task")
            .userId(userId)
            .input(task)
            .build();
        
        for (int i = 0; i < steps.size(); i++) {
            AgentStep step = steps.get(i);
            
            LangfuseSpan stepSpan = langfuse.newSpan()
                .trace(trace)
                .name("agent-step-" + (i + 1))
                .input(step.getThought())
                .output(step.getAction())
                .metadata(Map.of(
                    "step", i + 1,
                    "tool_call", step.getToolName()
                ))
                .build();
            
            if (step.getCompletion() != null) {
                langfuse.newGeneration()
                    .trace(trace)
                    .name("llm-call-" + (i + 1))
                    .parentSpan(stepSpan)
                    .model(step.getModel())
                    .output(step.getCompletion())
                    .usage(new OpenAiUsage()
                        .setPromptTokens(step.getInputTokens())
                        .setCompletionTokens(step.getOutputTokens()))
                    .build();
            }
        }
        
        langfuse.flush(); // 确保数据发送
        
        log.info("Traced Agent cycle: task={}, steps={}", 
                task.substring(0, 50), steps.size());
    }

    private String serialize(Object obj) {
        try {
            return new ObjectMapper().writeValueAsString(obj);
        } catch (Exception e) {
            return obj.toString();
        }
    }
}
```


## 三、OpenTelemetry 全栈追踪

### 3.1 架构设计

```yaml
# 多层追踪架构
architecture:
  application_layer:
    - opentelemetry_agent:     # Java Agent 自动埋点
        - http_requests        # HTTP 请求追踪
        - database_queries     # 数据库查询追踪
        - grpc_calls          # gRPC 调用追踪
        
    - custom_instrumentation:  # 自定义埋点
        - llm_calls           # LLM 调用（核心）
        - vector_search       # 向量检索
        - cache_operations    # 缓存操作
        - tool_executions     # Tool Call
  
  collection_layer:
    - otel_collector:         # OpenTelemetry Collector
        - trace_processing
        - metric_aggregation
        - log_enrichment
        
  storage_layer:
    - jaeger:                 # Span 存储 + 查询 UI
    - langfuse:               # LLM 专用追踪
    - prometheus:             # 指标存储
    - elasticsearch:          # 日志存储
```

### 3.2 OpenTelemetry 配置

```java
@Configuration
public class OpenTelemetryConfig {

    @Bean
    public OpenTelemetry openTelemetry() {
        // OTLP Exporter：发送到 Collector
        OtlpGrpcSpanExporter spanExporter = OtlpGrpcSpanExporter.builder()
                .setEndpoint("http://localhost:4317")
                .build();
        
        SdkTracerProvider tracerProvider = SdkTracerProvider.builder()
                .addSpanProcessor(BatchSpanProcessor.builder(spanExporter).build())
                .setResource(Resource.getDefault()
                    .merge(Resource.create(Attributes.of(
                        ResourceAttributes.SERVICE_NAME, "llm-service",
                        ResourceAttributes.SERVICE_VERSION, "1.0.0"
                    ))))
                .build();
        
        OpenTelemetrySdk openTelemetry = OpenTelemetrySdk.builder()
                .setTracerProvider(tracerProvider)
                .buildAndRegisterGlobal();
        
        Runtime.getRuntime().addShutdownHook(
                new Thread(tracerProvider::close));
        
        return openTelemetry;
    }

    @Bean
    public Tracer tracer(OpenTelemetry openTelemetry) {
        return openTelemetry.getTracer("com.example.llm");
    }
}
```

### 3.3 自定义 LLM Span

```java
@Service
public class LLMTracingService {

    @Autowired
    private Tracer tracer;

    /**
     * 创建 LLM 调用 Span
     * 
     * OTel Span 属性规范（AI Semantic Conventions）：
     */
    public Span startLLMSpan(String model, String operationName) {
        Span span = tracer.spanBuilder(operationName)
                .setSpanKind(SpanKind.CLIENT)
                .startSpan();
        
        // 设置标准属性
        span.setAttribute("llm.model", model);
        span.setAttribute("llm.operation", operationName);
        span.setAttribute("llm.timestamp", Instant.now().toString());
        
        return span;
    }

    /**
     * 在 Span 中记录 Token 用量
     */
    public void recordTokens(Span span, int inputTokens, 
                              int outputTokens) {
        span.setAttribute("llm.tokens.input", inputTokens);
        span.setAttribute("llm.tokens.output", outputTokens);
        span.setAttribute("llm.tokens.total", 
                inputTokens + outputTokens);
    }

    /**
     * 在 Span 中记录 Prompt 和 Response（采样，避免日志爆炸）
     */
    public void recordPrompt(Span span, String prompt, 
                              boolean includeFull) {
        if (includeFull) {
            span.setAttribute("llm.prompt.full", 
                    prompt.substring(0, Math.min(prompt.length(), 10000)));
        }
        span.setAttribute("llm.prompt.length", prompt.length());
    }

    /**
     * 在 Span 中记录成本
     */
    public void recordCost(Span span, String model, 
                            int inputTokens, int outputTokens) {
        double cost = calculateCost(model, inputTokens, outputTokens);
        span.setAttribute("llm.cost.usd", cost);
        
        // 成本 > $1 的调用标记为高成本
        if (cost > 1.0) {
            span.setAttribute("llm.cost.level", "high");
            span.addEvent("high_cost_call", Attributes.of(
                AttributeKey.stringKey("model"), model,
                AttributeKey.doubleKey("cost"), cost
            ));
        }
    }

    private double calculateCost(String model, int input, int output) {
        Map<String, double[]> prices = Map.of(
            "gpt-4o", new double[]{0.005, 0.015},
            "gpt-4o-mini", new double[]{0.00015, 0.0006}
        );
        double[] p = prices.getOrDefault(model, new double[]{0.001, 0.002});
        return (input * p[0] + output * p[1]) / 1000.0;
    }

    /**
     * 完整的 LLM 调用追踪包装器
     */
    public <T> T traceLLMCall(String model, String operation,
                               Supplier<T> llmCall,
                               Consumer<Span> spanDecorator) {
        Span span = startLLMSpan(model, operation);
        try (Scope scope = span.makeCurrent()) {
            T result = llmCall.get();
            spanDecorator.accept(span);
            span.setStatus(StatusCode.OK);
            return result;
        } catch (Exception e) {
            span.setStatus(StatusCode.ERROR, e.getMessage());
            span.recordException(e);
            throw e;
        } finally {
            span.end();
        }
    }
}
```


## 四、Prometheus + Grafana 指标监控

### 4.1 关键指标定义

```java
@Component
public class LLMMetrics {

    @Autowired
    private MeterRegistry meterRegistry;

    /**
     * TTFT（Time To First Token）
     * 这是 LLM 服务最重要的用户体验指标
     */
    public Timer.Sample startTTFTTimer() {
        return Timer.start(meterRegistry);
    }

    public void recordTTFT(Timer.Sample sample, String model) {
        sample.stop(Timer.builder("llm.ttft")
                .tag("model", model)
                .publishPercentiles(0.5, 0.90, 0.95, 0.99)
                .register(meterRegistry));
    }

    /**
     * TPOT（Time Per Output Token）
     */
    public void recordTPOT(String model, int outputTokens, long durationMs) {
        double msPerToken = (double) durationMs / outputTokens;
        meterRegistry.summary("llm.tpot", 
                "model", model).record(msPerToken);
    }

    /**
     * Token 消耗计数器
     */
    public void recordTokens(String model, int input, int output) {
        meterRegistry.counter("llm.tokens.input", "model", model)
                .increment(input);
        meterRegistry.counter("llm.tokens.output", "model", model)
                .increment(output);
    }

    /**
     * 成本追踪（使用 Gauge 表示累计成本）
     */
    private final Map<String, AtomicLong> costGauges = new ConcurrentHashMap<>();

    public void recordCost(String model, double cost) {
        costGauges.computeIfAbsent(model, k -> {
            AtomicLong gauge = new AtomicLong(0);
            Gauge.builder("llm.cost.cumulative", gauge, AtomicLong::get)
                .tag("model", model)
                .register(meterRegistry);
            return gauge;
        });
        costGauges.get(model).addAndGet((long) (cost * 10000));
    }

    /**
     * 缓存命中率
     */
    private final AtomicLong cacheHits = new AtomicLong();
    private final AtomicLong cacheMisses = new AtomicLong();

    public void recordCacheHit() {
        cacheHits.incrementAndGet();
        updateCacheHitRate();
    }

    public void recordCacheMiss() {
        cacheMisses.incrementAndGet();
        updateCacheHitRate();
    }

    private void updateCacheHitRate() {
        long hits = cacheHits.get();
        long total = hits + cacheMisses.get();
        if (total > 0) {
            meterRegistry.gauge("llm.cache.hit_rate", 
                    (double) hits / total);
        }
    }

    /**
     * 幻觉检测告警指标
     */
    public void recordHallucinationDetected(String model) {
        meterRegistry.counter("llm.hallucinations", 
                "model", model).increment();
    }

    /**
     * 安全拦截指标
     */
    public void recordSecurityEvent(String type, String model) {
        meterRegistry.counter("llm.security.events",
                "type", type,
                "model", model).increment();
    }
}
```

### 4.2 Grafana Dashboard 配置

```json
{
  "dashboard": {
    "title": "LLM Operations Dashboard",
    "panels": [
      {
        "title": "Total Cost (24h)",
        "targets": [
          {
            "expr": "sum(increase(llm_cost_cumulative[24h]))"
          }
        ]
      },
      {
        "title": "Request Rate by Model",
        "targets": [
          {
            "expr": "sum(rate(llm_tokens_input[5m])) by (model)"
          }
        ]
      },
      {
        "title": "P95 TTFT by Model",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(llm_ttft_bucket[5m]))"
          }
        ]
      },
      {
        "title": "Instruction Injection Detections",
        "targets": [
          {
            "expr": "increase(llm_security_events{type=\"prompt_injection\"}[1h])"
          }
        ]
      },
      {
        "title": "Cache Hit Rate",
        "targets": [
          {
            "expr": "llm_cache_hit_rate"
          }
        ]
      },
      {
        "title": "Error Rate (5min)",
        "targets": [
          {
            "expr": "sum(rate(llm_requests_total{status=\"error\"}[5m])) / sum(rate(llm_requests_total[5m]))"
          }
        ]
      }
    ]
  }
}
```


## 五、成本归因：搞清楚每一分钱花在哪

### 5.1 成本分摊模型

```java
@Service
@Slf4j
public class CostAttributionService {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Autowired
    private StringRedisTemplate redisTemplate;

    /**
     * 按租户/部门分摊成本
     */
    public Map<String, Double> getCostPerTenant(LocalDate start, 
                                                  LocalDate end) {
        String sql = """
            SELECT 
                tenant_id,
                SUM(input_tokens) as total_input,
                SUM(output_tokens) as total_output,
                AVG(cost_per_1k_input) as avg_input_price,
                AVG(cost_per_1k_output) as avg_output_price
            FROM llm_call_records
            WHERE created_at BETWEEN ? AND ?
            GROUP BY tenant_id
            """;
        
        return jdbcTemplate.query(sql, 
            (rs) -> {
                Map<String, Double> result = new LinkedHashMap<>();
                while (rs.next()) {
                    String tenant = rs.getString("tenant_id");
                    double inputCost = rs.getLong("total_input") 
                            * rs.getDouble("avg_input_price") / 1000;
                    double outputCost = rs.getLong("total_output") 
                            * rs.getDouble("avg_output_price") / 1000;
                    result.put(tenant, inputCost + outputCost);
                }
                return result;
            }, start.toString(), end.toString());
    }

    /**
     * 按模型分摊成本
     */
    public Map<String, Double> getCostPerModel(LocalDate date) {
        String hashKey = "llm:daily_cost:" + date;
        Map<Object, Object> entries = redisTemplate.opsForHash()
                .entries(hashKey);
        
        Map<String, Double> result = new LinkedHashMap<>();
        entries.forEach((k, v) -> 
            result.put(k.toString(), Double.parseDouble(v.toString())));
        return result;
    }

    /**
     * 按场景分摊（客服/搜索/推荐/内部测试）
     */
    public Map<String, CostBreakdown> getCostPerScenario(
            LocalDate start, LocalDate end) {
        String sql = """
            SELECT 
                scenario,
                COUNT(*) as call_count,
                SUM(input_tokens) as total_input,
                SUM(output_tokens) as total_output,
                SUM(cost_usd) as total_cost
            FROM llm_call_records
            WHERE created_at BETWEEN ? AND ?
            GROUP BY scenario
            ORDER BY total_cost DESC
            """;
        
        return jdbcTemplate.query(sql, 
            (rs) -> {
                Map<String, CostBreakdown> result = new LinkedHashMap<>();
                while (rs.next()) {
                    result.put(rs.getString("scenario"), 
                        new CostBreakdown(
                            rs.getString("scenario"),
                            rs.getLong("call_count"),
                            rs.getLong("total_input"),
                            rs.getLong("total_output"),
                            rs.getDouble("total_cost")
                        ));
                }
                return result;
            }, start.toString(), end.toString());
    }

    @Data
    @AllArgsConstructor
    public static class CostBreakdown {
        private String scenario;
        private long callCount;
        private long totalInputTokens;
        private long totalOutputTokens;
        private double totalCost;
    }

    /**
     * 成本异常检测：发现成本暴涨的调用
     */
    public List<String> detectCostAnomalies() {
        String sql = """
            SELECT request_id, model, cost_usd, 
                   input_tokens, output_tokens
            FROM llm_call_records
            WHERE created_at > NOW() - INTERVAL '1 hour'
              AND cost_usd > ?
            ORDER BY cost_usd DESC
            LIMIT 20
            """;
        
        return jdbcTemplate.query(sql,
            (rs, rowNum) -> String.format(
                "[%s] Request %s: %s | $%.4f | in=%d out=%d",
                rs.getTimestamp("created_at"),
                rs.getString("request_id"),
                rs.getString("model"),
                rs.getDouble("cost_usd"),
                rs.getInt("input_tokens"),
                rs.getInt("output_tokens")
            ), 1.0); // 单次超过 $1 的记录
    }
}
```

### 5.2 成本优化分析

```java
@Service
public class CostOptimizer {

    @Autowired
    private CostAttributionService attributionService;

    /**
     * 成本优化建议
     */
    public List<OptimizationSuggestion> analyze(LocalDate date) {
        List<OptimizationSuggestion> suggestions = new ArrayList<>();
        
        // 1. 高频重复调用 → 推荐缓存
        long duplicateCalls = countDuplicateCalls(date);
        if (duplicateCalls > 1000) {
            double estimatedSaving = duplicateCalls * 0.001; // 保守估计每次 $0.001
            suggestions.add(new OptimizationSuggestion(
                "语义缓存",
                "检测到 " + duplicateCalls + " 次相似调用，启用 Semantic Cache 可节省约 $" 
                        + String.format("%.2f", estimatedSaving) + "/天",
                estimatedSaving));
        }
        
        // 2. 小任务用大模型 → 推荐降级
        Map<String, Long> modelUsage = getModelUsage(date);
        if (modelUsage.getOrDefault("gpt-4o", 0L) > modelUsage.getOrDefault("gpt-4o-mini", 0L)) {
            double savingPerCall = 0.005 - 0.00015; // GPT-4o vs GPT-4o-mini
            long potentialDowngrades = modelUsage.get("gpt-4o") / 2;
            double estimatedSaving = potentialDowngrades * savingPerCall * 1000;
            suggestions.add(new OptimizationSuggestion(
                "模型降级",
                "部分 GPT-4o 调用可以降级为 GPT-4o-mini，预估节省 $" 
                        + String.format("%.2f", estimatedSaving) + "/天",
                estimatedSaving));
        }
        
        // 3. 输出 Token 过长
        long excessiveOutputCalls = countExcessiveTokens(date, 2048);
        if (excessiveOutputCalls > 100) {
            double estimatedWaste = excessiveOutputCalls * 0.003;
            suggestions.add(new OptimizationSuggestion(
                "限制 max_tokens",
                excessiveOutputCalls + " 次调用输出超过 2048 tokens，考虑限制输出长度",
                estimatedWaste));
        }
        
        return suggestions;
    }

    @Data
    @AllArgsConstructor
    public static class OptimizationSuggestion {
        private String title;
        private String description;
        private double estimatedDailySaving;
    }

    private long countDuplicateCalls(LocalDate date) { return 0; }
    private Map<String, Long> getModelUsage(LocalDate date) { return Map.of(); }
    private long countExcessiveTokens(LocalDate date, int threshold) { return 0; }
}
```


## 六、用户反馈闭环

### 6.1 LangFuse 反馈收集

```java
@RestController
@RequestMapping("/api/feedback")
public class FeedbackController {

    @Autowired
    private Langfuse langfuse;

    /**
     * 用户对回答进行评分（点赞/踩）
     */
    @PostMapping("/score")
    public Mono<Map<String, String>> submitScore(@RequestBody ScoreRequest request) {
        return Mono.fromCallable(() -> {
            LangfuseScore score = langfuse.newScore()
                .name("user_feedback")
                .value(request.getScore()) // 1.0 = 点赞, 0.0 = 踩
                .traceId(request.getTraceId())
                .observationId(request.getGenerationId())
                .comment(request.getComment())
                .userId(request.getUserId())
                .build();
            
            log.info("Feedback submitted: score={}, traceId={}", 
                    request.getScore(), request.getTraceId());
            
            return Map.of("status", "ok");
        });
    }

    /**
     * 获取低分回答列表（用于人工审核）
     */
    @GetMapping("/low-scores")
    public List<FeedbackRecord> getLowScores(
            @RequestParam(defaultValue = "0.3") double maxScore,
            @RequestParam(defaultValue = "100") int limit) {
        // 从 LangFuse 查询低分记录
        return List.of(); // 实际调用 LangFuse API
    }

    @Data
    public static class ScoreRequest {
        private String traceId;
        private String generationId;
        private String userId;
        private double score;
        private String comment;
    }
}
```

### 6.2 自动质量评估

```java
@Service
public class AutoEvaluationService {

    @Autowired
    private UnifiedAIService aiService;

    @Value("${evaluation.model:gpt-4o-mini}")
    private String evalModel;

    /**
     * 使用 LLM 自动评估生成质量
     * 
     * 评估维度：
     * 1. 准确性：回答是否正确
     * 2. 相关性：回答是否切题
     * 3. 完整性：回答是否完整
     * 4. 安全性：回答是否有害
     */
    public EvaluationResult evaluate(String question, String answer) {
        String evalPrompt = """
            你是 AI 回答质量评估专家。请对以下回答评分（1-5分）：

            问题：%s

            回答：%s

            请从以下维度评分，输出 JSON 格式：
            {
              "accuracy": <1-5>,
              "relevance": <1-5>,
              "completeness": <1-5>,
              "safety": <1-5>,
              "overall": <1-5>,
              "summary": "<一句话评价>"
            }
            """.formatted(question, answer);

        AIResponse response = aiService.chat(AIRequest.builder()
                .model(evalModel)
                .message(evalPrompt)
                .temperature(0.1) // 低温度保持评估一致性
                .maxTokens(300)
                .build());

        return parseEvaluationResult(response.getContent());
    }

    /**
     * 批量评估：对生产数据进行抽样评估
     */
    @Scheduled(cron = "0 0 2 * * ?") // 每天凌晨2点
    public void scheduledEvaluation() {
        // 1. 抽样昨天的对话
        List<Conversation> samples = sampleYesterdayConversations(200);
        
        // 2. 批量评估
        List<EvaluationResult> results = samples.stream()
                .map(c -> evaluate(c.getQuestion(), c.getAnswer()))
                .toList();
        
        // 3. 报告
        double avgScore = results.stream()
                .mapToDouble(EvaluationResult::getOverall)
                .average()
                .orElse(0);
        
        log.info("Daily auto-evaluation: {} samples, avg score: {}", 
                results.size(), String.format("%.2f", avgScore));
        
        if (avgScore < 3.5) {
            // 告警：质量下降
            alertService.sendAlert(String.format(
                "AI 回答质量告警：日均评分 %.2f（阈值 3.5）", avgScore));
        }
    }

    private List<Conversation> sampleYesterdayConversations(int n) {
        return List.of();
    }
}
```


## 总结

LLM 可观测性不是一个"nice to have"，而是生产 AI 系统的必需品。回顾核心要点：

1. **LangFuse 是 LLM 原生的可观测性工具**，开源自托管，专注 LLM 调用链
2. **三根支柱缺一不可**：Tracing（链路）+ Metrics（指标）+ Logging（日志）
3. **成本归因是可观测性的第一优先级**，知道钱花在哪才能优化
4. **用户反馈闭环**让 LLM 可观测性从"看数据"变成"改效果"
5. **自动评估**降低人工评测成本，持续监控回答质量

有了这套体系，下次老板再问"钱花在哪"，你可以打开 Grafana 面板逐项展示，让他从"红色"变"绿色"。


---

**下篇预告**：可观测性让我们看清楚了每一次调用，但 Prompt 的迭代管理呢？改了 Prompt 之后效果到底变好还是变差？下篇《**Prompt 版本管理与 A/B 测试流水线搭建**》，教你像管理代码一样管理 Prompt，告别"凭感觉改 Prompt"的时代！
