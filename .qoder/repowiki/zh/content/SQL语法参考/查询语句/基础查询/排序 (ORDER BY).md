# 排序 (ORDER BY)

<cite>
**本文档中引用的文件**  
- [sql.y](file://source/libs/parser/inc/sql.y)
- [test_orderby_double.py](file://test/cases/20-DataQuerying/04-OrderBy/test_orderby_double.py)
- [test_orderby_subquery.py](file://test/cases/20-DataQuerying/04-OrderBy/test_orderby_subquery.py)
</cite>

## 目录
1. [简介](#简介)
2. [语法结构](#语法结构)
3. [排序方向](#排序方向)
4. [多列排序](#多列排序)
5. [性能影响与最佳实践](#性能影响与最佳实践)
6. [测试用例验证](#测试用例验证)
7. [总结](#总结)

## 简介
`ORDER BY` 子句用于对查询结果进行排序，是时间序列数据库中数据检索的重要功能之一。在 TDengine 中，`ORDER BY` 支持按一个或多个列进行升序（ASC）或降序（DESC）排序，尤其适用于时间戳（`ts`）等字段的排序操作。本文档基于 `sql.y` 文件中的语法规则，详细说明 `ORDER BY` 的使用方法，并结合测试用例确保其功能的准确性。

## 语法结构
根据 `sql.y` 文件中的定义，`ORDER BY` 子句的语法规则如下：

```yacc
order_by_clause_opt(A) ::= .                                                      { A = NULL; }
order_by_clause_opt(A) ::= ORDER BY sort_specification_list(B).                   { A = B; }

sort_specification_list(A) ::= sort_specification(B).                             { A = createNodeList(pCxt, B); }
sort_specification_list(A) ::=
  sort_specification_list(B) NK_COMMA sort_specification(C).                      { A = addNodeToList(pCxt, B, C); }

sort_specification(A) ::=
  expr_or_subquery(B) ordering_specification_opt(C) null_ordering_opt(D).         { A = createOrderByExprNode(pCxt, releaseRawExprNode(pCxt, B), C, D); }
```

该语法表明：
- `ORDER BY` 子句是可选的。
- 可以指定一个或多个排序规范（`sort_specification`），用逗号分隔。
- 每个排序规范由表达式（或子查询）、排序方向（ASC/DESC）和空值排序规则组成。

**示例：**
```sql
SELECT ts, current FROM meters ORDER BY ts DESC, current ASC;
```

**排序子句来源**
- [sql.y](file://source/libs/parser/inc/sql.y#L2035-L2049)

## 排序方向
`ORDER BY` 支持两种排序方向：
- **ASC**：升序排序（默认）
- **DESC**：降序排序

在时间序列数据中，通常使用 `ts DESC` 来获取最新的数据记录。

**示例：**
```sql
-- 按时间戳降序排列
SELECT ts, voltage FROM power_data ORDER BY ts DESC;

-- 按电流升序排列
SELECT ts, current FROM meters ORDER BY current ASC;
```

**排序方向来源**
- [sql.y](file://source/libs/parser/inc/sql.y#L2048-L2065)

## 多列排序
`ORDER BY` 支持按多个列进行排序。排序优先级从左到右依次降低。

**示例：**
```sql
-- 先按时间戳降序，再按电流升序
SELECT ts, current FROM meters ORDER BY ts DESC, current ASC;
```

此查询首先按 `ts` 降序排列，若 `ts` 相同，则按 `current` 升序排列。

**多列排序来源**
- [sql.y](file://source/libs/parser/inc/sql.y#L2048-L2065)

## 性能影响与最佳实践
### 性能影响
- **索引利用**：如果排序字段上有索引（如时间戳索引），排序性能会显著提升。
- **内存消耗**：对大量数据进行排序可能消耗较多内存，尤其是在没有索引的情况下。
- **磁盘 I/O**：若数据无法完全放入内存，系统可能需要进行外部排序，增加磁盘 I/O。

### 最佳实践
1. **优先使用时间戳排序**：TDengine 针对时间序列数据优化了时间戳排序。
2. **结合 `LIMIT` 使用**：避免对全表数据排序，可通过 `LIMIT` 限制返回行数。
3. **避免在高基数列上排序**：如字符串或 UUID 类型，排序效率较低。
4. **使用 `SLIMIT` 和 `SOFFSET`**：在超级表查询中，合理使用分页参数。

**性能与最佳实践来源**
- [sql.y](file://source/libs/parser/inc/sql.y#L2035-L2065)

## 测试用例验证
TDengine 提供了多个测试用例来验证 `ORDER BY` 功能的正确性，位于 `test/cases/20-DataQuerying/04-OrderBy` 目录下。

### 测试用例 1：双精度浮点数排序
文件：`test_orderby_double.py`

该测试验证了对 `double` 类型数据进行排序的正确性。测试插入多条浮点数数据，并执行 `ORDER BY flag` 查询，验证返回结果是否按预期排序。

```python
tdSql.query(f"select * from t0 order by flag")
tdSql.checkData(0,1, -0.000045286994000)  # 最小值排在第一位
```

### 测试用例 2：子查询排序
文件：`test_orderby_subquery.py`

该测试验证了在子查询结果上进行排序的功能。

```python
tdSql.query(
    f"select * from (select * from t order by ts desc limit 3 offset 2) order by ts;"
)
```

此查询先对子查询结果按时间戳降序排列并取中间三行，再对外层结果按时间戳升序排列。

**测试用例来源**
- [test_orderby_double.py](file://test/cases/20-DataQuerying/04-OrderBy/test_orderby_double.py)
- [test_orderby_subquery.py](file://test/cases/20-DataQuerying/04-OrderBy/test_orderby_subquery.py)

## 总结
`ORDER BY` 是 TDengine 中用于控制查询结果顺序的关键子句。它支持单列和多列排序，可指定升序或降序。在处理大量时间序列数据时，合理使用 `ORDER BY` 并结合索引和分页参数，可以有效提升查询性能。通过 `test/cases/20-DataQuerying/04-OrderBy` 中的测试用例，可以确保排序功能的正确性和稳定性。