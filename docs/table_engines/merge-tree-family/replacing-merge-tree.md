## ReplacingMergeTree

该引擎和 `MergeTree` 的不同之处在于它会删除具有相同 `primary key` 的重复项。

数据的去重只会在合并的过程中出现。合并会在未知的时间在后台进行，因此你无法预先作出计划。有一些数据可能仍未被处理。尽管你可以调用  `OPTIMIZE`  语句发起计划外的合并，但请不要指望使用它，因为  `OPTIMIZE`  语句会引发对大量数据的读和写。

因此，`ReplacingMergeTree`  适用于在后台清除重复的数据以节省空间，但是它不保证没有重复的数据出现。

### Creating a Table

```sql
CREATE TABLE [IF NOT EXISTS] [db.]table_name [ON CLUSTER cluster]
(
    name1 [type1] [DEFAULT|MATERIALIZED|ALIAS expr1],
    name2 [type2] [DEFAULT|MATERIALIZED|ALIAS expr2],
    ...
) ENGINE = ReplacingMergeTree([ver])
[PARTITION BY expr]
[ORDER BY expr]
[SAMPLE BY expr]
[SETTINGS name=value, ...]
```

**ReplacingMergeTree Parameters**

- `ver` — 版本列。类型为  `UInt*`, `Date`  或  `DateTime`。可选参数。
  - 合并的时候，`ReplacingMergeTree`  从所有具有相同 `primary key` 的行中选择一行留下：
    - 如果  `ver`  列未指定，选择最后一条。 
    - 如果  `ver`  列已指定，选择  `ver`  值最大的版本。

**Query clauses**

创建  `ReplacingMergeTree`  表时，需要与创建  `MergeTree`  表时相同的 `clauses`
