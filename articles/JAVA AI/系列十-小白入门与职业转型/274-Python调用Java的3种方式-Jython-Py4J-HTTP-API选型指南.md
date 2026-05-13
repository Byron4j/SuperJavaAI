# Python 调用 Java 的 3 种方式：Jython/Py4J/HTTP API 选型指南，让两种语言协作

## 开篇：你不需要二选一

"做 AI 只能用 Python，那我写了十年的 Java 代码岂不是白费了？"

每次听到这个问题，我都想说：**谁告诉你必须二选一的？**

现实中大多数 AI 项目的架构如下：

```
┌─────────────────────────────────────────────┐
│                  用户请求                     │
└─────────────────┬───────────────────────────┘
                   │
         ┌─────────▼──────────┐
         │   Python FastAPI   │  ← AI 推理、模型调用
         │   (AI 层)          │
         └─────────┬──────────┘
                   │ 调用
         ┌─────────▼──────────┐
         │   Java Spring Boot │  ← 业务逻辑、数据管理
         │   (业务层)         │
         └─────────┬──────────┘
                   │
         ┌─────────▼──────────┐
         │   MySQL / Redis    │  ← 数据层
         └────────────────────┘
```

**Python 负责短平快的 AI 部分，Java 负责稳健的企业级后端**。各取所长，互不替代。

今天这篇文章，我介绍 Python 调用 Java 的 3 种方式，以及如何选型。每一种都附完整可运行的代码。

## 三种方式一览

| 方式 | 原理 | 延迟 | 复杂度 | 适用场景 |
|------|------|------|--------|---------|
| **Jython** | Python 语法运行在 JVM 上 | 低 | 中 | 简单脚本、复用 Java 类库 |
| **Py4J** | Socket 通信 + JVM 嵌入 | 低 | 中 | 需要频繁调用 Java 对象 |
| **HTTP API** | REST/gRPC 网络调用 | 高 | 低 | 微服务架构、跨机器协作 |

## 方式一：Jython — Python 语法 + JVM 运行

### Jython 是什么？

Jython = Python 语法 + 编译成 Java bytecode + 运行在 JVM 上。

你可以直接 `import` Java 类，就像它们是 Python 模块一样。

### 安装和 Hello World

```bash
# 下载 Jython（最新稳定版 2.7.3）
# https://www.jython.org/download
# 或者用 pip（只支持 Python 2.7 语法！这是最大硬伤）
pip install jython  # 不推荐

# 更推荐直接下载 standalone jar
wget https://repo1.maven.org/maven2/org/python/jython-standalone/2.7.3/jython-standalone-2.7.3.jar
```

```python
# hello_jython.py
# 直接导入 Java 类！
from java.util import ArrayList
from java.lang import System

# 使用 Java 的 ArrayList
list = ArrayList()
list.add("Hello")
list.add("from")
list.add("Jython")

System.out.println(list)  # 调用 Java 的 System.out.println

# 遍历
for item in list:
    print(item)
```

运行：
```bash
java -jar jython-standalone-2.7.3.jar hello_jython.py
```

### 调用你自定义的 Java 类

```java
// Calculator.java — 你要复用的 Java 业务代码
package com.example;

public class Calculator {
    public int add(int a, int b) {
        return a + b;
    }
    
    public double calculateDiscount(double price, double discountRate) {
        return price * (1 - discountRate);
    }
    
    public String generateReport(String userName, double totalAmount) {
        return String.format("用户 %s 的消费总额为 %.2f 元", userName, totalAmount);
    }
}
```

```bash
# 编译 Java 类
javac com/example/Calculator.java
```

```python
# ai_script.py — Python 中调用 Java 类
import sys
sys.path.append(".")  # 把当前目录加入 classpath

from com.example import Calculator

# 像使用 Python 类一样使用 Java 类！
calc = Calculator()

# 调用 Java 方法
result = calc.add(10, 20)
print(f"10 + 20 = {result}")

# 复杂计算
final_price = calc.calculateDiscount(299.99, 0.15)
print(f"折扣后价格: {final_price:.2f}")

# 调用返回字符串的方法
report = calc.generateReport("张三", 5999.00)
print(report)
```

运行：
```bash
java -cp jython-standalone-2.7.3.jar:. org.python.util.jython ai_script.py
```

### Jython 的致命缺陷

```python
# Jython 2.7.x 只支持 Python 2.7 语法！！！
# 这意味着你不能用：
# - f-string（f"hello {name}"）不支持
# - type hints 不支持
# - asyncio 不支持
# - 几乎所有现代 Python 库（numpy, pandas, fastapi）都不支持

# 如果你只是想用 Python 语法操作 Java 对象 → Jython 可以
# 如果你想用现代 Python 生态（AI/ML库）→ Jython 不行
```

### 适用和不适用场景

**适用**：
- 你有一堆 Java 工具类，想用 Python 脚本快速调用
- 自动化测试 Java 代码
- 简单的胶水代码

**不适用**：
- 任何需要现代 Python 库的场景（AI、Web、数据处理）
- 生产环境的复杂应用

## 方式二：Py4J — 让 Python 和 JVM 做邻居

### Py4J 的原理

Py4J 在 Python 进程中启动一个**网关**，通过 Socket 与 JVM 进程通信。

```
┌──────────────┐     Socket      ┌──────────────┐
│  Python 进程  │◄──────────────►│   JVM 进程    │
│              │   localhost:    │              │
│  py4j 客户端  │   25333        │  py4j 服务端  │
└──────────────┘                 └──────────────┘
```

### 安装

```bash
pip install py4j
```

### 从 Python 调用 Java

**Java 端**：

```java
// GatewayServer.java — 启动 Py4J 网关
import py4j.GatewayServer;

public class GatewayServer {
    
    private Calculator calculator = new Calculator();
    
    // 这些方法会自动暴露给 Python
    public Calculator getCalculator() {
        return calculator;
    }
    
    public String greet(String name) {
        return "你好，" + name + "！(来自Java)";
    }
    
    public static void main(String[] args) {
        GatewayServer server = new GatewayServer();
        GatewayServer gateway = new GatewayServer(server);
        gateway.start();
        System.out.println("Py4J 网关已启动...");
    }
}
```

```java
// Calculator.java
public class Calculator {
    public int add(int a, int b) {
        return a + b;
    }
    
    public int multiply(int a, int b) {
        return a * b;
    }
}
```

**Python 端**：

```python
# ai_caller.py — Python 通过 Py4J 调用 Java
from py4j.java_gateway import JavaGateway

# 连接到 JVM（需要先启动 Java 的 GatewayServer）
gateway = JavaGateway()

# 获取 Java 端的入口对象
server = gateway.entry_point

# 调用 Java 方法
greeting = server.greet("张三")
print(greeting)  # 你好，张三！(来自Java)

# 获取 Java 对象，调用其方法
calc = server.getCalculator()
print(calc.add(10, 20))       # 30
print(calc.multiply(6, 7))    # 42

# 甚至可以直接使用 Java 标准库！
from py4j.java_gateway import java_import

# 导入 Java 类
java_import(gateway.jvm, 'java.util.ArrayList')
java_import(gateway.jvm, 'java.util.HashMap')

# 使用 Java 集合
list = gateway.jvm.ArrayList()
list.add("苹果")
list.add("香蕉")
print(list)  # [苹果, 香蕉]

map = gateway.jvm.HashMap()
map.put("name", "张三")
map.put("age", 30)
print(map.get("name"))  # 张三
```

### 实战：AI 推荐引擎调用 Java 业务逻辑

一个更实际的场景：Python 做 AI 推荐，Java 处理用户数据和业务规则。

**Java 端**：

```java
// BusinessService.java
package com.example;

import java.util.*;
import java.util.stream.Collectors;

public class BusinessService {
    
    // 模拟数据库中的用户数据
    private Map<String, UserProfile> userDb = new HashMap<>();
    
    public BusinessService() {
        userDb.put("user001", new UserProfile("user001", "张三", 28, 
            Arrays.asList("科技", "编程", "AI")));
        userDb.put("user002", new UserProfile("user002", "李四", 35,
            Arrays.asList("金融", "投资", "理财")));
    }
    
    // 获取用户画像（Python 调用这个）
    public UserProfile getUserProfile(String userId) {
        return userDb.get(userId);
    }
    
    // 业务规则过滤（Java 擅长的逻辑）
    public List<String> filterByBusinessRules(List<String> candidates, 
                                               UserProfile user) {
        // 例如：VIP用户看高价内容，普通用户看免费内容
        // 复杂的 if-else 逻辑在 Java 里写比 Python 清晰
        return candidates.stream()
            .filter(item -> !item.contains("VIP") || user.isVip())
            .collect(Collectors.toList());
    }
    
    // 记录推荐日志
    public void logRecommendation(String userId, List<String> recommendations) {
        System.out.printf("[%s] 向用户 %s 推荐了: %s%n", 
            new Date(), userId, String.join(", ", recommendations));
    }
}

class UserProfile {
    private String userId;
    private String name;
    private int age;
    private List<String> interests;
    private boolean isVip = false;
    
    // 构造器、getter 省略...
    public UserProfile(String userId, String name, int age, List<String> interests) {
        this.userId = userId;
        this.name = name;
        this.age = age;
        this.interests = interests;
    }
    
    public List<String> getInterests() { return interests; }
    public String getName() { return name; }
    public boolean isVip() { return isVip; }
}
```

**Python 端（AI 部分）**：

```python
# ai_recommender.py
from py4j.java_gateway import JavaGateway
import numpy as np

class AIRecommender:
    """AI 推荐引擎——Python 做它擅长的事"""
    
    def __init__(self):
        self.gateway = JavaGateway()
        self.business_service = self.gateway.entry_point.getBusinessService()
    
    def recommend(self, user_id: str, top_k: int = 5):
        """为指定用户生成推荐"""
        
        # Step 1: 通过 Py4J 从 Java 获取用户画像
        user_profile = self.business_service.getUserProfile(user_id)
        if not user_profile:
            return {"error": f"用户 {user_id} 不存在"}
        
        user_name = user_profile.getName()
        interests = list(user_profile.getInterests())
        print(f"为用户 {user_name} 生成推荐，兴趣标签: {interests}")
        
        # Step 2: Python 做 AI 推理（这里模拟向量相似度计算）
        all_candidates = self._get_candidates()
        scored = self._score_by_interest(interests, all_candidates)
        
        # Step 3: 通过 Py4J 让 Java 做业务规则过滤
        candidates_list = self.gateway.jvm.java.util.ArrayList()
        for item, score in scored[:top_k * 2]:
            candidates_list.add(item)
        
        filtered = self.business_service.filterByBusinessRules(
            candidates_list, user_profile
        )
        
        # Step 4: 返回结果，让 Java 记录日志
        final_recommendations = list(filtered)[:top_k]
        self.business_service.logRecommendation(user_id, 
            self.gateway.jvm.java.util.Arrays.asList(final_recommendations))
        
        return {
            "user": user_name,
            "recommendations": final_recommendations
        }
    
    def _get_candidates(self):
        """模拟内容库"""
        return [
            "Python入门教程", "Java高级编程", "AI从零开始",
            "股票投资指南", "基金定投策略", "区块链技术",
            "VIP-深度强化学习", "VIP-量化交易实战", "前端框架对比"
        ]
    
    def _score_by_interest(self, interests, candidates):
        """简单的基于关键词的评分（实际会用 Embedding 向量相似度）"""
        scored = []
        for item in candidates:
            score = sum(1 for tag in interests if tag in item)
            scored.append((item, score))
        return sorted(scored, key=lambda x: x[1], reverse=True)


# 使用
if __name__ == "__main__":
    recommender = AIRecommender()
    result = recommender.recommend("user001")
    print(f"\n推荐结果：{result}")
```

### Py4J 的优缺点

**优点**：
- Python 可以用**任何**现代 Python 库（numpy/pandas/pytorch/fastapi）
- 延迟低（本地 Socket 通信）
- Java 对象可以像 Python 对象一样操作

**缺点**：
- 需要启动两个进程（Python + JVM）
- 调试困难（跨进程断点调试很痛苦）
- Java 端需要引入 py4j 依赖
- 不适合跨机器部署（如果 Java 服务在另一台服务器上）

## 方式三：HTTP API — 最通用、最解耦的方式

### 为什么 HTTP API 是最优选？

如果你已经在用 Spring Boot，这是**最自然**的选择。不需要引入任何新库，不需要处理进程间通信的复杂性。

```
┌──────────────┐   HTTP/JSON    ┌──────────────┐
│  Python      │◄─────────────►│  Java        │
│  FastAPI     │   REST API     │  Spring Boot │
│  (AI层)      │                │  (业务层)     │
└──────────────┘                └──────────────┘
```

### Java 端：提供 REST API

```java
// UserController.java
@RestController
@RequestMapping("/api/internal/users")
public class UserController {
    
    @Autowired
    private UserService userService;
    
    @GetMapping("/{userId}/profile")
    public UserProfileDTO getUserProfile(@PathVariable String userId) {
        return userService.getProfile(userId);
    }
    
    @PostMapping("/recommendations/filter")
    public FilterResultDTO filterRecommendations(@RequestBody FilterRequest request) {
        return userService.filterByRules(request);
    }
    
    @PostMapping("/recommendations/log")
    public void logRecommendation(@RequestBody LogRequest request) {
        userService.log(request);
    }
}
```

### Python 端：调用 Java API

```python
# ai_service.py
import httpx
from typing import List, Optional
from pydantic import BaseModel

class UserProfile(BaseModel):
    user_id: str
    name: str
    age: int
    interests: List[str]

class AIRecommendationService:
    """AI 推荐服务——只负责 AI 逻辑，业务数据通过 API 获取"""
    
    def __init__(self, java_base_url: str = "http://localhost:8080"):
        self.java_api = java_base_url
        # 使用 httpx 的异步客户端
        self.client = httpx.AsyncClient(base_url=java_base_url, timeout=30.0)
    
    async def get_user_profile(self, user_id: str) -> Optional[UserProfile]:
        """从 Java 服务获取用户画像"""
        try:
            response = await self.client.get(
                f"/api/internal/users/{user_id}/profile"
            )
            response.raise_for_status()
            return UserProfile(**response.json())
        except httpx.HTTPError as e:
            print(f"获取用户画像失败: {e}")
            return None
    
    async def filter_by_business_rules(
        self, candidates: List[str], user_id: str
    ) -> List[str]:
        """调用 Java 的业务规则过滤"""
        try:
            response = await self.client.post(
                "/api/internal/recommendations/filter",
                json={"user_id": user_id, "candidates": candidates}
            )
            response.raise_for_status()
            return response.json()["filtered"]
        except httpx.HTTPError as e:
            print(f"业务规则过滤失败: {e}")
            return candidates  # 降级：不过滤，全返回
    
    async def recommend(self, user_id: str) -> dict:
        """完整的推荐流程"""
        
        # 1. 从 Java 获取用户数据
        profile = await self.get_user_profile(user_id)
        if not profile:
            return {"error": "用户不存在"}
        
        # 2. Python 做 AI 推理（向量相似度计算等）
        raw_recommendations = await self._ai_scoring(profile.interests)
        
        # 3. 让 Java 做业务规则过滤
        filtered = await self.filter_by_business_rules(
            raw_recommendations, user_id
        )
        
        # 4. 记录日志（异步调用，不阻塞主流程）
        await self.client.post(
            "/api/internal/recommendations/log",
            json={"user_id": user_id, "items": filtered}
        )
        
        return {
            "user": profile.name,
            "recommendations": filtered[:10]
        }
    
    async def _ai_scoring(self, interests: List[str]) -> List[str]:
        """AI 评分排序（实际会调用 OpenAI API 或本地模型）"""
        import asyncio
        await asyncio.sleep(0.1)  # 模拟 AI 推理耗时
        # 实际逻辑：用 Embedding 向量做相似度检索
        return ["item1", "item2", "item3"]
```

### HTTP API 方式的优缺点

**优点**：
- 完全解耦：Java 和 Python 可以独立部署、独立升级
- 跨机器：服务可以部署在不同服务器上
- 标准协议：任何语言都能调用，不限于 Python↔Java
- 已有基础设施：监控、日志、限流、熔断都可以复用
- 不需要引入额外依赖

**缺点**：
- 网络延迟（但内网调用通常 <5ms，可以忽略）
- 序列化开销（JSON 序列化/反序列化）
- 需要维护两套代码

### 进阶：用 gRPC 降低延迟

如果 HTTP 的延迟对你来说太高（比如每秒几千次调用的高频场景），可以用 gRPC：

```protobuf
// recommendation.proto
service RecommendationService {
    rpc GetUserProfile(GetUserProfileRequest) returns (UserProfile);
    rpc FilterRecommendations(FilterRequest) returns (FilterResponse);
}
```

Java 端用 `grpc-java`，Python 端用 `grpcio`，性能比 HTTP+JSON 高 3-5 倍。

## 选型决策图

```
                              你需要调用 Java 代码
                                     │
                    ┌────────────────┼────────────────┐
                    │                │                │
              只是简单脚本？     需要频繁调用？      微服务架构？
                    │                │                │
                    ▼                ▼                ▼
              Java 类可以不     Python/JVM       Java 独立部署？
              重新编译？        同机部署？              │
                    │                │                │
              ┌───┬───┐        ┌───┬───┐        ┌───┬───┐
              │   │   │        │   │   │        │   │   │
             是   否  │       是   否  │       是   否  │
              │   │   │        │   │   │        │   │   │
              ▼   ▼   ▼        ▼   ▼   ▼        ▼   ▼   ▼
           Jython  ❌     Py4J  HTTP  ❌    HTTP  ❌ 不适用
           （仅限      （推荐）(也可以)       gRPC
            Python2.7）                    （推荐）
```

## 总结：三种方式一句话概括

| 方式 | 一句话建议 |
|------|-----------|
| **Jython** | 忘掉它。除非你维护的是一套 Python 2.7 的老项目 |
| **Py4J** | 适合本地开发工具、自动化测试、单机部署的中型项目 |
| **HTTP API** | **生产环境首选**。最解耦、最灵活、最适合团队协作 |

如果你现在启动一个新项目，直接用 HTTP API 方案。Spring Boot 提供业务 API，FastAPI 提供 AI 推理 API，通过 HTTP 互相调用。这是工业界验证过的最成熟方案。

**你不需要放弃 Java，你只需要给它配一个 Python 兄弟。**

---

**下篇预告**：理论说得够多了，来写代码！下一篇，你只需要 20 行 Java 代码就能调用 OpenAI API，让 ChatGPT 在你的代码里聊天。你甚至不需要学 Python！
