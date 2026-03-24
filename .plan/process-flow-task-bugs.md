# Flow 引擎任务调度 Bug 修复实施计划

## 目标
修复 4 个任务调度 Bug，确保 Flow 替换和移除时旧任务被正确取消，不留僵尸任务。

## Bug 总览

| # | 严重度 | 核心问题 | 修改文件 |
|---|--------|----------|----------|
| 1 | HIGH | `or_replace=true` 时不取消旧任务 | engine.rs |
| 2 | MEDIUM | `remove_flow_inner` 不中止 JoinHandle | engine.rs + state.rs |
| 3 | LOW-MEDIUM | 查询执行期间不检查 shutdown | task.rs |
| 4 | LOW | 替换失败路径未清理 | engine.rs |

---

## 修改 1: Bug 1 + Bug 4 — `create_flow_inner` 中先取消旧任务

**文件**: `src/flow/src/batching_mode/engine.rs`

**改动位置**: 行 391-408 的 match 分支

**当前代码** (行 391-408):
```rust
// or replace logic
{
    let is_exist = self.tasks.read().await.contains_key(&flow_id);
    match (create_if_not_exists, or_replace, is_exist) {
        // if replace, ignore that old flow exists
        (_, true, true) => {
            info!("Replacing flow with id={}", flow_id);
        }
        (false, false, true) => FlowAlreadyExistSnafu { id: flow_id }.fail()?,
        // already exists, and not replace, return None
        (true, false, true) => {
            info!("Flow with id={} already exists, do nothing", flow_id);
            return Ok(None);
        }
        // continue as normal
        (_, _, false) => (),
    }
}
```

**修改后代码**:
```rust
// or replace logic
{
    let is_exist = self.tasks.read().await.contains_key(&flow_id);
    match (create_if_not_exists, or_replace, is_exist) {
        // if replace, first properly cancel the old task before creating new one
        (_, true, true) => {
            info!("Replacing flow with id={}", flow_id);
            // Gracefully shut down the old task: send shutdown signal + abort handle
            // This ensures the old task stops before the new one starts, preventing
            // two tasks from running concurrently on the same flow
            if let Err(err) = self.remove_flow_inner(flow_id).await {
                warn!(
                    "Failed to remove old flow {flow_id} during replace, proceeding anyway: {err}"
                );
            }
        }
        (false, false, true) => FlowAlreadyExistSnafu { id: flow_id }.fail()?,
        // already exists, and not replace, return None
        (true, false, true) => {
            info!("Flow with id={} already exists, do nothing", flow_id);
            return Ok(None);
        }
        // continue as normal
        (_, _, false) => (),
    }
}
```

**说明**:
- 在创建新任务之前，先调用 `remove_flow_inner()` 优雅地关闭旧任务
- `remove_flow_inner()` 会: 发送 shutdown 信号 → 中止 JoinHandle → 从 maps 中移除
- 如果 `remove_flow_inner()` 失败 (比如旧任务已死)，打印警告但继续创建新任务
- 这同时修复了 Bug 4: 旧任务无论如何都会被清理，不会在失败路径中残留

---

## 修改 2: Bug 2 — `remove_flow_inner` 中中止 JoinHandle

**文件**: `src/flow/src/batching_mode/engine.rs`

**改动位置**: 行 664-681 的 `remove_flow_inner` 函数

**当前代码** (行 664-681):
```rust
pub async fn remove_flow_inner(&self, flow_id: FlowId) -> Result<(), Error> {
    if self.tasks.write().await.remove(&flow_id).is_none() {
        warn!("Flow {flow_id} not found in tasks");
        FlowNotFoundSnafu { id: flow_id }.fail()?;
    }
    let Some(tx) = self.shutdown_txs.write().await.remove(&flow_id) else {
        UnexpectedSnafu {
            reason: format!("Can't found shutdown tx for flow {flow_id}"),
        }
        .fail()?
    };
    if tx.send(()).is_err() {
        warn!(
            "Fail to shutdown flow {flow_id} due to receiver already dropped, maybe flow {flow_id} is already dropped?"
        )
    }
    Ok(())
}
```

**修改后代码**:
```rust
pub async fn remove_flow_inner(&self, flow_id: FlowId) -> Result<(), Error> {
    let task = self.tasks.write().await.remove(&flow_id);
    if task.is_none() {
        warn!("Flow {flow_id} not found in tasks");
        FlowNotFoundSnafu { id: flow_id }.fail()?;
    }

    let Some(tx) = self.shutdown_txs.write().await.remove(&flow_id) else {
        UnexpectedSnafu {
            reason: format!("Can't found shutdown tx for flow {flow_id}"),
        }
        .fail()?
    };

    // Send graceful shutdown signal first, so the task loop can clean up
    if tx.send(()).is_err() {
        warn!(
            "Fail to shutdown flow {flow_id} due to receiver already dropped, maybe flow {flow_id} is already dropped?"
        )
    }

    // Abort the background tokio task handle to ensure it stops promptly.
    // The shutdown signal above gives the task a chance to clean up gracefully,
    // but if the task is stuck in a long-running query or sleep, we need abort()
    // to force it to stop. We do abort() after send() so the task gets a chance
    // to exit cleanly first (the abort is a safety net).
    if let Some(task) = task {
        let handle = task.state.write().unwrap().task_handle.take();
        if let Some(handle) = handle {
            handle.abort();
        }
    }

    Ok(())
}
```

**说明**:
- 先发送 shutdown 信号 (graceful)，再中止 handle (force)
- 先 graceful 再 force 的顺序确保任务有机会清理
- `abort()` 是幂等的 — 如果任务已经退出，abort 是 no-op
- `task_handle.take()` 确保 handle 只被中止一次

---

## 修改 3: Bug 3 — `start_executing_loop` 中查询执行期间响应 shutdown

**文件**: `src/flow/src/batching_mode/task.rs`

**改动位置**: 行 461-586 的 `start_executing_loop` 函数

需要修改两处:
1. 查询生成 (`gen_insert_plan`) 期间的 shutdown 响应
2. 查询执行 (`execute_logical_plan`) 期间的 shutdown 响应
3. 休眠期间的 shutdown 响应

**当前代码** (行 475-586 核心循环):
```rust
loop {
    // first check if shutdown signal is received
    {
        let mut state = self.state.write().unwrap();
        match state.shutdown_rx.try_recv() {
            Ok(()) => break,
            Err(TryRecvError::Closed) => {
                warn!("Unexpected shutdown flow {}, shutdown anyway", self.config.flow_id);
                break;
            }
            Err(TryRecvError::Empty) => (),
        }
    }
    // ... gen_insert_plan ...
    // ... execute_logical_plan ...
    // ... sleep ...
}
```

**修改后代码** (完整替换 start_executing_loop 函数):

需要添加一个辅助方法来检查 shutdown，并修改循环中的关键操作:

```rust
/// start executing query in a loop, break when receive shutdown signal
///
/// any error will be logged when executing query
pub async fn start_executing_loop(
    &self,
    engine: QueryEngineRef,
    frontend_client: Arc<FrontendClient>,
) {
    let flow_id_str = self.config.flow_id.to_string();
    let mut max_window_cnt = None;
    let mut interval = self
        .config
        .flow_eval_interval
        .map(|d| tokio::time::interval(d));
    if let Some(tick) = &mut interval {
        tick.tick().await; // pass the first tick immediately
    }
    loop {
        // first check if shutdown signal is received
        // if so, break the loop
        if self.is_shutdown() {
            break;
        }

        METRIC_FLOW_BATCHING_ENGINE_START_QUERY_CNT
            .with_label_values(&[&flow_id_str])
            .inc();

        let min_refresh = self.config.batch_opts.experimental_min_refresh_duration;

        let new_query = match self.gen_insert_plan(&engine, max_window_cnt).await {
            Ok(new_query) => new_query,
            Err(err) => {
                common_telemetry::error!(err; "Failed to generate query for flow={}", self.config.flow_id);
                // also sleep for a little while before try again to prevent flooding logs
                if self.sleep_with_shutdown_check(min_refresh).await {
                    break;
                }
                continue;
            }
        };

        // Check shutdown before executing the potentially long-running query
        if self.is_shutdown() {
            break;
        }

        let res = if let Some(new_query) = &new_query {
            self.execute_logical_plan(&frontend_client, &new_query.plan)
                .await
        } else {
            Ok(None)
        };

        match res {
            // normal execute, sleep for some time before doing next query
            Ok(Some(_)) => {
                // can increase max_window_cnt to query more windows next time
                max_window_cnt = max_window_cnt.map(|cnt| {
                    (cnt + 1).min(self.config.batch_opts.experimental_max_filter_num_per_query)
                });

                // here use proper ticking if set eval interval
                if let Some(eval_interval) = &mut interval {
                    // Use select! to respond to shutdown during tick wait
                    tokio::select! {
                        _ = eval_interval.tick() => {}
                        _ = self.wait_shutdown() => { break; }
                    }
                } else {
                    // if not explicitly set, just automatically calculate next start time
                    // using time window size and more args
                    let sleep_until = {
                        let state = self.state.write().unwrap();

                        let time_window_size = self
                            .config
                            .time_window_expr
                            .as_ref()
                            .and_then(|t| *t.time_window_size());

                        state.get_next_start_query_time(
                            self.config.flow_id,
                            &time_window_size,
                            min_refresh,
                            Some(self.config.batch_opts.query_timeout),
                            self.config.batch_opts.experimental_max_filter_num_per_query,
                        )
                    };

                    // Use select! to respond to shutdown during sleep
                    tokio::select! {
                        _ = tokio::time::sleep_until(sleep_until) => {}
                        _ = self.wait_shutdown() => { break; }
                    }
                };
            }
            // no new data, sleep for some time before checking for new data
            Ok(None) => {
                debug!(
                    "Flow id = {:?} found no new data, sleep for {:?} then continue",
                    self.config.flow_id, min_refresh
                );
                if self.sleep_with_shutdown_check(min_refresh).await {
                    break;
                }
                continue;
            }
            Err(err) => {
                METRIC_FLOW_BATCHING_ENGINE_ERROR_CNT
                    .with_label_values(&[&flow_id_str])
                    .inc();
                match new_query {
                    Some(query) => {
                        common_telemetry::error!(err; "Failed to execute query for flow={} with query: {}", self.config.flow_id, query.plan);
                        self.state.write().unwrap().dirty_time_windows.add_windows(
                            query.filter.map(|f| f.time_ranges).unwrap_or_default(),
                        );
                        max_window_cnt = Some(1);
                    }
                    None => {
                        common_telemetry::error!(err; "Failed to generate query for flow={}", self.config.flow_id)
                    }
                }
                if self.sleep_with_shutdown_check(min_refresh).await {
                    break;
                }
            }
        }
    }
    info!("Flow {} execution loop exited", self.config.flow_id);
}
```

**同时在 `BatchingTask` impl 块中添加两个辅助方法** (添加在 `start_executing_loop` 之前):

```rust
/// Check if shutdown signal has been received (non-blocking).
/// Returns true if the task should stop.
fn is_shutdown(&self) -> bool {
    let mut state = self.state.write().unwrap();
    match state.shutdown_rx.try_recv() {
        Ok(()) => true,
        Err(TryRecvError::Closed) => {
            warn!(
                "Unexpected shutdown flow {}, shutdown anyway",
                self.config.flow_id
            );
            true
        }
        Err(TryRecvError::Empty) => false,
    }
}

/// Wait for shutdown signal (blocking). Used inside tokio::select!.
/// Returns when shutdown is signaled.
async fn wait_shutdown(&self) {
    // We can't easily use the oneshot::Receiver directly from a shared reference
    // because it's behind a RwLock. Instead, we poll periodically.
    loop {
        if self.is_shutdown() {
            return;
        }
        tokio::time::sleep(Duration::from_millis(100)).await;
    }
}

/// Sleep for the given duration, but return early if shutdown is signaled.
/// Returns true if shutdown was signaled, false if the sleep completed normally.
async fn sleep_with_shutdown_check(&self, duration: Duration) -> bool {
    tokio::select! {
        _ = tokio::time::sleep(duration) => false,
        _ = self.wait_shutdown() => true,
    }
}
```

**说明**:
- `is_shutdown()` 封装了 `try_recv()` 逻辑，避免重复代码
- `wait_shutdown()` 提供阻塞等待 shutdown 的能力 (轮询方式，100ms 间隔)
- `sleep_with_shutdown_check()` 使用 `tokio::select!` 在休眠期间响应 shutdown
- 所有 `tokio::time::sleep` 和 `interval.tick()` 都改为 `tokio::select!` 模式
- 添加退出日志 `"Flow {} execution loop exited"` 便于调试

---

## 修改 4: 添加 `use` 导入

**文件**: `src/flow/src/batching_mode/task.rs`

在文件顶部的 import 区域，确保以下导入存在:

```rust
use std::time::Duration;  // 已存在，确认即可
```

无需额外导入，因为 `Duration` 和 `warn!` 都已导入。

---

## 修改 5: 添加测试

### 测试基础设施

`engine.rs` 目前**没有任何测试**。需要新建 `#[cfg(test)] mod tests` 模块。

测试依赖以下已有的基础设施 (无需额外 mock):

| 依赖 | 来源 | 说明 |
|------|------|------|
| `create_test_query_engine()` | `src/flow/src/test_utils.rs:77` | 返回真实的 DataFusion 引擎, 已注册 `numbers_with_ts` 等内存表 |
| `FrontendClient::from_empty_grpc_handler()` | `src/flow/src/batching_mode/frontend_client.rs:110` | 创建空的 Standalone 客户端 + `HandlerMutable` |
| `FrontendClient::from_grpc_handler()` | `src/flow/src/batching_mode/frontend_client.rs:159` | 使用 `Weak<GrpcQueryHandlerWithBoxedError>` 创建已初始化的客户端 |
| `NoopHandler` (需在测试模块中定义) | 参考 `frontend_client.rs:503-515` | 实现 `GrpcQueryHandlerWithBoxedError`, 所有查询返回 `Ok(Output::new_with_affected_rows(0))` |
| `new_memory_catalog_manager()` | `catalog::memory` | 内存 CatalogManager |
| `MemoryKvBackend` | `common_meta::kv_backend::memory` | 内存 KV 后端 |
| `TableMetadataManager::new(kv_backend)` | `common_meta::key` | 表元数据管理器 |
| `FlowMetadataManager::new(kv_backend)` | `common_meta::key::flow` | Flow 元数据管理器 |

### 测试工具函数 (在测试模块中定义)

**`create_test_batching_engine()`** — 创建最小可用的 `BatchingEngine`:
```
输入: 无
输出: (BatchingEngine, HandlerMutable)
步骤:
1. 创建 MemoryKvBackend
2. 创建 TableMetadataManager, 调用 init()
3. 创建 FlowMetadataManager
4. 创建 MemoryCatalogManager
5. 调用 create_test_query_engine() 获取查询引擎
6. 调用 FrontendClient::from_empty_grpc_handler() 获取客户端和 handler
7. 创建 BatchingEngine::new(...)
8. 返回 (engine, handler_mutable)
```

**`create_test_batching_engine_with_handler()`** — 创建带 NoopHandler 的引擎:
```
输入: 无
输出: BatchingEngine
步骤:
1. 调用 create_test_batching_engine()
2. 创建 Arc<NoopHandler> 并通过 handler_mutable.set_handler() 注入
3. 返回 engine
```

**`make_create_flow_args(flow_id, sql, or_replace)`** — 创建 CreateFlowArgs:
```
输入: flow_id: u64, sql: &str, or_replace: bool
输出: CreateFlowArgs
步骤:
1. 构造 CreateFlowArgs, 填充:
   - flow_id
   - sink_table_name: ["greptime", "public", "sink_{flow_id}"]
   - source_table_ids: vec![1024] (对应 numbers_with_ts)
   - create_if_not_exists: false
   - or_replace
   - expire_after: None
   - eval_interval: Some(60)
   - sql: sql.to_string()
   - flow_options: HashMap::new()
   - query_ctx: Some(QueryContext::with("greptime", "public"))
```

**`NoopHandler` struct** — 模拟 Frontend 查询处理器:
```
实现 GrpcQueryHandlerWithBoxedError trait
do_query() 返回 Ok(Output::new_with_affected_rows(0))
用于让 BatchingTask 的执行循环能正常运行而不报错
```

---

### 测试用例 1: `test_remove_flow_sends_shutdown_signal`

**目标**: 验证 `remove_flow_inner()` 能正确发送 shutdown 信号。

**策略**: 创建一个 Flow, 获取其 shutdown receiver 的引用, 调用 remove, 验证 receiver 收到信号。

**步骤**:
```
1. 创建带 NoopHandler 的 BatchingEngine
2. 调用 create_flow_inner(args) 创建 flow_id=1
3. 验证 tasks map 中存在 flow_id=1
4. 验证 shutdown_txs map 中存在 flow_id=1
5. 调用 remove_flow_inner(flow_id=1)
6. 验证 tasks map 中不再有 flow_id=1
7. 验证 shutdown_txs map 中不再有 flow_id=1
8. 验证 remove_flow_inner 返回 Ok(())
```

**预期结果**: Flow 被成功移除, 两个 map 都不再包含该 flow_id。

---

### 测试用例 2: `test_remove_flow_aborts_task_handle`

**目标**: 验证 `remove_flow_inner()` 能中止 JoinHandle (Bug 2 修复验证)。

**策略**: 创建一个长时间运行的 Flow (通过空 handler 使其执行循环持续运行), 调用 remove, 验证任务确实停止。

**步骤**:
```
1. 创建带 NoopHandler 的 BatchingEngine
2. 调用 create_flow_inner(args) 创建 flow_id=1
3. 从 tasks map 中获取 task 的 JoinHandle 引用 (clone task 后读取)
4. 记录 handle 的状态 (is_finished 应为 false, 因为循环刚开始)
5. 调用 remove_flow_inner(flow_id=1)
6. 等待一小段时间 (tokio::time::sleep(200ms))
7. 验证 handle 现在已完成 (通过检查 task 已从 map 移除, 且 abort 已被调用)
    - 直接验证: tasks map 中不再有 flow_id=1
    - 间接验证: shutdown_txs map 中不再有 flow_id=1
```

**预期结果**: 任务被中止, 不再消耗资源。

---

### 测试用例 3: `test_replace_flow_cancels_old_task` (Bug 1 核心验证)

**目标**: 验证 `or_replace=true` 时旧任务被取消, 新任务正常运行。

**策略**: 创建一个 Flow, 然后用 `or_replace=true` 创建同 ID 的新 Flow, 验证:
- 旧任务的 shutdown 信号被发送
- 旧任务的 JoinHandle 被中止
- 新任务正常存在于 maps 中
- 最终只有一个任务 (不是两个)

**步骤**:
```
1. 创建带 NoopHandler 的 BatchingEngine
2. 创建 flow_id=1, or_replace=false (首次创建)
3. 验证 tasks len == 1, shutdown_txs len == 1
4. 获取旧任务的 shutdown_tx (clone 引用, 用于后续验证)
5. 创建 flow_id=1, or_replace=true (替换)
6. 验证 tasks len == 1 (不是 2!)
7. 验证 shutdown_txs len == 1 (不是 2!)
8. 验证旧的 shutdown_tx 已失效 (尝试 send 会失败, 因为已被 remove_flow_inner 移除)
9. 验证新的 shutdown_tx 有效 (尝试 send 成功)
10. 清理: 调用 remove_flow_inner(1)
```

**预期结果**: 替换后只有一个任务在运行, 旧任务被完全清理。

---

### 测试用例 4: `test_replace_nonexistent_flow_creates_normally`

**目标**: 验证当 Flow 不存在时, `or_replace=true` 的行为等同于正常创建。

**步骤**:
```
1. 创建带 NoopHandler 的 BatchingEngine
2. 创建 flow_id=99, or_replace=true (Flow 不存在)
3. 验证 tasks len == 1
4. 验证 shutdown_txs len == 1
5. 验证 flow_id=99 在两个 map 中都存在
```

**预期结果**: 正常创建, 无错误。

---

### 测试用例 5: `test_remove_nonexistent_flow_returns_error`

**目标**: 验证移除不存在的 Flow 返回 `FlowNotFound` 错误。

**步骤**:
```
1. 创建带 NoopHandler 的 BatchingEngine (不创建任何 Flow)
2. 调用 remove_flow_inner(flow_id=999)
3. 验证返回 Err
4. 验证错误类型是 FlowNotFound (通过 pattern match 或 err.to_string() 检查)
```

**预期结果**: 返回 `FlowNotFoundSnafu` 错误。

---

### 测试用例 6: `test_create_if_not_exists_returns_none_on_duplicate`

**目标**: 验证 `create_if_not_exists=true` 时重复创建返回 `None`。

**步骤**:
```
1. 创建带 NoopHandler 的 BatchingEngine
2. 创建 flow_id=1, create_if_not_exists=false
3. 验证返回 Ok(Some(1))
4. 创建 flow_id=1, create_if_not_exists=true
5. 验证返回 Ok(None) (不是 Err!)
6. 验证 tasks len == 1 (只有一个, 不是两个)
```

**预期结果**: 第二次创建返回 None, 不报错, 不重复。

---

### 测试用例 7: `test_create_duplicate_without_replace_returns_error`

**目标**: 验证 `create_if_not_exists=false, or_replace=false` 时重复创建返回错误。

**步骤**:
```
1. 创建带 NoopHandler 的 BatchingEngine
2. 创建 flow_id=1, create_if_not_exists=false, or_replace=false
3. 验证返回 Ok(Some(1))
4. 再次创建 flow_id=1, 同样的参数
5. 验证返回 Err
6. 验证错误类型是 FlowAlreadyExist
```

**预期结果**: 返回 `FlowAlreadyExistSnafu` 错误。

---

### 测试用例 8: `test_task_execution_loop_exits_on_shutdown_during_sleep`

**目标**: 验证 Bug 3 修复: 任务在休眠期间能响应 shutdown 信号。

**策略**: 创建一个 eval_interval 很长 (10 秒) 的 Flow, 立即发送 shutdown, 验证任务在远短于 10 秒内退出。

**步骤**:
```
1. 创建带 NoopHandler 的 BatchingEngine
2. 创建 flow_id=1, eval_interval=Some(10) (10 秒间隔)
3. 等待一小段时间 (100ms) 让执行循环进入休眠阶段
4. 调用 remove_flow_inner(1) 发送 shutdown 信号
5. 在 2 秒内等待任务退出 (tokio::time::timeout(2s, ...))
6. 验证 timeout 成功 (任务在 2 秒内退出, 而非等待完整的 10 秒)
```

**预期结果**: 任务在 2 秒内退出, 证明休眠期间正确响应了 shutdown。

---

### 测试用例 9: `test_replace_then_verify_old_task_handle_aborted`

**目标**: 综合验证 Bug 1 + Bug 2: 替换 Flow 时旧任务的 JoinHandle 被中止。

**策略**: 创建一个长时间运行的 Flow, 替换它, 验证旧 handle 被中止。

**步骤**:
```
1. 创建带 NoopHandler 的 BatchingEngine
2. 创建 flow_id=1 (首次)
3. 从 tasks map 获取旧 task 的 clone, 读取其 state 中的 task_handle
4. 验证 handle 存在且未完成
5. 创建 flow_id=1, or_replace=true (替换)
6. 等待 200ms 让 abort 生效
7. 验证 tasks map 中只有 1 个 task
8. 验证新 task 的 shutdown_tx 与旧的不同 (说明是新创建的)
```

**预期结果**: 旧 handle 被中止, 新任务正常运行。

---

### 测试用例 10: `test_list_flows_after_create_and_remove`

**目标**: 验证 `list_flows()` 在创建和移除后的正确性。

**步骤**:
```
1. 创建带 NoopHandler 的 BatchingEngine
2. 验证 list_flows() 返回空
3. 创建 flow_id=1 和 flow_id=2
4. 验证 list_flows() 包含 [1, 2]
5. 移除 flow_id=1
6. 验证 list_flows() 只包含 [2]
7. 替换 flow_id=2 (or_replace=true, 使用不同的 SQL)
8. 验证 list_flows() 只包含 [2]
```

**预期结果**: list_flows() 始终反映当前状态。

---

### 测试用例 11: `test_concurrent_remove_and_replace`

**目标**: 验证并发操作的线程安全性 (压力测试)。

**策略**: 同时创建多个 Flow, 随机替换和移除, 验证最终状态一致。

**步骤**:
```
1. 创建带 NoopHandler 的 BatchingEngine (Arc<BatchingEngine>)
2. spawn 5 个并发任务, 每个:
   a. 创建 flow_id=i (i=0..4)
   b. 随机等待 0-50ms
   c. 替换 flow_id=i (or_replace=true)
   d. 随机等待 0-50ms
   e. 移除 flow_id=i
3. 等待所有任务完成
4. 验证 tasks map 为空
5. 验证 shutdown_txs map 为空
```

**预期结果**: 无 panic, 无死锁, 最终状态一致。

---

### 测试模块完整结构

```
src/flow/src/batching_mode/engine.rs

#[cfg(test)]
mod tests {
    // 1. 导入依赖
    //    use super::*;
    //    use crate::test_utils::create_test_query_engine;
    //    use crate::batching_mode::frontend_client::{FrontendClient, GrpcQueryHandlerWithBoxedError, HandlerMutable};
    //    use crate::batching_mode::BatchingModeOptions;
    //    use common_meta::key::TableMetadataManager;
    //    use common_meta::key::flow::FlowMetadataManager;
    //    use common_meta::kv_backend::memory::MemoryKvBackend;
    //    use catalog::memory::new_memory_catalog_manager;
    //    use query::options::QueryOptions;
    //    use session::context::QueryContext;
    //    use std::collections::HashMap;
    //    use std::time::Duration;

    // 2. NoopHandler 定义

    // 3. 工具函数:
    //    - create_test_batching_engine() -> (BatchingEngine, HandlerMutable)
    //    - create_test_batching_engine_with_handler() -> BatchingEngine
    //    - make_create_flow_args(flow_id, sql, or_replace) -> CreateFlowArgs

    // 4. 测试函数 (11 个):
    //    - test_remove_flow_sends_shutdown_signal
    //    - test_remove_flow_aborts_task_handle
    //    - test_replace_flow_cancels_old_task
    //    - test_replace_nonexistent_flow_creates_normally
    //    - test_remove_nonexistent_flow_returns_error
    //    - test_create_if_not_exists_returns_none_on_duplicate
    //    - test_create_duplicate_without_replace_returns_error
    //    - test_task_execution_loop_exits_on_shutdown_during_sleep
    //    - test_replace_then_verify_old_task_handle_aborted
    //    - test_list_flows_after_create_and_remove
    //    - test_concurrent_remove_and_replace
}
```

---

## 修改文件清单

| 文件 | 修改类型 | 行范围 |
|------|----------|--------|
| `src/flow/src/batching_mode/engine.rs` | 修改 match 分支 | 391-408 |
| `src/flow/src/batching_mode/engine.rs` | 修改 remove_flow_inner | 664-681 |
| `src/flow/src/batching_mode/task.rs` | 添加辅助方法 | 新增 (约 30 行) |
| `src/flow/src/batching_mode/task.rs` | 重写 start_executing_loop | 461-586 |

## 验证步骤

1. `cargo check -p flow` — 确认编译通过
2. `cargo test -p flow` — 确认现有测试通过
3. `cargo clippy -p flow` — 确认无 lint 警告
4. 手动验证替换场景: 创建 Flow → 替换 Flow → 确认旧任务停止日志输出
