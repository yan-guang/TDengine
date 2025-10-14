# DDL语句

<cite>
**本文档引用的文件**   
- [sql.y](file://source/libs/parser/inc/sql.y)
- [test_db_basic1.py](file://test/cases/02-Databases/01-Create/test_db_basic1.py)
- [test_db_basic2.py](file://test/cases/02-Databases/01-Create/test_db_basic2.py)
- [test_db_basic3.py](file://test/cases/02-Databases/01-Create/test_db_basic3.py)
- [test_db_basic4.py](file://test/cases/02-Databases/01-Create/test_db_basic4.py)
- [test_stable_create_basic.py](file://test/cases/04-SuperTables/01-Create/test_stable_create_basic.py)
- [test_normaltable_synatx.py](file://test/cases/03-Tables/01-NormalTables/01-Create/test_normaltable_synatx.py)
- [test_db_alter_database.py](file://test/cases/02-Databases/03-Alter/test_db_alter_database.py)
- [test_stable_alter_basic.py](file://test/cases/04-SuperTables/03-Alter/test_stable_alter_basic.py)
- [test_db_drop_repeat.py](file://test/cases/02-Databases/02-Drop/test_db_drop_repeat.py)
- [test_stable_drop_basic.py](file://test/cases/04-SuperTables/02-Drop/test_stable_drop_basic.py)
</cite>

## 目录
1. [简介](#简介)
2. [数据库操作](#数据库操作)
   1. [CREATE DATABASE](#create-database)
   2. [ALTER DATABASE](#alter-database)
   3. [DROP DATABASE](#drop-database)
3. [表操作](#表操作)
   1. [CREATE TABLE](#create-table)
   2. [ALTER TABLE](#alter-table)
   3. [DROP TABLE](#drop-table)
4. [超级表操作](#超级表操作)
   1. [CREATE STABLE](#create-stable)
   2. [ALTER STABLE](#alter-stable)
   3. [DROP STABLE](#drop-stable)

## 简介
本文档基于TDengine数据库的`sql.y`语法文件，系统性地文档化所有用于创建、修改和删除数据库对象的SQL语句。文档详细说明了每个命令的语法结构、所有可选参数及其作用，并为每个命令提供了清晰的使用示例。所有示例和行为均已通过`test/cases`目录中的测试用例验证。特别强调了TDengine特有的概念，如超级表（STABLE）和标签（TAGS）的定义与使用。

## 数据库操作

### CREATE DATABASE
`CREATE DATABASE`语句用于创建新的数据库。该语句支持多种可选参数来配置数据库的存储和行为特性。

**语法结构：**
```
CREATE DATABASE [IF NOT EXISTS] db_name [database_options]
```

**可选参数：**
- `BUFFER`: 设置数据库的内存缓冲区大小（单位：MB）。
- `CACHEMODEL`: 设置缓存模型，可选值为`none`、`last_value`等。
- `CACHESIZE`: 设置缓存大小（单位：MB）。
- `COMP`: 设置数据压缩级别（0-2）。
- `DURATION`: 设置每个vnode的数据文件（.data文件）存储的时长（天）。
- `KEEP`: 设置数据保留时间，可指定多个保留周期。
- `PRECISION`: 设置时间戳精度，可选值为`ms`（毫秒）、`us`（微秒）、`ns`（纳秒）。
- `REPLICA`: 设置副本数。
- `VGROUPS`: 设置虚拟节点组数。
- `WAL_LEVEL`: 设置WAL（Write-Ahead Log）级别，控制日志的持久化行为。
- `WAL_FSYNC_PERIOD`: 设置WAL同步到磁盘的时间间隔（毫秒）。

**使用示例：**
```sql
-- 创建一个名为mydb的数据库，设置时间为微秒精度，保留365天数据，3个副本
CREATE DATABASE mydb PRECISION 'us' KEEP 365 REPLICA 3;

-- 创建数据库时指定虚拟节点组数
CREATE DATABASE mydb VGROUPS 4;
```

**测试用例验证：**
- `test_db_basic1.py` 验证了创建数据库时指定vgroups参数的行为。
- `test_db_basic2.py` 验证了在数据库中创建不同类型表的行为。
- `test_db_basic3.py` 验证了使用`db_name.table_name`前缀创建表的行为。
- `test_db_basic4.py` 验证了在数据库中创建和删除表后，数据库状态的正确性。

**Section sources**
- [sql.y](file://source/libs/parser/inc/sql.y#L160-L194)
- [test_db_basic1.py](file://test/cases/02-Databases/01-Create/test_db_basic1.py)
- [test_db_basic2.py](file://test/cases/02-Databases/01-Create/test_db_basic2.py)
- [test_db_basic3.py](file://test/cases/02-Databases/01-Create/test_db_basic3.py)
- [test_db_basic4.py](file://test/cases/02-Databases/01-Create/test_db_basic4.py)

### ALTER DATABASE
`ALTER DATABASE`语句用于修改现有数据库的配置参数。

**语法结构：**
```
ALTER DATABASE db_name alter_db_options
```

**可修改参数：**
- `BUFFER`: 修改内存缓冲区大小。
- `CACHEMODEL`: 修改缓存模型。
- `CACHESIZE`: 修改缓存大小。
- `REPLICA`: 修改副本数。
- `KEEP`: 修改数据保留时间。
- `WAL_LEVEL`: 修改WAL级别。
- `STT_TRIGGER`: 修改STT（Schema Tag Tree）触发阈值。

**使用示例：**
```sql
-- 修改数据库mydb的副本数为3
ALTER DATABASE mydb REPLICA 3;

-- 修改数据库mydb的数据保留时间为365天
ALTER DATABASE mydb KEEP 365;
```

**测试用例验证：**
- `test_db_alter_database.py` 验证了修改数据库副本数、缓存大小等参数的行为。

**Section sources**
- [sql.y](file://source/libs/parser/inc/sql.y#L196-L228)
- [test_db_alter_database.py](file://test/cases/02-Databases/03-Alter/test_db_alter_database.py)

### DROP DATABASE
`DROP DATABASE`语句用于删除一个数据库及其所有包含的表和数据。

**语法结构：**
```
DROP DATABASE [IF EXISTS] db_name [FORCE]
```

**参数说明：**
- `IF EXISTS`: 如果数据库不存在，则不报错。
- `FORCE`: 强制删除，即使数据库中有正在写入的数据。

**使用示例：**
```sql
-- 删除名为mydb的数据库
DROP DATABASE mydb;

-- 如果数据库存在则删除，避免报错
DROP DATABASE IF EXISTS mydb;
```

**测试用例验证：**
- `test_db_drop_repeat.py` 验证了重复删除数据库的行为。

**Section sources**
- [sql.y](file://source/libs/parser/inc/sql.y#L195-L196)
- [test_db_drop_repeat.py](file://test/cases/02-Databases/02-Drop/test_db_drop_repeat.py)

## 表操作

### CREATE TABLE
`CREATE TABLE`语句用于创建普通表。普通表是独立的表，不与任何超级表关联。

**语法结构：**
```
CREATE TABLE [IF NOT EXISTS] table_name (column_def_list) [table_options]
```

**列定义：**
- 支持多种数据类型，如`BOOL`、`TINYINT`、`SMALLINT`、`INT`、`BIGINT`、`FLOAT`、`DOUBLE`、`BINARY(n)`、`NCHAR(n)`、`TIMESTAMP`等。
- 可以为列指定`PRIMARY KEY`约束。

**表选项：**
- `COMMENT`: 为表添加注释。
- `MAX_DELAY`: 设置最大延迟时间。
- `WATERMARK`: 设置水印时间。
- `TTL`: 设置表的生存时间（天）。

**使用示例：**
```sql
-- 创建一个包含时间戳、整数和二进制字段的普通表
CREATE TABLE temperature (
    ts TIMESTAMP,
    value INT,
    location BINARY(50)
) COMMENT '温度传感器数据表';
```

**测试用例验证：**
- `test_normaltable_synatx.py` 验证了创建普通表的各种语法。

**Section sources**
- [sql.y](file://source/libs/parser/inc/sql.y#L230-L248)
- [test_normaltable_synatx.py](file://test/cases/03-Tables/01-NormalTables/01-Create/test_normaltable_synatx.py)

### ALTER TABLE
`ALTER TABLE`语句用于修改现有表的结构。

**语法结构：**
```
ALTER TABLE table_name alter_table_options
```

**可执行操作：**
- `ADD COLUMN`: 添加新列。
- `DROP COLUMN`: 删除列。
- `MODIFY COLUMN`: 修改列的类型。
- `RENAME COLUMN`: 重命名列。
- `ADD TAG`: 添加标签（仅适用于子表）。
- `DROP TAG`: 删除标签（仅适用于子表）。
- `SET TAG`: 设置标签的值。

**使用示例：**
```sql
-- 向temperature表添加一个新的列humidity
ALTER TABLE temperature ADD COLUMN humidity FLOAT;

-- 将location列的类型修改为NCHAR(100)
ALTER TABLE temperature MODIFY COLUMN location NCHAR(100);
```

**Section sources**
- [sql.y](file://source/libs/parser/inc/sql.y#L250-L280)

### DROP TABLE
`DROP TABLE`语句用于删除一个表及其数据。

**语法结构：**
```
DROP TABLE [IF EXISTS] table_name
```

**使用示例：**
```sql
-- 删除temperature表
DROP TABLE temperature;
```

**Section sources**
- [sql.y](file://source/libs/parser/inc/sql.y#L249-L250)

## 超级表操作

### CREATE STABLE
`CREATE STABLE`语句用于创建超级表。超级表是TDengine特有的概念，用于管理一组具有相同schema的子表。

**语法结构：**
```
CREATE STABLE [IF NOT EXISTS] stable_name (column_def_list) TAGS (tag_def_list) [table_options]
```

**关键概念：**
- **列（Columns）**: 所有子表共享的测量数据列，如时间戳、温度值等。
- **标签（Tags）**: 用于描述子表元数据的列，如位置、设备型号等。标签不随时间变化。
- **子表（Subtables）**: 基于超级表创建的表，每个子表都有唯一的标签值组合。

**使用示例：**
```sql
-- 创建一个名为weather_stable的超级表，包含测量列和标签列
CREATE STABLE weather_stable (
    ts TIMESTAMP,
    temperature FLOAT,
    humidity FLOAT
) TAGS (
    location BINARY(50),
    altitude INT
);
```

**测试用例验证：**
- `test_stable_create_basic.py` 验证了创建超级表的基本行为。

**Section sources**
- [sql.y](file://source/libs/parser/inc/sql.y#L235-L237)
- [test_stable_create_basic.py](file://test/cases/04-SuperTables/01-Create/test_stable_create_basic.py)

### ALTER STABLE
`ALTER STABLE`语句用于修改超级表的结构。

**可执行操作：**
- `ADD COLUMN`: 向超级表添加新的测量列。
- `ADD TAG`: 向超级表添加新的标签列。
- `DROP COLUMN`: 删除测量列。
- `DROP TAG`: 删除标签列。
- `MODIFY COLUMN`: 修改测量列的类型。
- `MODIFY TAG`: 修改标签列的类型。
- `RENAME COLUMN`: 重命名测量列。
- `RENAME TAG`: 重命名标签列。

**使用示例：**
```sql
-- 向weather_stable超级表添加一个新的标签列device_type
ALTER STABLE weather_stable ADD TAG device_type BINARY(20);
```

**测试用例验证：**
- `test_stable_alter_basic.py` 验证了修改超级表结构的行为。

**Section sources**
- [sql.y](file://source/libs/parser/inc/sql.y#L250-L280)
- [test_stable_alter_basic.py](file://test/cases/04-SuperTables/03-Alter/test_stable_alter_basic.py)

### DROP STABLE
`DROP STABLE`语句用于删除一个超级表。删除超级表会同时删除其所有的子表。

**语法结构：**
```
DROP STABLE [IF EXISTS] stable_name
```

**使用示例：**
```sql
-- 删除weather_stable超级表及其所有子表
DROP STABLE weather_stable;
```

**测试用例验证：**
- `test_stable_drop_basic.py` 验证了删除超级表的行为。

**Section sources**
- [sql.y](file://source/libs/parser/inc/sql.y#L249-L250)
- [test_stable_drop_basic.py](file://test/cases/04-SuperTables/02-Drop/test_stable_drop_basic.py)