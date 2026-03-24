# metric-engine/ 模块

## 概述
Region 引擎多路复用器，为 Prometheus 风格的监控/指标工作负载设计。不重新实现存储，而是包装 `MitoEngine`。

## 架构
```
Metric Region 1, 2, 3 ...
        |
   Metric Region Engine
    (contains Mito Engine)
        |
   Mito Engine Regions (Data Region + Metadata Region)
```

多个逻辑表共享一个物理宽表 (一对物理 Region: 一个数据 + 一个元数据)。

## 关键类型
- `MetricEngine`: 包装 `Arc<MetricEngineInner>`
- `MetricEngineInner`: 持有 MitoEngine, MetadataRegion, DataRegion, RowModifier
- `DataRegion`: 处理数据操作，将逻辑 ID 转换为数据 Region ID
- `MetadataRegion`: 使用 KV 结构管理逻辑 Region 元数据
- `MetricEngineState`: 跟踪逻辑 Region 与物理 Region 关联
- `RowModifier`: 写入时添加内部列 (`__table_id`, `__tsid`)
- `FlushMetadataRegionTask`: 周期 flush metadata region 的重复任务

## 架构模式
- **Multiplexer/Proxy**: 将逻辑 Region 操作转换为物理 Region 操作
- **每组两个物理 Region**: 数据 (`to_data_region_id`) + 元数据 (`to_metadata_region_id`)
- **内部 KV 存储**: metadata region 使用 timestamp/key/value schema
- **LRU 缓存**: 128MB, 5min TTL
- **批量操作**: 支持 batch open/catchup/ddl/put
- **RepeatedTask**: 元数据 region 周期 flush
