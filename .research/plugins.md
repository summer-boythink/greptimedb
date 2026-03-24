# plugins/ 模块

## 概述
插件架构框架，定义每种节点类型的 setup/start 钩子。

## 可用插件
当前主要是骨架/框架。唯一有实质逻辑的是 frontend.rs (认证)。

| 模块 | 节点类型 | Setup | Start |
|------|---------|-------|-------|
| frontend.rs | Frontend | setup_frontend_plugins | start_frontend_plugins |
| datanode.rs | Datanode | setup_datanode_plugins | start_datanode_plugins |
| flownode.rs | Flownode | setup_flownode_plugins | start_flownode_plugins |
| meta_srv.rs | Meta Server | setup_metasrv_plugins | start_metasrv_plugins |
| standalone.rs | Standalone | setup_standalone_plugins | start_standalone_plugins |

## 架构模式
- **两阶段生命周期**: 每个节点 `setup_*_plugins()` (初始化时) + `start_*_plugins()` (启动后)
- **Context 结构体**: 每种节点类型有 context 结构体传递配置
