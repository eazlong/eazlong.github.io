---
title: "K3s 集群部署 Cortex 监控系统完整指南"
description: "从零开始在 K3s 轻量级 Kubernetes 集群上部署 Cortex 分布式监控系统，涵盖 NFS 存储、Consul、MinIO、Memcached、Cortex 及 Grafana 的完整安装流程。"
date: 2020-08-11
categories:
    - 技术文章
tags:
    - K3s
    - Kubernetes
    - Cortex
    - 监控
    - DevOps
draft: false
---

## 架构概览

本文介绍如何在 K3s 集群上部署一套完整的 Cortex 监控体系。最终架构如下：

| 组件 | 作用 |
|------|------|
| **K3s** | 轻量级 Kubernetes 发行版，作为底层编排平台 |
| **NFS Provisioner** | 提供动态持久化存储 |
| **Consul** | 服务发现，保存 Cortex 的 Hash Ring 状态 |
| **MinIO** | S3 兼容对象存储，存放 TSDB 块数据、告警规则等 |
| **Memcached** | 缓存层，加速查询性能 |
| **Cortex** | 分布式、多租户的长期 Prometheus 存储 |
| **Grafana** | 可视化仪表盘 |

> **环境假设**：本文以 3 节点集群为例，Master 节点 IP 为 `10.10.13.4`，NFS 存储节点为 `10.10.13.6`（hostname: `kj6`）。请根据实际情况替换。

---

## 1. 安装 K3s 集群

### 1.1 Master 节点

```bash
curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn sh -
```

安装完成后 K3s 会自动启动，`kubectl` 命令即刻可用。

### 1.2 Worker 节点

先从 Master 上获取 join token：

```bash
cat /var/lib/rancher/k3s/server/node-token
```

然后在每个 Worker 节点执行（替换 token 和 Master IP）：

```bash
curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
  K3S_URL=https://10.10.13.4:6443 \
  K3S_TOKEN=<your-token> \
  sh -
```

验证集群状态：

```bash
kubectl get nodes
```

---

## 2. 准备命名空间与 TLS 证书

### 2.1 创建命名空间

```bash
kubectl create namespace cortex
kubectl create namespace minio
```

### 2.2 生成自签名证书

为 MinIO 和 Cortex Gateway 分别生成 TLS 证书：

```bash
# MinIO 证书
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls_minio.key -out tls_minio.crt \
  -subj "/CN=minio.node1.one"

# Cortex Gateway 证书
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls_cortex_gateway.key -out tls_cortex_gateway.crt \
  -subj "/CN=gateway.node1.one"
```

### 2.3 创建 Kubernetes Secret

```bash
kubectl create secret tls tls-cortex-gateway-secret \
  --cert=tls_cortex_gateway.crt --key=tls_cortex_gateway.key -n cortex

kubectl create secret tls tls-minio-secret \
  --cert=tls_minio.crt --key=tls_minio.key -n minio
```

> **提示**：生产环境建议使用 cert-manager 自动管理证书续期。

---

## 3. 安装 Helm

```bash
snap install helm
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```

建议将 `KUBECONFIG` 写入 `~/.bashrc` 或 `~/.zshrc`，避免每次手动设置。

---

## 4. 配置 NFS 持久化存储

Cortex 各组件需要持久化存储，这里通过 NFS 提供。

### 4.1 安装 NFS 服务

**存储节点**（提供磁盘空间的机器，如 `10.10.13.6`）：

```bash
apt install nfs-kernel-server
mkdir -p /var/nfs
```

编辑 `/etc/exports`，添加：

```
/var/nfs 10.10.13.0/24(rw,sync,no_subtree_check)
```

重启服务：

```bash
systemctl restart nfs-server.service
```

**所有节点**（包括 Master 和 Worker）：

```bash
apt install nfs-common
```

### 4.2 安装 NFS Provisioner

```bash
helm repo add nfs-ganesha-server-and-external-provisioner \
  https://kubernetes-sigs.github.io/nfs-ganesha-server-and-external-provisioner/

helm install nfs-server-provisioner \
  nfs-ganesha-server-and-external-provisioner/nfs-server-provisioner \
  -n default \
  --set persistence.enabled=true \
  --set persistence.size=512Gi \
  --set persistence.storageClass=standard \
  --set storageClass.defaultClass=true \
  --set nodeSelector.kubernetes\\.io/hostname=kj6
```

### 4.3 创建 PersistentVolume

```yaml
# nfs-server-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: data-nfs-server-provisioner-0
spec:
  capacity:
    storage: 512Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /mnt/nfs/
  claimRef:
    namespace: default
    name: data-nfs-server-provisioner-0
```

```bash
kubectl apply -f nfs-server-pv.yaml -n default
```

> `claimRef.name` 对应 Helm 自动生成的 PVC 名称，确保匹配。

---

## 5. 安装 Consul

Consul 用于 Cortex 各组件之间的服务发现，以及保存 Hash Ring 状态。

```yaml
# consul-values.yaml
global:
  storageClass: nfs
clusterDomain: cluster.local
```

```bash
helm upgrade --install consul bitnami/consul -n cortex -f consul-values.yaml
```

---

## 6. 安装 MinIO 对象存储

MinIO 提供 S3 兼容的对象存储，用于存放 Cortex 的 TSDB 块数据、告警规则和 Alertmanager 配置。

### 6.1 生成访问密钥

```bash
accessKey=$(cat /dev/random | head -c20 | base64)
secretKey=$(cat /dev/random | head -c50 | base64)
```

> **重要**：请妥善保存生成的密钥，后续 Cortex 配置需要用到。

### 6.2 部署 MinIO

```yaml
# minio-values.yaml
global:
  storageClass: "nfs"
  minio:
    accessKey: $accessKey
    secretKey: $secretKey
clusterDomain: cluster.local
mode: distributed
ingress:
  enabled: true
  certManager: false
  hostname: minio.node1.one
  extraTls:
  - hosts:
      - minio.node1.one
    secretName: tls-minio-secret
```

```bash
helm upgrade --install minio bitnami/minio -n minio -f minio-values.yaml
```

### 6.3 创建存储桶

部署完成后，创建 Cortex 所需的三个存储桶：

```bash
# 获取 MinIO 凭证
export ROOT_USER=$(kubectl get secret --namespace minio minio \
  -o jsonpath="{.data.root-user}" | base64 -d)
export ROOT_PASSWORD=$(kubectl get secret --namespace minio minio \
  -o jsonpath="{.data.root-password}" | base64 -d)

# 启动 MinIO Client Pod
kubectl run --namespace minio minio-client \
  --rm --tty -i --restart='Never' \
  --env MINIO_SERVER_ROOT_USER=$ROOT_USER \
  --env MINIO_SERVER_ROOT_PASSWORD=$ROOT_PASSWORD \
  --env MINIO_SERVER_HOST=minio \
  --image docker.io/bitnami/minio-client:2024.5.24-debian-12-r0 bash
```

在 Pod 内执行：

```bash
mc alias set minio http://minio:9000 $MINIO_SERVER_ROOT_USER $MINIO_SERVER_ROOT_PASSWORD
mc admin info minio
mc mb minio/cortex-tsdb
mc mb minio/cortex-ruler
mc mb minio/cortex-alertmanager
```

---

## 7. 安装 Memcached

Memcached 作为 Cortex 的查询缓存层，加速 index、chunks 和 metadata 的读取。

```yaml
# memcached-values.yaml
global:
  storageClass: nfs
clusterDomain: cluster.local
```

```bash
helm upgrade --install memcached bitnami/memcached -n cortex -f memcached-values.yaml
```

---

## 8. 安装 Cortex

这是核心步骤，Cortex 的配置较为复杂，下面逐模块说明。

### 8.1 准备配置文件

> **注意**：请将配置中的 MinIO `access_key_id` 和 `secret_access_key` 替换为你在第 6 步中部署的实际凭证。

```yaml
# cortex-values.yaml
image:
  tag: v1.10.0
clusterDomain: cluster.local

# --- Ingress 配置 ---
ingress:
  enabled: true
  annotations:
    kubernetes.io/ingress.class: nginx
  hosts:
    - host: gateway.node1.one
      paths:
        - /
  tls:
    - secretName: tls-cortex-gateway-secret
      hosts:
        - gateway.node1.one

# --- Nginx Gateway ---
nginx:
  enabled: true
  replicas: 2
  http_listen_port: 30333
  config:
    dnsResolver: kube-dns.kube-system.svc.cluster.local

# --- Ingester（数据写入） ---
ingester:
  replicas: 3
  persistentVolume:
    enabled: true
    accessModes: [ReadWriteOnce]
    size: 10Gi
    storageClass: nfs

# --- Compactor（数据压缩） ---
compactor:
  enabled: true
  replicas: 1
  persistentVolume:
    enabled: true
    accessModes: [ReadWriteOnce]
    size: 10Gi
    storageClass: nfs

# --- Store Gateway（历史数据查询） ---
store_gateway:
  replicas: 1
  persistentVolume:
    enabled: true
    accessModes: [ReadWriteOnce]
    size: 10Gi
    storageClass: nfs

# --- Cortex 核心配置 ---
config:
  auth_enabled: false

  distributor:
    shard_by_all_labels: true
    pool:
      health_check_ingesters: true
    instance_limits:
      max_ingestion_rate: 0          # 0 = 不限制
      max_inflight_push_requests: 0

  ingester_client:
    grpc_client_config:
      max_recv_msg_size: 104857600   # 100MB
      max_send_msg_size: 104857600
      grpc_compression: gzip

  ingester:
    lifecycler:
      join_after: 0                  # 立即加入
      final_sleep: 0s
      num_tokens: 512
      ring:
        kvstore:
          store: consul
          consul:
            host: consul-headless:8500
        replication_factor: 1
    instance_limits:
      max_ingestion_rate: 0
      max_tenants: 0
      max_series: 0
      max_inflight_push_requests: 0

  querier:
    query_ingesters_within: 3h
    store_gateway_addresses: store-gateway-1:9008,store-gateway-2:9009

  # --- 块存储配置（使用 MinIO 作为后端） ---
  blocks_storage:
    backend: s3
    tsdb:
      dir: /data/cortex-tsdb-ingester
      ship_interval: 1m
      block_ranges_period: [2h]
      retention_period: 3h
      max_exemplars: 50000
    bucket_store:
      sync_dir: /data/cortex-tsdb-querier
      consistency_delay: 5s
      index_cache:
        backend: memcached
        memcached:
          addresses: memcached:11211
      chunks_cache:
        backend: memcached
        memcached:
          addresses: memcached:11211
      metadata_cache:
        backend: memcached
        memcached:
          addresses: memcached:11211
    s3:
      endpoint: minio:9000
      bucket_name: cortex-tsdb
      access_key_id: <your-access-key>      # 替换为实际值
      secret_access_key: <your-secret-key>  # 替换为实际值
      insecure: true

  # --- Ruler（告警规则评估） ---
  ruler:
    enable_api: true
    enable_sharding: true
    ring:
      heartbeat_period: 5s
      heartbeat_timeout: 15s
      kvstore:
        store: consul
        consul:
          host: consul-headless:8500
    alertmanager_url: >-
      http://alertmanager-1:8031/alertmanager,
      http://alertmanager-2:8032/alertmanager,
      http://alertmanager-3:8033/alertmanager
    enable_alertmanager_v2: false

  ruler_storage:
    backend: s3
    s3:
      bucket_name: cortex-ruler
      endpoint: minio:9000
      access_key_id: <your-access-key>
      secret_access_key: <your-secret-key>
      insecure: true

  # --- Alertmanager（告警管理） ---
  alertmanager:
    enable_api: true
    sharding_enabled: true
    sharding_ring:
      replication_factor: 3
      heartbeat_period: 5s
      heartbeat_timeout: 15s
      kvstore:
        store: consul
        consul:
          host: consul-headless:8500

  alertmanager_storage:
    backend: s3
    s3:
      bucket_name: cortex-alertmanager
      endpoint: minio:9000
      access_key_id: <your-access-key>
      secret_access_key: <your-secret-key>
      insecure: true

  storage:
    engine: blocks

  # --- Compactor ---
  compactor:
    compaction_interval: 30s
    data_dir: /data/cortex-compactor
    consistency_delay: 1m
    sharding_enabled: true
    cleanup_interval: 1m
    tenant_cleanup_delay: 1m
    sharding_ring:
      kvstore:
        store: consul
        consul:
          host: consul-headless:8500

  # --- Store Gateway ---
  store_gateway:
    sharding_enabled: true
    sharding_ring:
      replication_factor: 1
      heartbeat_period: 5s
      heartbeat_timeout: 15s
      kvstore:
        store: consul
        consul:
          host: consul-headless:8500

  # --- 前端查询 ---
  frontend:
    query_stats_enabled: true

  frontend_worker:
    frontend_address: "query-frontend:9007"
    match_max_concurrent: true

  query_range:
    split_queries_by_interval: 24h

  limits:
    max_query_length: 744h            # 最大查询范围 31 天
```

### 8.2 部署 Cortex

```bash
helm upgrade --install cortex cortex/cortex -n cortex -f cortex-values.yaml
```

验证所有 Pod 正常运行：

```bash
kubectl get pods -n cortex
```

---

## 9. 安装 Grafana

### 9.1 获取并修改配置

```bash
helm show values grafana/grafana --version 6.24.1 > grafana-values.yaml
```

编辑 `grafana-values.yaml`，修改以下字段：

```yaml
persistence:
  enabled: true
  storageClassName: nfs
```

### 9.2 部署 Grafana

```bash
helm upgrade --install grafana grafana/grafana -f grafana-values.yaml -n cortex
```

### 9.3 暴露 NodePort 访问

```yaml
# nodeport_grafana.yaml
apiVersion: v1
kind: Service
metadata:
  name: grafana-nodeport
spec:
  selector:
    app.kubernetes.io/name: grafana
  type: NodePort
  ports:
    - port: 3000
      name: grafana
      targetPort: 3000
      nodePort: 30303
```

```bash
kubectl apply -f nodeport_grafana.yaml -n cortex
```

### 9.4 获取登录密码

```bash
# 用户名: admin
kubectl get secret --namespace cortex grafana \
  -o jsonpath="{.data.admin-password}" | base64 --decode; echo
```

---

## 10. 配置数据源

1. 浏览器访问 `http://gateway.node1.one:30303/`
2. 使用 `admin` 和上一步获取的密码登录
3. 进入 **Configuration > Data Sources > Add data source**
4. 选择 **Prometheus**
5. URL 填写：

```
http://cortex-query-frontend:8080/api/prom
```

6. 点击 **Save & Test** 验证连接

---

## 总结

至此，完整的监控体系已部署完成：

```
Prometheus (remote_write) --> Cortex Distributor --> Ingester --> MinIO (长期存储)
                                                         |
Grafana <-- Query Frontend <-- Querier <-- Store Gateway --+
```

**关键运维提示**：
- 定期检查 NFS 存储空间和 MinIO 桶容量
- TLS 证书有效期为 365 天，注意在到期前续期
- 通过 `kubectl get ring -n cortex` 检查 Hash Ring 健康状态
- Cortex 的 `auth_enabled: false` 仅适用于内部环境，生产环境请启用多租户认证
