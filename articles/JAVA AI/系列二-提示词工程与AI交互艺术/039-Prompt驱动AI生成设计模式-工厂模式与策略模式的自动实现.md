# Prompt 驱动 AI 生成设计模式：工厂模式与策略模式的自动实现，再也不用手敲模板代码

> 设计模式是解决特定问题的标准方案，但手写设计模式的样板代码让人头疼——工厂模式每次都要写 Factory 类 + Product 接口 + ConcreteProduct 类，策略模式要写 Context + Strategy 接口 + 多个 StrategyImpl，一个模式下来至少 5 个文件。今天，我把这些"标准动作"全部交给 AI，一句 Prompt 自动生成全套代码。

---

## 开篇：设计模式的两大痛点

干了十年 Java，设计模式的书翻烂了，GoF 23 种模式倒背如流，但每次真正要写的时候，依然有两个坎：

**痛点一：样板代码太多。** 工厂模式光骨架就要 4-5 个文件，每个文件还得写 import、注解、注释。真正有业务价值的逻辑可能就 10 行，但为了这 10 行，你要写 100 行模板代码。

**痛点二：选型纠结。** "这个场景到底用策略模式还是状态模式？""责任链要不要结合工厂？""建造者模式和工厂模式能不能一起用？"——每次都要翻书、查博客、纠结半小时。

我的解决方案：**把这两个痛点全部扔给 AI。** 你只需要描述场景，AI 自己选模式、自己起类名、自己写代码。下面给你看 4 个实战案例。

---

## 实战一：工厂模式——多支付渠道的创建

### 场景描述

电商系统需要支持支付宝、微信、银行卡三种支付方式。前端传一个 `payType` 参数（ALIPAY / WECHAT / BANK_CARD），后端根据类型创建对应的支付处理器，执行支付逻辑。每种支付的调用方式完全不同（支付宝走 SDK、微信走 API V3、银行卡走银联）。

### Prompt

```
你是一个资深Java架构师。请为以下场景设计并实现代码：

【业务场景】
一个电商支付系统，需要根据支付类型（ALIPAY/WECHAT/BANK_CARD）动态创建对应的支付处理器。
每种支付方式的实现差异很大：
- 支付宝：需要调用AlipayClient SDK，参数包括appId、私钥、公钥
- 微信：需要调用WechatPayClient V3 API，参数包括商户号、APIv3密钥、证书序列号
- 银行卡：需要调用UnionPay SDK，参数包括商户号、证书路径、密码

【设计要求】
1. 使用合适的创建型设计模式，保证新增支付方式时不需要修改已有代码
2. 支付处理器需要有统一的行为约定（支付、退款、查询）
3. 使用Spring管理Bean，通过依赖注入获取所有支付处理器
4. 客户端代码调用时只需要一行代码，根据类型获取处理器即可
5. 类名和方法名要体现业务语义，不要出现Factory/Strategy这类模式名

【输出要求】
- 给出完整的Java代码，包含所有需要的类/接口/枚举
- 说明为什么选择这个设计模式
- 新增一种支付方式（如京东支付）的步骤说明
```

### AI 生成的完整代码

```java
// ============ 支付类型枚举 ============
public enum PaymentChannel {
    ALIPAY("支付宝"),
    WECHAT("微信支付"),
    BANK_CARD("银行卡");

    private final String displayName;

    PaymentChannel(String displayName) {
        this.displayName = displayName;
    }

    public String getDisplayName() {
        return displayName;
    }
}

// ============ 支付结果 ============
@Data
@Builder
public class PaymentResult {
    private boolean success;
    private String transactionId;
    private String channelOrderNo;
    private BigDecimal amount;
    private String errorMessage;
}

// ============ 支付请求 ============
@Data
@Builder
public class PaymentRequest {
    private String orderNo;
    private BigDecimal amount;
    private String subject;
    private Map<String, Object> extraParams;
}

// ============ 统一支付契约（接口） ============
public interface PaymentProcessor {

    PaymentChannel supportedChannel();

    PaymentResult pay(PaymentRequest request);

    PaymentResult refund(String transactionId, BigDecimal amount);

    PaymentResult queryByOrderNo(String orderNo);
}

// ============ 支付宝处理器 ============
@Component
public class AlipayProcessor implements PaymentProcessor {

    private final AlipayClient alipayClient;

    public AlipayProcessor(@Value("${alipay.app-id}") String appId,
                           @Value("${alipay.private-key}") String privateKey,
                           @Value("${alipay.public-key}") String alipayPublicKey) {
        this.alipayClient = new DefaultAlipayClient(
            "https://openapi.alipay.com/gateway.do",
            appId, privateKey, "json", "UTF-8", alipayPublicKey, "RSA2"
        );
    }

    @Override
    public PaymentChannel supportedChannel() {
        return PaymentChannel.ALIPAY;
    }

    @Override
    public PaymentResult pay(PaymentRequest request) {
        AlipayTradePayRequest alipayRequest = new AlipayTradePayRequest();
        alipayRequest.setBizContent(JSON.toJSONString(Map.of(
            "out_trade_no", request.getOrderNo(),
            "total_amount", request.getAmount().toString(),
            "subject", request.getSubject()
        )));
        AlipayTradePayResponse response = alipayClient.execute(alipayRequest);
        return PaymentResult.builder()
            .success("10000".equals(response.getCode()))
            .transactionId(response.getTradeNo())
            .channelOrderNo(response.getOutTradeNo())
            .amount(request.getAmount())
            .build();
    }

    @Override
    public PaymentResult refund(String transactionId, BigDecimal amount) {
        // 支付宝退款逻辑
        return PaymentResult.builder().success(true).build();
    }

    @Override
    public PaymentResult queryByOrderNo(String orderNo) {
        // 支付宝查询逻辑
        return PaymentResult.builder().success(true).build();
    }
}

// ============ 微信支付处理器 ============
@Component
public class WechatPayProcessor implements PaymentProcessor {

    private final WechatPayClient wechatPayClient;

    public WechatPayProcessor(@Value("${wechat.merchant-id}") String merchantId,
                              @Value("${wechat.api-v3-key}") String apiV3Key,
                              @Value("${wechat.cert-serial-no}") String certSerialNo) {
        this.wechatPayClient = new WechatPayClient.Builder()
            .merchantId(merchantId)
            .apiV3Key(apiV3Key)
            .certSerialNo(certSerialNo)
            .build();
    }

    @Override
    public PaymentChannel supportedChannel() {
        return PaymentChannel.WECHAT;
    }

    @Override
    public PaymentResult pay(PaymentRequest request) {
        // 微信支付V3 API调用逻辑
        WechatPayRequest wechatRequest = WechatPayRequest.builder()
            .outTradeNo(request.getOrderNo())
            .amount(Amount.builder().total(request.getAmount().intValue()).build())
            .description(request.getSubject())
            .build();
        WechatPayResponse response = wechatPayClient.pay(wechatRequest);
        return PaymentResult.builder()
            .success(true)
            .transactionId(response.getPrepayId())
            .build();
    }

    @Override
    public PaymentResult refund(String transactionId, BigDecimal amount) {
        return PaymentResult.builder().success(true).build();
    }

    @Override
    public PaymentResult queryByOrderNo(String orderNo) {
        return PaymentResult.builder().success(true).build();
    }
}

// ============ 银行卡处理器 ============
@Component
public class BankCardPayProcessor implements PaymentProcessor {

    private final UnionPayClient unionPayClient;

    public BankCardPayProcessor(@Value("${unionpay.merchant-id}") String merchantId,
                                @Value("${unionpay.cert-path}") String certPath,
                                @Value("${unionpay.password}") String password) {
        this.unionPayClient = new UnionPayClient(merchantId, certPath, password);
    }

    @Override
    public PaymentChannel supportedChannel() {
        return PaymentChannel.BANK_CARD;
    }

    @Override
    public PaymentResult pay(PaymentRequest request) {
        return PaymentResult.builder().success(true).build();
    }

    @Override
    public PaymentResult refund(String transactionId, BigDecimal amount) {
        return PaymentResult.builder().success(true).build();
    }

    @Override
    public PaymentResult queryByOrderNo(String orderNo) {
        return PaymentResult.builder().success(true).build();
    }
}

// ============ 支付路由器（自动发现所有处理器） ============
@Component
public class PaymentRouter {

    private final Map<PaymentChannel, PaymentProcessor> processorMap;

    public PaymentRouter(List<PaymentProcessor> processors) {
        this.processorMap = processors.stream()
            .collect(Collectors.toMap(
                PaymentProcessor::supportedChannel,
                Function.identity()
            ));
    }

    public PaymentProcessor getProcessor(PaymentChannel channel) {
        PaymentProcessor processor = processorMap.get(channel);
        if (processor == null) {
            throw new IllegalArgumentException("不支持的支付渠道: " + channel);
        }
        return processor;
    }

    public PaymentResult pay(String channelCode, PaymentRequest request) {
        PaymentChannel channel = PaymentChannel.valueOf(channelCode);
        log.info("开始支付: 渠道={}, 订单号={}", channel.getDisplayName(), request.getOrderNo());
        return getProcessor(channel).pay(request);
    }
}

// ============ 控制器（客户端调用） ============
@RestController
@RequestMapping("/api/payment")
public class PaymentController {

    @Autowired
    private PaymentRouter paymentRouter;

    @PostMapping("/pay")
    public Result<PaymentResult> pay(@RequestBody PayRequestDTO dto) {
        PaymentRequest request = PaymentRequest.builder()
            .orderNo(dto.getOrderNo())
            .amount(dto.getAmount())
            .subject(dto.getSubject())
            .build();
        PaymentResult result = paymentRouter.pay(dto.getChannelCode(), request);
        return Result.success(result);
    }
}
```

### 模式说明

AI 自动选择了**工厂方法模式的 Spring 变体**：核心思想是用 Spring 的 `List<PaymentProcessor>` 自动注入所有实现类，然后根据 `supportedChannel()` 建立映射表。这不是传统的"new ConcreteProduct()"式工厂，而是更现代的"注册表模式"，好处是：

1. **新增渠道只需加一个 `@Component` 类**，不改任何已有代码（开闭原则）
2. **Spring 自动管理生命周期**，不需要手动注册
3. **客户端调用只用一行代码** `paymentRouter.pay(channel, request)`

> **技巧**：注意 Prompt 中我说"不要出现 Factory/Strategy 这类模式名"，AI 自己取了 `PaymentRouter` 这个名字，比 `PaymentFactory` 更贴近业务。

---

## 实战二：策略模式——多级会员折扣计算

### 场景描述

电商平台的会员体系有四级：普通会员（无折扣）、银卡（95折）、金卡（88折）、钻石（8折 + 满 1000 减 50）。每种折扣计算逻辑差异较大，未来可能新增铂金会员。要求新增会员等级时不需要修改已有的折扣计算代码。

### Prompt

```
请为以下业务场景设计代码：

【业务场景】
会员折扣系统，支持4种会员等级，每种享有不同的折扣政策：
- NORMAL：无折扣
- SILVER：整单95折
- GOLD：整单88折
- DIAMOND：整单8折，且订单金额满1000额外再减50

每种会员的折扣计算逻辑差异较大，未来会新增PLATINUM会员（85折+满2000减100）。

【设计要求】
1. 使用行为型设计模式，让每种折扣算法独立封装
2. 新增会员等级时不修改已有代码
3. 当传入的会员等级不存在时，默认按普通会员处理（无折扣）
4. 未来可能会支持按比例折扣、按固定金额减免、阶梯折扣三种计算方式的组合
5. 不要让调用方知道任何实现细节

【输出】
- 完整Java代码（所有类）
- 新增PLATINUM会员的示例代码（证明开闭原则成立）
```

### AI 生成的完整代码

```java
// ============ 会员等级枚举 ============
public enum MemberLevel {
    NORMAL("普通会员"),
    SILVER("银卡会员"),
    GOLD("金卡会员"),
    DIAMOND("钻石会员");

    private final String displayName;

    MemberLevel(String displayName) {
        this.displayName = displayName;
    }

    public String getDisplayName() {
        return displayName;
    }
}

// ============ 折扣规则接口 ============
public interface DiscountRule {

    MemberLevel supportedLevel();

    BigDecimal applyDiscount(OrderContext context);
}

// ============ 订单上下文 ============
@Data
@Builder
public class OrderContext {
    private BigDecimal originalAmount;
    private List<OrderItem> items;
    private Long userId;
    private MemberLevel memberLevel;
}

// ============ 普通会员（无折扣） ============
@Component
public class NormalDiscountRule implements DiscountRule {

    @Override
    public MemberLevel supportedLevel() {
        return MemberLevel.NORMAL;
    }

    @Override
    public BigDecimal applyDiscount(OrderContext context) {
        return context.getOriginalAmount();
    }
}

// ============ 银卡会员（95折） ============
@Component
public class SilverDiscountRule implements DiscountRule {

    private static final BigDecimal DISCOUNT_RATE = new BigDecimal("0.95");

    @Override
    public MemberLevel supportedLevel() {
        return MemberLevel.SILVER;
    }

    @Override
    public BigDecimal applyDiscount(OrderContext context) {
        return context.getOriginalAmount()
            .multiply(DISCOUNT_RATE)
            .setScale(2, RoundingMode.HALF_UP);
    }
}

// ============ 金卡会员（88折） ============
@Component
public class GoldDiscountRule implements DiscountRule {

    private static final BigDecimal DISCOUNT_RATE = new BigDecimal("0.88");

    @Override
    public MemberLevel supportedLevel() {
        return MemberLevel.GOLD;
    }

    @Override
    public BigDecimal applyDiscount(OrderContext context) {
        return context.getOriginalAmount()
            .multiply(DISCOUNT_RATE)
            .setScale(2, RoundingMode.HALF_UP);
    }
}

// ============ 钻石会员（8折 + 满1000减50） ============
@Component
public class DiamondDiscountRule implements DiscountRule {

    private static final BigDecimal DISCOUNT_RATE = new BigDecimal("0.80");
    private static final BigDecimal THRESHOLD = new BigDecimal("1000");
    private static final BigDecimal EXTRA_DEDUCTION = new BigDecimal("50");

    @Override
    public MemberLevel supportedLevel() {
        return MemberLevel.DIAMOND;
    }

    @Override
    public BigDecimal applyDiscount(OrderContext context) {
        BigDecimal afterRate = context.getOriginalAmount()
            .multiply(DISCOUNT_RATE);
        if (context.getOriginalAmount().compareTo(THRESHOLD) >= 0) {
            afterRate = afterRate.subtract(EXTRA_DEDUCTION);
        }
        return afterRate.max(BigDecimal.ZERO)
            .setScale(2, RoundingMode.HALF_UP);
    }
}

// ============ 折扣计算引擎（自动装配所有规则） ============
@Component
public class DiscountEngine {

    private final Map<MemberLevel, DiscountRule> ruleMap;

    public DiscountEngine(List<DiscountRule> rules) {
        this.ruleMap = rules.stream()
            .collect(Collectors.toMap(
                DiscountRule::supportedLevel,
                Function.identity()
            ));
    }

    public BigDecimal calculate(OrderContext context) {
        DiscountRule rule = ruleMap.getOrDefault(
            context.getMemberLevel(),
            ruleMap.get(MemberLevel.NORMAL)
        );
        log.info("会员等级: {}, 原价: {}, 折后: {}",
            context.getMemberLevel().getDisplayName(),
            context.getOriginalAmount(),
            rule.applyDiscount(context));
        return rule.applyDiscount(context);
    }
}

// ============ Service层调用 ============
@Service
public class OrderService {

    @Autowired
    private DiscountEngine discountEngine;

    @Autowired
    private MemberService memberService;

    public BigDecimal calculateFinalPrice(CreateOrderRequest request) {
        MemberLevel level = memberService.getMemberLevel(request.getUserId());
        OrderContext context = OrderContext.builder()
            .originalAmount(request.getTotalAmount())
            .memberLevel(level)
            .userId(request.getUserId())
            .build();
        return discountEngine.calculate(context);
    }
}

// ============ 新增铂金会员（证明开闭原则） ============
// 只需新增这一个类，不需要修改任何已有代码
@Component
public class PlatinumDiscountRule implements DiscountRule {

    private static final BigDecimal DISCOUNT_RATE = new BigDecimal("0.85");
    private static final BigDecimal THRESHOLD = new BigDecimal("2000");
    private static final BigDecimal EXTRA_DEDUCTION = new BigDecimal("100");

    @Override
    public MemberLevel supportedLevel() {
        return MemberLevel.PLATINUM;
    }

    @Override
    public BigDecimal applyDiscount(OrderContext context) {
        BigDecimal afterRate = context.getOriginalAmount()
            .multiply(DISCOUNT_RATE);
        if (context.getOriginalAmount().compareTo(THRESHOLD) >= 0) {
            afterRate = afterRate.subtract(EXTRA_DEDUCTION);
        }
        return afterRate.max(BigDecimal.ZERO)
            .setScale(2, RoundingMode.HALF_UP);
    }
}
```

### 模式说明

AI 再次用到了同样的 Spring 自动注入套路——但这次是**策略模式**的核心思想。每个 `DiscountRule` 实现就是一个策略，`DiscountEngine` 扮演 Context 角色。关键点：

1. **`ruleMap.getOrDefault()` 保证兜底逻辑**——未匹配的等级走 NORMAL，不会 NPE
2. **每种折扣规则只知道自己份内的事**——银卡只管 `×0.95`，钻石管 `×0.80` + 满减，互不干扰
3. **测试时可以独立测试每个规则**，不需要构造复杂环境

> **技巧**：Prompt 中我特意说了"不要出现 Strategy 这个类名"，AI 取了 `DiscountRule` + `DiscountEngine`，语意更自然。

---

## 实战三：观察者模式——订单状态变更的通知链

### 场景描述

订单状态从"待支付"变为"已支付"时，需要触发一系列动作：①发送短信通知用户 ②扣减库存 ③发放积分 ④记录审计日志 ⑤通知物流系统。每个"动作"之间互相独立，未来可能新增动作（如推送优惠券、发送邮件等）。

### Prompt

```
请为以下场景设计代码：

【业务场景】
订单状态变更时，需要触发多个独立的后续动作。当前需要：
- 发送短信通知用户
- 扣减对应商品的库存
- 发放会员积分（金额的1%）
- 记录操作审计日志
- 通知物流系统准备发货

每个动作之间完全独立，互不影响。未来可能新增推送优惠券、发送邮件、更新用户画像等动作。

【设计要求】
1. 使用合适的行为型设计模式，实现发布-订阅的解耦
2. 某个订阅者执行失败不影响其他订阅者的执行
3. 支持异步执行和同步执行两种模式
4. 新增订阅者时不需要修改订单状态变更的核心代码
5. 每个订阅者能明确表达自己监听的"状态变更类型"（如只监听"已支付"，不监听"已取消"）

【输出】
- 完整Java代码
- 新增一个EmailNotifier的步骤
```

### AI 生成的完整代码

```java
// ============ 订单状态枚举 ============
public enum OrderStatus {
    CREATED("待支付"),
    PAID("已支付"),
    SHIPPED("已发货"),
    COMPLETED("已完成"),
    CANCELLED("已取消");
    // ...
}

// ============ 订单状态变更事件 ============
@Getter
@ToString
public class OrderStatusChangedEvent extends ApplicationEvent {

    private final Long orderId;
    private final OrderStatus fromStatus;
    private final OrderStatus toStatus;
    private final Order order;

    public OrderStatusChangedEvent(Object source, Long orderId,
                                   OrderStatus fromStatus,
                                   OrderStatus toStatus,
                                   Order order) {
        super(source);
        this.orderId = orderId;
        this.fromStatus = fromStatus;
        this.toStatus = toStatus;
        this.order = order;
    }
}

// ============ 订阅者统一契约 ============
public interface OrderEventListener {

    /**
     * 感兴趣的状态变更
     */
    OrderStatus interestedIn();

    /**
     * 处理状态变更事件。返回false时不会阻塞其他监听者
     */
    void onStatusChanged(OrderStatusChangedEvent event);
}

// ============ 短信通知订阅者 ============
@Component
@Slf4j
public class SmsNotifier implements OrderEventListener {

    @Autowired
    private SmsService smsService;

    @Override
    public OrderStatus interestedIn() {
        return OrderStatus.PAID;
    }

    @Override
    @Async
    public void onStatusChanged(OrderStatusChangedEvent event) {
        try {
            String phone = event.getOrder().getUserPhone();
            String content = buildSmsContent(event);
            smsService.send(phone, content);
            log.info("支付成功短信已发送: orderId={}", event.getOrderId());
        } catch (Exception e) {
            log.error("短信发送失败: orderId={}", event.getOrderId(), e);
            // 不抛出异常，不影响其他监听者
        }
    }

    private String buildSmsContent(OrderStatusChangedEvent event) {
        return String.format("您的订单%s已支付成功，金额¥%s，我们将尽快为您发货。",
            event.getOrderId(), event.getOrder().getRealAmount());
    }
}

// ============ 库存扣减订阅者 ============
@Component
@Slf4j
public class InventoryDeductionListener implements OrderEventListener {

    @Autowired
    private InventoryService inventoryService;

    @Override
    public OrderStatus interestedIn() {
        return OrderStatus.PAID;
    }

    @Override
    @Async
    public void onStatusChanged(OrderStatusChangedEvent event) {
        event.getOrder().getItems().forEach(item -> {
            try {
                inventoryService.deduct(item.getSkuId(), item.getQuantity());
                log.info("库存扣减成功: sku={}, qty={}", item.getSkuId(), item.getQuantity());
            } catch (Exception e) {
                log.error("库存扣减失败，需要人工处理: sku={}", item.getSkuId(), e);
            }
        });
    }
}

// ============ 积分发放订阅者 ============
@Component
@Slf4j
public class PointsRewardListener implements OrderEventListener {

    private static final BigDecimal POINTS_RATE = new BigDecimal("0.01");

    @Autowired
    private PointsService pointsService;

    @Override
    public OrderStatus interestedIn() {
        return OrderStatus.COMPLETED;  // 确认收货后才发积分
    }

    @Override
    @Async
    public void onStatusChanged(OrderStatusChangedEvent event) {
        int points = event.getOrder().getRealAmount()
            .multiply(POINTS_RATE)
            .intValue();
        pointsService.addPoints(event.getOrder().getUserId(), points);
        log.info("积分发放: userId={}, points={}", event.getOrder().getUserId(), points);
    }
}

// ============ 审计日志订阅者（同步执行） ============
@Component
@Slf4j
public class AuditLogListener implements OrderEventListener {

    @Autowired
    private AuditLogMapper auditLogMapper;

    @Override
    public OrderStatus interestedIn() {
        return null;  // 对所有状态变更都感兴趣
    }

    @Override
    public void onStatusChanged(OrderStatusChangedEvent event) {
        AuditLog logEntry = AuditLog.builder()
            .orderId(event.getOrderId())
            .fromStatus(event.getFromStatus().name())
            .toStatus(event.getToStatus().name())
            .operateTime(LocalDateTime.now())
            .build();
        auditLogMapper.insert(logEntry);
    }
}

// ============ 事件分发器（核心引擎） ============
@Component
@Slf4j
public class OrderEventDispatcher {

    private final List<OrderEventListener> listeners;

    public OrderEventDispatcher(List<OrderEventListener> listeners) {
        this.listeners = listeners;
    }

    public void dispatch(OrderStatusChangedEvent event) {
        // 发布Spring事件（供 @EventListener 使用）
        applicationEventPublisher.publishEvent(event);

        // 遍历所有对当前目标状态感兴趣的监听者
        listeners.stream()
            .filter(l -> l.interestedIn() == null
                      || l.interestedIn() == event.getToStatus())
            .forEach(l -> {
                try {
                    l.onStatusChanged(event);
                } catch (Exception e) {
                    log.error("订单事件监听者执行异常: listener={}, orderId={}",
                        l.getClass().getSimpleName(), event.getOrderId(), e);
                    // 🔥 关键：吞掉异常，不影响其他监听者
                }
            });
    }
}

// ============ 订单Service（触发点） ============
@Service
public class OrderService {

    @Autowired
    private OrderMapper orderMapper;

    @Autowired
    private OrderEventDispatcher dispatcher;

    @Transactional
    public void paySuccess(Long orderId) {
        Order order = orderMapper.selectById(orderId);
        OrderStatus fromStatus = order.getStatus();

        order.setStatus(OrderStatus.PAID);
        order.setPaidTime(LocalDateTime.now());
        orderMapper.updateById(order);

        dispatcher.dispatch(new OrderStatusChangedEvent(
            this, orderId, fromStatus, OrderStatus.PAID, order));
    }
}
```

### 模式说明

AI 设计了一套**观察者模式 + Spring 事件机制**的混合方案：

1. **`OrderEventListener` 接口**——观察者模式的核心契约，每个订阅者实现此接口
2. **`interestedIn()` 方法**——订阅者自己声明关注的"状态类型"，不需要硬编码在事件分发器中
3. **@Async 异步执行**——短信、库存这些非关键动作异步执行，不阻塞主流程
4. **try-catch 兜底**——每个订阅者自己处理异常，一个挂了不影响其他人
5. **双向发布**——既走 Spring 原生 `ApplicationEventPublisher`（兼容 `@EventListener`），又走自定义分发器（有 `interestedIn` 过滤）

---

## 实战四：责任链模式——请求校验管道

### 场景描述

用户提交订单请求后，需要经过一系列校验：①参数基础校验（非空、格式）②用户黑名单检查 ③风控检查（是否异常下单行为）④库存校验 ⑤优惠券合法性校验。每个校验环节要么"通过并交给下一个"，要么"拒绝并终止整个流程"。

### Prompt

```
请为以下场景设计代码：

【业务场景】
订单提交流程需要经过多层验证，每一层验证失败都直接返回错误原因，不再执行后续校验：
①基础参数校验：订单金额>0、商品列表非空、用户ID非空
②用户风控拦截：检查用户是否在黑名单、是否异常频率下单（1分钟内超过3单）
③库存预校验：每种商品库存是否充足
④优惠券校验：优惠券是否存在、是否过期、是否达到使用门槛

每层校验是一个独立组件，未来可能新增"收货地址校验""发票信息校验"等。

【设计要求】
1. 使用行为型设计模式，让校验链可动态组合
2. 每一层校验失败立即返回错误信息，不执行后续校验
3. 支持通过配置决定校验开关（如风控校验可在配置中心动态关闭）
4. 校验顺序可通过配置调整

【输出】
- 完整Java代码
- 说明如何新增一个校验器
- 说明如何调整校验顺序
```

### AI 生成的完整代码

```java
// ============ 校验结果 ============
@Data
@Builder
public class ValidationResult {
    private boolean passed;
    private String errorCode;
    private String errorMsg;

    public static ValidationResult ok() {
        return ValidationResult.builder().passed(true).build();
    }

    public static ValidationResult fail(String errorCode, String errorMsg) {
        return ValidationResult.builder()
            .passed(false)
            .errorCode(errorCode)
            .errorMsg(errorMsg)
            .build();
    }
}

// ============ 订单提交上下文 ============
@Data
@Builder
public class SubmitOrderContext {
    private Long userId;
    private List<OrderItem> items;
    private BigDecimal totalAmount;
    private String couponId;
    private String addressId;
    private LocalDateTime submitTime;
    private String clientIp;
}

// ============ 校验处理器（责任链节点） ============
public abstract class OrderSubmitValidator {

    protected OrderSubmitValidator next;

    /**
     * 设置下一个校验节点（链式调用）
     */
    public OrderSubmitValidator setNext(OrderSubmitValidator next) {
        this.next = next;
        return next;  // 返回next以支持链式构建
    }

    /**
     * 执行校验。具体校验逻辑由子类实现。
     */
    public ValidationResult validate(SubmitOrderContext context) {
        ValidationResult result = doValidate(context);
        if (!result.isPassed()) {
            return result;  // 校验失败，终止链路
        }
        if (next != null) {
            return next.validate(context);
        }
        return ValidationResult.ok();  // 链路终点，全部通过
    }

    /**
     * 子类实现具体校验逻辑
     */
    protected abstract ValidationResult doValidate(SubmitOrderContext context);
}

// ============ ① 基础参数校验 ============
@Component
@Order(1)
public class BasicParamsValidator extends OrderSubmitValidator {

    @Override
    protected ValidationResult doValidate(SubmitOrderContext context) {
        if (context.getUserId() == null) {
            return ValidationResult.fail("PARAM_USER_ID_NULL", "用户ID不能为空");
        }
        if (context.getItems() == null || context.getItems().isEmpty()) {
            return ValidationResult.fail("PARAM_ITEMS_EMPTY", "订单商品不能为空");
        }
        if (context.getTotalAmount() == null
            || context.getTotalAmount().compareTo(BigDecimal.ZERO) <= 0) {
            return ValidationResult.fail("PARAM_AMOUNT_INVALID", "订单金额必须大于0");
        }
        return ValidationResult.ok();
    }
}

// ============ ② 用户风控拦截 ============
@Component
@Order(2)
public class RiskControlValidator extends OrderSubmitValidator {

    @Autowired
    private RiskControlService riskControlService;

    @Value("${order.risk-check.enabled:true}")
    private boolean enabled;

    @Override
    protected ValidationResult doValidate(SubmitOrderContext context) {
        if (!enabled) {
            return ValidationResult.ok();  // 配置开关关闭，跳过
        }

        if (riskControlService.isBlacklisted(context.getUserId())) {
            return ValidationResult.fail("RISK_BLACKLIST", "用户已被加入黑名单，无法下单");
        }

        if (riskControlService.isAbnormalFrequency(
                context.getUserId(), context.getClientIp())) {
            return ValidationResult.fail("RISK_ABNORMAL_FREQ",
                "下单频率异常，请稍后再试");
        }

        return ValidationResult.ok();
    }
}

// ============ ③ 库存预校验 ============
@Component
@Order(3)
public class InventoryValidator extends OrderSubmitValidator {

    @Autowired
    private InventoryService inventoryService;

    @Override
    protected ValidationResult doValidate(SubmitOrderContext context) {
        for (OrderItem item : context.getItems()) {
            int stock = inventoryService.getAvailableStock(item.getSkuId());
            if (stock < item.getQuantity()) {
                return ValidationResult.fail("INVENTORY_SHORTAGE",
                    String.format("商品[%s]库存不足，当前库存:%d，需要:%d",
                        item.getSkuId(), stock, item.getQuantity()));
            }
        }
        return ValidationResult.ok();
    }
}

// ============ ④ 优惠券校验 ============
@Component
@Order(4)
public class CouponValidator extends OrderSubmitValidator {

    @Autowired
    private CouponService couponService;

    @Override
    protected ValidationResult doValidate(SubmitOrderContext context) {
        if (context.getCouponId() == null) {
            return ValidationResult.ok();  // 未使用优惠券，跳过
        }

        Coupon coupon = couponService.getById(context.getCouponId());
        if (coupon == null) {
            return ValidationResult.fail("COUPON_NOT_FOUND", "优惠券不存在");
        }
        if (coupon.getExpireTime().isBefore(LocalDateTime.now())) {
            return ValidationResult.fail("COUPON_EXPIRED", "优惠券已过期");
        }
        if (coupon.getMinAmount().compareTo(context.getTotalAmount()) > 0) {
            return ValidationResult.fail("COUPON_THRESHOLD",
                String.format("未达到优惠券使用门槛，还需消费¥%s",
                    coupon.getMinAmount().subtract(context.getTotalAmount())));
        }

        return ValidationResult.ok();
    }
}

// ============ 校验链构建器（自动编排） ============
@Component
public class OrderSubmitValidationPipeline {

    private final OrderSubmitValidator head;

    public OrderSubmitValidationPipeline(List<OrderSubmitValidator> validators) {
        // 按 @Order 注解排序，构建责任链
        List<OrderSubmitValidator> sorted = validators.stream()
            .sorted(Comparator.comparingInt(v ->
                v.getClass().getAnnotation(Order.class).value()))
            .collect(Collectors.toList());

        this.head = sorted.get(0);
        for (int i = 0; i < sorted.size() - 1; i++) {
            sorted.get(i).setNext(sorted.get(i + 1));
        }
    }

    public ValidationResult execute(SubmitOrderContext context) {
        log.info("开始执行订单提交校验链, userId={}", context.getUserId());
        long start = System.currentTimeMillis();
        ValidationResult result = head.validate(context);
        log.info("校验链执行完成, 耗时:{}ms, 结果:{}",
            System.currentTimeMillis() - start, result.isPassed());
        return result;
    }
}

// ============ 控制器调用 ============
@RestController
@RequestMapping("/api/order")
public class OrderController {

    @Autowired
    private OrderSubmitValidationPipeline pipeline;

    @Autowired
    private OrderService orderService;

    @PostMapping("/submit")
    public Result<?> submit(@RequestBody SubmitOrderRequest request) {
        SubmitOrderContext context = SubmitOrderContext.builder()
            .userId(request.getUserId())
            .items(request.getItems())
            .totalAmount(request.getTotalAmount())
            .couponId(request.getCouponId())
            .submitTime(LocalDateTime.now())
            .clientIp(getClientIp())
            .build();

        ValidationResult validation = pipeline.execute(context);
        if (!validation.isPassed()) {
            return Result.fail(validation.getErrorCode(), validation.getErrorMsg());
        }

        String orderId = orderService.submit(context);
        return Result.success(orderId);
    }

    private String getClientIp() {
        // 从Request中获取客户端IP
        return "";
    }
}
```

### 模式说明

AI 精准实现了**责任链模式**：

1. **`OrderSubmitValidator` 抽象类**——责任链的"链节"，定义 `next` 引用和 `doValidate` 抽象方法
2. **`setNext()` 返回 `this.next`**——支持流畅的链式构建
3. **短路机制**——`doValidate` 返回失败时直接 `return result`，不执行后续节点
4. **`@Order(n)` 注解**——用于声明校验顺序，`OrderSubmitValidationPipeline` 自动按优先级排序并构建链条
5. **配置开关**——`RiskControlValidator` 通过 `@Value("${order.risk-check.enabled:true}")` 支持动态开关，不走 if-else

---

## 核心技巧：如何在 Prompt 中"潜移默化"地指定设计模式

很多同学一上来就写"请用工厂模式实现XXX"，结果 AI 生成了一个名为 `XXXFactory` 的类——这没问题，但太直白了，不符合真实项目的命名习惯。我的做法是：

### 技巧1：用"术语的描述"代替"术语本身"

| ❌ 不要这样写 | ✅ 应该这样写 |
|--------------|--------------|
| "使用工厂模式" | "根据类型动态创建对应的处理器，新增类型时不修改已有代码" |
| "使用策略模式" | "每种算法独立封装，运行时根据条件选择算法" |
| "使用观察者模式" | "状态变更时通知多个独立的订阅者，新增订阅者不需要改核心代码" |
| "使用责任链模式" | "多个校验步骤依次执行，某一环节失败就终止，校验顺序可配置" |
| "使用单例模式" | "全局只有一个实例，线程安全，延迟加载" |

**原理**：AI 理解的是"问题描述"，不是"术语标签"。你描述的是"想解决什么问题"，AI 自己会匹配到最合适的设计模式，而且类名会更贴近业务语义。

### 技巧2：提供"需求维度"让 AI 自己判断复杂度

```
这个场景有三个需求维度：易扩展、高性能、代码简洁。
优先级：易扩展 > 高性能 > 代码简洁。
请根据这个优先级选择合适的设计方案，给出理由。
```

这让 AI 在工厂模式、策略模式、甚至是简单 Map 映射之间做选择，而不是你教它做选择。**AI 的判断往往比你更客观，因为它不会被"我要用设计模式装逼"的心理干扰。**

---

## 踩坑实录：如果用错了设计模式，AI 什么表现？

我做过一个有趣的实验——故意让 AI 对一个简单场景使用过度设计。

### 场景：判断一个数字是正数、负数还是零

如果是你自己写，一个 if-else 搞定。我偏要 AI 用设计模式：

**Prompt**：
> "请使用策略模式+工厂模式实现一个数字符号判断器，输入一个整数，输出 POSITIVE / NEGATIVE / ZERO。"

**AI 的输出**：

```java
// AI 真的生成了5个文件：
// 1. SignStrategy.java (接口)
// 2. PositiveSignStrategy.java
// 3. NegativeSignStrategy.java
// 4. ZeroSignStrategy.java
// 5. SignStrategyFactory.java
// 6. SignContext.java

// 然后 AI 在代码末尾加了一段"反思"：
// "注意：对于当前场景，使用策略模式属于过度设计。
// 建议使用简单的 if-else 或三元运算符实现，代码量和维护成本更低。"
```

**AI 甚至会主动提醒你过度设计了！** 这就是好 Prompt 的价值——你不仅要问"怎么做"，还要问"该不该这么做"。

### 如何纠正过度设计

当你发现 AI 过度设计了，只需要追加一句 Prompt：

```
上面的实现过于复杂，这个场景不需要设计模式。
请用最简单直接的方式重写，保持代码可读性即可。
评估指标：代码行数、可读性、新人理解成本。
```

AI 会立即给出精简版。关键是你得**自己判断**是否过度设计——AI 不会拒绝你的要求，它只会忠实地执行，无论你的要求是否合理。

---

## 判断标准：什么时候该用设计模式？

| 判断维度 | 该用设计模式 | 不该用设计模式 |
|---------|-------------|---------------|
| 分支数量 | ≥3 个分支，且未来可能增加 | 1-2 个分支，固定不变 |
| 复杂度 | 每个分支逻辑差异大 | 每个分支只是参数不同 |
| 团队规模 | 多人维护，各自负责不同分支 | 一个人维护，改来改去就那几个分支 |
| 面试场景 | 面试官指定要用 XX 模式 | 面试官让你用最简单的方案 |

**核心原则**：设计模式不是越多越好。**当且仅当一段代码的"变更原因"不同时，才值得拆分。** 如果两段代码永远一起变，即使长得不一样，也不该拆。

---

## 附：AI 生成设计模式的验证 Prompt

当你让 AI 生成了一套设计模式的代码后，用这个 Prompt 快速验证：

```
请对刚才生成的代码进行以下检查：

1. 【开闭原则验证】
   - 新增一个XXX（新类型/新策略/新处理器）需要修改哪些已有文件？
   - 理想答案：只新增1个类 + 添加1个枚举值，不改任何已有文件。

2. 【单一职责验证】
   - 每个类是否只有一个变化原因？
   - 请逐类分析：类A变化的原因是什么？类B变化的原因是什么？

3. 【依赖倒置验证】
   - 客户端代码是否只依赖接口/抽象，不依赖具体实现？
   - 找出所有违反此原则的引用

4. 【可测试性验证】
   - 每个策略/处理器是否可以独立进行单元测试？
   - 哪些类必须依赖Spring容器才能测试？

5. 【过度设计检查】
   - 这个场景有2个分支，你生成了5个类——是否有过度设计嫌疑？
   - 请评估：用简单if-else实现需要多少行？当前方案多少行？
```

这个验证 Prompt 等于让 AI 自己 review 自己的代码，效果出奇地好。

---

## 总结

三个关键认知：

1. **描述"问题场景"，不要说"设计模式名字"。** AI 会自己选择最合适的模式，而且类名更自然。
2. **AI 是"过度设计检测器"，不是"过度设计制造器"。** 合理 Prompt 下，AI 甚至会主动提醒你该不该用设计模式。
3. **设计模式的核心价值是"应对变化"，不是"让代码看起来高级"。** 用 AI 生成了 100 个类，如果不解决任何一个"变更场景"，那就是 100 个技术债。

**把重复的、模板化的、有标准答案的代码生成交给 AI，你的大脑留给真正的架构决策。这才是 Prompt 工程的核心价值。**

---

## 下篇预告：Prompt 驱动 JVM 调优

设计模式是"架构之美"，下一篇我们聊点硬的：生产环境 Full GC 频繁、STW 200ms、QPS 腰斩——把 GC 日志 + JVM 配置喂给 AI，让它秒出优化方案。G1 参数、堆大小、年轻代比例，一条 Prompt 全搞定。

**如果这篇文章对你有帮助，请点赞、收藏、转发，我们下篇见。**

---

*作者：一个用 Prompt 让 AI 写了 100 种设计模式的老 Java 人*
