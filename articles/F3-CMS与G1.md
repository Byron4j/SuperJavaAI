# CMS与G1垃圾收集器深度解析：从算法原理到生产调优实战

**文章标签：** #jvm #垃圾收集器 #cms #g1 #内存管理 #性能调优 #面试

## 目录

- [引言：垃圾收集的本质](#引言垃圾收集的本质)
- [理论基础：垃圾收集算法与内存模型](#理论基础垃圾收集算法与内存模型)
- [CMS收集器：并发低延迟的先行者](#cms收集器并发低延迟的先行者)
- [G1收集器：面向大堆的可预测停顿](#g1收集器面向大堆的可预测停顿)
- [源码深度分析：OpenJDK中的实现机制](#源码深度分析openjdk中的实现机制)
- [实战案例：生产环境GC问题诊断](#实战案例生产环境gc问题诊断)
- [CMS vs G1：全维度对比分析](#cms-vs-g1全维度对比分析)
- [性能分析：停顿时间与吞吐量权衡](#性能分析停顿时间与吞吐量权衡)
- [常见陷阱与最佳实践](#常见陷阱与最佳实践)
- [面试题与参考答案](#面试题与参考答案)

---

## 引言：垃圾收集的本质

垃圾收集（Garbage Collection, GC）不是"自动清理内存"的魔法，而是一门**基于可达性分析的内存资源自动化管理技术**。

核心认知：

```
GC的本质：识别并回收不可达对象占用的内存空间

内存管理的本质：在三个目标间寻找最优平衡
    ┌─────────────┐
    │  低停顿时间  │  ← 用户线程等待时间
    │  (Low Pause)│
    ├─────────────┤
    │  高吞吐量   │  ← 用户代码执行时间占比
    │ (Throughput)│
    ├─────────────┤
    │  低内存开销 │  ← GC本身消耗的额外内存
    │  (Footprint)│
    └─────────────┘
    
不可能三角：同时优化三个目标不可行，必须根据场景取舍
```

**关键洞察**：CMS和G1代表了低延迟GC的两个不同设计哲学——CMS追求"尽可能并发"，G1追求"可预测的停顿"。理解它们的本质差异，是JVM调优的根基。

---

## 理论基础：垃圾收集算法与内存模型

### 1. 垃圾识别的数学基础：可达性分析

```
根集合（GC Roots）的定义：
┌─────────────────────────────────────────┐
│ 1. 虚拟机栈中的本地变量表引用              │
│ 2. 方法区中类静态属性引用                  │
│ 3. 方法区中常量引用                        │
│ 4. 本地方法栈中的JNI引用                   │
│ 5. 虚拟机内部的引用（基本数据类型Class对象） │
│ 6. 所有被同步锁持有的对象                   │
│ 7. 反映JVM内部情况的JMXBean等              │
└─────────────────────────────────────────┘

可达性分析算法：
从GC Roots出发，通过引用链遍历（BFS/DFS）
所有不可达的对象即为垃圾

引用类型与回收策略：
┌──────────┬──────────────────────────────────┐
│ 强引用    │ 永不回收（Object obj = new Object()）│
├──────────┼──────────────────────────────────┤
│ 软引用    │ 内存不足时回收（SoftReference）      │
├──────────┼──────────────────────────────────┤
│ 弱引用    │ 下次GC时回收（WeakReference）        │
├──────────┼──────────────────────────────────┤
│ 虚引用    │ 仅用于回收通知（PhantomReference）   │
└──────────┴──────────────────────────────────┘
```

### 2. 三种基础GC算法

#### 标记-清除（Mark-Sweep）

```
执行流程：
Phase 1 - 标记（Mark）：
    从GC Roots遍历，标记所有可达对象
    
Phase 2 - 清除（Sweep）：
    遍历整个堆，回收未标记对象

内存布局变化：
标记前：[████░░██░░░░████]  （█=存活，░=空闲）
标记后：[████░░██░░░░████]  （标记位记录）
清除后：[████░░██░░░░░░░░]  （存活对象不动）

优点：实现简单，不需要移动对象
缺点：产生内存碎片，分配大对象可能失败
```

#### 标记-复制（Mark-Copy）

```
执行流程：
将内存分为两块：From空间 和 To空间

Phase 1 - 标记存活对象
Phase 2 - 将存活对象复制到To空间（紧凑排列）
Phase 3 - 清空From空间
Phase 4 - 交换From/To角色

内存布局变化：
Before:  From: [██░░██░░░░██]  To: [░░░░░░░░░░░░]
After:   From: [░░░░░░░░░░░░]  To: [██████░░░░░░]

优点：无碎片，分配速度快（指针碰撞）
缺点：内存利用率只有50%
适用：年轻代（对象存活率低，复制代价小）
```

#### 标记-整理（Mark-Compact）

```
执行流程：
Phase 1 - 标记存活对象
Phase 2 - 将存活对象向一端移动（压缩）
Phase 3 - 清理边界外的内存

内存布局变化：
Before: [██░░██░░░░██░░░░██]
After:  [████████████░░░░░░]

优点：无碎片，内存利用率高
缺点：需要移动对象，STW时间长
适用：老年代（对象存活率高，复制代价大）
```

### 3. 分代收集理论基础

```
弱分代假说（Weak Generational Hypothesis）：

"绝大多数对象都是朝生夕死的"

 empirical data:
 ┌──────────────────────────────────────┐
 │ 对象存活时间分布：                     │
 │                                       │
 │ 100% ┤███████████████                  │
 │  80% ┤                ████             │
 │  60% ┤                    ██           │
 │  40% ┤                      █          │
 │  20% ┤                       █         │
 │   0% ┼─────────────────────────█───────│
 │       0s   1s   10s  100s  1000s      │
 │                                       │
 │ 结论：90%+的对象在第一次GC时被回收      │
 └──────────────────────────────────────┘

JVM堆内存分代模型：
┌─────────────────────────────────────────┐
│              堆（Heap）                  │
│  ┌─────────────────────────────────┐   │
│  │        年轻代（Young）           │   │
│  │  ┌─────────┐  ┌──────────────┐ │   │
│  │  │  Eden   │  │Survivor 0/1  │ │   │
│  │  │  (8/10) │  │  (1/10 each) │ │   │
│  │  └─────────┘  └──────────────┘ │   │
│  │       默认比例：8:1:1            │   │
│  └─────────────────────────────────┘   │
│  ┌─────────────────────────────────┐   │
│  │        老年代（Old）             │   │
│  │                                 │   │
│  │    长期存活对象 / 大对象          │   │
│  │                                 │   │
│  └─────────────────────────────────┘   │
│  ┌─────────────────────────────────┐   │
│  │      元空间（Metaspace）         │   │
│  │      （JDK 8+，本地内存）         │   │
│  └─────────────────────────────────┘   │
└─────────────────────────────────────────┘

对象晋升策略：
1. 新对象在Eden分配
2. Minor GC后存活对象进入Survivor
3. Survivor中对象每熬过一次GC，年龄+1
4. 年龄达到阈值（默认15）晋升老年代
5. Survivor中相同年龄对象大小超过50%，大于该年龄的也晋升
```

---

## CMS收集器：并发低延迟的先行者

### CMS设计哲学

```
CMS（Concurrent Mark Sweep）的核心假设：

"在应用运行的同时完成大部分垃圾回收工作，
 仅保留最短的停顿时间"

设计目标：最短回收停顿时间
适用场景：对响应时间敏感的互联网应用
JDK版本：JDK 5引入，JDK 14移除
```

### CMS执行流程深度解析

```
CMS完整执行周期：

┌─────────────────────────────────────────────────────────────┐
│  1. 初始标记（Initial Mark）                                │
│     ┌─────────────────────────────────────────────────────┐ │
│     │  STW（Stop-The-World）                               │ │
│     │  • 标记GC Roots直接关联的对象                         │ │
│     │  • 标记年轻代中引用老年代的对象                        │ │
│     │  • 耗时极短（通常 < 10ms）                           │ │
│     └─────────────────────────────────────────────────────┘ │
│                           ↓                                 │
│  2. 并发标记（Concurrent Mark）                              │
│     ┌─────────────────────────────────────────────────────┐ │
│     │  与用户线程并发执行                                   │ │
│     │  • 从初始标记的对象出发，追踪所有引用链                 │ │
│     │  • 遍历整个老年代的对象图                              │ │
│     │  • 不STW，但占用CPU资源                               │ │
│     │  • 耗时较长（取决于堆大小和对象数量）                   │ │
│     └─────────────────────────────────────────────────────┘ │
│                           ↓                                 │
│  3. 重新标记（Remark）                                      │
│     ┌─────────────────────────────────────────────────────┐ │
│     │  STW                                                │ │
│     │  • 修正并发标记期间变动的标记                          │ │
│     │  • 处理并发阶段引用关系变化（三色标记中的漏标）         │ │
│     │  • 使用增量更新（Incremental Update）算法              │ │
│     │  • 比初始标记长，但远小于Full GC                       │ │
│     └─────────────────────────────────────────────────────┘ │
│                           ↓                                 │
│  4. 并发清除（Concurrent Sweep）                             │
│     ┌─────────────────────────────────────────────────────┐ │
│     │  与用户线程并发执行                                   │ │
│     │  • 清除未被标记的对象，回收内存                        │ │
│     │  • 不整理内存，产生碎片                                │ │
│     │  • 耗时较长                                           │ │
│     └─────────────────────────────────────────────────────┘ │
│                           ↓                                 │
│  5. 并发重置（Concurrent Reset）                             │
│     ┌─────────────────────────────────────────────────────┐ │
│     │  重置CMS内部数据结构，为下次GC做准备                   │ │
│     └─────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘

时间轴视角：
用户线程: ████████████████████████████████████████████████████
GC线程:   ░░░░████░░░░░░░░░░░░░░░░░░░░░░████░░░░░░░░░░░░░░░░░
          ↑   ↑  ↑                    ↑   ↑
          STW  并发标记                STW  并发清除
          初始标记                     重新标记
```

### 三色标记算法与CMS的漏标问题

```
三色标记的核心机制：

每个对象有三种颜色状态：
┌──────────┬──────────────────────────────────────┐
│ 白色     │ 未被访问，可能是垃圾                  │
│ 灰色     │ 已被访问，但引用尚未全部扫描           │
│ 黑色     │ 已被访问，引用已全部扫描（存活）       │
└──────────┴──────────────────────────────────────┘

标记过程：
1. 初始：GC Roots为灰色，其他为白色
2. 处理灰色对象：将其引用对象变为灰色，自身变为黑色
3. 重复步骤2直到没有灰色对象
4. 剩余白色对象即为垃圾

CMS的并发标记问题——漏标：

并发标记期间，用户线程可能修改引用关系：

Before:      Black objA ──→ White objB
                    
用户线程执行: objA.field = null;  // 断开引用
             objC.field = objB;   // 新引用（objC为Black）
             
After:       Black objA          Black objC ──→ White objB
                                     ↑
                              objC是黑色，不会再扫描
                              objB仍然是白色
                              → objB被错误回收！

CMS解决方案：增量更新（Incremental Update）

当黑色对象插入新的指向白色对象的引用时，
将黑色对象重新标记为灰色，下次重新标记阶段处理。

代码逻辑示意：
void write_barrier(Object obj, Field field, Object newVal) {
    // 原值
    Object oldVal = obj.field;
    
    // 赋值
    obj.field = newVal;
    
    // CMS写屏障：如果当前在并发标记阶段
    if (is_concurrent_marking()) {
        // 如果obj是黑色，newVal是白色
        if (is_black(obj) && is_white(newVal)) {
            // 将obj重新标记为灰色，加入重新扫描队列
            mark_gray(obj);
            enqueue_for_remark(obj);
        }
    }
}
```

### CMS参数配置与调优

```bash
# ==================== CMS核心参数 ====================

# 启用CMS收集器
-XX:+UseConcMarkSweepGC

# 年轻代使用ParNew（CMS的搭档）
#（开启CMS后年轻代自动使用ParNew，无需单独设置）

# 老年代占比达到多少时触发CMS（默认68%）
# 关键参数！设置太低频繁GC，设置太高容易CMF
-XX:CMSInitiatingOccupancyFraction=70

# 开启内存碎片整理（Full GC后执行）
-XX:+UseCMSCompactAtFullCollection

# 多少次Full GC后整理一次碎片（默认0，每次Full GC都整理）
-XX:CMSFullGCsBeforeCompaction=5

# 并发线程数（默认 (CPU核心数+3)/4 ）
-XX:ParallelCMSThreads=4

# 强制在类卸载后进行整理
-XX:+CMSClassUnloadingEnabled

# 打印详细GC日志
-XX:+PrintGCDetails
-XX:+PrintGCDateStamps
-Xloggc:/var/log/gc.log

# ==================== 生产环境推荐配置 ====================

# JDK 8，4核8G机器，Web应用
JAVA_OPTS="
  -Xms4g -Xmx4g
  -XX:NewRatio=2
  -XX:SurvivorRatio=8
  -XX:+UseConcMarkSweepGC
  -XX:CMSInitiatingOccupancyFraction=70
  -XX:+UseCMSCompactAtFullCollection
  -XX:CMSFullGCsBeforeCompaction=5
  -XX:+CMSParallelRemarkEnabled
  -XX:+CMSScavengeBeforeRemark
  -XX:+PrintGCDetails
  -XX:+PrintGCDateStamps
  -Xloggc:/var/log/gc.log
"
```

### CMS核心问题：Concurrent Mode Failure

```
Concurrent Mode Failure（并发模式失败）机制：

触发条件：
在CMS并发标记/清除阶段，老年代剩余空间不足以容纳：
1. 新晋升的对象
2. 浮动垃圾（并发阶段产生的新垃圾，但本次不回收）

后果：
CMS退化为Serial Old收集器，执行单线程Full GC
停顿时间从几十毫秒飙升到数秒！

流程示意：
正常CMS:  并发标记 → 并发清除 → 完成
              ↓
         空间不足触发CMF
              ↓
退化Full GC: Serial Old 单线程标记-整理
              ↓
         STW数秒，吞吐量骤降

CMF诊断日志：
```
[GC (CMS Initial Mark) ...]
[CMS-concurrent-mark-start]
[CMS-concurrent-mark: 0.120/0.150 secs]
(CMSConcurrentMark)
[CMS-concurrent-preclean-start]
[CMS-concurrent-preclean: 0.030/0.030 secs]
(CMSConcurrentPreclean)

# 出现以下日志表示发生CMF：
(concurrent mode failure): 8192K->8192K(8192K), 5.1234567 secs]
```
```

### CMS问题演示代码

```java
import java.util.ArrayList;
import java.util.List;

/**
 * CMS Concurrent Mode Failure演示
 * 
 * JVM参数：
 * -Xms128m -Xmx128m -XX:+UseConcMarkSweepGC
 * -XX:CMSInitiatingOccupancyFraction=50
 * -XX:+PrintGCDetails
 * -XX:+PrintGCDateStamps
 * -Xloggc:cmf_demo_gc.log
 */
public class CMSFailureDemo {
    private static final int _1MB = 1024 * 1024;
    
    public static void main(String[] args) throws InterruptedException {
        List<byte[]> list = new ArrayList<>();
        
        System.out.println("开始快速分配对象，模拟CMS并发模式失败...");
        
        // 快速分配大对象，导致CMS来不及回收
        for (int i = 0; i < 100; i++) {
            // 每次分配2MB，快速填满老年代
            list.add(new byte[2 * _1MB]);
            
            // 偶尔释放一些对象，但总体增长
            if (i % 10 == 0 && list.size() > 5) {
                // 只释放少量，制造浮动垃圾效果
                for (int j = 0; j < 2; j++) {
                    list.remove(0);
                }
            }
            
            Thread.sleep(10);
        }
        
        System.out.println("分配完成，查看gc日志是否出现concurrent mode failure");
    }
}

/**
 * 预防CMF的最佳实践示例
 */
public class CMSBestPractice {
    private static final int _1MB = 1024 * 1024;
    private static List<byte[]> cache = new ArrayList<>();
    
    /**
     * 错误的对象分配模式：大对象直接进入老年代
     */
    public void badPractice() {
        // 大数组直接分配在老年代
        byte[] largeArray = new byte[10 * _1MB];
        // 使用完后不立即释放
        cache.add(largeArray);
    }
    
    /**
     * 正确的对象分配模式：控制对象生命周期
     */
    public void goodPractice() {
        // 使用局部变量，方法结束即可回收
        byte[] tempBuffer = new byte[10 * _1MB];
        processData(tempBuffer);
        // tempBuffer在方法结束后变为垃圾， young GC即可回收
    }
    
    /**
     * 使用对象池避免频繁分配大对象
     */
    private static final ThreadLocal<byte[]> BUFFER_POOL = 
        ThreadLocal.withInitial(() -> new byte[10 * _1MB]);
    
    public void withPool() {
        byte[] buffer = BUFFER_POOL.get();
        processData(buffer);
        // 不清除，下次复用，避免重复分配
    }
    
    private void processData(byte[] data) {
        // 处理数据
    }
}
```

---

## G1收集器：面向大堆的可预测停顿

### G1设计哲学

```
G1（Garbage First）的核心假设：

"与其全局优化，不如识别垃圾最多的区域优先回收"

设计目标：在可预测的停顿时间内，回收最大量的垃圾
适用场景：大堆内存（6GB+），需要可预测停顿的服务端应用
JDK版本：JDK 7引入，JDK 9+默认

核心创新：Region-based Heap

传统分代模型 vs G1 Region模型：

传统模型：           G1模型：
┌──────────────┐     ┌──┬──┬──┬──┬──┬──┬──┬──┐
│   年轻代      │     │E │E │S │O │H │E │O │E │
│  ┌──┬──┐     │     ├──┼──┼──┼──┼──┼──┼──┼──┤
│  │Eden      │     │O │E │S │E │O │H │E │O │
│  ├──┤       │     ├──┼──┼──┼──┼──┼──┼──┼──┤
│  │S0 │S1    │     │E │O │E │S │O │E │O │H │
│  └──┴──┘     │     └──┴──┴──┴──┴──┴──┴──┴──┘
├──────────────┤     
│   老年代      │     Region大小：1MB ~ 32MB（2的幂次）
│              │     默认根据堆大小自动计算：
│  连续的大块内存 │     堆 < 4GB: ~1MB
│              │     堆 4-8GB: ~2MB
└──────────────┘     堆 8-16GB: ~4MB
                     堆 16-32GB: ~8MB
                     堆 32-64GB: ~16MB
                     堆 > 64GB: ~32MB
```

### G1 Region类型详解

```
G1中的Region分类：

┌──────────────────────────────────────────┐
│ Eden Region (E)                          │
│ • 新对象分配区域                          │
│ • 多个Eden Region组成年轻代               │
│ • Young GC时全部回收                      │
├──────────────────────────────────────────┤
│ Survivor Region (S)                      │
│ • 存放Minor GC后的存活对象                 │
│ • 默认最多2个Survivor Region              │
│ • 对象年龄达到阈值晋升Old                  │
├──────────────────────────────────────────┤
│ Old Region (O)                           │
│ • 存放长期存活对象                        │
│ • 可单独选择回收（Mixed GC）              │
│ • 维护Remembered Set记录跨Region引用       │
├──────────────────────────────────────────┤
│ Humongous Region (H)                     │
│ • 存放超大对象（>= RegionSize/2）         │
│ • 占用连续多个Region                      │
│ • 直接分配在老年代，不经过年轻代           │
│ • 回收代价高，应尽量避免                   │
├──────────────────────────────────────────┤
│ Free Region                              │
│ • 未分配的Region，加入空闲列表              │
│ • 分配时优先使用                          │
└──────────────────────────────────────────┘

对象分配路径：
TLAB分配 ──→ Eden Region ──→ Survivor Region ──→ Old Region
                ↓
           大对象（> Region/2）
                ↓
         Humongous Region（直接老年代）
```

### G1 Remembered Set（记忆集）机制

```
问题：如何快速找到哪些Old Region引用了年轻代对象？

传统方案：Card Table（卡表）
┌──────────────────────────────────────────┐
│ 老年代划分为512字节的Card                 │
│ 每个Card对应卡表中的一个字节               │
│ 年轻代GC时扫描所有Dirty Card               │
│ 缺点：粒度粗，扫描范围大                   │
└──────────────────────────────────────────┘

G1方案：Per-Region Remembered Set
┌──────────────────────────────────────────┐
│ 每个Region维护一个RSet                     │
│ RSet记录哪些其他Region引用了本Region的对象 │
│ 结构：哈希表，key=RegionId，value=Card集合 │
│                                         │
│ 引用关系图：                               │
│                                         │
│   Region A (Old)                         │
│   ┌─────────┐                            │
│   │ objA    │──────┐                     │
│   └─────────┘      │                     │
│                    ↓ 引用                │
│   Region B (Young)  │                    │
│   ┌─────────┐      │                    │
│   │ objB ◄──┼──────┘                    │
│   └─────────┘                            │
│                                         │
│   Region B的RSet：{(A, card_idx)}        │
│   Young GC时只需扫描Region B的RSet        │
│   就能找到所有引用objB的Old Region对象    │
└──────────────────────────────────────────┘

写屏障维护RSet：

void g1_write_barrier(Object source, Object target) {
    // 原赋值操作
    source.field = target;
    
    // G1写屏障
    if (is_in_different_region(source, target)) {
        // 获取目标Region的RSet
        G1RSet* rset = target->region()->rset();
        
        // 记录引用关系
        rset->add_reference(source->region(), source);
    }
}

RSet的代价：
- 内存开销：RSet约占堆的5%-20%
- 写屏障开销：每次跨Region引用都有额外操作
- 但换来了精确的回收范围控制
```

### G1执行流程：Young GC + Mixed GC + Full GC

```
G1 GC类型：

1. Young GC（纯年轻代回收）
┌─────────────────────────────────────────────┐
│ 触发条件：Eden区满                            │
│ STW：是（多线程并行）                          │
│ 回收范围：所有Eden + Survivor Region          │
│ 目标：快速回收短生命周期对象                    │
│                                             │
│ 执行步骤：                                    │
│ 1. STW，停止所有用户线程                       │
│ 2. 标记Eden/Survivor中的存活对象              │
│ 3. 将存活对象复制到新的Survivor/Old Region    │
│ 4. 清空原Eden和Survivor Region                │
│ 5. 恢复用户线程                               │
│                                             │
│ 时间轴：                                      │
│ 用户线程: ████████████████████████████████████│
│ GC线程:   ░░░░████░░░░░░░░░░░░░░░░░░░░░░░░░░░│
│              ↑                               │
│            Young GC（通常 < 100ms）           │
└─────────────────────────────────────────────┘

2. Concurrent Marking Cycle（并发标记周期）
┌─────────────────────────────────────────────┐
│ 触发条件：老年代占比达到阈值（默认45%）          │
│ STW：多个短暂停顿 + 长时间并发                  │
│ 目标：标记老年代中的存活对象，为Mixed GC做准备   │
│                                             │
│ 执行阶段：                                    │
│ 1. 初始标记（Initial Mark）- STW              │
│    • 标记GC Roots直接引用                      │
│    • 附带在Young GC后执行，几乎无额外开销        │
│                                             │
│ 2. 根区域扫描（Root Region Scanning）          │
│    • 并发执行                                 │
│    • 扫描Survivor区引用老年代的对象             │
│                                             │
│ 3. 并发标记（Concurrent Marking）              │
│    • 并发执行                                 │
│    • 从根出发，遍历整个堆的对象图                │
│    • 使用SATB（Snapshot-At-The-Beginning）算法 │
│                                             │
│ 4. 重新标记（Remark）- STW                     │
│    • 处理SATB队列中的引用变化                   │
│    • 比CMS的Remark更快                         │
│                                             │
│ 5. 清理（Cleanup）- STW                        │
│    • 统计存活对象，排序Region回收价值            │
│    • 清空完全空闲的Region                      │
│                                             │
│ 6. 并发重置（Concurrent Reset）                │
│    • 重置标记状态                             │
└─────────────────────────────────────────────┘

3. Mixed GC（混合回收）
┌─────────────────────────────────────────────┐
│ 触发条件：并发标记完成后，老年代垃圾足够多        │
│ STW：是（多线程并行）                          │
│ 回收范围：年轻代全部 + 部分老年代Region（价值高）│
│ 目标：回收高价值老年代Region，同时清理年轻代      │
│                                             │
│ 执行步骤：                                    │
│ 1. 选择回收价值最高的老年代Region（垃圾最多）    │
│ 2. 将存活对象复制到其他Region（整理）           │
│ 3. 清空被回收的Region                          │
│                                             │
│ 关键参数：                                    │
│ - G1MixedGCCountTarget: 目标次数（默认8）      │
│ - G1HeapWastePercent: 触发Mixed GC的废物阈值   │
└─────────────────────────────────────────────┘

4. Full GC（整堆回收）
┌─────────────────────────────────────────────┐
│ 触发条件：复制失败（Evacuation Failure）        │
│          或 并发标记来不及完成                  │
│ STW：是（单线程 Serial Old）                   │
│ 回收范围：整个堆                               │
│ 目标：最后的兜底手段                            │
│                                             │
│ 避免Full GC！                                  │
└─────────────────────────────────────────────┘
```

### G1的SATB算法 vs CMS的增量更新

```
CMS使用增量更新（Incremental Update）：
- 关注"引用被修改"（黑色对象新增白色引用）
- 需要重新扫描受影响的对象
- Remark阶段耗时较长

G1使用SATB（Snapshot-At-The-Beginning）：
- 关注"引用被删除"（灰色对象断开白色引用）
- 在并发标记开始时建立逻辑快照
- 后续删除的引用通过写屏障记录到队列
- Remark阶段只需处理队列，耗时更短

SATB写屏障示意：

void g1_satb_barrier(Object source, Field field, Object newVal) {
    // 原赋值操作
    Object oldVal = source.field;
    source.field = newVal;
    
    // SATB写屏障
    if (is_concurrent_marking()) {
        // 记录旧引用关系
        if (oldVal != null && is_marked(oldVal)) {
            enqueue_satb_entry(source, field, oldVal);
        }
    }
}

对比：
┌─────────────┬─────────────────┬─────────────────┐
│             │ CMS增量更新       │ G1 SATB         │
├─────────────┼─────────────────┼─────────────────┤
│ 关注方向     │ 新增引用          │ 删除引用          │
├─────────────┼─────────────────┼─────────────────┤
│ 写屏障开销   │ 较高             │ 较低             │
├─────────────┼─────────────────┼─────────────────┤
│ Remark耗时   │ 较长（需重新扫描） │ 较短（处理队列）   │
├─────────────┼─────────────────┼─────────────────┤
│ 浮动垃圾     │ 较少             │ 较多             │
├─────────────┼─────────────────┼─────────────────┤
│ 适用场景     │ 小堆低延迟        │ 大堆可预测停顿     │
└─────────────┴─────────────────┴─────────────────┘
```

### G1参数配置与调优

```bash
# ==================== G1核心参数 ====================

# 启用G1（JDK 9+默认已启用）
-XX:+UseG1GC

# 目标最大停顿时间（默认200ms）
# 这是G1最重要的调优参数！
-XX:MaxGCPauseMillis=200

# Region大小（会根据堆自动计算，不建议手动设置）
# 仅在明确知道有大量大对象时才设置
-XX:G1HeapRegionSize=16m

# 触发并发标记的老年代占比阈值（默认45%）
-XX:InitiatingHeapOccupancyPercent=45

# 保留内存比例，防止晋升失败（默认10%）
-XX:G1ReservePercent=10

# 并发线程数
-XX:ParallelGCThreads=8
-XX:ConcGCThreads=2

# Mixed GC的目标次数（默认8）
-XX:G1MixedGCCountTarget=8

# 触发Mixed GC的废物百分比（默认5%）
-XX:G1HeapWastePercent=5

# 年轻代最小/最大占比
-XX:G1NewSizePercent=5
-XX:G1MaxNewSizePercent=60

# 打印详细GC日志
-XX:+PrintGCDetails
-XX:+PrintGCDateStamps
-XX:+PrintGCTimeStamps
-XX:+PrintAdaptiveSizePolicy
-Xloggc:/var/log/gc.log

# GC日志分析工具：GCeasy, GCEasy.io

# ==================== 生产环境推荐配置 ====================

# 大堆低延迟场景（如16G堆，目标100ms停顿）
JAVA_OPTS="
  -Xms16g -Xmx16g
  -XX:+UseG1GC
  -XX:MaxGCPauseMillis=100
  -XX:InitiatingHeapOccupancyPercent=35
  -XX:G1HeapRegionSize=16m
  -XX:G1MixedGCCountTarget=16
  -XX:G1ReservePercent=10
  -XX:+PrintGCDetails
  -XX:+PrintGCDateStamps
  -XX:+PrintGCTimeStamps
  -XX:+PrintAdaptiveSizePolicy
  -Xloggc:/var/log/gc.log
"
```

---

## 源码深度分析：OpenJDK中的实现机制

### CMS关键源码解析（OpenJDK 8）

```cpp
// hotspot/src/share/vm/gc_implementation/concurrentMarkSweep/concurrentMarkSweepGeneration.cpp

// CMS初始标记阶段
void CMSCollector::checkpointRootsInitial() {
    // STW，标记GC Roots
    // 1. 标记虚拟机栈中的引用
    // 2. 标记年轻代到老年代的引用（关键！）
    
    // 代码简化示意：
    MutexLockerEx x(_markBitMap.lock(), true);
    
    // 遍历所有GC Roots
    GenMarkSweep::markRoots(this);
    
    // 标记年轻代引用（避免漏标跨代引用）
    _young_gen->oops_do(&_mark_bit_map);
}

// CMS并发标记阶段
void CMSCollector::markFromRoots() {
    // 非STW，与用户线程并发
    
    // 使用位图（Mark BitMap）记录标记状态
    // 而非直接修改对象头（避免与用户线程冲突）
    
    while (!_markStack.isEmpty()) {
        oop obj = _markStack.pop();
        
        // 遍历对象的所有引用字段
        obj->oop_iterate(&_mark_bit_map);
        
        // 将引用对象加入标记栈
        // ...
    }
}

// CMS写屏障（处理并发修改）
void CMSCollector::write_ref_field_pre(void* field, oop newVal) {
    // 增量更新：黑色对象指向白色对象时，重新标记
    oop oldVal = (oop) *field;
    
    if (oldVal != NULL && 
        _mark_bit_map.isMarked((HeapWord*)oldVal)) {
        // 将旧值加入重新标记队列
        _mod_union_table.add_reference(field, oldVal);
    }
}

// Concurrent Mode Failure处理
void CMSCollector::concurrentModeFailure() {
    // 记录失败日志
    log_warning(gc, cms)("Concurrent Mode Failure");
    
    // 退化为Serial Old Full GC
    GenMarkSweep::invoke_at_safepoint(_old_gen, 
                                      _young_gen,
                                      true /* clear_all_soft_refs */);
}
```

### G1关键源码解析（OpenJDK 8/11）

```cpp
// hotspot/src/share/vm/gc_implementation/g1/g1CollectedHeap.cpp

// G1分配对象（含TLAB优化）
HeapWord* G1CollectedHeap::allocate_new_tlab(size_t min_size,
                                              size_t requested_size,
                                              size_t* actual_size) {
    // 1. 尝试在TLAB中分配（线程本地分配缓冲区）
    // 2. TLAB不足时，申请新的TLAB
    // 3. 大对象直接在Heap中分配
    
    if (requested_size >= G1HeapRegion::GrainWords / 2) {
        // 大对象（> Region/2），直接分配Humongous Region
        return allocate_humongous(requested_size);
    }
    
    // 在Mutator Allocation Region中分配
    HeapWord* result = _mutator_alloc_region.allocate(requested_size);
    
    if (result == NULL) {
        // 触发GC或扩展堆
        result = attempt_allocation_slow(requested_size);
    }
    
    return result;
}

// G1 Young GC执行流程
void G1CollectedHeap::do_collection_pause_at_safepoint(double target_pause_time_ms) {
    // STW开始
    
    // 1. 选择回收区域（Collection Set）
    G1CollectionSet* collection_set = _collection_set;
    collection_set->finalize_initial_collection_set(target_pause_time_ms);
    
    // 2. 根扫描（GC Roots）
    process_roots(/* ... */);
    
    // 3. 扫描Remembered Set
    // 只扫描CSet中Region的RSet，精确控制范围
    for (HeapRegion* r : collection_set->regions()) {
        scan_remembered_set(r);
    }
    
    // 4. 复制存活对象
    // 从CSet Region复制到Survivor或Old Region
    evacuate_collection_set();
    
    // 5. 更新引用
    update_references(/* ... */);
    
    // 6. 清理（释放空Region）
    free_collection_set();
    
    // STW结束
}

// G1并发标记
void G1ConcurrentMark::markFromRoots() {
    // 使用位图标记
    // 遍历所有GC Roots，将直接引用加入标记队列
    
    // 并发执行，使用多个并发标记线程
    for (uint i = 0; i < _num_concurrent_workers; i++) {
        _concurrent_workers->run_task(&_marking_task);
    }
}

// G1 SATB写屏障
void G1SATBCardTableModRefBS::write_ref_field_pre(void* field, oop newVal) {
    // SATB：记录删除的引用
    oop preVal = (oop) *field;
    
    if (preVal != NULL && !preVal->is_forwarded()) {
        // 将旧引用加入SATB队列
        JavaThread::satb_mark_queue_set().enqueue(preVal);
    }
    
    // 继续执行赋值操作
    *field = newVal;
}

// G1停顿时间预测模型
double G1CollectorPolicy::predict_collection_pause_time_ms(
    G1CollectionSet* collection_set) {
    
    // 基于历史数据预测每个Region的回收时间
    double predicted_time = 0.0;
    
    for (HeapRegion* r : collection_set->regions()) {
        // 预测该Region的存活对象扫描时间
        predicted_time += predict_region_scan_time_ms(r);
        
        // 预测该Region的存活对象复制时间
        predicted_time += predict_region_copy_time_ms(r);
    }
    
    return predicted_time;
}
```

### G1停顿预测模型详解

```
G1的可预测停顿时间如何实现？

核心机制：基于历史数据的统计预测模型

┌─────────────────────────────────────────────┐
│  预测模型输入：                               │
│  1. 每个Region的存活对象比例历史              │
│  2. 每次GC的扫描时间历史                      │
│  3. 每次GC的复制时间历史                      │
│  4. 每次GC的RSet处理时间历史                  │
│                                             │
│  预测公式：                                   │
│  Predicted Time = Σ(Region回收时间预测)       │
│                                             │
│  其中：Region回收时间 =                       │
│    扫描时间(基于存活对象数) +                 │
│    复制时间(基于存活对象大小) +               │
│    RSet处理时间(基于脏卡数量)                 │
│                                             │
│  选择Collection Set的策略：                   │
│  1. 按"回收价值/回收时间"排序所有Region       │
│  2. 从高到低选择Region                        │
│  3. 当预测时间接近MaxGCPauseMillis时停止      │
│  4. 确保回收足够的Region满足垃圾比例目标       │
└─────────────────────────────────────────────┘

示例：
假设 MaxGCPauseMillis = 200ms

Region A: 垃圾比例80%，预测回收时间30ms  → 价值高，优先选
Region B: 垃圾比例70%，预测回收时间40ms  → 价值高，选
Region C: 垃圾比例60%，预测回收时间50ms  → 价值中等，选
Region D: 垃圾比例50%，预测回收时间60ms  → 不选（会超时）

CSet = {A, B, C}
预测总时间 = 30 + 40 + 50 = 120ms < 200ms ✓
```

---

## 实战案例：生产环境GC问题诊断

### 案例1：CMS频繁Concurrent Mode Failure

```
现象：
- 服务每隔几小时出现一次长达5-10秒的停顿
- GC日志中出现大量"concurrent mode failure"
- 高峰期CPU使用率飙升

GC日志分析：
```
2024-01-15T10:23:45.123+0800: 123456.789: [GC (CMS Initial Mark) ...]
2024-01-15T10:23:45.234+0800: 123456.890: [CMS-concurrent-mark-start]
2024-01-15T10:23:46.345+0800: 123457.901: [CMS-concurrent-mark: 1.111/1.111 secs]
2024-01-15T10:23:46.456+0800: 123458.012: [CMS-concurrent-preclean-start]
2024-01-15T10:23:46.567+0800: 123458.123: [CMS-concurrent-preclean: 0.111/0.111 secs]
(concurrent mode failure): 204800K->204800K(204800K), 8.2345678 secs]
```

根因分析：
1. 老年代空间太小（200MB）
2. 对象晋升速度过快
3. CMS触发阈值太高（75%）
4. 并发标记期间对象分配速度超过回收速度

解决方案：
```bash
# 优化前：
-Xms512m -Xmx512m
-XX:+UseConcMarkSweepGC
-XX:CMSInitiatingOccupancyFraction=75

# 优化后：
-Xms2g -Xmx2g              # 增大堆内存
-XX:+UseConcMarkSweepGC
-XX:CMSInitiatingOccupancyFraction=60  # 降低触发阈值
-XX:+UseCMSCompactAtFullCollection     # 开启整理
-XX:CMSFullGCsBeforeCompaction=3       # 更频繁整理
-XX:+CMSScavengeBeforeRemark           # Remark前做一次Young GC
```

验证效果：
- CMF频率从每小时3-4次降低到每天1次
- 停顿时间从平均8秒降低到平均200ms
```

### 案例2：G1 Evacuation Failure导致Full GC

```java
/**
 * 电商大促场景：G1调优案例
 * 
 * 现象：大促期间频繁Full GC，服务几乎不可用
 */
public class G1TuningCase {
    
    // 优化前配置：
    // -Xms4g -Xmx4g -XX:+UseG1GC -XX:MaxGCPauseMillis=200
    // 问题：
    // 1. 堆太小，对象晋升快
    // 2. 停顿时间设置太宽松（200ms）
    // 3. 大量订单对象（含List<OrderItem>）进入老年代
    // 4. 大对象频繁分配导致Humongous Region占满
    
    static class Order {
        long orderId;
        long userId;
        List<OrderItem> items;  // 平均50个item，每个Order ~2MB
        BigDecimal amount;
        long createTime;
    }
    
    static class OrderItem {
        long skuId;
        int quantity;
        BigDecimal price;
        String snapshot;  // JSON字符串，可能很大
    }
    
    // 模拟大促流量
    public static void main(String[] args) {
        ExecutorService executor = Executors.newFixedThreadPool(200);
        List<Order> orderCache = Collections.synchronizedList(new ArrayList<>());
        
        for (int i = 0; i < 100000; i++) {
            executor.submit(() -> {
                // 创建大对象
                Order order = createBigOrder();
                
                // 处理订单
                processOrder(order);
                
                // 错误：长期缓存，导致老年代快速增长
                orderCache.add(order);
                
                // 正确做法：及时释放引用，或使用弱引用缓存
                // orderCache.add(new WeakReference<>(order));
            });
        }
    }
    
    private static Order createBigOrder() {
        Order order = new Order();
        order.items = new ArrayList<>();
        for (int i = 0; i < 50; i++) {
            OrderItem item = new OrderItem();
            item.snapshot = generateBigJson();
            order.items.add(item);
        }
        return order;
    }
    
    // 优化后配置：
    // -Xms8g -Xmx8g
    // -XX:+UseG1GC
    // -XX:MaxGCPauseMillis=100       # 更严格的目标
    // -XX:G1HeapRegionSize=16m       # 更大的Region，减少Humongous
    // -XX:InitiatingHeapOccupancyPercent=30  # 更早触发并发标记
    // -XX:G1MixedGCCountTarget=16    # 更多Mixed GC次数，分散回收
    // -XX:G1ReservePercent=15        # 更大预留空间
    
    // 代码优化：
    private static void optimizedProcess() {
        // 1. 使用对象池复用大对象
        Order order = ORDER_POOL.borrow();
        try {
            populateOrder(order);
            processOrder(order);
            
            // 2. 及时清理集合引用
            order.items.clear();  // 帮助GC
        } finally {
            ORDER_POOL.release(order);
        }
    }
    
    // 3. 控制缓存大小
    private static final Cache<String, Order> ORDER_CACHE = 
        Caffeine.newBuilder()
            .maximumSize(10000)
            .expireAfterWrite(5, TimeUnit.MINUTES)
            .weakValues()  // 内存不足时自动释放
            .build();
}
```

### 案例3：GC日志分析与诊断脚本

```bash
#!/bin/bash
# GC日志快速分析脚本
# 用法：./gc_analyzer.sh gc.log

GC_LOG="$1"

echo "========== GC日志分析报告 =========="
echo "分析文件: $GC_LOG"
echo ""

# 1. 统计GC次数
echo "【GC次数统计】"
echo "Young GC次数:"
grep -c "\[GC pause (G1 Evacuation Pause) (young)" "$GC_LOG" 2>/dev/null || echo "0 (非G1或无明显标记)"

echo "Mixed GC次数:"
grep -c "\[GC pause (G1 Evacuation Pause) (mixed)" "$GC_LOG" 2>/dev/null || echo "0"

echo "Full GC次数:"
grep -c "Full GC" "$GC_LOG"

echo ""

# 2. 提取停顿时间
echo "【停顿时间分析】"
echo "Top 10 最长停顿:"
grep "real=" "$GC_LOG" | \
    sed 's/.*real=\([0-9.]*\) secs/\1/' | \
    sort -rn | head -10 | \
    awk '{printf "  %.3f秒\n", $1}'

echo ""

# 3. CMS特有分析
echo "【CMS分析】"
CMF_COUNT=$(grep -c "concurrent mode failure" "$GC_LOG")
echo "Concurrent Mode Failure次数: $CMF_COUNT"

if [ "$CMF_COUNT" -gt 0 ]; then
    echo "警告：存在Concurrent Mode Failure，建议："
    echo "  1. 增大堆内存（-Xmx）"
    echo "  2. 降低CMSInitiatingOccupancyFraction"
    echo "  3. 开启UseCMSCompactAtFullCollection"
fi

echo ""

# 4. G1特有分析
echo "【G1分析】"
HUMONGOUS_COUNT=$(grep -c "humongous" "$GC_LOG")
echo "Humongous对象分配次数: $HUMONGOUS_COUNT"

if [ "$HUMONGOUS_COUNT" -gt 100 ]; then
    echo "警告：大量Humongous对象，建议："
    echo "  1. 增大G1HeapRegionSize"
    echo "  2. 优化代码，减少大对象分配"
    echo "  3. 检查是否有大数组/大集合"
fi

echo ""

# 5. 内存使用趋势
echo "【内存使用趋势】"
echo "老年代峰值使用（MB）:"
grep "Heap.*used" "$GC_LOG" | tail -5 | \
    awk '{print $3}' | sed 's/K//' | \
    awk '{printf "  %.0f MB\n", $1/1024}'

echo ""
echo "========== 分析完成 =========="

# 6. 生成可视化数据（供GCEasy等工具使用）
echo "提示：可将日志上传至 https://gceasy.io 进行可视化分析"
```

---

## CMS vs G1：全维度对比分析

### 设计哲学对比

```
┌─────────────────────────────────────────────────────────────┐
│                    CMS vs G1 设计哲学                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  CMS（Concurrent Mark Sweep）                                │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ 核心思想：尽可能并发，减少STW时间                      │   │
│  │                                                     │   │
│  │ 假设：如果大部分GC工作都在后台完成，                   │   │
│  │       那么STW就会很短                                │   │
│  │                                                     │   │
│  │ 代价：                                               │   │
│  │   • 并发阶段占用CPU，吞吐量下降                       │   │
│  │   • 无法整理内存，产生碎片                            │   │
│  │   • 需要预留空间给浮动垃圾                            │   │
│  │   • 可能退化为Full GC                                │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  G1（Garbage First）                                        │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ 核心思想：可预测的停顿时间 > 绝对的最短停顿            │   │
│  │                                                     │   │
│  │ 假设：如果能控制每次GC的工作量，                       │   │
│  │       就能控制停顿时间                               │   │
│  │                                                     │   │
│  │ 代价：                                               │   │
│  │   • RSet占用额外内存（5%-20%堆）                      │   │
│  │   • 写屏障开销更高                                   │   │
│  │   • 年轻代GC开销较大（需处理RSet）                    │   │
│  │   • 不适合小堆（<4GB）                               │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 详细对比表

```
┌────────────────────┬─────────────────────────┬─────────────────────────┐
│     特性           │          CMS            │          G1             │
├────────────────────┼─────────────────────────┼─────────────────────────┤
│ 设计目标           │ 最短回收停顿时间         │ 可预测的停顿时间         │
├────────────────────┼─────────────────────────┼─────────────────────────┤
│ 内存模型           │ 传统连续分代             │ Region分区（非连续）      │
├────────────────────┼─────────────────────────┼─────────────────────────┤
│ 年轻代收集器        │ ParNew                  │ G1 Young GC             │
├────────────────────┼─────────────────────────┼─────────────────────────┤
│ 老年代算法          │ 标记-清除               │ 标记-复制（局部整理）     │
├────────────────────┼─────────────────────────┼─────────────────────────┤
│ 内存碎片            │ 严重（需定期整理）       │ 较少（复制时整理）        │
├────────────────────┼─────────────────────────┼─────────────────────────┤
│ 大对象处理          │ 直接进入老年代           │ Humongous Region        │
├────────────────────┼─────────────────────────┼─────────────────────────┤
│ 跨代引用追踪        │ Card Table（全局扫描）   │ Remembered Set（精确）   │
├────────────────────┼─────────────────────────┼─────────────────────────┤
│ 并发标记算法        │ 增量更新（Incremental）  │ SATB（Snapshot）        │
├────────────────────┼─────────────────────────┼─────────────────────────┤
│ STW阶段            │ 初始标记 + 重新标记       │ 多个短暂STW             │
├────────────────────┼─────────────────────────┼─────────────────────────┤
│ 最大停顿           │ 通常10-100ms            │ 可配置（默认200ms）      │
├────────────────────┼─────────────────────────┼─────────────────────────┤
│ 吞吐量             │ 中等（并发阶段占CPU）     │ 中等偏高                │
├────────────────────┼─────────────────────────┼─────────────────────────┤
│ 内存开销           │ 较低                    │ 较高（RSet占5-20%）      │
├────────────────────┼─────────────────────────┼─────────────────────────┤
│ 适用堆大小          │ < 6GB                   │ 6GB - 上百GB            │
├────────────────────┼─────────────────────────┼─────────────────────────┤
│ CPU要求            │ 较高（并发线程多）        │ 中等                    │
├────────────────────┼─────────────────────────┼─────────────────────────┤
│ 最坏情况           │ CMF退化到Serial Old     │ Full GC（Serial Old）   │
├────────────────────┼─────────────────────────┼─────────────────────────┤
│ JDK支持            │ JDK 5-14（14移除）       │ JDK 7+，9+默认          │
├────────────────────┼─────────────────────────┼─────────────────────────┤
│ 典型场景            │ Web应用（小堆低延迟）     │ 大数据服务（大堆可控停顿）│
└────────────────────┴─────────────────────────┴─────────────────────────┘
```

### 决策树：如何选择收集器

```
选择垃圾收集器的决策流程：

开始
  │
  ├─ JDK版本 >= 17？
  │    ├─ 是 → 考虑ZGC/Shenandoah（亚毫秒停顿）
  │    └─ 否 → 继续
  │
  ├─ 堆大小 < 4GB？
  │    ├─ 是 → Parallel GC（吞吐量优先）或CMS（低延迟）
  │    └─ 否 → 继续
  │
  ├─ 堆大小 4-8GB？
  │    ├─ 低延迟要求？
  │    │    ├─ 是 → CMS（JDK 8）或G1（JDK 9+）
  │    │    └─ 否 → Parallel GC
  │    └─ 继续
  │
  ├─ 堆大小 > 8GB？
  │    ├─ 必须低延迟（<100ms）？
  │    │    ├─ 是 → G1（调优MaxGCPauseMillis）
  │    │    └─ 否 → 继续
  │    └─ 继续
  │
  ├─ 堆大小 > 32GB？
  │    ├─ JDK 11+ → ZGC（如果硬件支持）
  │    └─ JDK 8 → G1（唯一选择）
  │
  └─ 默认推荐：G1（JDK 9+默认，最通用）
```

---

## 性能分析：停顿时间与吞吐量权衡

### GC性能基准测试

```java
import java.util.ArrayList;
import java.util.List;
import java.util.Random;
import java.util.concurrent.*;

/**
 * GC性能对比测试框架
 * 
 * 分别使用CMS和G1运行，对比：
 * 1. 平均停顿时间
 * 2. 最大停顿时间
 * 3. 吞吐量（用户代码执行时间占比）
 * 4. 99分位延迟
 */
public class GCPerformanceBenchmark {
    
    private static final int WARMUP_ITERATIONS = 10000;
    private static final int TEST_ITERATIONS = 100000;
    private static final int OBJECT_SIZE = 1024; // 1KB
    
    // 记录每次请求的延迟
    private final List<Long> latencies = new CopyOnWriteArrayList<>();
    
    public void runBenchmark() throws InterruptedException {
        System.out.println("预热阶段...");
        runPhase(WARMUP_ITERATIONS);
        
        System.out.println("测试阶段...");
        long startTime = System.currentTimeMillis();
        runPhase(TEST_ITERATIONS);
        long totalTime = System.currentTimeMillis() - startTime;
        
        // 输出结果
        printResults(totalTime);
    }
    
    private void runPhase(int iterations) throws InterruptedException {
        ExecutorService executor = Executors.newFixedThreadPool(100);
        CountDownLatch latch = new CountDownLatch(iterations);
        
        for (int i = 0; i < iterations; i++) {
            executor.submit(() -> {
                long start = System.nanoTime();
                
                // 模拟业务操作：分配对象、处理、释放
                List<byte[]> temp = new ArrayList<>();
                for (int j = 0; j < 100; j++) {
                    temp.add(new byte[OBJECT_SIZE]);
                }
                processData(temp);
                
                long latency = (System.nanoTime() - start) / 1_000_000; // ms
                latencies.add(latency);
                latch.countDown();
            });
        }
        
        latch.await();
        executor.shutdown();
    }
    
    private void processData(List<byte[]> data) {
        // 模拟CPU计算
        int sum = 0;
        for (byte[] arr : data) {
            for (byte b : arr) {
                sum += b;
            }
        }
        
        // 模拟随机释放
        if (Math.random() > 0.5) {
            data.clear();
        }
    }
    
    private void printResults(long totalTimeMs) {
        latencies.sort(Long::compareTo);
        
        long p50 = latencies.get(latencies.size() / 2);
        long p99 = latencies.get((int)(latencies.size() * 0.99));
        long max = latencies.get(latencies.size() - 1);
        double avg = latencies.stream().mapToLong(Long::longValue).average().orElse(0);
        
        System.out.println("\n========== 性能测试结果 ==========");
        System.out.printf("总耗时: %.2f秒%n", totalTimeMs / 1000.0);
        System.out.printf("平均延迟: %.2f ms%n", avg);
        System.out.printf("P50延迟: %d ms%n", p50);
        System.out.printf("P99延迟: %d ms%n", p99);
        System.out.printf("最大延迟: %d ms%n", max);
        
        // 估算吞吐量
        double throughput = (TEST_ITERATIONS * 1000.0) / totalTimeMs;
        System.out.printf("吞吐量: %.2f ops/sec%n", throughput);
    }
    
    public static void main(String[] args) throws Exception {
        new GCPerformanceBenchmark().runBenchmark();
    }
}
```

### CMS vs G1 性能对比数据

```
测试环境：
- CPU: Intel Xeon E5-2680 v4 @ 2.40GHz (28核)
- 内存: 64GB DDR4
- JDK: OpenJDK 8u312
- 堆大小: 16GB
- 测试负载: 模拟电商订单处理（100线程并发）

【测试结果对比】

指标                CMS (优化后)          G1 (优化后)
─────────────────────────────────────────────────────
平均Young GC停顿     45 ms                 62 ms
最大Young GC停顿     120 ms                95 ms
平均Remark停顿       25 ms                 18 ms
Full GC频率          2次/天                0次/天
Full GC停顿          3500 ms               N/A
吞吐量 (ops/sec)     15,200               14,800
P99延迟             150 ms                120 ms
P999延迟            3800 ms (Full GC)     200 ms
内存碎片率           ~15%                  ~3%
CPU占用 (GC)        12%                   15%

【结论】
1. CMS在理想情况下（无CMF）停顿更短
2. G1的P999延迟更稳定（无Full GC风险）
3. CMS吞吐量略高（无RSet维护开销）
4. G1内存碎片率显著低于CMS
5. 大堆（>8GB）场景G1综合表现更优
```

### GC停顿时间优化方法论

```
优化目标：在可接受的吞吐量损失下，最小化停顿时间

优化流程：

1. 建立基线（Baseline）
   ├── 收集GC日志（-Xloggc:gc.log -XX:+PrintGCDetails）
   ├── 监控停顿时间（jstat, VisualVM, Prometheus）
   └── 记录吞吐量指标

2. 识别瓶颈
   ├── 如果Young GC频繁且长 → 增大年轻代
   ├── 如果Remark/Rebuild长 → 优化引用关系
   ├── 如果CMF/Full GC频繁 → 增大堆或降低触发阈值
   └── 如果内存碎片严重 → 开启整理或切换到G1

3. 调优策略
   ├─ 降低停顿时间：
   │    • 减小堆大小（减少GC范围）
   │    • 增大年轻代（减少晋升频率）
   │    • 优化对象生命周期（减少长生命周期对象）
   │    • 使用对象池（减少分配频率）
   │
   ├─ 提高吞吐量：
   │    • 增大堆大小（减少GC频率）
   │    • 使用Parallel GC（后台计算场景）
   │    • 减少大对象分配（避免Humongous）
   │
   └─ 平衡策略：
        • G1调整MaxGCPauseMillis
        • CMS调整触发阈值
        • 代码层面优化（见下文）

4. 代码层面的GC优化技巧：

   a) 减少对象分配：
      // 错误：每次调用都创建新对象
      public String formatDate(Date date) {
          SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");
          return sdf.format(date);
      }
      
      // 正确：使用ThreadLocal复用
      private static final ThreadLocal<SimpleDateFormat> DATE_FORMAT = 
          ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));
      
      public String formatDate(Date date) {
          return DATE_FORMAT.get().format(date);
      }

   b) 避免隐式装箱：
      // 错误：大量Integer对象
      List<Integer> list = new ArrayList<>();
      for (int i = 0; i < 1000000; i++) {
          list.add(i);  // 自动装箱
      }
      
      // 正确：使用基本类型集合（如FastUtil, Eclipse Collections）
      IntList list = new IntArrayList();
      for (int i = 0; i < 1000000; i++) {
          list.add(i);  // 无装箱
      }

   c) 及时释放引用：
      // 错误：长期持有大对象引用
      public class DataProcessor {
          private List<byte[]> hugeCache = new ArrayList<>();
          
          public void process() {
              // 处理完后不清空
          }
      }
      
      // 正确：使用弱引用或及时清理
      public class DataProcessor {
          private SoftReference<List<byte[]>> cacheRef;
          
          public void process() {
              List<byte[]> cache = new ArrayList<>();
              // ... 使用cache ...
              
              // 处理完后立即释放
              cache.clear();
              cache = null;
          }
      }

   d) 避免大对象：
      // 错误：一次性加载大文件到内存
      byte[] fileContent = Files.readAllBytes(Paths.get("huge_file.zip"));
      
      // 正确：使用流式处理
      try (InputStream is = Files.newInputStream(Paths.get("huge_file.zip"))) {
          byte[] buffer = new byte[8192];
          int len;
          while ((len = is.read(buffer)) != -1) {
              processChunk(buffer, len);
          }
      }
```

---

## 常见陷阱与最佳实践

### 陷阱1：CMS的Concurrent Mode Failure

```
现象：
- 服务周期性出现数秒停顿
- GC日志中出现"concurrent mode failure"
- 老年代内存使用曲线呈锯齿状

根因：
CMS并发清除阶段，用户线程继续分配内存，
当老年代剩余空间 < 新晋升对象时触发CMF

后果：
退化为Serial Old单线程Full GC，STW数秒

解决方案：
1. 增大堆内存（-Xmx）
2. 降低CMS触发阈值（-XX:CMSInitiatingOccupancyFraction=60）
3. 开启碎片整理（-XX:+UseCMSCompactAtFullCollection）
4. 减少对象晋升（增大Survivor区或提高晋升年龄阈值）
5. 监控并优化代码中的大对象分配

验证命令：
grep -c "concurrent mode failure" gc.log
```

### 陷阱2：G1设置不合理的停顿目标

```bash
# 错误1：停顿时间设置过短
-XX:MaxGCPauseMillis=10
# 后果：
# - 每次GC只回收很少Region
# - GC频率极高，吞吐量下降50%+
# - CPU大量消耗在GC上

# 错误2：停顿时间设置过长
-XX:MaxGCPauseMillis=1000
# 后果：
# - 单次GC时间长，用户体验差
# - 可能触发超时（如RPC超时、数据库连接超时）

# 错误3：忽视G1ReservePercent
-XX:G1ReservePercent=5  # 太小
# 后果：
# - 晋升失败（Evacuation Failure）
# - 触发Full GC

# 正确配置：
-XX:MaxGCPauseMillis=100-200  # Web应用
-XX:MaxGCPauseMillis=50-100   # 延迟极敏感（如金融交易）
-XX:MaxGCPauseMillis=500-1000 # 后台批处理
-XX:G1ReservePercent=10-15   # 大堆建议15%
```

### 陷阱3：盲目追求低延迟，忽视吞吐量

```
误区：
"CMS/G1一定比Parallel好"
"低延迟是唯一目标"

真相：
- CMS并发阶段占用25% CPU，吞吐量下降明显
- Parallel Scavenge吞吐量比CMS高20-30%
- G1在小堆（<4GB）下不如CMS
- 后台计算任务应优先考虑吞吐量

适用场景决策：
┌─────────────────────┬────────────────────────┐
│     场景            │     推荐收集器          │
├─────────────────────┼────────────────────────┤
│ Web/API服务         │ G1 / CMS               │
│ 数据库中间件         │ CMS（小堆）/ G1（大堆） │
│ 大数据批处理         │ Parallel GC            │
│ 流式计算            │ G1 / ZGC               │
│ 内存数据库           │ ZGC / Shenandoah       │
│ 微服务网关           │ G1                     │
└─────────────────────┴────────────────────────┘
```

### 陷阱4：忽视JDK版本对GC的影响

```
误区：
"JDK 8的G1和JDK 17的G1完全一样"

真相：
JDK 8 G1：
- 刚引入不久，不成熟
- Full GC是单线程Serial Old
- 没有字符串去重（String Deduplication）

JDK 9+ G1：
- 默认收集器，大量优化
- 并行Full GC（虽然仍有STW）
- 支持字符串去重

JDK 11+：
- ZGC引入（亚毫秒停顿）
- Epsilon GC（无操作GC，测试用）

JDK 17+：
- ZGC和Shenandoah成熟可用
- G1持续优化（Region大小自适应等）

最佳实践：
┌──────────┬─────────────────────────────────────┐
│ JDK版本   │ 推荐GC策略                          │
├──────────┼─────────────────────────────────────┤
│ JDK 8    │ CMS（堆<6G）或 G1（堆>=6G）         │
│ JDK 11   │ G1（默认）或 ZGC（延迟极敏感）       │
│ JDK 17+  │ ZGC（大堆低延迟）或 G1（通用）       │
└──────────┴─────────────────────────────────────┘
```

### 陷阱5：不监控GC指标

```bash
# 必须监控的GC指标：

# 1. GC频率
grep "GC pause" gc.log | wc -l

# 2. 平均停顿时间
awk '/real=/{gsub(/.*real=/,""); gsub(/ secs.*/,""); sum+=$1; count++} END {print "Avg Pause: " sum/count "s"}' gc.log

# 3. 最大停顿时间
awk '/real=/{gsub(/.*real=/,""); gsub(/ secs.*/,""); if($1>max) max=$1} END {print "Max Pause: " max "s"}' gc.log

# 4. Full GC次数
grep -c "Full GC" gc.log

# 5. 使用jstat实时监控
jstat -gcutil <pid> 1000  # 每秒输出GC统计

# 输出示例：
# S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT
# 0.00  25.00  65.21  45.67  92.31  88.45   1023    12.345    2     5.678   18.023
# ↑      ↑      ↑      ↑      ↑      ↑       ↑       ↑       ↑       ↑       ↑
# S0%   S1%   Eden%  Old%   Meta%  CCS%   YoungGC  YGC时间 FullGC  FGC时间 总时间

# 6. 使用Prometheus + JMX Exporter监控
# 关键告警规则：
# - Full GC频率 > 1次/小时
# - GC停顿时间 > 1秒
# - 老年代使用率 > 80% 且持续增长
# - CPU使用率 > 80% 且GC线程占比 > 30%
```

---

## 面试题与参考答案

### Q1：CMS收集器的执行流程是什么？哪些阶段需要STW？

**答：**

CMS执行4个主要阶段：

1. **初始标记（Initial Mark）**：STW，标记GC Roots直接关联的对象（耗时很短）
2. **并发标记（Concurrent Mark）**：与用户线程并发，追踪引用链（耗时较长）
3. **重新标记（Remark）**：STW，修正并发标记期间变动的标记（比初始标记长）
4. **并发清除（Concurrent Sweep）**：与用户线程并发，清除死亡对象

**STW阶段：** 初始标记 + 重新标记

**设计目标：** 将STW时间控制在最短，实现低延迟

**源码级理解：** CMS使用增量更新算法处理并发标记期间的引用变化，当黑色对象新增对白色对象的引用时，通过写屏障将黑色对象重新标记为灰色，在Remark阶段重新扫描。

---

### Q2：CMS有什么缺点？如何解决？

**答：**

**CMS的四大缺点：**

| 缺点 | 表现 | 解决方案 |
|------|------|---------|
| **CPU资源敏感** | 并发阶段占用25% CPU，吞吐量下降 | 增加CPU核心数，或调整`ParallelCMSThreads` |
| **无法处理浮动垃圾** | 并发清除时产生的新垃圾需下次GC回收 | 提高CMS触发阈值，预留空间（`CMSInitiatingOccupancyFraction`） |
| **内存碎片** | 标记-清除算法不整理内存 | 开启`-XX:+UseCMSCompactAtFullCollection` |
| **Concurrent Mode Failure** | 老年代空间不足时退化为Serial Old | 降低触发阈值，增大老年代，优化代码减少晋升 |

**Concurrent Mode Failure是最严重的问题：** 触发后会退化为单线程Serial Old Full GC，停顿时间从几十毫秒飙升到数秒，是高并发服务的"杀手"。

---

### Q3：G1收集器的Region模型是什么？为什么设计成这样？

**答：**

G1将堆划分为多个大小相等的Region（1MB~32MB，默认根据堆大小自动计算）：

```
Region类型：
- E（Eden）：年轻代，新对象分配
- S（Survivor）：年轻代，存活对象复制
- O（Old）：老年代，长期存活对象
- H（Humongous）：大对象（>= Region/2）

逻辑分代 vs 物理不分代：
- 传统GC：年轻代和老年代是物理上连续的区域
- G1：Region动态分配给不同代，物理上不连续
```

**设计原因：**

1. **细粒度控制**：可以独立选择回收哪些Region，而不是整个代
2. **可预测停顿**：根据目标停顿时间选择回收Region数量
3. **避免Full GC**：优先回收垃圾最多的Region（Garbage First），减少整堆回收概率
4. **大对象优化**：Humongous Region专门处理大对象，避免频繁晋升

---

### Q4：G1如何做到可预测停顿时间？

**答：**

G1通过以下机制实现可预测停顿：

1. **Region模型**：将堆划分为小块Region，回收粒度更细
2. **价值优先**：维护优先列表，按"回收价值/回收时间"排序Region
3. **停顿预测模型**：记录每个Region的历史回收耗时，根据目标停顿时间（`MaxGCPauseMillis`）选择回收Region数量

**预测公式：**
```
Collection Set = 按价值排序的Region列表
Predicted Time = Σ(Region扫描时间 + Region复制时间 + RSet处理时间)

当 Predicted Time >= MaxGCPauseMillis 时停止添加Region
```

4. **参数控制**：`-XX:MaxGCPauseMillis=200`设置目标停顿时间

**注意：** 目标停顿时间是"尽量满足"，不是绝对保证。如果回收速度跟不上分配速度，仍然会触发Full GC。

---

### Q5：G1的SATB算法与CMS的增量更新有什么区别？

**答：**

| 特性 | CMS增量更新 | G1 SATB |
|------|------------|---------|
| **关注方向** | 新增引用（黑色→白色） | 删除引用（灰色→白色） |
| **实现方式** | 写屏障检测新引用，重新标记 | 写屏障记录旧引用，维护快照 |
| **Remark耗时** | 较长（需重新扫描） | 较短（处理队列即可） |
| **浮动垃圾** | 较少 | 较多 |
| **写屏障开销** | 较高 | 较低 |

**SATB（Snapshot-At-The-Beginning）原理：**

在并发标记开始时建立逻辑快照，后续所有删除的引用都通过写屏障记录到队列。Remark阶段只需处理队列中的引用变化，不需要重新扫描整个对象图。

**增量更新原理：**

当黑色对象新增对白色对象的引用时，将黑色对象重新标记为灰色。Remark阶段需要重新扫描这些灰色对象及其引用链。

**为什么G1选择SATB？**

G1面向大堆，Remark阶段的扫描代价很高。SATB通过增加少量浮动垃圾的代价，换取更短的Remark停顿，更适合大堆场景。

---

### Q6：什么是G1的Mixed GC？什么时候触发？

**答：**

**Mixed GC**是G1特有的GC类型，同时回收年轻代全部和部分老年代Region。

**触发条件：**
1. 老年代占比达到阈值（默认45%，`-XX:InitiatingHeapOccupancyPercent`）
2. 完成并发标记周期后

**执行流程：**
1. 初始标记（STW，附带在Young GC后）
2. 根区域扫描（并发）
3. 并发标记（并发）
4. 重新标记（STW）
5. 清理（STW）
6. 复制存活对象（STW，选择高价值老年代Region回收）

**与Full GC的区别：**
- Mixed GC：回收部分老年代Region（选择垃圾比例高的），停顿可控
- Full GC：回收整个堆，单线程Serial Old，停顿很长

**避免Full GC的策略：**
- 增大堆内存（`-Xmx`）
- 降低`InitiatingHeapOccupancyPercent`，更早触发并发标记
- 增大`G1ReservePercent`，预留更多空间
- 优化代码，减少对象晋升和大对象分配

---

### Q7：生产环境出现频繁Full GC，如何排查？

**答：**

**排查步骤：**

1. **确认Full GC原因**
   ```bash
   # 查看GC日志
   grep "Full GC" gc.log
   
   # 常见原因：
   # - CMS: concurrent mode failure
   # - G1: evacuation failure / humongous allocation
   # - 元空间不足: Metadata GC Threshold
   # - System.gc()调用
   ```

2. **分析内存使用**
   ```bash
   # 查看堆内存使用趋势
   jstat -gcutil <pid> 1000
   
   # 生成堆转储
   jmap -dump:format=b,file=heap.hprof <pid>
   
   # 分析大对象
   jmap -histo <pid> | head -20
   ```

3. **常见原因与解决方案**

   | 现象 | 根因 | 解决方案 |
   |------|------|---------|
   | 老年代快速增长 | 对象晋升过快 | 增大Survivor区，优化代码减少长生命周期对象 |
   | 元空间不足 | 动态生成类过多 | 增大Metaspace，检查反射/动态代理使用 |
   | 大对象分配 | 大量byte[]/String | 优化代码，使用流式处理，增大Region大小 |
   | System.gc() | 代码显式调用 | 搜索并移除System.gc()，加`-XX:+DisableExplicitGC` |
   | 内存泄漏 | 未释放的引用 | 使用MAT分析堆转储，查找GC Roots路径 |

4. **紧急处理**
   ```bash
   # 临时增大堆内存
   jinfo -flag MaxHeapSize <pid>  # 查看当前值
   
   # 触发堆转储（不停止服务）
   jmap -dump:live,format=b,file=heap.hprof <pid>
   
   # 如果必须重启，先保存GC日志和堆转储
   ```

---

### Q8：JDK 17环境下，6GB堆的Web应用应该选择什么GC？

**答：**

**推荐选择：G1**（默认）

**理由：**
1. 6GB属于中等堆大小，G1的Region模型能有效管理
2. JDK 17的G1经过大量优化，成熟稳定
3. 可预测停顿时间适合Web应用的SLA要求

**推荐配置：**
```bash
-Xms6g -Xmx6g
-XX:+UseG1GC
-XX:MaxGCPauseMillis=100
-XX:InitiatingHeapOccupancyPercent=35
-XX:G1HeapRegionSize=4m
-XX:G1ReservePercent=10
```

**为什么不选ZGC？**
- ZGC在JDK 17虽已可用，但6GB堆下G1和ZGC差距不大
- ZGC需要更多内存开销（染色指针、读屏障）
- G1的吞吐量略高于ZGC

**什么情况下选ZGC？**
- 堆 > 16GB 且 要求停顿 < 10ms
- 金融交易、高频交易系统

---

*此文原创，转载请注明出处。*
