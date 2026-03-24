# cache/ 模块

## 概述
使用 `moka` 异步缓存库构建多层缓存注册表。

## 缓存策略
- 默认: 最大 65,536 条目, TTL 10分钟, TTI 5分钟

## 缓存注册表
### Datanode Cache Registry
- Schema cache (TTL/TTI)
- Table ID -> Schema name cache (永不过期)

### Fundamental Cache Registry
- Table info cache
- Table name cache
- Table route cache
- Table flownode set cache
- View info cache
- Schema cache
- Table ID -> Schema name cache

### Composite Cache Registry
- Table cache (table_info + table_name)
- Partition info cache (table_route)

## 命名常量
`TABLE_INFO_CACHE_NAME`, `VIEW_INFO_CACHE_NAME`, `TABLE_NAME_CACHE_NAME`, `TABLE_CACHE_NAME`, `SCHEMA_CACHE_NAME`, `TABLE_SCHEMA_NAME_CACHE_NAME`, `TABLE_FLOWNODE_SET_CACHE_NAME`, `TABLE_ROUTE_CACHE_NAME`, `PARTITION_INFO_CACHE_NAME`
