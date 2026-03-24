# log-query/ 模块

## 概述
日志查询 DSL，专为搜索和分析日志数据设计。

## 核心类型
- `LogQuery`: 顶层查询 (table, time_filter, limit, columns, filters, context, exprs)
- `TimeFilter`: 灵活时间范围 (start, end, span)
- `Filters`: 递归过滤器树 (Single/And/Or/Not)
- `ContentFilter`: 丰富过滤类型 (Exact/Prefix/Postfix/Contains/Regex/Exist/Between/GreatThan/LessThan/In)
- `LogExpr`: 后处理表达式 (NamedIdent/Literal/ScalarFunc/AggrFunc/Decompose/BinaryOp)
- `Context`: 相邻行上下文 (Lines/Seconds)

## 架构模式
- **嵌套过滤树**: 递归 And/Or/Not 支持任意复杂布尔表达式
- **表达式后处理**: `LogExpr` 支持标量函数、聚合、分组、JSON/CSV 提取
- **灵活时间解析**: `TimeFilter::canonicalize()` 处理多种格式
