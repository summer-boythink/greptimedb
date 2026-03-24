# query/ 模块

## 概述
中央查询执行引擎，基于 Apache DataFusion，提供 SQL/PromQL 解析、优化和执行。

## 关键类型
- `QueryEngine` trait: planner(), execute(), describe(), function registration
- `QueryEngineFactory`: 创建查询引擎的工厂
- `QueryEngineState`: 全局引擎状态 (DataFusion context, 函数注册, 扩展规则, 内存池)
- `DatafusionQueryEngine`: 使用 DataFusion 的具体实现
- `DfLogicalPlanner`: 通过 SqlToRel/PromPlanner 规划 SQL/PromQL
- `QueryLanguageParser`: 解析 SQL 和 PromQL 的入口点

## 优化器规则
### Analyzer Rules
- `ApplyFunctionRewrites`: 函数调用重写
- `CountWildcardToTimeIndexRule`: COUNT(*) -> COUNT(timestamp)
- `StringNormalizationRule`: 字符串处理规范化
- `DistPlannerAnalyzer`: 分布式规划
- `FixStateUdafOrderingAnalyzer`: 修复聚合函数排序

### Optimizer Rules
- `ScanHintRule`: 扫描提示
- `TypeConversionRule`: 类型转换

### Physical Optimizer Rules
- `ParallelizeScan`: 并行化表扫描
- `PassDistribution`: 传递分布需求到 MergeScanExec
- `EnforceSorting`: 强制排序
- `WindowedSortPhysicalRule`: 窗口排序优化

## 执行层次
```
SQL/PromQL Text
  -> Parser (QueryLanguageParser)
  -> Logical Plan (DfLogicalPlanner)
  -> Analyzer Rules
  -> Optimizer Rules
  -> Physical Plan (DfQueryPlanner + Extension Planners)
  -> Physical Optimizer Rules
  -> Execution (execute_stream)
```

## 内存管理
- `MetricsMemoryPool`: 包装 GreedyMemoryPool + Prometheus 指标
- `QUERY_MEMORY_POOL_USAGE_BYTES`: 使用量
- `QUERY_MEMORY_POOL_REJECTED_TOTAL`: 拒绝次数

## 扩展计划器
- `PromExtensionPlanner`: PromQL 逻辑 -> 物理
- `RangeSelectPlanner`: 范围选择操作
- `DistExtensionPlanner`: 分布式查询执行
- `MergeSortExtensionPlanner`: 合并排序操作
