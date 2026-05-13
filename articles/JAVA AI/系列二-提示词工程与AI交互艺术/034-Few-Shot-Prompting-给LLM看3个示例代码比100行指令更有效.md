# Few-Shot Prompting：给 LLM 看 3 个示例代码比 100 行指令更有效，AI 的学习方式是"模仿"而不是"理解"

> 你学新框架时，是抱着官方文档啃 300 页，还是直奔 GitHub 搜一个 star 最多的示例项目 clone 下来跑？  
> 99% 的人选了后者。  
> AI 也一样。

---

## 一、一个扎心的真相

我给你看两组 Prompt，你猜哪个产出的代码质量更高：

**Prompt A（Zero-Shot，纯指令）**：
> "请用 Spring Boot 写一个用户注册的 Controller，要有参数校验、统一返回格式、操作日志、异常处理。"

**Prompt B（Few-Shot，3 个示例）**：
> "下面是我们项目中 3 个已有的 Controller。请参考它们的风格，帮我写第 4 个——用户注册接口。"
> （然后贴了 3 个 Controller 的完整代码）

**答案：Prompt B 产出的代码，改动量通常不到 Prompt A 的 1/10。**

为什么？因为 AI 不"理解"你的规范——它只是"模仿"你的模式。你用 100 行文字去描述"我们的返回格式是 `Result<T>`，字段名叫 `code`/`msg`/`data`"，不如直接甩 3 个现成的 Controller 给它看。

这就是 **Few-Shot Prompting** 的核心威力。

---

## 二、Few-Shot Prompting 到底是什么？

简单说：**在 Prompt 里塞入几个示例（输入+期望输出），让 LLM 照葫芦画瓢。**

它的底层原理不是什么魔法——LLM 本质上是"下一个 Token 预测器"。当你给它看几个示例后，它内部的注意力机制会自动捕捉这些示例中的模式，然后"延续"这个模式去生成后续内容。

翻译成人话：**AI 的学习方式是"模仿"，不是"理解"。**

你告诉它"返回格式统一用 `Result<T>`"，它"听"得懂，但不一定"做"得到——因为训练数据里有 100 种 `Result` 的写法，它不知道你要哪一种。

你给它看 3 个 `Result<T>` 的实际代码，它瞬间就"学会"了——因为模式已经被精确锁定。

---

## 三、4 个 Java 实战案例：从"风格错乱"到"完美匹配"

下面我们用 4 个真实场景，展示 Few-Shot vs Zero-Shot 的天壤之别。

### 案例 1：定义一种自定义的 Controller 写法风格

**背景**：你们公司的 Controller 有一套管用风格：
- 返回类型统一用 `Result<T>`，包含 `code`/`msg`/`data` 三个字段
- 每个接口方法上必须有 `@OperateLog` 注解记录操作日志
- 入参对象必须用 `@Valid` 触发 JSR 校验
- URL 路径统一前缀 `/api/v1/`
- 不使用 `@Autowired` 字段注入，统一用构造器注入

#### Zero-Shot Prompt：

```markdown
请用 Spring Boot 写一个"创建订单"的 Controller。
要求：返回统一格式 Result<T>，要有操作日志注解，入参要校验，用构造器注入。
```

#### Zero-Shot 输出：

```java
@RestController
@RequestMapping("/order")
public class OrderController {

    @Autowired
    private OrderService orderService;

    @PostMapping("/create")
    public Result<OrderVO> createOrder(@RequestBody @Valid CreateOrderDTO dto) {
        log.info("创建订单：{}", dto);
        OrderVO result = orderService.createOrder(dto);
        return Result.success(result);
    }
}
```

**问题分析**：
- ❌ URL 路径是 `/order` 而不是 `/api/v1/order`
- ❌ 使用了 `@Autowired` 字段注入
- ❌ 没有 `@OperateLog` 注解
- ❌ `Result` 可能不是你们项目的那个 `Result`
- ❌ 日志格式不是你们项目统一用的格式

#### Few-Shot Prompt：

```markdown
下面是我们项目中的 3 个已有 Controller，请参考它们的编码风格，
帮我写第 4 个：订单管理 Controller，接口包括"创建订单"。

## 示例1：用户Controller
```java
@RestController
@RequestMapping("/api/v1/user")
@RequiredArgsConstructor
public class UserController {

    private final UserService userService;

    @PostMapping("/create")
    @OperateLog(module = "用户管理", type = "新增", description = "创建新用户")
    public Result<Long> createUser(@RequestBody @Valid CreateUserRequest request) {
        return Result.success(userService.createUser(request));
    }

    @GetMapping("/page")
    @OperateLog(module = "用户管理", type = "查询", description = "分页查询用户")
    public Result<PageResult<UserVO>> pageUsers(@Valid PageQuery query) {
        return Result.success(userService.pageUsers(query));
    }
}
```

## 示例2：商品Controller
```java
@RestController
@RequestMapping("/api/v1/product")
@RequiredArgsConstructor
public class ProductController {

    private final ProductService productService;

    @PostMapping("/create")
    @OperateLog(module = "商品管理", type = "新增", description = "新建商品")
    public Result<String> createProduct(@RequestBody @Valid CreateProductRequest request) {
        return Result.success(productService.createProduct(request));
    }
}
```

## 示例3：库存Controller
```java
@RestController
@RequestMapping("/api/v1/inventory")
@RequiredArgsConstructor
public class InventoryController {

    private final InventoryService inventoryService;

    @PostMapping("/adjust")
    @OperateLog(module = "库存管理", type = "调整", description = "调整库存数量")
    public Result<Void> adjustInventory(@RequestBody @Valid AdjustInventoryRequest request) {
        inventoryService.adjust(request);
        return Result.success();
    }
}
```

请按照上述风格，生成第4个：订单管理 Controller，
接口包括"创建订单"（POST /api/v1/order/create）。
```

#### Few-Shot 输出：

```java
@RestController
@RequestMapping("/api/v1/order")
@RequiredArgsConstructor
public class OrderController {

    private final OrderService orderService;

    @PostMapping("/create")
    @OperateLog(module = "订单管理", type = "新增", description = "创建新订单")
    public Result<Long> createOrder(@RequestBody @Valid CreateOrderRequest request) {
        return Result.success(orderService.createOrder(request));
    }
}
```

**对比分析**：

| 维度 | Zero-Shot | Few-Shot |
|------|-----------|----------|
| URL 前缀 | ❌ `/order` | ✅ `/api/v1/order` |
| 依赖注入 | ❌ `@Autowired` 字段注入 | ✅ `@RequiredArgsConstructor` + `final` |
| 操作日志 | ❌ 缺少 `@OperateLog` | ✅ 完整注解参数 |
| 类结构 | ❌ 不统一 | ✅ 与团队风格完全一致 |
| 改动量 | 5+ 处需手动调整 | 0 处 |

**关键洞察**：LLM 从 3 个示例中自动提取了以下模式：
1. 类注解必须是 `@RestController` + `@RequestMapping("/api/v1/xxx")` + `@RequiredArgsConstructor`
2. 依赖注入用 `private final` + Lombok
3. 每个接口方法都带 `@OperateLog` 且参数结构一致
4. 返回统一用 `Result.success()`
5. 入参统一用 `@RequestBody @Valid` + 自定义 Request 对象

**你什么都没说，它全学会了。**

---

### 案例 2：定义一种异常处理模式

**背景**：你们项目的异常处理有严格规范：
- 业务异常统一抛 `BusinessException`，传入错误码枚举
- 错误码枚举 `ErrorCode` 定义在 `enums` 包下，包含 `code`、`msg`、`httpStatus`
- 全局异常处理 `@RestControllerAdvice` 捕获后转为 `Result<T>` 返回
- 参数校验异常单独处理，返回字段级错误信息

#### Zero-Shot Prompt：

```markdown
请用 Spring Boot 写一个全局异常处理器，统一处理业务异常和参数校验异常。
```

#### Zero-Shot 输出：

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(RuntimeException.class)
    public ResponseEntity<String> handleRuntimeException(RuntimeException e) {
        return ResponseEntity.status(500).body(e.getMessage());
    }
}
```

**问题分析**：
- ❌ 返回类型是 `ResponseEntity<String>`，不是项目的 `Result<T>`
- ❌ 没有区分业务异常和系统异常
- ❌ 没有处理参数校验异常
- ❌ 没有使用错误码枚举

#### Few-Shot Prompt：

```markdown
以下是项目中已有的 3 个异常处理示例，请按照同样的模式写出完整的全局异常处理器。

## 示例1：BusinessException 定义
```java
public class BusinessException extends RuntimeException {
    private final ErrorCode errorCode;

    public BusinessException(ErrorCode errorCode) {
        super(errorCode.getMsg());
        this.errorCode = errorCode;
    }

    public ErrorCode getErrorCode() {
        return errorCode;
    }
}
```

## 示例2：ErrorCode 枚举（部分）
```java
public enum ErrorCode {

    SUCCESS(0, "操作成功", HttpStatus.OK),
    PARAM_INVALID(1001, "参数校验失败: %s", HttpStatus.BAD_REQUEST),
    USER_NOT_FOUND(2001, "用户不存在", HttpStatus.NOT_FOUND),
    ORDER_NOT_FOUND(3001, "订单不存在", HttpStatus.NOT_FOUND),
    INSUFFICIENT_INVENTORY(3002, "库存不足", HttpStatus.BAD_REQUEST),
    SYSTEM_ERROR(9999, "系统异常，请稍后重试", HttpStatus.INTERNAL_SERVER_ERROR);

    private final int code;
    private final String msg;
    private final HttpStatus httpStatus;

    ErrorCode(int code, String msg, HttpStatus httpStatus) {
        this.code = code;
        this.msg = msg;
        this.httpStatus = httpStatus;
    }
    // getters...
}
```

## 示例3：项目中已有的某个 RestControllerAdvice 片段
```java
@RestControllerAdvice
@Slf4j
public class UserModuleExceptionHandler {

    @ExceptionHandler(BusinessException.class)
    public Result<Void> handleBusinessException(BusinessException e) {
        log.warn("业务异常: code={}, msg={}", e.getErrorCode().getCode(), e.getMessage());
        return Result.fail(e.getErrorCode());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public Result<Void> handleValidationException(MethodArgumentNotValidException e) {
        String message = e.getBindingResult().getFieldErrors().stream()
                .map(f -> f.getField() + ": " + f.getDefaultMessage())
                .collect(Collectors.joining("; "));
        return Result.fail(ErrorCode.PARAM_INVALID.format(message));
    }
}
```

请参考以上示例，为"订单模块"写一个同样风格的全局异常处理器，
模块名 OrderModuleExceptionHandler，放在 order.exception 包下。
```

#### Few-Shot 输出：

```java
@RestControllerAdvice
@Slf4j
public class OrderModuleExceptionHandler {

    @ExceptionHandler(BusinessException.class)
    public Result<Void> handleBusinessException(BusinessException e) {
        log.warn("订单模块业务异常: code={}, msg={}", e.getErrorCode().getCode(), e.getMessage());
        return Result.fail(e.getErrorCode());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public Result<Void> handleValidationException(MethodArgumentNotValidException e) {
        String message = e.getBindingResult().getFieldErrors().stream()
                .map(f -> f.getField() + ": " + f.getDefaultMessage())
                .collect(Collectors.joining("; "));
        return Result.fail(ErrorCode.PARAM_INVALID.format(message));
    }

    @ExceptionHandler(Exception.class)
    public Result<Void> handleUnknownException(Exception e) {
        log.error("订单模块未知异常", e);
        return Result.fail(ErrorCode.SYSTEM_ERROR);
    }
}
```

**对比分析**：

| 维度 | Zero-Shot | Few-Shot |
|------|-----------|----------|
| 返回类型 | ❌ `ResponseEntity<String>` | ✅ `Result<Void>` |
| 业务异常 | ❌ 未区分 | ✅ 捕获 `BusinessException` 并读取 `ErrorCode` |
| 参数校验 | ❌ 未处理 | ✅ 流式拼接字段级错误信息 |
| 未知异常兜底 | ❌ 无 | ✅ 捕获 `Exception` 返回系统错误码 |
| 日志格式 | ❌ 无 | ✅ `log.warn` / `log.error` 格式统一 |
| 类注解 | ❌ 无 `@Slf4j` | ✅ `@RestControllerAdvice` + `@Slf4j` |

---

### 案例 3：定义一种单元测试风格

**背景**：你们团队的测试规范是：
- 用 JUnit 5 的 `@Nested` 做测试分组
- 用 AssertJ 做断言（不用 JUnit 自带的 Assertions）
- 用 Mockito 的 `@ExtendWith(MockitoExtension.class)`
- 测试类命名 `XxxServiceTest`，测试方法命名 `should_Xxx_When_Xxx`
- 每个 `@Nested` 类放 `@BeforeEach` 初始化

#### Zero-Shot Prompt：

```markdown
请为 OrderService 写单元测试，测试 createOrder 和 cancelOrder 两个方法。
```

#### Zero-Shot 输出：

```java
@Test
public void testCreateOrder() {
    Order order = new Order();
    when(orderRepository.save(any())).thenReturn(order);
    Order result = orderService.createOrder(new CreateOrderDTO());
    assertEquals(order, result);
}
```

**问题分析**：
- ❌ 用 JUnit 4 的 `@Test` 而非 JUnit 5
- ❌ 用 `assertEquals` 而非 AssertJ 的 `assertThat`
- ❌ 没有 `@Nested` 分组结构
- ❌ 测试方法命名不是 `should_xxx_When_xxx` 风格
- ❌ 没有 `@ExtendWith(MockitoExtension.class)`

#### Few-Shot Prompt：

```markdown
以下是项目中已有的 3 个测试类。请按相同风格为 OrderService 写单元测试。

## 示例1：UserServiceTest
```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserRepository userRepository;
    @Mock
    private PasswordEncoder passwordEncoder;

    @InjectMocks
    private UserService userService;

    @Nested
    @DisplayName("创建用户")
    class CreateUser {

        private CreateUserRequest request;

        @BeforeEach
        void setUp() {
            request = new CreateUserRequest();
            request.setUsername("test");
            request.setPassword("123456");
        }

        @Test
        @DisplayName("用户名已存在时应抛出异常")
        void should_ThrowException_When_UsernameExists() {
            when(userRepository.existsByUsername("test")).thenReturn(true);

            assertThatThrownBy(() -> userService.createUser(request))
                    .isInstanceOf(BusinessException.class)
                    .extracting("errorCode.code")
                    .isEqualTo(2002);
        }

        @Test
        @DisplayName("正常情况应返回用户ID")
        void should_ReturnUserId_When_ValidRequest() {
            when(userRepository.existsByUsername("test")).thenReturn(false);
            when(passwordEncoder.encode("123456")).thenReturn("encoded");
            when(userRepository.save(any(User.class))).thenReturn(new User(1L));

            Long userId = userService.createUser(request);

            assertThat(userId).isEqualTo(1L);
            verify(userRepository).save(any(User.class));
        }
    }
}
```

## 示例2：ProductServiceTest
```java
@ExtendWith(MockitoExtension.class)
class ProductServiceTest {

    @Mock
    private ProductRepository productRepository;

    @InjectMocks
    private ProductService productService;

    @Nested
    @DisplayName("查询商品列表")
    class PageProducts {

        @Test
        @DisplayName("无商品时应返回空列表")
        void should_ReturnEmptyList_When_NoProducts() {
            when(productRepository.findAll(any(Pageable.class)))
                    .thenReturn(new PageImpl<>(List.of()));

            PageResult<ProductVO> result = productService.pageProducts(new PageQuery());

            assertThat(result.getRecords()).isEmpty();
        }
    }
}
```

## 示例3：InventoryServiceTest
```java
@ExtendWith(MockitoExtension.class)
class InventoryServiceTest {

    @Mock
    private InventoryRepository inventoryRepository;

    @InjectMocks
    private InventoryService inventoryService;

    @Nested
    @DisplayName("扣减库存")
    class DeductInventory {

        private DeductRequest request;

        @BeforeEach
        void setUp() {
            request = new DeductRequest();
            request.setSkuId(1L);
            request.setQuantity(5);
        }

        @Test
        @DisplayName("库存不足时应抛出异常")
        void should_ThrowException_When_InsufficientInventory() {
            when(inventoryRepository.findBySkuId(1L))
                    .thenReturn(Optional.of(new Inventory(1L, 1L, 3)));

            assertThatThrownBy(() -> inventoryService.deduct(request))
                    .isInstanceOf(BusinessException.class)
                    .extracting("errorCode.code")
                    .isEqualTo(3002);
        }
    }
}
```

请按上述风格为 OrderService 写单元测试，OrderService 有以下方法：
- createOrder(CreateOrderRequest request): 创建订单，库存不足抛异常，返回订单ID
- cancelOrder(Long orderId): 取消订单，订单不存在抛异常
```

#### Few-Shot 输出：

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock
    private OrderRepository orderRepository;
    @Mock
    private InventoryClient inventoryClient;

    @InjectMocks
    private OrderService orderService;

    @Nested
    @DisplayName("创建订单")
    class CreateOrder {

        private CreateOrderRequest request;

        @BeforeEach
        void setUp() {
            request = new CreateOrderRequest();
            request.setUserId(1L);
            request.setSkuId(1L);
            request.setQuantity(2);
        }

        @Test
        @DisplayName("库存不足时应抛出异常")
        void should_ThrowException_When_InsufficientInventory() {
            when(inventoryClient.checkInventory(1L, 2)).thenReturn(false);

            assertThatThrownBy(() -> orderService.createOrder(request))
                    .isInstanceOf(BusinessException.class)
                    .extracting("errorCode.code")
                    .isEqualTo(3002);
        }

        @Test
        @DisplayName("正常情况应返回订单ID")
        void should_ReturnOrderId_When_ValidRequest() {
            when(inventoryClient.checkInventory(1L, 2)).thenReturn(true);
            when(orderRepository.save(any(Order.class))).thenReturn(new Order(100L));

            Long orderId = orderService.createOrder(request);

            assertThat(orderId).isEqualTo(100L);
            verify(orderRepository).save(any(Order.class));
        }
    }

    @Nested
    @DisplayName("取消订单")
    class CancelOrder {

        @Test
        @DisplayName("订单不存在时应抛出异常")
        void should_ThrowException_When_OrderNotFound() {
            when(orderRepository.findById(1L)).thenReturn(Optional.empty());

            assertThatThrownBy(() -> orderService.cancelOrder(1L))
                    .isInstanceOf(BusinessException.class)
                    .extracting("errorCode.code")
                    .isEqualTo(3001);
        }

        @Test
        @DisplayName("正常情况应成功取消")
        void should_CancelSuccessfully_When_OrderExists() {
            Order order = new Order(1L, OrderStatus.PENDING);
            when(orderRepository.findById(1L)).thenReturn(Optional.of(order));

            orderService.cancelOrder(1L);

            assertThat(order.getStatus()).isEqualTo(OrderStatus.CANCELLED);
        }
    }
}
```

**对比分析**：

| 维度 | Zero-Shot | Few-Shot |
|------|-----------|----------|
| 测试框架 | ❌ JUnit 4 | ✅ JUnit 5 + `@ExtendWith` |
| 断言库 | ❌ `assertEquals` | ✅ AssertJ `assertThat` / `assertThatThrownBy` |
| 测试组织 | ❌ 平铺 | ✅ `@Nested` + `@DisplayName` |
| 方法命名 | ❌ `testCreateOrder` | ✅ `should_ReturnOrderId_When_ValidRequest` |
| Mock 注解 | ❌ 手动 `when` 无注解 | ✅ `@Mock` + `@InjectMocks` |
| 异常验证 | ❌ 无 | ✅ `assertThatThrownBy` + `extracting` 链式验证错误码 |

---

### 案例 4：定义一种 MyBatis XML 写法风格

**背景**：你们项目的 MyBatis XML 有固定套路：
- 必须定义 `<resultMap>` 做字段映射，不用 `autoMapping`
- 必须用 `<sql id="Base_Column_List">` 抽取公共列
- 动态条件用 `<where>` + `<if>` 组合
- 批量操作必须用 `<foreach>`
- 分页查询统一格式：先查 `count`，再查 `list`

#### Zero-Shot Prompt：

```markdown
请写一个 MyBatis XML，实现订单的 CRUD 操作。
表名 order，字段：id, order_no, user_id, status, amount, create_time, update_time。
```

#### Zero-Shot 输出：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
    "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.example.mapper.OrderMapper">
    <select id="selectById" resultType="com.example.entity.Order">
        select * from `order` where id = #{id}
    </select>
</mapper>
```

**问题分析**：
- ❌ 用了 `select *` 而非明确的列列表
- ❌ 没有 `<resultMap>` 映射
- ❌ 没有 `<sql>` 片段复用
- ❌ 分页查询没有 count 阶段
- ❌ 表名用了 MySQL 的反引号（项目规定不用的）

#### Few-Shot Prompt：

```markdown
以下是项目中已有的 3 个 MyBatis XML 示例，请参考它们的风格为订单表写完整的 XML。

## 示例1：UserMapper.xml（部分）
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
    "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.example.mapper.UserMapper">

    <resultMap id="BaseResultMap" type="com.example.entity.User">
        <id column="id" property="id"/>
        <result column="username" property="username"/>
        <result column="phone" property="phone"/>
        <result column="status" property="status"/>
        <result column="create_time" property="createTime"/>
        <result column="update_time" property="updateTime"/>
    </resultMap>

    <sql id="Base_Column_List">
        id, username, phone, status, create_time, update_time
    </sql>

    <select id="selectById" resultMap="BaseResultMap">
        SELECT
        <include refid="Base_Column_List"/>
        FROM user
        WHERE id = #{id}
    </select>

    <select id="selectByCondition" resultMap="BaseResultMap">
        SELECT
        <include refid="Base_Column_List"/>
        FROM user
        <where>
            <if test="username != null and username != ''">
                AND username = #{username}
            </if>
            <if test="status != null">
                AND status = #{status}
            </if>
        </where>
    </select>

    <insert id="insert" parameterType="com.example.entity.User"
            useGeneratedKeys="true" keyProperty="id">
        INSERT INTO user (username, phone, status, create_time, update_time)
        VALUES (#{username}, #{phone}, #{status}, #{createTime}, #{updateTime})
    </insert>
</mapper>
```

## 示例2：ProductMapper.xml（部分）
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
    "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.example.mapper.ProductMapper">

    <resultMap id="BaseResultMap" type="com.example.entity.Product">
        <id column="id" property="id"/>
        <result column="name" property="name"/>
        <result column="price" property="price"/>
        <result column="status" property="status"/>
        <result column="create_time" property="createTime"/>
        <result column="update_time" property="updateTime"/>
    </resultMap>

    <sql id="Base_Column_List">
        id, name, price, status, create_time, update_time
    </sql>

    <select id="selectPage" resultMap="BaseResultMap">
        SELECT
        <include refid="Base_Column_List"/>
        FROM product
        <where>
            <if test="name != null and name != ''">
                AND name LIKE CONCAT('%', #{name}, '%')
            </if>
            <if test="status != null">
                AND status = #{status}
            </if>
        </where>
        ORDER BY create_time DESC
        LIMIT #{offset}, #{size}
    </select>

    <select id="countByCondition" resultType="long">
        SELECT COUNT(1)
        FROM product
        <where>
            <if test="name != null and name != ''">
                AND name LIKE CONCAT('%', #{name}, '%')
            </if>
            <if test="status != null">
                AND status = #{status}
            </if>
        </where>
    </select>
</mapper>
```

## 示例3：InventoryMapper.xml（部分）
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
    "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.example.mapper.InventoryMapper">

    <resultMap id="BaseResultMap" type="com.example.entity.Inventory">
        <id column="id" property="id"/>
        <result column="sku_id" property="skuId"/>
        <result column="quantity" property="quantity"/>
        <result column="version" property="version"/>
        <result column="create_time" property="createTime"/>
        <result column="update_time" property="updateTime"/>
    </resultMap>

    <sql id="Base_Column_List">
        id, sku_id, quantity, version, create_time, update_time
    </sql>

    <update id="deductByIds">
        UPDATE inventory
        SET quantity = quantity - #{quantity}, update_time = NOW()
        WHERE id IN
        <foreach collection="ids" item="id" open="(" separator="," close=")">
            #{id}
        </foreach>
        AND quantity >= #{quantity}
    </update>
</mapper>
```

请按上述风格为订单表（order）写完整的 XML，
包含：resultMap、Base_Column_List、insert、selectById、
selectPage + countByCondition、updateStatusByIds（批量更新状态）。
```

#### Few-Shot 输出：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
    "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.example.mapper.OrderMapper">

    <resultMap id="BaseResultMap" type="com.example.entity.Order">
        <id column="id" property="id"/>
        <result column="order_no" property="orderNo"/>
        <result column="user_id" property="userId"/>
        <result column="status" property="status"/>
        <result column="amount" property="amount"/>
        <result column="create_time" property="createTime"/>
        <result column="update_time" property="updateTime"/>
    </resultMap>

    <sql id="Base_Column_List">
        id, order_no, user_id, status, amount, create_time, update_time
    </sql>

    <select id="selectById" resultMap="BaseResultMap">
        SELECT
        <include refid="Base_Column_List"/>
        FROM `order`
        WHERE id = #{id}
    </select>

    <select id="selectPage" resultMap="BaseResultMap">
        SELECT
        <include refid="Base_Column_List"/>
        FROM `order`
        <where>
            <if test="orderNo != null and orderNo != ''">
                AND order_no = #{orderNo}
            </if>
            <if test="status != null">
                AND status = #{status}
            </if>
            <if test="userId != null">
                AND user_id = #{userId}
            </if>
        </where>
        ORDER BY create_time DESC
        LIMIT #{offset}, #{size}
    </select>

    <select id="countByCondition" resultType="long">
        SELECT COUNT(1)
        FROM `order`
        <where>
            <if test="orderNo != null and orderNo != ''">
                AND order_no = #{orderNo}
            </if>
            <if test="status != null">
                AND status = #{status}
            </if>
            <if test="userId != null">
                AND user_id = #{userId}
            </if>
        </where>
    </select>

    <insert id="insert" parameterType="com.example.entity.Order"
            useGeneratedKeys="true" keyProperty="id">
        INSERT INTO `order` (order_no, user_id, status, amount, create_time, update_time)
        VALUES (#{orderNo}, #{userId}, #{status}, #{amount}, #{createTime}, #{updateTime})
    </insert>

    <update id="updateStatusByIds">
        UPDATE `order`
        SET status = #{status}, update_time = NOW()
        WHERE id IN
        <foreach collection="ids" item="id" open="(" separator="," close=")">
            #{id}
        </foreach>
    </update>
</mapper>
```

**对比分析**：

| 维度 | Zero-Shot | Few-Shot |
|------|-----------|----------|
| 列列表 | ❌ `select *` | ✅ `<sql>` + `<include>` 明确列列表 |
| resultMap | ❌ `resultType` 自动映射 | ✅ `<resultMap>` 显式映射 |
| 分页查询 | ❌ 无 | ✅ `selectPage` + `countByCondition` 双查询模式 |
| 动态 SQL | ❌ 无 | ✅ `<where>` + `<if>` 标准写法 |
| 批量操作 | ❌ 无 | ✅ `<foreach>` 批量更新 |
| SQL 关键字 | ❌ 小写 `select` | ✅ 大写 `SELECT` |
| 表名 | ❌ 加了反引号 | ✅ 与示例一致 |

---

## 四、Few-Shot 示例的选择技巧

很多人在用 Few-Shot 时会问同一个问题：**"选几个？怎么选？"**

### 1. 选 3 个还是 5 个？多了反而不好

**3 个是甜点区。** 根据 OpenAI 和 Anthropic 的官方指南以及社区大量实践：

| 示例数量 | 效果 | 原因 |
|----------|------|------|
| 0 个 | 差 | LLM 靠"猜测"你的偏好，大概率猜错 |
| 1 个 | 一般 | 容易过拟合到这个孤例，不敢泛化 |
| 2 个 | 可以 | 2 个样本刚能确定一条"连线"的方向 |
| **3 个** | **最佳** | 能提取"共性规律"又不至于信息过载 |
| 4-5 个 | 略好但边际递减 | Context 窗口被挤占，留给实际任务的空间变少 |
| 6+ 个 | 开始变差 | 信号被噪声稀释，且容易触发 LLM 的"中间丢失"问题 |

**结论：3 个足矣，不超过 5 个。**

### 2. 示例的多样性 vs 一致性如何平衡？

这是一个核心矛盾：
- **太一致**：LLM 只会"一模一样抄"，遇到变体场景不会泛化
- **太多样**：模式信号被削弱，LLM 抓不住"到底什么是规则、什么是可变的"

**实践法则**：

- **保持一致的部分**：代码结构、注解组合、命名规范、返回类型、日志格式——这些都是你们代码风格的"DNA"
- **制造差异的部分**：业务流程（查询 vs 新增 vs 更新）、参数个数、实体字段——让 LLM 学会区分"必须遵守的格式"和"因场景而异的业务逻辑"

举个例子，案例 1 中的 3 个 Controller 示例：
- ✅ 一致：都用 `@RestControlle` + `@RequiredArgsConstructor` + `@Valid` + `Result<T>` + `@OperateLog`
- ✅ 差异：分别来自不同模块（用户、商品、库存），接口不同（创建、分页、调整），返回值类型不同（`Long`、`PageResult`、`Void`）

这样 LLM 才能学会：**"格式不能变，但业务逻辑随便换"**。

### 3. 示例应该覆盖边界情况吗？

**不需要刻意覆盖所有边界。** 除非你的边界情况需要特定的代码模式来处理。

- 正常流程示例：2 个就够
- 异常流程示例：1 个（如案例 2 中同时展示 `BusinessException` 和 `MethodArgumentNotValidException` 两种捕获）
- 极端边界：一般不需要——LLM 会从正常示例中推导出边界处理方式

**把有限的 Context 留给"风格定义"，而不是"逻辑完备性"。**

---

## 五、高级玩法：动态 Few-Shot——从代码库中自动检索最相关的示例

前面的方案有一个致命问题：**手写示例太累了。**

每换一个任务就得手挑 3 个示例，你让我这种懒人怎么活？

解决方案：**动态 Few-Shot**。核心思路是——根据当前任务，自动从你的代码库中检索最相关的示例代码塞进 Prompt。

### 架构设计

```
┌──────────────┐      ┌──────────────┐      ┌──────────────┐
│  用户输入任务  │ ──▶  │ 向量相似度检索 │ ──▶  │ 构建 Few-Shot │
│ "写一个订单    │      │ 在代码库中找到   │      │ Prompt 自动   │
│  Controller"  │      │ 最相似的3个文件  │      │ 拼入 LLM 请求  │
└──────────────┘      └──────────────┘      └──────────────┘
```

### Java 实现要点

```java
/**
 * 动态 Few-Shot 示例检索器
 * 思路：
 * 1. 提前把项目中所有 Controller 做 Embedding 存入向量数据库
 * 2. 用户输入"写一个订单Controller" → 转为向量 → 检索最相似的3个已有Controller
 * 3. 自动拼接到 Prompt 中发给 LLM
 */
@Service
public class DynamicFewShotService {

    private final VectorStore vectorStore;
    private final EmbeddingService embeddingService;

    /**
     * 根据用户任务描述，自动检索最相关的 N 个示例
     * @param taskDescription 用户的任务描述，如"帮我在订单模块写一个创建订单的Controller"
     * @param topK 检索前 K 个最相似的示例，默认 3
     */
    public List<CodeExample> retrieveRelevantExamples(String taskDescription, int topK) {
        // Step 1: 将任务描述转为向量
        float[] taskVector = embeddingService.embed(taskDescription);

        // Step 2: 在向量数据库中检索最相似的代码文件
        // metadata 中可以存文件路径、模块名、代码类型（Controller/Service/XML 等）
        List<VectorMatch> matches = vectorStore.search(
                taskVector,
                topK,
                Map.of("type", "Controller")  // 只搜 Controller 类型
        );

        // Step 3: 返回最相似的示例代码
        return matches.stream()
                .map(m -> new CodeExample(m.getFilePath(), m.getContent()))
                .toList();
    }

    /**
     * 自动构建 Few-Shot Prompt
     */
    public String buildFewShotPrompt(String taskDescription) {
        List<CodeExample> examples = retrieveRelevantExamples(taskDescription, 3);

        StringBuilder prompt = new StringBuilder();
        prompt.append("以下是项目中已有的 3 个相关代码示例，请参考它们的风格完成以下任务。\n\n");

        for (int i = 0; i < examples.size(); i++) {
            prompt.append("## 示例").append(i + 1).append("：")
                  .append(examples.get(i).getFilePath()).append("\n");
            prompt.append("```java\n");
            prompt.append(examples.get(i).getContent()).append("\n");
            prompt.append("```\n\n");
        }

        prompt.append("任务：").append(taskDescription).append("\n");
        prompt.append("请严格遵循上述示例的编码风格。");

        return prompt.toString();
    }
}
```

### 技术选型建议

| 组件 | 推荐方案 | 备选方案 |
|------|---------|---------|
| Embedding 模型 | text-embedding-3-small（OpenAI）/ bge-large-zh（本地） | Cohere、Voyage AI |
| 向量数据库 | Milvus / Pinecone | pgvector（PostgreSQL 插件）、Chroma |
| 索引策略 | 每个 Java 文件一个 Document，metadata 标记类型 | 按方法粒度切分 |
| 更新机制 | 代码提交时自动重新 Embedding | 定时全量扫描 |

### 动态 Few-Shot 的额外收益

1. **自动适应项目演变**：你们团队改了代码风格，Embedding 自动更新，下次 LLM 自动跟随新风格
2. **跨模块复用**：你在用户模块写的 Controller 风格，自动成为订单模块 AI 生成的参考
3. **新成员零成本上手**：新同事用 AI 辅助写代码，AI 自动学习项目规范，新同事直接获得"老员工风格"的代码

---

## 六、总结：一个公式记住 Few-Shot 的最佳实践

```
Few-Shot 效果 = 示例质量 × 示例数量^(0.5) − 噪声^2
```

翻译成人话：
1. **示例质量远比数量重要**——3 个精选示例 > 10 个随便贴的示例
2. **示例数量有边际递减效应**——3 个是甜点区，超过 5 个开始负优化
3. **噪声是头号杀手**——示例中有不一致的风格、多余的代码、无关的注解，反而让 LLM 困惑

**最后记住一句话：AI 不会"理解"你的规范，但它能完美"模仿"你的模式。所以别写说明书了，直接甩代码给它看。**

---

## 七、下篇预告：ReAct 模式实战

Few-Shot 教 AI"怎么写得像你"，但它还有一个致命短板——**遇到没见过的复杂场景，AI 一旦猜错就一路错到底。**

下一篇文章，我们聊 **ReAct（Reasoning + Acting）模式**：

> 让 AI 像人一样"边想边做"——先分析问题 → 写出方案 → 执行代码 → 看结果 → 修正方案 → 再执行……

结合 Java 项目中的真实场景（复杂业务逻辑拆解、跨多文件重构、疑难 Bug 排查），教你把 LLM 从一个"听话的代码生成器"升级成一个"能自己思考的编程搭档"。

**关注我，不迷路。**

---

*本系列基于 Java 后端研发的真实工作流，追求"看完就能用"的实战体验。如果觉得有帮助，欢迎点赞、收藏、转发三连。*
