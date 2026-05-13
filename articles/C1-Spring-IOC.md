# Spring IOC深度解析：容器启动与依赖注入全链路源码分析

**文章标签：** #java #spring #ioc #依赖注入 #源码分析 #设计模式 #面试 #深度解析

## 目录

- [引言：为什么需要IOC](#引言为什么需要ioc)
- [理论基础：控制反转与依赖注入](#理论基础控制反转与依赖注入)
- [来龙去脉：Spring IOC的发展史](#来龙去脉spring-ioc的发展史)
- [核心架构与源码深度分析](#核心架构与源码深度分析)
- [BeanDefinition详解与源码分析](#beandefinition详解与源码分析)
- [容器启动流程源码级分析](#容器启动流程源码级分析)
- [Bean创建流程源码深度分析](#bean创建流程源码深度分析)
- [设计模式解析](#设计模式解析)
- [实战案例：工业级应用](#实战案例工业级应用)
- [对比分析：Spring IOC vs 替代方案](#对比分析spring-ioc-vs-替代方案)
- [性能分析与优化](#性能分析与优化)
- [常见陷阱与最佳实践](#常见陷阱与最佳实践)
- [面试题与参考答案](#面试题与参考答案)

---

## 引言：为什么需要IOC

IOC（Inversion of Control，控制反转）不是简单的"对象创建工厂"，而是一种**解耦软件组件依赖关系**的架构思想。在传统的Java开发中，对象之间的依赖关系是硬编码在程序中的，导致代码高度耦合、难以测试和维护。

**核心认知：**

```
传统编程模式：
┌─────────────────────────────────────────┐
│  UserService                            │
│  ├─ new UserDaoImpl()  ← 强耦合         │
│  ├─ new OrderService()                  │
│  └─ new EmailService()                  │
└─────────────────────────────────────────┘
问题：
- 更换实现需要修改源代码
- 无法灵活切换环境（开发/测试/生产）
- 单元测试困难（无法mock依赖）
- 对象生命周期管理混乱

IOC模式：
┌─────────────────────────────────────────┐
│  ApplicationContext (容器)               │
│  ├─ UserDao ←──┐                        │
│  ├─ UserService ← 依赖注入              │
│  └─ 由容器统一管理生命周期               │
└─────────────────────────────────────────┘
优势：
- 依赖关系外部化配置
- 实现松耦合架构
- 支持AOP等高级特性
- 统一的对象生命周期管理
```

**关键洞察**：IOC的本质是**好莱坞原则**（Don't call us, we'll call you）——组件不再主动创建依赖，而是由容器在合适的时机将依赖注入给组件。

---

## 理论基础：控制反转与依赖注入

### 1. 控制反转的核心思想

控制反转是一种设计原则，它将程序的控制权从应用代码转移给外部容器或框架。

```
控制权转移的维度：

1. 对象创建的控制权
   传统：程序员在代码中 new Object()
   IOC：容器根据配置创建对象

2. 对象依赖的控制权
   传统：对象内部主动获取依赖
   IOC：容器将依赖注入到对象中

3. 对象生命周期的控制权
   传统：程序员手动管理（创建、初始化、销毁）
   IOC：容器统一管理生命周期

4. 配置信息的控制权
   传统：硬编码在程序中
   IOC：外部化配置（XML、注解、properties）
```

### 2. 依赖注入的三种方式

#### 构造器注入（Constructor Injection）

```java
@Service
public class UserService {
    private final UserDao userDao;
    private final EmailService emailService;
    
    // 必需依赖通过构造器注入
    @Autowired
    public UserService(UserDao userDao, EmailService emailService) {
        this.userDao = userDao;
        this.emailService = emailService;
    }
}
```

**优点：**
- 依赖明确，必需依赖不可为空
- 可以使用final关键字，保证不可变性
- 便于单元测试（直接传入mock对象）
- 循环依赖在启动时就能发现

**缺点：**
- 参数过多时构造器臃肿
- 可选依赖处理不够灵活

#### Setter注入（Setter Injection）

```java
@Service
public class UserService {
    private UserDao userDao;
    private Optional<CacheService> cacheService;
    
    // 必需依赖
    @Autowired
    public void setUserDao(UserDao userDao) {
        this.userDao = userDao;
    }
    
    // 可选依赖
    @Autowired(required = false)
    public void setCacheService(CacheService cacheService) {
        this.cacheService = Optional.ofNullable(cacheService);
    }
}
```

**优点：**
- 可选依赖处理灵活
- 可以在创建后重新注入（动态替换）
- 解决部分循环依赖问题

**缺点：**
- 对象可能处于不完整状态
- 依赖可能在运行时被修改

#### 字段注入（Field Injection）

```java
@Service
public class UserService {
    @Autowired
    private UserDao userDao;
    
    @Autowired
    private EmailService emailService;
}
```

**优点：**
- 代码简洁
- 无需编写构造器或Setter

**缺点：**
- 无法使用final
- 不便于单元测试（需要反射）
- 隐藏了类的依赖关系
- 不推荐在正式项目中使用

### 3. IOC容器的理论基础

```
容器的核心职责：

┌─────────────────────────────────────────┐
│           IOC Container                  │
├─────────────────────────────────────────┤
│  1. 配置读取与解析                        │
│     - XML配置文件解析                     │
│     - 注解扫描与处理                      │
│     - Java Config配置                     │
├─────────────────────────────────────────┤
│  2. BeanDefinition管理                   │
│     - 存储Bean的元数据                    │
│     - 管理Bean之间的依赖关系              │
├─────────────────────────────────────────┤
│  3. 对象实例化与组装                      │
│     - 根据BeanDefinition创建对象          │
│     - 注入依赖（DI）                      │
│     - 处理循环依赖                        │
├─────────────────────────────────────────┤
│  4. 生命周期管理                          │
│     - 初始化回调（@PostConstruct）        │
│     - 销毁回调（@PreDestroy）             │
│     - Aware接口回调                       │
├─────────────────────────────────────────┤
│  5. 扩展点支持                            │
│     - BeanFactoryPostProcessor            │
│     - BeanPostProcessor                   │
│     - 事件发布机制                        │
└─────────────────────────────────────────┘
```

---

## 来龙去脉：Spring IOC的发展史

### 第一阶段：XML配置时代（2004-2009）

Spring 1.x/2.x时期，IOC容器主要基于XML配置：

```xml
<!-- application-context.xml -->
<beans>
    <bean id="userDao" class="com.example.dao.UserDaoImpl"/>
    <bean id="userService" class="com.example.service.UserService">
        <property name="userDao" ref="userDao"/>
    </bean>
</beans>
```

**特点：**
- 配置与代码分离
- 类型不安全（运行时才能发现错误）
- 冗长繁琐

### 第二阶段：注解驱动时代（2009-2014）

Spring 2.5引入注解支持，Spring 3.x大力推广：

```java
@Service
public class UserService {
    @Autowired
    private UserDao userDao;
}
```

**特点：**
- 配置内聚在代码中
- 编译期类型安全
- 简化配置

### 第三阶段：Java Config时代（2013至今）

Spring 3.0引入`@Configuration`，Spring Boot时代全面推广：

```java
@Configuration
public class AppConfig {
    @Bean
    public UserDao userDao() {
        return new UserDaoImpl();
    }
    
    @Bean
    public UserService userService(UserDao userDao) {
        return new UserService(userDao);
    }
}
```

**特点：**
- 类型安全（编译期检查）
- 支持复杂的配置逻辑
- 便于重构

### 第四阶段：自动配置时代（2014至今）

Spring Boot的`@SpringBootApplication`：

```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

**特点：**
- 约定优于配置
- 自动装配依赖
- 内嵌容器

---

## 核心架构与源码深度分析

### Spring IOC容器架构图

```
┌─────────────────────────────────────────────────────────────┐
│                    Spring IOC 容器架构                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────────┐│
│  │              ApplicationContext（应用上下文）             ││
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────────┐ ││
│  │  │ MessageSource│ │EventPublisher│ │ResourcePattern   │ ││
│  │  │   国际化      │ │   事件发布   │ │    Resolver      │ ││
│  │  └──────────────┘ └──────────────┘ └──────────────────┘ ││
│  └─────────────────────────────────────────────────────────┘│
│                              │                              │
│  ┌─────────────────────────────────────────────────────────┐│
│  │              BeanFactory（Bean工厂）                     ││
│  │  ┌─────────────────────────────────────────────────────┐││
│  │  │         DefaultListableBeanFactory                   │││
│  │  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │││
│  │  │  │BeanDefinition│  │ singleton   │  │  early      │  │││
│  │  │  │   Registry   │  │  Objects    │  │ Singleton   │  │││
│  │  │  │              │  │  (一级缓存)  │  │  Objects    │  │││
│  │  │  └─────────────┘  └─────────────┘  └─────────────┘  │││
│  │  │  ┌─────────────────────────────────────────────────┐  │││
│  │  │  │  AbstractAutowireCapableBeanFactory              │  │││
│  │  │  │  - createBean()                                  │  │││
│  │  │  │  - autowireBean()                                │  │││
│  │  │  │  - initializeBean()                              │  │││
│  │  │  └─────────────────────────────────────────────────┘  │││
│  │  └─────────────────────────────────────────────────────┘││
│  └─────────────────────────────────────────────────────────┘│
│                              │                              │
│  ┌─────────────────────────────────────────────────────────┐│
│  │              BeanPostProcessor 扩展链                    ││
│  │  ┌─────────────┐ ┌─────────────┐ ┌────────────────────┐ ││
│  │  │ Autowired   │ │ Common      │ │ AbstractAutoProxy  │ ││
│  │  │ Annotation  │ │ Annotation  │ │ Creator (AOP代理)  │ ││
│  │  │ BeanPost    │ │ BeanPost    │ │                    │ ││
│  │  │ Processor   │ │ Processor   │ │                    │ ││
│  │  └─────────────┘ └─────────────┘ └────────────────────┘ ││
│  └─────────────────────────────────────────────────────────┘│
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### BeanFactory 源码深度分析

`BeanFactory`是Spring IOC最基础的接口，定义在`org.springframework.beans.factory.BeanFactory`：

```java
public interface BeanFactory {
    String FACTORY_BEAN_PREFIX = "&";
    
    // 按名称获取Bean
    Object getBean(String name) throws BeansException;
    
    // 按名称和类型获取
    <T> T getBean(String name, Class<T> requiredType) throws BeansException;
    
    // 按类型获取
    <T> T getBean(Class<T> requiredType) throws BeansException;
    
    // 按名称获取，附带构造参数
    Object getBean(String name, Object... args) throws BeansException;
    
    // 是否存在
    boolean containsBean(String name);
    
    // 是否单例
    boolean isSingleton(String name) throws NoSuchBeanDefinitionException;
    
    // 是否原型
    boolean isPrototype(String name) throws NoSuchBeanDefinitionException;
    
    // 获取类型
    Class<?> getType(String name) throws NoSuchBeanDefinitionException;
    
    // 获取别名
    String[] getAliases(String name);
}
```

**关键实现类层级：**

```
BeanFactory (接口)
  └── HierarchicalBeanFactory (接口，支持层级)
        └── ConfigurableBeanFactory (接口，配置能力)
              └── AbstractBeanFactory (抽象类，核心实现)
                    └── AbstractAutowireCapableBeanFactory (自动装配)
                          └── DefaultListableBeanFactory (完整实现)
```

**DefaultListableBeanFactory**是最常用的完整实现，包含：

```java
public class DefaultListableBeanFactory extends AbstractAutowireCapableBeanFactory
        implements ConfigurableListableBeanFactory, BeanDefinitionRegistry, Serializable {
    
    // BeanDefinition注册表，核心数据结构
    private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>(256);
    
    // 单例Bean缓存（一级缓存）
    private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);
    
    // 早期单例Bean缓存（二级缓存）
    private final Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>(16);
    
    // 单例Bean工厂缓存（三级缓存）
    private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
    
    // 按类型缓存Bean名称
    private final Map<Class<?>, String[]> allBeanNamesByType = new ConcurrentHashMap<>(64);
    
    // 按类型缓存单例Bean名称
    private final Map<Class<?>, String[]> singletonBeanNamesByType = new ConcurrentHashMap<>(64);
    
    // BeanDefinition名称列表，保证注册顺序
    private volatile List<String> beanDefinitionNames = new ArrayList<>(256);
    
    // 手动注册的单例名称集合
    private volatile Set<String> manualSingletonNames = new LinkedHashSet<>(16);
}
```

### ApplicationContext 源码深度分析

`ApplicationContext`继承多个接口，是完整的企业级容器：

```java
public interface ApplicationContext extends EnvironmentCapable, ListableBeanFactory, 
        HierarchicalBeanFactory, MessageSource, ApplicationEventPublisher, ResourcePatternResolver {
    
    // 获取应用名称
    String getApplicationName();
    
    // 获取应用启动时间
    long getStartupDate();
    
    // 获取父容器
    ApplicationContext getParent();
    
    // 获取自动装配的BeanFactory
    AutowireCapableBeanFactory getAutowireCapableBeanFactory() throws IllegalStateException;
    
    // 获取唯一标识
    String getDisplayName();
    
    // 获取ID
    String getId();
}
```

**核心实现类：**

```java
// 注解配置上下文
public class AnnotationConfigApplicationContext extends GenericApplicationContext 
        implements AnnotationConfigRegistry {
    
    // 读取@ComponentScan等注解的读取器
    private final AnnotatedBeanDefinitionReader reader;
    
    // 扫描指定包路径的扫描器
    private final ClassPathBeanDefinitionScanner scanner;
    
    public AnnotationConfigApplicationContext(String... basePackages) {
        this();
        scan(basePackages);
        refresh();
    }
}

// XML配置上下文
public class ClassPathXmlApplicationContext extends AbstractXmlApplicationContext {
    private Resource[] configResources;
    
    public ClassPathXmlApplicationContext(String configLocation) throws BeansException {
        this(new String[]{configLocation}, true, null);
    }
}
```

### ApplicationContext.refresh() 方法源码分析

```java
@Override
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        // 1. 准备刷新上下文
        prepareRefresh();
        
        // 2. 通知子类刷新内部的BeanFactory
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
        
        // 3. 准备BeanFactory供此上下文使用
        prepareBeanFactory(beanFactory);
        
        try {
            // 4. 允许子类后处理BeanFactory
            postProcessBeanFactory(beanFactory);
            
            // 5. 调用所有注册的BeanFactoryPostProcessor
            invokeBeanFactoryPostProcessors(beanFactory);
            
            // 6. 注册BeanPostProcessor，拦截Bean创建
            registerBeanPostProcessors(beanFactory);
            
            // 7. 初始化MessageSource用于国际化
            initMessageSource();
            
            // 8. 初始化事件广播器
            initApplicationEventMulticaster();
            
            // 9. 初始化特定上下文子类中的其他特殊Bean
            onRefresh();
            
            // 10. 注册监听器
            registerListeners();
            
            // 11. 实例化所有剩余的非懒加载单例Bean
            finishBeanFactoryInitialization(beanFactory);
            
            // 12. 最后一步：发布对应事件
            finishRefresh();
        }
        catch (BeansException ex) {
            // 销毁已创建的单例Bean以避免资源占用
            destroyBeans();
            // 关闭BeanFactory
            closeBeanFactory();
            throw ex;
        }
    }
}
```

---

## BeanDefinition详解与源码分析

### BeanDefinition的属性详解

```java
public class RootBeanDefinition extends AbstractBeanDefinition {
    // 1. Bean的类信息
    private volatile Object beanClass;
    
    // 2. 作用域
    private String scope = SCOPE_DEFAULT; // singleton/prototype/request/session
    
    // 3. 是否抽象
    private boolean abstractFlag = false;
    
    // 4. 是否延迟初始化
    private boolean lazyInit = false;
    
    // 5. 自动装配模式
    private int autowireMode = AUTOWIRE_NO; // no/byName/byType/constructor
    
    // 6. 依赖检查
    private int dependencyCheck = DEPENDENCY_CHECK_NONE;
    
    // 7. 构造函数参数
    private ConstructorArgumentValues constructorArgumentValues;
    
    // 8. 属性值
    private MutablePropertyValues propertyValues;
    
    // 9. 初始化方法
    private String initMethodName;
    
    // 10. 销毁方法
    private String destroyMethodName;
    
    // 11. 工厂方法
    private String factoryMethodName;
    private String factoryBeanName;
    
    // 12. 后置处理器
    private volatile BeanDefinitionHolder decoratedDefinition;
}
```

### BeanDefinition的来源

1. **XML配置**：`<bean>`标签解析
2. **注解配置**：`@Component`、`@Service`等扫描
3. **Java Config**：`@Bean`方法解析
4. **编程方式**：`BeanDefinitionBuilder`手动创建

### BeanDefinition的合并

```java
// AbstractBeanFactory.getMergedBeanDefinition()
protected RootBeanDefinition getMergedBeanDefinition(
        String beanName, BeanDefinition bd, @Nullable BeanDefinition containingBd)
        throws BeanDefinitionStoreException {
    
    synchronized (this.mergedBeanDefinitions) {
        RootBeanDefinition mbd = null;
        
        // 检查缓存
        if (containingBd == null) {
            mbd = this.mergedBeanDefinitions.get(beanName);
        }
        
        if (mbd == null) {
            if (bd.getParentName() == null) {
                // 没有父定义，直接克隆
                if (bd instanceof RootBeanDefinition) {
                    mbd = ((RootBeanDefinition) bd).cloneBeanDefinition();
                } else {
                    mbd = new RootBeanDefinition(bd);
                }
            } else {
                // 有父定义，需要合并
                BeanDefinition pbd;
                try {
                    String parentBeanName = transformedBeanName(bd.getParentName());
                    if (!beanName.equals(parentBeanName)) {
                        pbd = getMergedBeanDefinition(parentBeanName);
                    } else {
                        // 父定义在当前工厂
                        BeanFactory parent = getParentBeanFactory();
                        if (parent instanceof ConfigurableBeanFactory) {
                            pbd = ((ConfigurableBeanFactory) parent).getMergedBeanDefinition(parentBeanName);
                        } else {
                            throw new NoSuchBeanDefinitionException(parentBeanName,
                                    "Parent name '" + bd.getParentName() + "' is equal to bean name '" +
                                    beanName + "': cannot be resolved without an AbstractBeanFactory parent");
                        }
                    }
                } catch (NoSuchBeanDefinitionException ex) {
                    throw new BeanDefinitionStoreException(bd.getResourceDescription(), beanName,
                            "Could not resolve parent bean definition '" + bd.getParentName() + "'", ex);
                }
                // 深拷贝父定义
                mbd = new RootBeanDefinition(pbd);
                // 用子定义覆盖
                mbd.overrideFrom(bd);
            }
            
            // 设置默认的作用域
            if (!StringUtils.hasLength(mbd.getScope())) {
                mbd.setScope(RootBeanDefinition.SCOPE_SINGLETON);
            }
            
            // 缓存合并后的定义
            if (containingBd == null && isCacheBeanMetadata()) {
                this.mergedBeanDefinitions.put(beanName, mbd);
            }
        }
        
        return mbd;
    }
}
```

---

## 容器启动流程源码级分析

### 完整启动时序图

```
Client
  │
  ▼
AnnotationConfigApplicationContext(String...)
  │
  ├──► scan(basePackages) ──► ClassPathBeanDefinitionScanner.doScan()
  │                              │
  │                              ├──► findCandidateComponents()
  │                              │       └──► 扫描.class文件，匹配@Filter
  │                              │
  │                              └──► registerBeanDefinition()
  │                                      └──► DefaultListableBeanFactory.registerBeanDefinition()
  │                                              └──► beanDefinitionMap.put(beanName, beanDefinition)
  │
  └──► refresh()
          │
          ├──► prepareRefresh()
          │       └──► 初始化环境、验证必要属性、准备事件监听器集合
          │
          ├──► obtainFreshBeanFactory()
          │       └──► refreshBeanFactory()
          │               └──► 加载BeanDefinition到BeanFactory
          │
          ├──► prepareBeanFactory(beanFactory)
          │       └──► 设置类加载器、添加PropertyEditor、注册默认环境Bean、
          │            添加BeanPostProcessor（ApplicationContextAwareProcessor）
          │
          ├──► postProcessBeanFactory(beanFactory)
          │       └──► 子类扩展（Web应用注册request/session作用域）
          │
          ├──► invokeBeanFactoryPostProcessors(beanFactory)
          │       └──► ConfigurationClassPostProcessor.processConfigBeanDefinitions()
          │               └──► 解析@Configuration类，处理@Bean、@Import、@ComponentScan等
          │
          ├──► registerBeanPostProcessors(beanFactory)
          │       └──► 按优先级注册所有BeanPostProcessor到容器
          │            PriorityOrdered → Ordered → 普通
          │
          ├──► initMessageSource()
          │       └──► 初始化国际化消息源
          │
          ├──► initApplicationEventMulticaster()
          │       └──► 初始化事件广播器（SimpleApplicationEventMulticaster）
          │
          ├──► onRefresh()
          │       └──► 子类扩展（Spring Boot启动内嵌Tomcat）
          │
          ├──► registerListeners()
          │       └──► 注册ApplicationListener到事件广播器
          │
          ├──► finishBeanFactoryInitialization(beanFactory)
          │       └──► preInstantiateSingletons()
          │               └──► getBean(beanName)
          │                       └──► doGetBean()
          │                               └──► createBean()
          │                                       └──► doCreateBean()
          │                                               └──► 实例化 → 属性赋值 → 初始化
          │
          └──► finishRefresh()
                  └──► 发布ContextRefreshedEvent
```

### prepareRefresh() 源码分析

```java
protected void prepareRefresh() {
    // 记录启动时间
    this.startupDate = System.currentTimeMillis();
    // 关闭标志重置
    this.closed.set(false);
    // 激活标志设置
    this.active.set(true);
    
    // 初始化环境（子类可扩展）
    initPropertySources();
    
    // 验证必要的环境属性
    getEnvironment().validateRequiredProperties();
    
    // 准备早期应用事件监听器
    if (this.earlyApplicationListeners == null) {
        this.earlyApplicationListeners = new LinkedHashSet<>(this.applicationListeners);
    } else {
        this.applicationListeners.clear();
        this.applicationListeners.addAll(this.earlyApplicationListeners);
    }
    
    // 准备早期应用事件集合
    this.earlyApplicationEvents = new LinkedHashSet<>();
}
```

### obtainFreshBeanFactory() 源码分析

```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
    // 刷新BeanFactory（对于AnnotationConfigApplicationContext，
    // 实际上是重置BeanFactory状态）
    refreshBeanFactory();
    // 获取BeanFactory
    return getBeanFactory();
}

@Override
protected final void refreshBeanFactory() throws BeansException {
    // 如果已经有BeanFactory，销毁所有Bean并关闭
    if (hasBeanFactory()) {
        destroyBeans();
        closeBeanFactory();
    }
    try {
        // 创建新的DefaultListableBeanFactory
        DefaultListableBeanFactory beanFactory = createBeanFactory();
        // 设置序列化ID
        beanFactory.setSerializationId(getId());
        // 自定义BeanFactory
        customizeBeanFactory(beanFactory);
        // 加载BeanDefinition（子类实现）
        loadBeanDefinitions(beanFactory);
        // 保存引用
        this.beanFactory = beanFactory;
    } catch (IOException ex) {
        throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
    }
}
```

---

## Bean创建流程源码深度分析

### doCreateBean() 完整源码分析

```java
// AbstractAutowireCapableBeanFactory.doCreateBean()
protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
        throws BeanCreationException {
    
    BeanWrapper instanceWrapper = null;
    if (mbd.isSingleton()) {
        instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
    }
    if (instanceWrapper == null) {
        // 1. 实例化Bean
        instanceWrapper = createBeanInstance(beanName, mbd, args);
    }
    Object bean = instanceWrapper.getWrappedInstance();
    Class<?> beanType = instanceWrapper.getWrappedClass();
    
    // 设置bean类型（用于类型匹配）
    if (beanType != NullBean.class) {
        mbd.resolvedTargetType = beanType;
    }
    
    // 允许MergedBeanDefinitionPostProcessor修改BeanDefinition
    synchronized (mbd.postProcessingLock) {
        if (!mbd.postProcessed) {
            try {
                applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
            } catch (Throwable ex) {
                throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                        "Post-processing of merged bean definition failed", ex);
            }
            mbd.postProcessed = true;
        }
    }
    
    // 2. 解决循环依赖：提前暴露ObjectFactory到三级缓存
    boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
            isSingletonCurrentlyInCreation(beanName));
    if (earlySingletonExposure) {
        // 添加单例工厂到三级缓存
        addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
    }
    
    // 初始化返回的对象
    Object exposedObject = bean;
    try {
        // 3. 属性赋值（依赖注入）
        populateBean(beanName, mbd, instanceWrapper);
        
        // 4. 初始化Bean
        exposedObject = initializeBean(beanName, exposedObject, mbd);
    } catch (Throwable ex) {
        if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
            throw (BeanCreationException) ex;
        } else {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
        }
    }
    
    // 5. 检查循环依赖
    if (earlySingletonExposure) {
        Object earlySingletonReference = getSingleton(beanName, false);
        if (earlySingletonReference != null) {
            // 如果exposedObject与原始bean相同，说明没有被代理，使用早期引用
            if (exposedObject == bean) {
                exposedObject = earlySingletonReference;
            }
            // 如果exposedObject被代理，但依赖了原始bean，抛出异常
            else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
                String[] dependentBeans = getDependentBeans(beanName);
                Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
                for (String dependentBean : dependentBeans) {
                    if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                        actualDependentBeans.add(dependentBean);
                    }
                }
                if (!actualDependentBeans.isEmpty()) {
                    throw new BeanCurrentlyInCreationException(beanName,
                            "Bean with name '" + beanName + "' has been injected into other beans [" +
                            StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
                            "] in its raw version as part of a circular reference, but has eventually been " +
                            "wrapped. This means that said other beans do not use the final version of the " +
                            "bean. This is often the result of over-eager type matching - consider using " +
                            "'getBeanNamesForType' with the 'allowEagerInit' flag turned off, for example.");
                }
            }
        }
    }
    
    // 6. 注册销毁回调
    try {
        registerDisposableBeanIfNecessary(beanName, bean, mbd);
    } catch (BeanDefinitionValidationException ex) {
        throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
    }
    
    return exposedObject;
}
```

### 1. 实例化阶段源码分析

```java
// AbstractAutowireCapableBeanFactory.createBeanInstance()
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, Object[] args) {
    // 解析Bean类
    Class<?> beanClass = resolveBeanClass(mbd, beanName);
    
    // 检查类访问权限
    if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                "Bean class isn't public, and non-public access is not allowed: " + beanClass.getName());
    }
    
    // 使用工厂方法实例化
    Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
    if (instanceSupplier != null) {
        return obtainFromSupplier(instanceSupplier, beanName);
    }
    
    // 使用工厂方法
    if (mbd.getFactoryMethodName() != null) {
        return instantiateUsingFactoryMethod(beanName, mbd, args);
    }
    
    // 重新创建相同bean时的优化
    boolean resolved = false;
    boolean autowireNecessary = false;
    if (args == null) {
        synchronized (mbd.constructorArgumentLock) {
            if (mbd.resolvedConstructorOrFactoryMethod != null) {
                resolved = true;
                autowireNecessary = mbd.constructorArgumentsResolved;
            }
        }
    }
    if (resolved) {
        if (autowireNecessary) {
            return autowireConstructor(beanName, mbd, null, null);
        } else {
            return instantiateBean(beanName, mbd);
        }
    }
    
    // 自动装配构造器
    Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
    if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR ||
            mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args)) {
        return autowireConstructor(beanName, mbd, ctors, args);
    }
    
    // 首选构造器（Kotlin类支持）
    ctors = mbd.getPreferredConstructors();
    if (ctors != null) {
        return autowireConstructor(beanName, mbd, ctors, null);
    }
    
    // 使用默认无参构造器
    return instantiateBean(beanName, mbd);
}
```

### 2. 属性赋值阶段源码分析

```java
// AbstractAutowireCapableBeanFactory.populateBean()
protected void populateBean(String beanName, RootBeanDefinition mbd, BeanWrapper bw) {
    // 如果BeanWrapper为空，报错
    if (bw == null) {
        if (mbd.hasPropertyValues()) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
        } else {
            return;
        }
    }
    
    // 给InstantiationAwareBeanPostProcessor一个机会修改bean状态
    if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
        for (InstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().instantiationAware) {
            if (!bp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
                return;
            }
        }
    }
    
    PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);
    
    // 按名称自动装配
    int resolvedAutowireMode = mbd.getResolvedAutowireMode();
    if (resolvedAutowireMode == AUTOWIRE_BY_NAME || resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
        MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
        
        if (resolvedAutowireMode == AUTOWIRE_BY_NAME) {
            autowireByName(beanName, mbd, bw, newPvs);
        }
        
        if (resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
            autowireByType(beanName, mbd, bw, newPvs);
        }
        
        pvs = newPvs;
    }
    
    // 执行InstantiationAwareBeanPostProcessor.postProcessProperties
    // @Autowired、@Resource在这里处理
    boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
    boolean needsDepCheck = (mbd.getDependencyCheck() != AbstractBeanDefinition.DEPENDENCY_CHECK_NONE);
    
    PropertyDescriptor[] filteredPds = null;
    if (hasInstAwareBpps) {
        if (pvs == null) {
            pvs = mbd.getPropertyValues();
        }
        for (InstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().instantiationAware) {
            PropertyValues pvsToUse = bp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
            if (pvsToUse == null) {
                if (filteredPds == null) {
                    filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
                }
                pvsToUse = bp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
                if (pvsToUse == null) {
                    return;
                }
            }
            pvs = pvsToUse;
        }
    }
    
    // 依赖检查
    if (needsDepCheck) {
        if (filteredPds == null) {
            filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
        }
        checkDependencies(beanName, mbd, filteredPds, pvs);
    }
    
    // 应用PropertyValues
    if (pvs != null) {
        applyPropertyValues(beanName, mbd, bw, pvs);
    }
}
```

### 3. 初始化阶段源码分析

```java
// AbstractAutowireCapableBeanFactory.initializeBean()
protected Object initializeBean(String beanName, Object bean, RootBeanDefinition mbd) {
    // 1. 调用Aware接口方法
    invokeAwareMethods(beanName, bean);
    
    // 2. BeanPostProcessor.postProcessBeforeInitialization
    Object wrappedBean = bean;
    if (mbd == null || !mbd.isSynthetic()) {
        wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
    }
    
    // 3. 调用初始化方法
    try {
        invokeInitMethods(beanName, wrappedBean, mbd);
    } catch (Throwable ex) {
        throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Invocation of init method failed", ex);
    }
    
    // 4. BeanPostProcessor.postProcessAfterInitialization（AOP代理在此生成）
    if (mbd == null || !mbd.isSynthetic()) {
        wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    }
    
    return wrappedBean;
}

// 调用Aware接口
private void invokeAwareMethods(String beanName, Object bean) {
    if (bean instanceof Aware) {
        if (bean instanceof BeanNameAware) {
            ((BeanNameAware) bean).setBeanName(beanName);
        }
        if (bean instanceof BeanClassLoaderAware) {
            ClassLoader bcl = getBeanClassLoader();
            if (bcl != null) {
                ((BeanClassLoaderAware) bean).setBeanClassLoader(bcl);
            }
        }
        if (bean instanceof BeanFactoryAware) {
            ((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
        }
    }
}

// 调用初始化方法
protected void invokeInitMethods(String beanName, Object bean, RootBeanDefinition mbd)
        throws Throwable {
    // 1. 调用InitializingBean.afterPropertiesSet()
    boolean isInitializingBean = (bean instanceof InitializingBean);
    if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
        ((InitializingBean) bean).afterPropertiesSet();
    }
    
    // 2. 调用自定义init-method
    if (mbd != null && bean.getClass() != NullBean.class) {
        String initMethodName = mbd.getInitMethodName();
        if (StringUtils.hasLength(initMethodName) &&
                !(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
                !mbd.isExternallyManagedInitMethod(initMethodName)) {
            invokeCustomInitMethod(beanName, bean, mbd);
        }
    }
}
```

---

## 设计模式解析

Spring IOC容器大量使用了经典设计模式：

### 1. 工厂模式（Factory Pattern）

**体现：** `BeanFactory`、`ApplicationContext`

**为什么：** 将对象的创建逻辑封装在工厂中，客户端不需要知道具体的创建细节。Spring通过`BeanFactory`统一创建Bean，支持多种创建方式（构造器、工厂方法、静态工厂）。

```java
// 简单工厂的应用
Object bean = beanFactory.getBean("userService"); // 不需要关心如何创建
```

### 2. 单例模式（Singleton Pattern）

**体现：** `DefaultSingletonBeanRegistry`

**为什么：** 保证一个类只有一个实例，Spring默认作用域就是singleton。通过`ConcurrentHashMap`缓存单例对象，保证线程安全。

```java
// Spring的单例实现（非传统单例模式）
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);
```

### 3. 模板方法模式（Template Method）

**体现：** `AbstractApplicationContext.refresh()`、`AbstractBeanFactory.doGetBean()`

**为什么：** 定义算法骨架，将某些步骤延迟到子类实现。`refresh()`定义了12步启动流程，但`onRefresh()`和`postProcessBeanFactory()`由子类实现。

```java
// refresh()是模板方法
public void refresh() {
    prepareRefresh(); // 固定步骤
    obtainFreshBeanFactory(); // 固定步骤
    // ...
    onRefresh(); // 钩子方法，子类扩展
    // ...
}
```

### 4. 策略模式（Strategy Pattern）

**体现：** `InstantiationStrategy`、`BeanDefinitionReader`

**为什么：** 定义一系列算法，将它们封装起来并可以互相替换。Spring支持多种Bean实例化策略（简单实例化、CGLIB实例化）和多种配置读取策略（XML、注解、Groovy）。

```java
// InstantiationStrategy接口
public interface InstantiationStrategy {
    Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner)
            throws BeansException;
}
// 实现：SimpleInstantiationStrategy、CglibSubclassingInstantiationStrategy
```

### 5. 观察者模式（Observer Pattern）

**体现：** `ApplicationEvent`、`ApplicationListener`、`ApplicationEventMulticaster`

**为什么：** 实现事件驱动架构，降低模块间耦合。Spring容器发布事件，监听器异步或同步接收处理。

```java
// 发布事件
context.publishEvent(new ContextRefreshedEvent(context));

// 监听事件
@Component
public class MyListener implements ApplicationListener<ContextRefreshedEvent> {
    @Override
    public void onApplicationEvent(ContextRefreshedEvent event) {
        // 处理容器刷新完成事件
    }
}
```

### 6. 适配器模式（Adapter Pattern）

**体现：** `BeanDefinitionReader`对不同配置格式的适配

**为什么：** 将不同来源的配置（XML、注解、Properties）统一适配为`BeanDefinition`。

```java
// XmlBeanDefinitionReader读取XML → BeanDefinition
// AnnotatedBeanDefinitionReader读取注解 → BeanDefinition
// PropertiesBeanDefinitionReader读取Properties → BeanDefinition
```

---

## 实战案例：工业级应用

### 案例1：多环境配置管理

```java
@Configuration
public class EnvironmentConfig {
    
    @Bean
    @Profile("dev")
    public DataSource devDataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:h2:mem:testdb");
        config.setUsername("sa");
        config.setPassword("");
        config.setMaximumPoolSize(5);
        return new HikariDataSource(config);
    }
    
    @Bean
    @Profile("test")
    public DataSource testDataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:mysql://test-db:3306/test");
        config.setUsername("test_user");
        config.setPassword("test_pass");
        config.setMaximumPoolSize(10);
        return new HikariDataSource(config);
    }
    
    @Bean
    @Profile("prod")
    public DataSource prodDataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:mysql://prod-db:3306/prod");
        config.setUsername("prod_user");
        config.setPassword("prod_pass");
        config.setMaximumPoolSize(50);
        config.setConnectionTimeout(30000);
        return new HikariDataSource(config);
    }
}
```

### 案例2：条件化配置（@Conditional）

```java
public class OnMissingBeanCondition implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        String beanClass = getBeanClass(metadata);
        return !context.getBeanFactory().containsBean(beanClass);
    }
    
    private String getBeanClass(AnnotatedTypeMetadata metadata) {
        // 从注解元数据获取Bean类名
        Map<String, Object> attributes = metadata.getAnnotationAttributes(ConditionalOnMissingBean.class.getName());
        return (String) attributes.get("value");
    }
}

@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(OnMissingBeanCondition.class)
public @interface ConditionalOnMissingBean {
    Class<?> value();
}

@Configuration
public class AutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean(CacheManager.class)
    public CacheManager cacheManager() {
        return new ConcurrentMapCacheManager();
    }
}
```

### 案例3：自定义BeanPostProcessor实现日志代理

```java
@Component
public class LoggingProxyPostProcessor implements BeanPostProcessor {
    
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        // 只对Service层添加日志代理
        if (bean.getClass().isAnnotationPresent(Service.class)) {
            return createLoggingProxy(bean);
        }
        return bean;
    }
    
    private Object createLoggingProxy(Object target) {
        return Proxy.newProxyInstance(
            target.getClass().getClassLoader(),
            target.getClass().getInterfaces(),
            (proxy, method, args) -> {
                long start = System.currentTimeMillis();
                try {
                    Object result = method.invoke(target, args);
                    System.out.println("[LOG] " + method.getName() + " executed in " + 
                        (System.currentTimeMillis() - start) + "ms");
                    return result;
                } catch (Exception e) {
                    System.out.println("[LOG] " + method.getName() + " failed: " + e.getMessage());
                    throw e;
                }
            }
        );
    }
}
```

### 案例4：完整的企业级配置示例

```java
@Configuration
@ComponentScan(basePackages = "com.example.enterprise")
@PropertySource("classpath:application.properties")
@EnableAspectJAutoProxy
@EnableTransactionManagement
public class EnterpriseConfig {
    
    @Autowired
    private Environment env;
    
    @Bean
    public DataSource dataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(env.getProperty("db.url"));
        config.setUsername(env.getProperty("db.username"));
        config.setPassword(env.getProperty("db.password"));
        config.setMaximumPoolSize(env.getProperty("db.maxPoolSize", Integer.class, 20));
        return new HikariDataSource(config);
    }
    
    @Bean
    public PlatformTransactionManager transactionManager(DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }
    
    @Bean
    public JdbcTemplate jdbcTemplate(DataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }
    
    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        return template;
    }
}
```

---

## 对比分析：Spring IOC vs 替代方案

### Spring IOC vs Google Guice

| 特性 | Spring IOC | Google Guice |
|------|-----------|-------------|
| 配置方式 | XML/注解/Java Config | 纯Java（Module） |
| 学习曲线 | 较陡 | 较平缓 |
| 功能丰富度 | 极高（AOP、事务、MVC等） | 轻量，仅DI |
| 启动速度 | 较慢（功能多） | 较快 |
| 生态系统 | 极丰富 | 较小 |
| 绑定方式 | 自动扫描+显式配置 | 显式绑定（bind()） |
| AOP支持 | 原生支持 | 通过AOP Alliance |
| 循环依赖 | 三级缓存解决 | 构造器注入不支持 |

**Guice示例：**

```java
// Guice Module
public class AppModule extends AbstractModule {
    @Override
    protected void configure() {
        bind(UserDao.class).to(UserDaoImpl.class);
        bind(UserService.class).to(UserServiceImpl.class);
    }
}

// 使用
Injector injector = Guice.createInjector(new AppModule());
UserService userService = injector.getInstance(UserService.class);
```

**选择建议：** 企业级复杂应用选Spring；简单DI需求、Android开发选Guice。

### Spring IOC vs CDI (Contexts and Dependency Injection)

| 特性 | Spring IOC | CDI (Java EE) |
|------|-----------|---------------|
| 标准 | Spring专属 | JSR-299/365标准 |
| 容器 | Spring容器 | Weld/OpenWebBeans |
| 注解 | @Autowired, @Component | @Inject, @Named |
| 扩展性 | BeanPostProcessor | SPI扩展 |
| 适用范围 | Spring生态 | Java EE/Jakarta EE |
| 生产者方法 | @Bean | @Produces |

**CDI示例：**

```java
// CDI
@ApplicationScoped
public class UserService {
    @Inject
    private UserDao userDao;
}

// Spring也支持JSR-330注解
@Named
public class UserService {
    @Inject
    private UserDao userDao;
}
```

**选择建议：** Java EE环境用CDI；Spring生态用Spring原生注解。

### Spring IOC vs Dagger

| 特性 | Spring IOC | Dagger |
|------|-----------|--------|
| 编译时/运行时 | 运行时反射 | 编译时代码生成 |
| 性能 | 有反射开销 | 接近手写代码 |
| 适用场景 | 服务端 | Android |
| 配置方式 | 注解扫描 | 显式@Component/@Module |
| 依赖图验证 | 运行时 | 编译时 |

**选择建议：** 服务端应用选Spring；Android应用选Dagger。

---

## 性能分析与优化

### 1. Bean作用域选择

```java
// 默认singleton性能最好（只创建一次）
@Component
public class SingletonService { }

// prototype每次getBean都创建，开销大
@Scope("prototype")
@Component
public class PrototypeService { }

// 对于Web应用，request/session作用域需要额外的线程安全考虑
```

### 2. 懒加载优化启动时间

```java
@Configuration
public class AppConfig {
    
    @Bean
    @Lazy // 延迟初始化，容器启动时不创建
    public HeavyService heavyService() {
        return new HeavyService(); // 启动耗时操作
    }
}

// 或者在全局启用懒加载
@SpringBootApplication
@Lazy // Spring Boot 2.2+ 支持全局懒加载
public class Application { }
```

### 3. 减少BeanPostProcessor数量

```java
// 每个BeanPostProcessor都会遍历所有Bean，过多会影响启动性能
// 最佳实践：合并相似的Processor，使用条件判断减少处理范围
@Component
public class OptimizedPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        // 先判断类型，避免不必要的处理
        if (bean instanceof DataSource) {
            // 只处理DataSource
        }
        return bean;
    }
}
```

### 4. 使用索引加速组件扫描

```java
// Spring 5+ 支持组件索引
// 添加依赖：spring-context-indexer
// 编译时生成META-INF/spring.components，避免运行时扫描所有类
```

### 5. 避免在初始化阶段做耗时操作

```java
@Component
public class BadPractice {
    @PostConstruct
    public void init() {
        // 避免在这里连接远程服务、加载大量数据
        loadAllDataFromRemote(); // 阻塞启动！
    }
}

// 正确做法：异步初始化或延迟加载
@Component
public class GoodPractice implements ApplicationListener<ContextRefreshedEvent> {
    @Override
    public void onApplicationEvent(ContextRefreshedEvent event) {
        // 容器启动完成后异步加载
        CompletableFuture.runAsync(this::loadAllDataFromRemote);
    }
}
```

### 6. 启动性能分析

```java
// Spring Boot Actuator提供启动指标
// 添加依赖：spring-boot-starter-actuator
// 访问：http://localhost:8080/actuator/startup

// 输出示例：
// {
//   "springBootVersion": "2.7.0",
//   "timeline": {
//     "startTime": "2023-01-01T00:00:00.000Z",
//     "events": [
//       {
//         "endTime": "2023-01-01T00:00:01.500Z",
//         "duration": "PT1.5S",
//         "startupStep": {
//           "name": "spring.beans.instantiate",
//           "id": 1,
//           "tags": [
//             {
//               "key": "beanName",
//               "value": "userService"
//             }
//           ]
//         }
//       }
//     ]
//   }
// }
```

---

## 常见陷阱与最佳实践

### 陷阱1：混淆BeanFactory和ApplicationContext

```java
// 错误：需要国际化、事件发布时用了BeanFactory
BeanFactory factory = new XmlBeanFactory(...);
// 没有这些功能！
```

**最佳实践：** 直接用ApplicationContext（ClassPathXmlApplicationContext、AnnotationConfigApplicationContext），除非有特殊需求（如资源受限的嵌入式环境）。

### 陷阱2：滥用字段注入

```java
@Service
public class UserService {
    @Autowired
    private UserDao userDao; // 字段注入
}
```

**问题：** 1）无法使用final，依赖可被修改；2）不便于单元测试（必须通过反射注入）；3）隐藏了类的依赖，构造函数才能明确表达必需依赖。

**最佳实践：** 优先使用构造器注入，特别是Spring 4.3+支持单构造器隐式注入。Setter注入用于可选依赖。

```java
@Service
public class UserService {
    private final UserDao userDao; // 不可变
    
    public UserService(UserDao userDao) { // 必需依赖显式声明
        this.userDao = userDao;
    }
}
```

### 陷阱3：忽略Bean的作用域

```java
// 默认singleton，但某些场景应该用prototype
@Component
public class Task {
    // 有状态，被多线程共享会出问题
}
```

**最佳实践：** 无状态Bean用singleton（默认）；有状态Bean用prototype或request/session作用域。

```java
@Scope("prototype")
@Component
public class Task {
    private String taskId; // 每个实例独立
}
```

### 陷阱4：BeanFactoryPostProcessor和BeanPostProcessor混淆

- **BeanFactoryPostProcessor**：在BeanDefinition加载后、Bean实例化前执行，修改配置元数据（如PropertySourcesPlaceholderConfigurer替换`${}`）。
- **BeanPostProcessor**：在Bean实例化后、初始化前后执行，修改Bean实例（如AOP代理生成）。

**最佳实践：** 需要改配置用前者，需要改实例用后者。两者注册时机不同，不能混用。

### 陷阱5：循环依赖不自知

```java
@Service
public class A {
    @Autowired private B b;
}
@Service
public class B {
    @Autowired private A a;
}
```

**最佳实践：** Spring能解决单例字段注入的循环依赖（三级缓存），但构造器注入不行。更好的做法是重构设计，用事件驱动或中间层打破循环。

### 陷阱6：在@Bean方法中直接调用另一个@Bean方法

```java
@Configuration
public class Config {
    @Bean
    public ServiceA serviceA() {
        return new ServiceA(serviceB()); // 直接调用，绕过代理
    }
    
    @Bean
    public ServiceB serviceB() {
        return new ServiceB();
    }
}
```

**问题：** 如果在`@Configuration`类中直接调用`@Bean`方法，会绕过CGLIB代理，导致Bean被多次实例化。

**最佳实践：** 确保使用`@Configuration`（不是`@Component`），Spring会通过CGLIB代理拦截@Bean方法调用，保证单例。

### 陷阱7：忽视ApplicationContext的刷新成本

```java
// 错误：频繁创建和刷新ApplicationContext
for (int i = 0; i < 100; i++) {
    ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
    // 使用ctx
    ctx.close();
}
```

**最佳实践：** ApplicationContext创建成本很高，应该在应用启动时创建一次，全局复用。

### 陷阱8：混淆@Autowired和@Resource

```java
@Service
public class UserService {
    @Resource(name = "userDaoImpl") // 按名称注入
    private UserDao userDao;
    
    @Autowired // 按类型注入
    private OrderDao orderDao;
}
```

**最佳实践：**
- `@Autowired`：Spring专属，按类型注入，配合`@Qualifier`按名称注入
- `@Resource`：JSR-250标准，默认按名称注入，名称不匹配时按类型注入
- 推荐优先使用构造器注入（Spring 4.3+无需@Autowired）

---

## 面试题与参考答案

### Q1：什么是IOC和DI？

**答：** IOC（Inversion of Control，控制反转）是将对象创建和依赖管理的控制权从程序代码反转给容器。DI（Dependency Injection，依赖注入）是IOC的实现方式，通过构造器、Setter或字段注入，将依赖从外部传入，而不是在内部创建。好处是解耦、便于测试、集中管理对象生命周期。

**深度补充：**
- IOC是一种设计原则，DI是实现IOC的具体技术
- 好莱坞原则："Don't call us, we'll call you"
- 构造器注入 > Setter注入 > 字段注入

### Q2：BeanFactory和ApplicationContext的区别？

**答：** BeanFactory是Spring IOC的基础接口，提供基本的Bean获取和判断功能，默认延迟初始化。ApplicationContext继承自BeanFactory，扩展了：国际化（MessageSource）、资源加载（ResourcePatternResolver）、事件发布（ApplicationEventPublisher）、AOP集成、自动注册BeanPostProcessor和BeanFactoryPostProcessor。实际开发都用ApplicationContext。

**深度补充：**
- BeanFactory是延迟加载，ApplicationContext默认立即加载（可配置lazy）
- ApplicationContext实现了ResourcePatternResolver，支持通配符加载资源
- ApplicationContext继承了ApplicationEventPublisher，支持事件驱动

### Q3：Spring容器启动的完整流程？

**答：** 
1. `prepareRefresh()`：准备环境，初始化属性源；
2. `obtainFreshBeanFactory()`：加载BeanDefinition（XML/注解/Java Config）；
3. `prepareBeanFactory()`：配置BeanFactory的标准特性（类加载器、表达式解析器等）；
4. `postProcessBeanFactory()`：子类扩展；
5. `invokeBeanFactoryPostProcessors()`：执行BeanFactoryPostProcessor；
6. `registerBeanPostProcessors()`：注册BeanPostProcessor；
7. `initMessageSource()`：国际化；
8. `initApplicationEventMulticaster()`：事件广播器；
9. `onRefresh()`：子类扩展（如Spring Boot的WebServer启动）；
10. `registerListeners()`：注册监听器；
11. `finishBeanFactoryInitialization()`：实例化所有非lazy单例Bean；
12. `finishRefresh()`：发布ContextRefreshedEvent。

**深度补充：**
- 第5步执行ConfigurationClassPostProcessor，解析@Configuration类
- 第6步注册的BeanPostProcessor会影响后续所有Bean的创建
- 第11步是创建Bean的核心步骤，涉及三级缓存解决循环依赖

### Q4：BeanDefinition是什么？有哪些来源？

**答：** BeanDefinition是Bean的元数据定义，描述如何创建Bean，包括：类名、作用域、是否懒加载、构造参数、属性值、初始化/销毁方法等。来源：1）XML配置的`<bean>`标签；2）注解扫描的@Component/@Service等；3）Java Config的@Bean方法；4）编程方式通过BeanDefinitionBuilder手动创建。

**深度补充：**
- BeanDefinition有合并机制（子定义继承父定义）
- RootBeanDefinition是最终的合并结果
- BeanDefinition中的属性决定了Bean的创建方式

### Q5：Bean的生命周期有哪些阶段？

**答：** 
1. **实例化**（Instantiation）：调用构造器创建对象；
2. **属性赋值**（Populate）：注入依赖（@Autowired等）；
3. **初始化**（Initialization）：调用Aware接口方法 → BeanPostProcessor.postProcessBeforeInitialization → @PostConstruct → InitializingBean.afterPropertiesSet() → init-method → BeanPostProcessor.postProcessAfterInitialization（AOP代理生成）；
4. **使用**（In Use）；
5. **销毁**（Destruction）：@PreDestroy → DisposableBean.destroy() → destroy-method。

**深度补充：**
- 三级缓存解决循环依赖发生在实例化之后、初始化之前
- AOP代理在初始化阶段的最后生成
- Aware接口在初始化之前调用

### Q6：什么是BeanPostProcessor？有哪些典型应用？

**答：** BeanPostProcessor是Spring的扩展接口，在Bean初始化前后执行：`postProcessBeforeInitialization`和`postProcessAfterInitialization`。典型应用：1）AOP代理生成（AbstractAutoProxyCreator）；2）@Autowired依赖注入（AutowiredAnnotationBeanPostProcessor）；3）@PostConstruct/@PreDestroy处理（CommonAnnotationBeanPostProcessor）；4）属性校验；5）自定义初始化逻辑。

**深度补充：**
- BeanPostProcessor的注册顺序决定执行顺序
- 可以通过实现Ordered接口或标注@Order控制顺序
- BeanPostProcessor本身也是Bean，有特殊的注册时机

### Q7：Spring如何解决循环依赖？

**答：** Spring通过三级缓存解决单例Bean的字段/Setter注入循环依赖：
1. **singletonObjects**：一级缓存，存完整的单例Bean；
2. **earlySingletonObjects**：二级缓存，存提前暴露的Bean（已实例化未初始化）；
3. **singletonFactories**：三级缓存，存ObjectFactory（用于生成代理对象）。

流程：A实例化后放入三级缓存 → A属性赋值需要B → B实例化后放入三级缓存 → B属性赋值需要A → 从三级缓存获取A的ObjectFactory生成A → B初始化完成放入一级缓存 → A拿到B完成初始化。

**深度补充：**
- 构造器注入的循环依赖无法解决（实例化阶段就需要依赖）
- @Lazy可以解决构造器循环依赖（注入代理对象）
- 三级缓存的设计主要是为了处理AOP代理对象的循环依赖

### Q8：为什么构造器注入不能解决循环依赖？

**答：** 构造器注入在实例化阶段就需要依赖（调用构造器时传入），此时对象还没创建，无法提前暴露到三级缓存。而字段/Setter注入是先实例化（调用无参构造器），再注入属性，实例化完成后就可以暴露到缓存。解决构造器循环依赖：改用Setter注入，或使用@Lazy延迟注入（注入代理对象）。

**深度补充：**
- @Lazy注入的是代理对象，真正的依赖在首次使用时才初始化
- 最好的解决方案是重构代码，打破循环依赖

### Q9：Spring使用了哪些设计模式？

**答：** 
1. **工厂模式**：BeanFactory创建Bean；
2. **单例模式**：DefaultSingletonBeanRegistry管理单例；
3. **模板方法模式**：AbstractApplicationContext.refresh()定义算法骨架；
4. **策略模式**：InstantiationStrategy支持不同实例化策略；
5. **观察者模式**：ApplicationEvent事件机制；
6. **适配器模式**：BeanDefinitionReader适配不同配置来源；
7. **代理模式**：AOP代理生成；
8. **装饰器模式**：BeanWrapper包装Bean实例。

### Q10：@Autowired和@Resource的区别？

**答：** 
- **@Autowired**：Spring专属注解，默认按类型（byType）注入，配合@Qualifier按名称注入。可以标注在字段、构造器、Setter上。required=false可让依赖可选。
- **@Resource**：JSR-250标准注解，默认按名称（byName）注入，名称不匹配时按类型注入。只能标注在字段和Setter上。

**推荐：** Spring项目用@Autowired（功能更强），需要跨框架兼容用@Resource。

**深度补充：**
- Spring 4.3+单构造器无需@Autowired
- @Inject是JSR-330标准，功能类似@Autowired
- 推荐优先使用构造器注入

---

*此文原创，转载请注明出处。*
