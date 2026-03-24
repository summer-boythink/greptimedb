# standalone/ 模块

## 概述
单节点部署配置和工具。

## 关键类型
- `StandaloneInformationExtension`: 提供单节点模式的系统信息
- `StandaloneRepartitionProcedureFactory`: 单节点模式的 no-op 工厂

## 关键函数
- `build_metadata_kvbackend(dir, config)`: 创建 RaftEngine KV 后端
- `build_metadata_kv_from_url(url)`: 解析 `raftengine://` URL
- `build_procedure_manager(kv_backend, config)`: 创建 LocalManager

## 架构模式
- **Raft Engine 元数据**: 单节点使用 RaftEngineBackend 作为 KV 存储
- **本地过程管理器**: LocalManager 本地处理 DDL 过程
- **No-op 分布式功能**: StandaloneRepartitionProcedureFactory 返回不支持错误
