# 选择列表 (SELECT)

<cite>
**本文档中引用的文件**   
- [sql.y](file://source/libs/parser/inc/sql.y)
- [test_selectlist_basic.py](file://test/cases/20-DataQuerying/01-SelectList/test_selectlist_basic.py)
</cite>

## 目录
1. [简介](#简介)
2. [选择列表语法](#选择列表语法)
3. [选择所有列](#选择所有列)
4. [选择特定列](#选择特定列)
5. [使用表达式和函数](#使用表达式和函数)
6. [别名 (AS)](#别名-as)
7. [标量函数](#标量函数)
8. [聚合函数](#聚合函数)
9. [实际代码示例](#实际代码示例)
10. [测试用例验证](#测试用例验证)

## 简介
选择列表（SELECT子句）是SQL查询的核心部分，用于指定从数据库中检索哪些数据。本文档基于`sql.y`中的`select_clause`语法规则，详细解释了`SELECT`子句的构成，包括选择所有列（`*`）、选择特定列、使用表达式和函数（如`COUNT`, `SUM`）以及别名（`AS`）。同时，文档说明了在选择列表中支持的标量函数和聚合函数的使用方法，并通过实际代码示例和测试用例来验证语法的正确性。

## 选择列表语法
根据`sql.y`文件中的定义，`select_clause`的语法规则如下：
```sql
select_clause(A) ::= select_type(B) select_expr_list(C). { A = setSelectStmtInfo(pCxt, B, C, NULL, NULL); }
select_clause(A) ::= select_type(B) select_expr_list(C) from_clause(D). { A = setSelectStmtInfo(pCxt, B, C, D, NULL); }
select_clause(A) ::= select_type(B) select_expr_list(C) from_clause(D) where_clause(E). { A = setSelectStmtInfo(pCxt, B, C, D, E); }
```
其中，`select_type`可以是`SELECT`或`SELECT DISTINCT`，`select_expr_list`是选择表达式的列表，`from_clause`指定数据源，`where_clause`用于过滤数据。

**Section sources**
- [sql.y](file://source/libs/parser/inc/sql.y#L2067-L2070)

## 选择所有列
使用`*`可以表示选择所有列。例如，`SELECT * FROM meters`将返回`meters`表中的所有列。

```sql
SELECT * FROM meters;
```

**Section sources**
- [test_selectlist_basic.py](file://test/cases/20-DataQuerying/01-SelectList/test_selectlist_basic.py#L124-L146)

## 选择特定列
可以选择表中的特定列，只需在`SELECT`后列出列名，用逗号分隔。例如，`SELECT ts, current, voltage FROM meters`将返回`meters`表中的`ts`、`current`和`voltage`列。

```sql
SELECT ts, current, voltage FROM meters;
```

**Section sources**
- [test_selectlist_basic.py](file://test/cases/20-DataQuerying/01-SelectList/test_selectlist_basic.py#L124-L146)

## 使用表达式和函数
选择列表支持使用表达式和函数。例如，可以对列进行算术运算或调用内置函数。

```sql
SELECT ts, current + 1, voltage * 2 FROM meters;
```

**Section sources**
- [test_selectlist_basic.py](file://test/cases/20-DataQuerying/01-SelectList/test_selectlist_basic.py#L124-L146)

## 别名 (AS)
可以使用`AS`关键字为列或表达式指定别名，以便在结果集中使用更易读的名称。

```sql
SELECT ts, current AS current_value, voltage AS voltage_value FROM meters;
```

**Section sources**
- [test_selectlist_basic.py](file://test/cases/20-DataQuerying/01-SelectList/test_selectlist_basic.py#L124-L146)

## 标量函数
标量函数对单个值进行操作并返回单个值。常见的标量函数包括`COUNT`、`SUM`、`AVG`等。

```sql
SELECT COUNT(*) FROM meters;
SELECT SUM(current) FROM meters;
SELECT AVG(voltage) FROM meters;
```

**Section sources**
- [test_selectlist_basic.py](file://test/cases/20-DataQuerying/01-SelectList/test_selectlist_basic.py#L124-L146)

## 聚合函数
聚合函数对一组值进行操作并返回单个值。常见的聚合函数包括`COUNT`、`SUM`、`AVG`、`MIN`和`MAX`。

```sql
SELECT COUNT(*) FROM meters;
SELECT SUM(current) FROM meters;
SELECT AVG(voltage) FROM meters;
SELECT MIN(ts) FROM meters;
SELECT MAX(ts) FROM meters;
```

**Section sources**
- [test_selectlist_basic.py](file://test/cases/20-DataQuerying/01-SelectList/test_selectlist_basic.py#L124-L146)

## 实际代码示例
以下是一个实际的SQL查询示例，展示了如何结合使用选择列表的各种特性：

```sql
SELECT ts, current, voltage 
FROM meters 
WHERE location='Beijing';
```

此查询从`meters`表中选择`ts`、`current`和`voltage`列，且仅返回`location`为`Beijing`的记录。

**Section sources**
- [test_selectlist_basic.py](file://test/cases/20-DataQuerying/01-SelectList/test_selectlist_basic.py#L124-L146)

## 测试用例验证
为了验证`SELECT`子句的正确性，我们参考了`test/cases/20-DataQuerying/01-SelectList`中的测试用例。这些测试用例涵盖了选择所有列、选择特定列、使用表达式和函数以及别名的使用场景。

```python
def test_selectlist(self):
    """SelectList

    1. Projection queries
    2. Aggregation queries
    3. Scalar functions
    4. Combining GROUP BY, ORDER BY, Limit, and WHERE clauses

    Catalog:
        - Query:SelectList

    Since: v3.0.0.0

    Labels: common,ci

    Jira: None

    History:
        - 2025-8-20 Simon Guan Migrated from tsim/vector/single.sim
        - 2025-8-20 Simon Guan Migrated from tsim/vector/multi.sim
        - 2025-8-20 Simon Guan Migrated from tsim/parser/selectResNum.sim
        - 2025-8-20 Simon Guan Migrated from tsim/parser/constCol.sim
        - 2025-8-20 Simon Guan Migrated from tsim/query/const.sim
        - 2025-8-20 Simon Guan Migrated from tsim/query/read.sim
        - 2025-8-20 Simon Guan Migrated from tsim/query/complex_select.sim

    """
    self.VectorSingle()
    tdStream.dropAllStreamsAndDbs()
    self.VectorMulti()
    tdStream.dropAllStreamsAndDbs()
    self.SelectResNum()
    tdStream.dropAllStreamsAndDbs()
    self.ConstCol()
    tdStream.dropAllStreamsAndDbs()
    self.QueryRead()
    tdStream.dropAllStreamsAndDbs()
    self.ComplexSelect()
    tdStream.dropAllStreamsAndDbs()
```

**Section sources**
- [test_selectlist_basic.py](file://test/cases/20-DataQuerying/01-SelectList/test_selectlist_basic.py#L1-L123)