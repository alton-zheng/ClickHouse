## Merge

- `Merge`  引擎: 
  - 不要跟 `MergeTree` 引擎混淆
  - 不存储数据, 可读多张表
  - 并行读
  - 不能写
  - 用表索引（表有索引的话）

- 参数：一个数据库名和一个用于匹配表名的正则表达式。
  - 数据库名
    - 直接指定具体名称，如： `hits`
    - 返回字符串的常用表达式， 如：`currentDatabase()`
  
  - 匹配表名的正则表达式
    - [re2](https://github.com/google/re2) (支持 PCRE 一个子集的功能)，大小写敏感。
    - 了解关于正则表达式中转义字符的说明可参看 "match" 部分。
    - 例如： '`^WatchLog`'

示例：

```
Merge(hits, '^WatchLog')
```

---

### 读数据
- 当选择需要读的表时： 
  - `Merge` 表本身会被排除，即使它匹配上了该正则。
  - 这样设计为了避免循环。 
  - 当然，是能够创建两个相互无限递归读取对方数据的  `Merge`  表的，但这并没有什么意义。

---

### 使用场景
`Merge` 引擎的一个典型应用是可以像使用一张表一样使用大量的 `TinyLog` 表。

示例 2 ：

我们假定你有一个旧表（ `WatchLog_old` ），你想改变数据分区了，但又不想把旧数据转移到新表（ `WatchLog_new` ）里，并且你需要同时能看到这两个表的数据。

```clickhouse
CREATE TABLE WatchLog_old(date Date, UserId Int64, EventType String, Cnt UInt64)
ENGINE=MergeTree(date, (UserId, EventType), 8192);
INSERT INTO WatchLog_old VALUES ('2018-01-01', 1, 'hit', 3);

CREATE TABLE WatchLog_new(date Date, UserId Int64, EventType String, Cnt UInt64)
ENGINE=MergeTree PARTITION BY date ORDER BY (UserId, EventType) SETTINGS index_granularity=8192;
INSERT INTO WatchLog_new VALUES ('2018-01-02', 2, 'hit', 3);

CREATE TABLE WatchLog as WatchLog_old ENGINE=Merge(currentDatabase(), '^WatchLog');

SELECT *
FROM WatchLog
```

```log
┌───────date─┬─UserId─┬─EventType─┬─Cnt─┐
│ 2018-01-01 │      1 │ hit       │   3 │
└────────────┴────────┴───────────┴─────┘
┌───────date─┬─UserId─┬─EventType─┬─Cnt─┐
│ 2018-01-02 │      2 │ hit       │   3 │
└────────────┴────────┴───────────┴─────┘
```

---

### `Virtual column`

`Virtual column`是一种由 `table engines` 提供而不是在表定义中的列。

换种说法就是，这些列并没有在  `CREATE TABLE`  中指定，但可以在  `SELECT`  中使用。

下面列出 `Virtual column` 跟普通列的不同点：

- `Virtual column`不在表结构定义里指定。
- 不能用  `INSERT`  向 `Virtual column` 写数据。
- 使用不指定列名的  `INSERT`  语句时，`Virtual column` 要会被忽略掉。
- 使用星号通配符（ `SELECT *` ）时 `Virtual column` 不会包含在里面。
- `Virtual column` 不会出现在  `SHOW CREATE TABLE`  和  `DESC TABLE`  的查询结果里。


- `Merge`  引擎表包括一个  `String`  类型的  `_table`  `Virtual column`:  
  - 如果该表本来已有了一个  `_table`  的列，那这个`Virtual column`会命名为  `_table1` ；
  - 如果  `_table1`  也本就存在了，那这个`Virtual column`会被命名为  `_table2` ，依此类推）该列包含被读数据的表名。
  - 如果  `WHERE/PREWHERE`  从句包含了带 `_table` 的条件（如： `_table='xyz'`），并且没有依赖其他的列（如作为表达式谓词链接的一个子项或作为整个的表达式），`_table` 会充当索引列。
