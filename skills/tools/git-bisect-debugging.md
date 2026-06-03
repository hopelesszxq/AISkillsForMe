---
name: git-bisect-debugging
description: Git bisect 二分查找法快速定位 Bug 引入的提交，含自动化脚本与最佳实践
tags: [tools, git, debugging]
---

## 基本原理

Git bisect 利用二分查找算法，在 O(log n) 次比较内找到引入 Bug 的提交。

```
已知: good = v1.0 (正常)  bad = v1.5 (有 Bug)
提交链: g1-g2-g3-g4-g5-g6-b7-b8-b9-b10
           ^                  ^
         good               bad

Step 1: 检查中间提交 g6 ← 正常 → 右侧还有Bug
Step 2: 检查 b8 ← 有Bug → 左侧还有good
Step 3: 检查 b7 ← 有Bug → 找到! b7 引入了Bug
```

## 基础用法

```bash
# 1. 开始 bisect
git bisect start

# 2. 标记已知正常和异常提交
git bisect good v1.0    # 正常版本
git bisect bad v1.5     # 有Bug版本

# 3. Git 自动切换到一个中间提交
#    测试后标记
git bisect good          # 如果当前提交正常
git bisect bad           # 如果当前提交有Bug

# 4. 重复第3步，直到找到第一个bad提交
#    输出: xyz1234 是第一个bad提交

# 5. 结束 bisect
git bisect reset
```

## 自动化 bisect（无头模式）

```bash
# 让 bisect 自动运行测试脚本
git bisect start HEAD v1.0
git bisect run pytest tests/test_regression.py

# 或自定义脚本
git bisect run ./scripts/check_bug.sh
```

### 自动化脚本示例

```bash
#!/bin/bash
# check_bug.sh - 返回 0=正常, 1=有Bug
set -e

# 构建项目
mvn clean compile -q

# 运行特定测试
mvn test -Dtest=PaymentServiceTest#testRefund -q

# 检查测试结果
if [ $? -eq 0 ]; then
    exit 0  # 正常 (good)
else
    exit 1  # 有Bug (bad)
fi
```

```bash
# 跳过不可编译的提交
git bisect run sh -c "
  mvn clean compile -q 2>/dev/null || exit 125;
  mvn test -Dtest=OrderServiceTest -q
"
# exit 125 = 跳过该提交（不可测试）
```

## 实用场景

### 场景1：性能回退定位

```bash
#!/bin/bash
# bench_check.sh
mvn clean package -q
java -jar target/app.jar &
APP_PID=$!
sleep 5

# 性能测试
START=$(date +%s%N)
curl -s http://localhost:8080/api/orders > /dev/null
END=$(date +%s%N)
DURATION=$(( (END - START) / 1000000 ))

kill $APP_PID 2>/dev/null

# 如果延迟超过 200ms 认为异常
[ $DURATION -lt 200 ] && exit 0 || exit 1
```

```bash
git bisect start HEAD v1.0
git bisect run ./bench_check.sh
```

### 场景2：前端回归 Bug

```bash
#!/bin/bash
# e2e_check.sh
npm ci --silent
npx playwright test tests/login.spec.js --reporter=dot 2>&1 | tail -1

# 检查测试结果
if grep -q "passed" <<< "$LAST_LINE"; then
    exit 0
else
    exit 1
fi
```

### 场景3：只搜索特定文件范围内的提交

```bash
# 只跟踪修改过 payment-service/ 目录的提交
git bisect start HEAD v1.0 -- payment-service/
# 这样 bisect 会跳过不涉及该目录的提交，加速定位
```

## bisect 跳过策略

```bash
# 跳过无法测试的提交（编译失败、环境问题）
git bisect skip

# 一次跳过多个
git bisect skip v1.1 v1.2 v1.3

# 查看 bisect 进度
git bisect log

# 可视化 bisect 状态
git bisect visualize --oneline
```

## 进阶技巧

### 保存/恢复 bisect 会话

```bash
# 保存当前 bisect 状态
git bisect log > bisect-session.log

# 恢复（比如换机器继续）
git bisect replay bisect-session.log
```

### fixup! 提交的处理

```bash
# 当 history 中有 fixup/squash 提交时
# 使用 --first-parent 只看主分支提交
git bisect start --first-parent
```

### 用 bisect 找配置变更

```bash
# 配置文件的 bug 也可以用 bisect
git bisect start HEAD v1.0 -- config/application.yml

# 测试脚本
#!/bin/bash
diff <(cat config/application.yml | grep "timeout") \
     <(echo "api.timeout: 5000") && exit 0 || exit 1
```

### 批量跳过合并提交

```bash
# 一些合并提交太大，跳过它们
git bisect start HEAD v1.0
# 在 bisect 过程中
git log --oneline --merges HEAD | head -5
# 如果当前是 merge commit，跳过
git rev-parse --verify MERGE_HEAD > /dev/null 2>&1 && git bisect skip
```

## 常见问题

| 问题 | 解决方案 |
|---|---|
| 编译失败中断了 bisect run | 脚本中返回 exit 125 跳过 |
| 测试环境不一致导致假阳性 | 用 Docker 容器测试确保环境一致 |
| bisect 找到的提交与 Bug 无关 | 检查该提交是否真的修改了相关代码 |
| 回退太快没看清结果 | 用 `git bisect log` 查看完整日志 |
| bisect 被中断后恢复 | `git bisect replay` 恢复会话 |

## 别名配置

```bash
# 添加到 ~/.gitconfig
git config --global alias.bisect-run '!f() { 
  git bisect start "$@" && git bisect run; 
}; f'

git config --global alias.bisect-reset 'bisect reset'
git config --global alias.bisect-log 'bisect log'
```

## 注意事项

- **先确认 Bug 可复现**：bisect 依赖可靠的重现步骤
- **选择合理的 good/bad**：bad 越早越好，good 离 bad 越近越快定位
- **自动化脚本必须幂等**：多次运行结果一致，不依赖外部状态
- **编译时间长的项目**：考虑缓存或增量编译，避免 bisect run 过慢
- **merge commit 的处理**：bisect 默认深入 merge 内部，如果想跳过 merge 用 `--first-parent`
- **不要在生产环境跑**：bisect 会切换代码版本，应在开发/CI 环境执行
- **配合 git blame**：bisect 找到提交后，用 `git blame` 精确到行级
