---
title: "Git 使用指南"
description: "Git 常用命令速查表，涵盖分支管理、撤销操作、.gitignore 配置、自定义合并等核心功能"
date: 2020-08-15
categories:
    - 技术文章
tags:
    - Git
    - 版本控制
    - DevOps
draft: false
---

## 概述

Git 是当今最流行的分布式版本控制系统，广泛应用于软件开发、文档管理和协作场景。本文整理了 Git 的常用命令和实用技巧，涵盖日常开发中的核心需求。

## 基础命令

### 仓库初始化与配置

```bash
# 初始化新仓库
git init

# 克隆远程仓库
git clone <repository_url>

# 添加远程仓库
git remote add <name> <url>
```

### 文件操作

```bash
# 添加文件到暂存区
git add <file>      # 添加单个文件
git add .          # 添加所有文件
git add -A         # 添加所有变化

# 提交更改
git commit -m "提交说明"
git commit -a -m "提交说明"  # 自动添加已跟踪文件并提交

# 推送更改
git push <remote_name> <branch_name>
```

## 分支管理

### 分支操作命令

| 命令 | 说明 |
|------|------|
| `git branch <name>` | 创建新分支 |
| `git checkout <name>` | 切换到指定分支 |
| `git checkout -b <name>` | 创建并切换到新分支 |
| `git merge <name>` | 合并指定分支到当前分支 |
| `git branch -d <name>` | 删除本地分支 |
| `git branch origin -d <name>` | 删除远程分支 |

### 分支工作流示例

```bash
# 创建并切换到新功能分支
git checkout -b feature/new-feature

# 开发完成后提交
git add .
git commit -m "添加新功能"

# 切换回主分支并合并
git checkout main
git merge feature/new-feature

# 删除已合并的功能分支
git branch -d feature/new-feature
```

## 撤销与恢复

### 重置操作

```bash
# 重置到特定提交（保留修改）
git reset <commit_hash>

# 重置到特定提交（丢弃修改）
git reset --hard <commit_hash>

# 重置到远程分支状态
git reset --hard origin/<branch>
```

### 放弃本地修改

```bash
# 丢弃工作区修改（未暂存）
git checkout .

# 丢弃所有未跟踪文件和目录
git clean -xdf

# 重置为最近一次提交
git reset --hard HEAD
```

### 合并冲突解决

```bash
# 当出现合并冲突时
git merge --abort     # 中止合并
git reset --merge     # 重置合并状态
```

> **提示**：合并冲突时，Git 会显示 "You have not concluded your merge (MERGE_HEAD exists)" 错误信息，使用上述命令解决。

## 远程仓库

### 常用远程操作

```bash
# 添加远程仓库
git remote add origin <url>

# 查看远程仓库
git remote -v

# 拉取远程更新
git pull <remote> <branch>

# 推送本地更改
git push -u origin master  # 设置上游分支
```

## .gitignore 模式详解

.gitignore 文件用于指定 Git 应忽略的文件和目录模式。

### 基础语法规则

| 模式 | 说明 | 示例 |
|------|------|------|
| `#` | 注释行 | `# 这是注释` |
| `*.ext` | 忽略特定扩展名 | `*.log` `*.zip` |
| `/dir` | 忽略根目录下的目录 | `/build` |
| `dir/` | 忽略所有目录下的该目录 | `node_modules/` |
| `!pattern` | 排除忽略规则 | `!important.log` |
| `**/` | 匹配任意层级目录 | `**/temp/` |

### 实用示例配置

```gitignore
# 忽略所有 .log 文件
*.log

# 忽略 build 目录下的所有文件
build/

# 忽略 node_modules 目录
node_modules/

# 忽略当前路径下的 TODO 文件，不包括子目录
/TODO

# 忽略 doc 目录下的 .txt 文件，但不包括子目录
doc/*.txt

# 忽略所有 zip 文件
*.zip

# 但保留重要的 zip 文件
!important.zip

# 忽略任意层级的 temp 目录
**/temp/

# 忽略特定路径模式
a/**/b    # 忽略 a/b, a/x/b, a/x/y/b 等
```

### 特殊规则说明

1. **精确匹配**：
   ```
   /mtk/do.c           # 只忽略根目录 mtk/do.c
   config.php          # 忽略当前路径的 config.php
   ```

2. **目录与文件区别**：
   ```
   bin/:               # 忽略 bin 目录下的所有内容
   /bin:               # 忽略根目录下的 bin 文件（非目录）
   ```

3. **通配符匹配**：
   ```
   /*.c                # 忽略根目录下的 .c 文件
   debug/*.obj         # 忽略 debug/ 下的 .obj 文件
   ```

## 高级技巧

### 自定义合并引擎

在某些情况下，可能需要防止特定文件在合并分支时被修改（如配置文件）。

#### 步骤 1: 创建自定义合并驱动程序

```bash
# 创建简单的合并驱动程序（始终使用当前分支版本）
git config --global merge.preserve.driver true
```

#### 步骤 2: 配置 .gitattributes

```bash
# 创建或编辑 .gitattributes 文件
echo 'email.json merge=preserve' >> .gitattributes
```

#### 步骤 3: 提交配置

```bash
# 添加并提交 .gitattributes
git add .gitattributes
git commit -m 'Preserve email.json during merges'
```

#### 步骤 4: 执行合并

```bash
# 切换到要合并的分支
git checkout master

# 合并其他分支（email.json 将保持不变）
git merge newbranch
```

### SSH 密钥配置

```bash
# 生成 SSH 密钥
ssh-keygen -t rsa

# 将公钥添加到 Git 服务器（如 Gitosis）
# 通常需要将 ~/.ssh/id_rsa.pub 内容添加到服务器的 keydir 目录
```

## 常见问题解决

### 错误：fatal: This operation must be run in a work tree

**问题原因**：在非工作树目录执行 Git 操作。

**解决方案**：
```bash
# 初始化仓库
git init

# 或检查当前目录是否为 Git 仓库
git status
```

### 错误：MERGE_HEAD exists

**问题原因**：存在未解决的合并冲突。

**解决方案**：
```bash
# 中止合并
git merge --abort

# 或重置合并状态
git reset --merge
```

## 验证命令

```bash
# 检查 Git 版本
git --version

# 查看当前状态
git status

# 查看提交历史
git log --oneline

# 查看分支图
git log --graph --oneline --all
```

> **最佳实践**：建议配置用户名和邮箱，以便正确识别提交者：
> ```bash
> git config --global user.name "Your Name"
> git config --global user.email "your.email@example.com"
> ```