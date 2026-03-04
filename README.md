# RDS MySQL 监控方案

## 环境

| 项 | 值 |
|----|---|
| RDS | database-1, MySQL 8.4.7, db.m7g.large (2vCPU/8GB), 100GB gp3, Multi-AZ |
| Region | us-east-1, VPC vpc-7d049706 |
| 跳板机 | i-0ecf0ebb4ec863f9e (m6i.xlarge, 同 VPC, SSM) |
| 参数组 | mysql84-monitor (自定义, 已应用) |

---

## 1. 架构

```
YACE ─── CloudWatch API ────► Prometheus    (18 项, PI 发布到 CloudWatch 的指标, 每 60s)
mysqld_exporter ── MySQL 3306 ──► Prometheus    (5 项, CloudWatch 缺失的指标, 每 15s)
```

### 关键发现：PI 开启后自动发布 counter 到 CloudWatch

开启 Performance Insights 后，RDS 自动将大量 MySQL 引擎 counter 指标发布到 **CloudWatch Metrics（`AWS/RDS` 命名空间）**。实测 database-1 上 CloudWatch 可用指标从原来的约 20 个增加到 **135 个**，包括 `Queries`、`ThreadsRunning`、`SlowQueries`、`InnoDBBufferPoolHitRate`、`TableLocksWaited` 等。

这些指标直接在 CloudWatch 中，YACE 通过 `GetMetricData` 即可采集，不需要调用 PI 自身的 API。

### 23 项指标映射

| # | 指标 | 来源 | CloudWatch 指标名 |
|---|------|------|-------------------|
| 1 | CPU利用率 | YACE | `CPUUtilization` |
| 2 | 内存利用率 | YACE | `FreeableMemory` (Grafana 计算百分比) |
| 3 | 磁盘利用率 | YACE | `FreeStorageSpace` 或 `FileSysDiskSpaceUtilization` |
| 4 | QPS | YACE | `Queries` |
| 5 | TPS | **exporter** | `Handler_commit` + `Handler_rollback` (CloudWatch 无 commit/rollback 计数) |
| 6 | 连接数 | YACE | `ThreadsConnected` |
| 7 | 运行线程 | YACE | `ThreadsRunning` |
| 8 | 创建线程 | YACE | `ThreadsCreated` |
| 9 | 慢查询 | YACE | `SlowQueries` |
| 10 | Com_select | YACE | `SelectCommands` |
| 10 | Com_insert/update/delete | YACE (替代) | `InnoDBRowsInserted` / `InnoDBRowsUpdated` / `InnoDBRowsDeleted` (InnoDB 行级, 非 SQL 命令级) |
| 11 | InnoDB缓存命中率 | YACE | `InnoDBBufferPoolHitRate` |
| 12 | InnoDB缓存使用率 | YACE | `InnoDBBufferPoolUtilization` |
| 13 | InnoDB读磁盘 | YACE | `InnoDBBufferPoolReads` |
| 14 | InnoDB写磁盘 | YACE | `InnoDBDataWrites` |
| 15 | InnoDB fsync | **exporter** | `Innodb_data_fsyncs` (CloudWatch 无此指标) |
| 16 | 等待表锁 | YACE | `TableLocksWaited` |
| 17 | 立即表锁 | YACE | `TableLocksImmediate` |
| 18 | InnoDB行锁 | YACE | `InnoDBRowLockWaits` / `InnoDBRowLockTime` |
| 19 | 行锁平均时间 | YACE (计算) | Grafana: `InnoDBRowLockTime / InnoDBRowLockWaits` |
| 20 | 主从延迟距离 | **exporter** | `SHOW REPLICA STATUS` (CloudWatch 无 bytes 级延迟) * |
| 21 | 主从延迟时间 | **exporter** | `Seconds_Behind_Master` * |
| 22 | SlaveSqlRunning | **exporter** | `SHOW REPLICA STATUS` * |
| 23 | SlaveIoRunning | **exporter** | `SHOW REPLICA STATUS` * |

\* 需创建只读副本后生效

**汇总**: YACE 覆盖 18 项, mysqld_exporter 补齐 5 项 (#5 TPS, #15 fsync, #20-23 复制)

### MySQL 8.4 注意

`Com_*` 状态变量在 MySQL 8.4 中已移除。影响：
- **TPS**: 用 `Handler_commit` + `Handler_rollback`（mysqld_exporter 的 `global_status` 中仍有）
- **各类 SQL 次数**: CloudWatch 有 `SelectCommands` + `InnoDBRows*` 替代；mysqld_exporter 可通过 `perf_schema.eventsstatements` 获取精确的 `statement/sql/*` 计数

---

## 2. RDS 侧配置

### 2.1 Performance Insights — 必须开启

PI 开启后 RDS 自动将 MySQL 引擎 counter 发布到 CloudWatch（`AWS/RDS` 命名空间），YACE 可直接采集。

```bash
aws rds modify-db-instance --db-instance-identifier database-1 \
  --enable-performance-insights --performance-insights-retention-period 7 \
  --apply-immediately --region us-east-1
```

### 2.2 参数组 `mysql84-monitor`

| 参数 | 值 | 类型 | 用途 |
|------|---|------|------|
| `slow_query_log` | `1` | 动态 | 慢查询 #9 |
| `long_query_time` | `1` | 动态 | 慢查询阈值 (秒) |
| `innodb_monitor_enable` | `all` | 动态 | InnoDB 详细计数器 |
| `performance_schema` | `ON` (默认) | 静态 | PI 和 perf_schema collectors |

### 2.3 创建 exporter 用户 (补齐 5 项)

```sql
CREATE USER IF NOT EXISTS 'exporter'@'%'
  IDENTIFIED BY 'YOUR_EXPORTER_PASSWORD'
  WITH MAX_USER_CONNECTIONS 3;

GRANT PROCESS ON *.* TO 'exporter'@'%';
GRANT REPLICATION CLIENT ON *.* TO 'exporter'@'%';
GRANT SELECT ON performance_schema.* TO 'exporter'@'%';
FLUSH PRIVILEGES;
```

### 2.4 安全组

RDS 安全组入站规则: TCP 3306 ← exporter 所在主机 (同 VPC)

### 2.5 YACE 所需 IAM 权限

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "cloudwatch:GetMetricData",
      "cloudwatch:ListMetrics",
      "tag:GetResources",
      "rds:DescribeDBInstances",
      "rds:ListTagsForResource"
    ],
    "Resource": "*"
  }]
}
```

---

## 3. 部署

### 3.1 准备

```bash
# 下载 RDS CA 证书 (SSL 连接)
curl -o global-bundle.pem https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem

# 配置环境变量
cp .env.example .env
# 编辑 .env 填入 AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, MYSQL_EXPORTER_PASSWORD
```

### 3.2 启动

```bash
docker-compose up -d
```

### 3.3 验证

```bash
# mysqld_exporter
curl -s http://localhost:9104/metrics | grep mysql_up
# 预期: mysql_up 1

# YACE
curl -s http://localhost:5000/metrics | grep aws_rds
# 预期: aws_rds_queries_average{...}, aws_rds_threads_running_average{...} 等

# Prometheus
curl -s http://localhost:9090/api/v1/targets | python3 -m json.tool
# 预期: 两个 target 均为 "up"
```

---

## 4. 配置文件

### 4.1 docker-compose.yaml

见 [`docker-compose.yaml`](docker-compose.yaml)

mysqld_exporter 关键注意:
- `--no-collect.info_schema.query_response_time` — **必须禁用**, RDS 上报错 (Percona 专有)
- `--exporter.lock_wait_timeout=2` — 防止被 DDL 阻塞

### 4.2 yace-config.yaml

见 [`yace-config.yaml`](yace-config.yaml) — 18 项 CloudWatch 指标 (含 PI 发布的 counter)

### 4.3 其他配置

- [`mysqld-exporter.cnf`](mysqld-exporter.cnf) — MySQL 连接 + SSL
- [`prometheus.yaml`](prometheus.yaml) — Prometheus scrape 配置
- [`.env.example`](.env.example) — 环境变量模板

---

## 5. Grafana PromQL

### YACE 指标 (CloudWatch)

```promql
# #1 CPU利用率
aws_rds_cpuutilization_average{dimension_DBInstanceIdentifier="database-1"}

# #2 内存利用率 (db.m7g.large = 8GB)
(1 - aws_rds_freeable_memory_average / (8 * 1024 * 1024 * 1024)) * 100

# #3 磁盘利用率 (100GB)
(1 - aws_rds_free_storage_space_average / (100 * 1024 * 1024 * 1024)) * 100
# 或直接用 PI 发布的百分比指标:
aws_rds_file_sys_disk_space_utilization_average

# #4 QPS
aws_rds_queries_average

# #6 连接数
aws_rds_threads_connected_average

# #7 运行线程
aws_rds_threads_running_average

# #8 创建线程
aws_rds_threads_created_average

# #9 慢查询
aws_rds_slow_queries_average

# #10 SQL 次数
aws_rds_select_commands_average                  # SELECT
aws_rds_innodb_rows_inserted_average             # INSERT (InnoDB 行级)
aws_rds_innodb_rows_updated_average              # UPDATE (InnoDB 行级)
aws_rds_innodb_rows_deleted_average              # DELETE (InnoDB 行级)

# #11 InnoDB 缓存命中率
aws_rds_innodb_buffer_pool_hit_rate_average

# #12 InnoDB 缓存使用率
aws_rds_innodb_buffer_pool_utilization_average

# #13 InnoDB 读磁盘
aws_rds_innodb_buffer_pool_reads_average

# #14 InnoDB 写磁盘
aws_rds_innodb_data_writes_average

# #16 等待表锁
aws_rds_table_locks_waited_average

# #17 立即表锁
aws_rds_table_locks_immediate_average

# #18 InnoDB 行锁等待
aws_rds_innodb_row_lock_waits_average

# #19 行锁平均时间 (Grafana 计算)
aws_rds_innodb_row_lock_time_average / aws_rds_innodb_row_lock_waits_average
```

### mysqld_exporter 指标 (补齐)

```promql
# #5 TPS (CloudWatch 无 commit/rollback)
rate(mysql_global_status_handler_commit[5m]) + rate(mysql_global_status_handler_rollback[5m])

# #15 InnoDB fsync (CloudWatch 无此指标)
rate(mysql_global_status_innodb_data_fsyncs[5m])

# #20-23 复制指标 (仅只读副本, CloudWatch 无 REPLICA STATUS)
mysql_slave_status_read_master_log_pos - mysql_slave_status_exec_master_log_pos
mysql_slave_status_seconds_behind_master
mysql_slave_status_slave_sql_running
mysql_slave_status_slave_io_running
```

---

## 6. 成本

| 项目 | 月费 |
|------|------|
| CloudWatch 标准指标存储 | $0 (免费) |
| CloudWatch GetMetricData API (18 项指标) | ~$7.80 |
| Performance Insights (7天免费层) | $0 |
| Enhanced Monitoring Logs (60s) | ~$0.08 |
| 计算资源 (跳板机部署) | $0 |
| Prometheus 存储 (30d, ~105MB) | $0 |
| **月新增总计** | **~$7.88** |

> 如需降低 API 费用: 将 YACE `scraping-interval` 从 60s 改为 300s，费用降至 ~$1.56/月。

## 7. 系统压力

| 环节 | 对 RDS 影响 |
|------|-----------|
| YACE | 不接触 RDS (调 CloudWatch API) |
| mysqld_exporter | < 0.1% CPU, 0 IOPS, 峰值 +1 连接, 每次 scrape 20-60ms |
| Performance Insights | < 1% CPU (被动读取 performance_schema) |
| Enhanced Monitoring | 零 (hypervisor 层运行) |

## 8. RDS 上的关键坑

| 坑 | 处理 |
|----|------|
| PI 未开启时 CloudWatch 无引擎指标 | **必须开启 PI**, 否则 CloudWatch 只有基础 OS 指标 |
| `query_response_time` 默认开启 | `--no-collect.info_schema.query_response_time` |
| MySQL 8.4 移除 `Com_*` | TPS 用 `Handler_commit`, CloudWatch 用 `InnoDBRows*` 替代 |
| 密码含 `%?&#$` | 避免使用, 已知兼容性问题 |
| DDL 阻塞 exporter | `--exporter.lock_wait_timeout=2` |
