## Kafka[¶](https://clickhouse.yandex/docs/zh/single/#kafka "Permanent link")

此引擎与  [Apache Kafka](http://kafka.apache.org/)  结合使用。

Kafka 特性：

- 发布或者订阅数据流。
- 容错存储机制。
- 处理流数据。

老版格式：

Kafka(kafka_broker_list, kafka_topic_list, kafka_group_name, kafka_format
\[, kafka_row_delimiter, kafka_schema, kafka_num_consumers\])

新版格式：

Kafka SETTINGS
kafka_broker_list = 'localhost:9092',
kafka_topic_list = 'topic1,topic2',
kafka_group_name = 'group1',
kafka_format = 'JSONEachRow',
kafka_row_delimiter = '\\n',
kafka_schema = '',
kafka_num_consumers = 2

必要参数：

- `kafka_broker_list` – 以逗号分隔的 brokers 列表 (`localhost:9092`)。
- `kafka_topic_list` – topic 列表 (`my_topic`)。
- `kafka_group_name` – Kafka 消费组名称 (`group1`)。如果不希望消息在集群中重复，请在每个分片中使用相同的组名。
- `kafka_format` – 消息体格式。使用与 SQL 部分的  `FORMAT`  函数相同表示方法，例如  `JSONEachRow`。了解详细信息，请参考  `Formats`  部分。

可选参数：

- `kafka_row_delimiter` \- 每个消息体（记录）之间的分隔符。
- `kafka_schema` – 如果解析格式需要一个 schema 时，此参数必填。例如，[Cap'n Proto](https://capnproto.org/)  需要 schema 文件路径以及根对象  `schema.capnp:Message`  的名字。
- `kafka_num_consumers` – 单个表的消费者数量。默认值是：`1`，如果一个消费者的吞吐量不足，则指定更多的消费者。消费者的总数不应该超过 topic 中分区的数量，因为每个分区只能分配一个消费者。

示例：

CREATE TABLE queue (
timestamp UInt64,
level String,
message String
) ENGINE = Kafka('localhost:9092', 'topic', 'group1', 'JSONEachRow');

SELECT \* FROM queue LIMIT 5;

CREATE TABLE queue2 (
timestamp UInt64,
level String,
message String
) ENGINE = Kafka SETTINGS kafka_broker_list = 'localhost:9092',
kafka_topic_list = 'topic',
kafka_group_name = 'group1',
kafka_format = 'JSONEachRow',
kafka_num_consumers = 4;

CREATE TABLE queue2 (
timestamp UInt64,
level String,
message String
) ENGINE = Kafka('localhost:9092', 'topic', 'group1')
SETTINGS kafka_format = 'JSONEachRow',
kafka_num_consumers = 4;

消费的消息会被自动追踪，因此每个消息在不同的消费组里只会记录一次。如果希望获得两次数据，则使用另一个组名创建副本。

消费组可以灵活配置并且在集群之间同步。例如，如果群集中有 10 个主题和 5 个表副本，则每个副本将获得 2 个主题。 如果副本数量发生变化，主题将自动在副本中重新分配。了解更多信息请访问  [http://kafka.apache.org/intro](http://kafka.apache.org/intro)。

`SELECT`  查询对于读取消息并不是很有用（调试除外），因为每条消息只能被读取一次。使用物化视图创建实时线程更实用。您可以这样做：

1.  使用引擎创建一个 Kafka 消费者并作为一条数据流。
2.  创建一个结构表。
3.  创建物化视图，改视图会在后台转换引擎中的数据并将其放入之前创建的表中。

当  `MATERIALIZED VIEW`  添加至引擎，它将会在后台收集数据。可以持续不断地从 Kafka 收集数据并通过  `SELECT`  将数据转换为所需要的格式。

示例：

CREATE TABLE queue (
timestamp UInt64,
level String,
message String
) ENGINE = Kafka('localhost:9092', 'topic', 'group1', 'JSONEachRow');

CREATE TABLE daily (
day Date,
level String,
total UInt64
) ENGINE = SummingMergeTree(day, (day, level), 8192);

CREATE MATERIALIZED VIEW consumer TO daily
AS SELECT toDate(toDateTime(timestamp)) AS day, level, count() as total
FROM queue GROUP BY day, level;

SELECT level, sum(total) FROM daily GROUP BY level;

为了提高性能，接受的消息被分组为  [max_insert_block_size](https://clickhouse.yandex/docs/zh/single/#settings-max_insert_block_size)  大小的块。如果未在  [stream_flush_interval_ms](https://clickhouse.yandex/docs/zh/single/#../settings/settings/)  毫秒内形成块，则不关心块的完整性，都会将数据刷新到表中。

停止接收主题数据或更改转换逻辑，请 detach 物化视图：

DETACH TABLE consumer;
ATTACH MATERIALIZED VIEW consumer;

如果使用  `ALTER`  更改目标表，为了避免目标表与视图中的数据之间存在差异，推荐停止物化视图。

### 配置[¶](https://clickhouse.yandex/docs/zh/single/#pei-zhi "Permanent link")

与  `GraphiteMergeTree`  类似，Kafka 引擎支持使用 ClickHouse 配置文件进行扩展配置。可以使用两个配置键：全局 (`kafka`) 和 主题级别 (`kafka_*`)。首先应用全局配置，然后应用主题级配置（如果存在）。

<!\-\- Global configuration options for all tables of Kafka engine type -->
<kafka>
<debug>cgrp</debug>
<auto_offset_reset>smallest</auto_offset_reset>
</kafka>

<!\-\- Configuration specific for topic "logs" -->
<kafka_logs>
<retry_backoff_ms>250</retry_backoff_ms>
<fetch_min_bytes>100000</fetch_min_bytes>
</kafka_logs>

有关详细配置选项列表，请参阅  [librdkafka configuration reference](https://github.com/edenhill/librdkafka/blob/master/CONFIGURATION.md)。在 ClickHouse 配置中使用下划线 (`_`) ，并不是使用点 (`.`)。例如，`check.crcs=true`  将是  `<check_crcs>true</check_crcs>`。

## MySQL[¶](https://clickhouse.yandex/docs/zh/single/#mysql_1 "Permanent link")

MySQL 引擎可以对存储在远程 MySQL 服务器上的数据执行  `SELECT`  查询。

调用格式：

MySQL('host:port', 'database', 'table', 'user', 'password'\[, replace_query, 'on_duplicate_clause'\]);

**调用参数**

- `host:port` — MySQL 服务器地址。
- `database` — 数据库的名称。
- `table` — 表名称。
- `user` — 数据库用户。
- `password` — 用户密码。
- `replace_query` — 将  `INSERT INTO`  查询是否替换为  `REPLACE INTO`  的标志。如果  `replace_query=1`，则替换查询
- `'on_duplicate_clause'` — 将  `ON DUPLICATE KEY UPDATE 'on_duplicate_clause'`  表达式添加到  `INSERT`  查询语句中。例如：`impression = VALUES(impression) + impression`。如果需要指定  `'on_duplicate_clause'`，则需要设置  `replace_query=0`。如果同时设置  `replace_query = 1`  和  `'on_duplicate_clause'`，则会抛出异常。

此时，简单的  `WHERE`  子句（例如  `=, !=, >, >=, <, <=`）是在 MySQL 服务器上执行。

其余条件以及  `LIMIT`  采样约束语句仅在对 MySQL 的查询完成后才在 ClickHouse 中执行。

`MySQL`  引擎不支持  [Nullable](https://clickhouse.yandex/docs/zh/single/#../../data_types/nullable/)  数据类型，因此，当从 MySQL 表中读取数据时，`NULL`  将转换为指定列类型的默认值（通常为 0 或空字符串）。

## JDBC[¶](https://clickhouse.yandex/docs/zh/single/#table_engine-jdbc "Permanent link")

Allows ClickHouse to connect to external databases via [JDBC](https://en.wikipedia.org/wiki/Java_Database_Connectivity).

To implement the JDBC connection, ClickHouse uses the separate program [clickhouse-jdbc-bridge](https://github.com/alex-krash/clickhouse-jdbc-bridge) that should run as a daemon.

This engine supports the [Nullable](https://clickhouse.yandex/docs/zh/single/#../../data_types/nullable/) data type.

### Creating a Table[¶](https://clickhouse.yandex/docs/zh/single/#creating-a-table_2 "Permanent link")

CREATE TABLE \[IF NOT EXISTS\] \[db.\]table_name
ENGINE = JDBC(dbms_uri, external_database, external_table)

**Engine Parameters**

- `dbms_uri` — URI of an external DBMS.

  Format: `jdbc:<driver_name>://<host_name>:<port>/?user=<username>&password=<password>`. Example for MySQL: `jdbc:mysql://localhost:3306/?user=root&password=root`.

- `external_database` — Database in an external DBMS.

- `external_table` — Name of the table in `external_database`.

### Usage Example[¶](https://clickhouse.yandex/docs/zh/single/#usage-example_1 "Permanent link")

Creating a table in MySQL server by connecting directly with it's console client:

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

Creating a table in ClickHouse server and selecting data from it:

CREATE TABLE jdbc_table ENGINE JDBC('jdbc:mysql://localhost:3306/?user=root&password=root', 'test', 'test')

DESCRIBE TABLE jdbc_table

┌─name───────────────┬─type───────────────┬─default_type─┬─default_expression─┐
│ int_id │ Int32 │ │ │
│ int_nullable │ Nullable(Int32) │ │ │
│ float │ Float32 │ │ │
│ float_nullable │ Nullable(Float32) │ │ │
└────────────────────┴────────────────────┴──────────────┴────────────────────┘

SELECT \*
FROM jdbc_table

┌─int_id─┬─int_nullable─┬─float─┬─float_nullable─┐
│ 1 │ ᴺᵁᴸᴸ │ 2 │ ᴺᵁᴸᴸ │
└────────┴──────────────┴───────┴────────────────┘

### See Also[¶](https://clickhouse.yandex/docs/zh/single/#see-also "Permanent link")

- [JDBC table function](https://clickhouse.yandex/docs/zh/single/#../../query_language/table_functions/jdbc/).

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

## HDFS[¶](https://clickhouse.yandex/docs/zh/single/#table_engines-hdfs "Permanent link")

This engine provides integration with [Apache Hadoop](https://en.wikipedia.org/wiki/Apache_Hadoop) ecosystem by allowing to manage data on [HDFS](https://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-hdfs/HdfsDesign.htmll)via ClickHouse. This engine is similar to the [File](https://clickhouse.yandex/docs/zh/single/#file/) and [URL](https://clickhouse.yandex/docs/zh/single/#url/) engines, but provides Hadoop-specific features.

### Usage[¶](https://clickhouse.yandex/docs/zh/single/#usage "Permanent link")

ENGINE = HDFS(URI, format)

The `URI` parameter is the whole file URI in HDFS. The `format` parameter specifies one of the available file formats. To perform `SELECT` queries, the format must be supported for input, and to perform `INSERT` queries -- for output. The available formats are listed in the [Formats](https://clickhouse.yandex/docs/zh/single/#formats) section. The path part of `URI` may contain globs. In this case the table would be readonly.

**Example:**

**1.** Set up the `hdfs_engine_table` table:

CREATE TABLE hdfs_engine_table (name String, value UInt32) ENGINE=HDFS('hdfs://hdfs1:9000/other_storage', 'TSV')

**2.** Fill file:

INSERT INTO hdfs_engine_table VALUES ('one', 1), ('two', 2), ('three', 3)

**3.** Query the data:

SELECT \* FROM hdfs_engine_table LIMIT 2

┌─name─┬─value─┐
│ one │ 1 │
│ two │ 2 │
└──────┴───────┘

### Implementation Details[¶](https://clickhouse.yandex/docs/zh/single/#implementation-details "Permanent link")

- Reads and writes can be parallel
- Not supported:
  - `ALTER` and `SELECT...SAMPLE` operations.
  - Indexes.
  - Replication.

**Globs in path**

Multiple path components can have globs. For being processed file should exists and matches to the whole path pattern. Listing of files determines during `SELECT` (not at `CREATE` moment).

- `*` — Substitutes any number of any characters except `/` including empty string.
- `?` — Substitutes any single character.
- `{some_string,another_string,yet_another_one}` — Substitutes any of strings `'some_string', 'another_string', 'yet_another_one'`.
- `{N..M}` — Substitutes any number in range from N to M including both borders.

Constructions with `{}` are similar to the [remote](https://clickhouse.yandex/docs/zh/single/#../../query_language/table_functions/remote/) table function.

**Example**

1.  Suppose we have several files in TSV format with the following URIs on HDFS:

2.  'hdfs://hdfs1:9000/some_dir/some_file_1'

3.  'hdfs://hdfs1:9000/some_dir/some_file_2'
4.  'hdfs://hdfs1:9000/some_dir/some_file_3'
5.  'hdfs://hdfs1:9000/another_dir/some_file_1'
6.  'hdfs://hdfs1:9000/another_dir/some_file_2'
7.  'hdfs://hdfs1:9000/another_dir/some_file_3'

8.  There are several ways to make a table consisting of all six files:

CREATE TABLE table_with_range (name String, value UInt32) ENGINE = HDFS('hdfs://hdfs1:9000/{some,another}\_dir/some_file\_{1..3}', 'TSV')

Another way:

CREATE TABLE table_with_question*mark (name String, value UInt32) ENGINE = HDFS('hdfs://hdfs1:9000/{some,another}\_dir/some_file*?', 'TSV')

Table consists of all the files in both directories (all files should satisfy format and schema described in query):

CREATE TABLE table_with_asterisk (name String, value UInt32) ENGINE = HDFS('hdfs://hdfs1:9000/{some,another}\_dir/\*', 'TSV')

Warning

If the listing of files contains number ranges with leading zeros, use the construction with braces for each digit separately or use `?`.

**Example**

Create table with files named `file000`, `file001`, ... , `file999`:

CREARE TABLE big_table (name String, value UInt32) ENGINE = HDFS('hdfs://hdfs1:9000/big_dir/file{0..9}{0..9}{0..9}', 'CSV')

## Distributed[¶](https://clickhouse.yandex/docs/zh/single/#distributed "Permanent link")

**分布式引擎本身不存储数据**, 但可以在多个服务器上进行分布式查询。 读是自动并行的。读取时，远程服务器表的索引（如果有的话）会被使用。 分布式引擎参数：服务器配置文件中的集群名，远程数据库名，远程表名，数据分片键（可选）。 示例：

Distributed(logs, default, hits\[, sharding_key\])

将会从位于“logs”集群中 default.hits 表所有服务器上读取数据。 远程服务器不仅用于读取数据，还会对尽可能数据做部分处理。 例如，对于使用 GROUP BY 的查询，数据首先在远程服务器聚合，之后返回聚合函数的中间状态给查询请求的服务器。再在请求的服务器上进一步汇总数据。

数据库名参数除了用数据库名之外，也可用返回字符串的常量表达式。例如：currentDatabase()。

logs – 服务器配置文件中的集群名称。

集群示例配置如下：

<remote_servers>
<logs>
<shard>
<!\-\- Optional. Shard weight when writing data. Default: 1. -->
<weight>1</weight>
<!\-\- Optional. Whether to write data to just one of the replicas. Default: false (write data to all replicas). -->
<internal_replication>false</internal_replication>
<replica>
<host>example01-01-1</host>
<port>9000</port>
</replica>
<replica>
<host>example01-01-2</host>
<port>9000</port>
</replica>
</shard>
<shard>
<weight>2</weight>
<internal_replication>false</internal_replication>
<replica>
<host>example01-02-1</host>
<port>9000</port>
</replica>
<replica>
<host>example01-02-2</host>
<secure>1</secure>
<port>9440</port>
</replica>
</shard>
</logs>
</remote_servers>

这里定义了一个名为‘logs’的集群，它由两个分片组成，每个分片包含两个副本。 分片是指包含数据不同部分的服务器（要读取所有数据，必须访问所有分片）。 副本是存储复制数据的服务器（要读取所有数据，访问任一副本上的数据即可）。

集群名称不能包含点号。

每个服务器需要指定  `host`，`port`，和可选的  `user`，`password`，`secure`，`compression`  的参数：

- `host` – 远程服务器地址。可以域名、IPv4 或 IPv6。如果指定域名，则服务在启动时发起一个 DNS 请求，并且请求结果会在服务器运行期间一直被记录。如果 DNS 请求失败，则服务不会启动。如果你修改了 DNS 记录，则需要重启服务。
- `port` – 消息传递的 TCP 端口（「tcp_port」配置通常设为 9000）。不要跟 http_port 混淆。
- `user` – 用于连接远程服务器的用户名。默认值：default。该用户必须有权限访问该远程服务器。访问权限配置在 users.xml 文件中。更多信息，请查看“访问权限”部分。
- `password` – 用于连接远程服务器的密码。默认值：空字符串。
- `secure` – 是否使用 ssl 进行连接，设为 true 时，通常也应该设置  `port` = 9440。服务器也要监听  9440  并有正确的证书。
- `compression` \- 是否使用数据压缩。默认值：true。

配置了副本，读取操作会从每个分片里选择一个可用的副本。可配置负载平衡算法（挑选副本的方式） \- 请参阅“load_balancing”设置。 如果跟服务器的连接不可用，则在尝试短超时的重连。如果重连失败，则选择下一个副本，依此类推。如果跟所有副本的连接尝试都失败，则尝试用相同的方式再重复几次。 该机制有利于系统可用性，但不保证完全容错：如有远程服务器能够接受连接，但无法正常工作或状况不佳。

你可以配置一个（这种情况下，查询操作更应该称为远程查询，而不是分布式查询）或任意多个分片。在每个分片中，可以配置一个或任意多个副本。不同分片可配置不同数量的副本。

可以在配置中配置任意数量的集群。

要查看集群，可使用“system.clusters”表。

通过分布式引擎可以像使用本地服务器一样使用集群。但是，集群不是自动扩展的：你必须编写集群配置到服务器配置文件中（最好，给所有集群的服务器写上完整配置）。

不支持用分布式表查询别的分布式表（除非该表只有一个分片）。或者说，要用分布表查查询“最终”的数据表。

分布式引擎需要将集群信息写入配置文件。配置文件中的集群信息会即时更新，无需重启服务器。如果你每次是要向不确定的一组分片和副本发送查询，则不适合创建分布式表 \- 而应该使用“远程”表函数。 请参阅“表函数”部分。

向集群写数据的方法有两种：

一，自已指定要将哪些数据写入哪些服务器，并直接在每个分片上执行写入。换句话说，在分布式表上“查询”，在数据表上 INSERT。 这是最灵活的解决方案 – 你可以使用任何分片方案，对于复杂业务特性的需求，这可能是非常重要的。 这也是最佳解决方案，因为数据可以完全独立地写入不同的分片。

二，在分布式表上执行 INSERT。在这种情况下，分布式表会跨服务器分发插入数据。 为了写入分布式表，必须要配置分片键（最后一个参数）。当然，如果只有一个分片，则写操作在没有分片键的情况下也能工作，因为这种情况下分片键没有意义。

每个分片都可以在配置文件中定义权重。默认情况下，权重等于 1。数据依据分片权重按比例分发到分片上。例如，如果有两个分片，第一个分片的权重是 9，而第二个分片的权重是 10，则发送 9 / 19 的行到第一个分片， 10 / 19 的行到第二个分片。

分片可在配置文件中定义 'internal_replication' 参数。

此参数设置为“true”时，写操作只选一个正常的副本写入数据。如果分布式表的子表是复制表(\*ReplicaMergeTree)，请使用此方案。换句话说，这其实是把数据的复制工作交给实际需要写入数据的表本身而不是分布式表。

若此参数设置为“false”（默认值），写操作会将数据写入所有副本。实质上，这意味着要分布式表本身来复制数据。这种方式不如使用复制表的好，因为不会检查副本的一致性，并且随着时间的推移，副本数据可能会有些不一样。

选择将一行数据发送到哪个分片的方法是，首先计算分片表达式，然后将这个计算结果除以所有分片的权重总和得到余数。该行会发送到那个包含该余数的从'prev_weight'到'prev_weights + weight'的半闭半开区间对应的分片上，其中 'prev_weights' 是该分片前面的所有分片的权重和，'weight' 是该分片的权重。例如，如果有两个分片，第一个分片权重为 9，而第二个分片权重为 10，则余数在 \[0,9) 中的行发给第一个分片，余数在 \[9,19) 中的行发给第二个分片。

分片表达式可以是由常量和表列组成的任何返回整数表达式。例如，您可以使用表达式 'rand()' 来随机分配数据，或者使用 'UserID' 来按用户 ID 的余数分布（相同用户的数据将分配到单个分片上，这可降低带有用户信息的 IN 和 JOIN 的语句运行的复杂度）。如果该列数据分布不够均匀，可以将其包装在散列函数中：intHash64(UserID)。

这种简单的用余数来选择分片的方案是有局限的，并不总适用。它适用于中型和大型数据（数十台服务器）的场景，但不适用于巨量数据（数百台或更多服务器）的场景。后一种情况下，应根据业务特性需求考虑的分片方案，而不是直接用分布式表的多分片。

SELECT 查询会被发送到所有分片，并且无论数据在分片中如何分布（即使数据完全随机分布）都可正常工作。添加新分片时，不必将旧数据传输到该分片。你可以给新分片分配大权重然后写新数据 - 数据可能会稍分布不均，但查询会正确高效地运行。

下面的情况，你需要关注分片方案：

- 使用需要特定键连接数据（ IN 或 JOIN ）的查询。如果数据是用该键进行分片，则应使用本地 IN 或 JOIN 而不是 GLOBAL IN 或 GLOBAL JOIN，这样效率更高。
- 使用大量服务器（上百或更多），但有大量小查询（个别客户的查询 \- 网站，广告商或合作伙伴）。为了使小查询不影响整个集群，让单个客户的数据处于单个分片上是有意义的。或者，正如我们在 Yandex.Metrica 中所做的那样，你可以配置两级分片：将整个集群划分为“层”，一个层可以包含多个分片。单个客户的数据位于单个层上，根据需要将分片添加到层中，层中的数据随机分布。然后给每层创建分布式表，再创建一个全局的分布式表用于全局的查询。

数据是异步写入的。对于分布式表的 INSERT，数据块只写本地文件系统。之后会尽快地在后台发送到远程服务器。你可以通过查看表目录中的文件列表（等待发送的数据）来检查数据是否成功发送：/var/lib/clickhouse/data/database/table/ 。

如果在 INSERT 到分布式表时服务器节点丢失或重启（如，设备故障），则插入的数据可能会丢失。如果在表目录中检测到损坏的数据分片，则会将其转移到“broken”子目录，并不再使用。

启用 max_parallel_replicas 选项后，会在分表的所有副本上并行查询处理。更多信息，请参阅“设置，max_parallel_replicas”部分。

## External Data for Query Processing[¶](https://clickhouse.yandex/docs/zh/single/#external-data-for-query-processing "Permanent link")

ClickHouse 允许向服务器发送处理查询所需的数据以及 SELECT 查询。这些数据放在一个临时表中（请参阅 "临时表" 一节），可以在查询中使用（例如，在 IN 操作符中）。

例如，如果您有一个包含重要用户标识符的文本文件，则可以将其与使用此列表过滤的查询一起上传到服务器。

如果需要使用大量外部数据运行多个查询，请不要使用该特性。最好提前把数据上传到数据库。

可以使用命令行客户端（在非交互模式下）或使用 HTTP 接口上传外部数据。

在命令行客户端中，您可以指定格式的参数部分

--external --file=... \[--name=...\] \[--format=...\] \[--types=...|--structure=...\]

对于传输的表的数量，可能有多个这样的部分。

**--external** – 标记子句的开始。 **--file** – 带有表存储的文件的路径，或者，它指的是 STDIN。 只能从 stdin 中检索单个表。

以下的参数是可选的：**--name** – 表的名称，如果省略，则采用 \_data。 **--format** – 文件中的数据格式。 如果省略，则使用 TabSeparated。

以下的参数必选一个：**--types** – 逗号分隔列类型的列表。例如：`UInt64,String`。列将被命名为 \_1，\_2，... **--structure**– 表结构的格式  `UserID UInt64`，`URL String`。定义列的名字以及类型。

在 "file" 中指定的文件将由 "format" 中指定的格式解析，使用在 "types" 或 "structure" 中指定的数据类型。该表将被上传到服务器，并在作为名称为 "name"临时表。

示例：

echo -ne "1\\n2\\n3\\n" | clickhouse-client --query="SELECT count() FROM test.visits WHERE TraficSourceID IN \_data" --external --file=\- --types=Int8
849897
cat /etc/passwd | sed 's/:/\\t/g' | clickhouse-client --query="SELECT shell, count() AS c FROM passwd GROUP BY shell ORDER BY c DESC" --external --file=\- --name=passwd --structure='login String, unused String, uid UInt16, gid UInt16, comment String, home String, shell String'
/bin/sh 20
/bin/false 5
/bin/bash 4
/usr/sbin/nologin 1
/bin/sync 1

当使用 HTTP 接口时，外部数据以 multipart/form-data 格式传递。每个表作为一个单独的文件传输。表名取自文件名。"query_string" 传递参数 "name_format"、"name_types"和"name_structure"，其中 "name" 是这些参数对应的表的名称。参数的含义与使用命令行客户端时的含义相同。

示例：

cat /etc/passwd | sed 's/:/\\t/g' \> passwd.tsv

curl -F 'passwd=@passwd.tsv;' 'http://localhost:8123/?query=SELECT+shell,+count()+AS+c+FROM+passwd+GROUP+BY+shell+ORDER+BY+c+DESC&passwd_structure=login+String,+unused+String,+uid+UInt16,+gid+UInt16,+comment+String,+home+String,+shell+String'
/bin/sh 20
/bin/false 5
/bin/bash 4
/usr/sbin/nologin 1
/bin/sync 1

对于分布式查询，将临时表发送到所有远程服务器。

## Dictionary[¶](https://clickhouse.yandex/docs/zh/single/#dictionary "Permanent link")

`Dictionary`  引擎将字典数据展示为一个 ClickHouse 的表。

例如，考虑使用一个具有以下配置的  `products`  字典：

<dictionaries>
<dictionary>
        <name>products</name>
        <source>
            <odbc>
                <table>products</table>
                <connection_string>DSN=some-db-server</connection_string>
            </odbc>
        </source>
        <lifetime>
            <min>300</min>
            <max>360</max>
        </lifetime>
        <layout>
            <flat/>
        </layout>
        <structure>
            <id>
                <name>product_id</name>
            </id>
            <attribute>
                <name>title</name>
                <type>String</type>
                <null\_value></null\_value>
            </attribute>
        </structure>
</dictionary>
</dictionaries>

查询字典中的数据：

select name, type, key, attribute.names, attribute.types, bytes_allocated, element_count,source from system.dictionaries where name = 'products';

SELECT
name,
type,
key,
attribute.names,
attribute.types,
bytes_allocated,
element_count,
source
FROM system.dictionaries
WHERE name = 'products'

┌─name─────┬─type─┬─key────┬─attribute.names─┬─attribute.types─┬─bytes_allocated─┬─element_count─┬─source──────────┐
│ products │ Flat │ UInt64 │ \['title'\] │ \['String'\] │ 23065376 │ 175032 │ ODBC: .products │
└──────────┴──────┴────────┴─────────────────┴─────────────────┴─────────────────┴───────────────┴─────────────────┘

你可以使用  [dictGet\*](https://clickhouse.yandex/docs/zh/single/#../../query_language/functions/ext_dict_functions/)  函数来获取这种格式的字典数据。

当你需要获取原始数据，或者是想要使用  `JOIN`  操作的时候，这种视图并没有什么帮助。对于这些情况，你可以使用  `Dictionary`  引擎，它可以将字典数据展示在表中。

语法：

CREATE TABLE %table_name% (%fields%) engine = Dictionary(%dictionary_name%)`

示例：

create table products (product_id UInt64, title String) Engine = Dictionary(products);

CREATE TABLE products
(
product_id UInt64,
title String,
)
ENGINE = Dictionary(products)

Ok.

0 rows in set. Elapsed: 0.004 sec.

看一看表中的内容。

select \* from products limit 1;

SELECT \*
FROM products
LIMIT 1

┌────product_id─┬─title───────────┐
│ 152689 │ Some item │
└───────────────┴─────────────────┘

1 rows in set. Elapsed: 0.006 sec.

## Merge[¶](https://clickhouse.yandex/docs/zh/single/#merge "Permanent link")

`Merge`  引擎 (不要跟  `MergeTree`  引擎混淆) 本身不存储数据，但可用于同时从任意多个其他的表中读取数据。 读是自动并行的，不支持写入。读取时，那些被真正读取到数据的表的索引（如果有的话）会被使用。 `Merge`  引擎的参数：一个数据库名和一个用于匹配表名的正则表达式。

示例：

Merge(hits, '^WatchLog')

数据会从  `hits`  数据库中表名匹配正则 '`^WatchLog`' 的表中读取。

除了数据库名，你也可以用一个返回字符串的常量表达式。例如， `currentDatabase()` 。

正则表达式 — [re2](https://github.com/google/re2) (支持 PCRE 一个子集的功能)，大小写敏感。 了解关于正则表达式中转义字符的说明可参看 "match" 一节。

当选择需要读的表时，`Merge`  表本身会被排除，即使它匹配上了该正则。这样设计为了避免循环。 当然，是能够创建两个相互无限递归读取对方数据的  `Merge`  表的，但这并没有什么意义。

`Merge`  引擎的一个典型应用是可以像使用一张表一样使用大量的  `TinyLog`  表。

示例 2 ：

我们假定你有一个旧表（WatchLog_old），你想改变数据分区了，但又不想把旧数据转移到新表（WatchLog_new）里，并且你需要同时能看到这两个表的数据。

CREATE TABLE WatchLog_old(date Date, UserId Int64, EventType String, Cnt UInt64)
ENGINE=MergeTree(date, (UserId, EventType), 8192);
INSERT INTO WatchLog_old VALUES ('2018-01-01', 1, 'hit', 3);

CREATE TABLE WatchLog_new(date Date, UserId Int64, EventType String, Cnt UInt64)
ENGINE=MergeTree PARTITION BY date ORDER BY (UserId, EventType) SETTINGS index_granularity=8192;
INSERT INTO WatchLog_new VALUES ('2018-01-02', 2, 'hit', 3);

CREATE TABLE WatchLog as WatchLog_old ENGINE=Merge(currentDatabase(), '^WatchLog');

SELECT \*
FROM WatchLog

┌───────date─┬─UserId─┬─EventType─┬─Cnt─┐
│ 2018-01-01 │ 1 │ hit │ 3 │
└────────────┴────────┴───────────┴─────┘
┌───────date─┬─UserId─┬─EventType─┬─Cnt─┐
│ 2018-01-02 │ 2 │ hit │ 3 │
└────────────┴────────┴───────────┴─────┘

### `Virtual column`[¶](https://clickhouse.yandex/docs/zh/single/#xu-ni-lie_1 "Permanent link")

`Virtual column`是一种由`table engines`提供而不是在表定义中的列。换种说法就是，这些列并没有在  `CREATE TABLE`  中指定，但可以在  `SELECT`  中使用。

下面列出`Virtual column`跟普通列的不同点：

- `Virtual column`不在表结构定义里指定。
- 不能用  `INSERT`  向`Virtual column`写数据。
- 使用不指定列名的  `INSERT`  语句时，`Virtual column`要会被忽略掉。
- 使用星号通配符（ `SELECT *` ）时`Virtual column`不会包含在里面。
- `Virtual column`不会出现在  `SHOW CREATE TABLE`  和  `DESC TABLE`  的查询结果里。

`Merge`  类型的表包括一个  `String`  类型的  `_table`  `Virtual column`。（如果该表本来已有了一个  `_table`  的列，那这个`Virtual column`会命名为  `_table1` ；如果  `_table1`  也本就存在了，那这个`Virtual column`会被命名为  `_table2` ，依此类推）该列包含被读数据的表名。

如果  `WHERE/PREWHERE`  子句包含了带  `_table`  的条件，并且没有依赖其他的列（如作为表达式谓词链接的一个子项或作为整个的表达式），这些条件的作用会像索引一样。这些条件会在那些可能被读数据的表的表名上执行，并且读操作只会在那些满足了该条件的表上去执行。

## File(InputFormat)[¶](https://clickhouse.yandex/docs/zh/single/#table_engines-file "Permanent link")

数据源是以 Clickhouse 支持的一种输入格式（TabSeparated，Native 等）存储数据的文件。

用法示例：

- 从 ClickHouse 导出数据到文件。
- 将数据从一种格式转换为另一种格式。
- 通过编辑`disk`上的文件来更新 ClickHouse 中的数据。

### 在 ClickHouse 服务器中的使用[¶](https://clickhouse.yandex/docs/zh/single/#zai-clickhouse-fu-wu-qi-zhong-de-shi-yong "Permanent link")

File(Format)

选用的  `Format`  需要支持  `INSERT`  或  `SELECT` 。有关支持格式的完整列表，请参阅  [格式](https://clickhouse.yandex/docs/zh/single/#formats)。

ClickHouse 不支持给  `File`  指定文件系统路径。它使用服务器配置中  [path](https://clickhouse.yandex/docs/zh/single/#../server_settings/settings/)  设定的文件夹。

使用  `File(Format)`  创建表时，它会在该文件夹中创建空的子目录。当数据写入该表时，它会写到该子目录中的  `data.Format`  文件中。

你也可以在服务器文件系统中手动创建这些子文件夹和文件，然后通过  [ATTACH](https://clickhouse.yandex/docs/zh/single/#../../query_language/misc/)  将其创建为具有对应名称的表，这样你就可以从该文件中查询数据了。

!!! 注意 注意这个功能，因为 ClickHouse 不会跟踪这些文件在外部的更改。在 ClickHouse 中和 ClickHouse 外部同时写入会造成结果是不确定的。

**示例：**

**1.**  创建  `file_engine_table`  表：

CREATE TABLE file_engine_table (name String, value UInt32) ENGINE=File(TabSeparated)

默认情况下，Clickhouse 会创建目录  `/var/lib/clickhouse/data/default/file_engine_table` 。

**2.**  手动创建  `/var/lib/clickhouse/data/default/file_engine_table/data.TabSeparated`  文件，并且包含内容：

\$ cat data.TabSeparated
one 1
two 2

**3.**  查询这些数据:

SELECT \* FROM file_engine_table

┌─name─┬─value─┐
│ one │ 1 │
│ two │ 2 │
└──────┴───────┘

### 在 Clickhouse-local 中的使用[¶](https://clickhouse.yandex/docs/zh/single/#zai-clickhouse-local-zhong-de-shi-yong "Permanent link")

使用  [clickhouse-local](https://clickhouse.yandex/docs/zh/single/#../utils/clickhouse-local/)  时，File 引擎除了  `Format`  之外，还可以接受文件路径参数。可以使用数字或人类可读的名称来指定标准输入/输出流，例如  `0`  或  `stdin`，`1`  或  `stdout`。 **例如：**

\$ echo -e "1,2\\n3,4" | clickhouse-local -q "CREATE TABLE table (a Int64, b Int64) ENGINE = File(CSV, stdin); SELECT a, b FROM table; DROP TABLE table"

### 功能实现[¶](https://clickhouse.yandex/docs/zh/single/#gong-neng-shi-xian "Permanent link")

- 读操作可支持并发，但写操作不支持
- 不支持:
- `ALTER`
- `SELECT ... SAMPLE`
- 索引
- 副本

## Null[¶](https://clickhouse.yandex/docs/zh/single/#null_1 "Permanent link")

当写入 Null 类型的表时，将忽略数据。从 Null 类型的表中读取时，返回空。

但是，可以在 Null 类型的表上创建物化视图。写入表的数据将转发到视图中。

## Set[¶](https://clickhouse.yandex/docs/zh/single/#set_1 "Permanent link")

始终存在于 RAM 中的数据集。它适用于 IN 运算符的右侧（请参见 "IN 运算符" 部分）。

可以使用 INSERT 向表中插入数据。新元素将添加到数据集中，而重复项将被忽略。但是不能对此类型表执行 SELECT 语句。检索数据的唯一方法是在 IN 运算符的右半部分使用它。

数据始终存在于 RAM 中。对于 INSERT，插入数据块也会写入`disk`上的表目录。启动服务器时，此数据将加载到 RAM。也就是说，重新启动后，数据仍然存在。

对于强制服务器重启，`disk`上的数据块可能会丢失或损坏。在数据块损坏的情况下，可能需要手动删除包含损坏数据的文件。

## Join[¶](https://clickhouse.yandex/docs/zh/single/#join "Permanent link")

加载好的 JOIN 表数据会常驻内存中。

Join(ANY|ALL, LEFT|INNER, k1\[, k2, ...\])

引擎参数：`ANY|ALL` – 连接修饰；`LEFT|INNER` – 连接类型。更多信息可参考  [JOIN 子句](https://clickhouse.yandex/docs/zh/single/#select-join)。 这些参数设置不用带引号，但必须与要 JOIN 表匹配。 k1，k2，……是 USING 子句中要用于连接的关键列。

此引擎表不能用于 GLOBAL JOIN 。

类似于 Set 引擎，可以使用 INSERT 向表中添加数据。设置为 ANY 时，重复键的数据会被忽略（仅一条用于连接）。设置为 ALL 时，重复键的数据都会用于连接。不能直接对 JOIN 表进行 SELECT。检索其数据的唯一方法是将其作为 JOIN 语句右边的表。

跟 Set 引擎类似，Join 引擎把数据存储在`disk`中。

## URL(URL, Format)[¶](https://clickhouse.yandex/docs/zh/single/#table_engines-url "Permanent link")

用于管理远程 HTTP/HTTPS 服务器上的数据。该引擎类似  [File](https://clickhouse.yandex/docs/zh/single/#file/)  引擎。

### 在 ClickHouse 服务器中使用引擎[¶](https://clickhouse.yandex/docs/zh/single/#zai-clickhouse-fu-wu-qi-zhong-shi-yong-yin-qing "Permanent link")

`Format`  必须是 ClickHouse 可以用于  `SELECT`  查询的一种格式，若有必要，还要可用于  `INSERT` 。有关支持格式的完整列表，请查看  [Formats](https://clickhouse.yandex/docs/zh/single/#formats)。

`URL`  必须符合统一资源定位符的结构。指定的 URL 必须指向一个 HTTP 或 HTTPS 服务器。对于服务端响应， 不需要任何额外的 HTTP 头标记。

`INSERT`  和  `SELECT`  查询会分别转换为  `POST`  和  `GET`  请求。 对于  `POST`  请求的处理，远程服务器必须支持  [分块传输编码](https://en.wikipedia.org/wiki/Chunked_transfer_encoding)。

**示例：**

**1.**  在 Clickhouse 服务上创建一个  `url_engine_table`  表：

CREATE TABLE url_engine_table (word String, value UInt64)
ENGINE=URL('http://127.0.0.1:12345/', CSV)

**2.**  用标准的 Python 3 工具库创建一个基本的 HTTP 服务并 启动它：

from http.server import BaseHTTPRequestHandler, HTTPServer

class CSVHTTPServer(BaseHTTPRequestHandler):
def do_GET(self):
self.send_response(200)
self.send_header('Content-type', 'text/csv')
self.end_headers()

        self.wfile.write(bytes('Hello,1\\nWorld,2\\n', "utf-8"))

if \_\_name\_\_ == "\_\_main\_\_":
server_address = ('127.0.0.1', 12345)
HTTPServer(server_address, CSVHTTPServer).serve_forever()

python3 server.py

**3.**  查询请求:

SELECT \* FROM url_engine_table

┌─word──┬─value─┐
│ Hello │ 1 │
│ World │ 2 │
└───────┴───────┘

### 功能实现[¶](https://clickhouse.yandex/docs/zh/single/#gong-neng-shi-xian_1 "Permanent link")

- 读写操作都支持并发
- 不支持：
  - `ALTER`  和  `SELECT...SAMPLE`  操作。
  - 索引。
  - 副本。

## View[¶](https://clickhouse.yandex/docs/zh/single/#view "Permanent link")

用于构建视图（有关更多信息，请参阅  `CREATE VIEW 查询`）。 它不存储数据，仅存储指定的  `SELECT`  查询。 从表中读取时，它会运行此查询（并从查询中删除所有不必要的列）。

## 物化视图[¶](https://clickhouse.yandex/docs/zh/single/#wu-hua-shi-tu "Permanent link")

物化视图的使用（更多信息请参阅  [CREATE TABLE](https://clickhouse.yandex/docs/zh/single/#../../query_language/create/) ）。它需要使用一个不同的引擎来存储数据，这个引擎要在创建物化视图时指定。当从表中读取时，它就会使用该引擎。

## Memory[¶](https://clickhouse.yandex/docs/zh/single/#memory "Permanent link")

Memory 引擎以未压缩的形式将数据存储在 RAM 中。数据完全以读取时获得的形式存储。换句话说，从这张表中读取是很轻松的。并发数据访问是同步的。锁范围小：读写操作不会相互阻塞。不支持索引。阅读是并行化的。在简单查询上达到最大生产率（超过 10 GB /秒），因为没有`disk`读取，不需要解压缩或反序列化数据。（值得注意的是，在许多情况下，与 MergeTree 引擎的性能几乎一样高）。重新启动服务器时，表中的数据消失，表将变为空。通常，使用此`table engines`是不合理的。但是，它可用于测试，以及在相对较少的行（最多约 100,000,000）上需要最高性能的查询。

Memory 引擎是由系统用于临时表进行外部数据的查询（请参阅 "外部数据用于请求处理" 部分），以及用于实现  `GLOBAL IN`（请参见 "IN 运算符" 部分）。

## Buffer[¶](https://clickhouse.yandex/docs/zh/single/#buffer "Permanent link")

缓冲数据写入 RAM 中，周期性地将数据刷新到另一个表。在读取操作时，同时从缓冲区和另一个表读取数据。

Buffer(database, table, num_layers, min_time, max_time, min_rows, max_rows, min_bytes, max_bytes)

引擎的参数：database，table - 要刷新数据的表。可以使用返回字符串的常量表达式而不是数据库名称。 num_layers - 并行层数。在物理上，该表将表示为 num_layers 个独立缓冲区。建议值为 16。min_time，max_time，min_rows，max_rows，min_bytes，max_bytes - 从缓冲区刷新数据的条件。

如果满足所有 "min" 条件或至少一个 "max" 条件，则从缓冲区刷新数据并将其写入目标表。min_time，max_time — 从第一次写入缓冲区时起以秒为单位的时间条件。min_rows，max_rows - 缓冲区中行数的条件。min_bytes，max_bytes - 缓冲区中字节数的条件。

写入时，数据从 num_layers 个缓冲区中随机插入。或者，如果插入数据的大小足够大（大于 max_rows 或 max_bytes ），则会绕过缓冲区将其写入目标表。

每个 "num_layers" 缓冲区刷新数据的条件是分别计算。例如，如果 num_layers = 16 且 max_bytes = 100000000，则最大 RAM 消耗将为 1.6 GB。

示例：

CREATE TABLE merge.hits_buffer AS merge.hits ENGINE = Buffer(merge, hits, 16, 10, 100, 10000, 1000000, 10000000, 100000000)

创建一个 "merge.hits_buffer" 表，其结构与 "merge.hits" 相同，并使用 Buffer 引擎。写入此表时，数据缓冲在 RAM 中，然后写入 "merge.hits" 表。创建了 16 个缓冲区。如果已经过了 100 秒，或者已写入 100 万行，或者已写入 100 MB 数据，则刷新每个缓冲区的数据；或者如果同时已经过了 10 秒并且已经写入了 10,000 行和 10 MB 的数据。例如，如果只写了一行，那么在 100 秒之后，都会被刷新。但是如果写了很多行，数据将会更快地刷新。

当服务器停止时，使用 DROP TABLE 或 DETACH TABLE，缓冲区数据也会刷新到目标表。

可以为数据库和表名在单个引号中设置空字符串。这表示没有目的地表。在这种情况下，当达到数据刷新条件时，缓冲器被简单地清除。这可能对于保持数据窗口在内存中是有用的。

从 Buffer 表读取时，将从缓冲区和目标表（如果有）处理数据。 请注意，Buffer 表不支持索引。换句话说，缓冲区中的数据被完全扫描，对于大缓冲区来说可能很慢。（对于目标表中的数据，将使用它支持的索引。）

如果 Buffer 表中的列集与目标表中的列集不匹配，则会插入两个表中存在的列的子集。

如果类型与 Buffer 表和目标表中的某列不匹配，则会在服务器日志中输入错误消息并清除缓冲区。 如果在刷新缓冲区时目标表不存在，则会发生同样的情况。

如果需要为目标表和 Buffer 表运行 ALTER，我们建议先删除 Buffer 表，为目标表运行 ALTER，然后再次创建 Buffer 表。

如果服务器异常重启，缓冲区中的数据将丢失。

PREWHERE，FINAL 和 SAMPLE 对缓冲表不起作用。这些条件将传递到目标表，但不用于处理缓冲区中的数据。因此，我们建议只使用 Buffer 表进行写入，同时从目标表进行读取。

将数据添加到缓冲区时，其中一个缓冲区被锁定。如果同时从表执行读操作，则会导致延迟。

插入到 Buffer 表中的数据可能以不同的顺序和不同的块写入目标表中。因此，Buffer 表很难用于正确写入 CollapsingMergeTree。为避免出现问题，您可以将 "num_layers" 设置为 1。

如果目标表是复制表，则在写入 Buffer 表时会丢失复制表的某些预期特征。数据部分的行次序和大小的随机变化导致数据不能去重，这意味着无法对复制表进行可靠的 "exactly once" 写入。

由于这些缺点，我们只建议在极少数情况下使用 Buffer 表。

当在单位时间内从大量服务器接收到太多 INSERTs 并且在插入之前无法缓冲数据时使用 Buffer 表，这意味着这些 INSERTs 不能足够快地执行。

请注意，一次插入一行数据是没有意义的，即使对于 Buffer 表也是如此。这将只产生每秒几千行的速度，而插入更大的数据块每秒可以产生超过一百万行（参见 "性能" 部分）。
