---
title: 自定义pre-commit脚本
date: 2023-10-12 14:54:04
tags:
  - Front-end
---

## 需求背景

QNSolutions_Web 是一个多项目集成的 repo，front 下是多个项目，而 solution 下是一个 monorepo。

现需要实现当某个项目中有文件变更 commit 时，只对该项目进行 lint-staged 操作，而不是对整个 repo 进行 lint-staged 操作。

Repo 结构如下：

```
.
├── portal
│     └── front
│           ├── portal-solution
│           ├── portal-solution-admin
│           ├── portal-solution-desktop
│           └── portal-solution-dp-admin
├── solution
│       └── packages
│            ├── solution-a
│            ├── solution-b
│            ├── solution-c
│            └── ...
```

## 实现思路

1. 遍历所有项目，找到包含 package.json 且存在 lint-staged 命令的目录
2. 使用 git diff 来检查项目目录下的文件是否在提交列表中
3. 执行 lint-staged 命令

脚本实现如下：

```shell
#!/usr/bin/env sh
# 找到包含 package.json 且存在 lint-staged 命令的目录
projects=$(find . -type d -name "node_modules" -prune -o -type f -name "package.json" -exec grep -q "lint-staged" {} \; -exec dirname {} \;)

# 将发生文件变更的项目存储在 changed_projects 变量中
changed_projects=""

# 遍历每个项目，检查是否有文件变更
for project in $projects; do
  # 使用 git diff 来检查项目目录下的文件是否在提交列表中
  cd "$project"
  if git diff --cached --name-only --relative | grep -Fq ""; then
    changed_projects="$changed_projects $project"
  fi
  cd - > /dev/null
done

# 执行 lint-staged 命令
for project in $changed_projects; do
  echo "Running lint-staged for project: $project"
  (cd "$project" && pnpm install && pnpm run lint-staged)
done

```
