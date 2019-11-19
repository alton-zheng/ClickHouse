## Database Engines

数据库引擎允许您使用表。

默认情况下，ClickHouse 使用其native 数据库引擎，该引擎提供可配置的table engines和SQL。

您还可以使用以下数据库引擎：

- `MySQL`

- `Lazy`

## MySQL

允许连接到远程 MySQL 服务器上的数据库，并执行`INSERT`和`SELECT`查询表以在 ClickHouse 和 MySQL 之间交换数据。

该`MySQL`数据库引擎翻译查询到 `MySQL` 服务器，以便您可以执行的操作，如`SHOW TABLES`或`SHOW CREATE TABLE`。

您不能执行以下查询：

- `ATTACH`/`DETACH`
- `DROP`
- `RENAME`
- `CREATE TABLE`
- `ALTER`

### Creating a Database

```sql
CREATE DATABASE [IF NOT EXISTS] db_name [ON CLUSTER cluster]
ENGINE = MySQL('host:port', 'database', 'user', 'password')
```

**引擎参数**

- `host:port` — MySQL 服务器地址。
- `database` —远程数据库名称。
- `user`  MySQL 用户。
- `password` — 用户密码。

### Data Types Support

```
MySQL	                                   ClickHouse
UNSIGNED TINYINT	                       UInt8
TINYINT	                                   Int8
UNSIGNED SMALLINT	                       UInt16
SMALLINT	                               Int16
UNSIGNED INT, UNSIGNED MEDIUMINT	       UInt32
INT, MEDIUMINT	                           Int32
UNSIGNED BIGINT	                           UInt64
BIGINT	                                   Int64
FLOAT	                                   Float32
DOUBLE	                                   Float64
DATE	                                   Date
DATETIME, TIMESTAMP	                       DateTime
BINARY	                                   FixedString
```

- 所有其它类型都被转成`String`
- 支持`Nullable` 
 
### Examples Of Use

MySQL 中的表：

```mysql
USE test;
Database changed

CREATE TABLE `mysql_table` (
   `int_id` INT NOT NULL AUTO_INCREMENT,
   `float` FLOAT NOT NULL,
   PRIMARY KEY (`int_id`));
Query OK, 0 rows affected (0,09 sec)

insert into mysql_table (`int_id`, `float`) VALUES (1,2);
Query OK, 1 row affected (0,00 sec)

select * from mysql_table;
+--------+-------+
| int_id | value |
+--------+-------+
|      1 |     2 |
+--------+-------+
1 row in set (0,00 sec)
```

`ClickHouse`数据库与 `MySQL` 服务器交换数据：

```clickhouse
CREATE DATABASE mysql_db ENGINE = MySQL('localhost:3306', 'test', 'my_user', 'user_password')
SHOW DATABASES
┌─name─────┐
│ default  │
│ mysql_db │
│ system   │
└──────────┘
SHOW TABLES FROM mysql_db
┌─name─────────┐
│  mysql_table │
└──────────────┘
SELECT * FROM mysql_db.mysql_table
┌─int_id─┬─value─┐
│      1 │     2 │
└────────┴───────┘
INSERT INTO mysql_db.mysql_table VALUES (3,4)
SELECT * FROM mysql_db.mysql_table
┌─int_id─┬─value─┐
│      1 │     2 │
│      3 │     4 │
└────────┴───────┘
```
## Lazy

工作方式类似于`Ordinary`，但是`expiration_time_in_seconds`在上次访问后仅几秒钟将表保留在 RAM 中。只能与 `*Log` 表一起使用。

它针对存储许多小的`*Log` 表进行了优化，对于它们而言，两次访问之间的时间间隔很长。

### Create a Database

```clickhouse
CREATE DATABASE testlazy ENGINE = Lazy(expiration_time_in_seconds);
```
