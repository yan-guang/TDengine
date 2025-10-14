# 过滤条件 (WHERE)

<cite>
**Referenced Files in This Document**   
- [sql.y](file://source/libs/parser/inc/sql.y)
- [test_filter_column.py](file://test/cases/20-DataQuerying/02-Filter/test_filter_column.py)
- [test_filter_operator.py](file://test/cases/20-DataQuerying/02-Filter/test_filter_operator.py)
- [test_filter_tag.py](file://test/cases/20-DataQuerying/02-Filter/test_filter_tag.py)
- [test_filter_timestamp.py](file://test/cases/20-DataQuerying/02-Filter/test_filter_timestamp.py)
</cite>

## 目录
1. [简介](#简介)
2. [WHERE子句语法](#where子句语法)
3. [比较运算符](#比较运算符)
4. [逻辑运算符](#逻辑运算符)
5. [时间函数](#时间函数)
6. [数据类型支持](#数据类型支持)
7. [特殊语法](#特殊语法)
8. [实际示例](#实际示例)
9. [测试用例验证](#测试用例验证)

## 简介
`WHERE`子句是TDengine中用于过滤查询结果的核心组件。它允许用户根据特定条件筛选数据，支持多种运算符和函数。本文档基于`sql.y`中的`where_clause`语法规则，系统性地文档化`WHERE`子句的使用方法，重点介绍比较运算符、逻辑运算符、时间函数以及支持的数据类型和特殊语法。

**Section sources**
- [sql.y](file://source/libs/parser/inc/sql.y#L0-L2095)

## WHERE子句语法
`WHERE`子句的语法规则定义在`sql.y`文件中，支持复杂的条件表达式。基本语法结构如下：

```
WHERE <condition>
```

其中`<condition>`可以是简单的比较表达式，也可以是复杂的逻辑组合。

**Section sources**
- [sql.y](file://source/libs/parser/inc/sql.y#L1653-L1738)

## 比较运算符
TDengine支持以下比较运算符：

- `=`：等于
- `<`：小于
- `>`：大于
- `!=`：不等于
- `LIKE`：模式匹配
- `IN`：包含在集合中
- `BETWEEN`：在某个范围内

这些运算符可用于数值、字符串和时间戳等数据类型的比较。

**Section sources**
- [sql.y](file://source/libs/parser/inc/sql.y#L1653-L1665)
- [test_filter_operator.py](file://test/cases/20-DataQuerying/02-Filter/test_filter_operator.py#L0-L3367)

## 逻辑运算符
逻辑运算符用于组合多个条件：

- `AND`：逻辑与
- `OR`：逻辑或
- `NOT`：逻辑非

这些运算符遵循标准的优先级规则，可以使用括号来改变运算顺序。

**Section sources**
- [sql.y](file://source/libs/parser/inc/sql.y#L1724-L1738)
- [test_filter_operator.py](file://test/cases/20-DataQuerying/02-Filter/test_filter_operator.py#L0-L3367)

## 时间函数
TDengine提供了多种时间函数，用于处理时间相关的过滤条件：

- `NOW`：当前时间
- `TODAY`：今天的时间

这些函数可以与其他时间单位结合使用，如`1h`、`1d`等，以实现灵活的时间过滤。

**Section sources**
- [connectOptionsTest.cpp](file://test/cases/uncatalog/system-test/2-query/test_Now.py#L0-L43)
- [test_fun_sca_today.py](file://test/cases/22-Functions/01-Scalar/test_fun_sca_today.py#L0-L37)

## 数据类型支持
`WHERE`子句支持多种数据类型，包括：

- 时间戳
- 字符串
- 数值（整数、浮点数等）

不同数据类型可以使用相应的比较运算符进行过滤。

**Section sources**
- [test_filter_column.py](file://test/cases/20-DataQuerying/02-Filter/test_filter_column.py#L0-L1028)
- [test_filter_operator.py](file://test/cases/20-DataQuerying/02-Filter/test_filter_operator.py#L0-L3367)

## 特殊语法
除了基本的比较和逻辑运算符，`WHERE`子句还支持一些特殊语法，如正则表达式匹配等。这些特殊语法提供了更强大的过滤能力。

**Section sources**
- [test_filter_operator.py](file://test/cases/20-DataQuerying/02-Filter/test_filter_operator.py#L0-L3367)

## 实际示例
以下是一些实际的`WHERE`子句使用示例：

```sql
SELECT * FROM meters WHERE current > 10 AND ts > NOW - 1h
```

这个查询语句筛选出`current`值大于10且时间戳在过去1小时内的所有记录。

**Section sources**
- [test_filter_timestamp.py](file://test/cases/20-DataQuerying/02-Filter/test_filter_timestamp.py#L0-L1518)

## 测试用例验证
为了确保`WHERE`子句的正确性和可靠性，TDengine提供了多个测试用例。这些测试用例涵盖了各种使用场景，包括基本的比较操作、复杂的逻辑组合以及时间函数的使用。

**Section sources**
- [test_filter_column.py](file://test/cases/20-DataQuerying/02-Filter/test_filter_column.py#L0-L1028)
- [test_filter_operator.py](file://test/cases/20-DataQuerying/02-Filter/test_filter_operator.py#L0-L3367)
- [test_filter_tag.py](file://test/cases/20-DataQuerying/02-Filter/test_filter_tag.py#L0-L1053)
- [test_filter_timestamp.py](file://test/cases/20-DataQuerying/02-Filter/test_filter_timestamp.py#L0-L1518)