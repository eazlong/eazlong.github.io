---
title: "云服务器 GPU 环境搭建：从驱动安装到 Docker 容器化运行"
description: "在云服务器上搭建完整的 GPU 开发环境，涵盖 NVIDIA 驱动、CUDA、Docker、nvidia-docker 的安装配置，以及 TensorFlow GPU 容器的运行。"
date: 2020-03-27
categories:
    - 技术文章
tags:
    - GPU
    - NVIDIA
    - Docker
    - TensorFlow
    - 深度学习
draft: false
---

## 环境搭建流程

在云服务器上运行 GPU 计算任务，需要自底向上安装以下组件：

```
┌──────────────────────────────┐
│   TensorFlow / PyTorch 容器   │  ← 应用层
├──────────────────────────────┤
│   nvidia-docker / toolkit     │  ← 容器 GPU 支持
├──────────────────────────────┤
│   Docker Engine               │  ← 容器运行时
├──────────────────────────────┤
│   CUDA Toolkit                │  ← GPU 计算库
├──────────────────────────────┤
│   NVIDIA Driver               │  ← GPU 驱动
├──────────────────────────────┤
│   Linux Kernel + kernel-devel │  ← 操作系统
└──────────────────────────────┘
```

---

## 1. 安装 NVIDIA GPU 驱动

### 1.1 安装内核开发包

GPU 驱动需要编译内核模块，必须安装与当前内核版本完全匹配的 `kernel-devel` 包。

```bash
# 查看当前内核版本
uname -r
# 示例输出: 3.10.0-514.26.2.el7.x86_64
```

通过 yum 安装：

```bash
yum install kernel-devel-$(uname -r)
```

> 如果 yum 源中找不到对应版本，可手动下载 RPM 包安装：
> ```bash
> # 从镜像站下载（替换为实际内核版本）
> wget ftp://mirror.switch.ch/pool/4/mirror/scientificlinux/7.3/x86_64/updates/fastbugs/kernel-devel-3.10.0-514.26.2.el7.x86_64.rpm
> rpm -ivh kernel-devel-3.10.0-514.26.2.el7.x86_64.rpm
> ```

### 1.2 安装驱动

从 [NVIDIA 官网](https://www.nvidia.com/Download/index.aspx) 下载对应 GPU 型号的驱动：

```bash
# 添加执行权限并安装
chmod +x NVIDIA-Linux-x86_64-410.72.run
sh NVIDIA-Linux-x86_64-410.72.run
```

验证安装：

```bash
nvidia-smi
```

预期输出包含 GPU 型号、驱动版本和显存信息：

```
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 410.72       Driver Version: 410.72       CUDA Version: 10.0     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  Tesla P40           Off  | 00000000:00:08.0 Off |                    0 |
| N/A   26C    P0    10W / 250W |      0MiB / 22919MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
```

---

## 2. 安装 CUDA Toolkit

CUDA 是 NVIDIA 的并行计算平台，深度学习框架依赖它来调用 GPU 算力。

```bash
# CentOS 7 安装 CUDA 7.5（根据需要选择版本）
rpm -ivh cuda-repo-rhel7-7-5-local-7.5-18.x86_64.rpm
yum install -y cuda
```

验证安装：

```bash
nvcc --version
# 预期输出: Cuda compilation tools, release 7.5, V7.5.18
```

> **版本选择**：CUDA 版本需要与 GPU 驱动版本、深度学习框架版本匹配。参考 [CUDA 兼容性矩阵](https://docs.nvidia.com/cuda/cuda-toolkit-release-notes/index.html)。

---

## 3. 安装 Docker + NVIDIA Container Toolkit

### 3.1 安装 Docker

```bash
# 卸载旧版本（如果有）
yum remove docker docker-common docker-selinux docker-engine

# 安装 Docker CE
yum install -y docker
```

### 3.2 安装 NVIDIA Container Toolkit

NVIDIA Container Toolkit（原 nvidia-docker2）让 Docker 容器能够访问宿主机的 GPU。

```bash
# 添加 NVIDIA 仓库
distribution=$(. /etc/os-release; echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.repo \
  | sudo tee /etc/yum.repos.d/nvidia-docker.repo

# 安装
yum install -y nvidia-container-toolkit
yum install -y nvidia-docker2
```

### 3.3 配置 Docker

编辑 `/etc/docker/daemon.json`，添加镜像加速和 NVIDIA 运行时：

```json
{
    "registry-mirrors": ["https://mirror.ccs.tencentyun.com"],
    "runtimes": {
        "nvidia": {
            "path": "/usr/bin/nvidia-container-runtime",
            "runtimeArgs": []
        }
    }
}
```

重启 Docker：

```bash
systemctl restart docker
systemctl enable docker
```

验证 GPU 在容器内可用：

```bash
docker run --rm --gpus all nvidia/cuda:11.0-base nvidia-smi
```

---

## 4. 运行 GPU 容器

### 4.1 启动 TensorFlow GPU 容器

```bash
# 使用 nvidia-docker（旧方式）
nvidia-docker run -it \
  -p 8001:8001 \
  -p 8081:8081 \
  tensorflow/tensorflow:latest-gpu

# 使用 --gpus 参数（新方式，推荐）
docker run --gpus all -it \
  -p 8001:8001 \
  -p 8081:8081 \
  tensorflow/tensorflow:latest-gpu
```

| 参数 | 说明 |
|------|------|
| `--gpus all` | 将所有 GPU 暴露给容器 |
| `--gpus '"device=0"'` | 只暴露第 0 号 GPU |
| `-p 8001:8001` | 端口映射 |
| `-it` | 交互式终端 |

### 4.2 查找 GPU 镜像

```bash
# 搜索 TensorFlow 相关镜像
docker search tensorflow

# 常用 GPU 镜像
# tensorflow/tensorflow:latest-gpu          官方 TensorFlow GPU
# pytorch/pytorch:latest                    官方 PyTorch（含 GPU 支持）
# nvidia/cuda:11.0-devel                    CUDA 开发环境基础镜像
```

---

## 5. 常用运维操作

```bash
# 启动 Docker 服务
systemctl start docker

# 从 tar 导入镜像
docker import tf_watch.tar tf_watch:latest

# 查看运行中的容器
docker ps

# 查看 GPU 使用情况
nvidia-smi

# 持续监控 GPU（每 2 秒刷新）
watch -n 2 nvidia-smi

# 查看容器内 GPU 日志
docker logs <container_id>
```

---

## 常见问题

| 问题 | 解决方案 |
|------|----------|
| `nvidia-smi` 报错 `Failed to initialize NVML` | 驱动未正确安装或内核版本不匹配，重新安装驱动 |
| 容器内看不到 GPU | 确认使用了 `--gpus all` 或 `nvidia-docker` 启动 |
| CUDA 版本不兼容 | 检查驱动版本与 CUDA 版本的兼容矩阵 |
| `kernel-devel` 版本不匹配 | 必须与 `uname -r` 输出的内核版本完全一致 |
| Docker 启动失败 | 检查 `daemon.json` 语法是否正确（JSON 格式） |
