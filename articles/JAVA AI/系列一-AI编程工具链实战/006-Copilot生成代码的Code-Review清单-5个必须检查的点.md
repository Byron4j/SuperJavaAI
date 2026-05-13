# Copilot 生成代码的 Code Review 清单：5 个必须检查的点，漏一个都可能出生产事故

> 别让你的 AI 队友变成"猪队友"——这篇清单，建议打印贴在显示器旁边。

---

## 一、一个价值 $4000 的血泪教训

去年年底，一个朋友凌晨三点给我打电话，声音都在抖。

他和几个合伙人做的 SaaS 产品刚上线一个多月，用了 Copilot 辅助开发，效率确实起飞。那天他写了一个调用 OpenAI 的聚合接口，Copilot 帮他补全了配置类，一键 Tab 接受——代码看起来很漂亮，编译通过，测试也没问题，直接合到了主分支。

三周后，他收到 AWS 账单：**$4,000 的额外费用**。

排查了半天才发现——Copilot 生成的代码里，把从某篇技术博客"学"来的 API Key，当作示例值硬编码进了 `application.properties`。而在那段代码里，这个 Key 并没有被 `.gitignore` 排除，直接被推到了 GitHub 公开仓库。

更讽刺的是，GitHub 的 Secret Scanning 确实给他发了警告邮件，但那封邮件躺在垃圾箱里，和他每天收到的几十封 GitHub 通知混在了一起。

等他反应过来时，这个 Key 已经被三个国家的 IP 轮番调用，跑了大量 `gpt-4-turbo` 的推理请求。对方甚至还贴心地用他的 Key 训了一个小模型。

他说了一句话让我记到现在：

> **"Copilot 写得快，但快不等于对。它不知道你的数据库密码意味着什么，它不知道你的客户信息值多少钱。"**

这不是个例。随着 AI 编程工具的普及，越来越多的团队正在经历这种"高效开发→低级漏洞→生产事故"的噩梦链路。Copilot 很聪明，但它没有安全意识、没有业务上下文、没有性能嗅觉——它只是一个概率模型，在冥冥之中"猜"你最可能想要的代码。

**如果你对 AI 生成的代码不做 Code Review，本质上就是在让一个外行人替你写生产代码。**

下面这 5 个检查点，是我和几个 Tech Lead 从几十次事故复盘里提炼出来的。每一条都有真实案例背书，每一条都发生在你我身边的项目里。

---

## 二、检查点 1：硬编码的密钥和密码

### 为什么 Copilot 容易犯这个错？

Copilot 的训练数据包含了海量开源项目、技术博客和教程代码。在这些语料中，"硬编码密钥"的比例高得惊人——因为教程要演示功能，就必须写一个"能跑起来"的示例。Copilot 学到的是："这个字段大概率填一个看起来像 Key 的字符串"。

### 🔴 反例：Copilot 最常见的补全结果

```java
// Copilot 自动生成的配置类
@Component
@ConfigurationProperties(prefix = "openai")
public class OpenAIConfig {

    private String apiKey = "sk-proj-abc123def456ghi789jkl"; // 直接硬编码！

    private String baseUrl = "https://api.openai.com/v1";

    // getter / setter
}
```

```java
// Copilot 自动补全的数据库连接
@Service
public class DataSyncService {

    private static final String DB_URL = "jdbc:mysql://prod-db.internal:3306/user_center";
    private static final String DB_USER = "admin";
    private static final String DB_PASSWORD = "P@ssw0rd!2024"; // 生产密码明文写在代码里

    public void syncData() {
        try (Connection conn = DriverManager.getConnection(DB_URL, DB_USER, DB_PASSWORD)) {
            // ...
        }
    }
}
```

```java
// Copilot 自动生成的 JWT 签名密钥
public class JwtUtil {

    private static final String SECRET_KEY = "mySecretKey12345"; // 硬编码，且极弱

    public String generateToken(String userId) {
        return Jwts.builder()
                .setSubject(userId)
                .signWith(SignatureAlgorithm.HS256, SECRET_KEY.getBytes())
                .compact();
    }
}
```

### 🟢 正例：正确的密钥管理方式

```java
// ✅ 从环境变量或配置中心读取，绝不在代码中写死默认值
@Component
@ConfigurationProperties(prefix = "openai")
public class OpenAIConfig {

    private String apiKey; // 不设默认值，启动时若缺失则明确报错

    @PostConstruct
    public void validate() {
        if (apiKey == null || apiKey.isBlank()) {
            throw new IllegalStateException("openai.api-key must be configured via environment variable or config server");
        }
    }

    // getter / setter
}
```

```java
// ✅ 使用 Spring 的 jasypt 加密敏感配置
@Service
public class DataSyncService {

    private final DataSource dataSource; // 注入 DataSource，连接信息由配置管理

    public DataSyncService(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    public void syncData() {
        try (Connection conn = dataSource.getConnection()) {
            // ...
        }
    }
}
```

```java
// ✅ JWT 密钥从安全存储获取，长度合规
@Component
public class JwtUtil {

    @Value("${jwt.secret-key}")
    private String secretKey; // 至少 256 位，由运维通过 K8s Secret 注入

    public String generateToken(String userId) {
        // 使用 Keys.hmacShaKeyFor 确保密钥长度符合 HS256 要求
        SecretKey key = Keys.hmacShaKeyFor(Decoders.BASE64.decode(secretKey));
        return Jwts.builder()
                .setSubject(userId)
                .signWith(key)
                .compact();
    }
}
```

### 🔍 手动 Review 看什么？

| 检查项 | 为什么重要 |
|--------|------------|
| 搜索关键字：`password`、`secret`、`apiKey`、`token`、`Bearer` | Copilot 最常见的硬编码点 |
| 检查所有 `private static final String` 赋值的字符串值 | 静态常量是硬编码重灾区 |
| 检查 `application.properties` / `application.yml` 中的默认值 | Copilot 会"聪明地"帮你填一个假值 |
| 检查 `.gitignore` 是否排除了 `*.p12`、`*.pem`、`application-*.yml` | 防止证书文件泄露 |
| 跑一遍 `git log -p \| grep -E "(sk-\|AKIA\|ghp_)"` | 检查历史提交中是否已泄露 |

---

## 三、检查点 2：SQL 注入漏洞

### 为什么 Copilot 容易犯这个错？

当你让 Copilot 写一个"根据多个条件动态查询"的方法时，它倾向于直接拼接字符串——因为这样看起来最直观，最符合人类写伪代码的思路。它不理解 SQL 注入的危害，它只知道"这段代码在训练数据里出现过无数次"。

### 🔴 反例：Copilot 生成的"灵活查询"

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    @GetMapping("/search")
    public List<User> searchUsers(
            @RequestParam String keyword,
            @RequestParam(required = false) String role,
            @RequestParam(required = false) String status) {

        // Copilot 自动拼接——典型的 SQL 注入漏洞
        StringBuilder sql = new StringBuilder("SELECT * FROM users WHERE 1=1");
        if (keyword != null) {
            sql.append(" AND (name LIKE '%" + keyword + "%' OR email LIKE '%" + keyword + "%')");
        }
        if (role != null) {
            sql.append(" AND role = '" + role + "'");
        }
        if (status != null) {
            sql.append(" AND status = '" + status + "'");
        }
        sql.append(" ORDER BY create_time DESC");

        return jdbcTemplate.query(sql.toString(), new BeanPropertyRowMapper<>(User.class));
    }
}
```

攻击者只需传参：`keyword=test&role=admin' OR '1'='1`，即可绕过权限控制查询所有管理员数据。

### 🟢 正例：参数化查询

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    @GetMapping("/search")
    public List<User> searchUsers(
            @RequestParam String keyword,
            @RequestParam(required = false) String role,
            @RequestParam(required = false) String status) {

        // ✅ 使用参数化查询，杜绝 SQL 注入
        StringBuilder sql = new StringBuilder("SELECT * FROM users WHERE 1=1");
        List<Object> params = new ArrayList<>();

        if (keyword != null && !keyword.isBlank()) {
            sql.append(" AND (name LIKE ? OR email LIKE ?)");
            params.add("%" + keyword + "%");
            params.add("%" + keyword + "%");
        }
        if (role != null && !role.isBlank()) {
            sql.append(" AND role = ?");
            params.add(role);
        }
        if (status != null && !status.isBlank()) {
            sql.append(" AND status = ?");
            params.add(status);
        }
        sql.append(" ORDER BY create_time DESC");

        return jdbcTemplate.query(sql.toString(), params.toArray(), new BeanPropertyRowMapper<>(User.class));
    }
}
```

使用 MyBatis 时同理——`#{}` 是预编译，`${}` 是直接拼接。**Review 时必须搜索所有 `${`**。

```java
// ❌ Copilot 容易写出这种
@Select("SELECT * FROM orders WHERE status = '${status}'")
List<Order> findByStatus(String status);

// ✅ 正确写法
@Select("SELECT * FROM orders WHERE status = #{status}")
List<Order> findByStatus(String status);
```

### 🔍 手动 Review 看什么？

| 检查项 | 为什么重要 |
|--------|------------|
| 搜索 `"SELECT`、`"UPDATE`、`"INSERT`、`"DELETE` 等字符串拼接 SQL | 任何字符串拼接的 SQL 都是潜在注入点 |
| 搜索 `+= "` 后面跟 SQL 关键字的模式 | Copilot 擅长的"动态拼 SQL"写法 |
| 在 MyBatis XML / 注解中搜索 `${` | `${}` = 直接拼接，是注入的定时炸弹 |
| 检查 `ORDER BY`、`GROUP BY` 后的动态字段 | 这些字段通常无法用 `?` 参数化，必须做白名单校验 |
| 检查 `LIKE '%` + 变量组合 | 典型的 SQL 拼接特征 |

---

## 四、检查点 3：空指针与异常处理缺陷

### 为什么 Copilot 容易犯这个错？

Copilot 的补全逻辑是"给定上下文，预测最大概率的下一段代码"。当你写了一个 `try` 块，它看到大量的训练样本中 `catch` 块都是空的或者只打一行日志——于是它也这么干了。它不会思考："如果这里出异常，调用方怎么知道？事务会回滚吗？"

### 🔴 反例：吞掉异常

```java
@Service
public class OrderService {

    @Autowired
    private PaymentGateway paymentGateway;

    @Autowired
    private OrderRepository orderRepository;

    @Autowired
    private InventoryService inventoryService;

    @Transactional
    public OrderResult placeOrder(OrderRequest request) {
        // Copilot 没有检查 request 是否为 null
        // Copilot 没有检查 request.getItems() 是否为空
        Order order = new Order();
        order.setUserId(request.getUserId());
        order.setTotalAmount(request.getItems().stream()
                .mapToDouble(OrderItem::getPrice)
                .sum()); // 潜在 NPE

        try {
            PaymentResult payment = paymentGateway.charge(request.getPaymentInfo());
            order.setPaymentId(payment.getTransactionId());
        } catch (Exception e) {
            // Copilot 生成的吞异常代码——静默失败！
            log.error("payment failed");
            // 没有 throw，没有补偿，没有通知用户
        }

        try {
            inventoryService.deductStock(request.getItems());
        } catch (Exception e) {
            // 又吞了！库存扣减失败，但订单已经创建了
            e.printStackTrace(); // 更糟——生产环境连日志都看不到
        }

        orderRepository.save(order);
        return OrderResult.success(order);
    }
}
```

这个例子的问题链：
1. `request` 或 `request.getItems()` 为 null 时，直接在 `sum()` 行抛出 NPE
2. 支付异常被吞掉→订单状态错误→用户以为支付成功
3. 库存扣减异常被吞掉→超卖
4. `e.printStackTrace()` 输出到标准错误，没有任何监控系统能捕获

### 🟢 正例：分层防御与显式失败

```java
@Service
public class OrderService {

    private final PaymentGateway paymentGateway;
    private final OrderRepository orderRepository;
    private final InventoryService inventoryService;
    private final AlertService alertService;

    public OrderService(PaymentGateway paymentGateway,
                        OrderRepository orderRepository,
                        InventoryService inventoryService,
                        AlertService alertService) {
        this.paymentGateway = paymentGateway;
        this.orderRepository = orderRepository;
        this.inventoryService = inventoryService;
        this.alertService = alertService;
    }

    @Transactional
    public OrderResult placeOrder(OrderRequest request) {
        // ✅ 入口参数校验，快速失败
        Objects.requireNonNull(request, "OrderRequest must not be null");
        if (request.getItems() == null || request.getItems().isEmpty()) {
            throw new IllegalArgumentException("Order must contain at least one item");
        }
        if (request.getPaymentInfo() == null) {
            throw new IllegalArgumentException("Payment info is required");
        }

        // ✅ 安全计算
        double totalAmount = request.getItems().stream()
                .filter(Objects::nonNull)
                .mapToDouble(item -> Objects.requireNonNullElse(item.getPrice(), 0.0))
                .sum();

        Order order = new Order();
        order.setUserId(request.getUserId());
        order.setTotalAmount(totalAmount);
        order.setStatus(OrderStatus.PENDING);

        try {
            PaymentResult payment = paymentGateway.charge(request.getPaymentInfo());
            order.setPaymentId(payment.getTransactionId());
            order.setStatus(OrderStatus.PAID);
        } catch (PaymentException e) {
            log.error("Payment failed for user: {}", request.getUserId(), e);
            order.setStatus(OrderStatus.PAYMENT_FAILED);
            orderRepository.save(order);
            throw new BusinessException("支付失败，请重试", e); // ✅ 明确抛给上层
        }

        try {
            inventoryService.deductStock(request.getItems());
        } catch (InsufficientStockException e) {
            // ✅ 库存不足：退款 + 标记失败 + 告警
            paymentGateway.refund(order.getPaymentId());
            order.setStatus(OrderStatus.CANCELLED);
            orderRepository.save(order);
            alertService.sendAlert("库存异常", "订单 %s 因库存不足取消".formatted(order.getId()));
            throw new BusinessException("商品库存不足，订单已取消", e);
        }

        orderRepository.save(order);
        return OrderResult.success(order);
    }
}
```

### 🔍 手动 Review 看什么？

| 检查项 | 为什么重要 |
|--------|------------|
| 搜索 `catch (Exception e) { }` 或 `catch` 块只有一行日志 | 吞异常是 Copilot 的"舒适区" |
| 搜索 `e.printStackTrace()` | 生产环境无用，应换成 `log.error` |
| 检查所有流式调用 `.stream().findFirst().get()` | `Optional.get()` 无值检查即抛异常 |
| 搜索 `return null;` | 大多数时候应该返回 `Optional` 或抛异常 |
| 检查 `@Transactional` 方法中的 catch 块 | 事务内吞异常会导致部分提交 |

---

## 五、检查点 4：线程安全问题

### 为什么 Copilot 容易犯这个错？

Spring 的 Bean 默认是单例的，这是每个 Java 开发者都该知道的常识。但 Copilot 在补全时没有"这个类会被 Spring 管理"的感知能力——它看到你定义了一个字段，就自然地帮你生成了 setter 和方法来修改它。它不知道这个字段会被多个请求并发访问。

### 🔴 反例：Spring Bean 中的可变状态

```java
@Service
public class ReportService {

    // ❌ 单例 Bean 中的可变字段——线程不安全！
    private ReportContext currentContext;

    // ❌ SimpleDateFormat 不是线程安全的！
    private static final SimpleDateFormat DATE_FORMAT = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");

    // ❌ 共享的集合对象
    private List<String> processingQueue = new ArrayList<>();

    public Report generateReport(ReportRequest request) {
        // 请求 A 设置上下文，请求 B 覆盖——数据错乱
        currentContext = new ReportContext(request.getUserId(), request.getDateRange());

        // 请求 A 和 B 同时写入同一个 List——ConcurrentModificationException
        processingQueue.add(request.getReportType());

        // 日期格式化——多线程下结果可能完全错误甚至死循环
        String formattedDate = DATE_FORMAT.format(new Date());

        // 模拟一些耗时的报表生成逻辑
        try { Thread.sleep(2000); } catch (InterruptedException ignored) {}

        // 此时 currentContext 可能已经是其他请求的了！
        return buildReport(currentContext);
    }
}
```

### 🟢 正例：无状态设计 + 线程安全工具

```java
@Service
public class ReportService {

    // ✅ 使用线程安全的 DateTimeFormatter（Java 8+）
    private static final DateTimeFormatter DATE_FORMATTER =
            DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");

    // ✅ 使用线程安全的集合
    private final ConcurrentLinkedQueue<String> processingQueue = new ConcurrentLinkedQueue<>();

    public Report generateReport(ReportRequest request) {
        // ✅ 每个请求使用局部变量，不存在共享
        ReportContext context = new ReportContext(request.getUserId(), request.getDateRange());

        // ✅ 线程安全队列，不会抛出 ConcurrentModificationException
        processingQueue.add(request.getReportType());

        // ✅ DateTimeFormatter 是不可变对象，线程安全
        String formattedDate = LocalDateTime.now().format(DATE_FORMATTER);

        // 耗时操作不会影响其他线程的上下文
        try { Thread.sleep(2000); } catch (InterruptedException e) {
            Thread.currentThread().interrupt(); // ✅ 恢复中断状态
        }

        return buildReport(context);
    }

    // 如果确实需要共享状态，使用 ThreadLocal
    private static final ThreadLocal<ReportContext> CONTEXT_HOLDER = new ThreadLocal<>();

    public Report generateReportWithThreadLocal(ReportRequest request) {
        try {
            CONTEXT_HOLDER.set(new ReportContext(request.getUserId(), request.getDateRange()));
            // ... 业务逻辑
            return buildReport(CONTEXT_HOLDER.get());
        } finally {
            CONTEXT_HOLDER.remove(); // ✅ 必须清理，防止内存泄漏
        }
    }
}
```

### 🔍 手动 Review 看什么？

| 检查项 | 为什么重要 |
|--------|------------|
| 搜索 `@Service` / `@Component` / `@RestController` 类中的非 final 实例字段 | 单例 + 可变字段 = 线程不安全 |
| 搜索 `SimpleDateFormat` | 经典的线程不安全类，应替换为 `DateTimeFormatter` |
| 搜索 `HashMap` 在 Bean 中作为字段 | 应使用 `ConcurrentHashMap` |
| 搜索 `ArrayList` 在 Bean 中作为字段 | 应使用 `CopyOnWriteArrayList` 或 `ConcurrentLinkedQueue` |
| 搜索 `synchronized` 块 | 可能存在死锁或性能瓶颈 |

---

## 六、检查点 5：性能陷阱

### 为什么 Copilot 容易犯这个错？

Copilot 是"局部最优"的：它只看你当前方法的上下文，看不到 Spring Data JPA 的懒加载配置，看不到这个循环在生产环境会跑 10 万次，看不到下游 API 有 QPS 限制。它只关心"这里写什么代码能通过编译"。

### 🔴 反例：N+1 查询 + 循环调 API + 资源泄露

```java
@Service
public class ReportGenerationService {

    @Autowired
    private OrderRepository orderRepository;

    @Autowired
    private UserServiceClient userServiceClient;

    @Autowired
    private RestTemplate restTemplate;

    public void generateMonthlyReport(int year, int month) {
        // ❌ N+1 查询：查所有订单，再逐个查用户信息
        List<Order> orders = orderRepository.findAll();

        for (Order order : orders) {
            // ❌ 每次循环调用远程 API——1000 个订单 = 1000 次 HTTP 调用
            User user = userServiceClient.getUserById(order.getUserId());
            order.setUserName(user.getName());

            // ❌ 循环内创建 RestTemplate（也是资源浪费）
            RestTemplate localTemplate = new RestTemplate();
            String productName = localTemplate.getForObject(
                    "http://product-service/api/products/" + order.getProductId(),
                    String.class);

            // ❌ 循环内操作数据库
            orderRepository.updateReportStatus(order.getId(), "PROCESSED");
        }

        // ❌ 文件流未关闭，资源泄露
        try {
            FileInputStream fis = new FileInputStream("/tmp/report_template.xlsx");
            // 处理文件...
            // 没有 close()！
        } catch (IOException e) {
            log.error("failed to read template", e);
        }
    }
}
```

### 🟢 正例：批量操作 + 资源管理

```java
@Service
public class ReportGenerationService {

    private final OrderRepository orderRepository;
    private final UserServiceClient userServiceClient;
    private final RestTemplate restTemplate; // ✅ 单例注入，复用连接池

    public ReportGenerationService(OrderRepository orderRepository,
                                   UserServiceClient userServiceClient,
                                   RestTemplate restTemplate) {
        this.orderRepository = orderRepository;
        this.userServiceClient = userServiceClient;
        this.restTemplate = restTemplate;
    }

    public void generateMonthlyReport(int year, int month) {
        List<Order> orders = orderRepository.findAll();

        // ✅ 批量查询用户信息，一次 RPC 调用解决
        Set<Long> userIds = orders.stream()
                .map(Order::getUserId)
                .collect(Collectors.toSet());
        Map<Long, User> userMap = userServiceClient.getUsersByIds(userIds);

        // ✅ 批量查询产品信息
        Set<Long> productIds = orders.stream()
                .map(Order::getProductId)
                .collect(Collectors.toSet());
        Map<Long, String> productNameMap = productServiceClient.batchGetProductNames(productIds);

        // ✅ 内存中组装，避免循环操作数据库
        for (Order order : orders) {
            User user = userMap.get(order.getUserId());
            if (user != null) {
                order.setUserName(user.getName());
            }
            order.setProductName(productNameMap.getOrDefault(order.getProductId(), "Unknown"));
        }

        // ✅ 批量更新状态
        List<Long> orderIds = orders.stream().map(Order::getId).toList();
        orderRepository.batchUpdateReportStatus(orderIds, "PROCESSED");

        // ✅ try-with-resources 自动关闭资源
        try (FileInputStream fis = new FileInputStream("/tmp/report_template.xlsx");
             Workbook workbook = new XSSFWorkbook(fis)) {
            // 处理文件...
        } catch (IOException e) {
            log.error("Failed to read report template", e);
        }
    }
}
```

### 🔍 手动 Review 看什么？

| 检查项 | 为什么重要 |
|--------|------------|
| 搜索 `for (` 或 `forEach` 内是否有数据库操作 | N+1 查询的典型特征 |
| 搜索循环内的 HTTP 调用 / RPC 调用 | 循环调远程 API = 性能杀手 |
| 搜索 `new RestTemplate()` 在方法内 | 应单例注入，复用连接池 |
| 搜索 `new FileInputStream` / `new FileOutputStream` / `new BufferedReader` 等 | 检查是否在 try-with-resources 中 |
| 搜索 `List.add()` 在循环外（`Collectors.toList()`） | Copilot 喜欢写 for 循环逐条 add |

---

## 七、附赠：AI 代码审查 Prompt 模板

如果你想让另一个 AI（比如 ChatGPT、Claude）帮你 Review Copilot 生成的代码，可以直接复制下面的 Prompt：

```
你是一位资深 Java 安全审计专家。请审查以下代码，重点关注：

1. **安全漏洞**：
   - 是否有硬编码的密钥、密码、Token、API Key？
   - 是否存在 SQL 注入风险（字符串拼接、MyBatis ${}）？
   - 是否存在 SSRF、XXE、路径遍历等风险？

2. **异常处理**：
   - catch 块是否吞掉了异常（空块、只打日志不抛出）？
   - 是否有 e.printStackTrace()？
   - 事务方法中的异常是否正确传播？

3. **线程安全**：
   - Spring 管理的 Bean 中是否有可变实例字段？
   - 是否使用了 SimpleDateFormat、HashMap 等非线程安全类？

4. **性能问题**：
   - 是否存在 N+1 查询？
   - 循环内是否有 API 调用或数据库操作？
   - 资源（流、连接）是否正确关闭？

5. **代码质量**：
   - 缺少 null 检查的地方有哪些？
   - Optional 使用是否正确？

请对每一个发现的问题给出：
- 严重级别（Critical / High / Medium / Low）
- 所在行号（如果代码已标注）
- 风险说明
- 修复建议（附带代码示例）

--- 待审查代码 ---
[粘贴你的 Java 代码]
```

---

## 八、自动化方案：GitHub Actions + AI 自动 Code Review

手动 Review 每一段 AI 生成的代码当然最可靠，但现实是——你不可能在每次 PR 时手动检查 50 个文件。这就需要一个自动化防线。

以下是一个基于 GitHub Actions + OpenAI API 的轻量级自动化 Review 方案：

```yaml
# .github/workflows/ai-code-review.yml
name: AI Code Review

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  ai-review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Get changed Java files
        id: changed-files
        run: |
          git diff --name-only origin/${{ github.base_ref }}...HEAD | grep '\.java$' > changed_files.txt
          echo "count=$(wc -l < changed_files.txt)" >> $GITHUB_OUTPUT

      - name: AI Review each file
        if: steps.changed-files.outputs.count > 0
        run: |
          while IFS= read -r file; do
            echo "🔍 Reviewing: $file"
            CONTENT=$(cat "$file")

            # 构造 System Prompt（硬编码检测 + SQL 注入为重点）
            REVIEW_PROMPT=$(cat <<'PROMPT'
          Review this Java code for:
          1. Hardcoded secrets, passwords, keys, tokens
          2. SQL injection (string concatenation, ${} in MyBatis)
          3. Swallowed exceptions (empty catch, only log)
          4. Mutable fields in @Service/@Component classes
          5. N+1 queries, API calls inside loops
          6. Unclosed resources (streams, connections)

          Output format:
          - [SEVERITY] file:line - issue description
          If no issues, output "PASS".
          PROMPT
          )

            # 调用 OpenAI API（替换为你的 API Key）
            RESPONSE=$(curl -s https://api.openai.com/v1/chat/completions \
              -H "Content-Type: application/json" \
              -H "Authorization: Bearer ${{ secrets.OPENAI_API_KEY }}" \
              -d "$(jq -n --arg system "$REVIEW_PROMPT" --arg code "$CONTENT" '{
                model: "gpt-4o-mini",
                messages: [
                  {role: "system", content: $system},
                  {role: "user", content: $code}
                ],
                temperature: 0.1,
                max_tokens: 2000
              }')")

            REVIEW=$(echo "$RESPONSE" | jq -r '.choices[0].message.content')

            if [ "$REVIEW" != "PASS" ]; then
              echo "## 🔍 AI Code Review: \`$file\`" >> review_report.md
              echo "" >> review_report.md
              echo "$REVIEW" >> review_report.md
              echo "" >> review_report.md
              echo "---" >> review_report.md
            fi
          done < changed_files.txt

      - name: Post review as PR comment
        if: always()
        uses: thollander/actions-comment-pull-request@v2
        with:
          filePath: review_report.md
          comment_tag: ai-code-review
```

这个方案的几个关键设计点：

1. **只审查变更的文件**：`git diff` 获取范围，避免全量扫描浪费时间
2. **使用 `gpt-4o-mini`**：成本极低（$0.15/1M input tokens），审查一个中型项目也就几美分
3. **结构化输出**：让 AI 输出 `[SEVERITY] file:line - issue`，方便后续自动打标签
4. **PR 评论**：结果直接贴在 PR 下面，不阻塞 CI（`if: always()`）
5. **可以结合 SonarQube / SpotBugs**：AI 审查专注语义层，静态分析工具查规范层，互补

> 进阶建议：当 AI 发现 Critical / High 级别问题时，可以触发 `github-actions[bot]` 自动将 PR 标记为 "Changes Requested"，强制开发人员修复后再合入。

---

## 九、结语与下一篇预告

回到开篇那个故事。我那位朋友后来做了三件事：

1. 把 `.env.example` 之外的所有带默认 Key 的配置全部清理了一遍
2. 在 CI 流水线里加了一个 `grep` 脚本，扫描 `sk-`、`AKIA`、`ghp_` 等密钥特征
3. 定了一条铁律：**AI 生成的代码，必须经过另一个 AI 或另一个人的 Review 才能合入 master**

Copilot 不是银弹，它是一把极快的刀——快到你不需要会切菜就能下厨，但也快到你可能没发现自己切到了手指。

你要做的不是拒绝它，而是**给它套上一套防护体系**：

```
开发者 + Copilot 生成代码
      ↓
  本地 IDE 静态检查（SonarLint / SpotBugs）
      ↓
  Git Pre-commit Hook（密钥扫描）
      ↓
  PR 提交 → GitHub Actions 自动触发 AI Review
      ↓
  另一个开发者手动 Review（本文 5 个检查点）
      ↓
  合并到主分支
```

---

**下一篇预告：《Cursor 全面评测：它真的比 Copilot 强吗？——从代码生成、多文件重构到 Agent 模式的深度对比》**

我们将从以下维度横向对比 Cursor 和 Copilot：
- 代码生成质量（同题对比，盲评打分）
- 多文件重构能力（Cursor 的 Composer vs Copilot 的 Edit）
- Agent 模式实测（让 AI 自己读 Bug Report → 定位代码 → 生成 Fix → 写测试）
- 价格与性价比分析

关注我，不错过下一篇干货。如果这篇文章对你有用，欢迎收藏、点赞、转发——你的支持是我继续深挖 AI 编程工具链的动力。

---

*本文系原创，首发于 CSDN，转载请注明出处。*
