# mito-codec/ 模块

## 概述
Mito 存储引擎的底层编解码器工具，使用 `memcomparable` 编码方案。

## 关键类型
- `IndexValueCodec`: 非空索引值静态编码器
- `IndexValuesCodec`: 主键值解码器
- `PkColInfo`: 主键列信息 (index + SortField)
- `KeyValues/KeyValuesRef`: protobuf Mutation 视图 (owned vs borrowed)
- `DensePrimaryKeyFilter`: 密集编码主键过滤器
- `SparsePrimaryKeyFilter`: 稀疏编码主键过滤器
- `CompiledFastPath`: 预编译过滤操作符
- `DensePrimaryKeyCodec`: 固定顺序编码所有列
- `SparsePrimaryKeyCodec`: column-id + value 对编码

## 架构模式
- **Strategy 模式**: `PrimaryKeyCodec` trait + Dense/Sparse 实现
- **Factory 模式**: `build_primary_key_codec()` 根据 `PrimaryKeyEncoding` 创建
- **Compiled Fast Path**: 预编码过滤器字面量避免重复序列化
