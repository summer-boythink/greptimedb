# promql/ 模块

## 概述
PromQL 执行支持，实现自定义 DataFusion 扩展计划和函数。

## 扩展计划 (extension_plan/)
| 类型 | 描述 |
|------|------|
| SeriesNormalize/Exec | 时间序列数据对齐标准化 |
| InstantManipulate/Exec | 即时向量查询 |
| RangeManipulate/Exec | 范围向量查询 |
| SeriesDivide/Exec | 按分组划分数据 |
| EmptyMetric/Exec | 无数据时产生空指标 |
| ScalarCalculate/Exec | scalar() 函数 |
| HistogramFold/Exec | histogram_quantile 计算 |
| Absent/Exec | absent() 函数 |
| UnionDistinctOn/Exec | 去重联合 |

## PromQL 函数
Rate, Delta, Increase, Deriv, Changes, Resets, IDelta, PredictLinear, AvgOverTime, MinOverTime, MaxOverTime, SumOverTime, CountOverTime, LastOverTime, PresentOverTime, StddevOverTime, StdvarOverTime, QuantileOverTime, DoubleExponentialSmoothing, Round

## 核心数据结构
- `RangeArray`: 基于 Arrow DictionaryArray 的复合逻辑数组，打包 offset+length 到 i64 keys
- `Millisecond`: 毫秒时间戳类型别名

## 架构模式
- **Extension Plan Pattern**: 每个 PromQL 操作是自定义 DataFusion 逻辑节点
- **Stream-Based 执行**: 每个扩展计划有 `*Exec` + `*Stream`
- **RangeArray 抽象**: 高效表示滑动窗口
- **补偿求和**: Kahan 求和算法 (数值稳定)
