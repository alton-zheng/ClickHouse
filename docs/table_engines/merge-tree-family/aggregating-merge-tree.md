## AggregatingMergeTree

该引擎继承自  `MergeTree`，并改变了数据 `part` 的合并逻辑。

将相同 `primary key` 的所有行（在一个数据 `part` 内）替换为单个存储一系列聚合函数状态的行。

可以使用  `AggregatingMergeTree`  表来做增量数据统计聚合，包括物化视图的数据聚合。

引擎需使用  `AggregateFunction`(见数据类型)  类型来处理所有列。

如果要按一组规则来合并减少行数，则使用  `AggregatingMergeTree`  是合适的。

---

### Creating a Table

```sql
CREATE TABLE [IF NOT EXISTS] [db.]table_name [ON CLUSTER cluster]
(
    name1 [type1] [DEFAULT|MATERIALIZED|ALIAS expr1],
    name2 [type2] [DEFAULT|MATERIALIZED|ALIAS expr2],
    ...
) ENGINE = AggregatingMergeTree()
[PARTITION BY expr]
[ORDER BY expr]
[SAMPLE BY expr]
[TTL expr]
[SETTINGS name=value, ...]
```

**Query clauses**

创建  `AggregatingMergeTree`  表时，需用跟创建  `MergeTree`  表一样的 `clauses` 。

---

### SELECT 和 INSERT

- 插入数据，需使用带有 `aggregate -State- functions` 的  `INSERT SELECT` 语句。 
  - 从  `AggregatingMergeTree`  表中查询数据时，需使用  `GROUP BY`  子句并且要使用与插入时相同的聚合函数，但后缀要改为  `-Merge` 。

在  `SELECT`  查询的结果中，对于 `ClickHouse` 的所有输出格式  `AggregateFunction`  类型的值都实现了特定的二进制表示法。

如果直接用  `SELECT`  导出这些数据，例如如用  `TabSeparated`  格式，那么这些导出数据也能直接用  `INSERT`  语句加载导入。

### 聚合物化视图的示例

创建一个跟踪  `test.visits`  表的  `AggregatingMergeTree`  物化视图：

```sql
CREATE MATERIALIZED VIEW test.basic
ENGINE = AggregatingMergeTree() PARTITION BY toYYYYMM(StartDate) ORDER BY (CounterID, StartDate)
AS SELECT
    CounterID,
    StartDate,
    sumState(Sign)    AS Visits,
    uniqState(UserID) AS Users
FROM test.visits
GROUP BY CounterID, StartDate;
```

向  `test.visits`  表中插入数据。

```sql
INSERT INTO test.visits ...
```

数据会同时插入到表和视图中，并且视图  `test.basic`  会将里面的数据聚合。

要获取聚合数据，我们需要在  `test.basic`  视图上执行类似  `SELECT ... GROUP BY ...`  这样的查询 ：

```sql
SELECT
    StartDate,
    sumMerge(Visits) AS Visits,
    uniqMerge(Users) AS Users
FROM test.basic
GROUP BY StartDate
ORDER BY StartDate;
```