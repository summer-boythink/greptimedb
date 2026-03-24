# Flow 引擎 — 内部架构详解与系统定位

## Flow 在整个 GreptimeDB 中的位置

想象 GreptimeDB 是一座**大型物流仓库**:

```
客户 (SQL/PromQL/OTLP 查询)
    │
    ▼
┌─────────────────────────────────────────────────┐
│                  Frontend                        │
│          (仓库前台接待员)                          │
│  接收客户订单, 分拣路由, 返回结果                    │
└─────────────┬───────────────┬───────────────────┘
              │               │
              ▼               ▼
    ┌─────────────┐   ┌─────────────┐
    │  Datanode   │   │  Flownode   │
    │  (货架管理员) │   │ (流水线工人)  │
    │  存取数据    │   │  实时计算    │
    └─────────────┘   └─────────────┘
              │               │
              ▼               ▼
    ┌─────────────────────────────────────┐
    │            Metasrv                   │
    │         (仓库调度中心)                 │
    │  管理货架布局, 协调工人, 故障处理        │
    └─────────────────────────────────────┘
```

**Flow 引擎就是那条"流水线"**——当数据流入仓库时, Flow 工人实时地对数据进行加工、聚合、转换, 把半成品变成成品, 直接放到指定货架上。

---

## Flow 内部架构: 一个工厂的解剖

如果把整个 Flow 引擎比作一座**智能工厂**, 那么它内部有这些车间:

```
┌──────────────────────────────────────────────────────────────┐
│                    FlowDualEngine                             │
│                    (工厂厂长)                                   │
│                                                              │
│  负责: 接收订单(DDL), 决定派给哪个车间, 管理工人花名册            │
│                                                              │
│  ┌──────────────────────┐    ┌──────────────────────┐        │
│  │   StreamingEngine    │    │   BatchingEngine     │        │
│  │   (流水线车间)         │    │   (批处理车间)        │        │
│  │                      │    │                      │        │
│  │  每个数据点来了就处理   │    │  攒一批数据再统一处理   │        │
│  │  类似实时装配线         │    │  类似批量烤箱          │        │
│  └──────────────────────┘    └──────────────────────┘        │
└──────────────────────────────────────────────────────────────┘
```

### FlowDualEngine (工厂厂长)

**职责**: 作为 Flownode 和底层引擎之间的中介。

```
Flownode 的指令                    FlowDualEngine 的行为
─────────────────                  ─────────────────────
CREATE FLOW foo    ──────────►     查看 flow_options 决定类型
                                   ┌─ type=batching ──► BatchingEngine.create_flow()
                                   └─ type=streaming ► StreamingEngine.create_flow()

INSERT 数据到源表   ──────────►     查表找到对应的 Flow
                                   ┌─ 是 batching ───► BatchingEngine.handle_mark_window_dirty()
                                   └─ 是 streaming ──► StreamingEngine.handle_flow_inserts()

REMOVE FLOW foo    ──────────►     查表找到引擎类型
                                   └─ 调用对应引擎的 remove_flow()
```

厂长维护了一个**花名册** (`src_table2flow`):
- 哪些源表 → 哪些 Flow
- 每个 Flow 是哪种类型 (batching/streaming)

---

### StreamingEngine (流水线车间)

**比喻**: 像一条**汽车装配线**, 每辆车(数据点)经过时, 工人立刻拧螺丝、装轮胎。

```
                    InterThreadCallClient/Server
                    (工人之间的对讲机)
                           │
    ┌──────────────────────┼──────────────────────┐
    │                      ▼                      │
    │  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐       │
    │  │Wkr 1│  │Wkr 2│  │Wkr 3│  │Wkr 4│       │
    │  │     │  │     │  │     │  │     │       │
    │  │dfir │  │dfir │  │dfir │  │dfir │       │
    │  │图计算│  │图计算│  │图计算│  │图计算│       │
    │  └─────┘  └─────┘  └─────┘  └─────┘       │
    │     ▲         ▲        ▲         ▲          │
    │     │         │        │         │          │
    │  ┌──┴─────────┴────────┴─────────┴──┐       │
    │  │     StreamingEngine.run()         │       │
    │  │   (主控台: 定期触发所有 Worker)      │       │
    │  └──────────────────────────────────┘       │
    └─────────────────────────────────────────────┘
```

每个 Worker 运行在**独立的 OS 线程**上, 维护一个 `dfir_rs` 数据流图。数据通过 **InterThreadCallClient/Server** (对讲机) 在 Worker 间传递。

**Shutdown 机制**: 每个 Worker 有一个 `AtomicBool` 标志。`WorkerHandle` 在 Drop 时调用 `shutdown()`。

---

### BatchingEngine (批处理车间) — 本次修复的重点

**比喻**: 像一个**面包房**, 师傅(Flow 任务)定时检查烤箱里的面团(时间窗口), 如果有新面团就烤一批面包。

```
BatchingEngine (面包房老板)
    │
    │  维护两个 map:
    │  ├── tasks: BTreeMap<FlowId, BatchingTask>      (面包师花名册)
    │  └── shutdown_txs: BTreeMap<FlowId, Sender>     (每位面包师的紧急电话)
    │
    ├── BatchingTask A (面包师小张)
    │   ├── config: Arc<TaskConfig>    (配方: SQL, 时间窗口, 源表, sink 表)
    │   ├── state: Arc<RwLock<TaskState>>  (当前状态: 脏时间窗口, 上次执行时间)
    │   └── start_executing_loop()     (小张的工作循环)
    │       │
    │       │  loop {
    │       │      检查电话有没有紧急通知? ──► 有 → 收拾下班
    │       │      查看哪些时间窗口有新面团(脏窗口)
    │       │      生成 SQL (配方) → 把面团分组
    │       │      执行 SQL (放入烤箱) → 产出结果
    │       │      等待下次检查(休眠 eval_interval)
    │       │  }
    │
    ├── BatchingTask B (面包师小李)
    │   └── (同样的工作循环, 但处理不同的 Flow)
    │
    └── BatchingTask C ...
```

#### BatchingTask 的工作循环 (面包师的一天)

```
开始工作
    │
    ▼
┌──────────────────────────────────────────┐
│  1. 检查紧急电话(shutdown_rx.try_recv())   │
│     └─ 收到通知? → 收拾下班(break)          │
│     └─ 没收到? → 继续工作                    │
├──────────────────────────────────────────┤
│  2. 查看脏时间窗口                          │
│     哪些时间段有新数据到达?                   │
│     把这些时间段分组(合并相邻的)               │
├──────────────────────────────────────────┤
│  3. 生成 SQL 查询计划                       │
│     SELECT ... WHERE ts IN (脏窗口们)      │
│     INSERT INTO sink_table ...             │
├──────────────────────────────────────────┤
│  4. 执行查询 (通过 FrontendClient)          │
│     把面团放进烤箱, 等待出炉                  │
│     这一步可能需要很长时间 (可达 10 分钟)      │
├──────────────────────────────────────────┤
│  5. 休眠, 等待下次检查                      │
│     eval_interval 秒后再来                  │
└──────────────────────────────────────────┘
    │
    └──► 回到步骤 1
```

#### 脏时间窗口 (DirtyTimeWindows) — 面团追踪器

**比喻**: 像面包房墙上的**白板**, 记录了哪些时间段有新面团需要处理。

```
DirtyTimeWindows (白板)
    │
    │  windows: BTreeMap<Timestamp, Option<Timestamp>>
    │  (开始时间 → 结束时间)
    │
    │  ┌─────────────────────────────────────────────┐
    │  │ 00:00 ──── 05:00  (有面团!)                  │
    │  │ 05:00 ──── 10:00  (有面团!)                  │
    │  │ 10:00 ──── 15:00  (有面团!)                  │
    │  │ 20:00 ──── 25:00  (有面团!)                  │
    │  └─────────────────────────────────────────────┘
    │
    │  gen_filter_exprs() → 把白板转换成 SQL WHERE 条件
    │  merge_dirty_time_windows() → 合并相邻时间段
    │  clean() → 清空白板 (面团烤完了)
```

当新数据插入源表时:
1. Flownode 收到通知 (`handle_mark_window_dirty`)
2. 根据时间戳计算属于哪个时间窗口
3. 在白板上标记这个窗口"脏了"

当面包师执行查询时:
1. 读取白板上的脏窗口
2. 生成 `WHERE ts >= X AND ts < Y` 过滤条件
3. 执行 INSERT INTO sink_table SELECT ...
4. 清理白板

---

## 组件之间的关系: 一张数据流图

```
用户 INSERT 数据
    │
    ▼
Frontend (接待员)
    │
    ├──── 普通 INSERT ──────► Datanode (写入存储)
    │
    └──── Mirror INSERT ────► Flownode
                                │
                                ▼
                          FlowDualEngine (厂长)
                                │
                                ├──► 查花名册: 这个表有 Flow 关注吗?
                                │
                                ├──► StreamingEngine:
                                │      直接把数据喂给 dfir 图
                                │      实时计算, 立刻写入 sink 表
                                │
                                └──► BatchingEngine:
                                       根据时间戳标记脏窗口
                                       ┌──────────────────────┐
                                       │ handle_mark_dirty    │
                                       │   .time_window()     │
                                       │                      │
                                       │ → DirtyTimeWindows   │
                                       │   .add_window(ts)    │
                                       └──────────┬───────────┘
                                                  │
                                       (面包师下次醒来时看到白板)
                                                  │
                                                  ▼
                                       ┌──────────────────────┐
                                       │ start_executing_     │
                                       │   loop()             │
                                       │                      │
                                       │ 1. 读白板            │
                                       │ 2. 生成 SQL          │
                                       │ 3. 执行 → 写入 sink  │
                                       │ 4. 清空白板          │
                                       └──────────────────────┘
```

---

## Bug 的本质: 面包师没有收到下班通知

### Bug 1: 替换面包师时, 旧面包师没被叫停

**场景**: 老板(用户)说"把小张换成小李"。

**当前行为**: 小李直接上岗, 但没人通知小张下班。结果小张和小李同时在烤面包, 浪费资源还可能烤出重复的面包。

**修复**: 先打电话给小张说"下班了" → 等小张收拾好 → 再让小李上岗。

### Bug 2: 打了电话但没确认人走了

**场景**: 打电话通知小张下班, 但小张正在做一批大面包, 要 10 分钟才能停。

**当前行为**: 老板以为小张已经走了, 但小张还在烤箱前忙。

**修复**: 打完电话后, 再直接叫一声(abort handle), 确保小张立刻停下手头工作。

### Bug 3: 面包师烤面包期间不接电话

**场景**: 小张把面团放进烤箱后, 就去休息室了, 10 分钟不看电话。

**当前行为**: 紧急电话打了, 但小张要等面包烤完才看到。

**修复**: 小张在烤面包期间每 100ms 看一眼电话 (tokio::select!), 有通知立刻停。

---

## 关键数据结构速查

```
BatchingEngine
├── tasks: BTreeMap<FlowId, BatchingTask>     ← 所有活跃 Flow
├── shutdown_txs: BTreeMap<FlowId, Sender>    ← 每个 Flow 的关闭信号
├── frontend_client: Arc<FrontendClient>      ← 执行 SQL 的客户端
├── query_engine: QueryEngineRef              ← 生成查询计划
└── batch_opts: Arc<BatchingModeOptions>      ← 配置 (超时, 最小刷新间隔等)

BatchingTask
├── config: Arc<TaskConfig>                   ← 不可变配置
│   ├── flow_id, query, sink_table_name
│   ├── source_table_names, time_window_expr
│   ├── expire_after, flow_eval_interval
│   └── batch_opts, query_type
└── state: Arc<RwLock<TaskState>>             ← 可变状态
    ├── query_ctx                              ← 查询上下文
    ├── last_update_time                       ← 上次更新时间
    ├── last_exec_time_millis                  ← 上次成功执行时间
    ├── dirty_time_windows: DirtyTimeWindows   ← 脏时间窗口白板
    ├── shutdown_rx: Receiver<()>              ← 关闭信号接收器
    └── task_handle: Option<JoinHandle<()>>    ← 后台任务句柄
```
