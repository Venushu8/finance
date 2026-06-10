---
name: kingdee-integration
description: "金蝶 ERP 数据库对接基础设施层：提供统一的金蝶数据库连接管理、CRUD 操作、事务处理及数据映射框架。所有 domain skill 依赖此 skill 实现金蝶数据读写"
user-invocable: true
allowed-tools: Bash(multica *), Bash(mysql *), Bash(python3 *)
---

# Kingdee Integration (金蝶 ERP 数据库对接)

> 基础设施层 skill。所有 domain skill（应收、应付、总账等）依赖此 skill 实现金蝶数据库的统一读写。

## 数据库连接管理

```python
# 连接配置示例（实际由环境变量或配置文件注入）
KINGDEE_CONFIG = {
    "host": "${KINGDEE_DB_HOST}",
    "port": 3306,
    "user": "${KINGDEE_DB_USER}",
    "password": "${KINGDEE_DB_PASS}",
    "database": "${KINGDEE_DB_NAME}",
    "charset": "utf8mb4"
}
```

支持连接池复用、自动重连、读写分离。

## 通用 CRUD 接口

```python
class KingdeeClient:
    def query(sql: str, params: dict = None) -> list[dict]
    def insert(table: str, data: dict) -> int
    def update(table: str, data: dict, where: dict) -> int
    def delete(table: str, where: dict) -> int
    def call_proc(proc_name: str, args: list) -> any
```

所有数据库操作使用参数化查询防止 SQL 注入。

## 数据映射器

```python
class BaseMapper:
    """
    将金蝶数据库行记录映射为 Python 数据对象。
    每个 domain skill 的 mapper 继承此类，实现字段名转换、
    类型转换和业务默认值。
    """
    table_name: str
    primary_key: str
    field_mapping: dict[str, str]   # {python_field: db_column}
    type_converters: dict[str, Callable]
```

## 事务处理

- 显式事务：`begin_transaction()` / `commit()` / `rollback()`
- 自动重试：可配置重试次数和间隔，处理死锁错误 (1213)
- 分布式事务：支持 XA 协议两阶段提交（如需跨数据库实例）

## 错误处理策略

| 错误代码 | 金蝶错误 | 处理方式 |
|---------|---------|---------|
| ERR_CONN | 连接超时/断开 | 自动重连最多 3 次，间隔 2s 指数退避 |
| ERR_DUP | 主键/唯一键冲突 | 返回冲突详情，由调用方决定更新或跳过 |
| ERR_LOCK | 锁等待超时 | 重试最多 3 次，超时后回滚并上报 |
| ERR_FK | 外键约束失败 | 校验关联数据，返回详细错误上下文 |
| ERR_PERM | 权限不足 | 检查连接用户权限，日志审计告警 |

## 数据字典

统一数据字典接口，支持按表名查询字段注释、数据类型和约束：

```sql
-- 金蝶表结构查询
SELECT COLUMN_NAME, DATA_TYPE, COLUMN_COMMENT, IS_NULLABLE, COLUMN_DEFAULT
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_SCHEMA = ? AND TABLE_NAME = ?
```

## 安全规范

- 所有连接使用最小权限数据库账户
- 敏感字段（银行账号、身份证号）查询时自动脱敏
- SQL 操作日志记录完整的审计轨迹（谁、什么时间、改了哪些表）
- 写操作前自动校验参数类型，防止 SQL 注入
