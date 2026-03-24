# puffin/ 模块

## 概述
Apache Puffin 文件格式实现，用于存储列数据格式 (如 Apache Iceberg) 的元数据和统计信息。

## 文件格式
```
Magic Blob₁ Blob₂ ... Blobₙ Footer
```
- Magic: `[0x50, 0x46, 0x41, 0x31]` ("PF A1")
- Footer: Magic + FooterPayload (JSON, 可选 LZ4 压缩) + PayloadSize + Flags + Magic

## 关键类型
- `PuffinManager` trait: 创建 PuffinReader/PuffinWriter
- `PuffinWriter` trait: 写入 blobs 和目录
- `PuffinReader` trait: 读取 blobs 和目录
- `BlobGuard/DirGuard` trait: 数据访问保护
- `FileMetadata`: 文件级元数据 (blob list + properties)
- `BlobMetadata`: 每 blob 元数据 (type, offset, length, compression)
- `CompressionCodec`: Lz4 或 Zstd

## 架构模式
- **Trait-based 抽象**: 允许不同存储后端 (fs/对象存储)
- **Guard 模式**: Blob/目录访问受 guard 对象保护
- **Builder 模式**: FileMetadataBuilder/BlobMetadataBuilder
- **缓存层**: fs_puffin_manager 包含缓存和 staging
