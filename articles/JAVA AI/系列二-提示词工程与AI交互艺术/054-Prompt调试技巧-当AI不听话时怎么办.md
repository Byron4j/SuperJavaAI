# Prompt 调试技巧：当 AI 不听话时怎么办？10 个"AI 不听指令"场景的急救方案

## 开篇：到底是谁的问题？

你告诉 AI "不要用 Lombok"，它给你生成了 `@Getter` 和 `@Setter`。

你告诉 AI "只生成 Java 代码"，它给你写了一段 200 字的解释。

你告诉 AI "生成生产就绪的代码"，它在每个方法里加了 `// TODO: 需要完善`。

你开始怀疑：到底是 AI 傻，还是我的 Prompt 有问题？

答案是：**90% 的情况下是你的 Prompt 有问题，10% 的情况下是 AI 的理解偏差——但这两者都可以通过 Prompt 调试来修复。**

这篇文章总结了在日常 Java 开发中遇到的最常见的 10 个"AI 不听话"场景，每个场景都会给出：**错误 Prompt → 问题分析 → 修复后 Prompt → 效果对比**。你可以把它当成一本急救手册——遇到问题时对照着找解决方案。

## 场景一：AI 无视"不要 XXX"约束

### 症状

你在 Prompt 中写了"不要使用 Lombok"、"不要使用 ${}"、"不要吞掉异常"，但 AI 照做不误。

### 错误 Prompt

```
"生成一个 User 实体类，不要使用 Lombok"
```

### AI（错误）输出

```java
@Data           // ← 还是用了 Lombok！
@NoArgsConstructor
@AllArgsConstructor
public class User {
    private Long id;
    private String username;
    private String email;
}
```

### 为什么 AI 不听话？

AI 对人类语言中的**否定句**理解较差。"不要 XXX"的指令在模型的注意力机制中，关键词"XXX"反而被加强了——AI "看到"了 `Lombok` 这个词，但没有充分理解前面的"不要"。这就像你对孩子说"不要想大象"，结果他脑子里全是大象。

### 修复方案：用正向约束替代否定约束

```
"生成一个 User 实体类。
要求：手动编写所有 getter 和 setter 方法，手动编写构造方法（无参构造+全参构造）。
禁止使用 Lombok 的任何注解（@Data, @Getter, @Setter, @Builder 等）"
```

### 修复后的 AI 输出

```java
public class User {
    private Long id;
    private String username;
    private String email;

    public User() {}

    public User(Long id, String username, String email) {
        this.id = id;
        this.username = username;
        this.email = email;
    }

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getUsername() { return username; }
    public void setUsername(String username) { this.username = username; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
}
```

### 正向约束对照表

| 否定指令（无效） | 正向指令（有效） |
|-----------------|-----------------|
| 不要用 Lombok | 手动编写 getter/setter/构造方法 |
| 不要用 `${}` | 所有参数使用 `#{}` 参数化查询 |
| 不要吞掉异常 | 捕获异常后记录日志并向上抛出 |
| 不要返回 null | 使用 Optional 包装返回值 |
| 不要用 `SELECT *` | 明确列出需要查询的所有字段 |
| 不要过度设计 | 用最简洁的方式实现，不超过 3 个类 |
| 不要加 TODO 注释 | 生成的代码必须是生产就绪的完整实现 |

---

## 场景二：AI 输出格式不对

### 症状

你只想要代码，但 AI 输出了：
- 大段解释："首先，我们需要...然后，我们考虑...最后，我们..."
- 问好和祝福："好的，以下是代码：...希望这些对你有帮助！"
- 额外的建议："另外，你还可以考虑使用..."

### 错误 Prompt

```
"帮我写一个 RestTemplate 的工具类"
```

### AI（错误）输出

```
好的！我来帮你编写一个 RestTemplate 的工具类。首先我们需要了解你的需求...

（200字的前言）

接下来是代码实现：

```java
public class RestTemplateUtil {
    // ...
}
```

（100字的补充说明和优化建议）

希望这个工具类对你有帮助！如果有其他问题欢迎继续提问。
```

### 为什么 AI 不听话？

AI 被训练成"乐于助人"的助手，它的默认行为是提供解释和上下文。当你没有明确限制输出格式时，AI 倾向于提供"完整的对话体验"——包括开场白、解释、代码、建议、结束语。

### 修复方案：在 Prompt 末尾使用严格的输出格式指令

```
"帮我写一个 RestTemplate 的工具类。

输出格式：仅输出 Java 代码，不输出任何其他内容。
不要输出解释、建议、问候语。
第一个字符必须是 'p'（package 或 public 的第一个字母）。
如果一定要输出注释说明，请用单行注释 // 标注在代码中。"
```

### 修复后的 AI 输出

```java
package com.example.util;

import org.springframework.http.*;
import org.springframework.web.client.RestTemplate;
import java.util.Map;

public class RestTemplateUtil {

    private static final RestTemplate restTemplate = new RestTemplate();

    public static <T> ResponseEntity<T> get(String url, Class<T> responseType) {
        return restTemplate.getForEntity(url, responseType);
    }

    public static <T> ResponseEntity<T> post(String url, Object request, Class<T> responseType) {
        HttpEntity<Object> entity = new HttpEntity<>(request);
        return restTemplate.postForEntity(url, entity, responseType);
    }
}
```

### 输出格式约束模板

```markdown
## 输出格式约束（复制到你的 Prompt 末尾）

基础版：
"仅输出代码，不输出任何解释和问候语。"

严格版：
"输出格式要求：
- 不输出任何问候语（如：好的、以下是代码）
- 不输出任何解释（如：首先我们需要...）
- 不输出任何建议（如：另外你还可以...）
- 不输出任何结束语（如：希望对你有所帮助）
- 如果必须加注释，使用 // 或 /* */ 标注在代码中
- 输出的第一个可见字符必须是代码内容"

强制格式版：
"Code only. No explanations. No greetings. No suggestions. No markdown formatting."
```

---

## 场景三：AI 生成的代码引用了不存在的类

### 症状

AI 生成的代码 import 了一些你的项目里根本没有的类：
- `import com.github.pagehelper.PageHelper;`（你的项目用的是 MyBatis-Plus）
- `import lombok.Data;`（你的项目禁用了 Lombok）
- `import cn.hutool.core.util.StrUtil;`（你的项目没有引入 Hutool）

### 错误 Prompt

```
"生成一个分页查询用户列表的代码"
```

### AI（错误）输出

```java
import com.github.pagehelper.PageHelper;  // ← 项目中不存在
import com.github.pagehelper.PageInfo;    // ← 项目中不存在
import lombok.RequiredArgsConstructor;   // ← 项目禁用

@Service
@RequiredArgsConstructor
public class UserService {
    
    public PageInfo<User> listUsers(int pageNum, int pageSize) {
        PageHelper.startPage(pageNum, pageSize);  // ← 调用了不存在的 API
        List<User> users = userMapper.selectAll();
        return new PageInfo<>(users);
    }
}
```

### 为什么 AI 不听话？

AI 的训练数据涵盖了大量的 Java 库，当它看到"分页查询"时，它的知识库中 PageHelper、MyBatis-Plus Page、Spring Data Page 等多个方案都有权重。如果你没有明确指定，它会随机选择或选它"最熟悉"的那个。

### 修复方案：在 Prompt 开头明确技术栈和关键依赖

```
"技术栈：Spring Boot 3.2 + MyBatis-Plus 3.5.5 + JDK 21
项目依赖：仅包含 spring-boot-starter-web, mybatis-plus-boot-starter, mysql-connector-j
项目不包含：Lombok, PageHelper, Hutool

请使用 MyBatis-Plus 的 Page<T> 和 LambdaQueryWrapper 实现分页查询。"
```

### 修复后的代码

```java
import com.baomidou.mybatisplus.core.conditions.query.LambdaQueryWrapper;
import com.baomidou.mybatisplus.extension.plugins.pagination.Page;
import com.baomidou.mybatisplus.extension.service.impl.ServiceImpl;
// 所有 import 都来自项目中实际存在的依赖

@Service
public class UserService extends ServiceImpl<UserMapper, User> {

    private final UserMapper userMapper;

    public UserService(UserMapper userMapper) {
        this.userMapper = userMapper;
    }

    public Page<User> listUsers(Integer pageNum, Integer pageSize) {
        Page<User> page = new Page<>(pageNum, pageSize);
        LambdaQueryWrapper<User> wrapper = new LambdaQueryWrapper<>();
        wrapper.orderByDesc(User::getCreateTime);
        return userMapper.selectPage(page, wrapper);
    }
}
```

### 项目依赖声明模板

```markdown
## 使用此 Prompt 前，复制以下内容填入：

技术栈声明：
项目：{project_name}
框架：Spring Boot {version}
ORM：MyBatis-Plus {version}（注意：不使用 PageHelper、JPA）
JDK：{version}
构建工具：Maven {version} / Gradle {version}

已有依赖（仅这些，不要引用其他的）：
- spring-boot-starter-web
- mybatis-plus-boot-starter 3.5.5
- mysql-connector-j
- spring-boot-starter-validation

禁止引入的库（项目中没有，不要引用）：
- Lombok (团队禁用了)
- Hutool (没有引入)
- Guava (没有引入)
- Apache Commons Lang3 (没有引入)
```

---

## 场景四：AI 过度设计

### 症状

你让 AI 写一个简单的折扣计算，它给你生成了：
- `DiscountStrategy` 接口
- `VipDiscountStrategy` 实现类
- `NormalDiscountStrategy` 实现类
- `DiscountStrategyFactory` 工厂类
- `DiscountCalculator` 计算器类
- `DiscountType` 枚举类

6 个类，实现了一个 `if-else` 就能搞定的事。

### 错误 Prompt

```
"写一个根据用户等级计算折扣的功能：VIP 打 8 折，SVIP 打 7 折，普通用户不打折"
```

### AI（错误）输出

```java
// 1. DiscountStrategy.java - 策略接口
public interface DiscountStrategy {
    BigDecimal calculate(BigDecimal originalPrice);
}

// 2. VipDiscountStrategy.java - VIP折扣策略
@Component
public class VipDiscountStrategy implements DiscountStrategy {
    @Override
    public BigDecimal calculate(BigDecimal originalPrice) {
        return originalPrice.multiply(new BigDecimal("0.8"));
    }
}

// 3. SvipDiscountStrategy.java - SVIP折扣策略
@Component
public class SvipDiscountStrategy implements DiscountStrategy {
    @Override
    public BigDecimal calculate(BigDecimal originalPrice) {
        return originalPrice.multiply(new BigDecimal("0.7"));
    }
}

// 4. NormalDiscountStrategy.java - 普通折扣策略
@Component
public class NormalDiscountStrategy implements DiscountStrategy {
    @Override
    public BigDecimal calculate(BigDecimal originalPrice) {
        return originalPrice;
    }
}

// 5. DiscountStrategyFactory.java - 策略工厂
@Component
public class DiscountStrategyFactory {
    private final Map<UserLevel, DiscountStrategy> strategyMap;
    
    public DiscountStrategyFactory(List<DiscountStrategy> strategies) {
        // ...Spring 注入策略模式
    }
    
    public DiscountStrategy getStrategy(UserLevel level) {
        return strategyMap.get(level);
    }
}

// 6. UserLevel.java - 枚举
public enum UserLevel {
    NORMAL, VIP, SVIP
}
```

### 为什么 AI 不听话？

AI 被大量"设计模式最佳实践"的训练数据影响，当它看到"根据不同条件做不同处理"时，策略模式几乎是条件反射。AI 默认"复杂 = 好"，这是训练数据中过度强调设计模式带来的偏差。

### 修复方案：明确要求简洁，给出代码行数限制

```
"写一个根据用户等级计算折扣的功能：VIP 打 8 折，SVIP 打 7 折，普通用户不打折。

要求：用最简单的方式实现。
不需要使用任何设计模式（策略模式、工厂模式等）。
如果使用了接口、工厂、策略模式中的任何一个，视为错误。
整个功能不超过一个类，不超过 30 行代码。
直接写计算逻辑，不需要考虑"可扩展性"。"
```

### 修复后的输出

```java
public class DiscountCalculator {

    private static final BigDecimal VIP_DISCOUNT = new BigDecimal("0.8");
    private static final BigDecimal SVIP_DISCOUNT = new BigDecimal("0.7");

    public static BigDecimal calculate(String userLevel, BigDecimal originalPrice) {
        if (originalPrice == null || userLevel == null) {
            throw new IllegalArgumentException("参数不能为空");
        }
        return switch (userLevel.toUpperCase()) {
            case "VIP" -> originalPrice.multiply(VIP_DISCOUNT);
            case "SVIP" -> originalPrice.multiply(SVIP_DISCOUNT);
            default -> originalPrice;
        };
    }
}
```

简洁、直接、20 行代码搞定。

---

## 场景五：AI 总在代码里加 TODO 注释

### 症状

AI 生成的代码到处是：

```java
// TODO: 需要根据实际情况调整参数
// TODO: 这里需要加上权限校验
// TODO: 需要补充单元测试
// TODO: 完善异常处理逻辑
// TODO: 添加日志记录
```

### 错误 Prompt

```
"生成一个订单创建接口"
```

### AI（错误）输出

```java
@PostMapping("/orders")
public ResponseEntity<OrderResponse> createOrder(@RequestBody OrderRequest request) {
    // TODO: 需要校验用户登录状态
    // TODO: 需要校验库存
    // TODO: 需要对接真实的支付接口
    Order order = new Order();
    order.setUserId(request.getUserId());
    order.setAmount(request.getAmount());
    // TODO: 完善订单状态流转逻辑
    orderService.save(order);
    return ResponseEntity.ok(convertToResponse(order));
}
```

### 为什么 AI 不听话？

TODO 是 AI 的"免责声明"。当 AI 不确定某个需求时，它会用 TODO 标记来"保留余地"。这是训练数据中大量"不完整示例代码"带来的惯性——开源项目中的示例代码经常用 TODO 表示"这里留给读者自己实现"。

### 修复方案：明确要求生产就绪

```
"生成一个订单创建接口。
要求：
- 代码必须是生产就绪的，可以直接部署使用
- 如果某个功能无法在当前上下文中完整实现，请用完整的实现替代示例代码
- 禁止输出 TODO、FIXME、HACK 等占位注释
- 如果某个参数暂时无法确定，使用合理的默认值并说明
- 所有方法都必须是完整实现，不能是空方法或抛 UnsupportedOperationException"
```

### 修复后的输出

```java
@PostMapping("/orders")
public ResponseEntity<OrderResponse> createOrder(
        @Valid @RequestBody OrderRequest request,
        @AuthenticationPrincipal UserDetails userDetails) {
    
    Long userId = ((CustomUserDetails) userDetails).getUserId();
    
    Product product = productService.getByIdWithLock(request.getProductId());
    if (product == null) {
        throw new BusinessException("PROD_001", "商品不存在");
    }
    if (product.getStock() < request.getQuantity()) {
        throw new BusinessException("PROD_002", "库存不足");
    }
    
    Order order = new Order();
    order.setUserId(userId);
    order.setProductId(request.getProductId());
    order.setQuantity(request.getQuantity());
    order.setAmount(product.getPrice().multiply(new BigDecimal(request.getQuantity())));
    order.setStatus(OrderStatus.PENDING);
    order.setCreateTime(LocalDateTime.now());
    
    orderService.createOrder(order);
    
    productService.decreaseStock(product.getId(), request.getQuantity());
    
    log.info("订单创建成功: orderId={}, userId={}, amount={}", 
             order.getId(), userId, order.getAmount());
    
    return ResponseEntity.status(HttpStatus.CREATED)
            .body(OrderResponse.from(order));
}
```

---

## 场景六：AI 忘记前面的上下文

### 症状

在长对话中，你前面说了"使用 Java 21"，后面 AI 生成了 Java 8 的代码。你说过"不用 Lombok"，第 5 轮对话 AI 又开始用 `@Data`。

### 错误 Prompt（多轮对话中）

```
第1轮：
用户："帮我生成一个 Java 21 + Spring Boot 3.2 的项目，不使用 Lombok"
AI："好的..."

第2轮：
用户："给我加一个 User 实体类"
AI 生成了带 @Data 的 User 类  ← 忘了不用 Lombok

第3轮：
用户："再加一个 Order 实体类"
AI 生成了使用 var 的代码  ← 忘了 Java 21 的特性但用了 Java 10 的语法
```

### 为什么 AI 不听话？

AI 的上下文窗口是有限的，而且注意力会自然地向最近的消息倾斜。当对话超过一定轮次，早期的约束会被"稀释"。这不是 AI 故意忘记，而是长上下文中关键信息的信噪比降低了。

### 修复方案：在每轮对话中重新注入关键约束

```
"（关键上下文提醒）
当前项目：Spring Boot 3.2 + Java 21 + MyBatis-Plus 3.5.5
代码规范：不使用 Lombok / 构造器注入 / 所有方法有 Javadoc / 使用 Record 作为 DTO

请为这个项目添加一个 User 实体类。"
```

更好的方案——使用结构化 Prompt 模板，每次追加需求时把核心模板带上去：

```markdown
## 项目环境（每次对话都粘贴此段）
```
Spring Boot 3.2 | Java 21 | MyBatis-Plus 3.5.5 | MySQL 8.0
规范：无 Lombok / 构造器注入 / Javadoc / @Transactional(写操作)
禁止：${} SQL拼接 / 吞异常 / TODO注释 / 过度设计
```

## 本次需求
（在此填写本次具体需求）
```

---

## 场景七：AI 帮改不相关的代码

### 症状

你让 AI "修改 UserService 的 getUserById 方法"，它顺手把整个 UserService 重构了，还改了 UserController 和 UserMapper。

### 错误 Prompt

```
"UserService 的 getUserById 方法需要加上缓存，帮我改一下"
```

### AI（错误）输出

```java
// 不仅改了 getUserById，还改了：
// 1. 添加了 Spring Cache 的配置类
// 2. 修改了 UserController 添加了 @Cacheable
// 3. 重构了 UserService 的其他方法
// 4. 新增了 CacheConfig、CacheKeyGenerator 等类
```

### 为什么 AI 不听话？

AI 的"优化本能"超出了你的预期范围。当你提到"加缓存"，AI 认为这是在提示它做"完整的缓存方案设计"，而不仅仅是给一个方法加注解。

### 修复方案：明确修改范围和禁止事项

```
"仅修改 UserService.java 的 getUserById 方法，添加 @Cacheable 缓存注解。

约束：
- 只修改这一个方法，不修改任何其他方法
- 不修改 UserController、UserMapper 或其他任何文件
- 不需要创建新的配置类
- 不需要解释缓存策略
- 只输出修改后的 getUserById 方法代码，不要输出整个类

当前方法代码：
```java
public User getUserById(Long id) {
    return userMapper.selectById(id);
}
```"
```

### 修复后的输出

```java
@Cacheable(value = "user", key = "#id", unless = "#result == null")
public User getUserById(Long id) {
    return userMapper.selectById(id);
}
```

精准的增量修改，一行的改动。

---

## 场景八：AI 生成的异常处理太粗糙

### 症状

AI 生成的异常处理往往是这样的：

```java
try {
    // 业务代码
} catch (Exception e) {
    e.printStackTrace();  // ← 线上环境的禁术
    return null;           // ← 调用方没法知道为什么失败
}
```

### 错误 Prompt

```
"写一个文件上传的方法，要有异常处理"
```

### AI（错误）输出

```java
public String uploadFile(MultipartFile file) {
    try {
        String fileName = file.getOriginalFilename();
        Path filePath = Paths.get("/uploads/" + fileName);
        Files.copy(file.getInputStream(), filePath);
        return fileName;
    } catch (Exception e) {
        e.printStackTrace();
        return null;
    }
}
```

### 为什么 AI 不听话？

"要有异常处理"这个指令太模糊了。对于 AI 来说，`try-catch + printStackTrace` 确实是"异常处理"——它完成了"捕获并处理异常"这个任务。问题在于你没有说清楚"怎么处理"。

### 修复方案：明确异常处理的具体要求

```
"写一个文件上传的方法。异常处理要求：
1. 不要用 e.printStackTrace()，使用 SLF4J 日志记录
2. 不要吞掉原始异常，catch 后将业务异常向上抛出
3. 不要返回 null 表示失败，抛具体的业务异常
4. 日志中必须包含：异常类型、异常信息、关键业务参数（如文件名、文件大小）
5. 对不同类型的异常做不同的处理：
   - IOException：记录日志，抛出 FileStorageException
   - 文件大小超限：抛出 FileSizeExceededException 并告知最大限制
   - 文件类型不支持：抛出 InvalidFileTypeException 并告知支持的类型"
```

### 修复后的输出

```java
private static final Logger log = LoggerFactory.getLogger(FileService.class);
private static final long MAX_FILE_SIZE = 10 * 1024 * 1024; // 10MB
private static final Set<String> ALLOWED_TYPES = Set.of(
    "image/jpeg", "image/png", "application/pdf"
);

public String uploadFile(MultipartFile file) {
    if (file == null || file.isEmpty()) {
        throw new FileStorageException("FILE_001", "上传文件不能为空");
    }
    
    if (file.getSize() > MAX_FILE_SIZE) {
        throw new FileSizeExceededException("FILE_002", 
                "文件大小超出限制", file.getSize(), MAX_FILE_SIZE);
    }
    
    String contentType = file.getContentType();
    if (contentType == null || !ALLOWED_TYPES.contains(contentType)) {
        throw new InvalidFileTypeException("FILE_003", 
                "不支持的文件类型: " + contentType);
    }
    
    try {
        String originalName = file.getOriginalFilename();
        String storedName = UUID.randomUUID() + getExtension(originalName);
        Path targetPath = Paths.get(uploadDir, storedName);
        Files.copy(file.getInputStream(), targetPath, StandardCopyOption.REPLACE_EXISTING);
        
        log.info("文件上传成功: originalName={}, storedName={}, size={}, type={}",
                 originalName, storedName, file.getSize(), contentType);
        return storedName;
                
    } catch (IOException e) {
        log.error("文件存储失败: fileName={}, fileSize={}, contentType={}",
                  file.getOriginalFilename(), file.getSize(), file.getContentType(), e);
        throw new FileStorageException("FILE_004", "文件存储失败", e);
    }
}
```

---

## 场景九：AI 不理解领域术语

### 症状

你说了"请实现一个承保流程"，AI 一脸茫然，或者把"承保"理解成了"承诺保护"。

### 错误 Prompt

```
"实现一个保险承保流程"
```

### AI（错误）输出

```java
// AI 把"承保"理解成了类似"审批"的流程
public class InsuranceApproval {
    public void approve(Application application) {
        // 只是简单的状态变更
        application.setStatus("APPROVED");
    }
}
```

### 为什么 AI 不听话？

AI 的训练数据虽然庞大，但特定行业的专业术语可能理解不够精准。尤其是在金融、医疗、法律等专业领域，术语的含义非常精确，AI 可能会按字面理解或泛化理解。

### 修复方案：在 Prompt 开头定义术语表

```
"领域术语定义（请严格按照以下定义理解）：
- 承保：指保险公司对投保人的投保申请进行审核，同意接受其投保并签发保险单的过程。
  包含：核保（风险评估）、定价（费率计算）、出单（生成保单）三个子流程。
- 核保：评估投保标的（被保人/被保物）的风险等级，决定是否承保及承保条件。
  核保结果：标准体承保、加费承保、除外承保、拒保。
- 投保人：购买保险的人（付钱的人）。
- 被保人：保险保障的对象（可能不是投保人本人）。
- 保费：投保人向保险公司支付的费用。
- 保额：保险公司承担赔偿或给付保险金的最高限额。

请实现保险承保流程，严格使用以上术语定义。"
```

### 修复后的输出

```java
/**
 * 保险承保流程服务。
 * 承保 = 核保(风险评估) + 定价(费率计算) + 出单(生成保单)
 */
@Service
public class UnderwritingService {

    private final RiskAssessmentEngine riskEngine;
    private final PremiumCalculator premiumCalculator;
    private final PolicyGenerator policyGenerator;

    public UnderwritingService(RiskAssessmentEngine riskEngine, 
                               PremiumCalculator premiumCalculator,
                               PolicyGenerator policyGenerator) {
        this.riskEngine = riskEngine;
        this.premiumCalculator = premiumCalculator;
        this.policyGenerator = policyGenerator;
    }

    @Transactional
    public PolicyUnderwritingResult underwrite(InsuranceApplication application) {
        // 核保：风险评估
        RiskAssessmentResult riskResult = riskEngine.assess(
                application.getApplicant(), 
                application.getInsured());

        if (riskResult.getDecision() == UnderwritingDecision.REJECT) {
            log.info("拒保: applicationId={}, reason={}", 
                     application.getId(), riskResult.getRejectReason());
            return PolicyUnderwritingResult.rejected(riskResult.getRejectReason());
        }

        // 定价：根据风险等级计算保费
        BigDecimal premium = premiumCalculator.calculate(
                riskResult.getRiskLevel(),
                application.getCoverageAmount(), // 保额
                application.getPolicyTerm());

        // 出单：生成并保存保单
        InsurancePolicy policy = policyGenerator.generate(
                application, premium, riskResult.getUnderwritingCondition());

        log.info("承保成功: policyNo={}, premium={}, condition={}", 
                 policy.getPolicyNo(), premium, riskResult.getUnderwritingCondition());

        return PolicyUnderwritingResult.accepted(policy, premium);
    }
}
```

### 术语表模板

```markdown
## 领域术语表（添加在 Prompt 开头）

本项目在【XXX】领域，以下术语有特定含义，请严格按此理解：

| 术语 | 英文 | 定义 |
|------|------|------|
| 承保 | Underwriting | 保险公司审核并接受投保的过程 |
| 核保 | Risk Assessment | 评估被保人/物的风险等级 |
| 保费 | Premium | 投保人向保险公司支付的费用 |
| 保额 | Coverage Amount | 保险公司承担的最高赔偿限额 |

如果 Prompt 中出现了上述术语，请严格按照此定义理解，而不是通用含义。
```

---

## 场景十：AI 的回答太短/太长

### 症状

你需要一个详细的实现，AI 给了你 3 行伪代码。或者你只需要一个简单的回答，AI 写了 500 字的长文。

### 错误 Prompt（太短）

```
"写一个 JWT 鉴权的 Filter"
```

AI 输出：
```java
// 实现 JWT 验证 Filter
public class JwtFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest request, ...) {
        // 验证 JWT token
    }
}
```

### 错误 Prompt（太长）

```
"用 Spring Security 实现一个完整的 JWT 鉴权体系。首先需要配置 SecurityFilterChain，然后创建 JwtTokenProvider 来生成和验证 Token。JwtTokenProvider 需要包含生成 Token、验证 Token、从 Token 中提取用户信息、校验 Token 是否过期等方法。同时需要实现 UserDetailsService 来加载用户信息。此外还需要考虑 Token 刷新机制，当前端发现 Token 即将过期时..."
```

AI 输出：2500 字的详细实现，包含了你并不需要的 6 个额外类。

### 修复方案：明确字数/行数/复杂度约束

```
太短的修复：
"写一个完整的 JWT 鉴权 Filter，包含以下功能：
1. 从请求头 Authorization 中提取 Bearer Token
2. 解析和验证 JWT（使用 jjwt 库）
3. 验证成功后设置 SecurityContext
4. 验证失败返回 401 状态码和 JSON 错误信息
5. 跳过白名单 URL（如 /api/auth/login、/api/public/**）
请写出完整可运行的代码，每行都要是可执行的实现代码，不省略任何步骤。"

太长的修复：
"写一个最精简的 JWT 验证 Filter。
只实现核心功能：提取 Token→验证→设置 SecurityContext。
不需要 Token 刷新、不需要黑名单、不需要额外的 Provider 类。
限制：整个实现（含 import）不超过 40 行代码。
如果超过 40 行，请放弃次要功能。"
```

### 长度约束指南

| 需求 | 约束写法 |
|------|---------|
| 极简实现 | "用不超过 30 行代码实现，只包含核心逻辑" |
| 标准实现 | "完整可用，不需要过度设计" |
| 详尽实现 | "请详细实现每个方法，不要省略任何步骤，包含完整的 Javadoc" |
| 生产级 | "生产就绪的完整实现，包含日志、异常处理、校验、单元测试" |
| 一行答案 | "用一句话回答，不需要解释" |
| 3 句话答案 | "用最多 3 句话回答" |

---

## 综合调试清单

当你发现 AI 输出不符合预期时，按以下清单排查：

```markdown
## Prompt 调试清单

□ 1. 是否使用了正向约束而非否定约束？
      "使用手动 getter/setter" 而非 "不要用 Lombok"

□ 2. 是否明确了输出格式？
      加上 "仅输出 Java 代码，不输出解释"

□ 3. 是否声明了技术栈和依赖？
      加上 "Spring Boot 3.2 + MyBatis-Plus 3.5.5"

□ 4. 是否加入了简洁性约束？
      加上 "用最简单的方式实现，不超过 3 个类"

□ 5. 是否要求了生产就绪？
      加上 "代码必须是可直接部署的，不要 TODO"

□ 6. 长对话中是否重新注入了关键上下文？
      每轮对话前加上 "项目环境回顾：..."

□ 7. 是否限定了修改范围？
      加上 "只修改 X 方法，不要改动其他代码"

□ 8. 是否明确了异常处理的具体方式？
      加上 "异常处理：日志记录 + 向上抛业务异常 + 统一处理"

□ 9. 是否定义了领域术语？
      加上术语表："承保 = 核保 + 定价 + 出单"

□ 10. 是否明确了回答长度？
      加上 "用不超过 N 句话回答" 或 "完整实现每个细节"
```

## 写在最后

Prompt 调试本质上是**沟通调试**。你在跟一个学识渊博但需要精准引导的"同事"沟通。当它"不听话"时，不要急于生气或放弃，而是像 Code Review 一样分析问题的根源：

- 是我没说清楚？（90% 的情况）
- 还是它理解偏了？（10% 的情况，但也可以通过更清晰的措辞避免）

这篇文章的 10 个场景覆盖了日常使用中最常见的"AI 不听话"情况。建议收藏这份清单，遇到问题时快速定位。

记住：**一个好的 Prompt 不是命令集，而是一份清晰的开发规格说明书。** 当你的 Prompt 像 Spring Boot 官方文档一样清晰、结构化、无歧义时，AI 的输出也会像 Spring Boot 一样稳定、规范、可靠。

---

**下一篇预告**：元 Prompt（Meta-Prompt）实战——学到今天，Prompt 工程系列已经进入尾声。但等等，有没有一种可能：我们不应该费力学写 Prompt，而应该让 AI 帮我们写 Prompt？这就是元 Prompt 的核心理念。下一篇将给出可直接复制使用的元 Prompt 模板，终极目标是——**你说需求，AI 写 Prompt，你再执行 Prompt**。敬请期待！
