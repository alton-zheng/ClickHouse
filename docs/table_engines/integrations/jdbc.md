## JDBC

通过 `JDBC` 连接外部数据库。

要实现 `JDBC` 连接，使用单独的程序 `clickhouse-jdbc-bridge`，该程序应作为守护进程运行。

此引擎支持 `Nullable` 数据类型。

---

### Creating a Table 
```sql
CREATE TABLE [IF NOT EXISTS] [db.]table_name
ENGINE = JDBC(dbms_uri, external_database, external_table)
```

参数: 

- `dbms_uri` — 外部 DBMS `URI`.

- `Format`: 
  - `jdbc:<driver_name>://<host_name>:<port>/?user=<username>&password=<password>`. 
  - 例： `MySQL`: `jdbc:mysql://localhost:3306/?user=root&password=root`.

- `external_database` — 外部 DBMS `database`.

- `external_table` : `external_database` 表名称.

---

### Usage Example

直接连接到控制台客户端上，在 MySQL 服务器上创建一张表:

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

在ClickHouse服务器中创建一个表并从中选择数据:

```clickhouse
CREATE TABLE jdbc_table ENGINE JDBC('jdbc:mysql://localhost:3306/?user=root&password=root', 'test', 'test');
```
```clickhouse
DESCRIBE TABLE jdbc_table
```
```log
┌─name───────────────┬─type───────────────┬─default_type─┬─default_expression─┐
│ int_id             │ Int32              │              │                    │
│ int_nullable       │ Nullable(Int32)    │              │                    │
│ float              │ Float32            │              │                    │
│ float_nullable     │ Nullable(Float32)  │              │                    │
└────────────────────┴────────────────────┴──────────────┴────────────────────┘
```

```clickhouse
select * from jdbc_table
```

```log
┌─int_id─┬─int_nullable─┬─float─┬─float_nullable─┐
│      1 │         ᴺᵁᴸᴸ │     2 │           ᴺᵁᴸᴸ │
└────────┴──────────────┴───────┴────────────────┘
```
