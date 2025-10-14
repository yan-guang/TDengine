# DML语句

<cite>
**Referenced Files in This Document**   
- [sql.y](file://source/libs/parser/inc/sql.y)
- [test_write.py](file://test/cases/10-DataIngestion/test_write.py)
- [test_write_sml_influxdb_line.py](file://test/cases/10-DataIngestion/03-SML/test_write_sml_influxdb_line.py)
- [test_write_sml_opentsdb_telnet.py](file://test/cases/10-DataIngestion/03-SML/test_write_sml_opentsdb_telnet.py)
- [test_delete.py](file://test/cases/11-DataDeletion/test_delete.py)
</cite>

## 目录
1. [简介](#简介)
2. [INSERT语句](#insert语句)
   - [普通写入模式](#普通写入模式)
   - [Schemaless写入模式](#schemaless写入模式)
   - [批量插入](#批量插入)
3. [DELETE语句](#delete语句)
   - [使用限制](#使用限制)
   - [注意事项](#注意事项)
4. [常见错误](#常见错误)

## 简介
本文档基于`sql.y`语法文件和相关测试用例，系统性地文档化TDengine数据库中的数据操作语言（DML）语句。重点介绍`INSERT`和`DELETE`语句的语法、使用模式、参数说明和常见错误。内容与`test/cases/10-DataIngestion`和`11-DataDeletion`中的测试用例保持一致。

**Section sources**
- [sql.y](file://source/libs/parser/inc/sql.y#L0-L2095)
- [test_write.py](file://test/cases/10-DataIngestion/test_write.py#L0-L489)

## INSERT语句
`INSERT`语句用于向TDengine数据库中插入数据。支持多种写入模式，包括普通写入、Schemaless写入和批量插入。

### 普通写入模式
普通写入模式遵循标准的SQL `INSERT`语法，将数据插入到指定的表中。

**语法定义**
```
INSERT INTO table_name (column_list) VALUES (value_list);
```

**参数说明**
- `table_name`: 目标表的名称。
- `column_list`: 要插入数据的列名列表。
- `value_list`: 对应列的数据值列表。

**使用示例**
```sql
INSERT INTO t1 (ts, c1, c2) VALUES (NOW, 10, 2.0);
```

**Section sources**
- [sql.y](file://source/libs/parser/inc/sql.y#L0-L2095)
- [test_write.py](file://test/cases/10-DataIngestion/test_write.py#L0-L489)

### Schemaless写入模式
Schemaless写入模式允许通过`TELNET`或`MQTT`协议模拟写入数据，无需预先定义表结构。

**语法定义**
```
<measurement> <timestamp> <value> <tags>
```

**参数说明**
- `measurement`: 测量名称，对应表名。
- `timestamp`: 时间戳，支持纳秒、微秒、毫秒和秒。
- `value`: 数据值。
- `tags`: 标签，用于描述数据的元信息。

**使用示例**
```sql
-- 通过TELNET协议写入
put temperature 1626006833640 25.5 location=room1

-- 通过MQTT协议写入
temperature,location=room1 value=25.5 1626006833640
```

**Section sources**
- [sql.y](file://source/libs/parser/inc/sql.y#L0-L2095)
- [test_write_sml_opentsdb_telnet.py](file://test/cases/10-DataIngestion/03-SML/test_write_sml_opentsdb_telnet.py#L0-L1514)

### 批量插入
批量插入允许在一条`INSERT`语句中插入多行数据，提高写入效率。

**语法定义**
```
INSERT INTO table_name (column_list) VALUES (value_list1), (value_list2), ...;
```

**参数说明**
- `table_name`: 目标表的名称。
- `column_list`: 要插入数据的列名列表。
- `value_list1, value_list2, ...`: 多组数据值列表。

**使用示例**
```sql
INSERT INTO t1 (ts, c1, c2) VALUES 
(NOW, 10, 2.0),
(NOW + 1s, 11, 2.1),
(NOW + 2s, 12, 2.2);
```

**Section sources**
- [sql.y](file://source/libs/parser/inc/sql.y#L0-L2095)
- [test_write.py](file://test/cases/10-DataIngestion/test_write.py#L0-L489)

## DELETE语句
`DELETE`语句用于从TDengine数据库中删除数据。支持从普通表、超级表和子表中删除数据。

### 使用限制
- `DELETE`语句只能删除满足条件的数据行。
- 不能删除表结构，只能删除数据。
- 删除操作不可逆，请谨慎使用。

**语法定义**
```
DELETE FROM table_name WHERE condition;
```

**参数说明**
- `table_name`: 目标表的名称。
- `condition`: 删除条件，用于指定要删除的数据行。

**使用示例**
```sql
-- 删除普通表中的数据
DELETE FROM ntb WHERE ts < NOW - 1d;

-- 删除超级表中的数据
DELETE FROM stb WHERE ts < NOW - 1d;

-- 删除子表中的数据
DELETE FROM ct1 WHERE ts < NOW - 1d;
```

**Section sources**
- [sql.y](file://source/libs/parser/inc/sql.y#L0-L2095)
- [test_delete.py](file://test/cases/11-DataDeletion/test_delete.py#L0-L169)

### 注意事项
- 删除操作会影响数据的完整性，请确保删除条件正确。
- 删除大量数据时，建议分批删除，避免影响系统性能。
- 删除操作会触发WAL（Write-Ahead Log）记录，确保数据的一致性。

**Section sources**
- [sql.y](file://source/libs/parser/inc/sql.y#L0-L2095)
- [test_delete.py](file://test/cases/11-DataDeletion/test_delete.py#L0-L169)

## 常见错误
以下是一些常见的`INSERT`和`DELETE`语句错误及其解决方案。

**错误1: 时间戳格式错误**
- **错误信息**: `Timestamp data out of range`
- **原因**: 时间戳格式不正确或超出范围。
- **解决方案**: 确保时间戳格式正确，支持纳秒、微秒、毫秒和秒。

**错误2: 数据类型不匹配**
- **错误信息**: `Data type mismatch`
- **原因**: 插入的数据类型与表定义的类型不匹配。
- **解决方案**: 检查数据类型，确保匹配。

**错误3: 删除条件不明确**
- **错误信息**: `DELETE statement without WHERE clause`
- **原因**: `DELETE`语句缺少`WHERE`条件。
- **解决方案**: 添加`WHERE`条件，明确删除范围。

**Section sources**
- [test_write.py](file://test/cases/10-DataIngestion/test_write.py#L0-L489)
- [test_delete.py](file://test/cases/11-DataDeletion/test_delete.py#L0-L169)