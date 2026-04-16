---
created: 2026-04-15
updated: 2026-04-15
tags:
  - Git
  - 工作流
  - 开发效率
  - 版本控制
aliases:
  - worktree
  - linked worktree
  - 多工作树
source_type: official-doc
source_urls:
  - https://git-scm.com/docs/git-worktree
status: verified
---

## 是什么

**Git Worktree** 是 Git 提供的一个功能，允许同一个仓库同时关联多个工作树（working tree）。每个工作树可以独立 checkout 不同的分支，拥有独立的 `HEAD`、`index` 和工作目录文件，但共享同一个 `.git` 仓库数据（对象库、refs 等）。

仓库结构：

- **主工作树（main worktree）**：通过 `git init` 或 `git clone` 创建的原始工作目录。
- **链接工作树（linked worktree）**：通过 `git worktree add` 创建的额外工作目录，共享主仓库的所有对象数据。

## 为什么重要

在没有 worktree 之前，如果需要在同一仓库的不同分支间并行工作，开发者通常有两种选择：

1. **使用 `git stash`**：暂存当前修改，切换到其他分支处理完后再恢复。缺点是频繁 stash/unstash 容易出错，且工作目录状态混乱时风险较高。
2. **克隆多个仓库副本**：每个副本独立工作。缺点是占用额外磁盘空间，且每个副本需要单独 fetch/pull 同步远程更新。

Git Worktree 完美解决了这两个问题：

- **无需 stash**：多个分支同时存在于不同目录，互不干扰。
- **共享对象库**：所有 worktree 共享同一个 `.git` 目录，只存储一份对象数据，节省磁盘空间。
- **自动同步**：在任一 worktree 中 fetch/push 的更新，其他 worktree 立即可见。

## 核心命令

### 创建 worktree

```bash
# 基于 HEAD 创建新分支并 checkout 到指定路径
git worktree add <path>

# 基于指定分支创建 worktree
git worktree add <path> <branch>

# 创建新分支并 checkout
git worktree add -b <new-branch> <path> [<commit-ish>]

# 创建 detached HEAD 的临时 worktree（不关联任何分支）
git worktree add -d <path>

# 基于远程分支创建（自动设置 upstream）
git worktree add --track -b <branch> <path> <remote>/<branch>
```

### 查看 worktree 列表

```bash
# 列出所有 worktree
git worktree list

# 详细信息（显示 locked/prunable 状态）
git worktree list -v

# 机器可读格式（适合脚本解析）
git worktree list --porcelain -z
```

输出示例：

```
/path/to/main-repo          abcd1234 [main]
/path/to/linked-worktree    1234abcd [feature-branch]
/path/to/temp-fix           5678ef01 (detached HEAD)
```

### 删除 worktree

```bash
# 删除干净的 worktree（无未跟踪文件、无修改）
git worktree remove <path>

# 强制删除（包含未提交修改或子模块）
git worktree remove -f <path>
```

### 锁定与解锁

```bash
# 锁定 worktree（防止被 prune 或意外删除）
git worktree lock <path>
git worktree lock --reason "在外部硬盘上" <path>

# 解锁
git worktree unlock <path>
```

### 移动 worktree

```bash
# 移动 worktree 到新位置
git worktree move <old-path> <new-path>
```

### 修复损坏的链接

当手动移动了 worktree 目录或主仓库后，链接可能失效：

```bash
# 在主仓库中运行，修复所有链接
git worktree repair

# 指定特定路径修复
git worktree repair <path1> <path2>
```

### 清理过期 worktree 信息

```bash
# 清理已删除但未通过 git worktree remove 删除的 worktree 元数据
git worktree prune

# 预览会清理的内容（不实际删除）
git worktree prune -n

# 只清理超过指定时间的过期项
git worktree prune --expire "30 days ago"
```

## 典型应用场景

### 场景 1：紧急 Bug 修复

正在开发新功能时，线上突然出现紧急 Bug 需要立即修复。

```bash
# 当前在 feature 分支工作，不 stash、不中断
git worktree add -b hotfix ../hotfix-fix main
cd ../hotfix-fix
# 修复 Bug、提交、推送
git commit -am "fix: 修复线上紧急问题"
git push origin hotfix-fix
# 完成后删除临时 worktree
cd ../main-repo
git worktree remove ../hotfix-fix
```

### 场景 2：多分支并行开发

同时开发两个独立功能，需要在两个功能间频繁切换和对比。

```bash
# 主工作目录在 main 分支
# 创建第二个 worktree 用于 feature-a
git worktree add ../project-feature-a feature-a

# 创建第三个 worktree 用于 feature-b
git worktree add ../project-feature-b feature-b
```

现在三个目录分别对应三个分支，可以在不同终端/编辑器窗口中独立工作，互不干扰。

### 场景 3：代码审查与对比

需要对比两个分支的差异，或者在 PR 审查时查看另一个分支的实际效果。

```bash
# 基于 PR 分支创建 worktree 进行本地验证
git worktree add ../pr-review pr-branch-name
cd ../pr-review
# 运行测试、查看效果、对比差异
```

### 场景 4：CI/CD 构建

在 CI 环境中，需要在同一个仓库的多个分支上并行执行构建或测试。

```bash
# CI 脚本中创建临时 worktree 执行构建
git worktree add --detach build-dir main
cd build-dir
# 执行构建、测试
npm run build
# 构建完成后清理
cd ..
git worktree remove build-dir
```

### 场景 5：长期维护多版本

需要同时维护 `main`、`v1.x`、`v2.x` 等多个长期分支。

```bash
git worktree add ../project-v1 v1.x
git worktree add ../project-v2 v2.x
```

每个目录对应一个版本，可以独立接收 cherry-pick、安全补丁等。

## Refs 共享规则

Worktree 之间的 ref 共享遵循以下规则：

| Ref 类型 | 是否共享 | 说明 |
|----------|---------|------|
| `refs/heads/*` | ✅ 共享 | 所有分支引用共享 |
| `refs/remotes/*` | ✅ 共享 | 远程跟踪分支共享 |
| `refs/tags/*` | ✅ 共享 | 标签共享 |
| `HEAD` | ❌ 独立 | 每个 worktree 独立 |
| `index` | ❌ 独立 | 每个 worktree 独立 |
| `refs/bisect/*` | ❌ 独立 | bisect 状态独立 |
| `refs/worktree/*` | ❌ 独立 | worktree 特有引用 |
| `refs/rewritten/*` | ❌ 独立 | rebase 重写状态独立 |

跨 worktree 访问独立 ref 的方式：

```bash
# 访问主工作树的 HEAD
git rev-parse main-worktree/HEAD

# 访问某个 linked worktree 的 HEAD
git rev-parse worktrees/<name>/HEAD
```

## 配置

### Worktree 专属配置

默认情况下，仓库的 `config` 文件在所有 worktree 间共享。如果需要 worktree 专属配置（如不同的 `core.sparseCheckout`）：

```bash
# 启用 worktreeConfig 扩展
git config extensions.worktreeConfig true

# 添加 worktree 专属配置
git config --worktree core.sparseCheckout true
```

配置存储在 `.git/worktrees/<id>/config.worktree` 文件中。

> **注意**：启用此扩展后，旧版本 Git 将拒绝访问该仓库。

### 常用配置项

| 配置项 | 说明 | 默认值 |
|--------|------|--------|
| `worktree.guessRemote` | `add` 时自动匹配远程分支 | `false` |
| `worktree.useRelativePaths` | 使用相对路径链接 worktree（便于迁移） | `false` |
| `gc.worktreePruneExpire` | 过期 worktree 元数据的清理时间 | `3 months ago` |

## 限制与注意事项

### 不能 checkout 同一分支到多个 worktree

Git 默认不允许同一分支在多个 worktree 中同时 checkout。尝试这样做会报错：

```bash
# 错误示例：main 已在主 worktree 中 checkout
git worktree add ../temp main
# fatal: 'main' is already checked out at '/path/to/main-repo'
```

**解决方法**：使用 `--force` 强制创建（不推荐，可能导致冲突），或切换到其他分支/commit。

### 子模块支持不完整

官方文档明确指出：**多 worktree 对子模块（submodule）的支持尚不完整**。不推荐在包含子模块的超项目（superproject）中使用多个 worktree。

### 主工作树不能删除

```bash
# 错误：不能删除主工作树
git worktree remove /path/to/main-repo
# fatal: '/path/to/main-repo' is the main working tree
```

### 手动移动目录后需要修复

如果手动移动了 worktree 目录（而非使用 `git worktree move`），链接会失效。需要运行：

```bash
git worktree repair
```

### 裸仓库（bare repository）

裸仓库没有工作树，但可以作为主仓库创建多个 linked worktree。

### 磁盘位置建议

- worktree 目录建议放在与主仓库同级或相近的路径下，便于管理。
- 如果 worktree 在外部硬盘或网络共享上，使用 `git worktree lock` 防止元数据被自动清理。

## 与相关概念的对比

| 方案 | 适用场景 | 优点 | 缺点 |
|------|---------|------|------|
| **Git Worktree** | 同一仓库多分支并行 | 共享对象库、无需 stash、磁盘占用小 | 不支持同一分支多处 checkout、子模块支持不完整 |
| **Git Stash** | 临时切换分支 | 简单快速、无需额外目录 | 频繁操作易出错、工作目录混乱时风险高 |
| **多仓库克隆** | 完全独立的项目副本 | 完全隔离、无限制 | 磁盘占用大、需要分别同步 |
| **Git Branch** | 单分支切换 | 原生支持、简单 | 不能并行工作、切换时需 stash 或 commit |

## 最佳实践

### 1. 使用独立目录名

```bash
# 推荐：目录名清晰表达用途
git worktree add ../project-hotfix hotfix
git worktree add ../project-review-pr-123 pr-123

# 避免：使用临时或模糊的目录名
git worktree add ../temp main
git worktree add ../test feature
```

### 2. 及时清理不再使用的 worktree

```bash
# 定期检查
git worktree list

# 删除不再需要的
git worktree remove <path>

# 清理残留元数据
git worktree prune
```

### 3. 使用 detached HEAD 做一次性测试

```bash
# 创建临时测试环境，不关联任何分支
git worktree add -d ../temp-test
# 测试完成后直接删除
git worktree remove ../temp-test
```

### 4. 锁定长期使用的 worktree

```bash
# 防止被 gc 自动清理
git worktree lock ../project-v1 --reason "长期维护 v1.x 版本"
```

## 参考资料

- [Git 官方文档 - git-worktree](https://git-scm.com/docs/git-worktree)
- [Git 官方文档 - gitrepository-layout](https://git-scm.com/docs/gitrepository-layout)
- [Git 官方文档 - git-config (worktree 相关配置)](https://git-scm.com/docs/git-config)
