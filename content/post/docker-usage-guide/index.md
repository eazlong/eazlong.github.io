---
title: "Docker 使用指南"
description: "Docker 常用命令速查表，涵盖镜像管理、容器操作、数据导入导出等核心功能"
date: 2020-08-14
categories:
    - 技术文章
tags:
    - Docker
    - 容器
    - DevOps
draft: false
---

## 概述

Docker 是目前最流行的容器化技术，本文整理了 Docker 的常用命令和操作，涵盖安装配置、镜像管理、容器操作、数据导入导出等核心功能。

## 安装与配置

### Ubuntu 系统安装

```bash
# 安装 Docker
sudo apt-get install docker-io

# 启动 Docker 服务
service docker start

# 设置开机自启
chkconfig docker on

# 将当前用户加入 docker 组（免 sudo 执行）
usermod -a -G docker $USER
```

> **注意**：执行 `usermod` 后需要重新登录才能生效。

## 常用命令速查表

### 镜像管理

| 命令 | 说明 |
|------|------|
| `docker images` | 查看本地镜像列表 |
| `docker rmi <image_name>` | 删除指定镜像 |
| `docker save -o <file> <image>` | 导出镜像到文件 |
| `docker load --input <file>` | 从文件导入镜像 |
| `docker build -t <name> <dir>` | 从 Dockerfile 构建镜像 |

### 容器操作

| 命令 | 说明 |
|------|------|
| `docker ps -a` | 查看所有容器（包括停止的） |
| `docker run [OPTIONS] <image> [CMD]` | 创建并启动容器 |
| `docker start/stop <container>` | 启动/停止容器 |
| `docker rm <container>` | 删除容器 |
| `docker logs <container_id>` | 查看容器日志 |
| `docker commit <container> <image>` | 将容器保存为镜像 |
| `docker cp <src> <dest>` | 容器与宿主机间复制文件 |
| `docker update <container>` | 更新容器配置 |

### Docker run 参数说明

| 参数 | 说明 |
|------|------|
| `-t` | 分配伪终端（交互式命令行） |
| `-i` | 保持 STDIN 打开（接受输入） |
| `-d` | 后台运行（守护进程模式） |
| `--restart=always` | 容器退出后自动重启 |

## 镜像导入导出

### 使用 save/load（推荐）

```bash
# 导出镜像
docker save -o nginx.tar nginx:latest

# 导入镜像
docker load --input nginx.tar
```

### 使用 export/import

```bash
# 导出容器快照
docker export > container.tar

# 导入为镜像
docker import container.tar new_image:tag
```

> **区别**：`save/load` 保存完整的镜像层信息，`export/import` 只保存容器快照，体积更小但丢失历史记录。

## 常用技巧

### 同步时区

容器默认使用 UTC 时区，可通过拷贝宿主机时区文件解决：

```bash
docker cp /etc/localtime <container_id>:/etc/
```

### 内存管理

当宿主机运行多个 Docker 容器时，如果内存资源不足，系统可能会随机 kill 掉某些容器。如果容器设置了 `--restart=always`，则会被反复重启。

**解决方案**：
- 使用 `-m` 或 `--memory` 参数限制容器内存
- 监控宿主机内存使用情况
- 合理规划容器资源分配

### 配置文件位置

容器配置文件位于：

```
/var/lib/container/{container_id}/hostconfig.json
```

可通过编辑此文件修改容器配置（需停止容器后操作）。

## 验证命令

```bash
# 查看 Docker 版本
docker --version

# 查看 Docker 信息
docker info

# 测试 Docker 是否正常工作
docker run hello-world
```