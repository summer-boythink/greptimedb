# log-store/ 模块

## 概述
Write-Ahead Log (WAL) 实现，支持 Kafka (分布式) 和 Raft Engine (本地嵌入式) 两种后端。

## 关键类型
- `KafkaLogStore`: Kafka-backed WAL
- `RaftEngineBackend`: 本地 Raft Engine WAL 后端
- `BackgroundProducerWorker`: 批量写入 Kafka 的异步 worker
- `OrderedBatchProducer`: 聚合多个 produce 请求
- `PeriodicTopicStatsReporter`: 周期报告 Topic 统计
- `PeriodicOffsetFetcher`: 周期获取最新 offset

## 架构模式
1. **LogStore trait**: Kafka 和 Raft Engine 实现统一接口
2. **Producer-Worker 模式**: `OrderedBatchProducer` spawn `BackgroundProducerWorker`
3. **多部分条目拆分**: 大条目拆分为 First/Middle/Last
4. **Namespace 隔离**: 每个 Region 独立 Topic 或 Namespace
5. **索引感知读取**: 支持可选 `WalIndex` 跳过无关 offset

## 后台任务
- `BackgroundProducerWorker::run()`: 事件循环接收 WorkerRequest, 批量写入
- `PeriodicOffsetFetcher`: 定期获取最新 offset
- `PeriodicTopicStatsReporter`: 定期报告 Topic 统计
