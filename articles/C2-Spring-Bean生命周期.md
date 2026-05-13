# Spring Bean生命周期深度解析：从创建到销毁的全链路源码分析

**文章标签：** #java #spring #bean #生命周期 #循环依赖 #源码分析 #设计模式 #面试 #深度解析

## 目录

- [引言：为什么需要理解Bean生命周期](#引言为什么需要理解bean生命周期)
- [理论基础：对象生命周期管理的本质](#理论基础对象生命周期管理的本质)
- [来龙去脉：Spring生命周期机制的发展](#来龙去脉spring生命周期机制的发展)
- [Bean的生命周期阶段详解](#bean的生命周期阶段详解)
- [Aware接口详解与源码分析](#aware接口详解与源码分析)
- [BeanPostProcessor源码深度分析](#beanpostprocessor源码深度分析)
- [InitializingBean和DisposableBean源码分析](#initializingbean和disposablebean源码分析)
- [生命周期完整时序图](#生命周期完整时序图)
- [循环依赖问题与三级缓存源码分析](#循环依赖问题与三级缓存源码分析)
- [设计模式解析](#设计模式解析)
- [实战案例：工业级生命周期管理](#实战案例工业级生命周期管理)
- [对比分析：Spring vs 其他框架](#对比分析spring-vs-其他框架)
- [性能分析与优化](#性能分析与优化)
- [常见陷阱与最佳实践](#常见陷阱与最佳实践)
- [面试题与参考答案](#面试题与参考答案)

---

## 引言：为什么需要理解Bean生命周期

Bean生命周期不是简单的"创建-使用-销毁"，而是Spring框架**对象管理哲学的核心体现**。理解生命周期机制，是掌握Spring高级特性（AOP、事务、循环依赖解决）的基础。

**核心认知：**

```
Bean生命周期的管理维度：

┌─────────────────────────────────────────┐
│ 1. 创建时机控制                          │
│    - 立即创建（默认singleton）            │
│    - 延迟创建（@Lazy）                    │
│    - 按需创建（prototype）                │
├─────────────────────────────────────────┤
│ 2. 依赖解析与注入                          │
│    - 构造器注入（实例化时）               │
│    - Setter/字段注入（实例化后）          │
│    - 循环依赖处理                         │
├─────────────────────────────────────────┤
│ 3. 初始化回调                              │
│    - Aware接口（获取容器资源）            │
│    - @PostConstruct（JSR-250）            │
│    - InitializingBean（Spring接口）       │
│    - 自定义init-method                    │
├─────────────────────────────────────────┤
│ 4. 使用期管理                              │
│    - 作用域管理（singleton/prototype）    │
│    - 代理对象管理（AOP）                  │
├─────────────────────────────────────────┤
│ 5. 销毁回调                                │
│    - @PreDestroy（JSR-250）               │
│    - DisposableBean（Spring接口）         │
│    - 自定义destroy-method                 │
└─────────────────────────────────────────┘
```

**关键洞察**：Spring通过**模板方法模式**定义了Bean创建的标准流程，同时通过**BeanPostProcessor**机制提供了丰富的扩展点，使得框架可以在不修改用户代码的情况下，实现AOP代理、依赖注入、事务管理等高级功能。

---

## 理论基础：对象生命周期管理的本质

### 1. 对象生命周期的通用模型

所有面向对象系统中，对象都经历以下阶段：

```
通用对象生命周期：

分配内存 → 构造（Constructor） → 初始化（Initialization） → 使用（In Use） → 清理（Cleanup） → 销毁（Destruction） → 回收内存

Spring的增强：
- 在构造前后添加了依赖解析
- 在初始化前后添加了回调机制
- 在使用期添加了作用域管理
- 在销毁前添加了资源释放回调
```

### 2. 容器管理 vs 手动管理

```java
// 手动管理（传统方式）
public class ManualManagement {
    public void execute() {
        Connection conn = null;
        try {
            conn = dataSource.getConnection(); // 创建
            conn.setAutoCommit(false);         // 初始化
            // 使用...
            conn.commit();                     // 业务完成
        } catch (Exception e) {
            if (conn != null) conn.rollback(); // 异常处理
        } finally {
            if (conn != null) {
                try { conn.close(); }          // 销毁
                catch (SQLException e) { e.printStackTrace(); }
            }
        }
    }
}

// Spring容器管理
@Service
public class SpringManagement {
    @Autowired
    private ConnectionPool pool; // 容器创建、初始化、注入、销毁
    
    @Transactional
    public void execute() {
        // 只需关注业务逻辑
        // 事务、连接、异常处理都由Spring管理
    }
}
```

### 3. 回调机制的设计思想

回调机制是Spring生命周期管理的核心，它实现了**控制反转**在对象生命周期层面的应用：

```
传统方式：程序员在代码中显式调用初始化逻辑
Spring方式：容器在特定时机自动调用用户定义的回调

优势：
1. 解耦初始化逻辑与业务逻辑
2. 统一的生命周期管理
3. 支持框架级扩展（AOP、事务）
4. 便于测试（可以mock容器行为）
```

---

## 来龙去脉：Spring生命周期机制的发展

### 第一阶段：基础生命周期（Spring 1.x）

Spring 1.x提供了最基础的生命周期支持：

```java
// Spring 1.x的方式
public class OldStyleBean implements InitializingBean, DisposableBean {
    public void afterPropertiesSet() {
        // 初始化逻辑
    }
    
    public void destroy() {
        // 销毁逻辑
    }
}
```

**特点：**
- 只能通过接口方式定义回调
- 侵入性较强（必须实现Spring接口）
- 不支持注解

### 第二阶段：注解支持（Spring 2.5+）

引入JSR-250注解支持：

```java
public class ModernBean {
    @PostConstruct
    public void init() {
        // 初始化逻辑
    }
    
    @PreDestroy
    public void cleanup() {
        // 销毁逻辑
    }
}
```

**特点：**
- 非侵入式（不依赖Spring接口）
- 符合Java标准（JSR-250）
- 代码更简洁

### 第三阶段：Java Config与函数式（Spring 3.0+）

支持@Bean方法的initMethod和destroyMethod：

```java
@Configuration
public class Config {
    @Bean(initMethod = "customInit", destroyMethod = "customDestroy")
    public MyBean myBean() {
        return new MyBean();
    }
}
```

**特点：**
- 配置集中化
- 支持复杂的初始化逻辑
- 便于条件化配置

### 第四阶段：Reactive与函数式（Spring 5.0+）

支持Reactive编程模型和函数式Bean注册：

```java
// 函数式Bean注册
GenericApplicationContext context = new GenericApplicationContext();
context.registerBean("myBean", MyBean.class, () -> new MyBean());
context.refresh();
```

---

## Bean的生命周期阶段详解

Spring Bean从创建到销毁经历多个阶段：

```
┌─────────────────────────────────────────────────────────────┐
│                     Bean 完整生命周期                        │
├─────────────────────────────────────────────────────────────┤
│  1. 实例化（Instantiation）                                  │
│     └─► 调用构造器创建对象                                    │
│         - 构造器注入在此阶段完成                               │
│         - 如果失败，抛出BeanCreationException                  │
│         - 实例化策略：简单实例化、CGLIB实例化                  │
│                                                             │
│  2. 属性赋值（Populate）                                     │
│     └─► 注入依赖（@Autowired、@Value等）                      │
│         - InstantiationAwareBeanPostProcessor干预            │
│         - 自动装配byName/byType/constructor                   │
│         - 解决循环依赖（三级缓存）                             │
│                                                             │
│  3. 初始化（Initialization）                                 │
│     ├─► Aware接口回调（BeanNameAware、ApplicationContextAware）│
│     ├─► BeanPostProcessor.postProcessBeforeInitialization()   │
│     ├─► @PostConstruct（JSR-250）                            │
│     ├─► InitializingBean.afterPropertiesSet()                │
│     ├─► 自定义init-method                                    │
│     └─► BeanPostProcessor.postProcessAfterInitialization()    │
│         （AOP代理对象在此替换原始Bean）                        │
│                                                             │
│  4. 使用（In Use）                                           │
│     └─► Bean处于就绪状态，可被业务代码使用                      │
│         - singleton：全局复用                                 │
│         - prototype：每次getBean创建新实例                    │
│         - request/session：Web作用域                          │
│                                                             │
│  5. 销毁（Destruction）                                      │
│     ├─► @PreDestroy（JSR-250）                               │
│     ├─► DisposableBean.destroy()                             │
│     └─► 自定义destroy-method                                 │
│         - 释放资源（连接、线程池等）                           │
│         - 发送销毁通知                                        │
└─────────────────────────────────────────────────────────────┘
```

### 各阶段详细说明

#### 实例化阶段

```java
// AbstractAutowireCapableBeanFactory.createBeanInstance()
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, Object[] args) {
    // 1. 解析Bean类
    Class<?> beanClass = resolveBeanClass(mbd, beanName);
    
    // 2. 使用工厂方法实例化
    if (mbd.getFactoryMethodName() != null) {
        return instantiateUsingFactoryMethod(beanName, mbd, args);
    }
    
    // 3. 自动装配构造器
    Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
    if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR ||
            mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args)) {
        return autowireConstructor(beanName, mbd, ctors, args);
    }
    
    // 4. 使用默认无参构造器
    return instantiateBean(beanName, mbd);
}
```

#### 属性赋值阶段

```java
// AbstractAutowireCapableBeanFactory.populateBean()
protected void populateBean(String beanName, RootBeanDefinition mbd, BeanWrapper bw) {
    PropertyValues pvs = mbd.getPropertyValues();
    
    // 执行InstantiationAwareBeanPostProcessor.postProcessAfterInstantiation
    if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
        for (InstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().instantiationAware) {
            if (!bp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
                return;
            }
        }
    }
    
    // 按名称自动装配
    if (mbd.getResolvedAutowireMode() == AUTOWIRE_BY_NAME) {
        autowireByName(beanName, mbd, bw, new BeanDefinitionValueVisitor());
    }
    
    // 按类型自动装配
    if (mbd.getResolvedAutowireMode() == AUTOWIRE_BY_TYPE) {
        autowireByType(beanName, mbd, bw, new BeanDefinitionValueVisitor());
    }
    
    // 执行InstantiationAwareBeanPostProcessor.postProcessPropertyValues
    // @Autowired、@Resource在这里处理
    for (InstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().instantiationAware) {
        PropertyValues pvsToUse = bp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
        if (pvsToUse != null) {
            pvs = pvsToUse;
        }
    }
    
    // 应用PropertyValues
    applyPropertyValues(beanName, mbd, bw, pvs);
}
```

#### 初始化阶段

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
        throw new BeanCreationException(mbd.getResourceDescription(), beanName, ex.getMessage(), ex);
    }
    
    // 4. BeanPostProcessor.postProcessAfterInitialization（AOP代理在此生成）
    if (mbd == null || !mbd.isSynthetic()) {
        wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    }
    
    return wrappedBean;
}
```

---

## Aware接口详解与源码分析

Aware接口让Bean获取Spring容器的资源：

```java
// Aware根接口，标记接口
public interface Aware { }

// 常用Aware接口
public interface BeanNameAware extends Aware {
    void setBeanName(String name);
}

public interface BeanClassLoaderAware extends Aware {
    void setBeanClassLoader(ClassLoader classLoader);
}

public interface BeanFactoryAware extends Aware {
    void setBeanFactory(BeanFactory beanFactory) throws BeansException;
}

public interface EnvironmentAware extends Aware {
    void setEnvironment(Environment environment);
}

public interface EmbeddedValueResolverAware extends Aware {
    void setEmbeddedValueResolver(StringValueResolver resolver);
}

public interface ResourceLoaderAware extends Aware {
    void setResourceLoader(ResourceLoader resourceLoader);
}

public interface ApplicationEventPublisherAware extends Aware {
    void setApplicationEventPublisher(ApplicationEventPublisher applicationEventPublisher);
}

public interface MessageSourceAware extends Aware {
    void setMessageSource(MessageSource messageSource);
}

public interface ApplicationContextAware extends Aware {
    void setApplicationContext(ApplicationContext applicationContext) throws BeansException;
}

public interface ServletContextAware extends Aware {
    void setServletContext(ServletContext servletContext);
}
```

**使用示例：**

```java
@Component
public class ContextAwareBean implements BeanNameAware, ApplicationContextAware, EnvironmentAware {
    private String beanName;
    private ApplicationContext applicationContext;
    private Environment environment;
    
    @Override
    public void setBeanName(String name) {
        this.beanName = name;
        System.out.println("Bean名称: " + name);
    }
    
    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        this.applicationContext = ctx;
        System.out.println("容器: " + ctx.getDisplayName());
    }
    
    @Override
    public void setEnvironment(Environment env) {
        this.environment = env;
        System.out.println("当前profile: " + Arrays.toString(env.getActiveProfiles()));
    }
}
```

**执行顺序：** 在Bean实例化完成、属性赋值之后，初始化方法之前调用。

**Aware接口的执行源码：**

```java
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

// ApplicationContextAwareProcessor处理其他Aware接口
public class ApplicationContextAwareProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        if (bean instanceof EnvironmentAware) {
            ((EnvironmentAware) bean).setEnvironment(this.applicationContext.getEnvironment());
        }
        if (bean instanceof EmbeddedValueResolverAware) {
            ((EmbeddedValueResolverAware) bean).setEmbeddedValueResolver(this.embeddedValueResolver);
        }
        if (bean instanceof ResourceLoaderAware) {
            ((ResourceLoaderAware) bean).setResourceLoader(this.applicationContext);
        }
        if (bean instanceof ApplicationEventPublisherAware) {
            ((ApplicationEventPublisherAware) bean).setApplicationEventPublisher(this.applicationContext);
        }
        if (bean instanceof MessageSourceAware) {
            ((MessageSourceAware) bean).setMessageSource(this.applicationContext);
        }
        if (bean instanceof ApplicationContextAware) {
            ((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
        }
        return bean;
    }
}
```

---

## BeanPostProcessor源码深度分析

BeanPostProcessor是Spring提供的扩展点，在Bean初始化前后处理：

```java
public interface BeanPostProcessor {
    // 初始化前调用
    @Nullable
    default Object postProcessBeforeInitialization(Object bean, String beanName) 
            throws BeansException {
        return bean;
    }
    
    // 初始化后调用（AOP代理在此生成）
    @Nullable
    default Object postProcessAfterInitialization(Object bean, String beanName) 
            throws BeansException {
        return bean;
    }
}
```

**核心实现类：**

```java
// 1. ApplicationContextAwareProcessor - 处理Aware接口
public class ApplicationContextAwareProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        if (bean instanceof ApplicationContextAware) {
            ((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
        }
        // 处理其他Aware接口...
        return bean;
    }
}

// 2. AutowiredAnnotationBeanPostProcessor - 处理@Autowired
public class AutowiredAnnotationBeanPostProcessor implements 
        InstantiationAwareBeanPostProcessor, MergedBeanDefinitionPostProcessor {
    
    @Override
    public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) {
        // 查找@Autowire和@Value注解的字段/方法
        InjectionMetadata metadata = findAutowiringMetadata(beanName, bean.getClass(), pvs);
        // 执行依赖注入
        metadata.inject(bean, beanName, pvs);
        return pvs;
    }
}

// 3. CommonAnnotationBeanPostProcessor - 处理@PostConstruct/@PreDestroy/@Resource
public class CommonAnnotationBeanPostProcessor extends InitDestroyAnnotationBeanPostProcessor
        implements InstantiationAwareBeanPostProcessor {
    
    public CommonAnnotationBeanPostProcessor() {
        setInitAnnotationType(PostConstruct.class);
        setDestroyAnnotationType(PreDestroy.class);
    }
}

// 4. AbstractAutoProxyCreator - AOP代理生成
public abstract class AbstractAutoProxyCreator extends ProxyProcessorSupport
        implements SmartInstantiationAwareBeanPostProcessor {
    
    @Override
    public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
        if (bean != null) {
            // 检查是否需要代理（是否有切面匹配）
            Object cacheKey = getCacheKey(bean.getClass(), beanName);
            if (this.earlyProxyReferences.remove(cacheKey) != bean) {
                return wrapIfNecessary(bean, beanName, cacheKey);
            }
        }
        return bean;
    }
}
```

**BeanPostProcessor注册顺序（影响执行顺序）：**

```java
// AbstractApplicationContext.registerBeanPostProcessors()
// 按优先级排序：PriorityOrdered > Ordered > 普通
// 同优先级按beanName排序
```

### 自定义BeanPostProcessor

```java
@Component
public class MyBeanPostProcessor implements BeanPostProcessor, Ordered {
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        if (bean instanceof UserService) {
            System.out.println("Before init: " + beanName + ", class: " + bean.getClass());
        }
        return bean;
    }
    
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        if (bean instanceof UserService) {
            System.out.println("After init: " + beanName + ", class: " + bean.getClass());
            // 可以在这里生成代理、修改属性等
        }
        return bean;
    }
    
    @Override
    public int getOrder() {
        return Ordered.LOWEST_PRECEDENCE; // 最后执行
    }
}
```

---

## InitializingBean和DisposableBean源码分析

### 初始化方式（按执行顺序）

```java
public interface InitializingBean {
    void afterPropertiesSet() throws Exception;
}

public interface DisposableBean {
    void destroy() throws Exception;
}
```

**完整示例：**

```java
@Component
public class LifecycleBean implements InitializingBean, DisposableBean {
    
    private String config;
    
    // 1. @PostConstruct（JSR-250注解）
    @PostConstruct
    public void postConstruct() {
        System.out.println("1. @PostConstruct - Bean初始化开始");
        this.config = "default";
    }
    
    // 2. InitializingBean接口
    @Override
    public void afterPropertiesSet() {
        System.out.println("2. InitializingBean.afterPropertiesSet() - 属性已设置");
        if (this.config == null) {
            throw new IllegalStateException("config must not be null");
        }
    }
    
    // 3. 自定义init-method（通过@Bean(initMethod="customInit")或XML配置）
    public void customInit() {
        System.out.println("3. init-method - 自定义初始化逻辑");
    }
    
    // 销毁阶段
    // 1. @PreDestroy（JSR-250注解）
    @PreDestroy
    public void preDestroy() {
        System.out.println("1. @PreDestroy - Bean销毁开始");
    }
    
    // 2. DisposableBean接口
    @Override
    public void destroy() {
        System.out.println("2. DisposableBean.destroy() - 释放资源");
    }
    
    // 3. 自定义destroy-method
    public void customDestroy() {
        System.out.println("3. destroy-method - 自定义销毁逻辑");
    }
}
```

**配置方式：**

```java
@Configuration
public class BeanConfig {
    @Bean(initMethod = "customInit", destroyMethod = "customDestroy")
    public LifecycleBean lifecycleBean() {
        return new LifecycleBean();
    }
}
```

---

## 生命周期完整时序图

```
Container (AbstractAutowireCapableBeanFactory)
  │
  ├──► createBean(beanName, mbd, args)
  │       │
  │       ├──► doCreateBean()
  │       │       │
  │       │       ├──► createBeanInstance() ──► 实例化Bean
  │       │       │       └─► 调用构造器/工厂方法
  │       │       │
  │       │       ├──► addSingletonFactory() ──► 放入三级缓存（解决循环依赖）
  │       │       │
  │       │       ├──► populateBean() ──► 属性赋值
  │       │       │       ├─► InstantiationAwareBeanPostProcessor
  │       │       │       │       .postProcessAfterInstantiation()
  │       │       │       ├─► autowireByName/byType()
  │       │       │       └─► InstantiationAwareBeanPostProcessor
  │       │       │               .postProcessProperties() (@Autowired)
  │       │       │
  │       │       └──► initializeBean() ──► 初始化
  │       │               │
  │       │               ├──► invokeAwareMethods()
  │       │               │       ├─► BeanNameAware.setBeanName()
  │       │               │       ├─► BeanClassLoaderAware.setBeanClassLoader()
  │       │               │       └─► BeanFactoryAware.setBeanFactory()
  │       │               │
  │       │               ├──► applyBeanPostProcessorsBeforeInitialization()
  │       │               │       ├─► ApplicationContextAwareProcessor
  │       │               │       ├─► CommonAnnotationBeanPostProcessor (@PostConstruct)
  │       │               │       └─► 其他BeanPostProcessor...
  │       │               │
  │       │               ├──► invokeInitMethods()
  │       │               │       ├─► InitializingBean.afterPropertiesSet()
  │       │               │       └─► 自定义init-method
  │       │               │
  │       │               └──► applyBeanPostProcessorsAfterInitialization()
  │       │                       └─► AbstractAutoProxyCreator (AOP代理)
  │       │
  │       └──► registerDisposableBeanIfNecessary() ──► 注册销毁回调
  │
  ├──► Bean就绪，可被使用
  │
  └──► 容器关闭时
          │
          ├──► destroyBeans()
          │       │
          │       ├──► postProcessBeforeDestruction()
          │       │       └─► CommonAnnotationBeanPostProcessor (@PreDestroy)
          │       │
          │       ├──► DisposableBean.destroy()
          │       └──► 自定义destroy-method
          │
          └──► Bean销毁完成
```

---

## 循环依赖问题与三级缓存源码分析

### 什么是循环依赖

循环依赖：两个或多个Bean相互依赖，形成闭环。

```java
@Service
public class UserService {
    @Autowired
    private OrderService orderService;
}

@Service
public class OrderService {
    @Autowired
    private UserService userService;
}
```

问题：创建UserService需要OrderService，创建OrderService需要UserService，死循环！

### 三级缓存解决循环依赖

Spring通过**三级缓存**解决单例Bean的循环依赖：

```java
public class DefaultSingletonBeanRegistry extends SimpleAliasRegistry 
        implements SingletonBeanRegistry {
    
    // 一级缓存：完整的单例Bean（实例化+初始化完成）
    private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);
    
    // 二级缓存：提前暴露的Bean（已实例化，未初始化）
    private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);
    
    // 三级缓存：Bean的ObjectFactory（用于生成代理对象）
    private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
    
    // 当前正在创建中的单例Bean名称集合
    private final Set<String> singletonsCurrentlyInCreation = 
            Collections.newSetFromMap(new ConcurrentHashMap<>(16));
}
```

#### 解决流程

```
1. 创建UserService
   - 实例化UserService（调用构造器）
   - 标记UserService正在创建中
   - 将UserService的ObjectFactory放入三级缓存
   ↓
2. UserService属性赋值，需要OrderService
   - 从一级缓存查找OrderService：没找到
   - OrderService不在创建中，开始创建OrderService
   ↓
3. 创建OrderService
   - 实例化OrderService
   - 标记OrderService正在创建中
   - 将OrderService的ObjectFactory放入三级缓存
   ↓
4. OrderService属性赋值，需要UserService
   - 从一级缓存找：没找到（UserService还没初始化完）
   - 从二级缓存找：没找到
   - 从三级缓存找：找到UserService的ObjectFactory
   - 调用getObject()获取UserService（可能是代理对象）
   - 将UserService放入二级缓存，从三级缓存移除
   ↓
5. OrderService初始化完成
   - 放入一级缓存
   - 从二级、三级缓存移除
   ↓
6. 回到UserService的属性赋值
   - 拿到OrderService，注入
   ↓
7. UserService初始化完成
   - 放入一级缓存
```

#### 核心源码

```java
// DefaultSingletonBeanRegistry.getSingleton()
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    // 1. 从一级缓存拿
    Object singletonObject = this.singletonObjects.get(beanName);
    
    // 如果一级缓存没有，且该Bean正在创建中
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        synchronized (this.singletonObjects) {
            singletonObject = this.singletonObjects.get(beanName);
            if (singletonObject == null) {
                // 2. 从二级缓存拿
                singletonObject = this.earlySingletonObjects.get(beanName);
                if (singletonObject == null && allowEarlyReference) {
                    // 3. 从三级缓存拿
                    ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                    if (singletonFactory != null) {
                        // 通过ObjectFactory创建Bean（可能生成代理）
                        singletonObject = singletonFactory.getObject();
                        // 放入二级缓存，从三级缓存移除
                        this.earlySingletonObjects.put(beanName, singletonObject);
                        this.singletonFactories.remove(beanName);
                    }
                }
            }
        }
    }
    return singletonObject;
}

// AbstractAutowireCapableBeanFactory.doCreateBean() - 放入三级缓存
protected Object doCreateBean(String beanName, RootBeanDefinition mbd, Object[] args) {
    // 实例化Bean
    BeanWrapper instanceWrapper = createBeanInstance(beanName, mbd, args);
    Object bean = instanceWrapper.getWrappedInstance();
    
    // 如果是单例、允许循环依赖、且正在创建中
    boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
            isSingletonCurrentlyInCreation(beanName));
    if (earlySingletonExposure) {
        // 放入三级缓存
        addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
    }
    
    // 属性赋值（可能触发循环依赖的解决）
    populateBean(beanName, mbd, instanceWrapper);
    
    // 初始化
    Object exposedObject = initializeBean(beanName, bean, mbd);
    
    return exposedObject;
}

// 获取提前暴露的Bean引用（可能创建代理）
protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
    Object exposedObject = bean;
    if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
        for (BeanPostProcessor bp : getBeanPostProcessors()) {
            if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {
                SmartInstantiationAwareBeanPostProcessor ibp = 
                    (SmartInstantiationAwareBeanPostProcessor) bp;
                exposedObject = ibp.getEarlyBeanReference(exposedObject, beanName);
            }
        }
    }
    return exposedObject;
}
```

### 构造器注入的循环依赖

Spring无法解决**构造器注入**的循环依赖！

**原因：** 构造器注入在实例化阶段就需要依赖，此时对象还没创建，无法放入三级缓存。

```java
@Service
public class UserService {
    private final OrderService orderService;
    
    // 构造器注入：Spring无法解决循环依赖
    public UserService(OrderService orderService) {
        this.orderService = orderService;
    }
}

@Service
public class OrderService {
    private final UserService userService;
    
    public OrderService(UserService userService) {
        this.userService = userService;
    }
}
```

**解决方案：**

1. **改为Setter/字段注入**

```java
@Service
public class UserService {
    @Autowired
    private OrderService orderService; // 可以解决的循环依赖
}
```

2. **使用@Lazy**

```java
@Service
public class UserService {
    private final OrderService orderService;
    
    public UserService(@Lazy OrderService orderService) {
        this.orderService = orderService;
    }
}
```

`@Lazy`会创建代理对象，延迟真正依赖的获取。

3. **重新设计，打破循环**

```java
@Service
public class UserService {
    // 不直接依赖OrderService
}

@Service
public class OrderService {
    @Autowired
    private UserService userService;
}
```

---

## 设计模式解析

### 1. 模板方法模式（Template Method）

**体现：** `AbstractAutowireCapableBeanFactory.doCreateBean()`定义了Bean创建的标准流程（实例化→属性赋值→初始化），子类可以重写特定步骤。

**为什么：** 保证所有Bean都遵循相同的创建流程，同时允许特定类型的Bean自定义某些步骤。

### 2. 观察者模式（Observer）

**体现：** `ApplicationListener`监听容器生命周期事件（ContextRefreshedEvent、ContextClosedEvent）。

**为什么：** 生命周期各阶段需要通知外部系统，如启动后预热缓存、关闭前释放连接池。

### 3. 责任链模式（Chain of Responsibility）

**体现：** `BeanPostProcessor`链式执行，每个Processor决定是否处理或传递给下一个。

**为什么：** 不同的扩展点（AOP、依赖注入、初始化注解）可以独立实现，互不影响。

### 4. 代理模式（Proxy）

**体现：** `AbstractAutoProxyCreator`在初始化后生成AOP代理对象。

**为什么：** 在不修改原始Bean代码的情况下，添加事务、日志、权限等横切功能。

### 5. 状态模式（State Pattern）

**体现：** Bean在不同生命周期阶段处于不同状态（创建中、初始化中、就绪、销毁中）。

**为什么：** 明确区分Bean的不同状态，便于管理和调试。

---

## 实战案例：工业级生命周期管理

### 案例1：连接池的生命周期管理

```java
@Component
public class ConnectionPool implements InitializingBean, DisposableBean {
    private BlockingQueue<Connection> pool;
    private int maxConnections = 20;
    private DataSource dataSource;
    
    @Autowired
    public ConnectionPool(DataSource dataSource) {
        this.dataSource = dataSource;
    }
    
    @Override
    public void afterPropertiesSet() throws Exception {
        // 初始化连接池
        pool = new ArrayBlockingQueue<>(maxConnections);
        for (int i = 0; i < maxConnections; i++) {
            pool.add(dataSource.getConnection());
        }
        System.out.println("连接池初始化完成，大小：" + maxConnections);
    }
    
    public Connection getConnection() throws InterruptedException {
        return pool.take();
    }
    
    public void releaseConnection(Connection conn) {
        pool.offer(conn);
    }
    
    @Override
    public void destroy() throws Exception {
        // 销毁连接池
        for (Connection conn : pool) {
            try {
                conn.close();
            } catch (SQLException e) {
                System.err.println("关闭连接失败：" + e.getMessage());
            }
        }
        pool.clear();
        System.out.println("连接池已销毁");
    }
}
```

### 案例2：缓存预热与刷新

```java
@Component
public class CacheWarmer implements ApplicationListener<ContextRefreshedEvent> {
    
    @Autowired
    private UserService userService;
    
    @Autowired
    private CacheManager cacheManager;
    
    @Override
    public void onApplicationEvent(ContextRefreshedEvent event) {
        // 容器启动完成后预热缓存
        System.out.println("开始预热缓存...");
        
        // 预热用户数据
        List<User> users = userService.findAll();
        Cache userCache = cacheManager.getCache("users");
        for (User user : users) {
            userCache.put(user.getId(), user);
        }
        
        System.out.println("缓存预热完成，共加载 " + users.size() + " 条用户数据");
    }
}
```

### 案例3：优雅关闭与资源清理

```java
@Component
public class GracefulShutdown implements DisposableBean {
    
    @Autowired
    private ExecutorService executorService;
    
    @Autowired
    private MessageQueue messageQueue;
    
    @Override
    public void destroy() throws Exception {
        System.out.println("开始优雅关闭...");
        
        // 1. 停止接收新请求
        // 2. 等待正在处理的任务完成
        executorService.shutdown();
        if (!executorService.awaitTermination(60, TimeUnit.SECONDS)) {
            executorService.shutdownNow();
        }
        
        // 3. 刷新消息队列
        messageQueue.flush();
        
        // 4. 关闭连接
        messageQueue.close();
        
        System.out.println("优雅关闭完成");
    }
}
```

### 案例4：多阶段初始化

```java
@Component
public class MultiStageInitBean implements 
        BeanNameAware, ApplicationContextAware, InitializingBean, 
        SmartLifecycle {
    
    private String beanName;
    private ApplicationContext context;
    private boolean running = false;
    
    // 阶段1：Aware接口注入
    @Override
    public void setBeanName(String name) {
        this.beanName = name;
        System.out.println("阶段1 - BeanNameAware: " + name);
    }
    
    @Override
    public void setApplicationContext(ApplicationContext context) {
        this.context = context;
        System.out.println("阶段1 - ApplicationContextAware");
    }
    
    // 阶段2：InitializingBean
    @Override
    public void afterPropertiesSet() {
        System.out.println("阶段2 - afterPropertiesSet()");
    }
    
    // 阶段3：SmartLifecycle（所有Bean初始化完成后）
    @Override
    public void start() {
        System.out.println("阶段3 - SmartLifecycle.start()");
        this.running = true;
    }
    
    @Override
    public void stop() {
        System.out.println("阶段3 - SmartLifecycle.stop()");
        this.running = false;
    }
    
    @Override
    public boolean isRunning() {
        return running;
    }
    
    @Override
    public int getPhase() {
        return 100; // 执行顺序，数值越小越先执行
    }
}
```

---

## 对比分析：Spring vs 其他框架

### Spring Bean生命周期 vs JSR-330 @Inject生命周期

| 特性 | Spring | JSR-330 (CDI) |
|------|--------|---------------|
| 初始化注解 | @PostConstruct | @PostConstruct |
| 销毁注解 | @PreDestroy | @PreDestroy |
| 初始化接口 | InitializingBean | 无 |
| 销毁接口 | DisposableBean | 无 |
| Aware接口 | 多种Aware | 无 |
| 代理生成 | 初始化后 | 注入时 |
| 扩展点 | BeanPostProcessor | Interceptor |

### Spring vs EJB生命周期

| 特性 | Spring | EJB |
|------|--------|-----|
| 轻量化 | 是 | 否（需要容器） |
| 生命周期回调 | 灵活 | 规范固定 |
| 循环依赖支持 | 三级缓存解决 | 有限支持 |
| 销毁控制 | 容器关闭时 | 实例池管理 |
| 启动速度 | 快 | 慢 |

---

## 性能分析与优化

### 1. 减少初始化阶段的操作

```java
// 避免在初始化阶段连接远程服务或加载大量数据
@Component
public class BadExample {
    @PostConstruct
    public void init() {
        // 阻塞启动！
        loadHugeDataFromDatabase();
        connectToRemoteService();
    }
}

// 正确做法：异步或延迟加载
@Component
public class GoodExample implements ApplicationListener<ContextRefreshedEvent> {
    @Override
    public void onApplicationEvent(ContextRefreshedEvent event) {
        CompletableFuture.runAsync(() -> {
            loadHugeDataFromDatabase();
        });
    }
}
```

### 2. prototype Bean的性能影响

```java
// prototype Bean每次getBean都创建新实例，开销较大
@Scope("prototype")
@Component
public class PrototypeBean {
    public PrototypeBean() {
        // 复杂的构造逻辑
    }
}

// 优化：使用ObjectFactory延迟创建
@Component
public class SingletonBean {
    @Autowired
    private ObjectFactory<PrototypeBean> prototypeBeanFactory;
    
    public void doSomething() {
        PrototypeBean bean = prototypeBeanFactory.getObject(); // 按需创建
    }
}
```

### 3. BeanPostProcessor的性能影响

```java
// 每个BeanPostProcessor都会处理所有Bean
// 优化：先判断类型，避免不必要的处理
@Component
public class OptimizedProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        // 快速过滤
        if (!bean.getClass().getPackage().getName().startsWith("com.example.service")) {
            return bean;
        }
        // 处理逻辑...
        return bean;
    }
}
```

### 4. 生命周期监控

```java
@Component
public class LifecycleMetrics implements BeanPostProcessor {
    private final Map<String, Long> startTimes = new ConcurrentHashMap<>();
    
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        startTimes.put(beanName, System.currentTimeMillis());
        return bean;
    }
    
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        long cost = System.currentTimeMillis() - startTimes.get(beanName);
        if (cost > 100) {
            System.out.println("警告：Bean " + beanName + " 初始化耗时 " + cost + "ms");
        }
        return bean;
    }
}
```

---

## 常见陷阱与最佳实践

### 陷阱1：在@PostConstruct中调用依赖Bean的方法

```java
@PostConstruct
public void init() {
    // 此时依赖已经注入，但依赖的Bean可能还没初始化完
    orderService.init(); // 如果OrderService也依赖本Bean，可能出问题
}
```

**最佳实践：** @PostConstruct中只做本Bean的初始化，避免在初始化阶段触发复杂依赖链。需要联动初始化时，用ApplicationListener监听ContextRefreshedEvent。

### 陷阱2：销毁方法不处理异常

```java
@PreDestroy
public void cleanup() {
    connection.close(); // 可能抛异常，导致后续销毁不执行
}
```

**最佳实践：** 销毁方法中捕获异常，确保资源释放。

```java
@PreDestroy
public void cleanup() {
    try {
        if (connection != null) connection.close();
    } catch (Exception e) {
        logger.error("Close connection failed", e);
    }
}
```

### 陷阱3：BeanPostProcessor中处理所有Bean导致性能问题

```java
@Override
public Object postProcessAfterInitialization(Object bean, String beanName) {
    // 对每个Bean都执行，实际上只需要处理特定类型
    // 大量Bean时性能损耗明显
    return bean;
}
```

**最佳实践：** 先判断Bean类型，只对目标Bean处理。

```java
@Override
public Object postProcessAfterInitialization(Object bean, String beanName) {
    if (bean instanceof DataSource) {
        // 只处理DataSource
    }
    return bean;
}
```

### 陷阱4：混淆初始化顺序

很多开发者不清楚@PostConstruct、InitializingBean、init-method的执行顺序，导致初始化逻辑放错位置。

**最佳实践：** 记住顺序：@PostConstruct → InitializingBean.afterPropertiesSet() → init-method。三种方式选一种即可，不要混用，避免维护困难。

### 陷阱5：prototype Bean的内存泄漏

```java
@Scope("prototype")
@Component
public class PrototypeBean {
    @Autowired
    private ApplicationContext context; // 持有容器引用
}
```

prototype Bean如果依赖singleton Bean（如ApplicationContext），且该singleton持有prototype引用，会导致prototype Bean无法被GC。

**最佳实践：** prototype Bean避免直接依赖会缓存它的singleton。或用ObjectFactory.getObject()延迟获取。

### 陷阱6：忽视SmartLifecycle的使用

```java
// 错误：在所有Bean初始化完成前就启动服务
@Component
public class BadService {
    @PostConstruct
    public void init() {
        startServer(); // 可能依赖的Bean还没初始化完
    }
}

// 正确：使用SmartLifecycle
@Component
public class GoodService implements SmartLifecycle {
    @Override
    public void start() {
        startServer(); // 所有Bean初始化完成后才调用
    }
    
    @Override
    public int getPhase() {
        return Integer.MAX_VALUE; // 最后启动
    }
}
```

---

## 面试题与参考答案

### Q1：Spring Bean的完整生命周期？

**答：** 
1. **实例化**（Instantiation）：调用构造器创建对象；
2. **属性赋值**（Populate）：注入依赖；
3. **初始化**（Initialization）：调用Aware接口 → BeanPostProcessor.before → @PostConstruct → InitializingBean.afterPropertiesSet() → init-method → BeanPostProcessor.after；
4. **使用**；
5. **销毁**：@PreDestroy → DisposableBean.destroy() → destroy-method。

**深度补充：**
- 三级缓存解决循环依赖发生在实例化之后、初始化之前
- AOP代理在初始化阶段的最后生成
- Aware接口在初始化之前调用

### Q2：@PostConstruct、InitializingBean、init-method的执行顺序？

**答：** 
1. **@PostConstruct**（JSR-250注解）；
2. **InitializingBean.afterPropertiesSet()**；
3. **自定义init-method**。

三种方式功能相同，选一种即可。注解方式最简洁，是Spring推荐的方式。

**深度补充：**
- @PostConstruct由CommonAnnotationBeanPostProcessor处理
- InitializingBean.afterPropertiesSet()由invokeInitMethods()直接调用
- init-method由反射调用

### Q3：BeanPostProcessor和BeanFactoryPostProcessor的区别？

**答：** BeanFactoryPostProcessor在BeanDefinition加载后、Bean实例化前执行，操作的是配置元数据（如替换${}占位符）。BeanPostProcessor在Bean实例化后、初始化前后执行，操作的是Bean实例（如生成AOP代理、处理@Autowired）。两者执行时机完全不同。

**深度补充：**
- BeanFactoryPostProcessor的实现类：ConfigurationClassPostProcessor、PropertySourcesPlaceholderConfigurer
- BeanPostProcessor的实现类：AutowiredAnnotationBeanPostProcessor、AbstractAutoProxyCreator

### Q4：Spring三级缓存解决循环依赖的原理？

**答：** 
三级缓存：
1. **singletonObjects**：完整单例Bean；
2. **earlySingletonObjects**：提前暴露的Bean（已实例化未初始化）；
3. **singletonFactories**：ObjectFactory（可生成代理对象）。

流程：A实例化后放入三级缓存 → A属性赋值需B → B实例化放入三级缓存 → B属性赋值需A → 从三级缓存取A的工厂生成A → B完成初始化入一级缓存 → A拿到B完成初始化。

**深度补充：**
- 构造器注入的循环依赖无法解决（实例化阶段就需要依赖）
- @Lazy可以解决构造器循环依赖（注入代理对象）
- 三级缓存的设计主要是为了处理AOP代理对象的循环依赖

### Q5：为什么构造器注入不能解决循环依赖？如何解决？

**答：** 构造器注入在实例化阶段就需要依赖（构造器参数），此时对象还未创建，无法提前暴露到缓存。字段/Setter注入是先实例化（无参构造），再注入属性，实例化后就能暴露。

解决方案：
1. 改为Setter/字段注入；
2. 使用@Lazy，注入代理对象延迟真正依赖获取；
3. 重构设计，打破循环。

### Q6：ApplicationContextAware和@Autowire ApplicationContext有什么区别？

**答：** 功能相同，都是获取ApplicationContext。区别：ApplicationContextAware是接口方式，Spring回调setApplicationContext()方法注入；@Autowired是字段/构造器注入方式。推荐用@Autowired（更简洁）或构造器注入。实现Aware接口的侵入性更强。

**深度补充：**
- ApplicationContextAwareProcessor负责处理所有Aware接口
- 构造器注入ApplicationContext不需要@Autowired（Spring 4.3+）

### Q7：singleton和prototype作用域的区别？

**答：** 
- **singleton**（默认）：每个Spring容器只有一个实例，所有请求共享。
- **prototype**：每次请求都创建新实例。

singleton Bean注入prototype Bean时，由于只注入一次，实际用的还是同一个prototype实例。解决方案：
1. 用@Lookup方法注入；
2. 实现ApplicationContextAware，每次getBean()获取；
3. 使用ObjectFactory.getObject()。

**深度补充：**
- prototype Bean的销毁不由Spring管理
- request/session作用域需要Web环境
- 自定义作用域需要实现Scope接口

### Q8：Spring中AOP代理是在哪个阶段生成的？

**答：** AOP代理在BeanPostProcessor的postProcessAfterInitialization方法中生成。具体是AbstractAutoProxyCreator类实现的。此时Bean已经完成初始化，Spring检查是否有切面匹配该Bean，如果有则创建代理对象替换原始Bean放入容器。

**深度补充：**
- JDK动态代理：目标类实现接口
- CGLIB代理：目标类不实现接口
- Spring Boot 2.x+默认使用CGLIB

### Q9：SmartLifecycle和InitializingBean有什么区别？

**答：** InitializingBean在Bean初始化时调用（属于Bean创建过程）。SmartLifecycle在所有Bean初始化完成后调用（属于容器生命周期），支持启动/停止控制，可以控制执行顺序（getPhase()）。

**适用场景：**
- InitializingBean：Bean自身的初始化
- SmartLifecycle：需要等待所有Bean就绪后才启动的服务（如定时任务、消息监听）

### Q10：如何自定义Bean的创建和销毁逻辑？

**答：** 多种方式：
1. 实现InitializingBean/DisposableBean接口
2. 使用@PostConstruct/@PreDestroy注解
3. 在@Bean中指定initMethod/destroyMethod
4. 实现BeanPostProcessor
5. 实现SmartLifecycle

**推荐顺序：** @PostConstruct/@PreDestroy > InitializingBean/DisposableBean > init-method/destroy-method

---

*此文原创，转载请注明出处。*
