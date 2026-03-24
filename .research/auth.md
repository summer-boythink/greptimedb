# auth/ 模块

## 概述
可插拔认证授权框架，支持多种用户认证和基于权限的授权。

## 支持的认证方式
1. **StaticUserProvider**: 从静态文件或内联命令加载凭据
2. **WatchFileUserProvider**: 从文件加载并监视变更 (热加载)

## 密码类型
- `PlainText`: 明文密码
- `MysqlNativePassword`: MySQL 原生密码 (double-SHA1)
- `PgMD5`: PostgreSQL MD5 (占位，未实现)

## 权限模型
- `PermissionMode`: ReadWrite (默认), ReadOnly, WriteOnly
- `PermissionReq`: GrpcRequest, SqlStatement, PromQuery, LogQuery, Opentsdb, LineProtocol, PromStoreWrite/Read, Otlp, LogWrite, BulkInsert
- `PermissionResp`: Allow, Reject

## 关键类型
- `UserProvider` trait: `authenticate()`, `authorize()`, `auth()`
- `UserInfo` trait: 已认证用户
- `Identity`: 用户身份 (username + 可选 host)
- `DefaultUserInfo`: 具体 UserInfo 实现
- `DefaultPermissionChecker`: 权限检查器

## 后台任务
- WatchFileUserProvider 使用 `FileWatcherBuilder` 监视凭据文件变更
