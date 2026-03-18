---
title: "Linux 线程调度算法详解：SCHED_OTHER、SCHED_FIFO 与 SCHED_RR"
description: "深入讲解 Linux 的三种线程调度策略——分时调度 SCHED_OTHER、实时先到先服务 SCHED_FIFO 和实时时间片轮询 SCHED_RR，涵盖调度原理、优先级机制和 POSIX API 代码示例。"
date: 2020-03-31
categories:
    - 技术文章
tags:
    - Linux
    - 操作系统
    - 线程
    - 调度算法
draft: false
---

## 三种调度策略概览

Linux 提供三种线程调度策略，分为**分时调度**和**实时调度**两大类：

| 调度策略 | 类型 | 优先级参数 | 调度方式 | 适用场景 |
|----------|------|-----------|----------|----------|
| `SCHED_OTHER` | 分时调度 | nice (-20 ~ 19) | 时间片轮转，按动态优先级 | 普通应用程序（默认） |
| `SCHED_FIFO` | 实时调度 | rt_priority (1 ~ 99) | 先到先服务，不限时间 | 对延迟极敏感的任务 |
| `SCHED_RR` | 实时调度 | rt_priority (1 ~ 99) | 时间片轮询 | 实时任务 + 公平性需求 |

> **核心规则**：实时调度（FIFO/RR）的优先级**始终高于**分时调度（OTHER）。只要有实时线程可运行，分时线程就不会被调度。

---

## 1. SCHED_OTHER — 分时调度

`SCHED_OTHER` 是 Linux 的默认调度策略，适用于绝大多数普通应用程序。

### 调度原理

1. 每个线程有一个 **nice 值**（范围 -20 ~ 19，默认 0）
2. 内核根据 nice 值计算出**时间片**（counter）：nice 值越低，优先级越高，时间片越长
3. 调度器循环选择 `(counter + 20) - nice` 值最大的线程运行
4. 时间片耗尽后，线程释放 CPU 并被放到就绪队列尾部

```
nice 值越小 → 优先级越高 → 时间片越长 → 获得更多 CPU 时间
nice 值越大 → 优先级越低 → 时间片越短 → 获得更少 CPU 时间
```

### nice 值速查

| nice 值 | 优先级 | 含义 |
|:-------:|:------:|------|
| -20 | 最高 | 占用最多 CPU 时间 |
| 0 | 默认 | 标准优先级 |
| 19 | 最低 | 占用最少 CPU 时间 |

### 命令行操作

```bash
# 查看进程的 nice 值
ps -eo pid,ni,comm

# 以指定 nice 值启动进程
nice -n 10 ./my_program

# 修改运行中进程的 nice 值（需要 root 权限来降低 nice 值）
renice -n -5 -p <pid>
```

---

## 2. SCHED_FIFO — 实时先到先服务

`SCHED_FIFO`（First In First Out）是实时调度策略，线程一旦获得 CPU 就会一直运行，直到以下情况之一发生：

1. 线程主动让出 CPU（`sched_yield`、阻塞 I/O、sleep 等）
2. 更高优先级的实时线程到达
3. 线程结束

### 调度原理

```
rt_priority = 50 的线程 A 正在运行
                ↓
rt_priority = 80 的线程 B 就绪
                ↓
线程 A 被抢占，线程 B 立即运行
                ↓
线程 B 完成/阻塞 → 线程 A 继续运行
```

### 特点

- 同优先级的线程按**先来先服务**排列，不会互相抢占
- 高优先级线程可以**抢占**低优先级线程
- **没有时间片限制**——线程不会因为运行太久而被强制切换

> **风险**：如果一个 SCHED_FIFO 线程进入死循环，会饿死所有同优先级和低优先级的线程。使用时务必谨慎。

---

## 3. SCHED_RR — 实时时间片轮询

`SCHED_RR`（Round Robin）在 SCHED_FIFO 的基础上增加了**时间片轮转**，解决了同优先级线程的公平性问题。

### 调度原理

1. 设置 `rt_priority`（1-99）和 `nice`（-20 ~ 19）
2. 内核根据 nice 值计算出时间片
3. 优先运行 `rt_priority` 最高的线程
4. 时间片耗尽后，线程被放到同优先级队列尾部
5. 同优先级的线程轮流获得 CPU 时间

### 与 SCHED_FIFO 的对比

| 对比 | SCHED_FIFO | SCHED_RR |
|------|-----------|----------|
| 同优先级调度 | 先来先服务，不抢占 | 时间片轮询，轮流运行 |
| 高优先级抢占 | 支持 | 支持 |
| 时间片 | 无（运行到完成或让出） | 有（耗尽后排队） |
| 饥饿风险 | 高（可能饿死同级线程） | 低（同级线程公平） |
| 适用场景 | 严格优先级，延迟敏感 | 实时 + 公平性需求 |

---

## 4. 优先级体系

Linux 的完整优先级体系如下：

```
优先级从高到低：

┌─────────────────────────────────┐
│  SCHED_FIFO / SCHED_RR          │  rt_priority: 99（最高）
│  实时调度                        │  rt_priority: 1（最低）
├─────────────────────────────────┤
│  SCHED_OTHER                     │  nice: -20（最高）
│  分时调度                        │  nice: 19（最低）
└─────────────────────────────────┘

实时线程（rt_priority=1）仍然高于所有分时线程（nice=-20）
```

---

## 5. 代码示例

### 5.1 设置线程调度策略（POSIX API）

```c
#include <pthread.h>
#include <sched.h>

void *thread_func(void *arg) {
    // 实时线程任务
    return NULL;
}

int main() {
    pthread_t tid;
    pthread_attr_t attr;
    struct sched_param param;

    // 初始化线程属性
    pthread_attr_init(&attr);

    // 设置调度策略为 SCHED_RR
    pthread_attr_setschedpolicy(&attr, SCHED_RR);

    // 设置实时优先级（1-99）
    param.sched_priority = 10;
    pthread_attr_setschedparam(&attr, &param);

    // 必须设置为 PTHREAD_EXPLICIT_SCHED，否则继承父线程的调度策略
    pthread_attr_setinheritsched(&attr, PTHREAD_EXPLICIT_SCHED);

    // 创建线程
    pthread_create(&tid, &attr, thread_func, NULL);

    // 清理
    pthread_attr_destroy(&attr);
    pthread_join(tid, NULL);

    return 0;
}
```

编译（需要链接 pthread 库）：

```bash
gcc -o rt_thread rt_thread.c -lpthread
# 运行需要 root 权限（实时调度策略需要 CAP_SYS_NICE）
sudo ./rt_thread
```

### 5.2 查询和修改运行中线程的调度策略

```c
#include <sched.h>

// 查询当前线程的调度策略
int policy = sched_getscheduler(0);  // 0 表示当前线程

// 修改调度策略
struct sched_param param;
param.sched_priority = 50;
sched_setscheduler(0, SCHED_FIFO, &param);

// 查询时间片长度（仅对 SCHED_RR 有意义）
struct timespec ts;
sched_rr_get_interval(0, &ts);
printf("RR 时间片: %ld ms\n", ts.tv_nsec / 1000000);
```

### 5.3 命令行工具

```bash
# 以实时 FIFO 策略运行程序（优先级 50）
sudo chrt -f 50 ./my_program

# 以实时 RR 策略运行程序（优先级 10）
sudo chrt -r 10 ./my_program

# 查看进程的调度策略和优先级
chrt -p <pid>

# 修改运行中进程的调度策略
sudo chrt -f -p 50 <pid>
```

---

## 选型建议

| 场景 | 推荐策略 | 理由 |
|------|----------|------|
| Web 服务、普通应用 | `SCHED_OTHER` | 默认策略，无需特殊权限 |
| 音视频采集/播放 | `SCHED_RR` | 需要实时性，同时保证公平 |
| 硬件中断处理、工控 | `SCHED_FIFO` | 最低延迟，严格优先级 |
| 数据库后台线程 | `SCHED_OTHER` + 低 nice | 避免抢占前台查询 |

> **注意**：滥用实时调度策略可能导致系统卡死（实时线程占满 CPU，普通进程无法运行）。Linux 提供了 `/proc/sys/kernel/sched_rt_runtime_us` 参数来限制实时线程最多占用的 CPU 时间比例（默认 95%）。
