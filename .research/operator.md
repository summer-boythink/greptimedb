# operator/ 模块

## 概述
高级请求编排层，协调分布式操作 (insert/delete/flush/compaction/build index/SQL execution/flow management)。

## 关键类型
- `Inserter`: 处理所有插入工作流 (自动创建/修改表)
- `Deleter`: 处理列删除、行删除、表删除
- `Requester`: 处理管理操作 (flush/compaction/build index)
- `TableMutationOperator`: 统一表变更接口
- `StatementExecutor`: SQL 执行协调器 (DDL/DML/SHOW/DESCRIBE/COPY/KILL)
- `ProcedureServiceOperator`: 过程服务处理 (migrate_region/reconcile/gc)
- `FlowServiceOperator`: 流服务处理 (flush flownodes)
- `FlowMirrorTask`: 异步镜像插入请求到流节点

## 架构模式
- **Mediator 模式**: Inserter/Deleter/Requester 调度请求到 datanodes
- **Facade 模式**: TableMutationOperator 提供统一接口
- **Strategy 模式**: AutoCreateTableType 确定表创建策略
- **Pipeline 模式**: Insert 流: 预处理 -> 创建/修改表 -> 转换 -> 镜像到 flow -> 分发
- **Fan-out**: 按 Region/Peer 分组，并行执行

## 后台任务
- `FlowMirrorTask::detach()`: spawn 全局异步任务镜像插入
- `Inserter/Deleter::do_request()`: spawn 每 peer 异步任务并行执行
- `Requester::do_request()`: spawn 每 region 异步任务
- `FlowServiceOperator::flush_inner()`: FuturesUnordered 并行 flush
