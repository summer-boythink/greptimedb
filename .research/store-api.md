# store-api/ 模块

## 概述
核心存储 API 抽象，定义 Region 管理、请求处理、元数据和扫描的 trait 和类型。

## 关键类型
- `RegionId`: 唯一 Region 标识符 (table_id + region_number)
- `RegionMetadata`: 完整 Region schema (region_id, schema, column_metadatas, primary_key, schema_version)
- `ColumnMetadata`: 列定义 (column_schema, semantic_type, column_id)
- `RegionRequest`: 所有 Region 操作 (Put/Delete/Create/Drop/Open/Close/Alter/Flush/Compact/BuildIndex/Truncate/Catchup/BulkInserts)
- `AlterKind`: Schema 变更类型
- `RegionEngine` trait: 核心引擎接口 (handle_request, handle_query, get_metadata, sync_region)
- `RegionScanner` trait: 并发扫描接口
- `ScannerProperties`: 扫描器配置 (partitions, append_mode, target_partitions)

## 架构模式
- **Trait-based 引擎抽象**: 允许多种存储实现 (Mito/Metric/File)
- **请求/响应模式**: 所有操作通过 RegionRequest -> RegionResponse
- **角色复制**: Leader/Follower/DowngradingLeader
- **基于分区的扫描**: RegionScanner 将扫描分成分区并行执行
- **Manifest-based 状态**: Region 状态持久化在 manifest 中
