# API调用实战深度解析：多平台大模型API的协议、封装与生产级部署

**文章标签：** #ai #api #openai #deepseek #智谱 #kimi #claude #api-gateway #负载均衡 #降级策略

## 目录

- [引言：API调用的工程本质与协议演进](#引言api调用的工程本质与协议演进)
- [理论基础：RESTful API与大模型交互协议](#理论基础restful-api与大模型交互协议)
- [演进史：从OpenAI独占到多平台兼容的API生态](#演进史从openai独占到多平台兼容的api生态)
- [深度解析：API调用的核心技术与架构设计](#深度解析api调用的核心技术与架构设计)
- [实战案例：五个生产级API调用系统实现](#实战案例五个生产级api调用系统实现)
- [对比分析：OpenAI vs Claude vs DeepSeek vs 智谱 vs Kimi](#对比分析openai-vs-claude-vs-deepseek-vs-智谱-vs-kimi)
- [性能分析：延迟、成本与可用性优化](#性能分析延迟成本与可用性优化)
- [常见陷阱与最佳实践](#常见陷阱与最佳实践)
- [面试题与参考答案](#面试题与参考答案)

---

## 引言：API调用的工程本质与协议演进

API调用不是简单的"发送HTTP请求获取响应"，而是一门**基于异步消息传递的分布式系统通信工程**。

核心认知：

```
API调用的本质：客户端通过标准化协议向远程服务发送请求，
               服务在计算资源上执行推理，返回概率分布采样结果

关键挑战：
1. 网络不可靠：延迟、丢包、超时、断开
2. 服务不可靠：限流、故障、升级、下线
3. 结果不确定性：相同输入可能产生不同输出（temperature > 0）
4. 成本控制：按token计费，需要精确预算管理

质量差异的根源：
- 差实现：同步阻塞、无重试、无降级、单点故障
- 好实现：异步并发、智能重试、自动降级、多活备份

系统视角：
API调用 = 请求构造 → 网络传输 → 服务端推理 → 响应解析 → 错误处理
                    ↑_________________________________↓
                              （反馈循环）
```

**关键洞察**：生产级API调用的稳定性不取决于"SDK使用熟练度"，而取决于**分布式系统设计能力**和**容错架构设计**。

---

## 理论基础：RESTful API与大模型交互协议

### 1. HTTP协议与大模型API

#### HTTP/1.1 vs HTTP/2 vs HTTP/3

```python
"""
HTTP协议演进对大模型API的影响：

HTTP/1.1（1997）：
- 串行请求，队头阻塞
- 每次请求需要建立TCP连接（或复用keep-alive）
- 大模型API影响：高延迟，低吞吐

HTTP/2（2015）：
- 多路复用（单个TCP连接并发多个请求）
- 头部压缩（HPACK）
- 服务器推送
- 大模型API影响：提升并发能力，降低延迟

HTTP/3（2022）：
- 基于QUIC（UDP之上）
- 0-RTT连接建立
- 连接迁移（IP变化不影响）
- 大模型API影响：更快建立连接，更好的移动网络支持

实际影响：
- 流式输出（Streaming）必须使用HTTP/2或HTTP/3
- HTTP/1.1无法同时处理多个流式请求
- 大多数API提供商已支持HTTP/2

Connection管理策略：
1. 短连接：每次请求新建TCP连接（慢，开销大）
2. 长连接（Keep-Alive）：复用TCP连接（推荐）
3. 连接池：维护多个长连接（生产级方案）

最佳实践：
- 使用HTTP/2
- 启用keep-alive（timeout=60s）
- 连接池大小 = 并发请求数 × 1.5
"""
```

#### SSE（Server-Sent Events）协议

```python
"""
SSE协议详解：

用途：实现服务器向客户端的流式推送
适用场景：大模型的流式生成（逐字/逐句返回）

协议格式：
```
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive

data: {"choices": [{"delta": {"content": "Hello"}}]}

data: {"choices": [{"delta": {"content": " world"}}]}

data: [DONE]
```

特点：
1. 单向流：服务器 → 客户端
2. 文本格式：每条消息以\n\n分隔
3. 自动重连：客户端断开会自动重连（Last-Event-ID）
4. 基于HTTP：兼容现有基础设施

与WebSocket对比：

特性          SSE              WebSocket
─────────────────────────────────────────
方向          服务器→客户端     双向
协议          HTTP             ws://
重连          自动             手动实现
二进制        不支持           支持
实时性        高               更高
复杂度        低               高
适用场景      流式输出         实时对话

Python实现示例：
"""

import requests
import json

def stream_chat_completion(api_key: str, messages: list, model: str = "gpt-4o"):
    """SSE流式调用示例"""
    
    url = "https://api.openai.com/v1/chat/completions"
    headers = {
        "Authorization": f"Bearer {api_key}",
        "Content-Type": "application/json"
    }
    
    data = {
        "model": model,
        "messages": messages,
        "stream": True  # 启用流式输出
    }
    
    # 发送请求，设置stream=True
    response = requests.post(url, headers=headers, json=data, stream=True)
    
    # 逐行读取SSE数据
    for line in response.iter_lines():
        if line:
            line = line.decode('utf-8')
            
            # SSE格式：data: {...}
            if line.startswith('data: '):
                json_str = line[6:]  # 去掉 "data: " 前缀
                
                if json_str == '[DONE]':
                    break
                
                try:
                    chunk = json.loads(json_str)
                    # 提取生成的内容
                    if chunk['choices'][0]['delta'].get('content'):
                        content = chunk['choices'][0]['delta']['content']
                        print(content, end='', flush=True)
                except json.JSONDecodeError:
                    continue

"""
SSE事件类型：
- data: 事件数据（JSON格式）
- event: 事件名称（如"message", "error"）
- id: 事件ID（用于断线重连）
- retry: 重连间隔（毫秒）

错误处理：
- 网络中断：自动重连（SSE内置）
- 服务器错误：检查event类型为"error"
- 超时处理：设置合理的超时时间
"""
```

### 2. OpenAI API协议详解

#### Chat Completions API

```python
"""
Chat Completions API核心概念：

请求结构：
{
    "model": "gpt-4o",
    "messages": [
        {"role": "system", "content": "你是一个助手"},
        {"role": "user", "content": "你好"},
        {"role": "assistant", "content": "你好！有什么可以帮助你？"},
        {"role": "user", "content": "讲个笑话"}
    ],
    "temperature": 0.7,
    "max_tokens": 1000,
    "top_p": 1.0,
    "frequency_penalty": 0.0,
    "presence_penalty": 0.0,
    "stream": false,
    "tools": [...],
    "tool_choice": "auto"
}

角色（Role）：
- system: 系统提示，设定全局行为
- user: 用户输入
- assistant: 模型输出
- tool: 工具返回结果

核心参数：

1. temperature（温度）
   - 范围：0-2
   - 作用：控制输出的随机性
   - 0：确定性输出（贪婪解码）
   - 0.7：平衡（推荐）
   - 1.5+：高度随机/创意
   - 公式：softmax(logits / temperature)

2. max_tokens（最大token数）
   - 限制输出长度
   - 控制成本（输出token计费）
   - 需要根据任务调整

3. top_p（核采样）
   - 范围：0-1
   - 作用：从累积概率达到p的最小token集合中采样
   - 0.1：只考虑最可能的10% token
   - 1.0：考虑所有token
   - 通常与temperature配合使用

4. frequency_penalty（频率惩罚）
   - 范围：-2.0到2.0
   - 作用：降低重复token的概率
   - 正值：减少重复
   - 适用于长文本生成

5. presence_penalty（存在惩罚）
   - 范围：-2.0到2.0
   - 作用：降低已出现token的概率
   - 与frequency_penalty的区别：
     * presence：只要出现过就惩罚
     * frequency：根据出现次数惩罚

6. tools（工具调用）
   - 定义模型可以调用的工具
   - Function Calling的核心
   - 模型决定何时调用哪个工具

响应结构：
{
    "id": "chatcmpl-xxx",
    "object": "chat.completion",
    "created": 1677652288,
    "model": "gpt-4o",
    "choices": [
        {
            "index": 0,
            "message": {
                "role": "assistant",
                "content": "笑话内容...",
                "tool_calls": [...]
            },
            "finish_reason": "stop"
        }
    ],
    "usage": {
        "prompt_tokens": 50,
        "completion_tokens": 100,
        "total_tokens": 150
    }
}

finish_reason说明：
- stop: 正常结束（遇到停止词）
- length: 达到max_tokens限制
- content_filter: 内容被过滤
- tool_calls: 调用了工具
"""

class OpenAIAPIClient:
    """OpenAI API客户端（完整实现）"""
    
    def __init__(self, api_key: str, base_url: str = "https://api.openai.com/v1"):
        self.api_key = api_key
        self.base_url = base_url
        self.session = requests.Session()
        self.session.headers.update({
            "Authorization": f"Bearer {api_key}",
            "Content-Type": "application/json"
        })
    
    def chat_completion(self, messages: list, **kwargs) -> dict:
        """
        聊天完成API
        
        Args:
            messages: 消息列表
            model: 模型名称
            temperature: 温度
            max_tokens: 最大token数
            stream: 是否流式输出
            
        Returns:
            API响应字典
        """
        url = f"{self.base_url}/chat/completions"
        
        data = {
            "model": kwargs.get("model", "gpt-4o"),
            "messages": messages,
            "temperature": kwargs.get("temperature", 0.7),
            "max_tokens": kwargs.get("max_tokens", 1000),
            "stream": kwargs.get("stream", False)
        }
        
        # 添加可选参数
        if "top_p" in kwargs:
            data["top_p"] = kwargs["top_p"]
        if "frequency_penalty" in kwargs:
            data["frequency_penalty"] = kwargs["frequency_penalty"]
        if "presence_penalty" in kwargs:
            data["presence_penalty"] = kwargs["presence_penalty"]
        if "tools" in kwargs:
            data["tools"] = kwargs["tools"]
            data["tool_choice"] = kwargs.get("tool_choice", "auto")
        
        response = self.session.post(url, json=data, timeout=60)
        response.raise_for_status()
        
        return response.json()
    
    def stream_chat_completion(self, messages: list, **kwargs):
        """流式聊天完成"""
        url = f"{self.base_url}/chat/completions"
        
        data = {
            "model": kwargs.get("model", "gpt-4o"),
            "messages": messages,
            "stream": True,
            "temperature": kwargs.get("temperature", 0.7)
        }
        
        response = self.session.post(url, json=data, stream=True, timeout=60)
        response.raise_for_status()
        
        for line in response.iter_lines():
            if line:
                line = line.decode('utf-8')
                if line.startswith('data: '):
                    json_str = line[6:]
                    if json_str == '[DONE]':
                        break
                    try:
                        yield json.loads(json_str)
                    except json.JSONDecodeError:
                        continue
    
    def count_tokens(self, text: str, model: str = "gpt-4o") -> int:
        """
        计算token数
        
        注意：实际应使用tiktoken库
        """
        import tiktoken
        
        encoding = tiktoken.encoding_for_model(model)
        return len(encoding.encode(text))
    
    def calculate_cost(self, prompt_tokens: int, completion_tokens: int, 
                      model: str = "gpt-4o") -> float:
        """
        计算API调用成本（美元）
        
        2026年价格：
        - GPT-5.5: $25/1M input tokens, $100/1M output tokens
        - GPT-4o: $5/1M input tokens, $15/1M output tokens
        - GPT-4o-mini: $0.15/1M input tokens, $0.6/1M output tokens
        """
        pricing = {
            "gpt-5.5": {"input": 25.0, "output": 100.0},
            "gpt-4o": {"input": 5.0, "output": 15.0},
            "gpt-4o-mini": {"input": 0.15, "output": 0.6}
        }
        
        model_pricing = pricing.get(model, pricing["gpt-4o"])
        
        input_cost = (prompt_tokens / 1_000_000) * model_pricing["input"]
        output_cost = (completion_tokens / 1_000_000) * model_pricing["output"]
        
        return input_cost + output_cost
```

### 3. 认证与安全

```python
"""
API认证机制：

1. API Key认证
   - 在HTTP Header中发送：Authorization: Bearer sk-xxx
   - 简单、常用
   - 风险：密钥泄露

2. OAuth 2.0
   - 更安全的认证流程
   - 支持scope限制
   - 适用于第三方应用

3. JWT（JSON Web Token）
   - 自包含的token
   - 可以包含用户信息和权限
   - 有过期时间

安全最佳实践：

1. 密钥管理
   - 使用环境变量，不要硬编码
   - 密钥轮转（定期更换）
   - 使用密钥管理服务（AWS KMS, Azure Key Vault）

2. 请求签名
   - 某些API需要签名（如阿里云）
   - 防止请求被篡改

3. 限流保护
   - 客户端主动限流
   - 避免触发服务端的限流
   - 指数退避重试

4. 数据传输安全
   - 使用HTTPS（TLS 1.2+）
   - 验证服务器证书
   - 禁用不安全的TLS版本

5. 日志脱敏
   - 日志中不记录API Key
   - 敏感内容脱敏
   - 定期清理日志
"""

class SecureAPIClient:
    """安全的API客户端"""
    
    def __init__(self):
        self.api_key = self._load_api_key()
        self.max_retries = 3
        self.backoff_factor = 2
    
    def _load_api_key(self) -> str:
        """从环境变量加载API Key"""
        import os
        api_key = os.getenv("OPENAI_API_KEY")
        if not api_key:
            raise ValueError("OPENAI_API_KEY not set")
        return api_key
    
    def _make_request(self, url: str, data: dict) -> dict:
        """发送请求（带重试和限流）"""
        import time
        
        for attempt in range(self.max_retries):
            try:
                response = requests.post(
                    url,
                    headers={"Authorization": f"Bearer {self.api_key}"},
                    json=data,
                    timeout=60
                )
                
                if response.status_code == 429:  # 限流
                    wait_time = self.backoff_factor ** attempt
                    time.sleep(wait_time)
                    continue
                
                response.raise_for_status()
                return response.json()
                
            except requests.exceptions.RequestException as e:
                if attempt == self.max_retries - 1:
                    raise
                time.sleep(self.backoff_factor ** attempt)
        
        raise Exception("Max retries exceeded")
    
    def log_request(self, request_data: dict, response_data: dict):
        """记录请求日志（脱敏）"""
        # 移除敏感信息
        safe_request = {k: v for k, v in request_data.items() if k != 'api_key'}
        safe_response = {k: v for k, v in response_data.items() if k != 'content'}
        
        print(f"Request: {safe_request}")
        print(f"Response: {safe_response}")
```

---

## 演进史：从OpenAI独占到多平台兼容的API生态

### 第一阶段：OpenAI独占期（2020-2022）

```python
"""
2020-2022年：OpenAI一家独大

技术特点：
- OpenAI API是唯一选择（GPT-3, GPT-3.5, GPT-4）
- 专有协议，无标准化
- 按token计费模式建立行业标准

开发者体验：
- 只有一个SDK：openai-python
- 文档完善但封闭
- 价格较高但别无选择

局限性：
1. 供应商锁定
   - 所有代码绑定OpenAI API
   - 切换成本高

2. 单点故障
   - OpenAI服务不可用 = 系统瘫痪
   - 2023年多次大规模宕机

3. 成本不可控
   - 定价权在OpenAI
   - 无法议价

代码示例（2021年）：
"""

import openai

# 2021年的API调用方式
openai.api_key = "sk-xxx"

response = openai.Completion.create(
    model="text-davinci-003",
    prompt="Hello, how are you?",
    max_tokens=100
)

print(response.choices[0].text)

"""
当时的问题：
- 只有Completion API（非对话式）
- 没有Chat Completions
- 没有流式输出
- 没有Function Calling
"""
```

### 第二阶段：ChatGPT引发API革命（2022-2023）

```python
"""
2022-2023年：ChatGPT发布后API生态爆发

技术突破：
1. Chat Completions API（2023.3）
   - 对话式接口
   - 支持system/user/assistant角色
   - 成为行业标准

2. Function Calling（2023.6）
   - 模型可以调用外部工具
   - 开启Agent时代

3. 流式输出普及
   - SSE协议
   - 实时用户体验

4. 多模态API（2023.9）
   - GPT-4V支持图像
   - Vision API

开发者体验升级：
- openai-python v1.0（2023.11）
- 类型提示、异步支持
- 更优雅的API设计

代码示例（2023年）：
"""

from openai import OpenAI

client = OpenAI(api_key="sk-xxx")

response = client.chat.completions.create(
    model="gpt-4",
    messages=[
        {"role": "system", "content": "You are a helpful assistant"},
        {"role": "user", "content": "Hello"}
    ],
    stream=True
)

for chunk in response:
    if chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end="")

"""
生态变化：
- 第三方包装工具涌现（LangChain, LlamaIndex）
- 但底层仍绑定OpenAI
- 成本问题开始显现
"""
```

### 第三阶段：国产模型崛起与兼容格式（2023-2024）

```python
"""
2023-2024年：国产模型崛起，OpenAI兼容格式成为标准

国产模型API：
- DeepSeek：深度求索，代码能力强
- 智谱AI（GLM）：清华背景，中文优化
- 月之暗面（Kimi）：长文本，文档分析
- 通义千问（Qwen）：阿里出品
- 文心一言：百度出品

关键技术突破：
1. OpenAI兼容格式
   - 所有国产模型都支持OpenAI SDK格式
   - 只需改base_url和api_key
   - 降低切换成本

2. 价格竞争
   - DeepSeek-V3：$0.14/1M tokens（vs GPT-4 $30/1M）
   - 价格降低100-200倍
   - 推动API普及

3. 特色功能
   - Kimi：200万字上下文
   - DeepSeek：推理模型R1
   - 智谱：CodeGeeX代码模型

代码示例（2024年，多平台兼容）：
"""

from openai import OpenAI

# 统一接口，切换平台只需改配置

# OpenAI
openai_client = OpenAI(api_key="sk-openai")

# DeepSeek
deepseek_client = OpenAI(
    api_key="sk-deepseek",
    base_url="https://api.deepseek.com"
)

# 智谱
zhipu_client = OpenAI(
    api_key="sk-zhipu",
    base_url="https://open.bigmodel.cn/api/paas/v4"
)

# 调用方式完全相同
for client in [openai_client, deepseek_client, zhipu_client]:
    response = client.chat.completions.create(
        model="gpt-4",  # 或 "deepseek-chat", "glm-4"
        messages=[{"role": "user", "content": "Hello"}]
    )
    print(response.choices[0].message.content)

"""
生态成熟标志：
- OpenAI兼容格式成为事实标准
- 统一SDK可以调用所有平台
- 多平台备份成为可能
"""
```

### 第四阶段：2026年的多平台智能路由

```python
"""
2026年API调用生态：

1. 模型能力趋同
   - GPT-5.5、Claude 4、DeepSeek-V4能力接近
   - 各有特色但差距缩小
   - 选择更多基于成本、延迟、特色功能

2. 智能路由普及
   - 自动选择最优平台
   - 基于任务类型、成本、延迟
   - 负载均衡和故障转移

3. 标准化协议
   - OpenAI兼容格式成为ISO标准
   - Function Calling协议统一
   - 工具调用标准化

4. 边缘部署
   - 小模型部署在边缘设备
   - 大模型在云端
   - 边缘-云端协同

5. 成本透明化
   - 实时成本监控
   - 碳足迹追踪
   - 自动成本优化

2026年API调用特点：
- 多平台成为标配（平均使用3-5个平台）
- 智能路由降低30-50%成本
- 流式输出成为默认
- 工具调用（Function Calling）普及率>80%
"""
```

---

## 深度解析：API调用的核心技术与架构设计

### 1. 统一API封装层

#### 多平台抽象接口

```python
"""
统一API封装层设计：

目标：
- 屏蔽不同平台的差异
- 支持动态切换平台
- 统一的错误处理
- 统一的日志和监控

架构：
┌─────────────────────────────────────────┐
│         应用层                            │
│  - 业务逻辑                               │
│  - 不需要关心底层平台                      │
└─────────────────┬───────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│         统一API层                         │
│  - LLMClient（抽象接口）                   │
│  - 统一的方法签名                          │
│  - 统一的返回格式                          │
└─────────────────┬───────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│         平台适配层                        │
│  - OpenAIAdapter                          │
│  - DeepSeekAdapter                        │
│  - ClaudeAdapter                          │
│  - ZhipuAdapter                           │
│  - KimiAdapter                            │
└─────────────────┬───────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│         HTTP客户端层                      │
│  - 连接池管理                              │
│  - 重试机制                                │
│  - 超时控制                                │
│  - SSL/TLS                                 │
└─────────────────────────────────────────┘
"""

from abc import ABC, abstractmethod
from typing import List, Dict, Optional, Iterator
import requests
import json

class LLMProvider(ABC):
    """LLM提供商抽象基类"""
    
    @abstractmethod
    def chat(self, messages: List[Dict[str, str]], **kwargs) -> Dict:
        """非流式聊天"""
        pass
    
    @abstractmethod
    def chat_stream(self, messages: List[Dict[str, str]], **kwargs) -> Iterator[Dict]:
        """流式聊天"""
        pass
    
    @abstractmethod
    def embed(self, texts: List[str], **kwargs) -> List[List[float]]:
        """文本Embedding"""
        pass
    
    @property
    @abstractmethod
    def name(self) -> str:
        """提供商名称"""
        pass

class OpenAIAdapter(LLMProvider):
    """OpenAI适配器"""
    
    def __init__(self, api_key: str, base_url: str = "https://api.openai.com/v1"):
        self.api_key = api_key
        self.base_url = base_url
        self.session = requests.Session()
        self.session.headers.update({
            "Authorization": f"Bearer {api_key}",
            "Content-Type": "application/json"
        })
    
    @property
    def name(self) -> str:
        return "openai"
    
    def chat(self, messages: List[Dict[str, str]], **kwargs) -> Dict:
        url = f"{self.base_url}/chat/completions"
        
        data = {
            "model": kwargs.get("model", "gpt-4o"),
            "messages": messages,
            "temperature": kwargs.get("temperature", 0.7),
            "max_tokens": kwargs.get("max_tokens", 1000)
        }
        
        response = self.session.post(url, json=data, timeout=60)
        response.raise_for_status()
        
        return self._normalize_response(response.json())
    
    def chat_stream(self, messages: List[Dict[str, str]], **kwargs) -> Iterator[Dict]:
        url = f"{self.base_url}/chat/completions"
        
        data = {
            "model": kwargs.get("model", "gpt-4o"),
            "messages": messages,
            "stream": True,
            "temperature": kwargs.get("temperature", 0.7)
        }
        
        response = self.session.post(url, json=data, stream=True, timeout=60)
        response.raise_for_status()
        
        for line in response.iter_lines():
            if line:
                line = line.decode('utf-8')
                if line.startswith('data: '):
                    json_str = line[6:]
                    if json_str == '[DONE]':
                        break
                    try:
                        chunk = json.loads(json_str)
                        yield self._normalize_stream_chunk(chunk)
                    except json.JSONDecodeError:
                        continue
    
    def embed(self, texts: List[str], **kwargs) -> List[List[float]]:
        url = f"{self.base_url}/embeddings"
        
        data = {
            "model": kwargs.get("model", "text-embedding-3-small"),
            "input": texts
        }
        
        response = self.session.post(url, json=data, timeout=60)
        response.raise_for_status()
        
        result = response.json()
        return [item["embedding"] for item in result["data"]]
    
    def _normalize_response(self, raw_response: Dict) -> Dict:
        """标准化响应格式"""
        return {
            "content": raw_response["choices"][0]["message"]["content"],
            "role": "assistant",
            "model": raw_response.get("model", ""),
            "usage": raw_response.get("usage", {}),
            "finish_reason": raw_response["choices"][0].get("finish_reason", ""),
            "raw": raw_response
        }
    
    def _normalize_stream_chunk(self, chunk: Dict) -> Dict:
        """标准化流式响应"""
        delta = chunk["choices"][0].get("delta", {})
        return {
            "content": delta.get("content", ""),
            "role": delta.get("role", ""),
            "finish_reason": chunk["choices"][0].get("finish_reason", ""),
            "raw": chunk
        }

class DeepSeekAdapter(OpenAIAdapter):
    """DeepSeek适配器（继承OpenAI，覆盖特定逻辑）"""
    
    def __init__(self, api_key: str):
        super().__init__(api_key, "https://api.deepseek.com")
    
    @property
    def name(self) -> str:
        return "deepseek"
    
    def chat(self, messages: List[Dict[str, str]], **kwargs) -> Dict:
        # DeepSeek特定：支持reasoning_content
        response = super().chat(messages, **kwargs)
        
        # 提取reasoning_content（R1模型）
        raw = response.get("raw", {})
        if "choices" in raw and len(raw["choices"]) > 0:
            message = raw["choices"][0].get("message", {})
            reasoning = message.get("reasoning_content", "")
            if reasoning:
                response["reasoning"] = reasoning
        
        return response

class UnifiedLLMClient:
    """统一的LLM客户端"""
    
    def __init__(self):
        self.providers: Dict[str, LLMProvider] = {}
        self.default_provider = None
    
    def register_provider(self, name: str, provider: LLMProvider):
        """注册提供商"""
        self.providers[name] = provider
        if self.default_provider is None:
            self.default_provider = name
    
    def chat(self, messages: List[Dict[str, str]], provider: str = None, **kwargs) -> Dict:
        """统一聊天接口"""
        provider_name = provider or self.default_provider
        
        if provider_name not in self.providers:
            raise ValueError(f"Unknown provider: {provider_name}")
        
        return self.providers[provider_name].chat(messages, **kwargs)
    
    def chat_stream(self, messages: List[Dict[str, str]], provider: str = None, **kwargs) -> Iterator[Dict]:
        """统一流式聊天接口"""
        provider_name = provider or self.default_provider
        
        if provider_name not in self.providers:
            raise ValueError(f"Unknown provider: {provider_name}")
        
        return self.providers[provider_name].chat_stream(messages, **kwargs)
    
    def embed(self, texts: List[str], provider: str = None, **kwargs) -> List[List[float]]:
        """统一Embedding接口"""
        provider_name = provider or self.default_provider
        
        if provider_name not in self.providers:
            raise ValueError(f"Unknown provider: {provider_name}")
        
        return self.providers[provider_name].embed(texts, **kwargs)

# 使用示例
client = UnifiedLLMClient()
client.register_provider("openai", OpenAIAdapter("sk-openai"))
client.register_provider("deepseek", DeepSeekAdapter("sk-deepseek"))

# 调用OpenAI
response = client.chat(
    messages=[{"role": "user", "content": "Hello"}],
    provider="openai",
    model="gpt-4o"
)

# 调用DeepSeek
response = client.chat(
    messages=[{"role": "user", "content": "Hello"}],
    provider="deepseek",
    model="deepseek-chat"
)
```

### 2. 智能路由与负载均衡

#### 基于规则的路由

```python
"""
路由策略设计：

1. 固定路由
   - 所有请求到指定平台
   - 简单但不灵活

2. 轮询路由
   - 依次发送到不同平台
   - 简单负载均衡

3. 权重路由
   - 按权重分配流量
   - 例如：OpenAI 30%, DeepSeek 50%, Claude 20%

4. 智能路由
   - 基于任务特征选择平台
   - 基于成本选择平台
   - 基于延迟选择平台

5. 故障转移
   - 主平台故障 → 切换到备用平台
   - 自动检测和切换
"""

class LLMRouter:
    """LLM智能路由器"""
    
    def __init__(self):
        self.providers = {}
        self.rules = []
        self.fallback_chain = []
        self.health_status = {}
    
    def register_provider(self, name: str, provider: LLMProvider, 
                         weight: float = 1.0, priority: int = 1):
        """注册提供商"""
        self.providers[name] = {
            'provider': provider,
            'weight': weight,
            'priority': priority
        }
        self.health_status[name] = True
    
    def add_routing_rule(self, condition: callable, provider_name: str):
        """添加路由规则"""
        self.rules.append({
            'condition': condition,
            'provider': provider_name
        })
    
    def route(self, messages: List[Dict[str, str]], **kwargs) -> str:
        """路由到合适的提供商"""
        
        # 1. 检查规则匹配
        for rule in self.rules:
            if rule['condition'](messages, kwargs):
                if self.health_status.get(rule['provider'], False):
                    return rule['provider']
        
        # 2. 基于权重选择
        healthy_providers = [
            name for name, info in self.providers.items()
            if self.health_status.get(name, False)
        ]
        
        if not healthy_providers:
            raise Exception("No healthy providers available")
        
        # 按权重随机选择
        weights = [self.providers[p]['weight'] for p in healthy_providers]
        total_weight = sum(weights)
        probabilities = [w / total_weight for w in weights]
        
        import random
        return random.choices(healthy_providers, weights=probabilities)[0]
    
    def chat_with_fallback(self, messages: List[Dict[str, str]], **kwargs) -> Dict:
        """带故障转移的聊天"""
        
        # 尝试主提供商
        primary = self.route(messages, **kwargs)
        providers_to_try = [primary] + [
            p for p in self.fallback_chain 
            if p != primary and self.health_status.get(p, False)
        ]
        
        last_error = None
        for provider_name in providers_to_try:
            try:
                provider_info = self.providers[provider_name]
                response = provider_info['provider'].chat(messages, **kwargs)
                response['provider'] = provider_name
                return response
            except Exception as e:
                last_error = e
                # 标记提供商不健康
                self.health_status[provider_name] = False
                continue
        
        raise Exception(f"All providers failed. Last error: {last_error}")
    
    def health_check(self):
        """健康检查"""
        for name, info in self.providers.items():
            try:
                # 发送简单请求检查健康
                info['provider'].chat([{"role": "user", "content": "test"}], 
                                     max_tokens=1)
                self.health_status[name] = True
            except:
                self.health_status[name] = False

# 使用示例
router = LLMRouter()
router.register_provider("openai", OpenAIAdapter("sk-openai"), weight=0.3)
router.register_provider("deepseek", DeepSeekAdapter("sk-deepseek"), weight=0.5)
router.register_provider("claude", OpenAIAdapter("sk-claude", "https://api.anthropic.com/v1"), weight=0.2)

# 添加路由规则
router.add_routing_rule(
    lambda msgs, kwargs: kwargs.get('model', '').startswith('gpt'),
    "openai"
)
router.add_routing_rule(
    lambda msgs, kwargs: any('代码' in m.get('content', '') for m in msgs),
    "deepseek"
)

# 使用路由
response = router.chat_with_fallback(
    messages=[{"role": "user", "content": "Hello"}]
)
```

#### 动态负载均衡

```python
class DynamicLoadBalancer:
    """动态负载均衡器"""
    
    def __init__(self):
        self.providers = {}
        self.metrics = {}
        self.update_interval = 60  # 秒
    
    def register_provider(self, name: str, provider: LLMProvider):
        """注册提供商"""
        self.providers[name] = provider
        self.metrics[name] = {
            'latency_history': [],
            'error_rate': 0.0,
            'request_count': 0,
            'success_count': 0
        }
    
    def update_metrics(self, provider_name: str, latency: float, success: bool):
        """更新指标"""
        metrics = self.metrics[provider_name]
        metrics['latency_history'].append(latency)
        
        # 只保留最近100条记录
        if len(metrics['latency_history']) > 100:
            metrics['latency_history'] = metrics['latency_history'][-100:]
        
        metrics['request_count'] += 1
        if success:
            metrics['success_count'] += 1
        
        metrics['error_rate'] = 1 - (metrics['success_count'] / metrics['request_count'])
    
    def get_provider_score(self, provider_name: str) -> float:
        """
        计算提供商得分（越高越好）
        
        考虑因素：
        - 延迟（越低越好）
        - 错误率（越低越好）
        - 成本（越低越好）
        """
        metrics = self.metrics[provider_name]
        
        if not metrics['latency_history']:
            return 1.0  # 默认分数
        
        avg_latency = sum(metrics['latency_history']) / len(metrics['latency_history'])
        error_rate = metrics['error_rate']
        
        # 得分公式（可根据业务调整）
        latency_score = max(0, 1 - avg_latency / 10)  # 假设10秒为最差
        error_score = 1 - error_rate
        
        # 加权平均
        score = 0.4 * latency_score + 0.6 * error_score
        
        return score
    
    def select_provider(self) -> str:
        """选择最佳提供商"""
        scores = {}
        for name in self.providers:
            scores[name] = self.get_provider_score(name)
        
        # 选择得分最高的
        return max(scores, key=scores.get)
    
    async def chat_with_monitoring(self, messages: List[Dict[str, str]], **kwargs) -> Dict:
        """带监控的聊天"""
        import time
        
        provider_name = self.select_provider()
        provider = self.providers[provider_name]
        
        start_time = time.time()
        try:
            response = provider.chat(messages, **kwargs)
            latency = time.time() - start_time
            self.update_metrics(provider_name, latency, success=True)
            response['provider'] = provider_name
            response['latency'] = latency
            return response
        except Exception as e:
            latency = time.time() - start_time
            self.update_metrics(provider_name, latency, success=False)
            raise
```

### 3. 流式输出与异步处理

#### 异步API客户端

```python
"""
异步处理的优势：

同步调用：
- 阻塞等待响应
- 并发能力低
- 资源利用率低

异步调用：
- 非阻塞，可处理其他请求
- 高并发
- 资源利用率高

适用场景：
- 高并发服务
- 流式输出
- 长耗时任务

Python异步实现：
- asyncio
- aiohttp
- 与FastAPI/Starlette配合
"""

import asyncio
import aiohttp
from typing import AsyncIterator

class AsyncLLMClient:
    """异步LLM客户端"""
    
    def __init__(self, api_key: str, base_url: str):
        self.api_key = api_key
        self.base_url = base_url
        self.session = None
    
    async def __aenter__(self):
        self.session = aiohttp.ClientSession(
            headers={"Authorization": f"Bearer {self.api_key}"}
        )
        return self
    
    async def __aexit__(self, exc_type, exc_val, exc_tb):
        if self.session:
            await self.session.close()
    
    async def chat(self, messages: List[Dict[str, str]], **kwargs) -> Dict:
        """异步聊天"""
        url = f"{self.base_url}/chat/completions"
        
        data = {
            "model": kwargs.get("model", "gpt-4o"),
            "messages": messages,
            "temperature": kwargs.get("temperature", 0.7)
        }
        
        async with self.session.post(url, json=data) as response:
            response.raise_for_status()
            return await response.json()
    
    async def chat_stream(self, messages: List[Dict[str, str]], **kwargs) -> AsyncIterator[Dict]:
        """异步流式聊天"""
        url = f"{self.base_url}/chat/completions"
        
        data = {
            "model": kwargs.get("model", "gpt-4o"),
            "messages": messages,
            "stream": True,
            "temperature": kwargs.get("temperature", 0.7)
        }
        
        async with self.session.post(url, json=data) as response:
            async for line in response.content:
                line = line.decode('utf-8').strip()
                if line.startswith('data: '):
                    json_str = line[6:]
                    if json_str == '[DONE]':
                        break
                    try:
                        yield json.loads(json_str)
                    except json.JSONDecodeError:
                        continue
    
    async def batch_chat(self, requests: List[Dict], max_concurrent: int = 5) -> List[Dict]:
        """
        批量异步聊天（限制并发数）
        
        requests: [
            {"messages": [...], "model": "gpt-4o"},
            {"messages": [...], "model": "gpt-4o"},
            ...
        ]
        """
        semaphore = asyncio.Semaphore(max_concurrent)
        
        async def bounded_chat(req):
            async with semaphore:
                return await self.chat(req['messages'], model=req.get('model', 'gpt-4o'))
        
        tasks = [bounded_chat(req) for req in requests]
        return await asyncio.gather(*tasks, return_exceptions=True)

# 使用示例
async def main():
    async with AsyncLLMClient("sk-xxx", "https://api.openai.com/v1") as client:
        # 单个请求
        response = await client.chat([{"role": "user", "content": "Hello"}])
        print(response)
        
        # 流式请求
        async for chunk in client.chat_stream([{"role": "user", "content": "讲个故事"}]):
            print(chunk['choices'][0]['delta'].get('content', ''), end='')
        
        # 批量请求
        requests = [
            {"messages": [{"role": "user", "content": f"问题{i}"}]}
            for i in range(10)
        ]
        responses = await client.batch_chat(requests, max_concurrent=3)
        print(f"完成 {len(responses)} 个请求")

# 运行
# asyncio.run(main())
```

---

## 实战案例：五个生产级API调用系统实现

### 案例1：多平台智能路由网关

```python
"""
多平台智能路由网关：

功能：
1. 统一API接口
2. 智能路由（基于成本、质量、延迟）
3. 自动故障转移
4. 流量控制
5. 成本监控

架构：
┌─────────────────────────────────────────┐
│           客户端请求                      │
│  POST /v1/chat/completions              │
└─────────────────┬───────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│           API网关                         │
│  - 认证鉴权                               │
│  - 限流（Rate Limiting）                  │
│  - 请求校验                               │
└─────────────────┬───────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│         智能路由引擎                       │
│  - 任务分类                               │
│  - 平台选择                               │
│  - 负载均衡                               │
└─────────────────┬───────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│         平台适配层                         │
│  - OpenAI                                 │
│  - DeepSeek                               │
│  - Claude                                 │
│  - 智谱                                   │
│  - Kimi                                   │
└─────────────────┬───────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│         响应处理                           │
│  - 格式统一                               │
│  - 错误处理                               │
│  - 日志记录                               │
└─────────────────────────────────────────┘
"""

from fastapi import FastAPI, HTTPException, Depends
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
import time
import json

app = FastAPI(title="LLM API Gateway")
security = HTTPBearer()

class LLMAPIGateway:
    """LLM API网关"""
    
    def __init__(self):
        self.router = LLMRouter()
        self.rate_limiter = RateLimiter()
        self.cost_tracker = CostTracker()
        
        # 初始化提供商
        self._init_providers()
    
    def _init_providers(self):
        """初始化提供商"""
        # OpenAI
        self.router.register_provider(
            "openai",
            OpenAIAdapter("sk-openai"),
            weight=0.3
        )
        
        # DeepSeek
        self.router.register_provider(
            "deepseek",
            DeepSeekAdapter("sk-deepseek"),
            weight=0.4
        )
        
        # Claude
        self.router.register_provider(
            "claude",
            OpenAIAdapter("sk-claude", "https://api.anthropic.com/v1"),
            weight=0.3
        )
    
    async def chat_completion(self, request: dict, api_key: str) -> dict:
        """聊天完成接口"""
        
        # 1. 限流检查
        if not self.rate_limiter.allow_request(api_key):
            raise HTTPException(status_code=429, detail="Rate limit exceeded")
        
        # 2. 路由选择
        provider_name = self.router.route(request.get('messages', []), **request)
        
        # 3. 调用提供商
        start_time = time.time()
        try:
            provider_info = self.router.providers[provider_name]
            response = provider_info['provider'].chat(
                request['messages'],
                model=request.get('model'),
                temperature=request.get('temperature', 0.7),
                max_tokens=request.get('max_tokens', 1000)
            )
            
            latency = time.time() - start_time
            
            # 4. 成本跟踪
            usage = response.get('usage', {})
            self.cost_tracker.log_usage(
                api_key=api_key,
                provider=provider_name,
                model=request.get('model', 'unknown'),
                prompt_tokens=usage.get('prompt_tokens', 0),
                completion_tokens=usage.get('completion_tokens', 0),
                latency=latency
            )
            
            # 5. 添加网关信息
            response['gateway'] = {
                'provider': provider_name,
                'latency': latency,
                'timestamp': time.time()
            }
            
            return response
            
        except Exception as e:
            # 尝试故障转移
            return await self._fallback_chat(request, provider_name, e)
    
    async def _fallback_chat(self, request: dict, failed_provider: str, error: Exception) -> dict:
        """故障转移"""
        
        for provider_name, info in self.router.providers.items():
            if provider_name == failed_provider:
                continue
            
            try:
                response = info['provider'].chat(
                    request['messages'],
                    model=request.get('model'),
                    temperature=request.get('temperature', 0.7)
                )
                
                response['gateway'] = {
                    'provider': provider_name,
                    'fallback': True,
                    'original_error': str(error)
                }
                
                return response
            except:
                continue
        
        raise HTTPException(status_code=503, detail="All providers unavailable")

class RateLimiter:
    """限流器"""
    
    def __init__(self):
        self.requests = {}  # api_key -> [timestamp列表]
        self.limit = 100    # 每分钟请求数
        self.window = 60    # 时间窗口（秒）
    
    def allow_request(self, api_key: str) -> bool:
        """是否允许请求"""
        now = time.time()
        
        if api_key not in self.requests:
            self.requests[api_key] = []
        
        # 清理过期请求
        self.requests[api_key] = [
            ts for ts in self.requests[api_key]
            if now - ts < self.window
        ]
        
        # 检查是否超过限制
        if len(self.requests[api_key]) >= self.limit:
            return False
        
        # 记录请求
        self.requests[api_key].append(now)
        return True

class CostTracker:
    """成本追踪器"""
    
    def __init__(self):
        self.usage_log = []
        
        # 价格表（美元/1M tokens）
        self.pricing = {
            'openai': {
                'gpt-4o': {'input': 5.0, 'output': 15.0},
                'gpt-4o-mini': {'input': 0.15, 'output': 0.6}
            },
            'deepseek': {
                'deepseek-chat': {'input': 0.14, 'output': 0.28}
            },
            'claude': {
                'claude-3-sonnet': {'input': 3.0, 'output': 15.0}
            }
        }
    
    def log_usage(self, api_key: str, provider: str, model: str,
                  prompt_tokens: int, completion_tokens: int, latency: float):
        """记录使用情况"""
        
        # 计算成本
        model_pricing = self.pricing.get(provider, {}).get(model, {})
        input_cost = (prompt_tokens / 1_000_000) * model_pricing.get('input', 0)
        output_cost = (completion_tokens / 1_000_000) * model_pricing.get('output', 0)
        total_cost = input_cost + output_cost
        
        self.usage_log.append({
            'timestamp': time.time(),
            'api_key': api_key,
            'provider': provider,
            'model': model,
            'prompt_tokens': prompt_tokens,
            'completion_tokens': completion_tokens,
            'total_tokens': prompt_tokens + completion_tokens,
            'cost': total_cost,
            'latency': latency
        })
    
    def get_cost_report(self, api_key: str = None) -> dict:
        """获取成本报告"""
        
        if api_key:
            logs = [u for u in self.usage_log if u['api_key'] == api_key]
        else:
            logs = self.usage_log
        
        total_cost = sum(u['cost'] for u in logs)
        total_tokens = sum(u['total_tokens'] for u in logs)
        avg_latency = sum(u['latency'] for u in logs) / len(logs) if logs else 0
        
        # 按提供商统计
        by_provider = {}
        for log in logs:
            p = log['provider']
            if p not in by_provider:
                by_provider[p] = {'cost': 0, 'tokens': 0, 'requests': 0}
            by_provider[p]['cost'] += log['cost']
            by_provider[p]['tokens'] += log['total_tokens']
            by_provider[p]['requests'] += 1
        
        return {
            'total_cost': total_cost,
            'total_tokens': total_tokens,
            'avg_latency': avg_latency,
            'total_requests': len(logs),
            'by_provider': by_provider
        }

# FastAPI路由
gateway = LLMAPIGateway()

@app.post("/v1/chat/completions")
async def chat_completions(request: dict, credentials: HTTPAuthorizationCredentials = Depends(security)):
    """聊天完成接口"""
    api_key = credentials.credentials
    return await gateway.chat_completion(request, api_key)

@app.get("/v1/usage")
async def get_usage(credentials: HTTPAuthorizationCredentials = Depends(security)):
    """获取使用统计"""
    api_key = credentials.credentials
    return gateway.cost_tracker.get_cost_report(api_key)
```

### 案例2：流式输出实时服务

```python
"""
流式输出实时服务：

场景：AI聊天应用，需要实时显示生成内容

技术要点：
1. SSE协议
2. 前端实时渲染
3. 后端流式转发
4. 连接管理

架构：
前端 ←→ WebSocket/SSE ←→ 后端服务 ←→ LLM API
"""

from fastapi import FastAPI
from fastapi.responses import StreamingResponse
import json

app = FastAPI()

class StreamingChatService:
    """流式聊天服务"""
    
    def __init__(self):
        self.client = AsyncLLMClient("sk-xxx", "https://api.openai.com/v1")
    
    async def stream_chat(self, messages: list, **kwargs):
        """流式聊天生成器"""
        
        async with self.client:
            async for chunk in self.client.chat_stream(messages, **kwargs):
                # 标准化chunk格式
                content = chunk['choices'][0]['delta'].get('content', '')
                
                if content:
                    # SSE格式：data: {...}\n\n
                    data = json.dumps({
                        'content': content,
                        'done': False
                    })
                    yield f"data: {data}\n\n"
            
            # 发送结束标记
            yield f"data: {json.dumps({'done': True})}\n\n"
    
    async def stream_with_buffer(self, messages: list, buffer_size: int = 5, **kwargs):
        """
        带缓冲的流式输出
        
        减少前端渲染频率，提升性能
        """
        buffer = []
        
        async with self.client:
            async for chunk in self.client.chat_stream(messages, **kwargs):
                content = chunk['choices'][0]['delta'].get('content', '')
                
                if content:
                    buffer.append(content)
                    
                    if len(buffer) >= buffer_size:
                        # 发送缓冲区内容
                        data = json.dumps({
                            'content': ''.join(buffer),
                            'done': False
                        })
                        yield f"data: {data}\n\n"
                        buffer = []
            
            # 发送剩余内容
            if buffer:
                data = json.dumps({
                    'content': ''.join(buffer),
                    'done': False
                })
                yield f"data: {data}\n\n"
            
            yield f"data: {json.dumps({'done': True})}\n\n"

service = StreamingChatService()

@app.post("/v1/stream/chat")
async def stream_chat(request: dict):
    """流式聊天接口"""
    return StreamingResponse(
        service.stream_chat(request['messages'], model=request.get('model')),
        media_type="text/event-stream"
    )

"""
前端JavaScript接收示例：

```javascript
const eventSource = new EventSource('/v1/stream/chat');

eventSource.onmessage = (event) => {
    const data = JSON.parse(event.data);
    if (data.done) {
        eventSource.close();
    } else {
        appendToChat(data.content);
    }
};

eventSource.onerror = (error) => {
    console.error('SSE error:', error);
    eventSource.close();
};
```
"""
```

### 案例3：批量处理任务队列

```python
"""
批量处理任务队列：

场景：需要处理大量文档（如翻译、摘要、分析）

技术要点：
1. 任务队列（Redis/RabbitMQ）
2. 批量API调用
3. 进度跟踪
4. 错误重试
5. 结果存储

架构：
用户提交任务 → 任务队列 → 工作节点 → LLM API → 结果存储
                ↑_________↓
                  进度通知
"""

import redis
import json
from celery import Celery

# 使用Celery作为任务队列
celery_app = Celery('llm_tasks', broker='redis://localhost:6379/0')

class BatchTaskProcessor:
    """批量任务处理器"""
    
    def __init__(self):
        self.redis_client = redis.Redis(host='localhost', port=6379, db=1)
        self.llm_client = UnifiedLLMClient()
        self.llm_client.register_provider("openai", OpenAIAdapter("sk-openai"))
    
    def submit_task(self, task_type: str, items: list, config: dict) -> str:
        """
        提交批量任务
        
        task_type: 'translate', 'summarize', 'analyze'
        items: 待处理的文档列表
        config: 处理配置（模型、温度等）
        """
        task_id = f"batch_{int(time.time())}_{random.randint(1000, 9999)}"
        
        # 保存任务信息
        task_info = {
            'id': task_id,
            'type': task_type,
            'total': len(items),
            'completed': 0,
            'failed': 0,
            'status': 'pending',
            'config': config
        }
        
        self.redis_client.set(f"task:{task_id}", json.dumps(task_info))
        
        # 将任务拆分为子任务
        for i, item in enumerate(items):
            subtask = {
                'task_id': task_id,
                'index': i,
                'content': item,
                'config': config
            }
            
            # 发送到队列
            process_subtask.delay(subtask)
        
        return task_id
    
    def get_task_status(self, task_id: str) -> dict:
        """获取任务状态"""
        data = self.redis_client.get(f"task:{task_id}")
        if data:
            return json.loads(data)
        return None
    
    def get_task_results(self, task_id: str) -> list:
        """获取任务结果"""
        results = []
        pattern = f"result:{task_id}:*"
        
        for key in self.redis_client.scan_iter(match=pattern):
            data = self.redis_client.get(key)
            if data:
                results.append(json.loads(data))
        
        # 按索引排序
        results.sort(key=lambda x: x['index'])
        return results

@celery_app.task(bind=True, max_retries=3)
def process_subtask(self, subtask: dict):
    """处理子任务（Celery任务）"""
    
    task_id = subtask['task_id']
    index = subtask['index']
    content = subtask['content']
    config = subtask['config']
    
    processor = BatchTaskProcessor()
    
    try:
        # 根据任务类型处理
        if config['type'] == 'translate':
            result = processor._translate(content, config)
        elif config['type'] == 'summarize':
            result = processor._summarize(content, config)
        elif config['type'] == 'analyze':
            result = processor._analyze(content, config)
        else:
            raise ValueError(f"Unknown task type: {config['type']}")
        
        # 保存结果
        result_data = {
            'task_id': task_id,
            'index': index,
            'status': 'success',
            'result': result
        }
        
        processor.redis_client.set(
            f"result:{task_id}:{index}",
            json.dumps(result_data)
        )
        
        # 更新任务进度
        processor._update_progress(task_id, success=True)
        
    except Exception as exc:
        # 重试
        if self.request.retries < self.max_retries:
            raise self.retry(exc=exc, countdown=60)
        
        # 记录失败
        result_data = {
            'task_id': task_id,
            'index': index,
            'status': 'failed',
            'error': str(exc)
        }
        
        processor.redis_client.set(
            f"result:{task_id}:{index}",
            json.dumps(result_data)
        )
        
        processor._update_progress(task_id, success=False)
    
    def _translate(self, content: str, config: dict) -> str:
        """翻译"""
        messages = [
            {"role": "system", "content": f"Translate to {config.get('target_lang', 'English')}"},
            {"role": "user", "content": content}
        ]
        
        response = self.llm_client.chat(messages, provider=config.get('provider'))
        return response['content']
    
    def _summarize(self, content: str, config: dict) -> str:
        """摘要"""
        messages = [
            {"role": "system", "content": "Summarize the following text"},
            {"role": "user", "content": content}
        ]
        
        response = self.llm_client.chat(messages, provider=config.get('provider'))
        return response['content']
    
    def _analyze(self, content: str, config: dict) -> str:
        """分析"""
        messages = [
            {"role": "system", "content": "Analyze the following text"},
            {"role": "user", "content": content}
        ]
        
        response = self.llm_client.chat(messages, provider=config.get('provider'))
        return response['content']
    
    def _update_progress(self, task_id: str, success: bool):
        """更新任务进度"""
        data = self.redis_client.get(f"task:{task_id}")
        if data:
            task_info = json.loads(data)
            task_info['completed'] += 1
            if not success:
                task_info['failed'] += 1
            
            if task_info['completed'] + task_info['failed'] >= task_info['total']:
                task_info['status'] = 'completed'
            else:
                task_info['status'] = 'processing'
            
            self.redis_client.set(f"task:{task_id}", json.dumps(task_info))

import time
import random
```

### 案例4：Function Calling工具调用系统

```python
"""
Function Calling工具调用系统：

场景：AI助手需要调用外部工具（搜索、计算、数据库查询等）

技术要点：
1. 工具定义（JSON Schema）
2. 工具注册和管理
3. 调用链执行
4. 结果回传

架构：
用户请求 → LLM → 需要工具？ → 是 → 调用工具 → 返回结果 → LLM生成最终答案
              ↓
             否 → 直接生成答案
"""

class Tool:
    """工具基类"""
    
    def __init__(self, name: str, description: str, parameters: dict):
        self.name = name
        self.description = description
        self.parameters = parameters
    
    def to_openai_format(self) -> dict:
        """转换为OpenAI工具格式"""
        return {
            "type": "function",
            "function": {
                "name": self.name,
                "description": self.description,
                "parameters": self.parameters
            }
        }
    
    def execute(self, **kwargs) -> str:
        """执行工具"""
        raise NotImplementedError

class CalculatorTool(Tool):
    """计算器工具"""
    
    def __init__(self):
        super().__init__(
            name="calculator",
            description="执行数学计算",
            parameters={
                "type": "object",
                "properties": {
                    "expression": {
                        "type": "string",
                        "description": "数学表达式，如 '2 + 2'"
                    }
                },
                "required": ["expression"]
            }
        )
    
    def execute(self, expression: str) -> str:
        """安全计算"""
        try:
            # 使用eval（实际应使用更安全的计算方式）
            result = eval(expression, {"__builtins__": {}}, {})
            return str(result)
        except Exception as e:
            return f"计算错误: {str(e)}"

class WeatherTool(Tool):
    """天气查询工具"""
    
    def __init__(self):
        super().__init__(
            name="get_weather",
            description="查询指定城市的天气",
            parameters={
                "type": "object",
                "properties": {
                    "city": {
                        "type": "string",
                        "description": "城市名称"
                    }
                },
                "required": ["city"]
            }
        )
    
    def execute(self, city: str) -> str:
        """查询天气（模拟）"""
        # 实际应调用天气API
        return f"{city}今天晴，25°C"

class ToolCallingSystem:
    """工具调用系统"""
    
    def __init__(self, llm_client: UnifiedLLMClient):
        self.llm_client = llm_client
        self.tools: Dict[str, Tool] = {}
    
    def register_tool(self, tool: Tool):
        """注册工具"""
        self.tools[tool.name] = tool
    
    def chat_with_tools(self, messages: list, provider: str = None) -> dict:
        """
        支持工具调用的聊天
        
        流程：
        1. 发送请求（包含工具定义）
        2. 模型决定是否需要调用工具
        3. 如果需要，执行工具
        4. 将工具结果返回给模型
        5. 模型生成最终答案
        """
        # 转换工具为OpenAI格式
        tools_spec = [tool.to_openai_format() for tool in self.tools.values()]
        
        # 第一次调用：让模型决定是否需要工具
        response = self.llm_client.chat(
            messages=messages,
            provider=provider,
            tools=tools_spec,
            tool_choice="auto"
        )
        
        # 检查是否需要调用工具
        if response.get('finish_reason') == 'tool_calls':
            # 执行工具调用
            tool_results = self._execute_tool_calls(response)
            
            # 将工具结果添加到消息历史
            messages.append({
                "role": "assistant",
                "content": None,
                "tool_calls": response['raw']['choices'][0]['message'].get('tool_calls', [])
            })
            
            for result in tool_results:
                messages.append({
                    "role": "tool",
                    "tool_call_id": result['tool_call_id'],
                    "content": result['result']
                })
            
            # 第二次调用：获取最终答案
            final_response = self.llm_client.chat(
                messages=messages,
                provider=provider
            )
            
            return final_response
        
        return response
    
    def _execute_tool_calls(self, response: dict) -> list:
        """执行工具调用"""
        results = []
        
        raw_response = response.get('raw', {})
        tool_calls = raw_response['choices'][0]['message'].get('tool_calls', [])
        
        for tool_call in tool_calls:
            function_name = tool_call['function']['name']
            arguments = json.loads(tool_call['function']['arguments'])
            
            if function_name in self.tools:
                tool = self.tools[function_name]
                result = tool.execute(**arguments)
                
                results.append({
                    'tool_call_id': tool_call['id'],
                    'function': function_name,
                    'result': result
                })
        
        return results

# 使用示例
tool_system = ToolCallingSystem(UnifiedLLMClient())
tool_system.register_tool(CalculatorTool())
tool_system.register_tool(WeatherTool())

messages = [
    {"role": "user", "content": "北京今天天气怎么样？2+2等于几？"}
]

response = tool_system.chat_with_tools(messages)
print(response['content'])
```

### 案例5：多模态API调用系统

```python
"""
多模态API调用系统：

场景：需要处理文本+图像的复合请求

技术要点：
1. 图像编码（Base64）
2. 多模态消息格式
3. 图像理解API调用
4. 结果融合

支持的模型：
- GPT-4o / GPT-5.5（OpenAI）
- Claude 4（Anthropic）
- Gemini 2.0（Google）
- GLM-5（智谱）
"""

import base64
from pathlib import Path

class MultimodalAPIClient:
    """多模态API客户端"""
    
    def __init__(self, api_key: str, base_url: str):
        self.api_key = api_key
        self.base_url = base_url
    
    def encode_image(self, image_path: str) -> str:
        """将图片编码为base64"""
        with open(image_path, "rb") as image_file:
            return base64.b64encode(image_file.read()).decode('utf-8')
    
    def create_vision_message(self, text: str, image_paths: list) -> list:
        """
        创建包含图片的消息
        
        OpenAI格式：
        {
            "role": "user",
            "content": [
                {"type": "text", "text": "描述这张图片"},
                {"type": "image_url", "image_url": {"url": "data:image/jpeg;base64,..."}}
            ]
        }
        """
        content = [{"type": "text", "text": text}]
        
        for image_path in image_paths:
            base64_image = self.encode_image(image_path)
            content.append({
                "type": "image_url",
                "image_url": {
                    "url": f"data:image/jpeg;base64,{base64_image}"
                }
            })
        
        return [{"role": "user", "content": content}]
    
    def analyze_image(self, image_path: str, prompt: str = "描述这张图片") -> dict:
        """分析图片"""
        messages = self.create_vision_message(prompt, [image_path])
        
        # 调用API（实际应发送HTTP请求）
        return {
            "content": "这是一张包含...的图片",
            "model": "gpt-4o"
        }
    
    def compare_images(self, image_paths: list, prompt: str = "比较这两张图片") -> dict:
        """比较多张图片"""
        messages = self.create_vision_message(prompt, image_paths)
        
        return {
            "content": "这两张图片的相似点是...",
            "model": "gpt-4o"
        }
    
    def extract_text_from_image(self, image_path: str) -> dict:
        """OCR：从图片提取文字"""
        messages = self.create_vision_message(
            "提取图片中的所有文字，保持原有格式",
            [image_path]
        )
        
        return {
            "content": "提取到的文字...",
            "model": "gpt-4o"
        }
    
    def analyze_chart(self, image_path: str) -> dict:
        """分析图表"""
        messages = self.create_vision_message(
            "分析这张图表，提取关键数据和趋势",
            [image_path]
        )
        
        return {
            "content": "图表显示...",
            "model": "gpt-4o"
        }

# 使用示例
vision_client = MultimodalAPIClient("sk-xxx", "https://api.openai.com/v1")

# 分析单张图片
result = vision_client.analyze_image("photo.jpg", "这张图片里有什么？")
print(result['content'])

# 比较两张图片
result = vision_client.compare_images(
    ["image1.jpg", "image2.jpg"],
    "找出两张图片的不同之处"
)
print(result['content'])

# OCR
result = vision_client.extract_text_from_image("document.jpg")
print(result['content'])
```

---

## 对比分析：OpenAI vs Claude vs DeepSeek vs 智谱 vs Kimi

### 1. 功能对比

```
功能对比矩阵（2026年）：

┌─────────────────┬────────────┬────────────┬────────────┬────────────┬────────────┐
│     功能        │   OpenAI   │   Claude   │  DeepSeek  │    智谱    │    Kimi    │
├─────────────────┼────────────┼────────────┼────────────┼────────────┼────────────┤
│ 对话能力        │ ⭐⭐⭐⭐⭐   │ ⭐⭐⭐⭐⭐   │ ⭐⭐⭐⭐⭐   │ ⭐⭐⭐⭐    │ ⭐⭐⭐⭐    │
│ 代码能力        │ ⭐⭐⭐⭐⭐   │ ⭐⭐⭐⭐⭐   │ ⭐⭐⭐⭐⭐   │ ⭐⭐⭐⭐    │ ⭐⭐⭐⭐    │
│ 推理能力        │ ⭐⭐⭐⭐⭐   │ ⭐⭐⭐⭐⭐   │ ⭐⭐⭐⭐⭐   │ ⭐⭐⭐⭐    │ ⭐⭐⭐⭐    │
│ 中文能力        │ ⭐⭐⭐⭐    │ ⭐⭐⭐      │ ⭐⭐⭐⭐⭐   │ ⭐⭐⭐⭐⭐   │ ⭐⭐⭐⭐⭐   │
│ 长上下文        │ 128K       │ 200K       │ 64K        │ 128K       │ 1M         │
│ 多模态          │ ⭐⭐⭐⭐⭐   │ ⭐⭐⭐⭐⭐   │ ❌         │ ⭐⭐⭐⭐    │ ❌         │
│ Function Call   │ ✅         │ ✅         │ ✅         │ ✅         │ ✅         │
│ 流式输出        │ ✅         │ ✅         │ ✅         │ ✅         │ ✅         │
│ JSON Mode       │ ✅         │ ✅         │ ✅         │ ✅         │ ❌         │
│ 思维链          │ ✅(o3)     │ ✅         │ ✅(R1)     │ ❌         │ ❌         │
│ 联网搜索        │ ❌         │ ❌         │ ❌         │ ❌         │ ✅         │
│ 文件上传        │ ❌         │ ❌         │ ❌         │ ❌         │ ✅         │
└─────────────────┴────────────┴────────────┴────────────┴────────────┴────────────┘

特色功能：
- OpenAI：功能最全，生态最好，多模态最强
- Claude：推理最强，安全性高，长文本优秀
- DeepSeek：代码最强，性价比最高，推理模型R1
- 智谱：中文最强，CodeGeeX代码模型，国内合规
- Kimi：长文本最强（1M），支持文件上传和联网
```

### 2. 价格对比

```python
"""
价格对比（2026年，美元/1M tokens）：

输入价格：

模型                输入价格    输出价格    备注
─────────────────────────────────────────────────
GPT-5.5             $25.00     $100.00    旗舰多模态
GPT-4o              $5.00      $15.00     行业标准
GPT-4o-mini         $0.15      $0.60      轻量版
Claude Opus 4.7     $15.00     $75.00     推理最强
Claude Sonnet 4.7   $3.00      $15.00     平衡选择
DeepSeek-V4         $0.14      $0.28      性价比之王
DeepSeek-R1         $0.55      $2.19      推理模型
GLM-5               $1.00      $2.00      中文优化
GLM-4               $1.50      $3.00      上一代
GLM-4-Air           $0.10      $0.20      轻量版
Kimi K2.6           $3.00      $12.00     1M上下文
Qwen3-Max           $1.20      $3.60      阿里云

价格分析：
1. 高端市场（GPT-5.5, Claude Opus）
   - 价格：$15-25/1M input
   - 适用：复杂推理、创意任务
   - 成本：较高但效果最佳

2. 中端市场（GPT-4o, Claude Sonnet, Kimi）
   - 价格：$3-5/1M input
   - 适用：大多数生产场景
   - 成本：性价比平衡

3. 性价比市场（DeepSeek, GLM-4-Air, Qwen3）
   - 价格：$0.1-1.5/1M input
   - 适用：高并发、成本敏感
   - 成本：极低，适合大规模部署

4. 成本优化策略：
   - 简单任务 → DeepSeek/GLM-4-Air（节省90%）
   - 中等任务 → GPT-4o/Claude Sonnet
   - 复杂任务 → GPT-5.5/Claude Opus（必要时）
   - 平均节省：60-80%
"""

class PricingComparator:
    """价格比较器"""
    
    PRICING = {
        'gpt-5.5': {'input': 25.0, 'output': 100.0},
        'gpt-4o': {'input': 5.0, 'output': 15.0},
        'gpt-4o-mini': {'input': 0.15, 'output': 0.6},
        'claude-opus-4.7': {'input': 15.0, 'output': 75.0},
        'claude-sonnet-4.7': {'input': 3.0, 'output': 15.0},
        'deepseek-v4': {'input': 0.14, 'output': 0.28},
        'deepseek-r1': {'input': 0.55, 'output': 2.19},
        'glm-5': {'input': 1.0, 'output': 2.0},
        'glm-4-air': {'input': 0.1, 'output': 0.2},
        'kimi-k2.6': {'input': 3.0, 'output': 12.0}
    }
    
    @classmethod
    def calculate_cost(cls, model: str, input_tokens: int, output_tokens: int) -> float:
        """计算调用成本"""
        pricing = cls.PRICING.get(model, {'input': 5.0, 'output': 15.0})
        
        input_cost = (input_tokens / 1_000_000) * pricing['input']
        output_cost = (output_tokens / 1_000_000) * pricing['output']
        
        return input_cost + output_cost
    
    @classmethod
    def compare(cls, input_tokens: int, output_tokens: int) -> list:
        """比较所有模型的成本"""
        comparisons = []
        
        for model, pricing in cls.PRICING.items():
            cost = cls.calculate_cost(model, input_tokens, output_tokens)
            comparisons.append({
                'model': model,
                'cost': cost,
                'input_price': pricing['input'],
                'output_price': pricing['output']
            })
        
        # 按成本排序
        comparisons.sort(key=lambda x: x['cost'])
        
        return comparisons
    
    @classmethod
    def get_cheapest(cls, input_tokens: int, output_tokens: int) -> dict:
        """获取最便宜的模型"""
        comparisons = cls.compare(input_tokens, output_tokens)
        return comparisons[0] if comparisons else None
```

### 3. 延迟与可用性对比

```python
"""
延迟与可用性对比：

延迟（平均，秒）：

模型                首token延迟    完整响应(1K tokens)    可用性SLA
─────────────────────────────────────────────────────────────────────
GPT-5.5             1.5s           5.0s                  99.9%
GPT-4o              0.8s           3.0s                  99.9%
GPT-4o-mini         0.3s           1.0s                  99.9%
Claude Opus 4.7     2.0s           6.0s                  99.5%
Claude Sonnet 4.7   1.0s           3.5s                  99.5%
DeepSeek-V4         0.5s           2.0s                  99.0%
DeepSeek-R1         3.0s           10.0s                 99.0%
GLM-5               0.6s           2.5s                  99.0%
Kimi K2.6           1.2s           4.0s                  99.0%

延迟优化策略：
1. 模型选择
   - 延迟敏感 → GPT-4o-mini, GLM-4-Air, DeepSeek-V4
   - 质量优先 → GPT-5.5, Claude Opus（接受延迟）

2. 地理位置
   - 选择离用户近的API端点
   - 使用CDN加速

3. 流式输出
   - 首token延迟比完整响应更重要
   - 流式输出降低感知延迟

4. 缓存
   - 缓存常见查询结果
   - Embedding结果缓存

可用性保障：
1. 多平台备份
   - 主平台故障 → 自动切换到备用
   - 至少准备2-3个备用平台

2. 健康检查
   - 定期探测各平台可用性
   - 自动剔除故障节点

3. 降级策略
   - 高端模型不可用 → 降级到中端模型
   - 大模型不可用 → 降级到小模型
"""
```

---

## 性能分析：延迟、成本与可用性优化

### 1. 延迟优化

```python
"""
延迟优化策略：

1. 网络层优化
   - 使用HTTP/2或HTTP/3
   - 连接池（复用TCP连接）
   - 选择就近的API端点
   - CDN加速（静态资源）

2. 请求层优化
   - 流式输出（降低感知延迟）
   - 合理的max_tokens（避免过长生成）
   - 批量请求（减少HTTP开销）

3. 缓存层优化
   - 缓存常见查询
   - Embedding结果缓存
   - 响应缓存（TTL）

4. 架构层优化
   - 异步处理（非阻塞）
   - 预加载和预热
   - 边缘计算（就近处理）

延迟组成分析：

总延迟 = T_network + T_queue + T_inference + T_transfer

其中：
- T_network: 网络传输时间（RTT）
  - 优化：就近部署、连接池
  
- T_queue: 服务端排队时间
  - 优化：选择负载低的平台、错峰调用
  
- T_inference: 模型推理时间
  - 优化：选择小模型、降低max_tokens
  
- T_transfer: 响应传输时间
  - 优化：流式输出、压缩

延迟预算分配（以P95 < 3秒为目标）：
- 网络：< 500ms
- 排队：< 500ms
- 推理：< 1500ms
- 传输：< 500ms
"""

class LatencyOptimizer:
    """延迟优化器"""
    
    def __init__(self):
        self.cache = {}
        self.connection_pools = {}
    
    def optimize_request(self, messages: list, **kwargs) -> dict:
        """优化请求参数以降低延迟"""
        
        # 1. 使用流式输出
        kwargs['stream'] = True
        
        # 2. 限制输出长度
        if 'max_tokens' not in kwargs:
            kwargs['max_tokens'] = 500  # 默认限制
        
        # 3. 选择快速模型
        if 'model' not in kwargs:
            kwargs['model'] = 'gpt-4o-mini'  # 使用轻量模型
        
        # 4. 检查缓存
        cache_key = self._get_cache_key(messages, kwargs)
        if cache_key in self.cache:
            return {'cached': True, 'content': self.cache[cache_key]}
        
        return {'cached': False, 'optimized_params': kwargs}
    
    def _get_cache_key(self, messages: list, kwargs: dict) -> str:
        """生成缓存键"""
        import hashlib
        
        content = json.dumps({
            'messages': messages,
            'model': kwargs.get('model'),
            'temperature': kwargs.get('temperature')
        }, sort_keys=True)
        
        return hashlib.md5(content.encode()).hexdigest()
```

### 2. 成本优化

```python
"""
成本优化策略：

1. 模型分层
   - 简单任务 → 小模型（节省80-90%）
   - 中等任务 → 中模型
   - 复杂任务 → 大模型（仅必要时）

2. 缓存策略
   - 高频查询缓存（节省20-40%）
   - Embedding缓存
   - 结果复用

3. 批处理
   - 批量调用降低单位成本
   - 共享上下文

4. 提示词优化
   - 精简prompt（减少input tokens）
   - 使用system message代替重复指令
   - 压缩历史对话

5. 输出控制
   - 限制max_tokens
   - 使用stop序列提前停止
   - 避免不必要的生成

6. 多平台比价
   - 选择最便宜的可用平台
   - 利用价格差异

成本监控：
- 实时成本追踪
- 预算告警
- 成本归因分析
"""

class CostOptimizer:
    """成本优化器"""
    
    def __init__(self):
        self.cost_budget = 1000.0  # 日预算
        self.daily_cost = 0.0
        self.cache = {}
    
    def optimize(self, messages: list, **kwargs) -> tuple:
        """
        返回优化后的参数和预估成本
        """
        original_model = kwargs.get('model', 'gpt-4o')
        
        # 1. 模型降级（如果预算紧张）
        if self.daily_cost > self.cost_budget * 0.8:
            kwargs['model'] = self._downgrade_model(original_model)
        
        # 2. 限制输出长度
        if 'max_tokens' not in kwargs:
            kwargs['max_tokens'] = 500
        
        # 3. 压缩消息历史
        messages = self._compress_messages(messages)
        
        # 4. 计算预估成本
        estimated_input = sum(len(m['content']) for m in messages)
        estimated_output = kwargs.get('max_tokens', 500)
        
        estimated_cost = PricingComparator.calculate_cost(
            kwargs['model'], estimated_input, estimated_output
        )
        
        return kwargs, estimated_cost
    
    def _downgrade_model(self, model: str) -> str:
        """降级到更便宜的模型"""
        downgrade_map = {
            'gpt-5.5': 'gpt-4o',
            'gpt-4o': 'gpt-4o-mini',
            'claude-opus-4.7': 'claude-sonnet-4.7',
            'deepseek-r1': 'deepseek-v4'
        }
        return downgrade_map.get(model, model)
    
    def _compress_messages(self, messages: list) -> list:
        """压缩消息历史"""
        # 如果消息太多，只保留最近的5条
        if len(messages) > 10:
            # 保留system message和最近的消息
            system_msgs = [m for m in messages if m['role'] == 'system']
            recent_msgs = messages[-5:]
            return system_msgs + recent_msgs
        return messages
```

### 3. 可用性保障

```python
"""
可用性保障架构：

1. 健康检查
   - 每30秒探测一次
   - 检查API响应时间和错误率
   - 自动标记不健康节点

2. 故障转移
   - 主平台故障 → 自动切换到备用
   - 切换时间 < 5秒
   - 用户无感知

3. 熔断机制
   - 错误率 > 50% → 熔断
   - 熔断后5分钟尝试恢复
   - 渐进式恢复（10%流量）

4. 限流保护
   - 客户端主动限流
   - 避免触发服务端限流
   - 队列缓冲突发流量

5. 重试策略
   - 指数退避重试
   - 最多3次重试
   - 只在可重试错误时重试（5xx, 429）

SLA目标：
- 可用性：99.9%
- 故障转移：< 5秒
- 错误率：< 0.1%
"""

class AvailabilityManager:
    """可用性管理器"""
    
    def __init__(self):
        self.providers = {}
        self.health_status = {}
        self.circuit_breakers = {}
        self.retry_policies = {}
    
    def register_provider(self, name: str, provider: LLMProvider):
        """注册提供商"""
        self.providers[name] = provider
        self.health_status[name] = True
        self.circuit_breakers[name] = CircuitBreaker()
    
    async def call_with_resilience(self, provider_name: str, 
                                   messages: list, **kwargs) -> dict:
        """带弹性的调用"""
        
        # 1. 检查熔断器
        cb = self.circuit_breakers.get(provider_name)
        if cb and cb.is_open():
            raise Exception(f"Circuit breaker open for {provider_name}")
        
        # 2. 执行调用（带重试）
        last_error = None
        for attempt in range(3):
            try:
                provider = self.providers[provider_name]
                response = provider.chat(messages, **kwargs)
                
                # 成功，记录健康
                cb.record_success()
                return response
                
            except Exception as e:
                last_error = e
                
                # 检查是否可重试
                if not self._is_retryable_error(e):
                    break
                
                # 指数退避
                wait_time = 2 ** attempt
                await asyncio.sleep(wait_time)
        
        # 所有重试失败
        cb.record_failure()
        raise last_error
    
    def _is_retryable_error(self, error: Exception) -> bool:
        """检查错误是否可重试"""
        error_str = str(error).lower()
        
        retryable = [
            '429',  # 限流
            '500',  # 服务端错误
            '502',  # Bad Gateway
            '503',  # Service Unavailable
            '504',  # Gateway Timeout
            'timeout',
            'connection'
        ]
        
        return any(r in error_str for r in retryable)

class CircuitBreaker:
    """熔断器"""
    
    def __init__(self, failure_threshold: int = 5, recovery_timeout: int = 60):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.failure_count = 0
        self.last_failure_time = None
        self.state = 'closed'  # closed, open, half-open
    
    def is_open(self) -> bool:
        """检查熔断器是否打开"""
        if self.state == 'open':
            # 检查是否过了恢复时间
            if time.time() - self.last_failure_time > self.recovery_timeout:
                self.state = 'half-open'
                return False
            return True
        return False
    
    def record_success(self):
        """记录成功"""
        self.failure_count = 0
        self.state = 'closed'
    
    def record_failure(self):
        """记录失败"""
        self.failure_count += 1
        self.last_failure_time = time.time()
        
        if self.failure_count >= self.failure_threshold:
            self.state = 'open'
```

---

## 常见陷阱与最佳实践

### 常见陷阱

```
陷阱1：没有错误处理
- 错误：直接调用API，不处理异常
- 结果：服务崩溃或返回错误给用户
- 解决：try-except + 重试 + 降级

陷阱2：同步阻塞调用
- 错误：在高并发场景使用同步调用
- 结果：性能极差，请求堆积
- 解决：使用异步 + 连接池

陷阱3：没有限流
- 错误：无限制地向API发送请求
- 结果：触发服务端限流（429错误）
- 解决：客户端限流 + 退避重试

陷阱4：密钥泄露
- 错误：将API Key硬编码在代码中
- 结果：密钥被泄露，产生巨额账单
- 解决：环境变量 + 密钥管理服务

陷阱5：忽视成本
- 错误：没有成本监控
- 结果：月底账单超预期
- 解决：实时成本追踪 + 预算告警

陷阱6：单点故障
- 错误：只使用一个API平台
- 结果：平台故障时服务完全不可用
- 解决：多平台备份 + 自动故障转移

陷阱7：流式输出处理不当
- 错误：没有正确处理SSE连接断开
- 结果：输出不完整或连接泄露
- 解决：完善的连接管理 + 重连机制

陷阱8：提示词注入
- 错误：直接将用户输入拼接到prompt
- 结果：提示词注入攻击
- 解决：输入校验 + 参数化prompt

陷阱9：长连接不释放
- 错误：创建大量HTTP连接不关闭
- 结果：文件描述符耗尽
- 解决：使用连接池 + 超时关闭

陷阱10：没有日志和监控
- 错误：不记录API调用日志
- 结果：出问题无法排查
- 解决：完善的日志 + 监控告警
```

### 最佳实践

```python
"""
最佳实践清单：

1. 认证安全
   ✓ 使用环境变量存储API Key
   ✓ 定期轮换密钥
   ✓ 使用最小权限原则
   ✓ 日志中脱敏敏感信息

2. 错误处理
   ✓ 所有API调用包裹try-except
   ✓ 区分可重试错误和不可重试错误
   ✓ 指数退避重试
   ✓ 优雅降级（备用方案）

3. 性能优化
   ✓ 使用异步编程（asyncio）
   ✓ 连接池复用TCP连接
   ✓ 流式输出降低感知延迟
   ✓ 合理的超时设置（连接超时、读取超时）

4. 成本控制
   ✓ 实时成本监控
   ✓ 预算告警
   ✓ 模型分层（简单任务用小模型）
   ✓ 缓存高频查询
   ✓ 限制max_tokens

5. 高可用
   ✓ 多平台备份（至少2-3个）
   ✓ 健康检查
   ✓ 自动故障转移
   ✓ 熔断器保护
   ✓ 限流（避免触发服务端限流）

6. 可观测性
   ✓ 记录所有API调用（延迟、成本、错误）
   ✓ 实时监控仪表盘
   ✓ 告警（错误率、延迟、成本超标）
   ✓ 链路追踪

7. 代码质量
   ✓ 统一封装层（屏蔽平台差异）
   ✓ 类型提示
   ✓ 单元测试
   ✓ 文档完善

8. 合规
   ✓ 遵守API使用条款
   ✓ 数据隐私保护
   ✓ 审计日志
   ✓ 权限控制
"""

class BestPracticeLLMClient:
    """遵循最佳实践的LLM客户端"""
    
    def __init__(self):
        self.providers = {}
        self.router = LLMRouter()
        self.cost_tracker = CostTracker()
        self.rate_limiter = RateLimiter()
        self.availability_manager = AvailabilityManager()
        self.cache = {}
    
    async def chat(self, messages: list, **kwargs) -> dict:
        """
        遵循最佳实践的聊天接口
        
        包含：
        - 限流检查
        - 缓存检查
        - 智能路由
        - 故障转移
        - 成本追踪
        - 日志记录
        """
        # 1. 限流检查
        if not self.rate_limiter.allow_request('default'):
            raise Exception("Rate limit exceeded")
        
        # 2. 缓存检查
        cache_key = self._get_cache_key(messages, kwargs)
        if cache_key in self.cache:
            return {'cached': True, **self.cache[cache_key]}
        
        # 3. 选择提供商
        provider_name = self.router.route(messages, **kwargs)
        
        # 4. 带弹性的调用
        try:
            response = await self.availability_manager.call_with_resilience(
                provider_name, messages, **kwargs
            )
        except Exception as e:
            # 故障转移
            response = await self._fallback_call(messages, **kwargs)
        
        # 5. 成本追踪
        usage = response.get('usage', {})
        self.cost_tracker.log_usage(
            api_key='default',
            provider=provider_name,
            model=kwargs.get('model', 'unknown'),
            prompt_tokens=usage.get('prompt_tokens', 0),
            completion_tokens=usage.get('completion_tokens', 0),
            latency=0
        )
        
        # 6. 缓存结果
        if kwargs.get('cache', True):
            self.cache[cache_key] = response
        
        # 7. 添加元数据
        response['provider'] = provider_name
        response['cached'] = False
        
        return response
    
    async def _fallback_call(self, messages: list, **kwargs) -> dict:
        """备用调用"""
        # 尝试所有健康的提供商
        for name, info in self.router.providers.items():
            try:
                response = info['provider'].chat(messages, **kwargs)
                return response
            except:
                continue
        
        raise Exception("All providers failed")
    
    def _get_cache_key(self, messages: list, kwargs: dict) -> str:
        """生成缓存键"""
        import hashlib
        content = json.dumps({'messages': messages, **kwargs}, sort_keys=True)
        return hashlib.md5(content.encode()).hexdigest()
```

---

## 面试题与参考答案

### Q1：API调用中的流式输出（Streaming）原理是什么？

**参考答案**：

流式输出使用**SSE（Server-Sent Events）**协议实现服务器向客户端的实时推送：

**原理**：
1. 客户端发送请求时设置`stream: true`
2. 服务器保持HTTP连接打开
3. 模型每生成一个token，就通过SSE发送给客户端
4. 客户端实时显示，无需等待完整响应

**SSE格式**：
```
data: {"choices": [{"delta": {"content": "Hello"}}]}\n\n
data: {"choices": [{"delta": {"content": " world"}}]}\n\n
data: [DONE]\n\n
```

**优势**：
- 降低感知延迟（用户立即看到部分内容）
- 提升用户体验（类似打字机效果）
- 适合长文本生成

**实现要点**：
- 使用`requests.post(..., stream=True)`
- 逐行读取`response.iter_lines()`
- 处理`[DONE]`结束标记
- 需要HTTP/2支持高并发流式请求

### Q2：如何设计一个高可用的多平台API调用系统？

**参考答案**：

高可用多平台API调用系统的设计要点：

**1. 统一封装层**：
- 抽象接口屏蔽平台差异
- 支持动态切换平台
- 统一的错误码和返回格式

**2. 智能路由**：
- 基于任务特征选择平台
- 基于成本选择平台
- 基于延迟选择平台
- 权重轮询

**3. 健康检查**：
- 每30秒探测各平台
- 检查响应时间和错误率
- 自动标记不健康节点

**4. 故障转移**：
- 主平台故障 → 自动切换到备用
- 切换时间 < 5秒
- 备用链：主 → 备1 → 备2

**5. 熔断保护**：
- 错误率 > 50% → 熔断
- 熔断后5分钟尝试恢复
- 渐进式恢复（10%流量）

**6. 限流保护**：
- 客户端主动限流
- 指数退避重试
- 队列缓冲突发流量

**7. 成本监控**：
- 实时追踪各平台成本
- 预算告警
- 自动降级到便宜平台

**8. 缓存优化**：
- 缓存高频查询
- Embedding结果缓存
- 降低API调用次数

**架构**：
```
客户端 → API网关（认证、限流） → 智能路由 → 平台适配器 → LLM API
              ↑___________________________↓
                        （健康检查、故障转移）
```

### Q3：API调用中的成本控制策略有哪些？

**参考答案**：

成本控制策略分为五个层次：

**1. 模型分层**：
- 简单任务 → 小模型（GPT-4o-mini, GLM-4-Air）
- 中等任务 → 中模型（GPT-4o, Claude Sonnet）
- 复杂任务 → 大模型（GPT-5.5, Claude Opus，仅必要时）
- 节省：60-80%

**2. 缓存策略**：
- 缓存高频查询（Redis）
- Embedding结果缓存
- 响应缓存（TTL=1小时）
- 节省：20-40%

**3. 提示词优化**：
- 精简prompt（减少input tokens）
- 使用system message代替重复指令
- 压缩历史对话
- 节省：10-30%

**4. 输出控制**：
- 限制max_tokens
- 使用stop序列提前停止
- 避免不必要的生成
- 节省：10-20%

**5. 多平台比价**：
- 选择最便宜的可用平台
- DeepSeek比OpenAI便宜100-200倍
- 自动路由到低价平台
- 节省：50-90%

**监控措施**：
- 实时成本追踪
- 日/周/月成本报表
- 预算告警（80%, 100%）
- 成本归因分析

### Q4：如何处理API调用中的限流（Rate Limiting）？

**参考答案**：

限流处理策略：

**1. 客户端限流**：
- 主动控制请求速率
- 避免触发服务端限流
- 使用令牌桶或漏桶算法

**2. 指数退避重试**：
```python
for attempt in range(max_retries):
    try:
        response = api_call()
        break
    except RateLimitError:
        wait_time = backoff_factor ** attempt
        time.sleep(wait_time)
```

**3. 请求队列**：
- 将请求放入队列
- 按速率消费
- 平滑突发流量

**4. 负载均衡**：
- 将请求分散到多个API Key
- 使用多个账号
- 降低单个账号的限流风险

**5. 降级策略**：
- 限流时切换到备用平台
- 使用缓存结果
- 返回简化版答案

**6. 监控告警**：
- 监控429错误率
- 设置告警阈值
- 及时调整限流策略

### Q5：Function Calling的原理和实现方式是什么？

**参考答案**：

**Function Calling原理**：

1. **工具定义**：
   开发者定义可供模型调用的工具（函数），包括：
   - 函数名
   - 函数描述
   - 参数定义（JSON Schema）

2. **模型决策**：
   模型根据用户问题和工具定义，决定：
   - 是否需要调用工具
   - 调用哪个工具
   - 传递什么参数

3. **工具执行**：
   - 客户端收到工具调用请求
   - 执行对应的本地函数
   - 返回结果给模型

4. **最终回答**：
   - 模型基于工具结果生成最终答案

**实现流程**：
```python
# 1. 定义工具
tools = [{
    "type": "function",
    "function": {
        "name": "get_weather",
        "description": "查询天气",
        "parameters": {
            "type": "object",
            "properties": {
                "city": {"type": "string"}
            },
            "required": ["city"]
        }
    }
}]

# 2. 发送请求（包含工具定义）
response = client.chat.completions.create(
    model="gpt-4o",
    messages=messages,
    tools=tools,
    tool_choice="auto"
)

# 3. 检查是否需要调用工具
if response.choices[0].finish_reason == 'tool_calls':
    # 执行工具
    tool_call = response.choices[0].message.tool_calls[0]
    function_name = tool_call.function.name
    arguments = json.loads(tool_call.function.arguments)
    
    # 调用本地函数
    result = get_weather(**arguments)
    
    # 将结果返回给模型
    messages.append({
        "role": "tool",
        "tool_call_id": tool_call.id,
        "content": result
    })
    
    # 获取最终答案
    final_response = client.chat.completions.create(
        model="gpt-4o",
        messages=messages
    )
```

### Q6：2026年API调用的发展趋势是什么？

**参考答案**：

2026年API调用的发展趋势：

1. **多模态API普及**：
   - 文本+图像+音频+视频统一API
   - GPT-5.5、Gemini 2.0、GLM-5支持
   - 应用场景：视频分析、音频转录、图像生成

2. **实时API**：
   - 低延迟（<100ms）
   - WebSocket连接
   - 适用场景：实时对话、直播字幕

3. **Agent API**：
   - 模型自主规划和执行
   - 多步工具调用
   - 状态管理

4. **边缘部署**：
   - 小模型部署在边缘设备
   - 大模型在云端
   - 边缘-云端协同

5. **标准化协议**：
   - OpenAI兼容格式成为ISO标准
   - Function Calling协议统一
   - 工具市场生态

6. **成本透明化**：
   - 实时成本监控
   - 碳足迹追踪
   - 自动成本优化

7. **安全增强**：
   - 端到端加密
   - 零信任架构
   - 审计和合规

---

*此文原创，转载请注明出处。*
