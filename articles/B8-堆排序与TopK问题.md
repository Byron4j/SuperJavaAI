# 堆排序与TopK问题深度解析：优先队列实现原理与工业实践

**文章标签：** #java #算法 #堆排序 #优先队列 #TopK #面试 #数据结构

## 目录

- [引言：堆的本质](#引言堆的本质)
- [理论基础：堆的数学原理](#理论基础堆的数学原理)
- [底层原理：堆的实现机制](#底层原理堆的实现机制)
- [源码深度分析：PriorityQueue实现](#源码深度分析priorityqueue实现)
- [堆排序算法详解](#堆排序算法详解)
- [实战案例：TopK问题](#实战案例topk问题)
- [工程应用案例](#工程应用案例)
- [算法对比与性能分析](#算法对比与性能分析)
- [优化技术](#优化技术)
- [常见陷阱](#常见陷阱)
- [面试题与参考答案](#面试题与参考答案)

---

## 引言：堆的本质

堆（Heap）不是内存中的"堆栈"，而是一种**基于完全二叉树构建的抽象数据类型（ADT）**。其核心能力是在动态数据集合中**高效地获取和移除最值**。

核心认知：

```
数组的本质：连续存储的线性结构

堆的本质：将数组索引映射为完全二叉树的父子关系
                      通过父子比较维护局部有序性

质量差异的根源：
- 差的理解：把堆当成"排序后的数组"
- 好的理解：堆是"部分有序的完全二叉树"，只保证父子关系，不保证兄弟关系
```

**关键洞察**：堆的价值不在于全局有序，而在于以 **O(1)** 获取最值、以 **O(log n)** 插入和删除，这是解决TopK、任务调度、合并K路等问题的基础设施。

---

## 理论基础：堆的数学原理

### 1. 完全二叉树的数组映射

堆使用数组存储完全二叉树，利用索引公式快速定位父子节点：

```
对于索引 i（从0开始）：
- 父节点：parent(i) = (i - 1) / 2
- 左子节点：left(i) = 2i + 1
- 右子节点：right(i) = 2i + 2
```

**ASCII图示**：

```
数组索引: [0]  [1]  [2]  [3]  [4]  [5]  [6]
数组值:   90   80   70   60   50   40   30

树形结构:
              90
            /    \
          80      70
         /  \    /  \
       60   50  40   30
```

**关键性质**：
- 第 k 层最多有 2^k 个节点（k 从0开始）
- n 个节点的完全二叉树高度为 ⌊log₂n⌋ + 1
- 叶子节点索引范围：⌊n/2⌋ 到 n-1

### 2. 堆的数学定义

**大顶堆（Max Heap）**：
∀i, A[parent(i)] ≥ A[i]

**小顶堆（Min Heap）**：
∀i, A[parent(i)] ≤ A[i]

**注意**：堆只约束父子关系，不约束兄弟关系。即左子节点可以大于或小于右子节点。

### 3. 操作的时间复杂度证明

| 操作 | 时间复杂度 | 证明 |
|------|-----------|------|
| 插入（offer） | O(log n) | 最坏从叶子上浮到根，树高 log n |
| 删除堆顶（poll） | O(log n) | 最坏从根下沉到叶子，树高 log n |
| 查看堆顶（peek） | O(1) | 直接取数组首元素 |
| 建堆（Heapify） | O(n) | 见下方严格证明 |
| 堆排序 | O(n log n) | n 次 poll，每次 O(log n) |

**建堆 O(n) 的严格证明**：

高度为 h 的节点最多有 ⌈n / 2^(h+1)⌉ 个。
每个高度为 h 的节点下沉最多 h 层。

总时间复杂度：

```
T(n) = Σ(h=0 to ⌊log n⌋) ⌈n / 2^(h+1)⌉ × O(h)
     = O(n × Σ(h=0 to ∞) h / 2^(h+1))

设 S = Σ(h=0 to ∞) h / 2^h
S   = 0 + 1/2 + 2/4 + 3/8 + 4/16 + ...
S/2 = 0 + 1/4 + 2/8 + 3/16 + ...

S - S/2 = 1/2 + 1/4 + 1/8 + ... = 1
S = 2

因此 T(n) = O(n × 2/2) = O(n)
```

**结论**：建堆的时间复杂度是 **O(n)**，不是 O(n log n)。这是堆排序能优于其他 O(n log n) 排序算法的关键。

---

## 底层原理：堆的实现机制

### 1. 大顶堆完整实现

```java
/**
 * 大顶堆的数组实现
 * 核心操作：siftUp（上浮）和 siftDown（下沉）
 * @param <T> 元素类型，必须可比较
 */
public class MaxHeap<T extends Comparable<T>> {
    private Object[] data;     // 堆数组，使用Object[]以兼容泛型
    private int size;          // 当前元素数量
    private int capacity;      // 数组容量
    
    public MaxHeap(int capacity) {
        this.capacity = capacity;
        this.data = new Object[capacity];
        this.size = 0;
    }
    
    /**
     * 通过已有数组建堆（Heapify）
     * 时间复杂度：O(n) -- 优于逐个插入的 O(n log n)
     * @param arr 输入数组
     */
    public MaxHeap(T[] arr) {
        this.capacity = arr.length;
        this.data = new Object[capacity];
        System.arraycopy(arr, 0, data, 0, arr.length);
        this.size = arr.length;
        
        // 从最后一个非叶子节点开始下沉
        // 最后一个非叶子节点 = 最后一个节点的父节点 = (size-2)/2
        for (int i = (size - 2) / 2; i >= 0; i--) {
            siftDown(i);
        }
    }
    
    /**
     * 插入元素：先放末尾，再上浮
     * 时间复杂度：O(log n)
     * @param val 要插入的元素
     */
    public void insert(T val) {
        if (size >= capacity) {
            throw new IllegalStateException("Heap is full");
        }
        
        // 1. 将新元素放到数组末尾（完全二叉树的下一个位置）
        data[size] = val;
        
        // 2. 从最后一个位置开始上浮，维护堆性质
        siftUp(size);
        
        size++;
    }
    
    /**
     * 上浮操作：维护堆性质
     * 当前节点与父节点比较，如果大于父节点则交换
     * 类似于冒泡，但只向一个方向
     * @param index 当前节点索引
     */
    @SuppressWarnings("unchecked")
    private void siftUp(int index) {
        while (index > 0) {
            int parent = (index - 1) / 2;  // 父节点索引
            
            // 当前节点 <= 父节点，堆性质满足，停止上浮
            if (((Comparable<T>) data[parent]).compareTo((T) data[index]) >= 0) {
                break;
            }
            
            // 交换当前节点和父节点
            swap(parent, index);
            index = parent;  // 继续向上检查
        }
    }
    
    /**
     * 删除堆顶：用末尾元素替换，再下沉
     * 时间复杂度：O(log n)
     * @return 堆顶元素（最大值）
     */
    @SuppressWarnings("unchecked")
    public T extractMax() {
        if (size == 0) {
            throw new IllegalStateException("Heap is empty");
        }
        
        T max = (T) data[0];  // 堆顶元素（最大值）
        
        // 1. 用最后一个元素替换堆顶（保持完全二叉树结构）
        data[0] = data[size - 1];
        data[size - 1] = null;  // 帮助GC
        size--;
        
        // 2. 从堆顶开始下沉，恢复堆性质
        if (size > 0) {
            siftDown(0);
        }
        
        return max;
    }
    
    /**
     * 下沉操作：维护堆性质
     * 当前节点与左右子节点比较，如果小于最大子节点则交换
     * 类似于选择，但只向叶子方向
     * @param index 当前节点索引
     */
    @SuppressWarnings("unchecked")
    private void siftDown(int index) {
        while (true) {
            int maxIndex = index;
            int left = 2 * index + 1;   // 左子节点
            int right = 2 * index + 2;  // 右子节点
            
            // 找出当前节点和左右子节点中的最大值
            if (left < size && ((Comparable<T>) data[left]).compareTo((T) data[maxIndex]) > 0) {
                maxIndex = left;
            }
            if (right < size && ((Comparable<T>) data[right]).compareTo((T) data[maxIndex]) > 0) {
                maxIndex = right;
            }
            
            // 当前节点已经是最大值，停止下沉
            if (maxIndex == index) break;
            
            // 交换当前节点和最大子节点
            swap(index, maxIndex);
            index = maxIndex;  // 继续向下检查
        }
    }
    
    private void swap(int i, int j) {
        Object temp = data[i];
        data[i] = data[j];
        data[j] = temp;
    }
    
    @SuppressWarnings("unchecked")
    public T peek() {
        if (size == 0) throw new IllegalStateException("Heap is empty");
        return (T) data[0];
    }
    
    public int size() { return size; }
    public boolean isEmpty() { return size == 0; }
}
```

### 2. 建堆过程可视化

**输入数组**：[3, 1, 4, 1, 5, 9, 2, 6]

**Step 0**：初始完全二叉树

```
        3
      /   \
     1     4
    / \   / \
   1   5 9   2
  /
 6
```

**Step 1**：siftDown(3) -- 值为5
- 子节点：无（2*3+1=7超出size=8的范围？等等，索引7存在，值为6）
- 左子索引7，值6 > 5，交换

```
        3
      /   \
     1     4
    / \   / \
   1   6 9   2
  /
 5
```

**Step 2**：siftDown(2) -- 值为4
- 子节点：索引5(9), 索引6(2)
- 最大子节点：9 > 4，与索引5交换

```
        3
      /   \
     1     9
    / \   / \
   1   6 4   2
  /
 5
```

**Step 3**：siftDown(1) -- 值为1
- 子节点：索引3(1), 索引4(6)
- 最大子节点：6 > 1，与索引4交换

```
        3
      /   \
     6     9
    / \   / \
   1   1 4   2
  /
 5
```

**Step 4**：siftDown(0) -- 值为3
- 子节点：索引1(6), 索引2(9)
- 最大子节点：9 > 3，与索引2交换

```
        9
      /   \
     6     3
    / \   / \
   1   1 4   2
  /
 5
```

- 继续下沉：索引2值为3，子节点索引5(4), 索引6(2)
- 最大子节点：4 > 3，与索引5交换

**最终结果**：

```
        9
      /   \
     6     4
    / \   / \
   1   1 3   2
  /
 5

堆数组：[9, 6, 4, 1, 1, 3, 2, 5]
```

---

## 源码深度分析：PriorityQueue实现

Java 的 `PriorityQueue` 是基于**小顶堆**实现的优先队列。分析其源码有助于理解工业级堆的实现细节。

### 1. 核心字段

```java
public class PriorityQueue<E> extends AbstractQueue<E> {
    private static final int DEFAULT_INITIAL_CAPACITY = 11;
    
    /**
     * 堆数组，使用Object[]存储泛型元素
     * transient：不被序列化（序列化时自定义writeObject）
     */
    transient Object[] queue;
    
    /** 当前元素数量 */
    private int size = 0;
    
    /** 比较器，为null时使用元素自然排序 */
    private final Comparator<? super E> comparator;
    
    /** 结构性修改计数，用于fast-fail */
    transient int modCount = 0;
    
    // ...
}
```

### 2. 扩容策略

```java
/**
 * 扩容机制：小容量时翻倍+2，大容量时增长50%
 * 设计理念：小容量时减少频繁扩容，大容量时节省内存
 */
private void grow(int minCapacity) {
    int oldCapacity = queue.length;
    
    // 小容量（<64）：翻倍并加2
    // 大容量（>=64）：增长50%
    int newCapacity = oldCapacity + ((oldCapacity < 64) ? 
                                     (oldCapacity + 2) : 
                                     (oldCapacity >> 1));
    
    // 处理溢出
    if (newCapacity < 0) // 溢出
        newCapacity = Integer.MAX_VALUE;
    if (newCapacity < minCapacity)
        newCapacity = minCapacity;
    
    queue = Arrays.copyOf(queue, newCapacity);
}
```

**扩容策略分析**：
- **小容量时翻倍+2**：避免小数组频繁扩容（如从11到24到50）
- **大容量时50%**：避免大数组过度浪费内存（如100万到150万）
- 与 ArrayList 的 "1.5倍" 策略类似，但小容量时更激进

### 3. 上浮操作：siftUp

```java
/**
 * 入队操作：O(log n)
 */
public boolean offer(E e) {
    if (e == null) throw new NullPointerException();
    modCount++;
    int i = size;
    if (i >= queue.length)
        grow(i + 1);  // 必要时扩容
    size = i + 1;
    if (i == 0)
        queue[0] = e;  // 第一个元素直接放堆顶
    else
        siftUp(i, e);  // 新元素上浮到正确位置
    return true;
}

/**
 * 上浮：将元素 k 插入到位置 x
 * 使用 >>>(无符号右移) 代替 / 运算，提升性能
 */
private void siftUp(int k, E x) {
    if (comparator != null)
        siftUpUsingComparator(k, x);
    else
        siftUpComparable(k, x);
}

@SuppressWarnings("unchecked")
private void siftUpComparable(int k, E x) {
    Comparable<? super E> key = (Comparable<? super E>) x;
    while (k > 0) {
        int parent = (k - 1) >>> 1;  // 父节点索引，>>> 比 / 更高效
        Object e = queue[parent];
        if (key.compareTo((E) e) >= 0)
            break;  // 当前节点 >= 父节点，堆性质满足
        queue[k] = e;  // 父节点下移到当前位置
        k = parent;    // 继续向上比较
    }
    queue[k] = key;  // 将元素放到最终位置
}
```

**为什么用 `>>>` 而不是 `/`**：
1. **位运算更快**：CPU 的移位指令周期数远少于除法指令
2. **逻辑正确**：对于正整数，`>>> 1` 和 `/ 2` 等价；对于无符号处理更统一
3. **JDK 惯例**：HotSpot VM 中对这类操作有优化，但源码层面体现设计意图

### 4. 下沉操作：siftDown

```java
/**
 * 出队操作：移除并返回堆顶元素，O(log n)
 */
@SuppressWarnings("unchecked")
public E poll() {
    if (size == 0)
        return null;
    int s = --size;
    modCount++;
    E result = (E) queue[0];  // 堆顶元素
    E x = (E) queue[s];       // 末尾元素
    queue[s] = null;          // 帮助GC
    if (s != 0)
        siftDown(0, x);       // 末尾元素放到堆顶，然后下沉
    return result;
}

/**
 * 下沉：将元素 x 从位置 k 开始下沉
 */
private void siftDown(int k, E x) {
    if (comparator != null)
        siftDownUsingComparator(k, x);
    else
        siftDownComparable(k, x);
}

@SuppressWarnings("unchecked")
private void siftDownComparable(int k, E x) {
    Comparable<? super E> key = (Comparable<? super E>) x;
    int half = size >>> 1;  // 只有非叶子节点需要下沉
    while (k < half) {
        int child = (k << 1) + 1;  // 左子节点 = 2k + 1
        Object c = queue[child];
        int right = child + 1;     // 右子节点
        
        // 取左右子节点中较小的一个（小顶堆）
        if (right < size && ((Comparable<? super E>) c).compareTo((E) queue[right]) > 0)
            c = queue[child = right];
        
        // 当前节点 <= 最小子节点，停止
        if (key.compareTo((E) c) <= 0)
            break;
        
        queue[k] = c;      // 子节点上移
        k = child;         // 继续向下
    }
    queue[k] = key;  // 放到最终位置
}
```

### 5. 自定义比较器实现大顶堆

```java
// 方式1：Lambda 表达式（简洁，但可能有溢出风险）
PriorityQueue<Integer> maxHeap = new PriorityQueue<>((a, b) -> b - a);

// 方式2：Comparator.reversedOrder()（推荐，无溢出风险）
PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Comparator.reverseOrder());

// 方式3：自定义对象按属性排序
PriorityQueue<Task> taskQueue = new PriorityQueue<>(
    Comparator.comparingInt(Task::getPriority).reversed()
);

// 方式4：多字段排序
PriorityQueue<Student> pq = new PriorityQueue<>(
    Comparator.comparingInt(Student::getScore).reversed()
              .thenComparing(Student::getName)
);
```

**注意**：`(a, b) -> b - a` 在数值过大时可能溢出（如 b=Integer.MAX_VALUE, a=Integer.MIN_VALUE），生产环境推荐使用 `Comparator.reverseOrder()`。

---

## 堆排序算法详解

### 1. 核心思想

堆排序 = 建堆 + 反复提取最值

```
阶段1：将无序数组构建成大顶堆
       -> 堆顶是全局最大值

阶段2：将堆顶与末尾交换，缩小堆范围，重新堆化
       -> 末尾元素已排序
       -> 重复直到堆为空
```

### 2. 完整实现

```java
/**
 * 堆排序实现
 * 时间复杂度：O(n log n)
 * 空间复杂度：O(1)（原地排序，仅用常数额外空间）
 * 稳定性：不稳定
 */
public class HeapSort {
    
    /**
     * 堆排序主方法
     * @param arr 待排序数组（会被原地修改）
     */
    public static void sort(int[] arr) {
        if (arr == null || arr.length < 2) return;
        
        int n = arr.length;
        
        // 阶段1：建堆（大顶堆）
        // 从最后一个非叶子节点开始下沉
        for (int i = (n - 2) / 2; i >= 0; i--) {
            siftDown(arr, n, i);
        }
        
        // 阶段2：排序
        // 每次将堆顶（最大值）放到当前未排序部分的末尾
        for (int i = n - 1; i > 0; i--) {
            swap(arr, 0, i);       // 堆顶（最大）和末尾交换
            siftDown(arr, i, 0);   // 对前i个元素重新堆化
        }
    }
    
    /**
     * 下沉操作
     * @param arr 数组
     * @param size 当前堆大小（不是所有元素都参与堆化）
     * @param index 待下沉节点索引
     */
    private static void siftDown(int[] arr, int size, int index) {
        while (true) {
            int maxIndex = index;
            int left = 2 * index + 1;
            int right = 2 * index + 2;
            
            // 与左子节点比较
            if (left < size && arr[left] > arr[maxIndex])
                maxIndex = left;
            
            // 与右子节点比较
            if (right < size && arr[right] > arr[maxIndex])
                maxIndex = right;
            
            // 当前节点已经是最大值，停止
            if (maxIndex == index) break;
            
            swap(arr, index, maxIndex);
            index = maxIndex;  // 继续向下检查
        }
    }
    
    private static void swap(int[] arr, int i, int j) {
        int temp = arr[i];
        arr[i] = arr[j];
        arr[j] = temp;
    }
    
    /**
     * 验证数组是否有序（升序）
     */
    public static boolean isSorted(int[] arr) {
        for (int i = 1; i < arr.length; i++) {
            if (arr[i] < arr[i - 1]) return false;
        }
        return true;
    }
}
```

### 3. 堆排序执行追踪

**输入**：[3, 1, 4, 1, 5, 9, 2, 6]

**建堆后**：[9, 6, 4, 5, 1, 3, 2, 1]（大顶堆）

**排序过程**：

```
初始堆:        9
             / \
            6   4
           /\   /\
          5  1 3  2
         /
        1

第1轮：交换堆顶(9)和末尾(1)
[1,6,4,5,1,3,2,9] -- 9已排序

对前7个元素堆化：
        6
       / \
      5   4
     /\   /\
    1  1 3  2
   
第2轮：交换堆顶(6)和末尾(2)
[2,5,4,1,1,3,6,9] -- 6,9已排序

对前6个元素堆化：
        5
       / \
      2   4
     /\   /\
    1  1 3

第3轮：交换堆顶(5)和末尾(3)
[3,2,4,1,1,5,6,9] -- 5,6,9已排序

对前5个元素堆化：
        4
       / \
      2   3
     /\
    1  1

...继续直到完成

最终结果：[1, 1, 2, 3, 4, 5, 6, 9]
```

### 4. 堆排序的特性分析

| 特性 | 说明 |
|------|------|
| 时间复杂度 | O(n log n)，最好/最坏/平均都是 |
| 空间复杂度 | O(1)，原地排序 |
| 稳定性 | **不稳定**，相等元素相对位置可能改变 |
| 适用场景 | 内存受限、需要保证最坏情况 O(n log n) |
| 不适用场景 | 小规模数据（常数大，不如插排）、需要稳定性 |

**为什么不稳定**：

```
示例：[5a, 5b, 3]（假设5a和5b值相同，但原始顺序不同）

建堆后可能变为：[5b, 5a, 3]（堆不保证兄弟顺序）
第一轮交换：[3, 5a, 5b]
此时 5a 和 5b 的相对顺序改变了（原始5a在前，现在5a还在前？）

更明显的例子：[2, 2, 1]
建堆：[2, 2, 1]（假设索引0的2和索引1的2）
交换堆顶和末尾：[1, 2, 2]
此时两个2的相对位置改变了
```

---

## 实战案例：TopK问题

### 1. 问题定义

TopK 问题：从 N 个元素中找出最大（或最小）的 K 个元素。

**约束条件**：
- N 很大（如 1亿），无法全部加载到内存排序
- K 很小（如 100），只需前 K 个
- 数据可能动态到来（数据流场景）

### 2. 小顶堆解法（找最大的K个数）

```java
/**
 * 找出数组中最大的K个数
 * 核心思想：维护一个大小为K的小顶堆，堆顶是第K大的元素
 * 时间复杂度：O(n log k)
 * 空间复杂度：O(k)
 * 
 * @param nums 输入数组
 * @param k 要找的个数
 * @return 最大的K个数（无序）
 */
public static List<Integer> topK(int[] nums, int k) {
    if (k <= 0 || nums == null || nums.length == 0) {
        return new ArrayList<>();
    }
    
    // 小顶堆，堆顶是当前第K大的元素（堆中最小的）
    PriorityQueue<Integer> minHeap = new PriorityQueue<>(k);
    
    for (int num : nums) {
        if (minHeap.size() < k) {
            // 堆未满，直接加入
            minHeap.offer(num);
        } else if (num > minHeap.peek()) {
            // 当前元素比堆顶大，说明它属于前K大
            // 替换堆顶并调整
            minHeap.poll();
            minHeap.offer(num);
        }
        // 否则当前元素 <= 堆顶，不可能属于前K大，丢弃
    }
    
    return new ArrayList<>(minHeap);
}
```

**为什么用小顶堆**：

```
找最大的K个数：
- 小顶堆堆顶是这K个数中最小的（即第K大的）
- 新元素如果 > 堆顶，说明它应该进入前K大
- 替换堆顶后调整，始终保持堆内是前K大

如果用大顶堆：
- 堆顶是前K大中的最大值
- 新元素即使很大，也无法确定是否应该替换
- 因为不知道当前第K大是多少，无法决策
```

### 3. 数据流中的TopK

```java
/**
 * 数据流中的TopK查找器
 * 支持动态添加元素和实时获取当前TopK
 */
public class StreamTopK {
    private final int k;
    private final PriorityQueue<Integer> minHeap;
    
    public StreamTopK(int k) {
        this.k = k;
        this.minHeap = new PriorityQueue<>(k);
    }
    
    /**
     * 添加一个新元素到数据流
     * 时间复杂度：O(log k)
     */
    public void add(int num) {
        if (minHeap.size() < k) {
            minHeap.offer(num);
        } else if (num > minHeap.peek()) {
            minHeap.poll();
            minHeap.offer(num);
        }
    }
    
    /**
     * 获取当前TopK
     */
    public List<Integer> getTopK() {
        return new ArrayList<>(minHeap);
    }
    
    /**
     * 获取第K大的元素（堆顶）
     */
    public int getKthLargest() {
        if (minHeap.isEmpty()) throw new IllegalStateException("No elements");
        return minHeap.peek();
    }
}
```

### 4. 双堆求中位数

```java
/**
 * 数据流中位数查找器
 * 使用大顶堆 + 小顶堆
 * 时间复杂度：addNum O(log n)，findMedian O(1)
 */
public class MedianFinder {
    // 大顶堆，存较小的一半（堆顶是这半部分的最大值）
    private PriorityQueue<Integer> small = new PriorityQueue<>((a, b) -> b - a);
    // 小顶堆，存较大的一半（堆顶是这半部分的最小值）
    private PriorityQueue<Integer> large = new PriorityQueue<>();
    
    /**
     * 添加一个数
     * 核心：始终保持 small.size() == large.size() 或 small.size() == large.size() + 1
     */
    public void addNum(int num) {
        // 先放入small（大顶堆）
        if (small.isEmpty() || num <= small.peek()) {
            small.offer(num);
        } else {
            large.offer(num);
        }
        
        // 平衡两个堆的大小
        if (small.size() > large.size() + 1) {
            // small多了，移一个到large
            large.offer(small.poll());
        } else if (large.size() > small.size()) {
            // large多了，移一个到small
            small.offer(large.poll());
        }
    }
    
    /**
     * 获取中位数
     */
    public double findMedian() {
        if (small.size() == large.size()) {
            // 偶数个元素，中位数为两个堆顶的平均值
            return (small.peek() + large.peek()) / 2.0;
        } else {
            // 奇数个元素，中位数在small堆顶（small多一个）
            return small.peek();
        }
    }
}
```

**平衡策略可视化**：

```
添加元素3：
small: [3]  large: []
中位数：3

添加元素1：
small: [3]  large: [1] -> 调整
small: [1]  large: [3]
中位数：(1+3)/2 = 2

添加元素4：
small: [1]  large: [3, 4]
large多一个，调整：
small: [3, 1] -> [3,1]  large: [4]
中位数：3

添加元素2：
small: [3, 1]  large: [2, 4]
中位数：(2+3)/2 = 2.5
```

---

## 工程应用案例

### 1. 定时任务调度器

```java
/**
 * 基于优先队列的定时任务调度器
 * 核心：用小顶堆按执行时间排序，每次取最近要执行的任务
 */
public class TaskScheduler {
    private PriorityQueue<Task> taskQueue = new PriorityQueue<>(
        Comparator.comparingLong(Task::getExecuteTime)
    );
    
    /**
     * 调度一个任务
     */
    public void schedule(Task task) {
        taskQueue.offer(task);
    }
    
    /**
     * 运行调度器（单线程示例）
     */
    public void run() {
        while (!taskQueue.isEmpty()) {
            Task task = taskQueue.peek();
            long now = System.currentTimeMillis();
            
            if (task.getExecuteTime() <= now) {
                // 任务到期，执行
                taskQueue.poll();
                task.execute();
            } else {
                // 等待到下一个任务执行时间
                try {
                    Thread.sleep(task.getExecuteTime() - now);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    break;
                }
            }
        }
    }
    
    static class Task {
        private long executeTime;
        private Runnable action;
        
        public Task(long executeTime, Runnable action) {
            this.executeTime = executeTime;
            this.action = action;
        }
        
        public long getExecuteTime() { return executeTime; }
        public void execute() { action.run(); }
    }
}
```

### 2. 合并K个有序数组

```java
/**
 * 合并K个有序数组
 * 核心：小顶堆存储每个数组的当前元素，每次取出最小值
 * 时间复杂度：O(N log K)，N为总元素数，K为数组数
 * 
 * @param arrays K个有序数组
 * @return 合并后的有序列表
 */
public static List<Integer> mergeKArrays(int[][] arrays) {
    List<Integer> result = new ArrayList<>();
    
    // 小顶堆，存储 [值, 数组索引, 元素索引]
    PriorityQueue<int[]> minHeap = new PriorityQueue<>(
        Comparator.comparingInt(a -> a[0])
    );
    
    // 初始化：每个数组的第一个元素入堆
    for (int i = 0; i < arrays.length; i++) {
        if (arrays[i].length > 0) {
            minHeap.offer(new int[]{arrays[i][0], i, 0});
        }
    }
    
    // 不断从堆顶取出最小元素，并将该数组的下一个元素入堆
    while (!minHeap.isEmpty()) {
        int[] curr = minHeap.poll();
        result.add(curr[0]);
        
        int arrIdx = curr[1];
        int elemIdx = curr[2];
        
        // 如果该数组还有下一个元素，入堆
        if (elemIdx + 1 < arrays[arrIdx].length) {
            minHeap.offer(new int[]{
                arrays[arrIdx][elemIdx + 1],
                arrIdx,
                elemIdx + 1
            });
        }
    }
    
    return result;
}
```

**执行过程可视化**：

```
输入：
数组1: [1, 4, 7]
数组2: [2, 5, 8]
数组3: [3, 6, 9]

初始堆：[1(1,0), 2(2,0), 3(3,0)]

Step 1: 取出1，数组1的下一个4入堆
堆：[2(2,0), 4(1,1), 3(3,0)]
结果：[1]

Step 2: 取出2，数组2的下一个5入堆
堆：[3(3,0), 4(1,1), 5(2,1)]
结果：[1, 2]

Step 3: 取出3，数组3的下一个6入堆
堆：[4(1,1), 5(2,1), 6(3,1)]
结果：[1, 2, 3]

...继续
最终结果：[1, 2, 3, 4, 5, 6, 7, 8, 9]
```

---

## 算法对比与性能分析

### 1. 排序算法对比

| 算法 | 平均时间 | 最坏时间 | 空间 | 稳定性 | 适用场景 |
|------|---------|---------|------|--------|----------|
| 快排 | O(n log n) | O(n²) | O(log n) | 不稳定 | 通用首选，平均最快 |
| 归并 | O(n log n) | O(n log n) | O(n) | 稳定 | 需要稳定性，链表排序 |
| 堆排 | O(n log n) | O(n log n) | O(1) | 不稳定 | 内存受限，保证最坏性能 |
| 插排 | O(n²) | O(n²) | O(1) | 稳定 | 小规模数据（n < 50） |

**JDK 实践**：`Arrays.sort()` 对基本类型使用 Dual-Pivot QuickSort，对对象使用 TimSort（归并的优化版）。堆排序在工业级排序库中较少作为默认实现，因为快排的平均性能更好。

### 2. TopK 解法对比

| 方法 | 时间复杂度 | 空间复杂度 | 1亿数据找Top100 | 特点 |
|------|-----------|-----------|----------------|------|
| 全排序 | O(n log n) | O(1)或O(n) | ~15秒 | 简单，但浪费计算 |
| 小顶堆 | O(n log k) | O(k) | ~2秒 | 适合数据流，K较小时极优 |
| QuickSelect | O(n)平均 | O(log n) | ~1秒 | 平均最快，但最坏O(n²) |
| 桶排序 | O(n) | O(n) | ~0.5秒 | 数据范围有限时最优 |

**QuickSelect 实现**：

```java
/**
 * 快速选择：找第K大元素，平均O(n)
 */
public int findKthLargest(int[] nums, int k) {
    int left = 0, right = nums.length - 1;
    int target = nums.length - k;  // 第K大 = 第(n-K)小（0-based）
    
    while (true) {
        int pivotIndex = partition(nums, left, right);
        if (pivotIndex == target) {
            return nums[pivotIndex];
        } else if (pivotIndex < target) {
            left = pivotIndex + 1;
        } else {
            right = pivotIndex - 1;
        }
    }
}

private int partition(int[] nums, int left, int right) {
    int pivot = nums[right];
    int i = left;
    for (int j = left; j < right; j++) {
        if (nums[j] <= pivot) {
            swap(nums, i, j);
            i++;
        }
    }
    swap(nums, i, right);
    return i;
}
```

### 3. 堆变体对比

| 变体 | 结构 | insert | extractMin | merge | 特点 |
|------|------|--------|-----------|-------|------|
| 二叉堆 | 完全二叉树 | O(log n) | O(log n) | O(n) | 最简单，最常用 |
| 二项堆 | 二项树集合 | O(1)摊还 | O(log n) | O(log n) | 支持高效合并 |
| 斐波那契堆 | 斐波那契树 | O(1)摊还 | O(log n)摊还 | O(1) | Dijkstra/Prim优化 |
| 配对堆 | 多叉树 | O(1) | O(log n)实用 | O(1) | 实现简单，性能好 |

---

## 优化技术

### 1. d叉堆（d-ary Heap）

```java
/**
 * d叉堆：每个节点有d个子节点
 * 优点：树高更低（log_d n），插入更快
 * 缺点：删除需要比较d个子节点
 * 适用：插入多、删除少的场景
 */
public class DaryHeap {
    private int d = 4;  // 4叉堆
    private int[] data;
    private int size;
    
    // d叉堆的父子关系
    private int parent(int i) { return (i - 1) / d; }
    private int child(int i, int k) { return d * i + k + 1; }
    
    /**
     * 下沉：需要比较d个子节点
     */
    private void siftDown(int i) {
        int maxIndex = i;
        // 比较所有d个子节点
        for (int k = 0; k < d; k++) {
            int c = child(i, k);
            if (c < size && data[c] > data[maxIndex]) {
                maxIndex = c;
            }
        }
        if (maxIndex != i) {
            swap(i, maxIndex);
            siftDown(maxIndex);
        }
    }
}
```

**性能对比**：
- 二叉堆（d=2）：insert O(log n)，extract O(log n)
- 四叉堆（d=4）：insert O(log₄ n) ≈ O(0.5 log n)，extract O(4 log₄ n) ≈ O(2 log n)
- 结论：如果 insert 操作远多于 extract，d叉堆更优

### 2. 索引堆（Index Heap）

```java
/**
 * 索引堆：不直接存储数据，存储索引
 * 优点：可以高效修改任意元素的优先级（O(log n)）
 * 应用：Dijkstra算法、Prim算法
 */
public class IndexHeap {
    private int[] indexes;  // 索引数组，indexes[i]表示堆中第i个位置存的是哪个元素
    private int[] reverse;  // 反向索引：reverse[i]表示元素i在堆中的位置
    private int[] data;     // 实际数据
    private int size;
    
    /**
     * 修改元素i的值为newValue
     * 时间复杂度：O(log n)
     */
    public void change(int i, int newValue) {
        data[i] = newValue;
        // 通过reverse数组O(1)找到元素i在堆中的位置
        int heapIndex = reverse[i];
        // 上浮或下沉调整
        siftUp(heapIndex);
        siftDown(heapIndex);
    }
}
```

### 3. 堆的批量构建优化

```java
/**
 * 批量建堆优化：避免逐个插入的O(n log n)
 * 使用Floyd建堆算法（自底向下）
 */
public static void heapify(int[] arr) {
    int n = arr.length;
    // 从最后一个非叶子节点开始
    for (int i = (n - 2) / 2; i >= 0; i--) {
        siftDown(arr, n, i);
    }
}
```

---

## 常见陷阱

### 陷阱1：搞混大顶堆和小顶堆的使用场景

**问题**：找最大的K个数用了大顶堆，导致无法维护前K大。

```java
// 错误：找最大的3个数用了大顶堆
PriorityQueue<Integer> wrongHeap = new PriorityQueue<>((a, b) -> b - a);
// 堆顶是最大值，新元素无法确定是否该加入
```

**正确做法**：
- 找最大的K个数 → **小顶堆**（堆顶是第K大，更大的元素替换堆顶）
- 找最小的K个数 → **大顶堆**（堆顶是第K小，更小的元素替换堆顶）

### 陷阱2：认为建堆时间复杂度是O(n log n)

**问题**：逐个插入确实是O(n log n)，但Heapify自底向下建堆是O(n)。

```java
// 低效：逐个插入建堆 O(n log n)
for (int num : arr) heap.insert(num);

// 高效：Heapify建堆 O(n)
heap.heapify(arr);
```

**最佳实践**：
- 批量数据建堆用Heapify
- 数据流式到来时用逐个插入

### 陷阱3：PriorityQueue默认是小顶堆却当大顶堆用

**问题**：Java的PriorityQueue默认是小顶堆，直接用会导致逻辑错误。

```java
PriorityQueue<Integer> pq = new PriorityQueue<>();
pq.offer(3); pq.offer(1); pq.offer(2);
int max = pq.poll();  // 得到1，以为是最大值！
```

**正确做法**：
```java
// 需要大顶堆时传入比较器
PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Comparator.reverseOrder());
```

### 陷阱4：堆排序用于小规模数据

**问题**：堆排序常数较大，对于小规模数据不如插入排序快。

```java
// 对10个元素用堆排序
// 实际比插入排序慢2-3倍（缓存不友好，分支预测差）
```

**最佳实践**：
- 工程中通常结合使用，小数组（n < 50）用插入排序
- Java的Arrays.sort内部就是混合策略

### 陷阱5：忘记处理堆为空的情况

**问题**：调用peek()/poll()时堆为空，抛出异常或返回null。

```java
PriorityQueue<Integer> pq = new PriorityQueue<>();
int top = pq.peek();  // NullPointerException!
int val = pq.poll();  // NoSuchElementException!
```

**最佳实践**：
```java
// 方式1：检查isEmpty()
if (!pq.isEmpty()) { int top = pq.peek(); }

// 方式2：使用poll()返回null的特性
Integer val = pq.poll();
if (val != null) { ... }
```

### 陷阱6：比较器减法溢出

**问题**：使用 `(a, b) -> b - a` 在数值过大时溢出。

```java
// 危险：可能溢出
PriorityQueue<Integer> pq = new PriorityQueue<>((a, b) -> b - a);

// 当 a = Integer.MIN_VALUE, b = Integer.MAX_VALUE
// b - a = Integer.MAX_VALUE - (-2147483648) = -1（溢出！）
```

**正确做法**：
```java
// 安全：使用Comparator.reverseOrder()或Integer.compare
PriorityQueue<Integer> pq = new PriorityQueue<>(Comparator.reverseOrder());
// 或
PriorityQueue<Integer> pq = new PriorityQueue<>((a, b) -> Integer.compare(b, a));
```

---

## 面试题与参考答案

### Q1：堆排序的时间复杂度和空间复杂度是多少？稳定吗？

**参考答案**：

**时间复杂度**：
- 建堆：O(n)
- n-1次提取堆顶并下沉：O(n log n)
- **总时间：O(n log n)**

**空间复杂度**：
- **O(1)**，原地排序，仅用常数额外空间存储临时变量

**稳定性**：
- **不稳定**。堆排序在交换堆顶和末尾元素时，相等元素的相对位置可能改变。

**示例**：
```
[5a, 5b, 3]，假设5a和5b值相同
建堆后可能：[5b, 5a, 3]（堆不保证兄弟顺序）
交换堆顶和末尾：[3, 5a, 5b]
此时5a和5b的相对顺序与原始输入不同
```

---

### Q2：为什么PriorityQueue的siftUp用 `>>>` 而不是 `/`？

**参考答案**：

`(k - 1) >>> 1` 是无符号右移1位，等同于除以2但效率更高。

**原因**：
1. **位运算更快**：CPU的移位指令周期数通常远少于除法指令
2. **逻辑统一**：`>>>` 是无符号右移，对于索引计算（非负数）与 `/ 2` 等价，但语义更明确
3. **JDK惯例**：HotSpot VM 对这类操作有优化，但源码层面体现设计意图

**注意**：对于正整数，`>> 1`、`>>>`、`/ 2` 结果相同。但 `>>>` 更明确表示 "逻辑右移" 的意图。

---

### Q3：TopK问题，为什么用小顶堆而不是大顶堆？

**参考答案**：

**找最大的K个数时**：

**小顶堆方案**：
- 维护一个大小为K的小顶堆
- 堆顶是当前第K大的元素（堆中最小的）
- 新元素如果 > 堆顶，说明它属于前K大，替换堆顶并调整
- 始终保持堆内是前K大的元素

**大顶堆方案的问题**：
- 大顶堆堆顶是前K大中的最大值
- 新元素即使很大，也无法确定是否应该替换
- 因为不知道当前第K大的阈值是多少，无法做决策

**类比**：
- 小顶堆类似于 "守门员"，堆顶是门槛，新来的比门槛高就进来
- 大顶堆没有 "门槛" 的概念，无法判断新元素是否属于前K

---

### Q4：双堆求中位数，如何保证两个堆的大小平衡？

**参考答案**：

**核心规则**：
1. `small.size() == large.size()` 或 `small.size() == large.size() + 1`
2. `small`（大顶堆）中的所有元素 ≤ `large`（小顶堆）中的所有元素

**操作流程**：
1. **插入**：
   - 新元素先入 `small`（如果 `small` 为空或元素 <= small.peek()）
   - 否则入 `large`

2. **平衡**：
   - 如果 `small.size() > large.size() + 1`：将 `small` 堆顶移到 `large`
   - 如果 `large.size() > small.size()`：将 `large` 堆顶移到 `small`

3. **保证顺序**：
   - 每次插入后，将 `small` 堆顶移到 `large`（确保small的最大值 <= large的最小值）

**获取中位数**：
- 偶数个：` (small.peek() + large.peek()) / 2 `
- 奇数个：` small.peek() `（small多一个）

---

### Q5：合并K个有序数组/链表，为什么用堆？

**参考答案**：

**核心思想**：
- 每个数组有一个 "当前指针"，指向当前未合并的最小元素
- 将这些 "当前元素" 放入小顶堆，堆顶就是所有数组的全局最小值
- 取出堆顶后，将该数组的下一个元素入堆

**时间复杂度分析**：
- 每个元素入堆一次、出堆一次
- 每次堆操作 O(log K)，K是数组个数
- 总时间：O(N log K)，N为总元素数

**对比不用堆**：
- 逐一比较K个数组的当前元素：每次 O(K)
- 总时间：O(N × K)，效率低得多

**扩展**：
- 同理可用于合并K个有序链表
- 是外部排序（External Sort）的核心子过程

---

### Q6：建堆的时间复杂度为什么是O(n)而不是O(n log n)？

**参考答案**：

逐个插入建堆确实是O(n log n)，但 **Heapify（自底向下）** 建堆是O(n)。

**证明**：

高度为 h 的节点最多有 ⌈n / 2^(h+1)⌉ 个。
每个节点下沉最多 h 层。

```
T(n) = Σ(h=0 to log n) ⌈n / 2^(h+1)⌉ × O(h)
     = O(n × Σ(h=0 to ∞) h / 2^(h+1))

级数 Σ(h=0 to ∞) h / 2^h = 2（收敛）

因此 T(n) = O(n)
```

**直觉理解**：
- 大多数节点在堆的底部，高度很小
- 只有根节点可能下沉 log n 层，但只有一个根
- 底层节点（占一半）不需要下沉（高度为0）
- 倒数第二层节点（占1/4）最多下沉1层
- 加权平均后，总工作量是线性的

---

### Q7：堆和优先队列是什么关系？

**参考答案**：

**优先队列（Priority Queue）** 是**抽象数据类型（ADT）**，定义了接口：
- `insert(key)`：插入元素
- `extractMax()` / `extractMin()`：移除并返回最值
- `peekMax()` / `peekMin()`：查看最值

**堆（Heap）** 是优先队列的一种**实现方式**，基于完全二叉树，用数组存储。

**其他实现方式对比**：

| 实现方式 | insert | extract | 特点 |
|---------|--------|---------|------|
| 有序数组 | O(n) | O(1) | 插入慢，查询快 |
| 无序数组 | O(1) | O(n) | 插入快，查询慢 |
| 二叉搜索树 | O(log n) | O(log n) | 功能多，但实现复杂，可能退化 |
| **堆** | **O(log n)** | **O(log n)** | **实现简单，空间高效，最常用** |

**结论**：堆是优先队列的**最优实现**（综合考虑时间、空间、实现复杂度）。

---

## 小结

堆与优先队列是解决"最值"和"优先级"问题的核心基础设施：

1. **堆的本质**：完全二叉树的数组映射，通过父子比较维护局部有序
2. **核心操作**：上浮 siftUp（O(log n)）和下沉 siftDown（O(log n)）
3. **建堆优化**：Heapify 自底向下建堆是 O(n)，不是 O(n log n)
4. **堆排序**：O(n log n) 原地排序，空间 O(1)，但不稳定
5. **PriorityQueue**：Java 内置小顶堆，支持自定义 Comparator
6. **TopK 问题**：小顶堆维护 K 个最大，时间 O(n log k)，空间 O(k)
7. **中位数**：双堆技巧，大顶堆 + 小顶堆，add O(log n)，query O(1)
8. **工程应用**：任务调度、合并K路、Dijkstra算法、Huffman编码

**面试重点**：
- 堆的插入删除过程及可视化
- 建堆 O(n) 的数学证明
- TopK 的堆解法及为什么用小顶堆
- 双堆求中位数的平衡策略
- 合并K个有序数组的堆解法
- 堆排序的优缺点及适用场景

---

*此文原创，转载请注明出处。*
