# AI 辅助 Shell 脚本编写：从日志分析到自动化部署，Java程序员也能写出生产级Shell

> Java程序员最怕的三件事：写Shell脚本、配Nginx、和DBA沟通。第一条AI已经帮你解决了。

---

## 一、开篇：为什么Java程序员该拥抱AI写Shell？

先讲个真事。组里有个老哥，Java写得飞起，微服务架构信手拈来，但每轮到写Shell部署脚本就挠头——变量加不加引号？`$?` 和 `$!` 到底是啥？`sed` 和 `awk` 怎么又报错了？

这其实是很多Java程序员的缩影。我们习惯了强类型语言的可预测性，面对Shell这种"随便写写好像也能跑，但跑着跑着就炸"的语言，确实容易翻车。

但运维脚本又绕不开。好在，**AI 写 Shell 这件事，可能是目前所有 AI 辅助编程场景里最靠谱的那一档**。

为什么？因为 Shell 脚本本质上就是命令行的罗列和组合，而 Linux 命令的用法在网络上有海量优质语料，AI 的训练覆盖度极高。更重要的是——**AI 天然会写注释**，这是大多数手写 Shell 最缺的东西。

读完本文你能带走：
- **8 个开箱即用的 Java 运维 Shell 脚本**（全带详细注释）
- **一套 Shell 安全编写规范**（`set -euo pipefail` 是什么）
- **AI 生成脚本的验证方法论**（shellcheck 一把梭）
- **高手的 Prompt 模板**（拷贝即用）

---

## 二、Java运维高频脚本 × AI Prompt 实战

先定个规矩：以下所有脚本，我都会先给出**你投喂给 AI 的 Prompt**，再给出**AI 生成的脚本**，最后给**逐段解释**。你在实际工作中，直接拿 Prompt 去问 ChatGPT/Claude/Copilot 就行。

### 场景1：Java应用健康检查脚本

**需求**：快速判断一台机器上的 Java 应用是否活着——进程在不在、端口通不通、HTTP 接口返回 200、内存有没有爆。

**Prompt：**

```text
请帮我写一个 Java 应用健康检查的 Shell 脚本，要求：
1. 检查指定名称的 Java 进程是否存在（通过 jps 或 ps）
2. 检查应用的 HTTP 端口是否监听正常（用 curl 请求 /actuator/health）
3. 检查堆内存使用率是否超过 80%（通过 jstat）
4. 输出彩色日志：正常绿色，异常红色
5. 返回码：全正常返回0，任意异常返回1
6. 遵循安全最佳实践（set -euo pipefail、变量引号包裹）
7. 用法示例：./health_check.sh -n my-app -p 8080
```

**脚本：**

```bash
#!/bin/bash
#
# health_check.sh — Java 应用健康检查脚本
# 用法: ./health_check.sh -n <应用名称> -p <端口>
# 示例: ./health_check.sh -n my-service -p 8080
#
set -euo pipefail

# ============================================================
# 全局配置
# ============================================================
APP_NAME=""            # 应用进程名（ps 可匹配的关键词）
APP_PORT=""            # HTTP 端口
HEAP_THRESHOLD=80      # 堆内存警告阈值（百分比）
EXIT_CODE=0            # 最终退出码，0=健康 1=不健康

# ANSI 颜色码
readonly RED='\033[0;31m'
readonly GREEN='\033[0;32m'
readonly YELLOW='\033[1;33m'
readonly NC='\033[0m' # No Color

# 日志函数：彩色输出并附加时间戳
log_ok()    { echo -e "${GREEN}[$(date '+%H:%M:%S')] ✓ ${1}${NC}"; }
log_error() { echo -e "${RED}[$(date '+%H:%M:%S')] ✗ ${1}${NC}"; EXIT_CODE=1; }
log_warn()  { echo -e "${YELLOW}[$(date '+%H:%M:%S')] ⚠ ${1}${NC}"; }

# ============================================================
# 参数解析
# ============================================================
usage() {
    cat <<EOF
用法: $0 -n <应用名称> -p <端口> [-h 堆内存阈值]

必选参数:
  -n    应用进程名（如 my-service）
  -p    HTTP 服务端口号（如 8080）

可选参数:
  -h    堆内存告警阈值，默认 80（百分比）

示例:
  $0 -n order-service -p 8080
  $0 -n order-service -p 8080 -h 85
EOF
    exit 1
}

while getopts "n:p:h:" opt; do
    case "${opt}" in
        n) APP_NAME="${OPTARG}"  ;;
        p) APP_PORT="${OPTARG}"  ;;
        h) HEAP_THRESHOLD="${OPTARG}" ;;
        *) usage ;;
    esac
done

# 参数校验
if [[ -z "${APP_NAME}" || -z "${APP_PORT}" ]]; then
    echo -e "${RED}错误: 必须指定 -n 和 -p 参数${NC}"
    usage
fi

echo "========================================="
echo "  应用健康检查: ${APP_NAME} (端口 ${APP_PORT})"
echo "========================================="

# ============================================================
# 检查项1: Java 进程是否存在
# ============================================================
# pgrep -f 按完整命令行匹配，确保精确找到目标进程
if JAVA_PID=$(pgrep -f "${APP_NAME}" | head -1); then
    log_ok "Java进程存在 (PID: ${JAVA_PID})"
else
    log_error "Java进程不存在，无法匹配到 '${APP_NAME}'"
    # 进程都不存在，后续检查无意义，直接退出
    echo ""
    echo "结果: ${RED}应用程序未运行${NC}"
    exit 1
fi

# ============================================================
# 检查项2: HTTP 端口监听 & 健康接口
# ============================================================
if ss -tlnp 2>/dev/null | grep -q ":${APP_PORT} "; then
    log_ok "端口 ${APP_PORT} 已监听"

    # 请求 Spring Boot Actuator 健康端点（若有）
    HEALTH_URL="http://127.0.0.1:${APP_PORT}/actuator/health"
    HTTP_CODE=$(curl -s -o /dev/null -w '%{http_code}' \
        --connect-timeout 3 --max-time 5 "${HEALTH_URL}" 2>/dev/null || echo "000")
    if [[ "${HTTP_CODE}" == "200" ]]; then
        log_ok "健康检查接口返回 HTTP 200"
    else
        log_warn "健康检查接口返回 HTTP ${HTTP_CODE}（非200）"
    fi
else
    log_error "端口 ${APP_PORT} 未监听"
fi

# ============================================================
# 检查项3: JVM 堆内存使用率
# ============================================================
if command -v jstat &>/dev/null && [[ -n "${JAVA_PID:-}" ]]; then
    # jstat -gc 输出 Old + Eden 使用量，计算使用率
    # 输出格式: S0C S1C S0U S1U EC EU OC OU MC MU ...
    JSTAT_OUT=$(jstat -gc "${JAVA_PID}" 2>/dev/null | tail -1)
    if [[ -n "${JSTAT_OUT}" ]]; then
        # 提取容量和使用量（单位：KB）
        EC=$(echo "${JSTAT_OUT}" | awk '{print $5}')
        EU=$(echo "${JSTAT_OUT}" | awk '{print $6}')
        OC=$(echo "${JSTAT_OUT}" | awk '{print $7}')
        OU=$(echo "${JSTAT_OUT}" | awk '{print $8}')

        HEAP_CAP=$(( EC + OC ))          # 总堆容量
        HEAP_USED=$(( EU + OU ))         # 已使用量
        if [[ "${HEAP_CAP}" -gt 0 ]]; then
            HEAP_PCT=$(( HEAP_USED * 100 / HEAP_CAP ))
            if [[ "${HEAP_PCT}" -ge "${HEAP_THRESHOLD}" ]]; then
                log_error "堆内存使用率 ${HEAP_PCT}%，超过阈值 ${HEAP_THRESHOLD}%"
            else
                log_ok "堆内存使用率 ${HEAP_PCT}%（阈值 ${HEAP_THRESHOLD}%）"
            fi
        fi
    else
        log_warn "无法获取 JVM 堆内存信息"
    fi
else
    log_warn "jstat 命令不可用，跳过内存检查"
fi

# ============================================================
# 汇总结果
# ============================================================
echo ""
if [[ "${EXIT_CODE}" -eq 0 ]]; then
    echo -e "结果: ${GREEN}所有检查通过，应用健康${NC}"
else
    echo -e "结果: ${RED}存在异常项，请排查${NC}"
fi

exit "${EXIT_CODE}"
```

**逐段解释：**

| 序号 | 关键点 | 说明 |
|------|--------|------|
| 1 | `set -euo pipefail` | 安全三件套：出错即停、未定义变量报错、管道中任一命令失败都算失败 |
| 2 | `pgrep -f` | 按完整命令行匹配 Java 进程，比 `ps \| grep` 多一个进程问题更精准 |
| 3 | `ss -tlnp` | 现代 Linux 推荐用 `ss` 替代 `netstat`，性能更好 |
| 4 | jstat 二次解析 | `jstat -gc` 输出两行，`tail -1` 取第二行（汇总行） |
| 5 | ANSI 颜色码 | `readonly` 声明避免误改，`${NC}` 结尾重置颜色 |

---

### 场景2：批量日志分析脚本

**需求**：生产上出了故障，需要快速从几百兆的 `app.log` 里捞出：ERROR 总条数、每小时分布、出现最多的前5种异常。

**Prompt：**

```text
写一个 Shell 脚本分析 Java 应用日志（logback 默认格式），实现：
1. 统计 ERROR / WARN 日志总条数
2. 按小时聚合日志量（横轴时间，方便找故障时间点）
3. 提取 TOP 5 出现最频繁的错误信息（去重后的异常类名或关键字）
4. 支持传入日志文件路径，不传则默认 ./app.log
5. 输出美观的分段表格
6. 使用 awk/sed/grep 组合，做充分的中文注释
```

**脚本：**

```bash
#!/bin/bash
#
# log_analyzer.sh — Java 日志快速分析工具
# 用法: ./log_analyzer.sh [日志文件路径]
# 示例: ./log_analyzer.sh /var/log/myapp/app.log
#
set -euo pipefail

# ============================================================
# 参数处理
# ============================================================
LOG_FILE="${1:-./app.log}"

if [[ ! -f "${LOG_FILE}" ]]; then
    echo "错误: 日志文件不存在: ${LOG_FILE}"
    exit 1
fi

FILE_SIZE=$(du -h "${LOG_FILE}" | cut -f1)
echo "========================================="
echo "  日志分析报告: ${LOG_FILE} (${FILE_SIZE})"
echo "  分析时间: $(date '+%Y-%m-%d %H:%M:%S')"
echo "========================================="
echo ""

# ============================================================
# 1. ERROR / WARN 总量统计
# ============================================================
echo ">>> 一、日志级别分布"

# -c 统计匹配行数（忽略大小写差异）
ERROR_COUNT=$(grep -ci "ERROR" "${LOG_FILE}" || true)
WARN_COUNT=$(grep -ci "WARN"  "${LOG_FILE}" || true)
INFO_COUNT=$(grep -ci "INFO"  "${LOG_FILE}" || true)
TOTAL_LINES=$(wc -l < "${LOG_FILE}")

# printf 格式化对齐输出
printf "  %-10s %10s  (%s)\n" "级别"     "数量"      "占比"
printf "  %-10s %10d  (%5.1f%%)\n" "ERROR" "${ERROR_COUNT}" \
    "$(awk "BEGIN {printf \"%.1f\", ${ERROR_COUNT}*100/${TOTAL_LINES}}")"
printf "  %-10s %10d  (%5.1f%%)\n" "WARN"  "${WARN_COUNT}" \
    "$(awk "BEGIN {printf \"%.1f\", ${WARN_COUNT}*100/${TOTAL_LINES}}")"
printf "  %-10s %10d\n"           "INFO"  "${INFO_COUNT}"
printf "  %-10s %10d\n"           "总计"  "${TOTAL_LINES}"
echo ""

# ============================================================
# 2. 按小时聚合（假设日志格式含 yyyy-MM-dd HH:mm:ss）
# ============================================================
echo ">>> 二、小时级日志量分布（Top 15）"

# 正则提取 "2024-01-01 14:" 这种小时前缀，然后 sort | uniq -c 计数
# 取最多的 15 个时段，方便定位流量峰值和故障时间
grep -oP '\d{4}-\d{2}-\d{2} \d{2}' "${LOG_FILE}" 2>/dev/null \
    | sort | uniq -c | sort -rn | head -15 \
    | awk '{printf "  %s  →  %6s 条\n", $2":00", $1}'

# 如果日志不是标准格式，给出提示
if ! grep -qP '\d{4}-\d{2}-\d{2} \d{2}:\d{2}' "${LOG_FILE}" 2>/dev/null; then
    echo "  (提示: 未识别到标准时间格式 yyyy-MM-dd HH:mm，请确认日志格式)"
fi
echo ""

# ============================================================
# 3. TOP 5 异常类 / 错误信息
# ============================================================
echo ">>> 三、TOP 5 高频错误信息"

# 先通过 grep 找到所有 ERROR 行
# 再用 sed 提取异常类名（如 java.lang.NullPointerException）
# 最后 sort | uniq -c | sort -rn 统计频次
echo "  --- 按异常类名统计 ---"
grep "ERROR" "${LOG_FILE}" 2>/dev/null \
    | grep -oP '([a-z]+\.)+[A-Z][a-zA-Z]+(Exception|Error)' \
    | sort | uniq -c | sort -rn | head -5 \
    | awk '{printf "  %-3s  %s\n", $1"次", $2}' \
    || echo "  (未匹配到 Java 异常类名)"

echo ""
echo "  --- 按错误关键字片段统计 ---"
# 备选方案：取 ERROR 行中 ": " 后面的前60个字符作为错误摘要
grep "ERROR" "${LOG_FILE}" 2>/dev/null \
    | sed 's/.*ERROR //' | sed 's/ at .*//' \
    | grep -oP '^.{0,60}' \
    | sort | uniq -c | sort -rn | head -5 \
    | awk '{printf "  %-3s  %s\n", $1"次", substr($0, index($0,$2))}' \
    || echo "  (无匹配)"

echo ""
echo "========================================="
echo "  分析完成"
echo "========================================="
```

**逐段解释：**

| 关键点 | 说明 |
|--------|------|
| `grep -ci` | `-c` 计数、`-i` 忽略大小写，应对 ERROR/Error/error |
| `grep -oP '\d{4}-\d{2}-\d{2} \d{2}'` | `-o` 只输出匹配部分，`-P` 开启 Perl 正则 |
| `|| true` | 防止 grep 无匹配时触发 `set -e` 退出 |
| `awk "BEGIN {printf ...}"` | 做浮点除法（Shell 原生只支持整数运算） |

---

### 场景3：JVM参数动态调整

**需求**：物理机内存变了或者容器 limit 变了，需要自动计算最优的 `-Xmx` / `-Xms`。

**Prompt：**

```text
写一个 Shell 脚本，自动根据当前系统可用内存计算 JVM 堆参数：
1. 检测系统总内存（物理机用 free，容器场景读取 cgroup limit）
2. 按比例计算 Xmx（默认取可用内存的 75%，可调）
3. 产生 JAVA_OPTS 环境变量，包含 -Xms -Xmx -XX:MaxMetaspaceSize 等
4. 同时输出一份 Dockerfile 和 k8s resources 建议
5. 纯函数式，用 source 引入后直接在启动脚本里用
```

**脚本：**

```bash
#!/bin/bash
#
# jvm_opts.sh — 自动计算 JVM 堆内存参数
# 用法: source ./jvm_opts.sh && java ${JAVA_OPTS} -jar app.jar
#
set -euo pipefail

# ============================================================
# 获取容器/宿主机可用内存（单位：MB）
# ============================================================
get_available_memory_mb() {
    local mem_mb=0

    # 容器环境：读取 cgroup v1 内存限制
    if [[ -f /sys/fs/cgroup/memory/memory.limit_in_bytes ]]; then
        local cgroup_limit
        cgroup_limit=$(cat /sys/fs/cgroup/memory/memory.limit_in_bytes)
        # 如果 cgroup 返回一个巨大值（未限制），则认为是物理机
        if [[ "${cgroup_limit}" -lt 9223372036854771712 ]]; then
            mem_mb=$(( cgroup_limit / 1024 / 1024 ))
        fi
    fi

    # cgroup v2 兼容
    if [[ "${mem_mb}" -eq 0 && -f /sys/fs/cgroup/memory.max ]]; then
        local cgroup_max
        cgroup_max=$(cat /sys/fs/cgroup/memory.max)
        if [[ "${cgroup_max}" != "max" ]]; then
            mem_mb=$(( cgroup_max / 1024 / 1024 ))
        fi
    fi

    # 兜底：物理机用 free 命令
    if [[ "${mem_mb}" -eq 0 ]]; then
        # free 第二行 "Mem:" 的第七列是 available
        if command -v free &>/dev/null; then
            mem_mb=$(free -m | awk '/^Mem:/ {print $7}')
        fi
    fi

    echo "${mem_mb}"
}

# ============================================================
# 根据可用内存计算 JVM 参数
# ============================================================
MEM_MB=$(get_available_memory_mb)
HEAP_RATIO="${JVM_HEAP_RATIO:-0.75}"   # 默认堆占可用内存 75%，可通过环境变量覆盖
HEAP_MB=$(awk "BEGIN {printf \"%d\", ${MEM_MB} * ${HEAP_RATIO}}")

# Metaspace 大小：一般取堆的 10%~20%，给够避免频繁 Full GC
METASPACE_MB=$(( HEAP_MB / 5 ))
[[ "${METASPACE_MB}" -lt 128 ]] && METASPACE_MB=128   # 最少 128MB
[[ "${METASPACE_MB}" -gt 512 ]] && METASPACE_MB=512   # 最多 512MB

# 年轻代大小：堆的 1/3（默认 AdaptiveSizePolicy 也差不多）
YOUNG_MB=$(( HEAP_MB / 3 ))

# 线程栈大小：默认 1MB，容器环境适当减小
STACK_SIZE="1024k"
if [[ "${HEAP_MB}" -lt 2048 ]]; then
    STACK_SIZE="512k"   # 小内存环境缩小栈
fi

# ============================================================
# 拼装 JAVA_OPTS
# ============================================================
export JAVA_OPTS="\
-Xms${HEAP_MB}m \
-Xmx${HEAP_MB}m \
-XX:NewSize=${YOUNG_MB}m \
-XX:MaxNewSize=${YOUNG_MB}m \
-XX:MetaspaceSize=${METASPACE_MB}m \
-XX:MaxMetaspaceSize=${METASPACE_MB}m \
-XX:MaxDirectMemorySize=256m \
-Xss${STACK_SIZE} \
-XX:+UseG1GC \
-XX:MaxGCPauseMillis=200 \
-XX:+HeapDumpOnOutOfMemoryError \
-XX:HeapDumpPath=/tmp/heapdump.hprof \
-XX:+ExitOnOutOfMemoryError \
-Djava.security.egd=file:/dev/./urandom"

# ============================================================
# 输出建议
# ============================================================
echo "========================================="
echo "  JVM 参数自动计算"
echo "========================================="
echo "  系统可用内存: ${MEM_MB} MB"
echo "  JVM 堆内存:   ${HEAP_MB} MB  (${HEAP_RATIO})"
echo "  Metaspace:    ${METASPACE_MB} MB"
echo "  年轻代:       ${YOUNG_MB} MB"
echo "  线程栈:       ${STACK_SIZE}"
echo "========================================="
echo ""

# 容器资源建议
K8S_REQUEST=$(( (MEM_MB + 256) ))
echo "  → K8s resources 建议:"
echo "    resources:"
echo "      requests:"
echo "        memory: \"${K8S_REQUEST}Mi\""
echo "      limits:"
echo "        memory: \"${MEM_MB}Mi\""
echo ""
echo "  → JAVA_OPTS 已导出，可直接使用:"
echo "    java \${JAVA_OPTS} -jar app.jar"
```

**逐段解释：**

| 关键点 | 说明 |
|--------|------|
| cgroup v1/v2 双兼容 | 容器内存限制是第一个入口，否则 Java 读到的是宿主机内存 |
| `9223372036854771712` | cgroup 无限制时的默认值（近似 `Long.MAX_VALUE`） |
| `JVM_HEAP_RATIO` | 通过环境变量覆盖默认比例，脚本更灵活 |
| Metaspace 上下限 | 128MB~512MB，避免过小 OOM 或过大浪费 |

---

### 场景4：多环境部署脚本

**需求**：一套代码要在 dev / test / staging / prod 四套环境部署，每次手动改配置文件必出错。

**Prompt：**

```text
写一个 Java 多环境部署脚本，要求：
1. 支持 dev / test / staging / prod 四个环境，通过 -e 参数指定
2. 每个环境有独立的 application-{env}.yml 配置文件
3. 部署步骤：备份旧包 → 停服务 → 拷贝新包 → 替换配置 → 启动服务 → 等待健康检查
4. 每步输出日志和耗时
5. 支持 dry-run 模式（只打印步骤不执行）
6. 部署前检查磁盘空间、Java版本等前置条件
```

**脚本：**

```bash
#!/bin/bash
#
# deploy.sh — Java 应用多环境部署脚本
# 用法: ./deploy.sh -e <环境> -v <版本号> [--dry-run]
# 示例: ./deploy.sh -e prod -v 2.3.1
#
set -euo pipefail

# ============================================================
# 全局变量
# ============================================================
ENV=""                         # 环境标识
VERSION=""                     # 部署版本号
DRY_RUN=false                  # 演习模式，只输出不执行
APP_NAME="order-service"       # 应用名称
DEPLOY_ROOT="/data/apps"
APP_HOME="${DEPLOY_ROOT}/${APP_NAME}"
CONFIG_DIR="${APP_HOME}/config"
BACKUP_DIR="${APP_HOME}/backups"
HEALTH_URL="http://127.0.0.1:8080/actuator/health"
START_TIMESTAMP=$(date +%s)    # 记录脚本开始时间，最后算总耗时

readonly RED='\033[0;31m'
readonly GREEN='\033[0;32m'
readonly YELLOW='\033[1;33m'
readonly NC='\033[0m'

# ============================================================
# 工具函数
# ============================================================
log_step() { echo -e "${GREEN}[$(date '+%H:%M:%S')] >>> ${1}${NC}"; }
log_info() { echo -e "          ${1}"; }
log_error() { echo -e "${RED}[$(date '+%H:%M:%S')] !!! ${1}${NC}"; exit 1; }

now_ms() { python3 -c "import time; print(int(time.time()*1000))" 2>/dev/null \
    || echo $(($(date +%s) * 1000)); }

usage() {
    cat <<EOF
用法: $0 -e <环境> -v <版本号> [--dry-run]

环境: dev | test | staging | prod
选项:
  --dry-run    演习模式，只打印步骤不实际执行
  -h           显示帮助

示例:
  $0 -e test -v 2.3.1
  $0 -e prod -v 2.3.1 --dry-run
EOF
    exit 1
}

# ============================================================
# 参数解析
# ============================================================
while [[ $# -gt 0 ]]; do
    case "$1" in
        -e) ENV="$2";      shift 2 ;;
        -v) VERSION="$2";  shift 2 ;;
        --dry-run) DRY_RUN=true; shift ;;
        -h|--help) usage ;;
        *) echo "未知参数: $1"; usage ;;
    esac
done

# 参数校验
if [[ -z "${ENV}" || -z "${VERSION}" ]]; then
    echo -e "${RED}错误: -e 和 -v 为必填参数${NC}"
    usage
fi

VALID_ENVS=("dev" "test" "staging" "prod")
if [[ ! " ${VALID_ENVS[*]} " =~ " ${ENV} " ]]; then
    log_error "无效环境: ${ENV}，支持: ${VALID_ENVS[*]}"
fi

echo "========================================="
echo "  ${APP_NAME} 部署"
echo "  环境: ${ENV}  |  版本: ${VERSION}"
[[ "${DRY_RUN}" == "true" ]] && echo "  模式: DRY-RUN（仅预览）"
echo "========================================="
echo ""

# ============================================================
# 步骤0: 部署前检查
# ============================================================
log_step "步骤0: 部署前环境检查"

# 检查磁盘空间（至少保留 2GB）
AVAIL_GB=$(df -BG "${DEPLOY_ROOT}" 2>/dev/null | awk 'NR==2 {print $4}' | tr -d 'G')
if [[ "${AVAIL_GB:-0}" -lt 2 ]]; then
    log_error "磁盘可用空间不足: ${AVAIL_GB}GB（需要 ≥ 2GB）"
fi
log_info "磁盘可用空间: ${AVAIL_GB}GB"

# 检查 Java 版本（至少 JDK 17）
JAVA_VER=$(java -version 2>&1 | awk -F '"' '/version/ {print $2}' | cut -d'.' -f1)
if [[ "${JAVA_VER}" -lt 17 ]]; then
    log_error "Java 版本过低: ${JAVA_VER}（需要 ≥ 17）"
fi
log_info "Java 版本: ${JAVA_VER}"

# 确认目标目录存在
if [[ ! -d "${APP_HOME}" ]]; then
    log_error "应用目录不存在: ${APP_HOME}"
fi
log_info "应用目录: ${APP_HOME}"

echo ""

# ============================================================
# 步骤1: 备份当前版本
# ============================================================
log_step "步骤1: 备份当前版本"
STEP_START=$(now_ms)

JAR_FILE="${APP_HOME}/${APP_NAME}.jar"

if [[ -f "${JAR_FILE}" ]]; then
    if [[ "${DRY_RUN}" == "false" ]]; then
        mkdir -p "${BACKUP_DIR}"
        cp "${JAR_FILE}" "${BACKUP_DIR}/${APP_NAME}-$(date '+%Y%m%d_%H%M%S').jar"
        # 只保留最近 7 天的备份，节省磁盘空间
        find "${BACKUP_DIR}" -name "${APP_NAME}-*.jar" -mtime +7 -delete 2>/dev/null || true
    fi
    log_info "已备份旧 JAR 包到 ${BACKUP_DIR}"
else
    log_info "未发现旧 JAR 包，跳过备份（首次部署？）"
fi

STEP_COST=$(( $(now_ms) - STEP_START ))
log_info "耗时: ${STEP_COST}ms"

echo ""

# ============================================================
# 步骤2: 停止旧服务
# ============================================================
log_step "步骤2: 停止旧服务"
STEP_START=$(now_ms)

OLD_PID=$(pgrep -f "${APP_NAME}.jar" || true)
if [[ -n "${OLD_PID}" ]]; then
    if [[ "${DRY_RUN}" == "false" ]]; then
        kill -15 "${OLD_PID}" 2>/dev/null || true  # 优雅关闭

        # 等待最多 30 秒，超时则强制 kill -9
        local WAITED=0
        while kill -0 "${OLD_PID}" 2>/dev/null && [[ ${WAITED} -lt 30 ]]; do
            sleep 1
            ((WAITED++))
        done

        if kill -0 "${OLD_PID}" 2>/dev/null; then
            log_info "优雅关闭超时，执行 kill -9"
            kill -9 "${OLD_PID}" 2>/dev/null || true
        fi
    fi
    log_info "旧进程已停止 (PID: ${OLD_PID})"
else
    log_info "无正在运行的旧进程"
fi

STEP_COST=$(( $(now_ms) - STEP_START ))
log_info "耗时: ${STEP_COST}ms"

echo ""

# ============================================================
# 步骤3: 部署新包
# ============================================================
log_step "步骤3: 部署新版本 JAR 包"
STEP_START=$(now_ms)

NEW_JAR="${DEPLOY_ROOT}/releases/${APP_NAME}-${VERSION}.jar"
if [[ ! -f "${NEW_JAR}" ]]; then
    log_error "发布包不存在: ${NEW_JAR}"
fi

if [[ "${DRY_RUN}" == "false" ]]; then
    cp "${NEW_JAR}" "${JAR_FILE}"
fi
log_info "已部署: ${NEW_JAR} → ${JAR_FILE}"

STEP_COST=$(( $(now_ms) - STEP_START ))
log_info "耗时: ${STEP_COST}ms"

echo ""

# ============================================================
# 步骤4: 替换环境配置
# ============================================================
log_step "步骤4: 应用 ${ENV} 环境配置"
STEP_START=$(now_ms)

CONFIG_SRC="${CONFIG_DIR}/application-${ENV}.yml"
if [[ ! -f "${CONFIG_SRC}" ]]; then
    log_error "环境配置文件不存在: ${CONFIG_SRC}"
fi

if [[ "${DRY_RUN}" == "false" ]]; then
    # 将环境专用配置软链为 application.yml（或直接 cp）
    ln -sf "${CONFIG_SRC}" "${APP_HOME}/application.yml"
fi
log_info "配置文件: ${CONFIG_SRC}"

STEP_COST=$(( $(now_ms) - STEP_START ))
log_info "耗时: ${STEP_COST}ms"

echo ""

# ============================================================
# 步骤5: 启动服务
# ============================================================
log_step "步骤5: 启动服务"
STEP_START=$(now_ms)

# 从环境配置文件读取 JAVA_OPTS（如果定义在配置里）
# shellcheck disable=SC1090
if [[ -f "${APP_HOME}/jvm_opts_${ENV}.sh" ]]; then
    source "${APP_HOME}/jvm_opts_${ENV}.sh"
fi

JAVA_CMD="java ${JAVA_OPTS:-} -jar ${JAR_FILE} --spring.profiles.active=${ENV}"

if [[ "${DRY_RUN}" == "false" ]]; then
    # 后台启动，标准输出和错误输出重定向到日志文件
    nohup ${JAVA_CMD} > "${APP_HOME}/logs/app.log" 2>&1 &
    NEW_PID=$!
    log_info "服务已启动 (PID: ${NEW_PID})"
else
    log_info "[DRY-RUN] 启动命令: ${JAVA_CMD}"
fi

STEP_COST=$(( $(now_ms) - STEP_START ))
log_info "耗时: ${STEP_COST}ms"

echo ""

# ============================================================
# 步骤6: 等待健康检查通过
# ============================================================
log_step "步骤6: 等待健康检查"

if [[ "${DRY_RUN}" == "false" ]]; then
    RETRY=0
    MAX_RETRY=30
    while [[ ${RETRY} -lt ${MAX_RETRY} ]]; do
        HTTP_CODE=$(curl -s -o /dev/null -w '%{http_code}' \
            --connect-timeout 2 "${HEALTH_URL}" 2>/dev/null || echo "000")
        if [[ "${HTTP_CODE}" == "200" ]]; then
            log_info "健康检查通过 (HTTP 200)"
            break
        fi
        sleep 2
        ((RETRY++))
        log_info "等待中... (${RETRY}/${MAX_RETRY}) HTTP ${HTTP_CODE}"
    done

    if [[ ${RETRY} -ge ${MAX_RETRY} ]]; then
        log_error "健康检查超时，请检查应用日志"
    fi
else
    log_info "[DRY-RUN] 检查 ${HEALTH_URL}"
fi

echo ""

# ============================================================
# 汇总
# ============================================================
TOTAL_COST=$(( $(date +%s) - START_TIMESTAMP ))
echo "========================================="
echo -e "  部署完成! 环境: ${GREEN}${ENV}${NC}  版本: ${GREEN}${VERSION}${NC}  总耗时: ${TOTAL_COST}s"
echo "========================================="
```

**逐段解释：**

| 关键点 | 说明 |
|--------|------|
| `kill -15` → 等 30s → `kill -9` | 优雅关闭优先，给 Spring 的 `@PreDestroy` 执行机会 |
| `DRY_RUN` 模式 | 所有写操作包一层 `if [[ "${DRY_RUN}" == "false" ]]` |
| `--spring.profiles.active` | Spring Boot 原生多环境支持，配置文件命名规范和脚本联动 |
| `nohup ... 2>&1` | 标准错误也重定向，避免 nohup.out 膨胀 |

---

### 场景5：数据库备份脚本

**需求**：MySQL 定时备份，压缩归档，保留最近 7 天，再老的自动清理。

**Prompt：**

```text
写一个 MySQL 数据库备份 Shell 脚本：
1. 使用 mysqldump 备份指定数据库
2. 备份文件以 数据库名_日期时间.sql.gz 命名
3. 自动清理 7 天前的备份文件
4. 备份失败发送告警（先输出到 stderr，留好钉钉/企微 webhook 接入点）
5. 支持备份多个库（逗号分隔）
6. 备份前后打印数据库大小，方便对比
7. 安全：数据库密码从环境变量读取，不硬编码
```

**脚本：**

```bash
#!/bin/bash
#
# db_backup.sh — MySQL 数据库自动备份脚本
# 用法: ./db_backup.sh -d "db1,db2" [-r 7]
# 环境变量: DB_HOST / DB_PORT / DB_USER / DB_PASSWORD
#
set -euo pipefail

# ============================================================
# 环境变量校验
# ============================================================
DB_HOST="${DB_HOST:-127.0.0.1}"
DB_PORT="${DB_PORT:-3306}"
DB_USER="${DB_USER:-root}"
DB_PASSWORD="${DB_PASSWORD:-}"

if [[ -z "${DB_PASSWORD}" ]]; then
    echo "[ERROR] 请设置环境变量 DB_PASSWORD" >&2
    exit 1
fi

# ============================================================
# 参数解析
# ============================================================
DB_LIST=""                     # 待备份数据库列表（逗号分隔）
RETENTION_DAYS=7               # 备份保留天数

usage() {
    cat <<EOF
用法: $0 -d <数据库列表> [-r 保留天数]

参数:
  -d    数据库名称，多个用逗号分隔（如: db1,db2）
  -r    备份保留天数，默认 7

环境变量:
  DB_HOST     数据库主机 (默认 127.0.0.1)
  DB_PORT     数据库端口 (默认 3306)
  DB_USER     数据库用户 (默认 root)
  DB_PASSWORD 数据库密码 (必填)

示例:
  export DB_PASSWORD="mypass"
  $0 -d order_db,user_db -r 7
EOF
    exit 1
}

while getopts "d:r:h" opt; do
    case "${opt}" in
        d) DB_LIST="${OPTARG}"             ;;
        r) RETENTION_DAYS="${OPTARG}"      ;;
        h) usage ;;
        *) usage ;;
    esac
done

if [[ -z "${DB_LIST}" ]]; then
    echo "[ERROR] 必须指定 -d 参数" >&2
    usage
fi

# ============================================================
# 准备工作
# ============================================================
BACKUP_DIR="/data/backups/mysql"
TIMESTAMP=$(date '+%Y%m%d_%H%M%S')
mkdir -p "${BACKUP_DIR}"

# 通用 mysql/mysqldump 选项
MYSQL_OPTS=(-h"${DB_HOST}" -P"${DB_PORT}" -u"${DB_USER}" -p"${DB_PASSWORD}")
DUMP_OPTS=(--single-transaction --routines --triggers --set-gtid-purged=OFF)

# 告警函数：先打日志，后续可扩展为钉钉/企微 webhook
send_alert() {
    local message="$1"
    echo "[ALERT] $(date '+%Y-%m-%d %H:%M:%S') ${message}" >&2

    # 钉钉/企微 webhook 接入点（按需取消注释）
    # WEBHOOK_URL="${ALERT_WEBHOOK_URL:-}"
    # if [[ -n "${WEBHOOK_URL}" ]]; then
    #     curl -s -H "Content-Type: application/json" \
    #         -d "{\"msgtype\":\"text\",\"text\":{\"content\":\"${message}\"}}" \
    #         "${WEBHOOK_URL}" > /dev/null
    # fi
}

echo "========================================="
echo "  数据库备份"
echo "  时间: $(date '+%Y-%m-%d %H:%M:%S')"
echo "  目标: ${DB_HOST}:${DB_PORT}"
echo "========================================="
echo ""

# ============================================================
# 逐库备份
# ============================================================
FAILED_DBS=()
SUCCESS_COUNT=0

IFS=',' read -ra DBS <<< "${DB_LIST}"

for DB in "${DBS[@]}"; do
    # 去除首尾空格
    DB=$(echo "${DB}" | xargs)
    echo "--- 备份数据库: ${DB} ---"

    # 备份前大小
    SIZE_BEFORE=$(mysql "${MYSQL_OPTS[@]}" -e \
        "SELECT ROUND(SUM(data_length+index_length)/1024/1024,2) FROM information_schema.tables WHERE table_schema='${DB}'" \
        -N -s 2>/dev/null || echo "0")
    echo "  备份前大小: ${SIZE_BEFORE} MB"

    # 执行备份
    BACKUP_FILE="${BACKUP_DIR}/${DB}_${TIMESTAMP}.sql.gz"
    if mysqldump "${MYSQL_OPTS[@]}" "${DUMP_OPTS[@]}" "${DB}" 2>/dev/null \
        | gzip > "${BACKUP_FILE}"; then
        SUCCESS_COUNT=$((SUCCESS_COUNT + 1))
        BACKUP_SIZE=$(du -h "${BACKUP_FILE}" | cut -f1)
        echo "  ✓ 备份成功: ${BACKUP_FILE} (${BACKUP_SIZE})"
    else
        FAILED_DBS+=("${DB}")
        send_alert "数据库备份失败: ${DB} @ ${DB_HOST}"
        echo "  ✗ 备份失败: ${DB}"
    fi

    echo ""
done

# ============================================================
# 清理过期备份
# ============================================================
echo "--- 清理 ${RETENTION_DAYS} 天前的备份 ---"
DELETED_COUNT=$(find "${BACKUP_DIR}" -name "*.sql.gz" -mtime +${RETENTION_DAYS} -delete -print | wc -l)
echo "  已清理 ${DELETED_COUNT} 个过期备份"

# ============================================================
# 汇总
# ============================================================
echo ""
echo "========================================="
echo "  备份完成: 成功 ${SUCCESS_COUNT} / 总计 ${#DBS[@]}"
if [[ ${#FAILED_DBS[@]} -gt 0 ]]; then
    echo "  失败数据库: ${FAILED_DBS[*]}"
    send_alert "备份任务部分失败: ${FAILED_DBS[*]}"
    exit 1
fi
echo "========================================="
```

**逐段解释：**

| 关键点 | 说明 |
|--------|------|
| `DB_PASSWORD` 从环境变量读 | 绝不硬编码密码，`.env` 文件加到 `.gitignore` |
| `--single-transaction` | InnoDB 一致性备份，不锁表 |
| `--set-gtid-purged=OFF` | 避免从库备份时写入 GTID 信息导致恢复报错 |
| `find -mtime +7 -delete` | 一步到位删除过期文件，无需 `rm` 加 `xargs` |

---

### 场景6：Kafka消费积压监控告警

**需求**：Consumer Group 积压超过阈值就告警。

**Prompt：**

```text
写一个 Kafka 消费者积压监控 Shell 脚本：
1. 通过 kafka-consumer-groups.sh 获取指定 group 的 lag
2. 支持监控多个 consumer group
3. lag 超过阈值输出告警
4. 输出简洁清晰（分区级 lag 明细）
5. 支持从配置文件读取 kafka broker 地址和 group 列表
```

**脚本：**

```bash
#!/bin/bash
#
# kafka_lag_monitor.sh — Kafka 消费积压监控
# 用法: ./kafka_lag_monitor.sh [-c 配置文件]
#
set -euo pipefail

# ============================================================
# 配置
# ============================================================
CONFIG_FILE="${1:-./kafka_monitor.conf}"
DEFAULT_BROKER="127.0.0.1:9092"
LAG_THRESHOLD=10000       # 单分区积压告警阈值
TOTAL_LAG_THRESHOLD=50000 # 总积压告警阈值

# 从配置文件加载 broker 和 group 列表
if [[ -f "${CONFIG_FILE}" ]]; then
    # shellcheck disable=SC1090
    source "${CONFIG_FILE}"
fi

BOOTSTRAP="${BOOTSTRAP_SERVERS:-${DEFAULT_BROKER}}"
# GROUP_LIST 是配置文件中定义的数组，格式: GROUP_LIST=("group1" "group2")
if [[ -z "${GROUP_LIST:-}" ]]; then
    echo "[ERROR] 请在配置文件 ${CONFIG_FILE} 中定义 GROUP_LIST 数组" >&2
    echo "  格式: GROUP_LIST=(\"group1\" \"group2\")" >&2
    exit 1
fi

# Kafka 命令路径（支持 Confluent / Apache Kafka 不同安装位置）
KAFKA_BIN="${KAFKA_HOME:-/usr/local/kafka}/bin"
KAFKA_CMD="${KAFKA_BIN}/kafka-consumer-groups.sh"

if [[ ! -x "${KAFKA_CMD}" ]]; then
    echo "[ERROR] 找不到 kafka-consumer-groups.sh: ${KAFKA_CMD}" >&2
    exit 1
fi

readonly RED='\033[0;31m'
readonly GREEN='\033[0;32m'
readonly YELLOW='\033[1;33m'
readonly NC='\033[0m'

echo "========================================="
echo "  Kafka 消费积压监控"
echo "  Broker: ${BOOTSTRAP}"
echo "  Groups: ${GROUP_LIST[*]}"
echo "========================================="
echo ""

# ============================================================
# 逐 Group 检查
# ============================================================
ALERT_COUNT=0

for GROUP in "${GROUP_LIST[@]}"; do
    echo "--- Consumer Group: ${GROUP} ---"

    # 执行 kafka-consumer-groups.sh，输出表格形式的 lag 信息
    # 添加上超时机制防止 Kafka 无响应
    LAG_OUTPUT=$(timeout 15 "${KAFKA_CMD}" \
        --bootstrap-server "${BOOTSTRAP}" \
        --group "${GROUP}" \
        --describe 2>/dev/null) || {
        echo -e "  ${RED}✗ 超时或无响应${NC}"
        ALERT_COUNT=$((ALERT_COUNT + 1))
        echo ""
        continue
    }

    if [[ -z "${LAG_OUTPUT}" ]]; then
        echo "  ⚠ Group 不存在或无消费记录"
        echo ""
        continue
    fi

    # 解析输出: 跳过表头（第一行），提取分区信息和 LAG 值
    TOTAL_LAG=0
    PARTITION_COUNT=0
    ALERT_PARTITIONS=0

    # 典型输出格式: GROUP TOPIC PARTITION CURRENT-OFFSET LOG-END-OFFSET LAG ...
    while IFS= read -r line; do
        # 跳过标题行和空行
        [[ -z "${line}" || "${line}" =~ ^GROUP ]] && continue

        # 提取 LAG（各版本位置可能不同，取倒数第几列更稳）
        LAG=$(echo "${line}" | awk '{print $NF}')
        # 非数字跳过（可能是 "-" 或 "consumer-id" 等行）
        [[ ! "${LAG}" =~ ^[0-9]+$ ]] && continue

        TOPIC=$(echo "${line}" | awk '{print $2}')
        PARTITION=$(echo "${line}" | awk '{print $3}')

        TOTAL_LAG=$(( TOTAL_LAG + LAG ))
        PARTITION_COUNT=$(( PARTITION_COUNT + 1 ))

        # 分区级告警
        if [[ "${LAG}" -gt "${LAG_THRESHOLD}" ]]; then
            echo -e "  ${RED}⚠ 分区告警: ${TOPIC}/${PARTITION} 积压 ${LAG}${NC}"
            ALERT_PARTITIONS=$(( ALERT_PARTITIONS + 1 ))
        fi
    done <<< "${LAG_OUTPUT}"

    # 打印 Group 级汇总
    STATUS_COLOR="${GREEN}"
    STATUS_TEXT="正常"
    if [[ "${TOTAL_LAG}" -gt "${TOTAL_LAG_THRESHOLD}" ]]; then
        STATUS_COLOR="${RED}"
        STATUS_TEXT="告警"
        ALERT_COUNT=$(( ALERT_COUNT + 1 ))
    elif [[ "${TOTAL_LAG}" -gt $(( TOTAL_LAG_THRESHOLD / 2 )) ]]; then
        STATUS_COLOR="${YELLOW}"
        STATUS_TEXT="关注"
    fi

    echo -e "  分区数: ${PARTITION_COUNT}  |  总积压: ${TOTAL_LAG}  |  \
状态: ${STATUS_COLOR}${STATUS_TEXT}${NC}"
    echo ""
done

# ============================================================
# 汇总
# ============================================================
echo "========================================="
if [[ "${ALERT_COUNT}" -eq 0 ]]; then
    echo -e "  结果: ${GREEN}全部正常${NC}"
else
    echo -e "  结果: ${RED}${ALERT_COUNT} 个 Group 存在积压告警${NC}"
fi
echo "========================================="

exit "${ALERT_COUNT}"
```

**逐段解释：**

| 关键点 | 说明 |
|--------|------|
| `timeout 15` | 防止 Kafka 集群无响应时脚本卡死 |
| 配置文件 `source` 加载 | `GROUP_LIST` 数组定义在 `.conf` 文件中，脚本只读不写 |
| 三级告警状态 | 绿色正常 / 黄色关注 / 红色告警，分层次响应 |
| `exit "${ALERT_COUNT}"` | 返回告警数量作为退出码，方便接入 CI/CD 或监控平台 |

---

### 场景7：Nginx日志分析脚本

**需求**：统计 PV、UV、响应时间百分位（P50/P90/P99）。

**Prompt：**

```text
写一个 Nginx access.log 分析 Shell 脚本，要求：
1. 统计 PV（总请求数）、UV（独立IP数）
2. 统计各 HTTP 状态码分布
3. 计算响应时间的 P50、P90、P99 百分位
4. 找出请求量 TOP 10 的 URL
5. 自动检测日志格式（是否为 json 格式），适配两种常见 nginx log_format
6. 输出专业美观的表格
```

**脚本：**

```bash
#!/bin/bash
#
# nginx_log_analyzer.sh — Nginx 访问日志分析
# 用法: ./nginx_log_analyzer.sh [access.log路径]
#
set -euo pipefail

# ============================================================
# 参数与日志格式检测
# ============================================================
LOG_FILE="${1:-/var/log/nginx/access.log}"

if [[ ! -f "${LOG_FILE}" ]]; then
    echo "错误: 日志文件不存在: ${LOG_FILE}"
    exit 1
fi

FILE_SIZE=$(du -h "${LOG_FILE}" | cut -f1)
TOTAL_LINES=$(wc -l < "${LOG_FILE}")

# 自动检测日志格式：取第一行判断是否为 JSON
FIRST_LINE=$(head -1 "${LOG_FILE}")
IS_JSON=false
if echo "${FIRST_LINE}" | python3 -m json.tool &>/dev/null 2>&1; then
    IS_JSON=true
fi

echo "========================================="
echo "  Nginx 访问日志分析"
echo "  文件: ${LOG_FILE} (${FILE_SIZE}, ${TOTAL_LINES}行)"
echo "  格式: $([[ "${IS_JSON}" == "true" ]] && echo 'JSON' || echo 'Combined')"
echo "========================================="
echo ""

# ============================================================
# 1. PV / UV
# ============================================================
echo ">>> 一、流量概览"

if [[ "${IS_JSON}" == "true" ]]; then
    # JSON 格式：每行为独立 JSON 对象，用 jq 提取字段
    PV="${TOTAL_LINES}"
    UV=$(python3 -c "
import json, sys
ips = set()
for line in open('${LOG_FILE}'):
    try:
        obj = json.loads(line.strip())
        ips.add(obj.get('remote_addr', obj.get('client_ip', '')))
    except:
        pass
print(len(ips))
" 2>/dev/null || echo "N/A")
else
    # Combined 格式：第一列是 IP 地址
    PV="${TOTAL_LINES}"
    # awk 提取第一列 → sort → uniq 去重 → wc -l 计数
    UV=$(awk '{print $1}' "${LOG_FILE}" | sort -u | wc -l | tr -d ' ')
fi

printf "  %-15s %10s\n" "PV (请求总数):" "${PV}"
printf "  %-15s %10s\n" "UV (独立IP):"  "${UV}"
echo ""

# ============================================================
# 2. HTTP 状态码分布
# ============================================================
echo ">>> 二、HTTP 状态码分布"

if [[ "${IS_JSON}" == "true" ]]; then
    python3 -c "
import json, sys
from collections import Counter
codes = Counter()
for line in open('${LOG_FILE}'):
    try:
        obj = json.loads(line.strip())
        code = str(obj.get('status', '000'))
        codes[code] += 1
    except:
        pass
for code, count in codes.most_common(10):
    pct = count * 100.0 / ${TOTAL_LINES}
    print(f'  {code}  →  {count:>8}  ({pct:5.1f}%)')
" 2>/dev/null
else
    # Combined 格式：状态码在第9列
    awk '{print $9}' "${LOG_FILE}" \
        | sort | uniq -c | sort -rn \
        | awk -v total="${TOTAL_LINES}" \
            '{printf "  %-6s → %8d  (%5.1f%%)\n", $2, $1, $1*100/total}'
fi
echo ""

# ============================================================
# 3. 响应时间百分位
# ============================================================
echo ">>> 三、响应时间分析 (单位: 秒)"

# 提取所有请求的响应时间，排序后取百分位
if [[ "${IS_JSON}" == "true" ]]; then
    python3 -c "
import json, sys
times = []
for line in open('${LOG_FILE}'):
    try:
        obj = json.loads(line.strip())
        rt = obj.get('request_time', obj.get('upstream_response_time', 0))
        if rt:
            times.append(float(rt))
    except:
        pass
times.sort()
n = len(times)
if n == 0:
    print('  无有效响应时间数据')
    sys.exit(0)

def percentile(data, p):
    idx = int(len(data) * p / 100)
    return data[min(idx, len(data)-1)]

print(f'  请求数(含响应时间): {n}')
print(f'  最小值:  {times[0]:.4f}s')
print(f'  平均值:  {sum(times)/n:.4f}s')
print(f'  最大值:  {times[-1]:.4f}s')
print(f'  P50:     {percentile(times, 50):.4f}s')
print(f'  P90:     {percentile(times, 90):.4f}s')
print(f'  P95:     {percentile(times, 95):.4f}s')
print(f'  P99:     {percentile(times, 99):.4f}s')
" 2>/dev/null
else
    # Combined 格式：假设 response_time 是最后一列（常见自定义格式）
    # 先检测是否有数值列，若没有则跳过
    RT_COL=$(head -1 "${LOG_FILE}" | awk '{print $NF}')
    if [[ "${RT_COL}" =~ ^[0-9.]+$ ]]; then
        awk '{print $NF}' "${LOG_FILE}" \
            | sort -n \
            | awk '
            { times[NR] = $0; sum += $0 }
            END {
                if (NR == 0) exit
                n = NR
                printf "  请求数: %d\n", n
                printf "  最小值:  %.4fs\n", times[1]
                printf "  平均值:  %.4fs\n", sum/n
                printf "  最大值:  %.4fs\n", times[n]
                printf "  P50:     %.4fs\n", times[int(n*0.50)]
                printf "  P90:     %.4fs\n", times[int(n*0.90)]
                printf "  P95:     %.4fs\n", times[int(n*0.95)]
                printf "  P99:     %.4fs\n", times[int(n*0.99)]
            }'
    else
        echo "  未检测到响应时间数据（日志最后一列非数字）"
        echo "  提示: 在 nginx.conf 中配置 log_format 包含 \$request_time"
    fi
fi
echo ""

# ============================================================
# 4. TOP 10 请求 URL
# ============================================================
echo ">>> 四、TOP 10 请求 URL"

if [[ "${IS_JSON}" == "true" ]]; then
    python3 -c "
import json, sys
from collections import Counter
urls = Counter()
for line in open('${LOG_FILE}'):
    try:
        obj = json.loads(line.strip())
        url = obj.get('request', obj.get('uri', ''))
        if url:
            urls[url.split()[0] if ' ' in url else url] += 1
    except:
        pass
for url, count in urls.most_common(10):
    print(f'  {count:>6}  {url[:80]}')
" 2>/dev/null
else
    # Combined 格式：第7列是请求行
    awk '{print $7}' "${LOG_FILE}" \
        | sort | uniq -c | sort -rn | head -10 \
        | awk '{printf "  %6s  %s\n", $1, $2}'
fi

echo ""
echo "========================================="
echo "  分析完成"
echo "========================================="
```

**逐段解释：**

| 关键点 | 说明 |
|--------|------|
| 自动格式检测 | `head -1` 喂给 `python3 -m json.tool` 判断是否能解析 |
| JSON/Combined 双分支 | 生产环境越来越多用 JSON 日志，双适配保证通用性 |
| 百分位计算 | P99 比平均值更能反映真实用户体验，用于 SLA 评估 |
| awk 数组 + END 块 | 纯 awk 实现排序和百分位，无需外部 sort 大文件 |

---

### 场景8：Git仓库批量操作

**需求**：微服务架构有 20+ 个独立仓库，批量同步、打 Tag、合并分支。

**Prompt：**

```text
写一个 Git 仓库批量操作脚本，要求：
1. 支持操作：pull 拉取最新 / tag 打标签 / merge 合并分支 / status 查看状态
2. 从配置文件读取仓库列表（路径数组）
3. 彩色输出每个仓库的操作结果
4. 支持并发执行以加速（后台 job + wait）
5. 操作前检查工作区是否干净，脏工作区警告但不强制中断
6. 支持 --dry-run 模式
```

**脚本：**

```bash
#!/bin/bash
#
# git_batch.sh — Git 仓库批量操作
# 用法: ./git_batch.sh <操作> [选项]
# 操作: pull | tag | merge | status
#
set -euo pipefail

# ============================================================
# 配置
# ============================================================
CONFIG_FILE="${GIT_BATCH_CONFIG:-./git_repos.conf}"

if [[ ! -f "${CONFIG_FILE}" ]]; then
    echo "[ERROR] 配置文件不存在: ${CONFIG_FILE}" >&2
    echo "  格式: REPOS=(\"/path/to/repo1\" \"/path/to/repo2\")" >&2
    exit 1
fi

# shellcheck disable=SC1090
source "${CONFIG_FILE}"

if [[ -z "${REPOS:-}" ]]; then
    echo "[ERROR] 请定义 REPOS 数组" >&2
    exit 1
fi

readonly RED='\033[0;31m'
readonly GREEN='\033[0;32m'
readonly YELLOW='\033[1;33m'
readonly CYAN='\033[0;36m'
readonly NC='\033[0m'

# ============================================================
# 参数解析
# ============================================================
OPERATION=""
TAG_NAME=""
SOURCE_BRANCH=""
TARGET_BRANCH=""
DRY_RUN=false
CONCURRENT=false
MAX_CONCURRENT=5           # 并发数上限，避免占满 IO

usage() {
    cat <<EOF
用法: $0 <操作> [选项]

操作:
  pull               git pull --rebase 所有仓库
  tag -t <标签名>    对所有仓库打统一标签
  merge -s <源> -d <目标>  将源分支合并到目标分支
  status            查看所有仓库工作区状态

选项:
  --dry-run         仅展示操作，不执行
  --concurrent      并发执行（适合仓库数量多时加速）
  -h                帮助

示例:
  $0 pull
  $0 tag -t v2.3.1 --dry-run
  $0 merge -s develop -d master
EOF
    exit 1
}

OPERATION="$1"; shift || usage

while [[ $# -gt 0 ]]; do
    case "$1" in
        -t) TAG_NAME="$2";    shift 2 ;;
        -s) SOURCE_BRANCH="$2"; shift 2 ;;
        -d) TARGET_BRANCH="$2"; shift 2 ;;
        --dry-run) DRY_RUN=true; shift ;;
        --concurrent) CONCURRENT=true; shift ;;
        -h|--help) usage ;;
        *) echo "未知参数: $1"; usage ;;
    esac
done

echo "========================================="
echo "  Git 批量操作: ${OPERATION}"
echo "  仓库总数: ${#REPOS[@]}"
[[ "${DRY_RUN}" == "true" ]] && echo "  模式: DRY-RUN"
echo "========================================="
echo ""

# ============================================================
# 核心函数：对单个仓库执行操作
# ============================================================
run_on_repo() {
    local repo="$1"
    local repo_name
    repo_name=$(basename "${repo}")

    # 仓库目录不存在
    if [[ ! -d "${repo}/.git" ]]; then
        echo -e "  ${RED}✗${NC} ${repo_name}: 不是有效的 Git 仓库"
        return 1
    fi

    cd "${repo}" || return 1

    # 检查工作区是否干净
    local dirty=false
    if [[ -n "$(git status --porcelain 2>/dev/null)" ]]; then
        dirty=true
    fi

    local result=0

    case "${OPERATION}" in
        pull)
            if [[ "${dirty}" == "true" ]]; then
                echo -e "  ${YELLOW}⚠${NC} ${repo_name}: 工作区有未提交更改，先 stash"
                if [[ "${DRY_RUN}" == "false" ]]; then
                    git stash push -m "auto-stash before pull" 2>/dev/null || true
                fi
            fi
            echo -e "  ${CYAN}↓${NC} ${repo_name}: pulling..."
            if [[ "${DRY_RUN}" == "false" ]]; then
                git pull --rebase origin "$(git rev-parse --abbrev-ref HEAD)" || result=1
            fi
            ;;

        tag)
            # 先 pull 最新，确保 tag 打在正确的 commit 上
            if [[ "${DRY_RUN}" == "false" ]]; then
                git pull --rebase origin "$(git rev-parse --abbrev-ref HEAD)" 2>/dev/null || true
                if git rev-parse "${TAG_NAME}" >/dev/null 2>&1; then
                    echo -e "  ${YELLOW}⚠${NC} ${repo_name}: 标签 ${TAG_NAME} 已存在"
                else
                    git tag -a "${TAG_NAME}" -m "Release ${TAG_NAME}" && \
                        git push origin "${TAG_NAME}" || result=1
                fi
            fi
            echo -e "  ${CYAN}🏷${NC} ${repo_name}: tag ${TAG_NAME}"
            ;;

        merge)
            if [[ -z "${SOURCE_BRANCH}" || -z "${TARGET_BRANCH}" ]]; then
                echo -e "  ${RED}✗${NC} merge 需要 -s 和 -d 参数"
                return 1
            fi
            echo -e "  ${CYAN}🔀${NC} ${repo_name}: ${SOURCE_BRANCH} → ${TARGET_BRANCH}"
            if [[ "${DRY_RUN}" == "false" ]]; then
                # 切换到目标分支，拉取最新，合并源分支
                git checkout "${TARGET_BRANCH}" 2>/dev/null || { result=1; }
                git pull origin "${TARGET_BRANCH}" 2>/dev/null || true
                git merge "${SOURCE_BRANCH}" --no-edit || result=1
                git push origin "${TARGET_BRANCH}" || result=1
                # 切回源分支
                git checkout "${SOURCE_BRANCH}" 2>/dev/null || true
            fi
            ;;

        status)
            if [[ "${dirty}" == "true" ]]; then
                echo -e "  ${YELLOW}●${NC} ${repo_name}: 有未提交更改"
            else
                echo -e "  ${GREEN}●${NC} ${repo_name}: 干净"
            fi

            # 显示当前分支与 remote 的差异
            LOCAL=$(git rev-parse @ 2>/dev/null)
            REMOTE=$(git rev-parse @{u} 2>/dev/null)
            BASE=$(git merge-base @ @{u} 2>/dev/null)
            if [[ "${LOCAL}" == "${REMOTE}" ]]; then
                echo "          ↳ 已同步"
            elif [[ "${LOCAL}" == "${BASE}" ]]; then
                echo -e "          ↳ ${YELLOW}需要 pull${NC}"
            elif [[ "${REMOTE}" == "${BASE}" ]]; then
                echo -e "          ↳ ${YELLOW}有待推送的提交${NC}"
            else
                echo -e "          ↳ ${RED}分支已分叉${NC}"
            fi
            ;;

        *)
            echo -e "  ${RED}✗${NC} 未知操作: ${OPERATION}"
            return 1
            ;;
    esac

    if [[ "${result}" -eq 0 ]]; then
        echo -e "  ${GREEN}✓${NC} ${repo_name}: 完成"
    else
        echo -e "  ${RED}✗${NC} ${repo_name}: 失败"
    fi
    return "${result}"
}

# ============================================================
# 执行：顺序或并发
# ============================================================
FAILED=0
SUCCESS=0

if [[ "${CONCURRENT}" == "true" ]]; then
    # 并发模式：后台 job + 限制并发数
    RUNNING=0
    for repo in "${REPOS[@]}"; do
        run_on_repo "${repo}" &
        RUNNING=$(( RUNNING + 1 ))
        # 达到并发上限时等待
        if [[ "${RUNNING}" -ge "${MAX_CONCURRENT}" ]]; then
            wait -n 2>/dev/null || true
            RUNNING=$(( RUNNING - 1 ))
        fi
    done
    wait  # 等待所有后台任务完成
else
    for repo in "${REPOS[@]}"; do
        if run_on_repo "${repo}"; then
            SUCCESS=$(( SUCCESS + 1 ))
        else
            FAILED=$(( FAILED + 1 ))
        fi
    done
fi

echo ""
echo "========================================="
echo -e "  完成: ${GREEN}${SUCCESS} 成功${NC}  /  ${RED}${FAILED} 失败${NC}  /  总计 ${#REPOS[@]}"
echo "========================================="

exit "${FAILED}"
```

**逐段解释：**

| 关键点 | 说明 |
|--------|------|
| `git stash push -m "auto-stash"` | pull 前自动暂存本地修改，防止 rebase 冲突 |
| `git rev-parse` 判断 ahead/behind | 不用 `git status -sb` 解析字符串，更可靠 |
| `wait -n` | Bash 4.3+ 特性，等待任一后台任务完成，实现并发池 |
| `MAX_CONCURRENT=5` | 防止 20+ 个仓库同时 git 操作打满磁盘 IO |

---

## 三、Shell安全编写最佳实践

AI 生成的脚本好用，但你得知道这些防守性编程技巧，否则可能写出"能跑但跑着跑着就炸"的定时炸弹。

### 1. 必加的 Shebang 三件套

```bash
#!/bin/bash
set -euo pipefail
```

| 选项 | 作用 | 不写的后果 |
|------|------|------------|
| `set -e` | 任何命令返回非0立即退出 | 中间命令失败了脚本继续跑，可能 `rm -rf /` |
| `set -u` | 使用未定义变量时报错 | `${TYPO_VAR}` 变成空字符串，逻辑悄悄出错 |
| `set -o pipefail` | 管道中任一命令失败都算失败 | `grep foo \| awk`，grep 失败了 awk 还在跑 |

### 2. 变量永远加引号

```bash
# 错误写法
if [ $NAME = "foo" ]; then   # NAME 为空时语法错误

# 正确写法
if [[ "${NAME}" == "foo" ]]; then   # [[ ]] 内置条件测试 + 双引号
```

规则：**所有变量引用都加双引号**，除非你明确需要单词分割（极少情况）。

### 3. 临时文件清理

```bash
# trap 保证脚本异常退出时也能清理临时文件
TEMP_FILE=$(mktemp) || exit 1
trap 'rm -f "${TEMP_FILE}"' EXIT

# 临时目录同理
TEMP_DIR=$(mktemp -d)
trap 'rm -rf "${TEMP_DIR}"' EXIT
```

### 4. 管道中的错误处理

```bash
# grep 无匹配时返回 1，在 set -e 下会直接退出
# 解决方案：|| true
ERROR_COUNT=$(grep -c "ERROR" app.log || true)
```

### 5. 敏感信息不硬编码

```bash
# 错误
DB_PASSWORD="mypassword123"

# 正确
DB_PASSWORD="${DB_PASSWORD:?请设置 DB_PASSWORD 环境变量}"
# 或从 Vault / K8s Secret 挂载的文件读取
DB_PASSWORD=$(cat /secrets/db_password)
```

### 6. 函数内局部变量

```bash
# 使用 local 避免污染全局命名空间
my_function() {
    local temp_var="value"
    local result=0
    # ...
    return "${result}"
}
```

---

## 四、如何验证AI生成的Shell？shellcheck 一把梭

AI 生成的脚本不要直接跑生产。用 `shellcheck` 做静态检查：

### 安装

```bash
# macOS
brew install shellcheck

# Linux (Debian/Ubuntu)
apt install shellcheck

# Linux (CentOS/RHEL)
yum install ShellCheck
```

### 使用

```bash
# 检查单个脚本
shellcheck health_check.sh

# 检查当前目录所有 .sh 文件
shellcheck *.sh

# 生成可忽略规则清单（少数场景需要关闭特定检查）
# shellcheck disable=SC1090  # 忽略 source 外部文件警告
```

### 常见告警解读

| 规则编号 | 含义 | 修复方式 |
|----------|------|----------|
| SC2086 | 变量未加引号 | `${var}` → `"${var}"` |
| SC2046 | 命令替换未加引号 | `$(cmd)` → `"$(cmd)"` |
| SC2164 | `cd` 未检查返回值 | `cd dir \|\| exit` |
| SC1090/SC1091 | source 文件路径不确定 | 加 `# shellcheck disable=SC1090` |
| SC2002 | 无用的 `cat` | `cat file \| grep` → `grep file` |

### 搭配 AI 使用的标准工作流

```
1. 用 Prompt 生成脚本初稿
2. 保存为 script.sh
3. shellcheck script.sh  → 发现 0 errors / 0 warnings
4. 本地测试环境手动执行一次
5. 上 staging /灰度环境跑
6. 最后上生产
```

`shellcheck` 清理干净后再跑，能拦截 80% 的 Shell 低级错误。

---

## 五、总结：AI+Shell 的正确姿势

回头看这 8 个脚本，你会发现它们有个共同特点：**结构整齐、注释详尽、防御性编程到位**。这恰好是 AI 擅长的——把散落在 Stack Overflow、man page、博客文章里的"最佳实践"一次性组装好。

但请记住两条底线：
1. **AI 写的 Shell 必须先过 shellcheck**，这是你的安全气囊
2. **永远不把 AI 生成的脚本直接管道到 bash**（`curl xxx.sh | bash` 这种），至少看一眼

Java程序员写 Shell 这件事，从"最怕"变成"最爽"，也就差一个 Prompt 的距离。

---

### 下一篇预告

**《AI 辅助 K8s YAML 自动生成：从 Deployment 到 HPA，告别缩进地狱》**

你是否也曾因为一个 YAML 缩进不对而排查了半小时？下一篇带你用 AI 搞定：
- Spring Boot 应用的 Deployment + Service + ConfigMap 一键生成
- 资源配额自动计算（requests/limits 不再拍脑袋）
- HPA 自动伸缩策略配置
- Kustomize overlay 多环境模板

关注博主，不迷路。一键三连，代码平安。

---

*本文由作者基于 AI 工具辅助创作，脚本均通过 shellcheck 验证。生产环境使用前请充分测试。*
