# 模型热切换：零停机时间更换底层模型的 Java 实现，升级模型用户无感知

> 阅读时间：15分钟 | 适合人群：Java 高级开发/架构师 | 关键词：热切换、零停机、模型升级、动态路由

---

## 一、"GPT-4o 发布了，但我们不敢升级"

2025 年 5 月，OpenAI 发布了 GPT-4o 正式版，价格比 GPT-4 Turbo 便宜 50%，推理速度快 2 倍。全组兴奋。

但产品经理冷静地问了一句："**切换要停服多久？**"

这就是现实——线上跑着 300 个并发会话，每分钟 5000 次推理请求。你不可能说："大家等一下，我重启一下服务升级模型。" 这不是 2008 年的网页游戏，这是面向 C 端用户的 AI 产品。

更复杂的是：**不是所有场景都要切到新模型**。简单问答用 GPT-4o-mini 就够了，复杂推理用 GPT-4o，法律分析用 Claude 3.5——同一个接口背后可能是 5 个不同模型在服务。

本文将用 Java 实现一套**模型热切换框架**，支持：
- ✅ 零停机更换底层模型
- ✅ 按用户/场景/预算动态路由到不同模型
- ✅ A/B 测试：新模型先承接 5% 流量，验证后全量切换
- ✅ 灰度失败自动回滚

---

## 二、核心设计：模型注册表 + 动态路由 + 熔断回滚

### 2.1 架构思想

```
 用户请求 → ModelGateway（入口）
              │
              ▼
       ModelRouter（路由器）
         ├─ 根据 scene/user/budget 选模型
         ├─ 检查灰度比例
         ├─ 熔断检测
         │
         ▼
    DynamicModelRegistry（注册表）
         ├─ model:gpt-4o → GPT4oProvider
         ├─ model:claude-v3 → ClaudeProvider
         ├─ model:qwen-max → QwenProvider
         └─ model:local-llama → LocalLLMProvider
              │
              ▼
        具体模型 Provider → 执行推理
              │
              ▼
        结果 + Metrics 上报
```

热切换的本质是：**提供者和消费者之间的间接层**。消费者只依赖接口，提供者可以在运行时动态替换。

### 2.2 统一模型接口

```java
/**
 * 所有模型 Provider 的统一接口
 */
public interface ModelProvider {

    /** 模型唯一标识 */
    String getModelId();
    
    /** 模型显示名称 */
    String getDisplayName();
    
    /** 支持的场景列表 */
    List<String> getSupportedScenes();
    
    /** 能力标签 */
    Set<String> getCapabilityTags();
    
    /** 价格（元/1000 tokens） */
    Pricing getPricing();
    
    /** 执行推理 */
    ModelResponse chat(ModelRequest request);
    
    /** 健康检查 */
    boolean isHealthy();
    
    /** 模型优先级（数字越小越优先） */
    int getPriority();
}
```

### 2.3 模型注册表：支持运行时动态增删

```java
@Component
@Slf4j
public class DynamicModelRegistry {

    /** modelId → Provider 实现 */
    private final ConcurrentHashMap<String, ModelProvider> providers = new ConcurrentHashMap<>();
    
    /** 场景 → 候选模型列表 */
    private final ConcurrentHashMap<String, List<String>> sceneModelMapping = new ConcurrentHashMap<>();
    
    /** 模型灰度比例（modelId → 0~100） */
    private final ConcurrentHashMap<String, Integer> grayscaleRatio = new ConcurrentHashMap<>();
    
    /** 模型启用状态 */
    private final ConcurrentHashMap<String, Boolean> enabledStatus = new ConcurrentHashMap<>();

    /**
     * 注册一个新模型（可以随时调，不停机）
     */
    public void registerModel(ModelProvider provider) {
        String modelId = provider.getModelId();
        providers.put(modelId, provider);
        enabledStatus.put(modelId, true);
        grayscaleRatio.putIfAbsent(modelId, 0);
        
        // 自动建立场景映射
        for (String scene : provider.getSupportedScenes()) {
            sceneModelMapping.computeIfAbsent(scene, k -> new CopyOnWriteArrayList<>())
                .add(modelId);
        }
        
        log.info("✅ 模型已注册: {} (场景: {})", modelId, provider.getSupportedScenes());
    }
    
    /**
     * 下线一个模型（保留注册，停止流量）
     */
    public void disableModel(String modelId) {
        enabledStatus.put(modelId, false);
        log.warn("⏸️ 模型已停用: {}", modelId);
    }
    
    /**
     * 彻底移除模型
     */
    public void unregisterModel(String modelId) {
        providers.remove(modelId);
        enabledStatus.remove(modelId);
        grayscaleRatio.remove(modelId);
        
        // 从场景映射中移除
        sceneModelMapping.values().forEach(list -> list.remove(modelId));
        
        log.info("🗑️ 模型已移除: {}", modelId);
    }
    
    /**
     * 设置模型灰度比例
     */
    public void setGrayscaleRatio(String modelId, int percent) {
        if (percent < 0 || percent > 100) {
            throw new IllegalArgumentException("灰度比例必须在 0-100 之间");
        }
        grayscaleRatio.put(modelId, percent);
        log.info("📊 模型 {} 灰度比例设置为 {}%", modelId, percent);
    }
    
    /**
     * 获取场景下所有可用的模型（按优先级排序）
     */
    public List<ModelProvider> getCandidatesForScene(String scene) {
        List<String> modelIds = sceneModelMapping.getOrDefault(scene, List.of());
        
        return modelIds.stream()
            .filter(id -> enabledStatus.getOrDefault(id, false))
            .map(providers::get)
            .filter(Objects::nonNull)
            .filter(ModelProvider::isHealthy)
            .sorted(Comparator.comparingInt(ModelProvider::getPriority))
            .collect(Collectors.toList());
    }
    
    public ModelProvider getProvider(String modelId) {
        return providers.get(modelId);
    }
}
```

---

## 三、动态路由器：流量嗅探 + 灰度切分 + 熔断

### 3.1 核心路由逻辑

```java
@Component
@Slf4j
public class ModelRouter {

    @Autowired
    private DynamicModelRegistry registry;
    
    @Autowired
    private ModelMetricsCollector metrics;

    /** 熔断器状态（modelId → 最近故障次数） */
    private final Map<String, AtomicInteger> circuitBreaker = new ConcurrentHashMap<>();
    
    private static final int CIRCUIT_BREAKER_THRESHOLD = 10;  // 连续 10 次失败触发熔断
    private static final int CIRCUIT_RESET_MINUTES = 5;       // 5 分钟后尝试恢复

    /**
     * 为请求选择合适的模型
     * 
     * 决策链路：
     * 1. 用户是否指定了模型？
     * 2. 指定模型是否健康/启用/未熔断？
     * 3. 灰度用户是否命中了新模型？
     * 4. 按优先级降级到可用模型
     */
    public ModelProvider route(ModelRequest request) {
        String scene = request.getScene();
        String userId = request.getUserId();
        String forcedModel = request.getForcedModel();
        
        // 1. 用户指定了模型 → 检查是否可用
        if (forcedModel != null) {
            ModelProvider provider = registry.getProvider(forcedModel);
            if (provider != null && provider.isHealthy() && !isCircuitOpen(forcedModel)) {
                return provider;
            }
            log.warn("用户指定模型 {} 不可用，降级路由", forcedModel);
        }
        
        // 2. 获取场景候选模型
        List<ModelProvider> candidates = registry.getCandidatesForScene(scene);
        if (candidates.isEmpty()) {
            throw new NoModelAvailableException("场景 " + scene + " 无可用模型");
        }
        
        // 3. 灰度路由：检查是否命中新模型
        for (ModelProvider candidate : candidates) {
            int ratio = registry.getGrayscaleRatio(candidate.getModelId());
            if (ratio > 0 && shouldHitGrayscale(userId, ratio)) {
                if (!isCircuitOpen(candidate.getModelId())) {
                    log.debug("灰度命中: 用户 {} → 模型 {}", userId, candidate.getModelId());
                    return candidate;
                }
            }
        }
        
        // 4. 降级到稳定模型（第一个健康且未熔断的）
        for (ModelProvider candidate : candidates) {
            if (!isCircuitOpen(candidate.getModelId())) {
                return candidate;
            }
        }
        
        throw new AllModelsUnavailableException("所有模型均不可用或已熔断");
    }
    
    /**
     * 灰度命中判断：基于用户 ID 哈希
     */
    private boolean shouldHitGrayscale(String userId, int percent) {
        if (percent <= 0) return false;
        if (percent >= 100) return true;
        
        int hash = Math.abs(HashUtils.murmurHash3(userId));
        return (hash % 100) < percent;
    }
    
    /**
     * 熔断检查
     */
    private boolean isCircuitOpen(String modelId) {
        AtomicInteger failures = circuitBreaker.get(modelId);
        if (failures == null) return false;
        
        return failures.get() >= CIRCUIT_BREAKER_THRESHOLD;
    }
    
    /**
     * 上报调用结果，更新熔断状态
     */
    public void reportResult(String modelId, boolean success, long latencyMs) {
        metrics.recordCall(modelId, success, latencyMs);
        
        if (success) {
            // 成功一次就重置熔断计数（半开恢复）
            AtomicInteger failures = circuitBreaker.get(modelId);
            if (failures != null) {
                failures.set(0);
                log.info("🟢 模型 {} 恢复，熔断器重置", modelId);
            }
        } else {
            int current = circuitBreaker
                .computeIfAbsent(modelId, k -> new AtomicInteger(0))
                .incrementAndGet();
            
            if (current >= CIRCUIT_BREAKER_THRESHOLD) {
                log.error("🔴 模型 {} 触发熔断 (连续失敗 {} 次)", modelId, current);
                // 安排 5 分钟后尝试恢复
                scheduler.schedule(() -> semiOpenProbe(modelId), 
                    CIRCUIT_RESET_MINUTES, TimeUnit.MINUTES);
            }
        }
    }
}
```

### 3.2 模型调用门面

```java
@Service
public class ModelGateway {

    @Autowired
    private ModelRouter router;
    
    @Autowired
    private ModelResponseCache cache;

    /**
     * 统一的模型调用入口
     * 外部调用者不需要知道底层是哪家模型
     */
    public ModelResponse chat(ModelRequest request) {
        long startTime = System.currentTimeMillis();
        
        // 1. 检查缓存
        String cacheKey = buildCacheKey(request);
        ModelResponse cached = cache.get(cacheKey);
        if (cached != null) {
            log.debug("缓存命中: {}", cacheKey);
            return cached;
        }
        
        // 2. 路由到合适模型
        ModelProvider provider = router.route(request);
        log.info("路由决策: scene={} user={} → model={}", 
            request.getScene(), request.getUserId(), provider.getModelId());
        
        // 3. 调用并记录结果
        boolean success = false;
        long latencyMs;
        ModelResponse response;
        
        try {
            response = provider.chat(request);
            success = true;
        } catch (Exception e) {
            response = ModelResponse.error(e.getMessage());
            throw e;
        } finally {
            latencyMs = System.currentTimeMillis() - startTime;
            router.reportResult(provider.getModelId(), success, latencyMs);
        }
        
        // 4. 缓存写入
        if (success) {
            cache.put(cacheKey, response, 300);
        }
        
        // 5. 注入模型元信息（方便追踪）
        response.setMeta(Map.of(
            "model_id", provider.getModelId(),
            "provider", provider.getClass().getSimpleName(),
            "latency_ms", String.valueOf(latencyMs)
        ));
        
        return response;
    }
}
```

---

## 四、A/B 测试：新旧模型自动对比

### 4.1 双路并行调用 + 对比评估

灰度发布时，我们不仅需要把流量分到新模型，还需要**对比新老模型的输出质量**。

```java
@Component
public class ModelABTester {

    @Autowired
    private DynamicModelRegistry registry;
    
    @Autowired
    private ModelQualityEvaluator evaluator;

    /**
     * A/B 测试：对同一请求同时调用 A 模型和 B 模型，对比质量
     * 
     * 注意：用户只收到 A 模型的响应（不增加延迟）
     * B 模型的调用和对比是异步的
     */
    public void abTest(String modelA, String modelB, int trafficPercent) {
        // 路由器中已通过灰度控制了流量分配
        // 这里启动后台对比任务
        
        scheduler.scheduleAtFixedRate(() -> {
            // 从最近 5 分钟的日志中采样对比
            List<ComparePair> pairs = collectCompareData(modelA, modelB, 5);
            
            QualityReport report = evaluator.compare(pairs);
            
            if (report.getBScore() > report.getAScore() * 1.1) {
                log.info("📈 模型 {} 质量显著优于 {}，建议提高灰度比例", modelB, modelA);
                log.info("   A ({}) 得分: {:.2f}, B ({}) 得分: {:.2f}", 
                    modelA, report.getAScore(), modelB, report.getBScore());
            }
        }, 1, 5, TimeUnit.MINUTES);
    }
}
```

### 4.2 质量评估维度

```java
@Component
public class ModelQualityEvaluator {
    
    public QualityReport compare(List<ComparePair> pairs) {
        QualityReport report = new QualityReport();
        
        for (ComparePair pair : pairs) {
            // 1. 回复长度合理性（空回复 = -10）
            if (pair.getAnswerB() == null || pair.getAnswerB().length() < 5) {
                report.markBFailure("empty_response");
                continue;
            }
            
            // 2. 幻觉检测（是否虚构事实）
            int hallucinationScore = detectHallucination(pair.getAnswerB(), pair.getContext());
            report.addBScore("hallucination", hallucinationScore);
            
            // 3. 相关性评分（用 Embedding 相似度）
            double relevanceScore = calculateRelevance(
                pair.getQuestion(), pair.getAnswerB());
            report.addBScore("relevance", relevanceScore);
            
            // 4. 格式规范性（JSON/Markdown 等）
            boolean formatOK = checkFormatCompliance(pair.getAnswerB(), pair.getExpectedFormat());
            report.addBScore("format", formatOK ? 10 : 0);
        }
        
        return report;
    }
}
```

---

## 五、实战：从 GPT-4-Turbo 热切换到 GPT-4o

### 5.1 提供者实现对比

```java
@Component
public class GPT4TurboProvider implements ModelProvider {
    
    @Override
    public String getModelId() { return "gpt-4-turbo"; }
    
    @Override
    public List<String> getSupportedScenes() { 
        return List.of("general_chat", "code_generation", "analysis"); 
    }
    
    @Override
    public int getPriority() { return 100; }  // 老模型，优先级低
    
    @Override
    public Object execute(ModelRequest request) {
        // 调用 OpenAI GPT-4-Turbo API
        return openaiClient.chat("gpt-4-turbo", request);
    }
}

@Component
public class GPT4oProvider implements ModelProvider {
    
    @Override
    public String getModelId() { return "gpt-4o"; }
    
    @Override
    public List<String> getSupportedScenes() { 
        return List.of("general_chat", "code_generation", "analysis", "vision"); 
    }
    
    @Override
    public int getPriority() { return 1; }  // 新模型，优先级最高
    
    @Override
    public Object execute(ModelRequest request) {
        return openaiClient.chat("gpt-4o", request);
    }
}
```

### 5.2 灰度切换脚本（通过管理接口执行）

```bash
# 第 1 步：注册新模型（不停机，纯内存操作）
curl -X POST http://api.ai-company.internal/admin/models/register \
  -H "Content-Type: application/json" \
  -d '{
    "modelId": "gpt-4o",
    "providerClass": "com.company.ai.provider.GPT4oProvider",
    "scenes": ["general_chat", "code_generation", "analysis", "vision"],
    "priority": 1
  }'

# 第 2 步：设置灰度 5%（仅 5% 用户命中新模型）
curl -X PUT http://api.ai-company.internal/admin/models/gpt-4o/grayscale \
  -d '{"percent": 5}'

# 观察 30 分钟，检查指标...
# 错误率 0.12% ✅ | 平均延迟降 40% ✅ | 用户评分 ↑ 0.3 ✅

# 第 3 步：逐步放量
curl -X PUT http://api.ai-company.internal/admin/models/gpt-4o/grayscale \
  -d '{"percent": 20}'

# 继续观察...

# 第 4 步：全量切换（所有新请求都走 gpt-4o）
curl -X PUT http://api.ai-company.internal/admin/models/gpt-4o/grayscale \
  -d '{"percent": 100}'

# 第 5 步：保留老模型 hot standby 3 天，确认无问题后移除
curl -X PUT http://api.ai-company.internal/admin/models/gpt-4-turbo/disable
```

### 5.3 管理后台 API

```java
@RestController
@RequestMapping("/admin/models")
public class ModelAdminController {

    @Autowired
    private DynamicModelRegistry registry;
    
    @Autowired
    private ModelRouter router;

    @PostMapping("/register")
    public ApiResponse registerModel(@RequestBody ModelRegistrationDTO dto) {
        ModelProvider provider = createProviderFromDTO(dto);
        registry.registerModel(provider);
        return ApiResponse.success("模型已注册: " + dto.getModelId());
    }
    
    @PutMapping("/{modelId}/grayscale")
    public ApiResponse updateGrayscale(
            @PathVariable String modelId, 
            @RequestBody GrayscaleDTO dto) {
        registry.setGrayscaleRatio(modelId, dto.getPercent());
        return ApiResponse.success(
            String.format("灰度比例已更新: %s → %d%%", modelId, dto.getPercent()));
    }
    
    @PutMapping("/{modelId}/disable")
    public ApiResponse disableModel(@PathVariable String modelId) {
        registry.disableModel(modelId);
        return ApiResponse.success("模型已停用: " + modelId);
    }
    
    @GetMapping("/status")
    public List<ModelStatusVO> getAllStatus() {
        return registry.getAllModels().stream()
            .map(m -> ModelStatusVO.builder()
                .modelId(m.getModelId())
                .enabled(registry.isEnabled(m.getModelId()))
                .grayscale(registry.getGrayscaleRatio(m.getModelId()))
                .healthy(m.isHealthy())
                .build())
            .collect(Collectors.toList());
    }
}
```

---

## 六、灰度切换效果对比

我们在内部客服系统上做了一次 GPT-4-Turbo → GPT-4o 的灰度切换（20 万 DAU）：

| 阶段 | 时间 | gpt-4-turbo 流量 | gpt-4o 流量 | 错误率 | 平均延迟 | 备注 |
|------|------|------------------|-------------|--------|----------|------|
| 初始 | Day 1 09:00 | 100% | 0% | 0.3% | 2.1s | 基准 |
| 灰度 5% | Day 1 10:00 | 95% | 5% | 0.25% | 2.0s | 新模型延迟更低 |
| 灰度 20% | Day 1 14:00 | 80% | 20% | 0.2% | 1.8s | 稳定 |
| 灰度 50% | Day 2 09:00 | 50% | 50% | 0.15% | 1.4s | GPT-4o 速度优势明显 |
| 全量 | Day 2 16:00 | 0% | 100% | 0.1% | 1.3s | 全量完成 |

全程零停机，用户零感知。服务降级、熔断保护生效 0 次——因为 GPT-4o 实在太稳了。

---

## 七、关键注意事项

1. **会话亲和性**：同一 session 的多次调用必须路由到同一个模型（否则上下文丢失），可通过 `sessionId % 100` 做一致性哈希
2. **计费切换**：模型切换后，成本计算器必须实时感知新模型价格
3. **Prompt 兼容性**：不同模型对 System Prompt 格式要求不同，Provider 内部需要做适配
4. **监控先行**：切换前必须先确保 Metrics 采集（成功率、延迟、Token 消耗）就绪
5. **回滚预案**：保留老模型至少 72 小时，随时可一键切回

---

## 下篇预告

下一篇，我们来算笔账——**私有化部署到底划不划算？** GPU 租赁 vs 直接购买 vs API 调用，三年 TCO（总拥有成本）精确到分。QPS 多少时自建更便宜？高峰低峰怎么组合最优？我们用真实账本说话。

> 系列七-大模型部署与运维LLMOps / 162-私有化部署的成本核算：GPU 租赁 vs 购买 vs API 调用的 TCO 对比
