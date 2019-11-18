## MySQL

MySQL 引擎可以对存储在远程 MySQL 服务器上的数据执行  `SELECT`  查询。

---

### Creating a Table
```sql
CREATE TABLE [IF NOT EXISTS] [db.]table_name [ON CLUSTER cluster]
(
    name1 [type1] [DEFAULT|MATERIALIZED|ALIAS expr1] [TTL expr1],
    name2 [type2] [DEFAULT|MATERIALIZED|ALIAS expr2] [TTL expr2],
    ...
) ENGINE = MySQL('host:port', 'database', 'table', 'user', 'password'[, replace_query, 'on_duplicate_clause']);
```

- 表结构可以不同于原来的MySQL表结构:
  - `Column Name` 应该与原始MySQL表中的 `Column Name` 相同，但是您可以按照任何顺序使用其中的一些列。
  - `Column Type` 可能与原始MySQL表中的 `Column Type` 不同。
    - ClickHouse尝试将值转换为ClickHouse数据类型。

**调用参数**

- `host:port` ：MySQL 服务器地址。
- `database` ： 远程数据库的名称。
- `table` ：表名称。
- `user` ：数据库用户。
- `password` ：用户密码。
- `replace_query` ： 将  `INSERT INTO`  查询是否替换为  `REPLACE INTO`  的标志。如果  `replace_query=1`，则替换查询
- `on_duplicate_clause` ： 将  `ON DUPLICATE KEY on_duplicate_clause`  表达式添加到  `INSERT`  查询语句中。
  - 例如：`INSERT INTO t (c1,c2) VALUES ('a', 2) ON DUPLICATE KEY UPDATE c2 = c2 + 1`。
    - 其中 `on_duplicate_clause` = `UPDATE c2 = c2 + 1`.
    - 查找 `on_duplicate_clause` 与 `ON DUPLICATE KEY`一起使用的文档， 见[MySQL 文档](https://dev.mysql.com/doc/refman/8.0/en/insert-on-duplicate.html)
  - 如果需要指定  `on_duplicate_clause`，则需要设置  `replace_query = 0`。
    - 如果同时设置  `replace_query = 1`  和  `on_duplicate_clause`，则会抛出异常。

此时，简单的 `WHERE` （例如  `=, !=, >, >=, <, <=`）是在 MySQL 服务器上执行。

其余条件以及 `LIMIT` 采样约束语句仅在对 MySQL 的查询完成后才在 ClickHouse 中执行。

---

### Usage Example

**MySQL 中的表**
```mysql
mysql> CREATE TABLE `test`.`test` (
    ->   `int_id` INT NOT NULL AUTO_INCREMENT,
    ->   `int_nullable` INT NULL DEFAULT NULL,
    ->   `float` FLOAT NOT NULL,
    ->   `float_nullable` FLOAT NULL DEFAULT NULL,
    ->   PRIMARY KEY (`int_id`));
Query OK, 0 rows affected (0,09 sec)

mysql> insert into test (`int_id`, `float`) VALUES (1,2);
Query OK, 1 row affected (0,00 sec)

mysql> select * from test;
+--------+--------------+-------+----------------+
| int_id | int_nullable | float | float_nullable |
+--------+--------------+-------+----------------+
|      1 |         NULL |     2 |           NULL |
+--------+--------------+-------+----------------+
1 row in set (0,00 sec)
```


**`ClickHouse`表，从上面创建的 `MySQL` 表中检索数据：**

```sql
CREATE TABLE mysql_table
(
    `float_nullable` Nullable(Float32),
    `int_id` Int32
)
ENGINE = MySQL('localhost:3306', 'test', 'test', 'bayonet', '123')

SELECT * FROM mysql_table
```

```
┌─float_nullable─┬─int_id─┐
│           ᴺᵁᴸᴸ │      1 │
└────────────────┴────────┘
```