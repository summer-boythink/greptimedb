# pipeline/ 模块

## 概述
日志数据 ETL 框架，允许用户定义 YAML pipeline 解析、处理和转换原始日志/事件数据。

## 18 种处理器
csv, date, epoch, dissect, json_parse, json_path, vrl, regex, gsub, filter, select, join, digest, decolorize, letter, urlencoding, cmcd, simple_extract

## 关键类型
- `Pipeline`: 核心定义 (processors, transformer, dispatcher, table_suffix)
- `TransformerMode`: GreptimeTransformer (显式) 或 AutoTransform (自动推断)
- `PipelineExecOutput`: Transformed/DispatchedTo/Filtered
- `Dispatcher`: 按字段值路由到不同表/pipeline
- `PipelineOperator`: 管理 pipeline 生命周期 (CRUD, 缓存, 编译)
- `ContextOpt`: 每行数据库选项 (auto_create_table, ttl, append_mode, skip_wal 等)

## 三阶段执行 (`Pipeline::exec_mut`)
1. **Process**: 按顺序在输入值上运行处理器
2. **Dispatch**: 检查分发规则; 匹配则返回分发目标
3. **Transform**: 将处理后的值转换为数据库 Row 对象

## 架构模式
- **YAML 配置**: Pipeline 通过 YAML 定义
- **Versioned Pipelines**: V1 (显式 transform) 和 V2 (transform + auto-transform)
- **缓存**: PipelineOperator 每 catalog 维护 RwLock<HashMap> 缓存
