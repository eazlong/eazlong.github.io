---
title: "RTS 实时流媒体环境搭建指南"
description: "详细介绍在 Linux 系统中搭建 RTS (Real-Time Streaming) 实时流媒体环境所需的依赖库安装步骤，包括 Speex、RTMPDump、libcurl、OGG 和 MySQL"
date: 2020-08-28
categories:
    - 技术文章
tags:
    - RTS
    - 流媒体
    - Speex
    - RTMP
    - FFmpeg
    - Linux
draft: false
---

## 概述

RTS (Real-Time Streaming) 实时流媒体技术广泛应用于直播、视频会议、在线教育等场景。本文详细介绍在 Linux 系统中搭建 RTS 环境所需的各个依赖库的编译安装步骤。

> **前置条件**：本文基于 CentOS/RHEL 系统，部分命令可能需要根据您的发行版调整。

## 环境架构

```
┌─────────────────────────────────────────────────────────┐
│                    RTS 流媒体服务器                       │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐ │
│  │ Speex   │   │ RTMP    │   │ libcurl │   │  OGG    │ │
│  │ 音频处理│   │ 协议    │   │ HTTP库  │   │ 容器格式│ │
│  └─────────┘   └─────────┘   └─────────┘   └─────────┘ │
│                                                         │
│  ┌─────────┐   ┌─────────┐                            │
│  │ MySQL   │   │ OpenSSL │                            │
│  │ 数据存储│   │ 加密库  │                            │
│  └─────────┘   └─────────┘                            │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

## 安装前准备

首先安装必要的编译工具和开发库：

```bash
# 安装基础编译工具
yum install gcc gcc-c++ make

# 安装 Git
yum install git

# 安装 libglib 开发库
yum install libglib2.0-dev

# 安装 Autotools 工具链
yum install autoconf
yum install automake
yum install libtool

# 安装 OpenSSL 开发库
yum install openssl-devel
# Debian/Ubuntu 系统使用:
# sudo apt-get install openssl
# sudo apt-get install libssl-dev
```

> **提示**：不同 Linux 发行版的包管理器命令不同，CentOS/RHEL 使用 yum，Debian/Ubuntu 使用 apt-get。

## 第一步：安装 Speex

Speex 是一款开源的音频编解码器，支持语音优化，广泛用于 VoIP 和流媒体应用。

### 1.1 克隆 Speex 源码

```bash
git clone http://git.xiph.org/speex.git
cd speex
```

### 1.2 编译安装 Speex

```bash
# 生成 configure 脚本
./autogen.sh

# 配置编译选项
./configure

# 编译并安装
make && make install
```

### 1.3 安装 SpeexDSP

SpeexDSP 是 Speex 的数字信号处理部分：

```bash
# 返回上级目录
cd ../

# 克隆 SpeexDSP
git clone http://git.xiph.org/speexdsp.git
cd speexdsp

# 编译安装
./autogen.sh
./configure
make && make install
```

## 第二步：安装 RTMPDump

RTMP (Real-Time Messaging Protocol) 是 Adobe 开发的流媒体协议，RTMPDump 是其开源实现。

### 2.1 克隆并编译 RTMPDump

```bash
# 克隆源码
git clone git://git.ffmpeg.org/rtmpdump

cd rtmpdump

# 编译（不使用 SSL 加密）
make CRYPTO=
make install

# 复制头文件
cp librtmp/rtmp_sys.h /usr/include/
```

> **注意**：如果需要 SSL 支持，需要先安装 OpenSSL 并使用 `make` 而不是 `make CRYPTO=`。

## 第三步：安装 libcurl

libcurl 是一个强大的 URL 传输库，支持多种协议，是流媒体应用的重要依赖。

### 3.1 克隆 libcurl 源码

```bash
git clone https://github.com/curl/curl.git
cd curl
```

### 3.2 编译安装

```bash
# 生成 configure 脚本
./buildconf

# 配置编译选项
./configure

# 解决可能的库链接问题
ln -sf /usr/lib64/librtmp.so /usr/local/lib/librtmp.so.1

# 编译并安装
make && make install
```

> **常见问题**：如果出现找不到 librtmp 的错误，执行上述 ln 命令创建软链接。

## 第四步：安装 libogg

OGG 是一种开源的多媒体容器格式，Vorbis 和 Speex 等音频编码都使用 OGG 作为容器。

### 4.1 下载 libogg

```bash
# 返回上级目录
cd ../

# 下载 libogg
wget http://downloads.xiph.org/releases/ogg/libogg-1.3.2.tar.gz

# 解压
tar -zxf libogg-1.3.2.tar.gz
cd libogg-1.3.2
```

### 4.2 编译安装

```bash
./configure
make && make install
```

## 第五步：安装 MySQL

MySQL 用于存储流媒体服务的元数据、用户信息等。

### 5.1 安装 MySQL 服务器

```bash
# CentOS/RHEL
yum install mysql-server mysql-client
yum install mysql-devel

# Debian/Ubuntu
sudo apt-get install mysql-server mysql-client
sudo apt-get install libmysqlclient-dev
```

### 5.2 配置动态库路径

如果系统找不到 MySQL 动态库，需要配置链接器：

```bash
# 添加库路径
echo "/usr/local/lib" >> /etc/ld.so.conf

# 更新动态链接器缓存
ldconfig
```

> **最佳实践**：生产环境中建议使用单独的数据库服务器，并根据实际需求进行性能调优。

## 常见问题汇总

### 动态库找不到

```bash
# 创建软链接
ln -s /path/to/source /path/to/target

# 更新库缓存
ldconfig
```

### 编译错误

- 确保安装了所有依赖的开发库
- 检查是否有残留的编译文件，尝试 `make clean`

### 运行时库路径

如果程序运行时提示找不到库：

```bash
# 临时设置库路径
export LD_LIBRARY_PATH=/usr/local/lib:$LD_LIBRARY_PATH

# 永久生效，添加到 /etc/ld.so.conf.d/ 目录下
echo "/usr/local/lib" | sudo tee /etc/ld.so.conf.d/local.conf
sudo ldconfig
```

## 辅助工具

### URL 编码转换

在线工具：http://tool.chinaz.com/Tools/URLEncode.aspx

### 验证安装

```bash
# 检查库是否安装
ldconfig -p | grep -E "(speex|rtmp|curl|ogg|mysql)"

# 检查可执行文件
which ffmpeg  # 如果已安装 FFmpeg
```

## 总结

本文详细介绍了 RTS 实时流媒体环境的完整搭建流程：

| 组件 | 用途 | 安装方式 |
|------|------|---------|
| Speex | 音频编解码 | 源码编译 |
| RTMPDump | RTMP 协议支持 | 源码编译 |
| libcurl | HTTP 传输 | 源码编译 |
| libogg | OGG 容器格式 | 源码编译 |
| MySQL | 数据存储 | 包管理器/源码 |

完成以上步骤后，您就可以在此基础上部署 FFmpeg、Nginx-rtmp-module 等流媒体服务了。

---

**参考来源**：有道笔记导出