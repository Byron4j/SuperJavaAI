# vLLM 核心原理：PagedAttention 如何让推理速度提升 10 倍，读完这篇你就秒懂

> 同一个 Llama-3-8B 模型，你用 HuggingFace Transformers 部署，每秒只能跑 10 个 token，而且并发一高就 OOM。换到 vLLM 部署，同样的模型同样的 GPU，吞吐量直接飙到 100+ token/s——**这背后核心的功劳就是 PagedAttention**。今天我们不堆公式，用人话把这个 Berkeley 团队的杰作讲透。

---

## 一、KV Cache 的"隐形"内存黑洞

### 1.1 自回归生成的本质

大模型生成文本是一个 token 一个 token 往外"吐"的过程：

```
输入:  "今天天气"  →  输出: "真"
输入:  "今天天气真" →  输出: "好"
输入:  "今天天气真好" → 输出: "！"
```

每生成一个 token，都要把前面的所有 token 重新算一遍注意力——这里面 99% 的计算是重复的。

### 1.2 KV Cache 的初衷

为了不重复计算，业界引入了 **KV Cache**：把每一层的 Key 和 Value 向量缓存起来，生成下一个 token 时只需计算新增 token 的 Attention，前面的直接查缓存。

```python
# 伪代码：KV Cache 的使用
class AttentionWithKVCache:
    def __init__(self, max_seq_len=4096):
        self.k_cache = torch.zeros(32, max_seq_len, 128)  # 32 层 × 4096 长度 × 128 维度
        self.v_cache = torch.zeros(32, max_seq_len, 128)

    def forward(self, new_token_hidden_state, layer_idx, position):
        # 计算新 token 的 K, V
        new_k = self.k_proj(new_token_hidden_state)
        new_v = self.v_proj(new_token_hidden_state)

        # 写入缓存
        self.k_cache[layer_idx, position] = new_k
        self.v_cache[layer_idx, position] = new_v

        # 用缓存中的所有 K, V 计算 Attention
        return attention(new_k, self.k_cache[layer_idx, :position+1],
                        new_v, self.v_cache[layer_idx, :position+1])
```

### 1.3 问题在哪？—— 严重的内存浪费

KV Cache 最大的问题是 **物理内存浪费惊人**。来看 Llama-3-8B 的真实数据：

```
每层的 KV Cache 大小：
  - Key:   4096(token) × 8(head) × 128(dim) × 2(bytes/FP16) = 8 MB
  - Value: 4096(token) × 8(head) × 128(dim) × 2(bytes/FP16) = 8 MB
  - 每层合计: 16 MB
  - 32 层总计: 512 MB

但这是"预留"的最大长度！实际使用中：
  - 用户 A 的对话才 200 个 token：只用了 512MB × 200/4096 = 25MB
  - 用户 B 的对话才 50 个 token：只用了 6.25MB
  - 但系统给每个请求都预留了 512MB！(n_ctx=4096)
```

这就导致：
1. **内存利用率 < 5%**——95% 的预分配内存是空的
2. **并发能力极差**——一张 24GB 的显卡，抛开模型权重 16GB，只剩 8GB 用于 KV Cache，最多同时服务 **16 个请求**（8GB / 512MB）
3. **OOM 频繁**——稍有超出就直接爆显存

---

## 二、PagedAttention：借操作系统的智慧

### 2.1 一个绝妙的类比

vLLM 团队的洞察是：**KV Cache 的内存管理问题，和操作系统的虚拟内存管理一模一样！**

```
操作系统的虚拟内存：
  - 进程申请 1GB 虚拟内存，实际可能只用 10MB
  - 页表（Page Table）做虚拟地址 → 物理地址映射
  - 缺页时按需分配物理页
  - 不连续物理页通过页表组装成连续逻辑空间

PagedAttention：
  - 服务为每个请求预留 4096 token 的 KV Cache 槽位
  - 按需分配"KV 块"（类比物理页）
  - 不连续 KV 块通过块表映射到连续的逻辑序列
  - 零浪费：只分配实际用到的 KV 缓存空间
```

### 2.2 KV Block 的设计

PagedAttention 把 KV Cache 按固定大小的 **Block** 组织：

```python
class KVCacheBlock:
    """一个 KV Cache 块，类比操作系统的物理页"""
    def __init__(self, block_size: int = 16):
        # 固定大小的块，比如 16 个 token
        self.block_size = block_size
        self.num_layers = 32
        self.num_heads = 8
        self.head_dim = 128

        # 物理存储
        self.k = torch.zeros(
            (self.num_layers, self.block_size, self.num_heads, self.head_dim),
            dtype=torch.float16
        )
        self.v = torch.zeros(
            (self.num_layers, self.block_size, self.num_heads, self.head_dim),
            dtype=torch.float16
        )
        self.ref_count = 0  # 引用计数，用于内存回收

class BlockTable:
    """块表，将逻辑 token 位置映射到物理 KV Block"""
    def __init__(self):
        self.table: list[int | None] = []  # [block_3, block_7, None, None, ...]

    def append_block(self, physical_block_id: int):
        self.table.append(physical_block_id)

    def get_physical_blocks(self) -> list[int]:
        return [b for b in self.table if b is not None]
```

### 2.3 按需分配的内存策略

```python
class BlockAllocator:
    """KV Block 分配器——类比操作系统的物理页分配器"""
    def __init__(self, num_blocks: int):
        self.free_blocks: list[int] = list(range(num_blocks))
        self.used_blocks: dict[int, KVCacheBlock] = {}

    def allocate(self) -> KVCacheBlock | None:
        """分配一个新的 KV Block"""
        if not self.free_blocks:
            return None  # 触发内存回收（preempt）
        block_id = self.free_blocks.pop()
        block = KVCacheBlock(block_size=16)
        self.used_blocks[block_id] = block
        return block

    def free(self, block_id: int):
        """释放 Block"""
        self.used_blocks.pop(block_id, None)
        self.free_blocks.append(block_id)
```

### 2.4 实际效果对比

```
传统方案（预留最大长度）：
  请求 A（200 token）→ 预留 512MB → 实际使用 25MB → 浪费 487MB
  请求 B（50 token） → 预留 512MB → 实际使用 6MB  → 浪费 506MB
  ─────────────────────────────────────────────────────────────
  总计：8 个请求占满 8GB → 实际利用率 < 5%

PagedAttention 方案（按需分配）：
  请求 A（200 token）→ 分配 13 个 Block（208 token）→ 使用 ~26MB
  请求 B（50 token） → 分配 4 个 Block（64 token）   → 使用 ~8MB
  ─────────────────────────────────────────────────────────────
  总计：256 个请求才占满 8GB → 利用率 > 95%
```

**并发量直接提升 30 倍以上。**

---

## 三、Continuous Batching：GPU 不在等待中空闲

### 3.1 传统静态批处理的尴尬

传统的推理服务（如 TGI v0.x）使用 **Static Batching**：

```
时间线（Static Batching）:
Batch 1: [请求A ████████████████████ 完成_len=200]
         [请求B ████████ 完成_len=50]
         [请求C ██████████████████████████████ 完成_len=300]
         ↓ Batch 1 必须等最慢的请求C完成
         ↓ GPU 50% 的时间在空等
Batch 2: [请求D ████████████...]
```

**问题**：batch 中有一个慢请求就会拖累整个 batch，GPU 空闲等待。

### 3.2 Continuous Batching 的工作方式

vLLM 在每个推理步骤（iteration）都动态调整 batch 组成：

```
时间线（Continuous Batching）:
Iter 1: [A_0] [B_0] [C_0]        ← 3 个新请求加入
Iter 2: [A_1] [B_1] [C_1]
Iter 3: [A_2] [B_2] [C_2]
...
Iter 50: [A_49] [B_49 → DONE!] [C_49]
         ↓ B 完成，立刻踢出 batch
Iter 51: [A_50] [C_50] [D_0]     ← D 新请求立刻加入
         ↓ GPU 完全没有空闲
...
Iter 200: [A_199 → DONE!] [C_199] [D_149] [E_0]
Iter 201: [C_200] [D_150] [E_1]  ← A 完成，E 加入
...
Iter 300: [C_299 → DONE!] [D_249] [E_100] [F_50]
```

核心逻辑：

```python
class Scheduler:
    def schedule(self) -> tuple[list[Sequence], list[Sequence]]:
        """
        每个 iteration 动态决定：
          1. 哪些请求继续推理（running）
          2. 哪些新请求加入（waiting → running）
          3. 哪些完成的请求移除
        """
        running = self.running_sequences
        waiting = self.waiting_sequences

        # 踢出已完成的请求
        running = [s for s in running if not s.is_finished()]

        # 把等待队列中的请求尽可能加入
        while waiting and self._can_schedule(waiting[0]):
            seq = waiting.pop(0)
            self._allocate_blocks(seq)  # PagedAttention 按需分配 Block
            running.append(seq)

        return running, waiting

    def _can_schedule(self, seq: Sequence) -> bool:
        """检查 GPU 内存是否还能容纳新请求"""
        required_blocks = self._estimate_blocks(seq)
        return self.block_allocator.free_count >= required_blocks
```

### 3.3 吞吐量对比

```
Static Batching（传统）:
  平均 batch size: 8
  平均 token 长度: 200
  实际 GPU 利用率: ~45%
  吞吐量: ~15 token/s

Continuous Batching（vLLM）:
  平均 batch size: 32（动态变化）
  平均 token 长度: 200
  实际 GPU 利用率: ~92%
  吞吐量: ~100 token/s
```

---

## 四、Java 后端如何对接 vLLM

### 4.1 vLLM 部署

```bash
# 一行命令启动 vLLM 服务，兼容 OpenAI API
python -m vllm.entrypoints.openai.api_server \
    --model meta-llama/Llama-3-8B-Instruct \
    --max-model-len 4096 \
    --gpu-memory-utilization 0.95 \
    --port 8000
```

### 4.2 Spring AI 集成（兼容 OpenAI 协议）

vLLM 的 API 与 OpenAI 兼容，所以可以直接用任何 OpenAI 客户端：

```java
@Configuration
public class VLLMConfig {

    @Bean
    public OpenAiChatModel vllmChatModel() {
        return OpenAiChatModel.builder()
            .openAiApi(OpenAiApi.builder()
                .baseUrl("http://your-vllm-server:8000/v1")
                .apiKey("EMPTY")  // vLLM 不需要真实 API Key
                .build())
            .build();
    }
}

@RestController
public class VLLMController {

    @Autowired
    private OpenAiChatModel chatModel;

    @GetMapping("/generate")
    public String generate(@RequestParam String prompt) {
        return chatModel.call(prompt);
    }
}
```

### 4.3 用 Java Client 直接调用

```java
// 使用 OkHttp 或 HttpClient
OkHttpClient client = new OkHttpClient();
String json = """
    {
        "model": "meta-llama/Llama-3-8B-Instruct",
        "messages": [{"role": "user", "content": "解释一下PagedAttention"}],
        "max_tokens": 500,
        "temperature": 0.7
    }
    """;
Request request = new Request.Builder()
    .url("http://localhost:8000/v1/chat/completions")
    .post(RequestBody.create(json, MediaType.parse("application/json")))
    .build();
Response response = client.newCall(request).execute();
System.out.println(response.body().string());
```

---

## 总结

vLLM 高性能的三大支柱：

| 技术 | 解决的问题 | 核心做法 |
|------|-----------|---------|
| PagedAttention | KV Cache 内存浪费 | 虚拟内存式的按需分块分配 |
| Continuous Batching | GPU 空闲等待 | 每步动态调整 batch 组成 |
| 显存高效管理 | OOM / 并发低 | 引用计数 + Block 回收 |

**三句话总结**：
1. PagedAttention 让显存利用率从 < 5% 飙到 > 95%，同一张卡能服务 30 倍以上的并发。
2. Continuous Batching 让 GPU 不再等待短序列，利用率从 ~45% 提到 ~92%。
3. 两者结合，推理吞吐量提升 5-10 倍——这就是 vLLM 的魔法。

---

## 下篇预告

有了 Dify 编排应用、Ollama 本地推理、vLLM 高性能部署，我们还需要一个东西来管理复杂的多步骤 AI 流程。下一篇文章我们拆解 **LangGraph 状态机设计**——它是如何让 AI 从"一问一答"进化到"可靠的多步骤工作流"的。关注我，不错过后续内容！

---

*本文基于 vLLM v0.4.x 源码分析。欢迎评论区讨论你的推理部署经验。*
