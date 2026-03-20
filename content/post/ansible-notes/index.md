---
title: "Ansible 常用模块与实战技巧"
description: "整理 Ansible 日常运维中最常用的模块，包括命令执行、用户管理、文件操作、包管理等，附带实用命令示例和批量操作技巧。"
date: 2020-08-15
categories:
    - 技术文章
tags:
    - Ansible
    - 自动化运维
    - DevOps
draft: false
---

## 什么是 Ansible

Ansible 是一个**无需 Agent** 的 IT 自动化工具，通过 SSH 连接远程主机执行任务。它的核心优势是简单——不需要在被管理节点上安装任何软件，只需 SSH 可达即可。

```
控制节点（安装 Ansible）
    │
    ├── SSH ──→ 主机 A
    ├── SSH ──→ 主机 B
    └── SSH ──→ 主机 C
```

---

## 常用模块速查表

| 模块 | 用途 | 说明 |
|------|------|------|
| `command` | 执行简单命令 | 不支持管道、重定向 |
| `shell` | 执行 Shell 命令 | 支持管道、重定向 |
| `raw` | 原始命令执行 | 不依赖 Python，适合初始化环境 |
| `user` / `group` | 用户和组管理 | 创建、删除、修改用户和组 |
| `cron` | 定时任务管理 | 添加、删除 crontab 条目 |
| `apt` / `yum` | 包管理 | 安装、删除、更新软件包 |
| `file` | 文件和目录操作 | 创建、删除、修改权限 |
| `copy` | 文件拷贝 | 从控制节点拷贝到远程 |
| `synchronize` | 文件同步 | 基于 rsync 的高效同步 |
| `filesystem` / `mount` | 文件系统操作 | 创建文件系统、挂载分区 |
| `unarchive` | 解压文件 | 支持 tar、zip 等格式 |
| `script` | 执行脚本 | 将本地脚本传到远程执行 |
| `setup` | 收集主机信息 | 获取系统 facts（硬件、网络等） |
| `get_url` | 下载文件 | 从 URL 下载文件到远程主机 |
| `authorized_key` | SSH 密钥管理 | 批量分发公钥 |

---

## 1. 命令执行模块

### command / shell / raw 的区别

| 模块 | 支持管道/重定向 | 依赖 Python | 适用场景 |
|------|:--------------:|:-----------:|----------|
| `command` | 否 | 是 | 简单命令（最安全） |
| `shell` | 是 | 是 | 需要管道、重定向的命令 |
| `raw` | 是 | 否 | 初始化环境、安装 Python |

> **建议**：日常使用统一用 `raw` 模块执行 Shell 命令，兼容性最好，不依赖远程 Python 环境。

### shell 模块参数

```bash
ansible-doc -s shell
```

| 参数 | 说明 |
|------|------|
| `chdir` | 执行前先 `cd` 到指定目录 |
| `creates` | 指定文件存在时**跳过**执行（幂等性） |
| `removes` | 指定文件不存在时**跳过**执行 |
| `executable` | 指定 Shell 解释器（绝对路径） |
| `free_form` | 要执行的命令（必填） |

### 示例

```bash
# 在所有主机上执行命令
ansible all -m shell -a "df -h"

# 切换到指定目录后执行
ansible webservers -m shell -a "ls -la" --extra-vars "chdir=/var/log"

# 仅当文件不存在时才执行（幂等）
ansible all -m shell -a "creates=/opt/app/installed echo 'installing...'"

# 使用 raw 模块（推荐）
ansible all -m raw -a "uptime"
```

---

## 2. 用户与权限管理

```bash
# 创建用户
ansible all -m user -a "name=deploy state=present shell=/bin/bash"

# 创建组
ansible all -m group -a "name=devops state=present"

# 将用户加入组
ansible all -m user -a "name=deploy groups=devops append=yes"

# 删除用户
ansible all -m user -a "name=testuser state=absent remove=yes"
```

---

## 3. 包管理

```bash
# Debian/Ubuntu
ansible all -m apt -a "name=nginx state=latest update_cache=yes"

# CentOS/RHEL
ansible all -m yum -a "name=nginx state=latest"

# 卸载
ansible all -m apt -a "name=nginx state=absent"
```

---

## 4. 文件操作

```bash
# 创建目录
ansible all -m file -a "path=/opt/app state=directory mode=0755 owner=deploy"

# 拷贝文件到远程
ansible all -m copy -a "src=/local/config.yml dest=/opt/app/config.yml mode=0644"

# 使用 rsync 同步目录
ansible all -m synchronize -a "src=/local/project/ dest=/opt/project/"

# 下载文件
ansible all -m get_url -a "url=https://example.com/file.tar.gz dest=/tmp/"

# 解压文件
ansible all -m unarchive -a "src=/tmp/file.tar.gz dest=/opt/ remote_src=yes"
```

---

## 5. 定时任务

```bash
# 添加定时任务（每天凌晨 3 点执行备份）
ansible all -m cron -a "name='daily backup' hour=3 minute=0 job='/opt/scripts/backup.sh'"

# 删除定时任务
ansible all -m cron -a "name='daily backup' state=absent"
```

---

## 6. SSH 密钥批量分发

这是运维初始化中最常用的操作——批量将控制节点的公钥分发到所有主机：

```bash
ansible all -m authorized_key \
  -a "user=deploy exclusive=true manage_dir=true key='$(< /root/.ssh/id_rsa.pub)'" \
  -u deploy -k
```

**参数说明**：

| 参数 | 说明 |
|------|------|
| `user=deploy` | 目标用户 |
| `exclusive=true` | 清除该用户现有的其他公钥，只保留本次指定的 |
| `manage_dir=true` | 自动创建 `.ssh` 目录并设置权限 |
| `key='$(< ...)'` | 读取本地公钥内容 |
| `-u deploy` | 以 deploy 用户身份连接 |
| `-k` | 首次连接时手动输入密码 |

> **注意**：`exclusive=true` 会**清除**目标用户已有的其他公钥，仅在初始化时使用。日常添加公钥应设为 `false`。

---

## 7. 收集主机信息

```bash
# 查看所有 facts
ansible all -m setup

# 过滤特定信息
ansible all -m setup -a "filter=ansible_distribution*"
ansible all -m setup -a "filter=ansible_memory_mb"
ansible all -m setup -a "filter=ansible_default_ipv4"
```

---

## 查看模块帮助

```bash
# 列出所有可用模块
ansible-doc -l

# 查看模块的简要用法
ansible-doc -s MODULE_NAME

# 查看模块的完整文档
ansible-doc MODULE_NAME
```

---

## 常用技巧

```bash
# 指定主机清单文件
ansible -i hosts.ini all -m ping

# 限制执行范围
ansible all -m shell -a "hostname" --limit "web*"

# 检查模式（不实际执行，仅显示会做什么）
ansible all -m shell -a "rm -rf /tmp/cache" --check

# 提升权限执行
ansible all -m shell -a "systemctl restart nginx" --become

# 并发执行（默认 5 个并发）
ansible all -m shell -a "apt update" -f 20
```
