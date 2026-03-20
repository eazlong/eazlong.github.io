---
title: "Kubernetes 常见问题排查与环境初始化"
description: "记录 Kubernetes 集群搭建过程中的常见报错及解决方案，包括 API Server 连接失败、Swap 禁用、containerd 配置、kubeadm 初始化以及 NFS 和 MinIO 的部署配置。"
date: 2020-03-17
categories:
    - 技术文章
tags:
    - Kubernetes
    - 运维排查
    - containerd
    - DevOps
draft: false
---

## 问题速查表

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| `connection to server :6443 was refused` | API Server 未启动或启动失败 | 检查 kubelet 日志、确认 Swap 已关闭 |
| 节点 NotReady | Swap 未禁用 | 执行 `swapoff -a` 并持久化 |
| `kubectl` 命令报权限错误 | 未导出 KUBECONFIG | 导出 admin.conf 路径 |
| 镜像拉取超时 | 默认 registry.k8s.io 不可达 | 替换为阿里云镜像源 |

---

## 1. 关闭 Swap

Kubernetes 要求节点禁用 Swap，否则 kubelet 无法正常启动。

```bash
# 立即关闭
swapoff -a

# 持久化：注释 /etc/fstab 中的 swap 行，重启后仍然生效
sed -ri 's/.*swap.*/#&/' /etc/fstab
```

验证 Swap 已关闭：

```bash
free -h
# Swap 行应全部为 0
```

---

## 2. 导出 KUBECONFIG

`kubectl` 需要知道集群配置文件路径才能与 API Server 通信：

```bash
export KUBECONFIG=/etc/kubernetes/admin.conf
```

> **建议**：将此行写入 `~/.bashrc` 或 `~/.zshrc`，避免每次登录都要手动设置。

---

## 3. 配置 containerd

Kubernetes 1.24+ 已移除 dockershim，默认使用 containerd 作为容器运行时。以下步骤生成默认配置并进行必要调整：

```bash
# 生成默认配置
mkdir -p /etc/containerd/
containerd config default | tee /etc/containerd/config.toml >/dev/null 2>&1
```

### 3.1 启用 SystemdCgroup

确保 containerd 使用 systemd 作为 cgroup 驱动（与 kubelet 保持一致）：

```bash
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
```

### 3.2 替换镜像源为阿里云

国内环境无法直接访问 `registry.k8s.io`，替换为阿里云镜像源：

```bash
sed -i 's/registry.k8s.io/registry.aliyuncs.com\/google_containers/g' /etc/containerd/config.toml
```

### 3.3 更新 pause 镜像版本

```bash
sed -i 's/3.8/3.9/g' /etc/containerd/config.toml
```

### 3.4 重启 containerd

```bash
systemctl restart containerd
systemctl status containerd
```

---

## 4. 初始化 Kubernetes 集群

使用 `kubeadm init` 初始化 Master 节点：

```bash
kubeadm init \
  --apiserver-advertise-address 10.10.13.12 \
  --pod-network-cidr 10.244.0.0/16 \
  --image-repository registry.aliyuncs.com/google_containers
```

**参数说明**：

| 参数 | 作用 |
|------|------|
| `--apiserver-advertise-address` | API Server 监听的 IP，填写 Master 节点 IP |
| `--pod-network-cidr` | Pod 网络的 CIDR，`10.244.0.0/16` 适配 Flannel 网络插件 |
| `--image-repository` | 使用阿里云镜像源，解决国内无法拉取官方镜像的问题 |

> **初始化成功后**，会输出 `kubeadm join` 命令，请妥善保存，用于后续 Worker 节点加入集群。

---

## 5. 配置 NFS 存储

### 5.1 挂载 NFS 目录

```bash
mount -t nfs 10.10.13.12:/var/nfs/consul /var/nfs
```

### 5.2 安装 NFS Provisioner

提供两种方案，按需选择：

**方案 A：NFS Ganesha Server Provisioner**（自带 NFS Server）

```bash
helm repo add nfs-ganesha-server-and-external-provisioner \
  https://kubernetes-sigs.github.io/nfs-ganesha-server-and-external-provisioner/

helm install nfs-server-provisioner \
  nfs-ganesha-server-and-external-provisioner/nfs-server-provisioner
```

**方案 B：NFS Subdir External Provisioner**（使用已有 NFS Server）

```bash
helm repo add nfs-subdir-external-provisioner \
  https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner

helm install nfs-subdir-external-provisioner \
  nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --set nfs.server=10.10.13.6 \
  --set nfs.path=/var/nfs/consul
```

> **选择建议**：如果已有独立的 NFS Server，推荐方案 B，更轻量。

---

## 6. 部署 MinIO

### 6.1 准备配置

```yaml
# minio-values.yaml
global:
  storageClass: "nfs"
  minio:
    accessKey: <your-access-key>     # 替换为实际值
    secretKey: <your-secret-key>     # 替换为实际值
clusterDomain: cluster.local
mode: distributed
ingress:
  enabled: true
  certManager: false
  hostname: cortex-minio.onwalk.net
  extraTls:
  - hosts:
      - cortex-minio.onwalk.net
    secretName: tls-cortex-secret
```

### 6.2 安装或更新

```bash
helm upgrade --install minio bitnami/minio -n cortex -f minio-values.yaml
```

### 6.3 验证 MinIO

```bash
# 获取凭证
export ROOT_USER=$(kubectl get secret --namespace cortex minio \
  -o jsonpath="{.data.root-user}" | base64 -d)
export ROOT_PASSWORD=$(kubectl get secret --namespace cortex minio \
  -o jsonpath="{.data.root-password}" | base64 -d)

# 启动 MinIO Client 验证连接
kubectl run --namespace cortex minio-client \
  --rm --tty -i --restart='Never' \
  --env MINIO_SERVER_ROOT_USER=$ROOT_USER \
  --env MINIO_SERVER_ROOT_PASSWORD=$ROOT_PASSWORD \
  --env MINIO_SERVER_HOST=minio \
  --image docker.io/bitnami/minio-client:2023.9.13-debian-11-r0 \
  -- mc admin info minio
```

在 Pod 内配置别名并创建存储桶：

```bash
mc alias set minio http://minio:9000 $MINIO_SERVER_ROOT_USER $MINIO_SERVER_ROOT_PASSWORD
mc admin info minio
mc mb minio/cortex-tsdb
mc mb minio/cortex-ruler
mc mb minio/cortex-alertmanager
```

---

## 参考资料

- [Kubernetes 安装指南 - InfoQ](https://xie.infoq.cn/article/4a4bf681ef319ca80674e9c6e)
- [kubeadm 部署实践 - 简书](https://www.jianshu.com/p/39985a974000)
