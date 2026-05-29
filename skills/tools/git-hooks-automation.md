---
name: git-hooks-automation
description: Git Hooks 自动化实战：pre-commit 校验、commitlint 规范化、pre-push 检查与 Husky 集成
tags: [tools, git, hooks, automation, devops, conventional-commits]
---

## 概述

Git Hooks 是 Git 内置的事件回调机制，可以在特定操作（提交、推送、合并）前后自动执行脚本。结合 Husky、commitlint、lint-staged 等工具，可实现代码质量门禁和提交规范化。

## 1. Git Hooks 基础

### 常用 Hook 一览

| Hook | 触发时机 | 用途 |
|------|---------|------|
| **pre-commit** | `git commit` 前 | 代码格式化、lint 检查、单元测试 |
| **prepare-commit-msg** | 编辑提交信息前 | 自动生成提交信息模板 |
| **commit-msg** | 提交信息写入后 | 校验提交信息格式 |
| **post-commit** | 提交完成后 | 通知 CI、更新工单状态 |
| **pre-push** | `git push` 前 | 运行集成测试、检查分支命名 |
| **post-merge** | `git merge` 后 | 依赖安装、数据库迁移 |
| **pre-rebase** | `git rebase` 前 | 校验分支状态 |
| **post-checkout** | `git checkout` 后 | 自动安装依赖 |

### 手动安装 Hooks

```bash
# hooks 存储在 .git/hooks/ 目录
# 去掉 .sample 后缀即可启用
cp .git/hooks/pre-commit.sample .git/hooks/pre-commit

# 或直接创建自定义 hook
cat > .git/hooks/pre-commit << 'EOF'
#!/bin/sh
echo "Running pre-commit checks..."
# 检查是否有调试代码
if grep -r "debugger\|console.log" --include="*.js" --include="*.ts" src/; then
  echo "❌ 存在调试代码，请移除后提交"
  exit 1
fi
EOF
chmod +x .git/hooks/pre-commit
```

## 2. Husky 管理 Hooks（推荐）

Husky 可以将 hooks 配置为项目代码（`package.json` 或 `.husky/` 目录），团队成员共享。

### 安装

```bash
# npm 项目
npx husky-init
npm install

# 或手动安装
npm install --save-dev husky

# 启用 git hooks
npx husky install

# package.json 中添加 postinstall 确保其他人安装后自动启用
# "scripts": {
#   "postinstall": "husky install"
# }
```

### 创建 Hooks

```bash
# 创建 pre-commit hook
npx husky add .husky/pre-commit 'npx lint-staged'

# 创建 commit-msg hook
npx husky add .husky/commit-msg 'npx --no -- commitlint --edit $1'

# 创建 pre-push hook
npx husky add .husky/pre-push 'npm run test:ci'
```

### .husky/ 目录结构

```
.husky/
├── pre-commit         # 提交前检查
├── commit-msg         # 提交信息校验
├── pre-push           # 推送前检查
└── _/                 # Husky 内部文件
    └── husky.sh
```

## 3. commitlint — 提交信息规范化

### 安装配置

```bash
npm install --save-dev @commitlint/cli @commitlint/config-conventional
```

```javascript
// commitlint.config.js
module.exports = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'type-enum': [
      2, 'always',
      ['feat', 'fix', 'docs', 'style', 'refactor', 'perf',
       'test', 'build', 'ci', 'chore', 'revert']
    ],
    'scope-empty': [0],                          // scope 可选
    'subject-case': [2, 'always', 'lower-case'], // 小写开头
    'subject-max-length': [2, 'always', 72],     // 标题不超过 72 字
    'body-max-line-length': [2, 'always', 100],  // 正文每行不超过 100 字
  },
};
```

### Conventional Commits 格式

```
<type>(<scope>): <subject>

<body>

<footer>
```

```
feat(auth): add OAuth2 login support

Implement login with GitHub and Google OAuth2 providers.
Includes refresh token rotation and session management.

Closes #123
```

### 常用类型

| Type | 含义 | 版本影响 |
|------|------|---------|
| `feat` | 新功能 | minor |
| `fix` | Bug 修复 | patch |
| `BREAKING CHANGE` | 不兼容变更 | major |
| `docs` | 文档 | — |
| `refactor` | 重构 | — |
| `perf` | 性能提升 | — |
| `test` | 测试 | — |
| `chore` | 构建/工具 | — |
| `ci` | CI 配置 | — |

## 4. lint-staged — 暂存文件检查

只对 `git add` 的文件执行检查，避免全量检查的耗时。

```json
// package.json
{
  "lint-staged": {
    "*.{js,ts,tsx,vue}": [
      "eslint --fix",
      "prettier --write"
    ],
    "*.{css,scss,less}": [
      "stylelint --fix",
      "prettier --write"
    ],
    "*.{json,md,yaml,yml}": [
      "prettier --write"
    ],
    "*.java": [
      "java -jar google-java-format.jar --replace"
    ],
    "pom.xml": [
      "xmllint --noout --schema"
    ]
  }
}
```

```bash
# .husky/pre-commit
npx lint-staged
```

## 5. pre-push 检查策略

```bash
# .husky/pre-push
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

echo "🔍 Running pre-push checks..."

# 1. 运行单元测试（快速）
npm run test:unit
if [ $? -ne 0 ]; then
  echo "❌ 单元测试失败，终止推送"
  exit 1
fi

# 2. 检查分支命名规范
BRANCH_NAME=$(git rev-parse --abbrev-ref HEAD)
if [[ ! "$BRANCH_NAME" =~ ^(feature|fix|hotfix|release|chore)/[a-z0-9._-]+$ ]] && \
   [[ "$BRANCH_NAME" != "main" ]] && \
   [[ "$BRANCH_NAME" != "develop" ]]; then
  echo "❌ 分支名不符合规范: $BRANCH_NAME"
  echo "   应使用: feature/xxx, fix/xxx, hotfix/xxx"
  exit 1
fi

# 3. 检查是否忘记推送子模块
git submodule status | grep -q '^-' && {
  echo "❌ 子模块有未推送的变更"
  exit 1
}

# 4. 检查大文件（> 5MB）
LARGE_FILES=$(git diff --cached --name-only -z | xargs -0 ls -l 2>/dev/null | awk '$5 > 5000000')
if [ -n "$LARGE_FILES" ]; then
  echo "❌ 存在超过 5MB 的大文件:"
  echo "$LARGE_FILES"
  exit 1
fi

echo "✅ 所有检查通过"
```

## 6. 非 JS 项目使用 Hooks

### Python 项目（pre-commit 框架）

```bash
# 安装 pre-commit 框架
pip install pre-commit
```

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files
        args: ['--maxkb=5000']

  - repo: https://github.com/psf/black
    rev: 25.1.0
    hooks:
      - id: black

  - repo: https://github.com/PyCQA/flake8
    rev: 7.1.0
    hooks:
      - id: flake8
```

```bash
# 安装 hooks
pre-commit install

# 手动运行检查所有文件
pre-commit run --all-files

# 跳过某个 hook
SKIP=black git commit -m "chore: skip formatting"
```

### Java 项目（Maven + Hooks）

```xml
<!-- pom.xml 中使用 maven-git-hooks -->
<plugin>
    <groupId>com.github.ekryd.echo-plugin</groupId>
    <artifactId>maven-git-hooks</artifactId>
    <version>1.0.0</version>
    <executions>
        <execution>
            <goals><goal>install</goal></goals>
        </execution>
    </executions>
    <configuration>
        <hooks>
            <pre-commit>mvn checkstyle:check -q</pre-commit>
            <commit-msg>mvn validate -P commit-check</commit-msg>
        </hooks>
    </configuration>
</plugin>
```

## 7. 常见场景最佳实践

### 禁止提交调试代码

```bash
# .husky/pre-commit
#!/bin/sh

# 检查 Java TODO/FIXME
if git diff --cached --name-only --diff-filter=ACMR | grep '\.java$' | xargs grep -l 'TODO\|FIXME\|XXX' 2>/dev/null; then
  echo "⚠️  警告: 以下文件包含 TODO/FIXME:"
  git diff --cached --name-only --diff-filter=ACMR | grep '\.java$' | xargs grep -ln 'TODO\|FIXME\|XXX'
  # exit 1  # 严格模式取消注释
fi

# 检查密码/密钥泄露
git diff --cached -S'password' --diff-filter=ACMR -- '*.properties' '*.yml' '*.yaml'
```

### 自动更新文件头

```bash
# .git/hooks/prepare-commit-msg
#!/bin/bash
# 自动添加 JIRA 工单号到提交信息
BRANCH_NAME=$(git rev-parse --abbrev-ref HEAD)
JIRA_ID=$(echo "$BRANCH_NAME" | grep -oE '[A-Z]+-[0-9]+')

if [ -n "$JIRA_ID" ]; then
  # 如果提交信息还不包含 JIRA ID，追加
  if ! grep -q "$JIRA_ID" "$1"; then
    echo "[$JIRA_ID]" >> "$1"
  fi
fi
```

### 版本号自动标记

```bash
# .git/hooks/post-commit
#!/bin/sh
# 自动为 feat 或 fix 类型提交打标签
LAST_MSG=$(git log -1 --pretty=%B)
if echo "$LAST_MSG" | grep -qE '^(feat|fix)\(?'; then
  echo "✓ 检测到功能/修复提交，建议手动打标签"
  echo "  git tag v$(date +%Y%m%d.%H%M)"
fi
```

## 注意事项

- **Hooks 是客户端脚本**，不能替代 CI/CD 的服务端校验（用户可能 `git commit --no-verify`）
- **性能敏感**：pre-commit hook 应保持轻量（< 5秒），复杂检查放到 pre-push
- **团队共享**：推荐用 Husky 或 pre-commit 框架管理，不要直接编辑 `.git/hooks/`
- **Windows 兼容**：注意 shell 脚本的跨平台问题，Husky 会自动处理
- **二进制文件**：避免在大二进制文件上运行 diff 检查
- `.git/hooks/` **目录不参与 Git 提交**，必须通过工具（Husky/pre-commit）分发
- pre-push 中的**集成测试**时间不宜过长，超过 2 分钟可考虑移至 CI
