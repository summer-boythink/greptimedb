# Procedure 系统任务调度 Bug 报告

## 问题概述
用户报告: "系统有时会运行一些本应取消的任务"。Procedure 系统存在多个竞态条件和通知丢失问题。

---

## Bug 1 (CRITICAL): `on_suspended` 中通知丢失导致父过程永久挂起

**文件**: `/workspace/greptimedb/src/common/procedure/src/local/runner.rs`, 行 554-623

### 根本原因
`tokio::sync::Notify` **不存储通知**（如果没有人在等待）。代码自己的注释在 `local.rs:57-61` 明确警告:

```rust
/// [Notify] is not a condition variable, we can't guarantee the waiters are notified
/// if they didn't call `notified()` before we signal the notify.
```

### 竞态窗口

```rust
// runner.rs 行 591-615
for subprocedure in subprocedures {
    self.submit_subprocedure(...);  // 在 global runtime 上 spawn 子过程
}
// <--- 竞态窗口: 子过程可以在这里完成并调用 notify_by_subprocedure()

if has_child {
    self.meta.child_notify.notified().await;  // 太迟 - 通知已丢失
}
```

在 `submit_subprocedure` (行 528):
```rust
let _handle = common_runtime::spawn_global(async move {
    runner.run().trace(span).await  // 子过程完成，guard drop，调用 notify_by_subprocedure()
});
```

当 `ProcedureGuard` drop 时 (行 63-91):
```rust
if let Some(parent_id) = self.meta.parent_id {
    self.manager_ctx.notify_by_subprocedure(parent_id);  // 调用 notify_one()
}
```

### 触发场景
快速子过程在父过程调用 `notified().await` 之前完成。子过程的 `ProcedureGuard::drop()` 调用 `notify_one()`，但此时父过程还没有调用 `notified()`。通知丢失。父过程随后在行 615 的 `self.meta.child_notify.notified().await` 永久阻塞。

### 影响
- 父过程永久挂起等待已完成的子过程
- 用户观察此过程时看到它卡住
- **这是用户报告的最可能根因**：应该取消的任务看起来仍在运行（实际上是挂起）

---

## Bug 2 (HIGH): `submit_root()` 和 `stop()` 之间的 TOCTOU 竞态

**文件**: `/workspace/greptimedb/src/common/procedure/src/local.rs`, 行 617-691

### 问题描述
`running()` 检查和 spawn 不是原子的。

```rust
fn submit_root(...) -> Result<Watcher> {
    // 步骤 1: 检查 running (行 624)
    ensure!(self.manager_ctx.running(), ManagerNotStartSnafu);

    // ... meta 创建, runner 设置 (行 626-674) ...

    // 步骤 2: 插入和 spawn (行 671-688)
    self.manager_ctx.try_insert_procedure(meta);

    let _handle = common_runtime::spawn_global(async move {
        runner.run().trace(span).await;  // 即使 stop() 已被调用也会运行
    });
}
```

如果 `stop()` 在步骤 1 和步骤 2 之间被调用:
- `running()` 在步骤 1 是 `true` (过程被接受)
- `stop()` 在行 839 设置 `running = false`
- 过程仍然在步骤 2 被 spawn

spawned runner 开始执行。虽然 runner 在执行循环中检查 `running()`，但它首先获取锁 (行 164-173) **不检查** `running()`。

### 影响
- 已停止的管理器中仍可运行过程
- 可能获取不必要的锁

---

## Bug 3 (HIGH): `submit_subprocedure()` 不检查 `running()`

**文件**: `/workspace/greptimedb/src/common/procedure/src/local/runner.rs`, 行 476-540

### 问题描述
与 `submit_root()` 不同，`submit_subprocedure()` 从不检查管理器是否正在运行。

```rust
fn submit_subprocedure(...) {
    if self.manager_ctx.contains_procedure(procedure_id) {
        return;
    }
    // 这里没有检查 self.running()!
    // ... 创建 meta, 插入, spawn runner ...
    let _handle = common_runtime::spawn_global(async move {
        runner.run().trace(span).await  // 即使管理器已停止也会运行
    });
}
```

### 影响
在管理器停止后仍可 spawn 子过程。这些任务在已停止的管理器中运行，可能获取锁且永远不会被正确等待或取消。

---

## Bug 4 (MEDIUM): `wait_on_err()` 休眠不响应取消

**文件**: `/workspace/greptimedb/src/common/procedure/src/local/runner.rs`, 行 543-552

### 问题描述
```rust
async fn wait_on_err(&mut self, d: Duration, i: u64) {
    info!("Procedure {}-{} retry for the {} times after {} millis", ...);
    time::sleep(d).await;  // 盲休眠 - 无取消检查
}
```

在重试延迟期间（行 257, 272），如果管理器停止，runner 休眠整个持续时间后才检查 `running()`。使用指数退避时，这可能是长延迟。

### 影响
在最坏情况下（最大重试次数 + 大延迟），已停止的管理器看起来仍在运行过程数秒。

---

## Bug 5 (MEDIUM): `Runner::run()` 获取锁后无 `running()` 检查

**文件**: `/workspace/greptimedb/src/common/procedure/src/local/runner.rs`, 行 164-179

### 问题描述
```rust
pub(crate) async fn run(mut self) {
    let mut guard = ProcedureGuard::new(...);
    for key in self.meta.lock_key.keys_to_lock() {
        // 获取锁 - 可能阻塞很长时间
        let key_guard = match key {
            StringKey::Share(key) => self.manager_ctx.key_lock.read(key.clone()).await.into(),
            StringKey::Exclusive(key) => self.manager_ctx.key_lock.write(key.clone()).await.into(),
        };
        guard.key_guards.push(key_guard);
    }
    // 这里没有检查 running()!
    self.execute_procedure_in_loop().await;
}
```

spawned 任务可能长时间等待锁。在此期间 `stop()` 被调用。任务仍获取锁并进入执行循环，不必要地持有锁。

---

## Bug 6 (MEDIUM): `RepeatedTask` 无法中断执行中的任务函数

**文件**: `/workspace/greptimedb/src/common/runtime/src/repeated_task.rs`, 行 120-135

### 问题描述
```rust
let handle = runtime.spawn(async move {
    loop {
        let sleep_time = initial_delay.take().unwrap_or(interval);
        if sleep_time > Duration::ZERO {
            tokio::select! {
                _ = tokio::time::sleep(sleep_time) => {}
                _ = child.cancelled() => { return; }  // 只在休眠时检查
            }
        }
        // 这里没有取消检查!
        if let Err(e) = task_fn.call().await {  // <-- 无法被中断
            error!(e; "Failed to run repeated task: {}", task_fn.name());
        }
    }
});
```

`child.cancelled()` 检查只在休眠阶段发生。如果 `task_fn.call().await` 正在执行时 `stop()` 被调用，任务函数运行到完成后才注意到取消。

### 影响
`RemoveOutdatedMetaFunction` (procedure 系统中 RepeatedTask 的唯一用户) 可以在 `stop()` 后多运行一次。

---

## Bug 7 (LOW): `define_ticker!` 宏使用 fire-and-forget `abort()`

**文件**: `/workspace/greptimedb/src/meta-srv/src/utils.rs`, 行 90-96

### 问题描述
```rust
pub fn stop(&self) {
    let mut handle = self.tick_handle.lock().unwrap();
    if let Some(handle) = handle.take() {
        handle.abort();  // Fire and forget
    }
}
```

`abort()` 调用不等待 `JoinHandle`。`start()` 和 `stop()` 之间存在 TOCTOU 竞态。

---

## Bug 8 (LOW): `stop()` 不等待正在运行的过程完成

**文件**: `/workspace/greptimedb/src/common/procedure/src/local.rs`, 行 832-844

### 问题描述
`stop()` 设置 `running` flag 为 `false` 后立即返回。它不 join 或等待任何 spawned runner 任务。如果 `stop()` 后跟 `start()` (如 `test_procedure_manager_restart` 测试中)，新过程可在旧 runner 仍在完成时提交。

---

## 总结

| # | 严重度 | 文件 | 行 | 描述 |
|---|--------|------|-----|------|
| 1 | CRITICAL | local/runner.rs | 554-623 | 通知丢失: 子过程太快完成导致父过程永久挂起 |
| 2 | HIGH | local.rs | 617-691 | TOCTOU: submit_root() 检查 running() 后 spawn |
| 3 | HIGH | local/runner.rs | 476-540 | submit_subprocedure() 不检查 running() |
| 4 | MEDIUM | local/runner.rs | 543-552 | wait_on_err() 盲休眠无取消 |
| 5 | MEDIUM | local/runner.rs | 164-179 | 获取锁后无 running() 检查 |
| 6 | MEDIUM | repeated_task.rs | 120-135 | 取消只在休眠时检查 |
| 7 | LOW | utils.rs | 90-96 | define_ticker! fire-and-forget abort |
| 8 | LOW | local.rs | 832-844 | stop() 不等待 spawned tasks |

**Bug #1 是最可能的根因**: 父过程挂起等待已完成的子过程，导致整个过程树卡住。
