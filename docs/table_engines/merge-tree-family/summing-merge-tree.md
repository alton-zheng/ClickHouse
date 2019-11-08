## SummingMergeTree[¶](https://clickhouse.yandex/docs/zh/single/#summingmergetree "Permanent link")

该引擎继承自  [MergeTree](https://clickhouse.yandex/docs/zh/single/#mergetree/)。区别在于，当合并  `SummingMergeTree`  表的数据`parts`时，ClickHouse 会把所有具有相同 `primary key` 的行合并为一行，该行包含了被合并的行中具有数值数据类型的列的汇总值。如果 `primary key` 的组合方式使得单个键值对应于大量的行，则可以显著的减少存储空间并加快数据查询的速度。

我们推荐将该引擎和  `MergeTree`  一起使用。例如，在准备做报告的时候，将完整的数据存储在  `MergeTree`  表中，并且使用  `SummingMergeTree`  来存储聚合数据。这种方法可以使你避免因为使用不正确的 `primary key` 组合方式而丢失有价值的数据。

### 建表[¶](https://clickhouse.yandex/docs/zh/single/#jian-biao_1 "Permanent link")

CREATE TABLE \[IF NOT EXISTS\] \[db.\]table_name \[ON CLUSTER cluster\]
(
name1 \[type1\] \[DEFAULT|MATERIALIZED|ALIAS expr1\],
name2 \[type2\] \[DEFAULT|MATERIALIZED|ALIAS expr2\],
...
) ENGINE = SummingMergeTree(\[columns\])
\[PARTITION BY expr\]
\[ORDER BY expr\]
\[SAMPLE BY expr\]
\[SETTINGS name=value, ...\]

请求参数的描述，参考  [请求描述](https://clickhouse.yandex/docs/zh/single/#../../query_language/create/)。

**SummingMergeTree 的参数**

- `columns` \- 包含了将要被汇总的列的列名的元组。可选参数。 所选的列必须是数值类型，并且不可位于 `primary key` 中。

  如果没有指定  `columns`，ClickHouse 会把所有不在 `primary key` 中的数值类型的列都进行汇总。

**子句**

创建  `SummingMergeTree`  表时，需要与创建  `MergeTree`  表时相同的[子句](https://clickhouse.yandex/docs/zh/single/#mergetree/)。

已弃用的建表方法

### 用法示例[¶](https://clickhouse.yandex/docs/zh/single/#yong-fa-shi-li_2 "Permanent link")

考虑如下的表：

CREATE TABLE summtt
(
key UInt32,
value UInt32
)
ENGINE = SummingMergeTree()
ORDER BY key

向其中插入数据：

:) INSERT INTO summtt Values(1,1),(1,2),(2,1)

ClickHouse 可能不会完整的汇总所有行（[见下文](https://clickhouse.yandex/docs/zh/single/#data-processing)）,因此我们在查询中使用了聚合函数  `sum`  和  `GROUP BY`  子句。

SELECT key, sum(value) FROM summtt GROUP BY key

┌─key─┬─sum(value)─┐
│ 2 │ 1 │
│ 1 │ 3 │
└─────┴────────────┘

### 数据处理[¶](https://clickhouse.yandex/docs/zh/single/#data-processing "Permanent link")

当数据被插入到表中时，他们将被原样保存。ClickHouse 定期合并插入的数据`parts`，并在这个时候对所有具有相同 `primary key` 的行中的列进行汇总，将这些行替换为包含汇总数据的一行记录。

ClickHouse 会按`parts`合并数据，以至于不同的数据`parts`中会包含具有相同 `primary key` 的行，即单个汇总`parts`将会是不完整的。因此，聚合函数  [sum()](https://clickhouse.yandex/docs/zh/single/#agg_function-sum)  和  `GROUP BY`  子句应该在（`SELECT`）查询语句中被使用，如上文中的例子所述。

#### 汇总的通用规则[¶](https://clickhouse.yandex/docs/zh/single/#hui-zong-de-tong-yong-gui-ze "Permanent link")

列中数值类型的值会被汇总。这些列的集合在参数  `columns`  中被定义。

如果用于汇总的所有列中的值均为 0，则该行会被删除。

如果列不在 `primary key` 中且无法被汇总，则会在现有的值中任选一个。

 `primary key` 所在的列中的值不会被汇总。

#### AggregateFunction 列中的汇总[¶](https://clickhouse.yandex/docs/zh/single/#aggregatefunction-lie-zhong-de-hui-zong "Permanent link")

对于  [AggregateFunction 类型](https://clickhouse.yandex/docs/zh/single/#../../data_types/nested_data_structures/aggregatefunction/)的列，ClickHouse 根据对应函数表现为  [AggregatingMergeTree](https://clickhouse.yandex/docs/zh/single/#aggregatingmergetree/)  引擎的聚合。

#### 嵌套结构[¶](https://clickhouse.yandex/docs/zh/single/#qian-tao-jie-gou "Permanent link")

表中可以具有以特殊方式处理的嵌套数据结构。

如果嵌套表的名称以  `Map`  结尾，并且包含至少两个符合以下条件的列：

- 第一列是数值类型  `(*Int*, Date, DateTime)`，我们称之为  `key`,
- 其他的列是可计算的  `(*Int*, Float32/64)`，我们称之为  `(values...)`,

然后这个嵌套表会被解释为一个  `key => (values...)`  的映射，当合并它们的行时，两个数据集中的元素会被根据  `key`  合并为相应的  `(values...)`  的汇总值。

示例：

\[(1, 100)\] + \[(2, 150)\] -> \[(1, 100), (2, 150)\]
\[(1, 100)\] + \[(1, 150)\] -> \[(1, 250)\]
\[(1, 100)\] + \[(1, 150), (2, 150)\] -> \[(1, 250), (2, 150)\]
\[(1, 100), (2, 150)\] + \[(1, -100)\] -> \[(2, 150)\]

请求数据时，使用  [sumMap(key, value)](https://clickhouse.yandex/docs/zh/single/#../../query_language/agg_functions/reference/)  函数来对  `Map`  进行聚合。

对于嵌套数据结构，你无需在列的元组中指定列以进行汇总。
