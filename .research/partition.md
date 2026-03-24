# partition/ 模块

## 概述
数据分区规则和管理，决定行如何在 Region 间分布。

## 关键类型
- `PartitionRule` trait: `partition_columns()`, `find_region()`, `split_record_batch()`
- `MultiDimPartitionRule`: 主要实现，支持任意 AND/OR 表达式
- `PartitionRuleManager`: 管理表路由和分区规则
- `RowSplitter`: 按分区规则拆分行
- `RepartitionSubtask`: 重分区的独立连通分量
- `PartitionExpr`: 分区表达式 AST
- `PartitionChecker`: 验证分区表达式正确性
- `PartitionSimplifier`: 简化分区表达式
- `PartitionInfoCache`: 缓存的分区信息

## 架构模式
- **Strategy 模式**: `PartitionRule` trait 允许不同分区策略
- **Cache 模式**: 两级缓存 (TableRouteCache + PartitionInfoCache)
- **图算法**: `subtask.rs` 使用 BFS 找重分区连通分量
- **PhysicalExpr 缓存**: `MultiDimPartitionRule` 缓存编译的 DataFusion PhysicalExpr
