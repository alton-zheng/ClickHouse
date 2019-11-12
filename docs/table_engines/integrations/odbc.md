## ODBC[¶](https://clickhouse.yandex/docs/zh/single/#table_engine-odbc "Permanent link")

Allows ClickHouse to connect to external databases via [ODBC](https://en.wikipedia.org/wiki/Open_Database_Connectivity).

To safely implement ODBC connections, ClickHouse uses a separate program `clickhouse-odbc-bridge`. If the ODBC driver is loaded directly from `clickhouse-server`, driver problems can crash the ClickHouse server. ClickHouse automatically starts `clickhouse-odbc-bridge` when it is required. The ODBC bridge program is installed from the same package as the `clickhouse-server`.

This engine supports the [Nullable](https://clickhouse.yandex/docs/zh/single/#../../data_types/nullable/) data type.

### Creating a Table[¶](https://clickhouse.yandex/docs/zh/single/#creating-a-table_3 "Permanent link")

CREATE TABLE \[IF NOT EXISTS\] \[db.\]table_name \[ON CLUSTER cluster\]
(
name1 \[type1\],
name2 \[type2\],
...
)
ENGINE = ODBC(connection_settings, external_database, external_table)

See a detailed description of the [CREATE TABLE](https://clickhouse.yandex/docs/zh/single/#create-table-query) query.

The table structure can differ from the source table structure:

- Column names should be the same as in the source table, but you can use just some of these columns and in any order.
- Column types may differ from those in the source table. ClickHouse tries to [cast](https://clickhouse.yandex/docs/zh/single/#type_conversion_function-cast) values to the ClickHouse data types.

**Engine Parameters**

- `connection_settings` — Name of the section with connection settings in the `odbc.ini` file.
- `external_database` — Name of a database in an external DBMS.
- `external_table` — Name of a table in the `external_database`.

### Usage Example[¶](https://clickhouse.yandex/docs/zh/single/#usage-example_2 "Permanent link")

**Retrieving data from the local MySQL installation via ODBC**

This example is checked for Ubuntu Linux 18.04 and MySQL server 5.7.

Ensure that unixODBC and MySQL Connector are installed.

By default (if installed from packages), ClickHouse starts as user `clickhouse`. Thus, you need to create and configure this user in the MySQL server.

\$ sudo mysql

mysql> CREATE USER 'clickhouse'@'localhost' IDENTIFIED BY 'clickhouse';
mysql> GRANT ALL PRIVILEGES ON _._ TO 'clickhouse'@'clickhouse' WITH GRANT OPTION;

Then configure the connection in `/etc/odbc.ini`.

\$ cat /etc/odbc.ini
\[mysqlconn\]
DRIVER = /usr/local/lib/libmyodbc5w.so
SERVER = 127.0.0.1
PORT = 3306
DATABASE = test
USERNAME = clickhouse
PASSWORD = clickhouse

You can check the connection using the `isql` utility from the unixODBC installation.

\$ isql -v mysqlconn
+---------------------------------------+
| Connected! |
| |
...

Table in MySQL:

mysql> CREATE TABLE \`test\`.\`test\` (
-\> \`int_id\` INT NOT NULL AUTO_INCREMENT,
-\> \`int_nullable\` INT NULL DEFAULT NULL,
-\> \`float\` FLOAT NOT NULL,
-\> \`float_nullable\` FLOAT NULL DEFAULT NULL,
-\> PRIMARY KEY (\`int_id\`));
Query OK, 0 rows affected (0,09 sec)

mysql> insert into test (\`int_id\`, \`float\`) VALUES (1,2);
Query OK, 1 row affected (0,00 sec)

mysql> select \* from test;
+--------+--------------+-------+----------------+
| int_id | int_nullable | float | float_nullable |
+--------+--------------+-------+----------------+
| 1 | NULL | 2 | NULL |
+--------+--------------+-------+----------------+
1 row in set (0,00 sec)

Table in ClickHouse, retrieving data from the MySQL table:

CREATE TABLE odbc_t
(
`int_id` Int32,
`float_nullable` Nullable(Float32)
)
ENGINE = ODBC('DSN=mysqlconn', 'test', 'test')

SELECT \* FROM odbc_t

┌─int_id─┬─float_nullable─┐
│ 1 │ ᴺᵁᴸᴸ │
└────────┴────────────────┘

### See Also[¶](https://clickhouse.yandex/docs/zh/single/#see-also_1 "Permanent link")

- [ODBC external dictionaries](https://clickhouse.yandex/docs/zh/single/#dicts-external_dicts_dict_sources-odbc)
- [ODBC table function](https://clickhouse.yandex/docs/zh/single/#../../query_language/table_functions/odbc/)
