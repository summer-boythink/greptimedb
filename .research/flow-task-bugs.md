# Flow 引擎任务调度 Bug 报告

## 问题概述
用户报告: "系统有时会运行一些本应取消的任务"。Flow 引擎在 Flow 替换时旧任务未被正确取消。

---

## Bug 1 (HIGH): Flow 替换时旧任务未显式取消

**文件**: `/workspace/greptimedb/src/flow/src/batching_mode/engine.rs`, 行 391-408 和 530-534

### 问题描述
当 `or_replace=true` 时，代码**没有**:
1. 调用 `remove_flow_inner()` 来正确取消旧任务
2. 在创建新任务前向旧任务发送 shutdown 信号
3. 中止旧任务的 `JoinHandle`

### 代码分析
`create_flow_inner` 中的替换逻辑 (行 391-408):
```rust
match (create_if_not_exists, or_replace, is_exist) {
    (_, true, true) => {
        info!("Replacing flow with id={}", flow_id);
        // 仅仅打印日志，没有取消旧任务！
    }
    // ...
}
// 继续创建新任务，不先移除旧的
```

新任务创建后 (行 530-534):
```rust
let replaced_old_task_opt = self.tasks.write().await.insert(flow_id, task);  // 行 531
drop(replaced_old_task_opt);  // 行 532 - 旧任务被 drop 但仅是 Arc 引用

self.shutdown_txs.write().await.insert(flow_id, tx);  // 行 534 - 旧 sender 被隐式 drop
```

### 竞态窗口
旧 shutdown sender 在行 534 的 `insert()` 调用中被隐式 drop。这使旧 receiver 收到 `TryRecvError::Closed`，旧任务在下一次循环迭代时退出。

**但问题在于**: 旧任务可能已经在执行查询或休眠中。在下一个 shutdown check 之前:
- 旧任务和新任务同时运行
- 两者都执行查询，可能写入同一个 sink 表
- 消耗双倍资源

### 影响
- 旧任务继续运行，与新任务冲突
- 消耗额外的 CPU、内存、frontend 连接
- 可能产生数据重复或不一致

---

## Bug 2 (MEDIUM): JoinHandle 在移除时未显式中止

**文件**: `/workspace/greptimedb/src/flow/src/batching_mode/engine.rs`, 行 664-681

### 问题描述
`remove_flow_inner()` 发送 shutdown 信号但**不中止**任务的 `JoinHandle`。Handle 存储在 `task.state.task_handle` 中，但任务从 map 移除后无法访问 handle。

### 代码分析
```rust
async fn remove_flow_inner(&self, flow_id: FlowId) -> Result<()> {
    let task = self.tasks.write().await.remove(&flow_id);  // 从 map 移除
    let tx = self.shutdown_txs.write().await.remove(&flow_id);
    
    if let Some(tx) = tx {
        tx.send(()).ok();  // 发送 shutdown 信号
    }
    // 注意: 没有调用 task_handle.abort()
    // task.state.task_handle 中的 JoinHandle 未被中止
    Ok(())
}
```

### 影响
Shutdown 信号发出后，任务继续运行直到下一次循环迭代。如果任务正在:
- 执行长时间查询 (可达 query_timeout，默认 10 分钟)
- 休眠长 eval_interval
- 错误后重试 (休眠 min_refresh)

任务变成"僵尸" -- 不再被跟踪但仍在消耗资源。

---

## Bug 3 (LOW-MEDIUM): Shutdown check 和查询执行之间的竞态

**文件**: `/workspace/greptimedb/src/flow/src/batching_mode/task.rs`, 行 475-586

### 问题描述
Shutdown 信号只在循环**顶部**检查 (行 478-491)。一旦任务进入 `gen_insert_plan()` (行 498) 或 `execute_logical_plan()` (行 509)，在查询完成前不会再次检查 shutdown 信号。这些操作可达 `query_timeout` (默认 10 分钟)。

### 代码分析
```rust
loop {
    // Shutdown 只在这里检查
    match self.config.shutdown_rx.try_recv() {
        Ok(_) | Err(TryRecvError::Closed) => break,
        Err(TryRecvError::Empty) => {}
    }
    
    // 以下操作期间不检查 shutdown
    let plan = self.gen_insert_plan().await?;  // 行 498
    self.execute_logical_plan(plan).await?;    // 行 509
    tokio::time::sleep(eval_interval).await;   // 行 524-547
}
```

### 影响
已取消的任务可能在取消信号发出后继续运行长达 10 分钟。

---

## Bug 4 (LOW): 替换失败路径中 shutdown_txs 未清理

**文件**: `/workspace/greptimedb/src/flow/src/batching_mode/engine.rs`, 行 515-536

### 问题描述
如果新任务创建失败 (如 `BatchingTask::try_new()` 返回错误或 `check_or_create_sink_table()` 失败)，旧任务未被移除。替换逻辑已通过但旧任务仍在 `self.tasks` 中运行。

---

## 建议修复

### Bug 1 修复
在 `create_flow_inner` 的替换分支中先取消旧任务:
```rust
(_, true, true) => {
    info!("Replacing flow with id={}", flow_id);
    self.remove_flow_inner(flow_id).await?;  // 先取消旧任务
}
```

### Bug 2 修复
在 `remove_flow_inner` 中中止 handle:
```rust
async fn remove_flow_inner(&self, flow_id: FlowId) -> Result<()> {
    let task = self.tasks.write().await.remove(&flow_id);
    let tx = self.shutdown_txs.write().await.remove(&flow_id);
    
    if let Some(tx) = tx {
        tx.send(()).ok();
    }
    
    // 中止 JoinHandle
    if let Some(task) = task {
        let handle = task.state.write().await.task_handle.take();
        if let Some(handle) = handle {
            handle.abort();
        }
    }
    Ok(())
}
```

### Bug 3 修复
在执行查询时使用 `tokio::select!` 同时监听 shutdown 信号:
```rust
tokio::select! {
    result = self.execute_logical_plan(plan) => { result? }
    _ = self.config.shutdown_rx.recv() => { break }
}
```
