---
title: "Linux CPU 频率调节与性能模式设置"
description: "掌握 Linux 下 CPU 频率调节（CPU Governor）的使用方法，了解五种调频策略的区别，学会使用 cpufrequtils 将 CPU 设置为性能模式以获得最大算力。"
date: 2020-08-18
categories:
    - 技术文章
tags:
    - Linux
    - CPU
    - 性能优化
    - 运维
draft: false
---

## 什么是 CPU Governor

现代 CPU 支持**动态频率调节**（DVFS，Dynamic Voltage and Frequency Scaling）——根据负载自动调整运行频率和电压。Linux 内核通过 **CPU Governor**（调频策略）来控制这一行为。

默认情况下，Linux 使用节能策略，CPU 会在低负载时降频以节省电力。但在服务器、实时计算等场景中，你可能需要让 CPU 始终以最高频率运行。

---

## 五种调频策略

| Governor | 行为 | 适用场景 |
|----------|------|----------|
| `performance` | 始终以最高频率运行 | 服务器、计算密集型任务 |
| `powersave` | 始终以最低频率运行 | 最大化省电 |
| `ondemand` | 按需调频，负载高时升频 | 桌面系统（默认） |
| `conservative` | 渐进式升降频，更平滑 | 笔记本电脑 |
| `schedutil` | 由调度器直接驱动调频 | 新版内核默认（5.x+） |

---

## 快速设置性能模式

### 安装工具

```bash
# Debian / Ubuntu
sudo apt-get install -y cpufrequtils

# CentOS / RHEL
sudo yum install -y cpufrequtils
```

### 查看当前状态

```bash
# 查看所有 CPU 核心的频率信息
cpufreq-info

# 简洁模式：只看当前频率和策略
cpufreq-info -p
# 示例输出: 800000 2600000 ondemand（最低频率 最高频率 当前策略）

# 查看可用的调频策略
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_governors
# 示例输出: performance powersave ondemand conservative schedutil
```

### 设置性能模式

```bash
# 将所有 CPU 核心设置为 performance 模式
sudo cpufreq-set -g performance
```

### 设置频率范围

如果不想让 CPU 始终跑满，可以指定频率上下限：

```bash
# 设置最低频率 1.8GHz，最高频率 2.6GHz
sudo cpufreq-set -d 1800m -u 2600m
```

| 参数 | 说明 |
|------|------|
| `-g <governor>` | 设置调频策略 |
| `-d <freq>` | 设置最低频率（单位 MHz 或 GHz） |
| `-u <freq>` | 设置最高频率 |
| `-c <cpu>` | 指定 CPU 核心编号（默认所有核心） |

### 验证设置

```bash
# 查看当前策略
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
# 预期输出: performance

# 查看当前频率
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
# 单位: kHz

# 查看所有核心的实时频率
grep "MHz" /proc/cpuinfo
```

---

## 持久化配置

`cpufreq-set` 命令在重启后会失效。要持久化，有以下方式：

### 方式一：cpufrequtils 配置文件

```bash
# /etc/default/cpufrequtils
GOVERNOR="performance"
MIN_SPEED="1800000"
MAX_SPEED="2600000"
```

重启服务：

```bash
sudo systemctl restart cpufrequtils
```

### 方式二：systemd 开机脚本

创建 `/etc/systemd/system/cpu-performance.service`：

```ini
[Unit]
Description=Set CPU governor to performance
After=multi-user.target

[Service]
Type=oneshot
ExecStart=/usr/bin/cpufreq-set -g performance
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable cpu-performance
```

### 方式三：直接写 sysfs（无需安装工具）

```bash
# 设置所有核心为 performance（适用于没有 cpufrequtils 的系统）
for cpu in /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor; do
    echo performance | sudo tee $cpu
done
```

---

## 场景选择建议

| 场景 | 推荐策略 | 理由 |
|------|----------|------|
| 服务器 / 数据库 | `performance` | 最大吞吐量，避免调频延迟 |
| 实时音视频处理 | `performance` | 避免降频导致处理不及时 |
| 桌面日常使用 | `ondemand` / `schedutil` | 兼顾性能和功耗 |
| 笔记本移动办公 | `powersave` / `conservative` | 延长电池续航 |
| 基准测试 / 跑分 | `performance` + 固定频率 | 结果可复现 |

> **注意**：在云服务器（虚拟机）上，CPU Governor 可能不可用或被 hypervisor 接管。可以通过 `cpufreq-info` 检查是否支持。
