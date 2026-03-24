# flow/ 模块深度研究报告

## 概述

`flow/` 目录实现 GreptimeDB 的连续聚合引擎，支持流式(Streaming)和批处理(Batching)两种模式。流式模式使用 dfir_rs 数据流图，批处理模式使用独立的 Tokio 任务。

## 文件结构

```
src/flow/src/
├── lib.rs
├── adapter.rs                    -- StreamingEngine 和适配器
├── adapter/
│   ├── worker.rs                 -- Worker 线程管理
│   ├── server.rs                 -- 流式引擎服务器
│   └── flownode_impl.rs          -- FlowDualEngine 实现
├── batching_mode/
│   ├── engine.rs                 -- BatchingEngine 编排器 ★
│   ├── task.rs                   -- BatchingTask 单独任务 ★
│   ├── state.rs                  -- TaskState 可变状态
│   └── ...
├── heartbeat.rs
├── metrics.rs
└── ...
```

## 双引擎架构

### FlowDualEngine
同时管理流式和批处理引擎，根据 Flow 类型路由创建/删除/插入操作:

```rust
// flownode_impl.rs:63-75
FlowDualEngine {
    streaming_engine: StreamingEngine,
    batching_engine: BatchingEngine,
}
```

### StreamingEngine
- 多线程 Worker 池模型
- Worker 运行在专用 OS 线程上，通过 dfir_rs 处理数据流图
- Worker 使用 `AtomicBool` shutdown 标志
- `WorkerHandle` 在 Drop 时调用 `shutdown()`

### BatchingEngine
- 每个 Flow 一个 Tokio 任务
- 每个任务运行独立的 `start_executing_loop()`

## Batching Mode 任务生命周期

### 创建 (`engine.rs:376-537`, `create_flow_inner`)
1. 解析 SQL 为逻辑计划
2. 创建 `oneshot::channel` 用于 shutdown 信号
3. 创建 `BatchingTask` 包含 shutdown receiver
4. 通过 `common_runtime::spawn_global` 启动后台任务
5. 存储 `JoinHandle` 到 `task.state.task_handle`
6. 将任务插入 `self.tasks` map
7. 将 shutdown sender 插入 `self.shutdown_txs`

### 执行循环 (`task.rs:461-586`, `start_executing_loop`)
```rust
loop {
    // 1. 检查 shutdown 信号
    match self.config.shutdown_rx.try_recv() {
        Ok(_) | Err(TryRecvError::Closed) => break,
        Err(TryRecvError::Empty) => {}
    }
    // 2. 从 dirty windows 生成查询计划
    let plan = self.gen_insert_plan().await?;
    // 3. 通过 frontend client 执行查询
    self.execute_logical_plan(plan).await?;
    // 4. 休眠到下一个评估间隔
    tokio::time::sleep(eval_interval).await;
}
```

### 删除 (`engine.rs:664-681`, `remove_flow_inner`)
1. 从 `self.tasks` map 中移除任务
2. 从 `self.shutdown_txs` 中移除 shutdown sender
3. 通过 `tx.send(())` 发送 shutdown 信号
4. 后台任务通过 `try_recv()` 检测信号并退出循环

## 核心类型

### BatchingTask
```rust
pub struct BatchingTask {
    config: Arc<TaskConfig>,
    state: Arc<RwLock<TaskState>>,
}
```

### TaskState
```rust
pub struct TaskState {
    shutdown_rx: oneshot::Receiver<()>,
    task_handle: Option<JoinHandle<()>>,
    // ...
}
```

### BatchingEngine
```rust
pub struct BatchingEngine {
    tasks: RwLock<HashMap<FlowId, BatchingTask>>,
    shutdown_txs: RwLock<HashMap<FlowId, oneshot::Sender<()>>>,
    // ...
}
```

## 调度模式

- **请求驱动**: Worker 通过 inter-thread channels 接收请求
- **周期执行**: BatchingTask 按 `eval_interval` 周期执行
- **Dirty window 追踪**: 只处理有新数据的时间窗口
- **Error 重试**: 执行错误后休眠 `min_refresh` 再重试

## 已知问题

详见 `.research/flow-task-bugs.md`
