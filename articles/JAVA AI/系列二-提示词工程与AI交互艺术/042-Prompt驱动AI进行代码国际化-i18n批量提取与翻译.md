# Prompt 驱动 AI 进行代码国际化（i18n）：批量提取与翻译，200个硬编码中文提示语一分钟变多语言

> 产品经理突然宣布：下个月项目出海，要先上英文版、日文版、韩文版。你打开项目一看——200 个 Controller 里到处都是硬编码的中文提示信息："用户名不能为空""订单创建成功""优惠券已过期"……一个个改成 `messageSource.getMessage()` 再加资源文件，光想想就头皮发麻。今天，用 AI 一句 Prompt 全搞定。

---

## 开篇：i18n 改造的 5 个让人崩溃的步骤

传统国际化改造流程：

1. 逐个文件扫描硬编码中文 → 人肉扫描，大概率遗漏
2. 给每个中文定义 key → 命名纠结 5 分钟，100 个 key 就纠结 500 分钟
3. 把中文搬到 `messages_zh_CN.properties` → 复制粘贴到天荒地老
4. 翻译成英文/日文/韩文 → 机翻别扭，人工翻译贵且慢
5. 把代码里的硬编码改成 `messageSource.getMessage("xxx")` → 每改一个都怕改错

**5 个步骤，200 个文件，保守估计 3 天工时。** 而用 AI，一句 Prompt，全程不超过 5 分钟。

---

## 完整实战：从硬编码中文到多语言 i18n 的 4 步流程

以一个真实 Controller 为例，展示完整流程：

### 原始代码（Before）

```java
@RestController
@RequestMapping("/api/order")
@Slf4j
public class OrderController {

    @Autowired
    private OrderService orderService;

    @PostMapping("/create")
    public Result<OrderVO> create(@RequestBody @Valid CreateOrderRequest request) {
        try {
            OrderVO order = orderService.create(request);
            return Result.success(order, "订单创建成功");
        } catch (BusinessException e) {
            log.error("创建订单失败", e);
            return Result.fail(e.getCode(), e.getMessage());
        }
    }

    @PostMapping("/cancel/{orderId}")
    public Result<Void> cancel(@PathVariable Long orderId) {
        Order order = orderService.getById(orderId);
        if (order == null) {
            throw new BusinessException("订单不存在");
        }
        if (!order.getStatus().equals("CREATED")) {
            throw new BusinessException("该订单状态不允许取消，当前状态：" + order.getStatus());
        }
        if (order.getPaidTime() != null) {
            throw new BusinessException("已支付的订单无法取消，请申请退款");
        }
        orderService.cancel(orderId);
        return Result.success(null, "订单取消成功");
    }

    @GetMapping("/detail/{orderId}")
    public Result<OrderDetailVO> detail(@PathVariable Long orderId) {
        Order order = orderService.getById(orderId);
        if (order == null) {
            throw new BusinessException("订单不存在");
        }
        return Result.success(orderService.getDetail(orderId), "查询成功");
    }
}
```

---

## 第一步：用 AI 扫描代码找出所有硬编码中文

### Prompt

```
你是一个Java代码分析专家。请扫描以下代码，找出所有硬编码的中文字符串。

【分析目标】
- 找出所有直接在代码中写死的中文字符串（包括拼接的）
- 排除：日志中的中文、注释中的中文、JavaDoc中的中文
- 排除：字符串常量类中已定义的中文（如果引用了Constants类的话）

【输出格式】按表格列出：
| 行号 | 原始字符串                    | 出现位置        | 类型       |
|------|-----------------------------|----------------|------------|
|  15  | "订单创建成功"               | Result.success | 提示信息   |
|  20  | "订单不存在"                 | throw异常      | 错误信息   |

【扫描的代码】
[粘贴你的Controller/Service代码]
```

### AI 输出

```
═══════════════════════════════════════════
         硬编码中文字符串扫描结果
═══════════════════════════════════════════

| 行号 | 原始字符串                                      | 出现位置        | 类型       |
|------|-----------------------------------------------|----------------|------------|
|  15  | "订单创建成功"                                 | Result.success | 成功提示   |
|  20  | "订单不存在"                                   | throw异常      | 业务错误   |
|  26  | "该订单状态不允许取消，当前状态："+order.getStatus() | throw异常   | 业务错误   |
|  29  | "已支付的订单无法取消，请申请退款"               | throw异常      | 业务错误   |
|  32  | "订单取消成功"                                 | Result.success | 成功提示   |
|  40  | "订单不存在"                                   | throw异常      | 业务错误（重复）|
|  42  | "查询成功"                                     | Result.success | 成功提示   |

统计：
- 共发现 7 处硬编码中文
- 不重复字符串：6 个（"订单不存在"出现 2 次）
- 包含动态参数：1 个（"该订单状态不允许取消，当前状态："）
```

---

## 第二步：AI 生成 i18n 的 Key 命名建议

### Prompt

```
基于上面扫描出的硬编码中文，请为每个字符串设计 i18n 的 key。

【命名规范】
格式：模块_场景_含义（三段式，中间用 . 分隔，全部小写字母 + 下划线）
- 模块：order / user / product / payment / system 等
- 场景：create / cancel / detail / query / validate / error 等
- 含义：用英文简短描述（success / not_found / invalid_status / paid_cannot_cancel 等）

要求：
1. key 能"望文生义"，不看作中文也知道含义
2. 长度控制在 15-40 个字符
3. 不要用拼音

【输出格式】
| 中文原文                               | 建议key                              |
|---------------------------------------|--------------------------------------|
| "订单创建成功"                         | order.create.success                 |
| "订单不存在"                           | order.error.not_found                |
```

### AI 输出

```
| 中文原文                                      | 建议key                                      |
|----------------------------------------------|----------------------------------------------|
| "订单创建成功"                                | order.create.success                          |
| "订单不存在"                                  | order.error.not_found                         |
| "该订单状态不允许取消，当前状态：{0}"          | order.cancel.invalid_status                   |
| "已支付的订单无法取消，请申请退款"             | order.cancel.paid_cannot_cancel               |
| "订单取消成功"                                | order.cancel.success                          |
| "查询成功"                                    | order.detail.query_success                    |

命名说明：
- 后缀 success → 成功提示类消息
- 前缀 order.error. → 错误类消息
- 动态参数用 {0} 占位，对应 MessageFormat
```

---

## 第三步：AI 生成多语言 properties 文件

### Prompt

```
你是专业翻译。请将以下中文国际化消息翻译成英文、日文、韩文。

要求：
1. 英文：不直译，用地道的英语表达。不要Chinglish。
   - "订单不存在" → "Order not found"（不是 "Order does not exist"）
   - "订单创建成功" → "Order created successfully"（不是 "Order create success"）
   - "查询成功" → 如果只是接口返回提示可以不翻译，英文用户不看这个
     → 改为 "Success" 即可

2. 日文：使用敬体（です・ます調），电商场景用です调即可，不要太硬的敬语

3. 韩文：使用正式的합니다体，电商场景用即可

4. 包含 {0} 的格式化字符串，在所有语言中保持 {0} 位置不变

5. 每条中文对应三行翻译

输出格式：标准的 .properties 文件内容，包含中文原文注释。
```

### AI 输出

```properties
# ========================================
#  国际化资源文件
#  模块: order (订单模块)
# ========================================

# ============ 订单创建 ============
# 中文: 订单创建成功
order.create.success = 订单创建成功
order.create.success.en = Order created successfully
order.create.success.ja = 注文が作成されました
order.create.success.ko = 주문이 생성되었습니다

# ============ 订单错误 ============
# 中文: 订单不存在
order.error.not_found = 订单不存在
order.error.not_found.en = Order not found
order.error.not_found.ja = 注文が見つかりません
order.error.not_found.ko = 주문을 찾을 수 없습니다

# ============ 订单取消 ============
# 中文: 该订单状态不允许取消，当前状态：{0}
order.cancel.invalid_status = 该订单状态不允许取消，当前状态：{0}
order.cancel.invalid_status.en = This order cannot be cancelled in its current status: {0}
order.cancel.invalid_status.ja = 現在の状態ではこの注文をキャンセルできません：{0}
order.cancel.invalid_status.ko = 현재 상태에서는 이 주문을 취소할 수 없습니다: {0}

# 中文: 已支付的订单无法取消，请申请退款
order.cancel.paid_cannot_cancel = 已支付的订单无法取消，请申请退款
order.cancel.paid_cannot_cancel.en = A paid order cannot be cancelled. Please request a refund instead.
order.cancel.paid_cannot_cancel.ja = 支払い済みの注文はキャンセルできません。返金を申請してください。
order.cancel.paid_cannot_cancel.ko = 결제된 주문은 취소할 수 없습니다. 환불을 신청해 주세요.

# 中文: 订单取消成功
order.cancel.success = 订单取消成功
order.cancel.success.en = Order cancelled successfully
order.cancel.success.ja = 注文がキャンセルされました
order.cancel.success.ko = 주문이 취소되었습니다

# ============ 订单详情 ============
# 中文: 查询成功
order.detail.query_success = 查询成功
order.detail.query_success.en = Success
order.detail.query_success.ja = 照会が完了しました
order.detail.query_success.ko = 조회가 완료되었습니다
```

> **提示**：如果你的框架只支持标准的 `messages_zh_CN.properties` / `messages_en_US.properties` 分文件模式，可以让 AI 改成按语言分文件输出。上面的格式适合自定义的多语言 key 方案。

---

## 第四步：AI 重构代码——硬编码改为 MessageSource 调用

### Prompt

```
你是一个Java代码重构专家。请将以下Controller中的硬编码中文替换为国际化调用。

【技术方案】
使用Spring的MessageSource，通过依赖注入获取。
国际化key使用上面定义的命名。

【约束条件】
1. 严格保持原有业务逻辑不变
2. 带参数的字符串用MessageFormat：MessageSource.getMessage(key, new Object[]{param}, locale)
3. 需要引入Locale对象，从请求头Accept-Language解析
4. BusinessException 类需要支持传入 key + 参数（内部自己调messageSource）
5. Result.success / fail 需要支持传入 key，内部自己调messageSource

【输出要求】
- 重构后的完整代码
- 需要的配置类（MessageSource配置、LocaleResolver配置）
- 需要的工具类（I18nUtil）
- 如有无法直接替换的地方，标注出来并说明原因

原始代码：
[粘贴Controller代码]
```

### AI 生成的完整代码

```java
// ============ 1. i18n 配置 ============
@Configuration
public class I18nConfig {

    @Bean
    public MessageSource messageSource() {
        ReloadableResourceBundleMessageSource messageSource =
            new ReloadableResourceBundleMessageSource();
        messageSource.setBasename("classpath:i18n/messages");  // 基础名
        messageSource.setDefaultEncoding("UTF-8");
        messageSource.setCacheSeconds(3600);  // 1小时缓存
        messageSource.setFallbackToSystemLocale(false);
        // 返回key本身而不是null（方便排查未翻译的key）
        messageSource.setUseCodeAsDefaultMessage(true);
        return messageSource;
    }

    @Bean
    public LocaleResolver localeResolver() {
        AcceptHeaderLocaleResolver resolver = new AcceptHeaderLocaleResolver();
        resolver.setDefaultLocale(Locale.SIMPLIFIED_CHINESE);
        // 支持的语言
        resolver.setSupportedLocales(Arrays.asList(
            Locale.SIMPLIFIED_CHINESE,
            Locale.US,
            Locale.JAPAN,
            Locale.KOREA
        ));
        return resolver;
    }
}

// ============ 2. I18n 工具类 ============
@Component
public class I18nUtil {

    private static MessageSource messageSource;

    public I18nUtil(MessageSource messageSource) {
        I18nUtil.messageSource = messageSource;
    }

    /**
     * 获取当前请求的国际化消息
     */
    public static String get(String key) {
        return messageSource.getMessage(key, null, getLocale());
    }

    /**
     * 获取带参数的国际化消息
     * 使用方式：I18nUtil.get("order.cancel.invalid_status", status)
     */
    public static String get(String key, Object... args) {
        return messageSource.getMessage(key, args, getLocale());
    }

    private static Locale getLocale() {
        RequestAttributes attributes = RequestContextHolder.getRequestAttributes();
        if (attributes instanceof ServletRequestAttributes) {
            return ((ServletRequestAttributes) attributes)
                .getRequest().getLocale();
        }
        return Locale.SIMPLIFIED_CHINESE;
    }
}

// ============ 3. 国际化消息Key常量类 ============
public interface I18nKeys {

    // 订单模块
    String ORDER_CREATE_SUCCESS = "order.create.success";
    String ORDER_ERROR_NOT_FOUND = "order.error.not_found";
    String ORDER_CANCEL_INVALID_STATUS = "order.cancel.invalid_status";
    String ORDER_CANCEL_PAID_CANNOT_CANCEL = "order.cancel.paid_cannot_cancel";
    String ORDER_CANCEL_SUCCESS = "order.cancel.success";
    String ORDER_DETAIL_QUERY_SUCCESS = "order.detail.query_success";
}

// ============ 4. 重构后的 BusinessException ============
public class BusinessException extends RuntimeException {

    private final String errorCode;
    private final String i18nKey;        // 国际化 key
    private final Object[] i18nArgs;     // 国际化参数

    /**
     * 使用国际化key构造（推荐）
     */
    public BusinessException(String errorCode, String i18nKey, Object... i18nArgs) {
        super(I18nUtil.get(i18nKey, i18nArgs));
        this.errorCode = errorCode;
        this.i18nKey = i18nKey;
        this.i18nArgs = i18nArgs;
    }

    public String getErrorCode() { return errorCode; }

    public String getI18nKey() { return i18nKey; }

    public Object[] getI18nArgs() { return i18nArgs; }

    // 兼容旧代码：直接传中文（不推荐）
    @Deprecated
    public BusinessException(String directMessage) {
        super(directMessage);
        this.errorCode = "BIZ_ERROR";
        this.i18nKey = null;
        this.i18nArgs = null;
    }
}

// ============ 5. 重构后的统一响应 ============
public class Result<T> {

    private int code;
    private String message;
    private T data;

    /**
     * 成功：使用国际化消息
     */
    public static <T> Result<T> success(T data, String i18nKey) {
        Result<T> result = new Result<>();
        result.code = 200;
        result.message = I18nUtil.get(i18nKey);
        result.data = data;
        return result;
    }

    /**
     * 失败：使用国际化消息
     */
    public static <T> Result<T> fail(String errorCode, String i18nKey, Object... args) {
        Result<T> result = new Result<>();
        result.code = 500;
        result.message = I18nUtil.get(i18nKey, args);
        return result;
    }

    // 兼容旧代码（不推荐）
    @Deprecated
    public static <T> Result<T> fail(String errorCode, String directMessage) {
        Result<T> result = new Result<>();
        result.code = 500;
        result.message = directMessage;
        return result;
    }
}

// ============ 6. 重构后的 Controller ============
@RestController
@RequestMapping("/api/order")
@Slf4j
public class OrderController {

    @Autowired
    private OrderService orderService;

    @PostMapping("/create")
    public Result<OrderVO> create(@RequestBody @Valid CreateOrderRequest request) {
        try {
            OrderVO order = orderService.create(request);
            return Result.success(order, I18nKeys.ORDER_CREATE_SUCCESS);
        } catch (BusinessException e) {
            log.error("创建订单失败", e);
            return Result.fail(e.getErrorCode(), e.getI18nKey(), e.getI18nArgs());
        }
    }

    @PostMapping("/cancel/{orderId}")
    public Result<Void> cancel(@PathVariable Long orderId) {
        Order order = orderService.getById(orderId);
        if (order == null) {
            throw new BusinessException("ORDER_001",
                I18nKeys.ORDER_ERROR_NOT_FOUND);
        }
        if (!"CREATED".equals(order.getStatus())) {
            throw new BusinessException("ORDER_002",
                I18nKeys.ORDER_CANCEL_INVALID_STATUS,
                order.getStatus());  // 参数：当前状态
        }
        if (order.getPaidTime() != null) {
            throw new BusinessException("ORDER_003",
                I18nKeys.ORDER_CANCEL_PAID_CANNOT_CANCEL);
        }
        orderService.cancel(orderId);
        return Result.success(null, I18nKeys.ORDER_CANCEL_SUCCESS);
    }

    @GetMapping("/detail/{orderId}")
    public Result<OrderDetailVO> detail(@PathVariable Long orderId) {
        Order order = orderService.getById(orderId);
        if (order == null) {
            throw new BusinessException("ORDER_001",
                I18nKeys.ORDER_ERROR_NOT_FOUND);
        }
        return Result.success(orderService.getDetail(orderId),
            I18nKeys.ORDER_DETAIL_QUERY_SUCCESS);
    }
}
```

### 重构后代码质量分析

现在代码中完全消除了硬编码中文，所有提示信息都由 `I18nUtil.get(key)` 根据请求头 `Accept-Language` 自动选择语言。

---

## 多语言翻译的 Prompt 技巧：如何避免 Chinglish

这个章节单独拿出来说，因为翻译质量直接决定用户体验。同样是 AI 翻译，Prompt 写得好与坏差距巨大。

### 反面案例：不加技巧的翻译

```
Prompt: 翻译成英文："已支付的订单无法取消，请申请退款"
AI输出: "Paid orders cannot be cancelled, please apply for a refund"
```

**问题**："apply for a refund"——太生硬了。用户看到"apply for a refund"会联想到填表、审批、排队，但实际上点击一个按钮就行。

### 正面案例：加了地道表达技巧的翻译

在 Prompt 中加入以下约束：

```
翻译规则：
1. 电商/互联网产品场景，用口语化的用户表达，不要用银行/政府风格
2. "请申请退款" → "Request a refund"（不用 Apply for）
3. "不可用" → "Unavailable"（不用 Cannot be used）
4. "操作失败，请稍后重试" → "Something went wrong. Please try again later."
   （不用 "Operation failed, please retry later"）
5. 如果英文表达比中文简短很多，就用简短的（英文用户习惯简洁）
6. 错误提示不要写得像系统报错（如 "Error code 500: Internal Server Error"），
   要写成像客服告诉你的一样（如 "Oops! We're having some trouble. Please try again."）

参考例子：
| 中文直译（Chinglish）      | 地道英文                              |
|---------------------------|---------------------------------------|
| Operation success         | Done!                                 |
| Your operation is too frequent | Too many requests. Take a break! |
| Network error, retry later    | Connection lost. Tap to retry.    |
| Insufficient permissions  | You don't have access to this page    |
```

### AI 改进后的翻译

```
"已支付的订单无法取消，请申请退款"
→ "This order has been paid. Need a refund instead?"

"操作太频繁，请稍后重试"
→ "Whoa, slow down! Please try again in a moment."

"权限不足"
→ "You don't have permission to perform this action."

"网络异常，请重试"
→ "Connection lost. Let's try that again."
```

**对比效果**：AI 不仅能翻译，还能让你的产品文案从"能用"升级为"好用"。

---

## 代码重构的验证方案：如何确保功能不变

代码改完了，怎么验证没问题？用这个 Prompt 让 AI 帮你做行为等价性检查：

### Prompt

```
你是一个Java代码审查专家。以下是重构前后的代码，请做行为等价性验证。

【原始代码】
[粘贴原始Controller代码]

【重构后代码】
[粘贴重构后Controller代码]

【验证要求】
1. 每个 throw new BusinessException 的参数是否和原始保持一致？
   - 原始：throw new BusinessException("已支付的订单无法取消，请申请退款")
   - 重构后：throw new BusinessException("ORDER_003", I18nKeys.ORDER_CANCEL_PAID_CANNOT_CANCEL)
   - 验证：ORDER_CANCEL_PAID_CANNOT_CANCEL 这个 key 对应的中文消息是否和原始一致

2. 异常类型和异常传播路径是否保持一致？

3. Result.success/Result.fail 的 code/message/data 三个字段的值是否保持不变？

4. 日志输出是否保持不变？

5. 找出任何可能的行为差异并标注风险等级（高/中/低）
```

### AI 输出

```
═══════════════════════════════════════════
         重构行为等价性检查报告
═══════════════════════════════════════════

✅ 异常类型一致：仍为 BusinessException
✅ 异常传播路径一致：仍由 Controller 层抛给全局异常处理器
✅ 日志输出一致：未修改任何日志代码
⚠️ 低风险：message 字段的值
  - 重构前：Result.message = "订单创建成功"（固定中文）
  - 重构后：Result.message = I18nUtil.get("order.create.success")（根据当前Locale）
  - 结论：对中文用户行为一致；对非中文用户返回对应语言，这是预期行为
⚠️ 低风险：BusinessException.getMessage()
  - 重构前：new BusinessException("订单不存在") → getMessage() = "订单不存在"
  - 重构后：new BusinessException("ORDER_001", I18nKeys.ORDER_ERROR_NOT_FOUND)
    → getMessage() = I18nUtil.get("order.error.not_found")
  - 结论：通过全局异常处理器最终返回给用户的message不变，但Java层面的
    getMessage() 行为略有不同。如有代码依赖 getMessage() 做逻辑判断，
    需要改为依赖 getErrorCode()。

═══════════════════════════════════════════
       建议：添加单元测试验证
═══════════════════════════════════════════
```

### 自动化验证脚本

```java
@SpringBootTest
class OrderControllerI18nTest {

    @Autowired
    private MessageSource messageSource;

    @Test
    void testAllKeysHaveTranslations() {
        // 验证所有 key 在四种语言中都有翻译
        String[] keys = {
            I18nKeys.ORDER_CREATE_SUCCESS,
            I18nKeys.ORDER_ERROR_NOT_FOUND,
            // ...所有key
        };

        Locale[] locales = {
            Locale.SIMPLIFIED_CHINESE,
            Locale.US,
            Locale.JAPAN,
            Locale.KOREA
        };

        for (String key : keys) {
            for (Locale locale : locales) {
                String msg = messageSource.getMessage(key, null, locale);
                assertNotNull("key=" + key + " locale=" + locale + " is null", msg);
                assertNotEquals("key=" + key + " locale=" + locale + " returns key itself", key, msg);
            }
        }
    }

    @Test
    void testI18nMessageMatchesOriginal() {
        // 验证中文消息与原始硬编码一致
        assertEquals("订单创建成功",
            messageSource.getMessage(I18nKeys.ORDER_CREATE_SUCCESS, null, Locale.SIMPLIFIED_CHINESE));
        assertEquals("订单不存在",
            messageSource.getMessage(I18nKeys.ORDER_ERROR_NOT_FOUND, null, Locale.SIMPLIFIED_CHINESE));
        assertEquals("已支付的订单无法取消，请申请退款",
            messageSource.getMessage(I18nKeys.ORDER_CANCEL_PAID_CANNOT_CANCEL, null, Locale.SIMPLIFIED_CHINESE));
    }

    @Test
    void testParameterizedMessages() {
        String msg = messageSource.getMessage(
            I18nKeys.ORDER_CANCEL_INVALID_STATUS,
            new Object[]{"PAID"},
            Locale.US
        );
        assertEquals("This order cannot be cancelled in its current status: PAID", msg);
    }
}
```

---

## i18n 改造的常见陷阱和 AI 补救方案

### 陷阱1：前端硬编码中文忘了改

**问题**：后端改完了，前端 JS 里还有一堆硬编码。

**AI 补救**：把前端代码也喂给 AI，用同样的流程扫描 + 提取 + 重构。

### 陷阱2：数据库里的中文数据

**问题**：`t_dict` 表里存的中文状态描述，前端直接展示，没有国际化。

**AI 补救 Prompt**：

```
数据库t_dict表存储了业务字典的中文值，字段为dict_value_cn。
请设计一个方案，加字典国际化字段，或者前端做映射。
```

### 陷阱3：枚举值的中文 displayName

**问题**：

```java
public enum OrderStatus {
    CREATED("待支付"),
    PAID("已支付");
    
    private String displayName;
}
```

**AI 补救**：

```java
public enum OrderStatus {
    CREATED("order.status.created"),
    PAID("order.status.paid");
    
    private final String i18nKey;
    
    public String getDisplayName() {
        return I18nUtil.get(i18nKey);
    }
}
```

---

## 总结

一次完整的 i18n 改造，用 AI 配合完成的效果统计：

| 步骤 | 人工耗时 | AI 耗时 | 质量对比 |
|------|---------|---------|---------|
| 扫描硬编码 | 2-4小时 | 30秒 | AI 不会遗漏 |
| 生成 key | 1-2小时 | 30秒 | AI 命名更规范 |
| 翻译4种语言 | 2-4小时 | 1分钟 | AI 翻译+地道表达 |
| 重构代码 | 4-8小时 | 1分钟 | 行为等价性经过验证 |
| **合计** | **1-3天** | **3分钟** | **基本零遗漏** |

三个关键认知：

1. **i18n 的本质不是"翻译"，是"重构 + 翻译 + 验证"。** 翻译只是最后一步，前面三步（扫描、命名、重构）才是大头——而这些 AI 全部能做。

2. **不要担心 AI 翻译不地道——给它"风格约束"就行了。** Prompt 中明确"电商场景、用户友好、口语化"，AI 的翻译质量远超市面上大多数机翻产品。

3. **i18n 的最佳时机是"项目初期"——但你永远不会在项目初期做 i18n。** 所以，在"不得不做"的时候，AI 是你的救命稻草。

**200 个 Controller，200 个硬编码中文，3 分钟全部解决。这不叫效率提升，这叫维度碾压。**

---

## 下篇预告：多轮对话上下文管理

i18n 改造只是 Prompt 工程的一个应用场景。下一篇我们聊一个更核心的话题——**多轮对话上下文管理**：当你的对话历史超过 10 轮，AI 开始"失忆"怎么办？如何设计上下文的保留和淘汰策略？对话树、对话分支、对话回溯——把 AI 当成一个有记忆的结对编程伙伴来管理。

**如果这篇文章对你有帮助，请点赞、收藏、转发，我们下篇见。**

---

*作者：曾手动改了300个i18n key后崩溃，从此一切交给AI的老Java人*
