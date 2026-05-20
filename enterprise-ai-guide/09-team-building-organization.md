# 第 9 章 · 团队建设与组织变革

---

> 技术问题通常能找到答案，人的问题才是真正的瓶颈。AI 转型最大的挑战不是模型，而是**组织**。

---

## 9.1 AI 团队角色矩阵

```java
/**
 * 企业 AI 团队的核心角色
 * 
 * 不一定要每个人各司其职——初期可以一人多角色
 */
public class AITeamRoles {

    public enum Role {
        // ===== 业务侧 =====
        AI_PRODUCT_MANAGER,     // AI 产品经理: 定义场景、衡量价值
        DOMAIN_EXPERT,          // 领域专家: 律师、医生、资深客服...定义"好"的答案
        DATA_ANNOTATOR,         // 数据标注员: 构建 Golden Dataset

        // ===== 技术侧 =====
        AI_ENGINEER,            // AI 工程师: Prompt 设计、RAG 搭建、Agent 开发
        ML_ENGINEER,            // ML 工程师: Fine-tuning、模型部署、推理优化
        PLATFORM_ENGINEER,      // 平台工程师: AI Gateway、MLOps 平台、监控告警
        DATA_ENGINEER,          // 数据工程师: 知识库构建、数据管道、数据质量

        // ===== 治理侧 =====
        AI_SAFETY_ENGINEER,     // AI 安全工程师: 护栏设计、注入防御、内容审核
        AI_ETHICS_OFFICER,      // AI 伦理官: 偏见审计、合规审查、负责任 AI

        // ===== 领导侧 =====
        HEAD_OF_AI,             // AI 负责人: 战略、资源、跨部门协调
        AI_TECH_LEAD            // AI 技术负责人: 架构决策、技术方向
    }

    /**
     * 不同阶段需要的团队规模
     */
    public record TeamComposition(
        Phase phase,
        int aiEngineer,
        int mlEngineer,
        int platformEngineer,
        int dataEngineer,
        int pm,
        int domainExpert,
        int safety
    ) {}

    public static Map<Phase, TeamComposition> recommendedTeam() {
        return Map.of(
            // Phase 0: 基础准备
            Phase.PHASE_0, new TeamComposition(Phase.PHASE_0,
                2, 1, 1, 1, 1, 1, 0),

            // Phase 1: 1-2 场景上线
            Phase.PHASE_1, new TeamComposition(Phase.PHASE_1,
                3, 2, 1, 2, 1, 2, 1),

            // Phase 2: 平台化
            Phase.PHASE_2, new TeamComposition(Phase.PHASE_2,
                4, 2, 2, 2, 2, 3, 1),

            // Phase 3: 深度智能化
            Phase.PHASE_3, new TeamComposition(Phase.PHASE_3,
                5, 3, 3, 3, 3, 4, 2)
        );
    }
}
```

---

## 9.2 组织模式

```
三种主流 AI 团队组织模式:

模式 1: 集中式 (Center of Excellence)
  ┌─────────────────────────────────────────┐
  │             AI 中台 (CoE)                │
  │   为所有业务线提供 AI 能力               │
  └────┬──────────┬──────────┬──────────────┘
       │          │          │
       ▼          ▼          ▼
    业务A       业务B       业务C

  优点: 资源共享、标准统一、避免重复造轮子
  缺点: 离业务远、响应慢、容易变成瓶颈
  适合: 初期探索阶段、跨业务线协同


模式 2: 嵌入式 (Embedded)
  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │ 业务A     │  │ 业务B     │  │ 业务C     │
  │ +1 AI Eng│  │ +1 AI Eng│  │ +1 AI Eng│
  └──────────┘  └──────────┘  └──────────┘

  优点: 贴近业务、响应快、更懂场景
  缺点: 人才分散、标准不统一、容易重复
  适合: 业务差异化大、需要深度定制


模式 3: 混合式 (Federated) — 推荐!
  ┌─────────────────────────────────────────┐
  │          AI 平台团队 (10-15 人)          │
  │   Gateway/Monitoring/MlOps/Cost/基础RAG  │
  └──────────────────┬──────────────────────┘
                     │ 赋能
  ┌──────────┐ ┌──────────┐ ┌──────────────┐
  │ 业务A AI  │ │ 业务B AI  │ │ 业务C AI     │
  │ 小队 2-3人│ │ 小队 2-3人│ │ 小队 2-3人   │
  │ (嵌入式)  │ │ (嵌入式)  │ │ (嵌入式)     │
  └──────────┘ └──────────┘ └──────────────┘

  优点: 平台统一 + 业务灵活
  缺点: 需要协调 (但比前两种好管理)
  适合: 绝大多数中型以上企业
```

---

## 9.3 AI 工程师技能矩阵

```java
/**
 * AI 工程师的 T 型能力模型
 */
public class AIEngineerSkillMatrix {

    /**
     * 核心能力评估
     */
    public static Map<String, SkillRequirement> requiredSkills() {
        return Map.of(
            // ===== 必须掌握 =====
            "LLM 工作原理", new SkillRequirement("必须", "★★★★★",
                "理解 Transformer, Tokenization, Attention, Context Window"),
            "Prompt Engineering", new SkillRequirement("必须", "★★★★★",
                "设计 Prompt、Few-shot、Chain-of-Thought、结构化输出"),
            "RAG 架构", new SkillRequirement("必须", "★★★★★",
                "Chunking 策略、向量检索、混合检索、Re-ranking"),

            // ===== 应该掌握 =====
            "Python (或 Java + AI 框架)", new SkillRequirement("应当", "★★★★",
                "生产代码能力，不是 Jupyter Notebook 就好"),
            "评估与测试", new SkillRequirement("应当", "★★★★",
                "构建 Golden Dataset、LLM-as-Judge、A/B 测试"),
            "API 设计与集成", new SkillRequirement("应当", "★★★★",
                "REST/GraphQL、流式传输、错误处理"),

            // ===== 加分项 =====
            "Agent 开发", new SkillRequirement("加分", "★★★",
                "Function Calling、ReAct、Tool Orchestration"),
            "Fine-tuning", new SkillRequirement("加分", "★★★",
                "LoRA/QLoRA、数据准备、超参调优"),
            "向量数据库", new SkillRequirement("加分", "★★★",
                "Milvus/Pinecone/Weaviate 的运维"),
            "MLOps", new SkillRequirement("加分", "★★★",
                "CI/CD for Prompts、模型部署、监控"),

            // ===== 可选 =====
            "深度学习框架", new SkillRequirement("可选", "★★",
                "PyTorch (不需要精通，够用就行)"),
            "分布式训练", new SkillRequirement("可选", "★★",
                "DeepSpeed/FSDP (只在做大规模 Fine-tuning 时需要)")
        );
    }

    /**
     * 招聘要点
     * 
     * 好的 AI 工程师 != 好的 ML 研究员
     * 
     * AI 工程师需要更多的是:
     *   1. 工程能力 (不是调参)
     *   2. 产品思维 (不是发 paper)
     *   3. 系统设计 (不是只有模型)
     * 
     * 面试应该侧重:
     *   1. 设计一个 RAG 系统 (系统设计题)
     *   2. 给定一个场景，设计 Prompt 策略 (产品思维题)
     *   3. 如何评估一个 LLM 的回答质量 (评估思维题)
     */
}
```

---

## 9.4 组织变革管理

```java
/**
 * AI 转型的变革管理
 * 
 * 技术部署 ≠ 成功落地。没有组织变革管理，AI 项目必然失败。
 */
public class ChangeManagement {

    /**
     * 利益相关者地图
     */
    public enum StakeholderGroup {
        EXECUTIVE_SPONSOR,   // 高层支持者: 提供预算和政治庇护
        CHAMPIONS,           // 内部布道者: 早期采用者，帮忙推广
        SKEPTICS,            // 怀疑者: "AI 会取代我们吗？"
        USERS,               // 最终用户: 实际使用 AI 工具的人
        AFFECTED             // 受影响者: 工作流程被 AI 改变的人
    }

    /**
     * 变革管理的 8 步法 (Kotter Model 适配 AI 转型)
     */
    public class AIChangeManagement {

        // Step 1: 制造紧迫感
        public String createUrgency() {
            return "展示行业案例: 竞争对手已经在用 AI 客服，响应速度是我们的 10 倍";
        }

        // Step 2: 组建引导联盟
        public String buildCoalition() {
            return "找到 CTO + 业务 VP + 关键团队 Lead 的支持";
        }

        // Step 3: 明确愿景
        public String defineVision() {
            return "不是 '我们要用 AI'，而是 '我们要让客服响应时间从 30 分钟降到 1 分钟'";
        }

        // Step 4: 沟通愿景
        public String communicateVision() {
            return """
                全员宣讲: AI 的目标是增强而非替代
                核心信息:
                - AI 会替代重复性工作 → 解放你去做更有创造性的事
                - 第一批 AI 工具会优先给最忙的团队
                - 有完整的培训计划，不用害怕学不会
                """;
        }

        // Step 5: 移除障碍
        public String removeObstacles() {
            return "为早期采用者提供: 专属支持、快速反馈通道、可见的激励";
        }

        // Step 6: 创造短期胜利
        public String createQuickWins() {
            return """
                第 1 个月的胜利:
                - 客服团队: AI 自动回答了 40% 的常见问题 → 省出时间处理复杂案例
                - 数据在这个团队公开分享
                """;
        }

        // Step 7: 巩固成果，推进变革
        public String sustainAcceleration() {
            return "不要庆祝一次胜利就停——把成功经验复制到下一个团队";
        }

        // Step 8: 将变革融入文化
        public String anchorChange() {
            return """
                AI 能力成为:
                - 新员工入职培训的一部分
                - 晋升评估的考量维度
                - 季度 OKR 的常见主题
                """;
        }
    }
}
```

---

## 9.5 常见组织陷阱

```java
/**
 * AI 转型中最常见的组织错误
 */
public class OrganizationAntiPatterns {

    public void documentPitfalls() {

        pitfall("AI 团队挂在 IT 部门下，远离业务");
        // 后果: 做的都是"酷炫但没人用"的功能
        // 解决: AI 团队应该直接对 CTO/CEO 汇报，嵌入业务线

        pitfall("招聘要求 '5 年深度学习经验' ");
        // 后果: 找不到人（或者找到的人只会训模型不会做产品）
        // 解决: 招聘 AI 工程师 (工程+产品思维) 而非 ML 研究员

        pitfall("没有领域专家参与");
        // 后果: AI 输出的答案技术上对、业务上错
        // 解决: 每个 AI 项目必须配 1+ 领域专家 (律师、医生、资深客服)

        pitfall("把 AI 当成纯 IT 项目");
        // 后果: 技术上线了，没人用
        // 解决: AI 是业务变革项目，需要变革管理

        pitfall("所有人同时用 AI 做所有事");
        // 后果: 资源分散，哪个都做不好
        // 解决: 选 1-2 个场景聚焦，成功后复制
    }
}
```

---

> **下一章**：[落地路线图与实战案例](10-implementation-roadmap-cases.md)
