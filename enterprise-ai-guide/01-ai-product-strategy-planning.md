# 第 1 章 · AI 产品战略与规划

---

> 企业 AI 的第一步不是选模型，而是回答一个核心问题：**这个 AI 产品到底解决什么商业问题？**

---

## 1.1 AI 战略四象限

企业 AI 投入可以分为四种类型，各有不同的 ROI 逻辑：

```
                    高 ← 业务影响程度 → 低
              ┌──────────────────┬──────────────────┐
        高    │  🏆 战略型 AI     │  🌱 探索型 AI     │
              │  核心业务差异化    │  未来竞争力布局    │
   AI 成熟度  │  例：智能客服系统   │  例：AI Agent 工厂  │
              │  投资：百万级/年    │  投资：小团队试水    │
              ├──────────────────┼──────────────────┤
        低    │  ⚡ 增效型 AI     │  ❌ 暂缓型 AI     │
              │  替代重复性工作    │  ROI 尚不明确      │
              │  例：代码助手/Copilot│  例：通用AI聊天    │
              │  投资：工具订阅    │  投资：0          │
              └──────────────────┴──────────────────┘
```

### 四象限决策指南

| 象限 | 策略 | 典型投资 | 时间线 |
|---|---|---|---|
| **战略型** (高影响 × 高成熟) | 重投入，建壁垒 | 百万级/年，自研团队 | 6-12 月见效果 |
| **增效型** (高影响 × 低成熟) | 快速导入，立竿见影 | 工具订阅 + 少量定制 | 1-3 月见效 |
| **探索型** (低影响 × 高成熟) | 小团队先行，储备能力 | 2-3 人试水 | 持续迭代 |
| **暂缓型** (低影响 × 低成熟) | 不做或观察 | 0 | 等待技术成熟 |

---

## 1.2 场景优先级矩阵

不是所有场景都适合 AI。以下是评分框架：

```java
/**
 * AI 场景优先级评估器
 * 
 * 每个潜在场景在 5 个维度打分 (1-5)，按总分排序
 */
public class AIScenarioPrioritizer {

    public record ScenarioScore(
        String scenario,
        int businessValue,       // 业务价值：省多少成本 / 增多少收入
        int feasibility,         // 可行性：技术是否成熟
        int dataReadiness,       // 数据就绪度：有没有数据、质量如何
        int userTolerance,       // 容错率：用户能接受多少错误 (5=高容错)
        int differentiation,     // 差异化：是不是竞争对手做不到的
        int totalScore           // 总分
    ) {}

    /**
     * 评估示例：企业常见 AI 场景
     */
    public List<ScenarioScore> evaluate() {
        return List.of(
            // 高优先级
            new ScenarioScore("内部知识库问答",   5, 5, 4, 5, 2, 21),
            new ScenarioScore("代码生成/Review",  5, 5, 5, 4, 2, 21),
            new ScenarioScore("客服工单自动分类",  4, 5, 4, 4, 2, 19),

            // 中优先级
            new ScenarioScore("合同条款审查",     5, 4, 3, 2, 4, 18),  // 容错率低
            new ScenarioScore("营销文案生成",     3, 4, 4, 5, 2, 18),
            new ScenarioScore("日志异常检测",     4, 4, 5, 4, 1, 18),

            // 低优先级 (谨慎)
            new ScenarioScore("完全自动化财务审核", 5, 2, 3, 1, 5, 16), // 容错率极低
            new ScenarioScore("医疗诊断辅助",     5, 3, 2, 1, 5, 16),   // 合规风险高
            new ScenarioScore("全自动法律文书生成", 4, 2, 3, 1, 5, 15)  // 准确性要求极高
        );
    }

    /**
     * AI 适配度的核心判断法则
     */
    public boolean isGoodFit(ScenarioScore score) {
        // 法则 1: 容错率必须足够高
        if (score.userTolerance() < 3) {
            return false;  // 不能接受错误的场景 (金融交易、医疗诊断) 需要 HITL
        }

        // 法则 2: 数据和可行性至少有一个高
        if (score.dataReadiness() < 3 && score.feasibility() < 3) {
            return false;
        }

        // 法则 3: 有明确的业务价值
        if (score.businessValue() < 3) {
            return false;
        }

        return true;
    }
}
```

**黄金法则**：第一场景选择 = 高业务价值 × 高容错率 × 数据就绪 × 技术可行。

最佳第一场景通常不是"最酷的"，而是"最安全的"——**内部知识库问答、代码生成、客服工单分类**。

---

## 1.3 Make vs Buy 决策框架

```java
/**
 * Make vs Buy 决策引擎
 */
public class MakeVsBuyDecision {

    public enum Strategy { BUILD, BUY, HYBRID, WAIT }

    /**
     * 决策树
     */
    public Strategy decide(AICapability capability) {
        // ===== BUY (采购) 的条件 =====
        if (capability.isCommodity()) {
            // 通用能力 → 直接采购
            // 例: 代码补全 (GitHub Copilot)、通用聊天 (ChatGPT)
            return Strategy.BUY;
        }

        if (!capability.isCoreDifferentiator()) {
            // 非差异化能力 → 采购
            // 例: 语音识别、OCR、通用翻译
            return Strategy.BUY;
        }

        // ===== BUILD (自研) 的条件 =====
        if (capability.isCoreDifferentiator() && capability.hasInHouseExpertise()) {
            // 核心差异化 + 有团队能力 → 自研
            // 例: 行业专属的知识图谱 + RAG、定制化 Agent 工作流
            return Strategy.BUILD;
        }

        // ===== HYBRID (混合) =====
        if (capability.isCoreDifferentiator() && !capability.hasInHouseExpertise()) {
            // 核心差异化但团队能力不足 → 混合模式
            // 例: 基于开源模型 Fine-tuning + 外部顾问
            return Strategy.HYBRID;
        }

        // ===== WAIT =====
        return Strategy.WAIT;
    }
}

/**
 * 典型企业 AI 能力的 Make vs Buy 决策表
 * 
 * ┌─────────────────────────┬──────────┬───────────────┐
 * │ AI 能力                  │ 决策     │ 说明           │
 * ├─────────────────────────┼──────────┼───────────────┤
 * │ 代码补全/助手            │ BUY      │ Copilot/Cursor │
 * │ 通用聊天/写作            │ BUY      │ ChatGPT/Claude │
 * │ 图像/语音识别            │ BUY      │ 云服务 API     │
 * │ 文档 OCR                │ BUY      │ Azure/Google   │
 * ├─────────────────────────┼──────────┼───────────────┤
 * │ 行业知识库 RAG           │ BUILD    │ 核心差异化     │
 * │ 定制化 Agent 工作流       │ BUILD    │ 业务流程特有   │
 * │ 内部数据 Fine-tuning     │ BUILD    │ 数据是壁垒     │
 * ├─────────────────────────┼──────────┼───────────────┤
 * │ 大模型 Fine-tuning       │ HYBRID   │ 开源模型+顾问  │
 * │ AI 安全/合规审查         │ HYBRID   │ 工具+自研规则  │
 * └─────────────────────────┴──────────┴───────────────┘
 */
```

---

## 1.4 云端 API vs 自建推理

这是企业 AI 决策中最常见也最容易出错的选择：

```
                    云端 API (OpenAI/Claude/Azure)  vs  自建推理 (GPU 集群 + 开源模型)

成本结构:            按 Token 付费，零固定成本             高固定成本(硬件+运维)，低边际成本
                    月费 = 请求量 × 单价                  月费 = GPU租金/折旧 + 电费 + 人力

适用量级:            低~中请求量 (< 1000 req/min)        高请求量 (> 1000 req/min)
                                                或 对延迟有极致要求 (< 100ms)

安全合规:            数据传出（部分公司不允许）             数据完全本地

模型控制:            无法 Fine-tune (除 Ada/微调API)     完全控制

运维负担:            零                                    需要 ML 工程师团队

SLA:                 依赖云服务商                          需自己保障

推荐策略:            起步阶段——先用云端 API 验证 PMF
                    规模化后——评估自建的经济拐点
```

```java
/**
 * 成本交叉点计算器
 * 
 * 帮助判断在什么请求量下，自建比云端更便宜
 */
public class CostBreakEvenCalculator {

    /**
     * @param cloudPricePer1k   云端 API 每 1000 token 价格 (美元)
     * @param avgTokensPerReq   每次请求平均 token 数 (输入+输出)
     * @param selfHostMonthly   自建月成本 (GPU + 运维 + 人力)
     * @param selfHostQPS       自建集群每秒能处理的请求数
     * @return 月请求量临界点
     */
    public long calculateBreakEven(
            double cloudPricePer1k,
            int avgTokensPerReq,
            double selfHostMonthly,
            int selfHostQPS) {

        // 云端单次请求成本
        double cloudCostPerReq = (avgTokensPerReq / 1000.0) * cloudPricePer1k;

        // 自建集群月处理能力
        long selfHostMonthlyCapacity = selfHostQPS * 3600L * 24 * 30;

        // 临界点: 云端月费 = 自建月费
        // cloudCostPerReq × N = selfHostMonthly
        // N = selfHostMonthly / cloudCostPerReq
        long breakEvenRequests = (long)(selfHostMonthly / cloudCostPerReq);

        // 还需要确保临界点不超过自建集群的容量
        if (breakEvenRequests > selfHostMonthlyCapacity) {
            // 超过容量 → 需要更多 GPU，自建成本要重新算
            return -1;  // 自建不划算
        }

        return breakEvenRequests;
    }

    /**
     * 示例计算:
     * 
     * 假设:
     * - 云端 GPT-4: $0.03/1K input, $0.06/1K output
     * - 平均请求: 2000 input + 500 output = 2500 token
     * - 单价约: $0.10/1K (平均)
     * - 单次请求成本: 2500/1000 × 0.10 = $0.25
     * 
     * 自建 LLaMA 3.1 70B:
     * - 4 × A100 (80GB): ~$15,000/月 租金
     * - ML 工程师: ~$15,000/月
     * - 运维/电费: ~$3,000/月
     * - 自建月成本: ~$33,000/月
     * 
     * 临界点: $33,000 / $0.25 = 132,000 次请求/月 ≈ 4,400 次/天
     * 
     * 结论:
     *   < 4,000 次/天 → 云端 API 更便宜
     *   4,000-10,000 次/天 → 交叉区间，值得评估
     *   > 10,000 次/天 → 自建更便宜（且数据安全可控）
     */
}
```

---

## 1.5 AI 产品路线图模板

```java
/**
 * 企业 AI 产品分阶段路线图
 * 
 * Phase 0 → Phase 1 → Phase 2 → Phase 3
 */
public enum AILandingPhases {

    // ===== Phase 0: 基础准备 (1-3 个月) =====
    PHASE_0("基础设施与团队"),
    // 目标: 建成最小可行 AI 能力
    // 交付:
    //   ☐ AI 网关/平台选型并部署
    //   ☐ 模型访问权限 (API Key / 本地部署)
    //   ☐ 核心团队到位 (2-3 人)
    //   ☐ 数据合规审查通过
    //   ☐ 1 个内部试点场景启动

    // ===== Phase 1: 快速见效 (1-3 个月) =====
    PHASE_1("快速见效"),
    // 目标: 交付 1-2 个高价值、低风险场景
    // 交付:
    //   ☐ 场景 A 上线 (如内部知识库问答)
    //   ☐ 用户反馈循环建立
    //   ☐ Token 用量和成本基线确立
    //   ☐ 基础监控面板上线
    // 典型场景: RAG 问答、代码助手、文档摘要

    // ===== Phase 2: 平台化 (3-6 个月) =====
    PHASE_2("平台化与规模化"),
    // 目标: 从单场景扩展为 AI 平台
    // 交付:
    //   ☐ AI Gateway 标准化 (统一鉴权、限流、路由)
    //   ☐ Prompt 版本管理 + A/B 测试能力
    //   ☐ 3-5 个场景在生产环境运行
    //   ☐ Fine-tuning 能力就绪
    //   ☐ MLOps 流水线搭建完成
    //   ☐ 成本归因系统 (按部门/项目分摊)

    // ===== Phase 3: 深度智能化 (6-12 个月) =====
    PHASE_3("深度智能化与 Agent"),
    // 目标: AI 成为核心业务能力
    // 交付:
    //   ☐ Agent 工作流上线 (多步推理 + 工具调用)
    //   ☐ 定制化 Fine-tuned 模型替代通用模型
    //   ☐ 推荐/决策类 AI 功能上线
    //   ☐ 自建推理集群 (如果量级达标)
    //   ☐ AI 驱动的业务指标可量化提升
}
```

---

## 1.6 关键决策清单

```yaml
企业 AI 产品启动前的 10 个决策:

1. 场景选择:
   - [ ] 第一个 AI 场景是否满足"高价值 × 高容错"？
   - [ ] 该场景有清晰的成功指标吗？(如: 客服工单自动处理率 > 40%)
   
2. Make vs Buy:
   - [ ] 这个能力是差异化核心还是通用能力？
   - [ ] 如果自研，团队有足够的 AI 工程能力吗？

3. 部署模式:
   - [ ] 数据能离开企业内网吗？
   - [ ] 请求量是否达到自建的经济转折点？

4. 组织准备:
   - [ ] AI 项目有明确的执行负责人吗（非挂名）？
   - [ ] 业务方有专人对接 AI 团队吗？

5. 投入承诺:
   - [ ] 第一年预算已获批吗？（通常 50-200 万起）
   - [ ] 领导层接受 3-6 个月的探索期（可能无直接产出）吗？
```

---

> **下一章**：[企业 AI 架构设计](02-enterprise-ai-architecture.md)
