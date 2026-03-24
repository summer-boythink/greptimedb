# datatypes/ 模块

## 概述
GreptimeDB 类型系统，桥接 Arrow 类型系统和内部逻辑类型。

## 核心类型: ConcreteDataType
26 个变体，使用 `enum_dispatch`:
- Null, Boolean
- 整数: Int8/16/32/64, UInt8/16/32/64
- 浮点: Float32/64
- Decimal: Decimal128
- 字符串: String, Binary
- 日期时间: Date, Timestamp (Second/Millisecond/Microsecond/Nanosecond), Time
- Duration: Second/Millisecond/Microsecond/Nanosecond
- Interval: YearMonth/DayTime/MonthDayNano
- 复合: List, Dictionary, Struct
- JSON
- Vector (向量嵌入)

## DataType trait
```rust
pub trait DataType {
    fn name(&self) -> &str;
    fn logical_type_id(&self) -> LogicalTypeId;
    fn default_value(&self) -> Value;
    fn as_arrow_type(&self) -> ArrowDataType;
    fn create_mutable_vector(&self) -> Box<dyn MutableVector>;
    fn try_cast(&self, from: Value) -> Result<Value>;
}
```

## 架构模式
- **enum_dispatch**: 避免动态分发开销
- **Arrow 互操作**: 完整的双向转换
- **宏生成构造器**: `impl_new_concrete_type_functions!`
- **类型内省**: `is_float()`, `is_boolean()`, `is_string()`, `is_numeric()` 等
- **Postgres 兼容**: `postgres_datatype_name()` 类型映射
