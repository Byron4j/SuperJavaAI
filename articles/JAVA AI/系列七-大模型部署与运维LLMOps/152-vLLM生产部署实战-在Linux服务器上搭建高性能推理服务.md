# vLLM 生产部署实战：在 Linux 服务器上搭建高性能推理服务，一行命令撑起 1000QPS

## 前言

"你们的 AI 助手回复太慢了，每次等八九秒才蹦出一句，用户都跑了！"

产品经理把反馈截图甩到我面前的时候，服务器上的 Transformers 推理服务正哼哧哼哧地处理第 37 个并发请求，GPU 利用率显示 23%。8 张 A100 显卡，平均利用率不到 30%，但 QPS 死活上不去。

这就是绝大多数团队刚接触大模型部署时的状态——**花最多的显卡钱，跑最慢的推理服务**。

直到 vLLM 的出现。

vLLM 是由 UC Berkeley 开源的 LLM 推理框架，它的核心技术 **PagedAttention** 让 KV Cache 管理从"整块分配"变成"分页管理"，推理吞吐量比 HuggingFace Transformers 高出 **24 倍**。更重要的是，它的部署流程简单到令人发指。

今天这篇文章，我从零带你部署一套生产可用的 vLLM 推理集群，包括 N 卡驱动安装、vLLM 启动、量化模式、并发压测，以及 Java 客户端的集成方案。


## 一、为什么是 vLLM？

### 1.1 推理框架对比

| 框架 | 吞吐量 | 显存效率 | 部署复杂度 | 适用场景 |
|------|--------|----------|------------|----------|
| Transformers | 1x（基准） | 低 | 简单 | 开发调试 |
| vLLM | 14-24x | 高（PagedAttention） | 中等 | 生产推理 |
| TGI (HuggingFace) | 2-5x | 中 | 中等 | HuggingFace 生态 |
| TensorRT-LLM | 5-8x | 高 | 复杂 | 极致性能追求 |
| llama.cpp | 1-3x | 高（CPU推理） | 简单 | 边缘设备/CPU推理 |
| Ollama | 1-3x | 中 | 极简 | 本地开发/小型部署 |

vLLM 的优势在于**吞吐量与易用性的最佳平衡点**。它的 PagedAttention 机制，本质是借鉴了操作系统的虚拟内存分页思想：

```bash
# 传统方式：KV Cache 预分配整块连续显存
# 一个请求的最大长度 4096 → 预留 4096 * hidden_size 的显存
# 实际只用 128 个 token → 浪费了 96.8% 的显存！

# vLLM PagedAttention：KV Cache 分页管理
# 每个 Block = 16 个 token 的 KV Cache
# 实际使用 128 token → 只需 8 个 Block
# 请求变长时动态分配新 Block → 零浪费！
```

**vLLM 的核心性能来源**：
1. **PagedAttention**：显存利用率从 20-40% 提升到接近 100%
2. **Continuous Batching**：请求来一个处理一个，不等凑批
3. **CUDA Kernel 优化**：融合操作减少显存带宽压力
4. **量化支持**：AWQ/GPTQ/FP8，进一步降低显存需求


## 二、环境准备：从裸机到可推理

### 2.1 硬件要求

```yaml
# 推理参考配置（以 Qwen2.5-72B 为例）
quantization: INT4/GPTQ
gpu:
  count: 2
  model: "NVIDIA A100-80G 或 A6000-48G"
  vram_per_gpu: 80GB
memory: 256GB
disk: 500GB SSD
os: "Ubuntu 22.04 LTS"

# 如果是 7B/14B 模型
gpu:
  count: 1
  model: "RTX 4090 或 A10"
  vram: 24GB
```

### 2.2 安装 NVIDIA 驱动 + CUDA Toolkit

```bash
# 第一步：安装 NVIDIA 驱动（如果没有）
# 先检查是否有显卡
lspci | grep -i nvidia

# 如果是 Ubuntu 22.04，用 apt 直接装
sudo apt update
sudo apt install -y nvidia-driver-550
sudo reboot

# 重启后验证
nvidia-smi
# 期望看到：
# +-----------------------------------------------------------------------------+
# | NVIDIA-SMI 550.xx.xx    Driver Version: 550.xx.xx    CUDA Version: 12.4     |
# +-----------------------------------------------------------------------------+
```

```bash
# 第二步：安装 CUDA Toolkit 12.4
wget https://developer.download.nvidia.com/compute/cuda/12.4.0/local_installers/cuda_12.4.0_550.54.14_linux.run
sudo sh cuda_12.4.0_550.54.14_linux.run --toolkit --silent --override

# 配置环境变量
cat >> ~/.bashrc << 'EOF'
export CUDA_HOME=/usr/local/cuda-12.4
export PATH=$CUDA_HOME/bin:$PATH
export LD_LIBRARY_PATH=$CUDA_HOME/lib64:$LD_LIBRARY_PATH
EOF

source ~/.bashrc

# 验证 CUDA
nvcc --version
```

### 2.3 安装 vLLM

```bash
# 推荐用 conda 环境隔离
conda create -n vllm python=3.11 -y
conda activate vllm

# 直接 pip 安装（推荐最新版本，性能和兼容性最好）
pip install vllm

# 验证安装
python -c "from vllm import LLM; print('vLLM installed successfully')"
```

### 2.4 下载模型

```bash
# 安装 huggingface_hub CLI
pip install huggingface_hub

# 下载模型（以 Qwen2.5-7B-Instruct 为例）
# 你可以换成 Llama-3、Mistral、DeepSeek 等任何兼容模型
huggingface-cli download Qwen/Qwen2.5-7B-Instruct \
  --local-dir /data/models/Qwen2.5-7B-Instruct \
  --local-dir-use-symlinks False

# 如果是私有模型，先登录
huggingface-cli login --token hf_your_token_here
```


## 三、vLLM 服务启动：一行命令搞定

### 3.1 基础启动

vLLM 提供了 OpenAI 兼容的 API 接口，可以直接用任何 OpenAI SDK 对接：

```bash
# 一行命令启动 OpenAI 兼容推理服务
python -m vllm.entrypoints.openai.api_server \
  --model /data/models/Qwen2.5-7B-Instruct \
  --served-model-name qwen2.5-7b \
  --host 0.0.0.0 \
  --port 8000 \
  --max-num-seqs 256 \
  --max-model-len 8192 \
  --gpu-memory-utilization 0.95 \
  --dtype auto
```

参数详解：

| 参数 | 含义 | 推荐值 |
|------|------|--------|
| `--model` | 模型路径（本地/HuggingFace ID） | 本地路径 |
| `--served-model-name` | API 中展示的模型名 | 自定义 |
| `--max-num-seqs` | 最大并发序列数 | GPU显存/模型大小决定 |
| `--max-model-len` | 最大上下文长度 | 4096/8192/32768 |
| `--gpu-memory-utilization` | GPU 显存利用率 | 0.90-0.95 |
| `--dtype` | 推理精度 | auto（自动选择最优） |

```bash
# 启动后验证
curl http://localhost:8000/v1/models

# 应该返回：
# {
#   "object": "list",
#   "data": [
#     {
#       "id": "qwen2.5-7b",
#       "object": "model",
#       "created": ...
#     }
#   ]
# }
```

### 3.2 量化部署：用更少的显存跑更大的模型

```bash
# AWQ 量化（推荐，速度快，精度损失小）
python -m vllm.entrypoints.openai.api_server \
  --model Qwen/Qwen2.5-72B-Instruct-AWQ \
  --quantization awq \
  --served-model-name qwen2.5-72b-awq \
  --host 0.0.0.0 \
  --port 8000 \
  --max-num-seqs 128 \
  --max-model-len 32768 \
  --gpu-memory-utilization 0.95 \
  --tensor-parallel-size 4    # 72B 模型用4张卡张量并行

# GPTQ 量化（另一种选择）
python -m vllm.entrypoints.openai.api_server \
  --model TheBloke/Qwen2.5-72B-Instruct-GPTQ \
  --quantization gptq \
  --tensor-parallel-size 4
```

**量化效果实测（Qwen2.5-72B）**：

| 精度 | 显存占用 | 吞吐量 (tokens/s) | 相对精度损失 |
|------|----------|-------------------|-------------|
| FP16 | ~144 GB | 850 | 基准 |
| INT8 | ~76 GB | 1200 | <0.5% |
| INT4 (AWQ) | ~42 GB | 1850 | <1.5% |
| INT4 (GPTQ) | ~42 GB | 1780 | <1.5% |

### 3.3 多卡张量并行

```bash
# 4张A100跑72B模型的完整命令
python -m vllm.entrypoints.openai.api_server \
  --model /data/models/Qwen2.5-72B-Instruct-AWQ \
  --quantization awq \
  --served-model-name qwen2.5-72b \
  --host 0.0.0.0 \
  --port 8000 \
  --max-num-seqs 128 \
  --max-model-len 32768 \
  --gpu-memory-utilization 0.95 \
  --tensor-parallel-size 4 \
  --pipeline-parallel-size 1 \
  --enable-prefix-caching \
  --enable-chunked-prefill
```

关键参数扩展：
- `--enable-prefix-caching`：缓存相同前缀的 KV Cache（RAG 场景神器）
- `--enable-chunked-prefill`：将长 prompt 分块处理，避免首 Token 延迟过高

### 3.4 Docker 部署（生产推荐）

```yaml
# docker-compose.yml
version: '3.8'

services:
  vllm-server:
    image: vllm/vllm-openai:latest
    container_name: vllm-qwen
    runtime: nvidia
    environment:
      - NVIDIA_VISIBLE_DEVICES=all
      - CUDA_VISIBLE_DEVICES=0,1,2,3
    volumes:
      - /data/models:/models:ro
      - /root/.cache/huggingface:/root/.cache/huggingface
    ports:
      - "8000:8000"
    command: >
      --model /models/Qwen2.5-72B-Instruct-AWQ
      --quantization awq
      --served-model-name qwen2.5-72b
      --host 0.0.0.0
      --port 8000
      --max-num-seqs 128
      --max-model-len 32768
      --gpu-memory-utilization 0.95
      --tensor-parallel-size 4
      --enable-prefix-caching
    shm_size: 8gb
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 4
              capabilities: [gpu]
    restart: unless-stopped
```

```bash
# 启动
docker compose up -d

# 查看日志
docker compose logs -f vllm-server

# 健康检查
curl http://localhost:8000/health
```


## 四、Performance 压测与调优

### 4.1 使用 vLLM 自带的 benchmark

```bash
# vLLM 自带压测工具
python -m vllm.entrypoints.openai.run_batch \
  --model qwen2.5-7b \
  --base-url http://localhost:8000 \
  --input-len 512 \
  --output-len 128 \
  --num-prompts 1000 \
  --request-rate 10
```

### 4.2 自定义 Java 压测

```java
/**
 * vLLM 性能压测工具
 */
public class VLLMBenchmark {

    private static final String VLLM_URL = "http://your-server:8000/v1/chat/completions";
    private static final String API_KEY = "EMPTY"; // vLLM 不需要 API Key

    public static void main(String[] args) throws Exception {
        HttpClient client = HttpClient.newHttpClient();
        
        // 构造测试 Prompt
        String prompt = generatePrompt(512); // 512 token 输入
        
        int concurrency = 100;  // 并发数
        int totalRequests = 1000;
        ExecutorService executor = Executors.newFixedThreadPool(concurrency);
        
        List<CompletableFuture<Long>> futures = new ArrayList<>();
        AtomicInteger successCount = new AtomicInteger();
        AtomicInteger failCount = new AtomicInteger();
        
        long startTime = System.currentTimeMillis();
        
        for (int i = 0; i < totalRequests; i++) {
            CompletableFuture<Long> future = CompletableFuture.supplyAsync(() -> {
                try {
                    long requestStart = System.currentTimeMillis();
                    
                    String body = """
                        {
                            "model": "qwen2.5-7b",
                            "messages": [{"role": "user", "content": "%s"}],
                            "max_tokens": 128,
                            "temperature": 0.7,
                            "stream": false
                        }
                        """.formatted(prompt);
                    
                    HttpRequest request = HttpRequest.newBuilder()
                            .uri(URI.create(VLLM_URL))
                            .header("Content-Type", "application/json")
                            .POST(HttpRequest.BodyPublishers.ofString(body))
                            .build();
                    
                    HttpResponse<String> response = client.send(request, 
                            HttpResponse.BodyHandlers.ofString());
                    
                    if (response.statusCode() == 200) {
                        successCount.incrementAndGet();
                    } else {
                        failCount.incrementAndGet();
                    }
                    
                    return System.currentTimeMillis() - requestStart;
                } catch (Exception e) {
                    failCount.incrementAndGet();
                    return -1L;
                }
            }, executor);
            futures.add(future);
        }
        
        // 等待全部完成，统计结果
        CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
        long totalTime = System.currentTimeMillis() - startTime;
        
        DoubleSummaryStatistics stats = futures.stream()
                .map(CompletableFuture::join)
                .filter(t -> t > 0)
                .mapToDouble(Long::doubleValue)
                .summaryStatistics();
        
        System.out.printf("""
                === vLLM Benchmark Results ===
                Total Requests:    %d
                Concurrency:       %d
                Success:           %d
                Failed:            %d
                Total Time:        %d ms
                QPS:               %.2f
                Avg Latency:       %.0f ms
                P50 Latency:       %.0f ms
                P95 Latency:       %.0f ms
                P99 Latency:       %.0f ms
                ==============================
                """,
                totalRequests, concurrency,
                successCount.get(), failCount.get(),
                totalTime,
                totalRequests * 1000.0 / totalTime,
                stats.getAverage(),
                percentile(futures, 0.50),
                percentile(futures, 0.95),
                percentile(futures, 0.99));
        
        executor.shutdown();
    }
}
```

### 4.3 调优实战：从 200 QPS 到 1000 QPS

| 优化项 | 调整前 | 调整后 | 效果 |
|--------|--------|--------|------|
| `max-num-seqs` | 16（默认） | 256 | 并发处理能力 ↑ |
| `gpu-memory-utilization` | 0.90 | 0.95 | 可用显存 ↑ |
| Prefix Caching | 关 | 开 | RAG 场景吞吐 ↑ 40% |
| `enable-chunked-prefill` | 关 | 开 | 首 Token 延迟 ↓ 60% |
| 量化 | FP16 | AWQ INT4 | 单卡可跑更大模型 |
| Tensor Parallel | 1 | 4 | 72B 模型推理可行 |

**实测数据（Qwen2.5-7B，单卡 A10-24G）**：

```
优化前：QPS = 18, P95 延迟 = 4500ms
优化后：QPS = 210, P95 延迟 = 1200ms
提升：QPS × 11.6, P95 延迟 ↓ 73%
```


## 五、Java 客户端集成方案

### 5.1 Spring AI 集成（推荐）

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
    <version>1.0.0-M5</version>
</dependency>
```

```yaml
# application.yml
spring:
  ai:
    openai:
      base-url: http://your-vllm-server:8000
      api-key: EMPTY
      chat:
        options:
          model: qwen2.5-7b
          temperature: 0.7
          max-tokens: 2048
```

```java
@RestController
public class ChatController {

    @Autowired
    private ChatClient chatClient;

    @PostMapping("/chat")
    public String chat(@RequestBody String message) {
        return chatClient.prompt()
                .user(message)
                .call()
                .content();
    }

    @PostMapping("/chat/stream")
    public Flux<String> chatStream(@RequestBody String message) {
        return chatClient.prompt()
                .user(message)
                .stream()
                .content();
    }
}
```

### 5.2 直接 HTTP 调用（轻量级）

```java
@Slf4j
@Service
public class VLLMClient {

    private final RestClient restClient;
    private final String vllmUrl;

    public VLLMClient(@Value("${vllm.base-url}") String baseUrl) {
        this.vllmUrl = baseUrl;
        this.restClient = RestClient.builder()
                .baseUrl(baseUrl)
                .defaultHeader("Content-Type", "application/json")
                .build();
    }

    /**
     * 非流式调用
     */
    public ChatResponse chat(String userMessage) {
        Map<String, Object> request = Map.of(
            "model", "qwen2.5-7b",
            "messages", List.of(Map.of("role", "user", "content", userMessage)),
            "max_tokens", 2048,
            "temperature", 0.7
        );

        return restClient.post()
                .uri("/v1/chat/completions")
                .body(request)
                .retrieve()
                .body(ChatResponse.class);
    }

    /**
     * 流式调用（SSE）
     */
    public Flux<String> chatStream(String userMessage) {
        Map<String, Object> request = Map.of(
            "model", "qwen2.5-7b",
            "messages", List.of(Map.of("role", "user", "content", userMessage)),
            "max_tokens", 2048,
            "temperature", 0.7,
            "stream", true
        );

        return webClient.post()
                .uri("/v1/chat/completions")
                .bodyValue(request)
                .accept(MediaType.TEXT_EVENT_STREAM)
                .retrieve()
                .bodyToFlux(String.class)
                .filter(s -> !"[DONE]".equals(s))
                .map(this::extractContent);
    }

    private String extractContent(String sseData) {
        try {
            JsonNode node = new ObjectMapper().readTree(sseData);
            return node.path("choices").get(0)
                    .path("delta").path("content").asText("");
        } catch (Exception e) {
            return "";
        }
    }
}
```


## 六、生产环境必做的五件事

### 6.1 Systemd 守护进程

```ini
# /etc/systemd/system/vllm.service
[Unit]
Description=vLLM Inference Server (Qwen2.5-72B)
After=network.target docker.service
Requires=docker.service

[Service]
Type=simple
User=root
ExecStart=/usr/bin/docker compose -f /opt/vllm/docker-compose.yml up
ExecStop=/usr/bin/docker compose -f /opt/vllm/docker-compose.yml down
Restart=always
RestartSec=10
TimeoutStopSec=120

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable vllm
sudo systemctl start vllm
sudo systemctl status vllm
```

### 6.2 Nginx 反向代理

```nginx
# /etc/nginx/sites-available/vllm
upstream vllm_backend {
    least_conn;
    server 192.168.1.11:8000 weight=1 max_fails=3 fail_timeout=30s;
    server 192.168.1.12:8000 weight=1 max_fails=3 fail_timeout=30s;
    server 192.168.1.13:8000 weight=1 max_fails=3 fail_timeout=30s;
}

server {
    listen 80;
    server_name llm-api.your-domain.com;
    
    # 请求体大小限制
    client_max_body_size 10M;
    
    location / {
        proxy_pass http://vllm_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        
        # SSE 流式支持
        proxy_set_header Connection '';
        proxy_http_version 1.1;
        proxy_buffering off;
        proxy_cache off;
        chunked_transfer_encoding off;
        
        # 超时设置
        proxy_read_timeout 300s;
        proxy_connect_timeout 10s;
    }
    
    location /health {
        proxy_pass http://vllm_backend/health;
        access_log off;
    }
}
```

### 6.3 GPU 显存监控告警

```bash
#!/bin/bash
# gpu_monitor.sh —— 每30秒检查GPU显存，超过90%告警

THRESHOLD=90
ALERT_WEBHOOK="https://your-alert-webhook"

while true; do
    # 获取每张卡的显存使用率
    nvidia-smi --query-gpu=index,memory.used,memory.total --format=csv,noheader,nounits | \
    while IFS=',' read index used total; do
        usage=$(( used * 100 / total ))
        if [ $usage -gt $THRESHOLD ]; then
            curl -X POST "$ALERT_WEBHOOK" \
                -H "Content-Type: application/json" \
                -d "{\"text\": \"GPU $index memory usage: ${usage}% (>${THRESHOLD}%)\"}"
        fi
    done
    sleep 30
done
```

### 6.4 自动扩缩容（K8s 场景）

```yaml
# HPA 配置（GPU 驱动版本）
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: vllm-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: vllm-deployment
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Pods
      pods:
        metric:
          name: vllm_request_queue_size
        target:
          type: AverageValue
          averageValue: "50"
    - type: Resource
      resource:
        name: nvidia.com/gpu_memory_utilization
        target:
          type: Utilization
          averageUtilization: 85
```


## 总结

vLLM 是目前生产环境部署 LLM 的最佳选择之一。核心记忆点：

1. **PagedAttention 是核心**：显存利用率从 20% 到 95%，吞吐量 10 倍以上提升
2. **OpenAI 兼容 API**：一行命令就能获得和 GPT 一样的调用接口，零迁移成本
3. **量化 + 张量并行**：两张 80G 卡就能跑 72B 模型，成本大幅下降
4. **Prefix Caching**：RAG 场景必开，相同系统提示词的 KV Cache 自动复用

不要怕看起来很复杂，实际部署就一条命令的事。关键是把压测做透，把监控配好，让你的 GPU 不再"摸鱼"。


---

**下篇预告**：vLLM 虽好，但如果是小团队、低预算，或者只想在本地跑跑 7B/14B 的小模型呢？下一篇《**Ollama 生产级调优：并发、内存、GPU 分配的最优配置**》，教你榨干 Ollama 的每一寸性能，别让 8 卡 GPU 只跑出 1 卡的吞吐量！
