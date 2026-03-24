# Flow 引擎 Bug 修复 — 待办清单

## 阶段 A: Bug 修复代码改动

### engine.rs 改动

- [ ] **A1.** 修改 `remove_flow_inner` (行 664-681): 将 `self.tasks.write().await.remove(&flow_id)` 的返回值保存到 `task` 变量；将 `if .is_none()` 改为 `if task.is_none()`；在 `tx.send()` 之后添加 abort 逻辑: `if let Some(task) = task { let handle = task.state.write().unwrap().task_handle.take(); if let Some(handle) = handle { handle.abort(); } }`

- [ ] **A2.** 修改 `create_flow_inner` 的 `(_, true, true)` 分支 (行 396-398): 将 `info!("Replacing flow with id={}", flow_id);` 替换为调用 `self.remove_flow_inner(flow_id).await` 并用 `if let Err(err)` 包裹打印 warn

### task.rs 新增辅助方法

- [ ] **A3.** 在 `BatchingTask` impl 块中、`start_executing_loop` 之前，添加 `fn is_shutdown(&self) -> bool` 方法: 获取 `self.state.write().unwrap()` 的锁，调用 `state.shutdown_rx.try_recv()`，匹配 Ok→true, Closed→warn+true, Empty→false

- [ ] **A4.** 在 `BatchingTask` impl 块中添加 `async fn wait_shutdown(&self)`: 轮询循环，每次 `is_shutdown()` 返回 true 则 return，否则 `tokio::time::sleep(100ms)`

- [ ] **A5.** 在 `BatchingTask` impl 块中添加 `async fn sleep_with_shutdown_check(&self, duration: Duration) -> bool`: 使用 `tokio::select! { _ = sleep => false, _ = wait_shutdown => true }`

### task.rs 重写 start_executing_loop

- [ ] **A6a.** 循环顶部: 将 `self.state.write().unwrap()` + `try_recv()` + 3-way match 替换为 `if self.is_shutdown() { break; }`

- [ ] **A6b.** 在 `gen_insert_plan` 成功后、`execute_logical_plan` 调用前，插入 `if self.is_shutdown() { break; }`

- [ ] **A6c.** error 分支: 将 `tokio::time::sleep(min_refresh).await` 替换为 `if self.sleep_with_shutdown_check(min_refresh).await { break; }`

- [ ] **A6d.** `Ok(None)` 分支: 将 `tokio::time::sleep(min_refresh).await` + `continue` 替换为 `if self.sleep_with_shutdown_check(min_refresh).await { break; }` + `continue`

- [ ] **A6e.** `Ok(Some)` 分支有 interval: 将 `eval_interval.tick().await` 替换为 `tokio::select! { _ = eval_interval.tick() => {}, _ = self.wait_shutdown() => { break; } }`

- [ ] **A6f.** `Ok(Some)` 分支无 interval: 将 `tokio::time::sleep_until(sleep_until).await` 替换为 `tokio::select! { _ = tokio::time::sleep_until(sleep_until) => {}, _ = self.wait_shutdown() => { break; } }`

- [ ] **A6g.** 循环结束后添加 `info!("Flow {} execution loop exited", self.config.flow_id);`

### Import 确认

- [ ] **A7.** 确认 `task.rs` 顶部已有 `use std::time::Duration;` 和 `use tokio::time::error::TryRecvError;`
- [ ] **A8.** 确认 `engine.rs` 顶部已有 `use common_telemetry::tracing::warn;`

---

## 阶段 B: 编译验证

- [ ] **B1.** `cargo check -p flow` — 编译通过
- [ ] **B2.** `cargo clippy -p flow` — 无 lint 警告
- [ ] **B3.** `cargo test -p flow` — 现有测试全部通过

---

## 阶段 C: 测试基础设施搭建

- [ ] **C1.** 在 `engine.rs` 文件末尾新建 `#[cfg(test)] mod tests` 块，添加 use 导入: `super::*`, `create_test_query_engine`, `FrontendClient`, `GrpcQueryHandlerWithBoxedError`, `HandlerMutable`, `BatchingModeOptions`, `TableMetadataManager`, `FlowMetadataManager`, `MemoryKvBackend`, `new_memory_catalog_manager`, `QueryOptions`, `QueryContext`, `HashMap`, `Duration`, `Output`, `BoxedError`, `Request`

- [ ] **C2.** 在 tests 模块中定义 `struct NoopHandler;`，实现 `GrpcQueryHandlerWithBoxedError` trait: `do_query` 返回 `Ok(Output::new_with_affected_rows(0))`

- [ ] **C3.** 在 tests 模块中实现 `fn create_test_batching_engine() -> (BatchingEngine, HandlerMutable)`: 创建 MemoryKvBackend → TableMetadataManager + init() → FlowMetadataManager → MemoryCatalogManager → create_test_query_engine → FrontendClient::from_empty_grpc_handler → BatchingEngine::new

- [ ] **C4.** 在 tests 模块中实现 `async fn create_test_batching_engine_with_handler() -> BatchingEngine`: 调用 create_test_batching_engine → 创建 Arc<NoopHandler> → handler_mutable.set_handler → 返回 engine

- [ ] **C5.** 在 tests 模块中实现 `fn make_create_flow_args(flow_id: u64, sql: &str, or_replace: bool) -> CreateFlowArgs`: 构造 CreateFlowArgs，sink_table_name 为 `["greptime", "public", &format!("sink_{flow_id}")]`，source_table_ids 为 `vec![1024]`，eval_interval 为 `Some(60)`

---

## 阶段 D: 测试编写

- [ ] **D1.** 编写 `test_remove_flow_sends_shutdown_signal`: 创建 engine → create flow_id=1 → assert tasks/shutdown_txs 包含 1 → remove → assert 不再包含 → assert Ok

- [ ] **D2.** 编写 `test_remove_flow_aborts_task_handle`: 创建 engine → create flow_id=1 → remove → sleep 200ms → assert tasks map 为空

- [ ] **D3.** 编写 `test_replace_flow_cancels_old_task` (Bug 1 核心): 创建 engine → create flow_id=1 → save old tx → create flow_id=1 or_replace=true → assert len==1 → assert old tx 失效 → assert new tx 有效 → cleanup

- [ ] **D4.** 编写 `test_replace_nonexistent_flow_creates_normally`: 创建 engine → create flow_id=99 or_replace=true → assert len==1 → assert 存在

- [ ] **D5.** 编写 `test_remove_nonexistent_flow_returns_error`: 创建 engine (无 flow) → remove 999 → assert Err → assert FlowNotFound

- [ ] **D6.** 编写 `test_create_if_not_exists_returns_none_on_duplicate`: 创建 engine → create 1 → assert Ok(Some(1)) → create 1 with create_if_not_exists=true → assert Ok(None) → assert len==1

- [ ] **D7.** 编写 `test_create_duplicate_without_replace_returns_error`: 创建 engine → create 1 → create 1 again → assert Err → assert FlowAlreadyExist

- [ ] **D8.** 编写 `test_task_execution_loop_exits_on_shutdown_during_sleep` (Bug 3 核心): 创建 engine → create flow_id=1 eval_interval=10s → sleep 100ms → remove → timeout(2s, ...) assert 成功

- [ ] **D9.** 编写 `test_replace_then_verify_old_task_handle_aborted` (Bug 1+2 综合): 创建 engine → create 1 → assert handle 存在 → replace → sleep 200ms → assert len==1 → assert new tx 不同

- [ ] **D10.** 编写 `test_list_flows_after_create_and_remove`: 创建 engine → assert empty → create 1,2 → assert [1,2] → remove 1 → assert [2] → replace 2 → assert [2]

- [ ] **D11.** 编写 `test_concurrent_remove_and_replace` (并发压力): Arc<engine> → spawn 5 tasks 每个 create/replace/remove → join all → assert maps 为空

---

## 阶段 E: 最终验证

- [ ] **E1.** `cargo test -p flow` — 全部测试通过 (现有 11 个新测试 + 已有测试)
- [ ] **E2.** `cargo clippy -p flow` — 无警告
