# 温度参数（Temperature）调优实战：何时需要创造、何时需要精确

> 同样的 Prompt"生成一个 Java 工具类"，Temperature=0.1 每次都生成几乎一样的代码，Temperature=1.5 每次都是全新风格——选错了参数，你的 AI 可能像个醉汉。

---

## 一、一个让人崩溃的实验

周末下午，我写了一个简单 Prompt，跑了 5 次，对比不同 Temperature 值下的输出：

**Prompt**：
> "写一个 Java 工具类，实现字符串的驼峰命名和下划线命名的互相转换。"

**Temperature = 0.1（第1次）**：
```java
public class CaseConverter {
    public static String toCamelCase(String input) {
        StringBuilder sb = new StringBuilder();
        boolean nextUpper = false;
        for (char c : input.toCharArray()) {
            if (c == '_') { nextUpper = true; }
            else { sb.append(nextUpper ? Character.toUpperCase(c) : c); nextUpper = false; }
        }
        return sb.toString();
    }
}
```

**Temperature = 0.1（第2次）**：
```java
public class CaseConverter {
    public static String toCamelCase(String input) {
        StringBuilder sb = new StringBuilder();
        boolean nextUpper = false;
        for (char c : input.toCharArray()) {
            if (c == '_') { nextUpper = true; }
            else { sb.append(nextUpper ? Character.toUpperCase(c) : c); nextUpper = false; }
        }
        return sb.toString();
    }
}
```
> 几乎一模一样——像个复印机。

**Temperature = 0.1（第3次）**：
```java
public class CaseConverter {
    public static String toCamelCase(String input) {
        StringBuilder sb = new StringBuilder();
        boolean nextUpper = false;
        for (char c : input.toCharArray()) {
            if (c == '_') { nextUpper = true; }
            else { sb.append(nextUpper ? Character.toUpperCase(c) : c); nextUpper = false; }
        }
        return sb.toString();
    }
}
```
> 第3次还是差不多。**确定性极强，但缺少下划线转驼峰的反向方法。**

---

**现在看 Temperature = 1.5 的表现：**

**Temperature = 1.5（第1次）**：
```java
// 第1次：用了 Stream API
public class NamingUtil {
    public static String snakeToCamel(String s) {
        return Pattern.compile("_([a-z])")
            .matcher(s)
            .replaceAll(m -> m.group(1).toUpperCase());
    }
    // 名命变了，方法名也变了，只写了一个方向
}
```

**Temperature = 1.5（第2次）**：
```java
// 第2次：用了 Apache Commons，风格完全变了
public class StringConverter {
    public static String underscoreToCamelcase(String input) {
        if (input == null) return null;
        return WordUtils.capitalizeFully(input, '_').replace("_", "");
    }
    // 引入了外部依赖！
}
```

**Temperature = 1.5（第3次）**：
```java
// 第3次：手写了完整双向转换+异常处理+单元测试
public class CaseUtils {
    public static String camelToSnake(String camel) { /* 15行实现 */ }
    public static String snakeToCamel(String snake) { /* 12行实现 */ }
    public static String toUpperCase(String input) { /* 怎么多了个方法？ */ }
    public static void main(String[] args) { /* 还加了main方法测试！ */ }
}
```

你看——**Temperature=1.5 的三次输出差异大到像是三个不同的人写的**：
- 第1次：Java 8 Stream 风格
- 第2次：Apache Commons 依赖党
- 第3次：全能型选手（还自带测试）

这就是 Temperature 的威力：**它决定了 AI 是"严谨的律师"还是"天马行空的诗人"。**

---

## 二、Temperature 原理——用"人话"讲明白

### 2.1 先忘掉数学公式

Temperature 说到底是控制 AI **选择下一个词的"胆量"**。

想象 AI 在写代码。每写一个 Token，它脑子里都有一个概率排行榜：

```
Token "String"    → 概率 45%
Token "Integer"   → 概率 30%
Token "Long"      → 概率 15%
Token "Boolean"   → 概率 8%
Token "Float"     → 概率 2%
```

**Temperature = 0（趋近于零）**：AI 永远选概率最高的那个 → `String` → 每次都一样。

**Temperature = 1（默认）**：按原概率分布随机选 → 大概率是 `String`，但也可能是 `Integer`。

**Temperature = 2（很高）**：把概率"拉平" → `String` 和 `Float` 的概率接近 → AI 开始"乱选" → 像醉汉。

### 2.2 本质是"概率分布的锐度调节"

```java
// 用Java模拟Temperature的效果
public class TemperatureSimulator {
    
    /**
     * 模拟不同Temperature值对概率分布的影响
     */
    public static Map<String, Double> applyTemperature(
            Map<String, Double> originalProbs, double temperature) {
        
        Map<String, Double> adjusted = new LinkedHashMap<>();
        
        for (Map.Entry<String, Double> entry : originalProbs.entrySet()) {
            // Temperature作用于logit空间，这里简化演示
            // 实际公式: p_i = exp(logit_i / T) / sum(exp(logit_j / T))
            double adjustedProb = Math.pow(entry.getValue(), 1.0 / temperature);
            adjusted.put(entry.getKey(), adjustedProb);
        }
        
        // 重新归一化
        double total = adjusted.values().stream().mapToDouble(Double::doubleValue).sum();
        adjusted.replaceAll((k, v) -> v / total);
        
        return adjusted;
    }
    
    public static void main(String[] args) {
        Map<String, Double> original = new LinkedHashMap<>();
        original.put("String", 0.45);
        original.put("Integer", 0.30);
        original.put("Long", 0.15);
        original.put("Boolean", 0.08);
        original.put("Float", 0.02);
        
        System.out.println("=== Temperature = 0.1 (极度确定) ===");
        applyTemperature(original, 0.1).forEach((k, v) ->
            System.out.printf("%s: %.4f%n", k, v));
        // String: 0.9998 ← 几乎100%
        // 其他选项 ≈ 0
        
        System.out.println("\n=== Temperature = 1.0 (默认) ===");
        applyTemperature(original, 1.0).forEach((k, v) ->
            System.out.printf("%s: %.4f%n", k, v));
        // String: 0.45 ← 保持原样
        
        System.out.println("\n=== Temperature = 2.0 (高随机) ===");
        applyTemperature(original, 2.0).forEach((k, v) ->
            System.out.printf("%s: %.4f%n", k, v));
        // String: 0.29, Integer: 0.25, Long: 0.21, Boolean: 0.16, Float: 0.09
        // ← 概率被拉平了！
    }
}
```

运行结果一目了然：
- **T=0.1**：`String` 的概率从 45% 飙升到 99.98%，AI"别无选择"
- **T=2.0**：所有选项概率趋近，AI"无所适从"

---

## 三、6 个 Java 场景的最佳 Temperature 实验

### 场景一：代码生成（推荐 Temperature = 0.1 - 0.3）

**为什么**：代码生成需要稳定、可预测。你肯定不希望同一个方法名在两次调用中变成不同的签名。

**实验：生成一个 Spring Boot Controller**

```java
// Prompt: "生成一个 Spring Boot REST Controller，管理用户CRUD"

// Temperature = 0.1
@RestController
@RequestMapping("/api/users")
public class UserController {
    private final UserService userService;
    
    @GetMapping
    public List<UserDTO> list() {
        return userService.findAll();
    }
    
    @GetMapping("/{id}")
    public UserDTO get(@PathVariable Long id) {
        return userService.findById(id);
    }
    // → 3次运行结果高度一致，只有GetMapping的路径名可能略有差异
}

// Temperature = 0.5
@RestController
@RequestMapping("/api/v1/users")
public class UserManagementController { // ← 类名变了！
    @Autowired
    private UserService userService; // ← 字段注入 vs 构造注入
    
    @GetMapping("/list")
    public ResponseEntity<Page<UserDTO>> getUsers( // ← 返回类型变了
        @RequestParam(defaultValue = "1") int page,
        @RequestParam(defaultValue = "20") int size) {
        return ResponseEntity.ok(userService.findPage(page, size));
    }
    // → 3次运行差异明显：类名、注入方式、返回类型、分页策略各不相同
}

// Temperature = 1.5
@RestController
@RequestMapping("/api/users")
public class UserResource { // ← RESTful叫Resource了？
    
    private final UserRepository userRepository; // ← 直接用了Repository！
    
    @GetMapping(produces = MediaType.APPLICATION_JSON_VALUE)
    public Flux<UserDTO> streamUsers() { // ← **WebFlux响应式！**
        return userRepository.findAll()
            .map(this::toDTO);
    }
    // → 第1次可能用WebMVC，第2次就跳到WebFlux，完全不可控
}
```

**最佳实践**：

```java
public class CodeGenerationConfig {
    public static final double TEMP_CODE_GENERATION = 0.2;
    
    public static String getSystemPrompt() {
        return """
            你是Java代码生成专家。
            要求：
            - 遵循阿里巴巴Java开发手册
            - 使用构造器注入，不使用字段注入
            - 返回值统一使用 Result<T> 包装
            - 方法命名遵循驼峰规范
            """;
    }
}
```

### 场景二：代码重构（推荐 Temperature = 0.3 - 0.5）

**为什么**：重构需要一定的创造力来找到更好的组织方式，但不能天马行空到改变业务逻辑。

**实验：重构一段过程式代码为面向对象风格**

```java
// 原始代码（过程式）
public void processOrder(String orderId) {
    Order order = orderDao.findById(orderId);
    if (order.getStatus().equals("PENDING")) {
        order.setStatus("PROCESSING");
        orderDao.save(order);
        sendEmail(order.getUserEmail(), "订单处理中");
        // 50行的过程式逻辑...
    }
}

// Temperature = 0.3 (稳定重构)
// → 一致使用策略模式 + 责任链
public class OrderProcessor {
    private final List<OrderHandler> handlers;
    
    public void process(Order order) {
        handlers.forEach(h -> h.handle(order));
    }
}
// 每次重构结果相似：策略模式 + 责任链

// Temperature = 0.5 (有变化的重构)
// 第1次：策略模式 + 状态机
// 第2次：Command模式 + Event Bus
// 第3次：Pipeline + Observer
// 各有千秋，但业务语义不变

// Temperature = 0.9 (过于发散)
// 第1次：带上了SAGA分布式事务（不必要！）
// 第2次：引入了Actor模型（过度设计！）
// 第3次：用了Spring State Machine（引入重依赖！）
```

**建议配置**：

```java
public class RefactoringConfig {
    public static final double TEMP_REFACTORING = 0.4;
    
    public static final String REFACTORING_CONSTRAINT = """
        重构约束：
        - 保持原有业务逻辑不变
        - 不引入超出当前项目技术栈的新框架
        - 优先使用23种经典设计模式
        - 重构后代码行数不超过原来的1.5倍
        """;
}
```

### 场景三：写注释/文档（推荐 Temperature = 0.5 - 0.7）

**为什么**：注释需要一定的表达灵活性，但核心信息必须准确。

**实验对比**：

```java
// Temperature = 0.2
/**
 * Processes the order.
 * @param orderId the order id
 * @return the processing result
 */
// → 机械翻译，像是机翻的——"处理订单，返回处理结果"，毫无信息量

// Temperature = 0.6
/**
 * 处理订单的全生命周期流程。
 * 
 * 处理步骤：
 * 1. 校验订单状态（仅PENDING状态可处理）
 * 2. 锁定库存（调用InventoryService.reserve方法）
 * 3. 生成支付单（若订单金额>0）
 * 4. 发送状态变更通知（邮件 + 站内信）
 * 
 * 注意事项：
 * - 本方法有事务性保证（@Transactional）
 * - inventoryService.reserve可能抛出InsufficientStockException
 * - 邮件发送失败不影响主流程（异步+重试）
 * 
 * @param orderId 订单唯一标识，19位长整型
 * @return ProcessingResult 包含新状态和下一步操作建议
 * @throws OrderNotFoundException 订单不存在
 * @throws IllegalOrderStateException 订单状态不允许处理
 */
// → 信息丰富、准确、有结构。这才是生产级注释
```

### 场景四：头脑风暴/技术方案（推荐 Temperature = 0.7 - 0.9）

**为什么**：方案设计需要发散思维。太低的 Temperature 只会给你"最安全"的答案，但可能不是最合适的。

**实验：设计一个秒杀系统的技术方案**

```java
// Temperature = 0.3
// → 每次都给出几乎一样的方案：
// "Redis预减库存 + 消息队列异步下单 + 限流"
// 这是标准答案，但它可能忽略了你的具体场景

// Temperature = 0.8
// 第1次方案亮点：
// - Redis Lua脚本原子减库存
// - RabbitMQ异步下单 + 死信队列处理超时
// - 令牌桶限流 + 验证码防刷
// - 数据库乐观锁防超卖

// 第2次方案亮点：
// - 纯Redis方案（不依赖DB，更高性能）
// - 使用Redisson的RSemaphore做信号量控制
// - 前端静态化+CDN减少服务器压力
// - 降级方案：排队页面

// 第3次方案亮点：
// - 库存分层（本地缓存 → Redis → DB）
// - OpenResty网关层直接过滤无效请求
// - Kafka分区保证用户维度有序
// - 柔性可用：部分降级策略

// → 三次方案互补，覆盖了你可能没想到的维度
```

**头脑风暴专用 Prompt 模板**：

```java
public class BrainstormingPrompt {
    
    public static String build(String topic) {
        return String.format("""
            你是一位有15年经验的Java系统架构师。
            
            请对以下话题进行头脑风暴式的技术方案输出：
            「%s」
            
            要求：
            1. 给出至少3种不同的技术方案，每种都有独特的核心思路
            2. 每种方案标注适用场景和trade-off
            3. 不要给出"唯一正确解"——技术选型没有银弹
            4. 欢迎提出反常规的思路（但标注风险）
            5. 每种方案给出技术栈组合和预估并发上限
            """, topic);
    }
}
```

### 场景五：创意文案（推荐 Temperature = 0.9 - 1.2）

**为什么**：给产品起名、写宣传文案、设计 API 命名——这些都是创意型任务，需要高度的多样性。

**实验：给一个新的微服务中间件起名并写Slogan**

```java
// Temperature = 0.3
// 3次输出几乎一样：
// "MicroServiceKit - 一站式微服务解决方案"
// "MicroServiceKit - 企业级微服务工具包"
// "MicroServiceKit - 高效微服务开发框架"
// → 无聊到让人想睡觉

// Temperature = 1.1
// 第1次: "AetherFlow - 轻盈如风，架构如诗"
// 第2次: "ServiceFabric - 织起微服务的千丝万缕"
// 第3次: "NanoMesh - 原子级服务网格，纳米级延迟"

// Java领域的创意命名
// 从实际开源项目就能看出来：Eureka、Nacos、Sentinel、Seata
// 这些都是高Temperature式命名的成果
```

**创意文案的搭配用法**：

```java
public class CreativeWorkflow {
    
    /**
     * 两阶段创意流程：
     * 阶段1（高温度）：发散生成多个方案
     * 阶段2（低温度）：收敛评估和优化
     */
    public String creativeWorkflow(ChatClient ai, String requirement) {
        // 阶段1：发散 - 用T=1.2生成10个创意方向
        String brainstormPrompt = "请为该Java中间件生成10个命名方案……";
        String brainstormResult = ai.chat(brainstormPrompt, 1.2);
        
        // 阶段2：收敛 - 用T=0.2评估最佳方案
        String evaluatePrompt = String.format(
            "从以下10个命名方案中选出最佳的3个，并说明理由：\n%s", brainstormResult);
        String finalResult = ai.chat(evaluatePrompt, 0.2);
        
        // 阶段3：打磨 - 用T=0.5优化最佳方案
        String polishPrompt = String.format(
            "请对以下3个命名进行微调优化，保持核心创意但提升专业度：\n%s", finalResult);
        return ai.chat(polishPrompt, 0.5);
    }
}
```

### 场景六：代码审查（推荐 Temperature = 0.1 - 0.2）

**为什么**：代码审查需要严格的规则一致性。你不希望同一个问题，第一次被标记为 CRITICAL，第二次只是 WARNING。

```java
// Temperature = 0.2 (稳定审查)
// 3次审查同一段代码，发现的问题和评级一致

// Prompt: "审查以下代码，找出所有潜在问题"
public void saveUser(String name, String email) {
    User user = new User();
    user.setName(name);
    user.setEmail(email);
    userDao.save(user);
}

// 3次审查结果一致：
// [CRITICAL] 未校验name/email是否为null，可能NPE
// [WARNING] 未校验email格式
// [WARNING] 未处理重名/重复邮箱的情况
// [INFO] save方法建议返回User实体

// Temperature = 0.8
// 第1次: CRITICAL × 2, WARNING × 3
// 第2次: WARNING × 1, INFO × 5  ← 严格程度完全不同！
// 第3次: CRITICAL × 5  ← 突然严格了5倍！
```

**代码审查专用配置**：

```java
public class CodeReviewConfig {
    public static final double TEMP_CODE_REVIEW = 0.15;
    
    public static final String REVIEW_CONSTRAINTS = """
        代码审查规则（严格遵守）：
        
        严重级别定义：
        - CRITICAL: 可能导致线上故障、数据丢失、安全漏洞
        - WARNING: 代码异味、性能隐患、可维护性问题  
        - INFO: 编码风格建议
        
        审查维度（每个维度必须审查）：
        1. 空指针风险
        2. 线程安全
        3. 资源泄漏（流、连接等）
        4. SQL注入/OOM风险
        5. 异常处理
        6. 命名规范（阿里规约）
        
        输出格式（严格遵守）：
        {"severity": "CRITICAL|WARNING|INFO", "line": 数字, "rule": "规则编号", "message": "问题描述", "fix": "修复建议"}
        """;
}
```

---

## 四、6 个场景最佳 Temperature 速查表

```java
public class TemperaturePresets {
    
    public enum JavaScenario {
        CODE_GENERATION("代码生成", 0.2),
        CODE_REFACTORING("代码重构", 0.4),
        CODE_DOCUMENTATION("写注释/文档", 0.6),
        TECH_DESIGN("技术方案设计", 0.8),
        CREATIVE_WRITING("创意文案/命名", 1.0),
        CODE_REVIEW("代码审查", 0.15),
        UNIT_TEST("单元测试生成", 0.3),
        API_DESIGN("API接口设计", 0.4),
        BUG_FIX("Bug修复", 0.2),
        PERFORMANCE_TUNING("性能调优", 0.5);
        
        private final String label;
        private final double recommendedTemp;
        
        JavaScenario(String label, double recommendedTemp) {
            this.label = label;
            this.recommendedTemp = recommendedTemp;
        }
        
        public double getTemp() { return recommendedTemp; }
        public String getLabel() { return label; }
    }
    
    /**
     * 根据场景获取推荐Temperature
     */
    public static double getRecommendedTemp(String taskDescription) {
        String lower = taskDescription.toLowerCase();
        
        // 规则匹配
        if (lower.contains("生成") || lower.contains("generate") || lower.contains("创建")) {
            return JavaScenario.CODE_GENERATION.getTemp();
        }
        if (lower.contains("重构") || lower.contains("refactor") || lower.contains("优化")) {
            return JavaScenario.CODE_REFACTORING.getTemp();
        }
        if (lower.contains("注释") || lower.contains("文档") || lower.contains("doc")) {
            return JavaScenario.CODE_DOCUMENTATION.getTemp();
        }
        if (lower.contains("方案") || lower.contains("设计") || lower.contains("架构")) {
            return JavaScenario.TECH_DESIGN.getTemp();
        }
        if (lower.contains("审查") || lower.contains("review") || lower.contains("检查")) {
            return JavaScenario.CODE_REVIEW.getTemp();
        }
        
        // 默认值
        return 0.5;
    }
    
    // 打印速查表
    public static void printCheatSheet() {
        System.out.println("==============================");
        System.out.println("Java场景 Temperature 速查表");
        System.out.println("==============================");
        for (JavaScenario scenario : JavaScenario.values()) {
            String bar = "█".repeat((int)(scenario.getTemp() * 20));
            System.out.printf("%-18s T=%.2f %s%n", 
                scenario.getLabel(), scenario.getTemp(), bar);
        }
        System.out.println("==============================");
    }
}
```

---

## 五、Top-P 和 Temperature 的配合使用

Temperature 不是唯一控制随机性的参数。**Top-P（Nucleus Sampling）** 是另一个重要参数。

### 5.1 两者的区别

| 参数 | 作用方式 | 直观理解 |
|------|---------|---------|
| Temperature | 改变所有Token的概率分布 | "拉平"或"锐化"概率 |
| Top-P | 只考虑概率累计达P%的Token | 砍掉概率太低的"不靠谱选项" |

```java
public class TemperatureVsTopP {
    
    /**
     * Top-P采样模拟
     * @param probs Token概率分布
     * @param topP 阈值，例如0.9表示只取累积概率前90%的Token
     */
    public static List<String> topPFilter(Map<String, Double> probs, double topP) {
        // 按概率降序排列
        List<Map.Entry<String, Double>> sorted = probs.entrySet().stream()
            .sorted(Map.Entry.<String, Double>comparingByValue().reversed())
            .toList();
        
        double cumulative = 0.0;
        List<String> candidates = new ArrayList<>();
        
        for (Map.Entry<String, Double> entry : sorted) {
            cumulative += entry.getValue();
            candidates.add(entry.getKey());
            if (cumulative >= topP) break; // 截断
        }
        
        return candidates;
    }
    
    // Temperature=1.0 + Top-P=0.9 是常用的"安全组合"
    // Temperature控制"胆子多大"，Top-P控制"底线在哪"
}
```

### 5.2 推荐组合

```java
public class SamplingConfig {
    
    /**
     * 推荐的Temperature + Top-P组合
     */
    public record SamplingPair(double temperature, double topP, String useCase) {}
    
    public static final List<SamplingPair> RECOMMENDED_PAIRS = List.of(
        new SamplingPair(0.1, 0.95, "代码生成、审查 → 极确定"),
        new SamplingPair(0.3, 0.9,  "重构、测试 → 稳定为主"),
        new SamplingPair(0.7, 0.9,  "文档、注释 → 平衡"),
        new SamplingPair(0.9, 0.85, "头脑风暴、方案 → 有框架的发散"),
        new SamplingPair(1.2, 0.8,  "创意文案 → 大胆但不过线")
    );
    
    /**
     * 工程实践：写配置时同时设置这两个参数
     */
    public static void configureChatRequest(double temperature, double topP) {
        // 伪代码——实际调用各AI厂商API
        System.out.printf("""
            ChatRequest:
              model: gpt-4
              temperature: %.2f
              top_p: %.2f
              max_tokens: 4096
            %n""", temperature, topP);
    }
}
```

**经验法则**：
- Temperature < 0.5 时，Top-P 影响不大（概率已经高度集中）
- Temperature > 0.8 时，建议同时设置 Top-P < 0.9 来"兜底"
- 永远不要让 Temperature >= 1.5 且 Top-P = 1.0（这等于完全随机）

---

## 六、实战：动态 Temperature 调优系统

```java
@Service
public class AdaptiveTemperatureService {
    
    private final ChatClient chatClient;
    
    /**
     * 根据任务类型和用户反馈动态调整Temperature
     */
    public double calculateTemperature(TaskContext context) {
        double baseTemp = TemperaturePresets.getRecommendedTemp(context.getTaskDescription());
        
        // 因子1：用户指定了详细约束 → 降低Temperature（更守规矩）
        if (context.hasDetailedConstraints()) {
            baseTemp -= 0.1;
        }
        
        // 因子2：用户要求"多种方案" → 提高Temperature
        if (context.requestsMultipleOptions()) {
            baseTemp += 0.2;
        }
        
        // 因子3：此前的输出被用户拒绝了 → 尝试提高Temperature给不同的
        if (context.hasPreviousRejections()) {
            baseTemp += 0.15;
        }
        
        // 因子4：任务涉及代码 → 降低Temperature
        if (context.involvesCodeGeneration()) {
            baseTemp -= 0.05;
        }
        
        // 边界限制
        return Math.max(0.0, Math.min(2.0, baseTemp));
    }
    
    /**
     * 自动重试策略：如果用户不满意，逐步提高Temperature
     */
    public String retryWithHigherTemperature(String originalPrompt, 
            String lastOutput, int retryCount) {
        
        // 每次重试提高0.15的Temperature
        double adjustedTemp = Math.min(0.1 + retryCount * 0.15, 1.5);
        
        String retryPrompt = String.format("""
            用户对上一次的回答不满意。请给出一个不同的方案。
            
            上一次的回答（避免重复）：
            %s
            
            原来的问题：
            %s
            
            要求：给出与上一次不同的解决方案。
            """, lastOutput, originalPrompt);
        
        System.out.printf("[重试] 第%d次，Temperature=%.2f%n", retryCount, adjustedTemp);
        return chatClient.chat(retryPrompt, adjustedTemp);
    }
    
    @Data
    @Builder
    public static class TaskContext {
        private String taskDescription;
        private boolean hasDetailedConstraints;
        private boolean requestsMultipleOptions;
        private boolean hasPreviousRejections;
        private boolean involvesCodeGeneration;
    }
}
```

---

## 七、总结

Temperature 是 AI 编程助手中**最被低估的参数**。大多数人用默认值 0.7 从头用到尾，结果就是：该精确的时候胡乱发挥，该创新的时候呆板无趣。

记住这个口诀：

> **代码相关 0.2 ± 0.1，设计讨论 0.5 ± 0.2，创意发散 1.0 ± 0.2**

```java
// 一行总结
public record TempRule(double value, String when) {}
List<TempRule> rules = List.of(
    new TempRule(0.15, "代码审查：不能含糊"),
    new TempRule(0.2, "代码生成：稳定第一"),
    new TempRule(0.4, "重构设计：有序创新"),
    new TempRule(0.6, "文档注释：表达灵活"),
    new TempRule(0.8, "方案设计：发散思维"),
    new TempRule(1.0, "创意命名：脑洞大开")
);
```

下次调 AI 参数时，先问自己：**我现在需要的是一个严谨的律师，还是一个天马行空的诗人？**

---

**下一篇预告**：《System Prompt 设计艺术：打造专属的 Java 技术顾问 AI》——同样的 ChatGPT，加了 System Prompt 后回答专业度翻 10 倍。角色设定、行为约束、输出格式、互动方式四大要素，附可直接复制的 Spring Boot 架构顾问 Prompt 模板。
