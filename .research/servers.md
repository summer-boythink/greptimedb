# servers/ 模块

## 概述
多协议服务器层，提供统一架构处理多种数据库客户端协议。

## 支持的协议
| 协议 | 模块 | 描述 |
|------|------|------|
| HTTP/REST | http.rs | Axum 服务器 (SQL/PromQL/各种写入 API) |
| gRPC | grpc.rs | Tonic 服务器 (GreptimeDatabase/HealthCheck/Flight/OTel Arrow) |
| MySQL | mysql.rs | MySQL 线路协议 |
| PostgreSQL | postgres.rs | PostgreSQL 线路协议 |
| OpenTSDB | opentsdb.rs | OpenTSDB 行协议 |
| InfluxDB | influxdb.rs | InfluxDB Line Protocol |
| Prometheus Remote Write | prom_store.rs | Prometheus 远程写/读 |
| Prometheus HTTP API | prometheus.rs | Prometheus HTTP 查询 API |
| OpenTelemetry | otlp.rs | OTLP metrics/traces/logs |
| Loki | http/loki.rs | Grafana Loki 日志摄取 |
| Elasticsearch | elasticsearch.rs | Elasticsearch 兼容 bulk API |
| Jaeger | http/jaeger.rs | Jaeger 分布式追踪查询 |

## 关键类型
- `Server` trait: start(), shutdown(), name(), bind_addr()
- `ServerHandlers`: 管理所有服务器服务的生命周期
- `BaseTcpServer`: TCP 服务器基础 (abort handles, keep-alive)
- `HttpServer`: Axum HTTP 服务器
- `GrpcServer`: Tonic gRPC 服务器 (TLS, max connection age)

## 查询处理器 traits
- `InfluxdbLineProtocolHandler`
- `OpentsdbProtocolHandler`
- `PromStoreProtocolHandler`
- `OpenTelemetryProtocolHandler`
- `PipelineHandler`
- `DashboardHandler`
- `LogQueryHandler`
- `JaegerQueryHandler`
