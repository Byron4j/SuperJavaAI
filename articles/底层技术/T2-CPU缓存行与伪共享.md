# CPU 缓存行与伪共享：让多线程程序慢 10 倍的隐形杀手

**文章标签：** #CPU缓存 #缓存行 #伪共享 #多线程 #内存屏障 #并发性能 #Disruptor #volatile

## 目录

- [引言：两个 volatile 变量让吞吐量跌了 8 倍](#引言两个-volatile-变量让吞吐量跌了-8-倍)
- [CPU 缓存架构：从 L1 到主存的速度阶梯](#cpu-缓存架构从-l1-到主存的速度阶梯)
- [缓存行：CPU 读写的最小单元](#缓存行cpu-读写的最小单元)
- [MESI 协议：缓存一致性如何拖慢你的程序](#mesi-协议缓存一致性如何拖慢你的程序)
- [伪共享：无竞争的变量为何互相拖累](#伪共享无竞争的变量为何互相拖累)
- [火眼金睛：如何用工具发现伪共享](#火眼金睛如何用工具发现伪共享)
- [解决方案：从填充到注解](#解决方案从填充到注解)
- [JMH 基准测试：消除伪共享前后的天壤之别](#jmh-基准测试消除伪共享前后的天壤之别)
- [Disruptor 的缓存行填充艺术](#disruptor-的缓存行填充艺术)
- [常见陷阱与最佳实践](#常见陷阱与最佳实践)
- [面试题与参考答案](#面试题与参考答案)

---

## 引言：两个 volatile 变量让吞吐量跌了 8 倍

想象这样一个场景：你写了一个计数器，两个线程各更新自己的 volatile 变量，互不干扰。理论上应该线性扩展——两个线程应该翻倍吞吐量。实际呢？

```java
// 惊悚代码：两个线程各写各的 volatile，但它们在同一个缓存行上
public class FalseSharingHell {
    volatile long x;  // 线程 1 频繁写
    volatile long y;  // 线程 2 频繁写
}
// 线程 1 写 x → 导致 CPU0 的缓存行失效 → 线程 2 读 y 必须从内存重新加载
// 线程 2 写 y → 导致 CPU1 的缓存行失效 → 线程 1 读 x 必须从内存重新加载
// 结果：两个"无竞争"的变量疯狂互相失效，吞吐量暴跌
```

```
                    伪共享的破坏力（示意图）
 ┌─────────────────────────────────────────────────────────────┐
 │                                                             │
 │    缓存行（64 字节）                                          │
 │   ┌──────────────────────────────────────────────────┐      │
 │   │  x(8B) │  y(8B) │ ...未使用...                    │      │
 │   └──────────────────────────────────────────────────┘      │
 │          ▲          ▲                                        │
 │          │          │                                        │
 │      Core 0      Core 1                                      │
 │     写 x++       写 y++                                       │
 │          │          │                                        │
 │          └──────────┘                                        │
 │         互相让对方缓存行失效！                                 │
 │         每次写入 ≈ 一次主存访问延迟（~100ns）                  │
 │         而不是 L1 缓存延迟（~1ns）                            │
 │                                                             │
 │   性能损失：100x（100ns / 1ns）                               │
 └─────────────────────────────────────────────────────────────┘
```

**核心洞察**：伪共享是并发性能的隐形杀手——两个线程在逻辑上毫无竞争，却在硬件层面疯狂冲突。你从代码上看不出任何问题，只有从 CPU 缓存的角度才能理解。

---

## CPU 缓存架构：从 L1 到主存的速度阶梯

### 2.1 现代 CPU 的存储金字塔

```
                        访问延迟（约数）
 ┌────────────────────────────────────────────┐
 │                                            │
 │  ┌──────┐  L1 Cache    ~1 ns  (4 cycles)   │  ← 寄存器速度级
 │  │  L1  │  32-64 KB, 私有                  │
 │  ├──────┤  L2 Cache    ~3 ns  (12 cycles)  │  ← 各核心私有
 │  │  L2  │  256-512 KB, 私有                │
 │  ├──────┤  L3 Cache    ~12 ns (40 cycles)  │  ← 所有核心共享
 │  │  L3  │  8-64 MB, 共享                   │
 │  ├──────┤  DRAM 主存   ~100 ns (300+ cycles)│  ← 内存
 │  │ RAM  │  16-256 GB                       │
 │  ├──────┤  SSD/NVMe    ~10-100 μs          │  ← 持久化
 │  │ SSD  │                                   │
 │  └──────┘                                   │
 │                                            │
 │  L1 命中：1 个周期                            │
 │  主存访问：300+ 个周期                         │
 │  差距：300 倍+                               │
 └────────────────────────────────────────────┘
```

### 2.2 现代 CPU 的缓存拓扑

```
┌─────────────────────────────────────────────────────────────┐
│                    Intel Core i9 典型缓存拓扑                  │
│                                                             │
│    Core 0          Core 1          Core 2          Core 3   │
│  ┌────────┐      ┌────────┐      ┌────────┐      ┌────────┐ │
│  │ L1-I  │      │ L1-I  │      │ L1-I  │      │ L1-I  │     │
│  │ 32KB   │      │ 32KB   │      │ 32KB   │      │ 32KB   │     │
│  │ L1-D   │      │ L1-D   │      │ L1-D   │      │ L1-D   │     │
│  │ 48KB   │      │ 48KB   │      │ 48KB   │      │ 48KB   │     │
│  │ L2 2MB │      │ L2 2MB │      │ L2 2MB │      │ L2 2MB │     │
│  └───┬────┘      └───┬────┘      └───┬────┘      └───┬────┘     │
│      │               │               │               │         │
│      └───────────────┴───────┬───────┴───────────────┘         │
│                              │                                 │
│                       ┌──────┴──────┐                          │
│                       │  L3 (LLC)   │  共享 36MB                │
│                       │   Ring Bus  │                          │
│                       └──────┬──────┘                          │
│                              │                                 │
│                       ┌──────┴──────┐                          │
│                       │  内存控制器   │                          │
│                       └──────┬──────┘                          │
│                              │                                 │
│                       ┌──────┴──────┐                          │
│                       │    DRAM     │                          │
│                       └─────────────┘                          │
└─────────────────────────────────────────────────────────────┘

关键认知：L1/L2 是核心私有的，修改私有缓存中的数据
不会自动让其他核心看到——这就是缓存一致性问题
```

---

## 缓存行：CPU 读写的最小单元

### 3.1 缓存行不是你的变量

CPU **从不以字节为单位**读写缓存。它一次读写一个完整的**缓存行**（Cache Line），现代 x86-64 架构下通常为 **64 字节**。

```
┌─────────────────────────────────────────────────────────────┐
│                    缓存行结构（64 字节）                       │
│                                                             │
│  ┌──────┬──────────────────────────────────────────────┐    │
│  │ Tag  │             Data (64 bytes)                  │    │
│  │ 6 B  │  ┌────┬────┬────┬────┬────┬─── ... ──┬────┐ │    │
│  │      │  │B0  │B1  │B2  │B3  │B4  │          │B63 │ │    │
│  │      │  └────┴────┴────┴────┴────┴─── ... ──┴────┘ │    │
│  └──────┴──────────────────────────────────────────────┘    │
│                                                             │
│  你的变量：                                                   │
│  long x = 0;  // 8 字节，仅占缓存行的 1/8                    │
│  long y = 0;  // 8 字节，可能紧挨着 x                         │
│                                                             │
│  当 CPU 读取 x 时，整行 64 字节都被加载到 L1 缓存              │
│  当 CPU 修改 x 时，整行 64 字节被标记为 dirty                │
│                                                             │
│  x 和 y 如果在同一缓存行 → 伪共享！                           │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 缓存行的物理布局

```java
// 危险布局：x 和 y 大概率在同一缓存行
public class DangerousLayout {
    long x;   // offset 0,  占用 8 字节
    long y;   // offset 8,  占用 8 字节
    // x 和 y 共计 16 字节 < 64 字节 → 在同一缓存行！
}

// Java 对象内存布局（64 位 JVM，压缩指针开启）：
// offset 0:  Mark Word (8 字节)
// offset 8:  Klass Pointer (4 字节，压缩)
// offset 12: padding (4 字节对齐)
// offset 16: x (8 字节)
// offset 24: y (8 字节)
// 总计 32 字节 → 完全在一个 64 字节缓存行内
```

---

## MESI 协议：缓存一致性如何拖慢你的程序

### 4.1 缓存行的四种状态

```
┌─────────────────────────────────────────────────────────────┐
│                     MESI 状态转换                             │
│                                                             │
│           ┌──────────┐                                      │
│     ┌───►│ Modified │◄──┐    本地读/写/监听                   │
│     │    │ (脏数据)  │   │                                   │
│     │    └────┬─────┘   │                                   │
│     │         │          │                                   │
│     │    snoop│     snoop│                                   │
│     │    读请求│     读请求│                                   │
│     │         │          │                                   │
│     │    ┌────▼─────┐   │                                   │
│     │    │ Exclusive │   │                                   │
│     │    │ (干净唯一) │   │                                   │
│     │    └────┬─────┘   │                                   │
│     │         │          │                                   │
│     │   snoop │    snoop │                                   │
│     │   读请求 │    读请求│                                   │
│     │         │          │                                   │
│     │    ┌────▼─────┐   │                                   │
│     └────┤  Shared   │───┘                                   │
│          │ (干净共享) │                                       │
│          └──────────┘                                       │
│                                                             │
│  Modified: 只有当前核心有，已修改，与主存不一致                  │
│  Exclusive: 只有当前核心有，干净，与主存一致                    │
│  Shared:   多个核心共享，干净                                  │
│  Invalid:  无效，不能使用                                     │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 一次写入导致的"风暴"

```
场景：Core 0 写变量 x，Core 1 读变量 y，但 x 和 y 在同一缓存行

时间轴 ──────────────────────────────────────────────►

  Core 0 (写 x)              Core 1 (读 y)           总线事务
  ─────────────              ─────────────           ────────
                                                     缓存行 S 态
                            读 y (L1 命中, S 态)     ← 1ns

  写 x (需要 E/M 态)         ← 窥探到 Core 0 要写     RFO 请求
                             缓存行失效！ → I 态       失效消息

  x 写入完毕 (M 态)                                   写入回执
  ≈ 几十 ns = L1 变主存延迟

                            读 y (I 态, MISS!)       Read 请求
                             等待 Core 0 回写         探听消息

                            收到数据 (S 态)
                            ≈ 几十 ns

  结果：每次"无竞争"的写入都走了主存延迟路径！
```

---

## 伪共享：无竞争的变量为何互相拖累

### 5.1 经典灾难场景

```java
import java.util.concurrent.atomic.AtomicLong;

/**
 * 伪共享的完美演示——两个计数器在逻辑上完全独立，却互相拖累
 */
public class FalseSharingDisaster {
    // ⚠️ 灾难：这两个 AtomicLong 几乎肯定在同一缓存行
    private AtomicLong counter1 = new AtomicLong(0);
    private AtomicLong counter2 = new AtomicLong(0);

    public void increment1() {
        counter1.incrementAndGet();  // 写 counter1→使 counter2 所在缓存行失效
    }

    public void increment2() {
        counter2.incrementAndGet();  // 写 counter2→使 counter1 所在缓存行失效
    }
    // 线程 A 只调 increment1，线程 B 只调 increment2
    // 逻辑上零竞争，但 CPU 层面每分钟数百万次缓存行失效
}
```

### 5.2 伪共享的三大特征

```
┌─────────────────────────────────────────────────────────┐
│               伪共享三要素（三者缺一不可）                   │
│                                                         │
│  1. 邻近存储 ── 两个变量的内存地址落在同一缓存行            │
│                                                         │
│  2. 独立写入 ── 不同 CPU 核心对不同变量进行写入操作        │
│     （如果只是读取，不会触发伪共享）                       │
│                                                         │
│  3. 高频操作 ── 写入频率足够高，使得缓存一致性协议          │
│     的开销成为瓶颈（偶尔写一次可以忽略）                   │
│                                                         │
│  典型场景：                                              │
│  • 线程安全计数器数组                                      │
│  • volatile 标志位集群                                    │
│  • 生产者-消费者队列的 head/tail 指针                     │
│  • ConcurrentHashMap 的计数器单元                        │
└─────────────────────────────────────────────────────────┘
```

---

## 火眼金睛：如何用工具发现伪共享

### 6.1 perf 工具（Linux）

```bash
# 统计 L1 缓存未命中率
perf stat -e cache-references,cache-misses,L1-dcache-load-misses \
    -p <pid> sleep 10

# 如果 L1-dcache-load-misses 异常高 → 可能伪共享

# 更精确：统计 HITM（Modified 状态的缓存行命中）
# Intel 特定事件：
perf stat -e cpu/event=0xd1,umask=0x01,name=MEM_LOAD_L3_HIT_RETIRED_XSNP_HITM/ \
    -p <pid> sleep 10
# 高 HITM 计数 = 明确的伪共享信号
```

### 6.2 Java 层面的诊断

```java
// 使用 Unsafe 查看对象字段的实际内存偏移
import sun.misc.Unsafe;
import java.lang.reflect.Field;

public class LayoutInspector {
    private static final Unsafe U;

    static {
        try {
            Field f = Unsafe.class.getDeclaredField("theUnsafe");
            f.setAccessible(true);
            U = (Unsafe) f.get(null);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }

    public static void inspect(Class<?> clazz, String... fields) {
        for (String name : fields) {
            try {
                Field f = clazz.getDeclaredField(name);
                long offset = U.objectFieldOffset(f);
                long cacheLine = offset / 64;
                long lineOffset = offset % 64;
                System.out.printf("%s: offset=%d, cacheLine=%d, lineOffset=%d%n",
                    name, offset, cacheLine, lineOffset);
            } catch (NoSuchFieldException e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) {
        inspect(FalseSharingDisaster.class, "counter1", "counter2");
        // 预期输出：两个 offset 相差 < 64，在同一缓存行
    }
}
```

### 6.3 JOL（Java Object Layout）

```java
// Maven: org.openjdk.jol:jol-core
import org.openjdk.jol.info.ClassLayout;

public class JOLDemo {
    public static void main(String[] args) {
        System.out.println(ClassLayout.parseClass(MyClass.class).toPrintable());
    }
}
// 输出：
// OFFSET  SIZE   TYPE   DESCRIPTION
//      0    12          (object header)
//     12     4          (alignment)
//     16     8   long   x
//     24     8   long   y
// Instance size: 32 bytes ← 完全在一个缓存行内
```

---

## 解决方案：从填充到注解

### 7.1 方案一：手动填充（传统方式）

```java
/**
 * 消除伪共享——手动填充 64 字节
 */
public class PaddedCounter {
    // 前填充：确保 x 在缓存行的开头
    long p1, p2, p3, p4, p5, p6, p7;

    volatile long value = 0;

    // 后填充：确保下一个变量不在同一缓存行
    long q1, q2, q3, q4, q5, q6, q7;
    // 总计：8 × 7 + 8 + 8 × 7 = 120 字节，x 独占至少一个缓存行
}
// 缺点：浪费内存；JIT 可能把未使用的填充字段优化掉
```

### 7.2 方案二：@Contended 注解（JDK 8+，推荐）

```java
import jdk.internal.vm.annotation.Contended;

/**
 * JDK 内置的伪共享解决方案——最简单的用法
 */
public class ModernPaddedCounter {
    @Contended
    volatile long value = 0;
    // JDK 自动在 value 前后添加 padding，确保独占缓存行
}

/**
 * 分组隔离：需要一起访问的变量放一组
 */
public class GroupedFields {
    @Contended("group1")
    volatile long counter1;

    @Contended("group1")
    volatile long counter2;

    @Contended("group2")
    volatile long timestamp1;

    @Contended("group2")
    volatile long timestamp2;
    // group1 和 group2 共享同一组 padding，互不干扰
}
// 启动参数：-XX:-RestrictContended  （默认为 false，需要显式开启）
```

### 7.3 方案三：数组填充（适用于计数器数组）

```java
/**
 * 消除计数器数组的伪共享
 * 每个计数器独占一个或多个缓存行
 */
public class PaddedAtomicLongArray {
    // 假设缓存行 64 字节，long 8 字节
    // 每 8 个 long 填充 1 个实际计数器
    private static final int STRIDE = 8; // 64 / 8

    private final AtomicLongArray array;

    public PaddedAtomicLongArray(int logicalSize) {
        this.array = new AtomicLongArray(logicalSize * STRIDE);
    }

    public long get(int index) {
        return array.get(index * STRIDE);
    }

    public long incrementAndGet(int index) {
        return array.incrementAndGet(index * STRIDE);
    }
    // 每个计数器独占 64 字节，零伪共享
}
```

### 7.4 方案四：Striped64 模式（JDK LongAdder 的思路）

```java
/**
 * 借鉴 LongAdder 的分段计数思想
 * 每个线程操作不同的 Cell，Cell 之间有填充隔离
 */
public class StripedCounter {
    @Contended
    static final class Cell {
        volatile long value;
    }

    private final Cell[] cells;

    public StripedCounter(int parallelism) {
        this.cells = new Cell[parallelism];
        for (int i = 0; i < parallelism; i++) {
            cells[i] = new Cell();
        }
    }

    public void add(long delta) {
        // 根据线程哈希选择 Cell，减少竞争和伪共享
        int index = (int) (Thread.currentThread().threadId() % cells.length);
        cells[index].value += delta;
    }

    public long sum() {
        long total = 0;
        for (Cell cell : cells) {
            total += cell.value;
        }
        return total;
    }
}
```

---

## JMH 基准测试：消除伪共享前后的天壤之别

### 8.1 测试代码

```java
import org.openjdk.jmh.annotations.*;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicLong;

@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.MICROSECONDS)
@State(Scope.Group)
@Fork(1)
@Warmup(iterations = 3, time = 2)
@Measurement(iterations = 5, time = 2)
public class FalseSharingBenchmark {

    // 普通——两个 AtomicLong 紧挨着，同一缓存行
    @State(Scope.Group)
    public static class PlainState {
        final AtomicLong counter1 = new AtomicLong();
        final AtomicLong counter2 = new AtomicLong();
    }

    // 填充——每个 AtomicLong 独占缓存行
    @State(Scope.Group)
    public static class PaddedState {
        // 用对象包装+填充
        static class PaddedAtomicLong extends AtomicLong {
            long p1, p2, p3, p4, p5, p6, p7;  // 前填充 56 字节
            // AtomicLong 的 value 字段在父类中
            long q1, q2, q3, q4, q5, q6, q7;  // 后填充 56 字节
        }
        final AtomicLong counter1 = new PaddedAtomicLong();
        final AtomicLong counter2 = new PaddedAtomicLong();
    }

    @Benchmark
    @Group("plain")
    @GroupThreads(1)
    public long plain_t1(PlainState s) {
        return s.counter1.incrementAndGet();
    }

    @Benchmark
    @Group("plain")
    @GroupThreads(1)
    public long plain_t2(PlainState s) {
        return s.counter2.incrementAndGet();
    }

    @Benchmark
    @Group("padded")
    @GroupThreads(1)
    public long padded_t1(PaddedState s) {
        return s.counter1.incrementAndGet();
    }

    @Benchmark
    @Group("padded")
    @GroupThreads(1)
    public long padded_t2(PaddedState s) {
        return s.counter2.incrementAndGet();
    }
}
```

### 8.2 实测结果（M3 Pro, JDK 21）

```
Benchmark                     Mode  Cnt      Score      Error  Units
────────────────────────────────────────────────────────────────────
plain                         thrpt   5    38247.2 ±   1291.1  ops/us
padded                        thrpt   5   295384.8 ±   8532.3  ops/us

提升倍数：295384 / 38247 = 7.7x
单次操作延迟：1000/38247 = 26ns → 1000/295384 = 3.4ns
```

**结论：消除伪共享后吞吐量提升近 8 倍，单次操作延迟从 26ns 降至 3.4ns。**

---

## Disruptor 的缓存行填充艺术

### 9.1 经典的多字段填充

```java
// Disruptor Sequence.java 简化版
public class Sequence {
    // 前填充：确保 value 在其缓存行的起始位置附近
    long p1, p2, p3, p4, p5, p6, p7;

    volatile long value;  // 核心字段

    // 后填充：确保下一个 Sequence 对象不在同一缓存行
    long q1, q2, q3, q4, q5, q6, q7;
}
// 这种写法在 JDK 7 时代是消除伪共享的标准做法
// 但有一个隐患：JIT 可能会消除未使用的填充字段
```

### 9.2 Disruptor 的防御措施

```java
// 现代 Disruptor 使用 VarHandle + 手动填充
// 将填充字段声明为实际访问（即使什么都不做）
public class Sequence extends RhsPadding {
    // 通过继承层次保证填充不被 JIT 优化掉
}

class LhsPadding {
    long p1, p2, p3, p4, p5, p6, p7;
}

class Value extends LhsPadding {
    volatile long value;
}

class RhsPadding extends Value {
    long p1, p2, p3, p4, p5, p6, p7;
}
```

### 9.3 JDK 内部的用法

```java
// Thread.java 源码中的 @Contended 使用
@Contended("tlr")
long threadLocalRandomSeed;

@Contended("tlr")
int threadLocalRandomProbe;

@Contended("tlr")
int threadLocalRandomSecondarySeed;

// ForkJoinPool 中的使用
@Contended
volatile int scanState;

// ConcurrentHashMap 中的 counter cells
@Contended
static final class CounterCell {
    volatile long value;
    CounterCell(long x) { value = x; }
}
```

---

## 常见陷阱与最佳实践

### 陷阱 1：填充被 JIT 消掉

```java
// ❌ JIT 可以消除未使用的填充字段
volatile long value;
long p1, p2, p3, p4, p5, p6;  // 从未被读取→可能被消除

// ✅ 使用 Unsafe 或 VarHandle 声明占位
// 或直接用 @Contended 注解
```

### 陷阱 2：数组的伪共享更隐蔽

```java
// ❌ volatile 数组——相邻元素在同一缓存行
volatile long[] counters = new volatile long[16];
// counters[0] 和 counters[1] 共享缓存行 → 伪共享

// ✅ 填充数组
long[] counters = new long[16 * 8];  // stride = 8
// 实际使用的索引：0, 8, 16, 24, ...
```

### 陷阱 3：错误估计缓存行大小

```java
// ⚠️ x86-64: 通常 64B，ARM: 通常 64B 或 128B
// Apple M 系列: 128B 缓存行

// 安全做法：填充 128 字节（兼容各种平台）
@Contended  // JDK 自动处理平台差异
volatile long value;

// 或手动填充两倍
long p1,p2,p3,p4,p5,p6,p7,p8,p9,p10,p11,p12,p13,p14,p15,p16;
volatile long value;
long q1,q2,q3,q4,q5,q6,q7,q8,q9,q10,q11,q12,q13,q14,q15,q16;
```

### 最佳实践清单

1. **用 `@Contended` 替代手动填充**：JDK 8+，开启 `-XX:-RestrictContended`
2. **怀疑一切多线程计数器**：任何共享 volatile/Atomic 变量都可能是伪共享候选
3. **用 perf/JMH 验证**：直觉不可靠，数据不会撒谎
4. **优先用 LongAdder**：内部已做填充隔离，开箱即用
5. **Disruptor 模式**：RingBuffer 天然避免伪共享（预分配+序列号填充）
6. **权衡内存开销**：每个填充变量浪费 56-120 字节，大规模使用需评估

---

## 面试题与参考答案

### Q1：什么是伪共享？为什么它只在多核系统上出现？

**参考答案**：
伪共享是指两个线程访问逻辑上无关的变量，但这些变量恰好落在同一 CPU 缓存行中，导致缓存一致性协议将一个核心的写入误判为对另一个核心正在使用的数据的修改，触发不必要的缓存行失效和总线流量。它只在多核系统上出现，因为单核不存在缓存一致性问题——只有一个核心在操作缓存。

### Q2：`volatile` 为什么会让伪共享更严重？

**参考答案**：
`volatile` 强制每次写入都立即刷新到主存并让其他核心的缓存行失效。两个 volatile 变量在同一缓存行时，任何一个的写入都会让对方的缓存行失效，而 volatile 的语义是**每次读取都必须从主存重新加载**——这正好命中了最慢的路径（~100ns）而不是最快的 L1 缓存路径（~1ns）。

### Q3：`LongAdder` 比 `AtomicLong` 好在哪？跟伪共享有什么关系？

**参考答案**：
`LongAdder` 内部使用 `Striped64`，将单一 `AtomicLong` 拆分为多个 `@Contended` 标注的 `Cell`。不同线程更新不同的 Cell，Cell 之间有填充隔离，避免伪共享，因此在高并发下吞吐量远高于 `AtomicLong`。本质上是**用空间换并行度**：用多个填充过的 Cell + 最终 sum() 合并来消除伪共享和减少 CAS 竞争。

### Q4：为什么 CPU 厂商不做更大的 L1 缓存来避免缓存未命中？

**参考答案**：
1. **物理限制**：L1 缓存越大，访问延迟越高（走线更长，译码更复杂）
2. **功耗/面积**：SRAM 占 die 面积，大 L1 增加成本和发热
3. **命中率饱和**：超过一定大小（32-64KB）后，命中率提升微乎其微
4. **设计哲学**：分层缓存已经是 Pareto 最优解——L1 快而小，L2 中大，L3 大而慢

### Q5：设计一个多生产者-多消费者无锁队列时，如何避免伪共享？

**参考答案**：
```java
// 方案：将 head、tail、producer counter、consumer counter 分别放在独立缓存行
class MPSCQueue<T> {
    @Contended volatile long head;          // 消费者读取
    @Contended volatile long tail;          // 生产者 CAS 更新
    @Contended volatile long consumerIndex; // 消费者进度
    // 每个字段独占缓存行，各核心操作互不干扰
}
// 结合 RingBuffer 预分配内存，整个队列全程无锁 + 无伪共享
```

---

## 小结

```
┌────────────────────────────────────────────────────────────┐
│                 伪共享诊断决策树                              │
│                                                            │
│              多线程频繁写邻近变量？                            │
│                    │                                       │
│          ┌─────────┴─────────┐                             │
│          │ YES               │ NO                          │
│          ▼                   ▼                             │
│   用 JOL 查看布局         不是伪共享                          │
│          │                   ──────                        │
│    同一缓存行？                                            │
│          │                                                 │
│    ┌─────┴─────┐                                           │
│    │ YES       │ NO                                        │
│    ▼           ▼                                           │
│  伪共享确认   不是伪共享                                     │
│    │           ──────                                      │
│    ▼                                                       │
│  解决方案：                                                 │
│  1. @Contended 注解                                        │
│  2. 手动填充 64~128 字节                                    │
│  3. 数组 stride 设计                                       │
│  4. 换用 LongAdder                                         │
│    │                                                       │
│    ▼                                                       │
│  JMH 验证性能改善                                           │
└────────────────────────────────────────────────────────────┘
```

**最终建议**：不要过度优化。伪共享只在**多线程 + 高频写入**场景下才值得关注。CRUD 业务代码不需要考虑。但当你在写中间件、队列、计数器、Disruptor 风格的组件时，**缓存行意识**是区分普通代码与高性能代码的分水岭。

---

*本文是"底层技术"系列第二篇。从硬件到 Java，还原了伪共享从晶体管到代码的全链路。*

> 系列：底层技术  
> 编号：T2  
> 上一篇：T1 - 位运算与取模  
> 下一篇：T3 - 内存屏障（Memory Barrier）与指令重排
