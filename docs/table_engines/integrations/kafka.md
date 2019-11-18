## Kafka

此引擎与  [Apache Kafka](http://kafka.apache.org/)  结合使用。

Kafka 特性：

- 发布或者订阅数据流。
- 容错存储机制。
- 在流可用时对其进行处理。

---

### Creating a table

```sql
CREATE TABLE [IF NOT EXISTS] [db.]table_name [ON CLUSTER cluster]
(
    name1 [type1] [DEFAULT|MATERIALIZED|ALIAS expr1],
    name2 [type2] [DEFAULT|MATERIALIZED|ALIAS expr2],
    ...
) ENGINE = Kafka()
SETTINGS
    kafka_broker_list = 'host:port',
    kafka_topic_list = 'topic1,topic2,...',
    kafka_group_name = 'group_name',
    kafka_format = 'data_format'[,]
    [kafka_row_delimiter = 'delimiter_symbol',]
    [kafka_schema = '',]
    [kafka_num_consumers = N,]
    [kafka_skip_broken_messages = N]
```

**必要参数**：

- `kafka_broker_list` : 以 `comma-separated` 的 brokers 列表 (`localhost:9092`)
- `kafka_topic_list` : topic 列表 (`my_topic`)
- `kafka_group_name` : 
  - `Kafka` 消费者组名称。
  - 每组的阅读空白处都被单独记录。
  - 如果不希望消息在集群中重复，请在任何地方使用相同的组名。
- `kafka_format` – 消息格式。使用与 SQL 部分的  `FORMAT`  函数相同表示方法，例如  `JSONEachRow`。了解详细信息，请参考  `Formats`  部分。

可选参数：

- `kafka_row_delimiter` : 消息之间的分隔字符，用来结束一条消息。
- `kafka_schema` : 如果解析格式需要一个 schema 时，此参数必填。例如，[Cap'n Proto](https://capnproto.org/)  需要 schema 文件路径以及根对象  `schema.capnp:Message`  的名字。
- `kafka_num_consumers` : 单个表的消费者数量。默认值是：`1`，如果一个消费者的吞吐量不足，则指定更多的消费者。消费者的总数不应该超过 topic 中分区的数量，因为每个分区只能分配一个消费者。
- `kafka_skip_broken_messages` : 
  - 每个 `block` 对与 `schema-incompatible`(`schema` 不兼容) 的消息的Kafka消息解析器的容忍行数。
  - 默认值:0
  - 如果 `kafka_skip_broken_messages = N`，那么引擎将跳过N个无法解析的Kafka消息(消息等于一行数据)。


示例：

```sql
 CREATE TABLE queue (
    timestamp UInt64,
    level String,
    message String
  ) ENGINE = Kafka('localhost:9092', 'topic', 'group1', 'JSONEachRow');

  SELECT * FROM queue LIMIT 5;

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
```

---

### Description

消费的消息会被自动追踪，因此每个消息在不同的消费组里只会记录一次。如果希望获得两次数据，则使用另一个组名创建副本。

- 消费组可以灵活配置并且在集群之间同步。
  - 例如，如果群集中有 10 个 `topic` 和 5 个表副本，则每个副本将获得 2 个`topic`。 
  - 如果副本数量发生变化，`topic` 将自动在副本中重新分配。
  - 了解更多信息请访问  [http://kafka.apache.org/intro](http://kafka.apache.org/intro)。

`SELECT`  查询对于读取消息并不是很有用（调试除外），因为每条消息只能被读取一次。使用物化视图创建实时线程更实用。您可以这样做：
  - 使用该引擎创建Kafka消费者，并将其视为数据流。
  - 创建具有所需结构的表。
  - 创建物化视图，将数据从引擎转换到先前创建的表中。

- 当  `MATERIALIZED VIEW`  添加至引擎，它将会在后台收集数据。
  - 可以持续不断地从 Kafka 收集数据并通过  `SELECT`  将数据转换为所需要的格式。
  - 一个kafka表可以有任意多的`MATERIALIZED VIEW`，它们不直接从kafka表中读取数据，而是接收新记录(以 `block` 的形式)，这样你就可以写几个不同详细级别的表(有分组-聚合，没有分组-聚合)。
示例：

```sql
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
```

为了提高性能，接受的消息被分组为  `max_insert_block_size`(见设置)  大小的 `block`。如果未在  `stream_flush_interval_ms`  毫秒内形成 `block` ，则不关心块的完整性，都会将数据刷新到表中。

停止接收 `topic` 数据或更改转换逻辑，请 detach 物化视图：

```sql
DETACH TABLE consumer;
ATTACH MATERIALIZED VIEW consumer;
```

如果使用  `ALTER`  更改目标表，为了避免目标表与视图中的数据之间存在差异，推荐停止 `material view`

---

### Configuration

与  `GraphiteMergeTree`  类似，Kafka 引擎支持使用 ClickHouse 配置文件进行扩展配置。
- 可以使用两个配置键：
  - 全局 (`kafka`) 
  - 主题级别 (`kafka_*`)。
  - 首先应用全局配置，然后应用主题级配置（如果存在）。

```xml
<!-- Global configuration options for all tables of Kafka engine type -->
  <kafka>
    <debug>cgrp</debug>
    <auto_offset_reset>smallest</auto_offset_reset>
  </kafka>

  <!-- Configuration specific for topic "logs" -->
  <kafka_logs>
    <retry_backoff_ms>250</retry_backoff_ms>
    <fetch_min_bytes>100000</fetch_min_bytes>
  </kafka_logs>
```

有关详细配置选项列表，请参阅  [librdkafka configuration reference](https://github.com/edenhill/librdkafka/blob/master/CONFIGURATION.md)。

- 在 ClickHouse 配置中使用下划线 (`_`) ，并不是使用点 (`.`)。
  - 例如，`check.crcs=true`  将是  `<check_crcs>true</check_crcs>`。

---

### Virtual Columns

- `_topic`: Kafka topic.
- `_key` : 消息的`key`.
- `_offset` : 消息的 `offset`.
- `_timestamp` : 消息的 `timestamp`
- `_partition` : `Kafka topic`的 `Partition`.

[Virtual columns](../../table_engines/table_engines_introduction.md)