---
title: "Linux 进程间通信（IPC）六大机制详解"
description: "系统梳理 Linux 进程间通信的六种方式：管道、信号、消息队列、共享内存、信号量和套接字，涵盖原理、用法和代码示例。"
date: 2020-03-23
categories:
    - 技术文章
tags:
    - Linux
    - IPC
    - 操作系统
    - C语言
draft: false
---

## 概览

Linux 提供了六种主要的进程间通信（IPC）机制：

| 机制 | 数据流向 | 适用范围 | 特点 |
|------|----------|----------|------|
| **管道（Pipe）** | 单向 | 父子/兄弟进程 | 最简单，半双工 |
| **有名管道（FIFO）** | 双向 | 任意进程 | 通过文件系统路径标识 |
| **信号（Signal）** | 异步通知 | 任意进程 | 软件层面的中断机制 |
| **消息队列** | 有格式消息 | 任意进程 | 按消息类型收发 |
| **共享内存** | 直接读写 | 任意进程 | 速度最快，需配合同步机制 |
| **信号量（Semaphore）** | 同步控制 | 线程/进程 | 不传数据，只做同步 |
| **套接字（Socket）** | 双向字节流 | 本机/跨网络 | 最通用，支持网络通信 |

---

## 1. 管道（Pipe）

管道是最基本的 IPC 方式，创建后在内核中分配一块缓冲区（通常一个内存页面，4KB），以**环形缓冲区**方式使用。

### 特点

- **半双工**：数据只能单向流动
- **仅限亲缘进程**：只能在父子进程或共享祖先的进程之间使用
- **不属于任何文件系统**：由内核直接管理，是一种特殊文件

### 用法

```c
#include <unistd.h>

int fd[2];
pipe(fd);  // fd[0] 读端, fd[1] 写端

pid_t pid = fork();

if (pid > 0) {
    // 父进程：写入数据
    close(fd[0]);                    // 关闭读端
    write(fd[1], "hello", 5);
    close(fd[1]);
} else if (pid == 0) {
    // 子进程：读取数据
    close(fd[1]);                    // 关闭写端
    char buf[64];
    int n = read(fd[0], buf, sizeof(buf));
    buf[n] = '\0';
    printf("收到: %s\n", buf);       // 输出: 收到: hello
    close(fd[0]);
}
```

> **提示**：可以用 `fcntl` 将管道设置为非阻塞模式，避免 `read` 在无数据时阻塞。

---

## 2. 有名管道（FIFO）

有名管道解决了匿名管道只能用于亲缘进程的限制，任意进程只要知道 FIFO 文件的路径就能通信。

### 特点

- **全双工通信**，数据先进先出
- 可用于**任意进程**之间，通过指定相同的管道文件路径
- 文件名存在于文件系统中，但**数据保存在内存**中（文件大小始终为 0）
- 通过标准的 `open`、`read`、`write` 系统调用操作

### 用法

```c
#include <sys/stat.h>

// 创建有名管道
mkfifo("/tmp/myfifo", 0666);

// 写端进程
int fd = open("/tmp/myfifo", O_WRONLY);
write(fd, "hello fifo", 10);
close(fd);

// 读端进程
int fd = open("/tmp/myfifo", O_RDONLY);
char buf[64];
read(fd, buf, sizeof(buf));
close(fd);
```

### 注意事项

- 以只读或只写模式 `open` 时，如果对端尚未打开，会**阻塞**等待
- 读写出错时，内核会向进程发送 `SIGPIPE` 信号
- 每个管道使用一个内存页面（4KB）作为环形缓冲区

---

## 3. 信号（Signal）

信号是软件层面的**异步中断机制**，用于通知进程某个事件已发生。

### 常见信号

| 信号 | 编号 | 默认行为 | 说明 |
|------|:----:|----------|------|
| `SIGINT` | 2 | 终止 | Ctrl+C 产生 |
| `SIGKILL` | 9 | 终止 | 不可捕获，强制杀进程 |
| `SIGTERM` | 15 | 终止 | 优雅终止，可捕获处理 |
| `SIGPIPE` | 13 | 终止 | 向已关闭的管道/Socket 写入 |
| `SIGCHLD` | 17 | 忽略 | 子进程状态改变 |
| `SIGUSR1` | 10 | 终止 | 用户自定义信号 1 |
| `SIGUSR2` | 12 | 终止 | 用户自定义信号 2 |

### 用法

```c
#include <signal.h>

// 注册信号处理函数
void handler(int signo) {
    printf("收到信号: %d\n", signo);
}

signal(SIGUSR1, handler);

// 发送信号
kill(pid, SIGUSR1);

// 更可靠的方式：sigaction
struct sigaction sa;
sa.sa_handler = handler;
sigemptyset(&sa.sa_mask);
sa.sa_flags = 0;
sigaction(SIGUSR1, &sa, NULL);
```

> **最佳实践**：优先使用 `sigaction` 而非 `signal`，前者行为在不同平台上更一致。

---

## 4. 消息队列（Message Queue）

消息队列允许进程发送和接收带有**类型标识**的消息，可以按类型有选择地读取。

### 用法

```c
#include <sys/msg.h>

// 消息结构
struct msgbuf {
    long mtype;        // 消息类型（必须 > 0）
    char mtext[256];   // 消息内容
};

// 创建消息队列
key_t key = ftok("/tmp", 'A');
int msqid = msgget(key, IPC_CREAT | 0666);

// 发送消息
struct msgbuf msg = {1, "hello"};
msgsnd(msqid, &msg, strlen(msg.mtext), 0);

// 接收消息（type=1 的消息）
struct msgbuf recv;
msgrcv(msqid, &recv, sizeof(recv.mtext), 1, 0);

// 删除消息队列
msgctl(msqid, IPC_RMID, NULL);
```

### 与管道的对比

| 对比 | 管道 | 消息队列 |
|------|------|----------|
| 数据格式 | 无结构字节流 | 有类型的消息 |
| 读取方式 | 只能顺序读取 | 可按类型选择性读取 |
| 生命周期 | 随进程结束而销毁 | 独立于进程，需显式删除 |

---

## 5. 共享内存（Shared Memory）

共享内存是**速度最快**的 IPC 方式——多个进程直接映射同一块物理内存，数据不需要在内核和用户空间之间拷贝。

### 用法

```c
#include <sys/shm.h>

// 创建共享内存
key_t key = ftok("/tmp", 'B');
int shmid = shmget(key, 4096, IPC_CREAT | 0666);

// 映射到进程地址空间
char *addr = (char *)shmat(shmid, NULL, 0);

// 写入数据
strcpy(addr, "shared data");

// 读取数据（另一个进程）
printf("%s\n", addr);  // 输出: shared data

// 解除映射
shmdt(addr);

// 删除共享内存
shmctl(shmid, IPC_RMID, NULL);
```

> **重要**：共享内存本身没有同步机制，多个进程同时读写会产生竞态条件。通常需要配合**信号量**或**互斥锁**使用。

---

## 6. 信号量（Semaphore）

信号量不传递数据，而是用于**同步和互斥控制**，通常与共享内存配合使用。

### 6.1 POSIX 信号量

定义在 `<semaphore.h>` 中，分为两种：

**无名信号量**：用于线程间或父子进程间同步，必须共享变量。

```c
#include <semaphore.h>

sem_t sem;

// 初始化（第二个参数：0=线程间，非0=进程间；第三个参数：初始值）
sem_init(&sem, 0, 1);

sem_wait(&sem);     // P 操作：值 -1，若为 0 则阻塞
// ... 临界区 ...
sem_post(&sem);     // V 操作：值 +1，唤醒等待者

int val;
sem_getvalue(&sem, &val);  // 获取当前值

sem_destroy(&sem);  // 销毁
```

**有名信号量**：可用于**无关进程**之间同步，创建于 `/dev/shm` 目录下。

```c
// 创建/打开有名信号量
sem_t *sem = sem_open("/my_sem", O_CREAT, 0666, 1);

sem_wait(sem);
// ... 临界区 ...
sem_post(sem);

sem_close(sem);             // 关闭
sem_unlink("/my_sem");      // 从文件系统中删除
```

> 被保护的资源也需要位于共享内存区，这样才能被无关进程共享。

### 6.2 System V 信号量

定义在 `<sys/sem.h>` 中，特点是**信号量集合**（可同时操作多个信号量）。

```c
#include <sys/sem.h>

// 创建信号量集（包含 1 个信号量）
key_t key = ftok("/tmp", 'C');
int semid = semget(key, 1, IPC_CREAT | 0666);

// 初始化（设置第 0 个信号量的值为 1）
union semun {
    int val;
} arg;
arg.val = 1;
semctl(semid, 0, SETVAL, arg);

// P 操作
struct sembuf sop = {0, -1, 0};  // 第 0 个信号量，值 -1
semop(semid, &sop, 1);

// V 操作
struct sembuf sop_v = {0, 1, 0}; // 第 0 个信号量，值 +1
semop(semid, &sop_v, 1);

// 删除
semctl(semid, 0, IPC_RMID);
```

### 6.3 IPC Key 的获取方式

无关进程需要通过相同的 key 来访问同一个 IPC 对象：

| 方式 | 说明 |
|------|------|
| `IPC_PRIVATE` | 内核自动分配，需通过其他方式（文件、管道）传递给其他进程 |
| 约定固定值 | 在代码中硬编码双方认可的 key |
| `ftok(path, id)` | 通过同一个文件路径 + 项目 ID（0-255）生成 key，最常用 |

---

## 7. 套接字（Socket）

Socket 是最通用的 IPC 机制，既可用于本机进程通信，也可跨网络通信。

### Unix Domain Socket

用于同一台机器上的进程通信，性能优于 TCP/IP Socket（无需经过网络协议栈）：

```c
#include <sys/socket.h>
#include <sys/un.h>

// 服务端
int sockfd = socket(AF_UNIX, SOCK_STREAM, 0);

struct sockaddr_un addr;
addr.sun_family = AF_UNIX;
strcpy(addr.sun_path, "/tmp/my.sock");

bind(sockfd, (struct sockaddr *)&addr, sizeof(addr));
listen(sockfd, 5);
int clientfd = accept(sockfd, NULL, NULL);

char buf[256];
read(clientfd, buf, sizeof(buf));
```

### 与其他 IPC 的对比

| 对比 | Unix Socket | 共享内存 | 管道 |
|------|-------------|----------|------|
| 速度 | 快 | 最快 | 快 |
| 数据格式 | 字节流/数据报 | 自定义 | 字节流 |
| 适用范围 | 本机/跨网络 | 仅本机 | 亲缘进程 |
| 同步机制 | 内置 | 需额外实现 | 内置 |
| 编程复杂度 | 中等 | 较高 | 简单 |

---

## 选型建议

| 场景 | 推荐方案 |
|------|----------|
| 父子进程简单通信 | 管道（Pipe） |
| 无关进程单向数据传输 | 有名管道（FIFO） |
| 异步事件通知 | 信号（Signal） |
| 带类型的消息传递 | 消息队列 |
| 大量数据高速共享 | 共享内存 + 信号量 |
| 本机或跨网络通信 | Socket |
