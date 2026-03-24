# cmd/ 模块

## 概述
顶层入口点和 CLI 编排层，定义 `App` trait 和信号管理。

## App trait
```rust
pub trait App {
    fn name(&self) -> &str;
    async fn pre_start(&mut self) -> Result<()>;
    async fn start(&mut self) -> Result<()>;
    async fn wait_signal(&self);
    async fn stop(&self) -> Result<()>;
    async fn run(&mut self) -> Result<()> { ... }  // 默认实现
}
```

## 生命周期
`pre_start()` -> `start()` -> wait for signal -> `stop()`

## 节点类型
- Datanode: 存储层 Worker
- Frontend: 查询处理和协议处理
- Metasrv: 元数据服务
- Flownode: 流式计算节点
- Standalone: 单节点模式
- CLI: 命令行工具

## 信号管理
- Unix: SIGINT/SIGTERM
- 其他: Ctrl-C

## 架构模式
- **Strategy 模式**: 每个节点类型有自己的模块
- **Builder 模式**: `StartCommand::build()` 构建实例
- **Plugin 系统**: 使用 `Plugins` 和 `PluginOptions` 扩展
