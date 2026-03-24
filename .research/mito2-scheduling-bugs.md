# Mito2 存储引擎任务调度 Bug 报告

## 问题概述
用户报告: "系统有时会运行一些本应取消的任务"。Mito2 存储引擎存在关闭顺序错误和进行中的后台任务无法取消等问题。

---

## Bug 1 (HIGH): 关闭顺序错误 -- 调度器在 Worker 之前停止

**文件**: `/workspace/greptimedb/src/mito2/src/worker.rs`, 行 279-294

### 问题描述
所有四个调度器池 (compaction, flush, purge, index build) 在 Worker **之前**被优雅停止。池进入 `STATE_AWAIT_TERMINATION`，排空队列并完成。只有在所有池完全停止后才调用 `worker.stop()`。

### 竞态窗口
池 `stop()` 调用完成和 `worker.stop()` 调用之间，Worker 线程仍在运行主循环:
- 已 dispatch 到 Tokio 的后台任务 (flush, compaction, index build) 仍可执行
- 这些任务通过 channel 向 Worker 发送 `FlushFinished`, `CompactionFinished` 通知
- Worker 收到通知后调用 `schedule_compaction()` 或 `maybe_flush_worker()`
- 这些操作尝试在已停止的池上调度新任务 -- 失败

### 更关键的问题
已 dispatch 的后台任务**不会**被 `CancellationToken` 取消:
- `CancellationToken` 只影响池工作线程的 select 循环
- 不取消已 spawn 的 Tokio 任务
- 这些任务继续消耗 CPU/IO
- 尝试向已关闭的 Worker channel 发送结果

### 影响
- 后台任务在关闭后仍运行
- 结果发送到已关闭的 channel (静默丢失)
- 浪费计算资源

---

## Bug 2 (MEDIUM): `LocalScheduler` 中 `is_running()` 和 `try_send()` 的竞态

**文件**: `/workspace/greptimedb/src/mito2/src/schedule/scheduler.rs`, 行 115-146

### 问题描述
```rust
fn schedule(&self, job: Job) -> Result<()> {
    ensure!(self.is_running(), InvalidSchedulerStateSnafu);  // 检查 A
    self.sender.read().unwrap().as_ref()
        .context(InvalidSchedulerStateSnafu)?
        .try_send(job)                                        // 动作 B
        .map_err(|_| InvalidSenderSnafu {}.build())
}

async fn stop(&self, await_termination: bool) -> Result<()> {
    ensure!(self.is_running(), InvalidSchedulerStateSnafu);
    self.sender.write().unwrap().take();                      // 取走 sender
    self.state.store(state, Ordering::Relaxed);               // 设置状态
    self.cancel_token.cancel();
    // ...
}
```

检查 A 和动作 B 不是原子的。`stop()` 可以在检查和发送之间取走 sender。

### 影响
- 不同的错误类型 (`InvalidSender` vs `InvalidSchedulerState`)
- 调用者无法区分"调度器已满"和"调度器正在关闭"

---

## Bug 3 (MEDIUM): 关闭期间 Stalled 请求静默丢失

**文件**: `/workspace/greptimedb/src/mito2/src/worker.rs`, 行 915-1005 和 1243-1251

### 问题描述
Worker 循环退出后，`clean()` 方法被调用。但 `clean()` **不排空或拒绝** `self.stalled_requests`。等待 flush 完成的 stalled 写入请求被简单 drop。

调用者 (外部客户端等待 `oneshot::Sender` channel) 收到 `RecvError` (channel closed) 而不是有意义的 "shutting down" 错误。

### 影响
- 调用者收到令人困惑的错误
- 正在进行的写入被静默丢弃

---

## Bug 4 (HIGH): Region Drop 后进行中的后台任务无法取消

**文件**: `/workspace/greptimedb/src/mito2/src/worker/handle_drop.rs`, 行 78-98 和 `/workspace/greptimedb/src/mito2/src/region.rs`, 行 198-211

### 问题描述
Region 被 drop 时:
1. `region.stop()` 只停止 manifest 管理器
2. Region 从 region map 中移除
3. `on_region_dropped()` 移除跟踪条目

但正在进行的 flush/compaction/index build 任务**没有被取消**。`RegionFlushTask` 和 `CompactionTaskImpl` 没有 cancellation token。它们继续运行到完成:
- Flush 任务: flush memtable 到 SST, 写 manifest, 删除 WAL
- Compaction 任务: 合并 SST 文件, 写 manifest
- Index build 任务: 读 SST, 构建索引, 写 manifest

### 影响
- 浪费 IO 读写 SST 文件
- Manifest 更新被应用但 Region 已逻辑删除
- 新 SST 文件永远不会被读取 (之后会被 GC)

---

## Bug 5 (MEDIUM): Flush 结果在关闭期间丢失 -- 无错误传播

**文件**: `/workspace/greptimedb/src/mito2/src/flush.rs`, 行 661-672 和 `/workspace/greptimedb/src/mito2/src/compaction/task.rs`, 行 262-273

### 问题描述
Worker 线程退出后 drop `mpsc::Receiver`。后台任务尝试 `request_sender.send()` 时收到 `SendError`。两种方法都记录错误但**不传播**给调用者。

特别是 flush 任务:
```rust
self.send_worker_request(worker_request).await;  // 如果失败，错误被记录但不传播
```

### 影响
- 等待 flush 结果的调用者永远收不到响应
- 他们的 `oneshot::Receiver` 被 drop，导致 `RecvError`

---

## Bug 6 (LOW): `maybe_flush_worker()` 在关闭期间调用已停止的池

**文件**: `/workspace/greptimedb/src/mito2/src/worker.rs`, 行 943-956

### 问题描述
`flush_receiver` watch channel 在所有 Worker 间共享。当任何 Worker 完成 flush 时，所有 Worker 被唤醒。在关闭期间，flush 完成通知 (在池停止前发送) 可以到达。Worker 调用 `maybe_flush_worker()` -> `flush_regions_on_engine_full()` -> 在已停止的 flush 池上调用 `schedule()`。

### 影响
- 错误被传播和记录但不崩溃
- 浪费工作和混乱的错误消息

---

## Bug 7 (MEDIUM): 后台任务向已关闭的 Worker channel 发送

**文件**: `/workspace/greptimedb/src/mito2/src/compaction/task.rs`, 行 318-323 和 `/workspace/greptimedb/src/mito2/src/sst/index.rs`, 行 716-727

### 问题描述
Compaction 任务和 index build 任务在调度池停止后完成。Worker 的 channel receiver 可能已被 drop。

- compaction/task.rs: `send_to_worker()` 记录错误
- sst/index.rs: `let _ = self.request_sender.send(...)` 静默忽略

### 影响
- 结果丢失
- 调用者永远收不到通知

---

## Bug 8 (MEDIUM): `RegionWorker::Drop` vs `stop()` 竞态

**文件**: `/workspace/greptimedb/src/mito2/src/worker.rs`, 行 676-695 和 740-747

### 问题描述
如果 `stop()` 未被调用 (如 panic 或提前返回)，`Drop` 实现设置 `running = false` 但不 await worker 线程 handle。

### 影响
- Worker 线程被放弃 (detached tokio task)
- 进行中的写入未 flush 到磁盘
- Manifest 更新未持久化
- 资源泄漏

---

## 根本架构问题

Bug 1, 4, 5, 6, 7 的核心问题是 **Worker、调度器池和后台任务之间没有共享的取消机制**:
- `LocalScheduler` 中的 `CancellationToken` 只停止池工作线程
- 不取消已 dispatch 的 Tokio 任务
- `region.stop()` 或 `WorkerGroup::stop()` 无法向进行中的 flush/compaction/index 任务发出中止信号

## 总结

| # | 严重度 | 文件 | 描述 |
|---|--------|------|------|
| 1 | HIGH | worker.rs:279-294 | 调度器在 Worker 之前停止, 进行中任务未取消 |
| 2 | MEDIUM | scheduler.rs:115-146 | is_running() 和 try_send() 竞态 |
| 3 | MEDIUM | worker.rs:915-1005 | Stalled 请求在关闭时丢失 |
| 4 | HIGH | handle_drop.rs:78-98 | Region drop 后进行中任务未取消 |
| 5 | MEDIUM | flush.rs:661-672 | Flush 结果丢失, 调用者未收到通知 |
| 6 | LOW | worker.rs:943-956 | 关闭时调度到已停止的池 |
| 7 | MEDIUM | compaction/task.rs:318-323 | 后台任务向已关闭 channel 发送 |
| 8 | MEDIUM | worker.rs:676-695 | Drop 不 await worker handle |
