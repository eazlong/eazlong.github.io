---
title: "Samba 简单配置：实现 Windows 与 Linux 文件共享"
description: "详细介绍在 Linux 系统中安装和配置 Samba 服务，实现 Windows 访问 Linux 共享文件夹的完整步骤"
date: 2020-09-05
categories:
    - 技术文章
tags:
    - Samba
    - 文件共享
    - Linux
    - Windows
    - 网络配置
draft: false
---

## 概述

Samba 是 Linux/UNIX 系统与 Windows 系统之间实现文件共享和打印服务的主流方案。本文将详细介绍 Samba 的安装、配置步骤，帮助您快速搭建跨平台文件共享服务。

## 工作原理

```
┌─────────────────────┐         ┌─────────────────────┐
│    Windows 客户端   │────────▶│    Linux 服务器     │
│                     │  SMB/CIFS│   (Samba 服务)      │
│  \\server\share     │◀────────│                     │
└─────────────────────┘         └─────────────────────┘
```

Samba 使用 SMB（Server Message Block）协议，该协议是 Windows 文件共享的基础。通过 Samba，Linux 服务器可以无缝融入 Windows 网络环境。

## 安装 Samba

### 使用包管理器安装

```bash
# CentOS/RHEL
yum install samba

# Debian/Ubuntu
apt install samba
```

> **提示**：安装完成后，Samba 服务通常名为 `smb`，需要手动启动。

## 配置 Samba

### 配置文件位置

Samba 的主配置文件位于：

```
/etc/samba/smb.conf
```

### 配置文件结构

Samba 配置文件采用 INI 格式，由多个 section 组成：

```ini
[global]          # 全局配置
[homes]           # 用户主目录共享
[share_name]      # 自定义共享
```

## 配置用户主目录共享

### 修改配置文件

编辑 `/etc/samba/smb.conf`，找到 `[homes]` 部分：

```ini
# 启用用户主目录共享
# 访问路径：\\服务器IP\用户名

[homes]
comment = Home Directories
browseable = no
read only = no
```

### 配置项说明

| 配置项 | 说明 |
|--------|------|
| comment | 共享目录的注释说明 |
| browseable | 是否在网络邻居中可见 |
| read only | 是否只读（设为 no 表示可写） |

> **重要**：`read only = no` 必须设置，否则用户无法写入文件。

## 创建系统用户

在使用 Samba 共享之前，需要先创建系统用户：

```bash
# 创建系统用户
useradd 用户名

# 设置用户密码（系统密码）
passwd 用户名
```

## 添加 Samba 用户

将系统用户添加到 Samba 用户数据库：

```bash
# 添加 Samba 用户
smbpasswd -a 用户名

# 启用 Samba 用户
smbpasswd -e 用户名

# 查看所有 Samba 用户
pdbedit -L
```

> **注意**：Samba 用户必须是系统中已存在的用户。

## 启动服务

```bash
# 启动 Samba 服务
systemctl start smb

# 设置开机自启
systemctl enable smb

# 查看服务状态
systemctl status smb

# 重启服务（配置修改后需要重启）
systemctl restart smb
```

## 防火墙配置

如果开启了防火墙，需要允许 Samba 服务：

```bash
# CentOS/RHEL
firewall-cmd --permanent --add-service=samba
firewall-cmd --reload

# Debian/Ubuntu (如使用 ufw)
ufw allow samba
```

## Windows 客户端访问

### 方法一：文件管理器访问

1. 打开 Windows 文件管理器
2. 在地址栏输入：`\\服务器IP`
3. 输入用户名和密码登录

```
访问路径格式：\\192.168.1.100\用户名
```

### 方法二：映射网络驱动器

1. 右键「此电脑」→「映射网络驱动器」
2. 选择驱动器号，输入共享路径：`\\服务器IP\用户名`
3. 勾选「使用其他凭据连接」

### 方法三：命令行访问

```cmd
# 查看共享
net view \\服务器IP

# 映射网络驱动器
net use Z: \\服务器IP\用户名
```

## 常见问题

### 1. 无法访问共享

**可能原因**：服务未启动、防火墙阻止、用户权限不足

**排查步骤**：
1. 检查 Samba 服务状态
2. 检查防火墙规则
3. 验证用户密码

### 2. 无法写入文件

**检查要点**：
1. 确认配置中 `read only = no`
2. 检查文件系统权限
3. 确认 SELinux 上下文正确

### 3. 密码错误

```bash
# 重置 Samba 用户密码
smbpasswd 用户名
```

## 安全建议

| 建议 | 说明 |
|------|------|
| 使用强密码 | 设置复杂的 Samba 用户密码 |
| 限制访问IP | 在 `hosts allow` 中限制可访问的 IP 段 |
| 启用加密 | 在 `[global]` 中设置 `encrypt passwords = yes` |
| 定期更新 | 保持 Samba 版本为最新 |

## 总结

Samba 配置的主要步骤：

1. **安装 Samba**：使用包管理器安装
2. **配置共享**：修改 `smb.conf`，启用用户主目录
3. **创建用户**：系统用户 + Samba 用户
4. **启动服务**：启动并设置开机自启
5. **客户端访问**：Windows 通过 `\\IP\用户名` 访问

通过以上步骤，即可实现 Windows 与 Linux 之间的文件共享。

---

**参考来源**：有道笔记导出