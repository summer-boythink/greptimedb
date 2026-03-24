# GreptimeDB - 深度研究报告

## 项目概览

**GreptimeDB** 是一个开源的可观测性数据库，统一了指标(Metrics)、日志(Logs)和追踪(Traces)三种数据模型。作为 Prometheus、Loki 和 Elasticsearch 的替代品，它使用对象存储(S3/GCS/Azure Blob)作为主存储，支持 SQL + PromQL 查询。

- 版本: `1.0.0-rc.2` (GA 计划 2026年3月)
- 语言: Rust (Edition 2024)
- 许可证: Apache-2.0

## 架构模式

### 运行模式
1. **Standalone 模式**: 单二进制部署，用于开发和小规模部署
2. **Distributed 模式**: 分离组件用于生产规模
   - **Frontend**: 查询处理和协议处理
   - **Datanode**: 数据存储和检索
   - **Metasrv**: 元数据管理和协调

### 核心设计原则
- **Region-based 存储**: 数据按 Region 分片，每个 Region 是独立的存储单元
- **Engine 抽象**: `RegionEngine` trait 允许多种存储引擎(Mito/Metric/File)共存
- **Worker-per-Region 分片**: Mito 引擎使用 hash-based routing 将 Region 绑定到 Worker
- **Procedure 系统**: 分布式 DDL 操作通过持久化步骤执行，支持重试和回滚
- **Leadership 感知**: Metasrv 的后台任务在 leader 选举时启动/停止

## src/ 目录结构

```
src/
├── api/           - Protobuf API 定义层 (greptime_proto 包装)
├── auth/          - 认证授权框架 (Static/WatchFile UserProvider)
├── cache/         - 多层缓存注册表 (moka cache)
├── catalog/       - 三级目录层次 (Catalog -> Schema -> Table)
├── cli/           - 命令行工具 (数据导入导出、元数据管理)
├── client/        - gRPC 客户端实现 (Database/Flight/Region/Flow)
├── cmd/           - 顶层入口点和 CLI 编排 (App trait)
├── common/        - 共享基础设施库 (36个子模块)
│   ├── base/          - 基础类型和工具
│   ├── catalog/       - 目录命名常量
│   ├── config/        - 配置管理 (热加载)
│   ├── datasource/    - 数据源抽象 (Parquet/ORC/CSV)
│   ├── decimal/       - 十进制数类型
│   ├── error/         - 错误框架
│   ├── event-recorder/ - 事件记录系统
│   ├── frontend/      - 前端节点抽象
│   ├── function/      - SQL 函数注册表 (200+ 内置函数)
│   ├── greptimedb-telemetry/ - 遥测上报
│   ├── grpc/          - gRPC 基础设施
│   ├── grpc-expr/     - gRPC 表达式转换
│   ├── macro/         - 过程宏
│   ├── mem-prof/      - 内存分析 (jemalloc)
│   ├── memory-manager/ - 内存管理抽象
│   ├── meta/          - 元数据管理层 (DDL/协调/心跳/领导选举)
│   ├── options/       - 通用选项类型
│   ├── plugins/       - 插件系统
│   ├── pprof/         - CPU 分析
│   ├── procedure/     - 任务/过程执行框架 ★
│   ├── procedure-test/ - 过程测试工具
│   ├── query/         - 查询层抽象
│   ├── recordbatch/   - RecordBatch 工具
│   ├── runtime/       - 异步运行时管理 ★
│   ├── session/       - 会话管理
│   ├── sql/           - SQL 解析工具
│   ├── stat/          - 系统统计收集
│   ├── substrait/     - Substrait 计划序列化
│   ├── telemetry/     - 日志/追踪/指标基础设施
│   ├── test-util/     - 测试辅助
│   ├── time/          - 时间类型
│   ├── version/       - 构建版本信息
│   ├── wal/           - Write-Ahead Log 抽象
│   └── workload/      - 工作负载分类
├── datanode/      - 存储层 Worker (RegionServer + Heartbeat)
├── datatypes/     - 类型系统 (Arrow 互操作)
├── file-engine/   - 轻量级只读 Region 引擎 (外部文件)
├── flow/          - 流式/批处理连续聚合引擎 ★
├── frontend/      - 用户入口 (多协议代理/路由)
├── index/         - 索引策略 (倒排/布隆/全文/向量)
├── log-query/     - 日志查询 DSL
├── log-store/     - WAL 实现 (Kafka/Raft Engine)
├── meta-client/   - Metasrv gRPC 客户端
├── meta-srv/      - 元数据服务 (协调/故障检测/GC) ★
├── metric-engine/ - 指标引擎 (Region 多路复用器)
├── mito2/         - 核心时序存储引擎 (LSM-tree) ★
├── mito-codec/    - Mito 编解码器 (主键/索引)
├── object-store/  - 对象存储抽象层 (OpenDAL)
├── operator/      - 高级请求编排层
├── partition/     - 数据分区规则和管理
├── pipeline/      - 日志 ETL 框架 (YAML pipeline)
├── plugins/       - 插件架构骨架
├── promql/        - PromQL 执行支持
├── puffin/        - Apache Puffin 文件格式
├── query/         - 中央查询执行引擎 (DataFusion)
├── servers/       - 多协议服务器层 (HTTP/gRPC/MySQL/PG)
├── session/       - 连接会话管理
├── sql/           - SQL 解析 (sqlparser 扩展)
├── standalone/    - 单节点部署配置
├── store-api/     - 存储 API 抽象 (RegionEngine trait)
└── table/         - 表级抽象 (Table trait + DataFusion 集成)
```

## 关键架构组件

### 1. 存储引擎层次

```
RegionServer (datanode/region_server.rs)
├── MitoEngine (mito2/) -- 时序数据 LSM-tree 引擎
│   ├── WorkerGroup -> RegionWorker[] (hash-based routing)
│   ├── Flush Scheduler (LocalScheduler)
│   ├── Compaction Scheduler (LocalScheduler)
│   ├── Index Build Scheduler (LocalScheduler)
│   └── Purge Scheduler
├── MetricEngine (metric-engine/) -- 指标多路复用器
│   └── wraps MitoEngine (Data Region + Metadata Region)
└── FileRegionEngine (file-engine/) -- 只读外部文件
```

### 2. 运行时和任务调度

```
common/runtime/
├── DefaultRuntime -- tokio::runtime::Handle 包装
├── ThrottleableRuntime -- 基于 Priority 的 Token-Bucket 限速
├── Global Runtimes:
│   ├── global_runtime() -- 通用, num_cpus workers
│   ├── compact_runtime() -- 压缩, max(num_cpus/2, 1) workers
│   └── hb_runtime() -- 心跳, 2 workers
└── RepeatedTask -- 周期性后台任务 (CancellationToken)
```

**Priority 级别**: VeryLow(2000) -> Low(4000) -> Middle(6000) -> High(8000) -> VeryHigh(无限制) tokens/10ms

### 3. Procedure 系统 (分布式 DDL)

```
common/procedure/
├── LocalManager -- 过程管理器
│   ├── ManagerContext -- 共享状态 (loaders, locks, procedures)
│   ├── KeyRwLock -- 按 key 读写锁
│   ├── DynamicKeyLock -- 运行时动态锁
│   └── PoisonStore -- 资源冲突检测
├── Runner -- 执行引擎
│   ├── execute_procedure_in_loop() -- 主执行循环
│   ├── execute_once_with_retry() -- 重试逻辑
│   └── on_suspended() -- 子过程等待
└── ProcedureStore -- 持久化层 (step/commit/rollback files)
```

**状态机**: Running -> Done / Retrying -> PrepareRollback -> RollingBack -> Failed / Poisoned

### 4. Flow 引擎 (连续聚合)

```
flow/
├── StreamingEngine -- 多线程 Worker 池 (dfir_rs 数据流图)
└── BatchingEngine -- Tokio 任务/Flow 模式
    ├── BatchingTask -- 每个 Flow 独立任务
    ├── TaskState -- mutable state + shutdown_rx
    └── start_executing_loop() -- 查询执行循环
```

### 5. Metasrv 后台任务

```
meta-srv/
├── define_ticker! 宏生成的 ticker:
│   ├── GcTicker -- 垃圾回收 (5min interval)
│   ├── WalPruneTicker -- WAL 剪枝
│   └── RegionFlushTicker -- Region 刷新
├── RegionSupervisorTicker -- 故障检测 (1s interval + 初始化)
└── LeadershipChangeNotifier -- leader 变更广播
    ├── ProcedureManagerListenerAdapter
    ├── RegionSupervisorTicker
    ├── WalPruneTicker
    ├── RegionFlushTicker
    └── GcTicker
```

## 发现的关键 Bug

详见:
- `.research/flow-task-bugs.md` - Flow 引擎任务调度 Bug
- `.research/procedure-scheduling-bugs.md` - Procedure 系统 Bug
- `.research/mito2-scheduling-bugs.md` - Mito2 存储引擎 Bug

### 概要

| 模块 | 严重度 | Bug 数量 | 核心问题 |
|------|--------|----------|----------|
| Flow | 高 | 4 | 替换 Flow 时旧任务未取消, JoinHandle 未中止 |
| Procedure | 高 | 8 | 子过程通知丢失导致父过程挂起, TOCTOU 竞态 |
| Mito2 | 高 | 8 | 关闭顺序错误, 进行中的后台任务无法取消 |
| Runtime | 中 | 2 | RepeatedTask 无法中断执行中的函数 |
| Meta-srv | 低 | 1 | define_ticker! 使用 fire-and-forget abort |

## 技术栈

| 组件 | 技术 |
|------|------|
| 语言 | Rust (Edition 2024) |
| 异步运行时 | Tokio (full features) |
| 查询引擎 | Apache DataFusion 52.1 |
| 内存模型 | Apache Arrow 57.3 |
| 存储格式 | Apache Parquet 57.3 |
| 对象存储 | Apache OpenDAL |
| WAL | Raft Engine / Kafka |
| 元数据 | etcd / MySQL / PostgreSQL (KV backend) |
| gRPC | Tonic 0.14 (TLS + compression) |
| HTTP | Axum 0.8 |
| 序列化 | Protobuf (prost) / JSON (serde) |
| 日志/追踪 | tracing + OpenTelemetry |
| 指标 | Prometheus |
