# object-store/ 模块

## 概述
对象存储抽象层，包装 OpenDAL 库，支持多种后端 (Local/S3/OSS/Azure/GCS)。

## 关键类型
- `ObjectStore` (re-exported `opendal::Operator`)
- `ObjectStoreConfig`: Tagged enum (File/S3/Oss/Azblob/Gcs)
- `S3Config/OssConfig/AzblobConfig/GcsConfig/FileConfig`: 后端配置
- `HttpClientConfig`: HTTP 客户端设置
- `ObjectStoreManager`: 管理多个命名 ObjectStore 实例

## 支持的后端
- Local filesystem
- AWS S3
- Alibaba Cloud OSS
- Azure Blob Storage
- Google Cloud Storage

## 架构模式
- **Facade 模式**: 包装 OpenDAL Operator
- **Factory 模式**: `new_raw_object_store()` 根据配置创建后端
- **Builder 模式**: 每个后端使用 OpenDAL builder
- **Layer/中间件**: OpenDAL layers (HttpClientLayer, PrometheusLayer)
- **多存储注册表**: `ObjectStoreManager` 使用 `HashMap<String, ObjectStore>`
