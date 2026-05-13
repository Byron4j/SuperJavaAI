# 用 Prompt 驱动 AI 进行代码重构：从过程式到函数式的蜕变，200行烂代码30秒焕新

> 你有没有见过那种一个方法300行，if-else嵌套5层的"祖传代码"？
> 重构它需要半天，测试它需要一天，上线它提心吊胆一整周。
> 今天，我用一句Prompt，30秒搞定。

---

## 开篇暴击：Before 🆚 After

先给你看一眼，什么叫"AI一句话重构"：

### 🔴 Before：一个300行的"屎山"方法

```java
public void processOrder(Order order) {
    // 参数校验（20行）
    if (order == null) {
        throw new RuntimeException("订单不能为空");
    }
    if (order.getUserId() == null) {
        throw new RuntimeException("用户ID不能为空");
    }
    if (order.getItems() == null || order.getItems().isEmpty()) {
        throw new RuntimeException("订单项不能为空");
    }
    if (order.getTotalAmount() == null || order.getTotalAmount().compareTo(BigDecimal.ZERO) <= 0) {
        throw new RuntimeException("订单金额无效");
    }
    // ... 还有10多个校验

    // 查询用户信息（30行）
    User user = userMapper.selectById(order.getUserId());
    if (user == null) {
        throw new RuntimeException("用户不存在");
    }
    if (user.getStatus() != 1) {
        throw new RuntimeException("用户状态异常");
    }
    // 查询用户等级、优惠信息...

    // 库存校验和锁定（50行）
    for (OrderItem item : order.getItems()) {
        Inventory inv = inventoryMapper.selectBySkuId(item.getSkuId());
        if (inv == null) {
            throw new RuntimeException("商品不存在:" + item.getSkuId());
        }
        if (inv.getStock() < item.getQuantity()) {
            throw new RuntimeException("库存不足:" + item.getSkuId());
        }
        inv.setStock(inv.getStock() - item.getQuantity());
        inventoryMapper.updateById(inv);
    }

    // 价格计算（40行）
    BigDecimal totalPrice = BigDecimal.ZERO;
    for (OrderItem item : order.getItems()) {
        if (item.getPrice() == null) {
            item.setPrice(BigDecimal.ZERO);
        }
        BigDecimal itemTotal = item.getPrice().multiply(new BigDecimal(item.getQuantity()));
        if (item.getDiscountType() == 1) {
            itemTotal = itemTotal.multiply(new BigDecimal("0.9"));
        } else if (item.getDiscountType() == 2) {
            itemTotal = itemTotal.multiply(new BigDecimal("0.8"));
        } else if (item.getDiscountType() == 3) {
            itemTotal = itemTotal.subtract(new BigDecimal("10"));
        }
        totalPrice = totalPrice.add(itemTotal);
    }

    // 优惠券校验（40行）
    if (order.getCouponId() != null) {
        Coupon coupon = couponMapper.selectById(order.getCouponId());
        if (coupon == null) {
            throw new RuntimeException("优惠券不存在");
        }
        if (coupon.getExpireTime().before(new Date())) {
            throw new RuntimeException("优惠券已过期");
        }
        if (coupon.getMinAmount().compareTo(totalPrice) > 0) {
            throw new RuntimeException("未达到优惠券使用门槛");
        }
        // 计算优惠金额
        if (coupon.getType() == 1) {
            totalPrice = totalPrice.subtract(coupon.getAmount());
        } else if (coupon.getType() == 2) {
            totalPrice = totalPrice.multiply(coupon.getDiscountRate());
        }
    }

    // 创建订单（20行）
    order.setStatus(1);
    order.setCreateTime(new Date());
    order.setRealAmount(totalPrice);
    orderMapper.insert(order);

    // 发送消息（30行）
    rabbitTemplate.convertAndSend("order.exchange", "order.create", order);
    // 更新用户积分...
    // 记录日志...
}
```

**看到这段代码，你是不是已经开始头疼了？**

### 🟢 After：AI重构后的代码

```java
@Service
@RequiredArgsConstructor
public class OrderProcessor {
    private final OrderValidator validator;
    private final UserChecker userChecker;
    private final InventoryService inventoryService;
    private final PriceCalculator priceCalculator;
    private final CouponApplier couponApplier;
    private final OrderRepository orderRepository;
    private final OrderEventPublisher eventPublisher;

    public void processOrder(Order order) {
        validator.validate(order);
        userChecker.check(order.getUserId());
        inventoryService.lock(order.getItems());
        order.setRealAmount(calculateFinalPrice(order));
        orderRepository.save(order);
        eventPublisher.publishCreated(order);
    }
}
```

**300行 → 7行主流程。每个步骤都是一个独立的、可测试的类。这就是"一句话重构"的力量。**

---

## Prompt的核心心法：5个重构场景全解析

### 重构1：长方法拆分——按职责，不按行数

❌ **错误Prompt**：
> 帮我把这个方法拆成几个小方法，每个不超过30行。

这种Prompt会让AI机械地按行数切分，切出来的方法毫无语义，甚至更难看。

✅ **正确Prompt**：

```
你是一个资深Java重构专家。请将以下方法进行重构：

【核心要求】
1. 按业务职责拆分，而不是按行数拆分
2. 每个新方法应该有且仅有一个明确的职责
3. 方法名要能清晰表达"做什么"，而非"怎么做"
4. 保持原有执行顺序不变
5. 不修改任何业务逻辑

【拆分维度】
- 参数校验 → 独立方法
- 外部资源查询 → 独立方法
- 核心业务计算 → 独立方法
- 数据持久化 → 独立方法
- 消息/事件发送 → 独立方法

【输出格式】
- 原始代码和重构后代码的对照
- 每个新方法占用行数和职责说明
- 单元测试必须全部通过

待重构代码：
[粘贴你的长方法]
```

**重构说明**：关键不在于"让方法变短"，而在于让每个方法都有能力回答这个问题——**"如果这个逻辑要变，是哪个需求驱动的？"** 改变的来源不同，方法就应该不同。

---

### 重构2：if-else地狱 → 策略模式

原始代码：

```java
public BigDecimal calculateDiscount(Order order) {
    if (order.getUserLevel() == 1) {          // 普通会员
        return order.getAmount().multiply(new BigDecimal("0.95"));
    } else if (order.getUserLevel() == 2) {   // 黄金会员
        return order.getAmount().multiply(new BigDecimal("0.85"));
    } else if (order.getUserLevel() == 3) {   // 钻石会员
        return order.getAmount().multiply(new BigDecimal("0.75"));
    } else if (order.getUserLevel() == 4) {   // 黑金会员
        BigDecimal discount = order.getAmount().multiply(new BigDecimal("0.7"));
        if (order.getAmount().compareTo(new BigDecimal("10000")) > 0) {
            discount = discount.subtract(new BigDecimal("100")); // 额外减免
        }
        return discount;
    }
    return order.getAmount();
}
```

✅ **Prompt**：

```
请对以下if-else分支代码进行"策略模式"重构：

【约束条件】
1. 识别所有条件分支，每个分支转化为一个独立的策略实现类
2. 使用Spring注入策略（Map<String, DiscountStrategy>），而非手动switch
3. 保留每个分支的业务逻辑完全不变（包括特殊处理，如level=4的额外减免）
4. 新增策略时只需新增一个类 + 添加枚举值，不修改原有代码（开闭原则）

【额外检查】
- 确认default分支（无匹配时的行为）被正确保留
- 确认边界值（level=0/null）的处理逻辑不变

待重构代码：
[粘贴你的if-else代码]
```

AI重构后代码：

```java
@Component
public class DiscountContext {
    private final Map<UserLevel, DiscountStrategy> strategyMap;

    public DiscountContext(List<DiscountStrategy> strategies) {
        this.strategyMap = strategies.stream()
            .collect(Collectors.toMap(DiscountStrategy::supportedLevel, Function.identity()));
    }

    public BigDecimal calculate(Order order) {
        DiscountStrategy strategy = strategyMap.getOrDefault(
            UserLevel.from(order.getUserLevel()), 
            NoDiscountStrategy.INSTANCE
        );
        return strategy.calculate(order);
    }
}

// 示例：钻石会员策略
@Component
public class DiamondDiscountStrategy implements DiscountStrategy {
    @Override
    public UserLevel supportedLevel() { return UserLevel.DIAMOND; }
    
    @Override
    public BigDecimal calculate(Order order) {
        return order.getAmount().multiply(new BigDecimal("0.75"));
    }
}
```

**重构说明**：不是所有if-else都该转策略模式。判断标准：**分支数 ≥ 3 且未来可能继续增加**，用策略模式；分支数固定且永远不变，用Map映射即可；if-else只有2个分支，用卫语句。

---

### 重构3：过程式循环 → Stream API

原始代码：

```java
public List<UserVO> getActiveUsersWithOrders(List<User> users, List<Order> orders) {
    List<UserVO> result = new ArrayList<>();
    Map<Long, List<Order>> userOrderMap = new HashMap<>();
    for (Order order : orders) {
        if (order.getStatus() == 1) {
            userOrderMap.computeIfAbsent(order.getUserId(), k -> new ArrayList<>()).add(order);
        }
    }
    for (User user : users) {
        if (user.getStatus() == 1 && user.getAge() >= 18) {
            List<Order> userOrders = userOrderMap.get(user.getId());
            if (userOrders != null && !userOrders.isEmpty()) {
                BigDecimal totalAmount = BigDecimal.ZERO;
                for (Order order : userOrders) {
                    totalAmount = totalAmount.add(order.getAmount());
                }
                if (totalAmount.compareTo(new BigDecimal("500")) > 0) {
                    UserVO vo = new UserVO();
                    vo.setId(user.getId());
                    vo.setName(user.getName());
                    vo.setTotalAmount(totalAmount);
                    vo.setOrderCount(userOrders.size());
                    result.add(vo);
                }
            }
        }
    }
    result.sort((a, b) -> b.getTotalAmount().compareTo(a.getTotalAmount()));
    return result;
}
```

✅ **Prompt**：

```
请将下列传统for循环逻辑重构为Java Stream API：

【核心要求】
1. 保持业务逻辑100%不变
2. Stream表达式的语义要能直观反映业务含义，不要为了用Stream而失去可读性
3. 每一步操作（过滤、映射、分组、排序）用注释标注业务含义
4. 保持原有排序规则和空值处理逻辑

【不适用Stream的场景】请保持原样
- 有检查型异常（checked exception）需要处理的循环
- 需要提前break/continue的循环（除非明确用takeWhile等替代）
- 存在外部副作用（如打印日志、更新缓存）的循环——这些保持显式for循环

待重构代码：
[粘贴你的for循环代码]
```

AI重构后代码：

```java
public List<UserVO> getActiveUsersWithOrders(List<User> users, List<Order> orders) {
    // 筛选有效订单并按用户分组
    Map<Long, List<Order>> userOrderMap = orders.stream()
        .filter(o -> o.getStatus() == 1)
        .collect(Collectors.groupingBy(Order::getUserId));

    return users.stream()
        .filter(u -> u.getStatus() == 1 && u.getAge() >= 18)
        .map(user -> buildUserVO(user, userOrderMap))
        .filter(Objects::nonNull)
        .filter(vo -> vo.getTotalAmount().compareTo(new BigDecimal("500")) > 0)
        .sorted((a, b) -> b.getTotalAmount().compareTo(a.getTotalAmount()))
        .collect(Collectors.toList());
}

private UserVO buildUserVO(User user, Map<Long, List<Order>> userOrderMap) {
    List<Order> userOrders = userOrderMap.get(user.getId());
    if (userOrders == null || userOrders.isEmpty()) return null;

    BigDecimal totalAmount = userOrders.stream()
        .map(Order::getAmount)
        .reduce(BigDecimal.ZERO, BigDecimal::add);

    UserVO vo = new UserVO();
    vo.setId(user.getId());
    vo.setName(user.getName());
    vo.setTotalAmount(totalAmount);
    vo.setOrderCount(userOrders.size());
    return vo;
}
```

**重构说明**：Stream不是银弹。我刻意在Prompt中列出"不适用的场景"，避免AI强行把有副作用的代码也Stream化。一个关键判断：**如果这段代码的核心心智负担是"控制流"（何时该continue、何时该break），保留显式循环比强制Stream更清晰。**

---

### 重构4：上帝类 → 职责分离

原始代码（典型的ServiceImpl"全能神类"）：

```java
@Service
public class OrderServiceImpl implements OrderService {
    @Autowired private OrderMapper orderMapper;
    @Autowired private RedisTemplate<String, Object> redisTemplate;
    @Autowired private UserClient userClient;
    @Autowired private InventoryClient inventoryClient;
    @Autowired private CouponClient couponClient;
    @Autowired private RabbitTemplate rabbitTemplate;

    @Override
    public OrderVO createOrder(CreateOrderRequest request) {
        // ===== 缓存校验（混在Service里） =====
        String cacheKey = "user:" + request.getUserId();
        UserDTO user = (UserDTO) redisTemplate.opsForValue().get(cacheKey);
        if (user == null) {
            user = userClient.getUser(request.getUserId());
            redisTemplate.opsForValue().set(cacheKey, user, 30, TimeUnit.MINUTES);
        }

        // ===== 数据转换（混在Service里） =====
        Order order = new Order();
        order.setUserId(request.getUserId());
        order.setUserName(user.getName());
        List<OrderItem> items = request.getItems().stream().map(i -> {
            OrderItem item = new OrderItem();
            item.setSkuId(i.getSkuId());
            item.setQuantity(i.getQuantity());
            return item;
        }).collect(Collectors.toList());
        order.setItems(items);

        // ===== 外部调用 =====
        inventoryClient.lock(request.getItems());
        BigDecimal price = couponClient.calculate(request.getCouponId(), items);

        // ===== 持久化 =====
        order.setRealAmount(price);
        orderMapper.insert(order);

        // ===== 消息发送 =====
        rabbitTemplate.convertAndSend("order.exchange", "order.create", order);

        // ===== 数据转换返回 =====
        OrderVO vo = new OrderVO();
        vo.setOrderId(order.getId());
        vo.setRealAmount(price);
        return vo;
    }
}
```

✅ **Prompt**：

```
请对以下Service类按照"单一职责原则"进行拆分重构：

【识别并分离这些职责】
- 缓存逻辑 → 独立CacheManager
- 外部RPC调用 → 独立Gateway/Client层（Facade封装）
- 数据转换/组装 → 独立Converter/Assembler
- 业务编排逻辑 → 保留在原Service中，只做流程编排
- 消息/事件 → 独立EventPublisher

【重构原则】
1. 每层只能与自己相邻的下层交互，不能跨层
2. 数据转换使用MapStruct，不要手写setter
3. 外部调用必须有自己的防腐层（反序列化、降级、重试）
4. Service层最终只保留"流程编排"——调用A、调用B、调用C这样

【输出】
- 新类清单和职责描述
- 类依赖关系图（文字版）
- 重构后的完整代码

待重构代码：
[粘贴你的Service类]
```

AI重构后结构：

```
OrderServiceImpl (业务编排，只做流程编排)
  ├── UserCacheManager (缓存管理：getOrFetch)
  ├── UserGateway (外部调用防腐层)
  │     └── UserClient (Feign客户端)
  ├── InventoryGateway (库存防腐层)
  ├── CouponGateway (优惠防腐层)
  ├── OrderAssembler (请求→领域对象转换，使用MapStruct)
  ├── OrderVOAssembler (领域对象→响应转换)
  ├── OrderRepository (持久化封装)
  └── OrderEventPublisher (领域事件发布)
```

**重构说明**：判断一个Service是不是上帝类，看它的`import`列表。如果import了 `RedisTemplate` + `RabbitTemplate` + 3个以上的 `XXXClient`，说明它已经是一个"全能神"了，该拆了。

---

### 重构5：重复代码 → 跨类抽取公共方法

✅ **Prompt**（这是AI的独特优势——跨文件分析）：

```
请在以下多个类中发现重复或高度相似的代码模式：

【分析的类】（按路径列出）
- com.example.service.OrderService.java
- com.example.service.PaymentService.java  
- com.example.service.RefundService.java

【识别重点】
1. 结构相似但变量名不同的代码块
2. 业务逻辑相同但实现细节有差异的方法
3. 参数校验、异常处理、日志记录等横切关注点的重复

【抽取要求】
- 只抽取逻辑100%相同的代码
- 相似但不相同的代码，列出差异点由我判断
- 抽取到util/helper/common包下
- 保持原有调用方代码不受影响

【输出格式】
- 重复代码块位置（类名+行号）
- 重复度百分比
- 建议的公共方法签名和位置
```

**重构说明**：这是AI真正比人类强的地方。人类很难跨10个以上的文件发现重复模式，但AI可以。"重复不一定该抽取"——如果两段代码恰好相同但变化原因不同（属于不同业务域），让它们保持独立。抽取的标准不是"长得像"，而是"变的原因相同"。

---

## 重构安全指南：如何保证改完之后不出事故？

### 1. 保证行为不变的核心技巧

在Prompt中必须要求AI做以下输出：

```java
// AI输出：行为等价性证明

【输入输出等价性】
- 输入空间：所有边界值已覆盖（null, 空集合, 零值, 负数）
- 输出一致性：重构前后的同一组输入产生完全相同的结果
- 副作用一致性：数据库操作、消息发送的顺序和内容不变

【差异清单】
- ✅ 无行为变化
- ⚠️ 性能变化：Stream API在大数据集上可能比for循环慢5%（可接受）
- ✅ 异常类型不变：仍抛出原业务异常
```

### 2. 用单元测试验证重构正确性

✅ **验证Prompt**：

```
你已完成重构，现在请执行自动化验证：

1. 运行项目中所有`*Test.java`，确认全部通过
2. 如有测试失败，请逐个分析失败原因并修复（不能绕过测试）
3. 如无现有测试，请生成重构前后行为一致性的验证方案：
   - 构造相同的输入用例（边界值+正常值+异常值）
   - 分别调用重构前后的方法
   - 用AssertJ对比结果（值+异常+副作用）

报告格式：
- 总测试数：X
- 通过：X
- 失败：0
- 新增验证用例：N个
```

### 3. 渐进式重构 vs 大爆炸

| 策略 | Prompt关键词 | 适用场景 |
|------|-------------|---------|
| 渐进式重构 | "请先重构提取XX方法，验证通过后，再进行下一步" | 核心业务链、缺测试 |
| 大爆炸重构 | "一次性完成全部重构" | 工具类、有完整测试覆盖 |
| 平行重构 | "请在旁边新建一个类实现相同逻辑，旧类标记@Deprecated" | 对外暴露的API |

**推荐策略**：能用渐进式的坚决不用大爆炸。让AI生成重构步骤列表，每一步独立执行、独立验证、独立提交。万一某一步引入bug，你只需要回滚最后一步，而不是全部重来。

---

## 重构Prompt的核心口诀

### 🔥 最重要的一条："只重构不改功能"

AI有个坏习惯——看到"不够好"的代码会忍不住帮你"优化业务逻辑"。比如：
- 把`BigDecimal`的`.equals()`改成`.compareTo() == 0` ← 这其实是改对了但没告诉你
- 把catch里的`e.printStackTrace()`改成log ← 虽然更规范，但也是行为变化
- 把`new Date()`改成`LocalDateTime.now()` ← 运行时行为不同了

**如何阻止？在Prompt开头强化这一句：**

```
【铁律】本次任务只重构代码结构（方法拆分、类提取、模式应用），
严格禁止任何业务逻辑修改。
包括但不限于：
- 不修改任何条件判断的边界值
- 不修改异常类型和异常消息
- 不修改日志级别和日志内容
- 不修改任何引用的第三方库版本或API
- 不修改数值计算的精度和舍入方式
- 不修改序列化/反序列化格式

如有任何无法避免的行为变化，必须在输出中用 【⚠️ 差异告警】 明确标出。
```

### 重构Prompt模板（通用版）

```
你是一个资深Java重构专家。请重构以下代码。

【重构目标】（选填）
- 长方法拆分 / if-else转策略模式 / 过程式转Stream / 上帝类分离 / 去重提取

【铁律】只重构代码结构，严格禁止修改任何业务逻辑。
具体禁止项：[粘贴上面的禁止清单]

【保留项】
- 所有异常类型和消息
- 所有日志输出
- 事务边界
- 数据库操作顺序

【输出要求】
- Before/After完整代码
- 行为等价性说明（差异清单）
- 涉及的测试用例清单

待重构代码：
[粘贴代码]
```

---

## 总结：三个"不是"

1. **重构不是重写** —— 重构的目标是让代码更好维护，不是让你"重新实现一次"。如果你的Prompt让AI重写了一整个类，那你做的是重写，不是重构。

2. **重构不是修Bug** —— 如果你发现了一个Bug，先修Bug再重构，或者至少分两个独立提交。永远不要在重构中顺便修Bug，否则没有人知道到底是重构引入的新Bug还是老Bug被修掉了。

3. **重构不是目的** —— 重构是为了应对下一个需求变更。如果一段代码未来不会变，重构它就是浪费。**为即将到来的变化做准备的重构，才是值得的重构。**

---

## 下篇预告：Prompt 驱动设计模式

重构是"治病"，设计模式是"养生"。下一篇文章，我们将探讨如何让AI帮你从零设计一套"开箱即用"的策略模式 + 责任链 + 观察者的组合架构。

关键词：工厂方法、抽象工厂、建造者模式、适配器模式、装饰器模式……Prompt一条龙生成。

**如果这篇文章对你有帮助，请点赞、收藏、转发，我们下篇见。**

---

*作者：一个用AI把重构效率提升10倍的Java老码农*
