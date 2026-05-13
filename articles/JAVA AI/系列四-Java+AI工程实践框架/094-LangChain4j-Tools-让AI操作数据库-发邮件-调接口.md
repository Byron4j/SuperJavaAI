# LangChain4j Tools（函数调用）：让 AI 操作数据库、发邮件、调接口，给 AI 一双能干活的手

> 会聊天的 AI 千篇一律，能干活的 AI 万里挑一。上一篇文章我们解决了 AI 的"记忆"问题，今天我们来给 AI 装上"手"——通过 LangChain4j 的 Tools 机制，让大模型直接操作你的数据库、发送邮件、调用内部接口。

---

## 一、开篇：只会聊天的 AI 是"残疾人"

我们来看两个典型场景：

**场景 A**：用户问"帮我查一下订单 #12345 的状态"。如果一个 AI 只能回答"很抱歉，我无法访问您的订单系统，建议您登录官网查询"，那它就是**会聊天但没用**。

**场景 B**：用户问"帮我查一下订单 #12345 的状态"。AI 回答："您的订单 #12345 已发货，物流单号为 SF1192837465，预计明天上午送达。"——这是因为它**调用了你公司的订单查询 API**。

差距就在于：**Tools（函数调用）**。

2023 年 OpenAI 发布 Function Calling 以来，函数调用已经成为各大模型的标配能力。LangChain4j 对函数调用的封装极其优雅——**一个 `@Tool` 注解搞定一切**。但优雅的背后藏着不少坑：工具选择策略、错误处理、安全控制……今天一次性聊透。

---

## 二、核心原理：AI 如何"使用工具"？

在你写代码之前，先搞懂这 4 个步骤：

```
1. 用户发消息：帮我查一下北京今天的天气

2. LangChain4j 将消息 + 工具定义（JSON Schema）发给 LLM
   └── 工具定义包含：函数名、描述、参数、参数类型

3. LLM 判断需要调用工具，返回 ToolExecutionRequest：
   {
     "name": "getWeather",
     "arguments": "{ \"city\": \"北京\" }"
   }

4. LangChain4j 执行工具 → 获取结果 → 再次调用 LLM 生成最终回复
   └── 最终回复：北京今天晴，气温 18-28°C
```

**关键**：模型本身不执行工具，它只负责"决定调用哪个工具、传什么参数"。**执行是你的事**——LangChain4j 帮你把这个流程自动化了。

---

## 三、第一个 Tool：5 行代码让 AI 会查天气

### 3.1 定义工具类

```java
import dev.langchain4j.agent.tool.Tool;

public class WeatherTools {

    @Tool("查询指定城市的实时天气，返回天气状况和温度范围")
    public String getWeather(
            @P("城市名称，例如：北京、上海、广州") String city) {
        // 实际项目中这里调用天气 API
        // 这里用模拟数据
        Map<String, String> weatherMap = Map.of(
            "北京", "晴，18-28°C，湿度40%",
            "上海", "多云，22-30°C，湿度65%",
            "广州", "雷阵雨，25-33°C，湿度80%"
        );
        return weatherMap.getOrDefault(city, "未找到该城市的天气数据");
    }
}
```

### 3.2 注册工具并对话

```java
import dev.langchain4j.model.openai.OpenAiChatModel;
import dev.langchain4j.service.AiServices;

public class WeatherDemo {

    public interface Assistant {
        String chat(String message);
    }

    public static void main(String[] args) {
        OpenAiChatModel model = OpenAiChatModel.builder()
                .apiKey(System.getenv("OPENAI_API_KEY"))
                .modelName("gpt-4o")
                .build();

        Assistant assistant = AiServices.builder(Assistant.class)
                .chatLanguageModel(model)
                .tools(new WeatherTools())  // 注册工具
                .build();

        String response = assistant.chat("北京今天天气怎么样？");
        System.out.println(response);
    }
}
```

**输出**：
```
北京今天天气晴，气温 18-28°C，湿度 40%，非常适合户外活动。
```

注意：AI 不只是简单的"返回数据"，它**基于工具返回的结果做了自然语言的润色和推理**——这就是 Tools 的威力。

---

## 四、@Tool 注解深度解析

### 4.1 注解参数

```java
@Tool(name = "get_weather",              // 工具名称（默认取方法名）
      value = "查询天气信息，...")        // 工具描述（传给 LLM 的 Schema）
public String getWeather(@P("城市名称") String city) { ... }
```

| 参数 | 必填 | 说明 |
|------|------|------|
| `name` | 否 | 工具名称，不写默认用方法名 |
| `value` | 否（强烈建议写） | 工具描述，**这是 LLM 决定是否调用和如何调用的关键信息** |

**重要**：`value` 的描述质量直接影响 LLM 的工具选择准确率。写得太含糊，LLM 可能在该调用时不调用；写得太泛，LLM 可能在不该调用时调用。

### 4.2 @P 注解：参数描述

```java
public String transferMoney(
    @P("付款人的用户ID") String fromUserId,
    @P("收款人的用户ID") String toUserId,
    @P("转账金额，单位：元，最小0.01") double amount,
    @P("转账备注，可选") @ToolMemoryId String note
) { ... }
```

- `@P`：参数描述，LLM 据此判断该传什么值
- `@ToolMemoryId`：标记该参数用于区分工具调用来源（多用户场景有用）

### 4.3 支持的数据类型

LangChain4j 可以自动将 LLM 的 JSON 参数反序列化为 Java 类型：

| Java 类型 | 示例 |
|-----------|------|
| `String` | `"北京"` |
| `int/long/double` | `100`, `3.14` |
| `boolean` | `true/false` |
| `enum` | `OrderStatus.SHIPPED` |
| 自定义 POJO | `{ "name": "张三", "age": 25 }` |
| `List<T>` | `["北京", "上海"]` |

```java
// 复杂参数类型示例
public record SearchRequest(String keyword, int page, int size, String[] tags) {}

@Tool("搜索文章")
public List<Article> searchArticles(@P("搜索请求") SearchRequest request) {
    return articleService.search(request);
}
```

---

## 五、多工具场景：让 AI 同时操控 3 个系统

下面是一个完整的多工具实战案例——让 AI 同时具备**查数据库、发邮件、创建 Jira 工单**的能力。

### 5.1 数据库查询工具

```java
import dev.langchain4j.agent.tool.Tool;
import dev.langchain4j.agent.tool.P;
import java.sql.*;

public class DatabaseTools {

    private final DataSource dataSource;

    public DatabaseTools(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    @Tool("查询订单信息，可以按订单号、用户ID或日期范围查询")
    public String queryOrders(
            @P("订单号，格式如 ORD-2024-xxxx，可选") String orderId,
            @P("用户ID，可选") String userId,
            @P("开始日期，格式 yyyy-MM-dd，可选") String startDate,
            @P("结束日期，格式 yyyy-MM-dd，可选") String endDate) {

        StringBuilder sql = new StringBuilder(
            "SELECT order_id, user_id, status, amount, created_at FROM orders WHERE 1=1");
        List<Object> params = new ArrayList<>();

        if (orderId != null && !orderId.isEmpty()) {
            sql.append(" AND order_id = ?");
            params.add(orderId);
        }
        if (userId != null && !userId.isEmpty()) {
            sql.append(" AND user_id = ?");
            params.add(userId);
        }
        if (startDate != null && !startDate.isEmpty()) {
            sql.append(" AND created_at >= ?");
            params.add(startDate);
        }
        if (endDate != null && !endDate.isEmpty()) {
            sql.append(" AND created_at <= ?");
            params.add(endDate);
        }
        sql.append(" LIMIT 20");

        try (Connection conn = dataSource.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql.toString())) {

            for (int i = 0; i < params.size(); i++) {
                ps.setObject(i + 1, params.get(i));
            }

            ResultSet rs = ps.executeQuery();
            StringBuilder result = new StringBuilder("查询结果：\n");
            int count = 0;
            while (rs.next()) {
                count++;
                result.append(String.format(
                    "- 订单号: %s, 用户ID: %s, 状态: %s, 金额: ¥%.2f, 时间: %s\n",
                    rs.getString("order_id"),
                    rs.getString("user_id"),
                    rs.getString("status"),
                    rs.getDouble("amount"),
                    rs.getString("created_at")
                ));
            }
            if (count == 0) {
                result.append("未找到匹配的订单。");
            }
            return result.toString();
        } catch (SQLException e) {
            return "数据库查询失败: " + e.getMessage();
        }
    }
}
```

### 5.2 邮件发送工具

```java
import jakarta.mail.*;
import jakarta.mail.internet.*;
import java.util.Properties;

public class EmailTools {

    private final String smtpHost;
    private final String smtpPort;
    private final String username;
    private final String password;

    public EmailTools(String smtpHost, String smtpPort,
                      String username, String password) {
        this.smtpHost = smtpHost;
        this.smtpPort = smtpPort;
        this.username = username;
        this.password = password;
    }

    @Tool("发送邮件给指定收件人")
    public String sendEmail(
            @P("收件人邮箱地址") String to,
            @P("邮件主题/标题") String subject,
            @P("邮件正文内容") String body) {

        Properties props = new Properties();
        props.put("mail.smtp.host", smtpHost);
        props.put("mail.smtp.port", smtpPort);
        props.put("mail.smtp.auth", "true");
        props.put("mail.smtp.starttls.enable", "true");

        Session session = Session.getInstance(props, new Authenticator() {
            @Override
            protected PasswordAuthentication getPasswordAuthentication() {
                return new PasswordAuthentication(username, password);
            }
        });

        try {
            Message message = new MimeMessage(session);
            message.setFrom(new InternetAddress(username));
            message.setRecipients(Message.RecipientType.TO,
                    InternetAddress.parse(to));
            message.setSubject(subject);
            message.setText(body);
            Transport.send(message);
            return "邮件已成功发送至 " + to;
        } catch (MessagingException e) {
            return "邮件发送失败: " + e.getMessage();
        }
    }
}
```

### 5.3 Jira 工单创建工具

```java
public class JiraTools {

    private final String jiraUrl;
    private final String apiToken;
    private final HttpClient httpClient;

    public JiraTools(String jiraUrl, String apiToken) {
        this.jiraUrl = jiraUrl;
        this.apiToken = apiToken;
        this.httpClient = HttpClient.newHttpClient();
    }

    @Tool("在Jira中创建一个新的Issue工单")
    public String createJiraIssue(
            @P("项目Key，例如 PROJ") String projectKey,
            @P("工单摘要/标题") String summary,
            @P("工单详细描述") String description,
            @P("工单类型，可选值: Bug/Task/Story，默认为Task") String issueType) {

        if (issueType == null || issueType.isEmpty()) {
            issueType = "Task";
        }

        try {
            ObjectNode payload = JsonNodeFactory.instance.objectNode();
            ObjectNode fields = payload.putObject("fields");
            fields.putObject("project").put("key", projectKey);
            fields.put("summary", summary);
            fields.put("description", description);
            fields.putObject("issuetype").put("name", issueType);

            HttpRequest request = HttpRequest.newBuilder()
                    .uri(URI.create(jiraUrl + "/rest/api/2/issue"))
                    .header("Authorization", "Basic " + Base64.getEncoder()
                            .encodeToString(("email:" + apiToken).getBytes()))
                    .header("Content-Type", "application/json")
                    .POST(HttpRequest.BodyPublishers.ofString(payload.toString()))
                    .build();

            HttpResponse<String> response = httpClient.send(request,
                    HttpResponse.BodyHandlers.ofString());

            if (response.statusCode() == 201) {
                JsonNode json = new ObjectMapper().readTree(response.body());
                String issueKey = json.get("key").asText();
                return "Jira工单创建成功！工单号: " + issueKey +
                       ", 链接: " + jiraUrl + "/browse/" + issueKey;
            } else {
                return "Jira工单创建失败，HTTP " + response.statusCode() +
                       ": " + response.body();
            }
        } catch (Exception e) {
            return "Jira工单创建异常: " + e.getMessage();
        }
    }
}
```

### 5.4 组装：一个能干活的 AI 助理

```java
public class MultiToolAssistant {

    public interface Agent {
        String chat(String message);
    }

    public static void main(String[] args) {
        OpenAiChatModel model = OpenAiChatModel.builder()
                .apiKey(System.getenv("OPENAI_API_KEY"))
                .modelName("gpt-4o")  // 多工具场景建议用 gpt-4o
                .build();

        DataSource ds = createDataSource();
        Agent agent = AiServices.builder(Agent.class)
                .chatLanguageModel(model)
                .tools(new DatabaseTools(ds))
                .tools(new EmailTools("smtp.company.com", "587",
                        "noreply@company.com", System.getenv("SMTP_PWD")))
                .tools(new JiraTools("https://jira.company.com",
                        System.getenv("JIRA_API_TOKEN")))
                .build();

        // 场景1：查询订单
        System.out.println("=== 场景1：查订单 ===");
        System.out.println(agent.chat("帮我查一下用户 U1001 的所有订单"));

        // 场景2：发邮件
        System.out.println("\n=== 场景2：发邮件 ===");
        System.out.println(agent.chat(
            "给 admin@company.com 发一封邮件，主题是'系统维护通知'，" +
            "内容是'今晚22:00-24:00进行系统维护，请知悉。'"));

        // 场景3：创建Jira工单
        System.out.println("\n=== 场景3：创建工单 ===");
        System.out.println(agent.chat(
            "帮我在 PROJ 项目下创建一个Bug工单，" +
            "标题是'订单查询接口超时'，" +
            "描述是'最近一周订单查询接口P99耗时超过5秒，需要优化数据库索引'"));
    }
}
```

---

## 六、工具选择策略

当注册了多个工具后，LLM 需要**选择调用哪个**（或不调用）。LangChain4j 提供了几种选择策略：

### 6.1 默认策略：AUTO

```java
AiServices.builder(Assistant.class)
    .chatLanguageModel(model)
    .tools(tool1, tool2, tool3)
    // 默认 AUTO 模式：LLM 自己决定是否调用、调用哪个
    .build();
```

### 6.2 强制策略：REQUIRED

```java
AiServices.builder(Assistant.class)
    .chatLanguageModel(model)
    .tools(tool1, tool2, tool3)
    .toolChoice(ToolChoice.REQUIRED)  // 强制 LLM 必须调用工具
    .build();
```

适用场景：当你的 Assistant 必须通过工具获取数据再回复时。

### 6.3 指定工具：固定调用

```java
AiServices.builder(Assistant.class)
    .chatLanguageModel(model)
    .tools(tool1, tool2, tool3)
    .toolChoice(ToolChoice.from("getWeather"))  // 只允许调用指定工具
    .build();
```

### 6.4 工具描述的最佳实践

```java
// ❌ 差的描述
@Tool("查询")
public String query(String keyword) { ... }

// ✅ 好的描述
@Tool("根据关键词搜索产品信息，返回匹配产品的名称、价格和库存状态。" +
      "当用户询问产品/商品/库存相关信息时使用此工具。")
public String searchProducts(
    @P("搜索关键词，支持产品名称或SKU编码") String keyword
) { ... }
```

**三条黄金法则**：
1. **说清楚什么时候用**：明确告诉 LLM 在什么场景下该调用此工具
2. **说清楚返回什么**：让 LLM 理解返回数据的格式和含义
3. **写示例参数值**：`@P("城市名称，例如：北京、上海、广州")`

---

## 七、错误处理与异常场景

### 7.1 工具执行异常

默认情况下，工具抛出异常时，异常信息会作为工具返回结果传给 LLM。LLM 通常会向用户道歉或建议替代方案。但更好的做法是在工具内部做好异常处理：

```java
@Tool("执行SQL查询")
public String executeQuery(@P("SQL查询语句") String sql) {
    try {
        // 执行查询
        ResultSet rs = statement.executeQuery(sql);
        return formatResultSet(rs);
    } catch (SQLException e) {
        // 返回结构化的错误信息，帮助 LLM 理解问题
        return "SQL执行错误: " + e.getSQLState() + " - " + e.getMessage() +
               ". 请检查表名和字段名是否正确。";
    }
}
```

### 7.2 工具调用超时

```java
AiServices.builder(Assistant.class)
    .chatLanguageModel(model)
    .tools(tools)
    .toolProvider((toolExecutionRequest, memoryId) -> {
        // 自定义工具执行逻辑，加入超时控制
        return CompletableFuture
            .supplyAsync(() -> executeTool(toolExecutionRequest))
            .orTimeout(10, TimeUnit.SECONDS)
            .exceptionally(ex -> "工具执行超时，请稍后重试");
    })
    .build();
```

### 7.3 工具被"幻觉调用"

LLM 有时候会"幻觉"出一个不存在的方法名，或者传一个类型错误的参数。LangChain4j 会捕获这些错误并返回给 LLM，让它重新尝试。

```java
// 你可以在日志中监控这些重试
AiServices.builder(Assistant.class)
    .chatLanguageModel(model)
    .tools(tools)
    .maxRetries(3)  // 最多重试 3 次
    .build();
```

---

## 八、安全控制：别让 AI 把你的数据库删了

工具调用是 AI 与真实系统交互的**唯一通道**，也是安全风险的**最大敞口**。

### 8.1 权限校验

```java
public class SecureDatabaseTools {

    @Tool("查询订单信息")
    public String queryOrders(
            @P("用户ID") String userId,
            @ToolMemoryId Object memoryId) {

        // 从 memoryId 反查真实用户身份
        String currentUser = userContextHolder.getCurrentUser(memoryId);

        // 用户只能查自己的订单
        if (!currentUser.equals(userId) && !isAdmin(currentUser)) {
            return "权限不足：您只能查询自己的订单";
        }

        return orderService.queryByUserId(userId);
    }
}
```

### 8.2 SQL 注入防护

```java
@Tool("查询订单")
public String queryOrders(@P("用户ID") String userId) {
    // ✅ 正确：使用参数化查询
    String sql = "SELECT * FROM orders WHERE user_id = ?";

    // ❌ 绝对不要这样做：
    // String sql = "SELECT * FROM orders WHERE user_id = '" + userId + "'";

    return jdbcTemplate.query(sql, ps -> ps.setString(1, userId), rowMapper);
}
```

### 8.3 敏感操作二次确认

```java
@Tool("执行退款操作")
public String refund(
        @P("订单号") String orderId,
        @P("退款金额，单位元") double amount) {

    // 高风险操作：记录审计日志
    auditLogger.log("REFUND_REQUEST", orderId, amount);

    // 金额超过 1000 元需要上级审批
    if (amount > 1000) {
        return "退款金额超过 1000 元，需要上级审批。已自动创建审批单 " +
               createApproval(orderId, amount);
    }

    return paymentGateway.refund(orderId, amount);
}
```

### 8.4 工具白名单与限流

```yaml
# application.yml
langchain4j:
  tools:
    # 只允许注册这些工具类
    allowed-tools:
      - WeatherTools
      - DatabaseTools
      - EmailTools
    # 禁止这些高危操作
    blocked-patterns:
      - "DROP"
      - "DELETE"
      - "TRUNCATE"
    # 限流：每个用户每分钟最多 30 次工具调用
    rate-limit:
      enabled: true
      max-calls-per-minute: 30
```

```java
@Configuration
public class ToolSecurityConfig {

    @Bean
    public ToolExecutionInterceptor rateLimiter() {
        return new RateLimitingInterceptor(30, Duration.ofMinutes(1));
    }
}

// 拦截器实现
public class RateLimitingInterceptor implements ToolExecutionInterceptor {

    private final ConcurrentHashMap<String, RateLimiter> limiters = new ConcurrentHashMap<>();

    @Override
    public String onToolExecution(ToolExecutionRequest request,
                                   ChatMemory memory) {
        String userId = memory.id().toString();
        RateLimiter limiter = limiters.computeIfAbsent(userId,
            k -> RateLimiter.create(maxCallsPerMinute / 60.0));
        if (!limiter.tryAcquire()) {
            return "操作过于频繁，请稍后再试。";
        }
        return null; // null 表示放行
    }
}
```

---

## 九、实战架构：AI 工具调用流水线

```java
@SpringBootApplication
public class AIAgentApplication {

    public static void main(String[] args) {
        SpringApplication.run(AIAgentApplication.class, args);
    }

    @Bean
    public Assistant aiAgent(OpenAiChatModel model,
                              WeatherTools weather,
                              DatabaseTools database,
                              EmailTools email,
                              JiraTools jira) {
        return AiServices.builder(Assistant.class)
                .chatLanguageModel(model)
                .tools(weather, database, email, jira)
                .toolChoice(ToolChoice.AUTO)          // 自动选择工具
                .maxRetries(2)                         // 失败重试 2 次
                .chatMemoryProvider(memoryId ->
                        TokenWindowChatMemory.withMaxTokens(4000, tokenizer)
                )
                .systemMessageProvider(memoryId ->
                    "你是一个企业助理，可以查询订单、发送邮件、创建Jira工单。" +
                    "所有操作必须使用提供的工具，不得虚构数据。" +
                    "处理资金相关操作前务必确认。"
                )
                .build();
    }

    // 统一的异常处理切面
    @Around("@annotation(dev.langchain4j.agent.tool.Tool)")
    public Object handleToolException(ProceedingJoinPoint pjp) {
        try {
            return pjp.proceed();
        } catch (Exception e) {
            log.error("工具执行异常: {}", pjp.getSignature().getName(), e);
            return "系统异常: " + e.getClass().getSimpleName() +
                   "，请联系管理员。";
        }
    }
}
```

---

## 十、限制与考量

### Token 消耗
每个工具的定义（JSON Schema）会占用几百个 Token。如果你注册了 10+ 工具，光工具定义就吃掉上千 Token。**只注册必要的工具**。

### 模型兼容性
函数调用需要模型原生支持。目前主要支持的模型：
- **OpenAI**：gpt-4o、gpt-4o-mini、gpt-4-turbo 全系
- **Anthropic**：Claude 3+ 全系（通过 Tool Use）
- **本地模型**：需确认是否经过 Function Calling 微调

### 并发安全
工具实例默认是**单例**。如果有状态，需要自己处理并发。

---

## 十一、写在最后

Tools 是 LangChain4j 中最"生产力"的特性——它让 AI 从"会聊天的玩具"变成"能干活的工具"。一个带 Tools 的 AI Agent 可以：

- 查订单、改库存、管用户
- 发邮件、发短信、发通知
- 调 API、建工单、跑报表
- 甚至操作 IoT 设备、控制智能家居

你只需要写好 Java 方法，加个 `@Tool` 注解，剩下的交给 LangChain4j。

但记住：**权力越大，责任越大**。工具调用的每一条链路都要考虑权限、限流、审计。

---

**下一篇预告**：下一篇我们进入 LangChain4j RAG 的进阶实战——教你怎么**自定义 DocumentSplitter 和 EmbeddingStore**，打造专属于你业务的检索增强生成方案。别再死磕默认的递归分割了，如果你的文档是代码/表格/法律文书，通用分割策略会让你满地找牙。

> 觉得有用的话，点赞收藏转发三连！有问题评论区随时交流。
