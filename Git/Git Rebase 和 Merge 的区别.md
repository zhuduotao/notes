---
tags:
  - git
  - version-control
  - branching
created: 2026-04-15
updated: 2026-04-15
---
## 概述

`git merge` 和 `git rebase` 都是用于整合分支的命令，但它们的工作方式和产生的历史完全不同。

## Merge（合并）

### 工作原理

- 创建一个新的 **merge commit**，将两个分支的历史连接在一起
- 保留完整的分支历史，包括所有的分叉和合并点
- 是非破坏性操作，不会修改已有的提交

### 命令

```bash
git checkout main
git merge feature
```

### 特点

| 特性 | 说明 |
|------|------|
| 历史记录 | 保留完整的历史，包括分叉 |
| 安全性 | 安全，不会改变已有提交 |
| 复杂度 | 历史可能变得复杂，尤其是频繁合并时 |
| 冲突处理 | 只需解决一次冲突 |

### 适用场景

- 公共分支（如 main、develop）
- 需要保留完整历史的项目
- 多人协作的共享分支

## Rebase（变基）

### 工作原理

- 将当前分支的提交 **重新应用** 到目标分支的最新提交之上
- 重写提交历史，使历史看起来是线性的
- 会创建新的提交（新的 SHA），丢弃旧的提交

### 命令

```bash
git checkout feature
git rebase main
```

### 特点

| 特性 | 说明 |
|------|------|
| 历史记录 | 线性历史，更清晰简洁 |
| 安全性 | 会修改历史，不适合公共分支 |
| 复杂度 | 历史更清晰，易于理解 |
| 冲突处理 | 可能需要多次解决冲突（每个提交一次） |

### 适用场景

- 本地特性分支整理
- 提交给公共分支前的历史清理
- 个人分支保持线性历史

## 核心区别对比

| 对比项 | Merge | Rebase |
|--------|-------|--------|
| 历史形状 | 非线性，保留分叉 | 线性，无分叉 |
| 提交历史 | 保留原始提交 | 创建新提交，重写历史 |
| 安全性 | 安全，适合公共分支 | 危险，不适合已推送的公共分支 |
| 冲突处理 | 一次解决 | 可能多次解决 |
| 撤销难度 | 容易（一个 revert） | 困难（需要找到所有新提交） |
| 使用场景 | 公共分支整合 | 本地分支整理 |

## 可视化对比

### Merge 产生的历史

```
A---B---C main
     \
      D---E feature
           \
            F---G (merge commit)
```

### Rebase 产生的历史

```
A---B---C main
         \
          D'---E' (rebase 后的 feature)
```

## 黄金规则

> **永远不要 rebase 已经推送到公共仓库的提交**

原因：
- rebase 会改变提交历史
- 其他人如果基于旧提交工作，会导致历史冲突
- 强制推送（`git push --force`）会覆盖他人的工作

## 常见工作流

### 1. 特性分支开发（推荐）

```bash
# 开发过程中定期 rebase 保持最新
git checkout feature
git rebase main

# 完成后 merge 到主分支
git checkout main
git merge feature
```

### 2. Pull Request 前整理

```bash
# 整理提交历史，使 PR 更清晰
git rebase -i main  # 交互式 rebase，可以 squash、reorder 等
```

### 3. 解决冲突

```bash
# Merge 冲突
git merge feature
# 解决冲突后
git add .
git commit

# Rebase 冲突
git rebase main
# 解决冲突后
git add .
git rebase --continue
# 或放弃
git rebase --abort
```

## 总结

| 选择 | 建议 |
|------|------|
| 公共分支整合 | 使用 **merge** |
| 本地历史整理 | 使用 **rebase** |
| 保持历史清晰 | rebase 后 merge（`git merge --no-ff`） |
| 团队协作 | 统一团队规范，避免混用造成混乱 |

## 参考链接

- [Git 官方文档 - rebase](https://git-scm.com/docs/git-rebase)
- [Git 官方文档 - merge](https://git-scm.com/docs/git-merge)
- [Atlassian Git Tutorial - Merging vs. Rebasing](https://www.atlassian.com/git/tutorials/merging-vs-rebasing)
