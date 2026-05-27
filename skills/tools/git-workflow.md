---
name: git-workflow
description: Git 分支策略与团队协作最佳实践
tags: [tools, git, workflow, devops]
---

## 分支策略

### Git Flow（推荐中大型项目）

```bash
# 主分支
main        # 生产就绪代码
develop     # 开发集成分支

# 辅助分支
feature/*   # 新功能（从 develop 拉出，合并回 develop）
release/*   # 发布准备（从 develop 拉出，合并回 main + develop）
hotfix/*    # 紧急修复（从 main 拉出，合并回 main + develop）
```

```bash
# Feature 分支流程
git checkout -b feature/user-login develop
# ... 开发、提交 ...
git checkout develop
git merge --no-ff feature/user-login  # 保留分支历史
git branch -d feature/user-login
```

### Trunk-Based Development（CI/CD 推荐）

```bash
# 短命特性分支（存活 < 1 天）
git checkout -b feat/short-fix
# ... 小改动 ...
git commit -m "fix: typo in login validation"
git checkout main
git pull --rebase
git merge feat/short-fix
git push origin main
```

| 策略 | 适用场景 | 复杂度 | CI/CD 友好度 |
|------|---------|--------|-------------|
| Git Flow | 大型项目、版本发布严格 | 高 | 低 |
| GitHub Flow | 持续部署、中小团队 | 低 | 高 |
| GitLab Flow | 环境分支+功能分支混合 | 中 | 中 |
| Trunk-Based | 高频发布、微服务 | 低 | 极高 |

## Commit 规范（Conventional Commits）

```bash
# 格式: <type>(<scope>): <description>
feat: 新增用户登录功能
fix(api): 修复 NPE 当 userId 为空时
docs: 更新 API 文档
style: 格式化代码，补充分号
refactor: 重构订单状态机
perf: 优化缓存查询性能
test: 添加单元测试覆盖
chore: 更新依赖版本
ci: 修改 GitHub Actions 配置
```

```bash
# 实际示例
git commit -m "feat(order): 添加订单超时自动取消"
git commit -m "fix(payment): 修复微信支付回调验签失败"
git commit -m "perf(db): 添加 order_create_time 索引加速查询"
```

## Rebase vs Merge

```bash
# Merge（保留完整历史，有合并节点）
git checkout feature/add-login
git merge main
# 结果: 引入一个 merge commit，保留分支拓扑

# Rebase（线性历史，无合并节点）
git checkout feature/add-login
git rebase main
# 结果: 在 main 最新提交上重新应用 feature 的提交

# Interactive Rebase（整理提交）
git rebase -i HEAD~3
# pick → 保留
# squash → 合并到前一个提交
# reword → 修改提交信息
# fixup → 合并但不保留提交信息
```

### 黄金法则

```
不要在公共分支上 rebase！
rebase 只用于本地或私有特性分支
已 push 到远端的提交永远不要 rebase
```

## 团队协作最佳实践

### Pull Request 流程

```yaml
# .github/PULL_REQUEST_TEMPLATE.md
## 变更说明
- ...

## 关联 Issue
Closes #123

## 测试
- [ ] 单元测试通过
- [ ] 集成测试通过
- [ ] 本地验证通过

## 检查清单
- [ ] 代码符合规范
- [ ] 无敏感信息泄露
- [ ] API 兼容性确认
```

### 分支保护规则

```yaml
# GitHub Settings → Branches → Branch protection rules
Branch name pattern: main
# ☑ Require pull request reviews (至少 1 人)
# ☑ Dismiss stale pull request approvals
# ☑ Require status checks (CI 通过方可合并)
# ☑ Require branches to be up-to-date before merging
# ☑ Require conversation resolution
# ☑ Do not allow bypassing the above settings
```

### 工作流程

```bash
# 1. 同步最新代码
git checkout main
git pull --rebase

# 2. 创建特性分支
git checkout -b feat/xxx

# 3. 开发过程中定期同步
git fetch origin
git rebase origin/main    # 或 git merge origin/main

# 4. 提交前检查
git diff --stat            # 查看变更文件
git log origin/main..HEAD  # 查看未推送的提交
git push origin feat/xxx

# 5. 创建 PR → Code Review → Squash Merge
```

## Git Hooks 自动化

### pre-commit 配置

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files
        args: ['--maxkb=500']
```

```bash
# 安装 pre-commit
pre-commit install

# 手动运行
pre-commit run --all-files
```

### commit-msg 钩子

```bash
#!/bin/sh
# .git/hooks/commit-msg
# 校验 commit message 格式
commit_regex='^(feat|fix|docs|style|refactor|perf|test|chore|ci)(\(.+\))?: .{1,72}$'
if ! grep -qE "$commit_regex" "$1"; then
    echo "❌ 提交信息不符合规范: <type>(<scope>): <description>"
    exit 1
fi
```

## 常见问题的解决方案

```bash
# 撤销本地未推送的提交（保留改动）
git reset --soft HEAD~1

# 彻底丢弃本地未推送的提交
git reset --hard HEAD~1

# 修改最近一次提交的信息
git commit --amend -m "新提交信息"

# 把多个提交合并为一个
git rebase -i HEAD~3  # squash/fixup

# 误操作后找回提交
git reflog
git checkout -b recovered-branch <sha>

# 删除远程分支
git push origin --delete feature/old-branch

# 清理已合并的本地分支
git branch --merged main | grep -v '\*\|main\|develop' | xargs -n 1 git branch -d

# 子模块管理
git submodule add https://github.com/xxx/common-lib.git libs/common
git submodule update --init --recursive
git submodule foreach git pull origin main
```

## 注意事项

1. **大文件不要提交**：使用 `.gitignore` 排除，或 `git lfs` 管理二进制文件
2. **敏感信息禁止入库**：密码、token、密钥用环境变量或 secrets 管理，不要在代码中硬编码
3. **永远不要 force push 公共分支**：`git push --force` 会覆盖他人提交，改用 `--force-with-lease`
4. **保持提交原子化**：每个提交只做一个逻辑变更，方便回滚和 Code Review
5. **Gitignore 模板**：Java 项目参考 https://github.com/github/gitignore
6. **国内配置代理**：`git config --global http.proxy http://127.0.0.1:7890`
7. **大仓库优化**：`git config core.preloadindex=true` + `git config core.fsmonitor=true`
