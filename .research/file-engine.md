# file-engine/ 模块

## 概述
轻量级 Region 引擎，直接从外部文件源 (CSV/Parquet) 读取数据。

## 支持的操作
| 操作 | 支持? |
|------|-------|
| Create | ✓ |
| Open | ✓ |
| Close | ✓ |
| Drop | ✓ |
| Query/Scan | ✓ |
| Put/Write | ✗ |
| Delete | ✗ |
| Alter | ✗ |
| Flush | ✗ |
| Compact | ✗ |

## 关键类型
- `FileRegionEngine`: 实现 `RegionEngine` trait, engine name = "file"
- `EngineInner`: 包含 `regions: RwLock<HashMap<RegionId, FileRegionRef>>`
- `FileRegion`: Region 抽象 (region_dir, file_options, url, format, options, metadata)
- `FileOptions`: files list + column schemas
- `FileToScanRegionStream`: Stream 适配器

## 查询优化
- **投影下推**: 将 scan region 列索引转换为文件列索引
- **过滤下推**: 将仅引用文件列的过滤器下推到文件扫描
- **Schema 映射**: 当文件 schema 与 scan region schema 不同时填充默认值
