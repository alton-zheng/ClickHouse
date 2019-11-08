## **StripeLog**

### **建表** 

```sql
CREATE TABLE [IF NOT EXISTS] [db.]table_name [ON CLUSTER cluster]
(
    column1_name [type1] [DEFAULT|MATERIALIZED|ALIAS expr1],
    column2_name [type2] [DEFAULT|MATERIALIZED|ALIAS expr2],
    ...
) ENGINE = StripeLog
```

### `Write` 数据

每张 `table` 都会写入：

- `data.bin` — 数据文件。
- `index.mrk` — 带 `marks` 的文件。`marks` 包含了已插入的每个 `data block` 中每列的 `offset`。

不支持:
- `ALTER UPDATE`
- `ALTER DELETE`

### **`Read` 数据**

- 并发读取数据。
  - `SELECT`  顺序不可预测

### 使用示例

- 建表：

```sql
CREATE TABLE stripe_log_table
(
    timestamp DateTime,
    message_type String,
    message String
)
ENGINE = StripeLog
```

- 写数：

```sql
INSERT INTO stripe_log_table VALUES (now(),'REGULAR','The first regular message')
INSERT INTO stripe_log_table VALUES (now(),'REGULAR','The second regular message'),(now(),'WARNING','The first warning message')
```

每次 `INSERT`  请求 在`data.bin`  文件中都会创建一个 `data block`，上面示例会创建2个`data block`

每个线程读取单独的 `data block`

```sql
SELECT * FROM stripe_log_table
```

```
┌───────────timestamp─┬─message_type─┬─message────────────────────┐
│ 2019-01-18 14:27:32 │ REGULAR      │ The second regular message │
│ 2019-01-18 14:34:53 │ WARNING      │ The first warning message  │
└─────────────────────┴──────────────┴────────────────────────────┘
┌───────────timestamp─┬─message_type─┬─message───────────────────┐
│ 2019-01-18 14:23:43 │ REGULAR      │ The first regular message │
└─────────────────────┴──────────────┴───────────────────────────┘
```

对结果排序（默认 `ascending`）：

```sql
SELECT * FROM stripe_log_table ORDER BY timestamp
```

```
┌───────────timestamp─┬─message_type─┬─message────────────────────┐
│ 2019-01-18 14:23:43 │ REGULAR      │ The first regular message  │
│ 2019-01-18 14:27:32 │ REGULAR      │ The second regular message │
│ 2019-01-18 14:34:53 │ WARNING      │ The first warning message  │
└─────────────────────┴──────────────┴────────────────────────────┘
```

