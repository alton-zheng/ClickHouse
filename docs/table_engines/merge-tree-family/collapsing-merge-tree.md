## CollapsingMergeTree

该引擎继承于  `MergeTree`，并在数据块合并算法中添加了折叠行的逻辑。

如果一个排序键(ORDER BY)中的所有字段值都是相同的，那么除了特定的字段 `Sign` 可以有`1`和`-1`的值外，`CollapsingMergeTree` 将异步删除(折叠)行对。没有对的行被保留。下面会介绍 `Collapsing` 逻辑。

因此，该引擎可以显著的降低存储量并提高  `SELECT`  查询效率。

### Creating a Table

```sql
CREATE TABLE [IF NOT EXISTS] [db.]table_name [ON CLUSTER cluster]
(
    name1 [type1] [DEFAULT|MATERIALIZED|ALIAS expr1],
    name2 [type2] [DEFAULT|MATERIALIZED|ALIAS expr2],
    ...
) ENGINE = CollapsingMergeTree(sign)
[PARTITION BY expr]
[ORDER BY expr]
[SAMPLE BY expr]
[SETTINGS name=value, ...]
```

**CollapsingMergeTree 参数**

- `sign` — 类型列的名称： 
  - `1`  是 `State` 行
  - `-1`  是 `Cancel` 行

列数据类型 `Int8`

### Collapsing

#### Data

考虑需要为某个对象保存不断变化的数据的情景。似乎为一个对象保存一行记录并在其发生任何变化时更新记录是合乎逻辑的，但是更新操作对 `DBMS` 来说是昂贵且缓慢的，因为它需要重写存储中的数据。如果你需要快速的写入数据，则更新操作是不可接受的，但是你可以按下面的描述顺序地更新一个对象的变化。

- 在写入行的时候使用特定的列 `Sign`。
  - 如果 `Sign = 1` 则表示这一行是对象的状态，称之为 `State` 行。
  - 如果 `Sign = -1` 则表示是对具有相同属性的状态行的取消，称之为 `Cancel` 行。

例如，我们希望计算用户在某个站点上查看了多少页面，以及他们在那里停留了多长时间。在某个时刻，我们用用户活动的状态写下面一行:

```
┌──────────────UserID─┬─PageViews─┬─Duration─┬─Sign─┐
│ 4324182021466249494 │         5 │      146 │    1 │
└─────────────────────┴───────────┴──────────┴──────┘
```

稍后，我们将注册用户活动的更改，并将以下两行写入。

```
┌──────────────UserID─┬─PageViews─┬─Duration─┬─Sign─┐
│ 4324182021466249494 │         5 │      146 │   -1 │
│ 4324182021466249494 │         6 │      185 │    1 │
└─────────────────────┴───────────┴──────────┴──────┘
```

第一行取消了对象(`user`)以前的状态。它应该复制已取消状态异常 `Sign` 的排序键字段。

第二行包含了当前的状态。

因为我们只需要用户活动的最后状态: 

```
┌──────────────UserID─┬─PageViews─┬─Duration─┬─Sign─┐
│ 4324182021466249494 │         5 │      146 │    1 │
│ 4324182021466249494 │         5 │      146 │   -1 │
└─────────────────────┴───────────┴──────────┴──────┘
```

`CollapsingMergeTree` 在合并数据 `part` 做以上事情

为什么我们每次改变需要 2 行可以阅读 `Algorithm` 部分

#### **这种方法的特殊属性**

- 写入的程序应该记住对象的状态从而可以取消它。 
  - 字符串应该是 `State` 字符串的复制，除了相反的 `Sign`。
  - 它增加了存储的初始数据的大小，但使得写入数据更快速。

- 由于写入的负载，列中长的增长阵列会降低引擎的效率。数据越简单，效率越高。

- `SELECT` 的结果很大程度取决于对象变更历史的一致性。在准备插入数据时要准确。在不一致的数据中会得到不可预料的结果，例如，像会话深度这种非负指标的负值。


- 写入数据的程序应该记住对象的状态，以便能够取消它。`Cancel` 字符串应该包含 `State` 字符串的排序键字段的副本和相反的 `Sign`。它增加了存储的初始大小，但允许快速写入数据。

- 列中的长数组由于写入负载而降低了引擎的效率。数据越简单，效率越高。

- `SELECT` 结果在很大程度上取决于对象更改历史的一致性。
  - 准备 `INSERT` 数据时要准确。
  - 您可以在不一致的数据中获得不可预测的结果，例如，`non-negative` (如会话深度)的负值。


### Algorithm
当合并数据 `part` 时，每组具有相同主键的连续行被减少到不超过两行，一行 Sign = 1（“State”行），另一行 Sign = -1 （“Cancel”行），换句话说，数据项被折叠了。

- 对每个结果的数据 `part` ClickHouse 保存：
  - 第一个 `Cancel` 和最后一个 `State` 行(如果 `State` 和 `Cancel` 行的数量匹配)
  - 最后一个 `State` 行( `State` 行比 `Cancel` 行多一个)
  - 第一个 `Cancel` 行(如果 `Cancel` 行比 `State` 行多一个)
  - 没有行(在其他所有情况下)
    - 合并会继续，但是会把此情况视为逻辑错误并将其记录在服务日志中。
    - 这个错误会在相同的数据被插入超过一次时出现。

因此，折叠不应该改变统计数据的结果。 变化逐渐地被折叠，因此最终几乎每个对象都只剩下了最后的状态。

这个 sign 是必需的，因为合并算法不能保证具有相同排序键的所有行都位于相同的结果数据部分，甚至在相同的物理服务器上，甚至是在同一台物理服务器上。`ClickHouse` 用多线程来处理 `SELECT` 请求，所以它不能预测结果中行的顺序。如果要从 `CollapsingMergeTree` 表中获取完全 `CollapsingMergeTree` 后的数据，则需要聚合。

- 要完成折叠，请使用 `GROUP BY` 从句和用于处理 `Sign` 的聚合函数编写请求。
  - 例如，要计算数量，使用 `sum(Sign)` 而不是 `count()`。
  -  要计算某物的总和，使用 `sum(Sign * x)` 而不是 `sum(x)`
  - 并添加 `HAVING sum(Sign) > 0` 从句。

- 聚合体 `count`,`sum` 和 `avg` 可以用下面方式计算。
  - 如果一个对象至少有一个未被折叠的状态，则可以计算 `uniq` 聚合。
  - `min` 和 `max` 聚合无法计算，因为 `CollaspingMergeTree` 不会保存折叠状态的值的历史记录。

如果你需要在不进行聚合的情况下获取数据（例如，要检查是否存在最新值与特定条件匹配的行），你可以在 `FROM` 从句中使用 `FINAL` 修饰符。这种方法显然是更低效的。

### Example of use

示例数据:

```
┌──────────────UserID─┬─PageViews─┬─Duration─┬─Sign─┐
│ 4324182021466249494 │         5 │      146 │    1 │
│ 4324182021466249494 │         5 │      146 │   -1 │
│ 4324182021466249494 │         6 │      185 │    1 │
└─────────────────────┴───────────┴──────────┴──────┘
```

建表:

```sql
CREATE TABLE UAct
(
    UserID UInt64,
    PageViews UInt8,
    Duration UInt8,
    Sign Int8
)
ENGINE = CollapsingMergeTree(Sign)
ORDER BY UserID
```

插入数据:

```sql
INSERT INTO UAct VALUES (4324182021466249494, 5, 146, 1)

INSERT INTO UAct VALUES (4324182021466249494, 5, 146, -1),(4324182021466249494, 6, 185, 1)
```

我们使用两次 `INSERT` 请求来创建两个不同的 数据 `part`。如果我们使用一个请求插入数据，只会创建一个数据 `part` 且不会执行任何合并操作。

获取数据：

```sql
SELECT * FROM UAct
```

```
┌──────────────UserID─┬─PageViews─┬─Duration─┬─Sign─┐
│ 4324182021466249494 │         5 │      146 │   -1 │
│ 4324182021466249494 │         6 │      185 │    1 │
└─────────────────────┴───────────┴──────────┴──────┘
┌──────────────UserID─┬─PageViews─┬─Duration─┬─Sign─┐
│ 4324182021466249494 │         5 │      146 │    1 │
└─────────────────────┴───────────┴──────────┴──────┘
```

我们看到了什么，哪里有折叠？

通过两个 `INSERT` 请求，我们创建了两个数据`part`。`SELECT` 请求在两个线程中被执行，我们得到了随机顺序的行。没有发生折叠是因为还没有合并数据片段。`ClickHouse` 在一个我们无法预料的未知时刻合并数据片段。

因此我们需要聚合：

```sql
SELECT
    UserID,
    sum(PageViews * Sign) AS PageViews,
    sum(Duration * Sign) AS Duration
FROM UAct
GROUP BY UserID
HAVING sum(Sign) > 0
```

```
┌──────────────UserID─┬─PageViews─┬─Duration─┐
│ 4324182021466249494 │         6 │      185 │
└─────────────────────┴───────────┴──────────┘
```

如果我们不需要聚合并想要强制进行折叠，我们可以在 FROM 从句中使用 FINAL 修饰语。

```
SELECT * FROM UAct FINAL
┌──────────────UserID─┬─PageViews─┬─Duration─┬─Sign─┐
│ 4324182021466249494 │         6 │      185 │    1 │
└─────────────────────┴───────────┴──────────┴──────┘
```

这种查询数据的方法是非常低效的。不要在大表中使用它。

### Example of another approach

示例数据

```
┌──────────────UserID─┬─PageViews─┬─Duration─┬─Sign─┐
│ 4324182021466249494 │         5 │      146 │    1 │
│ 4324182021466249494 │        -5 │     -146 │   -1 │
│ 4324182021466249494 │         6 │      185 │    1 │
└─────────────────────┴───────────┴──────────┴──────┘
```

- 其思想是合并只考虑 `key` 字段。
  - 在 `Cancel` 行中，我们可以指定负的值，在不使用符号列求和的情况下使前一个版本的行相等。
  - 对于这种方法，需要改变数据类型`PageViews`、`Duration`来存储`UInt8 -> Int16`的负值。
  
```sql
CREATE TABLE UAct
(
    UserID UInt64,
    PageViews Int16,
    Duration Int16,
    Sign Int8
)
ENGINE = CollapsingMergeTree(Sign)
ORDER BY UserID
```

#### test
```sql
insert into UAct values(4324182021466249494,  5,  146,  1);
insert into UAct values(4324182021466249494, -5, -146, -1);
insert into UAct values(4324182021466249494,  6,  185,  1);

select * from UAct final; // avoid using final in production (just for a test or small tables)
```

```
┌──────────────UserID─┬─PageViews─┬─Duration─┬─Sign─┐
│ 4324182021466249494 │         6 │      185 │    1 │
└─────────────────────┴───────────┴──────────┴──────┘
```

```
SELECT
    UserID,
    sum(PageViews) AS PageViews,
    sum(Duration) AS Duration
FROM UAct
GROUP BY UserID
```text

┌──────────────UserID─┬─PageViews─┬─Duration─┐
│ 4324182021466249494 │         6 │      185 │
└─────────────────────┴───────────┴──────────┘
```

```sql
select count() FROM UAct
```

```log
┌─count()─┐
│       3 │
└─────────┘
```

```log
optimize table UAct final;

select * FROM UAct

┌──────────────UserID─┬─PageViews─┬─Duration─┬─Sign─┐
│ 4324182021466249494 │         6 │      185 │    1 │
└─────────────────────┴───────────┴──────────┴──────┘
```





