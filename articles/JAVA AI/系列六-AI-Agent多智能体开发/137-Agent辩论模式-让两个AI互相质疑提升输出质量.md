# Agent 辩论模式：让两个 AI 互相质疑提升输出质量，两个人吵架比你一个人想得全面

## 前言

你有没有这种感觉：让 AI 单次生成一个方案，出来的结果总觉得"还行，但差点意思"？然后你反复地让它"再优化一下""再想想有没有遗漏"，它确实能改，但始终是基于自己的思路修修补补，很难跳出思维定式。

我在做代码设计评审的 AI 工具时，灵光一闪想到了**辩论模式**：与其让一个 AI 反复修改自己的方案，不如让两个 AI 互相辩论。一个负责生成，一个负责挑刺，再来一个当裁判。三个臭皮匠顶个诸葛亮，更何况是三个 AI。

效果震惊了我：用辩论模式做代码设计评审，发现的问题数比单 Agent 模式多了 67%，而且遗漏率从 32% 降到了 8%。这篇文章就把辩论模式的原理和完整 Java 实现分享出来。

## 一、辩论模式的原理

### 1.1 核心思想

传统的"迭代优化"模式本质上是**一个人反复审视自己的方案**：

```
你 → 生成方案 → 自己检查 → 不满意 → 修改 → 再检查 → ...
```

问题是：人会犯错，AI 也会。AI 的思维方式受限于它自己的训练数据和 Prompt，在同一个对话上下文中，它很难真正"跳出自己的框框"。

辩论模式引入了**对抗性思维**：

```
生成者 → 产出初版方案
    ↓
批评者 → 找出方案的缺陷和盲区
    ↓
生成者 → 根据批评改进方案
    ↓
批评者 → 再次挑战改进后的方案
    ↓ ... 多轮辩论 ...
    ↓
裁判者 → 汇总双方论点，给出最终评估
```

三个 Agent，各司其职：
- **生成 Agent（Generator）**：立场是"我想出一个好方案"，目标是把方案做完整
- **批评 Agent（Critic）**：立场是"我来找茬"，目标是发现方案的漏洞、边界、风险
- **裁判 Agent（Judge）**：立场是"我来做最终判决"，目标是综合评估方案质量

### 1.2 为什么辩论模式有效？

这和人类的"红蓝对抗"、"同行评审"是一个道理：

1. **视角多元化**：生成者关注"怎么做对"，批评者关注"哪里做错"，互补覆盖盲区
2. **对抗性激发深度思考**：批评者的质疑迫使生成者考虑更全面，不是简单地"优化一下"而是"回应挑战"
3. **多轮迭代有方向**：每次辩论都有明确的"焦点议题"，不会像单 Agent 迭代那样漫无边际

### 1.3 辩论流程架构

```
轮次1:
  Generator → 生成初始方案 S1
  Critic    → 评审 S1，指出问题 P1

轮次2:
  Generator → 根据 P1 修正方案 → S2
  Critic    → 评审 S2，指出新问题 P2

轮次3:
  Generator → 根据 P2 修正方案 → S3
  Critic    → 评审 S3

...
辩论终止条件触发（达到最大轮次/双方达成共识/改进幅度收敛）
...

Judge → 综合所有轮次的方案和批评 → 最终评估报告
```

## 二、辩论引擎的 Java 实现

### 2.1 核心模型

```java
import java.util.*;
import java.time.Instant;

/**
 * 一轮辩论的记录
 */
public class DebateRound {
    private int roundNumber;
    private String generatorOutput;     // 生成者本轮输出
    private String criticFeedback;      // 批评者本轮反馈
    private List<String> issuesFound;   // 批评者发现的问题
    private double qualityScore;        // 批评者给出的质量评分 (0-100)
    private long timestamp;

    public DebateRound(int roundNumber) {
        this.roundNumber = roundNumber;
        this.timestamp = System.currentTimeMillis();
    }

    // Getters & Setters
    public int getRoundNumber() { return roundNumber; }
    public String getGeneratorOutput() { return generatorOutput; }
    public void setGeneratorOutput(String generatorOutput) { this.generatorOutput = generatorOutput; }
    public String getCriticFeedback() { return criticFeedback; }
    public void setCriticFeedback(String criticFeedback) { this.criticFeedback = criticFeedback; }
    public List<String> getIssuesFound() { return issuesFound; }
    public void setIssuesFound(List<String> issuesFound) { this.issuesFound = issuesFound; }
    public double getQualityScore() { return qualityScore; }
    public void setQualityScore(double qualityScore) { this.qualityScore = qualityScore; }
    public long getTimestamp() { return timestamp; }
}

/**
 * 辩论主题
 */
public class DebateTopic {
    private String topicId;
    private String title;
    private String description;
    private String domain;              // 领域：code_design, architecture, algorithm 等
    private Map<String, Object> context; // 上下文信息

    public DebateTopic(String title, String description) {
        this.topicId = UUID.randomUUID().toString();
        this.title = title;
        this.description = description;
        this.context = new HashMap<>();
    }

    public String getTopicId() { return topicId; }
    public String getTitle() { return title; }
    public String getDescription() { return description; }
    public Map<String, Object> getContext() { return context; }
    public void addContext(String key, Object value) { context.put(key, value); }
}

/**
 * 辩论结果
 */
public class DebateResult {
    private String topicId;
    private List<DebateRound> rounds;
    private String finalProposal;       // 最终方案
    private String judgeVerdict;        // 裁判判决
    private double finalScore;
    private List<String> unresolvedIssues;
    private long totalTokensUsed;
    private long durationMs;

    public DebateResult() {
        this.rounds = new ArrayList<>();
        this.unresolvedIssues = new ArrayList<>();
    }

    // Getters & Setters
    public String getTopicId() { return topicId; }
    public void setTopicId(String topicId) { this.topicId = topicId; }
    public List<DebateRound> getRounds() { return rounds; }
    public void addRound(DebateRound round) { rounds.add(round); }
    public String getFinalProposal() { return finalProposal; }
    public void setFinalProposal(String finalProposal) { this.finalProposal = finalProposal; }
    public String getJudgeVerdict() { return judgeVerdict; }
    public void setJudgeVerdict(String judgeVerdict) { this.judgeVerdict = judgeVerdict; }
    public double getFinalScore() { return finalScore; }
    public void setFinalScore(double finalScore) { this.finalScore = finalScore; }
    public List<String> getUnresolvedIssues() { return unresolvedIssues; }
    public long getTotalTokensUsed() { return totalTokensUsed; }
    public void setTotalTokensUsed(long totalTokensUsed) { this.totalTokensUsed = totalTokensUsed; }
    public long getDurationMs() { return durationMs; }
    public void setDurationMs(long durationMs) { this.durationMs = durationMs; }
}
```

### 2.2 辩论配置

```java
public class DebateConfig {
    private int maxRounds = 5;            // 最大辩论轮次
    private double qualityThreshold = 85.0; // 质量达标线
    private double improvementThreshold = 5.0; // 改进幅度收敛阈值
    private boolean enableJudge = true;   // 是否启用裁判
    private long roundTimeoutMs = 60000;  // 每轮超时时间

    public DebateConfig() {}

    public DebateConfig maxRounds(int n) { this.maxRounds = n; return this; }
    public DebateConfig qualityThreshold(double t) { this.qualityThreshold = t; return this; }
    public DebateConfig improvementThreshold(double t) { this.improvementThreshold = t; return this; }
    public DebateConfig enableJudge(boolean b) { this.enableJudge = b; return this; }
    public DebateConfig roundTimeout(long ms) { this.roundTimeoutMs = ms; return this; }

    public int getMaxRounds() { return maxRounds; }
    public double getQualityThreshold() { return qualityThreshold; }
    public double getImprovementThreshold() { return improvementThreshold; }
    public boolean isEnableJudge() { return enableJudge; }
    public long getRoundTimeoutMs() { return roundTimeoutMs; }
}
```

### 2.3 辩论引擎核心

```java
import java.util.concurrent.*;

public class DebateEngine {
    private final LLMService llmService;
    private final DebateConfig config;
    private final DebateResult result;
    private long totalTokens = 0;

    public DebateEngine(LLMService llmService, DebateConfig config) {
        this.llmService = llmService;
        this.config = config;
        this.result = new DebateResult();
    }

    /**
     * 执行辩论
     */
    public DebateResult debate(DebateTopic topic) {
        long startTime = System.currentTimeMillis();
        result.setTopicId(topic.getTopicId());

        System.out.println("\n==========================================");
        System.out.println("辩论开始: " + topic.getTitle());
        System.out.println("最大轮次: " + config.getMaxRounds());
        System.out.println("质量达标线: " + config.getQualityThreshold());
        System.out.println("==========================================\n");

        String currentProposal = null;
        double previousScore = 0;

        for (int round = 1; round <= config.getMaxRounds(); round++) {
            System.out.println("--- 第 " + round + " 轮辩论 ---");

            DebateRound debateRound = new DebateRound(round);

            try {
                // 阶段1: Generate（生成）
                String generatorPrompt = buildGeneratorPrompt(topic, currentProposal,
                    result.getRounds(), round);
                CompletableFuture<String> genFuture = CompletableFuture.supplyAsync(
                    () -> llmService.generate(generatorPrompt));

                String generatorOutput = genFuture.get(config.getRoundTimeoutMs(), TimeUnit.MILLISECONDS);
                debateRound.setGeneratorOutput(generatorOutput);
                currentProposal = generatorOutput;
                System.out.println("[Generate] 已生成方案 (长度: " + generatorOutput.length() + " 字符)");

                // 阶段2: Critique（批评）
                String criticPrompt = buildCriticPrompt(topic, generatorOutput, result.getRounds());
                CompletableFuture<String> critFuture = CompletableFuture.supplyAsync(
                    () -> llmService.analyze(criticPrompt));

                String criticOutput = critFuture.get(config.getRoundTimeoutMs(), TimeUnit.MILLISECONDS);
                debateRound.setCriticFeedback(criticOutput);
                System.out.println("[Critique] 已生成批评 (长度: " + criticOutput.length() + " 字符)");

                // 解析批评结果
                CritiqueResult critique = parseCritiqueResult(criticOutput);
                debateRound.setIssuesFound(critique.issues);
                debateRound.setQualityScore(critique.score);

                System.out.println("[Critique] 本轮评分: " + critique.score + "/100");
                System.out.println("[Critique] 发现问题: " + critique.issues.size() + " 个");

                result.addRound(debateRound);

                // 检查终止条件
                if (shouldTerminate(critique.score, previousScore, critique.issues, round)) {
                    System.out.println(">>> 辩论终止: 达到终止条件 <<<");
                    break;
                }

                previousScore = critique.score;

            } catch (TimeoutException e) {
                System.err.println("第 " + round + " 轮超时，终止辩论");
                break;
            } catch (Exception e) {
                System.err.println("第 " + round + " 轮异常: " + e.getMessage());
                break;
            }
        }

        // 阶段3: Judge（裁判）
        if (config.isEnableJudge() && currentProposal != null) {
            System.out.println("\n--- 裁判阶段 ---");
            String judgePrompt = buildJudgePrompt(topic, result);
            String judgeOutput = llmService.analyze(judgePrompt);
            result.setJudgeVerdict(judgeOutput);

            JudgeResult judgeResult = parseJudgeResult(judgeOutput);
            result.setFinalScore(judgeResult.finalScore);
            result.setUnresolvedIssues(judgeResult.unresolvedIssues);
            System.out.println("[Judge] 最终评分: " + judgeResult.finalScore + "/100");
        }

        result.setFinalProposal(currentProposal);
        result.setDurationMs(System.currentTimeMillis() - startTime);
        result.setTotalTokensUsed(totalTokens);

        System.out.println("\n辩论结束，历时 " + result.getDurationMs() + "ms");
        return result;
    }

    /**
     * 构建生成 Agent 的 Prompt
     */
    private String buildGeneratorPrompt(DebateTopic topic, String previousProposal,
                                         List<DebateRound> history, int round) {
        StringBuilder sb = new StringBuilder();
        sb.append("你是一名资深技术专家，正在参与一场技术辩论。\n\n");
        sb.append("【辩论主题】").append(topic.getTitle()).append("\n");
        sb.append("【主题描述】").append(topic.getDescription()).append("\n\n");

        sb.append("【你的角色】方案生成者（Generator）\n");
        sb.append("【你的目标】提出最优的技术方案\n\n");

        if (round == 1) {
            sb.append("这是第1轮辩论，请提出你的初始方案。\n");
            sb.append("要求：完整、具体、考虑边界条件\n");
        } else {
            sb.append("这是第").append(round).append("轮辩论。\n");
            sb.append("上一轮你的方案被批评了，以下是批评者的反馈：\n\n");
            sb.append("---批评反馈---\n");
            sb.append(history.get(history.size() - 1).getCriticFeedback()).append("\n");
            sb.append("---批评结束---\n\n");
            sb.append("请针对批评者的反馈改进你的方案。你需要：\n");
            sb.append("1. 逐条回应批评者提出的问题\n");
            sb.append("2. 修改方案中存在的缺陷\n");
            sb.append("3. 补充之前遗漏的内容\n");
        }

        sb.append("\n请以结构化格式输出你的方案。");
        return sb.toString();
    }

    /**
     * 构建批评 Agent 的 Prompt
     */
    private String buildCriticPrompt(DebateTopic topic, String proposal,
                                      List<DebateRound> history) {
        StringBuilder sb = new StringBuilder();
        sb.append("你是一名资深技术评审专家，正在参与一场技术辩论。\n");
        sb.append("你的角色是批评者（Critic），目标是找出方案中的所有问题。\n\n");
        sb.append("【辩论主题】").append(topic.getTitle()).append("\n\n");
        sb.append("【当前方案】\n").append(proposal).append("\n\n");
        sb.append("请以严苛的标准评审这个方案，找出以下问题：\n");
        sb.append("1. 逻辑漏洞和前后矛盾\n");
        sb.append("2. 未考虑的边界条件\n");
        sb.append("3. 性能隐患和扩展性问题\n");
        sb.append("4. 安全风险\n");
        sb.append("5. 可维护性问题\n\n");
        sb.append("请按以下格式输出：\n");
        sb.append("评分: [0-100]\n");
        sb.append("主要问题:\n");
        sb.append("- 问题1\n");
        sb.append("- 问题2\n");

        if (!history.isEmpty()) {
            sb.append("\n提醒：前几轮辩论中已发现但未解决的问题，请重点关注是否在本轮被修复。");
        }

        return sb.toString();
    }

    /**
     * 构建裁判 Agent 的 Prompt
     */
    private String buildJudgePrompt(DebateTopic topic, DebateResult result) {
        StringBuilder sb = new StringBuilder();
        sb.append("你是一名公正的裁判，你需要综合评估一场技术辩论。\n\n");
        sb.append("【辩论主题】").append(topic.getTitle()).append("\n\n");
        sb.append("【辩论过程】\n");

        for (DebateRound round : result.getRounds()) {
            sb.append("--- 第").append(round.getRoundNumber()).append("轮 ---\n");
            sb.append("[方案] ").append(abbreviate(round.getGeneratorOutput(), 500)).append("\n");
            sb.append("[批评] ").append(abbreviate(round.getCriticFeedback(), 500)).append("\n");
            sb.append("[评分] ").append(round.getQualityScore()).append("/100\n\n");
        }

        sb.append("请你作为裁判：\n");
        sb.append("1. 给出最终评分 (0-100)\n");
        sb.append("2. 列出仍未解决的问题\n");
        sb.append("3. 给出改进建议\n");
        sb.append("4. 判断方案是否可以投入使用\n\n");
        sb.append("最终评分: [0-100]\n");
        sb.append("未解决问题:\n");

        return sb.toString();
    }

    /**
     * 判断是否应该终止辩论
     */
    private boolean shouldTerminate(double currentScore, double previousScore,
                                     List<String> issues, int round) {
        // 条件1: 达到质量线
        if (currentScore >= config.getQualityThreshold()) {
            System.out.println("   => 达到质量线 " + config.getQualityThreshold());
            return true;
        }

        // 条件2: 没有新问题发现
        if (issues.isEmpty()) {
            System.out.println("   => 批评者未发现新问题");
            return true;
        }

        // 条件3: 改进幅度收敛
        if (round > 1 && (currentScore - previousScore) < config.getImprovementThreshold()) {
            System.out.println("   => 改进幅度收敛 (< " + config.getImprovementThreshold() + ")");
            return true;
        }

        // 条件4: 到达最大轮次
        if (round >= config.getMaxRounds()) {
            System.out.println("   => 达到最大辩论轮次");
            return true;
        }

        return false;
    }

    /**
     * 解析批评结果
     */
    private CritiqueResult parseCritiqueResult(String output) {
        CritiqueResult result = new CritiqueResult();
        String[] lines = output.split("\n");
        boolean inIssues = false;

        for (String line : lines) {
            line = line.trim();
            if (line.startsWith("评分:") || line.startsWith("Score:")) {
                try {
                    result.score = Double.parseDouble(line.replaceAll("[^0-9.]", ""));
                } catch (NumberFormatException ignored) {}
            }
            if (line.contains("主要问题") || line.contains("问题:")) {
                inIssues = true;
                continue;
            }
            if (inIssues && line.startsWith("-")) {
                result.issues.add(line.substring(1).trim());
            }
        }
        return result;
    }

    /**
     * 解析裁判结果
     */
    private JudgeResult parseJudgeResult(String output) {
        JudgeResult result = new JudgeResult();
        String[] lines = output.split("\n");
        boolean inUnresolved = false;

        for (String line : lines) {
            line = line.trim();
            if (line.startsWith("最终评分:")) {
                try {
                    result.finalScore = Double.parseDouble(line.replaceAll("[^0-9.]", ""));
                } catch (NumberFormatException ignored) {}
            }
            if (line.contains("未解决问题")) {
                inUnresolved = true;
                continue;
            }
            if (inUnresolved && (line.startsWith("-") || line.matches("\\d+\\..*"))) {
                result.unresolvedIssues.add(line.replaceAll("^[-\\d.]+\\s*", ""));
            }
            if (inUnresolved && line.isBlank()) {
                inUnresolved = false;
            }
        }
        return result;
    }

    private String abbreviate(String text, int maxLength) {
        if (text == null) return "";
        return text.length() > maxLength
            ? text.substring(0, maxLength) + "...(省略)"
            : text;
    }

    // 内部类
    private static class CritiqueResult {
        double score = 0;
        List<String> issues = new ArrayList<>();
    }

    private static class JudgeResult {
        double finalScore = 0;
        List<String> unresolvedIssues = new ArrayList<>();
    }
}
```

## 三、实战：用辩论模式做代码设计评审

### 3.1 评审场景定义

```java
public class CodeDesignDebate {

    public static void main(String[] args) {
        // 初始化
        LLMService llmService = new OpenAILLMService("your-api-key");

        DebateConfig config = new DebateConfig()
            .maxRounds(3)
            .qualityThreshold(80.0)
            .improvementThreshold(3.0)
            .enableJudge(true)
            .roundTimeout(90000);

        DebateEngine engine = new DebateEngine(llmService, config);

        // 构建辩论主题：评审用户服务的设计
        DebateTopic topic = new DebateTopic(
            "微服务架构下用户服务的设计方案评审",
            "背景：电商平台需要重构用户服务，要求支持以下功能：\n" +
            "1. 用户注册/登录（支持手机号+验证码、邮箱+密码两种方式）\n" +
            "2. 用户信息管理（基本信息、收货地址）\n" +
            "3. 用户等级体系（普通/银卡/金卡/钻石）\n" +
            "4. 日活用户 100万+，峰值 QPS 5000\n" +
            "5. 数据一致性要求：用户等级变更不能丢失\n\n" +
            "请评审以下设计方案的合理性：使用 MySQL 作为主存储 + Redis 缓存，" +
            "用户等级用定时任务批量计算，服务间通过 Kafka 同步用户变更事件。"
        );

        topic.addContext("system", "电商平台");
        topic.addContext("team_size", 5);
        topic.addContext("tech_stack", "Spring Boot + MyBatis + Redis + Kafka");

        // 执行辩论
        DebateResult result = engine.debate(topic);

        // 输出结果
        printDebateReport(result);
    }

    private static void printDebateReport(DebateResult result) {
        System.out.println("\n==================== 辩论报告 ====================");
        System.out.println("主题ID: " + result.getTopicId());
        System.out.println("辩论轮次: " + result.getRounds().size());
        System.out.println("最终评分: " + result.getFinalScore() + "/100");
        System.out.println("耗时: " + result.getDurationMs() + "ms");

        System.out.println("\n--- 各轮评分 ---");
        for (DebateRound round : result.getRounds()) {
            System.out.printf("第%d轮: %.1f分 | 问题数: %d%n",
                round.getRoundNumber(),
                round.getQualityScore(),
                round.getIssuesFound().size());
        }

        if (!result.getUnresolvedIssues().isEmpty()) {
            System.out.println("\n--- 未解决问题 ---");
            result.getUnresolvedIssues().forEach(i -> System.out.println("  - " + i));
        }

        System.out.println("\n--- 裁判意见 ---");
        System.out.println(result.getJudgeVerdict());
        System.out.println("\n==================================================\n");
    }
}
```

### 3.2 辩论过程中的对话示例

实际的 LLM 输出大概是这样的：

**【第1轮 — Generator】**
> 方案：采用 MySQL 分库分表 + Redis 热点缓存 + Kafka 事件驱动架构。
> 用户表按 user_id 分 16 个库，地址表独立分库。等级计算使用定时任务凌晨 2 点执行...

**【第1轮 — Critic】**
> 评分: 62/100
> 主要问题:
> - 定时任务凌晨批量计算等级，如果用户刚满足升级条件需要等到第二天才能生效，延迟最长 24 小时
> - 分库后跨库 JOIN 查询无法实现，但方案中未提及如何解决
> - Redis 缓存和 MySQL 之间的一致性策略不明确

**【第2轮 — Generator（改进）】**
> 已收到批评，做出以下改进：
> 1. 等级计算改为基于事件驱动的准实时方案，用户完成关键行为后立即更新等级
> 2. 跨库查询通过"用户 ID 路由 + 两次查询"解决，不使用 JOIN
> 3. 采用 Cache-Aside 模式 + 双删策略保证缓存一致性...

**【第2轮 — Critic】**
> 评分: 78/100
> 主要问题:
> - 事件驱动的等级计算在高并发下仍可能丢失事件（Kafka 分区内有序但跨分区无序）
> - Cache-Aside 双删在极端网络延迟下仍有脏读可能
> - 未考虑用户注销后的数据清理策略

经过三轮辩论，方案逐步完善，最终评分从 62 提升到 85。

## 四、辩论 vs 单 Agent：效果对比

我用同一个代码设计评审任务，对比了三种模式：

| 维度 | 单Agent直接生成 | 单Agent反复迭代（3次） | 辩论模式（3轮） |
|------|----------------|---------------------|----------------|
| 发现问题数 | 6 | 11 | 18 |
| 遗漏风险点 | 7 | 4 | 1 |
| 方案完整度 | 中 | 高 | 很高 |
| Token消耗 | 2.1K | 6.5K | 8.9K |
| 耗时 | 3s | 12s | 21s |

辩论模式的 Token 消耗和耗时确实更高，但**发现的问题多了 67%，遗漏率降低了 75%**。对于代码设计、架构评审、安全审计这些"质量优先"的场景，多花点 Token 是值得的。

特别有意思的是，辩论模式发现的 18 个问题中，有 5 个是单 Agent 模式完全没意识到的"盲区问题"——比如"等级变更事件在跨分区的 Kafka 中如何保证全序"这种分布式系统特有的坑。

## 五、优化建议

### 5.1 辩论节奏控制

如果批评者太"佛系"（不给力），辩论就流于形式。可以通过调整批评者的 Prompt 温度来增加"攻击性"：

```java
public class CriticAgentConfig {
    private double temperature = 0.3;    // 默认温和
    private double aggressiveTemp = 0.8; // 激进模式
    private String critiqueStyle;        // "constructive" / "adversarial"

    public void setAggressiveMode() {
        this.temperature = aggressiveTemp;
        this.critiqueStyle = "adversarial";
    }
}
```

### 5.2 议题聚焦

不要让批评者"什么都可以批评"，那样容易跑题。根据场景限定批评维度：

```java
public enum CritiqueDimension {
    CORRECTNESS,     // 正确性
    PERFORMANCE,     // 性能
    SECURITY,        // 安全性
    SCALABILITY,     // 可扩展性
    MAINTAINABILITY, // 可维护性
    COST             // 成本
}

// 在 Prompt 中指定批评维度
"请仅从【性能】和【安全性】两个维度进行评审"
```

### 5.3 多批评者模式

一个批评者可能有偏见，可以引入多个不同领域的批评者：

```java
List<String> critics = Arrays.asList(
    "架构评审专家（关注性能和扩展性）",
    "安全评审专家（关注安全漏洞）",
    "业务评审专家（关注业务逻辑正确性）"
);
```

## 六、总结

辩论模式不是银弹，但它是目前最有效的"AI 输出质量提升"手段之一：

1. **原理**：三人行必有我师 → 三个 AI 互相辩论，从对抗中获取更全面的视角
2. **架构**：Generator（生成）+ Critic（批评）+ Judge（裁判）三方协作
3. **流程**：轮次迭代，每轮 Generate → Critique，直到方案收敛
4. **效果**：发现的问题多 67%，遗漏率降 75%，代价是多消耗一些 Token

什么时候该用辩论模式？一个简单的判断标准：**如果这个方案的错误代价高于辩论的 Token 成本，就用辩论模式。** 比如架构设计、安全方案、资金相关的业务逻辑，这些场景多花几分钟和几万 Token 换一个更可靠的方案，绝对值得。

---

**下一篇预告**：《Agent 分层架构：Master Agent + Sub Agent 的任务分配策略》— 让一个 AI 学会"管理"其他 AI。Master Agent 负责拆解任务、分配工作、监控进度，Sub Agent 专注执行。这本质上就是 AI 时代的"技术主管 + 开发团队"。
