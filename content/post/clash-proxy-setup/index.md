---
title: "Linux 下 Clash 代理部署指南：Docker 模式与 TUN 模式"
description: "在 Linux 服务器上部署 Clash 代理的两种方式——Docker 容器快速部署和 TUN 模式全局透明代理，涵盖配置文件、环境变量、systemd 服务管理的完整流程。"
date: 2020-08-16
categories:
    - 工具与效率
tags:
    - Clash
    - 代理
    - Docker
    - Linux
draft: false
---

## 两种部署模式对比

| 模式 | 原理 | 优点 | 缺点 |
|------|------|------|------|
| **Docker 模式** | 容器内运行，通过端口暴露代理服务 | 部署简单、隔离性好 | 需要手动设置环境变量或应用配置 |
| **TUN 模式** | 创建虚拟网卡，内核层面劫持所有流量 | 全局透明代理，应用无感知 | 配置复杂，需要 root 权限 |

---

## 方式一：Docker 快速部署

一条命令启动 Clash 容器：

```bash
docker run -d \
  --name clash \
  --restart always \
  -p 7890:7890 \
  -p 7891:7891 \
  -p 7893:7893 \
  -p 9090:9090 \
  -v /home/user/clash/config.yaml:/root/.config/clash/config.yaml \
  dreamacro/clash:latest
```

**端口说明**：

| 端口 | 协议 | 用途 |
|:----:|------|------|
| 7890 | HTTP | HTTP 代理端口 |
| 7891 | SOCKS5 | SOCKS5 代理端口 |
| 7893 | Mixed | 混合代理端口（HTTP + SOCKS5） |
| 9090 | HTTP | RESTful API / Web Dashboard |

### 设置环境变量

Docker 模式下，需要手动为终端或应用设置代理环境变量：

```bash
export http_proxy='http://127.0.0.1:7890'
export https_proxy='http://127.0.0.1:7890'
```

> **持久化**：将上面两行写入 `~/.bashrc` 或 `~/.zshrc`，每次登录自动生效。

### 验证代理

```bash
curl -I https://www.google.com
# 如果返回 200，说明代理生效
```

### 管理容器

```bash
# 查看日志
docker logs -f clash

# 重启（更新配置后）
docker restart clash

# 停止
docker stop clash
```

---

## 方式二：TUN 模式（全局透明代理）

TUN 模式在内核层面创建虚拟网卡，**所有流量**自动经过 Clash 处理，无需逐个应用配置代理。

### Step 1: 下载 Clash

```bash
wget https://github.com/Kuingsmile/clash-core/releases/download/1.18/clash-linux-amd64-v3-v1.18.0.gz
gunzip clash-linux-amd64-v3-v1.18.0.gz
chmod +x clash-linux-amd64-v3-v1.18.0
mv clash-linux-amd64-v3-v1.18.0 /usr/bin/clash
```

### Step 2: 配置文件

在已有的 `config.yaml` 中添加 DNS 和 TUN 配置：

```yaml
# 追加到 config.yaml

dns:
  enable: true
  listen: 0.0.0.0:53
  enhanced-mode: fake-ip
  nameserver:
    - 114.114.114.114
  fallback:
    - 8.8.8.8

tun:
  enable: true
  stack: system          # system 或 gvisor（system 性能更好）
  dns-hijack:
    - 8.8.8.8:53
    - tcp://8.8.8.8:53
    - any:53
    - tcp://any:53
  auto-route: true                # 自动设置全局路由
  auto-detect-interface: true     # 自动检测出口网卡
```

**关键配置说明**：

| 配置 | 说明 |
|------|------|
| `enhanced-mode: fake-ip` | 使用虚假 IP 响应 DNS 请求，加速解析 |
| `stack: system` | 使用系统网络栈（性能好），`gvisor` 为用户态实现（兼容性好） |
| `auto-route: true` | 自动添加路由规则，所有流量走 TUN 网卡 |
| `auto-detect-interface` | 自动检测物理网卡，避免流量回环 |
| `dns-hijack` | 劫持 DNS 请求，确保域名解析也经过 Clash |

### Step 3: 开启 IP 转发

编辑 `/etc/sysctl.conf`，添加：

```ini
net.ipv4.ip_forward=1
```

使其生效：

```bash
sysctl -p
```

### Step 4: 创建 systemd 服务

创建服务文件 `/etc/systemd/system/clash.service`：

```ini
[Unit]
Description=Clash - A rule based proxy tunnel
After=network-online.target nftables.service iptables.service

[Service]
Type=simple
LimitNOFILE=65535
ExecStart=/usr/bin/clash -d /srv/clash

[Install]
WantedBy=multi-user.target
```

> **配置目录**：`-d /srv/clash` 指定配置文件目录，将 `config.yaml` 放到 `/srv/clash/` 下。

启动并设置开机自启：

```bash
# 创建配置目录
mkdir -p /srv/clash
cp config.yaml /srv/clash/

# 重新加载 systemd
systemctl daemon-reload

# 启动服务
systemctl start clash

# 设置开机自启
systemctl enable clash

# 查看状态
systemctl status clash
```

### Step 5: 验证

```bash
# 检查 TUN 网卡是否创建
ip addr show utun

# 检查路由表
ip route show table main

# 测试代理
curl https://www.google.com
```

---

## 常用运维命令

```bash
# 查看 Clash 日志
journalctl -u clash -f

# 重启（修改配置后）
systemctl restart clash

# 通过 API 切换节点
curl -X PUT http://127.0.0.1:9090/proxies/<group-name> \
  -H "Content-Type: application/json" \
  -d '{"name": "<proxy-name>"}'

# 查看当前流量统计
curl http://127.0.0.1:9090/traffic

# 刷新配置（无需重启）
curl -X PUT http://127.0.0.1:9090/configs \
  -H "Content-Type: application/json" \
  -d '{"path": "/srv/clash/config.yaml"}'
```

---

## Web Dashboard

Clash 内置 RESTful API（端口 9090），可配合 Web 面板管理：

在 `config.yaml` 中添加：

```yaml
external-controller: 0.0.0.0:9090
secret: "<your-secret>"           # API 访问密钥
external-ui: dashboard            # Web UI 目录
```

常用 Dashboard：
- **Yacd**：`https://yacd.haishan.me`
- **Clash Dashboard**：`https://clash.razord.top`

打开浏览器访问上述地址，输入 `http://<服务器IP>:9090` 和 secret 即可管理节点。
