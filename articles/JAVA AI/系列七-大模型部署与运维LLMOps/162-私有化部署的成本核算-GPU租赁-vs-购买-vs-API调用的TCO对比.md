# 私有化部署的成本核算：GPU 租赁 vs 购买 vs API 调用的 TCO 对比

> 阅读时间：17分钟 | 适合人群：CTO/技术决策者/架构师 | 关键词：TCO、GPU成本、私有化部署、API经济性

---

## 一、老板灵魂拷问："我们为什么要自己买 GPU？OpenAI API 不香吗？"

这是每个推私有化部署的技术负责人都会被问的问题。

"API 按量付费，随用随走。自己买 GPU，一台 H100 就要 25 万，还要配机房、运维、电费——图啥？"

乍一听完全正确。但当我们把账单拉出来算了一笔三年 TCO（Total Cost of Ownership），结论出乎所有人意料。

先说结论（基于 70B 参数模型、日均 500 万 Token 的场景）：

| 方案 | 三年总成本 | 月均成本 | 平均延迟 | 适用场景 |
|------|-----------|---------|----------|----------|
| **API 调用**（GPT-4o） | ¥286 万 | ¥7.9 万 | 1.2s | 低 QPS、波动大 |
| **GPU 租赁**（A100×2 包年） | ¥142 万 | ¥3.9 万 | 0.8s | 中 QPS、1-2 年 |
| **GPU 购买**（A100×4 自建） | ¥163 万 | ¥4.5 万 | 0.5s | 高 QPS、3 年以上 |
| **混合方案**（基座自建+尖峰 API） | ¥118 万 | ¥3.3 万 | 0.7s | **推荐** |

**核心拐点：日调用量超过 200 万 Token 后，自建开始省钱。**

下面我们把账算清楚，每一分钱都有依据。

---

## 二、API 调用的隐形成本：不是只有 Token 计费

### 2.1 OpenAI API 计费模型

以 GPT-4o 为例（2025 年定价）：

```java
// GPT-4o 官方价格（每 100 万 Token）
public class GPT4oPricing {
    public static final double INPUT_PRICE_PER_1M  = 5.00;   // $5/M input tokens
    public static final double OUTPUT_PRICE_PER_1M = 15.00;  // $15/M output tokens
    public static final double CACHED_INPUT_PRICE  = 2.50;   // $2.5/M cached
}
```

但实际成本远不止 Token 计费：

| 隐藏成本项 | 说明 | 月均估算（500 万 Token/日） |
|-----------|------|---------------------------|
| **Token 计费** | 输入+输出 token 直接成本 | ¥6.5 万 |
| **网络出站流量** | API 响应数据量 | ¥0.2 万 |
| **重试/失败重传** | 网络抖动导致的重试 Token | ¥0.5 万 |
| **数据出境合规** | 金融/政务需要专线/VPN | ¥0.5-2 万 |
| **速率限制降级** | 峰值被限流导致的用户体验损失 | 不可量化但真实 |
| **供应商锁定** | 迁移成本（Prompt 适配等） | 一次性 ¥10-20 万 |

### 2.2 速率限制是硬伤

```yaml
# OpenAI Tier 5 最高级别的速率限制
gpt-4o:
  RPM: 10000   # 每分钟请求数
  TPM: 30000000 # 每分钟 Token 数（3000 万）
  
# 听起来很多？高峰期间秒级峰值 500+ QPS 时：
# 500 QPS * 60s = 30000 RPM >> 10000 RPM → 触发限流
```

当你的业务有突发流量（比如教育产品晚上 8 点高峰、电商大促），API 的速率限制会让你眼睁睁看着用户被拒绝。

---

## 三、GPU 租赁方案：AutoDL vs 云厂商竞价实例

### 3.1 主要云厂商 GPU 实例对比

| 云厂商 | GPU 型号 | 显存 | 包月价格 | 竞价实例 | 推荐场景 |
|--------|---------|------|---------|---------|----------|
| **AutoDL** | A100 80G | 80GB | ¥10,800 | ¥4,200 | 灵活测试 |
| **阿里云** | A100 80G | 80GB | ¥15,500 | ¥5,800 | 国内部署 |
| **AWS** | p4d.24xlarge (A100×8) | 640GB | ¥98,000 | ¥32,000 | 海外部署 |
| **腾讯云** | H20 96G | 96GB | ¥12,000 | ¥4,800 | 推理优化 |
| **Lambda Labs** | H100 80G | 80GB | ¥19,800 | - | 训练场景 |

**竞价/Spot 实例注意事项**：
- 价格约为按需的 30-40%
- 随时可能被回收（通常有 2 分钟通知）
- **不适合**需要 7×24 稳定的生产环境
- **适合**开发测试、离线批处理

### 3.2 推理吞吐量实测

我们在一台 A100 80G 上用 vLLM 部署 Llama-3-70B（INT8 量化），实测数据：

```bash
# vLLM 部署命令
python -m vllm.entrypoints.openai.api_server \
  --model /data/models/Llama-3-70B-Instruct-AWQ \
  --quantization awq \
  --dtype auto \
  --max-model-len 8192 \
  --gpu-memory-utilization 0.95 \
  --max-num-seqs 256
```

| 并发数 | 吞吐量 (tokens/s) | 首 Token 延迟 | 每 Token 延迟 | GPU 利用率 |
|--------|-------------------|--------------|--------------|-----------|
| 1 | 45 | 0.8s | 22ms | 35% |
| 8 | 280 | 1.2s | 29ms | 78% |
| 16 | 420 | 2.1s | 38ms | 92% |
| 32 | 520 | 3.5s | 61ms | 98% |
| 64 | 550 | 6.8s | 116ms | 99% |

**单卡 A100 日均处理能力**：以 16 并发稳定运行为例，每小时约 150 万 Token，日均约 **3600 万 Token**。

### 3.3 Java 推理客户端：连接池管理

```java
@Configuration
public class InferenceClientConfig {

    @Bean
    public OkHttpClient inferenceHttpClient() {
        return new OkHttpClient.Builder()
            .connectionPool(new ConnectionPool(
                100,           // 最大空闲连接数
                5, TimeUnit.MINUTES))  // 空闲超时
            .connectTimeout(10, TimeUnit.SECONDS)
            .readTimeout(60, TimeUnit.SECONDS)
            .writeTimeout(60, TimeUnit.SECONDS)
            .addInterceptor(new RetryInterceptor(3))
            .build();
    }
    
    @Bean
    public ExecutorService inferenceThreadPool() {
        return new ThreadPoolExecutor(
            32, 128, 
            60L, TimeUnit.SECONDS,
            new LinkedBlockingQueue<>(1000),
            new ThreadPoolExecutor.CallerRunsPolicy()
        );
    }
}

/**
 * vLLM 兼容的 OpenAI 协议客户端
 */
@Service
public class VLLMInferenceClient {

    @Autowired
    private OkHttpClient httpClient;
    
    @Value("${inference.vllm.endpoint}")
    private String vllmEndpoint;

    public CompletionResponse complete(CompletionRequest request) {
        String json = JSON.toJSONString(request);
        
        Request httpRequest = new Request.Builder()
            .url(vllmEndpoint + "/v1/chat/completions")
            .post(RequestBody.create(json, MediaType.parse("application/json")))
            .addHeader("Authorization", "Bearer " + getInternalToken())
            .build();
        
        try (Response response = httpClient.newCall(httpRequest).execute()) {
            if (!response.isSuccessful()) {
                throw new InferenceException("推理失败: " + response.code());
            }
            return JSON.parseObject(response.body().string(), CompletionResponse.class);
        } catch (IOException e) {
            throw new InferenceException("网络异常", e);
        }
    }
}
```

---

## 四、自购 GPU 方案：买卡还是买服务器？

### 4.1 GPU 硬件成本（2025 年 5 月行情）

| GPU 型号 | 显存 | FP16 算力 | 市场价（人民币） | 70B 模型可行性 |
|----------|------|----------|-----------------|---------------|
| **RTX 4090** | 24GB | 82.6 TFLOPS | ¥13,000 | ❌ 显存不够 |
| **RTX 4090 × 2** | 48GB | 165 TFLOPS | ¥26,000 | ✅ INT4 量化 |
| **A100 80G** | 80GB | 312 TFLOPS | ¥120,000 | ✅ INT8/FP16 |
| **H100 80G** | 80GB | 989 TFLOPS | ¥250,000 | ✅ 推理+训练 |
| **A6000 Ada** | 48GB | 91 TFLOPS | ¥45,000 | ✅ INT4 |
| **H200** | 141GB | 989 TFLOPS | ¥350,000 | ✅ 3 卡跑 70B FP16 |

**单机配置（推理专用）**：

| 硬件项 | 型号 | 数量 | 单价 | 小计 |
|--------|------|------|------|------|
| GPU | A100 80G PCIe | 4 | ¥120,000 | ¥480,000 |
| CPU | AMD EPYC 7543 | 1 | ¥24,000 | ¥24,000 |
| 内存 | DDR4 3200 64GB | 8 | ¥1,600 | ¥12,800 |
| SSD | NVMe 4TB | 2 | ¥3,500 | ¥7,000 |
| 主板 | 服务器主板 | 1 | ¥8,000 | ¥8,000 |
| 电源 | 2000W 冗余 | 2 | ¥3,000 | ¥6,000 |
| 机箱/散热 | 4U 机箱+液冷 | 1 | ¥15,000 | ¥15,000 |
| **硬件合计** | | | | **¥552,800** |

### 4.2 三年 TCO 完整账本

```java
public class ServerTCOCalculator {

    public static void main(String[] args) {
        // ====== 硬件成本 ======
        double hardwareCost = 552_800;  // 4×A100 服务器
        
        // ====== 运维成本（3年） ======
        double colocationCost = 3000 * 36;        // 机房托管 3000/月
        double electricityCost = 2500 * 36;       // 电费 2000W*24h*30d = 1440度/月
        double bandwidthCost = 1500 * 36;         // 带宽 1500/月
        double maintenanceLabor = 8000 * 36;      // 半个人力 8000/月
        double annualOpex = colocationCost + electricityCost + bandwidthCost + maintenanceLabor;
        // 年运维成本 = (3000+2500+1500+8000)*12 = 180,000
        
        // ====== 软件成本（3年） ======
        double softwareCost = 0;  // 开源模型免费
        
        // ====== 三年总成本 ======
        double threeYearTCO = hardwareCost + annualOpex * 3;
        // = 552,800 + 180,000*3 = 552,800 + 540,000 = 1,092,800
        
        System.out.printf("三年总拥有成本: ¥%.2f\n", threeYearTCO);
        System.out.printf("月均成本: ¥%.2f\n", threeYearTCO / 36);
        System.out.printf("日均成本: ¥%.2f\n", threeYearTCO / 1095);
        // 输出: 月均成本 ¥30,356  日均成本 ¥998
        
        // ====== 日均处理能力 ======
        double dailyTokens = 36_000_000;  // 单日 3600 万 Token（4卡并行）
        double costPerMillionTokens = (threeYearTCO / 1095) / (dailyTokens / 1_000_000);
        System.out.printf("每百万 Token 成本: ¥%.2f\n", costPerMillionTokens);
        // 输出: ¥27.72/百万 Token
        
        // ====== 对比 GPT-4o API ======
        double gpt4oCostPerMillion = (5.0 + 15.0) / 2 * 7.2;  // 平均 $10/M, 汇率 7.2
        // = ¥72/百万 Token
        System.out.printf("GPT-4o API: ¥%.2f/百万Token vs 自建: ¥%.2f/百万Token\n", 
            gpt4oCostPerMillion, costPerMillionTokens);
        System.out.printf("自建成本仅为 API 的 %.1f%%\n", 
            costPerMillionTokens / gpt4oCostPerMillion * 100);
        // 输出: 38.5%
    }
}
```

---

## 五、混合方案：最优性价比的"基座+弹性"架构

真实业务不是恒定 QPS。白天的 QPS 是凌晨的 10 倍，大促是平时的 20 倍。

**最优解：自建 GPU 覆盖 80% 的常态流量，API 补充 20% 的峰值。**

```java
@Component
public class HybridInferenceDispatcher {

    @Value("${inference.local.capacity:3600}")  // 本地每秒最大请求数
    private int localCapacity;
    
    @Autowired
    private VLLMInferenceClient localClient;
    
    @Autowired
    private OpenAIClient apiClient;
    
    private final AtomicInteger localCurrentLoad = new AtomicInteger(0);

    /**
     * 混合调度：
     * - 本地容量充足 → 走自建推理
     * - 本地容量满 → 溢出到 OpenAI API
     */
    public CompletionResponse dispatch(CompletionRequest request) {
        int currentLoad = localCurrentLoad.incrementAndGet();
        
        try {
            if (currentLoad <= localCapacity) {
                return localClient.complete(request);
            } else {
                // 溢出到 API
                log.info("本地容量已满 ({}/{}), 溢出到 API", currentLoad, localCapacity);
                metrics.incrementOverflowCount();
                return apiClient.chat(request);
            }
        } finally {
            localCurrentLoad.decrementAndGet();
        }
    }
    
    /**
     * 定时调整本地容量（基于 GPU 利用率）
     */
    @Scheduled(fixedRate = 60000)
    public void adjustLocalCapacity() {
        double gpuUtil = getLocalGPUUtilization();
        int newCapacity;
        
        if (gpuUtil > 95) {
            newCapacity = (int)(localCapacity * 1.1);
        } else if (gpuUtil < 60) {
            newCapacity = (int)(localCapacity * 0.9);
        } else {
            return;
        }
        
        localCapacity = Math.max(10, Math.min(10000, newCapacity));
        log.info("动态调整本地容量: {} (GPU利用率: {:.1f}%)", localCapacity, gpuUtil);
    }
}
```

### 混合方案三年成本

| 方案 | 硬件 | 月常成本 | 三年总成本 | 节省 |
|------|------|---------|-----------|------|
| 纯 API | 无 | ¥7.9 万 | ¥286 万 | - |
| 纯自建 | 4×A100 | ¥4.5 万 | ¥163 万 | 43% |
| **混合** | **2×A100 + API** | **¥3.3 万** | **¥118 万** | **59%** |

2 张 A100 覆盖 80% 流量（¥2.3 万/月），20% 峰值走 API（¥1 万/月），**三年比纯自建再省 ¥45 万**。

---

## 六、决策树：什么时候该自建？

```
你的日 Token 消耗 > 200 万？ ──── 否 ──→ 纯 API 调用，别折腾
        │
       是
        │
  你的 QPS 波动 < 3 倍？ ──── 否 ──→ 混合方案（自建 + API 溢出）
        │
       是
        │
  有合规/数据不出境要求？ ──── 是 ──→ 纯自建，必须的
        │
       否
        │
  预计使用 > 2 年？ ──── 是 ──→ 买 GPU，最划算
        │
       否
        │
  租 GPU，灵活不折腾
```

### 各模型 API vs 自建成本对比（日 500 万 Token）

| 模型 | API 月成本 | 自建月成本 | 节省比例 | 回本周期 |
|------|-----------|-----------|----------|----------|
| GPT-4o | ¥7.9 万 | ¥4.5 万 | 43% | 8 个月 |
| GPT-4o-mini | ¥1.2 万 | ¥1.1 万 | 8% | 19 个月 |
| Claude 3 Opus | ¥14.2 万 | ¥4.5 万 | 68% | 2.5 个月 |
| DeepSeek-V3 | ¥0.5 万 | ¥4.5 万 | -800% | 永远不回本 |

**关键洞察**：便宜的 API（DeepSeek、GPT-4o-mini）自建不划算，贵的 API（Claude Opus、GPT-4-Turbo）自建极划算。

---

## 七、团队实战 checklist

```bash
# 1. 估算日 Token 消耗
# 方法：打开你现有的 API 后台，看最近 30 天的日均 Token 数

# 2. 选卡指南
# < 1000 万 Token/天: 2×RTX 4090（¥26,000）跑量化模型
# 1000-5000 万 Token/天: 2×A100 80G（¥24,000/月租）  
# > 5000 万 Token/天: 4×A100 购买（¥55 万一次性）

# 3. 部署验证
# 先用 AutoDL 租一台 4090 测试，确认性能达标再下单买卡
curl -X POST https://api.autodl.com/v1/instance/create \
  -H "Authorization: Bearer $AUTODL_TOKEN" \
  -d '{"gpu_type": "A100-80G", "duration": "1h", "image": "vllm:latest"}'

# 4. 监控成本
# 参照上篇文章的审计日志，精确记录每 Token 成本
```

---

## 下篇预告

账算清楚了，下一篇落到实操——**GPU 型号到底怎么选？** A100 vs H100 vs RTX4090 在 7B/13B/70B 模型上的实测推理数据，包括吞吐量、首 Token 延迟、并发能力。不要凭感觉买显卡，数据说话。

> 系列七-大模型部署与运维LLMOps / 163-大模型推理的 GPU 选型指南：A100/H100/RTX4090 实测数据
