# Cursor Composer 模式：多文件联动修改的正确姿势，别再一个一个文件改了

> 需求改了接口入参，结果你要改 Controller、Service、DTO、测试、文档 5 个文件，改完还漏了一个——这种痛，Composer 三句话治好。

---

## 一、开篇痛点：你改的不是代码，是连锁反应

下午两点，你正喝着咖啡看 PRD，产品经理在群里 @你：

> "订单接口加一个 `couponCode` 字段，不用校验，可为空。跟紧上，今天下班前要上灰度。"

你心里默默盘算了一下：
- `OrderEntity.java` —— 加字段、加 getter/setter（或用 Lombok 加一行注解）
- `OrderDTO.java` —— 同步加字段
- `CreateOrderRequest.java` —— 接收参数加字段
- `OrderController.java` —— Swagger 注解更新
- `OrderService.java` —— 赋值逻辑补一行
- `OrderMapper.xml` —— 如果没开自动映射，SQL 里也得加
- `OrderControllerTest.java` —— 测试用例里得把新字段带入断言
- `接口文档.md` —— 或者 YApi/Swagger 上得补字段描述

这还不算——如果你用的是 MyBatis 手写 XML 的分层项目、DTO 和 VO 还是分开建的、前端还等着你更新 TS 类型定义……保守估计 **8 个文件起步**。

你深吸一口气，打开 IDE，准备一个一个文件改过去。两个小时后改完，提测，CI 挂了——漏改了 `OrderMapper.xml` 里的 `resultMap`。

**这就是"多文件联动修改"的日常。改一个字段容易，确保所有关联文件同步更新才是真正的体力活。**

而今天这篇文章，我要告诉你：**从今天起，这活儿交给 Cursor Composer。你只需要用自然语言描述"改什么"，Composer 自动帮你找出所有需要改的文件，一个一个改好，连测试用例的断言都给你更新了。**

先给你看一个真实对比。

### 1.1 Before/After：同一需求，两种人生

**Before（传统方式）：**

```
需求：给图书实体加一个 publisher（出版社）字段

操作步骤：
1. Book.java                   →  加 private String publisher;                          （1分钟）
2. BookDTO.java               →  加 private String publisher;                          （1分钟）
3. BookRequest.java           →  加 private String publisher; 加 @Schema 注解           （2分钟）
4. BookServiceImpl.java       →  createBook 方法里加 .publisher(request.getPublisher()) （1分钟）
5. BookServiceImpl.java       →  updateBook 方法里加 update逻辑                        （2分钟）
6. BookController.java        →  查找 API 文档注解是否自动生成，手动补描述               （1分钟）
7. BookControllerTest.java    →  构造请求时加 .publisher("机械工业出版社")                （2分钟）
8. BookServiceTest.java       →  修改 mock 数据、更新断言                                （2分钟）
9. BOOK_API.md                 →  打开文档，在请求/响应示例中补字段                        （3分钟）
10. Postman Collection         →  导入 JSON，手动改示例请求体                             （2分钟）
11. BookMapper.xml（如有）     →  resultMap 补 <result column="publisher" .../>          （2分钟）
12. 全局搜索确认没有遗漏        →  grep publisher 全项目扫一遍                            （2分钟）
13. 跑测试                      →  mvn test，期望全绿                                    （1分钟）

总计：保守 20 分钟。如果中间发现遗漏，再来一轮。
```

**After（Composer 模式）：**

```text
你（在 Composer 输入框里敲下一段 Prompt）：

"在图书管理系统中，给 Book 实体增加一个 publisher 字段（String 类型，出版社名称，可为空）。
请帮我同步更新所有关联文件，包括 Entity、DTO、Request、Service、Controller、单元测试、
API 文档和 Postman Collection。"

Composer 输出（2 分钟内完成）：

  ✓ Book.java                          — 新增 publisher 字段 + @Column 注解
  ✓ BookDTO.java                       — 新增 publisher 字段 + Swagger 描述
  ✓ BookRequest.java                   — 新增 publisher 字段 + @NotBlank 校验（可为空，不加）
  ✓ BookServiceImpl.java               — createBook() 补字段赋值
  ✓ BookServiceImpl.java               — updateBook() 补更新逻辑
  ✓ BookController.java                — 检查无误（DTO 已含，无需改动）
  ✓ BookControllerTest.java            — 测试数据补 publisher，断言更新
  ✓ BookServiceTest.java               — Mock 数据更新，verify 断言适配
  ✓ BOOK_API.md                        — 请求/响应 JSON 示例补 publisher 字段
  ✓ BookManager.postman_collection.json — body 示例补 publisher
  ✓ BookMapper.xml                     — resultMap 补 <result column="publisher" .../>

修改了 11 个文件，比传统方式多检查了 2 个你容易遗漏的文件。
而且 Composer 在最后主动说："已确认所有关联文件同步完成，无异样遗漏。"
```

看到了吗？不是省几分钟的问题，是**从根本上解决了"人类大脑记不住所有关联文件"这个生理缺陷**。

---

## 二、Composer 到底是什么？为什么它能做多文件联动？

在进入实战案例之前，先用 30 秒把概念对齐。

Cursor 里有三种 AI 交互模式：
- **Ctrl+K（行内编辑）**——改当前文件的一小段代码，类似 Copilot 的 Inline Chat
- **Ctrl+L（Chat 面板）**——右侧对话窗，可以 `@file` 引用上下文，一问一答
- **Ctrl+I（Composer 模式）**——**本文的主角**，全项目感知、多文件生成与修改、可执行终端命令

Composer 的核心能力在于它不只是"看到"你当前打开的文件，而是会**主动扫描整个项目**，理解文件之间的引用关系（比如 DTO 被 Controller 引用、Entity 被 Service 引用），然后制定一个修改计划，按依赖顺序逐个文件修改。

打个比方：
- Ctrl+K 是"帮我改这一行"
- Ctrl+L 是"帮我解答这个问题"
- Composer 是"帮我完成这个需求，涉及多少文件你来定，改完你自己检查"

---

## 三、实战案例 1：REST API 增加一个字段（全链路联动）

> 场景：线上图书管理系统，给 `Book` 加上 `publisher`（出版社）字段，要求从数据库到 API 返回全链路同步。

### 3.1 项目背景

这是一个典型的 Spring Boot 项目，分层结构如下：

```
book-manager/
├── entity/Book.java                  ← JPA Entity，@Entity + @Table
├── dto/BookDTO.java                  ← 返回给前端的 DTO
├── dto/BookRequest.java              ← 前端提交的 Request Body
├── controller/BookController.java    ← REST Controller
├── service/BookService.java          ← 接口
├── service/impl/BookServiceImpl.java ← 实现
├── mapper/BookMapper.java            ← MyBatis Mapper（部分项目混用 JPA + MyBatis）
├── mapper/BookMapper.xml             ← SQL 映射文件
├── test/.../BookControllerTest.java  ← 集成测试
├── test/.../BookServiceTest.java     ← 单元测试
├── BOOK_API.md                       ← 手写的 API 文档
└── postman/BookManager.postman_collection.json
```

### 3.2 关键：怎么写这个 Prompt？

很多同学第一次用 Composer 做多文件修改，Prompt 写得像跟 Siri 说话：

> ❌ 错误示范："帮我加个 publisher 字段"

—— 这种 Prompt 下去，Composer 要么只改了 Entity，要么不知道该加什么校验规则，要么漏了一堆文件。

正确的 Prompt 应该包含**四个要素**：要改什么 → 涉及范围 → 字段约束 → 验证要求。

> ✅ 正确 Prompt：

```
在图书管理系统中，给 Book 实体增加一个 publisher（出版社）字段。

字段约束：
- 类型：String
- 可为空（nullable = true）
- 长度限制：最大 100 字符
- 数据库列名：publisher
- API 文档描述："出版社名称"

涉及的文件范围：
- Entity: Book.java
- DTO: BookDTO.java（响应）和 BookRequest.java（请求）
- Service: BookServiceImpl.java（createBook 和 updateBook 两个方法）
- Controller: BookController.java（Swagger 注解按需更新）
- 单元测试: BookControllerTest.java、BookServiceTest.java
- 文档: BOOK_API.md
- Postman: BookManager.postman_collection.json
- MyBatis: BookMapper.xml（如果存在）

要求：
1. 先列出你打算修改的所有文件清单，让我确认
2. 依次修改，每改完一个文件标注状态
3. 全部改完后，做一次全项目检查，确认没有遗漏
```

### 3.3 Composer 的实际执行过程

把上面这段 Prompt 敲进 Composer（`Ctrl+I`），回车。Composer 的第一步输出通常是这样的：

```text
我将按以下计划修改：

1. Book.java              — 新增 publisher 字段（@Column，nullable=true，length=100）
2. BookDTO.java           — 新增 publisher 字段 + @Schema(description="出版社名称")
3. BookRequest.java       — 新增 publisher 字段（可为空，不加 @NotBlank）
4. BookServiceImpl.java   — createBook() 中补 .publisher(request.getPublisher())
                          — updateBook() 中补更新逻辑
5. BookControllerTest.java  — 构造测试请求时补 publisher 字段
6. BookServiceTest.java     — Mock 数据补 publisher，verify 断言更新
7. BOOK_API.md            — 响应/请求示例中补 publisher
8. BookManager.postman_collection.json — body 示例补字段
9. BookMapper.xml         — resultMap 补 <result column="publisher" property="publisher"/>

共 9 个文件。确认后我开始执行。需要调整吗？
```

这一步非常关键！**一定要看一眼这个清单**。有时候 Composer 会多改一些你不想改的文件（比如把 IDE 配置文件也拉进去了），或者漏了一些你想改但没找到的文件（比如你的项目里还有个 `BookExcelDTO.java` 用于导出报表）。

确认无误后，回复"执行"或"继续"。Composer 会逐个文件进行修改。

### 3.4 修改过程中的细节

**Book.java（Entity）—— Composer 的修改：**

```java
// before
@Column(name = "price")
private BigDecimal price;

// after
@Column(name = "price")
private BigDecimal price;

@Column(name = "publisher", length = 100, nullable = true)
private String publisher;
```

注意 Composer 自动加了 `@Column` 注解，并恰当地设置了 `nullable = true` 和 `length = 100`——因为它读了你 Prompt 里的约束条件。

**BookDTO.java —— Composer 的修改：**

```java
// before
@Schema(description = "图书价格")
private BigDecimal price;

// after
@Schema(description = "图书价格")
private BigDecimal price;

@Schema(description = "出版社名称")
private String publisher;
```

如果你用的是 Swagger 2（`@ApiModelProperty`）而不是 Swagger 3（`@Schema`），Composer 会根据你项目中已有的注解风格自动适配。它不会突然给你换一套注解体系。

**BookServiceImpl.java —— Composer 的修改（createBook 方法）：**

```java
// before
Book book = Book.builder()
    .title(request.getTitle())
    .author(request.getAuthor())
    .price(request.getPrice())
    .build();

// after
Book book = Book.builder()
    .title(request.getTitle())
    .author(request.getAuthor())
    .price(request.getPrice())
    .publisher(request.getPublisher())
    .build();
```

**BookControllerTest.java —— Composer 的修改（测试数据构造）：**

```java
// before
BookRequest request = new BookRequest();
request.setTitle("深入浅出Java虚拟机");
request.setAuthor("周志明");
request.setPrice(new BigDecimal("99.00"));

// after
BookRequest request = new BookRequest();
request.setTitle("深入浅出Java虚拟机");
request.setAuthor("周志明");
request.setPrice(new BigDecimal("99.00"));
request.setPublisher("机械工业出版社");
```

它不仅加上了字段，还智能地补了一个合理的测试值'机械工业出版社'——而不是给个空字符串或不填。

**BookMapper.xml —— Composer 的修改：**

```xml
<!-- before -->
<resultMap id="BaseResultMap" type="com.example.bookmanager.entity.Book">
    <id column="id" property="id"/>
    <result column="title" property="title"/>
    <result column="author" property="author"/>
    <result column="price" property="price"/>
</resultMap>

<!-- after -->
<resultMap id="BaseResultMap" type="com.example.bookmanager.entity.Book">
    <id column="id" property="id"/>
    <result column="title" property="title"/>
    <result column="author" property="author"/>
    <result column="price" property="price"/>
    <result column="publisher" property="publisher"/>
</resultMap>
```

如果你的 MyBatis 开启了 `map-underscore-to-camel-case: true`，且字段名本身就不含下划线，那 resultMap 不改也没事。但 Composer **宁可多改，不会少改**——这对你来说是好事，因为多改不会挂，少改可能出 bug。

### 3.5 全部改完后

Composer 最后会输出一个汇总：

```text
✓ 修改完成。共涉及 9 个文件，均无语法错误。

建议下一步操作：
1. 运行 mvn test 确认所有测试通过
2. 如果数据库表已有生产数据，记得执行 ALTER TABLE 添加 publisher 列
```

这时候你跑一下 `mvn test`——如果绿灯全亮，恭喜，需求完成。从提出到改完，前后不超过 3 分钟。

---

## 四、实战案例 2：重构异常处理体系（20+ 文件联动）

> 场景：项目初期用了 `RuntimeException` 到处抛，现在要统一替换为自定义 `BusinessException`，涉及 20+ 个文件。

### 4.1 背景还原

很多项目在 MVP 阶段，开发为了快，异常处理写成这样：

```java
// 各种地方都是
throw new RuntimeException("订单不存在");
throw new RuntimeException("库存不足");
throw new RuntimeException("优惠券已过期");
```

等到项目上了生产，问题来了：
- 前端拿到的永远是 500，无法区分具体错误
- 日志里一堆 `RuntimeException` 堆栈，根本看不出业务错误还是系统故障
- 想要统一加错误码，发现得把所有 `new RuntimeException` 改一遍

这时候你决定引入 `BusinessException`：

```java
public class BusinessException extends RuntimeException {
    private final int code;
    private final String message;

    public BusinessException(int code, String message) {
        super(message);
        this.code = code;
        this.message = message;
    }
}
```

然后你发现——项目里 `grep -r "new RuntimeException"` 出来的结果是 **27 个文件，43 处**。

### 4.2 Composer Prompt 设计

面对这种大范围重构，你的 Prompt 需要更精确，重点关注**不该改的地方**。

```
项目路径：@folder src/main/java

任务：将项目中所有业务逻辑层的 RuntimeException 替换为 BusinessException。

具体规则：
1. Controller 层抛出的 RuntimeException 不改（那是全局异常处理兜底的）
2. Service 层、工具类中的 RuntimeException 全部替换
3. 替换模式：
   - throw new RuntimeException("xxx")
     → throw new BusinessException(ErrorCode.UNKNOWN.getCode(), "xxx")
   - throw new RuntimeException(e)
     → throw new BusinessException(ErrorCode.INTERNAL_ERROR.getCode(), e.getMessage(), e)
4. ErrorCode 是一个已有的枚举类，路径在 constant/ErrorCode.java，请先阅读它以了解可用的错误码
5. 如果某个 RuntimeException 有明显的业务语义（比如"库存不足"），请使用精确的错误码而不是 UNKNOWN

执行流程：
Step 1: 先扫描 src/main/java 下所有含 RuntimeException 的文件，列出完整的文件清单
Step 2: 逐个文件分析，判断哪些是 Controller 层的（不改），哪些是 Service/Utils 层的（改）
Step 3: 按修改计划逐一替换
Step 4: 全部完成后，运行 mvn compile 确认编译通过

注意：
- 不要修改 BusinessException 类本身
- 不要修改 ErrorCode 枚举类
- 不要修改测试文件中的 RuntimeException（测试里硬编码 exception 是正常的）
```

### 4.3 执行过程

Composer 的 Step 1 输出：

```text
扫描完成，src/main/java 下共发现 43 处 new RuntimeException：

com/example/bookmanager/controller/
  - BookController.java                    2 处  ← Controller层，不改
  - OrderController.java                   1 处  ← Controller层，不改

com/example/bookmanager/service/impl/
  - BookServiceImpl.java                   5 处  ← 需要修改
  - OrderServiceImpl.java                  8 处  ← 需要修改
  - UserServiceImpl.java                   3 处  ← 需要修改

com/example/bookmanager/util/
  - FileUtil.java                          2 处  ← 需要修改
  - DateUtil.java                          1 处  ← 需要修改

com/example/bookmanager/manager/
  - InventoryManager.java                  4 处  ← 需要修改

...（省略中间输出）

需要修改的文件：21 个。不修改的文件：3 个（Controller 层）。
```

确认后开始执行。每改完一个文件，Composer 会在右侧 Diff 视图展示 Before/After 对比。

**修改示例（OrderServiceImpl.java 中的一段）：**

```java
// before —— 随处可见的 RuntimeException
public Order createOrder(CreateOrderRequest request) {
    Product product = productRepository.findById(request.getProductId())
        .orElseThrow(() -> new RuntimeException("商品不存在"));

    if (product.getStock() < request.getQuantity()) {
        throw new RuntimeException("库存不足，当前库存：" + product.getStock());
    }

    User user = userRepository.findById(request.getUserId())
        .orElseThrow(() -> new RuntimeException("用户不存在"));

    // ... 下单逻辑
}

// after —— 替换为 BusinessException + 语义明确的错误码
public Order createOrder(CreateOrderRequest request) {
    Product product = productRepository.findById(request.getProductId())
        .orElseThrow(() -> new BusinessException(
            ErrorCode.PRODUCT_NOT_FOUND.getCode(), "商品不存在"));

    if (product.getStock() < request.getQuantity()) {
        throw new BusinessException(
            ErrorCode.INSUFFICIENT_STOCK.getCode(),
            "库存不足，当前库存：" + product.getStock());
    }

    User user = userRepository.findById(request.getUserId())
        .orElseThrow(() -> new BusinessException(
            ErrorCode.USER_NOT_FOUND.getCode(), "用户不存在"));

    // ... 下单逻辑
}
```

注意 Composer 的表现：
1. 自动区分了"商品不存在 → PRODUCT_NOT_FOUND"、"库存不足 → INSUFFICIENT_STOCK"、"用户不存在 → USER_NOT_FOUND"，而不是一律用 UNKNOWN
2. 保留了原始的 message 信息（"库存不足，当前库存：" + product.getStock()）
3. 保持了原有的代码缩进和格式风格

**全部改完后，Composer 主动执行了 `mvn compile`：**

```text
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
所有 21 个文件编译通过，无语法错误。修改完成。
```

### 4.4 这个案例的核心启示

- **大范围重构是 Composer 的甜点区** —— 规则越明确，它的表现越稳定。如果你有一个清晰的替换规则（A→B），Composer 执行起来比人工快 10 倍，而且不会"中间改疲劳了就漏两个文件"。
- **一定要圈定范围** —— `不要修改 Controller 层`、`不要修改测试文件`，这些"不改"指令和"改"指令同样重要。
- **Step 1 先扫描清单这个习惯，救了我无数次** —— 我至少有 3 次在"确认清单"这一步发现 Composer 扫进了不该改的文件（比如 `pom.xml` 里的插件配置中也有 `RuntimeException` 字样），及时叫停调整。

---

## 五、实战案例 3：数据库字段类型变更（Long → String）

> 场景：历史遗留问题，用户表的 `id` 用的是自增 Long，现在要迁到雪花算法，需要改成 String。涉及 Entity、Mapper、Service、Controller、DTO、前端接口文档。

这个案例的复杂度比前两个高一个量级，因为**类型变更意味着所有使用该类型的地方都受影响**。Composer 能不能正确处理这个问题？

### 5.1 影响范围分析

```
UserId: Long → String

直接影响：
├── entity/User.java                    — id 字段类型：Long → String
├── dto/UserDTO.java                    — id 字段类型：Long → String
├── dto/UserRequest.java                — 如果 request 中有 userId，也要改
├── mapper/UserMapper.java              — 方法签名中的 id 参数类型：Long → String
├── mapper/UserMapper.xml               — parameterType 和 resultMap 检查
├── service/UserService.java            — 接口中的 id 参数：Long → String
├── service/impl/UserServiceImpl.java   — 实现类中的 id 参数：Long → String
├── controller/UserController.java      — @PathVariable Long id → String id
├── test/.../UserControllerTest.java    — 测试数据中的 id 类型
├── test/.../UserServiceTest.java       — Mock 中的 id 类型
└── BOOK_API.md                         — 接口文档中的 id 类型描述

间接影响：
├── Order 表中如果有外键 userId，也要改（但这通常是关联查询，影响在主表）
├── 前端 TS 类型定义（如果有的话）
└── 其他引用了 UserService 的 Service
```

### 5.2 Composer Prompt

```text
项目中 User 实体当前使用自增 Long 作为主键 id，现需要迁到雪花算法，将 id 类型从 Long 改为 String。

请修改以下所有关联文件：

@file src/main/java/com/example/bookmanager/entity/User.java
  — 将 id 字段类型从 Long 改为 String
  — 移除 @GeneratedValue 注解（雪花算法由应用层生成 id）
  — id 生成策略改为 ASSIGNED 或在 Service 层手动赋值

@folder src/main/java/com/example/bookmanager/
  — 同步修改所有引用了 User.id 的文件，包括：
    - dto/UserDTO.java
    - mapper/UserMapper.java + UserMapper.xml
    - service/UserService.java + UserServiceImpl.java
    - controller/UserController.java
    - 其他任何使用了 User.getId() 或 user.setId(Long) 的地方

@folder src/test/
  — 同步修改单元测试中的测试数据

@file BOOK_API.md
  — 同步修改接口文档中的 id 字段描述

注意事项：
1. 先扫描全部影响范围，列出文件清单，确认后再执行
2. @PathVariable、@RequestParam 中的 id 参数类型也要改
3. 如果某个方法的参数 xxxId 使用了 Long，请判断它是否关联 User.id
4. 测试文件中的 Long.valueOf("xxx") 或 123L 字面量如果引用了 userId，需要改为字符串
5. MyBatis Mapper XML 中的 parameterType 如果写了 Long，也需要更新
```

### 5.3 执行实录

Composer 的扫描结果：

```text
影响范围分析：

核心修改（8个文件）：
1. entity/User.java                     — id: Long → String，移除 @GeneratedValue
2. dto/UserDTO.java                     — id: Long → String
3. mapper/UserMapper.java               — findById(String id), deleteById(String id)
4. mapper/UserMapper.xml                — 检查 parameterType，SQL 中的 #{id} 无需改动
5. service/UserService.java             — 接口签名中的 Long id → String id
6. service/impl/UserServiceImpl.java    — 实现中所有 Long id → String id
7. controller/UserController.java       — @PathVariable Long id → String id
8. BOOK_API.md                          — 文档中 id 类型描述

关联修改（3个文件）：
9. OrderServiceImpl.java                — 调用 userService.findById(order.getUserId())
                                          参数类型适配（可能已是 String）
10. UserControllerTest.java             — 测试中的 Long id → String id
11. UserServiceTest.java                — Mock 中的 Long id → String id

是否需要修改（需确认）：
? Order.java                            — userId 字段目前是 Long，是否同步改为 String？
? 前端 TS 类型定义文件                   — 项目中未找到

共 11~13 个文件需要修改。请确认是否将 Order.userId 也改为 String？
```

这里暴露了一个很有意思的细节：**Composer 会自动发现关联影响，并主动向你确认。** 比如它发现 `OrderServiceImpl` 中调用了 `UserService` 的方法，参数类型会受影响。

### 5.4 修改细节

**User.java（核心变更）：**

```java
// before
@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    // ... 其他字段
}

// after
@Entity
@Table(name = "users")
public class User {

    @Id
    private String id;

    @PrePersist
    public void prePersist() {
        if (this.id == null) {
            this.id = SnowflakeIdGenerator.generate();
        }
    }

    // ... 其他字段
}
```

Composer 不仅帮你改了类型，还贴心地加了一个 `@PrePersist` 钩子来处理雪花 ID 的自动生成。如果你不想用 `@PrePersist` 的方式，在 Prompt 里说明即可。

**UserController.java：**

```java
// before
@GetMapping("/{id}")
public ApiResponse<UserDTO> getUserById(@PathVariable Long id) {
    return ApiResponse.success(userService.getUserById(id));
}

// after
@GetMapping("/{id}")
public ApiResponse<UserDTO> getUserById(@PathVariable String id) {
    return ApiResponse.success(userService.getUserById(id));
}
```

注意 `@PathVariable` 的类型从 `Long` 变成了 `String`，Spring MVC 的路径变量绑定会自动适配，所以路由本身无需修改。

**UserControllerTest.java：**

```java
// before
@Test
void shouldGetUserById() throws Exception {
    mockMvc.perform(get("/api/users/{id}", 1L))
        .andExpect(status().isOk())
        .andExpect(jsonPath("$.data.id").value(1));
}

// after
@Test
void shouldGetUserById() throws Exception {
    mockMvc.perform(get("/api/users/{id}", "1234567890123456789"))
        .andExpect(status().isOk())
        .andExpect(jsonPath("$.data.id").value("1234567890123456789"));
}
```

测试用例中的 `1L` 被替换成一个雪花算法风格的 19 位字符串，并且 `jsonPath` 的 `value(1)` 改成了 `value("1234567890123456789")`。这种细节，人工改很容易漏——特别是 `jsonPath` 那行，编译器不会报错，但测试跑起来会挂。

---

## 六、Composer Prompt 编写核心技巧

掌握了上面三个案例，你应该对 Composer 的能力有了直观认知。接下来我总结一下让 Composer 稳定输出的 Prompt 编写套路。

### 6.1 "先调研 - 后修改 - 再验证"三段式指令

这是我在几十次使用后总结出来的黄金模板：

```
## 阶段一：调研
请先扫描项目，列出所有受影响的文件和具体的修改位置。不要直接改代码，先给我一份修改计划。

## 阶段二：修改
按以下规则逐个文件修改：
- 规则 1：xxx
- 规则 2：xxx
- 不改的地方：xxx

## 阶段三：验证
全部改完后，执行 mvn compile（或 npm run build 等），确保编译通过。如有报错，请修复。最后给一份修改汇总清单。
```

为什么要分三段？

**阶段一的价值**在于"先看菜再下单"。AI 对项目的理解可能和你预期的不同，如果在"修改计划"阶段就发现问题，直接纠正比改完再回退省 10 倍时间。

**阶段二的规则必须写得像法律条文**——模糊的地方就是 Composer"自由发挥"的地方，而 AI 的自由发挥约等于随机行为。

**阶段三的验证不是可选项**——Composer 写出来的代码虽然大概率能编译通过（毕竟它见过地球上最多的 Java 代码），但在你的项目特定环境下，可能会有 import 缺失、类名拼写偏差等问题。让它在同一个对话中修复，保持上下文连续性。

### 6.2 如何限制 Composer 的修改范围

Composer 最让人头疼的问题之一是"越改越嗨"——你让它加个字段，它顺手帮你重构了整个 Service。控制范围有四招：

**第一招：给文件白名单**

```text
只修改以下文件，其他文件一律不动：
- entity/Book.java
- dto/BookDTO.java
- dto/BookRequest.java
- service/impl/BookServiceImpl.java
- test/.../BookControllerTest.java
```

**第二招：给文件黑名单**

```text
以下文件禁止修改：
- pom.xml
- application.yml
- 任何配置文件
- test/ 目录下的非 Book 相关测试
- .cursorrules、.gitignore 等工程配置文件
```

**第三招：给出风格约束**

```text
代码风格要求：
- 不改变现有的代码组织结构（不要新建 package、不要移动文件）
- 不引入新的依赖库（不要在 pom.xml 中加东西）
- 不修改已有的方法签名（只新增字段/修改逻辑，不改变返回值类型/参数列表）
- 使用项目已有的 Lombok 注解风格（@Data、@Builder 等）
```

**第四招：设定"不需要修改"的兜底规则**

```text
如果你不确定某个文件是否需要修改，默认不修改，并在报告中标注"不确定"。由我来决定。
```

这四招组合用好，Composer 基本不会"越界"。

### 6.3 @folder 和 @file 的精确用法

Composer 支持用 `@` 符号引用文件和目录，这和 Chat 模式的用法一样，但在 Composer 中效果差异巨大。

**`@folder` —— 告诉 Composer 约束范围**

```text
请在 @folder src/main/java/com/example/bookmanager/service 下，将所有
RuntimeException 替换为 BusinessException。不要修改该目录外的任何文件。
```

这相当于对 Composer 说："你的工作范围就是 service 这个包，别的地方别碰。"

**`@file` —— 给 Composer 提供参考样本**

```text
@file src/main/java/com/example/bookmanager/constant/ErrorCode.java
@file src/main/java/com/example/bookmanager/exception/BusinessException.java

参考以上两个文件，将 service/ 目录下的 RuntimeException 替换为 BusinessException。
```

这相当于给 Composer 看了两个"参考答案"，它就会按照你已有的编码风格来写。

**组合使用最佳实践：**

```text
@folder src/main/java/com/example/bookmanager/   （工作范围）
@file src/main/java/com/example/bookmanager/constant/ErrorCode.java   （参考样本）

任务：xxx

请不要修改 @folder 范围以外的文件。
```

### 6.4 一些我踩出来的 Prompt 小技巧

**技巧 1：不要让 Composer "猜"字段名**

> ❌ 坏 Prompt："给 User 加上手机号的字段"
> ✅ 好 Prompt："给 User 实体新增 `phone` 字段，类型 String，可为空，数据库列名 `phone`，API 描述"用户手机号""

—— AI 会猜，但猜的结果不一定是你想要的。字段名、类型、约束，能明确的就明确。

**技巧 2：用 "before/after" 格式来描述代码转换**

当你想做模式替换时（比如案例 2 的 RuntimeException → BusinessException），直接在 Prompt 里给出 Before/After 示例，比文字描述效果好得多：

```text
替换模式：
Before:
throw new RuntimeException("商品不存在")
After:
throw new BusinessException(ErrorCode.PRODUCT_NOT_FOUND.getCode(), "商品不存在")
```

**技巧 3：加上"最终检查"提示**

```text
全部改完后，请做一次全项目检查，确认以下事项：
1. 所有目标文件的修改是否一致（比如 DTO 和 Entity 的字段名、类型是否对齐）
2. 是否有多余的 import 残留
3. 测试文件中的断言是否与新结构匹配
4. 是否有其他文件引用了修改前的方法签名但未同步更新
```

这句"最终检查"经常能揪出遗漏——我自己试过很多次，漏改的文件有 30% 是在"最终检查"阶段被 Composer 自己发现并补上的。

---

## 七、Composer 常见翻车场景及补救

AI 工具不是万能的，Composer 也有翻车的时候。下面是我真实遇到过的翻车场景和对应的补救方案。

### 7.1 翻车场景 1：改错了，怎么回退？

**场景**：让 Composer 重构异常处理，它把 Controller 里 `@ExceptionHandler` 的 `RuntimeException` 也改了，导致全局异常处理失效。

**抢救方案**：

**方案 A（推荐）：用 Git 回退**

Composer 每次修改都会自动生成一个 commit（如果你在 Cursor 的 Git 面板里开启了 Auto Commit，默认是文件级 diff）。最直接的方式：

```bash
# 回退最近一次 Composer 的修改
git checkout -- src/main/java/com/example/bookmanager/controller/GlobalExceptionHandler.java
```

如果你用了 Cursor 的 "Checkpoint" 功能（`Cmd+Shift+P` → "Cursor: Restore Checkpoint"），可以直接恢复到 Composer 执行前的快照。

**方案 B（在 Composer 对话中纠正）**

与其用 Git 回退整个对话，不如直接在同一个 Composer 会话中追加指令：

```text
刚才的修改有问题。GlobalExceptionHandler.java 中的 RuntimeException 不应该被替换。
请将该文件恢复到修改前的状态，并确保不再修改它。
```

Composer 会基于上下文理解你的纠正意图，重新处理。

### 7.2 翻车场景 2：改了一半停了

**场景**：给 20 个文件做字段批量替换，Composer 改到第 12 个文件时跳了个"Token limit reached"或"Request interrupted"。

**抢救方案**：

**不要关闭这个 Composer 会话！** 在同一个对话框中输入：

```text
你刚才的任务没有完成。请继续从上次中断的地方继续。已完成修改的文件不要重复修改。
```

Composer 会回忆上下文，从第 13 个文件开始继续。如果你关掉了会话重新开一个，Composer 就不知道哪些已经改过了，可能会重复修改或遗漏。

如果 Token 确实不够了，分两次完成：

```text
本次只修改以下 10 个文件（列出文件清单）：
1. xxx
2. xxx
...
完成后告诉我，我再开下一个会话继续。
```

### 7.3 翻车场景 3：改完编译不通过

**场景**：Composer 修改了 `UserServiceImpl` 的方法签名（从 `Long` 改成 `String`），但忘了改另一个 Service 调用该方法的地方。

**抢救方案**：

```text
刚才的修改后 mvn compile 报错如下：
[ERROR] xxx cannot be converted to java.lang.String

请扫描 src/main/java 下所有调用了 UserService.findById() 的地方，
确认参数类型是否匹配，不匹配的请修复。
```

更好的做法是在 Prompt 阶段就加上验证指令（参见 6.1 的三段式模板），让 Composer 主动跑编译并修复，而不是改完了让你去发现。

### 7.4 翻车场景 4：Composer 自己"发明"了不存在的方法

**场景**：Composer 在代码里写了一个 `snowflakeIdGenerator.nextId()`，但你项目里根本没这个类。

**抢救方案**：

```text
你刚才在 User.java 中使用了 SnowflakeIdGenerator.generate()，但项目中没有这个类。
请在项目中搜索已有的 ID 生成工具类。如果没有，使用 UUID.randomUUID().toString() 替代。
```

这是 AI 的一个常见毛病——会假设一些"理所当然"的依赖存在。**SoP（Standard Operating Procedure）建议：检查 Composer 生成的代码中是否有陌生的方法调用或类引用。**

### 7.5 翻车场景 5：文件数量太多，Composer "挑着改"

**场景**：40 个文件需要改字段类型，Composer 改了 28 个就声称"完成了"。

这通常是因为 Prompt 中的约束范围和 Composer 理解的文件结构有差异。比如你说"service 目录下"，但 Composer 不知道 `util/` 目录下的文件也引用了 User.id。

**抢救方案**：

不要依赖 Composer 的判断，**自己先跑一次全局搜索**：

```bash
# 先用 grep 找出所有引用
rg "\.getId\(\)" src/main/java/ -l
rg "setId\(Long" src/main/java/ -l
```

然后把输出结果作为文件名单交给 Composer：

```text
请修改以下所有文件中的 Long id → String id：
@file src/main/java/.../UserService.java
@file src/main/java/.../UserServiceImpl.java
@file src/main/java/.../InventoryManager.java
...（列出全部文件）
```

**人定范围，AI 执行**——这是目前最稳妥的分工模式。

---

## 八、Composer vs Chat vs Inline：什么时候该用谁？

最后一起来做个总结决策矩阵：

| 场景 | 推荐模式 | 原因 |
|------|----------|------|
| 给当前方法加一行校验逻辑 | Ctrl+K（Inline） | 单文件单点修改，无需上下文 |
| 问"这个项目的认证逻辑在哪？" | Ctrl+L（Chat） | 需要回答，不需要改代码 |
| 新增一个 CRUD 接口（Controller+Service+DTO+Test） | Composer | 涉及多个文件的新增 |
| 给 Entity 加字段，同步更新全链路 | Composer | 多文件联动修改 |
| 全局替换 RuntimeException → BusinessException | Composer | 大范围重构，规则明确 |
| 修复一个单文件 Bug | Ctrl+K | 用炮打蚊子不值得 |
| 追加单元测试 | Composer | 需要理解现有类的全部方法 |
| 写一个新的 Mapper XML | Composer | 需要同时参考 Entity 和 Mapper 接口 |

一句话总结：**单文件小改 → Ctrl+K，有疑问要问 → Ctrl+L，多文件联动 → Composer。**

---

## 九、总结：你不再是"改代码的人"，而是"下指令的人"

回顾一下这篇文章的核心观点：

1. **多文件联动修改是软件工程的高频场景**——给 Entity 加一个字段、改一个方法签名、换一个异常类型，这些看似"小需求"的背后都是 5~20 个文件的一致性修改。人脑天然不擅长"穷举所有关联文件"这件事。

2. **Composer 的核心价值不是"帮你写代码"，而是"帮你记住所有关联文件"**。它不会因为改到第 15 个文件就疲劳、不会因为到了下班点就仓促收尾。对于"规则明确、文件众多"的修改任务，Composer 的效率是人工的 5~10 倍。

3. **Prompt 的质量直接决定 Composer 的输出质量**。三段式指令（调研→修改→验证）、@folder/@file 精确指定上下文、用 Before/After 示例描述转换模式——这些技巧能让 Composer 从"有时靠谱有时翻车"变成"几乎总能交付"。

4. **人定范围，AI 执行**是当前最合理的分工。你自己用 `grep`/`rg` 确定影响范围，然后把文件清单交给 Composer 逐一修改。不要让 AI 替你"想"，让它替你"做"。

最后，用一个表格对比 Composer 模式给你带来的改变：

| | Before（传统方式） | After（Composer 模式） |
|---|---|---|
| 工作方式 | 逐个文件手动改 | 一句话描述需求，AI 逐文件执行 |
| 遗漏风险 | 高（总有忘掉的文件） | 低（AI 自动分析引用关系） |
| 一致性问题 | 常见（DTO 改了 Entity 没改） | 罕见（AI 保证文件间一致性） |
| 测试更新 | 经常忘记（改完直接提测） | 自动同步（测试断言一并更新） |
| 耗时（加一个字段） | 15~30 分钟 | 1~3 分钟 |
| 耗时（重构异常处理） | 1~3 小时 | 3~5 分钟 |
| 心智负担 | 高（要记住所有待改文件） | 低（只需确认修改计划） |
| 文档同步 | 经常遗漏（API 文档过时） | 自动更新（MD 和 Postman 同步改） |

如果你还没试过 Composer 的多文件联动修改，今天就打开你的项目，找一个"改一个字段涉及 5+ 个文件"的需求，试试上面的 Prompt 模板。第一次看到它一口气改完 10 个文件的时候，那种感觉——我只能说，像第一次用智能机换掉诺基亚。

---

## 下一篇预告

**《Cursor vs IntelliJ IDEA AI Assistant：Java 开发者的终极选择》**

IntelliJ IDEA 终于推出了自己的 AI Assistant，背靠 JetBrains 的代码分析引擎（那玩意儿可是能精准识别所有调用链和重构影响的）。而 Cursor 则背靠 VS Code 生态 + Claude 大模型。两强相遇，Java 开发者到底该怎么选？

下篇我将从以下维度做横向对比：
- **代码补全质量**：Tam 键到底谁更懂你要写什么？
- **重构能力**：Rename、Extract Method、Change Signature 这些经典操作，谁更稳？
- **多文件联动修改**：IDEA AI Assistant 能不能做到本文 Composer 的效果？
- **Java 生态适配**：Spring Boot、MyBatis、Maven/Gradle，谁的上下文理解更准？
- **价格与性价比**：Cursor Pro vs JetBrains AI，谁更值得付费？
- **最终结论**：给出明确的使用建议——是二选一，还是组合使用？

关注我，不迷路。下一篇可能是这个系列最"左右互搏"的一篇——毕竟两个工具我都在生产环境用过一年以上，该夸的夸，该骂的骂，绝不含糊。

---

*本文首发于 CSDN，转载请注明出处。*

*作者：深耕 Java 后端 10 年的技术博主，专注 AI 赋能研发效能。*

*本文使用的工具：Cursor 0.46 + Claude 4（内置）*
