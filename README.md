# RDS MySQL 监控方案

## 环境

| 项 | 值 |
|----|---|
| RDS | database-1, MySQL 8.4.7, db.m7g.large (2vCPU/8GB), 100GB gp3, Multi-AZ |
| Region | us-east-1, VPC vpc-7d049706 |
| 跳板机 | i-0ecf0ebb4ec863f9e (m6i.xlarge, 同 VPC, SSM) |
| 参数组 | mysql84-monitor (自定义, 已应用) |

---

## 1. 方案对比

### 方案 A：mysqld_exporter + YACE（推荐，PI 不开）

```
mysqld_exporter ── MySQL 3306 ──► Prometheus    (20 项引擎指标, 每 15s)
YACE ─── CloudWatch API ─────────► Prometheus    (3 项 OS 指标, 每 60s)
```

### 方案 B：YACE + PI（需开启 Performance Insights）

```
YACE ─── CloudWatch API ─────────► Prometheus    (18 项, 每 60s)
mysqld_exporter ── MySQL 3306 ──► Prometheus    (仍需补齐 5 项)
```

### 23 项指标逐项对比

| # | 指标 | 方案 A (exporter+YACE) | 方案 B (PI+YACE) | 方案 B 仍需 exporter？ |
|---|------|----------------------|-----------------|---------------------|
| 1 | CPU利用率 | YACE `CPUUtilization` | YACE `CPUUtilization` | 否 |
| 2 | 内存利用率 | YACE `FreeableMemory` | YACE `FreeableMemory` | 否 |
| 3 | 磁盘利用率 | YACE `FreeStorageSpace` | YACE `FreeStorageSpace` | 否 |
| 4 | QPS | exporter `Queries` | YACE `Queries` | 否 |
| 5 | TPS | exporter `Handler_commit/rollback` | **缺失** (CloudWatch 无 commit/rollback) | **是** |
| 6 | 连接数 | exporter `Threads_connected` | YACE `ThreadsConnected` | 否 |
| 7 | 运行线程 | exporter `Threads_running` | YACE `ThreadsRunning` | 否 |
| 8 | 创建线程 | exporter `Threads_created` | YACE `ThreadsCreated` | 否 |
| 9 | 慢查询 | exporter `Slow_queries` | YACE `SlowQueries` | 否 |
| 10 | Com_select | exporter `perf_schema statement/sql/select` | YACE `SelectCommands` | 否 |
| 10 | Com_insert | exporter `perf_schema statement/sql/insert` | ⚠️ `InnoDBRowsInserted` (行级替代, 非精确) | 看是否接受替代 |
| 10 | Com_update | exporter `perf_schema statement/sql/update` | ⚠️ `InnoDBRowsUpdated` (行级替代, 非精确) | 看是否接受替代 |
| 10 | Com_delete | exporter `perf_schema statement/sql/delete` | ⚠️ `InnoDBRowsDeleted` (行级替代, 非精确) | 看是否接受替代 |
| 11 | InnoDB缓存命中率 | exporter 计算 `reads/read_requests` | YACE `InnoDBBufferPoolHitRate` | 否 |
| 12 | InnoDB缓存使用率 | exporter 计算 `pages_total/free` | YACE `InnoDBBufferPoolUtilization` | 否 |
| 13 | InnoDB读磁盘 | exporter `Innodb_data_reads` | YACE `InnoDBBufferPoolReads` | 否 |
| 14 | InnoDB写磁盘 | exporter `Innodb_data_writes` | YACE `InnoDBDataWrites` | 否 |
| 15 | InnoDB fsync | exporter `Innodb_data_fsyncs` | **缺失** (CloudWatch 无此指标) | **是** |
| 16 | 等待表锁 | exporter `Table_locks_waited` | YACE `TableLocksWaited` | 否 |
| 17 | 立即表锁 | exporter `Table_locks_immediate` | YACE `TableLocksImmediate` | 否 |
| 18 | InnoDB行锁 | exporter `Innodb_row_lock_waits` | YACE `InnoDBRowLockWaits` | 否 |
| 19 | 行锁平均时间 | exporter `Innodb_row_lock_time_avg` | YACE 计算 `Time/Waits` | 否 |
| 20 | 主从延迟距离 | exporter `SHOW REPLICA STATUS` * | **缺失** (CloudWatch 无 bytes 级延迟) | **是** |
| 21 | 主从延迟时间 | exporter `Seconds_Behind_Master` * | YACE `ReplicaLag` (标准 CW 指标, 不需要 PI) | 否 |
| 22 | SlaveSqlRunning | exporter `SHOW REPLICA STATUS` * | **缺失** | **是** |
| 23 | SlaveIoRunning | exporter `SHOW REPLICA STATUS` * | **缺失** | **是** |

\* 需创建只读副本后生效

### 汇总

| | 方案 A (exporter+YACE) | 方案 B (PI+YACE) |
|---|---|---|
| 精确覆盖 | **23/23 (100%)** | 15/23 (65%) |
| 替代覆盖 | — | +3 项 (InnoDBRows\* 替代 Com_insert/update/delete) |
| 缺失 | 0 | 5 项 (#5 TPS, #15 fsync, #20 延迟距离, #22-23 复制线程) |
| 方案 B 仍需 exporter | — | **是**，至少需要补 #5 TPS 和 #15 fsync |
| 需要开 PI | **不需要** | 需要 |
| 数据精度 | 15s (exporter) / 60s (YACE) | 60s (CloudWatch 最小粒度) |
| CloudWatch API 费用 | ~$1.30/月 (3 项) | ~$7.80/月 (18 项) |
| 对 RDS 压力 | exporter < 0.1% CPU | PI < 1% CPU |

### 结论

**选方案 A**。原因：
1. 方案 B 的 PI+YACE 无法独立工作，仍需 mysqld_exporter 补齐 TPS、fsync、复制状态，多开一个 PI 没有减少任何组件
2. 方案 A 的 exporter 已经覆盖 20/23，只需 YACE 补 3 项 OS 指标，**PI 不需要开**
3. 方案 A 数据精度更高 (15s vs 60s)，API 费用更低 ($1.30 vs $7.80)

---

## 2. 架构（方案 A）

```
mysqld_exporter ── 直连 MySQL 3306 ──► Prometheus    (20 项, 每 15s)
YACE ─── CloudWatch API ─────────────► Prometheus    (3 项 OS, 每 60s)
PI: 不开启
```

### 23 项指标映射

| # | 指标 | 来源 | 指标名 / Collector |
|---|------|------|-------------------|
| 1 | CPU利用率 | YACE | `CPUUtilization` |
| 2 | 内存利用率 | YACE | `FreeableMemory` |
| 3 | 磁盘利用率 | YACE | `FreeStorageSpace` |
| 4 | QPS | exporter | `global_status` → `Queries` |
| 5 | TPS | exporter | `global_status` → `Handler_commit` + `Handler_rollback` |
| 6 | 连接数 | exporter | `global_status` → `Threads_connected` |
| 7 | 运行线程 | exporter | `global_status` → `Threads_running` |
| 8 | 创建线程 | exporter | `global_status` → `Threads_created` |
| 9 | 慢查询 | exporter | `global_status` → `Slow_queries` |
| 10 | 各类SQL次数 | exporter | `perf_schema.eventsstatements` → `statement/sql/*` |
| 11 | InnoDB缓存命中率 | exporter | `global_status` → 计算 `read_requests` vs `reads` |
| 12 | InnoDB缓存使用率 | exporter | `global_status` → `pages_total` / `pages_free` |
| 13 | InnoDB读磁盘 | exporter | `global_status` → `Innodb_data_reads` |
| 14 | InnoDB写磁盘 | exporter | `global_status` → `Innodb_data_writes` |
| 15 | InnoDB fsync | exporter | `global_status` → `Innodb_data_fsyncs` |
| 16 | 等待表锁 | exporter | `global_status` → `Table_locks_waited` |
| 17 | 立即表锁 | exporter | `global_status` → `Table_locks_immediate` |
| 18 | InnoDB行锁 | exporter | `global_status` → `Innodb_row_lock_waits` |
| 19 | 行锁平均时间 | exporter | `global_status` → `Innodb_row_lock_time_avg` |
| 20 | 主从延迟距离 | exporter | `slave_status` → `Read_Master_Log_Pos - Exec_Master_Log_Pos` * |
| 21 | 主从延迟时间 | exporter | `slave_status` → `Seconds_Behind_Master` * |
| 22 | SlaveSqlRunning | exporter | `slave_status` → `Slave_SQL_Running` * |
| 23 | SlaveIoRunning | exporter | `slave_status` → `Slave_IO_Running` * |

\* 需创建只读副本后生效

### MySQL 8.4 注意

`Com_*` 状态变量已移除。替代：
- **TPS**: `Handler_commit` + `Handler_rollback`（global_status 中仍有）
- **各类 SQL 次数**: `perf_schema.events_statements_summary_global_by_event_name` 中的 `statement/sql/select|insert|update|delete`

---

## 3. RDS 侧配置

### 3.1 参数组 `mysql84-monitor`

| 参数 | 值 | 用途 |
|------|---|------|
| `slow_query_log` | `1` | 慢查询 |
| `long_query_time` | `1` | 慢查询阈值 (秒) |
| `innodb_monitor_enable` | `all` | InnoDB 详细计数器 |
| `performance_schema` | `ON` (默认) | perf_schema collectors |

### 3.2 创建 exporter 用户

```sql
CREATE USER IF NOT EXISTS 'exporter'@'%'
  IDENTIFIED BY 'YOUR_EXPORTER_PASSWORD'
  WITH MAX_USER_CONNECTIONS 3;

GRANT PROCESS ON *.* TO 'exporter'@'%';
GRANT REPLICATION CLIENT ON *.* TO 'exporter'@'%';
GRANT SELECT ON performance_schema.* TO 'exporter'@'%';
FLUSH PRIVILEGES;
```

### 3.3 安全组

RDS 安全组: TCP 3306 ← exporter 所在主机 (同 VPC)

### 3.4 YACE 所需 IAM 权限

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

## 4. 部署

```bash
# 下载 RDS CA 证书
curl -o global-bundle.pem https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem

# 配置环境变量
cp .env.example .env
# 编辑 .env: AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, MYSQL_EXPORTER_PASSWORD

# 启动
docker-compose up -d

# 验证
curl -s http://localhost:9104/metrics | grep mysql_up        # 预期: 1
curl -s http://localhost:5000/metrics | grep aws_rds         # 预期: 有数据
curl -s http://localhost:9090/api/v1/targets                 # 预期: 两个 target UP
```

---

## 5. Grafana PromQL

### YACE (3 项 OS 指标)

```promql
# #1 CPU利用率
aws_rds_cpuutilization_average{dimension_DBInstanceIdentifier="database-1"}

# #2 内存利用率 (db.m7g.large = 8GB)
(1 - aws_rds_freeable_memory_average / (8 * 1024 * 1024 * 1024)) * 100

# #3 磁盘利用率 (100GB)
(1 - aws_rds_free_storage_space_average / (100 * 1024 * 1024 * 1024)) * 100
```

### mysqld_exporter (20 项引擎指标)

```promql
# #4 QPS
rate(mysql_global_status_queries[5m])

# #5 TPS
rate(mysql_global_status_handler_commit[5m]) + rate(mysql_global_status_handler_rollback[5m])

# #6 连接数 / 使用率
mysql_global_status_threads_connected
mysql_global_status_threads_connected / mysql_global_variables_max_connections * 100

# #7 运行线程
mysql_global_status_threads_running

# #8 创建线程
rate(mysql_global_status_threads_created[5m])

# #9 慢查询
rate(mysql_global_status_slow_queries[5m])

# #10 各类SQL次数 (MySQL 8.4: Com_* 已移除, 用 perf_schema)
rate(mysql_perf_schema_events_statements_total{event_name=~"statement/sql/(select|insert|update|delete)"}[5m])

# #11 InnoDB缓存命中率
(1 - rate(mysql_global_status_innodb_buffer_pool_reads[5m])
   / rate(mysql_global_status_innodb_buffer_pool_read_requests[5m])) * 100

# #12 InnoDB缓存使用率
(mysql_global_status_innodb_buffer_pool_pages_total - mysql_global_status_innodb_buffer_pool_pages_free)
  / mysql_global_status_innodb_buffer_pool_pages_total * 100

# #13-15 InnoDB 读/写/fsync
rate(mysql_global_status_innodb_data_reads[5m])
rate(mysql_global_status_innodb_data_writes[5m])
rate(mysql_global_status_innodb_data_fsyncs[5m])

# #16-17 表锁
rate(mysql_global_status_table_locks_waited[5m])
rate(mysql_global_status_table_locks_immediate[5m])

# #18-19 InnoDB行锁
rate(mysql_global_status_innodb_row_lock_waits[5m])
mysql_global_status_innodb_row_lock_time_avg

# #20-23 复制指标 (仅只读副本)
mysql_slave_status_read_master_log_pos - mysql_slave_status_exec_master_log_pos
mysql_slave_status_seconds_behind_master
mysql_slave_status_slave_sql_running
mysql_slave_status_slave_io_running
```

---

## 6. 成本与压力

| 项目 | 月费 |
|------|------|
| CloudWatch GetMetricData API (3 项) | ~$1.30 |
| Performance Insights | $0 (不开) |
| 计算资源 (跳板机部署) | $0 |
| **月新增总计** | **~$1.30** |

| 环节 | 对 RDS 影响 |
|------|-----------|
| mysqld_exporter | < 0.1% CPU, 0 IOPS, 峰值 +1 连接 |
| YACE | 不接触 RDS |

## 7. mysqld_exporter 在 RDS 上的关键坑

| 坑 | 处理 |
|----|------|
| `query_response_time` 默认开启 | `--no-collect.info_schema.query_response_time` |
| MySQL 8.4 移除 `Com_*` | TPS 用 `Handler_commit`, SQL 类型用 `perf_schema` |
| 密码含 `%?&#$` | 避免使用 |
| DDL 阻塞 exporter | `--exporter.lock_wait_timeout=2` |
