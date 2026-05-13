# Spring MVC深度解析：请求处理流程与核心组件全链路源码分析

**文章标签：** #java #springmvc #servlet #web框架 #源码分析 #设计模式 #面试 #深度解析

## 目录

- [引言：为什么需要MVC框架](#引言为什么需要mvc框架)
- [理论基础：MVC设计模式的本质](#理论基础mvc设计模式的本质)
- [来龙去脉：Spring MVC的发展史](#来龙去脉spring-mvc的发展史)
- [Spring MVC架构与核心组件](#spring-mvc架构与核心组件)
- [DispatcherServlet源码深度分析](#dispatcherservlet源码深度分析)
- [HandlerMapping源码深度分析](#handlermapping源码深度分析)
- [HandlerAdapter源码深度分析](#handleradapter源码深度分析)
- [设计模式解析](#设计模式解析)
- [实战案例：工业级Web应用](#实战案例工业级web应用)
- [对比分析：Spring MVC vs 替代方案](#对比分析spring-mvc-vs-替代方案)
- [性能分析与优化](#性能分析与优化)
- [常见陷阱与最佳实践](#常见陷阱与最佳实践)
- [面试题与参考答案](#面试题与参考答案)

---

## 引言：为什么需要MVC框架

MVC（Model-View-Controller）不是简单的"三层架构"，而是一种**关注点分离**的架构模式。在Web开发中，MVC框架将请求处理、业务逻辑和视图渲染解耦，使得代码更易于维护和测试。

**核心认知：**

```
传统Servlet的问题：
┌─────────────────────────────────────────┐
│  UserServlet                            │
│  ├─ doGet()                             │
│  │   ├─ 解析请求参数                     │
│  │   ├─ 调用业务逻辑                     │
│  │   ├─ 查询数据库                       │
│  │   ├─ 处理异常                         │
│  │   ├─ 设置响应头                       │
│  │   └─ 输出HTML                         │
│  └─ doPost()                            │
│      └─ 类似doGet()...                  │
└─────────────────────────────────────────┘
问题：
- 代码臃肿，职责混乱
- 视图与业务逻辑耦合
- 难以单元测试
- 重复代码多

MVC的解决方式：
┌─────────────────────────────────────────┐
│  Controller（控制器）                    │
│  ├─ 接收请求                            │
│  ├─ 调用Service                         │
│  └─ 返回视图名/数据                      │
├─────────────────────────────────────────┤
│  Model（模型）                           │
│  └─ 业务逻辑 + 数据                      │
├─────────────────────────────────────────┤
│  View（视图）                            │
│  └─ 渲染页面/JSON/XML                    │
└─────────────────────────────────────────┘
优势：
- 职责清晰
- 易于测试
- 视图可替换
- 代码复用
```

**关键洞察**：Spring MVC的核心价值在于**前端控制器模式**（Front Controller Pattern），通过DispatcherServlet统一接收所有请求，再根据配置分发给不同的处理器，实现了请求的集中管理和灵活分发。

---

## 理论基础：MVC设计模式的本质

### 1. MVC的核心职责

```
MVC的职责划分：

Model（模型）：
- 封装业务逻辑
- 管理数据状态
- 处理业务规则
- 不依赖视图和控制器

View（视图）：
- 展示数据
- 接收用户输入（部分框架）
- 不包含业务逻辑
- 可替换（JSP、Thymeleaf、JSON等）

Controller（控制器）：
- 接收用户请求
- 调用Model处理业务
- 选择View展示结果
- 协调Model和View
```

### 2. MVC的变体

```
MVC的演进：

1. 经典MVC
   Controller → Model → View
   View观察Model变化

2. Web MVC（前端控制器模式）
   请求 → Front Controller → Controller → Model → View
   
3. MVP（Model-View-Presenter）
   View和Presenter双向通信
   Presenter代替Controller

4. MVVM（Model-View-ViewModel）
   View和ViewModel数据绑定
   如Vue.js、Angular
```

### 3. Spring MVC的设计哲学

```
Spring MVC的设计原则：

1. 开闭原则（OCP）
   - 对扩展开放：支持多种视图技术、处理器类型
   - 对修改关闭：核心流程不变

2. 依赖倒置原则（DIP）
   - Controller依赖Service接口
   - Service依赖DAO接口

3. 单一职责原则（SRP）
   - DispatcherServlet：请求分发
   - HandlerMapping：请求映射
   - HandlerAdapter：处理器适配
   - ViewResolver：视图解析
```

---

## 来龙去脉：Spring MVC的发展史

### 第一阶段：Spring MVC诞生（2004）

Spring 1.0引入MVC模块：

```xml
<!-- Spring 1.x MVC配置 -->
<bean id="viewResolver" class="org.springframework.web.servlet.view.InternalResourceViewResolver">
    <property name="prefix" value="/WEB-INF/views/"/>
    <property name="suffix" value=".jsp"/>
</bean>
```

### 第二阶段：注解驱动（2009）

Spring 2.5引入`@Controller`、`@RequestMapping`：

```java
@Controller
public class UserController {
    @RequestMapping("/user/{id}")
    public String getUser(@PathVariable Long id, Model model) {
        model.addAttribute("user", userService.getById(id));
        return "user/detail";
    }
}
```

### 第三阶段：RESTful支持（2011）

Spring 3.0引入`@RestController`、`@PathVariable`、`@RequestBody`：

```java
@RestController
public class UserApiController {
    @GetMapping("/api/users/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.getById(id);
    }
}
```

### 第四阶段：Spring Boot时代（2014至今）

Spring Boot自动配置Spring MVC：

```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

---

## Spring MVC架构与核心组件

### Spring MVC架构图

```
+-------------------------------------------------------------+
|                    Spring MVC 架构                          |
+-------------------------------------------------------------+
|                                                             |
|   客户端请求                                                  |
|      |                                                      |
|      V                                                      |
|   +-----------------------------------------------------+   |
|   |              DispatcherServlet                        |   |
|   |  +-----------+ +-----------+ +-------------------+  |   |
|   |  |Handler    | |Handler    | | HandlerException  |  |   |
|   |  |Mapping    | |Adapter    | | Resolver          |  |   |
|   |  +-----------+ +-----------+ +-------------------+  |   |
|   |  +-----------+ +-----------+ +-------------------+  |   |
|   |  |View       | |View       | | HandlerInterceptor|  |   |
|   |  |Resolver   | |           | |                   |  |   |
|   |  +-----------+ +-----------+ +-------------------+  |   |
|   +-----------------------------------------------------+   |
|                          |                                  |
|   +-----------------------------------------------------+   |
|   |              Controller / @RestController             |   |
|   +-----------------------------------------------------+   |
|                          |                                  |
|   +-----------------------------------------------------+   |
|   |              Service / DAO 层                         |   |
|   +-----------------------------------------------------+   |
|                          |                                  |
|   +-----------------------------------------------------+   |
|   |              View (JSP/Thymeleaf/JSON)                |   |
|   +-----------------------------------------------------+   |
|                                                             |
+-------------------------------------------------------------+
```

### 核心组件

| 组件 | 作用 |
|------|------|
| DispatcherServlet | 前端控制器，统一接收请求 |
| HandlerMapping | 请求与处理器的映射 |
| HandlerAdapter | 适配不同的处理器类型 |
| HandlerInterceptor | 拦截器，预处理和后处理 |
| Controller | 业务处理器 |
| ModelAndView | 模型和视图封装 |
| ViewResolver | 视图解析器 |
| View | 视图渲染 |

---

## DispatcherServlet源码深度分析

DispatcherServlet是Spring MVC的核心，继承自HttpServlet。

### 类继承结构

```
HttpServlet (javax.servlet)
  └── GenericServlet
        └── HttpServletBean (Spring)
              └── FrameworkServlet (Spring)
                    └── DispatcherServlet (Spring)
```

### 初始化：onRefresh()

```java
public class DispatcherServlet extends FrameworkServlet {
    
    // 9大组件
    @Nullable
    private List<HandlerMapping> handlerMappings;
    @Nullable
    private List<HandlerAdapter> handlerAdapters;
    @Nullable
    private List<HandlerExceptionResolver> handlerExceptionResolvers;
    @Nullable
    private List<ViewResolver> viewResolvers;
    // ...
    
    @Override
    protected void onRefresh(ApplicationContext context) {
        // 初始化9大组件
        initMultipartResolver(context);    // 文件上传解析器
        initLocaleResolver(context);       // 国际化解析器
        initThemeResolver(context);        // 主题解析器
        initHandlerMappings(context);      // 处理器映射器
        initHandlerAdapters(context);      // 处理器适配器
        initHandlerExceptionResolvers(context); // 异常处理器
        initRequestToViewNameTranslator(context); // 视图名转换器
        initViewResolvers(context);        // 视图解析器
        initFlashMapManager(context);      // 重定向参数管理器
    }
}
```

### 请求分发：doDispatch()

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) 
        throws Exception {
    HttpServletRequest processedRequest = request;
    HandlerExecutionChain mappedHandler = null;
    boolean multipartRequestParsed = false;
    
    WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
    
    try {
        ModelAndView mv = null;
        Exception dispatchException = null;
        
        try {
            // 1. 检查文件上传
            processedRequest = checkMultipart(request);
            multipartRequestParsed = (processedRequest != request);
            
            // 2. 获取HandlerExecutionChain（包含Handler和拦截器链）
            mappedHandler = getHandler(processedRequest);
            if (mappedHandler == null) {
                noHandlerFound(processedRequest, response);
                return;
            }
            
            // 3. 获取HandlerAdapter
            HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
            
            // 4. 执行拦截器preHandle
            if (!mappedHandler.applyPreHandle(processedRequest, response)) {
                return; // 拦截器返回false，结束处理
            }
            
            // 5. 调用Handler（Controller方法）
            mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
            
            // 6. 应用默认视图名
            applyDefaultViewName(processedRequest, mv);
            
            // 7. 执行拦截器postHandle
            mappedHandler.applyPostHandle(processedRequest, response, mv);
        }
        catch (Exception ex) {
            dispatchException = ex;
        }
        catch (Throwable err) {
            dispatchException = new NestedServletException("Handler dispatch failed", err);
        }
        
        // 8. 处理结果（渲染视图或处理异常）
        processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
    }
    catch (Exception ex) {
        triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
    }
    catch (Throwable err) {
        triggerAfterCompletion(processedRequest, response, mappedHandler,
                new NestedServletException("Handler processing failed", err));
    }
    finally {
        if (asyncManager.isConcurrentHandlingStarted()) {
            if (mappedHandler != null) {
                mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
            }
        }
        else {
            if (multipartRequestParsed) {
                cleanupMultipart(processedRequest);
            }
        }
    }
}
```

### processDispatchResult() 处理结果

```java
private void processDispatchResult(HttpServletRequest request, HttpServletResponse response,
        @Nullable HandlerExecutionChain mappedHandler, @Nullable ModelAndView mv,
        @Nullable Exception exception) throws Exception {
    
    boolean errorView = false;
    
    // 1. 处理异常
    if (exception != null) {
        if (exception instanceof ModelAndViewDefiningException) {
            mv = ((ModelAndViewDefiningException) exception).getModelAndView();
        }
        else {
            Object handler = (mappedHandler != null ? mappedHandler.getHandler() : null);
            mv = processHandlerException(request, response, handler, exception);
            errorView = (mv != null);
        }
    }
    
    // 2. 渲染视图
    if (mv != null && !mv.wasCleared()) {
        render(mv, request, response);
        if (errorView) {
            WebUtils.clearErrorRequestAttributes(request);
        }
    }
    else {
        if (logger.isTraceEnabled()) {
            logger.trace("No view rendering, null ModelAndView returned.");
        }
    }
    
    // 3. 执行拦截器afterCompletion
    if (mappedHandler != null) {
        mappedHandler.triggerAfterCompletion(request, response, null);
    }
}
```

---

## HandlerMapping源码深度分析

HandlerMapping负责将请求映射到处理器。

```java
public interface HandlerMapping {
    String PATH_WITHIN_HANDLER_MAPPING_ATTRIBUTE = 
        HandlerMapping.class.getName() + ".pathWithinHandlerMapping";
    String BEST_MATCHING_PATTERN_ATTRIBUTE = 
        HandlerMapping.class.getName() + ".bestMatchingPattern";
    
    // 根据请求获取HandlerExecutionChain
    @Nullable
    HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception;
}
```

### RequestMappingHandlerMapping

```java
public class RequestMappingHandlerMapping extends RequestMappingInfoHandlerMapping
        implements MatchableHandlerMapping, EmbeddedValueResolverAware {
    
    @Override
    protected void initHandlerMethods() {
        // 扫描所有Bean，查找@RequestMapping方法
        for (String beanName : getCandidateBeanNames()) {
            if (!beanName.startsWith(SCOPED_TARGET_NAME_PREFIX)) {
                processCandidateBean(beanName);
            }
        }
    }
    
    protected void processCandidateBean(String beanName) {
        Class<?> beanType = null;
        try {
            beanType = obtainApplicationContext().getType(beanName);
        }
        catch (Throwable ex) {
            // ...
        }
        // 检查是否有@RequestMapping注解
        if (beanType != null && isHandler(beanType)) {
            detectHandlerMethods(beanName);
        }
    }
    
    @Override
    protected boolean isHandler(Class<?> beanType) {
        // 检查是否有@Controller或@RequestMapping注解
        return (AnnotatedElementUtils.hasAnnotation(beanType, Controller.class) ||
                AnnotatedElementUtils.hasAnnotation(beanType, RequestMapping.class));
    }
}
```

### @RequestMapping的映射

```java
@Controller
@RequestMapping("/user")
public class UserController {
    
    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.getUser(id);
    }
    
    @PostMapping
    public User createUser(@RequestBody User user) {
        return userService.save(user);
    }
}
```

Spring MVC的HandlerMapping实现：
- `RequestMappingHandlerMapping`：处理@RequestMapping
- `BeanNameUrlHandlerMapping`：根据bean名称映射
- `SimpleUrlHandlerMapping`：简单URL映射
- `RouterFunctionMapping`：Spring 5+ 函数式路由

---

## HandlerAdapter源码深度分析

HandlerAdapter适配不同类型的处理器：

```java
public interface HandlerAdapter {
    // 是否支持该处理器
    boolean supports(Object handler);
    
    // 调用处理器
    @Nullable
    ModelAndView handle(HttpServletRequest request, HttpServletResponse response, 
                        Object handler) throws Exception;
    
    // 获取Last-Modified时间
    long getLastModified(HttpServletRequest request, Object handler);
}
```

### RequestMappingHandlerAdapter

```java
public class RequestMappingHandlerAdapter extends AbstractHandlerMethodAdapter
        implements BeanFactoryAware, InitializingBean {
    
    @Override
    protected ModelAndView handleInternal(HttpServletRequest request,
            HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
        
        ModelAndView mav;
        // 检查请求方法是否支持
        checkRequest(request);
        
        // 是否在session同步模式下执行
        if (this.synchronizeOnSession) {
            HttpSession session = request.getSession(false);
            if (session != null) {
                Object mutex = WebUtils.getSessionMutex(session);
                synchronized (mutex) {
                    mav = invokeHandlerMethod(request, response, handlerMethod);
                }
            }
            else {
                mav = invokeHandlerMethod(request, response, handlerMethod);
            }
        }
        else {
            mav = invokeHandlerMethod(request, response, handlerMethod);
        }
        
        return mav;
    }
    
    @Nullable
    protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
            HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
        
        ServletWebRequest webRequest = new ServletWebRequest(request, response);
        
        try {
            WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
            ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);
            
            // 创建可执行的HandlerMethod
            ServletInvocableHandlerMethod invocableMethod = 
                createInvocableHandlerMethod(handlerMethod);
            
            // 设置参数解析器
            if (this.argumentResolvers != null) {
                invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
            }
            // 设置返回值处理器
            if (this.returnValueHandlers != null) {
                invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
            }
            
            // 执行方法
            invocableMethod.invokeAndHandle(webRequest, mavContainer);
            
            // 获取ModelAndView
            return getModelAndView(mavContainer, modelFactory, webRequest);
        }
        finally {
            webRequest.requestCompleted();
        }
    }
}
```

**主要实现：**
- `RequestMappingHandlerAdapter`：处理@RequestMapping方法
- `HttpRequestHandlerAdapter`：处理HttpRequestHandler
- `SimpleControllerHandlerAdapter`：处理Controller接口

---

## 设计模式解析

### 1. 前端控制器模式（Front Controller Pattern）

**体现：** `DispatcherServlet`作为所有请求的入口，统一分发到不同的处理器。

**为什么：** 集中处理请求的共同逻辑（如编码设置、权限检查、日志记录），避免每个Controller重复处理。

### 2. 适配器模式（Adapter Pattern）

**体现：** `HandlerAdapter`适配不同类型的处理器（注解Controller、接口Controller、HttpRequestHandler等）。

**为什么：** Spring MVC支持多种处理器类型，通过适配器统一调用接口，无需修改DispatcherServlet。

```java
// 统一的handle接口
public interface HandlerAdapter {
    boolean supports(Object handler);
    ModelAndView handle(HttpServletRequest request, HttpServletResponse response, 
                        Object handler) throws Exception;
}

// 适配@RequestMapping方法
public class RequestMappingHandlerAdapter implements HandlerAdapter { }

// 适配HttpRequestHandler
public class HttpRequestHandlerAdapter implements HandlerAdapter { }
```

### 3. 策略模式（Strategy Pattern）

**体现：** `HandlerMapping`、`ViewResolver`、`HandlerExceptionResolver`都可以配置多个实现，Spring按顺序尝试。

**为什么：** 允许灵活配置请求映射策略、视图解析策略、异常处理策略。

### 4. 模板方法模式（Template Method）

**体现：** `FrameworkServlet.processRequest()`定义了请求处理的标准流程（doService -> doDispatch），子类（DispatcherServlet）实现具体逻辑。

**为什么：** 定义标准流程，同时允许子类扩展。

---

## 实战案例：工业级Web应用

### 案例1：完整Spring MVC配置

```java
@Configuration
@EnableWebMvc
@ComponentScan(basePackages = "com.example.mvc.demo")
public class MvcConfig implements WebMvcConfigurer {
    
    @Override
    public void configureViewResolvers(ViewResolverRegistry registry) {
        InternalResourceViewResolver resolver = new InternalResourceViewResolver();
        resolver.setPrefix("/WEB-INF/views/");
        resolver.setSuffix(".jsp");
        resolver.setCache(false); // 开发环境关闭缓存
        registry.viewResolver(resolver);
    }
    
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new AuthInterceptor())
                .addPathPatterns("/**")
                .excludePathPatterns("/login", "/static/**");
        
        registry.addInterceptor(new LoggingInterceptor())
                .addPathPatterns("/**");
    }
    
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/static/**")
                .addResourceLocations("/static/");
    }
    
    @Override
    public void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer) {
        configurer.enable(); // 启用默认Servlet处理静态资源
    }
}
```

```java
@Controller
@RequestMapping("/user")
public class UserController {
    
    private final UserService userService;
    
    public UserController(UserService userService) {
        this.userService = userService;
    }
    
    @GetMapping("/{id}")
    public String getUser(@PathVariable Long id, Model model) {
        User user = userService.getUserById(id);
        model.addAttribute("user", user);
        return "user/detail"; // 视图名
    }
    
    @GetMapping("/list")
    public String listUsers(Model model) {
        List<User> users = userService.getAllUsers();
        model.addAttribute("users", users);
        return "user/list";
    }
    
    @PostMapping
    public String createUser(@ModelAttribute User user, 
                             RedirectAttributes redirectAttrs) {
        User saved = userService.saveUser(user);
        redirectAttrs.addFlashAttribute("message", "用户创建成功");
        return "redirect:/user/" + saved.getId();
    }
    
    @DeleteMapping("/{id}")
    @ResponseBody
    public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
        userService.deleteUser(id);
        return ResponseEntity.ok().build();
    }
}
```

```java
@RestController
@RequestMapping("/api/users")
public class UserApiController {
    
    private final UserService userService;
    
    public UserApiController(UserService userService) {
        this.userService = userService;
    }
    
    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.getUserById(id);
    }
    
    @GetMapping
    public List<User> getAllUsers() {
        return userService.getAllUsers();
    }
    
    @PostMapping
    public User createUser(@RequestBody @Valid UserDTO userDTO) {
        return userService.saveUser(userDTO.toEntity());
    }
    
    @PutMapping("/{id}")
    public User updateUser(@PathVariable Long id, @RequestBody @Valid UserDTO userDTO) {
        return userService.updateUser(id, userDTO.toEntity());
    }
    
    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void deleteUser(@PathVariable Long id) {
        userService.deleteUser(id);
    }
}
```

### 案例2：拦截器实现

```java
public class AuthInterceptor implements HandlerInterceptor {
    
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, 
                             Object handler) throws Exception {
        // 登录校验
        String token = request.getHeader("Authorization");
        if (token == null || !isValidToken(token)) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            response.getWriter().write("Unauthorized");
            return false; // 拦截
        }
        
        // 将用户信息放入请求属性
        request.setAttribute("currentUser", getUserFromToken(token));
        return true; // 继续执行
    }
    
    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response,
                           Object handler, ModelAndView modelAndView) throws Exception {
        // Controller执行后，View渲染前
        if (modelAndView != null) {
            modelAndView.addObject("serverTime", new Date());
        }
    }
    
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response,
                                Object handler, Exception ex) throws Exception {
        // 请求完成后的清理工作
        if (ex != null) {
            System.out.println("请求处理异常: " + ex.getMessage());
        }
    }
    
    private boolean isValidToken(String token) {
        // 验证token逻辑
        return token.startsWith("Bearer ");
    }
    
    private User getUserFromToken(String token) {
        // 解析token获取用户
        return new User();
    }
}
```

```java
public class LoggingInterceptor implements HandlerInterceptor {
    
    private static final ThreadLocal<Long> startTime = new ThreadLocal<>();
    
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, 
                             Object handler) throws Exception {
        startTime.set(System.currentTimeMillis());
        System.out.println("[REQUEST] " + request.getMethod() + " " + request.getRequestURI());
        return true;
    }
    
    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response,
                           Object handler, ModelAndView modelAndView) throws Exception {
        long cost = System.currentTimeMillis() - startTime.get();
        System.out.println("[RESPONSE] " + request.getMethod() + " " + request.getRequestURI() 
            + " - " + response.getStatus() + " - " + cost + "ms");
    }
    
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response,
                                Object handler, Exception ex) throws Exception {
        startTime.remove(); // 清理ThreadLocal，避免内存泄漏
    }
}
```

### 案例3：全局异常处理

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handleBusinessException(BusinessException e) {
        ErrorResponse error = new ErrorResponse(e.getCode(), e.getMessage());
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(error);
    }
    
    @ExceptionHandler(NotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFoundException(NotFoundException e) {
        ErrorResponse error = new ErrorResponse(404, e.getMessage());
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidationException(
            MethodArgumentNotValidException e) {
        List<String> errors = e.getBindingResult().getFieldErrors().stream()
            .map(error -> error.getField() + ": " + error.getDefaultMessage())
            .collect(Collectors.toList());
        ErrorResponse error = new ErrorResponse(400, "参数校验失败", errors);
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(error);
    }
    
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleException(Exception e) {
        ErrorResponse error = new ErrorResponse(500, "系统内部错误");
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(error);
    }
}

@Data
@AllArgsConstructor
class ErrorResponse {
    private int code;
    private String message;
    private List<String> details;
    
    public ErrorResponse(int code, String message) {
        this(code, message, null);
    }
}
```

---

## 对比分析：Spring MVC vs 替代方案

### Spring MVC vs Struts2

| 特性 | Spring MVC | Struts2 |
|------|-----------|---------|
| 架构 | 基于Servlet，DispatcherServlet | 基于Filter，StrutsPrepareAndExecuteFilter |
| 配置方式 | 注解/XML | XML/注解 |
| 与Spring集成 | 原生支持 | 需要插件 |
| 性能 | 高 | 中 |
| 学习曲线 | 低 | 高 |
| 活跃度 | 高（Spring生态核心） | 低（已停止维护） |

**选择建议：** 新项目直接选Spring MVC或Spring WebFlux，Struts2已不再维护。

### Spring MVC vs JAX-RS (Jersey/RESTEasy)

| 特性 | Spring MVC | JAX-RS |
|------|-----------|--------|
| 标准 | Spring专属 | JSR-311/339标准 |
| 注解 | @GetMapping等 | @GET/@POST等 |
| 生态集成 | Spring全家桶 | Java EE/Jakarta EE |
| 功能 | 完整MVC（视图+REST） | 仅REST |

**选择建议：** Spring生态用Spring MVC；需要标准兼容性用JAX-RS。

### Spring MVC vs Spring WebFlux

| 特性 | Spring MVC | Spring WebFlux |
|------|-----------|---------------|
| 编程模型 | 同步阻塞 | 异步非阻塞 |
| 底层 | Servlet API | Reactive Streams |
| 适用场景 | 传统Web应用 | 高并发、流处理 |
| 学习曲线 | 低 | 高 |
| 数据库支持 | JDBC/JPA | R2DBC/Mongo Reactive |

**选择建议：** 大多数场景Spring MVC足够；需要高并发或事件流处理用WebFlux。

---

## 性能分析与优化

### 1. 静态资源处理

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/static/**")
                .addResourceLocations("/static/")
                .setCacheControl(CacheControl.maxAge(365, TimeUnit.DAYS)); // 长期缓存
    }
    
    @Override
    public void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer) {
        configurer.enable(); // 让容器默认Servlet处理静态资源
    }
}
```

### 2. 视图解析器缓存

```java
@Configuration
public class MvcConfig implements WebMvcConfigurer {
    @Bean
    public ViewResolver viewResolver() {
        InternalResourceViewResolver resolver = new InternalResourceViewResolver();
        resolver.setPrefix("/WEB-INF/views/");
        resolver.setSuffix(".jsp");
        resolver.setCache(true); // 生产环境开启缓存
        return resolver;
    }
}
```

### 3. 异步请求处理

```java
@RestController
public class AsyncController {
    
    @GetMapping("/async/data")
    public Callable<String> getDataAsync() {
        return () -> {
            // 在独立线程中执行
            Thread.sleep(5000);
            return "异步数据";
        };
    }
    
    @GetMapping("/async/deferred")
    public DeferredResult<String> getDeferredResult() {
        DeferredResult<String> result = new DeferredResult<>();
        
        // 在另一个线程中设置结果
        CompletableFuture.runAsync(() -> {
            try {
                Thread.sleep(5000);
                result.setResult("延迟结果");
            } catch (InterruptedException e) {
                result.setErrorResult(e);
            }
        });
        
        return result;
    }
}
```

### 4. HTTP缓存控制

```java
@RestController
public class CacheController {
    
    @GetMapping("/users/{id}")
    public ResponseEntity<User> getUser(@PathVariable Long id) {
        User user = userService.getUser(id);
        
        return ResponseEntity.ok()
                .cacheControl(CacheControl.maxAge(60, TimeUnit.SECONDS))
                .eTag(String.valueOf(user.getVersion()))
                .body(user);
    }
}
```

---

## 常见陷阱与最佳实践

### 陷阱1：在Controller中写业务逻辑

```java
@Controller
public class OrderController {
    @PostMapping("/order")
    public String createOrder(OrderDTO dto) {
        // 直接在Controller中写业务逻辑
        if (dto.getAmount().compareTo(BigDecimal.ZERO) <= 0) {
            return "error";
        }
        // 调用DAO、处理业务... 
        // 错误！Controller应该只负责请求处理和视图跳转
    }
}
```

**最佳实践：** Controller层只做参数校验、调用Service、返回结果，业务逻辑放在Service层。

### 陷阱2：忽略HTTP方法语义

```java
@Controller
public class UserController {
    @RequestMapping("/user/delete") // 应该用DELETE或POST
    public String deleteUser(Long id) {
        userService.delete(id);
        return "success";
    }
}
```

**最佳实践：** 遵循RESTful规范，GET用于查询，POST用于创建，PUT用于更新，DELETE用于删除。

```java
@DeleteMapping("/user/{id}")
public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
    userService.delete(id);
    return ResponseEntity.ok().build();
}
```

### 陷阱3：全局异常处理缺失

```java
@Controller
public class OrderController {
    @GetMapping("/order/{id}")
    public Order getOrder(@PathVariable Long id) {
        return orderService.getById(id); // 可能抛异常，返回500页面
    }
}
```

**最佳实践：** 使用`@ControllerAdvice`统一处理异常，返回统一的错误响应。

```java
@ControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(BusinessException.class)
    @ResponseBody
    public Result handleBusinessException(BusinessException e) {
        return Result.fail(e.getCode(), e.getMessage());
    }
}
```

### 陷阱4：拦截器链顺序混乱

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new AuthInterceptor());
        registry.addInterceptor(new LogInterceptor());
        // 顺序不清晰，可能导致日志记录未授权访问的请求
    }
}
```

**最佳实践：** 明确拦截器顺序，权限拦截器通常放在前面，日志拦截器放在最后。

```java
@Override
public void addInterceptors(InterceptorRegistry registry) {
    registry.addInterceptor(new AuthInterceptor()).order(1);
    registry.addInterceptor(new LogInterceptor()).order(2);
}
```

### 陷阱5：忽略请求参数校验

```java
@PostMapping("/user")
public User createUser(@RequestBody UserDTO dto) {
    // 没有校验，可能导致无效数据进入系统
    return userService.save(dto);
}
```

**最佳实践：** 使用Bean Validation进行参数校验。

```java
@PostMapping("/user")
public User createUser(@RequestBody @Valid UserDTO dto) {
    return userService.save(dto);
}

public class UserDTO {
    @NotBlank(message = "用户名不能为空")
    private String username;
    
    @Email(message = "邮箱格式不正确")
    private String email;
    
    @Min(value = 0, message = "年龄不能为负数")
    private Integer age;
}
```

### 陷阱6：忽视Spring Boot的自动配置

```java
// 错误：在Spring Boot中手动配置DispatcherServlet
@Configuration
public class WebConfig {
    @Bean
    public ServletRegistrationBean dispatcherServlet() {
        return new ServletRegistrationBean(new DispatcherServlet(), "/");
    }
}
```

**最佳实践：** Spring Boot已经自动配置了DispatcherServlet，无需手动配置。

---

## 面试题与参考答案

### Q1：请描述Spring MVC的完整请求处理流程。

**答：** 
1. 客户端发送HTTP请求
2. `DispatcherServlet`接收请求（前端控制器）
3. `HandlerMapping`根据URL找到对应的Handler（Controller方法）
4. 执行`HandlerInterceptor.preHandle()`拦截器前置处理
5. `HandlerAdapter`调用Handler执行Controller方法
6. Controller处理业务，返回`ModelAndView`（或`@ResponseBody`直接返回数据）
7. 执行`HandlerInterceptor.postHandle()`拦截器后置处理
8. `ViewResolver`解析视图名，找到具体View对象
9. `View`渲染视图，生成HTML/JSON响应
10. 执行`HandlerInterceptor.afterCompletion()`拦截器完成处理
11. 返回HTTP响应给客户端

### Q2：Spring MVC的9大组件是什么？各自的作用？

**答：** 
1. **DispatcherServlet**：前端控制器，统一接收和分发请求
2. **HandlerMapping**：请求与处理器的映射，如`RequestMappingHandlerMapping`
3. **HandlerAdapter**：适配不同的处理器类型，如注解方式、接口方式
4. **HandlerInterceptor**：拦截器，提供preHandle、postHandle、afterCompletion
5. **Controller**：业务处理器，处理具体请求
6. **ModelAndView**：模型数据和视图信息的封装
7. **ViewResolver**：视图解析器，将逻辑视图名解析为具体View
8. **View**：视图渲染，如JSP、Thymeleaf、JSON
9. **HandlerExceptionResolver**：异常处理器，统一处理Controller抛出的异常

### Q3：@RequestBody和@RequestParam有什么区别？

**答：** 
- **@RequestParam**：用于接收URL参数（query string）或表单数据。适用于GET请求和简单POST表单提交。
- **@RequestBody**：用于接收请求体中的JSON/XML数据，Spring自动将JSON反序列化为对象。适用于POST/PUT请求的JSON数据。

```java
@GetMapping("/user")
public User getUser(@RequestParam Long id) { } // /user?id=1

@PostMapping("/user")
public User createUser(@RequestBody UserDTO dto) { } // JSON in body
```

### Q4：Spring MVC和Spring Boot有什么区别和联系？

**答：** 
- **Spring MVC**是Spring框架的Web模块，提供MVC架构的Web开发支持，需要手动配置DispatcherServlet、视图解析器等。
- **Spring Boot**是快速开发框架，**内置了Spring MVC**（spring-boot-starter-web），自动配置DispatcherServlet和默认组件，提供嵌入式Tomcat。

**联系：** Spring Boot基于Spring MVC构建，简化了配置和部署。使用Spring Boot开发Web应用时，底层仍是Spring MVC处理请求。

### Q5：HandlerInterceptor和Filter有什么区别？

**答：** 

| 特性 | HandlerInterceptor | Filter |
|------|-------------------|--------|
| 归属 | Spring MVC | Servlet规范 |
| 执行时机 | DispatcherServlet之后，Controller之前 | Servlet容器层面，最早执行 |
| 获取Bean | 可以注入Spring Bean | 无法直接注入Spring Bean |
| 灵活性 | 可以访问Controller信息 | 更底层，处理所有请求 |
| 配置方式 | 实现WebMvcConfigurer | web.xml或@WebFilter |

执行顺序：Filter -> DispatcherServlet -> Interceptor -> Controller

### Q6：@Controller和@RestController有什么区别？

**答：** 
- **@Controller**：标识Spring MVC控制器，方法返回视图名或ModelAndView。需要配合`@ResponseBody`才能返回JSON。
- **@RestController**：组合注解，等同于`@Controller + @ResponseBody`。所有方法默认返回JSON/XML数据，不经过视图解析器。

```java
@RestController
public class UserController {
    @GetMapping("/user/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.getById(id); // 自动转为JSON
    }
}
```

### Q7：DispatcherServlet的初始化流程是什么？

**答：** DispatcherServlet继承自FrameworkServlet，初始化时会调用`onRefresh()`方法，初始化9大组件：
1. MultipartResolver（文件上传解析器）
2. LocaleResolver（国际化解析器）
3. ThemeResolver（主题解析器）
4. HandlerMappings（处理器映射器列表）
5. HandlerAdapters（处理器适配器列表）
6. HandlerExceptionResolvers（异常处理器列表）
7. RequestToViewNameTranslator（视图名转换器）
8. ViewResolvers（视图解析器列表）
9. FlashMapManager（重定向参数管理器）

### Q8：Spring MVC如何处理静态资源？

**答：** Spring MVC提供多种方式处理静态资源：
1. **addResourceHandlers**：配置资源处理器，如`/static/**`映射到`/static/`目录
2. **defaultServletHandling**：启用容器默认Servlet处理静态资源
3. **mvc:resources**（XML配置）：配置资源映射

最佳实践：开发环境关闭缓存，生产环境开启长期缓存。

### Q9：Spring MVC的HandlerAdapter有哪些实现？

**答：** 
- **RequestMappingHandlerAdapter**：处理@RequestMapping方法（最常用）
- **HttpRequestHandlerAdapter**：处理HttpRequestHandler接口
- **SimpleControllerHandlerAdapter**：处理Controller接口
- **SimpleServletHandlerAdapter**：处理Servlet接口

### Q10：如何在Spring MVC中实现文件上传？

**答：** 
1. 配置MultipartResolver
```java
@Bean
public MultipartResolver multipartResolver() {
    CommonsMultipartResolver resolver = new CommonsMultipartResolver();
    resolver.setMaxUploadSize(10485760); // 10MB
    return resolver;
}
```

2. 编写Controller
```java
@PostMapping("/upload")
public String upload(@RequestParam("file") MultipartFile file) {
    // 处理文件上传
    String fileName = file.getOriginalFilename();
    file.transferTo(new File("/uploads/" + fileName));
    return "success";
}
```

### Q11：Spring MVC的请求参数绑定有哪些方式？

**答：** 
- **@RequestParam**：绑定URL参数或表单数据
- **@PathVariable**：绑定URL路径变量
- **@RequestBody**：绑定请求体JSON/XML数据
- **@ModelAttribute**：绑定表单数据到对象
- **@RequestHeader**：绑定请求头
- **@CookieValue**：绑定Cookie值

### Q12：Spring MVC的视图解析器有哪些？

**答：** 
- **InternalResourceViewResolver**：解析JSP视图
- **ThymeleafViewResolver**：解析Thymeleaf模板
- **FreeMarkerViewResolver**：解析FreeMarker模板
- **VelocityViewResolver**：解析Velocity模板（已废弃）
- **ContentNegotiatingViewResolver**：根据请求内容类型选择视图

### Q13：如何在Spring MVC中实现跨域（CORS）？

**答：** 
1. 全局配置：
```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
                .allowedOrigins("*")
                .allowedMethods("GET", "POST", "PUT", "DELETE")
                .allowedHeaders("*")
                .maxAge(3600);
    }
}
```

2. 注解配置：
```java
@CrossOrigin(origins = "*", maxAge = 3600)
@RestController
public class UserController { }
```

### Q14：Spring MVC的请求处理流程中，HandlerMapping和HandlerAdapter有什么区别？

**答：** 
- **HandlerMapping**：负责根据请求的URL找到对应的Handler（Controller方法）。它只负责"找到"处理器，不执行处理器。
- **HandlerAdapter**：负责"调用"Handler。由于Spring MVC支持多种类型的Handler（注解Controller、接口Controller等），HandlerAdapter提供了统一的调用接口。

### Q15：如何在Spring MVC中自定义参数解析器？

**答：** 实现`HandlerMethodArgumentResolver`接口：

```java
public class CurrentUserArgumentResolver implements HandlerMethodArgumentResolver {
    
    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.getParameterType().equals(User.class) &&
               parameter.hasParameterAnnotation(CurrentUser.class);
    }
    
    @Override
    public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
                                  NativeWebRequest webRequest, WebDataBinderFactory binderFactory) {
        // 从请求属性或Session中获取当前用户
        return webRequest.getAttribute("currentUser", RequestAttributes.SCOPE_REQUEST);
    }
}
```

注册解析器：
```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
        resolvers.add(new CurrentUserArgumentResolver());
    }
}
```

### Q16：Spring MVC的异常处理机制是怎样的？

**答：** Spring MVC的异常处理流程：
1. Controller抛出异常
2. `DispatcherServlet`捕获异常
3. `HandlerExceptionResolver`链处理异常
4. 如果异常未被处理，`DispatcherServlet`抛出给Servlet容器

自定义异常处理器：
```java
@Component
public class CustomExceptionResolver implements HandlerExceptionResolver {
    @Override
    public ModelAndView resolveException(HttpServletRequest request, HttpServletResponse response,
                                         Object handler, Exception ex) {
        if (ex instanceof BusinessException) {
            ModelAndView mav = new ModelAndView("error");
            mav.addObject("message", ex.getMessage());
            return mav;
        }
        return null; // 返回null表示不处理，继续下一个Resolver
    }
}
```

### Q17：Spring MVC中的Model、ModelMap、ModelAndView有什么区别？

**答：** 
- **Model**：接口，提供`addAttribute`方法添加数据
- **ModelMap**：实现了Model接口，继承自LinkedHashMap，可以直接使用Map方法
- **ModelAndView**：包含模型数据和视图信息，可以在一个返回值中同时指定数据和视图

### Q18：如何在Spring MVC中实现请求参数的全局格式化？

**答：** 使用`Formatter`或`Converter`：

```java
@Component
public class DateFormatter implements Formatter<LocalDate> {
    @Override
    public LocalDate parse(String text, Locale locale) {
        return LocalDate.parse(text, DateTimeFormatter.ISO_LOCAL_DATE);
    }
    
    @Override
    public String print(LocalDate object, Locale locale) {
        return object.format(DateTimeFormatter.ISO_LOCAL_DATE);
    }
}
```

注册Formatter：
```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addFormatters(FormatterRegistry registry) {
        registry.addFormatter(new DateFormatter());
    }
}
```

### Q19：Spring MVC的HandlerInterceptor和WebFilter有什么区别？

**答：** 
- **HandlerInterceptor**：Spring MVC层面，可以访问Controller信息，通过`WebMvcConfigurer`配置
- **WebFilter**：Servlet规范层面，更底层，处理所有请求，通过`@WebFilter`或`FilterRegistrationBean`配置

执行顺序：Filter → DispatcherServlet → Interceptor → Controller

### Q20：如何在Spring MVC中实现请求限流？

**答：** 使用拦截器实现简单的限流：

```java
@Component
public class RateLimitInterceptor implements HandlerInterceptor {
    
    private final Map<String, RateLimiter> limiters = new ConcurrentHashMap<>();
    
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, 
                             Object handler) throws Exception {
        String clientId = getClientId(request);
        RateLimiter limiter = limiters.computeIfAbsent(clientId, k -> RateLimiter.create(10)); // 每秒10个请求
        
        if (!limiter.tryAcquire()) {
            response.setStatus(429); // Too Many Requests
            response.getWriter().write("请求过于频繁，请稍后再试");
            return false;
        }
        return true;
    }
    
    private String getClientId(HttpServletRequest request) {
        return request.getRemoteAddr();
    }
}
```

### Q21：Spring MVC的RequestMappingHandlerMapping是如何初始化的？

**答：** `RequestMappingHandlerMapping`在应用启动时初始化，会扫描Spring容器中所有带有`@Controller`或`@RequestMapping`注解的Bean，然后查找这些Bean中带有`@RequestMapping`注解的方法，建立URL到HandlerMethod的映射关系。映射信息存储在内部的`MappingRegistry`中。

### Q22：Spring MVC中的DataBinder有什么作用？

**答：** `DataBinder`负责将请求参数绑定到Java对象，包括：
- 类型转换（String → Integer, Date等）
- 数据格式化（日期格式、货币格式等）
- 数据校验（JSR-303 Bean Validation）
- 自定义属性编辑器

### Q23：Spring MVC的FlashAttributes是什么？

**答：** `FlashAttributes`用于在重定向（redirect）时传递数据。由于重定向是两个独立的请求，普通Model数据无法传递。FlashAttributes通过Session临时存储数据，在重定向后的请求中自动移除。

```java
@PostMapping("/user")
public String createUser(@ModelAttribute User user, RedirectAttributes redirectAttrs) {
    User saved = userService.save(user);
    redirectAttrs.addFlashAttribute("message", "用户创建成功");
    return "redirect:/user/" + saved.getId();
}
```

---

*此文原创，转载请注明出处。*
