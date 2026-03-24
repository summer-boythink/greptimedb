# meta-srv/ 模块深度研究报告

## 概述

`meta-srv/` 是 GreptimeDB 的元数据服务，是分布式数据库集群的集中协调层。它管理集群拓扑、Region 放置、故障检测和过程执行。

## 文件结构

```
src/meta-srv/src/
├── lib.rs
├── metasrv.rs                        -- 核心 Metasrv 结构体
├── metasrv/builder.rs                -- 构建器 (组装所有子系统)
├── procedure.rs                      -- ProcedureManager 适配器
├── procedure/region_migration/       -- Region 迁移过程
├── procedure/wal_prune/              -- WAL 剪枝过程
├── procedure/repartition/            -- 重分区过程
├── region/supervisor.rs              -- Region 故障检测 + failover
├── gc/scheduler.rs                   -- 垃圾回收调度器
├── gc/procedure.rs                   -- GC 过程 (BatchGcProcedure)
├── gc/handler.rs                     -- GC 处理器
├── gc/tracker.rs                     -- GC 跟踪器
├── gc/dropped.rs                     -- 已删除 Region 收集器
├── utils.rs                          -- define_ticker! 宏
└── ...
```

## 架构层次

```
Metasrv (metasrv.rs)
├── LeadershipChangeNotifier (广播 leader 变更)
│   ├── ProcedureManagerListenerAdapter (启停 ProcedureManager)
│   ├── RegionSupervisorTicker (启停故障检测)
│   ├── WalPruneTicker (启停 WAL 剪枝)
│   ├── RegionFlushTicker (启停 flush)
│   └── GcTicker (启停 GC)
├── ProcedureManager (LocalManager)
│   ├── RegionMigrationManager
│   ├── DdlManager
│   └── WalPruneManager
├── RegionSupervisor (事件驱动故障检测)
│   ├── RegionFailureDetector (Phi Accrual)
│   └── RegionMigrationManager (failover 提交)
├── GcScheduler (后台垃圾回收)
└── RegionFlushTrigger (远程 WAL flush 管理)
```

## define_ticker! 宏

位于 `utils.rs:23-105`，生成完整的 ticker 类型:
- 包含 `tick_handle: Mutex<Option<JoinHandle<()>>>`, `tick_interval`, `sender`
- 实现 `LeadershipChangeListener`: `on_leader_start()` -> `start()`, `on_leader_stop()` -> `stop()`
- `start()`: 在 Tokio 上 spawn `interval_at` 定时发送事件
- `stop()`: 获取 handle 并调用 `abort()`
- `Drop` 实现: 调用 `stop()` 清理

**Ticker 实现**:
1. `GcTicker` (gc/scheduler.rs:108)
2. `WalPruneTicker` (procedure/wal_prune/manager.rs:97)
3. `RegionFlushTicker` (region/flush_trigger.rs:57)

## RegionSupervisorTicker

手动实现 (未使用宏)，因为需要两个 tick 循环:
1. **Tick 循环**: 按 `DEFAULT_TICK_INTERVAL` (1秒) 发送 `Event::Tick` 用于故障检测
2. **初始化循环**: 按配置延迟和重试周期发送 `Event::InitializeAllRegions`

## 过程类型

### RegionMigrationProcedure
状态机: `MigrationStart` -> `OpenCandidateRegion` -> `FlushLeaderRegion` -> `UpdateMetadata::Downgrade` -> `DowngradeLeaderRegion` -> `UpgradeCandidateRegion` -> `UpdateMetadata::Upgrade` -> `CloseDowngradedRegion` -> `MigrationEnd`

- 支持回滚 (恢复 downgraded leader 状态)
- 使用 `RegionMigrationProcedureGuard` 去重 (每个 Region 同时只有一个迁移)

### BatchGcProcedure
状态机: `Start` -> `Acquiring` (获取文件引用) -> `Gcing` (发送 GC 指令) -> `UpdateRepartition` (清理元数据)

### WalPruneProcedure
修剪远程 WAL (Kafka) topics，使用 `WalPruneProcedureGuard` 去重。

## Leadership 感知

`ProcedureManagerListenerAdapter` 桥接 leadership 通知到过程管理器:
- `on_leader_start()`: 调用 `procedure_manager.start()` -- 开始接受和运行过程
- `on_leader_stop()`: 调用 `procedure_manager.stop()` -- 停止接受新过程

**重要**: `LocalManager::stop()` 只设置 `running = false` 并停止 GC 任务。它不取消正在运行的过程。过程在 step-down 后继续运行，但新提交被拒绝。

## GC 调度

`GcScheduler` 作为后台任务通过 mpsc channel 处理事件:
- `Event::Tick`: 周期触发 (每 5 分钟)
- `Event::Manually`: 管理员 API 手动触发

GC 流程:
1. 从心跳收集 Region 统计
2. 按表选择 GC 候选 (评分: SST 数量、文件移除率、Region 大小)
3. 从 `table_repart` 元数据收集已删除 Region
4. 按 datanode 分配 Region
5. 为每个 datanode 批次启动 `BatchGcProcedure`

## 事件循环架构

所有后台服务遵循一致的模式:
```
Ticker (define_ticker!) --发送事件--> Manager/Handler --处理--> Actions
```

Ticker 受 leadership 变更控制: leader 选举时启动，step-down 时停止。
