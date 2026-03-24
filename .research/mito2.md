# mito2/ 模块深度研究报告

## 概述

Mito 是 GreptimeDB 的核心时序 Region 引擎，实现 LSM-tree 存储架构。它管理 WAL、Memtable、SST 文件、Flush、Compaction、索引构建和 GC。

## 文件结构

```
src/mito2/src/
├── lib.rs
├── engine.rs               -- MitoEngine 公共接口
├── engine/
│   ├── set_readonly.rs     -- 只读模式设置
│   └── create_region.rs    -- Region 创建
├── worker.rs               -- WorkerGroup + RegionWorker ★
├── worker/
│   ├── handle_write.rs     -- 写入处理
│   ├── handle_flush.rs     -- flush 通知处理
│   ├── handle_compaction.rs
│   ├── handle_drop.rs
│   ├── handle_open.rs
│   ├── handle_truncate.rs
│   └── ...
├── region.rs               -- MitoRegion
├── region/
│   ├── version.rs          -- Version 版本控制
│   └── version_control.rs  -- VersionControl (COW)
├── flush.rs                -- Flush 调度器 ★
├── compaction.rs           -- Compaction 调度器 ★
├── compaction/
│   ├── task.rs             -- CompactionTask
│   ├── twcs.rs             -- TWCS 策略
│   └── picker.rs           -- Compaction picker
├── schedule/
│   └── scheduler.rs        -- LocalScheduler ★
├── sst/
│   ├── index.rs            -- SST 索引构建
│   ├── parquet/            -- Parquet 读写
│   └── ...
├── manifest.rs             -- Manifest 管理
├── wal.rs                  -- WAL 集成
├── cache.rs                -- 缓存管理
├── read.rs                 -- 读取路径
├── write_buffer.rs         -- WriteBufferManager
└── ...
```

## 架构核心

### Worker-per-Region 分片

Region 通过 `region_id_to_index()` 哈希路由到特定 Worker:
```rust
fn region_id_to_index(region_id: RegionId, num_workers: usize) -> usize {
    let table_id = region_id.table_id();
    let region_number = region_id.region_number();
    // hash(table_id, region_number) % num_workers
}
```

### WorkerGroup
```rust
pub struct WorkerGroup {
    workers: Vec<Arc<RegionWorker>>,
    flush_job_pool: LocalScheduler,
    compact_job_pool: LocalScheduler,
    index_build_job_pool: LocalScheduler,
    purge_scheduler: LocalScheduler,
    cache_manager: CacheManagerRef,
    file_ref_manager: FileRefManager,
    gc_limiter: GcLimiter,
}
```

### RegionWorker
```rust
pub struct RegionWorker {
    id: usize,
    regions: RegionMapRef,
    sender: Sender<WorkerRequestWithTime>,
    running: Arc<AtomicBool>,
    handle: Mutex<Option<JoinHandle<()>>>,
}
```

### RegionWorkerLoop
实际 Worker 线程状态:
```rust
struct RegionWorkerLoop<S: LogStore> {
    running: Arc<AtomicBool>,
    receiver: Receiver<WorkerRequestWithTime>,
    regions: RegionMapRef,
    wal: S,
    memtable_builder: MemtableBuilderRef,
    flush_scheduler: FlushSchedulerRef,
    compaction_scheduler: CompactionSchedulerRef,
    stalled_requests: StalledRequests,
    cache_manager: CacheManagerRef,
    // ...
}
```

## 主循环 (`worker.rs`)

```rust
while self.running.load(Ordering::Relaxed) {
    tokio::select! {
        request_opt = self.receiver.recv() => {
            // 批量接收请求 (try_recv)
            self.handle_requests(requests).await;
        }
        recv_res = self.flush_receiver.changed() => {
            // flush 完成通知
            self.maybe_flush_worker();
            self.handle_stalled_requests().await;
        }
        _ = &mut sleep => {
            // 周期任务 (60s)
            self.handle_periodical_tasks();
        }
    }
}
```

### 关键行为:
- **批量处理**: 接收一个请求后，尝试 `try_recv()` 最多 `worker_request_batch_size` 个
- **周期任务**: 每 60 秒检查 Region，可能触发自动 flush
- **Flush 通知**: 任何 Worker 完成 flush 后，所有 Worker 被唤醒处理 stalled 请求
- **Stalled 请求**: 写缓冲区满时，写入被暂停直到 flush 完成

## Flush 系统

### WriteBufferManagerImpl
跟踪全局和可变 Memtable 内存使用:
- `should_flush_engine()`: 使用量超过限制时发出 flush 信号
- 可以暂停写入当内存耗尽时

### FlushScheduler
跟踪每个 Region 的 flush 状态:
- 合并同一 Region 的并发 flush 请求
- 管理等待任务 (DDL, 等待 flush 的写入)
- Flush 原因: `EngineFull`, `Manual`, `Alter`, `Periodically`, `Downgrading`, `EnterStaging`, `Closing`

### RegionFlushTask
将不可变 Memtable 转换为 SST 文件:
- 使用信号量限制并发 flush 文件写入
- 工作流: Flush 请求 -> 冻结可变 Memtable -> 调度到 `flush_job_pool` -> 写 SST -> 发送 `FlushFinished` 通知 -> 更新版本 -> 解锁 stalled 请求

## Compaction 系统

### CompactionScheduler
跟踪每个 Region 的压缩状态:
- 管理手动压缩请求和阻塞的 DDL 请求
- 支持远程压缩 (通过 plugins 委派)
- 使用 `CompactionMemoryManager` 限制并发压缩内存

### TWCS (Time Window Compaction Strategy)
基于时间窗口的压缩策略:
- 按时间窗口分组 SST 文件
- 选择时间窗口内的文件进行压缩
- 保留时间窗口边界

## LocalScheduler (`schedule/scheduler.rs`)

简单的固定并发异步任务调度器:
- 使用 `async_channel::unbounded()` 提交任务
- spawn N 个 worker 任务消费任务
- 状态: `RUNNING` -> `AWAIT_TERMINATION` (排空) 或 `STOP` (立即取消)
- 使用 `CancellationToken` 协作取消

```rust
pub struct LocalScheduler {
    state: Arc<AtomicU8>,
    sender: Arc<RwLock<Option<Sender<Job>>>>,
    workers: Vec<JoinHandle<()>>,
    cancel_token: CancellationToken,
}
```

## 关闭流程 (`WorkerGroup::stop()`)

```rust
pub(crate) async fn stop(&self) -> Result<()> {
    // 1. 先停止所有调度器池 (graceful drain)
    self.compact_job_pool.stop(true).await?;
    self.flush_job_pool.stop(true).await?;
    self.purge_scheduler.stop(true).await?;
    self.index_build_job_pool.stop(true).await?;
    
    // 2. 再停止 Worker
    try_join_all(self.workers.iter().map(|worker| worker.stop())).await?;
    Ok(())
}
```

## 版本控制 (Copy-on-Write)

```rust
pub struct VersionControl {
    // 使用 CowCell 实现无锁读取
    version: CowCell<Version>,
}
```

`Version` 包含:
- `MemtableVersion`: 当前和冻结的 Memtable
- `SstVersion`: SST 文件列表
- `ManifestVersion`: Manifest 版本

## 已知问题

详见 `.research/mito2-scheduling-bugs.md`
