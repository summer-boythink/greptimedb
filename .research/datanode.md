# datanode/ 模块

## 概述
存储层 Worker，管理 Region Server、与 MetaSrv 通信、管理 WAL、构建存储引擎(Mito/Metric/File)。

## 关键类型
- `Datanode`: 主服务 struct (ServerHandlers + HeartbeatTask + RegionServer)
- `DatanodeBuilder`: 构建器
- `RegionServer`: 中央请求分发器 (RegionId -> RegionEngineWithStatus)
- `RegionServerInner`: 内部状态 (engines, region_map, query_engine)
- `HeartbeatTask`: 周期心跳发送
- `RegionAliveKeeper`: Region 租约管理
- `CountdownTask`: 每 Region 异步任务 (租约倒计时)
- `RegionServerEventSender/Receiver`: 事件通道

## Region 状态机
`Registering` -> `Ready` -> `Deregistering`

## HeartbeatTask
- 周期向 Metasrv 发送心跳 (Region 统计、Topic 统计、资源使用)
- 使用 `tokio::select!` 处理发件箱消息、周期休眠、退出信号
- 响应处理: 租约续期、邮箱消息、缓存失效

## RegionAliveKeeper
- 每 Region 租约倒计时任务
- 租约到期时将 Region 转换为 follower
- 监视 RegionServerEventReceiver (注册/注销事件)

## 架构模式
- **Engine Registry**: RegionServer 维护 engine name -> engine 的映射
- **Handler Chain**: 心跳响应通过 `HandlerGroupExecutor` 链处理
- **Builder Pattern**: `DatanodeBuilder` 流式 API
- **批量优化**: `try_handle_metric_batch_puts()` 优化 metric engine 批量写入
