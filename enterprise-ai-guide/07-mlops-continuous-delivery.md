# 第 7 章 · MLOps 与持续交付

---

> "LLM 应用开发 80% 的工作量在 Prompt 迭代和评估，而不是写代码。" MLOps for AI 的核心是 Prompt 版本管理、评估流水线和安全发布。

---

## 7.1 AI MLOps vs 传统 MLOps

| 维度 | 传统 ML MLOps | AI (LLM) MLOps |
|---|---|---|
| **模型训练** | 需要 (从数据训练) | 通常不需要 (用现成模型) |
| **核心迭代对象** | 模型权重 + 特征工程 | **Prompt** + RAG 配置 |
| **版本管理** | 模型版本 + 数据集版本 | **Prompt 版本** + RAG 语料版本 |
| **评估方式** | 指标: Accuracy, F1, AUC | **LLM-as-Judge** + 人工评测 |
| **实验提速** | 训练一次数小时到数天 | 改 Prompt 即可，几分钟验证 |
| **部署方式** | 模型文件 (pkl/pt/onnx) | Prompt 模板 + 配置 |
| **回滚** | 切换模型文件 | **切换 Prompt 版本** |
| **基础设施** | GPU 训练集群 | API Gateway / 推理集群 |

**核心差异**：AI 应用开发的重心从"训练模型"转移到了"设计 Prompt + RAG"。版本管理的主体从模型文件变成了**Prompt 模板和 RAG 配置**。

---

## 7.2 Prompt 的 CI/CD 流水线

```java
/**
 * Prompt CI/CD 流水线
 * 
 * 类比: Git Flow for Prompts
 * 
 * draft → review → staging → canary (10%流量) → production (100%)
 */
@Service
public class PromptCICDPipeline {

    /**
     * Prompt 的 Pull Request 流程
     */
    public void promotePrompt(PromptChangeRequest pr) {
        // ===== Stage 1: 自动化评估 =====
        EvalReport eval = evaluator.runEvaluation(pr.newPrompt());

        if (eval.passRate() < pr.minPassRate()) {
            pr.reject("自动化评估未通过: {}/{}".format(eval.passRate(), pr.minPassRate()));
            return;
        }

        // ===== Stage 2: Staging 环境部署 =====
        deployToStaging(pr.newPrompt());

        // 在 Staging 上跑完整的回归测试
        RegressionReport regression = regressionTester.run(pr.newPrompt());
        if (regression.hasRegression()) {
            rollbackStaging(pr.oldPrompt());
            pr.reject("回归测试发现降级: " + regression.summary());
            return;
        }

        // ===== Stage 3: Canary 发布 (10% 流量) =====
        deployToCanary(pr.newPrompt(), 0.10);

        // 观察 24 小时的核心指标
        CanaryMetrics metrics = monitor.collectCanaryMetrics(Duration.ofHours(24));

        if (metrics.hasAnomaly()) {
            rollbackCanary(pr.oldPrompt());
            pr.reject("Canary 指标异常: " + metrics.anomalies());
            return;
        }

        // ===== Stage 4: 全量发布 =====
        deployToProduction(pr.newPrompt(), 1.0);

        // 记录变更
        promptVersionRepo.save(new PromptVersion(
            pr.promptId(),
            pr.newPrompt(),
            pr.version(),
            pr.author(),
            Instant.now(),
            eval
        ));

        log.info("Prompt {} v{} 全量发布成功!", pr.promptId(), pr.version());
    }
}

/**
 * 自动化评估引擎
 */
@Service
class PromptEvaluator {

    /**
     * 每次 Prompt 变更必须通过的评估
     */
    public EvalReport runEvaluation(PromptTemplate newPrompt) {

        // 1. Golden Dataset 评估 (100+ 条标准测试用例)
        GoldenEvalResult golden = evaluateGoldenDataset(newPrompt);

        // 2. 安全性评估 (越狱、注入、有害内容)
        SafetyEvalResult safety = evaluateSafety(newPrompt);

        // 3. 结构化输出验证 (如果适用)
        StructuredEvalResult structured = evaluateStructuredOutput(newPrompt);

        // 4. 性能评估 (延迟、Token 消耗)
        PerformanceEvalResult perf = evaluatePerformance(newPrompt);

        // 5. 对比旧版本 (确保不比旧版本差)
        PromptTemplate oldPrompt = promptRepo.getLatestProduction();
        ComparisonEvalResult comparison = comparePrompts(oldPrompt, newPrompt);

        return EvalReport.aggregate(
            golden, safety, structured, perf, comparison
        );
    }
}
```

---

## 7.3 评估驱动开发 (Eval-Driven Development)

```java
/**
 * Eval-Driven Development (EDD)
 * 
 * 类比 TDD: 先写测试再写代码
 * EDD: 先写评估标准再写 Prompt
 */
public class EvalDrivenDevelopment {

    /**
     * EDD 循环
     */
    public PromptTemplate developPrompt(PromptSpec spec) {

        // Step 1: 编写评估用例 (Golden Dataset)
        List<TestCase> testCases = createGoldenDataset(spec);
        // 至少 50+ 条覆盖主要场景的测试

        // Step 2: 设定通过标准
        EvalThreshold threshold = new EvalThreshold(
            0.95,  // 功能正确性 >= 95%
            0.80,  // 质量评分 >= 80/100
            0.99,  // 安全通过率 >= 99%
            0.0    // 零退化 (不能比旧版本差)
        );

        // Step 3: 写第 1 版 Prompt
        PromptTemplate v1 = createInitialPrompt(spec);

        // Step 4: 评估 → 分析失败 → 改进 Prompt (迭代循环)
        int iteration = 1;
        while (iteration <= 10) {  // 最多 10 轮
            EvalReport report = evaluator.runEvaluation(v1, testCases);

            if (report.meetsThreshold(threshold)) {
                log.info("Prompt 达标! 迭代 {} 次", iteration);
                return v1;
            }

            // 分析失败用例，指导 Prompt 改进方向
            String feedback = analyzeFailures(report.failures());
            v1 = improvePrompt(v1, feedback);
            iteration++;
        }

        throw new PromptOptimizationException(
            "Prompt 优化 {} 轮后仍未达标", iteration
        );
    }

    /**
     * 自动分析失败用例生成改进建议
     */
    private String analyzeFailures(List<TestCase> failures) {
        StringBuilder analysis = new StringBuilder("以下用例失败，请分析原因:\n");

        for (TestCase tc : failures) {
            analysis.append("---\n");
            analysis.append("输入: ").append(tc.prompt()).append("\n");
            analysis.append("期望: ").append(tc.expectedOutput()).append("\n");
            analysis.append("实际: ").append(tc.actualOutput()).append("\n");
            analysis.append("差异: ").append(tc.diff()).append("\n");
        }

        // 让 LLM 分析失败模式并给出改进建议
        return llm.generate("""
            以下是 AI 输出的失败用例分析，总结所有用例的主要失败原因，
            并给出 Prompt 的具体改进建议。

            %s
            """.formatted(analysis));
    }
}
```

---

## 7.4 A/B 测试与实验平台

```java
/**
 * Prompt A/B 测试框架
 */
@Service
public class ABTestPlatform {

    /**
     * 创建实验
     */
    public Experiment createExperiment(ExperimentConfig config) {
        return new Experiment(
            config.name(),
            // 对照组: 当前生产 Prompt
            new Variant("control", config.currentPrompt(), 0.50),
            // 实验组: 新 Prompt
            new Variant("treatment", config.newPrompt(), 0.50),
            // 关键指标
            config.metrics()
        );
    }

    /**
     * 流量分配
     */
    public Variant assignVariant(String userId, Experiment experiment) {
        // 一致性哈希: 确保同一个用户始终看到同一个版本
        int hash = Math.abs(userId.hashCode());
        double bucket = (hash % 10000) / 10000.0;

        double cumulative = 0;
        for (Variant variant : experiment.variants()) {
            cumulative += variant.trafficPercent();
            if (bucket <= cumulative) {
                return variant;
            }
        }
        return experiment.control();  // fallback
    }

    /**
     * 结果分析
     */
    public ExperimentResult analyzeResults(Experiment experiment, Duration duration) {
        // 收集指标数据
        Map<Variant, MetricSnapshot> snapshots = new HashMap<>();
        for (Variant variant : experiment.variants()) {
            snapshots.put(variant, metricsCollector.collect(variant, duration));
        }

        Variant control = experiment.control();
        Variant treatment = experiment.treatment();

        // 统计显著性检验
        for (String metric : experiment.metrics()) {
            double controlValue = snapshots.get(control).get(metric);
            double treatmentValue = snapshots.get(treatment).get(metric);
            double pValue = tTest(controlValue, treatmentValue);

            if (pValue < 0.05) {
                log.info("指标 {}: 有统计显著差异 (p={:.4f})", metric, pValue);
            }
        }

        return new ExperimentResult(snapshots, experiment);
    }
}
```

---

## 7.5 发布策略

```json
{
  "release_strategies": {
    "canary": {
      "description": "金丝雀发布 - 最安全",
      "traffic_split": "10% → 25% → 50% → 100%",
      "monitoring_duration": "每阶段至少 24 小时",
      "triggers": [
        "错误率上升 > 1% → 立即回滚",
        "P99 延迟增加 > 50% → 暂停扩大",
        "用户 👎 比例上升 > 20% → 回滚"
      ]
    },
    "blue_green": {
      "description": "蓝绿部署 - 适合重大变更",
      "approach": "两套完整环境，一键切换",
      "switch_time": "< 1 分钟",
      "rollback_time": "< 1 分钟"
    },
    "feature_flag": {
      "description": "特性开关 - 按用户/租户灰度",
      "approach": "代码中嵌入开关，无需重新部署",
      "granularity": "按用户 ID、租户、部门、百分比"
    },
    "shadow": {
      "description": "影子模式 - 新版本不返回用户，只记录差异",
      "use_case": "高风险变更的最终验证",
      "advantage": "零用户影响"
    }
  }
}
```

---

## 7.6 Prompt 版本管理规范

```yaml
Prompt 版本管理约定:

目录结构:
  prompts/
    ├── customer_service/
    │   ├── v1.0.0.yaml          # 初始生产版本
    │   ├── v1.1.0-draft.yaml    # 开发中的版本
    │   ├── v1.1.0-staging.yaml  # 评估通过的版本
    │   └── evaluations/
    │       ├── v1.1.0-eval.json # 评估报告
    │       └── v1.1.0-compare.json # 对比报告

版本号规则 (SemVer):
  - MAJOR: 对话策略根本性变化 (如: 从单轮变多轮)
  - MINOR: 新增功能、优化质量 (如: 增加新的处理规则)
  - PATCH: 修复 Bug、措辞微调

每个版本文件必须包含:
  version: 1.1.0
  author: byron
  date: 2025-05-21
  changelog: |
    - 新增: 退款流程引导
    - 优化: 缩短了回复长度(平均减少 30 token)
    - 修复: 特定场景下错误拒绝退货的问题
  eval_report: evaluations/v1.1.0-eval.json
  rollback_to: v1.0.0         # 遇到问题时的回滚目标
```

---

> **下一章**：[可观测性与运维](08-observability-operations.md)
