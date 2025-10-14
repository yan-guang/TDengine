# 分组 (GROUP BY)

<cite>
**本文档中引用的文件**   
- [sql.y](file://source/libs/parser/inc/sql.y)
- [test_groupby_basic.py](file://test/cases/20-DataQuerying/03-GroupBy/test_groupby_basic.py)
</cite>

## 目录
1. [简介](#简介)
2. [语法结构](#语法结构)
3. [按标签分组](#按标签分组)
4. [按时间窗口分组](#按时间窗口分组)
5. [聚合函数使用](#聚合函数使用)
6. [实际应用示例](#实际应用示例)
7. [测试验证](#测试验证)

## 简介
`GROUP BY` 子句是 TDengine 数据库中用于对查询结果进行分组的核心功能。通过分组，可以将具有相同特征的数据聚合在一起，并对每个分组应用聚合函数来计算统计结果。该功能广泛应用于数据分析场景，如按地理位置、设备类型或时间区间对数据进行汇总分析。

**Section sources**
- [sql.y](file://source/libs/parser/inc/sql.y#L1854-L1992)

## 语法结构
`GROUP BY` 子句的语法规则定义在 `sql.y` 文件中，其基本结构如下：

```sql
group_by_clause_opt ::= GROUP BY group_by_list
group_by_list ::= expr_or_subquery | group_by_list ',' expr_or_subquery
```

该语法表明，`GROUP BY` 子句可以包含一个或多个表达式，表达式之间用逗号分隔。这些表达式可以是列名、标签（TAGS）或子查询，用于指定数据分组的依据。

**Section sources**
- [sql.y](file://source/libs/parser/inc/sql.y#L1986-L1992)

## 按标签分组
在 TDengine 中，可以使用 `TAGS` 关键字对数据进行分组。标签是时序数据的重要属性，通常用于标识设备、位置或其他元数据。例如，以下查询按 `location` 标签对 `meters` 表中的数据进行分组：

```sql
SELECT AVG(current), MAX(voltage) FROM meters GROUP BY location
```

此查询将计算每个不同 `location` 值对应的所有记录的电流平均值和电压最大值。

**Section sources**
- [test_groupby_basic.py](file://test/cases/20-DataQuerying/03-GroupBy/test_groupby_basic.py#L100-L120)

## 按时间窗口分组
TDengine 支持基于时间窗口的分组查询，允许用户按固定的时间间隔（如每分钟、每小时）对数据进行聚合。时间窗口通过 `INTERVAL` 函数指定，例如 `INTERVAL(1m)` 表示每分钟一个时间窗口。

```sql
SELECT COUNT(*), AVG(value) FROM metrics GROUP BY INTERVAL(1m)
```

该查询将数据按每分钟的时间窗口进行分组，并计算每个窗口内的记录数和值的平均值。时间窗口分组特别适用于监控和趋势分析场景。

**Section sources**
- [sql.y](file://source/libs/parser/inc/sql.y#L908-L910)
- [test_groupby_basic.py](file://test/cases/20-DataQuerying/03-GroupBy/test_groupby_basic.py#L150-L170)

## 聚合函数使用
分组查询通常与聚合函数结合使用，以计算每个分组的统计信息。常用的聚合函数包括：
- `AVG()`：计算平均值
- `MAX()`：获取最大值
- `MIN()`：获取最小值
- `COUNT()`：统计记录数
- `SUM()`：求和

在 `GROUP BY` 查询中，SELECT 子句中的非聚合列必须出现在 GROUP BY 子句中，否则会导致语法错误。聚合函数会对每个分组内的数据进行计算，并返回单个结果值。

**Section sources**
- [test_groupby_basic.py](file://test/cases/20-DataQuerying/03-GroupBy/test_groupby_basic.py#L200-L250)

## 实际应用示例
以下是一个完整的分组查询示例，展示了如何结合标签分组和聚合函数：

```sql
SELECT 
    location,
    AVG(temperature) AS avg_temp,
    MAX(humidity) AS max_humid,
    COUNT(*) AS record_count
FROM sensors 
WHERE ts >= '2023-01-01' 
GROUP BY location 
ORDER BY avg_temp DESC;
```

此查询从 `sensors` 表中选择数据，按 `location` 标签分组，计算每个位置的平均温度、最高湿度和记录数量，并按平均温度降序排列结果。该查询可用于分析不同位置的环境监测数据。

**Section sources**
- [test_groupby_basic.py](file://test/cases/20-DataQuerying/03-GroupBy/test_groupby_basic.py#L300-L350)

## 测试验证
`test/cases/20-DataQuerying/03-GroupBy/test_groupby_basic.py` 文件中的测试用例验证了分组查询的正确性。测试涵盖了多种场景，包括：
- 按数据列和标签列分组
- 结合 `ORDER BY` 和 `LIMIT` 子句
- 使用各种聚合函数
- 不同时间窗口的分组查询

这些测试确保了 `GROUP BY` 功能的稳定性和正确性，为用户提供了可靠的分组查询能力。

**Section sources**
- [test_groupby_basic.py](file://test/cases/20-DataQuerying/03-GroupBy/test_groupby_basic.py#L10-L50)