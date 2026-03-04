# RDS MySQL 监控方案 — YACE + mysqld_exporter

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
mysqld_exporter ── 直连 MySQL 3306 ──► Prometheus    (20 项引擎指标, 每 15s)
YACE ────────── CloudWatch API ──────► Prometheus    (3 项 OS 指标, 每 60s)
```

为什么是这个架构:

- **mysqld_exporter (主力)**: 23 项指标中 20 项来自 MySQL 引擎 (`SHOW GLOBAL STATUS` / `perf_schema` / `SHOW REPLICA STATUS`)，只能直连数据库获取
- **YACE (辅助)**: CPU/内存/磁盘是 OS 级指标，RDS 托管服务无法安装 node_exporter，只有 CloudWatch 能提供
- **Performance Insights 已关闭**: 与 mysqld_exporter 完全重叠，且 PI 数据不在 CloudWatch Metrics 中（存在 PI 自有存储），YACE 无法采集。唯一能让 PI 数据进入 CloudWatch 的方式是开启 Database Insights Advanced（额外付费），但仍缺 9 项指标（TPS、Com_insert/update/delete、Innodb_data_reads/fsyncs、行锁 avg、复制线程状态），依然需要 mysqld_exporter 补齐

### MySQL 8.4 破坏性变更

`Com_*` 状态变量全部移除（Com_select/insert/update/delete/commit/rollback 均不存在）。替代方案：
- **TPS**: `Handler_commit` + `Handler_rollback`（仍在 global_status 中）
- **各类 SQL 次数**: `performance_schema.events_statements_summary_global_by_event_name`（需 `--collect.perf_schema.eventsstatements`）

### 23 项指标映射

| # | 指标 | 来源 | Collector / 指标名 |
|---|------|------|--------------------|
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

---

## 2. RDS 侧配置 (已完成)

### 2.1 参数组 `mysql84-monitor`

| 参数 | 值 | 类型 | 用途 |
|------|---|------|------|
| `slow_query_log` | `1` | 动态 | 慢查询 #9 |
| `long_query_time` | `1` | 动态 | 慢查询阈值 (秒) |
| `innodb_monitor_enable` | `all` | 动态 | InnoDB 详细计数器 |
| `performance_schema` | `ON` (默认) | 静态 | perf_schema collectors |

### 2.2 创建 exporter 用户

```sql
CREATE USER IF NOT EXISTS 'exporter'@'%'
  IDENTIFIED BY 'YOUR_EXPORTER_PASSWORD'
  WITH MAX_USER_CONNECTIONS 3;

GRANT PROCESS ON *.* TO 'exporter'@'%';
GRANT REPLICATION CLIENT ON *.* TO 'exporter'@'%';
GRANT SELECT ON performance_schema.* TO 'exporter'@'%';
FLUSH PRIVILEGES;
```

### 2.3 安全组

RDS 安全组入站规则: TCP 3306 ← exporter 所在主机 (同 VPC)

### 2.4 YACE 所需 IAM 权限

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
curl -s http://localhost:5000/metrics | grep aws_rds_cpuutilization
# 预期: aws_rds_cpuutilization_average{...} <value>

# Prometheus
curl -s http://localhost:9090/api/v1/targets | python3 -m json.tool
# 预期: 两个 target 均为 "up"
```

---

## 4. 配置文件

### 4.1 docker-compose.yaml

见 [`docker-compose.yaml`](docker-compose.yaml)

mysqld_exporter 关键注意:
- `--no-collect.info_schema.query_response_time` — **必须禁用**, 此 collector 默认开启但 RDS 上报错 (Percona 专有)
- `--collect.perf_schema.eventsstatements` — **MySQL 8.4 必须**, Com_* 已移除, 各类 SQL 次数从此获取
- `--exporter.lock_wait_timeout=2` — 防止被 DDL 阻塞

### 4.2 yace-config.yaml

见 [`yace-config.yaml`](yace-config.yaml) — 仅 3 项指标: CPUUtilization, FreeableMemory, FreeStorageSpace

### 4.3 其他配置

- [`mysqld-exporter.cnf`](mysqld-exporter.cnf) — MySQL 连接 + SSL
- [`prometheus.yaml`](prometheus.yaml) — Prometheus scrape 配置
- [`.env.example`](.env.example) — 环境变量模板

---

## 5. Grafana PromQL

> MySQL 8.4: `Com_*` 已移除。TPS 用 `Handler_commit/rollback`, SQL 类型用 `perf_schema`。

### #1 CPU利用率
```promql
aws_rds_cpuutilization_average{dimension_DBInstanceIdentifier="database-1"}
```

### #2 内存利用率
```promql
(1 - aws_rds_freeable_memory_average{dimension_DBInstanceIdentifier="database-1"}
  / (8 * 1024 * 1024 * 1024)) * 100
```

### #3 磁盘利用率
```promql
(1 - aws_rds_free_storage_space_average{dimension_DBInstanceIdentifier="database-1"}
  / (100 * 1024 * 1024 * 1024)) * 100
```

### #4 QPS
```promql
rate(mysql_global_status_queries[5m])
```

### #5 TPS
```promql
rate(mysql_global_status_handler_commit[5m]) + rate(mysql_global_status_handler_rollback[5m])
```

### #6 连接数 / 使用率
```promql
mysql_global_status_threads_connected
mysql_global_status_threads_connected / mysql_global_variables_max_connections * 100
```

### #7 运行线程
```promql
mysql_global_status_threads_running
```

### #8 创建线程
```promql
rate(mysql_global_status_threads_created[5m])
```

### #9 慢查询
```promql
rate(mysql_global_status_slow_queries[5m])
```

### #10 各类SQL次数
```promql
rate(mysql_perf_schema_events_statements_total{event_name=~"statement/sql/(select|insert|update|delete)"}[5m])
```

### #11 InnoDB缓存命中率
```promql
(1 - rate(mysql_global_status_innodb_buffer_pool_reads[5m])
   / rate(mysql_global_status_innodb_buffer_pool_read_requests[5m])) * 100
```

### #12 InnoDB缓存使用率
```promql
(mysql_global_status_innodb_buffer_pool_pages_total - mysql_global_status_innodb_buffer_pool_pages_free)
  / mysql_global_status_innodb_buffer_pool_pages_total * 100
```

### #13-15 InnoDB 读/写/fsync
```promql
rate(mysql_global_status_innodb_data_reads[5m])
rate(mysql_global_status_innodb_data_writes[5m])
rate(mysql_global_status_innodb_data_fsyncs[5m])
```

### #16-17 表锁
```promql
rate(mysql_global_status_table_locks_waited[5m])
rate(mysql_global_status_table_locks_immediate[5m])
```

### #18-19 InnoDB行锁
```promql
rate(mysql_global_status_innodb_row_lock_waits[5m])
mysql_global_status_innodb_row_lock_time_avg
```

### #20-23 复制指标 (仅只读副本)
```promql
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
| CloudWatch GetMetricData API (3 项指标) | ~$1.30 |
| Enhanced Monitoring Logs (60s, 已开启) | ~$0.08 |
| Performance Insights | $0 (已关闭) |
| 计算资源 (跳板机部署) | $0 |
| Prometheus 存储 (30d, ~105MB) | $0 |
| **月新增总计** | **~$1.38** |

## 7. 系统压力

| 环节 | 对 RDS 影响 |
|------|-----------|
| mysqld_exporter | < 0.1% CPU, 0 IOPS, 峰值 +1 连接, 每次 scrape 20-60ms |
| YACE | 不接触 RDS (调 CloudWatch API) |
| Enhanced Monitoring | 零 (hypervisor 层运行) |

## 8. RDS 上的关键坑

| 坑 | 处理 |
|----|------|
| `query_response_time` 默认开启 | `--no-collect.info_schema.query_response_time` |
| MySQL 8.4 移除 `Com_*` | TPS 用 `Handler_commit`, SQL 类型用 `perf_schema` |
| 密码含 `%?&#$` | 避免使用, 已知兼容性问题 |
| `info_schema.tables` | 表多时极慢, 不要开启 |
| DDL 阻塞 exporter | `--exporter.lock_wait_timeout=2` |
