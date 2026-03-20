---
title: "Docker 部署 Prometheus + Grafana 监控栈"
description: "使用 Docker 快速搭建 Prometheus 监控体系，涵盖 Prometheus、Node Exporter、GPU Exporter、Pushgateway 和 Grafana 的完整部署流程。"
date: 2020-08-20
categories:
    - 技术文章
tags:
    - Prometheus
    - Grafana
    - Docker
    - 监控
    - DevOps
draft: false
---

## 架构概览

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ Node Exporter│     │ GPU Exporter │     │ Pushgateway  │
│   :9100      │     │   :9400      │     │   :9091      │
└──────┬───────┘     └──────┬───────┘     └──────┬───────┘
       │                    │                    │
       └────────────┬───────┴────────────────────┘
                    │ pull metrics
              ┌─────┴─────┐
              │ Prometheus │
              │   :9090    │
              └─────┬──────┘
                    │ query
              ┌─────┴─────┐
              │  Grafana   │
              │   :3000    │
              └────────────┘
```

| 组件 | 作用 | 端口 |
|------|------|:----:|
| **Prometheus** | 指标采集和存储 | 9090 |
| **Node Exporter** | 采集主机指标（CPU、内存、磁盘、网络） | 9100 |
| **GPU Exporter** | 采集 NVIDIA GPU 指标 | 9400 |
| **Pushgateway** | 接收主动推送的指标 | 9091 |
| **Grafana** | 可视化仪表盘 | 3000 |

---

## 1. 部署 Prometheus

### 1.1 准备配置文件

```bash
mkdir -p /opt/prometheus
```

```yaml
# /opt/prometheus/prometheus.yml
global:
  scrape_interval: 60s       # 采集间隔
  evaluation_interval: 60s   # 告警规则评估间隔

scrape_configs:
  # 监控 Prometheus 自身
  - job_name: prometheus
    static_configs:
      - targets: ['localhost:9090']
        labels:
          instance: prometheus

  # 监控 Linux 主机
  - job_name: linux
    static_configs:
      - targets: ['<host-ip>:9100']
        labels:
          instance: my-server

  # 监控 GPU（如果有）
  - job_name: gpu
    static_configs:
      - targets: ['<host-ip>:9400']
        labels:
          instance: gpu-server

  # Pushgateway
  - job_name: pushgateway
    honor_labels: true
    static_configs:
      - targets: ['<host-ip>:9091']
```

> **注意**：将 `<host-ip>` 替换为宿主机的实际 IP。容器内的 `localhost` 指向容器自身，不是宿主机。

### 1.2 启动容器

```bash
docker run -d \
  --name prometheus \
  --restart always \
  -p 9090:9090 \
  -v /opt/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml \
  prom/prometheus
```

### 1.3 验证

浏览器访问 `http://<host-ip>:9090`，进入 **Status > Targets** 查看采集目标是否正常。

---

## 2. 部署 Node Exporter

Node Exporter 采集主机级别的指标（CPU、内存、磁盘、网络等）。

```bash
docker run -d \
  --name node-exporter \
  --restart always \
  -p 9100:9100 \
  -v "/proc:/host/proc:ro" \
  -v "/sys:/host/sys:ro" \
  -v "/:/rootfs:ro" \
  prom/node-exporter
```

**挂载说明**：

| 挂载 | 作用 |
|------|------|
| `/proc:/host/proc:ro` | 读取进程和系统信息 |
| `/sys:/host/sys:ro` | 读取硬件和内核信息 |
| `/:/rootfs:ro` | 读取磁盘使用情况 |

验证：

```bash
curl http://localhost:9100/metrics | head -20
```

---

## 3. 部署 GPU Exporter

如果服务器有 NVIDIA GPU，可以部署 DCGM Exporter 采集 GPU 指标。

### 3.1 安装 NVIDIA Container Toolkit

```bash
# 添加仓库和密钥
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey \
  | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list \
  | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' \
  | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit

# 配置 Docker 运行时
nvidia-ctk runtime configure --runtime=docker
systemctl restart docker
```

### 3.2 启动 GPU Exporter

```bash
docker run -d \
  --name gpu-exporter \
  --restart always \
  --gpus all \
  -p 9400:9400 \
  nvcr.io/nvidia/k8s/dcgm-exporter:3.2.5-3.1.8-ubuntu22.04
```

验证：

```bash
curl http://localhost:9400/metrics | grep DCGM
```

---

## 4. 部署 Pushgateway

Pushgateway 允许短生命周期的任务（如批处理脚本、定时任务）主动推送指标。

```bash
docker run -d \
  --name pushgateway \
  --restart always \
  -p 9091:9091 \
  prom/pushgateway
```

### 推送指标示例

```bash
# 推送单条指标
echo "my_job_duration_seconds 42" | curl --data-binary @- \
  http://localhost:9091/metrics/job/my_batch_job

# 将 Node Exporter 的指标转推到 Pushgateway
curl -s http://localhost:9100/metrics \
  | grep -v "^\(go_\|http_request\|http_response\|process_\)" \
  | curl --data-binary @- \
    http://localhost:9091/metrics/job/node/instance/my-server
```

> **适用场景**：Prometheus 默认是 Pull 模式（主动拉取），但某些短生命周期任务（跑完就退出的脚本）无法被持续拉取，这时通过 Pushgateway 推送指标。

---

## 5. 部署 Grafana

```bash
mkdir -p /opt/grafana-storage /opt/grafana-conf
chmod 777 -R /opt/grafana-storage

docker run -d \
  --name grafana \
  --restart always \
  -p 3000:3000 \
  -v /opt/grafana-storage:/var/lib/grafana \
  -v /opt/grafana-conf:/etc/grafana \
  grafana/grafana
```

### 配置数据源

1. 浏览器访问 `http://<host-ip>:3000`（默认账号 `admin/admin`）
2. 进入 **Configuration > Data Sources > Add data source**
3. 选择 **Prometheus**
4. URL 填写 `http://<host-ip>:9090`
5. 点击 **Save & Test**

### 推荐 Dashboard

| Dashboard ID | 名称 | 用途 |
|:------------:|------|------|
| 1860 | Node Exporter Full | Linux 主机全面监控 |
| 12239 | NVIDIA DCGM Exporter | GPU 监控 |
| 3662 | Prometheus 2.0 Overview | Prometheus 自身监控 |

导入方式：**Dashboards > Import > 输入 Dashboard ID > Load**

---

## 端口汇总

| 服务 | 宿主机端口 | 容器端口 |
|------|:----------:|:--------:|
| Prometheus | 9090 | 9090 |
| Node Exporter | 9100 | 9100 |
| GPU Exporter | 9400 | 9400 |
| Pushgateway | 9091 | 9091 |
| Grafana | 3000 | 3000 |

---

## 一键启动（docker-compose）

将以上所有组件整合为 `docker-compose.yml`：

```yaml
# docker-compose.yml
version: '3'
services:
  prometheus:
    image: prom/prometheus
    restart: always
    ports:
      - "9090:9090"
    volumes:
      - /opt/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml

  node-exporter:
    image: prom/node-exporter
    restart: always
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro

  pushgateway:
    image: prom/pushgateway
    restart: always
    ports:
      - "9091:9091"

  grafana:
    image: grafana/grafana
    restart: always
    ports:
      - "3000:3000"
    volumes:
      - /opt/grafana-storage:/var/lib/grafana
```

```bash
docker-compose up -d
```
