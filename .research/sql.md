# sql/ 模块

## 概述
SQL 解析层，包装和扩展 `sqlparser`，支持 GreptimeDB 特有语法。

## 关键类型
- `ParserContext`: 主 SQL 解析器包装
- `Statement` enum: CreateTable, AlterTable, DropTable, Insert, Select, Delete, Copy, Truncate, Show, Describe, Explain, Use, Kill, TQL, Admin, Cursor, Comment, SetVariables 等

## 解析流程
1. `ParserContext::create_with_dialect(sql, dialect, opts)` 解析 SQL
2. `parse_statement()` 按首个关键字分派 (CREATE/SELECT/INSERT/ALTER 等)
3. 自定义扩展: TQL 语句, 自定义 CREATE TABLE (TIME INDEX/ENGINE/PARTITION BY), KILL QUERY
4. `transform_statements()` 后处理
5. 未引用标识符小写化

## 架构模式
- **Wrapper 模式**: 包装 sqlparser::Parser 并扩展
- **关键字分派**: 按 SQL 关键字分派解析
- **Dialect 抽象**: 支持 MySQL/PostgreSQL/Generic 方言
