# table/ 模块

## 概述
表级抽象，位于存储引擎层之上，处理表元数据、Schema 管理和 DataFusion 集成。

## 关键类型
- `Table`: 主表 handle (table_info, filter_pushdown, data_source, column_defaults, partition_rules)
- `TableInfo`: 完整表信息 (ident, name, catalog_name, schema_name, meta, table_type)
- `TableMeta`: 表元数据 (schema, primary_key_indices, value_indices, engine, next_column_id, options)
- `TableType`: Base, View, Temporary
- `FilterPushDownType`: Unsupported, Inexact, Exact
- `AlterKind`: AddColumns, DropColumns, ModifyColumnTypes, RenameTable, SetTableOptions 等

## 架构模式
- **Builder 模式**: TableMetaBuilder 构造不可变元数据
- **Arc-based 共享**: TableRef = Arc<Table>, TableInfoRef = Arc<TableInfo>
- **Schema 版本化**: 每次 alter 操作递增 schema 版本
- **DataFusion 集成**: Table 实现 DataFusion table provider
- **列 ID 管理**: 列有名字和不可变 ID 用于可靠跟踪
