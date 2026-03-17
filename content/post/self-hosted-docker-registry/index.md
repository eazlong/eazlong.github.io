---
title: "自建 Docker 私有镜像仓库指南"
description: "使用官方 Registry 镜像快速搭建 Docker 私有仓库，配置镜像加速和客户端访问，实现内网环境下的镜像自主管理。"
date: 2020-08-13
categories:
    - 技术文章
tags:
    - Docker
    - 容器
    - DevOps
    - 私有仓库
draft: false
---

## 为什么需要私有仓库

在内网或离线环境中，直接从 Docker Hub 拉取镜像往往受限于网络速度和访问策略。搭建私有仓库可以：

| 优势 | 说明 |
|------|------|
| **加速分发** | 内网传输，镜像拉取速度大幅提升 |
| **离线可用** | 不依赖外网，适合隔离环境部署 |
| **版本管控** | 统一管理镜像版本，避免生产环境使用未经验证的镜像 |
| **安全可控** | 敏感业务镜像不暴露到公网 |

> **环境假设**：仓库部署在 `10.10.13.7` 节点，使用 `/mnt/ssd/nvme1/harbor/harbor` 作为镜像存储目录。请根据实际情况替换。

---

## 1. 部署 Registry 服务

使用 Docker 官方的 `registry` 镜像，一条命令即可启动：

```bash
docker run -itd \
  -p 5000:5000 \
  -v /mnt/ssd/nvme1/harbor/harbor:/var/lib/registry \
  -e 'REGISTRY_HTTP_SECRET=true' \
  docker.io/registry
```

**参数说明**：

| 参数 | 作用 |
|------|------|
| `-p 5000:5000` | 将容器的 5000 端口映射到宿主机 |
| `-v ...:/var/lib/registry` | 持久化镜像数据到宿主机磁盘，防止容器重建后数据丢失 |
| `-e REGISTRY_HTTP_SECRET=true` | 设置 HTTP Secret，防止 Registry 启动时因缺少 secret 而报错 |

验证服务是否正常运行：

```bash
curl http://10.10.13.7:5000/v2/_catalog
# 预期输出: {"repositories":[]}
```

---

## 2. 配置 Docker 客户端

在**所有需要访问私有仓库的机器**上，编辑 Docker 守护进程配置：

```json
// /etc/docker/daemon.json
{
  "registry-mirrors": [
    "https://05f073ad3c0010ea0f4bc00b7105ec20.mirror.swr.myhuaweicloud.com"
  ],
  "insecure-registries": [
    "http://10.10.13.7:5000"
  ]
}
```

**字段说明**：

- `registry-mirrors`：Docker Hub 镜像加速地址，加速从公网拉取镜像
- `insecure-registries`：允许通过 HTTP（非 HTTPS）访问私有仓库

> **注意**：生产环境建议配置 TLS 证书，使用 HTTPS 访问仓库，避免镜像传输被中间人篡改。

重启 Docker 使配置生效：

```bash
systemctl restart docker
```

---

## 3. 推送和拉取镜像

整个流程分为三步：**拉取 → 打标签 → 推送**。

### 3.1 从 Docker Hub 拉取镜像

```bash
docker pull nginx
```

### 3.2 为镜像打上私有仓库标签

```bash
docker tag docker.io/nginx 10.10.13.7:5000/nginx
```

标签格式为 `<仓库地址>:<端口>/<镜像名>:<版本>`，未指定版本时默认为 `latest`。

### 3.3 推送到私有仓库

```bash
docker push 10.10.13.7:5000/nginx
```

### 3.4 验证推送结果

```bash
# 查看仓库中的所有镜像
curl http://10.10.13.7:5000/v2/_catalog
# 预期输出: {"repositories":["nginx"]}

# 查看某个镜像的所有标签
curl http://10.10.13.7:5000/v2/nginx/tags/list
# 预期输出: {"name":"nginx","tags":["latest"]}
```

### 3.5 从私有仓库拉取

在其他机器上（已配置 `insecure-registries`），可直接拉取：

```bash
docker pull 10.10.13.7:5000/nginx
```

---

## 常用运维命令

```bash
# 查看仓库所有镜像
curl http://10.10.13.7:5000/v2/_catalog

# 查看镜像标签列表
curl http://10.10.13.7:5000/v2/<镜像名>/tags/list

# 查看仓库存储占用
du -sh /mnt/ssd/nvme1/harbor/harbor

# 重启 Registry 容器
docker restart <container_id>
```

---

## 进阶建议

- **添加 TLS**：使用 `openssl` 生成自签名证书或通过 Let's Encrypt 获取证书，配置 HTTPS 访问
- **添加认证**：通过 `htpasswd` 配置基本认证，限制谁可以推送/拉取镜像
- **垃圾回收**：定期运行 `registry garbage-collect` 清理已删除镜像的磁盘空间
- **Web UI**：部署 [docker-registry-ui](https://github.com/Joxit/docker-registry-ui) 提供可视化管理界面
