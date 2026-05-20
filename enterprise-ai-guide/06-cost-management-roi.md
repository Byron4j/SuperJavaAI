# 第 6 章 · 成本管理与 ROI

---

> AI 项目最常见的死亡原因不是技术问题，而是 **"钱花完了还没看到效果"**。本章教你如何计算、优化和证明 AI 投入的回报。

---

## 6.1 企业 AI 成本全景

```
企业 AI 总拥有成本 (TCO) = 云端 API 费用 + 自建基础设施 + 人力 + 工具/平台 + 机会成本

                         ┌─────────────────────────────┐
                         │        AI TCO 冰山水面图      │
                         ├─────────────────────────────┤
  显性成本 (看得到)       │  ██ API Token 费用          │
                         │  ██ GPU 云服务费用           │
                         │  ██ SaaS 工具订阅            │
  ───────────────────────┼─────────────────────────────┤
  隐性成本 (容易忽略)     │  ██ ML 工程师薪资           │
                         │  ██ Prompt 迭代人力         │
                         │  ██ 数据标注成本            │
                         │  ██ 安全审计费用            │
                         │  ██ 员工培训成本            │
                         │  ██ 系统集成开发            │
  ───────────────────────┼─────────────────────────────┤
  隐性损失 (最难量化)     │  ██ Prompt 迭代的试错成本   │
                         │  ██ 模型选择错误的重做成本    │
                         │  ██ 用户信任损失(幻觉事故)   │
                         └─────────────────────────────┘
```

---

## 6.2 Token 成本优化策略

```java
/**
 * Token 成本优化 —— 直接影响月度账单
 */
@Service
public class TokenCostOptimizer {

    /**
     * 策略 1: 语义缓存 (Semantic Cache)
     * 
     * 相似度 > 0.98 的问题直接返回缓存答案
     * 省钱效果: 20-40% (取决于问答重复率)
     */
    @Cacheable(value = "ai_semantic", key = "#questionHash")
    public AIResponse semanticCache(String questionHash, String question) {
        // 余弦相似度 > 0.98 = 相同问题
        // 直接返回，0 Token 消耗
        return null; // 未命中时正常调 LLM
    }

    /**
     * 策略 2: 分级模型路由
     * 
     * 省钱效果: 30-60% (将简单任务从 GPT-4 降到 LLaMA-8B)
     */
    public ModelTarget routeByComplexity(AIRequest request) {
        ComplexityScore score = complexityEstimator.estimate(request.prompt());

        if (score.level() <= 2) {
            return ModelTarget.LLAMA_3B;   // ~$0.0001/1K tokens
        } else if (score.level() <= 3) {
            return ModelTarget.LLAMA_8B;   // ~$0.0003/1K tokens
        } else if (score.level() <= 4) {
            return ModelTarget.LLAMA_70B;  // ~$0.005/1K tokens
        } else {
            return ModelTarget.GPT_4;      // ~$0.03/1K tokens (仅 5% 的流量)
        }
    }

    /**
     * 策略 3: Prompt 长度优化
     * 
     * System Prompt 每天被调用 N 次，每减少 100 token 就省一大笔钱
     */
    public String optimizePrompt(String prompt) {
        String optimized = prompt;

        // 1. 删除冗余说明 (LLM 不需要读 3 遍同样的规则)
        optimized = removeRedundancy(optimized);

        // 2. Few-shot 示例精简 (保留 2-3 个最有代表性的)
        optimized = trimExamples(optimized, 3);

        // 3. 用缩写 (不影响模型理解的标准化缩写)
        // 但不要牺牲清晰度 !!! (Prompt 的清晰度 > 长度)

        return optimized;
    }

    /**
     * 策略 4: 输出长度控制
     */
    public AIResponse controlOutputLength(AIRequest request) {
        // 在 System Prompt 中加入输出限制
        String constrainedPrompt = request.systemPrompt() + """
            
            ## 输出长度限制
            - 回答不超过 200 字 (除非用户明确要求详细)
            - 不要重复问题
            - 不要加无意义的客套话 (如"很高兴为你服务")
            """;

        // API 层面的 max_tokens 限制
        request.setMaxTokens(300);

        return invokeLLM(request.withSystemPrompt(constrainedPrompt));
    }

    /**
     * 策略 5: 流式输出提前终止
     * 
     * 用户在生成过程中点了 Stop → 立即停止，节省剩余 Token
     */
    public Flux<String> streamWithEarlyStop(AIRequest request, SinkHandle sink) {
        return llm.stream(request)
            .takeUntil(token -> sink.isCancelled());  // 用户取消 → 停止
    }
}
```

---

## 6.3 GPU 容量规划

```java
/**
 * GPU 容量规划 —— 自建推理的第一步
 */
public class GPUCapacityPlanner {

    record CapacityInput(
        int queriesPerSecond,       // 峰值 QPS
        int avgInputTokens,         // 平均输入 Token
        int avgOutputTokens,        // 平均输出 Token
        String modelId,             // 模型名称
        int targetP99LatencyMs      // 目标 P99 延迟
    ) {}

    /**
     * 计算需要的 GPU 数量
     */
    public GPUCapacity calculate(CapacityInput input) {
        // ===== 模型内存需求 =====
        double modelSizeGB = getModelSizeGB(input.modelId());     // 模型权重
        double kvCachePerReqGB = estimateKVCache(                 // 单个请求的 KV Cache
            input.avgInputTokens() + input.avgOutputTokens(),
            input.modelId()
        );
        double overheadGB = modelSizeGB * 0.2;                    // 框架开销

        // ===== GPU 选型 =====
        GPUSpec gpu = selectGPU(input.modelId());
        // A100-80GB: 80GB, 312 TFLOPS (FP16), $1.5-2/GPU/h
        // H100:      80GB, 756 TFLOPS (FP16), $2-3/GPU/h

        // ===== 吞吐量估算 =====
        int maxConcurrent = (int)((gpu.vramGB() - modelSizeGB - overheadGB)
                                   / kvCachePerReqGB);

        // 每 GPU 每秒能处理的请求数 (基于基准测试)
        double qpsPerGPU = estimateQPSPerGPU(input.modelId(), gpu, input);
        // 简化公式: qpsPerGPU ≈ (gpu.tflops / model.tflopsNeeded) * efficiency_factor

        // ===== 需要的 GPU 数量 (加上 30% 余量) =====
        int gpuCount = (int)Math.ceil(input.queriesPerSecond() / qpsPerGPU * 1.3);
        int nodeCount = (int)Math.ceil((double)gpuCount / gpu.gpusPerNode());

        return new GPUCapacity(
            gpuCount, nodeCount, gpu,
            maxConcurrent * gpuCount,
            qpsPerGPU * gpuCount
        );
    }

    /**
     * 成本估算
     */
    public MonthlyCost estimateMonthlyCost(GPUCapacity capacity) {
        // GPU 云服务器 (如 AWS p4d.24xlarge: 8×A100, ~$32/h)
        double gpuCloudHourly = capacity.nodeCount() * capacity.gpu().cloudPricePerHour();
        double gpuCloudMonthly = gpuCloudHourly * 24 * 30;

        // 自建机房 (GPU 采购 + 运维)
        // A100 80GB: ~$15,000/块 (一次性)
        // H100:       ~$30,000/块 (一次性)
        // 3 年折旧: A100/H100 月均 = $417/833
        double selfHostDepreciation = capacity.gpuCount() * capacity.gpu().purchasePrice() / 36;
        double selfHostMonthly = selfHostDepreciation + 5000;  // + 运维人力/电费

        return new MonthlyCost(
            gpuCloudMonthly,
            selfHostMonthly,
            gpuCloudMonthly - selfHostMonthly > 0
                ? "自有GPU 3年节约: $" + ((gpuCloudMonthly - selfHostMonthly) * 36)
                : "当前量级使用云GPU更划算"
        );
    }
}
```

---

## 6.4 ROI 量化框架

```java
/**
 * AI 产品的 ROI 量化
 */
@Service
public class AIROICalculator {

    /**
     * 量化的三个维度
     */
    public enum ROIDimension {
        COST_SAVING,      // 降本: AI 替代人工或减少处理时间
        REVENUE_GROWTH,   // 增收: AI 驱动的销售提升、转化率提高
        RISK_REDUCTION    // 降风险: AI 审核减少合规罚款、安全事故
    }

    /**
     * ROI 计算
     * 
     * ROI = (收益 - 成本) / 成本 × 100%
     */
    public double calculateROI(AIProject project) {
        double totalBenefit = 0;
        double totalCost = 0;

        // ===== 收益量化 =====
        for (ROIDimension dim : project.dimensions()) {
            switch (dim) {
                case COST_SAVING -> {
                    double hoursSaved = project.monthlyHoursSaved();
                    double hourlyRate = project.averageHourlyRate();
                    totalBenefit += hoursSaved * hourlyRate * 12;  // 年化
                }
                case REVENUE_GROWTH -> {
                    double conversionLift = project.estimatedConversionLift();
                    double annualRevenue = project.relatedAnnualRevenue();
                    totalBenefit += annualRevenue * conversionLift;
                }
                case RISK_REDUCTION -> {
                    double incidentCostAvg = project.avgIncidentCost();
                    double incidentReduction = project.expectedIncidentReduction();
                    totalBenefit += incidentCostAvg * incidentReduction;
                }
            }
        }

        // ===== 成本汇总 =====
        totalCost += project.annualTokenCost();
        totalCost += project.annualInfraCost();
        totalCost += project.annualPersonnelCost();
        totalCost += project.annualToolingCost();

        return totalBenefit > 0 && totalCost > 0
            ? (totalBenefit - totalCost) / totalCost
            : -1;
    }

    /**
     * ROI 量化示例:
     * 
     * 项目: 智能客服 AI Agent
     * 
     * 收益:
     *   - 替代 50% 人工客服: 10 人 × $40,000/年 × 50% = $200,000/年
     *   - 7×24 小时服务 → 减少夜间外包含量: $60,000/年
     *   - 总计: $260,000/年
     * 
     * 成本:
     *   - Token 费用: 50,000 次/天 × $0.005/次 × 365 = $91,250/年
     *   - 基础设施 (向量库 + 推理): $24,000/年
     *   - AI 团队 (3 人): $180,000/年
     *   - 总计: $295,250/年
     * 
     * ROI = (260,000 - 295,250) / 295,250 = -12%
     * 
     * → 第一年可能不赚钱！需要优化成本或增加影响范围
     * 
     * 优化后 (第 2 年):
     *   - 替代率提升到 70%: $280,000/年
     *   - Token 优化后降低 30%: $63,875/年
     *   - 团队降至维护 (2 人): $120,000/年
     *   - 总成本: $207,875/年
     * 
     * ROI = (280,000 - 207,875) / 207,875 = +35%  ✅
     */
}
```

---

## 6.5 成本归因系统

```java
/**
 * 成本归因 —— 知道每一分钱花在哪
 * 
 * CFO 最喜欢的功能：按团队/项目/场景做成本拆分
 */
@Service
public class CostAttribution {

    /**
     * 多维成本归因
     */
    public CostReport generateReport(LocalDate start, LocalDate end) {
        List<CostRecord> records = costRepository.findBetween(start, end);

        // 按团队
        Map<String, Double> byTeam = records.stream()
            .collect(groupingBy(CostRecord::teamId, summingDouble(CostRecord::cost)));

        // 按模型
        Map<String, Double> byModel = records.stream()
            .collect(groupingBy(CostRecord::modelId, summingDouble(CostRecord::cost)));

        // 按场景
        Map<String, Double> byScenario = records.stream()
            .collect(groupingBy(CostRecord::scenario, summingDouble(CostRecord::cost)));

        // 按用户
        Map<String, Double> byUser = records.stream()
            .collect(groupingBy(CostRecord::userId, summingDouble(CostRecord::cost)));

        // 找出成本异常 (单日成本超过均值 3 倍的项目)
        List<CostAnomaly> anomalies = detectAnomalies(records);

        return new CostReport(
            byTeam, byModel, byScenario, byUser,
            sumAll(records), anomalies
        );
    }
}
```

---

## 6.6 省钱最佳实践清单

```yaml
成本优化 10 条铁律:

1. 语义缓存 (Semantic Cache):
   效果: 20-40% 节省
   实现: 向量相似度 > 阈值 → 直接返回缓存

2. 模型分级路由:
   效果: 30-60% 节省
   实现: 简单任务用小模型 (0.1-1% GPT-4的成本)

3. Prompt 精简:
   效果: 10-30% 节省
   实现: 删除重复说明、压缩 Few-shot 示例

4. 上下文窗口管理:
   效果: 20-50% 节省
   实现: 只传相关历史，不要全量对话扔进去

5. 输出长度控制:
   效果: 15-25% 节省
   实现: max_tokens 限制 + "请简洁回答"提示

6. 流式提前终止:
   效果: 5-15% 节省
   实现: 用户取消 / 答案已出现 → 停止

7. 批处理:
   效果: 10-20% 节省 (自建推理)
   实现: 非实时请求合并为 batch

8. 量化部署:
   效果: 50-70% 硬件节省
   实现: FP16 → INT8/INT4 (精度换速度)

9. 选择正确的模型:
   效果: 可达 90% 节省
   实现: 不要什么都用 GPT-4/GPT-4.5

10. 定期审计:
    效果: 持续优化
    实现: 每月看成本报告，找出浪费大户
```

---

> **下一章**：[MLOps 与持续交付](07-mlops-continuous-delivery.md)
