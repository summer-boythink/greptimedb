# cli/ 模块

## 概述
命令行工具，用于数据导入导出、元数据管理和基准测试。

## CLI 命令
### Data Commands
- `Export`: V1 遗留数据导出
- `Import`: V1 遗留数据导入
- `ExportV2`: JSON-based Schema 导出 (支持 manifest)
- `ImportV2`: 从 V2 snapshot 导入

### Metadata Commands
- `Snapshot`: 保存和恢复元数据快照
- `Get`: 获取元数据条目
- `Del`: 删除元数据条目
- `Repair`: 诊断和修复元数据

### Bench Commands
- 表元数据 CRUD 基准测试
- 支持 etcd/PostgreSQL/MySQL/memory 后端
- 支持 `--etcd_addr`, `--postgres_addr`, `--mysql_addr`, `--count` 标志

## 架构模式
- Command 模式: 每个 CLI 命令实现 `Tool` trait (async `do_work()`)
- Builder 模式: 命令异步构建运行时组件
- Subcommand 层次: 使用 `clap::Subcommand`
