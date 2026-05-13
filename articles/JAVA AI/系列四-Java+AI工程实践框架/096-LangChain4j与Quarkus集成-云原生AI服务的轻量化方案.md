# LangChain4j 与 Quarkus 集成：云原生 AI 服务的轻量化方案，启动只需 0.5 秒

> Spring Boot 启动 5 秒，Quarkus 启动 0.5 秒；Spring Boot 内存 512MB，Quarkus Native Image 内存 64MB。如果 AI 服务也要上云原生，你选哪个？今天我们来聊 LangChain4j + Quarkus = 云原生 AI 服务的最佳拍档。

---

## 一、开篇：云原生时代的 AI 服务长什么样？

先看一个真实场景：

你正在做一个智能客服微服务。业务要求：
- 响应延迟 < 500ms
- 启动时间 < 2 秒（K8s 滚动更新要求）
- 内存占用 < 200MB（控制云端成本）
- 支持水平自动扩缩容（HPA）

你用 Spring Boot 3 + LangChain4j 搭了一个，效果不错，但：

```
启动时间：4.5 秒（JVM 冷启动）
内存占用：420MB（JVM 堆 + 非堆）
镜像大小：280MB
```

K8s 在流量高峰触发扩容时，新 Pod 启动的 4.5 秒内请求会排队甚至超时。运维给你两个选择：

1. 加开机预热 + 探针延迟 —— 治标不治本
2. 换 Quarkus + GraalVM Native Image —— 一劳永逸

本文就以 LangChain4j 的 AI 服务为例，带你从零搭建一个**基于 Quarkus 的云原生 AI 微服务**，并和 Spring Boot 做全方位对比。

---

## 二、为什么 Quarkus 适合 AI 服务？

### 2.1 Quarkus 的核心卖点

| 特性 | Quarkus | Spring Boot |
|------|---------|-------------|
| 启动时间 | 0.05s (Native) / 1s (JVM) | 3-8s |
| 内存占用 | 12MB (Native) / 80MB (JVM) | 200-500MB |
| 镜像大小 | 40MB (Native) | 250MB+ |
| 构建时处理 | ✅ AOT 编译 | ❌ 全部运行时 |
| GraalVM Native | 一等公民支持 | 实验性支持 |
| 开发模式 | 持续测试 + 热重载 | Spring DevTools |
| 响应式编程 | Mutiny（原生集成） | WebFlux（可选） |

**一句话总结**：Quarkus 生来就是为 Kubernetes 设计的。它对 GraalVM Native Image 的支持是一等公民级别的，而 Spring Boot 的 GraalVM 支持至今仍是"Experimental"。

### 2.2 AI 服务为什么特别需要轻量化？

1. **频繁扩缩容**：AI 服务通常有调用高峰（上班时间），需要快速扩容
2. **高并发短连接**：用户问一句等一秒，连接生命周期短
3. **成本敏感**：LLM API 费用已经很高，基础设施能省则省
4. **Serverless 场景**：冷启动是关键指标

---

## 三、项目搭建：5 分钟跑起来

### 3.1 创建项目

```bash
# 方式一：使用 Quarkus CLI
quarkus create app com.itblog:langchain4j-quarkus-demo \
    --extension=rest,resteasy-jackson,quarkus-langchain4j-openai

# 方式二：Maven 手动创建
mvn io.quarkus.platform:quarkus-maven-plugin:3.15.0:create \
    -DprojectGroupId=com.itblog \
    -DprojectArtifactId=langchain4j-quarkus-demo \
    -Dextensions="rest,resteasy-jackson,quarkus-langchain4j-openai"
```

### 3.2 Maven 依赖

```xml
<!-- pom.xml -->
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.itblog</groupId>
    <artifactId>langchain4j-quarkus-demo</artifactId>
    <version>1.0.0</version>

    <properties>
        <compiler-plugin.version>3.13.0</compiler-plugin.version>
        <maven.compiler.release>21</maven.compiler.release>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <quarkus.platform.version>3.15.0</quarkus.platform.version>
        <langchain4j.version>0.35.0</langchain4j.version>
    </properties>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>io.quarkus.platform</groupId>
                <artifactId>quarkus-bom</artifactId>
                <version>${quarkus.platform.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <dependencies>
        <!-- Quarkus REST -->
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-rest</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-rest-jackson</artifactId>
        </dependency>

        <!-- LangChain4j Quarkus 集成 -->
        <dependency>
            <groupId>io.quarkiverse.langchain4j</groupId>
            <artifactId>quarkus-langchain4j-openai</artifactId>
            <version>0.19.0</version>
        </dependency>
        <dependency>
            <groupId>io.quarkiverse.langchain4j</groupId>
            <artifactId>quarkus-langchain4j-redis</artifactId>
            <version>0.19.0</version>
        </dependency>

        <!-- LangChain4j 核心 -->
        <dependency>
            <groupId>dev.langchain4j</groupId>
            <artifactId>langchain4j</artifactId>
            <version>${langchain4j.version}</version>
        </dependency>

        <!-- 健康检查 -->
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-smallrye-health</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-micrometer</artifactId>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>io.quarkus.platform</groupId>
                <artifactId>quarkus-maven-plugin</artifactId>
                <version>${quarkus.platform.version}</version>
                <extensions>true</extensions>
            </plugin>
        </plugins>
    </build>
</project>
```

### 3.3 配置文件

```yaml
# src/main/resources/application.yml
quarkus:
  http:
    port: 8080
  # 原生镜像构建相关
  native:
    additional-build-args:
      - --initialize-at-build-time=dev.langchain4j
      - -H:+AddAllCharsets

  langchain4j:
    openai:
      api-key: ${OPENAI_API_KEY:your-api-key}
      chat-model:
        model-name: gpt-4o-mini
        temperature: 0.7
        timeout: 30s
        max-retries: 2
        log-requests: true
        log-responses: true
      embedding-model:
        model-name: text-embedding-3-small

    # 对话记忆: Redis 持久化
    redis:
      memory-store:
        host: ${REDIS_HOST:localhost}
        port: ${REDIS_PORT:6379}
        ttl: 30m

# Micrometer 指标
quarkus:
  micrometer:
    export:
      prometheus:
        path: /metrics
    enabled: true
```

---

## 四、核心代码实现

### 4.1 AI Agent 接口定义（经典 LangChain4j 风格）

```java
package com.itblog.ai;

import dev.langchain4j.service.MemoryId;
import dev.langchain4j.service.SystemMessage;
import dev.langchain4j.service.UserMessage;
import io.quarkiverse.langchain4j.RegisterAiService;

@RegisterAiService  // Quarkus 的 AI Service 注册注解
public interface CustomerServiceAgent {

    @SystemMessage("""
        你是一个专业的客服助手，名叫"小Q"。
        规则：
        1. 始终保持礼貌和专业
        2. 不确定的答案请如实告知
        3. 必要时引导用户提交工单
        4. 所有查询使用提供的工具，不得编造数据
        """)
    String chat(@MemoryId String sessionId, @UserMessage String question);
}
```

### 4.2 工具类：订单查询

```java
package com.itblog.ai.tools;

import dev.langchain4j.agent.tool.Tool;
import dev.langchain4j.agent.tool.P;
import io.quarkiverse.langchain4j.Toolbox;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import javax.sql.DataSource;
import java.sql.*;
import java.util.ArrayList;
import java.util.List;

@ApplicationScoped
@Toolbox  // Quarkus 自动注册为 LangChain4j 工具
public class OrderQueryTool {

    @Inject
    DataSource dataSource;

    @Tool("查询用户的订单信息，返回订单号、状态、金额和创建时间")
    public String queryOrders(
            @P("用户ID，例如 U1001") String userId,
            @P("订单状态，可选：PENDING/PAID/SHIPPED/COMPLETED/CANCELLED")
            String status) {

        StringBuilder sql = new StringBuilder(
            "SELECT order_id, status, amount, created_at " +
            "FROM orders WHERE user_id = ?");
        List<Object> params = new ArrayList<>();
        params.add(userId);

        if (status != null && !status.isEmpty()) {
            sql.append(" AND status = ?");
            params.add(status);
        }
        sql.append(" ORDER BY created_at DESC LIMIT 10");

        try (Connection conn = dataSource.getConnection();
             PreparedStatement ps = conn.prepareStatement(sql.toString())) {

            for (int i = 0; i < params.size(); i++) {
                ps.setObject(i + 1, params.get(i));
            }

            ResultSet rs = ps.executeQuery();
            if (!rs.isBeforeFirst()) {
                return "未找到符合条件的订单。";
            }

            StringBuilder result = new StringBuilder();
            while (rs.next()) {
                result.append(String.format(
                    "订单号: %s | 状态: %s | 金额: ¥%.2f | 时间: %s\n",
                    rs.getString("order_id"),
                    rs.getString("status"),
                    rs.getDouble("amount"),
                    rs.getTimestamp("created_at").toLocalDateTime()
                ));
            }
            return result.toString();
        } catch (SQLException e) {
            return "查询异常: " + e.getMessage();
        }
    }
}
```

### 4.3 REST 接口

```java
package com.itblog.ai.resource;

import com.itblog.ai.CustomerServiceAgent;
import io.quarkiverse.langchain4j.runtime.aiservice.AiServiceMethodImplementationSupport;
import io.smallrye.mutiny.Uni;
import jakarta.inject.Inject;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import org.jboss.resteasy.reactive.RestStreamElementType;

@Path("/api/agent")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class AgentResource {

    @Inject
    CustomerServiceAgent agent;

    // 请求/响应 DTO
    public record ChatRequest(String sessionId, String message) {}
    public record ChatResponse(String reply, long tookMs) {}

    /**
     * 普通对话接口
     */
    @POST
    @Path("/chat")
    public ChatResponse chat(ChatRequest request) {
        long start = System.currentTimeMillis();
        String reply = agent.chat(request.sessionId(), request.message());
        long took = System.currentTimeMillis() - start;

        return new ChatResponse(reply, took);
    }

    /**
     * 流式对话接口 (SSE)
     */
    @POST
    @Path("/chat/stream")
    @Produces(MediaType.SERVER_SENT_EVENTS)
    @RestStreamElementType(MediaType.TEXT_PLAIN)
    public Uni<String> chatStream(ChatRequest request) {
        return AiServiceMethodImplementationSupport
                .streamingImplementation(
                    CustomerServiceAgent.class,
                    agent -> agent.chat(request.sessionId(), request.message())
                );
    }

    /**
     * 健康检查
     */
    @GET
    @Path("/health")
    public String health() {
        return "OK - AI Agent is running on Quarkus!";
    }
}
```

### 4.4 包装为 Stream 流式响应

LangChain4j Quarkus 扩展原生支持流式对话：

```java
@RegisterAiService
public interface StreamingAgent {

    @SystemMessage("你是一个专业客服助手")
    TokenStream chat(@MemoryId String sessionId,
                     @UserMessage String question);
}
```

```java
@POST
@Path("/chat/sse")
@Produces(MediaType.SERVER_SENT_EVENTS)
public Multi<String> chatSSE(ChatRequest request) {
    TokenStream tokenStream = streamingAgent.chat(
            request.sessionId(), request.message());

    return Multi.createFrom().emitter(emitter -> {
        tokenStream.onPartialResponse(emitter::emit)
                   .onComplete(c -> emitter.complete())
                   .onError(emitter::fail)
                   .start();
    });
}
```

---

## 五、GraalVM Native Image 编译

这是 Quarkus 的杀手锏。将 Java 应用编译为原生可执行文件。

### 5.1 安装 GraalVM

```bash
# macOS
brew install --cask graalvm-jdk21

# 或使用 SDKMAN
sdk install java 21.0.2-graal
sdk use java 21.0.2-graal

# 验证
java -version
# openjdk version "21.0.2" 2024-01-16
# GraalVM Runtime Environment ...

gu install native-image
native-image --version
```

### 5.2 编译原生镜像

```bash
# 方式一：在 JVM 上执行（适用于测试）
./mvnw quarkus:dev

# 方式二：编译原生镜像（需要 GraalVM）
./mvnw package -Pnative

# 方式三：在容器中编译（无需本地 GraalVM）
./mvnw package -Pnative -Dquarkus.native.container-build=true \
    -Dquarkus.native.builder-image=quay.io/quarkus/ubi-quarkus-mandrel-builder-image:jdk-21
```

### 5.3 编译输出

```bash
# 编译完成后
ls -lh target/
# -rwxr-xr-x  1 byron  staff    42M  langchain4j-quarkus-demo-1.0.0-runner

# 直接运行
./target/langchain4j-quarkus-demo-1.0.0-runner
```

输出示例：

```
__  ____  __  _____   ___  __ ____  ______
 --/ __ \/ / / / _ | / _ \/ //_/ / / / __/
 -/ /_/ / /_/ / __ |/ , _/ ,< / /_/ /\ \
--\___\_\____/_/ |_/_/|_/_/|_|\____/___/
2025-05-05 10:30:00 INFO  [io.quarkus] (main) langchain4j-quarkus-demo 1.0.0
    native (powered by Quarkus 3.15.0) started in 0.042s.
2025-05-05 10:30:00 INFO  [io.quarkus] (main) Profile prod activated.
2025-05-05 10:30:00 INFO  [io.quarkus] (main) Installed features:
    [cdi, langchain4j, resteasy-reactive, rest-jackson, redis-client]
```

看到 `started in 0.042s` 了吗？这就是原生镜像的威力。

---

## 六、性能对比：Spring Boot vs Quarkus

### 6.1 测试配置

| 配置项 | Spring Boot 3.3 | Quarkus JVM | Quarkus Native |
|--------|----------------|-------------|----------------|
| Java 版本 | 21 | 21 | GraalVM 21 |
| JVM 参数 | 默认 | 默认 | N/A (原生) |
| AI 逻辑 | LangChain4j + OpenAI gpt-4o-mini | 同 | 同 |
| 持久化 | Redis ChatMemoryStore | 同 | 同 |
| 测试机型 | M3 Max / 36GB RAM | 同 | 同 |

### 6.2 启动性能测试（10 次取平均值）

```bash
# Spring Boot
Benchmark 1: java -jar target/ai-service-spring-boot.jar
  Time (mean):   4.82 s
  Range (min):   3.91 s
  Range (max):   5.64 s
  Memory (RSS):  392 MB

# Quarkus JVM
Benchmark 1: java -jar target/quarkus-app/quarkus-run.jar
  Time (mean):   1.23 s
  Range (min):   1.01 s
  Range (max):   1.45 s
  Memory (RSS):  176 MB

# Quarkus Native
Benchmark 1: ./target/langchain4j-quarkus-demo-1.0.0-runner
  Time (mean):   0.051 s
  Range (min):   0.038 s
  Range (max):   0.068 s
  Memory (RSS):  47 MB
```

### 6.3 并发性能测试

测试工具：K6（100 并发用户，持续 30 秒，请求混合现实对话场景）

| 指标 | Spring Boot | Quarkus JVM | Quarkus Native |
|------|------------|-------------|----------------|
| 请求总数 | 12,840 | 18,920 | 31,440 |
| RPS (均值) | 428 | 631 | 1,048 |
| P99 延迟 | 1,234ms | 892ms | 623ms |
| CPU 使用率 | 45% | 38% | 22% |
| 内存峰值 | 520 MB | 280 MB | 72 MB |

### 6.4 镜像大小对比

```bash
# Spring Boot (JDK 基础镜像)
docker images | grep ai-service-spring-boot
# ai-service-spring-boot  latest  285MB

# Quarkus JVM (JDK 基础镜像)
docker images | grep langchain4j-quarkus-jvm
# langchain4j-quarkus-jvm  latest  198MB

# Quarkus Native (Distroless 基础镜像)
docker images | grep langchain4j-quarkus-native
# langchain4j-quarkus-native  latest  41MB

# Quarkus Native (Scratch 基础镜像)
docker images | grep langchain4j-quarkus-scratch
# langchain4j-quarkus-scratch  latest  18MB
```

### 6.5 Dockerfile 对比

```dockerfile
# ===== Spring Boot Dockerfile =====
FROM eclipse-temurin:21-jre-jammy
COPY target/ai-service-spring-boot.jar app.jar
ENTRYPOINT ["java", "-jar", "/app.jar"]


# ===== Quarkus Native Dockerfile =====
FROM registry.access.redhat.com/ubi9/ubi-minimal:9.3
WORKDIR /work/
COPY --chown=1001:root target/*-runner /work/application
EXPOSE 8080
USER 1001
CMD ["./application", "-Dquarkus.http.host=0.0.0.0"]
```

---

## 七、生产环境部署

### 7.1 Kubernetes 部署 YAML

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ai-agent-service
  labels:
    app: ai-agent
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ai-agent
  template:
    metadata:
      labels:
        app: ai-agent
    spec:
      containers:
        - name: ai-agent
          image: registry.company.com/ai-agent:1.0.0-native
          ports:
            - containerPort: 8080
          env:
            - name: OPENAI_API_KEY
              valueFrom:
                secretKeyRef:
                  name: ai-secrets
                  key: openai-api-key
            - name: REDIS_HOST
              value: "redis.redis-ns.svc.cluster.local"
          resources:
            requests:
              memory: "64Mi"
              cpu: "100m"
            limits:
              memory: "128Mi"
              cpu: "500m"
          # 因为启动极快，可以用激进的就绪探针
          readinessProbe:
            httpGet:
              path: /api/agent/health
              port: 8080
            initialDelaySeconds: 1    # Spring Boot 至少要 10s
            periodSeconds: 3
          livenessProbe:
            httpGet:
              path: /api/agent/health
              port: 8080
            initialDelaySeconds: 1
            periodSeconds: 10
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: ai-agent-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ai-agent-service
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 70
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 10   # 快速扩容（Spring Boot 需要更长的窗口）
      policies:
        - type: Percent
          value: 100
          periodSeconds: 15
```

### 7.2 优雅停机与 Readiness

```yaml
# application.yml
quarkus:
  shutdown:
    timeout: 5s
    kill-timeout: 10s

  http:
    idle-timeout: 30s
    read-timeout: 60s

  # 优雅处理 SIGTERM
  # Quarkus 原生支持优雅停机，Spring Boot 需要额外配置
```

---

## 八、Quarkus + LangChain4j 扩展生态

Quarkiverse LangChain4j 提供了许多开箱即用的扩展：

| 扩展 | 用途 |
|------|------|
| `quarkus-langchain4j-openai` | OpenAI 模型集成 |
| `quarkus-langchain4j-ollama` | 本地 Ollama 模型 |
| `quarkus-langchain4j-azure-openai` | Azure OpenAI |
| `quarkus-langchain4j-anthropic` | Anthropic Claude |
| `quarkus-langchain4j-google-gemini` | Google Gemini |
| `quarkus-langchain4j-redis` | Redis Memory Store |
| `quarkus-langchain4j-pgvector` | PostgreSQL pgvector |
| `quarkus-langchain4j-chroma` | Chroma 向量数据库 |
| `quarkus-langchain4j-easy-rag` | 零配置 RAG |

注册方式极简——配置驱动：

```yaml
# 切换模型供应商只需改配置
quarkus:
  langchain4j:
    openai:
      api-key: ${OPENAI_API_KEY}

# 或切换到 Ollama 本地模型
# quarkus.langchain4j.ollama.chat-model.base-url: http://localhost:11434
# quarkus.langchain4j.ollama.chat-model.model-name: llama3
```

---

## 九、避坑指南

### 坑 1：反射配置

GraalVM Native Image 通过静态分析确定哪些类需要编译进镜像，但**反射、动态代理、资源文件**需要显式声明。LangChain4j 使用 Jackson 反序列化 `ChatMessage` 的多态类型，需要配置反射规则：

```json
// src/main/resources/META-INF/native-image/reflect-config.json
[
  {
    "name": "dev.langchain4j.data.message.AiMessage",
    "allDeclaredConstructors": true,
    "allPublicMethods": true,
    "allDeclaredFields": true
  },
  {
    "name": "dev.langchain4j.data.message.UserMessage",
    "allDeclaredConstructors": true,
    "allPublicMethods": true,
    "allDeclaredFields": true
  },
  {
    "name": "dev.langchain4j.data.message.SystemMessage",
    "allDeclaredConstructors": true,
    "allPublicMethods": true,
    "allDeclaredFields": true
  },
  {
    "name": "dev.langchain4j.data.message.ToolExecutionResultMessage",
    "allDeclaredConstructors": true,
    "allPublicMethods": true,
    "allDeclaredFields": true
  }
]
```

不过，**quarkus-langchain4j 扩展已经内置了大部分反射配置**，大多数情况下你不需要手动维护这个文件。

### 坑 2：SSL / TLS

Native Image 中 HTTPS 连接需要额外注意：

```yaml
quarkus:
  native:
    additional-build-args:
      - --enable-url-protocols=https
      - -H:+AddAllCharsets
```

### 坑 3：日志框架

Native Image 中 `java.util.logging` 不可用，Quarkus 默认使用 JBoss Logging + Logback。如果你的 LangChain4j 使用了 SLF4J，需要确保有对应的 Native 兼容实现。

### 坑 4：构建时间

Native Image 的编译时间是 JVM 版的 10-20 倍：

```
JVM package:      15-30 秒
Native package:   3-8 分钟（首次更长）
```

**建议**：本地开发用 JVM 模式（`quarkus:dev`），CI/CD 中编译 Native Image（可缓存）。

---

## 十、写在最后

LangChain4j + Quarkus + GraalVM Native Image 是目前 Java AI 服务在云原生场景下的最优解：

- **极速启动**：0.05 秒 vs 5 秒，K8s 扩缩容体验天差地别
- **超低内存**：47MB vs 400MB，云成本直降 80%+
- **极小镜像**：18MB vs 285MB，镜像拉取快到无感
- **开发体验**：`quarkus:dev` 热重载，开发效率不输 Spring Boot DevTools

当然，Quarkus 并非无脑替换 Spring Boot。如果你的团队全是 Spring 技术栈、不敏感启动时间和内存，Spring Boot 仍然是最稳妥的选择。但如果你在做**云原生/SaaS/FaaS 场景的 AI 服务**，Quarkus 绝对值得尝试。

---

**系列总结 & 下一篇预告**：

这是"Java + AI 工程实践"系列的第 4 篇，也是"系列四——LangChain4j 工程实践框架"的最后一篇。我们从 Memory 方案选型讲到 Tools 函数调用，从自定义 RAG 管道到 Quarkus 云原生部署，完整走通了用 Java 构建生产级 AI 服务的技术栈。

**下一篇**：番外篇我们聊聊 **LangChain4j 在生产环境的性能优化**——连接池调优、缓存策略、多模型负载均衡、Token 成本监控……让你不只是在"能用"，而是"用好"。

> 觉得这系列文章有帮助？点赞 + 收藏 + 关注三连，不错过后续更新。评论区聊聊你在生产环境用哪种框架跑 AI 服务？
