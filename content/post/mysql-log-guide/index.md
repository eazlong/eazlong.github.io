---
title: "MySQL 四大日志详解：错误日志、Binlog、查询日志与慢查询日志"
description: "全面解析 MySQL 的四种日志类型——错误日志、二进制日志（Binlog）、查询日志和慢查询日志，涵盖配置方法、查看命令、清理策略和实战技巧。"
date: 2020-03-26
categories:
    - 技术文章
tags:
    - MySQL
    - 数据库
    - 运维
    - 日志
draft: false
---

## 日志类型概览

MySQL 提供四种日志，各有不同用途：

| 日志类型 | 记录内容 | 默认状态 | 主要用途 |
|----------|----------|:--------:|----------|
| **错误日志** | 启停信息、严重错误 | 开启 | 排查 MySQL 启动/运行故障 |
| **二进制日志（Binlog）** | DDL + DML（不含查询） | 关闭 | 主从复制、数据恢复 |
| **查询日志** | 客户端所有语句 | 关闭 | 调试、审计（生产环境慎用） |
| **慢查询日志** | 执行超时的 SQL | 关闭 | SQL 性能优化 |

**日志刷新命令**：

```bash
mysqladmin flush-logs   # 触发日志滚动，关闭并重新打开所有日志文件
```

---

## 1. 错误日志（Error Log）

错误日志是最重要的日志，记录 MySQL 的启动、停止信息以及运行过程中的严重错误。

### 配置

```ini
# my.cnf
[mysqld]
log_error = /var/log/mysql/error.log
```

或通过启动参数：

```bash
mysqld --log_error=/var/log/mysql/error.log
```

默认文件名为 `<hostname>.err`，存放在数据目录下。

### 查看

```bash
# 查看错误日志路径
mysql -e "SHOW VARIABLES LIKE 'log_error';"

# 查看最近的错误
tail -100 /var/log/mysql/error.log

# 过滤错误信息
grep -i "error\|warning" /var/log/mysql/error.log
```

### 常见排查场景

| 场景 | 关键字 |
|------|--------|
| MySQL 启动失败 | `[ERROR] Aborting` |
| 表损坏 | `Table is marked as crashed` |
| 内存不足 | `Out of memory` |
| 权限问题 | `Access denied` |
| 连接数超限 | `Too many connections` |

---

## 2. 二进制日志（Binary Log / Binlog）

Binlog 是 MySQL 最核心的日志，记录所有**修改数据的语句**（DDL + DML），但不记录 SELECT 和 SHOW 等查询语句。

### 核心用途

- **主从复制**：Slave 通过读取 Master 的 Binlog 来同步数据
- **数据恢复**：通过 `mysqlbinlog` 重放 Binlog 恢复误操作的数据

### 配置

```ini
# my.cnf
[mysqld]
log_bin = /var/log/mysql/mysql-bin    # 开启并指定文件前缀
server_id = 1                         # 主从复制必须设置
expire_logs_days = 7                  # 日志过期天数，自动清理
sync_binlog = 1                       # 每次事务提交都同步到磁盘（最安全）
```

### 查看

```bash
# 查看所有 Binlog 文件
mysql -e "SHOW BINARY LOGS;"

# 查看当前正在写入的 Binlog
mysql -e "SHOW MASTER STATUS;"

# 查看 Binlog 内容（可读格式）
mysqlbinlog mysql-bin.000001

# 按时间范围查看
mysqlbinlog --start-datetime="2024-03-26 00:00:00" \
            --stop-datetime="2024-03-26 23:59:59" \
            mysql-bin.000001

# 按位置查看
mysqlbinlog --start-position=154 --stop-position=1024 mysql-bin.000001
```

### 清理

Binlog 会持续增长，需要定期清理：

```sql
-- 删除所有 Binlog，重新从 000001 开始
RESET MASTER;

-- 删除指定编号之前的日志
PURGE MASTER LOGS TO 'mysql-bin.000010';

-- 删除指定时间之前的日志
PURGE MASTER LOGS BEFORE '2024-03-01 00:00:00';
```

> **推荐**：通过 `expire_logs_days` 参数自动清理，避免手动操作。

### 高级选项

| 参数 | 说明 |
|------|------|
| `binlog-do-db=db_name` | 只记录指定数据库的 Binlog |
| `binlog-ignore-db=db_name` | 忽略指定数据库的 Binlog |
| `sync-binlog=N` | 每写 N 次日志同步到磁盘（1 最安全，0 由 OS 决定） |
| `SQL_LOG_BIN=0` | SUPER 权限用户可临时禁止自己的操作写入 Binlog |

```sql
-- 当前会话禁用 Binlog（需要 SUPER 权限）
SET SQL_LOG_BIN = 0;

-- 执行不需要同步到从库的操作
ALTER TABLE big_table ENGINE=InnoDB;

-- 恢复
SET SQL_LOG_BIN = 1;
```

> **注意**：`sync-binlog=1` 配合 InnoDB 的 `innodb_flush_log_at_trx_commit=1` 使用，可实现最高的数据安全性（双 1 配置）。

---

## 3. 查询日志（General Log）

查询日志记录客户端发送的**所有语句**，包括 SELECT、SHOW 等，是最详尽的日志。

### 配置

```ini
# my.cnf
[mysqld]
general_log = 1                              # 开启
general_log_file = /var/log/mysql/query.log  # 日志文件路径
```

或通过启动参数：

```bash
mysqld --general-log --general-log-file=/var/log/mysql/query.log
```

也可以动态开关（无需重启）：

```sql
SET GLOBAL general_log = 'ON';
SET GLOBAL general_log = 'OFF';
```

### 使用建议

> **生产环境强烈建议关闭**。查询日志记录所有语句，会产生巨大的 I/O 开销和磁盘空间消耗。仅在以下场景临时开启：
> - 调试应用程序发送了哪些 SQL
> - 安全审计，排查异常操作
> - 排查连接和权限问题

---

## 4. 慢查询日志（Slow Query Log）

慢查询日志记录执行时间超过 `long_query_time`（默认 10 秒）的 SQL 语句，是**性能优化的核心工具**。

### 配置

```ini
# my.cnf
[mysqld]
slow_query_log = 1                                # 开启
slow_query_log_file = /var/log/mysql/slow.log     # 日志文件
long_query_time = 2                                # 超过 2 秒记录
log_slow_admin_statements = 1                      # 记录慢管理语句（ALTER、OPTIMIZE 等）
log_queries_not_using_indexes = 1                  # 记录未使用索引的查询
```

### 查看与分析

```bash
# 直接查看慢查询日志
tail -50 /var/log/mysql/slow.log

# 使用 mysqldumpslow 工具汇总分析（MySQL 自带）
# 按执行次数排序，显示前 10 条
mysqldumpslow -s c -t 10 /var/log/mysql/slow.log

# 按执行时间排序
mysqldumpslow -s t -t 10 /var/log/mysql/slow.log

# 按锁等待时间排序
mysqldumpslow -s l -t 10 /var/log/mysql/slow.log
```

### mysqldumpslow 常用参数

| 参数 | 说明 |
|------|------|
| `-s c` | 按执行次数排序（count） |
| `-s t` | 按总执行时间排序（time） |
| `-s l` | 按总锁等待时间排序（lock） |
| `-s r` | 按总返回行数排序（rows） |
| `-t N` | 显示前 N 条结果 |
| `-g pattern` | 按正则过滤 |

### 优化工作流

```
1. 开启慢查询日志
2. 运行一段时间，收集慢 SQL
3. 使用 mysqldumpslow 汇总分析
4. 针对 Top N 慢 SQL 执行 EXPLAIN
5. 添加索引 / 重写 SQL / 优化表结构
6. 验证优化效果
```

---

## 日志管理最佳实践

| 实践 | 说明 |
|------|------|
| 错误日志 | 始终开启，定期检查 |
| Binlog | 生产环境必须开启，设置合理过期天数 |
| 查询日志 | 生产环境关闭，仅临时调试时开启 |
| 慢查询日志 | 生产环境开启，`long_query_time` 设为 1-2 秒 |
| 日志轮转 | 配合 `logrotate` 或 `flush-logs` 定期滚动 |
| 磁盘监控 | 监控日志目录磁盘空间，防止写满导致 MySQL 宕机 |
