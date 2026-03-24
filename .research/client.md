# client/ 模块

## 概述
gRPC 客户端实现，用于与 GreptimeDB 集群节点 (frontend/datanode/flownode) 通信。

## 关键类型
- `Client`: 核心 gRPC 客户端，包装 ChannelManager
- `Database`: 高级数据库客户端 (SQL/insert/DDL via gRPC)
- `DatabaseClient`: 包装 GreptimeDatabaseClient
- `FlightClient`: 包装 FlightServiceClient (Arrow Flight)
- `RegionRequester`: 实现 `Datanode` trait
- `FlowRequester`: 实现 `Flownode` trait
- `NodeClients`: 客户端管理器，实现 `DatanodeManager`/`FlownodeManager`
- `Inserter` trait: insert_rows, set_options

## 架构模式
- **连接池**: ChannelManager 管理 gRPC 通道
- **客户端缓存**: NodeClients 使用 moka cache (TTL 30min, TTI 5min)
- **负载均衡**: Random peer 选择
- **流式协议**: Arrow Flight 流式查询; DoPut 流式插入
- **压缩**: 支持 Zstd 和 Gzip
- **重试逻辑**: `handle_with_retry()` 对 Unavailable 错误重试
