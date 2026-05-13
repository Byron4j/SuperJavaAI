# AI应用监控深度解析：指标体系、A/B测试与持续优化

**文章标签：** #ai #监控 #可观测性 #llm-ops #ab测试 #模型评估 #持续优化 #mlops

## 目录

- [引言：AI监控的本质与边界](#引言ai监控的本质与边界)
- [理论基础：可观测性工程的AI时代演进](#理论基础可观测性工程的ai时代演进)
- [来龙去脉：从传统监控到ML可观测性](#来龙去脉从传统监控到ml可观测性)
- [核心指标体系深度解析](#核心指标体系深度解析)
- [实战案例：构建完整的AI监控体系](#实战案例构建完整的ai监控体系)
- [对比分析：监控工具与方案选型](#对比分析监控工具与方案选型)
- [性能分析：监控开销与采样策略](#性能分析监控开销与采样策略)
- [常见陷阱与最佳实践](#常见陷阱与最佳实践)
- [面试题与参考答案](#面试题与参考答案)

---

## 引言：AI监控的本质与边界

AI监控不是"看仪表盘"，而是**建立对AI系统行为、性能和风险的完整感知能力**。它回答三个根本问题：模型是否在正常工作？输出是否符合预期？成本是否在可控范围？

核心认知：

```
传统软件监控 vs AI监控：

┌─────────────────┬──────────────────────┬──────────────────────┐
│     维度        │     传统软件          │     AI应用            │
├─────────────────┼──────────────────────┼──────────────────────┤
│ 正确性判断      │ 明确（200 OK / 404）  │ 模糊（输出是否准确）  │
│ 状态确定性      │ 高（确定性的）        │ 低（概率性的）        │
│ 错误模式        │ 异常崩溃、超时        │ 幻觉、偏见、低质量    │
│ 根因分析        │ 代码/配置问题         │ 数据漂移、模型退化    │
│ 性能指标        │ 延迟、错误率          │ + Token成本、生成质量 │
│ 版本管理        │ 代码版本              │ + 模型版本、Prompt版本│
│ 测试方法        │ 单元/集成测试         │ + A/B测试、人工评估   │
└─────────────────┴──────────────────────┴──────────────────────┘

AI监控的独特挑战：
1. 输出质量难以自动化评估
2. 模型行为随数据分布变化
3. 成本和延迟与输入长度强相关
4. 多模型、多版本同时在线
5. 安全性和合规性要求更高
```

**关键洞察**：AI监控的核心不是"收集指标"，而是**建立从数据输入到业务结果的完整因果链**，并能在异常时快速定位根因。

---

## 理论基础：可观测性工程的AI时代演进

### 1. 可观测性的三大支柱

```
可观测性（Observability）三大支柱：

┌─────────────────────────────────────────────────────────┐
│ 支柱1：指标（Metrics）                                    │
│ - 可聚合的数值数据                                        │
│ - 时序数据库（Prometheus/InfluxDB）                       │
│ - 用于：趋势分析、告警、容量规划                           │
│ 示例：延迟P99、Token吞吐量、错误率                         │
├─────────────────────────────────────────────────────────┤
│ 支柱2：日志（Logs）                                       │
│ - 离散的事件记录                                          │
│ - 文本数据，可搜索                                        │
│ - 用于：调试、审计、根因分析                               │
│ 示例：请求日志、模型输入输出、异常堆栈                     │
├─────────────────────────────────────────────────────────┤
│ 支柱3：链路追踪（Traces）                                  │
│ - 请求在系统中的完整路径                                  │
│ - 分布式追踪（OpenTelemetry/Jaeger）                      │
│ - 用于：性能分析、依赖梳理、瓶颈定位                       │
│ 示例：请求从API Gateway → 路由 → 模型服务 → 数据库        │
└─────────────────────────────────────────────────────────┘

AI时代的扩展：
┌─────────────────────────────────────────────────────────┐
│ 支柱4：模型元数据（Model Metadata）                        │
│ - 模型版本、训练数据、超参数                              │
│ - 用于：版本管理、回滚、影响分析                           │
├─────────────────────────────────────────────────────────┤
│ 支柱5：评估指标（Evaluation Metrics）                      │
│ - 任务特定的质量指标                                      │
│ - 用于：模型性能监控、漂移检测                             │
│ 示例：BLEU、ROUGE、准确率、幻觉率                         │
├─────────────────────────────────────────────────────────┤
│ 支柱6：反馈闭环（Feedback Loop）                           │
│ - 用户反馈、业务结果                                      │
│ - 用于：模型优化、A/B测试                                 │
│ 示例：用户点赞/点踩、转化率、留存率                        │
└─────────────────────────────────────────────────────────┘
```

### 2. AI系统的因果模型

```
AI系统的因果链：

输入层 ───────────────────────────────────────────────
│
├─ 数据分布：P(X) 是否变化？                              
├─ 输入质量：是否有噪声、异常值？                         
├─ 输入长度：Token数是否突增？                            
│
↓
模型层 ───────────────────────────────────────────────
│
├─ 模型版本：是否更新？性能变化？                         
├─ 推理延迟：TTFT/TPOT是否正常？                         
├─ Token成本：是否在预算内？                              
├─ 资源使用：GPU/CPU/内存是否健康？                       
│
↓
输出层 ───────────────────────────────────────────────
│
├─ 输出质量：是否准确、相关、安全？                       
├─ 输出格式：是否符合预期（JSON/文本）？                  
├─ 幻觉检测：是否包含编造信息？                           
├─ 偏见检测：是否含有歧视性内容？                         
│
↓
业务层 ───────────────────────────────────────────────
│
├─ 用户满意度：NPS、评分                                  
├─ 业务指标：转化率、留存率、GMV                          
├─ 成本效益：ROI、单次调用成本                            
└─ 合规性：是否符合法规要求                               

监控的目标：建立从输入到业务的完整因果图，
           任何一层异常都能快速定位根因。
```

### 3. 监控的统计学基础

```python
# 监控指标的统计学基础

import numpy as np
from scipy import stats
import pandas as pd

"""
监控中的统计概念：

1. 分位数（Quantile）：
   - P50（中位数）：一半请求比它快
   - P95：95%的请求比它快（排除长尾）
   - P99：99%的请求比它快（极端情况）
   
   为什么不用平均值？
   - 平均值受极端值影响大
   - 长尾延迟被平均掉
   - P95/P99更能反映用户体验

2. 假设检验：
   - 用于A/B测试判断差异是否显著
   - t检验：比较两组均值
   - 卡方检验：比较分类变量
   - Mann-Whitney U：非参数检验

3. 异常检测：
   - 3-sigma原则：偏离均值3个标准差为异常
   - IQR：四分位距的1.5倍为异常
   - Z-score：标准化后的偏离程度
"""

class StatisticalMonitor:
    """统计学监控工具"""
    
    def __init__(self, window_size=1000):
        self.window_size = window_size
        self.data_window = []
    
    def add_measurement(self, value):
        """添加测量值"""
        self.data_window.append(value)
        if len(self.data_window) > self.window_size:
            self.data_window.pop(0)
    
    def get_percentiles(self, percentiles=[50, 95, 99]):
        """计算分位数"""
        if not self.data_window:
            return {}
        
        data = np.array(self.data_window)
        return {
            f"p{p}": np.percentile(data, p)
            for p in percentiles
        }
    
    def detect_anomaly_3sigma(self, value):
        """3-sigma异常检测"""
        if len(self.data_window) < 30:
            return False  # 数据不足
        
        mean = np.mean(self.data_window)
        std = np.std(self.data_window)
        
        if std == 0:
            return False
        
        z_score = abs(value - mean) / std
        return z_score > 3
    
    def detect_anomaly_iqr(self, value):
        """IQR异常检测"""
        if len(self.data_window) < 30:
            return False
        
        q1 = np.percentile(self.data_window, 25)
        q3 = np.percentile(self.data_window, 75)
        iqr = q3 - q1
        
        lower_bound = q1 - 1.5 * iqr
        upper_bound = q3 + 1.5 * iqr
        
        return value < lower_bound or value > upper_bound
    
    def detect_change_point(self, recent_window=100):
        """变点检测（数据分布是否突变）"""
        if len(self.data_window) < recent_window * 2:
            return False
        
        historical = self.data_window[:-recent_window]
        recent = self.data_window[-recent_window:]
        
        # 使用Mann-Whitney U检验
        statistic, p_value = stats.mannwhitneyu(
            historical, recent, alternative='two-sided'
        )
        
        # p值 < 0.05 认为分布发生变化
        return p_value < 0.05
    
    def ab_test(self, group_a, group_b):
        """A/B测试：t检验"""
        # 独立样本t检验
        t_stat, p_value = stats.ttest_ind(group_a, group_b)
        
        # 计算效应量（Cohen's d）
        pooled_std = np.sqrt(
            (np.std(group_a)**2 + np.std(group_b)**2) / 2
        )
        cohens_d = (np.mean(group_a) - np.mean(group_b)) / pooled_std
        
        return {
            "t_statistic": t_stat,
            "p_value": p_value,
            "cohens_d": cohens_d,
            "significant": p_value < 0.05,
            "effect_size": "small" if abs(cohens_d) < 0.5 else 
                          "medium" if abs(cohens_d) < 0.8 else "large",
        }

# 使用示例
monitor = StatisticalMonitor(window_size=1000)

# 模拟延迟数据
for latency in np.random.normal(100, 20, 500):  # 正常数据
    monitor.add_measurement(latency)

# 检测异常
anomaly = monitor.detect_anomaly_3sigma(200)  # 异常值
print(f"Is anomaly: {anomaly}")

# 获取分位数
percentiles = monitor.get_percentiles()
print(f"Percentiles: {percentiles}")

# A/B测试
group_a = np.random.normal(100, 15, 1000)
group_b = np.random.normal(105, 15, 1000)
result = monitor.ab_test(group_a, group_b)
print(f"A/B Test: {result}")
```

---

## 来龙去脉：从传统监控到ML可观测性

### 第一阶段：基础设施监控（2010-2015）

```
基础设施监控时代：

关注点：
┌─────────────────────────────────────────┐
│ - CPU利用率                              │
│ - 内存使用                               │
│ - 磁盘I/O                                │
│ - 网络带宽                               │
│ - 服务可用性                             │
└─────────────────────────────────────────┘

工具栈：
- Nagios（1999）：主机和服务监控
- Zabbix（2001）：企业级监控
- Cacti（2001）：网络图形监控

局限性：
- 只关注基础设施，不关注应用逻辑
- 静态阈值告警，误报率高
- 无法处理动态、概率性系统
```

### 第二阶段：APM与分布式追踪（2015-2020）

```
应用性能监控（APM）时代：

关注点：
┌─────────────────────────────────────────┐
│ - 应用响应时间                           │
│ - 错误率                                 │
│ - 吞吐量（QPS）                          │
│ - 数据库查询性能                         │
│ - 外部API调用                            │
│ - 分布式追踪（请求链路）                  │
└─────────────────────────────────────────┘

工具栈：
- New Relic（2008）：APM先驱
- Datadog（2010）：云原生监控
- AppDynamics（2008）：应用智能
- Zipkin（2012）：分布式追踪
- Jaeger（2016）：Uber开源的追踪系统

关键技术：
- Distributed Tracing（OpenTracing/OpenTelemetry）
- Service Mesh（Istio/Linkerd）自动采集指标
- 动态阈值和基线告警

对AI监控的启示：
- 需要追踪请求在AI pipeline中的完整路径
- 需要关注模型推理的延迟和错误
- 需要理解调用链中的依赖关系
```

### 第三阶段：MLOps与模型监控（2020-2022）

```
MLOps监控时代：

关注点：
┌─────────────────────────────────────────┐
│ - 模型准确率、精确率、召回率             │
│ - 数据漂移（Data Drift）                 │
│ - 概念漂移（Concept Drift）              │
│ - 特征重要性变化                         │
│ - 模型版本管理                           │
│ - A/B测试                                │
└─────────────────────────────────────────┘

工具栈：
- MLflow（2018）：模型生命周期管理
- Kubeflow（2018）：K8s上的ML工作流
- Evidently（2021）：ML模型监控
- Arize（2020）：ML可观测平台
- WhyLabs（2019）：数据质量监控

关键概念：
1. 数据漂移：
   - 输入数据分布发生变化
   - 检测：KS检验、PSI（Population Stability Index）
   - 影响：模型性能下降

2. 概念漂移：
   - 输入到输出的映射关系变化
   - 示例：用户购买行为随季节变化
   - 检测：模型性能监控、标签分布变化

3. 模型退化：
   - 模型性能随时间自然下降
   - 原因：数据分布变化、概念漂移
   - 应对：定期重训练、在线学习

局限性：
- 主要针对传统ML（分类/回归）
- 不适用于LLM的开放生成任务
- 缺乏对Prompt、输出质量的监控
```

### 第四阶段：LLM可观测性（2023-2024）

```
LLM可观测性时代：

新挑战：
┌─────────────────────────────────────────┐
│ 1. 输出质量难以量化                      │
│    - 传统ML有明确标签和指标              │
│    - LLM输出开放，评估困难               │
│                                         │
│ 2. 成本和延迟与输入长度强相关            │
│    - Token计费模式                       │
│    - 长prompt导致高延迟和高成本          │
│                                         │
│ 3. 多模型、多版本同时在线                │
│    - 模型路由、A/B测试                   │
│    - Prompt版本管理                      │
│                                         │
│ 4. 安全性和合规性                        │
│    - 幻觉、偏见、有害内容                │
│    - 数据隐私                            │
│                                         │
│ 5. RAG系统的特殊性                       │
│    - 检索质量影响生成质量                │
│    - 需要监控检索和生成两个阶段          │
└─────────────────────────────────────────┘

新工具：
- LangSmith（2023）：LangChain官方监控
- Langfuse（2023）：LLM应用可观测
- Weights & Biases Prompts（2023）：Prompt管理
- Helicone（2023）：LLM可观测平台
- Phoenix（2023）：Arize的LLM可观测工具
- HoneyHive（2023）：LLM评估和监控

监控维度扩展：
├─ 输入监控：Token数、Prompt模板、用户意图
├─ 模型监控：版本、延迟、成本、错误率
├─ 输出监控：质量、幻觉、偏见、格式
├─ 业务监控：转化率、满意度、留存
└─ 安全监控：毒性、PII泄露、越狱攻击
```

### 第五阶段：2026年现状

```
2026年AI监控的工业标准：

监控体系成熟度模型：

Level 1：基础监控
┌─────────────────────────────────────────┐
│ - 基础设施：CPU/GPU/内存/磁盘           │
│ - 应用层：延迟/错误率/吞吐量            │
│ - 日志：请求/响应记录                   │
│ - 告警：静态阈值                        │
└─────────────────────────────────────────┘

Level 2：AI专项监控
┌─────────────────────────────────────────┐
│ - 模型性能：准确率/F1/困惑度            │
│ - 数据质量：漂移检测/分布变化           │
│ - 特征监控：重要性/相关性               │
│ - 模型版本：实验追踪/模型注册           │
└─────────────────────────────────────────┘

Level 3：LLM可观测性
┌─────────────────────────────────────────┐
│ - Prompt工程：版本/效果/A/B测试         │
│ - 生成质量：幻觉/相关性/流畅度          │
│ - 成本优化：Token消耗/模型路由          │
│ - 安全合规：毒性/偏见/PII检测           │
│ - 用户反馈：点赞/点踩/人工评估          │
└─────────────────────────────────────────┘

Level 4：智能运维
┌─────────────────────────────────────────┐
│ - 自动诊断：根因分析/异常检测           │
│ - 自动优化：模型切换/Prompt调优         │
│ - 预测性维护：容量规划/趋势预测         │
│ - 闭环优化：自动重训练/持续学习         │
└─────────────────────────────────────────┘

主流工具对比（2026）：
┌─────────────────┬──────────┬──────────┬──────────┐
│     工具        │ 开源     │ LLM专项  │ 企业级   │
├─────────────────┼──────────┼──────────┼──────────┤
│ LangSmith       │ 部分     │ ⭐⭐⭐⭐⭐   │ ⭐⭐⭐⭐   │
│ Langfuse        │ ✅       │ ⭐⭐⭐⭐⭐   │ ⭐⭐⭐⭐   │
│ Arize Phoenix   │ ✅       │ ⭐⭐⭐⭐⭐   │ ⭐⭐⭐⭐⭐  │
│ Weights & Biases│ 部分     │ ⭐⭐⭐⭐   │ ⭐⭐⭐⭐⭐  │
│ Evidently       │ ✅       │ ⭐⭐⭐     │ ⭐⭐⭐    │
│ Datadog         │ ❌       │ ⭐⭐⭐     │ ⭐⭐⭐⭐⭐  │
│ Dynatrace       │ ❌       │ ⭐⭐⭐     │ ⭐⭐⭐⭐⭐  │
│ Prometheus+Grafana│ ✅     │ ⭐⭐      │ ⭐⭐⭐    │
└─────────────────┴──────────┴──────────┴──────────┘
```

---

## 核心指标体系深度解析

### 1. 技术指标体系

#### 延迟指标（Latency）

```python
# 延迟指标的深度解析

"""
LLM推理延迟的关键指标：

1. TTFT（Time To First Token）：
   - 定义：从发送请求到收到第一个token的时间
   - 组成：网络传输 + 请求排队 + Prefill计算
   - 重要性：用户感知的首响应时间
   - 目标：
     * 优秀：< 200ms
     * 良好：200-500ms
     * 可接受：500ms-1s
     * 差：> 1s

2. TPOT（Time Per Output Token）：
   - 定义：生成每个后续token的平均时间
   - 组成：Decode计算 + 网络传输
   - 重要性：影响生成流畅度
   - 目标：
     * 优秀：< 20ms/token（50 tokens/s）
     * 良好：20-50ms/token
     * 可接受：50-100ms/token
     * 差：> 100ms/token

3. TBT（Time Between Tokens）：
   - 定义：相邻token之间的时间间隔（流式输出）
   - 重要性：影响用户"打字机"体验
   - 目标：< 100ms（用户感知流畅）

4. 总延迟：
   - Total Latency = TTFT + (N-1) × TPOT
   - N = 输出token数
"""

class LatencyMonitor:
    """延迟监控器"""
    
    def __init__(self):
        self.ttft_measurements = []
        self.tpot_measurements = []
        self.total_measurements = []
    
    def record_request(self, request_start, first_token_time, last_token_time, num_tokens):
        """记录一次请求的延迟数据"""
        ttft = (first_token_time - request_start) * 1000  # ms
        total = (last_token_time - request_start) * 1000  # ms
        tpot = (total - ttft) / (num_tokens - 1) if num_tokens > 1 else 0
        
        self.ttft_measurements.append(ttft)
        self.tpot_measurements.append(tpot)
        self.total_measurements.append(total)
    
    def get_statistics(self):
        """获取延迟统计"""
        import numpy as np
        
        def calc_stats(data, name):
            if not data:
                return {}
            
            arr = np.array(data)
            return {
                f"{name}_p50": np.percentile(arr, 50),
                f"{name}_p95": np.percentile(arr, 95),
                f"{name}_p99": np.percentile(arr, 99),
                f"{name}_mean": np.mean(arr),
                f"{name}_std": np.std(arr),
                f"{name}_min": np.min(arr),
                f"{name}_max": np.max(arr),
            }
        
        stats = {}
        stats.update(calc_stats(self.ttft_measurements, "ttft_ms"))
        stats.update(calc_stats(self.tpot_measurements, "tpot_ms"))
        stats.update(calc_stats(self.total_measurements, "total_ms"))
        
        return stats
    
    def detect_latency_regression(self, window_recent=100, window_historical=1000):
        """检测延迟退化"""
        if len(self.total_measurements) < window_historical + window_recent:
            return False
        
        historical = self.total_measurements[-(window_historical + window_recent):-window_recent]
        recent = self.total_measurements[-window_recent:]
        
        historical_p95 = np.percentile(historical, 95)
        recent_p95 = np.percentile(recent, 95)
        
        # 如果P95延迟增加超过20%，认为退化
        if recent_p95 > historical_p95 * 1.2:
            return {
                "regression_detected": True,
                "historical_p95": historical_p95,
                "recent_p95": recent_p95,
                "increase_pct": (recent_p95 / historical_p95 - 1) * 100,
            }
        
        return {"regression_detected": False}

# 延迟归因分析
def attribute_latency(request):
    """分解延迟到各个环节"""
    
    latency_breakdown = {
        "network_request": request["timestamp_api_received"] - request["timestamp_client_sent"],
        "queue_wait": request["timestamp_processing_start"] - request["timestamp_api_received"],
        "tokenization": request["timestamp_tokenization_end"] - request["timestamp_processing_start"],
        "prefill": request["timestamp_first_token"] - request["timestamp_tokenization_end"],
        "decode": request["timestamp_last_token"] - request["timestamp_first_token"],
        "network_response": request["timestamp_client_received"] - request["timestamp_last_token"],
        "total": request["timestamp_client_received"] - request["timestamp_client_sent"],
    }
    
    return latency_breakdown

"""
延迟优化策略：

TTFT优化：
1. 使用FlashAttention减少Prefill计算时间
2. 减少输入长度（Prompt压缩）
3. 使用更快的GPU（H100 > A100）
4. 预热模型（减少首次加载时间）
5. 优化网络传输（使用HTTP/2、连接池）

TPOT优化：
1. 使用投机采样（Speculative Decoding）
2. 量化减少内存带宽（INT8/FP8）
3. 增大batch size（提高GPU利用率）
4. 使用Continuous Batching
5. KV Cache优化（PagedAttention）
"""
```

#### 成本指标（Cost）

```python
# Token成本监控

"""
LLM成本的关键指标：

1. 输入Token成本：
   - 每1000输入token的价格
   - 示例：GPT-4 $0.03/1K input tokens

2. 输出Token成本：
   - 每1000输出token的价格
   - 示例：GPT-4 $0.06/1K output tokens

3. 单次请求成本：
   - Cost = (input_tokens × input_price + output_tokens × output_price) / 1000

4. 日均/月均成本：
   - 累计所有请求的成本

5. 成本效率：
   - 每美元生成的token数
   - 每美元完成的任务数

6. 成本构成分析：
   - Prompt模板成本（固定）
   - 用户输入成本（可变）
   - 生成输出成本（可变）
   - 上下文历史成本（多轮对话）
"""

class CostMonitor:
    """成本监控器"""
    
    # 模型价格表（每1000 tokens）
    PRICING = {
        "gpt-4": {"input": 0.03, "output": 0.06},
        "gpt-4-turbo": {"input": 0.01, "output": 0.03},
        "gpt-3.5-turbo": {"input": 0.0015, "output": 0.002},
        "claude-3-opus": {"input": 0.015, "output": 0.075},
        "claude-3-sonnet": {"input": 0.003, "output": 0.015},
        "claude-3-haiku": {"input": 0.00025, "output": 0.00125},
        "local-llama-2-7b": {"input": 0.0, "output": 0.0},  # 本地模型
    }
    
    def __init__(self):
        self.daily_costs = {}
        self.request_costs = []
    
    def calculate_cost(self, model, input_tokens, output_tokens):
        """计算单次请求成本"""
        pricing = self.PRICING.get(model, {"input": 0, "output": 0})
        
        input_cost = (input_tokens * pricing["input"]) / 1000
        output_cost = (output_tokens * pricing["output"]) / 1000
        total_cost = input_cost + output_cost
        
        return {
            "input_cost_usd": input_cost,
            "output_cost_usd": output_cost,
            "total_cost_usd": total_cost,
            "input_tokens": input_tokens,
            "output_tokens": output_tokens,
        }
    
    def record_request(self, model, input_tokens, output_tokens, user_id=None):
        """记录请求成本"""
        cost = self.calculate_cost(model, input_tokens, output_tokens)
        cost["timestamp"] = time.time()
        cost["model"] = model
        cost["user_id"] = user_id
        
        self.request_costs.append(cost)
        
        # 更新日成本
        date = datetime.now().strftime("%Y-%m-%d")
        if date not in self.daily_costs:
            self.daily_costs[date] = 0
        self.daily_costs[date] += cost["total_cost_usd"]
        
        return cost
    
    def get_cost_report(self, days=7):
        """生成成本报告"""
        report = {
            "period_days": days,
            "daily_costs": {},
            "model_breakdown": {},
            "user_breakdown": {},
            "total_cost": 0,
        }
        
        cutoff = time.time() - days * 24 * 3600
        recent_requests = [r for r in self.request_costs if r["timestamp"] > cutoff]
        
        for req in recent_requests:
            # 日成本
            date = datetime.fromtimestamp(req["timestamp"]).strftime("%Y-%m-%d")
            report["daily_costs"][date] = report["daily_costs"].get(date, 0) + req["total_cost_usd"]
            
            # 模型成本
            model = req["model"]
            if model not in report["model_breakdown"]:
                report["model_breakdown"][model] = {"cost": 0, "requests": 0, "tokens": 0}
            report["model_breakdown"][model]["cost"] += req["total_cost_usd"]
            report["model_breakdown"][model]["requests"] += 1
            report["model_breakdown"][model]["tokens"] += req["input_tokens"] + req["output_tokens"]
            
            # 用户成本
            user = req.get("user_id", "anonymous")
            report["user_breakdown"][user] = report["user_breakdown"].get(user, 0) + req["total_cost_usd"]
            
            report["total_cost"] += req["total_cost_usd"]
        
        return report
    
    def check_budget_alert(self, daily_budget=100):
        """检查预算告警"""
        today = datetime.now().strftime("%Y-%m-%d")
        today_cost = self.daily_costs.get(today, 0)
        
        if today_cost > daily_budget:
            return {
                "alert": True,
                "message": f"Daily budget exceeded: ${today_cost:.2f} > ${daily_budget}",
                "current_cost": today_cost,
                "budget": daily_budget,
            }
        
        if today_cost > daily_budget * 0.8:
            return {
                "alert": True,
                "message": f"Daily budget at 80%: ${today_cost:.2f}/${daily_budget}",
                "current_cost": today_cost,
                "budget": daily_budget,
            }
        
        return {"alert": False}

# 成本优化策略
"""
成本优化策略：

1. 模型路由：
   - 简单查询 → 便宜模型（GPT-3.5/Claude-Haiku）
   - 复杂查询 → 昂贵模型（GPT-4/Claude-Opus）
   - 节省：50-80%

2. Prompt优化：
   - 减少系统提示长度
   - 使用更简洁的指令
   - 压缩历史对话
   - 节省：20-40%

3. 缓存：
   - 相同Prompt缓存结果
   - 语义相似Prompt复用结果
   - 节省：30-60%

4. 批处理：
   - 合并多个请求
   - 使用批量API
   - 节省：10-20%

5. 本地模型：
   - 简单任务使用本地小模型
   - 复杂任务使用云端大模型
   - 节省：70-90%

6. 输出长度限制：
   - 设置合理的max_tokens
   - 避免无限生成
   - 节省：10-30%
"""
```

#### 质量指标（Quality）

```python
# LLM输出质量评估指标

"""
LLM输出质量的关键指标：

1. 幻觉率（Hallucination Rate）：
   - 定义：输出中包含编造信息的比例
   - 检测方法：
     * 基于事实一致性（Factuality）
     * 基于检索内容对比（RAG场景）
     * 基于NLI（自然语言推理）模型
   - 目标：< 5%（RAG系统）
   - < 10%（开放生成）

2. 相关性（Relevance）：
   - 定义：输出与输入的相关程度
   - 检测方法：
     * Embedding相似度
     * 人工评分
     * LLM-as-a-Judge
   - 目标：> 0.8（Embedding余弦相似度）

3. 流畅度（Fluency）：
   - 定义：输出的语言流畅程度
   - 检测方法：
     * 困惑度（Perplexity）
     * 语法检查
     * 人工评分
   - 目标：低困惑度（相对于基线模型）

4. 安全性（Safety）：
   - 毒性（Toxicity）：是否含有有害内容
   - 偏见（Bias）：是否含有歧视性内容
   - PII泄露：是否泄露个人身份信息
   - 越狱（Jailbreak）：是否被攻击成功

5. 任务完成度（Task Completion）：
   - 定义：是否完成了用户请求的任务
   - 检测方法：
     * 基于规则的检查
     * LLM评估
     * 人工评估
"""

from typing import List, Dict
import numpy as np

class QualityEvaluator:
    """质量评估器"""
    
    def __init__(self, embedding_model=None, nli_model=None):
        self.embedding_model = embedding_model
        self.nli_model = nli_model
    
    def evaluate_hallucination_rag(self, output: str, retrieved_contexts: List[str]) -> Dict:
        """
        评估RAG场景的幻觉率
        
        方法：将输出分解为多个陈述，逐一检查是否有检索内容支持
        """
        # Step 1: 将输出分解为事实陈述
        statements = self._extract_statements(output)
        
        # Step 2: 检查每个陈述是否被检索内容支持
        supported_count = 0
        hallucinated_count = 0
        uncertain_count = 0
        
        for statement in statements:
            # 使用NLI模型检查
            # entailment: 支持, contradiction: 矛盾, neutral: 不确定
            label = self._check_entailment(statement, retrieved_contexts)
            
            if label == "entailment":
                supported_count += 1
            elif label == "contradiction":
                hallucinated_count += 1
            else:
                uncertain_count += 1
        
        total = len(statements)
        
        return {
            "hallucination_rate": hallucinated_count / total if total > 0 else 0,
            "supported_rate": supported_count / total if total > 0 else 0,
            "uncertain_rate": uncertain_count / total if total > 0 else 0,
            "total_statements": total,
            "hallucinated_statements": hallucinated_count,
        }
    
    def evaluate_relevance(self, query: str, output: str) -> float:
        """
        评估输出与查询的相关性
        
        方法：Embedding余弦相似度
        """
        if self.embedding_model is None:
            return 0.5  # 默认值
        
        query_embedding = self.embedding_model.encode(query)
        output_embedding = self.embedding_model.encode(output)
        
        # 计算余弦相似度
        similarity = np.dot(query_embedding, output_embedding) / (
            np.linalg.norm(query_embedding) * np.linalg.norm(output_embedding)
        )
        
        return float(similarity)
    
    def evaluate_fluency(self, output: str, reference_model=None) -> Dict:
        """
        评估流畅度
        
        方法：困惑度（Perplexity）
        """
        if reference_model is None:
            return {"perplexity": None, "score": 0.5}
        
        # 计算困惑度
        # PPL = exp(-sum(log P(w_i)) / N)
        # 低困惑度 = 高流畅度
        
        # 这里使用简化的实现
        tokens = output.split()
        if len(tokens) == 0:
            return {"perplexity": float('inf'), "score": 0}
        
        # 实际实现需要使用语言模型计算概率
        # 这里返回占位符
        perplexity = 10.0  # 示例值
        
        # 转换为0-1分数（越低越好）
        score = max(0, 1 - perplexity / 100)
        
        return {
            "perplexity": perplexity,
            "score": score,
        }
    
    def evaluate_safety(self, output: str) -> Dict:
        """
        评估安全性
        
        检测：毒性、偏见、PII
        """
        results = {
            "toxicity": self._detect_toxicity(output),
            "bias": self._detect_bias(output),
            "pii": self._detect_pii(output),
            "overall_safe": True,
        }
        
        # 如果任何一项不安全，整体不安全
        if results["toxicity"]["is_toxic"] or results["bias"]["has_bias"] or results["pii"]["has_pii"]:
            results["overall_safe"] = False
        
        return results
    
    def _extract_statements(self, text: str) -> List[str]:
        """提取文本中的事实陈述"""
        # 简化实现：按句子分割
        import re
        sentences = re.split(r'[。！？.!?]', text)
        return [s.strip() for s in sentences if len(s.strip()) > 10]
    
    def _check_entailment(self, statement: str, contexts: List[str]) -> str:
        """使用NLI模型检查陈述是否被上下文支持"""
        if self.nli_model is None:
            return "neutral"
        
        # 将陈述与每个上下文组合，检查entailment
        # 简化实现
        for context in contexts:
            if statement.lower() in context.lower():
                return "entailment"
        
        return "neutral"
    
    def _detect_toxicity(self, text: str) -> Dict:
        """检测毒性内容"""
        # 使用毒性检测模型（如Perspective API）
        # 简化实现：关键词检测
        toxic_keywords = ["仇恨", "暴力", "歧视", "侮辱"]
        
        toxicity_score = 0
        for keyword in toxic_keywords:
            if keyword in text:
                toxicity_score += 0.2
        
        return {
            "is_toxic": toxicity_score > 0.5,
            "score": min(toxicity_score, 1.0),
        }
    
    def _detect_bias(self, text: str) -> Dict:
        """检测偏见内容"""
        # 简化实现
        bias_keywords = ["男人都", "女人都", "种族", "歧视"]
        
        bias_score = 0
        for keyword in bias_keywords:
            if keyword in text:
                bias_score += 0.3
        
        return {
            "has_bias": bias_score > 0.5,
            "score": min(bias_score, 1.0),
        }
    
    def _detect_pii(self, text: str) -> Dict:
        """检测PII（个人身份信息）"""
        import re
        
        # 检测手机号
        phone_pattern = r'1[3-9]\d{9}'
        # 检测身份证号
        id_pattern = r'\d{17}[\dXx]|\d{15}'
        # 检测邮箱
        email_pattern = r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b'
        
        pii_found = []
        
        if re.search(phone_pattern, text):
            pii_found.append("phone")
        if re.search(id_pattern, text):
            pii_found.append("id_card")
        if re.search(email_pattern, text):
            pii_found.append("email")
        
        return {
            "has_pii": len(pii_found) > 0,
            "pii_types": pii_found,
        }
    
    def llm_as_judge(self, query: str, output: str, criteria: str) -> Dict:
        """
        使用LLM评估输出质量（LLM-as-a-Judge）
        
        方法：让更强大的模型作为评判者
        """
        judge_prompt = f"""请评估以下AI助手的回答质量。

用户问题：{query}

AI回答：{output}

评估标准：{criteria}

请给出：
1. 评分（1-5分）
2. 优点
3. 缺点
4. 改进建议

请以JSON格式输出。"""
        
        # 调用评判模型
        # response = call_judge_model(judge_prompt)
        
        # 解析评判结果
        # 简化实现
        return {
            "score": 4,
            "strengths": ["回答准确", "结构清晰"],
            "weaknesses": ["可以更详细"],
            "suggestions": ["添加更多例子"],
        }

# 使用示例
evaluator = QualityEvaluator()

# 评估RAG输出
result = evaluator.evaluate_hallucination_rag(
    output="巴黎是法国的首都，埃菲尔铁塔位于巴黎。",
    retrieved_contexts=[
        "巴黎（Paris）是法国的首都和最大城市。",
        "埃菲尔铁塔（Eiffel Tower）位于巴黎第七区。",
    ],
)
print(f"幻觉率: {result['hallucination_rate']:.2%}")

# 评估安全性
safety = evaluator.evaluate_safety("这是一个安全的回答。")
print(f"是否安全: {safety['overall_safe']}")
```

### 2. 业务指标体系

```python
# 业务指标监控

"""
AI应用的业务指标：

1. 用户参与度：
   - DAU/MAU（日活/月活）
   - 会话数/用户
   - 平均会话长度（轮数）
   - 功能使用率

2. 用户满意度：
   - NPS（净推荐值）
   - CSAT（客户满意度评分）
   - 点赞/点踩比例
   - 用户留存率

3. 任务成功率：
   - 任务完成率
   - 重试率（用户重复问同样问题）
   - 转人工率
   - 错误率

4. 业务转化：
   - 转化率（咨询 → 购买）
   - 平均客单价
   - GMV贡献
   - ROI

5. 效率提升：
   - 人工处理时间减少
   - 自动化率
   - 响应时间缩短
"""

class BusinessMetricsTracker:
    """业务指标追踪器"""
    
    def __init__(self):
        self.user_sessions = {}
        self.feedback_records = []
        self.conversion_events = []
    
    def track_session(self, user_id, session_id, event_type, metadata=None):
        """追踪会话事件"""
        if session_id not in self.user_sessions:
            self.user_sessions[session_id] = {
                "user_id": user_id,
                "start_time": time.time(),
                "events": [],
                "message_count": 0,
                "feedback": None,
            }
        
        self.user_sessions[session_id]["events"].append({
            "type": event_type,
            "timestamp": time.time(),
            "metadata": metadata or {},
        })
        
        if event_type == "message":
            self.user_sessions[session_id]["message_count"] += 1
    
    def track_feedback(self, session_id, feedback_type, rating=None, comment=None):
        """追踪用户反馈"""
        self.feedback_records.append({
            "session_id": session_id,
            "type": feedback_type,  # thumbs_up / thumbs_down / rating
            "rating": rating,
            "comment": comment,
            "timestamp": time.time(),
        })
        
        if session_id in self.user_sessions:
            self.user_sessions[session_id]["feedback"] = feedback_type
    
    def track_conversion(self, user_id, session_id, event_type, value=None):
        """追踪转化事件"""
        self.conversion_events.append({
            "user_id": user_id,
            "session_id": session_id,
            "event": event_type,  # purchase / signup / click
            "value": value,
            "timestamp": time.time(),
        })
    
    def calculate_metrics(self, time_window_hours=24) -> Dict:
        """计算业务指标"""
        cutoff = time.time() - time_window_hours * 3600
        
        # 过滤最近的数据
        recent_sessions = {
            sid: s for sid, s in self.user_sessions.items()
            if s["start_time"] > cutoff
        }
        recent_feedback = [f for f in self.feedback_records if f["timestamp"] > cutoff]
        recent_conversions = [e for e in self.conversion_events if e["timestamp"] > cutoff]
        
        metrics = {}
        
        # 1. 用户参与度
        unique_users = len(set(s["user_id"] for s in recent_sessions.values()))
        total_sessions = len(recent_sessions)
        total_messages = sum(s["message_count"] for s in recent_sessions.values())
        
        metrics["unique_users"] = unique_users
        metrics["total_sessions"] = total_sessions
        metrics["total_messages"] = total_messages
        metrics["messages_per_session"] = total_messages / total_sessions if total_sessions > 0 else 0
        
        # 2. 用户满意度
        thumbs_up = sum(1 for f in recent_feedback if f["type"] == "thumbs_up")
        thumbs_down = sum(1 for f in recent_feedback if f["type"] == "thumbs_down")
        total_feedback = thumbs_up + thumbs_down
        
        metrics["thumbs_up_ratio"] = thumbs_up / total_feedback if total_feedback > 0 else 0
        metrics["thumbs_down_ratio"] = thumbs_down / total_feedback if total_feedback > 0 else 0
        
        if recent_feedback:
            ratings = [f["rating"] for f in recent_feedback if f["rating"] is not None]
            metrics["avg_rating"] = sum(ratings) / len(ratings) if ratings else 0
        
        # 3. 任务成功率（基于反馈推断）
        metrics["success_rate"] = metrics["thumbs_up_ratio"]
        
        # 4. 转化率
        purchase_events = [e for e in recent_conversions if e["event"] == "purchase"]
        metrics["conversion_count"] = len(purchase_events)
        metrics["conversion_rate"] = len(purchase_events) / total_sessions if total_sessions > 0 else 0
        metrics["total_revenue"] = sum(e["value"] or 0 for e in purchase_events)
        
        return metrics

# 业务指标告警
def check_business_alerts(metrics):
    """检查业务指标告警"""
    alerts = []
    
    if metrics["thumbs_up_ratio"] < 0.7:
        alerts.append({
            "severity": "warning",
            "metric": "thumbs_up_ratio",
            "value": metrics["thumbs_up_ratio"],
            "threshold": 0.7,
            "message": "User satisfaction is below threshold",
        })
    
    if metrics["conversion_rate"] < 0.05:
        alerts.append({
            "severity": "warning",
            "metric": "conversion_rate",
            "value": metrics["conversion_rate"],
            "threshold": 0.05,
            "message": "Conversion rate is below threshold",
        })
    
    return alerts
```

---

## 实战案例：构建完整的AI监控体系

### 案例1：LLM应用全链路监控

```python
"""
LLM应用全链路监控架构：

┌─────────────────────────────────────────────────────────┐
│  客户端：Web/App/小程序                                   │
│  - 用户行为埋点                                           │
│  - 性能指标采集（首屏时间、交互延迟）                     │
├─────────────────────────────────────────────────────────┤
│  API Gateway：Kong/AWS API Gateway                       │
│  - 请求日志                                               │
│  - 速率限制                                               │
│  - 认证鉴权                                               │
├─────────────────────────────────────────────────────────┤
│  应用服务：FastAPI/Flask                                  │
│  - 业务逻辑                                               │
│  - 输入验证                                               │
│  - 输出后处理                                             │
├─────────────────────────────────────────────────────────┤
│  LLM服务：vLLM/TGI/OpenAI API                             │
│  - 模型推理                                               │
│  - Prompt管理                                             │
│  - 输出格式化                                             │
├─────────────────────────────────────────────────────────┤
│  向量数据库：PGVector/Milvus                              │
│  - RAG检索                                               │
│  - 向量存储                                               │
├─────────────────────────────────────────────────────────┤
│  数据库：PostgreSQL/MongoDB                               │
│  - 对话历史                                               │
│  - 用户数据                                               │
│  - 业务数据                                               │
└─────────────────────────────────────────────────────────┘

监控数据采集点：
1. 客户端：页面加载时间、交互延迟、错误率
2. API Gateway：请求数、延迟、状态码、请求大小
3. 应用服务：业务指标、自定义事件
4. LLM服务：TTFT、TPOT、Token消耗、模型版本
5. 数据库：查询延迟、连接数、错误率
"""

from fastapi import FastAPI, Request, Response
from contextlib import asynccontextmanager
import time
import json
import uuid
from typing import Optional
import structlog

# 配置结构化日志
structlog.configure(
    processors=[
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.JSONRenderer(),
    ],
    wrapper_class=structlog.make_filtering_bound_logger(logging.INFO),
    context_class=dict,
    logger_factory=structlog.PrintLoggerFactory(),
)

logger = structlog.get_logger()

# 监控中间件
class MonitoringMiddleware:
    """监控中间件：采集请求指标"""
    
    def __init__(self, app):
        self.app = app
    
    async def __call__(self, scope, receive, send):
        if scope["type"] != "http":
            await self.app(scope, receive, send)
            return
        
        request_id = str(uuid.uuid4())
        start_time = time.time()
        
        # 采集请求信息
        request_info = {
            "request_id": request_id,
            "method": scope.get("method"),
            "path": scope.get("path"),
            "client_host": scope.get("client", [None])[0],
        }
        
        # 包装send以捕获响应信息
        response_status = None
        response_headers = None
        
        async def wrapped_send(message):
            nonlocal response_status, response_headers
            if message["type"] == "http.response.start":
                response_status = message["status"]
                response_headers = dict(message.get("headers", []))
            await send(message)
        
        try:
            await self.app(scope, receive, wrapped_send)
        except Exception as e:
            logger.error(
                "request_failed",
                request_id=request_id,
                error=str(e),
                **request_info,
            )
            raise
        finally:
            # 计算延迟
            latency_ms = (time.time() - start_time) * 1000
            
            # 记录指标
            logger.info(
                "request_completed",
                request_id=request_id,
                status_code=response_status,
                latency_ms=latency_ms,
                **request_info,
            )
            
            # 发送到指标系统（Prometheus/InfluxDB）
            MetricsCollector.record_http_request(
                method=request_info["method"],
                path=request_info["path"],
                status_code=response_status,
                latency_ms=latency_ms,
            )

# LLM调用监控装饰器
def monitor_llm_call(func):
    """监控LLM调用"""
    async def wrapper(*args, **kwargs):
        call_id = str(uuid.uuid4())
        start_time = time.time()
        
        # 提取监控信息
        model = kwargs.get("model", "unknown")
        prompt_length = len(kwargs.get("messages", []))
        
        try:
            result = await func(*args, **kwargs)
            
            # 计算指标
            latency_ms = (time.time() - start_time) * 1000
            input_tokens = result.get("usage", {}).get("prompt_tokens", 0)
            output_tokens = result.get("usage", {}).get("completion_tokens", 0)
            total_tokens = result.get("usage", {}).get("total_tokens", 0)
            
            # 记录日志
            logger.info(
                "llm_call_completed",
                call_id=call_id,
                model=model,
                latency_ms=latency_ms,
                input_tokens=input_tokens,
                output_tokens=output_tokens,
                total_tokens=total_tokens,
                prompt_length=prompt_length,
            )
            
            # 记录指标
            MetricsCollector.record_llm_call(
                model=model,
                latency_ms=latency_ms,
                input_tokens=input_tokens,
                output_tokens=output_tokens,
                total_tokens=total_tokens,
            )
            
            return result
        
        except Exception as e:
            latency_ms = (time.time() - start_time) * 1000
            
            logger.error(
                "llm_call_failed",
                call_id=call_id,
                model=model,
                latency_ms=latency_ms,
                error=str(e),
                error_type=type(e).__name__,
            )
            
            MetricsCollector.record_llm_error(
                model=model,
                error_type=type(e).__name__,
            )
            
            raise
    
    return wrapper

# 应用监控类
class MetricsCollector:
    """指标采集器"""
    
    _instance = None
    
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance._init()
        return cls._instance
    
    def _init(self):
        self.http_requests_total = {}
        self.http_request_duration = {}
        self.llm_calls_total = {}
        self.llm_tokens_total = {}
        self.llm_errors_total = {}
    
    @classmethod
    def record_http_request(cls, method, path, status_code, latency_ms):
        """记录HTTP请求指标"""
        key = f"{method}_{path}_{status_code}"
        
        if key not in cls._instance.http_requests_total:
            cls._instance.http_requests_total[key] = 0
            cls._instance.http_request_duration[key] = []
        
        cls._instance.http_requests_total[key] += 1
        cls._instance.http_request_duration[key].append(latency_ms)
    
    @classmethod
    def record_llm_call(cls, model, latency_ms, input_tokens, output_tokens, total_tokens):
        """记录LLM调用指标"""
        if model not in cls._instance.llm_calls_total:
            cls._instance.llm_calls_total[model] = 0
            cls._instance.llm_tokens_total[model] = {"input": 0, "output": 0, "total": 0}
        
        cls._instance.llm_calls_total[model] += 1
        cls._instance.llm_tokens_total[model]["input"] += input_tokens
        cls._instance.llm_tokens_total[model]["output"] += output_tokens
        cls._instance.llm_tokens_total[model]["total"] += total_tokens
    
    @classmethod
    def record_llm_error(cls, model, error_type):
        """记录LLM错误"""
        key = f"{model}_{error_type}"
        if key not in cls._instance.llm_errors_total:
            cls._instance.llm_errors_total[key] = 0
        cls._instance.llm_errors_total[key] += 1
    
    @classmethod
    def get_metrics(cls) -> Dict:
        """获取所有指标"""
        return {
            "http_requests": cls._instance.http_requests_total,
            "http_latency": {
                k: {
                    "count": len(v),
                    "p50": sorted(v)[len(v)//2] if v else 0,
                    "p95": sorted(v)[int(len(v)*0.95)] if v else 0,
                    "p99": sorted(v)[int(len(v)*0.99)] if v else 0,
                }
                for k, v in cls._instance.http_request_duration.items()
            },
            "llm_calls": cls._instance.llm_calls_total,
            "llm_tokens": cls._instance.llm_tokens_total,
            "llm_errors": cls._instance.llm_errors_total,
        }

# FastAPI应用
app = FastAPI()

# 添加监控中间件
app.add_middleware(MonitoringMiddleware)

@app.post("/chat")
@monitor_llm_call
async def chat(request: Request):
    """聊天接口"""
    body = await request.json()
    
    # 调用LLM
    response = await call_llm(
        model=body.get("model", "gpt-4"),
        messages=body.get("messages"),
        temperature=body.get("temperature", 0.7),
    )
    
    return response

@app.get("/metrics")
async def metrics():
    """Prometheus指标端点"""
    return MetricsCollector.get_metrics()

@app.get("/health")
async def health():
    """健康检查"""
    return {"status": "healthy"}
```

### 案例2：A/B测试框架实现

```python
"""
A/B测试框架：

目标：
- 科学对比不同模型/Prompt/策略的效果
- 确保统计显著性
- 自动决策最佳方案

架构：
┌─────────────────────────────────────────────────────────┐
│  用户请求                                                │
│     │                                                    │
│     ▼                                                    │
│  分流器（Traffic Splitter）                              │
│  - 根据用户ID哈希或随机分配                               │
│  - 支持多层实验（正交实验）                               │
│     │                                                    │
│     ├─ 50% → 对照组（Control）                           │
│     │        - 模型A                                      │
│     │        - Prompt v1                                  │
│     │                                                    │
│     └─ 50% → 实验组（Treatment）                         │
│              - 模型B                                      │
│              - Prompt v2                                  │
│     │                                                    │
│     ▼                                                    │
│  指标采集                                                │
│  - 技术指标：延迟、Token消耗                             │
│  - 质量指标：幻觉率、相关性                              │
│  - 业务指标：转化率、满意度                              │
│     │                                                    │
│     ▼                                                    │
│  统计分析                                                │
│  - t检验/Mann-Whitney U                                  │
│  - 功效分析（Power Analysis）                            │
│  - 置信区间                                              │
│     │                                                    │
│     ▼                                                    │
│  决策引擎                                                │
│  - 自动判断实验是否显著                                  │
│  - 推荐最佳方案                                          │
│  - 自动扩量或回滚                                        │
└─────────────────────────────────────────────────────────┘
"""

import hashlib
import random
import numpy as np
from scipy import stats
from typing import Dict, List, Optional, Callable
from dataclasses import dataclass
from datetime import datetime, timedelta

@dataclass
class Experiment:
    """实验配置"""
    id: str
    name: str
    control_variant: str
    treatment_variants: List[str]
    traffic_split: Dict[str, float]  # {"control": 0.5, "treatment": 0.5}
    primary_metric: str
    secondary_metrics: List[str]
    min_sample_size: int
    significance_level: float = 0.05
    min_detectable_effect: float = 0.05  # 最小可检测效应

class ABTestFramework:
    """A/B测试框架"""
    
    def __init__(self):
        self.experiments = {}
        self.assignments = {}  # user_id -> {experiment_id: variant}
        self.results = {}  # experiment_id -> {variant: [metrics]}
    
    def create_experiment(self, experiment: Experiment):
        """创建实验"""
        self.experiments[experiment.id] = experiment
        self.results[experiment.id] = {
            variant: []
            for variant in [experiment.control_variant] + experiment.treatment_variants
        }
    
    def assign_variant(self, user_id: str, experiment_id: str) -> str:
        """
        为用户分配实验组
        
        策略：
        1. 一致性：同一用户始终分配到同一组
        2. 随机性：新用户随机分配
        3. 比例：按指定比例分配
        """
        # 检查是否已分配
        if user_id in self.assignments and experiment_id in self.assignments[user_id]:
            return self.assignments[user_id][experiment_id]
        
        experiment = self.experiments.get(experiment_id)
        if not experiment:
            return experiment.control_variant if experiment else "control"
        
        # 确定性哈希（保证一致性）
        hash_input = f"{experiment_id}:{user_id}"
        hash_value = int(hashlib.md5(hash_input.encode()).hexdigest(), 16)
        
        # 根据hash值和分配比例决定分组
        cumulative_prob = 0
        for variant, prob in experiment.traffic_split.items():
            cumulative_prob += prob
            if (hash_value % 10000) / 10000 < cumulative_prob:
                # 记录分配
                if user_id not in self.assignments:
                    self.assignments[user_id] = {}
                self.assignments[user_id][experiment_id] = variant
                return variant
        
        # 默认返回对照组
        return experiment.control_variant
    
    def record_metric(self, experiment_id: str, variant: str, metric_name: str, value: float):
        """记录实验指标"""
        if experiment_id not in self.results:
            return
        
        if variant not in self.results[experiment_id]:
            self.results[experiment_id][variant] = []
        
        self.results[experiment_id][variant].append({
            "metric": metric_name,
            "value": value,
            "timestamp": datetime.now(),
        })
    
    def analyze_experiment(self, experiment_id: str) -> Dict:
        """分析实验结果"""
        experiment = self.experiments.get(experiment_id)
        if not experiment:
            return {"error": "Experiment not found"}
        
        results = self.results.get(experiment_id, {})
        
        # 提取主要指标数据
        control_data = [
            r["value"] for r in results.get(experiment.control_variant, [])
            if r["metric"] == experiment.primary_metric
        ]
        
        analysis = {
            "experiment_id": experiment_id,
            "primary_metric": experiment.primary_metric,
            "variants": {},
        }
        
        # 对照组统计
        if control_data:
            analysis["variants"][experiment.control_variant] = {
                "sample_size": len(control_data),
                "mean": np.mean(control_data),
                "std": np.std(control_data),
                "p50": np.percentile(control_data, 50),
                "p95": np.percentile(control_data, 95),
            }
        
        # 实验组统计和对比
        for variant in experiment.treatment_variants:
            treatment_data = [
                r["value"] for r in results.get(variant, [])
                if r["metric"] == experiment.primary_metric
            ]
            
            if not treatment_data or not control_data:
                continue
            
            # 实验组统计
            variant_stats = {
                "sample_size": len(treatment_data),
                "mean": np.mean(treatment_data),
                "std": np.std(treatment_data),
                "p50": np.percentile(treatment_data, 50),
                "p95": np.percentile(treatment_data, 95),
            }
            
            # 统计检验
            # 使用t检验（假设正态分布）或Mann-Whitney U（非参数）
            if len(control_data) > 30 and len(treatment_data) > 30:
                t_stat, p_value = stats.ttest_ind(control_data, treatment_data)
                test_type = "t_test"
            else:
                t_stat, p_value = stats.mannwhitneyu(
                    control_data, treatment_data, alternative='two-sided'
                )
                test_type = "mann_whitney_u"
            
            # 效应量（Cohen's d）
            pooled_std = np.sqrt(
                (np.std(control_data)**2 + np.std(treatment_data)**2) / 2
            )
            cohens_d = (np.mean(treatment_data) - np.mean(control_data)) / pooled_std if pooled_std > 0 else 0
            
            # 相对提升
            relative_lift = (
                (np.mean(treatment_data) - np.mean(control_data)) / np.mean(control_data)
                if np.mean(control_data) != 0 else 0
            )
            
            # 置信区间（95%）
            se = np.sqrt(
                np.var(control_data)/len(control_data) + 
                np.var(treatment_data)/len(treatment_data)
            )
            diff = np.mean(treatment_data) - np.mean(control_data)
            ci_lower = diff - 1.96 * se
            ci_upper = diff + 1.96 * se
            
            variant_stats["comparison"] = {
                "test_type": test_type,
                "t_statistic": t_stat,
                "p_value": p_value,
                "significant": p_value < experiment.significance_level,
                "cohens_d": cohens_d,
                "relative_lift": relative_lift,
                "ci_95": [ci_lower, ci_upper],
            }
            
            analysis["variants"][variant] = variant_stats
        
        # 样本量检查
        total_samples = sum(
            v["sample_size"] for v in analysis["variants"].values()
        )
        analysis["sample_size_check"] = {
            "total": total_samples,
            "required": experiment.min_sample_size,
            "sufficient": total_samples >= experiment.min_sample_size,
        }
        
        # 实验结论
        analysis["conclusion"] = self._generate_conclusion(analysis, experiment)
        
        return analysis
    
    def _generate_conclusion(self, analysis: Dict, experiment: Experiment) -> str:
        """生成实验结论"""
        if not analysis["sample_size_check"]["sufficient"]:
            return "样本量不足，请继续收集数据"
        
        conclusions = []
        for variant, stats in analysis["variants"].items():
            if variant == experiment.control_variant:
                continue
            
            comparison = stats.get("comparison", {})
            if not comparison:
                continue
            
            if comparison["significant"]:
                direction = "提升" if comparison["relative_lift"] > 0 else "下降"
                conclusions.append(
                    f"{variant}: 相对于对照组{direction} {abs(comparison['relative_lift']):.2%} "
                    f"(p={comparison['p_value']:.4f}, 效应量={comparison['cohens_d']:.2f})"
                )
            else:
                conclusions.append(
                    f"{variant}: 与对照组无显著差异 "
                    f"(p={comparison['p_value']:.4f})"
                )
        
        return "; ".join(conclusions) if conclusions else "暂无结论"
    
    def calculate_required_sample_size(
        self,
        baseline_rate: float,
        min_detectable_effect: float,
        power: float = 0.8,
        alpha: float = 0.05,
    ) -> int:
        """
        计算所需样本量
        
        使用功效分析（Power Analysis）
        """
        from scipy.stats import norm
        
        # 效应量（比例差异）
        p1 = baseline_rate
        p2 = baseline_rate * (1 + min_detectable_effect)
        
        # 合并比例
        p_pooled = (p1 + p2) / 2
        
        # Z值
        z_alpha = norm.ppf(1 - alpha / 2)
        z_beta = norm.ppf(power)
        
        # 样本量公式
        n = (
            (z_alpha * np.sqrt(2 * p_pooled * (1 - p_pooled)) +
             z_beta * np.sqrt(p1 * (1 - p1) + p2 * (1 - p2))) ** 2
        ) / (p2 - p1) ** 2
        
        return int(np.ceil(n))

# 使用示例
ab_test = ABTestFramework()

# 创建实验：对比两个Prompt版本
experiment = Experiment(
    id="prompt_v2_test",
    name="Prompt Version 2 Test",
    control_variant="prompt_v1",
    treatment_variants=["prompt_v2"],
    traffic_split={"prompt_v1": 0.5, "prompt_v2": 0.5},
    primary_metric="user_satisfaction",
    secondary_metrics=["response_length", "latency"],
    min_sample_size=1000,
)

ab_test.create_experiment(experiment)

# 用户请求处理
def handle_request(user_id: str, prompt: str):
    # 分配实验组
    variant = ab_test.assign_variant(user_id, "prompt_v2_test")
    
    # 根据实验组使用不同Prompt
    if variant == "prompt_v2":
        final_prompt = f"[v2] {prompt}"
    else:
        final_prompt = f"[v1] {prompt}"
    
    # 调用LLM
    response = call_llm(final_prompt)
    
    # 记录指标（异步）
    # 假设用户给出了满意度评分
    satisfaction = get_user_feedback(user_id)
    ab_test.record_metric("prompt_v2_test", variant, "user_satisfaction", satisfaction)
    
    return response

# 分析实验结果（每天运行）
analysis = ab_test.analyze_experiment("prompt_v2_test")
print(json.dumps(analysis, indent=2, default=str))

# 计算所需样本量
required_n = ab_test.calculate_required_sample_size(
    baseline_rate=0.75,  # 当前满意度75%
    min_detectable_effect=0.05,  # 期望检测5%的提升
)
print(f"所需样本量: {required_n} per group")
```

### 案例3：自动化监控告警系统

```python
"""
自动化监控告警系统：

功能：
1. 实时指标采集
2. 智能异常检测
3. 多维度告警
4. 自动根因分析
5. 告警收敛和升级

架构：
┌─────────────────────────────────────────────────────────┐
│  指标采集层                                              │
│  - Prometheus Exporter                                   │
│  - 应用日志（结构化）                                    │
│  - 自定义埋点                                            │
├─────────────────────────────────────────────────────────┤
│  指标存储层                                              │
│  - Prometheus（时序数据）                                │
│  - Elasticsearch（日志）                                 │
│  - ClickHouse（事件数据）                                │
├─────────────────────────────────────────────────────────┤
│  告警引擎层                                              │
│  - 规则引擎（静态阈值）                                  │
│  - ML模型（异常检测）                                    │
│  - 模式匹配（日志告警）                                  │
├─────────────────────────────────────────────────────────┤
│  告警处理层                                              │
│  - 告警收敛（去重、聚合）                                │
│  - 告警分级（P0/P1/P2/P3）                               │
│  - 自动恢复检测                                          │
│  - 升级策略（超时未处理升级）                            │
├─────────────────────────────────────────────────────────┤
│  通知渠道层                                              │
│  - 企业微信/钉钉/飞书                                    │
│  - PagerDuty/OpsGenie                                    │
│  - 邮件/SMS                                              │
│  - Webhook                                               │
└─────────────────────────────────────────────────────────┘
"""

from enum import Enum
from typing import List, Dict, Optional
import json
import requests
from datetime import datetime, timedelta

class AlertSeverity(Enum):
    """告警级别"""
    P0 = "critical"      # 严重：服务不可用
    P1 = "high"          # 高：核心功能受损
    P2 = "medium"        # 中：部分功能受影响
    P3 = "low"           # 低：警告，需关注

class AlertChannel(Enum):
    """告警渠道"""
    WECHAT = "wechat"
    DINGTALK = "dingtalk"
    PAGERDUTY = "pagerduty"
    EMAIL = "email"
    WEBHOOK = "webhook"

@dataclass
class AlertRule:
    """告警规则"""
    id: str
    name: str
    metric: str
    condition: str  # >, <, ==, !=
    threshold: float
    duration: int  # 持续多长时间触发（秒）
    severity: AlertSeverity
    channels: List[AlertChannel]
    description: str
    auto_resolve: bool = True

@dataclass
class Alert:
    """告警实例"""
    id: str
    rule_id: str
    severity: AlertSeverity
    metric: str
    current_value: float
    threshold: float
    message: str
    timestamp: datetime
    status: str  # firing / resolved
    labels: Dict[str, str]

class AlertManager:
    """告警管理器"""
    
    def __init__(self):
        self.rules = []
        self.active_alerts = {}  # alert_id -> Alert
        self.alert_history = []
        self.notification_handlers = {
            AlertChannel.WECHAT: self._send_wechat,
            AlertChannel.DINGTALK: self._send_dingtalk,
            AlertChannel.PAGERDUTY: self._send_pagerduty,
            AlertChannel.EMAIL: self._send_email,
            AlertChannel.WEBHOOK: self._send_webhook,
        }
    
    def add_rule(self, rule: AlertRule):
        """添加告警规则"""
        self.rules.append(rule)
    
    def evaluate_rules(self, metrics: Dict[str, float]):
        """评估所有规则"""
        for rule in self.rules:
            if rule.metric not in metrics:
                continue
            
            current_value = metrics[rule.metric]
            
            # 检查条件
            triggered = self._check_condition(
                current_value, rule.condition, rule.threshold
            )
            
            if triggered:
                self._trigger_alert(rule, current_value, metrics)
            else:
                self._resolve_alert(rule, metrics)
    
    def _check_condition(self, value: float, condition: str, threshold: float) -> bool:
        """检查条件"""
        if condition == ">":
            return value > threshold
        elif condition == ">=":
            return value >= threshold
        elif condition == "<":
            return value < threshold
        elif condition == "<=":
            return value <= threshold
        elif condition == "==":
            return value == threshold
        elif condition == "!=":
            return value != threshold
        return False
    
    def _trigger_alert(self, rule: AlertRule, current_value: float, labels: Dict):
        """触发告警"""
        alert_id = f"{rule.id}:{json.dumps(labels, sort_keys=True)}"
        
        # 检查是否已存在
        if alert_id in self.active_alerts:
            # 更新告警信息
            self.active_alerts[alert_id].current_value = current_value
            return
        
        # 创建新告警
        alert = Alert(
            id=alert_id,
            rule_id=rule.id,
            severity=rule.severity,
            metric=rule.metric,
            current_value=current_value,
            threshold=rule.threshold,
            message=f"{rule.name}: {rule.metric} = {current_value} {rule.condition} {rule.threshold}",
            timestamp=datetime.now(),
            status="firing",
            labels=labels,
        )
        
        self.active_alerts[alert_id] = alert
        self.alert_history.append(alert)
        
        # 发送通知
        self._send_notification(alert, rule.channels)
    
    def _resolve_alert(self, rule: AlertRule, labels: Dict):
        """恢复告警"""
        if not rule.auto_resolve:
            return
        
        alert_id = f"{rule.id}:{json.dumps(labels, sort_keys=True)}"
        
        if alert_id in self.active_alerts:
            alert = self.active_alerts[alert_id]
            alert.status = "resolved"
            alert.current_value = None
            
            del self.active_alerts[alert_id]
            
            # 发送恢复通知
            self._send_notification(alert, rule.channels, resolved=True)
    
    def _send_notification(self, alert: Alert, channels: List[AlertChannel], resolved: bool = False):
        """发送告警通知"""
        status = "已恢复" if resolved else "触发"
        
        message = f"""
【{alert.severity.value}】告警{status}

规则: {alert.rule_id}
指标: {alert.metric}
当前值: {alert.current_value}
阈值: {alert.threshold}
时间: {alert.timestamp}

{alert.message}
"""
        
        for channel in channels:
            handler = self.notification_handlers.get(channel)
            if handler:
                try:
                    handler(alert, message)
                except Exception as e:
                    print(f"Failed to send notification via {channel}: {e}")
    
    def _send_wechat(self, alert: Alert, message: str):
        """发送企业微信通知"""
        webhook_url = "https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY"
        payload = {
            "msgtype": "text",
            "text": {
                "content": message,
            },
        }
        requests.post(webhook_url, json=payload)
    
    def _send_dingtalk(self, alert: Alert, message: str):
        """发送钉钉通知"""
        webhook_url = "https://oapi.dingtalk.com/robot/send?access_token=YOUR_TOKEN"
        payload = {
            "msgtype": "text",
            "text": {
                "content": message,
            },
        }
        requests.post(webhook_url, json=payload)
    
    def _send_pagerduty(self, alert: Alert, message: str):
        """发送PagerDuty通知"""
        # PagerDuty集成代码
        pass
    
    def _send_email(self, alert: Alert, message: str):
        """发送邮件通知"""
        # 邮件发送代码
        pass
    
    def _send_webhook(self, alert: Alert, message: str):
        """发送Webhook通知"""
        webhook_url = "https://your-webhook-endpoint.com/alerts"
        payload = {
            "alert_id": alert.id,
            "severity": alert.severity.value,
            "message": message,
            "timestamp": alert.timestamp.isoformat(),
            "status": alert.status,
        }
        requests.post(webhook_url, json=payload)
    
    def get_active_alerts(self) -> List[Alert]:
        """获取当前活跃告警"""
        return list(self.active_alerts.values())
    
    def get_alert_history(self, hours: int = 24) -> List[Alert]:
        """获取告警历史"""
        cutoff = datetime.now() - timedelta(hours=hours)
        return [a for a in self.alert_history if a.timestamp > cutoff]

# 使用示例
alert_manager = AlertManager()

# 添加告警规则
alert_manager.add_rule(AlertRule(
    id="high_latency",
    name="LLM延迟过高",
    metric="llm_latency_p99",
    condition=">",
    threshold=5000,  # 5秒
    duration=300,    # 持续5分钟
    severity=AlertSeverity.P1,
    channels=[AlertChannel.WECHAT, AlertChannel.PAGERDUTY],
    description="LLM P99延迟超过5秒",
))

alert_manager.add_rule(AlertRule(
    id="high_error_rate",
    name="错误率过高",
    metric="error_rate",
    condition=">",
    threshold=0.05,  # 5%
    duration=60,
    severity=AlertSeverity.P0,
    channels=[AlertChannel.DINGTALK, AlertChannel.PAGERDUTY],
    description="错误率超过5%",
))

alert_manager.add_rule(AlertRule(
    id="cost_spike",
    name="成本突增",
    metric="daily_cost_usd",
    condition=">",
    threshold=1000,  # $1000/天
    duration=3600,
    severity=AlertSeverity.P2,
    channels=[AlertChannel.WECHAT],
    description="日成本超过$1000",
))

# 在监控循环中评估规则
metrics = {
    "llm_latency_p99": 6000,  # 6秒，触发告警
    "error_rate": 0.02,        # 2%，正常
    "daily_cost_usd": 1200,    # $1200，触发告警
}

alert_manager.evaluate_rules(metrics)

# 查看活跃告警
active_alerts = alert_manager.get_active_alerts()
for alert in active_alerts:
    print(f"[{alert.severity.value}] {alert.message}")
```

---

## 对比分析：监控工具与方案选型

### 1. 工具对比矩阵

```
┌─────────────────┬──────────┬──────────┬──────────┬──────────┐
│     工具        │ 开源     │ AI专项   │ 可观测性 │ 企业级   │
├─────────────────┼──────────┼──────────┼──────────┼──────────┤
│ Prometheus+Grafana│ ✅      │ ❌       │ ⭐⭐⭐    │ ⭐⭐⭐    │
│ Datadog         │ ❌       │ ⭐⭐⭐    │ ⭐⭐⭐⭐⭐  │ ⭐⭐⭐⭐⭐  │
│ New Relic       │ ❌       │ ⭐⭐⭐    │ ⭐⭐⭐⭐⭐  │ ⭐⭐⭐⭐⭐  │
│ LangSmith       │ 部分     │ ⭐⭐⭐⭐⭐  │ ⭐⭐⭐⭐   │ ⭐⭐⭐⭐   │
│ Langfuse        │ ✅       │ ⭐⭐⭐⭐⭐  │ ⭐⭐⭐⭐   │ ⭐⭐⭐    │
│ Arize Phoenix   │ ✅       │ ⭐⭐⭐⭐⭐  │ ⭐⭐⭐⭐   │ ⭐⭐⭐⭐   │
│ Weights & Biases│ 部分     │ ⭐⭐⭐⭐   │ ⭐⭐⭐⭐   │ ⭐⭐⭐⭐⭐  │
│ Evidently       │ ✅       │ ⭐⭐⭐⭐   │ ⭐⭐⭐    │ ⭐⭐⭐    │
│ WhyLabs         │ 部分     │ ⭐⭐⭐⭐   │ ⭐⭐⭐    │ ⭐⭐⭐⭐   │
│ Dynatrace       │ ❌       │ ⭐⭐⭐    │ ⭐⭐⭐⭐⭐  │ ⭐⭐⭐⭐⭐  │
│ Splunk          │ ❌       │ ⭐⭐⭐    │ ⭐⭐⭐⭐⭐  │ ⭐⭐⭐⭐⭐  │
│ ELK Stack       │ ✅       │ ⭐⭐      │ ⭐⭐⭐⭐   │ ⭐⭐⭐    │
└─────────────────┴──────────┴──────────┴──────────┴──────────┘
```

### 2. 场景化选型建议

```markdown
## 场景1：初创公司（预算有限）
- 需求：基础监控 + LLM追踪
- 推荐：Langfuse（开源）+ Prometheus/Grafana（开源）
- 成本：免费（自托管）
- 能力：
  - Langfuse：LLM调用追踪、成本分析、反馈收集
  - Prometheus：基础设施和应用指标
  - Grafana：可视化仪表盘

## 场景2：中型公司（需要企业级）
- 需求：全链路可观测 + AI专项
- 推荐：Datadog/New Relic（APM）+ LangSmith（LLM）
- 成本：中等（SaaS订阅）
- 能力：
  - Datadog：基础设施、应用性能、日志、APM
  - LangSmith：Prompt管理、版本控制、A/B测试

## 场景3：大型互联网公司
- 需求：超大规模 + 自定义需求
- 推荐：自研 + 开源组合
- 成本：高（人力+基础设施）
- 架构：
  - 指标：Prometheus + Thanos（长期存储）
  - 日志：ELK Stack / Loki
  - 追踪：Jaeger + OpenTelemetry
  - AI监控：自研（基于Arize Phoenix或Langfuse二次开发）
  - 告警：Alertmanager + PagerDuty

## 场景4：AI原生公司
- 需求：深度LLM可观测
- 推荐：Weights & Biases（实验管理）+ LangSmith（生产监控）
- 成本：中等
- 能力：
  - W&B：模型训练追踪、超参数调优、artifact管理
  - LangSmith：生产环境监控、Prompt版本、反馈闭环
```

---

## 性能分析：监控开销与采样策略

### 1. 监控开销分析

```python
# 监控开销分析

"""
监控开销的来源：

1. 指标采集开销：
   - CPU：< 1%（Prometheus pull模式）
   - 内存：取决于指标数量和标签基数
   - 网络：少量（metrics体积小）

2. 日志记录开销：
   - CPU：1-5%（取决于日志量）
   - 磁盘：日志文件大小
   - I/O：异步写入影响小

3. 链路追踪开销：
   - CPU：2-10%（取决于采样率）
   - 内存：存储span信息
   - 网络：传输trace数据

4. LLM监控额外开销：
   - 输入输出记录：增加内存和存储
   - Embedding计算：用于相似度分析
   - 质量评估：调用评估模型

开销优化策略：
1. 采样：只记录部分请求
2. 异步：监控数据异步发送
3. 批量：批量发送指标和日志
4. 压缩：使用压缩算法减少传输
5. 聚合：先聚合再发送
"""

class SamplingStrategy:
    """采样策略"""
    
    def __init__(self, strategy: str = "random", rate: float = 0.1):
        self.strategy = strategy
        self.rate = rate
    
    def should_sample(self, request_context: Dict) -> bool:
        """决定是否采样该请求"""
        
        if self.strategy == "random":
            # 随机采样
            return random.random() < self.rate
        
        elif self.strategy == "error_only":
            # 只采样错误请求
            return request_context.get("is_error", False)
        
        elif self.strategy == "adaptive":
            # 自适应采样：错误率高时增加采样
            error_rate = request_context.get("error_rate", 0)
            adaptive_rate = min(1.0, self.rate * (1 + error_rate * 10))
            return random.random() < adaptive_rate
        
        elif self.strategy == "user_based":
            # 基于用户采样：VIP用户全采样
            user_id = request_context.get("user_id", "")
            is_vip = request_context.get("is_vip", False)
            
            if is_vip:
                return True
            
            # 普通用户按哈希采样（保证同一用户始终采样或不采样）
            hash_value = int(hashlib.md5(user_id.encode()).hexdigest(), 16)
            return (hash_value % 10000) / 10000 < self.rate
        
        elif self.strategy == "tail_based":
            # 尾延迟采样：只采样慢请求
            latency_ms = request_context.get("latency_ms", 0)
            latency_threshold = request_context.get("latency_threshold", 1000)
            
            if latency_ms > latency_threshold:
                return True
            
            return random.random() < self.rate
        
        return False

# 监控开销测量
class OverheadMonitor:
    """监控开销测量器"""
    
    def measure_logging_overhead(self, num_requests: int = 10000):
        """测量日志记录开销"""
        
        # 无日志基线
        start = time.time()
        for i in range(num_requests):
            # 模拟处理
            result = i * 2
        baseline_time = time.time() - start
        
        # 有日志
        start = time.time()
        for i in range(num_requests):
            # 模拟处理
            result = i * 2
            # 记录日志
            logger.info("request_processed", request_id=i, result=result)
        logging_time = time.time() - start
        
        overhead = ((logging_time - baseline_time) / baseline_time) * 100
        
        return {
            "baseline_ms": baseline_time * 1000,
            "with_logging_ms": logging_time * 1000,
            "overhead_pct": overhead,
            "per_request_overhead_us": (logging_time - baseline_time) * 1e6 / num_requests,
        }
    
    def measure_tracing_overhead(self, num_requests: int = 1000):
        """测量链路追踪开销"""
        
        # 无追踪基线
        start = time.time()
        for i in range(num_requests):
            self._simulate_request()
        baseline_time = time.time() - start
        
        # 有追踪
        start = time.time()
        for i in range(num_requests):
            with self._start_span("request"):
                with self._start_span("validation"):
                    pass
                with self._start_span("llm_call"):
                    self._simulate_request()
                with self._start_span("post_processing"):
                    pass
        tracing_time = time.time() - start
        
        overhead = ((tracing_time - baseline_time) / baseline_time) * 100
        
        return {
            "baseline_ms": baseline_time * 1000,
            "with_tracing_ms": tracing_time * 1000,
            "overhead_pct": overhead,
        }
    
    def _simulate_request(self):
        """模拟请求处理"""
        time.sleep(0.001)  # 1ms处理时间
    
    def _start_span(self, name):
        """模拟span创建"""
        # 实际实现使用OpenTelemetry等库
        return self
    
    def __enter__(self):
        return self
    
    def __exit__(self, *args):
        pass

# 开销优化建议
"""
监控开销优化策略：

1. 采样率调优：
   - 开发环境：100%采样
   - 测试环境：50%采样
   - 生产环境：1-10%采样
   - 错误请求：100%采样

2. 异步处理：
   - 指标采集：异步定时任务
   - 日志写入：异步缓冲区
   - 追踪发送：后台线程

3. 批量发送：
   - 指标：每10秒批量push
   - 日志：每100条或5秒批量flush
   - 追踪：每100个span批量发送

4. 数据压缩：
   - 使用gzip压缩传输
   - 减少标签基数（避免高基数字段作为标签）

5. 本地聚合：
   - 客户端先聚合再发送
   - 减少网络传输量

推荐配置（生产环境）：
- 指标采集：1%随机采样 + 100%错误采样
- 日志记录：异步，批量flush（100条/5秒）
- 链路追踪：1%随机采样 + 尾延迟采样（>1s的请求）
- LLM监控：10%采样（记录输入输出）
"""
```

---

## 常见陷阱与最佳实践

### 1. 常见陷阱

```markdown
## 陷阱1：监控指标过多

问题：采集了成百上千个指标，但没人看
后果：存储成本高，噪音大，关键指标被淹没

解决：
- 遵循USE方法（Utilization、Saturation、Errors）
- 遵循RED方法（Rate、Errors、Duration）
- 每个服务定义不超过10个核心指标
- 定期清理无用指标

## 陷阱2：静态阈值告警泛滥

问题：阈值设置不合理，告警风暴
后果：团队对告警麻木，真正的问题被忽略

解决：
- 使用动态基线（基于历史数据）
- 设置合理的告警窗口（持续5分钟才告警）
- 告警分级（P0/P1/P2/P3）
- 告警收敛（相同问题只发一次）

## 陷阱3：只监控技术指标，忽略业务指标

问题：系统正常，但业务效果差
后果：发现问题时已经造成损失

解决：
- 建立从技术指标到业务指标的映射
- 监控转化率、满意度、留存率
- 业务指标异常时触发告警

## 陷阱4：缺乏上下文信息

问题：告警只有"CPU使用率 > 90%"，没有上下文
后果：排查困难，MTTR（平均修复时间）长

解决：
- 告警包含：时间、服务、实例、最近变更、相关日志链接
- 使用Runbook（标准操作程序）
- 建立知识库（常见问题解决方案）

## 陷阱5：监控数据未用于优化

问题：只监控不行动
后果：监控成为摆设

解决：
- 建立监控-告警-排查-优化的闭环
- 定期复盘告警（周会/月会）
- 将监控数据用于容量规划和性能优化

## 陷阱6：忽略AI特有的监控维度

问题：用传统监控思维监控AI应用
后果：遗漏关键问题（幻觉、偏见、Prompt注入）

解决：
- 建立AI专项监控（质量、安全、成本）
- 定期人工评估输出质量
- 监控Prompt版本和效果

## 陷阱7：采样率过低导致遗漏问题

问题：生产环境只采样1%，错误未被采集
后果：问题发现延迟

解决：
- 错误请求100%采样
- 使用自适应采样（错误率高时自动增加采样）
- 关键用户/场景全量采样

## 陷阱8：监控本身成为瓶颈

问题：监控代码消耗大量资源
后果：影响正常服务

解决：
- 异步采集
- 批量发送
- 控制监控代码复杂度
- 定期评估监控开销
```

### 2. 最佳实践清单

```markdown
## 设计阶段

1. **指标设计**
   - 使用SLI（Service Level Indicator）定义服务质量
   - 设置SLO（Service Level Objective）目标
   - 定义SLA（Service Level Agreement）承诺

2. **告警设计**
   - 遵循"5个9"原则：99.999%可用性
   - 告警 = 症状（Symptom）+ 影响（Impact）
   - 避免基于原因的告警（如"CPU高"），改为基于症状的告警（如"响应慢"）

3. **Dashboard设计**
   - 按服务组织Dashboard
   - 核心指标放在顶部
   - 使用颜色编码（绿/黄/红）

## 实施阶段

4. **代码埋点**
   - 使用OpenTelemetry标准
   - 统一的trace_id传递
   - 结构化日志（JSON格式）

5. **监控集成**
   - CI/CD中集成监控配置检查
   - 自动化Dashboard和告警部署
   - 版本控制监控配置

6. **测试验证**
   - 混沌工程：故意制造故障验证监控
   - 告警测试：定期触发测试告警
   - Dashboard演练：模拟故障场景

## 运营阶段

7. **告警响应**
   - 定义清晰的On-call轮值
   - 建立Runbook（标准操作程序）
   - 设置告警升级策略（15分钟未处理升级）

8. **持续优化**
   - 定期Review告警（去除无效告警）
   - 优化阈值（基于历史数据）
   - 完善Dashboard（添加新视角）

9. **知识沉淀**
   - 记录故障Post-mortem
   - 更新Runbook
   - 分享监控最佳实践

10. **安全合规**
    - 监控数据脱敏（隐藏PII）
    - 访问控制（谁可以看什么）
    - 审计日志（谁修改了配置）
    - 数据保留策略（定期清理旧数据）
```

---

## 面试题与参考答案

### 基础题

**Q1：什么是可观测性（Observability）？与监控（Monitoring）有什么区别？**

```markdown
参考答案：

监控（Monitoring）：
- 定义：基于预定义的指标和阈值，检测已知问题
- 关注点：系统是否正常工作？
- 方法：采集指标，设置告警
- 局限：只能发现预期内的问题

可观测性（Observability）：
- 定义：通过系统的外部输出（指标、日志、追踪）理解内部状态
- 关注点：系统为什么不正常工作？
- 方法：三大支柱（Metrics/Logs/Traces）
- 优势：能发现未知问题，支持根因分析

区别：
| 维度 | 监控 | 可观测性 |
|------|------|----------|
| 目标 | 发现问题 | 理解系统 |
| 范围 | 已知问题 | 未知问题 |
| 方法 | 阈值告警 | 探索性分析 |
| 数据 | 预定义指标 | 任意系统输出 |
| 用户 | 运维人员 | 开发+运维+产品 |

关系：
监控是可观测性的子集
可观测性 = 监控 + 日志 + 追踪 + 关联分析
```

**Q2：AI监控与传统软件监控的主要区别是什么？**

```markdown
参考答案：

主要区别：

1. 正确性判断：
   - 传统：明确（HTTP 200 vs 500）
   - AI：模糊（输出是否准确、相关、安全）

2. 状态确定性：
   - 传统：确定性系统（相同输入→相同输出）
   - AI：概率性系统（相同输入→可能不同输出）

3. 性能特征：
   - 传统：延迟相对稳定
   - AI：延迟与输入长度强相关（Token数）

4. 成本模型：
   - 传统：按资源使用计费
   - AI：按Token消耗计费

5. 版本管理：
   - 传统：代码版本
   - AI：代码 + 模型版本 + Prompt版本

6. 测试方法：
   - 传统：单元/集成测试
   - AI：A/B测试、人工评估、自动化评估

7. 安全维度：
   - 传统：SQL注入、XSS等
   - AI：Prompt注入、幻觉、偏见、越狱

AI监控特有维度：
- 输出质量：幻觉率、相关性、流畅度
- 成本：Token消耗、每美元产出
- 安全：毒性、偏见、PII泄露
- 反馈：用户点赞/点踩、人工评估
```

**Q3：如何设计一个LLM应用的监控指标体系？**

```markdown
参考答案：

指标体系分层设计：

Layer 1：基础设施层
- CPU/GPU利用率
- 内存/显存使用
- 磁盘I/O
- 网络带宽

Layer 2：应用服务层
- 请求QPS
- 延迟（P50/P95/P99）
- 错误率（4xx/5xx/超时）
- 并发连接数

Layer 3：模型推理层
- TTFT（首token时间）
- TPOT（每token时间）
- Token吞吐量
- 模型加载时间
- GPU利用率

Layer 4：业务质量层
- 用户满意度（NPS/评分）
- 任务完成率
- 重试率
- 转人工率

Layer 5：成本效益层
- Token消耗量
- API调用成本
- 每美元生成的token数
- ROI

Layer 6：安全合规层
- 幻觉率
- 毒性检测率
- 偏见检测率
- PII泄露次数
- 越狱攻击检测

关键指标示例：
```python
CORE_METRICS = {
    # 技术指标
    "latency_p99": {"threshold": 2000, "unit": "ms"},
    "error_rate": {"threshold": 0.01, "unit": "ratio"},
    "throughput": {"target": 1000, "unit": "tokens/s"},
    
    # 业务指标
    "user_satisfaction": {"target": 0.85, "unit": "ratio"},
    "task_completion_rate": {"target": 0.90, "unit": "ratio"},
    
    # 成本指标
    "cost_per_session": {"budget": 0.5, "unit": "USD"},
    "token_efficiency": {"target": 100, "unit": "tokens/USD"},
    
    # 质量指标
    "hallucination_rate": {"threshold": 0.05, "unit": "ratio"},
    "toxicity_rate": {"threshold": 0.01, "unit": "ratio"},
}
```
```

### 进阶题

**Q4：如何检测LLM输出的幻觉（Hallucination）？**

```markdown
参考答案：

幻觉检测方法：

1. RAG场景（有检索内容）：
   a. 基于NLI（自然语言推理）：
      - 将输出分解为事实陈述
      - 使用NLI模型检查每个陈述是否被检索内容支持
      - entailment: 支持, contradiction: 幻觉, neutral: 不确定
   
   b. 基于Embedding相似度：
      - 计算输出与检索内容的Embedding相似度
      - 低相似度可能表示幻觉
   
   c. 基于LLM评估：
      - 让更强的模型判断输出是否基于给定上下文
      - "请判断以下回答是否基于提供的资料"

2. 开放生成场景（无检索内容）：
   a. 事实一致性检查：
      - 使用知识图谱验证实体关系
      - 使用搜索引擎验证事实
   
   b. 自我一致性：
      - 多次采样，检查输出是否一致
      - 不一致可能表示幻觉
   
   c. 不确定性量化：
      - 检查模型输出概率分布
      - 低概率token可能是幻觉

3. 自动化评估指标：
   - Faithfulness：输出与源材料的一致性
   - FactScore：事实准确性评分
   - BERTScore：语义相似度

实现示例：
```python
def detect_hallucination(output, contexts=None):
    """
    幻觉检测流程
    """
    # Step 1: 提取事实陈述
    statements = extract_statements(output)
    
    hallucinated = []
    for statement in statements:
        if contexts:
            # RAG场景：检查是否被上下文支持
            supported = check_entailment(statement, contexts)
            if supported == "contradiction":
                hallucinated.append(statement)
        else:
            # 开放场景：使用外部知识验证
            verified = verify_with_search(statement)
            if not verified:
                hallucinated.append(statement)
    
    return {
        "hallucination_rate": len(hallucinated) / len(statements),
        "hallucinated_statements": hallucinated,
    }
```

挑战：
- 没有完美的方法，通常是多方法组合
- 需要平衡准确性和计算成本
- 某些领域知识难以验证
```

**Q5：A/B测试在AI应用中的特殊考虑是什么？**

```markdown
参考答案：

AI应用A/B测试的特殊性：

1. 随机化挑战：
   - 传统：用户随机分配到A/B组
   - AI：用户历史对话可能影响当前体验
   - 解决：基于用户ID哈希，保证同一用户始终在同一组

2. 样本量计算：
   - 传统：基于转化率等明确指标
   - AI：输出质量难以量化，指标方差大
   - 解决：
     * 使用人工评估作为ground truth
     * 增加样本量（方差大需要更多样本）
     * 使用更敏感的指标（如用户留存而非满意度）

3. 指标选择：
   - 技术指标：延迟、Token消耗
   - 质量指标：幻觉率、相关性（需自动化评估）
   - 业务指标：转化率、留存率、NPS
   - 注意：短期指标（延迟）vs 长期指标（留存）

4. 实验设计：
   - 多层实验：同时测试模型和Prompt
   - 正交实验：不同实验之间不干扰
   - 时间因素：模型效果可能随时间变化（概念漂移）

5. 伦理考虑：
   - 安全性：新模型是否更安全？
   - 偏见：是否对特定群体不公平？
   - 知情同意：用户是否知道参与实验？

最佳实践：
- 小流量开始（1-5%）
- 设置自动止损（效果太差自动回滚）
- 多维度评估（技术+质量+业务）
- 长期追踪（至少2周）
- 灰度发布（逐步放量）
```

**Q6：如何设计一个低成本的AI监控方案？**

```markdown
参考答案：

低成本监控方案设计：

架构：
┌─────────────────────────────────────────────────────────┐
│  开源工具栈（零成本）                                     │
│                                                         │
│  指标采集：Prometheus（时序数据库）                       │
│  日志：Loki（轻量级日志聚合）                             │
│  追踪：Jaeger（分布式追踪）                               │
│  可视化：Grafana（仪表盘）                                │
│  告警：Alertmanager（告警管理）                           │
│  AI监控：Langfuse（开源LLM可观测）                        │
└─────────────────────────────────────────────────────────┘

成本优化策略：

1. 自托管：
   - 使用云服务器（而非SaaS）
   - 按需使用Spot实例（节省70%）
   - 使用对象存储（S3/MinIO）代替昂贵数据库

2. 采样策略：
   - 生产环境：1-10%采样
   - 错误请求：100%采样
   - 自适应采样（错误率高时增加采样）

3. 数据保留：
   - 热数据（7天）：高性能存储
   - 温数据（30天）：标准存储
   - 冷数据（1年）：归档存储
   - 自动清理过期数据

4. 计算优化：
   - 批量处理：每10秒聚合一次
   - 压缩：使用gzip减少传输和存储
   - 预聚合：客户端先聚合再发送

5. 存储优化：
   - 只存储核心指标（10-20个）
   - 使用降采样（旧数据保留小时级而非分钟级）
   - 标签基数控制（避免高基数字段）

6. 告警优化：
   - 减少无效告警（动态阈值）
   - 告警收敛（相同问题只发一次）
   - 使用免费渠道（Webhook → 企业微信/钉钉）

预估成本（月度）：
- 云服务器：$100-200（2-4核8G）
- 存储：$50-100（1-2TB）
- 网络：$20-50
- 总计：$170-350/月

vs 商业方案：
- Datadog：$1000-5000/月
- New Relic：$500-2000/月
- LangSmith：$500-2000/月

节省：70-90%

权衡：
- 节省成本但增加运维负担
- 需要专人维护开源工具
- 功能不如商业方案完善
- 适合技术能力强的团队
```

### 高级题

**Q7：如何设计一个支持多模型、多版本的AI监控体系？**

```markdown
参考答案：

多模型多版本监控体系设计：

架构：
┌─────────────────────────────────────────────────────────┐
│  模型注册中心（Model Registry）                          │
│  - 模型元数据：版本、训练数据、评估指标                  │
│  - 模型血缘：从数据到模型的完整链路                      │
│  - 模型状态：开发中/测试中/生产中/已下线                 │
├─────────────────────────────────────────────────────────┤
│  实验管理（Experiment Tracking）                         │
│  - 实验配置：超参数、Prompt、数据集                      │
│  - 实验结果：指标、输出样本                              │
│  - 实验对比：可视化对比不同实验                          │
├─────────────────────────────────────────────────────────┤
│  生产监控（Production Monitoring）                       │
│  - 模型版本路由：A/B测试、金丝雀发布                     │
│  - 版本级指标：每个版本的延迟、错误率、质量              │
│  - 版本对比：实时对比不同版本的效果                      │
├─────────────────────────────────────────────────────────┤
│  反馈闭环（Feedback Loop）                               │
│  - 用户反馈：点赞/点踩/评分                              │
│  - 人工评估：定期抽样人工评估                            │
│  - 自动评估：自动化质量检测                              │
│  - 模型迭代：基于反馈自动触发重训练                      │
└─────────────────────────────────────────────────────────┘

关键设计决策：

1. 模型标识：
   - 模型ID：model_name@version
   - 示例：gpt-4@v20240101
   - 包含：基模型 + 微调版本 + Prompt版本

2. 指标维度：
   - 全局指标：整体服务质量
   - 模型级指标：每个模型的性能
   - 版本级指标：每个版本的性能
   - 用户级指标：每个用户的体验

3. 数据模型：
```python
# 模型元数据
model_metadata = {
    "model_id": "llama-2-7b-finetuned-v3",
    "base_model": "meta-llama/Llama-2-7b-hf",
    "version": "v3",
    "training_data": "custom_dataset_v2",
    "hyperparameters": {"lr": 2e-5, "epochs": 3},
    "evaluation_metrics": {"accuracy": 0.92, "f1": 0.89},
    "created_at": "2024-01-15",
    "status": "production",
    "tags": ["customer_service", "zh"],
}

# 监控指标（带维度）
metric = {
    "name": "llm_latency_p99",
    "value": 1200,
    "unit": "ms",
    "timestamp": "2024-01-20T10:00:00Z",
    "dimensions": {
        "model_id": "llama-2-7b-finetuned-v3",
        "model_version": "v3",
        "deployment": "production",
        "region": "us-east-1",
        "gpu_type": "a100",
    },
}
```

4. 版本路由：
```python
class ModelRouter:
    def __init__(self):
        self.versions = {}
        self.traffic_split = {}
    
    def register_version(self, model_id, version, endpoint, weight=0):
        """注册新版本"""
        self.versions[f"{model_id}@{version}"] = {
            "endpoint": endpoint,
            "weight": weight,
            "metrics": {},
        }
    
    def set_traffic_split(self, model_id, splits):
        """
        设置流量分配
        splits: {"v1": 0.7, "v2": 0.3}
        """
        self.traffic_split[model_id] = splits
    
    def route(self, model_id, user_id):
        """根据流量分配路由到具体版本"""
        splits = self.traffic_split.get(model_id, {})
        
        # 确定性路由（同一用户始终路由到同一版本）
        hash_value = hash(f"{model_id}:{user_id}")
        cumulative = 0
        
        for version, weight in splits.items():
            cumulative += weight
            if hash_value < cumulative:
                return f"{model_id}@{version}"
        
        return f"{model_id}@latest"
```

5. 版本对比：
   - 实时Dashboard：同时显示多个版本的指标
   - 自动分析：检测版本间差异是否显著
   - 自动决策：效果好的版本自动扩量

6. 回滚机制：
   - 自动回滚：错误率突增时自动切换回旧版本
   - 一键回滚：手动触发，秒级切换
   - 影子测试：新版本接收流量但不返回结果（仅记录指标）
```

**Q8：如何设计一个能够自动发现AI系统异常的监控系统？**

```markdown
参考答案：

自动异常检测系统设计：

架构：
┌─────────────────────────────────────────────────────────┐
│  数据采集层                                              │
│  - 指标：Prometheus / InfluxDB                          │
│  - 日志：ELK / Loki                                     │
│  - 追踪：Jaeger / Zipkin                                │
├─────────────────────────────────────────────────────────┤
│  特征工程层                                              │
│  - 时序特征：滑动窗口统计、趋势、季节性                   │
│  - 文本特征：输出长度、词汇多样性、主题分布               │
│  - 结构化特征：Token分布、延迟分布、错误模式              │
├─────────────────────────────────────────────────────────┤
│  异常检测层                                              │
│  ┌─────────────────────────────────────────────────┐    │
│  │  统计方法（无监督）                               │    │
│  │  - 3-sigma / Z-score                             │    │
│  │  - IQR（四分位距）                                │    │
│  │  - 指数平滑（EWMA）                               │    │
│  │  - 变点检测（CPD）                                │    │
│  └─────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────┐    │
│  │  机器学习方法（无监督）                           │    │
│  │  - 孤立森林（Isolation Forest）                   │    │
│  │  - 自编码器（Autoencoder）                        │    │
│  │  - LSTM预测 + 残差分析                            │    │
│  │  - 聚类（DBSCAN）                                 │    │
│  └─────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────┐    │
│  │  深度学习方法                                     │    │
│  │  - Transformer-based时序预测                      │    │
│  │  - VAE（变分自编码器）                            │    │
│  │  - GAN（生成对抗网络）                            │    │
│  └─────────────────────────────────────────────────┘    │
├─────────────────────────────────────────────────────────┤
│  根因分析层                                              │
│  - 关联分析：异常指标与变更事件的关联                     │
│  - 拓扑分析：依赖服务是否异常                           │
│  - 日志分析：异常时间段的关键日志                       │
├─────────────────────────────────────────────────────────┤
│  告警与响应层                                            │
│  - 告警分级：P0/P1/P2/P3                               │
│  - 告警收敛：合并相关告警                               │
│  - 自动响应：自动扩容/降级/回滚                         │
└─────────────────────────────────────────────────────────┘

关键技术：

1. 时序异常检测：
```python
class TimeSeriesAnomalyDetector:
    def __init__(self, algorithm="zscore"):
        self.algorithm = algorithm
        self.baseline = None
    
    def fit(self, historical_data):
        """学习正常模式"""
        if self.algorithm == "zscore":
            self.baseline = {
                "mean": np.mean(historical_data),
                "std": np.std(historical_data),
            }
        elif self.algorithm == "ewma":
            self.baseline = {
                "ewma": pd.Series(historical_data).ewm(span=20).mean().iloc[-1],
                "std": pd.Series(historical_data).ewm(span=20).std().iloc[-1],
            }
    
    def predict(self, value):
        """检测异常"""
        if self.algorithm == "zscore":
            z_score = abs(value - self.baseline["mean"]) / self.baseline["std"]
            return z_score > 3, z_score
        
        elif self.algorithm == "ewma":
            residual = abs(value - self.baseline["ewma"])
            threshold = 3 * self.baseline["std"]
            return residual > threshold, residual / self.baseline["std"]
```

2. 文本输出异常检测：
```python
class TextAnomalyDetector:
    def __init__(self, embedding_model):
        self.embedding_model = embedding_model
        self.normal_embeddings = []
    
    def fit(self, normal_outputs):
        """学习正常输出的Embedding分布"""
        self.normal_embeddings = [
            self.embedding_model.encode(text)
            for text in normal_outputs
        ]
        self.centroid = np.mean(self.normal_embeddings, axis=0)
        self.threshold = np.percentile(
            [np.linalg.norm(emb - self.centroid) for emb in self.normal_embeddings],
            95
        )
    
    def predict(self, output):
        """检测异常输出"""
        emb = self.embedding_model.encode(output)
        distance = np.linalg.norm(emb - self.centroid)
        
        # 距离超过阈值认为是异常
        is_anomaly = distance > self.threshold * 1.5
        
        return is_anomaly, distance
```

3. 多维度关联检测：
```python
class MultivariateAnomalyDetector:
    def __init__(self):
        self.correlation_matrix = None
    
    def fit(self, data):
        """学习多变量间的相关性"""
        self.correlation_matrix = np.corrcoef(data.T)
        self.mean_vector = np.mean(data, axis=0)
        self.cov_matrix = np.cov(data.T)
    
    def predict(self, vector):
        """使用马氏距离检测异常"""
        diff = vector - self.mean_vector
        
        try:
            inv_cov = np.linalg.inv(self.cov_matrix)
            mahalanobis_distance = np.sqrt(diff @ inv_cov @ diff)
            
            # 卡方分布阈值
            threshold = np.sqrt(stats.chi2.ppf(0.99, df=len(vector)))
            
            return mahalanobis_distance > threshold, mahalanobis_distance
        except:
            return False, 0
```

自动响应策略：
1. 延迟异常：
   - 自动扩容（增加实例数）
   - 降级到更快模型
   - 启用缓存

2. 错误率异常：
   - 自动回滚到上一版本
   - 切换到备用模型
   - 限流保护

3. 质量异常（幻觉率上升）：
   - 增加检索内容（RAG）
   - 切换到更保守的Prompt
   - 人工审核介入

4. 成本异常：
   - 自动切换到 cheaper 模型
   - 降低采样率
   - 启用限流

挑战：
- 误报率：需要平衡灵敏度和特异性
- 延迟：实时检测需要低延迟算法
- 解释性：需要说明为什么认为是异常
- 适应性：系统行为变化时需要重新学习
```

---

*此文原创，转载请注明出处。*
