# Spring AI Prompt Template：模板化管理与参数绑定实战，Prompt不再散落在代码各处

> 你的Prompt散落在50个Service类的字符串里，想改一句措辞就要全局搜索——用Prompt Template来管理。

---

## 一、痛点：Prompt管理的灾难现场

先来看一个真实的噩梦场景。假设你维护一个电商系统的客服AI模块，代码里到处是这样的写法：

```java
// OrderService.java
String prompt = "你是一个电商客服助手，请根据以下订单信息回答用户问题：" +
                "订单号：" + order.getOrderId() + "，金额：" + order.getAmount() +
                "，状态：" + order.getStatus() + "，用户问题：" + userQuestion;

// RefundService.java
String prompt = "你是一个电商退款处理助手，请根据以下退款单信息回答用户问题：" +
                "退款单号：" + refund.getRefundId() + "，原订单：" + refund.getOrderId() +
                "，退款金额：" + refund.getAmount() + "，用户问题：" + userQuestion;

// ProductRecommendService.java
String prompt = "你是商品推荐专家。用户最近浏览了以下商品：" + 
                productHistory + "，请推荐5款相似商品。";
```

当产品经理突然说"把'用户问题'改成'用户咨询'"——你就要在50个Service类里做全局搜索替换，改错一处就是线上事故。更糟糕的是：

- Prompt措辞的A/B测试无从下手
- 多语言Prompt维护失控
- 不同环境的Prompt差异无法管理
- 没有版本记录，不知道谁改了什么

**Spring AI 的 PromptTemplate 就是为了解决这些问题而生的。**

---

## 二、PromptTemplate 基础用法

### 2.1 添加依赖

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter</artifactId>
    <version>1.0.0</version>
</dependency>
```

### 2.2 最简单的模板

```java
@Service
public class OrderAssistantService {

    private final ChatClient chatClient;

    public OrderAssistantService(ChatClient.Builder chatClientBuilder) {
        this.chatClient = chatClientBuilder.build();
    }

    public String answerOrderQuestion(Order order, String userQuestion) {
        String template = """
            你是一个专业的电商客服助手。
            请根据以下订单信息，礼貌地回答用户的问题。
            
            ## 订单信息
            - 订单号：{orderId}
            - 商品名称：{productName}
            - 订单金额：{amount}元
            - 订单状态：{status}
            - 下单时间：{createTime}
            
            ## 用户问题
            {question}
            
            ## 回答要求
            1. 语气友好、专业
            2. 如果涉及退款金额，请精确到分
            3. 如果订单状态不支持操作，请说明原因并给出建议
            """;

        PromptTemplate promptTemplate = new PromptTemplate(template);
        
        // 绑定参数
        promptTemplate.add("orderId", order.getOrderId());
        promptTemplate.add("productName", order.getProductName());
        promptTemplate.add("amount", order.getAmount());
        promptTemplate.add("status", order.getStatus());
        promptTemplate.add("createTime", order.getCreateTime());
        promptTemplate.add("question", userQuestion);

        Prompt prompt = promptTemplate.create();
        ChatResponse response = chatClient.call(prompt);
        return response.getResult().getOutput().getContent();
    }
}
```

### 2.3 Map批量绑定参数

逐个 `add` 太啰嗦？用 Map 批量绑定：

```java
public String answerWithMap(Order order, String userQuestion) {
    Map<String, Object> params = Map.of(
        "orderId", order.getOrderId(),
        "productName", order.getProductName(),
        "amount", order.getAmount(),
        "status", order.getStatus(),
        "createTime", order.getCreateTime(),
        "question", userQuestion
    );

    PromptTemplate promptTemplate = new PromptTemplate(template);
    promptTemplate.addAll(params);
    
    return chatClient.call(promptTemplate.create())
        .getResult().getOutput().getContent();
}
```

### 2.4 使用Message对象构建

Spring AI的Prompt其实是一个`List<Message>`，你可以更精细地控制每条消息的角色：

```java
public String answerWithSystemPrompt(Order order, String userQuestion) {
    // 系统级指令
    String systemTemplate = "你是电商客服助手，当前时间为{currentTime}。";
    PromptTemplate systemPt = new PromptTemplate(systemTemplate);
    systemPt.add("currentTime", LocalDateTime.now().toString());
    Message systemMessage = systemPt.createMessage();

    // 用户消息
    String userTemplate = "订单{orderId}，{question}";
    PromptTemplate userPt = new PromptTemplate(userTemplate);
    userPt.add("orderId", order.getOrderId());
    userPt.add("question", userQuestion);
    Message userMessage = userPt.createMessage();

    Prompt prompt = new Prompt(List.of(systemMessage, userMessage));
    return chatClient.call(prompt).getResult().getOutput().getContent();
}
```

---

## 三、资源文件管理 Prompt（.st 扩展名）

把Prompt字符串硬编码在Java代码里，只是把问题从"散落各处"变成了"集中一处"，本质上还是在代码里。更好的做法是**放到资源文件中**。

### 3.1 创建 .st 模板文件

Spring AI 使用 [StringTemplate](https://www.stringtemplate.org/) 作为模板引擎，模板文件使用 `.st` 扩展名。在 `src/main/resources/prompts/` 目录下创建：

```st
// src/main/resources/prompts/order-assistant.st
你是一个专业的电商客服助手。
请根据以下订单信息，礼貌地回答用户的问题。

## 订单信息
- 订单号：{orderId}
- 商品名称：{productName}
- 订单金额：{amount}元
- 订单状态：{status}
- 下单时间：{createTime}

## 用户问题
{question}

## 回答要求
1. 语气友好、专业
2. 如果涉及退款金额，请精确到分
3. 如果订单状态为"已发货"，提醒用户预计{deliveryDays}天内送达
```

### 3.2 从Classpath加载模板

```java
@Service
public class OrderAssistantService {

    private final ChatClient chatClient;
    private final ResourceLoader resourceLoader;

    public OrderAssistantService(ChatClient.Builder chatClientBuilder,
                                  ResourceLoader resourceLoader) {
        this.chatClient = chatClientBuilder.build();
        this.resourceLoader = resourceLoader;
    }

    public String answer(Order order, String userQuestion) {
        // 从classpath加载模板
        Resource resource = resourceLoader.getResource(
            "classpath:prompts/order-assistant.st");
        
        PromptTemplate promptTemplate = new PromptTemplate(resource);
        promptTemplate.addAll(buildParams(order, userQuestion));
        
        return chatClient.call(promptTemplate.create())
            .getResult().getOutput().getContent();
    }

    private Map<String, Object> buildParams(Order order, String question) {
        Map<String, Object> params = new HashMap<>();
        params.put("orderId", order.getOrderId());
        params.put("productName", order.getProductName());
        params.put("amount", order.getAmount());
        params.put("status", order.getStatus());
        params.put("createTime", order.getCreateTime());
        params.put("question", question);
        params.put("deliveryDays", calculateDeliveryDays(order));
        return params;
    }
}
```

### 3.3 @Value 注入方式（更简洁）

```java
@Service
public class CustomerSupportService {

    @Value("classpath:prompts/customer-support.st")
    private Resource customerSupportPrompt;

    private final ChatClient chatClient;

    public String handle(String question) {
        PromptTemplate pt = new PromptTemplate(customerSupportPrompt);
        pt.add("question", question);
        return chatClient.call(pt.create())
            .getResult().getOutput().getContent();
    }
}
```

### 3.4 模板文件中使用条件逻辑

StringTemplate 支持简单的条件判断，让你的Prompt更灵活：

```st
// src/main/resources/prompts/customer-support.st
你是一个{customerType}客服助手。

{if isVip}
请注意：当前用户是VIP会员，请使用尊称"尊敬的会员"开头，
给予最高优先级的服务态度。
{else}
请使用标准服务流程。
{endif}

## 用户问题
{question}
```

```java
pt.add("customerType", "电商");
pt.add("isVip", true);  // 动态控制显示VIP提示还是标准流程
pt.add("question", question);
```

---

## 四、多语言 Prompt 模板

如果你的系统需要支持多语言，Prompt模板的多语言管理就很重要了。

### 4.1 按语言组织模板文件

```
src/main/resources/prompts/
├── zh/
│   ├── order-assistant.st
│   └── customer-support.st
├── en/
│   ├── order-assistant.st
│   └── customer-support.st
└── ja/
    ├── order-assistant.st
    └── customer-support.st
```

### 4.2 中文模板示例

```st
// prompts/zh/order-assistant.st
你是一个专业的电商客服助手。
请根据以下订单信息，礼貌地回答用户的问题。

## 订单信息
- 订单号：{orderId}
- 商品名称：{productName}
- 订单金额：{amount}元
- 订单状态：{status}

## 用户问题
{question}

请以友好、专业的语气回答。如果用户对物流有疑问，请引导至物流查询页面。
```

### 4.3 英文模板示例

```st
// prompts/en/order-assistant.st
You are a professional e-commerce customer service assistant.
Please answer the user's question politely based on the order information below.

## Order Information
- Order ID: {orderId}
- Product: {productName}
- Amount: ${amount}
- Status: {status}

## User Question
{question}

Respond in a friendly and professional tone. If the user has questions about shipping,
guide them to the logistics tracking page.
```

### 4.4 动态加载语言模板

```java
@Service
public class I18nPromptService {

    private final ResourceLoader resourceLoader;
    private final ChatClient chatClient;

    public String answer(Order order, String question, String language) {
        // 根据语言选择模板路径
        String promptPath = String.format(
            "classpath:prompts/%s/order-assistant.st", language);
        
        Resource resource = resourceLoader.getResource(promptPath);
        
        if (!resource.exists()) {
            // 降级到默认语言（中文）
            resource = resourceLoader.getResource(
                "classpath:prompts/zh/order-assistant.st");
        }

        PromptTemplate pt = new PromptTemplate(resource);
        pt.addAll(buildOrderParams(order));
        pt.add("question", question);

        return chatClient.call(pt.create())
            .getResult().getOutput().getContent();
    }
}
```

### 4.5 结合Spring的Locale解析

```java
@Service
public class LocaleAwarePromptService {

    private final ResourceLoader resourceLoader;
    private final ChatClient chatClient;

    public String answerWithLocale(Order order, String question) {
        // 从RequestContext中获取当前请求的Locale
        String language = LocaleContextHolder.getLocale().getLanguage();
        
        String path = String.format(
            "classpath:prompts/%s/order-assistant.st", 
            language != null ? language : "zh");
        
        Resource resource = resourceLoader.getResource(path);
        // ... 后续处理同上
    }
}
```

---

## 五、条件性 Prompt 模板（动态选择）

实际业务中，同一个AI功能在不同场景下可能需要完全不同的Prompt。比如退货流程中：

- **场景A**：用户刚提交退货申请 → 需要安抚+说明流程
- **场景B**：退货已审核通过 → 告知退款时间和金额
- **场景C**：退货被拒绝 → 解释原因+申诉引导

### 5.1 场景枚举

```java
public enum RefundScenario {
    SUBMITTED("refund-submitted.st"),
    APPROVED("refund-approved.st"),
    REJECTED("refund-rejected.st");

    private final String templateFile;

    RefundScenario(String templateFile) {
        this.templateFile = templateFile;
    }
    public String getTemplateFile() { return templateFile; }
}
```

### 5.2 条件路由实现

```java
@Service
public class RefundAssistantService {

    private final ResourceLoader resourceLoader;
    private final ChatClient chatClient;

    public String handleRefundQuestion(Refund refund, String userQuestion) {
        RefundScenario scenario = identifyScenario(refund);
        
        String path = "classpath:prompts/refund/" + scenario.getTemplateFile();
        Resource resource = resourceLoader.getResource(path);

        PromptTemplate pt = new PromptTemplate(resource);
        pt.addAll(buildRefundParams(refund));
        pt.add("question", userQuestion);

        return chatClient.call(pt.create())
            .getResult().getOutput().getContent();
    }

    private RefundScenario identifyScenario(Refund refund) {
        return switch (refund.getStatus()) {
            case "PENDING"   -> RefundScenario.SUBMITTED;
            case "APPROVED"  -> RefundScenario.APPROVED;
            case "REJECTED"  -> RefundScenario.REJECTED;
            default -> throw new IllegalArgumentException(
                "Unknown refund status: " + refund.getStatus());
        };
    }
}
```

### 5.3 三种场景的模板

```st
// prompts/refund/refund-submitted.st
你是电商售后客服助手。用户刚提交了一笔退货退款申请。
请在回复中做到：
1. 确认已收到申请（退款单号：{refundId}）
2. 告知预计审核时间：{reviewHours}小时内
3. 安抚用户情绪，表达理解
用户问题：{question}
```

```st
// prompts/refund/refund-approved.st
你是电商售后客服助手。用户的退货退款申请已审核通过。
请在回复中做到：
1. 告知审核通过
2. 明确退款金额：{amount}元，预计{refundDays}个工作日内到账
3. 提醒保管好退货凭证
用户问题：{question}
```

```st
// prompts/refund/refund-rejected.st
你是电商售后客服助手。用户的退货退款申请因{rejectReason}被拒绝。
请在回复中做到：
1. 清晰解释拒绝原因
2. 告知申诉渠道和流程
3. 表达遗憾并保持友善
用户问题：{question}
```

### 5.4 更优雅的工厂模式

```java
@Component
public class PromptTemplateFactory {

    private final ResourceLoader resourceLoader;

    /**
     * 根据业务类型和场景返回对应的PromptTemplate
     */
    public PromptTemplate create(String businessType, String scenario) {
        String path = String.format(
            "classpath:prompts/%s/%s.st", businessType, scenario);
        Resource resource = resourceLoader.getResource(path);

        if (!resource.exists()) {
            throw new IllegalStateException(
                "Prompt template not found: " + path);
        }
        return new PromptTemplate(resource);
    }
}
```

---

## 六、Prompt 模板的版本管理

当你的AI系统上线后，Prompt会持续迭代优化。没有版本管理，你永远不知道哪个版本的Prompt效果最好。

### 6.1 策略一：Git管理模板文件

最简单的方式——直接把 `.st` 文件纳入Git版本控制：

```
prompts/
├── order-assistant.st          # 当前生产版本
├── order-assistant_v2.st       # 实验中的新版本
└── order-assistant_v1.st       # 存档
```

```java
@Service
public class VersionedPromptService {

    @Value("${ai.prompt.version:default}")
    private String promptVersion;

    public PromptTemplate loadTemplate(String businessType) {
        String templateName = businessType;
        if (!"default".equals(promptVersion)) {
            templateName = businessType + "_" + promptVersion;
        }
        Resource resource = resourceLoader.getResource(
            "classpath:prompts/" + templateName + ".st");
        return new PromptTemplate(resource);
    }
}
```

```yaml
# application.yml
ai:
  prompt:
    version: v2  # 一键切换Prompt版本，A/B测试轻松搞定
```

### 6.2 策略二：数据库存储（适合运营人员管理）

如果Prompt需要运营人员频繁调整，存到数据库更合适：

```sql
CREATE TABLE prompt_template (
    id          BIGINT PRIMARY KEY AUTO_INCREMENT,
    biz_type    VARCHAR(50)  NOT NULL COMMENT '业务类型',
    version     INT          NOT NULL COMMENT '版本号',
    content     TEXT         NOT NULL COMMENT '模板内容',
    is_active   TINYINT      DEFAULT 0 COMMENT '是否激活',
    created_by  VARCHAR(50)  COMMENT '创建人',
    created_at  DATETIME     DEFAULT CURRENT_TIMESTAMP,
    updated_at  DATETIME     DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY uk_biz_version (biz_type, version)
);
```

```java
@Service
public class DatabasePromptTemplateService {

    private final JdbcTemplate jdbcTemplate;

    public PromptTemplate loadFromDb(String bizType) {
        String sql = """
            SELECT content FROM prompt_template
            WHERE biz_type = ? AND is_active = 1
            ORDER BY version DESC LIMIT 1
            """;
        String templateContent = jdbcTemplate.queryForObject(
            sql, String.class, bizType);
        return new PromptTemplate(templateContent);
    }
}
```

### 6.3 策略三：混合模式（推荐）

```java
@Service
public class SmartPromptLoader {

    private final ResourceLoader resourceLoader;
    private final JdbcTemplate jdbcTemplate;

    public PromptTemplate load(String bizType) {
        // 优先从数据库加载（支持运营在线修改）
        PromptTemplate dbTemplate = loadFromDb(bizType);
        if (dbTemplate != null) {
            return dbTemplate;
        }
        // 降级到资源文件
        return loadFromResource(bizType);
    }

    private PromptTemplate loadFromDb(String bizType) {
        try {
            String sql = """
                SELECT content FROM prompt_template
                WHERE biz_type = ? AND is_active = 1 LIMIT 1""";
            String content = jdbcTemplate.queryForObject(
                sql, String.class, bizType);
            return content != null ? new PromptTemplate(content) : null;
        } catch (Exception e) {
            log.warn("从数据库加载Prompt失败，降级到资源文件", e);
            return null;
        }
    }

    private PromptTemplate loadFromResource(String bizType) {
        Resource resource = resourceLoader.getResource(
            "classpath:prompts/" + bizType + ".st");
        return new PromptTemplate(resource);
    }
}
```

---

## 七、进阶技巧

### 7.1 Prompt模板中的循环（遍历列表）

StringTemplate支持循环，适合处理动态列表：

```st
// prompts/product-recommend.st
你是一个商品推荐专家。用户最近浏览了以下商品：

{userHistory}

请根据浏览记录推荐5款相似商品，输出格式为：
1. 商品名：xxx，推荐理由：xxx
2. ...
```

```java
// 在Java层构建列表字符串
String userHistory = productHistoryList.stream()
    .map(p -> String.format("- %s（%s，%s元）", 
        p.getName(), p.getCategory(), p.getPrice()))
    .collect(Collectors.joining("\n"));
pt.add("userHistory", userHistory);
```

### 7.2 结构化输出约束嵌入Prompt

在模板中嵌入输出格式约束，配合OutputParser使用：

```st
// prompts/order-extractor.st
请从以下文本中提取订单信息。

文本内容：
{rawText}

请严格按照以下JSON格式输出，不要输出任何其他内容：
{
  "orderId": "订单号",
  "productName": "商品名称",
  "amount": 金额数字,
  "status": "订单状态"
}
```

### 7.3 组合多个模板

```java
public Prompt assemblePrompt(Order order, String question) {
    // 通用系统提示
    PromptTemplate systemPt = new PromptTemplate(systemPrompt);
    systemPt.add("role", "电商客服");

    // 业务上下文
    PromptTemplate contextPt = new PromptTemplate(bizContextPrompt);
    contextPt.add("orderId", order.getOrderId());

    // 用户问题
    PromptTemplate userPt = new PromptTemplate(userPrompt);
    userPt.add("question", question);

    return new Prompt(List.of(
        systemPt.createMessage(),
        contextPt.createMessage(),
        userPt.createMessage()
    ));
}
```

---

## 八、最佳实践总结

| 实践 | 说明 |
|------|------|
| **模板放资源文件** | 不要硬编码在Java代码中 |
| **分类管理** | 按业务模块分目录：`prompts/order/`、`prompts/refund/` |
| **版本控制** | `.st`文件纳入Git，改动可追溯 |
| **多语言支持** | 按语言分文件，结合Spring的Locale机制 |
| **参数校验** | 绑定参数前校验必填字段，避免LLM收到 `null` |
| **日志记录** | 记录最终发送的完整Prompt，方便排查问题 |
| **A/B测试** | 通过配置文件切换Prompt版本 |
| **分级降级** | 数据库→资源文件→硬编码默认值 |

---

## 九、完整示例：一个聊天服务全流程

```java
@RestController
@RequestMapping("/api/chat")
public class ChatController {

    private final ChatClient chatClient;
    private final SmartPromptLoader promptLoader;

    @PostMapping("/order")
    public ResponseEntity<String> chatAboutOrder(
            @RequestBody ChatRequest request) {
        
        // 1. 加载模板（优先数据库，降级到资源文件）
        PromptTemplate pt = promptLoader.load("order-assistant");

        // 2. 从数据库查询订单信息
        Order order = orderService.findById(request.getOrderId());

        // 3. 绑定参数
        Map<String, Object> params = new HashMap<>();
        params.put("orderId", order.getOrderId());
        params.put("productName", order.getProductName());
        params.put("amount", order.getAmount().toString());
        params.put("status", order.getStatus().getDisplayName());
        params.put("createTime", order.getCreateTime().toString());
        params.put("question", request.getQuestion());
        pt.addAll(params);

        // 4. 记录日志（方便排查）
        log.info("Sending prompt: {}", pt.render());

        // 5. 调用AI
        ChatResponse response = chatClient.call(pt.create());

        // 6. 返回结果
        return ResponseEntity.ok(
            response.getResult().getOutput().getContent());
    }
}
```

---

## 十、小结

Prompt Template 不是银弹，但它能帮你：

1. **统一管理**：所有Prompt集中在一个地方，修改无需全局搜索
2. **多语言支持**：一套代码，多套模板，自动按Locale切换
3. **版本管理**：Git/数据库双重保障，A/B测试无需改代码
4. **条件路由**：根据业务场景动态选择最合适的Prompt
5. **运营友好**：数据库存储后，产品经理也能在线调整Prompt

当你把Prompt管好了，下一步就是把AI的"动手能力"交给它——让AI不再只是动嘴皮子，而是真正能帮用户干活。这就是我们下一篇要讲的 **Function Calling**。

---

> **下一篇预告**：Spring AI Function Calling——让LLM直接调用你的Service层，AI不仅能聊天还能帮你创建订单、发送邮件、查询数据库。敬请期待！

---

*如果觉得有帮助，欢迎点赞收藏关注，你的支持是我持续输出的动力！*

---
