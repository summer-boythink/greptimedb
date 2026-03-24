# common/ 模块深度研究报告

## 概述

`common/` 目录包含 GreptimeDB 的共享基础设施库，共有 36 个子模块，涵盖运行时管理、过程执行、元数据管理、错误处理等核心功能。

## 子模块列表

| 子模块 | 包名 | 用途 |
|--------|------|------|
| base | common-base | 基础类型: 可读大小、位向量、异步辅助、正则 |
| catalog | common-catalog | 目录命名常量 (schema/table names) |
| config | common-config | 配置管理 (热加载 via notify, TOML, env) |
| datasource | common-datasource | 数据源抽象 (Parquet/ORC/CSV + 压缩) |
| decimal | common-decimal | 十进制数类型 (bigdecimal, rust_decimal) |
| error | common-error | 核心错误框架 (status codes, HTTP/tonic 转换) |
| event-recorder | common-event-recorder | 事件记录系统 (审计日志) |
| frontend | common-frontend | 前端节点抽象 (gRPC client, meta client) |
| function | common-function | SQL 函数注册表 (200+ 内置函数) |
| greptimedb-telemetry | common-greptimedb-telemetry | 遥测上报 |
| grpc | common-grpc | gRPC 基础设施 (flight service, 中间件, TLS) |
| grpc-expr | common-grpc-expr | gRPC 表达式转换 |
| macro | common-macro | 过程宏 (stack_trace_debug, protobuf 转换) |
| mem-prof | common-mem-prof | 内存分析 (jemalloc, heap dumps) |
| memory-manager | common-memory-manager | 内存管理抽象 |
| meta | common-meta | 元数据管理层 ★ |
| options | common-options | 通用选项类型 |
| plugins | common-plugins | 插件系统 (框架) |
| pprof | common-prof | CPU 分析 (pprof, flame graphs) |
| procedure | common-procedure | 任务/过程执行框架 ★ |
| procedure-test | common-procedure-test | 过程测试工具 |
| query | common-query | 查询层抽象 (logical/physical plans) |
| recordbatch | common-recordbatch | RecordBatch 工具 |
| runtime | common-runtime | 异步运行时管理 ★ |
| session | common-session | 会话管理 |
| sql | common-sql | SQL 解析工具 |
| stat | common-stat | 系统统计 (CPU, memory via sysinfo) |
| substrait | substrait | Substrait 计划序列化 |
| telemetry | common-telemetry | 日志/追踪/OTel/Prometheus 指标 |
| test-util | common-test-util | 测试辅助 |
| time | common-time | 时间类型 (Timestamp, Date, Time) |
| version | common-version | 构建版本信息 |
| wal | common-wal | Write-Ahead Log 抽象 (Kafka backend) |
| workload | common-workload | 工作负载分类 (Hybrid) |

## 重点模块详细分析

### 1. common/runtime - 运行时管理

**文件结构**:
```
src/common/runtime/src/
├── lib.rs                  -- 模块声明和公共导出
├── runtime.rs              -- 核心 trait, Builder, Priority, Dropper
├── runtime_default.rs      -- DefaultRuntime 实现
├── runtime_throttleable.rs -- ThrottleableRuntime (token-bucket 限速)
├── global.rs               -- 全局运行时单例
├── repeated_task.rs        -- RepeatedTask 周期任务
├── metrics.rs              -- Prometheus 线程池监控
├── error.rs                -- 错误类型
└── bin.rs                  -- 基准测试二进制
```

**核心类型**:

#### RuntimeTrait
```rust
pub trait RuntimeTrait {
    fn spawn<F>(&self, future: F) -> JoinHandle<F::Output>;
    fn spawn_blocking<F, R>(&self, func: F) -> JoinHandle<R>;
    fn block_on<F: Future>(&self, future: F) -> F::Output;
    fn name(&self) -> &str;
}
```

#### Priority (CPU 限速级别)
| 级别 | 值 | Token 数/10ms |
|------|-----|---------------|
| VeryLow | 0 | 2000 |
| Low | 1 | 4000 |
| Middle | 2 | 6000 |
| High | 3 | 8000 |
| VeryHigh | 4 | 无限制 |

#### Global Runtimes (单例)
- `global_runtime()`: 通用, num_cpus workers
- `compact_runtime()`: 压缩, max(num_cpus/2, 1) workers
- `hb_runtime()`: 心跳, 2 workers

#### RepeatedTask
```rust
pub struct RepeatedTask<E> {
    name: String,
    cancel_token: CancellationToken,
    inner: Mutex<TaskInner<E>>,
    started: AtomicBool,
    interval: Duration,
    initial_delay: Option<Duration>,
}
```
- 使用 `tokio_util::sync::CancellationToken` 优雅取消
- `start(runtime)`: 在指定 runtime 上启动无限循环
- `stop()`: 取消并等待完成
- 支持不同于 interval 的 initial_delay

#### Dropper (运行时关闭)
使用 oneshot channel 实现: 当 Dropper 被 drop 时，发送信号解除运行时阻塞。

### 2. common/procedure - 任务执行框架

**文件结构**:
```
src/common/procedure/src/
├── lib.rs
├── procedure.rs              -- 核心类型 (Procedure trait, Status, ProcedureState)
├── local.rs                  -- LocalManager, ManagerContext, ProcedureMeta
├── local/runner.rs           -- Runner 执行引擎
├── error.rs                  -- 错误类型
├── event.rs                  -- ProcedureEvent
├── watcher.rs                -- Watcher 类型
├── options.rs                -- ProcedureConfig
├── rwlock.rs                 -- KeyRwLock<K>
├── store.rs                  -- ProcedureStore
├── store/state_store.rs      -- StateStore trait
├── store/poison_store.rs     -- PoisonStore trait
└── store/util.rs             -- Stream 工具
```

#### Procedure trait
```rust
#[async_trait]
pub trait Procedure: Send {
    fn type_name(&self) -> &str;
    async fn execute(&mut self, ctx: &Context) -> Result<Status>;  // 幂等
    async fn rollback(&mut self, _: &Context) -> Result<()>;       // 幂等
    fn rollback_supported(&self) -> bool;
    fn dump(&self) -> Result<String>;                              // 序列化状态
    fn recover(&mut self) -> Result<()>;                           // 恢复后钩子
    fn lock_key(&self) -> LockKey;                                 // 所需锁
    fn poison_keys(&self) -> PoisonKeys;                           // 毒键
    fn user_metadata(&self) -> Option<UserMetadata>;               // 事件元数据
}
```

#### Status (执行状态)
```rust
pub enum Status {
    Executing { persist: bool, clean_poisons: bool },
    Suspended { subprocedures: Vec<ProcedureWithId>, persist: bool },
    Poisoned { keys: PoisonKeys, error: Error },
    Done { output: Option<Output> },
}
```

#### ProcedureState (外部状态)
```rust
pub enum ProcedureState {
    Running,
    Done { output: Option<Output> },
    Retrying { error: Arc<Error> },
    PrepareRollback { error: Arc<Error> },
    RollingBack { error: Arc<Error> },
    Failed { error: Arc<Error> },
    Poisoned { keys: PoisonKeys, error: Arc<Error> },
}
```

#### 执行流程
1. **提交**: `LocalManager::submit()` 创建 `ProcedureMeta`, 在 global runtime 上 spawn `Runner`
2. **锁获取**: `Runner::run()` 在执行前获取所有锁
3. **执行循环**: `execute_procedure_in_loop()` -> `execute_once_with_retry()` -> `execute_once()`
4. **状态机**: Running -> Executing/Suspended/Done/Poisoned; 错误 -> Retrying -> Failed
5. **持久化**: 每步通过 `ProcedureStore` 持久化状态
6. **清理**: 完成后删除过程文件、释放锁、通知父过程

#### KeyRwLock
```rust
pub struct KeyRwLock<K> {
    inner: Mutex<HashMap<K, Arc<RwLock<()>>>>,
}
```
- `read(key)`: 共享访问
- `write(key)`: 独占访问
- `clean_keys(keys)`: 移除未使用的锁条目

#### 持久化存储
- Step 文件: `{proc_path}/{id}/{step:010}.step` - 中间状态
- Commit 文件: `{proc_path}/{id}/{step:010}.commit` - 完成标记
- Rollback 文件: `{proc_path}/{id}/{step:010}.rollback` - 回滚状态

### 3. common/meta - 元数据管理层

提供 DDL 操作、集群协调、Region 管理、KV 存储后端(etcd/MySQL/PostgreSQL)、心跳、领导选举和协调功能。

关键组件:
- `LeadershipChangeNotifier`: 广播 leader 开始/停止事件
- `RegionFailureDetector`: Phi Accrual 故障检测器
- `TableMetadataManager`: 表元数据 CRUD
- `PartitionManager`: 分区管理
- `KvBackend`: KV 存储抽象

## 架构模式总结

| 模式 | 使用的模块 |
|------|-----------|
| Trait-based 抽象 | 所有模块 |
| Builder 模式 | runtime, procedure, meta |
| 单例模式 | runtime (global runtimes) |
| 状态机 | procedure (ProcedureState) |
| RAII/Dropper | runtime (Dropper), procedure (ProcedureGuard) |
| 通道通信 | runtime (oneshot), procedure (watch) |
| 限速/节流 | runtime (ThrottleableRuntime) |
| 幂等设计 | procedure (execute/rollback) |
| 毒键检测 | procedure (PoisonStore) |
| 指数退避 | procedure (ExponentialBuilder) |
