# meta-client/ 模块

## 概述
连接 Metasrv 的 gRPC 客户端，是 Datanode/Flownode/Frontend 与元数据服务通信的主要通道。

## 关键类型
- `MetaClient`: 主客户端，包含可选子客户端 (HeartbeatClient/StoreClient/ProcedureClient/ClusterClient/ConfigClient)
- `MetaClientBuilder`: Builder 模式
- `AskLeader`: 实现 LeaderProvider trait，并行询问多个 Metasrv peers
- `HeartbeatSender/HeartbeatStream`: 双向 gRPC 心跳流包装器
- `StoreClient`: gRPC store 客户端 (随机 peer 选择)
- `ProcedureClient`: DDL 提交、Region 迁移、协调、GC

## 架构模式
- **Builder Pattern**: 根据节点角色启用不同子客户端
- **Leader 发现**: 并行询问多个 peers，随机 shuffle，接受第一个有效响应
- **角色组合**:
  - Datanode: heartbeat + store
  - Flownode: heartbeat + store + procedure + cluster info
  - Frontend: heartbeat + store + procedure + cluster info + DDL timeout + region follower
- **压缩**: Zstd 和 Gzip
