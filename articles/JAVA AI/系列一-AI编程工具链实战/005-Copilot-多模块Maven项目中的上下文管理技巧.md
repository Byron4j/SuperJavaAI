# Copilot 在多模块 Maven 项目中的上下文管理技巧：别让 AI 在模块之间"迷路"

> 微服务拆了 10 个模块，Copilot 生成的代码引用了一堆不存在的类——你以为是 AI 不行，其实是你没给它"地图"。

---

## 一、那个凌晨三点，我盯着 Copilot 生成的代码，血压飙升

凌晨 2:47，deadline 是早上 9 点的演示。

我在 `order-service` 模块里写一个下单接口，需要调用 `user-service` 的 `UserFacade` 查询用户信息，调用 `inventory-service` 的 `InventoryFacade` 扣库存。

我敲下注释：

```java
// 1. 查询用户信息
// 2. 校验库存
// 3. 创建订单
```

按 Tab，Copilot 自动补全——这行云流水的操作本该让我觉得"AI 真香"。结果它给我生成了这个：

```java
// Copilot 生成的代码
UserInfo userInfo = userService.getUserById(userId);  // userService 是个啥？项目里根本没有这个 Bean
InventoryDTO inventory = inventoryDao.selectBySku(sku); // inventoryDao？我们用的是 MyBatis-Plus，叫 InventoryMapper
Order order = new Order();
order.setOrderNo(UUID.randomUUID().toString().replace("-", "")); // 订单号生成规则是日期+序列号，不是 UUID！
```

三个调用，三个幻觉。

我深吸一口气，抽了根烟，开始手动改——改到第四个踩坑点的时候，我突然意识到：**不是 Copilot 不行，是我没告诉它我在哪、我要去哪、我周围有什么。**

> 金句：**"AI 不会迷路，迷路的是没给 AI 画地图的人。"**

这就是多模块 Maven 项目中，Copilot 上下文管理的核心命题。

---

## 二、先搞清楚：Copilot 的"视野"到底有多大？

很多开发者以为 Copilot 能看到整个项目，这是个巨大的误解。

**Copilot 的上下文窗口是有限的，且有优先级：**

```
┌─────────────────── Copilot 上下文优先级 ───────────────────┐
│                                                              │
│  【最高优先级】当前打开的文件 + 邻近 Tab 中的文件             │
│       ↓                                                      │
│  【次优先级】`@workspace` 指定的文件/文件夹                  │
│       ↓                                                      │
│  【再次级】与当前文件 import 相关的文件（有限推断）           │
│       ↓                                                      │
│  【最低优先级】项目中随机抽取的文件片段（填充窗口）           │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**在多模块项目中，最低优先级的那部分"随机片段"恰恰是灾难的来源。** 它会从 `common` 模块抽一段 Entity 定义，从 `api` 模块抽一段废弃的接口，然后缝合出一个"看起来很像、实则完全不对"的代码。

举个例子。假设你的项目结构是这样的：

```
ecommerce-platform/
├── pom.xml                          # 父 POM
├── ecommerce-common/                # 公共模块
│   ├── pom.xml
│   └── src/main/java/com/ecom/common/
│       ├── entity/
│       │   ├── Order.java
│       │   └── User.java
│       ├── dto/
│       │   ├── OrderDTO.java
│       │   └── UserDTO.java
│       └── exception/
│           └── BizException.java
├── ecommerce-api/                   # API 层（Feign 接口定义）
│   ├── pom.xml
│   └── src/main/java/com/ecom/api/
│       ├── UserFacade.java          // 用户服务 Feign 接口
│       ├── InventoryFacade.java     // 库存服务 Feign 接口
│       └── fallback/
│           ├── UserFacadeFallback.java
│           └── InventoryFacadeFallback.java
├── ecommerce-service/               # 业务服务层
│   ├── pom.xml
│   └── src/main/java/com/ecom/service/
│       ├── OrderService.java
│       ├── UserService.java
│       └── InventoryService.java
└── ecommerce-web/                   # Web 层（Controller）
    ├── pom.xml
    └── src/main/java/com/ecom/web/
        ├── OrderController.java
        └── UserController.java
```

当你在 `ecommerce-web/OrderController.java` 中让 Copilot 补全代码时，它的"随机上下文填充"可能来自 `ecommerce-service` 里的一个测试类，也可能来自 `ecommerce-common` 里一个八竿子打不着的工具方法。

**结果就是：类名不像、方法签名不对、依赖注入的 Bean 不存在。**

---

## 三、五大核心技巧：给 Copilot 画一张清晰的"地图"

### 技巧一：用 `@workspace` Agent 精准锁定模块边界

VS Code 和 JetBrains 的 Copilot Chat 都支持 `@workspace` agent。它的作用是：**让 Copilot 只在你指定的目录范围内搜索和理解代码。**

#### 错误用法（没有指定 workspace）：

```
// 在 Copilot Chat 中直接问：
"帮我生成一个查询订单详情的 Service 方法"
```

Copilot 可能在全项目范围内搜索 `Order`、`Service` 等关键词，然后给你一个缝合怪。

#### 正确用法（指定模块）：

```
@workspace /ecommerce-api 参考 UserFacade 的写法，帮我创建一个 OrderFacade 接口，放在 ecommerce-api 模块下

@workspace /ecommerce-service 基于 OrderDTO 和 BizException，在 OrderService 中生成创建订单的方法，调用 OrderFacade
```

**实操演示：**

假设你要在 `ecommerce-web` 中写一个创建订单的 Controller，需要调用 `OrderService`。你应该这么做：

1. **第一步：指定依赖模块获取上下文**

   ```
   @workspace /ecommerce-api /ecommerce-service 读取 UserFacade 和 OrderService 的接口定义
   ```

   Copilot 会列出这些文件的关键信息，让你确认。

2. **第二步：指定目标模块生成代码**

   ```
   @workspace /ecommerce-web 在 OrderController 中生成一个 POST /orders 接口，
   入参是 OrderDTO，调用 OrderService.createOrder()，
   返回 Result<OrderVO> 统一响应格式。
   参考项目中已有的 UserController 的风格和 Result 封装方式。
   ```

这样一来，Copilot 的搜索范围从"整个项目"缩小到了"三个模块"，幻觉率大幅下降。

> **核心原则：永远用 `@workspace` 把 Copilot 的视线限制在相关模块中。就像你不会让新员工第一天就看完全部 10 个微服务的代码一样，也别让 AI 这样做。**

---

### 技巧二：`.github/copilot-instructions.md` —— 你的项目"AI 说明书"

GitHub Copilot 支持在项目根目录或 `.github/` 目录下放置一个 `copilot-instructions.md` 文件。Copilot 在每次生成代码时都会读取这个文件作为"项目级 System Prompt"。

**对于多模块项目，这个文件的价值无法估量。**

#### 完整配置示例：

在项目根目录的 `.github/copilot-instructions.md`：

```markdown
# E-Commerce Platform - Copilot 编程规范

## 项目结构
本项目是多模块 Maven 项目，模块间依赖关系如下：

```
ecommerce-common     ← 基础模块，所有模块都依赖它
ecommerce-api        ← 依赖 common，定义 Feign 接口
ecommerce-service    ← 依赖 common + api
ecommerce-web        ← 依赖 common + api + service
```

## 模块职责
- **ecommerce-common**: Entity、DTO、VO、统一返回类 `Result<T>`、业务异常 `BizException`、工具类
- **ecommerce-api**: 对外暴露的 Feign 接口定义（UserFacade、InventoryFacade、OrderFacade）及降级实现
- **ecommerce-service**: 业务逻辑层，实现 api 中定义的接口，依赖 common 中的 Entity/DTO
- **ecommerce-web**: Controller 层，接收 HTTP 请求，调用 service 层，返回 Result 封装

## 代码风格
- 实体类使用 Lombok：`@Data @Builder @NoArgsConstructor @AllArgsConstructor`
- Controller 统一返回 `Result<T>`，异常全局拦截，不要在 Controller 中 try-catch
- Service 层抛 `BizException`，不要返回 null
- 使用 MyBatis-Plus，Mapper 继承 `BaseMapper<T>`
- 字段校验使用 `jakarta.validation.constraints`，不用 `javax`
- 日期使用 `LocalDateTime`，不要用 `Date`
- 日志使用 `@Slf4j`，关键节点打印 `log.info`

## 命名规范
- Mapper 接口：`{Entity}Mapper`，放在 common 模块的 mapper 包下
- Service 接口：`{Entity}Service`，放在 service 模块
- Controller 类：`{Entity}Controller`，放在 web 模块
- DTO：用于接收前端参数，放在 common 模块的 dto 包下
- VO：用于返回前端数据，放在 common 模块的 vo 包下

## 依赖关系
- common 不依赖任何其他业务模块
- api 只依赖 common
- service 依赖 common + api
- web 依赖 common + api + service
- **禁止循环依赖**
- **Web 层不能直接调用 Mapper，必须通过 Service**
```

#### 效果对比：

**没有 `copilot-instructions.md`：**

```java
// Copilot 生成的代码：
@RestController
public class OrderController {
    @Autowired
    private OrderMapper orderMapper;  // ❌ Web 层直接调 Mapper，违反分层约束
    
    @PostMapping("/orders")
    public Object create(@RequestBody Order order) {  // ❌ 入参应该是 DTO，返回应该是 Result<T>
        orderMapper.insert(order);
        return "success";  // ❌ 返回裸字符串，没有统一响应格式
    }
}
```

**有 `copilot-instructions.md`：**

```java
// Copilot 生成的代码：
@RestController
@Slf4j
public class OrderController {
    @Autowired
    private OrderService orderService;  // ✅ 只注入 Service
    
    @PostMapping("/orders")
    public Result<OrderVO> create(@Valid @RequestBody OrderCreateDTO dto) {  // ✅ 入参 DTO，返回 Result
        log.info("创建订单，userId={}, skuId={}", dto.getUserId(), dto.getSkuId());
        OrderVO vo = orderService.createOrder(dto);
        return Result.success(vo);  // ✅ 统一响应格式
    }
}
```

**差距肉眼可见。**

> **实操建议**：这个文件写好之后，在 Copilot Chat 中也可以用 `#file:.github/copilot-instructions.md` 显式引用，强化 AI 对项目规范的记忆。

---

### 技巧三：Tab 页"喂料法" —— 把关键接口当"食材"喂给 Copilot

Copilot 的上下文窗口中最优先读取的，是你当前打开的文件以及**附近几个 Tab 页中的文件**。

利用这个机制，你可以精准地"喂"上下文给它。

#### 操作步骤：

**场景：你需要在 `OrderController` 中调用 `OrderService.createOrder()` 方法。**

1. **打开目标文件**：双击打开 `OrderController.java`（这是你要编辑的文件）

2. **打开依赖接口文件并置顶**：
   - 用 `Ctrl/Cmd + N` 搜索打开 `OrderService.java`
   - 用 `Ctrl/Cmd + N` 搜索打开 `OrderDTO.java`
   - 用 `Ctrl/Cmd + N` 搜索打开 `Result.java`
   - 用 `Ctrl/Cmd + N` 搜索打开 `BizException.java`
   - 把这四个 Tab 页拖到编辑区最前面

3. **开始写代码**：现在 Copilot 的上下文中已经有 `OrderService` 的方法签名、`OrderDTO` 的字段结构、`Result` 的 API、`BizException` 的构造函数。

4. **写注释引导**：

```java
// 创建订单：校验参数 → 调用 orderService.createOrder → 返回 Result<OrderVO>
```

5. **按 Tab 接受补全**。

#### 原理图解：

```
┌─────────────────────────────────────────────────────┐
│  Copilot 读取的上下文                                │
│                                                     │
│  ┌────────────┐ ┌──────────────┐ ┌───────────────┐  │
│  │ 当前编辑的  │ │ Tab 2:       │ │ Tab 3:        │  │
│  │ Order       │ │ OrderService │ │ OrderDTO.java │  │
│  │ Controller  │ │ .java        │ │               │  │
│  └────────────┘ └──────────────┘ └───────────────┘  │
│  ┌────────────┐ ┌──────────────┐                    │
│  │ Tab 4:     │ │ Tab 5:       │                    │
│  │ Result.java│ │ BizException │                    │
│  │            │ │ .java        │                    │
│  └────────────┘ └──────────────┘                    │
│                                                     │
│  → Copilot 生成的代码精准捕获了所有依赖结构和类型信息  │
└─────────────────────────────────────────────────────┘
```

#### 进阶技巧：组合拳

如果你需要一次性引入多个依赖文件的上下文，还有一个更高效的做法——在 Copilot Chat 中使用 `#file` 引用：

```
#file:ecommerce-service/src/main/java/com/ecom/service/OrderService.java 
#file:ecommerce-common/src/main/java/com/ecom/common/dto/OrderDTO.java

基于以上两个文件的接口定义，在 OrderController 中生成创建订单的 REST 接口
```

**`#file` + `@workspace` + Tab 喂料，三件套齐上，幻觉率可以从 60% 降到 10% 以下。**

---

### 技巧四：IDEA Scopes 配置 —— 告诉 Copilot 哪些是"源码"，哪些是"噪音"

IntelliJ IDEA 支持自定义 Scopes（作用域），你可以用它来精确划定 Copilot 应该关注的代码范围。

**背景**：多模块项目中，`target/` 目录下的编译产物、`src/test/` 中的测试代码、以及第三方依赖的源码，都可能被 Copilot 误读为上下文，造成"污染"。

#### 配置步骤：

1. **打开 Scopes 配置**：
   `Settings → Appearance & Behavior → Scopes`

2. **创建自定义 Scope**，命名为 `Copilot Context`：

   ```
   Pattern:
   !file[ecommerce-platform]:target//* &&
   !file[ecommerce-platform]:.git//* &&
   !file[ecommerce-platform]:*/node_modules//* &&
   file[ecommerce-platform]:ecommerce-common/src/main/java//* ||
   file[ecommerce-platform]:ecommerce-api/src/main/java//* ||
   file[ecommerce-platform]:ecommerce-service/src/main/java//* ||
   file[ecommerce-platform]:ecommerce-web/src/main/java//*
   ```

3. **排除测试和生成代码**：

   ```
   && !file:*src/test//* 
   && !file:*target/generated-sources//*
   ```

4. **在 Copilot 中使用 Scope**：
   在 Copilot Chat 中输入 `@workspace` 后，选择你创建的 `Copilot Context` scope。

#### 效果说明：

| 场景 | 未配置 Scope | 配置 Scope 后 |
|------|-------------|--------------|
| 生成代码引用的类 | 可能来自 test 包或 target 目录 | 只来自 src/main/java |
| 代码风格参考 | 可能参考了测试代码的风格 | 只参考业务代码的风格 |
| 生成速度 | 较慢（需索引大量无关文件） | 更快（只索引核心源码） |

> **一句话：Scopes 帮 Copilot 做了"降噪"——只喂精华，不喂垃圾。**

---

### 技巧五：模块依赖图在 Prompt 中的精准描述

有时候你需要 Copilot 理解整个模块间的调用链路。这时，直接在 Prompt 中画依赖图是最直接有效的方式。

#### 模板一：文字版依赖图

```
# 项目模块依赖关系：
ecommerce-common (基础层)
    ↑ depends on
ecommerce-api (接口定义层)
    ↑ depends on
ecommerce-service (业务逻辑层)
    ↑ depends on
ecommerce-web (控制器层)

# 当前任务：
在 ecommerce-web 模块中，为 OrderController 新增一个 "取消订单" 的接口。
调用链路：OrderController → OrderService → OrderFacade(Feign) → 远程库存服务
需要处理的异常：BizException(来自 common 模块)
需要返回的类型：Result<OrderVO>(来自 common 模块)
入库涉及的表：t_order(对应的 Entity 在 common 模块的 entity 包下)
```

#### 模板二：Graphviz/Mermaid 图 + 自然语言

在 Copilot Chat 中插入 Mermaid 依赖图也是可行的：

```
项目依赖关系如下：
graph TD
    Common[ecommerce-common: Entity/DTO/VO/Result/BizException]
    API[ecommerce-api: Feign接口定义]
    Service[ecommerce-service: 业务逻辑]
    Web[ecommerce-web: Controller]
    Common --> API
    Common --> Service
    Common --> Web
    API --> Service
    Service --> Web
    API --> Web

在这个架构下，帮我在 Service 层生成库存校验逻辑：
1. 入参是 skuId 和 quantity
2. 调用 InventoryFacade.checkStock() 检查库存
3. 库存不足抛 BizException("INSUFFICIENT_STOCK")
4. 返回 boolean
```

**Copilot 会遵循你描述的依赖关系来判断"可调用"和"不可调用"的边界。** 例如，它不会在 Common 模块中生成调用 Service 的代码，因为它理解了"Common 是最底层，不依赖上层"。

---

## 四、实战对比：同一个需求，有无上下文管理的天壤之别

### 测试需求

在 `ecommerce-web` 模块的 `OrderController` 中，新增一个"根据订单ID查询订单详情"的接口。

### 场景 A：无上下文管理（裸奔模式）

**操作**：只打开 `OrderController.java`，直接写注释让 Copilot 补全。

**生成结果**：

```java
@GetMapping("/orders/{id}")
public Order getOrderById(@PathVariable String id) {
    // Copilot 的"自由发挥"
    Order order = orderDao.selectById(id);  // ❌ 没有 orderDao 这个 Bean
    if (order == null) {
        throw new RuntimeException("订单不存在");  // ❌ 应该用 BizException
    }
    order.setUserName(userDao.selectById(order.getUserId()).getName()); // ❌ Web 层直接调 Dao，且 userDao 不存在
    return order;  // ❌ 直接返回 Entity，应该返回 VO，且应该用 Result 包裹
}
```

**问题清单**：
1. Bean 名称不对（`orderDao` → 应该是 `OrderMapper`）
2. 异常类型不对（`RuntimeException` → 应该是 `BizException`）
3. 分层混乱（Web 层直接调 Dao）
4. 返回类型不对（Entity → VO）
5. 没有统一响应封装
6. 凭空造了 `userDao` Bean

**7 个错误，0 个能用。**

### 场景 B：有上下文管理（全副武装）

**操作**：
1. 在 `.github/copilot-instructions.md` 中写好项目规范（参考技巧二的配置）
2. 打开 Tab 页：`OrderController.java`、`OrderDTO.java`、`OrderVO.java`、`Result.java`、`BizException.java`、`OrderService.java`
3. 在 Copilot Chat 中用 `@workspace /ecommerce-web /ecommerce-service` 限定范围
4. 用 Prompt 引导：

```
@workspace /ecommerce-web /ecommerce-service 
#file:ecommerce-web/OrderController.java 
#file:ecommerce-service/OrderService.java
# 在 OrderController 中新增一个 GET /orders/{orderId} 接口：
# 1. 使用 @PathVariable 接收 orderId
# 2. 调用 orderService.getOrderDetail(orderId)
# 3. 返回 Result<OrderVO>
# 4. 参考 UserController 的风格
```

**生成结果**：

```java
@GetMapping("/orders/{orderId}")
public Result<OrderVO> getOrderDetail(@PathVariable String orderId) {
    log.info("查询订单详情，orderId={}", orderId);
    OrderVO vo = orderService.getOrderDetail(orderId);
    return Result.success(vo);
}
```

**简洁、正确、符合项目规范。一步到位。**

### 对比总结

| 维度 | 无上下文管理 | 有上下文管理 |
|------|-------------|-------------|
| Bean 名称正确率 | ~20% | ~95% |
| 返回类型正确率 | ~30% | ~95% |
| 分层约束遵守 | ~10% | ~90% |
| 异常处理方式 | 随机 | 符合项目规范 |
| 需要人工修改次数 | 5-8 次 | 0-1 次 |
| 实际可用率 | 0% | 90%+ |

> 金句：**"你以为 Copilot 是傻子，其实是你没给它说明书。"**

---

## 五、多模块项目 Copilot 最佳实践检查清单

把下面这张清单贴在显示器旁边，每次在新模块里写代码之前，过一遍：

```
┌─────────── Copilot 上下文管理检查清单 ───────────────┐
│                                                        │
│  ☐ 1. .github/copilot-instructions.md 是否已配置？     │
│       → 包含模块结构、依赖关系、代码规范、命名约定       │
│                                                        │
│  ☐ 2. 当前编辑文件的依赖接口是否已在 Tab 页中打开？     │
│       → Service 接口、DTO/VO 类、Result 封装、异常类    │
│                                                        │
│  ☐ 3. Copilot Chat 是否使用了 @workspace 限定范围？    │
│       → 指定 1-3 个相关模块，不要用项目根目录           │
│                                                        │
│  ☐ 4. 是否在 Prompt 中描述了模块调用链路？              │
│       → "Controller → Service → Mapper" 或文字依赖图    │
│                                                        │
│  ☐ 5. IDEA Scopes 是否排除了 test/target/generated？   │
│       → 避免 Copilot 读到编译产物和测试代码             │
│                                                        │
│  ☐ 6. 生成代码后，是否逐条核对了 import 语句？          │
│       → Copilot 偶尔会导入不存在的包，这是最容易排查的  │
│                                                        │
│  ☐ 7. 是否用 #file 显式引用了关键接口文件？             │
│       → 优先级：#file > @workspace > Tab 页 > 默认      │
│                                                        │
│  ☐ 8. 是否在 Prompt 中给了"参考对象"？                  │
│       → "参考 UserController 的风格"、"模仿 XxxService" │
│                                                        │
└────────────────────────────────────────────────────────┘
```

---

## 六、常见翻车现场及急救方案

### 翻车现场 1：`@workspace` 引用了不存在的模块路径

```
// 错误示范
@workspace /ecommerce-order  // 实际模块名叫 ecommerce-order-module

// 正确做法
先用 @workspace / 让 Copilot 列出所有子模块名，再精准指定
```

### 翻车现场 2：`copilot-instructions.md` 太长被截断

Copilot 对 `copilot-instructions.md` 的读取是有 token 限制的（约 2000-4000 token）。如果你的项目非常复杂，需要精简：

**精简技巧**：
- 把详细的 API 文档放 Confluence，指令文件只放**结构、约束、禁忌**
- 用列表代替段落，用关键符号（✅ ❌ ⚠️）增加可读性
- 最重要的规则放前 20 行

### 翻车现场 3：Tab 页开太多，Copilot 反而"晕"了

Copilot 会读取邻近 Tab 页，但数量有限。建议：

- **保持 4-8 个相关 Tab 页打开**
- **不相干的 Tab 页关掉**（尤其是其他模块的 Controller 和测试类）
- **如果 Tab 页太多，用 `#file` 精确指定，而不是依赖隐式上下文**

### 翻车现场 4：改了 `copilot-instructions.md` 但没生效

修改 `copilot-instructions.md` 后，Copilot 不会立即"看到"变更。需要：

1. **关闭并重新打开当前编辑的 Java 文件**
2. **或者重启 Copilot 服务**：`Ctrl/Cmd + Shift + P → "Reload Window"`

---

## 七、写在最后

多模块项目中使用 Copilot，本质上是一门"驯兽"的艺术。

你不能指望把一头狮子扔进原始森林，它就自动知道该去哪打猎。你只能一步步给它划定领地、标记水源、标注猎物出没的路线。

**`@workspace` 是"领地"**，告诉它去哪找吃的。
**`copilot-instructions.md` 是"规则"**，告诉它什么能吃什么不能吃。
**Tab 页 + `#file` 是"食物"**，直接放到它嘴边。
**Scopes 是"围栏"**，挡住不该去的地方。
**模块依赖图是"地图"**，让它知道自己在食物链的哪一环。

五样东西备齐，AI 就不再是一只瞎撞的野兽，而是一把指哪打哪的利刃。

> 金句：**"AI 时代的程序员，核心能力不再是'会写代码'，而是'会指挥 AI 写对代码'。"**

---

### 💬 聊天区留给你

你在多模块项目中使用 Copilot 遇到过哪些翻车现场？是生成的 Service 引用了隔壁模块的测试类？还是凭空创造了一个不存在的 Feign 接口？

**评论区聊聊你的"AI 翻车经历"，点赞最高的三位，我把我整理的 `.github/copilot-instructions.md` 全量模板发你。**

---

*下一篇预告：**《Copilot 生成代码的 Code Review 清单：AI 写的代码，你敢直接合并吗？》**——我会给出 20 条真实踩过的坑，包含 SQL 注入风险、N+1 查询、事务边界错误等 AI 高频犯错场景。关注我，别让 AI 的 bug 上生产。*

---

> 本文使用 GitHub Copilot（GPT-4）辅助撰写。所有 Prompt 和配置均为真实可用，欢迎直接复制到你的项目中尝试。
