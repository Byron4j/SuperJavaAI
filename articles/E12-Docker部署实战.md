# Docker深度解析：容器化部署实战与生产环境最佳实践

**文章标签：** #docker #容器化 #部署 #devops #kubernetes #微服务 #面试

## 目录

- [引言：容器化的本质](#引言容器化的本质)
- [理论基础：Linux容器技术原理](#理论基础linux容器技术原理)
- [演进史：从LXC到Kubernetes生态](#演进史从lxc到kubernetes生态)
- [核心原理深度解析](#核心原理深度解析)
  - [Namespace隔离机制](#namespace隔离机制)
  - [Cgroups资源限制](#cgroups资源限制)
  - [UnionFS与镜像分层](#unionfs与镜像分层)
  - [Docker网络模型](#docker网络模型)
  - [Docker存储驱动](#docker存储驱动)
  - [容器安全机制](#容器安全机制)
- [实战案例：工业级部署](#实战案例工业级部署)
  - [案例1：Spring Boot应用容器化](#案例1spring-boot应用容器化)
  - [案例2：多阶段构建优化](#案例2多阶段构建优化)
  - [案例3：Docker Compose编排](#案例3docker-compose编排)
  - [案例4：生产环境部署](#案例4生产环境部署)
  - [案例5：CI/CD集成](#案例5cicd集成)
  - [案例6：Kubernetes部署](#案例6kubernetes部署)
- [对比分析：Docker vs VM vs Podman](#对比分析docker-vs-vm-vs-podman)
- [性能分析：容器开销与优化](#性能分析容器开销与优化)
- [常见陷阱与最佳实践](#常见陷阱与最佳实践)
- [面试题与参考答案](#面试题与参考答案)

---

## 引言：容器化的本质

Docker不是简单的"轻量级虚拟机"，而是**基于Linux内核特性的进程级隔离技术**。

核心认知：

```
传统部署：
  开发环境 -> 测试环境 -> 生产环境
  问题：
    - "在我机器上可以运行"
    - 依赖冲突（不同服务需要不同版本的库）
    - 环境配置差异导致故障

虚拟机部署：
  物理机 -> Hypervisor -> Guest OS -> App + Dependencies
  问题：
    - 启动慢（分钟级）
    - 资源占用大（GB级）
    - 镜像体积大

容器化部署：
  宿主机 -> Docker Engine -> Container（进程级隔离）-> App + Dependencies
  优势：
    - 启动快（秒级）
    - 资源占用小（MB级）
    - 一致的环境（Build Once, Run Anywhere）
    - 轻量级隔离（共享宿主机内核）

Docker的本质：
  - 不是虚拟机，而是隔离的进程
  - 利用Linux Namespace实现隔离
  - 利用Cgroups实现资源限制
  - 利用UnionFS实现分层存储
```

**关键洞察**：容器化的核心价值是**环境一致性**和**部署标准化**，而非资源隔离（隔离性弱于VM）。

---

## 理论基础：Linux容器技术原理

### 1. Namespace隔离

```
Linux Namespace（命名空间）是内核提供的一种资源隔离机制：

┌─────────────────────────────────────────────────────────────┐
│                    Linux Namespace                           │
├──────────────┬──────────────────────────────────────────────┤
│   类型       │              隔离资源                         │
├──────────────┼──────────────────────────────────────────────┤
│   PID        │  进程ID（容器内PID 1是初始化进程）              │
├──────────────┼──────────────────────────────────────────────┤
│   NET        │  网络设备、端口、路由表                         │
├──────────────┼──────────────────────────────────────────────┤
│   IPC        │  信号量、消息队列、共享内存                     │
├──────────────┼──────────────────────────────────────────────┤
│   MNT        │  挂载点（文件系统视图）                         │
├──────────────┼──────────────────────────────────────────────┤
│   UTS        │  主机名和域名                                 │
├──────────────┼──────────────────────────────────────────────┤
│   USER       │  用户和组ID（UID/GID映射）                     │
├──────────────┼──────────────────────────────────────────────┤
│   CGROUP     │  Cgroups根目录（Docker 1.6+）                 │
└──────────────┴──────────────────────────────────────────────┘

容器 = 进程 + Namespace隔离

示例：
  宿主机看到容器进程：PID 1234
  容器内看到同一进程：PID 1
  
  这就是PID Namespace的作用
```

### 2. Cgroups资源限制

```
Linux Cgroups（Control Groups）是内核提供的资源限制机制：

┌─────────────────────────────────────────────────────────────┐
│                    Cgroups子系统                             │
├──────────────┬──────────────────────────────────────────────┤
│   子系统     │              控制资源                         │
├──────────────┼──────────────────────────────────────────────┤
│   cpu        │  CPU使用率（shares、quota、period）           │
├──────────────┼──────────────────────────────────────────────┤
│   cpuacct    │  CPU使用统计                                  │
├──────────────┼──────────────────────────────────────────────┤
│   cpuset     │  CPU和内存节点绑定                            │
├──────────────┼──────────────────────────────────────────────┤
│   memory     │  内存限制（limit、swap）                       │
├──────────────┼──────────────────────────────────────────────┤
│   blkio      │  块设备IO限制                                 │
├──────────────┼──────────────────────────────────────────────┤
│   devices    │  设备访问控制（白名单/黑名单）                  │
├──────────────┼──────────────────────────────────────────────┤
│   net_cls    │  网络流量标记（用于tc流量控制）                 │
├──────────────┼──────────────────────────────────────────────┤
│   net_prio   │  网络优先级                                   │
├──────────────┼──────────────────────────────────────────────┤
│   pids       │  进程数限制                                   │
├──────────────┼──────────────────────────────────────────────┤
│   freezer    │  进程挂起/恢复                                │
└──────────────┴──────────────────────────────────────────────┘

Docker资源限制对应：
  docker run --memory=512m --cpus=1.0 --pids-limit=100
  
  --memory：对应memory子系统
  --cpus：对应cpu子系统
  --pids-limit：对应pids子系统
```

### 3. UnionFS分层存储

```
UnionFS（联合文件系统）：

镜像分层结构：

┌─────────────────────────────────────────┐
│           容器可写层（Container Layer）    │  读写
│         （新增/修改/删除的文件）            │
├─────────────────────────────────────────┤
│          应用层（Application Layer）       │  只读
│         （Spring Boot JAR）               │
├─────────────────────────────────────────┤
│          依赖层（Dependencies Layer）      │  只读
│         （JDK、Maven依赖）                 │
├─────────────────────────────────────────┤
│          基础镜像层（Base Image Layer）    │  只读
│         （Ubuntu/CentOS/Alpine）          │
└─────────────────────────────────────────┘

Copy-on-Write（写时复制）：
- 读取：从上到下查找文件
- 写入：复制文件到可写层，然后修改
- 删除：在可写层创建.whiteout文件标记删除

优势：
- 镜像共享基础层，节省存储
- 镜像构建缓存复用
- 容器启动快速（只需创建可写层）
```

---

## 演进史：从LXC到Kubernetes生态

### 第一阶段：LXC（2008）

```
LXC（Linux Containers）：
- 基于Linux Namespace和Cgroups
- 提供操作系统级虚拟化
- 需要手动配置Namespace和Cgroups

局限性：
- 使用复杂
- 没有镜像概念
- 可移植性差
```

### 第二阶段：Docker诞生（2013）

```
Docker（dotCloud公司，2013年开源）：
- 基于LXC（后期改为libcontainer，再到runC）
- 引入镜像概念（UnionFS + 分层）
- Dockerfile定义构建过程
- Docker Hub镜像仓库

革命性创新：
1. 镜像标准化：Build Once, Run Anywhere
2. Dockerfile：基础设施即代码
3. 分层存储：高效复用，快速分发
4. 生态系统：Docker Hub、Docker Compose、Docker Swarm
```

### 第三阶段：OCI标准与容器运行时（2015）

```
OCI（Open Container Initiative，2015）：
- Docker、CoreOS、Google等公司发起
- 制定容器标准：
  * Runtime Spec：容器运行时标准
  * Image Spec：容器镜像标准

容器运行时演进：
- runC：Docker捐赠给OCI的参考实现
- containerd：Docker核心容器运行时（2016年捐赠给CNCF）
- CRI-O：Kubernetes专用的轻量级运行时

Docker架构演进：
Docker CLI -> Docker Daemon -> containerd -> runC -> Container
```

### 第四阶段：Kubernetes编排时代（2015-至今）

```
Kubernetes（Google，2014年开源）：
- 基于Borg（Google内部系统）经验
- 容器编排的事实标准
- 2015年成为CNCF第一个项目

容器编排工具对比：
- Docker Swarm：Docker原生，简单，但功能弱
- Kubernetes：功能强大，生态丰富，复杂
- Mesos：资源调度框架，支持多种工作负载

Kubernetes胜出原因：
1. 声明式API
2. 丰富的控制器（Deployment、StatefulSet、DaemonSet等）
3. 自动扩缩容（HPA、VPA）
4. 服务发现和负载均衡
5. 存储编排
6. 生态丰富（Helm、Istio、Prometheus等）
```

### 第五阶段：云原生与Serverless（2018-至今）

```
云原生（Cloud Native）：
- 容器化（Containers）
- 微服务（Microservices）
- 服务网格（Service Mesh）
- DevOps
- 持续交付（CI/CD）

Serverless容器：
- AWS Fargate
- Azure Container Instances
- Google Cloud Run
- 阿里云ECI

趋势：
- 容器即服务（CaaS）
- 无服务器容器（Serverless Containers）
- 边缘计算容器化
```

---

## 核心原理深度解析

### Namespace隔离机制

```bash
# 查看容器的Namespace
ls -l /proc/<pid>/ns

# 输出：
lrwxrwxrwx 1 root root 0 Jan  1 00:00 cgroup -> cgroup:[4026531835]
lrwxrwxrwx 1 root root 0 Jan  1 00:00 ipc -> ipc:[4026531839]
lrwxrwxrwx 1 root root 0 Jan  1 00:00 mnt -> mnt:[4026531840]
lrwxrwxrwx 1 root root 0 Jan  1 00:00 net -> net:[4026531841]
lrwxrwxrwx 1 root root 0 Jan  1 00:00 pid -> pid:[4026531836]
lrwxrwxrwx 1 root root 0 Jan  1 00:00 user -> user:[4026531837]
lrwxrwxrwx 1 root root 0 Jan  1 00:00 uts -> uts:[4026531838]
```

```go
// 创建Namespace的Go代码示例
package main

import (
    "os"
    "os/exec"
    "syscall"
)

func main() {
    cmd := exec.Command("/bin/sh")
    cmd.SysProcAttr = &syscall.SysProcAttr{
        Cloneflags: syscall.CLONE_NEWUTS | 
                    syscall.CLONE_NEWPID | 
                    syscall.CLONE_NEWNS |
                    syscall.CLONE_NEWNET |
                    syscall.CLONE_NEWIPC |
                    syscall.CLONE_NEWUSER,
    }
    cmd.Stdin = os.Stdin
    cmd.Stdout = os.Stdout
    cmd.Stderr = os.Stderr
    
    if err := cmd.Run(); err != nil {
        panic(err)
    }
}
```

### Cgroups资源限制

```bash
# Docker容器Cgroups配置路径
/sys/fs/cgroup/memory/docker/<container-id>/
/sys/fs/cgroup/cpu/docker/<container-id>/
/sys/fs/cgroup/pids/docker/<container-id>/

# 查看内存限制
cat /sys/fs/cgroup/memory/docker/<container-id>/memory.limit_in_bytes

# 查看CPU限制
cat /sys/fs/cgroup/cpu/docker/<container-id>/cpu.cfs_quota_us
cat /sys/fs/cgroup/cpu/docker/<container-id>/cpu.cfs_period_us
```

```bash
# Docker资源限制命令
docker run -d \
  --memory=512m \
  --memory-swap=512m \
  --memory-reservation=256m \
  --cpus=1.0 \
  --cpu-shares=1024 \
  --pids-limit=100 \
  --blkio-weight=300 \
  myapp:1.0.0
```

### UnionFS与镜像分层

```
Docker支持的存储驱动：

┌─────────────────────┬─────────────────────────────────────┐
│     存储驱动         │              特点                    │
├─────────────────────┼─────────────────────────────────────┤
│     overlay2        │  现代Linux默认，性能最好，层数有限制   │
├─────────────────────┼─────────────────────────────────────┤
│     aufs            │  早期Ubuntu默认，层数无限制，已弃用    │
├─────────────────────┼─────────────────────────────────────┤
│     devicemapper    │  CentOS/RHEL早期默认，性能一般        │
├─────────────────────┼─────────────────────────────────────┤
│     btrfs           │  需要btrfs文件系统，支持快照          │
├─────────────────────┼─────────────────────────────────────┤
│     zfs             │  需要zfs文件系统，性能优秀            │
└─────────────────────┴─────────────────────────────────────┘

overlay2工作原理：

LowerDir（底层）：
  /var/lib/docker/overlay2/<id>/diff
  包含基础镜像层

UpperDir（可写层）：
  /var/lib/docker/overlay2/<id>-init/diff
  容器可写层

MergedDir（合并视图）：
  /var/lib/docker/overlay2/<id>/merged
  容器看到的文件系统

WorkDir：
  /var/lib/docker/overlay2/<id>/work
  工作目录（Copy-on-Write使用）
```

### Docker网络模型

```
Docker网络模式：

┌─────────────────────────────────────────────────────────────┐
│                    Bridge（桥接，默认）                       │
│  - 创建虚拟网桥docker0                                       │
│  - 容器通过veth pair连接网桥                                 │
│  - 容器间可通信，外部访问需端口映射                             │
│  - 适用：单机多容器通信                                       │
├─────────────────────────────────────────────────────────────┤
│                    Host（主机）                               │
│  - 容器共享宿主机网络栈                                       │
│  - 无网络隔离，性能最好                                       │
│  - 端口冲突风险                                              │
│  - 适用：网络性能要求高的场景                                  │
├─────────────────────────────────────────────────────────────┤
│                    None（无网络）                             │
│  - 容器只有lo接口                                            │
│  - 完全隔离                                                  │
│  - 适用：不需要网络的场景                                      │
├─────────────────────────────────────────────────────────────┤
│                    Container（共享容器网络）                   │
│  - 新容器共享已有容器的网络栈                                  │
│  - 两个容器localhost互通                                      │
│  - 适用：Sidecar模式                                         │
├─────────────────────────────────────────────────────────────┤
│                    Overlay（覆盖网络）                        │
│  - 跨主机容器通信                                            │
│  - 基于VXLAN或加密隧道                                       │
│  - 适用：Docker Swarm、Kubernetes                            │
└─────────────────────────────────────────────────────────────┘
```

```bash
# 查看容器网络配置
docker network ls

# 创建自定义桥接网络
docker network create --driver bridge --subnet 172.20.0.0/16 my-network

# 容器加入自定义网络
docker run -d --name app1 --network my-network myapp:1.0.0
docker run -d --name app2 --network my-network myapp:1.0.0

# 容器间通过容器名通信
# app1 -> ping app2
```

### Docker存储驱动

```
容器数据持久化方式：

1. Volumes（数据卷）：
   - Docker管理的数据卷
   - 存储在 /var/lib/docker/volumes/
   - 推荐用于生产环境
   - 支持驱动扩展（NFS、云存储等）

2. Bind Mounts（绑定挂载）：
   - 将宿主机目录挂载到容器
   - 路径可任意指定
   - 适合开发环境
   - 性能最好

3. tmpfs Mounts（内存挂载）：
   - 数据存储在内存中
   - 容器停止后数据丢失
   - 适合敏感数据或临时文件

4. Named Pipes（Windows）：
   - Windows容器特有
   - 进程间通信
```

```bash
# Volume方式
docker run -d \
  -v mysql_data:/var/lib/mysql \
  mysql:8.0

# Bind Mount方式
docker run -d \
  -v /host/path:/container/path \
  -v $(pwd)/config:/app/config \
  myapp:1.0.0

# tmpfs方式
docker run -d \
  --tmpfs /app/cache:noexec,nosuid,size=100m \
  myapp:1.0.0
```

### 容器安全机制

```
容器安全层次：

1. Linux Capabilities：
   - 细粒度的权限控制
   - 默认丢弃大部分特权
   - docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE

2. Seccomp（Secure Computing Mode）：
   - 系统调用过滤
   - 默认过滤44个系统调用
   - docker run --security-opt seccomp=default.json

3. AppArmor/SELinux：
   - 强制访问控制（MAC）
   - 限制容器的文件访问
   - Docker默认启用AppArmor（Ubuntu）

4. User Namespace：
   - UID/GID映射
   - 容器内root映射到宿主机普通用户
   - docker run --userns=host

5. 只读文件系统：
   - docker run --read-only
   - 配合tmpfs写入临时目录

6. 非Root运行：
   - Dockerfile中USER指令
   - 最小权限原则
```

```dockerfile
# 安全加固的Dockerfile
FROM eclipse-temurin:17-jre-alpine

# 创建非root用户
RUN addgroup -g 1001 -S appgroup && \
    adduser -u 1001 -S appuser -G appgroup

# 设置工作目录
WORKDIR /app

# 复制应用文件
COPY --chown=appuser:appgroup target/*.jar app.jar

# 切换到非root用户
USER appuser

# 只读文件系统运行时配合tmpfs
# docker run --read-only --tmpfs /tmp --tmpfs /app/logs

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
```

---

## 实战案例：工业级部署

### 案例1：Spring Boot应用容器化

```dockerfile
# Dockerfile
FROM eclipse-temurin:17-jre-alpine AS base

# 安装必要的工具
RUN apk add --no-cache curl ca-certificates

# 创建非root用户
RUN addgroup -g 1001 -S appgroup && \
    adduser -u 1001 -S appuser -G appgroup

# 设置时区
ENV TZ=Asia/Shanghai
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

WORKDIR /app

# 从构建阶段复制JAR
COPY --chown=appuser:appgroup target/*.jar app.jar

# 健康检查
HEALTHCHECK --interval=30s --timeout=3s --start-period=60s --retries=3 \
  CMD curl -f http://localhost:8080/actuator/health || exit 1

# 切换到非root用户
USER appuser

EXPOSE 8080

# JVM参数优化
ENV JAVA_OPTS="-XX:+UseContainerSupport \
  -XX:InitialRAMPercentage=75.0 \
  -XX:MaxRAMPercentage=75.0 \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=200 \
  -XX:+UseStringDeduplication \
  -Djava.security.egd=file:/dev/./urandom"

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

```bash
# 构建镜像
docker build -t myapp:1.0.0 .

# 运行容器
docker run -d \
  --name myapp \
  --hostname myapp \
  -p 8080:8080 \
  -e SPRING_PROFILES_ACTIVE=prod \
  -e JAVA_OPTS="-Xms1g -Xmx1g" \
  -v /data/logs:/app/logs \
  -v /data/config:/app/config:ro \
  --memory=1.5g \
  --memory-swap=1.5g \
  --cpus=1.0 \
  --pids-limit=100 \
  --restart=unless-stopped \
  --health-cmd="curl -f http://localhost:8080/actuator/health || exit 1" \
  --health-interval=30s \
  --health-timeout=3s \
  --health-start-period=60s \
  --health-retries=3 \
  myapp:1.0.0

# 查看日志
docker logs -f --tail=100 myapp

# 进入容器
docker exec -it myapp /bin/sh
```

### 案例2：多阶段构建优化

```dockerfile
# Dockerfile with multi-stage build

# 阶段1：依赖缓存
FROM maven:3.9-eclipse-temurin-17-alpine AS deps
WORKDIR /build
COPY pom.xml .
RUN mvn dependency:go-offline

# 阶段2：编译构建
FROM deps AS build
COPY src ./src
RUN mvn clean package -DskipTests

# 阶段3：提取层（Spring Boot 2.3+分层Jar）
FROM build AS extractor
WORKDIR /extracted
COPY --from=build /build/target/*.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract

# 阶段4：运行
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app

# 创建非root用户
RUN addgroup -g 1001 -S appgroup && \
    adduser -u 1001 -S appuser -G appgroup

# 分层复制，提高缓存命中率
COPY --from=extractor --chown=appuser:appgroup /extracted/dependencies/ ./
COPY --from=extractor --chown=appuser:appgroup /extracted/spring-boot-loader/ ./
COPY --from=extractor --chown=appuser:appgroup /extracted/snapshot-dependencies/ ./
COPY --from=extractor --chown=appuser:appgroup /extracted/application/ ./

USER appuser

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --start-period=60s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

### 案例3：Docker Compose编排

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    image: myapp:1.0.0
    container_name: myapp
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=prod
      - SPRING_DATASOURCE_URL=jdbc:mysql://mysql:3306/mydb
      - SPRING_DATASOURCE_USERNAME=appuser
      - SPRING_DATASOURCE_PASSWORD=${DB_PASSWORD}
      - SPRING_REDIS_HOST=redis
      - SPRING_REDIS_PORT=6379
      - SPRING_REDIS_PASSWORD=${REDIS_PASSWORD}
      - JAVA_OPTS=-Xms1g -Xmx1g -XX:+UseG1GC
    volumes:
      - app_logs:/app/logs
      - ./config/application-prod.yml:/app/config/application-prod.yml:ro
    depends_on:
      mysql:
        condition: service_healthy
      redis:
        condition: service_started
    networks:
      - backend
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 1.5G
        reservations:
          cpus: '0.5'
          memory: 512M
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]
      interval: 30s
      timeout: 3s
      start_period: 60s
      retries: 3

  mysql:
    image: mysql:8.0
    container_name: mysql
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
      MYSQL_DATABASE: mydb
      MYSQL_USER: appuser
      MYSQL_PASSWORD: ${DB_PASSWORD}
    volumes:
      - mysql_data:/var/lib/mysql
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro
      - ./mysql/my.cnf:/etc/mysql/conf.d/my.cnf:ro
    ports:
      - "3306:3306"
    networks:
      - backend
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    container_name: redis
    command: redis-server --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis_data:/data
      - ./redis/redis.conf:/usr/local/etc/redis/redis.conf:ro
    ports:
      - "6379:6379"
    networks:
      - backend
    restart: unless-stopped

  nginx:
    image: nginx:alpine
    container_name: nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
      - ./nginx/html:/usr/share/nginx/html:ro
    depends_on:
      - app
    networks:
      - backend
    restart: unless-stopped

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus
    networks:
      - backend
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/dashboards:/etc/grafana/provisioning/dashboards:ro
      - ./grafana/datasources:/etc/grafana/provisioning/datasources:ro
    depends_on:
      - prometheus
    networks:
      - backend
    restart: unless-stopped

volumes:
  app_logs:
  mysql_data:
  redis_data:
  prometheus_data:
  grafana_data:

networks:
  backend:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
```

```bash
# 启动所有服务
docker-compose up -d

# 查看服务状态
docker-compose ps

# 查看日志
docker-compose logs -f app

# 重启服务
docker-compose restart app

# 停止并删除
docker-compose down -v

# 扩展服务实例
docker-compose up -d --scale app=3
```

### 案例4：生产环境部署

```bash
#!/bin/bash
# deploy.sh - 生产环境部署脚本

set -e

IMAGE_NAME="myapp"
IMAGE_TAG="${1:-latest}"
CONTAINER_NAME="myapp-prod"

# 1. 拉取最新镜像
docker pull registry.example.com/${IMAGE_NAME}:${IMAGE_TAG}

# 2. 优雅停止旧容器（如果有）
if docker ps -q -f name=${CONTAINER_NAME} | grep -q .; then
    echo "Stopping old container..."
    docker stop -t 30 ${CONTAINER_NAME}
    docker rm ${CONTAINER_NAME}
fi

# 3. 启动新容器
echo "Starting new container..."
docker run -d \
  --name ${CONTAINER_NAME} \
  --hostname ${CONTAINER_NAME} \
  -p 8080:8080 \
  -e SPRING_PROFILES_ACTIVE=prod \
  -e JAVA_OPTS="-Xms2g -Xmx2g -XX:+UseG1GC" \
  -v /data/logs:/app/logs \
  -v /data/config:/app/config:ro \
  --memory=2.5g \
  --memory-swap=2.5g \
  --cpus=2.0 \
  --pids-limit=200 \
  --restart=unless-stopped \
  --log-driver=json-file \
  --log-opt max-size=100m \
  --log-opt max-file=5 \
  --security-opt=no-new-privileges:true \
  --read-only \
  --tmpfs /tmp:noexec,nosuid,size=100m \
  --tmpfs /app/logs:noexec,nosuid,size=200m \
  registry.example.com/${IMAGE_NAME}:${IMAGE_TAG}

# 4. 健康检查
echo "Waiting for health check..."
sleep 30

HEALTH_STATUS=$(docker inspect --format='{{.State.Health.Status}}' ${CONTAINER_NAME})
if [ "$HEALTH_STATUS" != "healthy" ]; then
    echo "Health check failed! Rolling back..."
    docker stop ${CONTAINER_NAME}
    docker rm ${CONTAINER_NAME}
    # 启动旧版本（需要保存旧版本信息）
    exit 1
fi

echo "Deployment successful!"
```

### 案例5：CI/CD集成

```yaml
# .github/workflows/docker-build.yml
name: Docker Build and Push

on:
  push:
    branches: [ main ]
    tags: [ 'v*' ]

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up JDK 17
      uses: actions/setup-java@v3
      with:
        java-version: '17'
        distribution: 'temurin'
        
    - name: Build with Maven
      run: mvn clean package -DskipTests
      
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v2
      
    - name: Login to Registry
      uses: docker/login-action@v2
      with:
        registry: registry.example.com
        username: ${{ secrets.REGISTRY_USERNAME }}
        password: ${{ secrets.REGISTRY_PASSWORD }}
        
    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v4
      with:
        images: registry.example.com/myapp
        tags: |
          type=ref,event=branch
          type=ref,event=pr
          type=semver,pattern={{version}}
          type=semver,pattern={{major}}.{{minor}}
          
    - name: Build and push
      uses: docker/build-push-action@v4
      with:
        context: .
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max
```

```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - deploy

variables:
  DOCKER_IMAGE: registry.example.com/myapp
  MAVEN_OPTS: "-Dmaven.repo.local=$CI_PROJECT_DIR/.m2/repository"

cache:
  paths:
    - .m2/repository

build:
  stage: build
  image: maven:3.9-eclipse-temurin-17-alpine
  script:
    - mvn clean package -DskipTests
    - docker build -t $DOCKER_IMAGE:$CI_COMMIT_SHA .
    - docker push $DOCKER_IMAGE:$CI_COMMIT_SHA
  only:
    - main

test:
  stage: test
  image: maven:3.9-eclipse-temurin-17-alpine
  script:
    - mvn test
  artifacts:
    reports:
      junit:
        - target/surefire-reports/*.xml

deploy:
  stage: deploy
  image: alpine/k8s:latest
  script:
    - kubectl set image deployment/myapp app=$DOCKER_IMAGE:$CI_COMMIT_SHA
    - kubectl rollout status deployment/myapp
  environment:
    name: production
    url: https://app.example.com
  only:
    - main
```

### 案例6：Kubernetes部署

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
  labels:
    app: myapp
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
        fsGroup: 1001
      containers:
      - name: app
        image: registry.example.com/myapp:1.0.0
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
          protocol: TCP
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "prod"
        - name: JAVA_OPTS
          value: "-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0 -XX:+UseG1GC"
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 30
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 3
          failureThreshold: 3
        volumeMounts:
        - name: logs
          mountPath: /app/logs
        - name: config
          mountPath: /app/config
          readOnly: true
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
      volumes:
      - name: logs
        emptyDir: {}
      - name: config
        configMap:
          name: myapp-config
      imagePullSecrets:
      - name: registry-secret
---
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: production
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
  namespace: production
data:
  application-prod.yml: |
    server:
      port: 8080
    spring:
      datasource:
        url: jdbc:mysql://mysql:3306/mydb
        username: appuser
        password: ${DB_PASSWORD}
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 100
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
```

---

## 对比分析：Docker vs VM vs Podman

```
┌─────────────────────┬─────────────────┬─────────────────┬─────────────────┐
│       特性          │     Docker      │      VM         │     Podman      │
├─────────────────────┼─────────────────┼─────────────────┼─────────────────┤
│     隔离级别        │    进程级        │   操作系统级     │    进程级        │
├─────────────────────┼─────────────────┼─────────────────┼─────────────────┤
│     启动速度        │     秒级        │     分钟级      │     秒级         │
├─────────────────────┼─────────────────┼─────────────────┼─────────────────┤
│     资源占用        │    轻量(MB级)   │    重(GB级)     │    轻量(MB级)    │
├─────────────────────┼─────────────────┼─────────────────┼─────────────────┤
│     性能            │    接近原生      │    有虚拟化开销  │    接近原生      │
├─────────────────────┼─────────────────┼─────────────────┼─────────────────┤
│     镜像大小        │     小          │     大          │     小           │
├─────────────────────┼─────────────────┼─────────────────┼─────────────────┤
│     操作系统        │    共享宿主机内核 │    独立内核      │    共享宿主机内核 │
├─────────────────────┼─────────────────┼─────────────────┼─────────────────┤
│     安全性          │     中          │     高          │     中           │
├─────────────────────┼─────────────────┼─────────────────┼─────────────────┤
│     生态            │     最丰富      │     成熟        │     较丰富       │
├─────────────────────┼─────────────────┼─────────────────┼─────────────────┤
│     守护进程        │     需要        │     不需要      │     不需要       │
├─────────────────────┼─────────────────┼─────────────────┼─────────────────┤
│     Root权限        │     需要        │     不需要      │     不需要       │
├─────────────────────┼─────────────────┼─────────────────┼─────────────────┤
│     兼容性          │     Docker API  │     无          │     Docker兼容   │
└─────────────────────┴─────────────────┴─────────────────┴─────────────────┘

选择建议：
- 需要完整隔离：VM
- 开发/生产容器化：Docker/Podman
- 安全敏感场景（无Root）：Podman
- 大规模编排：Kubernetes + containerd/CRI-O
```

---

## 性能分析：容器开销与优化

### 容器启动性能

```
启动时间对比：
- 虚拟机：30-60秒
- Docker容器：1-3秒
- 优化后的容器（使用tiny base image）：< 1秒

优化策略：
1. 使用轻量级基础镜像（Alpine、Distroless）
2. 减少镜像层数
3. 使用多阶段构建
4. 避免在启动时执行耗时操作
```

### 资源开销

```
资源占用对比（相同应用）：

VM方式：
- CPU：有虚拟化开销（5-15%）
- 内存：Guest OS占用数百MB
- 磁盘：镜像数GB

容器方式：
- CPU：接近原生（1-5%开销）
- 内存：仅应用本身占用
- 磁盘：镜像数十MB到数百MB

优化策略：
1. CPU：
   - 使用--cpus限制
   - 使用--cpu-shares设置权重
   - 避免CPU节流（throttling）

2. 内存：
   - 设置--memory和--memory-swap
   - 使用JVM的UseContainerSupport
   - 监控OOM事件

3. 磁盘：
   - 使用Volume持久化数据
   - 清理不需要的镜像和容器
   - 使用overlay2存储驱动
```

### 网络性能

```
Docker网络模式性能：

Host模式：
- 性能最好（无NAT开销）
- 无网络隔离

Bridge模式：
- 有NAT开销
- 端口映射有性能损失
- 适合大多数场景

Overlay模式：
- 有VXLAN封装开销
- 适合跨主机通信

优化策略：
1. 高吞吐场景使用Host模式
2. 使用macvlan/ipvlan直接分配IP
3. 调整内核网络参数
```

---

## 常见陷阱与最佳实践

### 1. 容器以root运行

**陷阱：** 默认以root运行容器，若容器被入侵，攻击者直接获得宿主机root权限。

**最佳实践：**
```dockerfile
# 创建非root用户
RUN addgroup -g 1001 -S appgroup && \
    adduser -u 1001 -S appuser -G appgroup

# 设置文件权限
COPY --chown=appuser:appgroup . /app

# 切换用户
USER appuser
```

### 2. 镜像层缓存失效

**陷阱：** Dockerfile中频繁变动的指令放在前面，导致每次构建都重新执行后续指令。

**最佳实践：**
```dockerfile
# 好的Dockerfile：将不常变动的指令放在前面
FROM node:18-alpine
WORKDIR /app

# 先复制package.json（依赖不常变）
COPY package.json package-lock.json ./
RUN npm ci --only=production

# 再复制源码（经常变动）
COPY . .

CMD ["node", "server.js"]
```

### 3. 未设置资源限制

**陷阱：** 容器无限制地使用CPU和内存，导致宿主机资源耗尽，影响其他服务。

**最佳实践：**
```bash
docker run -d \
  --memory=512m \
  --memory-swap=512m \
  --cpus=1.0 \
  --pids-limit=100 \
  myapp:1.0.0
```

### 4. 容器日志无限增长

**陷阱：** 默认json-file日志驱动不限制日志大小，长期运行后占满磁盘。

**最佳实践：**
```bash
docker run -d \
  --log-driver json-file \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  myapp:1.0.0
```

或在daemon.json中全局配置：
```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

### 5. 敏感信息泄露

**陷阱：** 将密码、密钥等敏感信息硬编码在Dockerfile或镜像中。

**最佳实践：**
```dockerfile
# 不好的做法
ENV DB_PASSWORD=mysecretpassword

# 好的做法：通过运行时传入
# docker run -e DB_PASSWORD_FILE=/run/secrets/db_password

# 或使用Docker Secrets（Swarm）
# 或使用Kubernetes Secrets
```

---

## 面试题与参考答案

### Q1：Docker和虚拟机的区别？

**答：**

| 特性 | Docker容器 | 虚拟机 |
|------|-----------|--------|
| 隔离级别 | 进程级隔离 | 操作系统级隔离 |
| 启动速度 | 秒级 | 分钟级 |
| 资源占用 | 轻量（MB级） | 重（GB级） |
| 性能 | 接近原生 | 有虚拟化开销 |
| 镜像大小 | 小（MB级） | 大（GB级） |
| 内核 | 共享宿主机内核 | 独立内核 |

### Q2：Dockerfile中CMD、ENTRYPOINT、RUN的区别？

**答：**
- **RUN**：在镜像构建时执行，用于安装软件、执行命令，产生新镜像层
- **CMD**：容器启动时执行的默认命令，可被docker run的命令覆盖
- **ENTRYPOINT**：容器启动时执行的命令，不易被覆盖（配合CMD提供默认参数）
- **最佳实践**：ENTRYPOINT定义固定命令，CMD提供默认参数

### Q3：多阶段构建的优势？

**答：**
1. **减小镜像体积**：最终镜像只包含运行所需文件，不包含编译工具
2. **提高安全性**：减少攻击面，不暴露源码和构建工具
3. **优化缓存**：不同阶段可独立缓存，加速构建
4. **分层优化**：Spring Boot 2.3+支持分层Jar，提高缓存命中率

### Q4：Docker Compose的作用？

**答：**
- **多服务编排**：用一个YAML文件定义多个服务（应用、数据库、缓存等）
- **依赖管理**：通过depends_on控制服务启动顺序
- **网络配置**：自动创建隔离网络，服务间通过服务名通信
- **数据持久化**：通过volumes管理数据卷
- **适合场景**：开发环境、测试环境、小型生产部署

### Q5：容器数据持久化的方式？

**答：**
1. **Volumes**：Docker管理的数据卷，存储在/var/lib/docker/volumes/，推荐方式
2. **Bind Mounts**：将宿主机目录挂载到容器，适合开发环境
3. **tmpfs Mounts**：存储在内存中，适合临时文件，容器停止后数据丢失
4. **选择建议**：生产环境优先使用Volumes，便于备份和迁移

### Q6：Docker的网络模式有哪些？

**答：** Docker有五种网络模式：

1. **Bridge（默认）**：创建虚拟网桥docker0，容器通过veth pair连接网桥，适合单机多容器通信
2. **Host**：容器共享宿主机网络栈，无网络隔离，性能最好
3. **None**：容器只有lo接口，完全隔离
4. **Container**：新容器共享已有容器的网络栈，适合Sidecar模式
5. **Overlay**：跨主机容器通信，基于VXLAN，适合Swarm/Kubernetes

### Q7：Docker如何实现资源限制？

**答：** Docker通过Linux Cgroups实现资源限制：

- **CPU限制**：
  - --cpus：限制CPU核心数
  - --cpu-shares：设置CPU权重
  - --cpuset-cpus：绑定特定CPU核心

- **内存限制**：
  - --memory：限制内存使用量
  - --memory-swap：限制内存+Swap总量
  - --memory-reservation：软限制

- **IO限制**：
  - --blkio-weight：块设备IO权重
  - --device-read-bps：限制读取速率

- **其他限制**：
  - --pids-limit：限制进程数
  - --ulimit：限制文件句柄数等

### Q8：Docker镜像分层的原理是什么？

**答：** Docker镜像使用UnionFS（联合文件系统）实现分层存储：

1. **分层结构**：
   - 基础镜像层（如Alpine）
   - 依赖层（如JDK）
   - 应用层（如JAR文件）
   - 可写层（容器运行时，Copy-on-Write）

2. **Copy-on-Write**：
   - 读取：从上到下查找文件
   - 写入：复制文件到可写层，然后修改
   - 删除：在可写层创建.whiteout文件标记删除

3. **优势**：
   - 镜像共享基础层，节省存储
   - 镜像构建缓存复用
   - 容器启动快速

### Q9：Docker容器如何实现隔离？

**答：** Docker容器通过Linux Namespace实现隔离：

- **PID Namespace**：进程ID隔离（容器内PID 1是初始化进程）
- **NET Namespace**：网络设备、端口、路由表隔离
- **IPC Namespace**：信号量、消息队列、共享内存隔离
- **MNT Namespace**：挂载点隔离（文件系统视图）
- **UTS Namespace**：主机名和域名隔离
- **USER Namespace**：用户和组ID隔离（UID/GID映射）
- **CGROUP Namespace**：Cgroups根目录隔离

### Q10：Docker和Kubernetes的关系是什么？

**答：** Docker和Kubernetes是互补关系：

- **Docker**：容器运行时和镜像构建工具
- **Kubernetes**：容器编排平台，管理多个Docker容器

**关系演进**：
- Kubernetes早期使用Docker作为容器运行时
- Kubernetes 1.20+弃用Docker（使用containerd/CRI-O）
- Docker镜像格式（OCI）仍是标准
- Docker仍是开发环境的首选工具

### Q11：Docker容器的健康检查是如何工作的？

**答：** Docker健康检查通过HEALTHCHECK指令实现：

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=60s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1
```

**参数说明**：
- **interval**：检查间隔（默认30s）
- **timeout**：超时时间（默认30s）
- **start-period**：启动宽限期（默认0s）
- **retries**：失败重试次数（默认3次）

**健康状态**：
- **starting**：启动宽限期内
- **healthy**：检查通过
- **unhealthy**：连续失败超过retries

### Q12：如何排查Docker容器问题？

**答：** Docker容器排查常用命令：

```bash
# 查看容器日志
docker logs -f --tail=100 container_name

# 查看容器资源使用
docker stats container_name

# 进入容器排查
docker exec -it container_name /bin/sh

# 查看容器详情
docker inspect container_name

# 查看容器进程
docker top container_name

# 查看容器网络
docker network inspect network_name

# 查看容器存储
docker volume inspect volume_name
```

---

*此文原创，转载请注明出处。*
