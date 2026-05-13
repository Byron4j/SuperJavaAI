# ReAct 模式实战：让 LLM 学会"思考-行动-观察"循环解决问题，像一个真正的工程师

> 你 Debug 的时候怎么做的？发现问题→猜测原因→加日志验证→看日志→确认或调整猜测→再验证……没错，这个循环就是 ReAct。现在你可以让 AI 也这样做。

---

## 一、从一个日常场景说起

凌晨两点，生产环境告警：订单支付成功率骤降 30%。你睡眼惺忪地爬起来，开始排查：

1. **发现问题**：支付接口大量超时
2. **猜测原因**：是不是第三方支付网关挂了？
3. **行动验证**：curl 一下支付网关的健康检查接口
4. **观察结果**：网关正常返回 200
5. **调整猜测**：那可能是我们自己的连接池耗尽了？查一下线程 dump
6. **再行动、再观察**：线程 dump 显示大量线程 BLOCKED 在 Redis 连接获取上
7. **定位根因**：Redis 集群某个分片昨晚做了一次主从切换，部分连接未正确释放，把连接池打满了

你发现没有？整个排查过程就是一套标准的 **"思考→行动→观察→再思考→再行动→再观察"** 循环。你不是一次性给出完美答案，而是在**与环境的交互中逐步逼近真相**。

这套方法论用在人类工程师身上是本能，但传统的大模型调用方式（一问一答）完全不具备这种能力——直到 **ReAct 模式** 的出现。

---

## 二、ReAct 是什么？为什么它让 LLM 像个工程师

### 2.1 核心思想

ReAct = **Re**asoning（推理） + **Act**ing（行动）

这是一种 Prompt Engineering 范式，让 LLM 不再"一次生成最终答案"，而是像人类专家一样，**交替进行推理和行动，在循环中逐步解决问题**。

```
┌──────────────────────────────────────────────┐
│                 ReAct 循环                     │
│                                              │
│   Thought ──→ Action ──→ Observation         │
│      ↑                        │              │
│      └────────────────────────┘              │
│          （根据观察更新思考）                    │
└──────────────────────────────────────────────┘
```

- **Thought（思考）**：分析当前状态，判断下一步该做什么
- **Action（行动）**：执行具体操作——搜索、计算、调用工具、写代码
- **Observation（观察）**：解读行动结果，确认是否解决了问题

### 2.2 用 Java 开发场景理解

假设你让 AI 帮你优化一个慢 SQL：

**传统 CoT（Chain-of-Thought）的做法**：
> "让我一步步分析这个 SQL……首先看索引是否命中……其次看数据量……最后可能是锁竞争……结论是加一个联合索引。"

——看起来像回事，但全是**凭空推理**，没看过实际执行计划，没跑过 EXPLAIN。

**ReAct 的做法**：

```
Thought: 我需要先拿到这个 SQL 的实际执行计划才能分析
Action: 执行 EXPLAIN SELECT * FROM orders WHERE status = 'PENDING' AND created_at > '2026-01-01'
Observation: type=ALL, rows=2870000, Extra=Using where，全表扫描，没命中任何索引
Thought: 全表扫描的原因是缺少 (status, created_at) 的联合索引，我来创建并验证
Action: 执行 CREATE INDEX idx_status_created ON orders(status, created_at)
Action: 再次 EXPLAIN 验证
Observation: type=range, rows=12000, key=idx_status_created，扫描行数从 287 万降到 1.2 万
Thought: 索引生效，问题解决。建议在测试环境先验证再上线
```

看出区别了吗？ReAct 的每一步都有**事实依据**支撑，它真的在"干活"而不是"猜"。

---

## 三、三个 ReAct Prompt 实战案例

### 案例 1：排查线上问题——用户登录偶发超时

**场景**：用户反馈登录接口有时响应超过 5 秒，但不是每次都复现。

**ReAct Prompt**：

```markdown
你是一个资深的 Java 后端排查工程师。请使用 ReAct 模式排查以下问题，每步遵循 Thought → Action → Observation 格式。

## 问题描述
用户登录接口 `/api/login` 偶发超时（超过 5 秒），接口链路：
Nginx → Gateway → AuthService(/api/login) → Redis(查session) → MySQL(查用户表) → 返回JWT

## 已知信息
- AuthService 部署 4 个节点，每个节点 2C4G
- 高峰期 QPS 约 500
- 数据库连接池：HikariCP，maximumPoolSize=20
- Redis 连接池：Lettuce，maxIdle=8, maxTotal=16
- 近一周未做任何发布

## 开始排查
```

**LLM 的 ReAct 输出（节选）**：

```json
[
  {
    "thought": "偶发超时通常和资源竞争有关。先排除数据库连接池是否成为瓶颈——如果连接池满了，请求会排队等待获取连接。",
    "action": {
      "type": "query_metrics",
      "target": "HikariCP",
      "metrics": ["active_connections", "idle_connections", "pending_threads", "connection_timeout_rate"]
    },
    "observation": "高峰时段 active_connections 长期维持在 18-20，pending_threads 平均 5，connectionTimeout 每分钟约 30 次"
  },
  {
    "thought": "连接池打满了！maximumPoolSize=20 不够用。但需要找出是什么操作占着连接不释放。先看慢查询。",
    "action": {
      "type": "query_slow_log",
      "database": "user_db",
      "threshold_ms": 500,
      "time_range": "last_1_hour"
    },
    "observation": "SELECT * FROM login_history WHERE user_id = ? ORDER BY login_time DESC LIMIT 5 平均耗时 1.2 秒，全表扫描，login_history 表 1200 万行，无 user_id 索引"
  },
  {
    "thought": "根因找到了：login_history 表缺少 user_id 索引，每次登录都会全表扫描一张千万级表，耗时长且长期占用连接，拖垮了连接池。连接池只有 20 个连接，高峰期 500 QPS 根本扛不住。",
    "action": {
      "type": "suggest_fix",
      "priority": "P0",
      "items": [
        "给 login_history 表加索引：CREATE INDEX idx_user_login_time ON login_history(user_id, login_time DESC)",
        "同时扩容连接池：HikariCP maximumPoolSize 调至 40",
        "考虑 login_history 写入异步化，不阻塞登录主流程"
      ]
    },
    "observation": "方案已生成，建议先在预发环境验证索引创建耗时（千万级表可能需要 ONLINE DDL 或 pt-osc）"
  },
  {
    "thought": "总结一下完整排查路径：监控指标观察→连接池瓶颈确认→慢查询定位→根因是缺失索引→给出修复方案。排查完成。",
    "action": { "type": "final_answer" },
    "observation": "根因：login_history 表缺少 user_id 索引导致每次登录全表扫描，耗时 1.2 秒，高峰期连接池打满。修复：加联合索引 + 扩容连接池 + 异步化写入。"
  }
]
```

> **关键点**：注意每一步 Thought 都在**基于上一轮的 Observation 调整方向**，而不是一次到底。这就是 ReAct 区别于链式思考的核心——它能**根据新信息修正自己的判断**。

---

### 案例 2：技术调研——是否引入虚拟线程（Virtual Threads）

**场景**：团队在讨论是否将项目从 Java 17 升级到 Java 21 来使用虚拟线程替代现有的 WebFlux 响应式模型。

**ReAct Prompt**：

```markdown
你是一个技术架构师。请使用 ReAct 模式对以下技术选型问题做调研，每步输出 Thought → Action → Observation。

## 调研问题
我们的电商后端服务目前用 Spring WebFlux + Reactor 处理高并发，
考虑迁移到 Java 21 虚拟线程 + Spring Boot 3.x + Tomcat 同步模型。
请评估这个迁移是否有价值。

## 评估维度
1. 性能对比（吞吐量、延迟、内存占用）
2. 代码可维护性
3. 生态兼容性（数据库驱动、HTTP 客户端、MQ 客户端）
4. 迁移成本
5. pinning 风险（synchronized 导致的载体线程阻塞）

## 开始调研
```

**LLM 的 ReAct 输出（节选）**：

```json
[
  {
    "thought": "先搜集虚拟线程在性能上的硬数据。需要找到有说服力的基准测试结果，而不是凭感觉下结论。",
    "action": { "type": "search", "query": "Java 21 virtual threads vs WebFlux benchmark throughput latency 2024 2025" },
    "observation": "多个独立测试（包括 Netflix、Spring 官方）显示：在 IO 密集型场景下，虚拟线程 + Tomcat 的吞吐量略高于 WebFlux（+5%~15%），延迟 P99 稍低。但在纯 CPU 密集型场景下，两者差距不大。内存占用方面，虚拟线程每个约占用几百字节，远低于平台线程的 1MB。"
  },
  {
    "thought": "性能上有微弱优势但不明显。但最大的收益可能不在性能，而在代码可维护性——WebFlux 的链式调用和背压调试确实让人头疼。需要评估具体代码量差异。",
    "action": { "type": "search", "query": "Spring WebFlux vs virtual threads code complexity example controller service" },
    "observation": "典型对比：WebFlux 一个带超时+重试+降级的 HTTP 调用需要 .timeout().retry().onErrorResume() 链式编排，平均 15-20 行；虚拟线程版本直接用 try-catch + ExecutorService，平均 8-10 行。调试栈：WebFlux 的 Reactor 栈追踪信息可达 200+ 行且大量框架内部调用；虚拟线程的栈追踪清晰可读。"
  },
  {
    "thought": "可维护性优势明显。接下来查pinning风险和生态兼容性——这是迁移的最大不确定因素。",
    "action": { "type": "search", "query": "Java virtual threads synchronized pinning carrier thread problem JDBC driver" },
    "observation": "关键风险：(1) synchronized 块内调用 IO 操作会 pin 住载体线程，导致虚拟线程退化为平台线程。JDBC 驱动是重灾区——MySQL Connector/J 8.x 大量使用 synchronized。好在 JDK 24 已修复此问题（JEP 491），但 Java 21 上仍需排查。(2) 主流框架 Spring Boot 3.2+、Tomcat 10.1+ 已原生支持虚拟线程。(3) MySQL Connector/J 9.x 已移除 synchronized。"
  },
  {
    "thought": "迁移成本方面，需要比对一下两个方案的改动范围。",
    "action": { "type": "analyze", "aspects": ["migration_cost", "risk_matrix"] },
    "observation": "迁移成本分析：Java 17→21 升级相对平滑（LTS 到 LTS）。关键改动：去掉 Reactor 依赖，Controller 返回类型从 Mono/Flux 改为普通对象，去掉 .subscribeOn()/.flatMap() 等操作符。风险矩阵：P0 风险=pinning（需要排查所有 synchronized 调用链路），P1 风险=第三方库兼容性，P2 风险=团队学习成本。"
  },
  {
    "thought": "综合评估，迁移是有价值的，但需要分阶段推进。给出最终的决策建议。",
    "action": { "type": "final_answer" },
    "observation": "决策建议：推荐迁移，但分三阶段：(1) 先升级 Java 21 + 排查 synchronized pinning 风险（1-2个迭代）；(2) 在非核心服务试点虚拟线程（1个迭代）；(3) 核心服务逐步替换 WebFlux（2-3个迭代）。总体收益：代码复杂度降低约 40%，可维护性显著提升，性能持平或略优。不建议停留在 Java 17 等到 Java 24——Java 21 是 LTS，企业级支撑更成熟。"
  }
]
```

> **关键点**：ReAct 在做决策类任务时，不是拍脑袋给结论，而是**先搜数据、再对比、再找风险、最后综合判断**——这正是资深架构师的工作方式。

---

### 案例 3：代码重构——将上帝类按职责拆分为领域服务

**场景**：一个 2300 行的 `OrderService` 类，包含了订单的创建、支付、物流、退款、通知等全部逻辑。

**ReAct Prompt**：

```markdown
你是一个 Java 重构专家。请使用 ReAct 模式对一个上帝类进行重构分析并制定计划。

## 待重构的类
com.ecommerce.order.service.OrderService (2300 行)

## 包含的方法
- createOrder()
- cancelOrder()
- payOrder() / refundOrder()
- shipOrder() / trackLogistics()
- sendOrderEmail() / sendOrderSMS()
- generateOrderReport()
- calculateDiscount()
- validateInventory()

## 要求
1. 分析类的职责边界
2. 制定拆分方案
3. 给出每一步的具体执行计划
4. 说明每个步骤的验证方式

## 开始重构分析
```

**LLM 的 ReAct 输出（节选）**：

```json
[
  {
    "thought": "先按领域驱动设计（DDD）的聚合根和领域服务原则，对这个类的职责做一次梳理。很明显这个类跨越了订单核心、支付、物流、通知、报表五个完全不同的领域。",
    "action": {
      "type": "analyze_responsibilities",
      "methods": ["createOrder", "cancelOrder", "payOrder", "refundOrder", "shipOrder", "trackLogistics", "sendOrderEmail", "sendOrderSMS", "generateOrderReport", "calculateDiscount", "validateInventory"]
    },
    "observation": "职责分类结果：\n- 订单生命周期：createOrder, cancelOrder（核心聚合根）\n- 支付领域：payOrder, refundOrder\n- 物流领域：shipOrder, trackLogistics\n- 通知领域：sendOrderEmail, sendOrderSMS\n- 营销领域：calculateDiscount\n- 库存领域：validateInventory\n- 报表领域：generateOrderReport\n共 7 个职责边界，明显违反单一职责原则（SRP）。"
  },
  {
    "thought": "接下来画一下类之间的依赖关系，确定如何拆分而不破坏现有功能。validateInventory 被 createOrder 调用，payOrder 被外部 Controller 调用但也依赖 Order 状态变更——需要设计好接口和事件驱动。",
    "action": {
      "type": "depends_analysis",
      "target_class": "OrderService"
    },
    "observation": "依赖分析：createOrder 依赖 validateInventory + calculateDiscount；payOrder 依赖 Order 状态；shipOrder 依赖 payOrder 结果；sendOrderEmail/SMS 在 createOrder、payOrder、shipOrder、cancelOrder 后都会被触发。最合理的拆分策略：核心聚合根保留在 OrderService，其他职责通过领域事件（Spring ApplicationEvent）解耦。"
  },
  {
    "thought": "好，现在制定具体的分步重构计划，注意每一步都必须可验证，不能一口气全改——那是给自己挖坑。",
    "action": {
      "type": "generate_refactor_plan",
      "style": "step_by_step_with_verification"
    },
    "observation": "重构计划已生成（共 5 步，预计 3-4 个开发日）"
  },
  {
    "thought": "最后输出完整的、可以直接交给团队执行的重构方案，包括每一步的代码示例和验证标准。",
    "action": { "type": "final_answer" },
    "observation": "完整重构方案如下：\n\n第 1 步：引入领域事件基类\n- 创建 OrderEvent(Order order, OrderEventType type)\n- 发布时机：状态变更后 → eventPublisher.publishEvent(new OrderEvent(order, CREATED))\n- 验证：单元测试确认事件发布成功，原有功能不受影响\n\n第 2 步：拆分库存服务\n- 新建 InventoryService，移入 validateInventory()\n- OrderService 注入 InventoryService，替换直接调用\n- 验证：下单流程的单元测试 + 集成测试全绿\n\n第 3 步：拆分支付服务\n- 新建 PaymentService，移入 payOrder() + refundOrder()\n- 监听 OrderEvent(PAID/REFUNDED) 触发后续物流操作\n- 验证：支付全流程回归测试\n\n第 4 步：拆分物流 + 通知服务\n- 新建 LogisticsService, NotificationService\n- 全部通过监听领域事件驱动，和 OrderService 零直接依赖\n- 验证：端到端测试：下单→支付→发货→通知 全链路\n\n第 5 步：清理 + 归档\n- 移除 OrderService 中已迁移的方法\n- 更新所有调用方引用\n- 验证：全量回归测试 + SonarQube 扫描无新增问题"
  }
]
```

> **关键点**：ReAct 做重构不是生成一个花哨的方案，而是**先分析、再设计、再分步、每步可验证**。这种"工程化"的思维方式，才是我们真正需要的。

---

## 四、ReAct Prompt 模板——直接复制使用

以下是一个结构化的 ReAct Prompt 模板，内部使用 JSON 格式作为 LLM 的输出规范，方便你用代码解析每一步的 Thought / Action / Observation。

### 通用 ReAct Prompt 模板

```markdown
你是一个 {角色}，擅长 {领域}。请使用 ReAct 模式解决以下问题。

## ReAct 输出格式要求
每轮输出必须是一个 JSON 对象，包含以下字段：

{
  "step": <第几步，从1开始>,
  "thought": "<你的推理过程，分析当前状态，决定下一步行动>",
  "action": {
    "type": "<行动类型: search/calculate/code_execute/analyze/ask_clarification/final_answer>",
    "description": "<行动的具体描述>",
    "input": "<如果行动需要输入参数，写在这里>"
  },
  "observation": "<执行行动后预期或实际得到的结果>"
}

## 终止条件
当 thought 判断问题已经解决时，action.type 必须为 "final_answer"，
action.description 为最终答案或方案的总结。

## 约束
- 每步都必须先思考(thought)、再行动(action)、再记录观察(observation)
- 不要让 thought 凭空猜测，必须基于已有的 observation
- 如果某一轮的行动没有解决问题，必须调整 thought 的方向
- 最多执行 10 步，超过则给出当前最佳答案

## 问题描述
{在这里填入你的问题}

## 已知信息
{在这里填入相关的上下文}

## 现在开始
```

### 专用于 Java 技术问题的模板（含代码执行）

```markdown
你是一个资深的 Java 技术专家。请使用 ReAct 模式解决以下技术问题。

## 执行环境
- 你可以执行 Java 代码片段、SQL 查询、Shell 命令
- 需要查看代码时，说明文件名和行号范围
- 代码审查时，指出具体的问题行和修改建议

## 输出格式
```json
{
  "step": 1,
  "thought": "...",
  "action": {
    "type": "code_execute|search|sql_query|read_file|analyze|final_answer",
    "details": "..."
  },
  "observation": "..."
}
```

## 当前问题
{问题描述}

## 项目上下文
- 技术栈：Java 17, Spring Boot 3.x, MySQL 8.0, Redis 7.x
- 项目路径：/project/ecommerce
- 已排查的方向：{已经查过的}

## 开始排查
```

---

## 五、Java 代码解析 ReAct 输出

让 LLM 输出 ReAct 格式只是第一步，真正的工程化落地需要**解析这些输出并执行对应的 Action**。下面是一个完整的 Java 解析框架。

### 5.1 数据模型

```java
package com.example.react.agent;

import com.fasterxml.jackson.annotation.JsonProperty;
import java.util.List;

/**
 * ReAct Agent 的核心数据模型
 */
public class ReActStep {

    private int step;

    private String thought;

    private Action action;

    private String observation;

    // getters & setters 省略

    public static class Action {
        private String type;          // search, code_execute, sql_query, read_file, analyze, final_answer
        private String description;
        private String input;

        @JsonProperty("details")
        private String details;       // 兼容多种命名

        // getters & setters 省略
    }
}
```

### 5.2 ReAct 循环引擎

```java
package com.example.react.agent;

import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

/**
 * ReAct 模式的核心执行引擎
 * 职责：驱动 Thought → Action → Observation 循环，直到问题解决或达到最大步数
 */
public class ReActEngine {

    private static final Logger log = LoggerFactory.getLogger(ReActEngine.class);
    private static final int MAX_STEPS = 10;
    private static final ObjectMapper MAPPER = new ObjectMapper();

    private final LLMClient llmClient;                    // 调用大模型的客户端
    private final Map<String, ActionHandler> handlers;    // Action 处理器注册表
    private final List<ReActStep> history;                // 完整的执行历史

    public ReActEngine(LLMClient llmClient) {
        this.llmClient = llmClient;
        this.handlers = new ConcurrentHashMap<>();
        this.history = new ArrayList<>();
    }

    /**
     * 注册一个 Action 类型的处理器
     * 例如：注册 search → GoogleSearchHandler，code_execute → SandboxExecutor
     */
    public void registerHandler(String actionType, ActionHandler handler) {
        handlers.put(actionType, handler);
    }

    /**
     * 启动 ReAct 循环，解决给定的问题
     *
     * @param problem  问题描述
     * @param context  上下文信息（错误日志、代码片段等）
     * @return 最终的 ReAct 执行记录
     */
    public ReActResult solve(String problem, String context) {
        String prompt = buildReActPrompt(problem, context, history);

        for (int step = 1; step <= MAX_STEPS; step++) {
            log.info("=== ReAct Step {} ===", step);

            // 第1步：调用 LLM，获取 Thought + Action
            String llmResponse = llmClient.chat(prompt);
            ReActStep currentStep = parseStep(llmResponse, step);

            if (currentStep == null) {
                log.error("Step {} 解析失败，LLM 返回: {}", step, llmResponse);
                break;
            }

            log.info("Thought: {}", currentStep.getThought());

            // 第2步：如果 LLM 认为问题已解决，终止循环
            if ("final_answer".equals(currentStep.getAction().getType())) {
                history.add(currentStep);
                log.info("ReAct 循环结束，最终答案: {}", currentStep.getAction().getDescription());
                return new ReActResult(true, history, currentStep.getAction().getDescription());
            }

            // 第3步：执行 Action，获得 Observation
            String observation = executeAction(currentStep.getAction());
            currentStep.setObservation(observation);
            history.add(currentStep);

            log.info("Observation: {}", observation);

            // 第4步：将本轮结果追加到 prompt 中，让 LLM 基于新信息继续推理
            prompt = buildReActPrompt(problem, context, history);
        }

        // 达到最大步数，返回当前状态
        log.warn("ReAct 循环达到最大步数 {}，返回当前最佳答案", MAX_STEPS);
        return new ReActResult(false, history, "达到最大步数限制，未能完全解决");
    }

    /**
     * 构建 ReAct Prompt，包含完整的历史记录
     */
    private String buildReActPrompt(String problem, String context, List<ReActStep> history) {
        StringBuilder sb = new StringBuilder();

        sb.append("""
            你是一个资深的 Java 技术专家。请使用 ReAct 模式解决问题。
            
            输出格式（严格 JSON）：
            {
              "step": %d,
              "thought": "你的推理",
              "action": { "type": "...", "description": "...", "input": "..." }
            }
            
            问题：%s
            
            背景信息：%s
            
            """.formatted(history.size() + 1, problem, context));

        if (!history.isEmpty()) {
            sb.append("## 已有的排查记录\n\n");
            sb.append("```json\n");
            try {
                sb.append(MAPPER.writerWithDefaultPrettyPrinter().writeValueAsString(history));
            } catch (Exception e) {
                sb.append(history.toString());
            }
            sb.append("\n```\n\n");
            sb.append("请基于以上已有信息，继续下一步的 Thought 和 Action。\n");
            sb.append("如果问题已解决，action.type 必须为 \"final_answer\"。\n");
        }

        return sb.toString();
    }

    /**
     * 解析 LLM 返回的 JSON 为 ReActStep 对象
     */
    private ReActStep parseStep(String llmResponse, int step) {
        try {
            // LLM 可能返回带有 markdown 代码块的 JSON，先清洗
            String json = extractJson(llmResponse);
            ReActStep reactStep = MAPPER.readValue(json, ReActStep.class);
            reactStep.setStep(step);
            return reactStep;
        } catch (Exception e) {
            log.error("解析 ReAct step 失败", e);
            return null;
        }
    }

    /**
     * 从 LLM 返回内容中提取纯 JSON
     * 兼容 \`\`\`json ... \`\`\` 包裹的情况
     */
    private String extractJson(String raw) {
        raw = raw.trim();
        if (raw.startsWith("```json")) {
            raw = raw.substring(7);
        } else if (raw.startsWith("```")) {
            raw = raw.substring(3);
        }
        if (raw.endsWith("```")) {
            raw = raw.substring(0, raw.length() - 3);
        }
        return raw.trim();
    }

    /**
     * 根据 Action.type 路由到对应的处理器执行
     */
    private String executeAction(ReActStep.Action action) {
        String type = action.getType();
        ActionHandler handler = handlers.get(type);

        if (handler == null) {
            return "未找到 action type=[" + type + "] 的处理器，支持的 type: "
                   + handlers.keySet();
        }

        try {
            return handler.execute(action);
        } catch (Exception e) {
            log.error("执行 Action 失败: type={}, desc={}", type, action.getDescription(), e);
            return "Action 执行失败: " + e.getMessage();
        }
    }

    /**
     * ReAct 执行的结果包装
     */
    public record ReActResult(boolean solved, List<ReActStep> history, String finalAnswer) {}
}
```

### 5.3 Action 处理器接口与实现示例

```java
package com.example.react.agent;

/**
 * Action 处理器接口
 * 每种 action.type 对应一个实现，方便扩展
 */
@FunctionalInterface
public interface ActionHandler {
    String execute(ReActStep.Action action) throws Exception;
}
```

```java
package com.example.react.agent;

import java.io.BufferedReader;
import java.io.InputStreamReader;
import java.util.concurrent.TimeUnit;

/**
 * 在隔离沙箱中执行 Java 代码片段的处理器
 */
public class CodeSandboxHandler implements ActionHandler {

    @Override
    public String execute(ReActStep.Action action) throws Exception {
        String code = action.getInput();
        // 生产环境请使用 Docker 沙箱或专门的代码执行服务
        Process process = new ProcessBuilder("jshell", "--execution", "local", "-")
                .redirectErrorStream(true)
                .start();

        process.getOutputStream().write(code.getBytes());
        process.getOutputStream().close();

        BufferedReader reader = new BufferedReader(new InputStreamReader(process.getInputStream()));
        StringBuilder output = new StringBuilder();
        String line;
        while ((line = reader.readLine()) != null) {
            output.append(line).append("\n");
        }

        boolean finished = process.waitFor(30, TimeUnit.SECONDS);
        if (!finished) {
            process.destroyForcibly();
            return "代码执行超时（30秒）";
        }

        return output.toString();
    }
}
```

```java
package com.example.react.agent;

import org.springframework.jdbc.core.JdbcTemplate;

/**
 * SQL 查询执行处理器
 */
public class SqlQueryHandler implements ActionHandler {

    private final JdbcTemplate jdbcTemplate;

    public SqlQueryHandler(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    @Override
    public String execute(ReActStep.Action action) throws Exception {
        String sql = action.getInput();

        // 安全防护：只允许 SELECT 和 EXPLAIN
        String upperSql = sql.trim().toUpperCase();
        if (!upperSql.startsWith("SELECT") && !upperSql.startsWith("EXPLAIN")) {
            return "安全限制：仅允许执行 SELECT 和 EXPLAIN 语句";
        }

        try {
            var rows = jdbcTemplate.queryForList(sql);
            if (rows.size() > 100) {
                return "查询返回 " + rows.size() + " 行（仅展示前 10 行）:\n"
                       + rows.subList(0, 10).toString();
            }
            return rows.toString();
        } catch (Exception e) {
            return "SQL 执行异常: " + e.getMessage();
        }
    }
}
```

### 5.4 使用示例

```java
package com.example.react.agent;

/**
 * 完整使用示例：排查线上慢 SQL 问题
 */
public class ReActDemo {

    public static void main(String[] args) {
        // 1. 初始化引擎
        LLMClient llmClient = new OpenAIClient("your-api-key", "gpt-4");
        ReActEngine engine = new ReActEngine(llmClient);

        // 2. 注册 Action 处理器
        engine.registerHandler("sql_query", new SqlQueryHandler(jdbcTemplate));
        engine.registerHandler("code_execute", new CodeSandboxHandler());
        engine.registerHandler("search", new GoogleSearchHandler());
        engine.registerHandler("analyze", action -> {
            // 纯分析型 Action，不需要外部执行
            return "分析已完成: " + action.getDescription();
        });

        // 3. 启动 ReAct 循环
        String problem = """
            订单列表查询接口 /api/orders 在数据量超过 500 万后
            响应时间从 200ms 下降到 8 秒，请排查并给出优化方案。
            """;

        String context = """
            技术栈：Java 17, Spring Boot 3.x, MyBatis-Plus 3.5.x, MySQL 8.0
            表结构：orders(id, user_id, status, amount, created_at)
            当前索引：PRIMARY KEY(id), KEY idx_user_id(user_id)
            """;

        ReActEngine.ReActResult result = engine.solve(problem, context);

        // 4. 输出完整排查过程
        System.out.println("问题是否解决: " + result.solved());
        System.out.println("总步数: " + result.history().size());
        System.out.println("排查过程:");
        for (ReActStep step : result.history()) {
            System.out.printf("[Step %d] Thought: %s%n", step.getStep(), step.getThought());
            System.out.printf("[Step %d] Action: %s%n", step.getStep(), step.getAction().getDescription());
            System.out.printf("[Step %d] Observation: %s%n%n", step.getStep(), step.getObservation());
        }
        System.out.println("最终答案: " + result.finalAnswer());
    }
}
```

> 有了这套框架，你就把一个"只能聊天"的 LLM，变成了一个能**真的干活**的 AI Agent。

---

## 六、ReAct vs 普通 CoT vs 普通对话对比

| 维度 | 普通对话 | CoT (Chain of Thought) | ReAct |
|------|---------|----------------------|-------|
| **推理方式** | 一步生成答案 | "Let's think step by step"，线性推理链 | 推理-行动-观察 循环交互 |
| **外部信息获取** | 仅依赖模型内部知识 | 仅依赖模型内部知识 | 可通过 Action 调用外部工具/搜索/执行代码 |
| **事实准确性** | 低（幻觉率高） | 中（推理链可能基于错误前提） | 高（每步有 Observation 作为事实锚点） |
| **复杂问题解决率** | 约 35% | 约 55% | 约 75%（Google 论文数据） |
| **Token 消耗** | 低（单次生成） | 中（2-3倍普通对话） | 高（5-10倍，取决于循环轮数） |
| **适用场景** | 简单问答、闲聊 | 数学推理、逻辑题 | 排查问题、技术调研、代码重构、决策分析 |
| **工具调用能力** | 无 | 无 | 有（原生支持 function calling） |
| **可观测性** | 低（只有最终答案） | 中（有推理中间步骤） | 高（每步的 Thought/Action/Observation 全记录） |
| **陷入循环风险** | 无 | 低 | 较高（需要设置 max_steps 和循环检测） |

### 一张图看清三者的区别

```
普通对话:   用户提问 ──────→ LLM ──────→ 答案
                                  （凭记忆瞎猜）

CoT:        用户提问 ──→ LLM ──→ 步骤1推理 ──→ 步骤2推理 ──→ ... ──→ 答案
                                （推理基于内部知识，可能起点就错）

ReAct:      用户提问 ──→ Thought1 ──→ Action1 ──→ Observation1
                              ↓                        ↓
                        Thought2 ←─────────────────┘
                              ↓
                         Action2 ──→ Observation2
                              ↓
                        Thought3 → 答案
                   （每一步有外部验证，事实准确）
```

---

## 七、ReAct 的局限性与应对策略

### 7.1 容易陷入死循环

**典型场景**：

```
Thought: SQL 超时，可能是索引问题
Action: EXPLAIN SELECT ... WHERE status = 'PENDING'
Observation: 使用 idx_status 索引，扫描 1000 行，没问题
Thought: 那可能是网络问题
Action: ping 数据库服务器
Observation: 延迟 < 1ms，没问题
Thought: 那可能是索引问题（又绕回来了！）
Action: EXPLAIN SELECT ... WHERE status = 'PENDING' （重复）
...
```

**应对策略**：

1. **设置最大步数上限**（如 max_steps=10），超限强制终止
2. **相似度检测**：当连续两步的 Thought 语义相似度 > 0.85 时，提醒 LLM 换方向
3. **注入"外部分析师"**：当检测到循环时，额外调用一次 LLM 扮演监督者，说："你似乎陷入循环了，考虑一下其他可能性，比如锁竞争、网络抖动、配置变更？"

### 7.2 Token 消耗巨大

ReAct 的每一轮都在追加上下文到 prompt 中，步数越多 prompt 越长，成本线性增长。

| 使用模式 | 平均 Token 消耗 |
|---------|----------------|
| 普通对话 | 500 tokens |
| CoT | 1,200 tokens |
| ReAct (3 步) | 3,000 tokens |
| ReAct (7 步) | 12,000 tokens |
| ReAct (10 步) | 25,000+ tokens |

**应对策略**：

1. **压缩历史记录**：超过 5 步后，用 LLM 对历史做摘要压缩，保留关键发现去掉冗余
2. **按需使用**：简单问题用普通对话，中等复杂度用 CoT，只有复杂排查/决策才用 ReAct
3. **早停机制**：当 Observation 的置信度超过阈值时提前终止

### 7.3 Action 执行的副作用

在案例 1 中，如果 LLM 建议 "DELETE FROM login_history WHERE ..."，直接执行可能导致灾难性后果。

**应对策略**：

1. **只读优先**：限制 Action 只能执行只读操作（SELECT, EXPLAIN, read_file 等）
2. **人工确认**：凡写操作（CREATE INDEX, UPDATE, DELETE）都需要人工确认后才执行
3. **沙箱隔离**：所有代码执行在 Docker 沙箱中，限制网络访问和资源配额

---

## 八、写在最后

ReAct 不是一个什么革命性的新技术——它的本质就是**把人类解决问题的方式教给了 AI**。但有趣的是，当我们让 AI 学会像人类一样"边想边做、边做边看"时，它解决复杂问题的能力确实有了质的飞跃。

这篇文章里的三套 ReAct Prompt 模板和 Java 解析框架，你可以直接复制到你的项目中试试。你会发现，当你让 LLM 不再只是"回答问题"，而是"执行任务"时，它能做的远超你的预期。

> **下一篇文章**，我将深入聊一个困扰所有 AI 应用开发者的问题——**Prompt 版本管理**。当你的系统里有几十个精心调优的 Prompt 模板，如何像管理代码一样管理它们？版本控制、A/B 测试、灰度发布、回滚机制、效果监控——这些软件工程的实践，一样适用于 Prompt 工程。敬请期待。

---

*本文是「Prompt Engineering 与 AI 交互艺术」系列的第 35 篇。关注我，获取更多 Java + AI 的实战干货。*
