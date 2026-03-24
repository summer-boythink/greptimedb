# api/ 模块

## 概述
Protobuf API 定义层，包装和重新导出 `greptime_proto` crate 的类型。

## 关键类型
- `ColumnDataTypeWrapper`: protobuf 类型 <-> 内部 `ConcreteDataType` 双向转换
- `RegionResponse`: 从 protobuf `RegionRegionResponseV1` 派生
- 重新导出 `greptime_proto::v1::*` 和 `greptime_proto::prometheus::remote::*`

## 支持的服务
- GreptimeDatabase: 主数据库 gRPC 服务
- Region: Region 级操作
- Flow: 连续聚合操作
- HealthCheck: 健康检查
- PrometheusGateway: PromQL 查询
- Arrow Flight: 流式查询结果
- Frontend: list_process/kill_process

## 辅助函数
- `pb_value_to_value_ref()` / `to_grpc_value()`: 值转换
- `request_type()`: 请求类型内省 (用于指标)
- `encode_json_value()` / `decode_json_value()`: JSON 编解码
