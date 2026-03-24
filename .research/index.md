# index/ 模块

## 概述
四种索引策略，用于高效数据检索。

## 索引类型
| 类型 | 模块 | 描述 |
|------|------|------|
| Inverted Index | inverted_index/ | 使用 FST Map 映射值到行位置 |
| Bloom Filter | bloom_filter/ | 概率成员测试，分段创建 |
| Fulltext Index | fulltext_index/ | 文本搜索 (English/Chinese tokenizer) |
| Vector Index | vector/ | KNN 搜索 (USearch HNSW) |

## 架构模式
- **Creator/Reader/Applier 模式**: 每种索引有创建/读取/应用三个接口
- **外部内存管理**: `ExternalTempFileProvider` 允许将中间数据外存
- **Bitmap 抽象**: `Bitmap` enum (Roaring/BitVec) 提供统一接口
- **分段布隆过滤器**: 按行分段，每段独立创建
- **可配置距离度量**: 向量索引支持 L2/Cosine/InnerProduct
