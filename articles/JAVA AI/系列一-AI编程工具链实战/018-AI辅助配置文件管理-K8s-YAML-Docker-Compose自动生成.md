# AI 辅助配置文件管理：K8s YAML / Docker Compose 自动生成，运维文件再也不用复制粘贴了

> Deployment 写错一个缩进，Pod 起不来排查了一下午。你有没有想过——把 pom.xml 和 application.yml 丢给 AI，让它直接吐出可用的 Dockerfile 和 K8s 全套编排文件？

---

## 一、开篇：YAML 缩进，Java 程序员的一生之敌

先讲个真事。

公司来了个新项目——Spring Boot 3.2 + JDK 21 微服务，需要上 K8s。leader 说："小李，你把这服务的 Dockerfile、docker-compose、K8s Deployment/Service/ConfigMap 都配一下，下班前给我。"

你打开项目，看着 pom.xml 里的十几个依赖，深吸一口气，打开浏览器，开始搜索 "Spring Boot Dockerfile 多阶段构建模板"。然后打开另一个标签页——"K8s Deployment YAML 最佳实践"。再打开一个——"docker-compose MySQL Redis Kafka 完整配置"。

两个小时后，你写出了自认为完美的 Deployment YAML，`kubectl apply -f deployment.yaml` ——

```
Error: error parsing deployment.yaml: error converting YAML to JSON:
yaml: line 23: did not find expected key
```

第 23 行。你盯着那个缩进看了 10 分钟，空格？Tab？少了一个 `-`？`apiVersion` 写成了 `api-version`？你改了一行，又报错下一行。等你终于部署成功，窗外已经是路灯通明。

**这就是 Java 程序员写运维配置文件的真实写照：格式地狱 + 模板搬运 + 反复试错。**

但你有想过吗——**如果你把项目代码直接喂给 AI，让它分析依赖、识别框架、推断端口，自动吐出全套配置文件呢？**

今天这篇文章，我就带你打通**"Java 项目源码 → AI 分析 → 全套运维配置文件一键生成"**的完整链路。读完你会发现，那些让你挠头的 YAML 文件，AI 写得比你抄得还快、还准。

> **金句：运维文件不是写给 YAML 解析器看的，是写给凌晨三点被报警叫醒的自己看的。AI 能帮你写好，但"对的配置"和"安全的配置"之间，还差一个检查清单。**

---

## 二、场景1：根据 pom.xml + application.yml 自动生成 Dockerfile

### 2.1 痛点

手写 Dockerfile 的典型翻车现场：

- JDK 版本对不上（pom.xml 指定 Java 21，Dockerfile 里写了个 `openjdk:17`）
- 构建命令写错（多模块项目，`COPY` 路径不对）
- 分层构建顺序有问题（改一行代码全部重跑 `mvn package`）
- 不会写多阶段构建（镜像体积 800MB+）

### 2.2 素材准备

先给你的 AI 工具（ChatGPT/Claude/Cursor Composer/Copilot Chat）喂上下文。假设你有个标准 Spring Boot 3.2 项目：

**pom.xml 关键片段：**

```xml
<project>
  <parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.2.5</version>
  </parent>

  <properties>
    <java.version>21</java.version>
  </properties>

  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <dependency>
      <groupId>org.postgresql</groupId>
      <artifactId>postgresql</artifactId>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
      </plugin>
    </plugins>
  </build>
</project>
```

**application.yml 关键片段：**

```yaml
server:
  port: 8080

spring:
  application:
    name: order-service
  datasource:
    url: jdbc:postgresql://${DB_HOST:localhost}:5432/orderdb
    username: ${DB_USER:orderuser}
    password: ${DB_PASSWORD:secret}

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
  endpoint:
    health:
      probes:
        enabled: true
```

### 2.3 Prompt

```text
你是一个 DevOps 专家。请根据以下 Spring Boot 3.2 项目的 pom.xml 
和 application.yml，生成一个生产级别的 Dockerfile，要求：

1. 使用多阶段构建（Multi-stage Build）
2. 第一阶段用 Maven + JDK 21 构建
3. 第二阶段用 Eclipse Temurin JDK 21 的 JRE 精简镜像运行
4. 正确的分层缓存策略（先 COPY pom.xml，再 COPY src）
5. 创建一个非 root 用户（nonroot）运行应用
6. 暴露 8080 端口
7. 配置 JVM 启动参数：-XX:+UseZGC -XX:MaxRAMPercentage=75.0
8. 添加 HEALTHCHECK 指令，使用 Spring Boot Actuator 的 /actuator/health/liveness
9. 同时生成一个使用 GraalVM Native Image 的版本（在另一个 Dockerfile.native 中）

---
以下是 pom.xml：
[pom.xml 内容]

以下是 application.yml：
[application.yml 内容]
---
```

### 2.4 AI 输出

**Dockerfile（标准 JVM 版）：**

```dockerfile
# ============================================================
# Stage 1: Build — Maven + JDK 21 构建阶段
# ============================================================
FROM maven:3.9-eclipse-temurin-21-alpine AS builder

WORKDIR /workspace

# 分层缓存：先复制依赖描述文件，利用 Docker 缓存层
COPY pom.xml .
# 如果需要多模块，也一起复制
# COPY common/pom.xml common/
# COPY service/pom.xml service/

# 预下载依赖（这一步会被缓存，改动 pom.xml 才重跑）
RUN mvn dependency:go-offline -B -q

# 复制源码并打包
COPY src ./src
RUN mvn package -DskipTests -B -q \
    && cp target/*.jar /workspace/app.jar

# ============================================================
# Stage 2: Runtime — 最小 JRE 镜像运行
# ============================================================
FROM eclipse-temurin:21-jre-alpine

# 安全：创建非 root 用户
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# 设置时区（Java 日志时区常见坑）
RUN apk add --no-cache tzdata && \
    cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && \
    echo "Asia/Shanghai" > /etc/timezone && \
    apk del tzdata

WORKDIR /app

# 从构建阶段复制 JAR
COPY --from=builder --chown=appuser:appgroup /workspace/app.jar /app/app.jar

# 切换非 root 用户
USER appuser

# JVM 参数（ZGC + 自适应内存）
ENV JAVA_OPTS="-XX:+UseZGC \
    -XX:MaxRAMPercentage=75.0 \
    -XX:+ExitOnOutOfMemoryError \
    -Djava.security.egd=file:/dev/./urandom"

EXPOSE 8080

# 健康检查：K8s liveness probe 也会用到这个接口
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8080/actuator/health/liveness || exit 1

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar /app/app.jar"]
```

**Dockerfile.native（GraalVM Native Image 版）：**

```dockerfile
# ============================================================
# Stage 1: GraalVM Native Image 构建
# ============================================================
FROM ghcr.io/graalvm/native-image-community:21 AS builder

WORKDIR /workspace

# 安装 Maven
RUN microdnf install -y maven && microdnf clean all

COPY pom.xml .
RUN mvn dependency:go-offline -B -q

COPY src ./src

# 编译为 Native Image（注意：需要 spring-boot-starter-parent 3.0+ 才内置 AOT 支持）
RUN mvn -Pnative native:compile -DskipTests -B -q

# ============================================================
# Stage 2: 极简运行时 — 基于 distroless
# ============================================================
FROM gcr.io/distroless/static-debian12:nonroot

WORKDIR /app

COPY --from=builder /workspace/target/order-service /app/app

EXPOSE 8080

ENTRYPOINT ["/app/app"]
```

### 2.5 逐段解读

| 行 | 关键点 | 为什么这么写 |
|---|---|---|
| `maven:3.9-eclipse-temurin-21-alpine` | 构建镜像选 Alpine | 镜像体积小（~150MB vs ~650MB），且 Temurin 是 OpenJDK 官方推荐发行版 |
| `COPY pom.xml .` + `mvn dependency:go-offline` | 分层缓存策略 | 只要 pom.xml 不变，这层就不会重建，节省 90% 的每次构建时间 |
| `adduser -S appuser` | 非 root 用户 | **安全红线**：root 运行容器 = 容器被拿下即可在宿主机提权 |
| `-XX:+UseZGC` | ZGC 垃圾回收器 | JDK 21 的 ZGC 已支持分代，亚毫秒级停顿，适合云原生场景 |
| `HEALTHCHECK ... /actuator/health/liveness` | 健康检查 | 没有 HEALTHCHECK 的容器，K8s 无法感知应用是否真正就绪 |
| `gcr.io/distroless/static-debian12:nonroot` | Distroless 镜像 | Native Image 编译产物是静态二进制，不需要 glibc，用 distroless 镜像仅 ~3MB |

---

## 三、场景2：自动生成 docker-compose.yml（Spring Boot + MySQL + Redis + Kafka 完整开发环境）

### 3.1 痛点

"你 docker-compose up 一下本地就启起来了"——这句话的含金量，只有经历过以下场景的人才懂：

- 新人入职配环境配了两天
- MySQL 版本和线上不一致，本地跑的 SQL 上线就炸
- Kafka Topic 没提前创建，消费者启动就报 `UnknownTopicOrPartition`
- Redis 没配密码，和公司安全规范冲突

### 3.2 Prompt

```text
请为一个 Spring Boot 微服务生成完整的 docker-compose.yml， 
服务名为 order-service（端口 8080），依赖以下中间件：

1. MySQL 8.0 — 端口 3306，数据库 orderdb，用户名 orderuser，
   密码通过环境变量 MYSQL_PASSWORD 传入，
   需要初始化 SQL 脚本（./db/init.sql）
2. Redis 7 — 端口 6379，启用 AOF 持久化，设置密码
3. Kafka（KRaft 模式，无需 Zookeeper）— 端口 9092，
   自动创建 topic: order-events (3 分区 2 副本)
4. 应用自身的 Dockerfile 在 ./Dockerfile

要求：
- 所有服务加入同一个自定义网络 order-net
- MySQL 在应用启动前完成健康检查
- Redis 和 Kafka 在应用启动前完成健康检查
- 挂载数据卷以确保重启不丢数据
- 设置合理的资源限制（memory limits）
- 环境变量集中管理（使用 .env 文件）
```

### 3.3 AI 输出

**docker-compose.yml：**

```yaml
version: '3.9'

# ============================================================
# 服务定义
# ============================================================
services:

  # --- MySQL 8.0 ---
  mysql:
    image: mysql:8.0.36
    container_name: order-mysql
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD:-rootsecret}
      MYSQL_DATABASE: orderdb
      MYSQL_USER: orderuser
      MYSQL_PASSWORD: ${MYSQL_PASSWORD:-ordersecret}
    ports:
      - "3306:3306"
    volumes:
      - mysql-data:/var/lib/mysql
      - ./db/init.sql:/docker-entrypoint-initdb.d/01-init.sql:ro
    networks:
      - order-net
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-p${MYSQL_ROOT_PASSWORD:-rootsecret}"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 40s
    deploy:
      resources:
        limits:
          memory: 512M
        reservations:
          memory: 256M

  # --- Redis 7 ---
  redis:
    image: redis:7.2-alpine
    container_name: order-redis
    restart: unless-stopped
    command: >
      redis-server
      --appendonly yes
      --requirepass ${REDIS_PASSWORD:-redispass}
      --maxmemory 256mb
      --maxmemory-policy allkeys-lru
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    networks:
      - order-net
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD:-redispass}", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5
    deploy:
      resources:
        limits:
          memory: 256M

  # --- Kafka (KRaft 模式，无需 Zookeeper) ---
  kafka:
    image: confluentinc/cp-kafka:7.6.0
    container_name: order-kafka
    restart: unless-stopped
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: 'CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT'
      KAFKA_LISTENERS: 'PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093'
      KAFKA_ADVERTISED_LISTENERS: 'PLAINTEXT://kafka:9092'
      KAFKA_CONTROLLER_LISTENER_NAMES: 'CONTROLLER'
      KAFKA_CONTROLLER_QUORUM_VOTERS: '1@kafka:9093'
      KAFKA_PROCESS_ROLES: 'broker,controller'
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: 'false'
      CLUSTER_ID: 'MkU3OEVBNTcwNTJENDM2Qk'  # 固定 cluster ID，避免每次启动重建数据
    ports:
      - "9092:9092"
    volumes:
      - kafka-data:/var/lib/kafka/data
    networks:
      - order-net
    healthcheck:
      test: ["CMD", "nc", "-z", "localhost", "9092"]
      interval: 10s
      timeout: 5s
      retries: 10
    deploy:
      resources:
        limits:
          memory: 1G

  # --- Kafka Topic 初始化 ---
  kafka-init:
    image: confluentinc/cp-kafka:7.6.0
    container_name: order-kafka-init
    depends_on:
      kafka:
        condition: service_healthy
    entrypoint: ["/bin/sh", "-c"]
    command:
      - |
        echo "Waiting for Kafka to be fully ready..."
        sleep 10
        echo "Creating topics..."
        kafka-topics --bootstrap-server kafka:9092 --create \
          --topic order-events \
          --partitions 3 \
          --replication-factor 1 \
          --if-not-exists
        echo "Topics created successfully."
    networks:
      - order-net

  # --- Spring Boot 应用 ---
  order-service:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: order-service
    restart: unless-stopped
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://mysql:3306/orderdb?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=Asia/Shanghai
      SPRING_DATASOURCE_USERNAME: orderuser
      SPRING_DATASOURCE_PASSWORD: ${MYSQL_PASSWORD:-ordersecret}
      SPRING_REDIS_HOST: redis
      SPRING_REDIS_PORT: 6379
      SPRING_REDIS_PASSWORD: ${REDIS_PASSWORD:-redispass}
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      JAVA_OPTS: "-XX:+UseZGC -XX:MaxRAMPercentage=75.0"
    ports:
      - "8080:8080"
    networks:
      - order-net
    depends_on:
      mysql:
        condition: service_healthy
      redis:
        condition: service_healthy
      kafka:
        condition: service_healthy
      kafka-init:
        condition: service_completed_successfully
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:8080/actuator/health"]
      interval: 15s
      timeout: 5s
      retries: 5
      start_period: 60s
    deploy:
      resources:
        limits:
          memory: 512M
        reservations:
          memory: 256M

# ============================================================
# 网络
# ============================================================
networks:
  order-net:
    driver: bridge
    name: order-net

# ============================================================
# 数据卷
# ============================================================
volumes:
  mysql-data:
    driver: local
  redis-data:
    driver: local
  kafka-data:
    driver: local
```

**.env 文件：**

```bash
# MySQL
MYSQL_ROOT_PASSWORD=rootsecret
MYSQL_PASSWORD=ordersecret

# Redis
REDIS_PASSWORD=redispass

# App
SPRING_PROFILES_ACTIVE=docker
```

### 3.4 关键细节解读

| 配置项 | 关键点 | 为什么 |
|---|---|---|
| `condition: service_healthy` | 启动依赖条件 | 老版的 `depends_on` 只等容器启动，不等服务就绪。MySQL 还没 accept 连接，应用就启动了 → 必炸 |
| `KAFKA_AUTO_CREATE_TOPICS_ENABLE: 'false'` | 禁止自动创建 Topic | 自动创建的 Topic 默认 1 分区 1 副本，跟线上配置不符，容易产生"我本地能跑啊"的幻觉 |
| `kafka-init` 容器 | 独立初始化容器 | 在 `depends_on` 链中加一个一次性容器创建 Topic，任务完成后退出，不占用资源 |
| `memlock` 和 `maxmemory` | Redis 资源限制 | 不设 maxmemory 的 Redis 在内存吃紧时会触发 OOM Killer |
| `CLUSTER_ID` 固定值 | Kafka KRaft 模式 | 每次重启生成新 cluster ID 会导致元数据丢失，固定后数据持久化才有意义 |
| `:ro` 挂载 + `:rw` 挂载 | 文件系统权限 | SQL 初始化脚本只读挂载，防止误操作；数据目录读写挂载 |

---

## 四、场景3：自动生成 K8s Deployment + Service + ConfigMap + Ingress

### 4.1 痛点

K8s YAML 是配置复杂度的巅峰。一个 Spring Boot 服务从代码到在集群里跑起来，最少需要以下资源：

- **Namespace**（环境隔离）
- **ConfigMap**（把 application.yml 注入进去）
- **Secret**（数据库密码等敏感信息）
- **Deployment**（副本管理、滚动更新、资源限制）
- **Service**（集群内访问）
- **Ingress**（外部流量入口）

手写这套 YAML，谷歌搜索 "k8s deployment yaml example" 的次数比写 Java 代码还多。

### 4.2 Prompt

```text
你是一位 Kubernetes 专家。我有一个 Spring Boot 3.2 微服务 
order-service（端口 8080），需要生成全套 K8s 部署文件。要求：

1. ConfigMap：
   - 使用文件挂载方式挂载 application.yml（不要逐条环境变量）
   - 非敏感配置放 ConfigMap，敏感信息放 Secret（数据库密码、Redis密码）

2. Secret：
   - 数据库密码 base64 编码
   - 不要硬编码，标明需要通过 kubectl create secret 或 SealedSecret 管理

3. Deployment：
   - 副本数 3，分布在不同可用区（podAntiAffinity）
   - 资源限制：request=256Mi/0.5CPU, limit=512Mi/1CPU
   - 使用上一步生成的 Docker 镜像 order-service:1.0.0
   - livenessProbe 和 readinessProbe（路径 /actuator/health/liveness 和 /actuator/health/readiness）
   - 优雅停机 30 秒（terminationGracePeriodSeconds）
   - 滚动更新策略：maxSurge=1, maxUnavailable=0

4. Service：
   - ClusterIP 类型，端口 8080

5. Ingress：
   - 域名 order.example.com
   - 基于 nginx-ingress controller
   - 路径前缀 /api 转发到 order-service

6. 所有文件使用 kustomize 目录结构组织：
   base/
     ├── kustomization.yaml
     ├── configmap.yaml
     ├── secret.yaml
     ├── deployment.yaml
     ├── service.yaml
     └── ingress.yaml

7. 资源标签统一为 app: order-service, team: backend
```

### 4.3 AI 输出

**base/configmap.yaml：**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: order-service-config
  labels:
    app: order-service
    team: backend
data:
  application.yml: |
    server:
      port: 8080
      shutdown: graceful
    spring:
      application:
        name: order-service
      lifecycle:
        timeout-per-shutdown-phase: 25s
      datasource:
        url: jdbc:mysql://${DB_HOST:mysql.default.svc.cluster.local}:3306/orderdb?useSSL=true&serverTimezone=Asia/Shanghai
        username: ${DB_USERNAME}
        driver-class-name: com.mysql.cj.jdbc.Driver
        hikari:
          maximum-pool-size: 20
          minimum-idle: 5
          connection-timeout: 3000
      redis:
        host: ${REDIS_HOST:redis.default.svc.cluster.local}
        port: 6379
      kafka:
        bootstrap-servers: ${KAFKA_BOOTSTRAP:kafka.default.svc.cluster.local:9092}
    management:
      server:
        port: 8081
      endpoints:
        web:
          exposure:
            include: health,info,metrics,prometheus
      endpoint:
        health:
          probes:
            enabled: true
          show-details: when-authorized
      metrics:
        tags:
          application: order-service
    logging:
      level:
        root: INFO
        com.example.orderservice: DEBUG
```

**base/secret.yaml：**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: order-service-secret
  labels:
    app: order-service
    team: backend
type: Opaque
stringData:
  DB_USERNAME: orderuser
  DB_PASSWORD: "<替换为实际密码，生产环境建议使用 SealedSecret 或 External Secrets Operator>"
  REDIS_PASSWORD: "<替换为实际密码>"
```

> ⚠️ **生产警告**：`stringData` 仅用于示例。生产环境务必使用 `kubectl create secret generic` 手动创建，或使用 SealedSecret / External Secrets Operator / Vault 注入。

**base/deployment.yaml：**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  labels:
    app: order-service
    team: backend
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1         # 滚动时最多额外 1 个 Pod
      maxUnavailable: 0   # 滚动期间不允许不可用 Pod
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
        team: backend
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8081"
        prometheus.io/path: "/actuator/prometheus"
    spec:
      # 优雅停机：给应用 30 秒处理完现有请求
      terminationGracePeriodSeconds: 30

      # 反亲和：将 Pod 尽量分散到不同节点/可用区
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - order-service
                topologyKey: kubernetes.io/hostname

      # 镜像拉取策略
      imagePullSecrets:
        - name: registry-credentials

      containers:
        - name: order-service
          image: registry.example.com/order-service:1.0.0
          imagePullPolicy: IfNotPresent

          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
            - name: management
              containerPort: 8081
              protocol: TCP

          env:
            - name: JAVA_OPTS
              value: >-
                -XX:+UseZGC
                -XX:MaxRAMPercentage=75.0
                -XX:+ExitOnOutOfMemoryError
                -Djava.security.egd=file:/dev/./urandom
                -Dspring.config.additional-location=/etc/config/

            # 从 Secret 注入环境变量
            - name: DB_USERNAME
              valueFrom:
                secretKeyRef:
                  name: order-service-secret
                  key: DB_USERNAME
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: order-service-secret
                  key: DB_PASSWORD
            - name: REDIS_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: order-service-secret
                  key: REDIS_PASSWORD

          # 资源限制（基于压测数据设定）
          resources:
            requests:
              memory: "256Mi"
              cpu: "500m"
            limits:
              memory: "512Mi"
              cpu: "1000m"

          # 就绪探针 — 流量是否接入
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8081
            initialDelaySeconds: 20
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 3

          # 存活探针 — Pod 是否需要重启
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8081
            initialDelaySeconds: 60
            periodSeconds: 20
            timeoutSeconds: 3
            failureThreshold: 5

          # 启动探针 — 给应用充足的初始化时间
          startupProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8081
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 30     # 30 * 5 = 150 秒窗口

          # 容器安全上下文
          securityContext:
            runAsNonRoot: true
            runAsUser: 1000
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL

          # ConfigMap 挂载为文件
          volumeMounts:
            - name: config
              mountPath: /etc/config
              readOnly: true
            - name: tmp
              mountPath: /tmp

      volumes:
        - name: config
          configMap:
            name: order-service-config
        - name: tmp
          emptyDir: {}
```

**base/service.yaml：**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: order-service
  labels:
    app: order-service
    team: backend
spec:
  type: ClusterIP
  selector:
    app: order-service
  ports:
    - name: http
      port: 8080
      targetPort: 8080
      protocol: TCP
    - name: management
      port: 8081
      targetPort: 8081
      protocol: TCP
```

**base/ingress.yaml：**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: order-service-ingress
  labels:
    app: order-service
    team: backend
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60s"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - order.example.com
      secretName: order-tls-secret
  rules:
    - host: order.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: order-service
                port:
                  number: 8080
```

**base/kustomization.yaml：**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - configmap.yaml
  - secret.yaml
  - deployment.yaml
  - service.yaml
  - ingress.yaml

commonLabels:
  app: order-service
  team: backend
```

### 4.4 实战验证步骤

拿到 AI 生成的 YAML 后，不要直接 `kubectl apply`，先走这套验证流水线：

```bash
# 1. 语法验证（kustomize build 会自动验证 YAML 合法性）
kustomize build ./base > /tmp/manifests.yaml

# 2. 干跑验证（不会真正部署，但会检查集群兼容性）
kubectl apply --dry-run=server -f /tmp/manifests.yaml

# 3. 查看渲染后的完整 YAML（确认变量替换正确）
kustomize build ./base | less

# 4. Polaris 合规检查（开源 K8s 最佳实践扫描）
polaris audit --audit-path ./base/ --format pretty
```

---

## 五、场景4：根据 Java 项目结构自动生成 Helm Chart

### 5.1 痛点

Helm 是 K8s 的包管理工具，但 Helm Chart 的目录结构和模板语法（Go template）是另一个维度的复杂度。大部分团队的 Helm Chart 就一个 `values.yaml` + `templates/deployment.yaml` 裸奔，没有 `_helpers.tpl`、没有 `NOTES.txt`、没有环境区分。

### 5.2 Prompt

```text
请为一个 Spring Boot 3.2 的 order-service 微服务生成完整的 Helm Chart，
项目结构如下：

项目端口：8080（业务），8081（管理端点）
依赖中间件：MySQL、Redis、Kafka
镜像仓库：registry.example.com/order-service

要求：
1. 标准 Helm Chart 目录结构（Chart.yaml, values.yaml, 
   templates/*, .helmignore, NOTES.txt）
2. templates/ 中包含：deployment.yaml, service.yaml, 
   configmap.yaml, ingress.yaml, hpa.yaml, _helpers.tpl
3. values.yaml 支持多环境（dev/staging/prod 三套默认值）
4. HPA（Horizontal Pod Autoscaler）基于 CPU 和内存自动伸缩
5. _helpers.tpl 中定义 name/app 模板函数
6. NOTES.txt 输出部署后的实用信息（访问地址、端口、健康检查命令）
7. 支持通过 values.yaml 控制 feature flags：
   - ingress.enabled: true/false
   - hpa.enabled: true/false
   - serviceMonitor.enabled: true/false（用于 Prometheus 自动发现）
```

### 5.3 AI 输出

**完整目录结构：**

```
helm/order-service/
├── Chart.yaml
├── values.yaml
├── values-dev.yaml
├── values-staging.yaml
├── values-prod.yaml
├── .helmignore
├── templates/
│   ├── _helpers.tpl
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── hpa.yaml
│   └── servicemonitor.yaml
└── NOTES.txt
```

**Chart.yaml：**

```yaml
apiVersion: v2
name: order-service
description: A Helm chart for order-service Spring Boot microservice
type: application
version: 0.1.0
appVersion: "1.0.0"
keywords:
  - spring-boot
  - microservice
  - order
maintainers:
  - name: backend-team
    email: backend@example.com
```

**values.yaml（核心配置文件）：**

```yaml
# ============================================================
# 副本与部署
# ============================================================
replicaCount: 3

image:
  repository: registry.example.com/order-service
  tag: "1.0.0"
  pullPolicy: IfNotPresent
  pullSecrets:
    - name: registry-credentials

# ============================================================
# 滚动更新策略
# ============================================================
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0

# ============================================================
# 优雅停机
# ============================================================
terminationGracePeriodSeconds: 30

# ============================================================
# 资源限制
# ============================================================
resources:
  requests:
    memory: "256Mi"
    cpu: "500m"
  limits:
    memory: "512Mi"
    cpu: "1000m"

# ============================================================
# JVM 参数
# ============================================================
javaOpts: >-
  -XX:+UseZGC
  -XX:MaxRAMPercentage=75.0
  -XX:+ExitOnOutOfMemoryError

# ============================================================
# 应用配置（注入到 ConfigMap）
# ============================================================
appConfig:
  server:
    port: 8080
    shutdown: graceful
  spring:
    lifecycle:
      timeout-per-shutdown-phase: 25s
    datasource:
      hikari:
        maximum-pool-size: 20
        minimum-idle: 5
  logging:
    level:
      root: INFO
      com.example.orderservice: DEBUG

# ============================================================
# 环境变量
# ============================================================
env:
  DB_HOST: mysql.default.svc.cluster.local
  REDIS_HOST: redis.default.svc.cluster.local
  KAFKA_BOOTSTRAP: kafka.default.svc.cluster.local:9092

# ============================================================
# Secret（生产环境不要在这里写明文）
# ============================================================
secrets:
  DB_USERNAME: orderuser
  DB_PASSWORD: ""
  REDIS_PASSWORD: ""

# ============================================================
# Service
# ============================================================
service:
  type: ClusterIP
  port: 8080
  managementPort: 8081

# ============================================================
# 健康检查探针
# ============================================================
probes:
  liveness:
    path: /actuator/health/liveness
    port: 8081
    initialDelaySeconds: 60
    periodSeconds: 20
    failureThreshold: 5
  readiness:
    path: /actuator/health/readiness
    port: 8081
    initialDelaySeconds: 20
    periodSeconds: 10
    failureThreshold: 3
  startup:
    path: /actuator/health/liveness
    port: 8081
    initialDelaySeconds: 10
    periodSeconds: 5
    failureThreshold: 30

# ============================================================
# Feature Flags
# ============================================================
ingress:
  enabled: true
  className: nginx
  host: order.example.com
  tlsSecretName: order-tls-secret
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"

hpa:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80

serviceMonitor:
  enabled: true
  interval: 30s
  path: /actuator/prometheus
```

**templates/_helpers.tpl：**

```yaml
{{/* 生成资源名称 */}}
{{- define "order-service.name" -}}
{{- .Chart.Name | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/* 生成完整名称（包含 Release 名称） */}}
{{- define "order-service.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name .Chart.Name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}

{{/* 通用标签 */}}
{{- define "order-service.labels" -}}
helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version }}
app.kubernetes.io/name: {{ include "order-service.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/* 选择器标签 */}}
{{- define "order-service.selectorLabels" -}}
app.kubernetes.io/name: {{ include "order-service.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

**templates/hpa.yaml（条件渲染）：**

```yaml
{{- if .Values.hpa.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "order-service.fullname" . }}
  labels:
    {{- include "order-service.labels" . | nindent 4 }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "order-service.fullname" . }}
  minReplicas: {{ .Values.hpa.minReplicas }}
  maxReplicas: {{ .Values.hpa.maxReplicas }}
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: {{ .Values.hpa.targetCPUUtilizationPercentage }}
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: {{ .Values.hpa.targetMemoryUtilizationPercentage }}
{{- end }}
```

**values-prod.yaml（生产环境覆盖）：**

```yaml
# 生产环境覆盖值
replicaCount: 5

image:
  tag: "1.0.0"  # 生产环境使用明确版本号

resources:
  requests:
    memory: "512Mi"
    cpu: "1000m"
  limits:
    memory: "1Gi"
    cpu: "2000m"

hpa:
  minReplicas: 5
  maxReplicas: 20
  targetCPUUtilizationPercentage: 60

ingress:
  host: order.prod.example.com

appConfig:
  logging:
    level:
      root: WARN
      com.example.orderservice: INFO
```

**NOTES.txt：**

```txt
🚀 Order Service 部署成功！

Release: {{ .Release.Name }}
Namespace: {{ .Release.Namespace }}

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📋 快速命令
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

查看 Pod 状态:
  kubectl get pods -n {{ .Release.Namespace }} -l app.kubernetes.io/instance={{ .Release.Name }}

查看日志:
  kubectl logs -f deployment/{{ include "order-service.fullname" . }} -n {{ .Release.Namespace }}

健康检查:
  kubectl exec -it deployment/{{ include "order-service.fullname" . }} -n {{ .Release.Namespace }} -- wget -q -O- http://localhost:{{ .Values.probes.liveness.port }}{{ .Values.probes.liveness.path }}

{{- if .Values.ingress.enabled }}
访问地址（Ingress）:
  https://{{ .Values.ingress.host }}/api
{{- end }}

端口转发（本地调试）:
  kubectl port-forward svc/{{ include "order-service.fullname" . }} {{ .Values.service.port }}:{{ .Values.service.port }} -n {{ .Release.Namespace }}

{{- if .Values.hpa.enabled }}
查看自动伸缩状态:
  kubectl get hpa {{ include "order-service.fullname" . }} -n {{ .Release.Namespace }}
{{- end }}

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### 5.4 环境差异化部署

```bash
# 开发环境
helm upgrade --install order-service ./helm/order-service \
  -n dev -f ./helm/order-service/values-dev.yaml

# 预发环境
helm upgrade --install order-service ./helm/order-service \
  -n staging -f ./helm/order-service/values-staging.yaml

# 生产环境
helm upgrade --install order-service ./helm/order-service \
  -n production -f ./helm/order-service/values-prod.yaml \
  --set secrets.DB_PASSWORD="$DB_PASSWORD"    # 密码从 CI/CD 变量注入
```

---

## 六、配置文件的"AI 化"检查清单：AI 写完了，你还得做的 10 件事

AI 能帮你写配置，但不能替你做安全审计。拿到 AI 生成的配置后，**一项一项过这个清单**。

### 6.1 端口暴露检查

| 检查项 | 坏配置 | 好配置 |
|---|---|---|
| 管理端点是否暴露到公网 | `ports: [8080, 8081]` 两个端口都在 Ingress 上 | Ingress 只转发 8080，8081 只在集群内通过 Service 访问 |
| 数据库端口是否对外 | `docker-compose` 里 MySQL `ports: "3306:3306"` 无绑定 IP | `ports: "127.0.0.1:3306:3306"` |

> **Prompt 技巧**：生成后追问——"请分析以上配置中哪些端口有安全风险，并给出修复建议。"

### 6.2 资源限制检查

```yaml
# ❌ 坏人配置 — 无资源限制
resources: {}

# ✅ 好人配置 — 设置了 request/limit
resources:
  requests:
    memory: "256Mi"
    cpu: "500m"
  limits:
    memory: "512Mi"
    cpu: "1000m"
```

**规则**：`request = limit` 才是真正的 Guaranteed QoS，不会被 OOM Killer 优先杀掉。

### 6.3 健康检查完整性

| 探针类型 | 作用 | 常见错误 |
|---|---|---|
| `livenessProbe` | 决定是否重启 Pod | 路径写错、端口写错；或用同一个接口做 liveness 和 readiness |
| `readinessProbe` | 决定是否接入流量 | 不做 readiness，Pod 还没连上数据库就开始接收请求 |
| `startupProbe` | 给慢启动留时间 | 没设 startupProbe，liveness 太激进导致 Pod 反复重启 |

```yaml
# ❌ liveness 和 readiness 用同一个接口
livenessProbe:  { httpGet: { path: /actuator/health } }
readinessProbe: { httpGet: { path: /actuator/health } }

# ✅ 正确分离
livenessProbe:  { httpGet: { path: /actuator/health/liveness } }
readinessProbe: { httpGet: { path: /actuator/health/readiness } }
```

### 6.4 敏感信息暴露

```text
Prompt 追问：
"请检查以上 K8s 配置，标记所有可能泄露敏感信息的地方，
 并给出使用 SealedSecret/External Secrets Operator 的替代方案。"
```

AI 通常会指出：
- Secret 里的 `stringData` 不应提交 Git → 改用 `kubectl create secret` + CI 注入
- 日志级别 `DEBUG` 可能打印请求体中的密码 → 生产环境至少 `INFO`
- `JAVA_OPTS` 中的 `-XX:+HeapDumpOnOutOfMemoryError` + 挂载卷 → HPROF 文件可能含敏感数据

### 6.5 容器安全上下文

```yaml
# ✅ 最小权限安全上下文
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop:
      - ALL
```

### 6.6 检查清单速查表

| # | 检查项 | 你的配置里有没有？ |
|---|--------|-------------------|
| 1 | 容器是否以非 root 运行？ | ☐ |
| 2 | 资源 request/limit 都设了吗？ | ☐ |
| 3 | 三个探针（liveness/readiness/startup）分离了吗？ | ☐ |
| 4 | 数据库密码在 Secret 里，不在 ConfigMap？ | ☐ |
| 5 | `readOnlyRootFilesystem: true`？ | ☐ |
| 6 | 所有 CAP 都 drop 了？ | ☐ |
| 7 | Ingress 只暴露业务端口，不暴露管理端口？ | ☐ |
| 8 | terminationGracePeriodSeconds >= Spring 的 shutdown timeout？ | ☐ |
| 9 | HPA 的 minReplicas >= 2（高可用底线）？ | ☐ |
| 10 | 镜像 tag 不是 latest？ | ☐ |

---

## 七、高级技巧：Docker Compose ↔ K8s YAML 互转

### 7.1 Docker Compose → K8s YAML

**场景**：你手里有个完整的 docker-compose.yml 本地开发环境，老板说"下周上 K8s"。你不可能手写全套 K8s YAML。

**Prompt：**

```text
请将以下 docker-compose.yml 转换为 K8s 资源文件，
输出格式为 kustomize 目录结构（base/ 文件夹），包含：

- Deployment（每个 service）
- Service（每个需要暴露端口或集群内访问的服务）
- ConfigMap（环境变量集中的）
- PersistentVolumeClaim（挂载的数据卷）
- Ingress（对外暴露的 HTTP 服务）

注意：
1. depends_on 的启动依赖关系，用 initContainers 实现
2. docker-compose 的 volumes 转为 PVC
3. 健康检查转为 liveness/readiness probe
4. 环境变量转为 ConfigMap + Secret

---
[docker-compose.yml 内容]
---
```

AI 会分析 docker-compose 中的每个 service，推断端口映射、依赖关系、数据卷需求，然后生成对应的 K8s 原生资源。**准确率约 85%**——Service 和 Deployment 基本不用改，但 InitContainer 的启动顺序可能需要微调。

### 7.2 K8s YAML → Docker Compose

**逆向场景**：线上 K8s 环境跑得好好的，要给新同事搭本地开发环境。

**Prompt：**

```text
请将以下 K8s Deployment + Service + ConfigMap 合并为一个 
docker-compose.yml，要求：

1. 保留所有中间件依赖（MySQL/Redis/Kafka）
2. K8s Secret 转为 .env 文件
3. K8s ConfigMap 转为直接注入的环境变量
4. K8s 的 liveness/readiness probe 转为 docker-compose healthcheck
5. k8s probe 的探针路径和端口要保持一致
6. 去掉 K8s 特有概念（HPA、PodAntiAffinity、ServiceAccount）

---
[K8s YAML 文件内容]
---
```

### 7.3 互转的局限性（知道不能做什么，比知道能做什么更重要）

| 能自动转的 | 转不了/需要人工介入的 |
|---|---|
| Deployment → docker-compose service | RBAC (ServiceAccount, RoleBinding) |
| Service → 端口映射 | NetworkPolicy（K8s 的网络策略在 Docker 网络里无等价物） |
| PVC → volumes | 资源配额 (ResourceQuota/LimitRange) |
| ConfigMap → environment / volumes | Sidecar 容器模式 |
| Ingress → 端口暴露（无反向代理） | PodDisruptionBudget |

> **金句**：AI 做的是"翻译"，不是"设计"。架构决策（比如 K8s 的 Sidecar 模式在 Docker Compose 里应该拆成独立 service 还是合并到主容器）必须由你来拍板。

---

## 八、本篇总结

**你学到了什么：**

1. **一键生成 Dockerfile**：喂 pom.xml + application.yml，出多阶段构建 + GraalVM Native Image 双版本
2. **一键生成 docker-compose.yml**：完整 Spring Boot + MySQL + Redis + Kafka 开发环境，含健康检查、资源限制、KRaft 模式
3. **一键生成 K8s 全家桶**：Deployment + Service + ConfigMap + Secret + Ingress，kustomize 目录结构，开箱即用
4. **一键生成 Helm Chart**：_helpers.tpl、values 多环境、HPA、NOTES.txt 一应俱全
5. **AI 配置检查清单**：10 项安全检查，过完再上线
6. **Docker Compose ↔ K8s 互转**：正向翻译 85% 可用，架构决策你来拍板

**三个记住：**

- **记住第一句**：把项目源码（pom.xml + application.yml）喂给 AI，它能推断出 90% 的配置需求——这是 AI 辅助运维的起点
- **记住第二句**：AI 写得快 ≠ AI 写得对。检查清单里的 10 项，一项都不能省
- **记住第三句**：K8s 和 Docker Compose 之间的翻译，AI 能做 "形"，做不了 "神"。架构决策永远是人的事

---

## 九、下一篇预告

下一篇我们将进入 **系列一的第 19 篇：《AI 辅助 Git Commit Message 自动化——从此告别 update/fix/tmp 提交信息》**。

你将学到：

- 如何用 AI 根据 `git diff` 自动生成 Conventional Commits 格式的提交信息
- Git Hooks + AI 在 `git commit` 时自动分析变更并撰写规范的 Commit Message
- 多文件改动的 commit 拆分策略——AI 帮你决定哪些文件应该合在一个 commit 里
- 团队 Commit Message 风格统一：用 `.cursorrules` / `.copilot-instructions.md` 固化规范

**让你的 Git 日志从 `fix bug`、`update`、`tmp backup` 变成：**

```
feat(order): 新增订单超时自动取消功能
fix(payment): 修复支付宝回调签名验证失败的问题 #2341
chore(deps): 升级 spring-boot-starter-parent 至 3.2.5
refactor(user): 提取用户认证逻辑至独立模块 auth-common
```

—— **下篇见！**

---

> **本文是《AI 编程工具链实战》系列第 18 篇。** 全系列规划 30 篇，覆盖从 Copilot 入门到 AI Agent 自动化开发的全链路。关注我，不错过任何一篇。
