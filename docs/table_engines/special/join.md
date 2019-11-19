## Join

准备用于 `JOIN` 操作的数据结构。

---

### Creating a Table
```clickhouse
CREATE TABLE [IF NOT EXISTS] [db.]table_name [ON CLUSTER cluster]
(
    name1 [type1] [DEFAULT|MATERIALIZED|ALIAS expr1] [TTL expr1],
    name2 [type2] [DEFAULT|MATERIALIZED|ALIAS expr2] [TTL expr2],
) ENGINE = Join(join_strictness, join_type, k1[, k2, ...])
```

**Parameters**: 

- `join_strictness`: 连接严格(见 `SELECT` 语法部分知识)。
  - `ALL` : 如果右表有几个匹配的行，ClickHouse将从匹配的行创建一个笛卡尔积。这是SQL中的标准连接行为。 
  - `ANY` : 
    - 如果正确的表有几个匹配的行，那么只有找到的第一个行被连接。
    - 如果正确的表只有一个匹配的行，那么查询任意和所有关键字的结果都是相同的。类似于 `Oracle Exists`
  - `ASOF` : 用于连接具有非精确匹配的序列。
  
- `join_type` : 连接类型。

- `k1[, k2, ...]` : 使用连接操作的USING子句中的 `key` 列。

- 输入 `join_strictness` 和 `join_type` 参数(不带引号)，例如`Join(ANY, LEFT, col1)`。
  - 它们必须匹配表将被用于的连接操作。
  - 如果参数不匹配： 
    - 不会抛出异常
    - 可能返回错误的数据。
    
---

### Table Usage

**栗子**： 

- 创建 `left-side` `Join` 表:

```clickhouse
CREATE TABLE id_val(`id` UInt32, `val` UInt32) ENGINE = TinyLog
```
```clickhouse
INSERT INTO id_val VALUES (1,11)(2,12)(3,13)
```

- 创建 `right-side` `Join` 表:

```clickhouse
CREATE TABLE id_val_join(`id` UInt32, `val` UInt8) ENGINE = Join(ANY, LEFT, id)
```
```clickhouse
INSERT INTO id_val_join VALUES (1,21)(1,22)(3,23)
```

- `Join`: 

```clickhouse
SELECT * FROM id_val ANY LEFT JOIN id_val_join USING (id) SETTINGS join_use_nulls = 1
```

```log
┌─id─┬─val─┬─id_val_join.val─┐
│  1 │  11 │              21 │
│  2 │  12 │            ᴺᵁᴸᴸ │
│  3 │  13 │              23 │
└────┴─────┴─────────────────┘
```

- 作为替代方法，您可以从 `Join` 表中检索数据，并指定联接`key`:

```clickhouse
SELECT joinGet('id_val_join', 'val', toUInt32(1))
```

```log
┌─joinGet('id_val_join', 'val', toUInt32(1))─┐
│                                         21 │
└────────────────────────────────────────────┘
```

---

### Selecting and Inserting data

- `ANY` 严格性，则会忽略 `duplicate key`(重复键) 的数据。
  - 在所有严格性下，添加所有行。（ `Insert` ）

- 不能直接执行SELECT查询。必须使用以下方法之一执行 Selecting 操作:
  - 在JOIN从句中将表放到右边。
  - 调用joinGet函数，该函数允许您以与从字典中提取数据相同的方式从表中提取数据。上面有示例： 具体见 Sql 语法知识： 
  
---

### Limitations and Settings
 
- 创建表时，应用以下[设置](../../operations/settings/settings.md):
  - `join_use_nulls`
  - `max_rows_in_join`
  - `max_bytes_in_join`
  - `join_overflow_mode`
  - `join_any_take_last_row`

- 连接引擎表不能用于全局连接操作。

---

### Data Storage

- 存储逻辑，与 `Set` 存储类似： 
  - 连接表数据总是位于 `RAM` 中。
  - 当将数据插入到表中时，ClickHouse将数据块写入磁盘上的目录，以便在服务器重新启动时恢复它们。

- 如果服务器重新启动不正确，磁盘上的数据块可能会丢失或损坏。在这种情况下，您可能需要手动删除数据损坏的文件。