# frontend/ 模块

## 概述
用户入口点，接受多协议连接 (HTTP/gRPC/MySQL/PG/OpenTSDB/InfluxDB/Prometheus/OTLP/Jaeger)。

## 关键类型
- `Frontend`: 顶层入口 (Instance + ServerHandlers + HeartbeatTask)
- `Instance`: 核心查询执行引擎 (catalog_manager, pipeline_operator, statement_executor, query_engine)
- `Services<T>`: 构建所有服务器 (gRPC/HTTP/MySQL/PG)
- `HeartbeatTask`: 后台心跳任务
- `EventHandlerImpl`: 事件记录

## 架构模式
1. **多协议服务器**: `Services` builder 创建协议特定服务器，共享同一个 `Instance`
2. **委托模式**: `Instance` 实现 `SqlQueryHandler`, `PrometheusHandler`
3. **心跳/成员**: 后台心跳任务注册到 Metasrv
4. **Suspend 机制**: `AtomicBool` 标志允许 Metasrv 暂停前端
5. **权限检查**: 每个语句通过 `check_permission()`
6. **可取消查询**: `CancellableFuture` + `ProcessManager`

## HeartbeatTask
- 两个后台任务 (via `common_runtime::spawn_hb()`):
  - 响应流处理: 读取心跳响应并分发
  - 心跳报告: 周期发送节点信息 (CPU/内存)
