# SQL语法参考

<cite>
**本文档中引用的文件**  
- [sql.y](file://source/libs/parser/inc/sql.y)
- [parAst.h](file://source/libs/parser/inc/parAst.h)
- [parUtil.h](file://source/libs/parser/inc/parUtil.h)
- [nodes.h](file://include/libs/nodes.h)
- [test/cases/24-TimeSeriesExtensions](file://test/cases/24-TimeSeriesExtensions)
- [test/cases/20-DataQuerying](file://test/cases/20-DataQuerying)
- [test/cases/03-Tables](file://test/cases/03-Tables)
- [test/cases/02-Databases](file://test/cases/02-Databases)
</cite>

## 目录
1. [简介](#简介)
2. [数据定义语言（DDL）](#数据定义语言ddl)
   - [数据库操作](#数据库操作)
   - [表与超级表操作](#表与超级表操作)
   - [索引操作](#索引操作)
   - [用户与权限管理](#用户与权限管理)
   - [集群与节点管理](#集群与节点管理)
3. [数据操作语言（DML）](#数据操作语言dml)
   - [数据插入](#数据插入)
   - [数据删除](#数据删除)
   - [数据更新](#数据更新)
4. [查询语言（SELECT）](#查询语言select)
   - [基础查询结构](#基础查询结构)
   - [时序扩展子句](#时序扩展子句)
   - [窗口函数与聚合](#窗口函数与聚合)
   - [连接查询](#连接查询)
5. [时序特有语法扩展](#时序特有语法扩展)
   - [INTERVAL子句](#interval子句)
   - [FILL子句](#fill子句)
   - [SESSION_WINDOW子句](#session_window子句)
   - [STATE_WINDOW子句](#state_window子句)
   - [SLIDING子句](#sliding子句)
6. [SHOW语句](#show语句)
7. [错误代码参考](#错误代码参考)
8. [附录：测试用例覆盖](#附录测试用例覆盖)

## 简介
本手册系统性地文档化TDengine数据库支持的SQL语法，涵盖数据定义语言（DDL）、数据操作语言（DML）和查询语言（SELECT）。重点说明TDengine为时序数据处理扩展的SQL语法，如`INTERVAL`、`FILL`、`SESSION_WINDOW`等子句。所有语法结构均基于`sql.y`语法文件定义，并与`test/cases`目录下的测试用例保持一致，确保覆盖所有边缘情况。

**本文档中引用的文件**
- [sql.y](file://source/libs/parser/inc/sql.y)

## 数据定义语言（DDL）

### 数据库操作
支持创建、删除、修改和使用数据库。创建数据库时可指定多种选项，如缓存大小、副本数、数据保留策略等。

```sql
CREATE DATABASE [IF NOT EXISTS] db_name [option1 value1] [option2 value2]...
DROP DATABASE [IF EXISTS] db_name [FORCE]
ALTER DATABASE db_name [option value]
USE db_name
```

**参数说明：**
- `IF NOT EXISTS`：若数据库已存在则不报错
- `IF EXISTS`：若数据库不存在则不报错
- `FORCE`：强制删除数据库，即使有活跃连接
- 选项包括：`BUFFER`, `CACHEMODEL`, `CACHESIZE`, `COMP`, `DURATION`, `KEEP`, `REPLICA`, `WAL_LEVEL`等

**使用示例：**
```sql
CREATE DATABASE power_db KEEP 365,315,180 DURATION 10d REPLICAS 3;
ALTER DATABASE power_db KEEP 730;
```

**可能的错误代码：**
- `TSDB_CODE_PAR_SYNTAX_ERROR`：语法错误
- `TSDB_CODE_PAR_DB_NOT_SPECIFIED`：未指定数据库
- `TSDB_CODE_TDB_DB_NOT_EXIST`：数据库不存在

**本节来源**
- [sql.y](file://source/libs/parser/inc/sql.y#L280-L350)

### 表与超级表操作
支持创建普通表、超级表（STABLE）、虚拟表（VTABLE）及其子表。超级表是TDengine的核心概念，用于定义一组具有相同Schema的子表。

```sql
-- 创建超级表
CREATE STABLE [IF NOT EXISTS] stb_name (column_def) TAGS (tag_def) [TABLE_OPTIONS];

-- 创建子表
CREATE TABLE [IF NOT EXISTS] tb_name USING stb_name TAGS (tag_values);

-- 创建多子表
CREATE TABLE tb1 USING stb_name TAGS (v1) , tb2 USING stb_name TAGS (v2);

-- 修改表结构
ALTER TABLE tb_name ADD COLUMN col_name type;
ALTER STABLE stb_name ADD TAG tag_name type;
```

**参数说明：**
- `column_def`：列定义，格式为`col_name type [options]`
- `tag_def`：标签定义，格式为`tag_name type`
- `TABLE_OPTIONS`：表选项，如`TTL`, `SMA`, `ROLLUP`等
- 支持的数据类型包括：`BOOL`, `TINYINT`, `SMALLINT`, `INT`, `BIGINT`, `FLOAT`, `DOUBLE`, `BINARY(n)`, `NCHAR(n)`, `TIMESTAMP`, `JSON`等

**使用示例：**
```sql
CREATE STABLE weather (ts TIMESTAMP, temperature FLOAT, humidity INT) TAGS (location BINARY(20), groupId INT);
CREATE TABLE beijing USING weather TAGS ('Beijing', 1);
```

**可能的错误代码：**
- `TSDB_CODE_PAR_INVALID_COLUMN_DEF`：无效的列定义
- `TSDB_CODE_PAR_INVALID_TAGS_DEF`：无效的标签定义
- `TSDB_CODE_TDB_TABLE_NOT_EXIST`：表不存在

**本节来源**
- [sql.y](file://source/libs/parser/inc/sql.y#L352-L470)

### 索引操作
支持创建和删除索引，包括SMA（Summary Metadata Acceleration）索引和普通索引。

```sql
-- 创建SMA索引
CREATE SMA INDEX [IF NOT EXISTS] index_name ON tb_name (col_name) FUNCTION (func_list) INTERVAL (interval);

-- 创建普通索引
CREATE INDEX [IF NOT EXISTS] index_name ON tb_name (col_name_list);

-- 删除索引
DROP INDEX [IF EXISTS] index_name;
```

**参数说明：**
- `SMA INDEX`：针对时序数据优化的索引，支持预计算聚合函数
- `FUNCTION`：指定预计算的聚合函数，如`SUM`, `AVG`, `MIN`, `MAX`
- `INTERVAL`：指定聚合的时间间隔
- `col_name_list`：普通索引的列列表

**使用示例：**
```sql
CREATE SMA INDEX temp_idx ON weather (temperature) FUNCTION (avg, max) INTERVAL (1h);
```

**可能的错误代码：**
- `TSDB_CODE_PAR_INVALID_INDEX_DEF`：无效的索引定义
- `TSDB_CODE_PAR_INDEX_ALREADY_EXISTS`：索引已存在

**本节来源**
- [sql.y](file://source/libs/parser/inc/sql.y#L1000-L1030)

### 用户与权限管理
支持用户创建、修改、删除和权限管理。

```sql
-- 用户管理
CREATE USER user_name PASS 'password' [SYSINFO 1|0] [CREATEDB 1|0];
ALTER USER user_name PASS 'new_password';
DROP USER user_name;

-- 权限管理
GRANT READ,WRITE ON db_name.* TO user_name;
REVOKE ALL ON *.tb_name FROM user_name;
```

**参数说明：**
- `SYSINFO`：是否允许查看系统信息
- `CREATEDB`：是否允许创建数据库
- 权限类型：`READ`, `WRITE`, `ALTER`, `ALL`
- 权限级别：`db.*`, `db.tb`, `*.*`

**使用示例：**
```sql
CREATE USER analyst PASS 'secret' SYSINFO 1 CREATEDB 0;
GRANT READ ON power_db.* TO analyst;
```

**可能的错误代码：**
- `TSDB_CODE_PAR_USER_ALREADY_EXISTS`：用户已存在
- `TSDB_CODE_PAR_USER_NOT_EXIST`：用户不存在
- `TSDB_CODE_PAR_INVALID_PRIVILEGE`：无效的权限

**本节来源**
- [sql.y](file://source/libs/parser/inc/sql.y#L100-L150)

### 集群与节点管理
支持集群级别的节点管理操作。

```sql
-- 节点管理
CREATE DNODE 'endpoint';
DROP DNODE dnodeId [FORCE];
ALTER DNODE dnodeId 'role';

-- 组件管理
CREATE QNODE ON DNODE dnodeId;
CREATE SNODE ON DNODE dnodeId;
CREATE MNODE ON DNODE dnodeId;

-- 集群管理
ALTER CLUSTER 'config_key' 'config_value';
```

**参数说明：**
- `DNODE`：数据节点，负责数据存储
- `QNODE`：查询节点，负责查询处理
- `SNODE`：流计算节点，负责流处理
- `MNODE`：管理节点，负责元数据管理

**使用示例：**
```sql
CREATE DNODE 'taosdemo:6030';
CREATE QNODE ON DNODE 1;
```

**可能的错误代码：**
- `TSDB_CODE_PAR_INVALID_DNODE_ENDPOINT`：无效的节点端点
- `TSDB_CODE_PAR_DNODE_NOT_EXIST`：节点不存在

**本节来源**
- [sql.y](file://source/libs/parser/inc/sql.y#L200-L270)

## 数据操作语言（DML）

### 数据插入
支持单条、批量和无模式（schemaless）数据插入。

```sql
-- 标准插入
INSERT INTO tb_name VALUES (ts, col1, col2, ...);

-- 指定列插入
INSERT INTO tb_name (ts, col1, col2) VALUES (ts_val, val1, val2);

-- 多表插入
INSERT INTO tb1 VALUES (ts, val1) tb2 VALUES (ts, val2);

-- 无模式插入
INSERT INTO tb_name USING stb_name TAGS (tag_values) VALUES (ts, val);
```

**参数说明：**
- 第一列必须是时间戳
- 支持多种数据格式：行协议、JSON、InfluxDB Line Protocol
- 批量插入时，多条记录用空格分隔

**使用示例：**
```sql
INSERT INTO beijing VALUES ('2023-01-01 00:00:00', 25.5, 60);
INSERT INTO beijing USING weather TAGS ('Beijing', 1) VALUES ('2023-01-01 00:01:00', 25.6, 61);
```

**可能的错误代码：**
- `TSDB_CODE_PAR_INVALID_INSERT_VALUE`：无效的插入值
- `TSDB_CODE_PAR_WRONG_VALUE_TYPE`：值类型错误
- `TSDB_CODE_TDB_INVALID_TIMESTAMP`：无效的时间戳

**本节来源**
- [sql.y](file://source/libs/parser/inc/sql.y#L500-L550)

### 数据删除
支持删除表和数据。

```sql
-- 删除表
DROP TABLE [IF EXISTS] tb_name;

-- 删除超级表
DROP STABLE [IF EXISTS] stb_name;

-- 删除虚拟表
DROP VTABLE [IF EXISTS] vtb_name;
```

**参数说明：**
- `IF EXISTS`：若表不存在则不报错
- 删除超级表会同时删除其所有子表

**使用示例：**
```sql
DROP TABLE beijing;
DROP STABLE weather;
```

**可能的错误代码：**
- `TSDB_CODE_TDB_TABLE_NOT_EXIST`：表不存在
- `TSDB_CODE_TDB_STABLE_HAS_CHILDREN`：超级表仍有子表

**本节来源**
- [sql.y](file://source/libs/parser/inc/sql.y#L450-L470)

### 数据更新
目前TDengine不支持标准的UPDATE语句，数据更新通过重新插入实现。

```sql
-- 通过重新插入实现更新
INSERT INTO tb_name VALUES (ts, new_val1, new_val2);
```

**说明：**
- TDengine采用追加写入模型，不支持原地更新
- 相同时间戳的数据会自动合并，最新值优先

**可能的错误代码：**
- `TSDB_CODE_PAR_INVALID_INSERT_VALUE`：无效的插入值

**本节来源**
- [sql.y](file://source/libs/parser/inc/sql.y)

## 查询语言（SELECT）

### 基础查询结构
支持标准的SELECT查询语法。

```sql
SELECT select_expr_list FROM table_list [WHERE condition] [GROUP BY group_expr_list] [HAVING condition] [ORDER BY order_expr_list] [LIMIT limit_count] [OFFSET offset_count];
```

**参数说明：**
- `select_expr_list`：选择的表达式列表，支持列名、函数、常量等
- `table_list`：表列表，支持多表查询
- `WHERE`：过滤条件，支持时间范围、标签过滤等
- `GROUP BY`：分组表达式，支持按标签分组
- `HAVING`：分组后过滤条件
- `ORDER BY`：排序表达式
- `LIMIT/OFFSET`：结果限制和偏移

**使用示例：**
```sql
SELECT AVG(temperature), MAX(humidity) FROM weather WHERE ts > '2023-01-01' GROUP BY location;
```

**可能的错误代码：**
- `TSDB_CODE_PAR_INVALID_SELECT_EXPR`：无效的选择表达式
- `TSDB_CODE_PAR_INVALID_WHERE_CLAUSE`：无效的WHERE子句

**本节来源**
- [sql.y](file://source/libs/parser/inc/sql.y#L600-L700)

### 时序扩展子句
TDengine扩展了标准SQL，支持时序特有的查询子句。

```sql
SELECT ... FROM ... [WHERE ...] [GROUP BY ...] [INTERVAL(interval) [FILL(fill_type)] [SLIDING(slide_interval)] [SESSION_WINDOW(gap_interval)] [STATE_WINDOW(col_name)]]
```

**参数说明：**
- `INTERVAL`：时间窗口间隔
- `FILL`：空值填充方式
- `SLIDING`：滑动窗口间隔
- `SESSION_WINDOW`：会话窗口间隔
- `STATE_WINDOW`：状态窗口列

**使用示例：**
```sql
SELECT AVG(temperature) FROM weather INTERVAL(1h) FILL(LINEAR);
SELECT COUNT(*) FROM events SESSION_WINDOW(10m);
```

**可能的错误代码：**
- `TSDB_CODE_PAR_INVALID_INTERVAL`：无效的时间间隔
- `TSDB_CODE_PAR_INVALID_FILL_TYPE`：无效的填充类型

**本节来源**
- [sql.y](file://source/libs/parser/inc/sql.y#L800-L900)

### 窗口函数与聚合
支持多种聚合函数和窗口函数。

```sql
SELECT COUNT(*), AVG(col), SUM(col), MIN(col), MAX(col), STDDEV(col), LEASTSQUARES(col), PERCENTILE(col, 90) FROM tb_name;
```

**支持的函数：**
- 聚合函数：`COUNT`, `SUM`, `AVG`, `MIN`, `MAX`, `STDDEV`
- 时序函数：`DIFF`, `SPREAD`, `DERIVATIVE`, `MOVING_AVERAGE`
- 统计函数：`PERCENTILE`, `LEASTSQUARES`, `HISTOGRAM`
- 选择函数：`FIRST`, `LAST`, `TOP`, `BOTTOM`

**使用示例：**
```sql
SELECT DERIVATIVE(temperature) FROM sensor_data;
SELECT PERCENTILE(value, 95) FROM metrics;
```

**可能的错误代码：**
- `TSDB_CODE_PAR_INVALID_FUNC_NAME`：无效的函数名
- `TSDB_CODE_PAR_INVALID_FUNC_PARAM`：无效的函数参数

**本节来源**
- [sql.y](file://source/libs/parser/inc/sql.y#L900-L950)

### 连接查询
支持表之间的连接查询。

```sql
SELECT ... FROM table1 [INNER|LEFT|RIGHT] JOIN table2 ON condition [WINDOW ...];
```

**参数说明：**
- 支持内连接、左连接、右连接
- 连接条件通常基于时间戳对齐
- 支持在连接上应用窗口函数

**使用示例：**
```sql
SELECT t1.temperature, t2.pressure FROM temp_tb t1 INNER JOIN press_tb t2 ON t1.ts = t2.ts;
```

**可能的错误代码：**
- `TSDB_CODE_PAR_INVALID_JOIN_CONDITION`：无效的连接条件
- `TSDB_CODE_PAR_UNSUPPORTED_JOIN_TYPE`：不支持的连接类型

**本节来源**
- [sql.y](file://source/libs/parser/inc/sql.y#L750-L790)

## 时序特有语法扩展

### INTERVAL子句
`INTERVAL`子句用于将时间序列数据按固定时间间隔进行分组和聚合。

```sql
INTERVAL(time_interval) [SLIDING(slide_interval)]
```

**参数说明：**
- `time_interval`：时间窗口的大小，如`1h`, `30m`, `10s`
- `SLIDING`：滑动窗口的步长，如果不指定则为`time_interval`

**工作原理：**
- 将时间轴划分为连续的、不重叠的时间窗口
- 对每个窗口内的数据进行聚合计算
- 支持与`GROUP BY`结合，实现多维度的时序分析

**使用示例：**
```sql
-- 每小时计算一次平均温度
SELECT AVG(temperature) FROM weather INTERVAL(1h);

-- 每15分钟滑动一次，计算过去1小时的平均温度
SELECT AVG(temperature) FROM weather INTERVAL(1h) SLIDING(15m);
```

**可能的错误代码：**
- `TSDB_CODE_PAR_INVALID_INTERVAL`：无效的时间间隔格式
- `TSDB_CODE_PAR_INTERVAL_TOO_SMALL`：时间间隔过小
- `TSDB_CODE_PAR_SLIDING_LARGER_THAN_INTERVAL`：滑动间隔大于窗口间隔

**本节来源**
- [sql.y](file://source/libs/parser/inc/sql.y#L850-L860)

### FILL子句
`FILL`子句用于处理时间窗口中没有数据的空缺。

```sql
FILL(fill_type [, fill_value])
```

**支持的填充类型：**
- `NONE`：不填充，返回NULL
- `VALUE`：用指定值填充
- `PREV`：用前一个非NULL值填充
- `LINEAR`：用线性插值填充
- `NEXT`：用后一个非NULL值填充

**使用示例：**
```sql
-- 用0填充空缺
SELECT AVG(temperature) FROM weather INTERVAL(1h) FILL(VALUE, 0);

-- 用前一个值填充
SELECT AVG(temperature) FROM weather INTERVAL(1h) FILL(PREV);

-- 用线性插值填充
SELECT AVG(temperature) FROM weather INTERVAL(1h) FILL(LINEAR);
```

**可能的错误代码：**
- `TSDB_CODE_PAR_INVALID_FILL_TYPE`：无效的填充类型
- `TSDB_CODE_PAR_MISSING_FILL_VALUE`：指定VALUE填充但未提供值

**本节来源**
- [sql.y](file://source/libs/parser/inc/sql.y#L862-L870)

### SESSION_WINDOW子句
`SESSION_WINDOW`子句用于识别和分析用户会话。

```sql
SESSION_WINDOW(gap_interval)
```

**参数说明：**
- `gap_interval`：会话间隔，当两次事件的时间差超过此间隔时，认为是新的会话

**工作原理：**
- 将连续的、时间间隔小于`gap_interval`的事件归为同一个会话
- 每个会话作为一个独立的窗口进行聚合计算
- 适用于用户行为分析、设备会话分析等场景

**使用示例：**
```sql
-- 分析用户会话，会话间隔为30分钟
SELECT COUNT(*) FROM user_events SESSION_WINDOW(30m);

-- 计算每个会话的持续时间
SELECT SESSION_START(), SESSION_END(), SESSION_DURATION() FROM user_events SESSION_WINDOW(30m);
```

**可能的错误代码：**
- `TSDB_CODE_PAR_INVALID_SESSION_WINDOW`：无效的会话窗口间隔
- `TSDB_CODE_PAR_SESSION_WINDOW_TOO_SMALL`：会话窗口间隔过小

**本节来源**
- [sql.y](file://source/libs/parser/inc/sql.y#L872-L880)

### STATE_WINDOW子句
`STATE_WINDOW`子句用于基于状态变化进行分组。

```sql
STATE_WINDOW(state_column)
```

**参数说明：**
- `state_column`：状态列，当该列的值发生变化时，开始新的窗口

**工作原理：**
- 将连续的、状态值相同的记录归为同一个窗口
- 每个状态持续期作为一个独立的窗口进行聚合计算
- 适用于设备状态监控、服务状态分析等场景

**使用示例：**
```sql
-- 按设备状态分组，计算每种状态的持续时间
SELECT state, SESSION_DURATION() FROM device_status STATE_WINDOW(state);

-- 计算设备在"运行"状态的平均温度
SELECT AVG(temperature) FROM sensor_data STATE_WINDOW(device_status) WHERE device_status = 'running';
```

**可能的错误代码：**
- `TSDB_CODE_PAR_INVALID_STATE_WINDOW`：无效的状态窗口列
- `TSDB_CODE_PAR_COLUMN_NOT_FOUND`：状态列不存在

**本节来源**
- [sql.y](file://source/libs/parser/inc/sql.y#L882-L890)

### SLIDING子句
`SLIDING`子句用于定义滑动窗口的步长。

```sql
SLIDING(slide_interval)
```

**参数说明：**
- `slide_interval`：滑动窗口的步长，即每次计算的时间间隔

**工作原理：**
- 与`INTERVAL`结合使用，定义滑动窗口分析
- 窗口大小由`INTERVAL`决定，滑动步长由`SLIDING`决定
- 支持窗口重叠，提供更细粒度的时序分析

**使用示例：**
```sql
-- 每5分钟计算一次过去15分钟的平均温度
SELECT AVG(temperature) FROM weather INTERVAL(15m) SLIDING(5m);
```

**可能的错误代码：**
- `TSDB_CODE_PAR_INVALID_SLIDING_INTERVAL`：无效的滑动间隔
- `TSDB_CODE_PAR_SLIDING_LARGER_THAN_INTERVAL`：滑动间隔大于窗口间隔

**本节来源**
- [sql.y](file://source/libs/parser/inc/sql.y#L860-L862)

## SHOW语句
`SHOW`语句用于查看系统元数据和状态信息。

```sql
SHOW DNODES;
SHOW DATABASES;
SHOW TABLES;
SHOW STABLES;
SHOW VGROUPS;
SHOW QUERIES;
SHOW VARIABLES;
```

**支持的SHOW命令：**
- `SHOW DNODES`：显示数据节点信息
- `SHOW DATABASES`：显示数据库列表
- `SHOW TABLES`：显示表列表
- `SHOW STABLES`：显示超级表列表
- `SHOW VGROUPS`：显示虚拟组信息
- `SHOW QUERIES`：显示当前查询
- `SHOW VARIABLES`：显示系统变量

**使用示例：**
```sql
SHOW DATABASES;
SHOW TABLES LIKE 'weather%';
SHOW CREATE TABLE weather;
```

**可能的错误代码：**
- `TSDB_CODE_PAR_INVALID_SHOW_CMD`：无效的SHOW命令
- `TSDB_CODE_PAR_INVALID_LIKE_PATTERN`：无效的LIKE模式

**本节来源**
- [sql.y](file://source/libs/parser/inc/sql.y#L552-L600)

## 错误代码参考
本节列出TDengine SQL解析和执行过程中可能遇到的主要错误代码。

| 错误代码 | 错误名称 | 说明 |
|---------|--------|------|
| TSDB_CODE_SUCCESS | 成功 | 操作成功完成 |
| TSDB_CODE_PAR_SYNTAX_ERROR | 语法错误 | SQL语句语法错误 |
| TSDB_CODE_PAR_INCOMPLETE_SQL | 不完整的SQL | SQL语句不完整 |
| TSDB_CODE_PAR_DB_NOT_SPECIFIED | 未指定数据库 | 未指定要操作的数据库 |
| TSDB_CODE_PAR_INVALID_COLUMN_DEF | 无效的列定义 | 列定义格式错误 |
| TSDB_CODE_PAR_INVALID_TAGS_DEF | 无效的标签定义 | 标签定义格式错误 |
| TSDB_CODE_PAR_INVALID_INSERT_VALUE | 无效的插入值 | 插入的值格式或类型错误 |
| TSDB_CODE_PAR_WRONG_VALUE_TYPE | 值类型错误 | 值的类型与列定义不匹配 |
| TSDB_CODE_TDB_DB_NOT_EXIST | 数据库不存在 | 指定的数据库不存在 |
| TSDB_CODE_TDB_TABLE_NOT_EXIST | 表不存在 | 指定的表不存在 |
| TSDB_CODE_TDB_INVALID_TIMESTAMP | 无效的时间戳 | 时间戳格式错误或超出范围 |
| TSDB_CODE_PAR_USER_ALREADY_EXISTS | 用户已存在 | 创建用户时用户名已存在 |
| TSDB_CODE_PAR_USER_NOT_EXIST | 用户不存在 | 指定的用户不存在 |
| TSDB_CODE_PAR_INVALID_PRIVILEGE | 无效的权限 | 指定的权限类型无效 |
| TSDB_CODE_PAR_INVALID_INTERVAL | 无效的时间间隔 | INTERVAL子句的时间间隔格式错误 |
| TSDB_CODE_PAR_INVALID_FILL_TYPE | 无效的填充类型 | FILL子句的填充类型无效 |
| TSDB_CODE_PAR_INVALID_SESSION_WINDOW | 无效的会话窗口 | SESSION_WINDOW子句的间隔无效 |

**本节来源**
- [sql.y](file://source/libs/parser/inc/sql.y#L20-L50)
- [parAst.h](file://source/libs/parser/inc/parAst.h)

## 附录：测试用例覆盖
本节说明SQL语法参考手册与测试用例的对应关系，确保文档覆盖所有边缘情况。

**测试用例目录结构：**
- `test/cases/02-Databases`：数据库相关测试
- `test/cases/03-Tables`：表相关测试
- `test/cases/10-DataIngestion`：数据插入测试
- `test/cases/20-DataQuerying`：数据查询测试
- `test/cases/24-TimeSeriesExtensions`：时序扩展语法测试

**覆盖验证方法：**
1. 对每个SQL语法结构，检查是否有对应的测试用例
2. 测试用例应覆盖正常情况和各种边缘情况
3. 错误处理应有相应的负向测试用例
4. 性能相关功能应有性能测试用例

**验证示例：**
- `INTERVAL`子句：`test/cases/24-TimeSeriesExtensions/01-Interval`
- `FILL`子句：`test/cases/24-TimeSeriesExtensions/02-Fill`
- `SESSION_WINDOW`子句：`test/cases/24-TimeSeriesExtensions/03-SessionWindow`

**本节来源**
- [test/cases/24-TimeSeriesExtensions](file://test/cases/24-TimeSeriesExtensions)
- [test/cases/20-DataQuerying](file://test/cases/20-DataQuerying)