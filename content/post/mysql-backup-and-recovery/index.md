---
title: "MySQL 备份与恢复完整指南"
description: "系统整理 MySQL 的逻辑备份、物理备份、表导入导出等常用操作，涵盖 mysqldump、mysqlbinlog、mysqlhotcopy 等工具的使用方法和注意事项。"
date: 2020-03-19
categories:
    - 技术文章
tags:
    - MySQL
    - 数据库
    - 备份恢复
    - DevOps
draft: false
---

## 备份方式概览

| 备份方式 | 原理 | 适用场景 | 是否需要停机 |
|----------|------|----------|:------------:|
| 逻辑备份（mysqldump） | 导出 SQL 语句 | 数据量适中、跨版本迁移 | 否 |
| 冷备份 | 直接拷贝数据文件 | 完整备份、灾难恢复 | 是 |
| 热备份（MyISAM） | 锁表后拷贝文件 | MyISAM 引擎在线备份 | 否（但会锁表） |
| 热备份（InnoDB） | ibbackup/xtrabackup | InnoDB 引擎在线备份 | 否 |

---

## 1. 逻辑备份与恢复

逻辑备份通过 `mysqldump` 将数据导出为 SQL 语句文件，是最常用的备份方式。

### 1.1 备份

```bash
# InnoDB 引擎（推荐）：使用 --single-transaction 保证一致性快照
mysqldump -uroot -p --single-transaction dbname > backup.sql

# 备份单个表
mysqldump -uroot -p --single-transaction dbname tablename > table_backup.sql

# 备份所有数据库
mysqldump -uroot -p --single-transaction --all-databases > all_backup.sql
```

> **MyISAM 引擎**：不支持事务，需要加 `-l`（`--lock-tables`）锁定所有表来保证一致性：
> ```bash
> mysqldump -uroot -p -l dbname > backup.sql
> ```

### 1.2 恢复备份

```bash
mysql -uroot -p dbname < backup.sql
```

也可以在 MySQL 客户端内恢复：

```sql
mysql> source /path/to/backup.sql;
```

### 1.3 通过 binlog 恢复（增量恢复）

当需要恢复到特定时间点时，可以使用 binlog 重放：

```bash
# 重放整个 binlog 文件
mysqlbinlog binlog-file | mysql -uroot -p

# 恢复到指定时间点
mysqlbinlog --stop-datetime="2024-03-19 10:00:00" binlog-file | mysql -uroot -p

# 从指定位置开始恢复
mysqlbinlog --start-position=154 --stop-position=1024 binlog-file | mysql -uroot -p
```

> **最佳实践**：先用 `mysqldump` 做全量恢复，再用 `mysqlbinlog` 做增量恢复，覆盖全量备份之后到故障时刻的变更。

---

## 2. 物理备份与恢复

物理备份直接拷贝数据库的物理文件（数据文件、日志文件），速度远快于逻辑备份。

### 2.1 冷备份

最简单的物理备份方式——停止 MySQL 服务后直接拷贝数据目录：

```bash
# 1. 停止 MySQL
systemctl stop mysql

# 2. 拷贝数据目录
cp -r /var/lib/mysql /backup/mysql_$(date +%Y%m%d)

# 3. 恢复时反向操作
systemctl stop mysql
cp -r /backup/mysql_20240319/* /var/lib/mysql/
chown -R mysql:mysql /var/lib/mysql
systemctl start mysql
```

> **注意**：冷备份期间数据库完全不可用，适合维护窗口或非关键业务。

### 2.2 热备份

在不停止服务的情况下进行备份。

**MyISAM 引擎**：

```bash
# 方式一：mysqlhotcopy（锁表后拷贝文件）
mysqlhotcopy dbname /backup/

# 方式二：手工锁表
mysql -uroot -p -e "FLUSH TABLES WITH READ LOCK;"
cp -r /var/lib/mysql/dbname /backup/dbname_$(date +%Y%m%d)
mysql -uroot -p -e "UNLOCK TABLES;"
```

**InnoDB 引擎**：

推荐使用 Percona XtraBackup（ibbackup 的开源替代）：

```bash
# 全量备份
xtrabackup --backup --target-dir=/backup/full

# 准备恢复（应用 redo log）
xtrabackup --prepare --target-dir=/backup/full

# 恢复
systemctl stop mysql
xtrabackup --copy-back --target-dir=/backup/full
chown -R mysql:mysql /var/lib/mysql
systemctl start mysql
```

---

## 3. 表的导入与导出

适用于在不同数据库或系统之间迁移单表数据。

### 3.1 导出

**SQL 方式**（导出为文本文件）：

```sql
SELECT * FROM tablename INTO OUTFILE '/tmp/tablename.txt'
  FIELDS TERMINATED BY ','
  LINES TERMINATED BY '\n';
```

> **注意**：`INTO OUTFILE` 要求 MySQL 对目标目录有写入权限，且文件不能已存在。

**命令行方式**：

```bash
mysqldump -u root -p -T /tmp dbname tablename \
  --fields-terminated-by=',' \
  --lines-terminated-by='\n'
```

### 3.2 导入

**SQL 方式**：

```sql
LOAD DATA LOCAL INFILE '/tmp/tablename.txt' INTO TABLE tablename
  FIELDS TERMINATED BY ','
  LINES TERMINATED BY '\n';
```

**命令行方式**：

```bash
mysqlimport -u root -p --local dbname /tmp/tablename.txt \
  --fields-terminated-by=',' \
  --lines-terminated-by='\n'
```

---

## 4. 备份策略建议

| 策略 | 适用场景 | 方案 |
|------|----------|------|
| 小型数据库（< 10GB） | 开发/测试环境 | 每日 `mysqldump` 全量备份 |
| 中型数据库（10-100GB） | 生产环境 | 每周全量 + 每日增量（binlog） |
| 大型数据库（> 100GB） | 高可用生产环境 | XtraBackup 全量 + binlog 增量 + 主从复制 |

**通用建议**：
- 定期验证备份可用性（实际恢复测试）
- 备份文件存储在与源数据库不同的物理机器上
- 开启 binlog 并设置合理的过期时间（`expire_logs_days`）
- 备份脚本加入 crontab 自动执行
