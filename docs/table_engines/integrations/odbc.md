## ODBC

允许ClickHouse通过 `ODBC` 连接外部数据库。

- 为了安全地实现 `ODBC` 连接，ClickHouse使用一个单独的程序 `clickhouse-odbc-bridge`。
  - 如果直接从 `clickhouse-server` 加载 `ODBC` 驱动程序，驱动程序问题可能会导致 `ClickHouse` 服务器崩溃。
  - 需要时，`ClickHouse` 自动启动 `clickhouse-odbc-bridge`。
  - ODBC 桥程序是从与 `clickhouse-server` 相同的包中安装的。

此引擎支持 `Nullable` 的数据类型。

---

### Creating a Table

```clickhouse
CREATE TABLE [IF NOT EXISTS] [db.]table_name [ON CLUSTER cluster]
(
    name1 [type1],
    name2 [type2],
    ...
)
ENGINE = ODBC(connection_settings, external_database, external_table)
```

- 表结构可以不同于源表结构:
  - `Column Name` 应该与源表中的 `Column Name` 相同，但是您可以按照任何顺序使用其中的一些列。
  - `Column Type` 可能与源表中的 `Column Type` 不同。
    - ClickHouse尝试将值转换为ClickHouse数据类型。
    
**参数:** 

- `connection_settings`:  `odbc.ini` 文件中带有连接设置部分的名称. 
- `external_database` : 外部 DBMS `database`.
- `external_table` : `external_database` 表名称.

---

### Usage Example

- 通过ODBC从本地MySQL安装中检索数据
  - 系统和Mysql版本信息： `Ubuntu Linux 18.04` 和 `MySQL server 5.7`。
  - 确保安装了 `unixODBC` 和 `MySQL Connector`。
  - 默认情况下(如果从包中安装)，ClickHouse作为用户 `ClickHouse` 启动。
  - 因此，您需要在MySQL服务器中创建和配置这个用户。

```bash
$ sudo mysql
```

```mysql
CREATE USER 'clickhouse'@'localhost' IDENTIFIED BY 'clickhouse';
GRANT ALL PRIVILEGES ON *.* TO 'clickhouse'@'clickhouse' WITH GRANT OPTION;
```

然后在 `/etc/odbc.ini` 中配置连接。

```bash
$ cat /etc/odbc.ini
[mysqlconn]
DRIVER = /usr/local/lib/libmyodbc5w.so
SERVER = 127.0.0.1
PORT = 3306
DATABASE = test
USERNAME = clickhouse
PASSWORD = clickhouse
```

可以使用 `unixODBC` 安装中的 `isql` 实用程序检查连接。

```bash
$ isql -v mysqlconn
+---------------------------------------+
| Connected!                            |
|                                       |
...
```

MySQL表:

```mysql
CREATE TABLE `test`.`test` (
  `int_id` INT NOT NULL AUTO_INCREMENT,
  `int_nullable` INT NULL DEFAULT NULL,
  `float` FLOAT NOT NULL,
  `float_nullable` FLOAT NULL DEFAULT NULL,
  PRIMARY KEY (`int_id`));
Query OK, 0 rows affected (0,09 sec)

insert into test (`int_id`, `float`) VALUES (1,2);
Query OK, 1 row affected (0,00 sec)

select * from test;
+--------+--------------+-------+----------------+
| int_id | int_nullable | float | float_nullable |
+--------+--------------+-------+----------------+
|      1 |         NULL |     2 |           NULL |
+--------+--------------+-------+----------------+
1 row in set (0,00 sec)
```  

ClickHouse表，从MySQL表检索数据:
```clickhouse
CREATE TABLE odbc_t
(
    `int_id` Int32,
    `float_nullable` Nullable(Float32)
)
ENGINE = ODBC('DSN=mysqlconn', 'test', 'test')

SELECT * FROM odbc_t
```

```log
┌─int_id─┬─float_nullable─┐
│      1 │           ᴺᵁᴸᴸ │
└────────┴────────────────┘
```


