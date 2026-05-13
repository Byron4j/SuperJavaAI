# 用 AI 生成 E2E 自动化测试脚本：Playwright + Java 的智能化测试，测试用例 AI 自动生成

## 开场白：测试是研发的"护城河"，但谁写测试？

产品经理走过来："上线了吗？" 你说："还在写测试用例。" PM 说："先上线吧，没什么大改动。" 你犹豫了一下，点了合并。

结果上线 30 分钟后，用户反馈："注册按钮点不了！" Hotfix。复盘。发现前端改了按钮的 `data-testid`，E2E 测试跑不到地方。

这不是你的错。E2E 测试维护成本高、用例编写慢、元素定位脆弱——这三个问题困扰着每一个 UI 测试工程师。

本文教你用 **Playwright + Java + AI** 搭建智能化 E2E 测试体系，实现 **测试用例自动生成 + 元素智能定位 + 失败自愈**，让测试从"成本中心"变成"价值中心"。

## 一、为什么是 Playwright + Java？

### 1.1 Playwright 的核心优势

| 特性 | Selenium | Playwright |
|------|---------|------------|
| 架构 | WebDriver 协议，额外进程 | CDP 协议，浏览器内建 |
| 启动速度 | 慢（WebDriver 通信） | 快（直接控制浏览器） |
| 自动等待 | 需手动写 | 内置 auto-wait |
| 网络拦截 | 需第三方库 | 内置 route API |
| 并行执行 | 需 Grid | 内置 Browser Context |
| 多标签/iframe | 需切换 | 无缝支持 |
| Trace Viewer | 无 | 内置每一步截图+日志 |

Playwright 的 Java API 与 TypeScript 版本功能完全对等，且天然支持 **录制回放**，是 AI 生成测试的基础。

### 1.2 Maven 依赖

```xml
<dependency>
    <groupId>com.microsoft.playwright</groupId>
    <artifactId>playwright</artifactId>
    <version>1.43.0</version>
</dependency>
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>5.10.2</version>
    <scope>test</scope>
</dependency>
```

## 二、架构设计：AI 驱动的测试生成管道

```
┌─────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────┐
│ 需求文档/PRD │───▶│  AI 解析      │───▶│  测试用例生成  │───▶│  代码输出 │
│ 或 Swagger  │    │  (LLM)       │    │  (结构化)     │    │  .java   │
└─────────────┘    └──────────────┘    └──────────────┘    └──────────┘
                                                                  │
┌──────────┐    ┌──────────────┐    ┌──────────────┐              │
│ 执行结果  │◀───│  自动修复     │◀───│  执行测试     │◀─────────────┘
│ 报告     │    │  (AI自愈)    │    │  (Playwright) │
└──────────┘    └──────────────┘    └──────────────┘
```

### 2.1 核心引擎：从页面描述到测试代码

```java
/**
 * 测试用例 AI 生成引擎
 */
@Service
public class TestGenerationEngine {

    private final AIClient aiClient;
    private final PageAnalyzer pageAnalyzer;

    /**
     * 从页面 URL 自动生成 E2E 测试用例
     */
    public List<GeneratedTest> generateTests(String pageUrl, String requirement) {
        // Step 1: 分析页面结构
        PageStructure structure = pageAnalyzer.analyze(pageUrl);

        // Step 2: 结合需求生成测试场景
        String prompt = buildGenerationPrompt(structure, requirement);

        // Step 3: LLM 生成测试代码
        String rawTests = aiClient.chat(prompt);

        return parseTestCases(rawTests);
    }

    private String buildGenerationPrompt(PageStructure structure, String requirement) {
        return """
            基于以下页面结构和需求描述，生成 Playwright Java 的 E2E 测试用例。

            ## 页面结构
            URL: %s
            元素列表：
            %s

            ## 需求描述
            %s

            ## 生成规则
            1. 使用 JUnit 5 + Playwright Java API
            2. 每个测试方法必须是独立的（独立的 BrowserContext）
            3. 使用 page.waitForSelector() 确保元素加载
            4. 使用语义化定位策略（getByRole/getByLabel/getByTestId）
            5. 为每个操作添加断言
            6. 测试正常流程 + 异常流程（如空输入、超长输入）
            7. 使用 @DisplayName 中文描述测试目的
            8. 代码放在 test/java/com/example/e2e/GeneratedTests.java 中

            输出完整可编译的 Java 文件内容。
            """.formatted(
                structure.url(),
                structure.elements().stream()
                    .map(e -> "- " + e.type() + ": " + e.label() + " (selector: " + e.selector() + ")")
                    .collect(Collectors.joining("\n")),
                requirement
            );
    }
}
```

### 2.2 页面结构分析器

```java
/**
 * 使用 Playwright 自动抓取页面结构
 */
@Component
public class PageAnalyzer {

    /**
     * 访问页面并提取所有可交互元素的语义信息
     */
    public PageStructure analyze(String url) {
        try (Playwright playwright = Playwright.create()) {
            Browser browser = playwright.chromium().launch(
                new BrowserType.LaunchOptions().setHeadless(true));
            BrowserContext context = browser.newContext();
            Page page = context.newPage();

            page.navigate(url);
            page.waitForLoadState(LoadState.NETWORKIDLE);

            // 执行 JS 提取所有可交互元素
            Object elements = page.evaluate("""
                () => {
                    const interactive = [];
                    const selectors = [
                        'button', 'a', 'input', 'select', 'textarea',
                        '[role="button"]', '[role="link"]',
                        '[data-testid]', '[aria-label]'
                    ];

                    selectors.forEach(sel => {
                        document.querySelectorAll(sel).forEach(el => {
                            interactive.push({
                                tag: el.tagName.toLowerCase(),
                                type: el.type || '',
                                text: el.innerText?.trim().slice(0, 100) || '',
                                label: el.getAttribute('aria-label') || '',
                                testid: el.getAttribute('data-testid') || '',
                                id: el.id || '',
                                name: el.name || '',
                                placeholder: el.placeholder || '',
                                role: el.getAttribute('role') || '',
                                visible: el.offsetParent !== null
                            });
                        });
                    });
                    return interactive;
                }
            """);

            context.close();
            browser.close();

            return new PageStructure(url, parseElements(elements));
        }
    }
}
```

## 三、实战：AI 生成的测试用例示例

### 3.1 输入：页面 + 需求

```
页面：https://example.com/login
需求：用户可以使用邮箱+密码登录，也可以使用Google OAuth登录。
      错误密码显示错误提示，错误3次锁定账号。
```

### 3.2 输出：AI 生成的完整测试文件

```java
import com.microsoft.playwright.*;
import com.microsoft.playwright.options.*;
import org.junit.jupiter.api.*;
import static org.junit.jupiter.api.Assertions.*;

class LoginPageE2ETest {

    private static Playwright playwright;
    private static Browser browser;

    @BeforeAll
    static void setup() {
        playwright = Playwright.create();
        browser = playwright.chromium().launch(
            new BrowserType.LaunchOptions()
                .setHeadless(false)  // 调试时可设为 false
                .setSlowMo(100)       // 慢速执行便于观察
        );
    }

    @AfterAll
    static void teardown() {
        browser.close();
        playwright.close();
    }

    @Test
    @DisplayName("正常登录流程：输入正确邮箱密码 → 成功跳转首页")
    void shouldLoginWithValidCredentials() {
        BrowserContext context = browser.newContext();
        Page page = context.newPage();

        page.navigate("https://example.com/login");
        page.waitForSelector("#email");

        page.fill("#email", "user@example.com");
        page.fill("#password", "Password123!");
        page.click("button[type='submit']");

        // 验证跳转到首页
        page.waitForURL("**/dashboard");
        assertTrue(page.url().contains("/dashboard"));
        assertTrue(page.isVisible("text=欢迎回来"));

        context.close();
    }

    @Test
    @DisplayName("异常流程：错误密码 → 显示错误提示")
    void shouldShowErrorForInvalidPassword() {
        BrowserContext context = browser.newContext();
        Page page = context.newPage();

        page.navigate("https://example.com/login");
        page.fill("#email", "user@example.com");
        page.fill("#password", "WrongPassword");
        page.click("button[type='submit']");

        // 验证错误提示
        Locator errorToast = page.locator(".error-message");
        assertTrue(errorToast.isVisible());
        assertEquals("密码错误，请重试", errorToast.textContent());

        context.close();
    }

    @Test
    @DisplayName("安全策略：连续3次错误密码 → 账号锁定")
    void shouldLockAccountAfterThreeFailedAttempts() {
        BrowserContext context = browser.newContext();
        Page page = context.newPage();

        page.navigate("https://example.com/login");

        for (int i = 0; i < 3; i++) {
            page.fill("#email", "user@example.com");
            page.fill("#password", "WrongPassword");
            page.click("button[type='submit']");
            page.waitForTimeout(500); // 等待错误提示动画
        }

        // 验证锁定提示
        Locator lockMessage = page.locator(".lockout-message");
        assertTrue(lockMessage.isVisible());
        assertTrue(lockMessage.textContent().contains("账户已锁定"));

        context.close();
    }

    @Test
    @DisplayName("边界测试：空邮箱提交 → 前端校验拦截")
    void shouldValidateEmptyEmail() {
        BrowserContext context = browser.newContext();
        Page page = context.newPage();

        page.navigate("https://example.com/login");
        page.fill("#password", "Password123!");
        page.click("button[type='submit']");

        Locator emailError = page.locator("#email-error");
        assertTrue(emailError.isVisible());
        assertEquals("请输入邮箱地址", emailError.textContent());

        context.close();
    }

    @Test
    @DisplayName("Google OAuth 登录 → 跳转到 Google 授权页")
    void shouldRedirectToGoogleOAuth() {
        BrowserContext context = browser.newContext();
        Page page = context.newPage();

        page.navigate("https://example.com/login");
        page.click("button:has-text('使用 Google 登录')");

        // 等待跳转到 Google
        page.waitForURL("**/accounts.google.com/**");
        assertTrue(page.url().contains("accounts.google.com"));

        context.close();
    }
}
```

## 四、智能化增强：AI 自愈测试

传统 E2E 测试最脆弱的部分：**元素定位**。前端改一个 CSS class，测试全部挂掉。

### 4.1 智能选择器推导

```java
/**
 * 当原始选择器失败时，用 AI 找到备选选择器
 */
public class SmartLocatorHealer {

    private final AIClient aiClient;

    /**
     * 自愈：给定页面结构和目标元素描述，找到新的选择器
     */
    public String healLocator(Page page, String failedSelector,
                               String elementDescription) {
        // 获取当前页面所有元素
        String pageSnapshot = (String) page.evaluate("""
            () => {
                const elements = [];
                document.querySelectorAll('*').forEach(el => {
                    if (el.offsetParent !== null && el.innerText?.trim().length < 200) {
                        elements.push({
                            tag: el.tagName,
                            id: el.id,
                            classes: el.className,
                            text: el.innerText?.trim().slice(0, 80),
                            testid: el.getAttribute('data-testid'),
                            role: el.getAttribute('role'),
                            aria: el.getAttribute('aria-label')
                        });
                    }
                });
                return JSON.stringify(elements.slice(0, 500));
            }
        """);

        String prompt = """
            原选择器 "%s" 定位失败。
            目标元素描述："%s"

            当前页面可交互元素：
            %s

            请给出 3 个可能的新 Playwright 选择器（按优先级排序），
            优先使用 data-testid、aria-label、role 等语义化定位。
            返回 JSON 数组格式：["选择器1", "选择器2", "选择器3"]
            """.formatted(failedSelector, elementDescription, pageSnapshot);

        String response = aiClient.chat(prompt);
        List<String> candidates = parseCandidates(response);

        // 逐个尝试，返回第一个可用的选择器
        for (String candidate : candidates) {
            if (page.locator(candidate).count() > 0) {
                return candidate;
            }
        }

        throw new RuntimeException(
            "无法自愈：所有候选选择器均无效。请人工检查页面结构。");
    }
}
```

### 4.2 自愈拦截器

```java
/**
 * 包装 Playwright Page，自动捕获定位失败并尝试自愈
 */
public class SelfHealingPage {

    private final Page page;
    private final SmartLocatorHealer healer;

    public SelfHealingPage(Page page, SmartLocatorHealer healer) {
        this.page = page;
        this.healer = healer;
    }

    public void fill(String selector, String value, String description) {
        try {
            page.fill(selector, value);
        } catch (PlaywrightException e) {
            if (e.getMessage().contains("waiting for selector")) {
                String newSelector = healer.healLocator(page, selector, description);
                System.out.printf("[自愈] %s → %s%n", selector, newSelector);
                page.fill(newSelector, value);
            } else {
                throw e;
            }
        }
    }

    public void click(String selector, String description) {
        try {
            page.click(selector);
        } catch (PlaywrightException e) {
            if (e.getMessage().contains("waiting for selector")) {
                String newSelector = healer.healLocator(page, selector, description);
                System.out.printf("[自愈] %s → %s%n", selector, newSelector);
                page.click(newSelector);
            } else {
                throw e;
            }
        }
    }
}
```

## 五、AI 生成测试数据

```java
/**
 * AI 生成符合业务规则的测试数据
 */
public class TestDataGenerator {

    private final AIClient aiClient;

    /**
     * 生成边界值测试数据
     */
    public TestDataDTO generateBoundaryValues(String fieldSchema) {
        String prompt = """
            基于以下字段 Schema，生成边界值测试数据 JSON：

            Schema: %s

            生成规则：
            - 正常值（valid）
            - null 值
            - 空字符串
            - 超长字符串（>1000 字符）
            - SQL 注入尝试（'; DROP TABLE --）
            - XSS 注入尝试（<script>alert(1)</script>）
            - Unicode/emoji（🐛💥🔥）
            - 数字字段的 0/负数/极大值

            {{
              "testCases": [
                {{"name": "场景描述", "data": {{...}}, "expectedValid": true/false}}
              ]
            }}
            """.formatted(fieldSchema);

        return parseTestData(aiClient.chat(prompt));
    }
}
```

## 六、集成 CI/CD：GitHub Actions 示例

```yaml
# .github/workflows/e2e-tests.yml
name: AI E2E Tests

on:
  pull_request:
    types: [opened, synchronize]
  push:
    branches: [main]

jobs:
  e2e:
    runs-on: ubuntu-latest
    services:
      app:
        image: myapp:latest
        ports:
          - 3000:3000

    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: 21
          distribution: 'temurin'

      - name: Install Playwright Browsers
        run: mvn exec:java -e -D exec.mainClass=com.microsoft.playwright.CLI -D exec.args="install --with-deps"

      - name: Generate E2E Tests with AI
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
        run: mvn test -Dtest=TestGeneratorApp

      - name: Run Generated E2E Tests
        run: mvn test -Dtest="*E2ETest" -Dplaywright.headless=true

      - name: Upload Playwright Traces
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: playwright-traces
          path: target/playwright-traces/
```

```java
/**
 * CI 中运行的测试生成器
 */
public class TestGeneratorApp {

    public static void main(String[] args) {
        TestGenerationEngine engine = new TestGenerationEngine(
            new OpenAIClient(System.getenv("OPENAI_API_KEY")),
            new PageAnalyzer());

        String pageUrl = System.getProperty("app.url", "http://localhost:3000");
        String requirement = System.getProperty("test.requirement",
            "测试用户注册、登录、商品搜索全流程");

        List<GeneratedTest> tests = engine.generateTests(pageUrl, requirement);

        // 写入文件
        for (GeneratedTest test : tests) {
            test.writeToFile("src/test/java/com/example/e2e/");
        }

        System.out.println("Generated " + tests.size() + " E2E tests.");
    }
}
```

## 七、独特观点：测试的未来是"声明式"

我认为 E2E 测试的终极形态是：

```java
// 未来：声明式测试（AI 执行层）
@AITest
@Scenario("用户登录并下单")
public interface CheckoutFlow {

    @Step(1) @Given("用户 {email} 已登录")
    void login(@Param String email, @Param String password);

    @Step(2) @When("搜索商品 {keyword}")
    SearchResult search(@Param String keyword);

    @Step(3) @When("添加第一个商品到购物车")
    void addToCart(SearchResult result);

    @Step(4) @When("进入结算页并选择 {paymentMethod}")
    void checkout(@Param String paymentMethod);

    @Step(5) @Then("订单创建成功，显示订单号")
    void verifyOrderCreated();
}

// AI 自动将声明式描述转为 Playwright 操作
// 页面结构变化时自动适应，无需修改测试代码
```

声明式测试 + AI 执行引擎 = 永不过期的测试套件。

## 八、总结

AI + Playwright 让 E2E 测试从"手工劳动"进化到"智能生产"：

- **自动生成**：从需求文档/页面结构一键生成测试用例
- **智能定位**：语义化选择器 + 自愈机制，告别脆弱定位
- **数据工厂**：AI 生成边界值和异常测试数据
- **CI 集成**：每次 PR 自动生成 + 执行，失败自动上报 Trace

测试不应该成为瓶颈。用 AI 把测试变成你的加速器。

---

**本文是「Java + AI 编程从入门到精通」310 篇系列的第 290 篇。全系列覆盖：**

- 基础篇（1-50）：Java 基础 + AI 概念入门
- 框架篇（51-120）：Spring AI、LangChain4j、Semantic Kernel
- 进阶篇（121-200）：RAG、Agent、多模态、微调
- 实战篇（201-250）：企业级 AI 应用实战
- 延伸篇（251-310）：专项技术突破、性能优化、架构设计

**系列全部文章持续更新中，关注获取最新内容。**
