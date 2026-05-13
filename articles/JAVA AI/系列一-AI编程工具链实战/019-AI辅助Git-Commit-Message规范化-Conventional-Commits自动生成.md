# AI 辅助 Git Commit Message 规范化：Conventional Commits 自动生成，再也不用写"fix bug"了

## 一、开篇：每个程序员都经历过的"社死现场"

想象一下，你加入一个新团队，打开 Git 历史，打算了解一下项目的演进脉络。结果映入眼帘的是：

```
张三: update
李四: fix
王五: 改了点东西
赵六: 测试
钱七: bug修复
孙八: feat: 这是历史正文第3行，前面有6个空格换行第二行第三行反正随便写写反正也不会有人看
```

你深吸一口气，默默关掉终端，打开飞书开始投简历。

这不是段子。这是无数团队的**真实日常**。

更可怕的是，三个月后的你，面对自己写的 `git log`，陷入了深深的自我怀疑——"这 `fix bug` 到底修的是哪个 bug？下订单崩了还是支付回调挂了？"

**Commit Message 写不好，事故复盘火葬场。**

而今天，AI 可以帮你彻底解决这个问题。从一个 `git diff` 开始，到一封符合 Conventional Commits 规范的 Commit Message，全程零手工。

---

## 二、先搞懂规范：Conventional Commits 速查表

在让 AI 干活之前，我们先明确"标准答案"长什么样。

### 2.1 基本格式

```
<type>[optional scope]: <description>

[optional body]

[optional footer]
```

### 2.2 type 速查表

| Type | 说明 | 示例 |
|------|------|------|
| `feat` | 新功能 | `feat: 用户登录支持短信验证码` |
| `fix` | 修 Bug | `fix: 修复订单金额计算精度丢失` |
| `docs` | 文档变更 | `docs: 更新支付接口文档` |
| `style` | 代码格式（不影响逻辑） | `style: 统一缩进为 4 空格` |
| `refactor` | 重构（不修 Bug，不加功能） | `refactor: 抽取公共校验逻辑` |
| `perf` | 性能优化 | `perf: 用户列表查询添加联合索引` |
| `test` | 测试相关 | `test: 新增订单超时取消的单元测试` |
| `build` | 构建系统或外部依赖 | `build: 升级 Spring Boot 到 3.2.0` |
| `ci` | CI/CD 配置 | `ci: 添加代码覆盖率检查步骤` |
| `chore` | 杂项（不修改 src/test） | `chore: 更新 .gitignore 排除 logs 目录` |
| `revert` | 回滚 | `revert: 回滚 feat: 新增自动拆单逻辑` |

### 2.3 scope（作用域）建议

对于 Java 项目，常见的 scope 示例：

```
feat(order): 新增批量下单接口
fix(payment): 修复微信支付回调验签失败
refactor(user): 用户模块 DTO 拆分
perf(gateway): 网关层添加 Caffeine 本地缓存
```

### 2.4 破坏性变更标记

当变更不向后兼容时，在 type/scope 后加 `!`，或者在 footer 加 `BREAKING CHANGE`：

```
feat(api)!: 重构用户接口，v1 接口全部废弃
```

或者：

```
feat(api): 重构用户接口

BREAKING CHANGE: v1 接口已全部移除，请迁移至 v2
```

### 2.5 关联 Issue

```
fix(order): 修复重复下单问题

Closes #1234
Refs #5678
```

---

## 三、终极目标：从三段进化到全自动

我们的目标很明确：**让每一次 `git commit` 都自动生成符合规范的 Commit Message**。

分三步走，从"手动"到"半自动"再到"全自动"：

```
Level 1  →  把 git diff 贴给 AI，手动复制粘贴 Commit Message
Level 2  →  Git Hook (prepare-commit-msg) + AI API，git commit 时自动生成
Level 3  →  GitHub Actions 工作流，PR Merge 时强制校验 + 自动修正
```

下面我们逐级打怪升级。

---

## 四、Level 1：手动用 AI 工具生成 Commit Message

### 4.1 核心思路

把当前暂存区的变更（`git diff --cached`）丢给 AI，让它帮你生成规范的 Commit Message。

### 4.2 操作流程

```bash
# 第一步：暂存你的改动
git add .

# 第二步：获取 diff 内容
git diff --cached > /tmp/diff.txt

# 第三步：把 diff.txt 贴给 ChatGPT / Copilot Chat / Cursor
```

### 4.3 Prompt 设计

这个 Prompt 是整个方案的核心。好的 Prompt 能让 AI 输出精准的 Commit Message：

```
你是一个 Conventional Commits 专家。请根据以下 git diff 生成符合规范的 Commit Message。

要求：
1. 严格遵循 Conventional Commits 规范：<type>[optional scope]: <description>
2. type 必须是以下之一：feat, fix, docs, style, refactor, perf, test, build, ci, chore, revert
3. description 用英文，简洁准确，不超过 72 字符，首字母小写，结尾不加句号
4. 如果有破坏性变更（API 不兼容、参数变更等），type 后面加 !
5. 如果需要写 body（变更复杂时），用英文说明原因和影响
6. 只输出 Commit Message 本身，不要输出任何解释性文字

git diff:
---
[在这里粘贴 git diff 内容]
---
```

### 4.4 进阶：中文描述转英文 Commit Message

如果你的团队习惯用中文描述变更，可以多加一句：

```
上面所有的要求不变，但 description 部分先用中文描述，再翻译为英文：

中文描述：用户登录模块支持短信验证码
英文 Commit Message：feat(login): support SMS verification code login
```

这样 AI 会同时理解你的中文意图，并输出英文的规范格式。

### 4.5 Level 1 的局限性

- 每次都要手动复制粘贴，效率不高
- 需要切出终端，打断编码心流
- 多人协作时，无法强制所有人使用

所以我们需要升级到 Level 2。

---

## 五、Level 2：Git Hook + AI API，git commit 自动生成

### 5.1 核心思路

利用 Git 的 `prepare-commit-msg` Hook，在每次执行 `git commit` 时，自动调用 AI API（如 OpenAI），根据 `git diff --cached` 生成 Commit Message，预填充到编辑器中。

### 5.2 Git Hook 触发时机全景图

```bash
# Git 内置的 4 种 Commit Hook
pre-commit          → 提交前（常用于 lint、format 检查）
prepare-commit-msg  → 准备提交信息前（我们的主战场）
commit-msg          → 提交信息写好后（commitlint 在这里拦截）
post-commit         → 提交完成后（通知、打日志）
```

我们选择 `prepare-commit-msg`，因为此时文件已暂存，但 Commit Message 还没写——正好让 AI 提前帮你写好草稿。

### 5.3 完整 Shell 脚本

在项目根目录创建 `.githooks/prepare-commit-msg`：

```bash
#!/bin/bash
# ============================================================
# AI 自动生成 Conventional Commits Message
# 调用 OpenAI API，根据 git diff --cached 生成规范提交信息
# ============================================================

# ---------- 配置区 ----------
# 你的 OpenAI API Key（建议用环境变量，不要硬编码）
OPENAI_API_KEY="${OPENAI_API_KEY:-}"
# OpenAI API 地址（国内可用代理）
OPENAI_API_BASE="${OPENAI_API_BASE:-https://api.openai.com/v1}"
# 使用的模型
MODEL="${AI_COMMIT_MODEL:-gpt-3.5-turbo}"
# 超时时间（秒）
TIMEOUT="${AI_COMMIT_TIMEOUT:-15}"
# 最大 Token 数
MAX_TOKENS="150"

# 你希望保留的现有 message（如 merge、rebase 时 git 自动生成的）
COMMIT_MSG_FILE="$1"
COMMIT_SOURCE="$2"

# ---------- 跳过场景 ----------
# 非新建 commit 时跳过（如 merge、rebase、squash、amend）
if [ -n "$COMMIT_SOURCE" ]; then
    echo "[AI Commit] 跳过：非手动新建提交（source=$COMMIT_SOURCE）"
    exit 0
fi

# 如果没有配置 API Key，跳过
if [ -z "$OPENAI_API_KEY" ]; then
    echo "[AI Commit] 跳过：未设置 OPENAI_API_KEY 环境变量"
    exit 0
fi

# ---------- 获取暂存区变更 ----------
DIFF_CONTENT=$(git diff --cached)

# 没有暂存内容时跳过
if [ -z "$DIFF_CONTENT" ]; then
    echo "[AI Commit] 跳过：暂存区为空"
    exit 0
fi

# 限制 diff 大小（避免 token 浪费，超大 diff 建议拆分提交）
DIFF_CHAR_COUNT=$(echo "$DIFF_CONTENT" | wc -c)
MAX_DIFF_CHARS=8000

if [ "$DIFF_CHAR_COUNT" -gt "$MAX_DIFF_CHARS" ]; then
    echo "[AI Commit] 警告：diff 过大（${DIFF_CHAR_COUNT} 字符），将截取前 ${MAX_DIFF_CHARS} 字符"
    DIFF_CONTENT=$(echo "$DIFF_CONTENT" | head -c "$MAX_DIFF_CHARS")
fi

# 转义 diff 内容为 JSON 安全字符串
DIFF_ESCAPED=$(echo "$DIFF_CONTENT" | python3 -c "
import sys, json
print(json.dumps(sys.stdin.read()))
")

# ---------- 构建 AI Prompt ----------
SYSTEM_PROMPT="You are a Conventional Commits expert. Generate a commit message strictly following the Conventional Commits specification: <type>[optional scope]: <description>.
Rules:
1. type must be one of: feat, fix, docs, style, refactor, perf, test, build, ci, chore, revert
2. description in English, lowercase first letter, no period at the end, max 72 chars
3. Add scope in parentheses after type if the change scope is clear (e.g. feat(auth): , fix(order): )
4. If there are breaking changes, add ! after type/scope
5. For complex changes, include a body explaining what and why
6. Only output the commit message, no explanations, no markdown formatting
7. Keep it concise and accurate based on the actual code changes"

USER_PROMPT="Generate a conventional commit message for the following git diff:\n\n$DIFF_ESCAPED"

# ---------- 构建 JSON 请求体 ----------
REQUEST_BODY=$(cat <<EOF
{
  "model": "$MODEL",
  "messages": [
    {"role": "system", "content": $(echo "$SYSTEM_PROMPT" | python3 -c "import sys, json; print(json.dumps(sys.stdin.read()))")},
    {"role": "user", "content": $DIFF_ESCAPED}
  ],
  "max_tokens": $MAX_TOKENS,
  "temperature": 0.3
}
EOF
)

# ---------- 调用 OpenAI API ----------
echo "[AI Commit] 正在生成 Commit Message..."
echo "[AI Commit] 模型：${MODEL} | Diff 大小：${DIFF_CHAR_COUNT} 字符"

API_RESPONSE=$(curl -s \
    --max-time "$TIMEOUT" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $OPENAI_API_KEY" \
    -d "$REQUEST_BODY" \
    "$OPENAI_API_BASE/chat/completions" 2>&1)

CURL_EXIT_CODE=$?

if [ $CURL_EXIT_CODE -ne 0 ]; then
    echo "[AI Commit] 错误：API 请求失败（exit code: $CURL_EXIT_CODE）"
    echo "[AI Commit] 将使用默认空 Commit Message，请手动输入"
    exit 0
fi

# ---------- 解析 API 响应 ----------
COMMIT_MSG=$(echo "$API_RESPONSE" | python3 -c "
import sys, json
try:
    resp = json.load(sys.stdin)
    if 'choices' in resp and len(resp['choices']) > 0:
        msg = resp['choices'][0]['message']['content']
        print(msg.strip())
    elif 'error' in resp:
        print('__ERROR__:' + resp['error'].get('message', 'Unknown error'))
    else:
        print('__ERROR__:Unexpected response format')
except Exception as e:
    print(f'__ERROR__:{e}')
")

if [[ "$COMMIT_MSG" == __ERROR__:* ]]; then
    echo "[AI Commit] 错误：${COMMIT_MSG#__ERROR__:}"
    echo "[AI Commit] 将使用默认空 Commit Message，请手动输入"
    exit 0
fi

# ---------- 写入 Commit Message ----------
# 将 AI 生成的内容写入 commit message 文件
# 保留原有的注释行（git 会自动追加）
ORIGINAL_MSG=$(cat "$COMMIT_MSG_FILE" 2>/dev/null)

if [ -n "$ORIGINAL_MSG" ]; then
    # 如果已有内容（如模板），将 AI 生成的内容加到前面
    echo "$COMMIT_MSG" > "$COMMIT_MSG_FILE"
    echo "" >> "$COMMIT_MSG_FILE"
    echo "$ORIGINAL_MSG" >> "$COMMIT_MSG_FILE"
else
    echo "$COMMIT_MSG" > "$COMMIT_MSG_FILE"
fi

echo "[AI Commit] 完成！已生成 Commit Message："
echo "----------------------------------------"
echo "$COMMIT_MSG"
echo "----------------------------------------"
echo "[AI Commit] 提示：请检查后保存退出，或直接 :wq 确认"
```

### 5.4 安装 Git Hook

```bash
# 创建 hooks 目录
mkdir -p .githooks

# 复制脚本
cp prepare-commit-msg .githooks/prepare-commit-msg

# 赋予执行权限
chmod +x .githooks/prepare-commit-msg

# 设置 Git 读取自定义 hooks 路径
git config core.hooksPath .githooks
```

**推荐**：把 hooks 路径配置写入项目根目录的 `.gitconfig` 或通过 `Makefile`/`gradle` 任务自动设置，确保团队成员 clone 项目后自动生效。

### 5.5 团队共享 Hook（可选）

在项目中创建一个安装脚本 `scripts/install-git-hooks.sh`：

```bash
#!/bin/bash
# 团队成员 clone 项目后执行此脚本，一键安装 AI Commit Hook

PROJECT_ROOT=$(git rev-parse --show-toplevel)

if [ ! -d "$PROJECT_ROOT/.githooks" ]; then
    echo ".githooks 目录不存在，请确认脚本在项目根目录下执行"
    exit 1
fi

git config core.hooksPath .githooks

echo "✅ Git Hooks 已安装到 $(git config core.hooksPath)"

# 检查 API Key 是否配置
if [ -z "$OPENAI_API_KEY" ]; then
    echo ""
    echo "⚠️  未检测到 OPENAI_API_KEY 环境变量"
    echo "请设置环境变量以启用 AI 自动生成 Commit Message："
    echo "  # zsh/bash"
    echo '  echo "export OPENAI_API_KEY=sk-xxx" >> ~/.zshrc'
    echo '  source ~/.zshrc'
    echo ""
    echo "  # 或者使用 .env 文件（需自行实现加载逻辑）"
fi

echo ""
echo "🎉 安装完成！下次 git commit 将自动调用 AI 生成 Commit Message"
```

### 5.6 Level 2 的优势与不足

**优势：**
- 在编辑器里看到的是 AI 预填充的 Commit Message，可以直接修改确认
- API 调用时间短（1-3 秒），几乎无感知
- 不依赖外部工具，纯 Git 原生能力

**不足：**
- 需要每位开发者在本地配置 `OPENAI_API_KEY`
- 无法强制校验（开发者可以手动改掉 AI 生成的内容）
- 网络波动时可能超时，需做好降级处理

---

## 六、Level 3：GitHub Actions 跑 AI，PR 级别强制校验 + 兜底生成

### 6.1 为什么需要 Level 3

Level 2 解决了"懒得写"的问题，但解决不了"瞎写"的问题。如果有人把 AI 生成的 Commit Message 改成了 `fix bug`，Hook 也拦不住。

Level 3 的目标：

1. **每个 PR 都跑 commitlint** → 检查 Commit Message 是否符合 Conventional Commits 规范
2. **commitlint 没通过？** → AI 自动生成修正后的 Commit Message，贴到 PR 评论区
3. **Merge 前强制校验** → 规范不通过，Merge Button 灰色不可点

### 6.2 完整 GitHub Actions 工作流

创建 `.github/workflows/commitlint-ai.yml`：

```yaml
name: Commit Message Validation & AI Assist

on:
  pull_request:
    types: [opened, synchronize, reopened, edited]

permissions:
  contents: read
  pull-requests: write

jobs:
  # ============================================
  # Job 1：commitlint 规范检查
  # ============================================
  commitlint:
    runs-on: ubuntu-latest
    outputs:
      lint_result: ${{ steps.check.outputs.result }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install commitlint
        run: |
          npm init -y
          npm install --save-dev @commitlint/cli @commitlint/config-conventional

      - name: Create commitlint config
        run: |
          cat > commitlint.config.js << 'EOF'
          module.exports = {
            extends: ['@commitlint/config-conventional'],
            rules: {
              'type-enum': [2, 'always', [
                'feat', 'fix', 'docs', 'style', 'refactor',
                'perf', 'test', 'build', 'ci', 'chore', 'revert'
              ]],
              'subject-case': [2, 'never', ['sentence-case', 'start-case', 'pascal-case', 'upper-case']],
              'subject-full-stop': [2, 'never', '.'],
              'header-max-length': [2, 'always', 72],
              'body-max-line-length': [0, 'always', 100]
            }
          };
          EOF

      - name: Run commitlint on PR commits
        id: check
        run: |
          # 获取 PR 新增的 commit 范围
          git fetch origin "${{ github.event.pull_request.base.ref }}"
          BASE_SHA=$(git merge-base origin/"${{ github.event.pull_request.base.ref }}" HEAD)

          echo "检查 $BASE_SHA..HEAD 范围内的提交..."

          # 运行 commitlint，捕获失败信息
          if npx commitlint --from "$BASE_SHA" --to HEAD --verbose 2>&1 | tee /tmp/commitlint.log; then
            echo "result=pass" >> $GITHUB_OUTPUT
            echo "✅ 所有 Commit Message 符合规范"
          else
            echo "result=fail" >> $GITHUB_OUTPUT
            echo "❌ 存在不符合规范的 Commit Message"
            # 保存日志供下一步使用
            cp /tmp/commitlint.log /tmp/commitlint_failed.log
          fi

      - name: Upload commitlint log
        if: steps.check.outputs.result == 'fail'
        uses: actions/upload-artifact@v4
        with:
          name: commitlint-log
          path: /tmp/commitlint_failed.log

  # ============================================
  # Job 2：AI 分析并给出修正建议
  # ============================================
  ai-suggestion:
    needs: commitlint
    if: needs.commitlint.outputs.lint_result == 'fail'
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Download commitlint log
        uses: actions/download-artifact@v4
        with:
          name: commitlint-log

      - name: Get failed commits info
        id: failed_commits
        run: |
          git fetch origin "${{ github.event.pull_request.base.ref }}"
          BASE_SHA=$(git merge-base origin/"${{ github.event.pull_request.base.ref }}" HEAD)

          FAILED_COMMITS_JSON=$(git log "$BASE_SHA..HEAD" --format='{"hash":"%H","message":"%s","body":"%b"}' | \
            python3 -c "
import sys, json
commits = []
for line in sys.stdin:
    line = line.strip()
    if line:
        commits.append(json.loads(line))
print(json.dumps(commits[:10]))  # 最多处理 10 个 commit
")

          echo "commits=$FAILED_COMMITS_JSON" >> $GITHUB_OUTPUT

      - name: AI Generate Correction Suggestions
        id: ai
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
        run: |
          COMMITS='${{ steps.failed_commits.outputs.commits }}'

          PROMPT=$(cat << 'PROMPT_END'
          You are a Conventional Commits compliance expert. Analyze the following git commits that failed validation.
          For EACH commit that has an invalid message, provide:

          1. The original commit message (short hash + message)
          2. Why it fails the spec
          3. A corrected version that passes Conventional Commits validation
          4. Format your response as a readable markdown block, grouped by commit

          Rules reminder:
          - type must be: feat, fix, docs, style, refactor, perf, test, build, ci, chore, revert
          - Format: <type>[scope]: <description>
          - Description in English, lowercase first, no period, max 72 chars

          Failed commits data:
          PROMPT_END
          )"$COMMITS""

          RESPONSE=$(curl -s https://api.openai.com/v1/chat/completions \
            -H "Content-Type: application/json" \
            -H "Authorization: Bearer $OPENAI_API_KEY" \
            -d "$(python3 -c "
          import json, os
          body = {
              'model': 'gpt-4o-mini',
              'messages': [
                  {
                      'role': 'system',
                      'content': 'You are a Conventional Commits compliance expert. You help developers fix invalid commit messages. Respond in Chinese with English commit message corrections.'
                  },
                  {'role': 'user', 'content': os.environ['PROMPT']}
              ],
              'temperature': 0.3,
              'max_tokens': 2000
          }
          print(json.dumps(body))
          ")")

          echo "$RESPONSE" | python3 -c "
          import sys, json
          resp = json.load(sys.stdin)
          content = resp['choices'][0]['message']['content']
          print(content)
          " > /tmp/ai_suggestion.md

          echo "ai_suggestion_path=/tmp/ai_suggestion.md" >> $GITHUB_OUTPUT

      - name: Post AI suggestion as PR comment
        uses: thollander/actions-comment-pull-request@v2
        with:
          filePath: /tmp/ai_suggestion.md
          comment_tag: commitlint-ai-suggestion
          mode: recreate
          message: |
            ## 🤖 AI Commit Message 规范检查未通过

            你的 PR 中有提交信息不符合 [Conventional Commits](https://www.conventionalcommits.org/) 规范。

            $(cat /tmp/ai_suggestion.md)

            ---
            **操作指南：**

            1. **修改本地 Commit Message**：
               ```bash
               git rebase -i HEAD~N  # N 为需要修改的 commit 数量
               # 将需要修改的 commit 标记为 reword
               # 在弹出的编辑器中修改 Commit Message
               git push --force-with-lease
               ```
            2. **或使用 git-absorb / fixup** 工具合并 fixup commits
            3. **修改后 Push**，此检查将自动重新运行

            > ⚠️ 在 Branch Protection 开启的情况下，规范不通过将无法 Merge。
            > 此建议由 AI 自动生成，仅供参考，请结合代码上下文确认。

      - name: Set check status
        run: |
          echo "::warning::Commit Message 规范检查未通过，请修改后重新 Push"

  # ============================================
  # Job 3：分支保护说明
  # ============================================
  branch-protection-note:
    needs: [commitlint]
    runs-on: ubuntu-latest
    if: needs.commitlint.outputs.lint_result == 'pass'
    steps:
      - name: Mark as passed
        run: |
          echo "✅ 所有 Commit Message 通过 Conventional Commits 规范检查"
```

### 6.3 配合 Branch Protection 使用

在 GitHub 仓库的 **Settings → Branches → Add rule** 中，为 `main` / `master` 分支添加保护规则：

```
Branch name pattern: main
✅ Require status checks to pass before merging
   勾选：commitlint (Commit Message Validation & AI Assist)
✅ Require branches to be up to date before merging
```

配置后效果：如果 PR 中有任何一个 Commit Message 不符合规范，Merge Button 直接变灰，上面写着：

> ⚠️ 1 required status check missing: commitlint

### 6.4 配置文件补充

项目中还需要 `commitlint.config.js`（GitHub Actions 是临时生成的，但本地开发也需要）：

```javascript
module.exports = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'type-enum': [2, 'always', [
      'feat', 'fix', 'docs', 'style', 'refactor',
      'perf', 'test', 'build', 'ci', 'chore', 'revert'
    ]],
    'subject-case': [2, 'never', ['sentence-case', 'start-case', 'pascal-case', 'upper-case']],
    'subject-full-stop': [2, 'never', '.'],
    'header-max-length': [2, 'always', 72],
    'body-max-line-length': [0, 'always', 100]
  }
};
```

以及 `package.json` 中添加 husky + commitlint 的本地 hook：

```json
{
  "devDependencies": {
    "@commitlint/cli": "^19.0.0",
    "@commitlint/config-conventional": "^19.0.0",
    "husky": "^9.0.0"
  },
  "scripts": {
    "prepare": "husky install"
  }
}
```

```bash
# 安装 husky
npx husky install

# 添加 commit-msg hook
npx husky add .husky/commit-msg 'npx --no -- commitlint --edit $1'
```

---

## 七、AI + commitlint 双重保障架构

把 Level 2 和 Level 3 结合起来，形成完整的闭环：

```
开发者 git add + git commit
        │
        ▼
┌──────────────────────────┐
│  prepare-commit-msg Hook │  ← AI 自动生成 Commit Message
│  (OpenAI API)            │     预填充到编辑器中
└──────────┬───────────────┘
           │
           ▼
     开发者确认 / 修改
           │
           ▼
┌──────────────────────────┐
│  commit-msg Hook         │  ← commitlint 本地检查
│  (@commitlint/cli)       │     不合格直接拒绝提交
└──────────┬───────────────┘
           │
           ▼
       git push
           │
           ▼
┌──────────────────────────┐
│  GitHub Actions          │  ← CI 再次检查 + AI 兜底
│  commitlint + AI Review  │     不合格贴修正建议到 PR
└──────────┬───────────────┘
           │
      ✅ 全部通过
           │
           ▼
       PR Ready to Merge
```

这个架构达到的效果是：

- **生成层**（prepare-commit-msg）：AI 帮你写，不用动脑
- **校验层**（commit-msg Hook）：本地就拦住不合格的
- **兜底层**（GitHub Actions）：CI 再验一遍，AI 给修正建议
- **强制层**（Branch Protection）：不规范就不让 Merge

---

## 八、成本与方案选择

### 8.1 API 调用成本估算

以 OpenAI GPT-3.5-turbo 为例：

- 单次 Commit Message 生成消耗约 300-800 Token
- 价格：约 $0.0005/次（GPT-3.5-turbo）或 $0.003/次（GPT-4o-mini）
- 一个 20 人团队，每人每天 10 次 commit：
  - 日成本：约 $0.10 - $0.60
  - 月成本：约 $2 - $12

### 8.2 方案选择建议

| 场景 | 推荐方案 |
|------|----------|
| 个人项目 / 小团队 | Level 1（手动贴 AI）+ commitlint 本地检查 |
| 中型团队，追求效率 | Level 2（Git Hook + AI API）+ commitlint 本地 |
| 大型团队，追求规范 | Level 2 + Level 3 全套（GitHub Actions + Branch Protection） |

### 8.3 替代 API 方案

如果你不方便使用 OpenAI，还可以选择以下方案：

- **Gemini API**（Google）：免费额度较大，适合个人和小团队
- **DeepSeek API**：国产模型，价格极低，中文理解更好
- **Ollama 本地部署**：如果有 GPU 服务器，可以用 Llama 3 / Qwen 2.5 本地跑，零成本
- **GitHub Copilot CLI**：`gh copilot suggest` 也可以生成 Commit Message

---

## 九、总结

回顾一下我们的进化之路：

| 阶段 | 做了什么 | 核心工具 |
|------|----------|----------|
| Level 1 | 手动贴 git diff 给 AI | ChatGPT / Copilot Chat / Cursor |
| Level 2 | git commit 时自动调用 AI API | Git Hook (prepare-commit-msg) |
| Level 3 | CI 强制校验 + AI 修正建议 | GitHub Actions + commitlint |

三个 Level 逐级递进，你可以根据团队的实际情况选择适合自己的层级。最推荐的组合是 **Level 2（本地 Git Hook）+ Level 3 的 commitlint 校验部分**——既能享受 AI 自动生成的便利，又有硬性的规范校验兜底。

从此，团队 Git 历史从这样：

```
张三: update
李四: fix
王五: 改了点东西
```

变成了这样：

```
张三: feat(payment): add wechat pay refund callback handler
李四: fix(order): correct decimal precision in amount calculation
王五: refactor(user): extract validation logic to UserValidator
```

**项目经理看了流泪，Tech Lead 看了欣慰，新人 Onboarding 看了秒懂。**

---

## 十、下期预告

下一篇：《**AI 辅助技术方案编写：从需求描述到完整设计文档，只需 5 个 Prompt**》

我们将探讨：

- 如何用 AI 将一句话需求拆解为技术方案大纲
- 架构图 Mermaid.js 的 AI 自动生成技巧
- 技术评审 Checklist 的 AI 自动生成
- 方案对比表格（优劣分析）的 AI 自动输出

敬请期待！

---

*本文是「AI 编程工具链实战」系列的第 19 篇，完整系列可在本专栏查看。*
