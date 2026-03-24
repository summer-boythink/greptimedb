# catalog/ 模块

## 概述
实现三级目录层次 (Catalog -> Schema -> Table)，管理表、Schema 和 Catalog 的生命周期。

## 关键类型
- `CatalogManager` trait: 核心接口 (catalog_names, schema_names, table_names, table, table_id 等)
- `KvBackendCatalogManager`: 主要实现，使用 KV 存储 + 分层缓存
- `MemoryCatalogManager`: 内存实现 (测试用)
- `DfTableSourceProvider`: 解析 DataFusion 表引用，支持视图
- `ProcessManager`: 查询进程生命周期管理

## 系统 Schema
- information_schema: 系统信息
- pg_catalog: PostgreSQL 兼容目录

## ProcessManager
- 使用 `AtomicU32` 跟踪运行查询
- `register_query()`: 注册运行查询，返回 `Ticket` (RAII guard)
- `list_all_processes()`: 跨所有前端节点列进程
- `kill_process()`: 使用 `CancellationHandle` 杀死查询
- `SlowQueryTimer`: RAII 计时器，记录慢查询

## 架构模式
- 三级层次: Catalog -> Schema -> Table
- KV-backed 元数据: 用户表元数据持久化在 KV 后端
- 系统目录覆盖: 系统表叠加在用户表之上
- 缓存: 多层缓存注册表 (TTL/TTI)
- Stream-based 表列示: `tables()` 返回 `BoxStream`
