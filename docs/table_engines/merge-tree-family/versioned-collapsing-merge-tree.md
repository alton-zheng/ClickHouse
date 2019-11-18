## VersionedCollapsingMergeTree

- 允许快速写入不断变化的对象状态。

- 删除后台中的旧对象状态。这大大减少了存储的容量。

- 该引擎继承了 `MergeTree` 并将行 `Collapsing` 的逻辑添加到合并 `data part` 的算法中。
  - `VersionedCollapsingMergeTree` 的作用与 `CollapsingMergeTree` 相同，但是它使用了一种不同的折叠算法，这种算法允许以多个线程的任意顺序插入数据。
  - 特别是，`Version` 列有助于正确地折叠行，即使它们是以错误的顺序插入的。
  - 相反，`CollapsingMergeTree` 只允许严格的连续插入。
  
### Creating a Table
```sql
CREATE TABLE [IF NOT EXISTS] [db.]table_name [ON CLUSTER cluster]
(
    name1 [type1] [DEFAULT|MATERIALIZED|ALIAS expr1],
    name2 [type2] [DEFAULT|MATERIALIZED|ALIAS expr2],
    ...
) ENGINE = VersionedCollapsingMergeTree(sign, version)
[PARTITION BY expr]
[ORDER BY expr]
[SAMPLE BY expr]
[SETTINGS name=value, ...]
```

#### 参数
```log
VersionedCollapsingMergeTree(sign, version)
```

- `sign`: 具有行类型的列名称:
  - 1是 `State` 行
  - -1是 `Cancel` 行。
  - 列数据类型是 `Int8` 

- `version`: 具有对象状态版本的列名称。
  - 列数据类型是 `UInt*`

---

### Collapsing

#### **data** 
考虑需要为某个对象保存不断变化的数据的情景。似乎为一个对象保存一行记录并在其发生任何变化时更新记录是合乎逻辑的，但是更新操作对 `DBMS` 来说是昂贵且缓慢的，因为它需要重写存储中的数据。如果你需要快速的写入数据，则更新操作是不可接受的，但是你可以按下面的描述顺序地更新一个对象的变化。

- 在写入行的时候使用特定的列 `Sign`。
  - 如果 `Sign = 1` 则表示这一行是对象的状态，称之为 `State` 行。
  - 如果 `Sign = -1` 则表示是对具有相同属性的状态行的取消，称之为 `Cancel` 行。
  - 还可以使用 `Version` 列，它应该用单独的数字标识对象的每个状态。

例如，我们希望计算用户在某个站点上访问了多少页面以及他们在该站点上停留了多长时间。在某个时间点，我们用用户活动的状态写下面一行:

```log
┌──────────────UserID─┬─PageViews─┬─Duration─┬─Sign─┬─Version─┐
│ 4324182021466249494 │         5 │      146 │    1 │       1 |
└─────────────────────┴───────────┴──────────┴──────┴─────────┘
```

稍后，我们将注册用户活动的更改，并将其写入以下两行。

```log
┌──────────────UserID─┬─PageViews─┬─Duration─┬─Sign─┬─Version─┐
│ 4324182021466249494 │         5 │      146 │   -1 │       1 |
│ 4324182021466249494 │         6 │      185 │    1 │       2 |
└─────────────────────┴───────────┴──────────┴──────┴─────────┘
```


第一行取消了对象(用户)以前的状态。它应该复制除 `Sign` 之外的所有已取消状态的字段。

第二行包含当前状态。

因为我们只需要用户活动的最后一个状态，即行

```log
┌──────────────UserID─┬─PageViews─┬─Duration─┬─Sign─┬─Version─┐
│ 4324182021466249494 │         5 │      146 │    1 │       1 |
│ 4324182021466249494 │         5 │      146 │   -1 │       1 |
└─────────────────────┴───────────┴──────────┴──────┴─────────┘
```

可以删除，折叠对象的无效(旧)状态。`VersionedCollapsingMergeTree` 在合并数据部分时执行此操作。

要了解为什么每个更改需要两行，请参见算法。

**Note:** 

- 写入的程序应该记住对象的状态从而可以取消它。 
  - 字符串应该是 `State` 字符串的复制，除了相反的 `Sign`。
  - 它增加了存储的初始数据的大小，但使得写入数据更快速。

- 由于写入的负载，列中长的增长阵列会降低引擎的效率。数据越简单，效率越高。

- `SELECT` 的结果很大程度取决于对象变更历史的一致性。在准备插入数据时要准确。在不一致的数据中会得到不可预料的结果，例如，像会话深度这种非负指标的负值。


#### Algorithm

- 合并 `data part` 时，它会删除每一对具有相同主键和版本以及不同符号的行。行数的顺序无关紧要。

- 插入数据时，它按主键排序。如果 `Version` 列不在主键中，ClickHouse将它隐式地添加到主键作为最后一个字段，并使用它进行排序。

#### Selecting data
不保证所有具有相同主键的行都在相同的结果 `data part` 中，甚至在相同的物理服务器上。`ClickHouse` 用多线程来处理 `SELECT` 请求，所以它不能预测结果中行的顺序。如果要从 `CollapsingMergeTree` 表中获取完全 `CollapsingMergeTree` 后的数据，则需要聚合。

- 要完成折叠，请使用 `GROUP BY` 从句和用于处理 `Sign` 的聚合函数编写请求。
  - 例如，要计算数量，使用 `sum(Sign)` 而不是 `count()`。
  -  要计算某物的总和，使用 `sum(Sign * x)` 而不是 `sum(x)`
  - 并添加 `HAVING sum(Sign) > 0` 从句。

- 聚合体 `count`,`sum` 和 `avg` 可以用下面方式计算。
  - 如果一个对象至少有一个未被折叠的状态，则可以计算 `uniq` 聚合。
  - `min` 和 `max` 聚合无法计算，因为 `VersionedCollapsingMergeTree` 不会保存折叠状态的值的历史记录。

如果你需要在不进行聚合的情况下获取数据（例如，要检查是否存在最新值与特定条件匹配的行），你可以在 `FROM` 从句中使用 `FINAL` 修饰符。这种方法显然是更低效的。

### Exanoke if Use
示例数据:

```log
┌──────────────UserID─┬─PageViews─┬─Duration─┬─Sign─┬─Version─┐
│ 4324182021466249494 │         5 │      146 │    1 │       1 |
│ 4324182021466249494 │         5 │      146 │   -1 │       1 |
│ 4324182021466249494 │         6 │      185 │    1 │       2 |
└─────────────────────┴───────────┴──────────┴──────┴─────────┘
```

创建表:

```sql
CREATE TABLE UAct
(
    UserID UInt64,
    PageViews UInt8,
    Duration UInt8,
    Sign Int8,
    Version UInt8
)
ENGINE = VersionedCollapsingMergeTree(Sign, Version)
ORDER BY UserID
```

插入数据:

```sql
INSERT INTO UAct VALUES (4324182021466249494, 5, 146, 1, 1)
INSERT INTO UAct VALUES (4324182021466249494, 5, 146, -1, 1),(4324182021466249494, 6, 185, 1, 2)
```

我们使用两个 `INSERT` 查询来创建两个不同的 `data part`。如果我们使用单个查询插入数据，ClickHouse将创建一个`data part`，并且永远不会执行任何合并。

得到的数据:

```log
SELECT * FROM UAct
┌──────────────UserID─┬─PageViews─┬─Duration─┬─Sign─┬─Version─┐
│ 4324182021466249494 │         5 │      146 │    1 │       1 │
└─────────────────────┴───────────┴──────────┴──────┴─────────┘
┌──────────────UserID─┬─PageViews─┬─Duration─┬─Sign─┬─Version─┐
│ 4324182021466249494 │         5 │      146 │   -1 │       1 │
│ 4324182021466249494 │         6 │      185 │    1 │       2 │
└─────────────────────┴───────────┴──────────┴──────┴─────────┘
```

我们看到了什么，哪里有折叠？

通过两个 `INSERT` 请求，我们创建了两个数据`part`。`SELECT` 请求在两个线程中被执行，我们得到了随机顺序的行。没有发生折叠是因为还没有合并数据片段。`ClickHouse` 在一个我们无法预料的未知时刻合并数据片段。

这就是为什么我们需要聚集:

```log
SELECT
    UserID,
    sum(PageViews * Sign) AS PageViews,
    sum(Duration * Sign) AS Duration,
    Version
FROM UAct
GROUP BY UserID, Version
HAVING sum(Sign) > 0

┌──────────────UserID─┬─PageViews─┬─Duration─┬─Version─┐
│ 4324182021466249494 │         6 │      185 │       2 │
└─────────────────────┴───────────┴──────────┴─────────┘
```

如果不需要聚合而希望强制折叠，可以使用FROM子句的FINAL修饰符。

```log
SELECT * FROM UAct FINAL

┌──────────────UserID─┬─PageViews─┬─Duration─┬─Sign─┬─Version─┐
│ 4324182021466249494 │         6 │      185 │    1 │       2 │
└─────────────────────┴───────────┴──────────┴──────┴─────────┘
```

这种查询数据的方法是非常低效的。不要在大表中使用它。