# 第 10 章 · KV Cache 优化详解

---

> 这是让 Transformer 推理从"不可用"到"可部署"的最关键优化技术。

---

## 10.1 问题：每次生成都要重算一切

### 10.1.1 自回归生成的浪费

```java
// 生成 "我 爱 你" 的过程
int[] generated = {START_TOKEN};  // ["<s>"]

// Step 1: 输入 ["<s>"]
float[][] output1 = decoder.forward(generated, encoderOutput);
// → 计算了 Q_0, K_0, V_0  ← "<s>" 的 Key 和 Value
int next1 = argmax(output1[0]);  // "我"

generated = {START_TOKEN, "我"};

// Step 2: 输入 ["<s>", "我"]
float[][] output2 = decoder.forward(generated, encoderOutput);
// → 重复计算了 Q_0, K_0, V_0 ← 浪费！和 Step 1 完全一样！
// → 新增计算了 Q_1, K_1, V_1  ← 只需要这个
int next2 = argmax(output2[1]);  // "爱"

generated = {START_TOKEN, "我", "爱"};

// Step 3: 输入 ["<s>", "我", "爱"]
float[][] output3 = decoder.forward(generated, encoderOutput);
// → 又重复计算了 Q_0, K_0, V_0, Q_1, K_1, V_1 ← 又浪费了！
// → 新增计算了 Q_2, K_2, V_2
```

**每次生成新 token 时，之前所有 token 的 K 和 V 都被重新计算！**

生成一个长度为 N 的序列，总计算量 = O(N²·d)（而不是 O(N·d)）。

### 10.1.2 视觉化浪费

```
Step 1: 计算 [K0 V0]                            ← 1 次计算
Step 2: 计算 [K0 V0] + [K1 V1]                   ← 重复了 K0,V0
Step 3: 计算 [K0 V0] + [K1 V1] + [K2 V2]        ← 重复了 K0,V0, K1,V1
Step 4: 计算 [K0 V0] + [K1 V1] + [K2 V2] + ...  ← 重复了更多
...

总计算 ≈ 1 + 2 + 3 + ... + N = N(N+1)/2 = O(N²)
```

---

## 10.2 解决方案：KV Cache

**核心思想**：已经算过的 K 和 V 存起来，下次直接用，只算新的 Q。

```java
/**
 * KV Cache: 用 Map 缓存过去所有 token 的 K 和 V
 * 
 * Java 类比: 
 * - 没有 KV Cache = 每次请求都重新查数据库
 * - 有 KV Cache   = 用 Redis 缓存查询结果，只查增量数据
 */
public class KVCacheDecoder {
    private final TransformerDecoder decoder;

    /**
     * 带 KV Cache 的自回归生成
     */
    public int[] generateWithCache(int[] srcTokens, int maxLen) {
        float[][] encoderOutput = encoder.forward(srcTokens, false);

        // KV Cache: 存储每一层每个 token 的 K 和 V
        // Map<LayerIndex, List<TokenKV>> 
        // 其中 TokenKV = {float[] key, float[] value}
        Map<Integer, List<float[][]>> pastKeyCache = new HashMap<>();  // past K
        Map<Integer, List<float[][]>> pastValueCache = new HashMap<>(); // past V

        int[] generated = {START_TOKEN};
        int pastLen = 0;  // 已缓存的历史 token 数

        while (generated[generated.length - 1] != END_TOKEN && generated.length < maxLen) {
            // 只为最后一个 token 计算 Q, K, V
            int[] newToken = {generated[generated.length - 1]};

            // 带 cache 的 forward：复用过去的 K, V
            float[][] logits = decoder.forwardWithCache(
                newToken,           // 只传新 token
                encoderOutput,
                pastKeyCache,       // 历史 K
                pastValueCache,     // 历史 V
                pastLen             // 历史长度（用于 Positional Encoding）
            );

            // 取最后一个（唯一一个）位置的预测
            float[] probs = softmax(logits[0]);
            int nextToken = argmax(probs);
            generated = append(generated, nextToken);
            pastLen++;
        }
        return generated;
    }
}
```

### 10.2.1 单层 Attention 的 KV Cache 实现

```java
/**
 * 带 KV Cache 的 Multi-Head Attention
 */
public class KVCacheMultiHeadAttention {
    // ... (省略 W_Q, W_K, W_V, W_O)

    /**
     * @param x             当前输入 [1, d_model]  — 只有新 token！
     * @param pastKeys      历史 K [numHeads][pastLen][d_k]
     * @param pastValues    历史 V [numHeads][pastLen][d_v]
     * @param pastLen       历史长度
     * @return 注意力输出 [1, d_v]
     */
    public AttentionOutput forwardWithCache(
            float[][] x,
            float[][][] pastKeys,      // 传入 + 传出（就地更新）
            float[][][] pastValues,
            int pastLen) {

        // 只算新 token 的 Q, K, V
        float[][] Q = matMul(x, W_Q);  // [1, d_k]  ← 只有一行！
        float[][] K = matMul(x, W_K);  // [1, d_k]
        float[][] V = matMul(x, W_V);  // [1, d_v]

        // 把新的 K, V 追加到缓存
        float[][][] fullKeys = appendToCache(pastKeys, K, pastLen);    // [heads][pastLen+1, d_k]
        float[][][] fullValues = appendToCache(pastValues, V, pastLen); // [heads][pastLen+1, d_v]

        int totalLen = pastLen + 1;

        // 用完整的 K, V 计算 Attention
        // Q: [1, d_k] — 只查询新的
        // K: [totalLen, d_k] — 使用全部历史
        // V: [totalLen, d_v]
        float[][] scores = matMul(Q, transpose(fullKeys));  // [1, totalLen]
        float scale = (float) Math.sqrt(d_k);
        rowDivide(scores, scale);
        float[][] weights = rowSoftmax(scores);            // [1, totalLen]
        float[][] output = matMul(weights, fullValues);    // [1, d_v]

        return new AttentionOutput(output, fullKeys, fullValues);
    }

    /**
     * 将新 token 的 K 追加到缓存末尾
     */
    private float[][][] appendToCache(float[][][] cache, float[][] newKV, int pastLen) {
        int numHeads = cache.length;
        int d = newKV[0].length;
        float[][][] updated = new float[numHeads][pastLen + 1][d];

        for (int h = 0; h < numHeads; h++) {
            // 复制历史
            for (int i = 0; i < pastLen; i++) {
                updated[h][i] = cache[h][i];
            }
            // 追加新的
            updated[h][pastLen] = newKV[0];  // 只有 1 个 token
        }
        return updated;
    }
}
```

---

## 10.3 效果对比

```java
// ===== 没有 KV Cache =====
// Step t: 计算 [0..t] 所有 token 的 Q, K, V
//        Attention: Q_{0..t} × K_{0..t}^T → [t+1, t+1]
//        耗时: O(t + t²)
// 总耗时: Σ_t O(t + t²) = O(N²) → O(N³)

// ===== 有 KV Cache =====
// Step t: 只计算新 token 的 Q, K, V
//        K_cache = [K_0, ..., K_t]  ← 历史 K 从缓存读
//        V_cache = [V_0, ..., V_t]  ← 历史 V 从缓存读
//        Attention: Q_t × K_cache^T → [1, t+1]
//        耗时: O(t)  ← 因为 Q 只有 1 行
// 总耗时: Σ_t O(t) = O(N²) ← 降了一个量级！
```

| 场景 | 无 KV Cache | 有 KV Cache | 加速比 |
|---|---|---|---|
| 生成 100 token | O(100³) ≈ 1M | O(100²) ≈ 10K | ~100x |
| 生成 1000 token | O(1000³) ≈ 1B | O(1000²) ≈ 1M | ~1000x |
| 生成 4000 token | O(4000³) ≈ 64B | O(4000²) ≈ 16M | ~4000x |

**没有 KV Cache，GPT-4 推理一次可能需要几百美元；有了 KV Cache，降到几分钱。**

---

## 10.4 KV Cache 的代价：显存

KV Cache 虽然加速，但显存占用会随序列长度**线性增长**：

```java
/**
 * KV Cache 内存计算
 * 
 * 每层: 2 × (K + V) = 2 × (n × d_model × num_heads / heads) ？不对，需要重新算
 * 
 * 实际计算公式:
 * KV_cache_size = 2 (K+V) × num_layers × seq_len × d_model × bytes_per_element
 *                = 2 × N_layers × n × d_model × dtype_bytes
 * 
 * 例子: LLaMA 2 7B (fp16, d_model=4096, N_layers=32)
 * 
 * n=1024:  2 × 32 × 1024 × 4096 × 2 = 537 MB
 * n=2048:  2 × 32 × 2048 × 4096 × 2 = 1.07 GB
 * n=4096:  2 × 32 × 4096 × 4096 × 2 = 2.15 GB
 * n=8192:  2 × 32 × 8192 × 4096 × 2 = 4.29 GB
 * n=32768: 2 × 32 × 32768 × 4096 × 2 = 17.2 GB  ← 已经超过很多 GPU！
 * 
 * 这就是为什么长上下文推理需要 A100 80GB 或多卡部署。
 */
```

---

## 10.5 进阶优化

### 10.5.1 Multi-Query Attention (MQA)

```java
/**
 * MQA: 所有头共享同一个 K 和 V
 * 将 KV Cache 大小减少 h 倍！
 * 
 * 头 0: Q_0, {K_shared, V_shared}
 * 头 1: Q_1, {K_shared, V_shared}
 * ...
 * 头 7: Q_7, {K_shared, V_shared}
 * 
 * Cache 大小从 h × 2 × seqLen × d 降为 2 × seqLen × d
 */
```

### 10.5.2 Grouped-Query Attention (GQA)

```java
/**
 * GQA: 分组共享（折中方案）
 * 
 * 64 个头分成 8 组，每组共享一套 K, V
 * Cache 减少 8 倍，但保留大部分多头表达能力
 * 
 * LLaMA 2 70B, LLaMA 3 使用 GQA
 */
```

### 10.5.3 PagedAttention (vLLM)

```java
/**
 * PagedAttention: 把 KV Cache 管理得像操作系统的虚拟内存
 * 
 * 把 KV Cache 分成固定大小的"页"(page)
 * 不需要连续内存空间，减少碎片
 * 
 * 类比: swap 分区管理
 * - 没有 PagedAttention = 需要连续大块内存
 * - 有 PagedAttention   = 可以东一块西一块，OS 管理映射
 * 
 * vLLM 框架的核心优化，吞吐提升 10-20x
 */
```

### 10.5.4 Flash Attention

```java
/**
 * Flash Attention: 不把完整的注意力矩阵存到显存中
 * 而是分块计算，边算边聚合，减少 HBM (高带宽显存) 的读写
 * 
 * 核心思想: GPU 的瓶颈不在计算，而在内存带宽
 * Flash Attention 将 O(n²) 的内存访问降为 O(n)
 * 
 * 类比: 不用把所有数据 load 到内存再处理，
 *       而是用 stream 逐批读取 + 处理 (map-reduce 模式)
 */
```

---

## 10.6 完整带 Cache 的 Decoder

```java
public class CachedTransformerDecoder {
    private final List<CachedDecoderLayer> layers;

    /**
     * 每步只处理一个新 token
     */
    public float[][] forwardStep(
            int newToken,
            float[][] encoderOutput,
            KVMultiCache cache,  // 携带所有层的历史 K、V
            int stepIndex) {     // 当前在第几步

        // Embedding + Positional Encoding
        float[][] x = embedding.forward(new int[]{newToken});  // [1, d_model]
        float[][] pe = posEncoding.encodeAt(stepIndex);         // 只有第 stepIndex 个位置
        x = add(x, pe);

        // 逐层传递
        for (int l = 0; l < layers.size(); l++) {
            DecoderLayerOutput out = layers.get(l).forwardStep(
                x, encoderOutput,
                cache.getLayerCache(l),  // 该层的历史 KV
                stepIndex
            );
            x = out.output;
        }

        // 输出投影
        return outputProjection.forward(x);  // [1, vocabSize]
    }
}
```

---

> **下一章**：[Spring Boot 集成实战](11-spring-boot-integration.md)
