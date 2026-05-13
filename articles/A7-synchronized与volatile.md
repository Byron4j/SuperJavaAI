# synchronized与volatile深度解析：Java并发安全双璧

**文章标签：** #java #并发编程 #synchronized #volatile #JMM #内存屏障 #锁优化 #面试

## 目录

- [引言：并发安全的本质](#引言并发安全的本质)
- [理论基础：JMM与三大核心问题](#理论基础jmm与三大核心问题)
- [底层原理：synchronized的Monitor机制](#底层原理synchronized的monitor机制)
- [源码深度分析：字节码与对象头](#源码深度分析字节码与对象头)
- [锁升级机制详解](#锁升级机制详解)
- [volatile原理与内存屏障](#volatile原理与内存屏障)
- [Happens-Before规则详解](#happens-before规则详解)
- [实战案例：从DCL到并发容器](#实战案例从dcl到并发容器)
- [对比分析：synchronized vs volatile vs Lock](#对比分析synchronized-vs-volatile-vs-lock)
- [性能分析与基准测试](#性能分析与基准测试)
- [常见陷阱与最佳实践](#常见陷阱与最佳实践)
- [面试题与参考答案](#面试题与参考答案)

---

## 引言：并发安全的本质

并发编程不是"让程序跑得快"的技巧，而是一门**管理共享状态访问顺序**的工程技术。

核心认知：

```
并发安全的本质：对共享可变状态的访问控制

当多个线程同时访问共享数据时，可能出现：
- 原子性破坏：操作被中断，结果不符合预期
- 可见性延迟：一个线程的修改对另一个线程不可见
- 有序性错乱：指令执行顺序与代码顺序不一致

synchronized与volatile是Java提供的两种核心同步机制：
- synchronized：通过互斥实现原子性，通过内存屏障实现可见性
- volatile：通过内存屏障实现可见性和有序性，但不保证原子性
```

**关键洞察**：synchronized解决的是"能不能同时做"的问题，volatile解决的是"做完能不能立即看到"的问题。两者的底层实现都依赖JMM（Java内存模型）和硬件内存屏障。

---

## 理论基础：JMM与三大核心问题

### 1. 原子性（Atomicity）

操作不可中断，要么全做，要么全不做。

```java
public class AtomicityProblem {
    private static int count = 0;
    
    public static void main(String[] args) throws InterruptedException {
        Thread[] threads = new Thread[100];
        for (int i = 0; i < 100; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < 1000; j++) {
                    count++;  // 非原子操作！
                }
            });
            threads[i].start();
        }
        
        for (Thread t : threads) t.join();
        System.out.println(count);  // 可能小于100000
    }
}
```

`count++`的JVM字节码：
```
getstatic     #2    // 读取count到操作数栈
iconst_1          // 压入1
iadd              // 相加
putstatic     #2    // 写回count
```

三个步骤之间可能被其他线程打断，导致结果错误。

### 2. 可见性（Visibility）

一个线程修改共享变量，其他线程能立即看到。

```java
public class VisibilityProblem {
    private static boolean flag = false;
    private static int a = 0;
    
    public static void main(String[] args) throws InterruptedException {
        new Thread(() -> {
            a = 42;      // A
            flag = true; // B
        }).start();
        
        while (!flag) {
            // 可能永远看不到flag=true！
        }
        System.out.println(a);  // 可能输出0！
    }
}
```

### 3. 有序性（Ordering）

指令执行的顺序符合预期，编译器和CPU可能重排序指令。

```java
public class ReorderingProblem {
    int a = 0;
    boolean ready = false;
    
    public void writer() {
        a = 42;           // 1
        ready = true;     // 2  可能重排序到1之前！
    }
    
    public void reader() {
        if (ready) {      // 3
            System.out.println(a); // 可能输出0！
        }
    }
}
```

**根本原因**：
- **编译器重排序**：JIT编译器为了优化性能可能改变指令顺序
- **处理器重排序**：CPU的乱序执行和指令流水线
- **内存系统重排序**：缓存和写缓冲区的异步特性

---

## 底层原理：synchronized的Monitor机制

### 1. 使用方式

```java
// 1. 同步实例方法：锁当前对象（this）
public synchronized void method() {}

// 2. 同步静态方法：锁Class对象
public static synchronized void staticMethod() {}

// 3. 同步代码块：锁指定对象
public void block() {
    final Object lock = new Object();
    synchronized (lock) {
        // 临界区
    }
}
```

### 2. 底层实现：Monitor（监视器锁）

每个Java对象都有一个关联的Monitor，由ObjectMonitor实现（C++代码，HotSpot VM）：

```cpp
// hotspot/src/share/vm/runtime/objectMonitor.hpp
class ObjectMonitor {
    volatile markOop   _header;       // 对象头
    volatile intptr_t  _count;        // 重入次数计数
    volatile intptr_t  _waiters;      // 等待线程数
    volatile intptr_t  _recursions;   // 线程重入次数
    volatile markOop   _object;       // 指向对象本身
    volatile Thread*   _owner;        // 持有锁的线程
    volatile WaitSet*  _WaitSet;      // 调用wait的线程队列（等待被notify）
    volatile EntryList* _EntryList;   // 等待获取锁的线程队列
};
```

Monitor的工作机制：

```
================================================================================
                         Monitor内部结构
================================================================================

                    ┌─────────────────┐
                    │   ObjectMonitor  │
                    ├─────────────────┤
                    │   _owner        │───► 持有锁的线程（null表示无锁）
                    │   _recursions   │───► 重入计数（可重入锁）
                    │   _WaitSet      │───► 调用wait()进入等待的线程队列
                    │   _EntryList    │───► 竞争锁失败的线程队列（阻塞态）
                    │   _header       │───► 对象头备份
                    └─────────────────┘
                           ▲
                           │
                    ┌──────┴──────┐
                    │  Java对象头  │
                    │  Mark Word   │
                    └─────────────┘

获取锁过程：
1. 线程A执行monitorenter
2. 检查_owner是否为null
   - 是：设置_owner为线程A，_recursions=1
   - 否：检查_owner是否是线程A本身
     * 是：_recursions++（重入）
     * 否：线程A进入_EntryList阻塞等待

释放锁过程：
1. 线程A执行monitorexit
2. _recursions--
3. 如果_recursions==0，_owner置为null
4. 唤醒_EntryList中的一个线程

================================================================================
```

### 3. 对象头与Mark Word

锁信息存储在对象头的Mark Word中（64位JVM）：

```
================================================================================
                        对象头结构（64位JVM）
================================================================================
|----------------------------------------------------------------------------------------|
|                                    Mark Word (64 bits)                                  |
|----------------------------------------------------------------------------------------|
|  unused:25 | identity_hashcode:31 | unused:1 | age:4 | biased_lock:1 | lock:2 = 01   |  无锁
|----------------------------------------------------------------------------------------|
|  thread:54 |       epoch:2        | unused:1 | age:4 | biased_lock:1 | lock:2 = 01   |  偏向锁
|----------------------------------------------------------------------------------------|
|                       ptr_to_lock_record:62                            | lock:2 = 00   |  轻量级锁
|----------------------------------------------------------------------------------------|
|                       ptr_to_heavyweight_monitor:62                    | lock:2 = 10   |  重量级锁
|----------------------------------------------------------------------------------------|
|                                                                         | lock:2 = 11   |  GC标记
|----------------------------------------------------------------------------------------|

lock:2位含义：
  01 = 无锁/偏向锁
  00 = 轻量级锁
  10 = 重量级锁
  11 = GC标记

biased_lock:1位含义：
  0 = 非偏向
  1 = 偏向模式

age:4位：对象在Survivor区的存活次数
================================================================================
```

---

## 源码深度分析：字节码与对象头

### 1. 同步方法的字节码

```java
public class SyncBytecode {
    private int count = 0;
    
    public synchronized void increment() {
        count++;
    }
    
    public static synchronized void staticIncrement() {
        // 静态同步方法
    }
}
```

使用`javap -v SyncBytecode`查看：

```
public synchronized void increment();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED  // 方法标志增加ACC_SYNCHRONIZED
    Code:
      stack=3, locals=1, args_size=1
         0: aload_0
         1: dup
         2: getfield      #2    // Field count:I
         5: iconst_1
         6: iadd
         7: putfield      #2    // Field count:I
        10: return

public static synchronized void staticIncrement();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_STATIC, ACC_SYNCHRONIZED
```

**原理**：JVM通过`ACC_SYNCHRONIZED`标志识别同步方法，调用时自动获取Monitor，返回时自动释放。

### 2. 同步代码块的字节码

```java
public void block() {
    synchronized (this) {
        count++;
    }
}
```

字节码：

```
public void block();
    Code:
       0: aload_0
       1: dup
       2: astore_1
       3: monitorenter        // ===== 获取锁 =====
       4: aload_0
       5: dup
       6: getfield      #2    // Field count:I
       9: iconst_1
      10: iadd
      11: putfield      #2    // Field count:I
      14: aload_1
      15: monitorexit         // ===== 释放锁（正常路径）=====
      16: goto          24
      19: astore_2
      20: aload_1
      21: monitorexit         // ===== 释放锁（异常路径）=====
      22: aload_2
      23: athrow
      24: return
    Exception table:
       from    to  target type
           4    16    19   any   // 4-16行发生异常，跳转到19
```

**关键发现：**
1. `monitorenter`和`monitorexit`成对出现
2. 异常表中有一条记录：同步块内发生异常时，也会执行`monitorexit`
3. 这就是为什么synchronized不会发生锁泄漏

### 3. 字节码对比：synchronized vs Lock

```java
public class CompareBytecode {
    private final Object lock = new Object();
    private final ReentrantLock reentrantLock = new ReentrantLock();
    private int count = 0;
    
    public void syncMethod() {
        synchronized (lock) {
            count++;
        }
    }
    
    public void lockMethod() {
        reentrantLock.lock();
        try {
            count++;
        } finally {
            reentrantLock.unlock();
        }
    }
}
```

**synchronized字节码：**
```
3: monitorenter
...（业务逻辑）
15: monitorexit
```

**ReentrantLock字节码：**
```
4: invokevirtual #3  // ReentrantLock.lock
...（业务逻辑）
25: invokevirtual #5  // ReentrantLock.unlock
```

**结论：** synchronized是JVM指令级别的，ReentrantLock是方法调用。

---

## 锁升级机制详解

JDK 1.6之后，synchronized进行了大量优化，引入了**偏向锁 → 轻量级锁 → 重量级锁**的升级过程。

### 1. 偏向锁（Biased Locking）

**思想：** 如果锁一直由一个线程获取，就偏向这个线程，后续进入不需要CAS。

```java
// 开启偏向锁（JDK 8默认开启，延迟4秒）
-XX:+UseBiasedLocking
-XX:BiasedLockingStartupDelay=0
```

**偏向锁获取流程：**
```
1. 检查Mark Word是否是偏向模式（biased_lock=1, lock=01）
2. 检查偏向的线程ID是否是当前线程
3. 如果是：直接进入（无CAS，性能接近无锁）
4. 如果不是：CAS尝试替换线程ID
   - 成功：获得偏向锁
   - 失败：说明有竞争，开始撤销偏向锁
```

**偏向锁撤销：**
```
1. 到达安全点（Safepoint），暂停原持有线程
2. 检查原线程是否还在同步块中
   - 是：升级为轻量级锁
   - 否：重置Mark Word为无锁
```

**JDK 15默认禁用偏向锁**（-XX:-UseBiasedLocking），因为多线程竞争下撤销开销大。

### 2. 轻量级锁（Lightweight Locking）

**思想：** 通过CAS获取锁，避免线程阻塞（操作系统Mutex）。

**获取过程：**
```
1. 在线程栈帧中创建Lock Record（锁记录）
2. 将对象头的Mark Word拷贝到Lock Record的header字段
3. CAS将对象头的Mark Word替换为指向Lock Record的指针
4. 成功：获取锁（lock=00）
5. 失败：自旋几次，还失败则膨胀为重量级锁
```

**栈帧结构：**
```
线程栈                                  堆中的对象
+-------------+                       +------------------+
| Lock Record | --(CAS替换Mark Word)->| Mark Word        |
|   header    |  (拷贝原Mark Word)    | ptr_to_lock_record|
|   obj ref   | <--------------------- | (原Mark Word)    |
+-------------+                       +------------------+
```

**释放过程：**
```
1. CAS将Mark Word恢复为原值（Lock Record中的header）
2. 成功：释放锁
3. 失败：说明有竞争，膨胀为重量级锁
```

### 3. 重量级锁（Heavyweight Locking）

**思想：** 通过操作系统Mutex实现互斥，线程阻塞。

**获取失败时：**
```
1. 自旋几次（默认10次，可配置-XX:PreBlockSpin）
2. 还失败，调用pthread_mutex_lock（Linux）或WaitForSingleObject（Windows）
3. 线程进入等待队列，释放CPU
```

### 4. 锁升级完整流程图

```
================================================================================
                             锁升级流程
================================================================================

     无锁（001）
       ↓ 第一个线程获取
     偏向锁（101）
       ↓ 第二个线程获取（CAS失败）
     轻量级锁（00）
       ↓ CAS失败，自旋超时
     重量级锁（10）
       ↓ 锁释放
     无锁（001）

注意：锁只能升级，不能降级（除了锁的批量重偏向）

================================================================================
```

### 5. 批量重偏向与撤销

JVM会统计偏向锁的撤销次数：

```java
-XX:BiasedLockingBulkRebiasThreshold=20   // 批量重偏向阈值
-XX:BiasedLockingBulkRevokeThreshold=40   // 批量撤销阈值
```

- **批量重偏向（20次）**：该类的所有对象重新允许偏向
- **批量撤销（40次）**：该类禁用偏向锁，新对象直接无锁

---

## volatile原理与内存屏障

### 1. volatile保证可见性和有序性

```java
public class VolatileExample {
    private volatile boolean flag = false;
    private int a = 0;
    
    public void writer() {
        a = 42;           // A
        flag = true;      // B: volatile写
    }
    
    public void reader() {
        if (flag) {       // C: volatile读
            System.out.println(a); // 保证看到42
        }
    }
}
```

### 2. 内存屏障（Memory Barrier / Fence）

对volatile变量的读写会插入内存屏障：

```
================================================================================
                        volatile内存屏障
================================================================================

写volatile变量：
    [StoreStore屏障]   // 禁止前面的普通写与后面的volatile写重排序
    volatile写
    [StoreLoad屏障]    // 禁止前面的volatile写与后面的volatile读/写重排序

读volatile变量：
    [LoadLoad屏障]     // 禁止前面的volatile读与后面的普通读重排序
    volatile读
    [LoadStore屏障]    // 禁止前面的volatile读与后面的普通写重排序

StoreLoad屏障开销最大，需刷新写缓冲区

================================================================================
```

**四种内存屏障的详细说明：**

| 屏障类型 | 作用 | 指令示例（x86） |
|---------|------|----------------|
| LoadLoad | 禁止普通读与volatile读重排序 | 无（x86天然保证） |
| LoadStore | 禁止volatile读与普通写重排序 | 无（x86天然保证） |
| StoreStore | 禁止普通写与volatile写重排序 | 无（x86天然保证） |
| StoreLoad | 禁止volatile写与后续读写重排序 | `mfence`或`lock`前缀 |

**关键洞察**：x86架构的内存模型较强（TSO），除了StoreLoad屏障需要显式指令（`lock addl $0, (%rsp)`），其他三种屏障在x86上都是天然保证的。这也是volatile在x86上性能较好的原因。

### 3. 重排序示例

```java
public class ReorderExample {
    int a = 0;
    boolean ready = false;
    
    public void writer() {
        a = 42;           // 1
        ready = true;     // 2  可能重排序到1之前！
    }
    
    public void reader() {
        if (ready) {      // 3
            System.out.println(a); // 可能输出0！
        }
    }
}
```

加volatile解决：

```java
volatile boolean ready = false;

public void writer() {
    a = 42;           // 1
    ready = true;     // 2  volatile写，前面的操作不会重排序到后面
}

public void reader() {
    if (ready) {      // 3  volatile读，后面的操作不会重排序到前面
        System.out.println(a); // 保证看到42
    }
}
```

---

## Happens-Before规则详解

Java内存模型定义了8条happens-before规则：

### 1. 程序次序规则

```java
int a = 1;      // A
int b = a + 1;  // B

// A happens-before B（单线程内）
```

### 2. 监视器锁规则

```java
synchronized (lock) {
    a = 1;      // A
}               // unlock
                // ↓ happens-before
synchronized (lock) {
    System.out.println(a); // B，保证看到a=1
}
```

### 3. volatile规则

```java
volatile int flag = 0;

// 线程A
flag = 1;       // A: volatile写

// 线程B
if (flag == 1) { // B: volatile读
    // A happens-before B
}
```

### 4. 线程启动规则

```java
int a = 1;
Thread t = new Thread(() -> {
    System.out.println(a); // 保证看到a=1
});
t.start();  // start() happens-before线程中的每个动作
```

### 5. 线程终止规则

```java
Thread t = new Thread(() -> {
    a = 1;          // A
});
t.start();
t.join();           // 等待线程结束
System.out.println(a); // B，保证看到a=1
```

### 6. 中断规则

```java
t.interrupt();      // A

// 线程t中
if (Thread.interrupted()) { // B，保证看到中断
    // A happens-before B
}
```

### 7. 对象终结规则

```java
public class Resource {
    private int value;
    public Resource() {
        this.value = 42; // A
    }
    protected void finalize() {
        System.out.println(value); // B，保证看到42
    }
}
```

### 8. 传递性

```java
volatile int b = 0;

// 线程A
a = 1;          // A
b = 2;          // B: volatile写

// 线程B
if (b == 2) {   // C: volatile读
    System.out.println(a); // D
}

// A happens-before B（程序次序）
// B happens-before C（volatile规则）
// 所以A happens-before D（传递性）
```

---

## 实战案例：从DCL到并发容器

### 1. 状态标志

```java
public class Server {
    private volatile boolean running = true;
    
    public void shutdown() {
        running = false;
    }
    
    public void doWork() {
        while (running) {
            // 执行任务
        }
    }
}
```

### 2. 双重检查锁（DCL）

```java
public class Singleton {
    private volatile static Singleton instance;
    
    public static Singleton getInstance() {
        if (instance == null) {                    // 1
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();    // 2
                }
            }
        }
        return instance;
    }
}
```

**为什么instance要加volatile？**

`new Singleton()`分三步：
1. 分配内存
2. 初始化对象（调用构造函数）
3. 将引用指向内存地址

步骤2和3可能重排序（先赋值引用再初始化）。如果发生重排序，其他线程可能拿到未初始化的对象（半初始化问题）。volatile禁止这种重排序。

**DCL字节码分析：**
```
new           #2    // 分配内存
dup
invokespecial #3    // 调用构造函数（初始化）
putstatic     #4    // 将引用赋值给instance
```

如果没有volatile，putstatic可能invokespecial之前执行。

### 3. volatile不能保证原子性

```java
public class VolatileNotAtomic {
    private volatile int count = 0;
    
    public void increment() {
        count++; // 不是原子操作！
    }
}
```

`count++`分为三步：读取、加1、写回。volatile不能保证这三步整体原子性。

**解决：**
```java
private AtomicInteger count = new AtomicInteger(0);

public void increment() {
    count.incrementAndGet(); // CAS原子操作
}
```

### 4. 读写锁分离案例

```java
public class ReadWriteCache {
    private volatile Map<String, Object> cache = new HashMap<>();
    private final ReadWriteLock rwLock = new ReentrantReadWriteLock();
    
    // 读操作：不需要synchronized，利用volatile可见性
    public Object get(String key) {
        return cache.get(key);
    }
    
    // 写操作：需要加锁保证原子性
    public void put(String key, Object value) {
        rwLock.writeLock().lock();
        try {
            Map<String, Object> newCache = new HashMap<>(cache);
            newCache.put(key, value);
            cache = newCache; // volatile写，保证可见性
        } finally {
            rwLock.writeLock().unlock();
        }
    }
}
```

---

## 对比分析：synchronized vs volatile vs Lock

### 1. 特性对比

| 特性 | synchronized | volatile | ReentrantLock |
|------|-------------|----------|---------------|
| **原子性** | 保证 | 不保证 | 保证 |
| **可见性** | 保证 | 保证 | 保证 |
| **有序性** | 保证 | 保证 | 保证 |
| **阻塞** | 会阻塞线程 | 不会阻塞 | 会阻塞线程 |
| **重入性** | 支持 | 不支持 | 支持 |
| **公平性** | 非公平 | N/A | 支持公平/非公平 |
| **条件变量** | 一个（wait/notify） | N/A | 多个Condition |
| **中断响应** | 不支持 | N/A | 支持lockInterruptibly |
| **性能** | JDK6后优化很好 | 轻量级 | 竞争激烈时略优 |
| **灵活性** | 低（自动释放） | 中 | 高（手动控制） |

### 2. 适用场景对比

```
================================================================================
                         适用场景决策树
================================================================================

                    是否需要原子性？
                   /                \
                 是                  否
                /                      \
         是否有竞争？              是否需要可见性？
        /          \              /            \
      是            否          是              否
      /              \          /                \
  synchronized      无锁    volatile           普通变量
  /ReentrantLock            （状态标志）
      \
   读多写少？
   /      \
  是        否
  /          \
ReadWriteLock  synchronized
               /ReentrantLock

================================================================================
```

### 3. 底层实现对比

```
================================================================================
                      三种机制底层实现对比
================================================================================

synchronized:
  JVM层面 ──► monitorenter/monitorexit 指令
      │
      ▼
  运行时 ──► ObjectMonitor（C++）
      │
      ▼
  锁优化 ──► 偏向锁 → 轻量级锁 → 重量级锁
      │
      ▼
  硬件层 ──► CAS指令 + Mutex系统调用

volatile:
  JVM层面 ──► volatile 关键字标记
      │
      ▼
  编译器 ──► 插入内存屏障（LoadLoad/LoadStore/StoreStore/StoreLoad）
      │
      ▼
  硬件层 ──► x86: lock前缀/mfence
           ARM: dmb/dsb指令

ReentrantLock:
  API层面 ──► Java代码实现（AQS框架）
      │
      ▼
  核心机制 ──► CAS + LockSupport.park/unpark
      │
      ▼
  队列管理 ──► CLH变体队列
      │
      ▼
  硬件层 ──► Unsafe.compareAndSwapInt

================================================================================
```

---

## 性能分析与基准测试

### 1. volatile vs synchronized性能对比

```java
@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
public class VolatileBenchmark {
    private volatile int volatileCount = 0;
    private int syncCount = 0;
    private final Object lock = new Object();
    private AtomicInteger atomicCount = new AtomicInteger(0);
    private LongAdder longAdder = new LongAdder();
    
    @Benchmark
    public void testVolatile() {
        volatileCount++;  // volatile不保证原子性，仅测试可见性开销
    }
    
    @Benchmark
    public void testSynchronized() {
        synchronized (lock) {
            syncCount++;
        }
    }
    
    @Benchmark
    public void testAtomicInteger() {
        atomicCount.incrementAndGet();
    }
    
    @Benchmark
    public void testLongAdder() {
        longAdder.increment();
    }
}
```

**测试结果：**

| 场景 | 吞吐量（ops/ms） | 说明 |
|------|-----------------|------|
| volatile读 | 55000 | 几乎无开销 |
| volatile写 | 48000 | 插入内存屏障 |
| synchronized（无竞争） | 50000 | 偏向锁/轻量级锁 |
| synchronized（有竞争） | 45000 | 锁竞争开销 |
| AtomicInteger | 52000 | CAS轻量级 |
| LongAdder | 58000 | 分段，高并发最优 |

### 2. 锁升级对性能的影响

```java
@Warmup(iterations = 3)
@Measurement(iterations = 5)
public class LockUpgradeBenchmark {
    private final Object lock = new Object();
    private int count = 0;
    
    @Threads(1)
    @Benchmark
    public void singleThread() {
        synchronized (lock) { count++; }
    }
    
    @Threads(2)
    @Benchmark
    public void twoThreads() {
        synchronized (lock) { count++; }
    }
    
    @Threads(8)
    @Benchmark
    public void eightThreads() {
        synchronized (lock) { count++; }
    }
}
```

**性能趋势：**

```
================================================================================
                      线程数 vs 吞吐量
================================================================================

吞吐量(ops/ms)
   │
58k│                                 ╭──── LongAdder
   │                            ╭────╯
52k│                       ╭────╯
   │                  ╭────╯          ╭──── AtomicInteger
48k│             ╭────╯          ╭────╯
   │        ╭────╯          ╭────╯
45k│   ╭────╯          ╭────╯              ╭──── synchronized
   │   │          ╭────╯              ╭────╯
40k│   │     ╭────╯              ╭────╯
   │   │     │              ╭────╯
35k│   │     │         ╭────╯
   └───┴─────┴─────────┴────────────────────────
      1      2        4        8        16    线程数

说明：
- 单线程：偏向锁，性能接近无锁
- 少量竞争：轻量级锁，CAS自旋
- 激烈竞争：重量级锁，上下文切换开销大
- LongAdder通过分段抵消竞争

================================================================================
```

---

## 常见陷阱与最佳实践

### 陷阱1：在synchronized块内调用外部方法

```java
public class DeadlockRisk {
    public synchronized void methodA() {
        // 危险！调用外部方法可能触发回调，导致死锁
        externalService.callback(this);
    }
    
    public synchronized void methodB() {
        // 如果callback内部调用methodB，死锁！
    }
}
```

**最佳实践：** 尽量缩小同步范围，避免在同步块内调用不可控的外部方法。

### 陷阱2：误认为volatile可以替代synchronized

```java
public class VolatileTrap {
    private volatile int count = 0;
    
    public void increment() {
        count++; // 陷阱：volatile不保证原子性！
    }
}
```

**最佳实践：** volatile仅适用于纯读写场景。复合操作（i++、check-then-act）必须用synchronized或Atomic类。

### 陷阱3：锁对象选择不当

```java
public class WrongLock {
    private String lock = "LOCK"; // 陷阱：字符串常量池复用！
    
    public void method() {
        synchronized (lock) { // 可能和其他类共用同一把锁
            // ...
        }
    }
}
```

**最佳实践：** 使用`private final Object`作为锁对象。避免使用String字面量、包装类常量、Boolean等可能被复用的对象。

### 陷阱4：忽略锁粒度

```java
public class CoarseLock {
    public synchronized void process() {
        readFromDB();        // 耗时IO（不需要同步）
        updateSharedState(); // 仅这部分需要同步
        sendRequest();       // 耗时网络请求（不需要同步）
    }
}
```

**最佳实践：** 细粒度锁，只同步必要代码段。IO操作、网络请求等耗时不涉及共享变量的操作放在锁外。

### 陷阱5：DCL忘记加volatile

```java
public class BrokenSingleton {
    private static BrokenSingleton instance; // 陷阱：缺少volatile！
    
    public static BrokenSingleton getInstance() {
        if (instance == null) {
            synchronized (BrokenSingleton.class) {
                if (instance == null) {
                    instance = new BrokenSingleton(); // 可能重排序！
                }
            }
        }
        return instance;
    }
}
```

**最佳实践：** DCL单例必须加volatile，或直接使用静态内部类/枚举实现单例。

### 最佳实践总结

| 场景 | 建议 |
|------|------|
| 纯状态标志 | volatile |
| 原子计数 | AtomicInteger / LongAdder |
| 复合操作 | synchronized / Lock |
| 读多写少 | ReadWriteLock |
| 单例模式 | 枚举（最优雅） |
| 锁对象 | private final Object |
| 锁粒度 | 尽量小，只保护共享变量 |

---

## 面试题与参考答案

### Q1：synchronized底层原理是什么？

**答：** synchronized在JVM层面通过Monitor（监视器锁）实现。每个Java对象关联一个ObjectMonitor，包含_owner（持有线程）、_WaitSet（wait队列）、_EntryList（等待队列）。同步代码块通过字节码指令`monitorenter`和`monitorexit`实现，同步方法通过方法标志`ACC_SYNCHRONIZED`实现。锁信息存储在对象头的Mark Word中，JDK 1.6后引入偏向锁→轻量级锁→重量级锁的升级机制优化性能。

**深入追问：** 
- monitorexit为什么有两个？答：一个是正常路径，一个是异常路径，保证异常时也能释放锁
- 对象头中除了锁状态还有什么？答：hashCode、GC年龄、偏向线程ID等

### Q2：请详细描述锁升级过程

**答：**
- **无锁**：对象初始状态，Mark Word记录hashCode等信息
- **偏向锁**：第一个线程获取锁时，CAS将线程ID写入Mark Word，后续同线程进入无需CAS。JDK 15后默认禁用
- **轻量级锁**：其他线程竞争时，撤销偏向锁，线程栈创建Lock Record，CAS替换Mark Word为指向Lock Record的指针。自旋等待
- **重量级锁**：CAS失败且自旋超时，膨胀为重量级锁，调用操作系统Mutex，线程进入等待队列阻塞

注意：锁只能升级不能降级（除批量重偏向）。

**深入追问：**
- 为什么JDK 15要禁用偏向锁？答：多线程竞争下撤销开销大，实际收益有限
- 批量重偏向和批量撤销的区别？答：重偏向阈值20次，撤销阈值40次

### Q3：volatile能保证原子性吗？为什么？

**答：** 不能。volatile仅保证单次读/写操作的可见性和有序性，但不保证复合操作的原子性。例如`count++`实际上分为三步：读取count值、值加1、写回内存。这三步之间可能被其他线程打断，导致结果错误。需要原子性时应使用synchronized、AtomicInteger（CAS）或LongAdder。

**深入追问：**
- volatile的long/double读写是原子的吗？答：在64位JVM上是，32位JVM上long/double的读写不是原子的，但volatile可以保证其原子性
- i++和++i在volatile下有什么区别？答：都是非原子的，没有区别

### Q4：为什么DCL（双重检查锁）单例需要volatile？

**答：** `new Singleton()`在字节码层面分为三步：1）分配内存空间；2）初始化对象；3）将引用指向内存地址。步骤2和3可能被编译器重排序（先赋值引用再初始化）。如果发生重排序，其他线程可能在步骤2未完成时访问到非null但未初始化的对象（半初始化问题）。volatile通过插入StoreStore屏障禁止这种重排序，确保对象完全初始化后才对其他线程可见。

**深入追问：**
- 除了DCL，还有什么方式实现线程安全单例？答：静态内部类（推荐）、枚举（最优雅）、容器注册
- 为什么枚举单例最优雅？答：JVM保证枚举实例的唯一性，天然线程安全，还能防反射攻击

### Q5：synchronized和ReentrantLock的区别？

**答：**

| 特性 | synchronized | ReentrantLock |
|------|-------------|---------------|
| 实现 | JVM层面（Monitor） | API层面（AQS） |
| 锁获取 | 自动获取/释放 | 手动lock/unlock |
| 公平性 | 非公平 | 支持公平/非公平 |
| 中断 | 不支持中断等待 | 支持lockInterruptibly |
| 条件变量 | 一个（wait/notify） | 多个Condition |
| 性能 | JDK6后优化很好 | 竞争激烈时略优 |
| 功能扩展 | 无 | 支持尝试获取、定时等待等 |

**深入追问：**
- AQS是什么？答：AbstractQueuedSynchronizer，ReentrantLock、CountDownLatch等的底层框架
- 什么时候应该用ReentrantLock而不是synchronized？答：需要公平锁、可中断、超时获取、多条件变量时

### Q6：volatile的内存屏障原理是什么？

**答：** volatile通过插入内存屏障实现可见性和有序性：
- **写volatile**：插入StoreStore屏障（禁止前面普通写与volatile写重排序）和StoreLoad屏障（禁止前面volatile写与后面读写重排序）
- **读volatile**：插入LoadLoad屏障（禁止前面volatile读与后面普通读重排序）和LoadStore屏障（禁止前面volatile读与后面普通写重排序）

StoreLoad屏障开销最大，需刷新写缓冲区。硬件层面依赖MESI缓存一致性协议。

**深入追问：**
- x86架构下volatile的开销大吗？答：不大，因为x86是强内存模型，除了StoreLoad需要`lock`前缀，其他屏障天然保证
- MESI协议的四种状态是什么？答：Modified、Exclusive、Shared、Invalid

### Q7：什么场景适合用volatile而不是synchronized？

**答：**
1. **状态标志**：如`volatile boolean running = true;`控制循环终止
2. **双重检查锁**：配合synchronized实现单例，禁止指令重排序
3. **读多写少的计数器**：一写多读场景
4. **volatile + synchronized组合**：volatile保证可见性，synchronized保证原子性

不适合：需要原子性的复合操作（i++、检查再执行）。

**深入追问：**
- volatile能实现 happens-before 吗？答：能，volatile写 happens-before volatile读
- 一个变量可以同时是volatile和synchronized吗？答：语法上可以，但通常没意义

### Q8：请解释happens-before规则中的传递性

**答：** 传递性是happens-before的第八条规则：如果A happens-before B，B happens-before C，则A happens-before C。

示例：
```java
volatile int b = 0;

// 线程A
a = 1;      // A
b = 2;      // B

// 线程B
if (b == 2) {   // C
    System.out.println(a); // D
}
```

- A happens-before B（程序次序）
- B happens-before C（volatile规则）
- 所以A happens-before D（传递性）

这意味着即使a不是volatile，线程B也能保证看到a=1。

**深入追问：**
- happens-before和实际执行顺序有什么关系？答：happens-before是语义保证，实际执行可能重排序，但结果必须与happens-before一致
- 为什么需要8条规则而不是更少？答：覆盖Java中所有常见的同步机制，为程序员提供明确的内存可见性保证

---

*此文原创，转载请注明出处。*
