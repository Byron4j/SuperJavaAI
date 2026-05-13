# Prompt 驱动 AI 排查线上 Bug：把堆栈信息和日志喂给 AI，5分钟解决原本要排查一下午的线上故障

> 线上报 `java.lang.NullPointerException`，没有定位到具体代码行，堆栈信息全是框架层的（Spring AOP 代理、CGLIB、`MethodProxy.invoke`……）。你盯着堆栈看了半小时，心里默默问候了写这段代码的同事（其实是你自己上个月写的）。今天，一招解决：把堆栈喂给 AI，5分钟出结论。

---

## 开篇：线上 Bug 排查的"三重困境"

干了这么多年 Java，线上出 Bug 最让人崩溃的不是 Bug 本身，而是：

**困境一：信息太少。** 就一行 `NullPointerException`，堆栈全是框架代码，你根本不知道是哪个业务对象为 null。

**困境二：信息太多。** 日志刷了几万行，你一行行翻，眼睛都看花了还不知道重点在哪。

**困境三：复现不了。** 偶发的并发 Bug，开发环境死活重现不了，只能在生产环境"等它再来一次"。

今天给每个困境一套 AI Prompt 解法。

---

## 实战一：NPE / Caused-by 链分析——把堆栈喂给 AI 让它找到真正原因

### 现象

```
2025-04-12 14:23:15.456 ERROR [http-nio-8080-exec-7] 
  c.e.o.controller.OrderController:78 [traceId=abc123] 
  订单查询异常: orderId=98765

java.lang.NullPointerException: null
  at com.example.order.service.OrderService.getOrderDetail(OrderService.java:156)
  at com.example.order.service.OrderService$$FastClassBySpringCGLIB$$a1b2c3d4.invoke(<generated>)
  at org.springframework.cglib.proxy.MethodProxy.invoke(MethodProxy.java:218)
  at org.springframework.aop.framework.CglibAopProxy$CglibMethodInvocation.invokeJoinpoint(CglibAopProxy.java:792)
  at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:163)
  at org.springframework.aop.framework.CglibAopProxy$CglibMethodInvocation.proceed(CglibAopProxy.java:762)
  at org.springframework.transaction.interceptor.TransactionInterceptor$1.proceedWithInvocation(TransactionInterceptor.java:123)
  at org.springframework.transaction.interceptor.TransactionAspectSupport.invokeWithinTransaction(TransactionAspectSupport.java:388)
  ... 40 more lines of framework code ...
```

看到这个堆栈，你是不是心里一紧：`OrderService.java:156`，我这一行到底做了什么？

### 整理信息

在问 AI 之前，你需要收集三样东西：

1. **完整堆栈**（包括 Caused by 链，不要只截最上面几行）
2. **出错的代码片段**（第 156 行前后各 20 行）
3. **请求参数**（如果能拿到的话）

### Prompt

```
你是一个资深Java线上故障排查专家。以下是一个NullPointerException的完整上下文，请分析并定位根因。

【环境信息】
框架：Spring Boot 2.7 + MyBatis-Plus
数据库：MySQL 8.0
缓存：Redis

【异常堆栈】（完整）
java.lang.NullPointerException: null
  at com.example.order.service.OrderService.getOrderDetail(OrderService.java:156)
  at com.example.order.service.OrderService$$FastClassBySpringCGLIB$$a1b2c3d4.invoke(<generated>)
  ...(完整堆栈)...

【出错的代码】OrderService.java 第 140-170 行
```java
public OrderDetailVO getOrderDetail(Long orderId) {
    // 第145行：查询订单
    Order order = orderMapper.selectById(orderId);
    if (order == null) {
        throw new BusinessException("订单不存在: " + orderId);
    }

    // 第151行：获取用户信息（可能为null）
    UserDTO user = userCacheManager.getUser(order.getUserId());
    
    // 第153行：获取订单商品
    List<OrderItem> items = orderItemMapper.selectByOrderId(orderId);
    
    // 第156行 ← 这里报NPE！
    List<Product> products = items.stream()
        .map(item -> productService.getProduct(item.getProductId()))
        .filter(Objects::nonNull)
        .collect(Collectors.toList());

    // 第162行：计算优惠
    CouponDTO coupon = couponService.getByOrderId(orderId);
    BigDecimal discount = coupon.getDiscountAmount(); // 这里也可能是null

    // 组装VO
    return OrderDetailVO.builder()
        .order(order)
        .user(user)
        .items(items)
        .products(products)
        .discount(discount)
        .build();
}
```

【请求上下文】（从日志中提取）
- orderId: 98765
- userId: 10023
- traceId: abc123

【分析要求】
1. 第156行除了items.stream()本身，还有哪些操作可能导致NPE？请逐一排查
2. 对于排查出来的每个可能为null的对象，给出"为什么它会为null"的合理推测
3. 给出修复方案（不仅是try-catch，要有根因修复）
4. 扫描这个方法中其他潜在的NPE风险点

【约束】
- 不能简单地说"加if判空"——要分析为什么那个对象会为null，是业务逻辑问题还是数据问题
```

### AI 输出

```
═══════════════════════════════════════════
         NPE 根因分析报告
═══════════════════════════════════════════

【根因定位】

第156行 NPE 的根因是：productService.getProduct() 返回了 null。

分析链路：
  items.stream()                         ← items 非空（否则上面 selectByOrderId 返回空list）
    .map(item -> item.getProductId())    ← productId 可能为 null！
    .map(id -> productService.getProduct(id))
    ↑
    productService.getProduct(null)
      → 内部调用了 productMapper.selectById(null)
      → MyBatis-Plus 对 null 的处理：返回 null
      → stream 中产生了 null 元素
      → 但由于有 .filter(Objects::nonNull)，这里不会直接 NPE

真正的问题：
  回到代码第156行，编译器报的第156行是完整的 stream 链，
  JVM 报的 NPE 位置不一定精确到链式调用的具体哪一步。

最可能的情况（按概率排序）：

1. 【90%概率】items 为 null
   orderItemMapper.selectByOrderId(orderId) 返回了 null 而非空集合
   → items.stream() 直接 NPE
   → 补充验证：查看 MyBatis 的 resultType 配置，是否指定返回类型

2. 【8%概率】productService 为 null（Spring 注入失败）
   但如果是依赖注入问题，启动时就会报错，上线后不会偶发

3. 【2%概率】productService.getProduct() 内部抛出异常，但异常被吞掉
   导致 stream 中产生了一个"出乎意料的 null"

═══════════════════════════════════════════
        修复方案
═══════════════════════════════════════════

【方案一：防御式编程（紧急修复）】
*/

public OrderDetailVO getOrderDetail(Long orderId) {
    Order order = orderMapper.selectById(orderId);
    if (order == null) {
        throw new BusinessException("订单不存在: " + orderId);
    }

    // 修复1：items 空值保护
    List<OrderItem> items = Optional.ofNullable(
        orderItemMapper.selectByOrderId(orderId))
        .orElse(Collections.emptyList());

    // 修复2：过滤掉 productId 为 null 的商品项
    List<Product> products = items.stream()
        .filter(item -> item.getProductId() != null)
        .map(item -> {
            try {
                return productService.getProduct(item.getProductId());
            } catch (Exception e) {
                log.warn("获取商品信息失败: productId={}", item.getProductId(), e);
                return null;
            }
        })
        .filter(Objects::nonNull)
        .collect(Collectors.toList());

    // 修复3：优惠券空值保护
    BigDecimal discount = Optional.ofNullable(
        couponService.getByOrderId(orderId))
        .map(CouponDTO::getDiscountAmount)
        .orElse(BigDecimal.ZERO);

    UserDTO user = userCacheManager.getUser(order.getUserId());

    return OrderDetailVO.builder()
        .order(order)
        .user(user)
        .items(items)
        .products(products)
        .discount(discount)
        .build();
}

/*
【方案二：根本性修复（推荐）】

问题根源：MyBatis 的 Mapper 方法可能返回 null
解决：在 MyBatis 配置中确保返回空集合而非 null
*/

// ① MyBatis-Plus 配置
@Configuration
public class MybatisPlusConfig {
    @Bean
    public ConfigurationCustomizer configurationCustomizer() {
        return configuration -> {
            // 确保查询结果 null 时返回空集合
            configuration.setReturnInstanceForEmptyRow(true);
            // 或者使用 MyBatis-Plus 的配置
            GlobalConfig globalConfig = new GlobalConfig();
            globalConfig.setDbConfig(new GlobalConfig.DbConfig()
                .setSelectStrategy(DbType.MYSQL));
        };
    }
}

// ② 在 Mapper 层统一处理 null
@Mapper
public interface OrderItemMapper extends BaseMapper<OrderItem> {

    default List<OrderItem> selectByOrderIdSafe(Long orderId) {
        return Optional.ofNullable(selectByOrderId(orderId))
            .orElse(Collections.emptyList());
    }

    List<OrderItem> selectByOrderId(Long orderId);
}

/*
═══════════════════════════════════════════
      该方法中其他潜在的 NPE 风险
═══════════════════════════════════════════

| 行号 | 代码                                | 风险点              | 严重程度 |
|------|------------------------------------|--------------------|---------|
| 146  | order.getUserId()                  | order 可空但已判空   | ❌ 安全 |
| 156  | items.stream()                     | items 可能为 null   | 🔴 高   |
| 157  | item.getProductId()                | productId 可能为null | 🟡 中   |
| 162  | coupon.getDiscountAmount()         | coupon 可能为 null  | 🔴 高   |
| 166  | order.getXxx() → builder.xxx()     | order 内部字段可能为null | 🟡 中   |

建议：引入 @Nullable 注解 + IDEA 检查 + 静态代码扫描（SonarQube）
*/
```

### 修复验证

改完代码后，用这个 Prompt 让 AI 替你检查：

```
我修改了OrderService.getOrderDetail方法，请对修改后的代码做"NPE安全审查"：

1. 扫描每一个 . 操作符（方法调用、字段访问），确认左侧引用一定非空
2. 检查所有 Stream 链式调用，确保中间操作不会引入 null
3. 检查所有 @Autowired 字段，确认它们在方法调用前已初始化
4. 如果发现任何可能为 null 的引用，请指出并给出修复建议

修改后的代码：
[粘贴你的代码]
```

---

## 实战二：并发问题——偶发的 ConcurrentModificationException

### 现象

每隔几天就会报一次 `ConcurrentModificationException`，没有任何规律，重启后就好。堆栈指向一个 `ArrayList` 的 `iterator` 操作。

### 整理信息

```java
// 报错代码
@Service
public class PromotionService {

    // 共享的促销规则列表（通过定时任务从DB加载）
    private static final List<PromotionRule> ACTIVE_RULES = new ArrayList<>();

    @Scheduled(fixedDelay = 60000)
    public void reloadRules() {
        List<PromotionRule> latestRules = ruleMapper.selectActiveRules();
        ACTIVE_RULES.clear();           // ← 线程A在这里清空
        ACTIVE_RULES.addAll(latestRules); // ← 线程A在这里重新填充
    }

    public BigDecimal calculateDiscount(Order order) {
        List<PromotionRule> matched = new ArrayList<>();
        for (PromotionRule rule : ACTIVE_RULES) {  // ← 线程B在这里遍历 → CME!
            if (rule.match(order)) {
                matched.add(rule);
            }
        }
        // ...
    }
}
```

### Prompt

```
你是一个Java并发专家。以下代码偶发ConcurrentModificationException，请分析根因并给出修复方案。

【问题代码】
[粘贴上面的PromotionService代码]

【问题现象】
- 每隔1-3天出现一次，没有固定时间规律
- 报错信息：ConcurrentModificationException at ArrayList$Itr.next
- 重启服务后恢复
- 只有在高并发查促销 + 定时任务刷新同时发生时出现

【分析要求】
1. 精确描述触发条件（什么情况下线程A和线程B会冲突）
2. 给出3种修复方案，按推荐度排序
3. 每种方案写明适用场景、优缺点、性能影响
4. 不得使用synchronized给整个方法加锁（会影响所有读操作）

【约束】
- 这是一个读多写少的场景（每秒1000次读，每分钟1次写）
- 不能接受写操作阻塞读操作
```

### AI 输出

```java
/*
═══════════════════════════════════════════
      ConcurrentModificationException 根因分析
═══════════════════════════════════════════

【触发条件分析】

时序如下：
  T0: 线程B开始 for (rule : ACTIVE_RULES)
        → ArrayList 内部创建了 Itr 迭代器，记录了 expectedModCount = N
  T1: 线程A执行 ACTIVE_RULES.clear()
        → ArrayList.modCount 变成 N+1
  T2: 线程B迭代器调用 Itr.next()
        → 检查 modCount(N+1) != expectedModCount(N)
        → 抛出 ConcurrentModificationException

关键：只要 reloadRules() 和 calculateDiscount() 在同一个毫秒内并发就触发。

═══════════════════════════════════════════
        方案一：CopyOnWriteArrayList（推荐★★★）
═══════════════════════════════════════════
*/

@Service
public class PromotionService {

    // ★ 换成 CopyOnWriteArrayList
    private static final List<PromotionRule> ACTIVE_RULES = new CopyOnWriteArrayList<>();

    @Scheduled(fixedDelay = 60000)
    public void reloadRules() {
        List<PromotionRule> latestRules = ruleMapper.selectActiveRules();
        ACTIVE_RULES.clear();
        ACTIVE_RULES.addAll(latestRules);
    }

    public BigDecimal calculateDiscount(Order order) {
        // 读操作无需任何同步，直接遍历即可
        // CopyOnWriteArrayList 的迭代器操作的是"快照"，不会抛 CME
        for (PromotionRule rule : ACTIVE_RULES) {
            if (rule.match(order)) {
                // ...
            }
        }
    }
}

/*
  优点：读写完全无锁，读操作零开销
  缺点：写操作（reload）时要复制整个数组（规则列表通常很小，可忽略）
  适用：读多写少（1000次读/秒 vs 1次写/分钟 → 完美匹配）
  注意：ACTIVE_RULES 声明为 final，只替换内容不替换引用

═══════════════════════════════════════════
        方案二：读写锁 ReentrantReadWriteLock（推荐★★）
═══════════════════════════════════════════
*/

@Service
public class PromotionService {

    private static final List<PromotionRule> ACTIVE_RULES = new ArrayList<>();
    private final ReadWriteLock lock = new ReentrantReadWriteLock();

    @Scheduled(fixedDelay = 60000)
    public void reloadRules() {
        List<PromotionRule> latestRules = ruleMapper.selectActiveRules();
        lock.writeLock().lock();
        try {
            ACTIVE_RULES.clear();
            ACTIVE_RULES.addAll(latestRules);
        } finally {
            lock.writeLock().unlock();
        }
    }

    public BigDecimal calculateDiscount(Order order) {
        lock.readLock().lock();
        try {
            for (PromotionRule rule : ACTIVE_RULES) {
                if (rule.match(order)) {
                    // ...
                }
            }
        } finally {
            lock.readLock().unlock();
        }
        // ...
    }
}

/*
  优点：JDK 原生，不需要引入额外依赖；读操作可以并发
  缺点：读操作也需要获取锁（虽然是共享锁），有轻微性能开销
  适用：规则列表较大（CopyOnWriteArray 复制成本高时）

═══════════════════════════════════════════
        方案三：原子替换引用（最简单）
═══════════════════════════════════════════
*/

@Service
public class PromotionService {

    // ★ 用 volatile 保证可见性 + AtomicReference 保证原子替换
    private static volatile List<PromotionRule> ACTIVE_RULES = Collections.emptyList();

    @Scheduled(fixedDelay = 60000)
    public void reloadRules() {
        List<PromotionRule> latestRules = ruleMapper.selectActiveRules();
        // 先构建好新的不可变列表，再原子替换引用
        ACTIVE_RULES = Collections.unmodifiableList(latestRules);
    }

    public BigDecimal calculateDiscount(Order order) {
        // 获取当前快照（volatile保证读到最新值）
        List<PromotionRule> rules = ACTIVE_RULES;
        for (PromotionRule rule : rules) {
            if (rule.match(order)) {
                // ...
            }
        }
    }
}

/*
  优点：零锁、零竞争、代码最简单
  缺点：reload 期间的读操作可能读到旧数据（但这里是促销规则，可接受）
  适用：对数据一致性要求不高的缓存场景（推荐）
*/

/*
═══════════════════════════════════════════
              方案对比
═══════════════════════════════════════════

| 方案   | 写性能 | 读性能 | 一致性   | 复杂度 | 推荐场景     |
|--------|--------|--------|---------|--------|-------------|
| COWAL  | 低(复制)| 极高   | 最终一致 | 低     | 小数据量     |
| RWLock | 高      | 高     | 强一致   | 中     | 大数据量     |
| 引用替换| 高      | 极高   | 最终一致 | 极低   | 缓存场景     |

推荐：当前场景最适用「方案三：原子替换引用」或「方案一：CopyOnWriteArrayList」
*/
```

---

## 实战三：数据不一致——订单状态对不上

### 现象

运营反馈：一个订单在"已支付"状态下，用户却收到了"支付超时取消"的短信。查数据库发现 `order.status = 'PAID'`，但看用户端显示"订单已取消"。

### Prompt

```
你是一个Java分布式系统专家。以下是一个数据不一致的排查场景，请分析所有可能的原因。

【问题描述】
- 订单号：ORD-20250415-00888
- 数据库中的 status 字段：PAID（已支付）
- 用户端显示：订单已取消
- 支付回调日志显示：14:30:15 收到支付宝回调，更新订单状态为 PAID
- 取消定时任务日志显示：14:30:18 检测到订单超时未支付，执行取消逻辑
- 但数据库最终状态是 PAID，不是 CANCELLED

【关键代码】订单取消定时任务
@Service
public class OrderCancelJob {

    @Scheduled(fixedDelay = 10000)
    @Transactional
    public void cancelTimeoutOrders() {
        LocalDateTime deadline = LocalDateTime.now().minusMinutes(30);
        
        // 查询超时未支付的订单
        List<Order> timeoutOrders = orderMapper.selectList(
            new LambdaQueryWrapper<Order>()
                .eq(Order::getStatus, OrderStatus.CREATED)  // 只要CREATED状态
                .lt(Order::getCreateTime, deadline)
        );

        for (Order order : timeoutOrders) {
            // 取消订单
            order.setStatus(OrderStatus.CANCELLED);
            order.setCancelTime(LocalDateTime.now());
            orderMapper.updateById(order);
            
            // 发送取消短信
            smsService.sendCancelMessage(order.getUserId(), order.getId());
            
            // 恢复库存
            inventoryService.restoreStock(order.getItems());
        }
    }
}

【支付回调代码】
@PostMapping("/callback/alipay")
@Transactional
public String alipayCallback(@RequestBody AlipayCallbackRequest request) {
    String orderId = request.getOutTradeNo();
    Order order = orderMapper.selectById(orderId);
    
    if (order == null) {
        return "fail";
    }
    
    // 只有 CREATED 状态的订单才处理支付
    if (order.getStatus() != OrderStatus.CREATED) {
        // ★ 如果订单已被取消，这里返回成功但实际不处理
        log.warn("订单状态异常，不处理支付回调: orderId={}, status={}", 
            orderId, order.getStatus());
        return "success";  // ← 告诉支付宝成功了，实际没处理！
    }
    
    order.setStatus(OrderStatus.PAID);
    order.setPaidTime(LocalDateTime.now());
    orderMapper.updateById(order);
    return "success";
}

【分析要求】
1. 详细推理出最可能的事故发生过程（时序图）
2. 指出代码中最关键的3个Bug
3. 给出完整的修复方案
4. 说一下：为什么支付宝说"支付成功"而我们没有记录？
```

### AI 输出

```
═══════════════════════════════════════════
         数据不一致根因分析报告
═══════════════════════════════════════════

【事故时序还原】

T1 14:30:00  用户创建订单 ORD-00888，状态 = CREATED
T2 14:30:15  支付宝异步通知到达 → alipayCallback()
             → 查到 order.status = CREATED ✓
             → 开始事务
             → order.setStatus(PAID)
             → orderMapper.updateById(order)  ← 事务尚未提交！
T3 14:30:18  cancelTimeoutOrders() 读取数据
             → 因为 T2 事务还没提交（或者取消任务读的是快照/从库）
             → 读到 order.status = CREATED（脏读！）
             → 认为订单超时
             → order.setStatus(CANCELLED)
T4 14:30:19  T2 的事务提交：status = PAID（覆盖了 CANCELLED？不完全是）
T5 14:30:20  T3 的事务也提交：status = CANCELLED？
             ...
最终数据库显示 PAID，但现在取决于 T2 和 T3 的事务提交顺序

═══════════════════════════════════════════
      代码中最关键的 3 个 Bug
═══════════════════════════════════════════

Bug 1：【致命】支付回调对"非CREATED状态"返回success
  订单已被取消时，支付宝回调返回 "success"
  → 支付宝认为我们处理成功了
  → 不会重试通知
  → 用户的 300 块钱扣了，但订单显示"已取消"

Bug 2：【致命】取消任务没用"乐观锁"防并发
  orderMapper.updateById(order) 直接更新，没有任何版本控制
  → 如果支付回调和取消任务同时修改同一条订单
  → 后提交的事务覆盖先提交的

Bug 3：【严重】事务隔离级别问题
  @Transactional 默认隔离级别 READ_COMMITTED
  → 如果支付回调事务未提交，取消任务可能读到旧状态
  → 如果使用了读写分离（从库读），从库延时会放大这个问题

═══════════════════════════════════════════
           完整修复方案
═══════════════════════════════════════════

【修复1：乐观锁防并发】
*/

// 在 SQL 层面加版本号条件
@Update("UPDATE t_order SET status = #{status}, version = version + 1 " +
        "WHERE id = #{id} AND version = #{version}")
int updateWithVersion(Order order);

// 在代码中检查更新行数
public void cancelOrder(Long orderId) {
    Order order = orderMapper.selectById(orderId);
    int rows = orderMapper.updateWithVersion(
        Order.builder()
            .id(orderId)
            .status(OrderStatus.CANCELLED)
            .version(order.getVersion())
            .build()
    );
    if (rows == 0) {
        // 版本号不匹配，说明订单已经被其他操作修改了
        log.warn("订单取消失败(版本冲突)，可能已被支付: orderId={}", orderId);
        return;  // 不做后续操作（不发短信、不恢复库存）
    }
    // 取消成功，执行后续操作...
}

/*
【修复2：状态机约束】
*/

public enum OrderStatus {
    CREATED("待支付"),
    PAID("已支付") {
        @Override
        public boolean canTransitionFrom(OrderStatus from) {
            return from == CREATED;  // 只能从CREATED转为PAID
        }
    },
    CANCELLED("已取消") {
        @Override
        public boolean canTransitionFrom(OrderStatus from) {
            return from == CREATED;  // 只能从CREATED转为CANCELLED
        }
    },
    COMPLETED("已完成") {
        @Override
        public boolean canTransitionFrom(OrderStatus from) {
            return from == PAID;
        }
    };

    public boolean canTransitionFrom(OrderStatus from) {
        return true;
    }
}

// 统一的状态变更入口
@Transactional
public void changeStatus(Long orderId, OrderStatus targetStatus,
                          StatusChangeContext context) {
    Order order = orderMapper.selectById(orderId);
    if (order == null) {
        throw new BusinessException("订单不存在");
    }
    if (!targetStatus.canTransitionFrom(order.getStatus())) {
        throw new BusinessException(
            String.format("订单状态变更非法: %s → %s",
                order.getStatus(), targetStatus));
    }
    // 使用乐观锁更新
    int rows = orderMapper.updateStatusWithVersion(
        orderId, targetStatus, order.getVersion());
    if (rows == 0) {
        throw new BusinessException("订单状态变更冲突，请重试");
    }
    // 事件通知...
}

/*
【修复3：支付回调正确处理所有状态】
*/

@PostMapping("/callback/alipay")
public String alipayCallback(@RequestBody AlipayCallbackRequest request) {
    String orderId = request.getOutTradeNo();
    Order order = orderMapper.selectById(orderId);

    if (order == null) {
        return "fail";  // 订单不存在，让支付宝重试
    }

    if (order.getStatus() == OrderStatus.PAID) {
        return "success";  // 已支付，幂等返回
    }

    if (order.getStatus() == OrderStatus.CANCELLED) {
        // ★ 订单已取消但钱已付 → 触发退款流程，不是直接返回success！
        log.error("订单已取消但收到支付通知，将自动退款: orderId={}", orderId);
        refundService.autoRefund(orderId, request.getTradeNo());
        return "success";  // 退款已发起，告诉支付宝成功
    }

    if (order.getStatus() == OrderStatus.CREATED) {
        try {
            orderService.changeStatus(orderId, OrderStatus.PAID,
                StatusChangeContext.fromAlipay(request));
        } catch (BusinessException e) {
            log.error("支付回调处理失败: orderId={}", orderId, e);
            return "fail";  // 让支付宝重试
        }
        return "success";
    }

    return "fail";  // 未知状态
}
```

---

## 实战四：性能突降——同一接口昨天 100ms 今天 2s

### 场景

接口 `/api/product/list`，昨天 P99 是 100ms，今天同样的流量变成了 2000ms。没有发布代码，没有配置变更。把慢 SQL 日志、代码、最近的操作日志一起喂给 AI。

### Prompt

```
你是一个Java性能诊断专家。以下接口性能突降，请分析可能原因。

【异常表现】
- 接口：GET /api/product/list?categoryId=1&page=1&size=20
- 昨天 15:00 P99: 100ms
- 今天 15:00 P99: 2000ms（同样QPS 5000）
- 中间无代码发布、无配置变更

【慢SQL统计】（Druid SQL监控）
昨天 15:00：
  SQL: SELECT * FROM product WHERE category_id=? AND status=1
       LIMIT 20 OFFSET 0
  平均: 3ms, 扫描: 20行, 索引: idx_category_status

今天 15:00：
  同一SQL
  平均: 1800ms, 扫描: 800000行, 索引: 无（全表扫描）

【补充信息】
- MySQL 8.0, InnoDB
- 索引 idx_category_status(category_id, status) 确认存在
- 今天凌晨运营导入了 50 万条新商品（category_id 分布变化）
- 执行 SHOW INDEX FROM product 显示 Cardinality 值很小（统计信息未更新）

【代码】
```java
@GetMapping("/list")
public Result<PageVO<ProductVO>> list(ProductQuery query) {
    PageHelper.startPage(query.getPage(), query.getSize());
    List<Product> products = productMapper.selectByCategory(
        query.getCategoryId());
    return Result.success(convertToVO(products));
}
```

【分析要求】
1. 根因分析（为什么昨天走索引今天不走？）
2. 紧急修复（5分钟内上线）
3. 长期优化方案
```

### AI 输出

```
═══════════════════════════════════════════
         性能突降诊断报告
═══════════════════════════════════════════

【根因：索引统计信息过期导致MySQL选错执行计划】

MySQL 优化器基于"索引统计信息"（Cardinality）决定是否使用索引。
数据量小的表，全表扫描可能比走索引更快（因为回表开销）。

昨天的 product 表：
  - category_id=1 的商品有 500 条（总表 2000 条）
  - MySQL 认为：走索引只需扫描 500 行 → 用索引 ✓

今天导入 50 万条后：
  - category_id=1 的商品有 10 万条（总表 52 万条）
  - 但统计信息未更新！MySQL 仍然认为只有 500 条
  - MySQL 的计划：扫描 500 行 vs 全表扫描 2000 行
  - 但实际上：扫描 10 万行 vs 全表扫描 52 万行
  - 由于统计信息过时，MySQL 选了全表扫描 ← 1800ms!

═══════════════════════════════════════════
        紧急修复（1分钟）
═══════════════════════════════════════════
*/

-- ① 立即更新统计信息
ANALYZE TABLE product;

-- ② 验证索引恢复
EXPLAIN SELECT * FROM product WHERE category_id=1 AND status=1 LIMIT 20;
-- 应该看到 key=idx_category_status, type=ref

/*
═══════════════════════════════════════════
        长期优化方案
═══════════════════════════════════════════
*/

-- ① 定期更新统计信息（MySQL 默认只在 10% 数据变化时更新）
SET GLOBAL innodb_stats_auto_recalc = ON;
SET GLOBAL innodb_stats_persistent = ON;

-- ② 在大批量导入后主动触发
-- 在数据导入脚本末尾加上：
ANALYZE TABLE product;

-- ③ 使用 FORCE INDEX 兜底（在关键业务SQL中）
@Select("SELECT /*+ FORCE_INDEX(idx_category_status) */ " +
        "* FROM product WHERE category_id=#{categoryId} AND status=1 " +
        "LIMIT #{limit} OFFSET #{offset}")
List<Product> selectByCategoryWithForce(@Param("categoryId") Long categoryId,
                                         @Param("limit") int limit,
                                         @Param("offset") int offset);

-- ④ 代码层加缓存（减少数据库依赖）
@Cacheable(value = "productList", key = "#query.cacheKey()",
    unless = "#result.isEmpty()")
public List<Product> selectByCategory(ProductQuery query) { ... }

-- ⑤ 监控告警
-- 配置 Druid SQL 慢查询告警：单条 SQL 超过 500ms 马上钉钉通知

/*
═══════════════════════════════════════════
          排查方法论总结
═══════════════════════════════════════════

生产环境性能问题排查优先级：
1. 先怀疑"变化"（数据量变化、流量变化、配置变化）
2. 再看"慢的地方"（SQL→缓存→RPC→CPU→GC）
3. 最后怀疑"代码"（没发版的情况下可能性最低）
*/
```

---

## 通用 Bug 排查 Prompt 模板

把这 7 个要素填进去，AI 就能给出高质量分析：

```
你是一个资深Java线上故障排查专家。请分析以下问题。

【问题类型】（选择：NPE / 并发 / 数据不一致 / 性能下降 / 死锁 / OOM / 其他）

【异常现象】
（用一句话描述：什么接口、什么时间、什么异常、影响范围）

【环境信息】
- JDK版本：
- 框架版本：
- 中间件（Redis/MQ/MySQL/ES等）：
- 服务器配置（CPU/内存）：

【异常堆栈/错误日志】（完整粘贴，不要截断）
[粘贴堆栈]

【相关代码】（出错的类/方法的前后30行）
[粘贴代码]

【时间线】
- 问题首次出现时间：
- 最近一次发布/配置变更时间：
- 是否有流量/DATA变化：

【补充信息】
（监控截图/Grafana面板/慢SQL日志/jstack/Arthas数据等）

【分析要求】
1. 定位根因（不是表象，是为什么发生）
2. 给出2-3个修复方案（紧急止血+长期根治）
3. 列出验证方法（如何确认修复有效）
4. 风险评估（修复可能引入的副作用）
```

---

## 总结

三个反直觉的认知：

1. **堆栈信息越"框架"，越应该喂给 AI。** 人类看 Spring AOP 的代理堆栈头疼，AI 不会——它一眼就能穿透框架层定位到你的业务代码。
2. **偶发 Bug 的排查不是靠"运气"，是靠"穷举可能性"。** AI 的优势是能同时列出 5 种可能性，你逐个验证，而不是凭直觉赌一个方向。
3. **Bug 排查的最高境界不是"查到根因"，而是"让 Bug 根本不可能发生"。** 让你的代码在编译阶段、Code Review 阶段、单测阶段就被拦下来——这才是 AI 辅助排查的终极价值。

**把堆栈喂给 AI，不是因为它比你聪明，而是因为它不会焦虑、不会烦躁、不会凌晨三点脑子转不动。**

---

## 下篇预告：Prompt 驱动代码国际化 i18n

Bug 排查是"救火"，下一篇我们聊"出海"——项目要国际化，200 个 Controller 里硬编码了中文提示信息，手动改 i18n 改到崩溃。一句 Prompt，200 个硬编码中文自动提取、自动生成 key、自动翻译成英日韩三语、自动重构代码。

**如果这篇文章对你有帮助，请点赞、收藏、转发，我们下篇见。**

---

*作者：一个凌晨三点被 NPE 叫醒无数次的老 Java 人*
