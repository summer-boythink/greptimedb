# session/ 模块

## 概述
管理 MySQL/PostgreSQL 等有状态协议的持久连接会话。

## 关键类型
- `Session`: 主会话 struct (catalog, mutable_inner, conn_info, configuration_variables, process_id)
- `MutableInner`: 可变会话状态 (schema, user_info, timezone, query_timeout, read_preference, cursors, warnings)
- `QueryContext`: 每查询执行上下文 (catalog, snapshot_seqs, sql_dialect, channel)
- `Channel`: 协议通道枚举 (MySQL/Postgres/HTTP/Prometheus/OTLP/GRPC 等)
- `ConnInfo`: 客户端地址和通道
- `ConfigurationVariables`: PostgreSQL 特定设置

## 架构模式
- **RwLock 模式**: 并发读取 + 独占写入
- **Arc-Swap**: 无锁配置变量更新
- **Builder 模式**: QueryContextBuilder
- **共享可变状态**: Session 和 QueryContext 共享 `Arc<RwLock<MutableInner>>`
