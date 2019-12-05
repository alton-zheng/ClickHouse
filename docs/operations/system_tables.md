# System tables

`System tables`用于实现系统的部分功能，并提供对有关系统如何工作的信息的访问。您不能删除`System tables`(但可以执行分离)。`System tables`没有磁盘上的数据文件或元数据文件。服务器在启动时创建所有`System tables`。`System tables`是只读的。它们位于`system`数据库中。

- `system.asynchronous_metrics` :   包含在后台定期计算的指标。例如，正在使用的RAM的数量。
- `system.clusters` :  包含配置文件中可用的集群和其中的服务器的信息。
- `system.columns` :  包含有关所有表中列的信息
- `system.contributors` :  包含有关贡献者的信息。所有贡献者以随机顺序排列。该顺序在`query`执行时是随机的。 Column,  metric(String)   value(Float64)
- `system.databases` :  包含服务中所有 databases .  name(String)
- `system.detached_parts`: 包含关于合并树表的分离部分的信息。
- `system.dictionaries` :包含关于外部字典的信息
- `System.events`:包含关于系统中发生的事件数量的信息。例如，在表中，您可以找到自ClickHouse服务器启动以来处理了多少SELECT查询。
- `System.functions`:包含关于普通函数和聚合函数的信息。
- `system.graphite_retentions`:包含有关参数graphite_rollup的信息，这些参数在带有*GraphiteMergeTree引擎的表中使用。
- `System.merges`:包含有关合并和部分突变的信息，目前用于处理中 MergeTree 家族的表。
- `System.metrics`:包含可以立即计算或具有当前值的度量。例如，同时处理的查询数或当前副本延迟。这张桌子总是最新的。
- `System.numbers`: `number(UInt64)` ，其中几乎包含所有从零开始的自然数。您可以使用此表进行测试，或者如果需要进行蛮力搜索，也可以使用此表。对该表的读取不是并行的。 
- `System.numbers_mt`: `number(UInt64)` , 与'system.numbers'相同，但读取是并行的。可以按任何顺序返回数字。用于测试。
- `System.one`:  `dummy(UInt8)`: 该表仅有一行数据， 值为0。如果 SELECT query未指定 FROM 子句，则使用此表。这类似于在其他 DBMS 中找到的 DUAL 表。
- `System.parts`: 包含有关 MergeTree 表各 part 的信息。
- `System.part_log`: 此表包含关于 MergeTree 族表中data part发生的事件的信息，例如添加或合并数据。
- `System.processes`: 该系统表用于实现SHOW PROCESSLIST 请求
- `system.query_log`： 包含有关query执行的信息。对于每个query，您可以看到处理开始时间、处理持续时间、错误消息和其他信息。
- `System.replicas`: 包含驻留在本地服务器上的 replicated table 的信息和状态。此表可用于监视。该表包含为每个Replicated*表记录的一行信息。
- `System.settings`: 包含有关当前正在使用的设置的信息。例如，用于执行用来你从 system.settings 表读查询。
- `System.tables`: 包含服务器知道的每个表的元数据。detached(分离)的表未显示在中system.tables。
- `system.zookeeper`: 如果没有配置ZooKeeper，则该表不存在 ,允许从配置中定义的 ZooKeeper 集群读取数据。查询必须在WHERE子句中有一个 "path" 条件。这是你想要获取数据的 children 在 ZooKeeper 中的路径。
- `system.mutations`: 有关 MergeTree 表的mutation及其进度的信息。每个mutation命令由单行表示
- `System.disks`: 包含有关服务器配置中定义的磁盘的信息。
- `system.storage_policies`: 包含有关服务器配置中定义的 `storage policy` 和 `volume` 的信息。

## system.asynchronous_metrics

包含在后台定期计算的指标。例如，正在使用的RAM的数量。

列：

- `metric`（`String`）-指标名称。
- `value`（`Float64`）-指标值。

**例**

```sql
SELECT * FROM system.asynchronous_metrics LIMIT 10
```

```
┌─metric──────────────────────────────────┬──────value─┐
│ jemalloc.background_thread.run_interval │          0 │
│ jemalloc.background_thread.num_runs     │          0 │
│ jemalloc.background_thread.num_threads  │          0 │
│ jemalloc.retained                       │  422551552 │
│ jemalloc.mapped                         │ 1682989056 │
│ jemalloc.resident                       │ 1656446976 │
│ jemalloc.metadata_thp                   │          0 │
│ jemalloc.metadata                       │   10226856 │
│ UncompressedCacheCells                  │          0 │
│ MarkCacheFiles                          │          0 │
└─────────────────────────────────────────┴────────────┘
```

## system.clusters

包含配置文件中可用的集群和其中的服务器的信息。

列：

- `cluster` （`String`）: 群集名称。
- `shard_num` （`UInt32`）: 集群中的分片号，从 1 开始。
- `shard_weight` （`UInt32`）:写入数据时分片的相对重量。
- `replica_num` （`UInt32`）:分片中的副本号，从 1 开始。
- `host_name` （`String`）:主机名，在配置中指定。
- `host_address` (`String`）:从 DNS 获得的主机 IP 地址。
- `port` （`UInt16`）:用于连接到服务器的端口。
- `user` （`String`）:用于连接到服务器的用户名。
- `errors_count` （`UInt32`）:该主机无法访问副本的次数。
- `estimated_recovery_time` （`UInt32`）: 直到副本错误计数为零为止还剩几秒，它被认为恢复了正常。

请注意，对于集群的每个`query``，`errors_count` 更新一次，但是 `estimated_recovery_time` 按需重新计算。因此，可能会出现非零 `errors_count` 和零 `estimated_recovery_time` 的情况，下一个`query`将为零 `errors_count` 并尝试使用副本，就好像它没有错误一样。

## system.columns

包含有关所有表中列的信息。

您可以使用此表来获取与 `DESCRIBE TABLE` `query`类似的信息，但是一次可以获取多个表。

该`system.columns`表包含以下列（列类型显示在方括号中）：

- `database` （`String`）: 数据库名称。
- `table` （`String`）: 表名。
- `name` （`String`）: 列名。
- `type` （`String`）: 列类型。
- `default_kind`（`String`） : 表达类型（`DEFAULT`，`MATERIALIZED`，`ALIAS`）作为默认值，或者如果没有定义它为空`String`。
- `default_expression` （`String`）: 默认值的表达式；如果未定义，则为空`String`。
- `data_compressed_bytes` （UInt64）: 压缩数据的大小，以字节为单位。
- `data_uncompressed_bytes` （UInt64）: 解压缩数据的大小，以字节为单位。
- `marks_bytes` （UInt64）: 标记的大小，以字节为单位。
- `comment` （String）: 对列进行注释，如果未定义，则为空`String`。
- `is_in_partition_key` （UInt8）: 标志，指示列是否在分区表达式中。
- `is_in_sorting_key` （UInt8）: 标志，指示列是否在排序键表达式中。
- `is_in_primary_key` （UInt8）: 标志，指示列是否在`PK`表达式中。
- `is_in_sampling_key` （UInt8）: 标志，指示列是否在采样关键字表达式中。

## system.contributors

包含有关贡献者的信息。所有贡献者以随机顺序排列。该顺序在`query`执行时是随机的。

列：

- `name` （ `String` ）—来自 git log 的`Contributor`（author）名称。

**例**

```sql
SELECT * FROM system.contributors LIMIT 10
```

```
┌─name─────────────┐
│ Olga Khvostikova │
│ Max Vetrov       │
│ LiuYangkuan      │
│ svladykin        │
│ zamulla          │
│ Šimon Podlipský  │
│ BayoNet          │
│ Ilya Khomutov    │
│ Amy Krishnevsky  │
│ Loud_Scream      │
└──────────────────┘
```



要在表中查找 `Olga Khvostikova`，请使用`query`：

```
SELECT * FROM system.contributors WHERE name='Olga Khvostikova'
┌─name─────────────┐
│ Olga Khvostikova │
└──────────────────┘
```

## system.databases

只有一个`String` 类型字段： `name` 实现`SHOW DATABASES``query`的底层表。

## system.detached_parts

包含关于 `MergeTree` 表的分离部分的信息。`reason` 列指定了部分分离的原因。对于用户分离的部分，原因是空的。这些 `part` 可以用 `ALTER TABLE ATTACH PARTITION|PART` 命令。有关其他列的描述，请参见 `system.parts` 。如果 `part` 名称无效，某些列的值可能为空。可以使用 `ALTER TABLE DROP DETACHED PART` 命令删除这些`part`。

## system.dictionaries

包含关于外部字典的信息。

列：

- `name` （String）: 字典名称。
- `type` （String）: 字典类型：Flat，Hashed，Cache。
- `origin` （String）: 描述字典的配置文件的路径。
- `attribute.names` （Array（String））: 字典提供的属性名称的Array。
- `attribute.types` （Array（String））: 字典提供的相应属性类型Array。
- `has_hierarchy` （UInt8）: 字典是否为分层字典。
- `bytes_allocated` （UInt64）: 词典使用的 RAM 数量。
- `hit_rate` （Float64）: 对于缓存字典，该值在缓存中的使用百分比。
- `element_count` （UInt64）: 词典中存储的项目数。
- `load_factor` （Float64）: 字典中填充的百分比（对于哈希字典，填充在哈希表中的百分比）。
- `creation_time` （DateTime）: 创建字典或最后一次成功重新加载字典的时间。
- `last_exception` （String）: 如果无法创建字典，则在创建或重新加载字典时发生的错误的文本。
- `source` （String）: 描述字典数据源的文本。

请注意，字典使用的内存量与字典中存储的项的数量不成比例。因此，对于平面字典和缓存字典，所有的内存单元都是预先分配的，不管字典内容完整程度。

## system.events

包含有关系统中发生的事件数的信息。例如，在表中，您可以找到`SELECT`自 ClickHouse 服务器启动以来已处理的`query`数量。

列：

- `event`（String）-事件名称。
- `value`（UInt64）-发生的事件数。
- `description`（String）—事件描述。

**例**

```log
SELECT * FROM system.events LIMIT 5

┌─event─────────────────────────────────┬─value─┬─description────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Query                                 │    12 │ Number of queries to be interpreted and potentially executed. Does not include queries that failed to parse or were rejected due to AST size limits, quota limits or limits on the number of simultaneously running queries. May include internal queries initiated by ClickHouse itself. Does not count subqueries.                  │
│ SelectQuery                           │     8 │ Same as Query, but only for SELECT queries.                                                                                                                                                                                                                │
│ FileOpen                              │    73 │ Number of files opened.                                                                                                                                                                                                                                    │
│ ReadBufferFromFileDescriptorRead      │   155 │ Number of reads (read/pread) from a file descriptor. Does not include sockets.                                                                                                                                                                             │
│ ReadBufferFromFileDescriptorReadBytes │  9931 │ Number of bytes read from file descriptors. If the file is compressed, this will show the compressed data size.                                                                                                                                            │
└───────────────────────────────────────┴───────┴────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

## system.functions

包含有关一般和聚合函数的信息。

列：

- `name`（`String`）–函数的名称。
- `is_aggregate`（`UInt8`）—该函数是否聚合。

## system.graphite_retentions

包含有关参数 `graphite_rollup` 的信息，这些参数在带有 `*GraphiteMergeTree` 引擎的表中使用。

列：

- `config_name`（`String`）: `graphite_rollup` 参数名称。
- `regexp` （String）: 指标名称的模式。
- `function` （String）: 聚合函数的名称。
- `age` （UInt64）: 数据的最小生存时间（以秒为单位）。
- `precision` （UInt64）: 如何精确定义数据的生存时间（以秒为单位）。
- `priority` （UInt16）: 模式优先级。
- `is_default` （UInt8）: 模式是否为默认模式。
- `Tables.database`（Array（String））: 使用该`config_name`参数的数据库表的名称的Array。
- `Tables.table`（Array（String））: 使用该`config_name`参数的表名的Array。

## system.merges

包含有关 `MergeTree` 系列表中当前正在处理的 `merge` 和`part mutation`的信息。

列：

- `database` （`String`）: 表所在的数据库的名称。
- `table` （`String`）: 表名。
- `elapsed` （`Float64`）: 自合并开始以来经过的时间（以秒为单位）。
- `progress` （`Float64`）: 已完成工作从 0 到 1 的百分比。
- `num_parts` （`UInt64`）: 要合并的 `parts` 数。
- `result_part_name` （`String`）: 合并结果将形成的`part`的名称。
- `is_mutation` （UInt8）: 如果此过程是`part mutation`，则为 1。
- `total_size_bytes_compressed` （UInt64）: `merge` 块中压缩数据的总大小。
- `total_size_marks` （UInt64）: `merge` 部分中的标记总数。
- `bytes_read_uncompressed` （UInt64）: 读取的字节数，未压缩。
- `rows_read` （UInt64）: 读取的行数。
- `bytes_written_uncompressed` （UInt64）: 写入的未压缩字节数。
- `rows_written` （UInt64）: 写入的行数。

## system.metrics

包含可以立即计算或具有当前值的指标。例如，同时处理的`query`数或当前副本延迟。表状态实时更新。

列：

- `metric`（String）: 指标名称。
- `value`（Int64）: 指标值。
- `description`（String）: 指标描述。

在ClickHouse的 [dbms/src/Common/CurrentMetrics.cpp](https://github.com/ClickHouse/ClickHouse/blob/master/dbms/src/Common/CurrentMetrics.cpp) 源文件中可以找到支持的指标列表。

**例**

```log
SELECT * FROM system.metrics LIMIT 10
┌─metric─────────────────────┬─value─┬─description──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Query                      │     1 │ Number of executing queries                                                                                                                                                                      │
│ Merge                      │     0 │ Number of executing background merges                                                                                                                                                            │
│ PartMutation               │     0 │ Number of mutations (ALTER DELETE/UPDATE)                                                                                                                                                        │
│ ReplicatedFetch            │     0 │ Number of data parts being fetched from replicas                                                                                                                                                │
│ ReplicatedSend             │     0 │ Number of data parts being sent to replicas                                                                                                                                                      │
│ ReplicatedChecks           │     0 │ Number of data parts checking for consistency                                                                                                                                                    │
│ BackgroundPoolTask         │     0 │ Number of active tasks in BackgroundProcessingPool (merges, mutations, fetches, or replication queue bookkeeping)                                                                                │
│ BackgroundSchedulePoolTask │     0 │ Number of active tasks in BackgroundSchedulePool. This pool is used for periodic ReplicatedMergeTree tasks, like cleaning old data parts, altering data parts, replica re-initialization, etc.   │
│ DiskSpaceReservedForMerge  │     0 │ Disk space reserved for currently running background merges. It is slightly more than the total size of currently merging parts.                                                                     │
│ DistributedSend            │     0 │ Number of connections to remote servers sending data that was INSERTed into Distributed tables. Both synchronous and asynchronous mode.                                                          │
└────────────────────────────┴───────┴──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

## system.numbers

- `number`(`UInt64`)，其中几乎包含所有从零开始的自然数。您可以使用此表进行测试，或者如果需要进行蛮力搜索，也可以使用此表。对该表的读取不是并行的。

## system.numbers_mt

- `number`(`UInt64`), 与'system.numbers'相同，但读取是并行的。可以按任何顺序返回数字。用于测试。

## system.one

- `dummy`(UInt8): 该表仅有一行数据， 值为0。如果 `SELECT` `query`未指定 `FROM` 子句，则使用此表。这类似于在其他 `DBMS` 中找到的 `DUAL` 表。 

## system.parts

包含有关 `MergeTree` 表各 `part` 的信息。

每一行描述一个数据 `part`。

列：

- `partition`（String）–分区名称。要了解什么是分区，请参阅`ALTER``query`的描述
  - `YYYYMM`  用于按月自动分区。
  - `any_string`  手动分区时。
  
- `name`（`String`）: 数据 `part` 的名称。
- `active`（`UInt8`）: 指示数据 `part` 是否处于活动状态的标志。如果`data part`处于活动状态，则会在表中使用它。否则，将其删除。合并后，不活动的`data part`仍然保留。
- `marks`（`UInt64`）: 标记数。要获得`data part`中大约的行数，请乘以 `marks` 索引粒度（通常为 8192）（此提示不适用于自适应粒度）。
- `rows`（`UInt64`）: 行数。
- `bytes_on_disk`（`UInt64`）: 所有`data part`文件的总大小（以字节为单位）。
- `data_compressed_bytes`（`UInt64`）: `data part`中压缩数据的总大小。不包括所有辅助文件（例如，带有标记的文件）。
- `data_uncompressed_bytes`（`UInt64`）: `data part`中未压缩数据的总大小。不包括所有辅助文件（例如，带有标记的文件）。
- `marks_bytes`（`UInt64`）: 带标记的文件的大小。
- `modification_time`（`DateTime`）: 包含`data part`的目录被修改的时间。这通常对应于`data part`创建的时间。
- `remove_time`（`DateTime`）: `data part`变为非活动状态的时间。
- `refcount`（`UInt32`）: 使用`data part`的位置数。大于 2 的值表示在`query`或合并中使用了`data part`。
- `min_date`（`Date`）: `data part`中日期键的最小值。
- `max_date`（`Date`）: `data part`中日期键的最大值。
- `min_time`（`DateTime`）: `data part`中日期和时间键的最小值。
- `max_time`（`DateTime`）: `data part`中日期和时间键的最大值。
- `partition_id`（`String`）: 分区的 ID。
- `min_block_number`（`UInt64`）: 合并后组成当前部分的`data part`的最小数量。
- `max_block_number`（`UInt64`）: 合并后组成当前部分的最大`data part`数。
- `level`（`UInt32`）: 合并树的深度。零表示当前`part`是通过插入而不是通过合并其他`part`来创建的。
- `data_version`（`UInt64`）: 用于确定应将哪些`mutation`应用于`data part`（版本高于的`mutation`data_version`）的编号。
- `primary_key_bytes_in_memory`（`UInt64`）: `PK`值使用的内存量（以字节为单位）。
- `primary_key_bytes_in_memory_allocated`（`UInt64`）: 为`PK`值保留的内存量（以字节为单位）。
- `is_frozen`（`UInt8`）: 表明是否存在分区数据备份的标志。1、备份存在。0，备份不存在。有关详细信息，请参见 `FREEZE PARTITION`
- `database`（`String`）: 数据库名称。
- `table`（`String`）: 表名称。
- `engine`（`String`）： 不带参数的 `table engines` 的名称。
- `path`（`String`）: 包含 `data part` 文件的文件夹的绝对路径。
- `hash_of_all_files`（`String`）:   压缩文件的 `sipHash128`
- `hash_of_uncompressed_files`（`String`）:   未压缩文件（带有标记的文件，索引文件等）的 `sipHash128`
- `uncompressed_hash_of_compressed_files`（`String`）:   压缩文件中的数据的 `sipHash128`，就好像它们是未压缩的一样。
- `bytes`（`UInt64`）: `bytes_on_disk` 的 别名
- `marks_size`（`UInt64`）:`marks_bytes` 的别名

## system.part_log

`system.part_log`仅当指定[part_log](server_configuration_parameters.md)服务器设置时，才创建该表。

此表包含关于 `MergeTree` 族表中`data part`发生的事件的信息，例如添加或合并数据。

该`system.part_log`表包含以下列：

- `event_type`（`Enum`）: `data part`发生的事件的类型。可以具有以下值之一：
  - `NEW_PART` : 插入新的`data part`。
  - `MERGE_PARTS` : 合并`data part`。
  - `DOWNLOAD_PART` : 下载`data part`。
  - `REMOVE_PART`: 使用[DETACH PARTITION](https://clickhouse.yandex/docs/en/query_language/alter/#alter_detach-partition)删除或分离`data part`。
  - `MUTATE_PART` : 修改`data part`。
  - `MOVE_PART` : 将`data part`从一个磁盘移动到另一磁盘。
- `event_date` （Date）: 活动日期。
- `event_time` （DateTime）: 事件时间。
- `duration_ms` （UInt64）: 持续时间。
- `database` （String）: `data part`所在的数据库的名称。
- `table` （String）: `data part`所在的表的名称。
- `part_name` （String）: `data part`的名称。
- `partition_id`（String）: 数据部分插入到的分区的ID。如果分区是通过tuple()进行的，那么该列将接受'all'值。
- `rows` （UInt64）: `data part`中的行数。
- `size_in_bytes` （UInt64）: `data part`的大小，以字节为单位。
- `merged_from` （Array（String））: 组成当前`part`的`part`名称的Array（合并后）。
- `bytes_uncompressed` （UInt64）: 未压缩字节的大小。
- `read_rows` （UInt64）: 合并期间读取的行数。
- `read_bytes` （UInt64）: 合并期间读取的字节数。
- `error` （UInt16）: 发生的错误的代码号。
- `exception` （String）: 发生错误的文本消息。

该`system.part_log`表是在第一次向`MergeTree`表中插入数据之后创建的。

## system.processes

该系统表用于实现`SHOW PROCESSLIST` `query`。

列：

- `user`（String）: 进行`query`的用户。请记住，对于分布式处理，`query`将发送到`default`用户下的远程服务器。该字段包含特定`query`的用户名，而不是该`query`启动的`query`的用户名。
- `address`（String）: 发出请求的 IP 地址。分布式处理也是如此。要跟踪最初来自何处的分布式`query`，请在`system.processes``query`requestor server`上查看。
- `elapsed` （Float64）: 自请求开始执行以来的时间（以秒为单位）。
- `rows_read`（UInt64）: 从表中读取的行数。对于分布式处理，在`requestor server`上，这是所有远程服务器的总数。
- `bytes_read`（UInt64）: 从表中读取的未压缩字节数。对于分布式处理，在`requestor server`上，这是所有远程服务器的总数。
- `total_rows_approx`（UInt64）: 应该读取的总行数的近似值。对于分布式处理，在`requestor server`上，这是所有远程服务器的总数。当已知要处理的新资源时，可以在请求处理期间对其进行更新。
- `memory_usage`（UInt64）: 请求使用的 RAM 量。它可能不包括某些类型的专用内存。请参阅`max_memory_usage`设置。
- `query`（String）: `query`文本。对于`INSERT`，它不包含要插入的数据。
- `query_id` （String）: `query` ID（如果已定义）。

## system.query_log

包含有关`query`执行的信息。对于每个`query`，您可以看到处理开始时间、处理持续时间、错误消息和其他信息。

注意

该表不包含`INSERT``query`的输入数据。

只有指定了[query_log]()服务器参数，才会创建此表。此参数设置日志记录规则，例如日志记录间隔或将要登录`query`的表的名称。

要启用`query`日志记录，请将[log_queries]()参数设置为 1。有关详细信息，请参阅`设置`部分。

该`system.query_log`表注册两种`query`：

1.  客户端直接运行的初始`query``。
2.  由其他`query``（用于分布式`query`执行）启动的子`query`。对于这些类型的`query`，有关父`query`的信息显示在各`initial_*`列中。

列：

- `type`（`Enum8`）-执行`query`时发生的事件类型。值：
  - `'QueryStart' = 1` —成功开始`query`执行。
  - `'QueryFinish' = 2` —成功结束`query`执行。
  - `'ExceptionBeforeStart' = 3` —开始执行`query`之前的异常。
  - `'ExceptionWhileProcessing' = 4` —`query`执行期间的异常。
- `event_date` （Date）-活动日期。
- `event_time` （DateTime）-事件时间。
- `query_start_time` （DateTime）-`query`执行的开始时间。
- `query_duration_ms` （UInt64）-`query`执行的持续时间。
- `read_rows` （UInt64）-读取的行数。
- `read_bytes` （UInt64）-读取的字节数。
- `written_rows`（UInt64）-对于`INSERT``query`，为已写入的行数。对于其他`query``，列值为 0。
- `written_bytes`（UInt64）-对于`INSERT``query`，为写入字节数。对于其他`query``，列值为 0。
- `result_rows` （UInt64）-结果中的行数。
- `result_bytes` （UInt64）-结果中的字节数。
- `memory_usage` （UInt64）-`query`占用的内存。
- `query` （String）-`query``String`。
- `exception` （String）—异常消息。
- `stack_trace`（String）—堆栈跟踪(在发生错误之前调用的方法列表)。如果`query`成功完成，则为空`String`。
- `is_initial_query`（UInt8）-`query`类型。可能的值：
  - 1-`query`是由客户端发起的。
  - 0-由另一个`query`启动`query`以执行分布式`query`。
- `user` （`String`）—发起当前`query`的用户的名称。
- `query_id` （String）—`query`的 ID。
- `address` （FixedString（16））-从中发起`query`的 IP 地址。
- `port` （UInt16）-用于接收`query`的服务器端口。
- `initial_user` （String）—运行父`query`（用于分布式`query`执行）的用户名。
- `initial_query_id` （String）—父`query`的 ID。
- `initial_address` （FixedString（16））-从其启动父`query`的 IP 地址。
- `initial_port` （UInt16）-用于从客户端接收父`query`的服务器端口。
- `interface`（UInt8）-发起`query`的接口。可能的值：
  - 1-TCP。
  - 2-HTTP。
- `os_user` （`String`）-用户的操作系统。
- `client_hostname`（`String`）— `Clickhouse-client` 连接到的服务器名称。
- `client_name`（`String`）— `clickhouse-client` 名称。
- `client_revision`（UInt32）`clickhouse-client` 修订版。
- `client_version_major`（UInt32）`clickhouse-client` 的主要版本。
- `client_version_minor`（UInt32）`clickhouse-client` 的次要版本。
- `client_version_patch`（UInt32）`clickhouse-client` 版本的补丁组件。
- `http_method`（UInt8）-启动`query`的 HTTP 方法。可能的值：
  - 0-`query`是从 TCP 接口启动的。
  - 1- `GET`使用方法。
  - 2- `POST`使用方法。
- `http_user_agent`（`String`）— `UserAgent`在 HTTP 请求中传递的 `header`。
- `quota_key`（`String`）—在[Quatas]()设置中指定的`quotas key`。
- `revision` （UInt32）-ClickHouse 修订版。
- `thread_numbers` （Array（UInt32））-参与`query`执行的线程数。
- `ProfileEvents.Names` （Array（String））—衡量以下指标的计数器：
  - 通过网络进行读写所花费的时间。
  - 在读取和写入磁盘上花费的时间。
  - 网络错误数。
  - 网络带宽有限时等待所花费的时间。
- `ProfileEvents.Values`（Array（UInt64））- `ProfileEvents.Names`列中列出的指标值。
- `Settings.Names`（Array（`String`））-客户端运行`query`时更改的设置的名称。要启用对设置的日志记录更改，请将`log_query_settings`参数设置为 1。
- `Settings.Values`（Array（String））- `Settings.Names`列中列出的设置的值。

- 每个`query`在`query_log`表中创建一两行，具体取决于`query`的状态：
  - 如果查询执行成功，将创建两个类型为1和2的事件(请参阅`type` 列）。
  - 如果在查询处理期间发生错误，将创建两个类型为1和4的事件。
  - 如果在启动查询之前发生错误，则创建一个类型为3的单一事件。

默认情况下，每隔7.5秒向表添加一次日志。您可以在[query_log]()服务器设置中设置此间隔（请参阅`flush_interval_milliseconds`参数）。要将日志从内存缓冲区强行刷新到表中，请使用`SYSTEM FLUSH LOGS``query`。

手动删除表格后，将即时自动创建它。请注意，所有先前的日志将被删除。

注意

日志的存储期限不受限制。日志不会自动从表格中删除。您需要自己组织删除过时的日志。

您可以`system.query_log`在[query_log]()服务器设置中为表指定一个任意分区键（请参阅`partition_by`参数）。

## system.replicas

包含驻留在本地服务器上的 `replicated table` 的信息和状态。此表可用于监视。该表包含为每个Replicated*表记录的一行信息。

例：

```sql
SELECT *
FROM system.replicas
WHERE table = 'visits'
FORMAT Vertical
```

```log
Row 1:
──────
database:           merge
table:              visits
engine:             ReplicatedCollapsingMergeTree
is_leader:          1
is_readonly:        0
is_session_expired: 0
future_parts:       1
parts_to_check:     0
zookeeper_path:     /clickhouse/tables/01-06/visits
replica_name:       example01-06-1.yandex.ru
replica_path:       /clickhouse/tables/01-06/visits/replicas/example01-06-1.yandex.ru
columns_version:    9
queue_size:         1
inserts_in_queue:   0
merges_in_queue:    1
log_max_index:      596273
log_pointer:        596274
total_replicas:     2
active_replicas:    2
```

列：

```log
database:          
table:              Table name
engine:            Table engine name

is_leader:          Whether the replica is the leader.

Only one replica at a time can be the leader. The leader is responsible for selecting background merges to perform.
Note that writes can be performed to any replica that is available and has a session in ZK, regardless of whether it is a leader.

is_readonly:        Whether the replica is in read-only mode.
This mode is turned on if the config doesn't have sections with ZooKeeper, if an unknown error occurred when reinitializing sessions in ZooKeeper, and during session reinitialization in ZooKeeper.

is_session_expired: Whether the session with ZooKeeper has expired.
Basically the same as 'is_readonly'.

future_parts:       The number of data parts that will appear as the result of INSERTs or merges that haven't been done yet.

parts_to_check:    The number of data parts in the queue for verification.
A part is put in the verification queue if there is suspicion that it might be damaged.

zookeeper_path:     Path to table data in ZooKeeper.
replica_name:       Replica name in ZooKeeper. Different replicas of the same table have different names.
replica_path:      Path to replica data in ZooKeeper. The same as concatenating 'zookeeper_path/replicas/replica_path'.

columns_version:    Version number of the table structure.
Indicates how many times ALTER was performed. If replicas have different versions, it means some replicas haven't made all of the ALTERs yet.

queue_size:         Size of the queue for operations waiting to be performed.
Operations include inserting blocks of data, merges, and certain other actions.
It usually coincides with 'future_parts'.

inserts_in_queue:   Number of inserts of blocks of data that need to be made.
Insertions are usually replicated fairly quickly. If this number is large, it means something is wrong.

merges_in_queue:    The number of merges waiting to be made.
Sometimes merges are lengthy, so this value may be greater than zero for a long time.

The next 4 columns have a non-zero value only where there is an active session with ZK.

log_max_index:      Maximum entry number in the log of general activity.
log_pointer:        Maximum entry number in the log of general activity that the replica copied to its execution queue, plus one.
If log_pointer is much smaller than log_max_index, something is wrong.

total_replicas:     The total number of known replicas of this table.
active_replicas:    The number of replicas of this table that have a session in ZooKeeper (i.e., the number of functioning replicas).
```

如果您请求所有的列，那么表的查询速度可能会有点慢，因为每一行都需要从ZooKeeper读取几次数据。如果您没有请求最后4列(log_max_index、log_pointer、total_replicas、active_replicas)，那么该表可以快速工作。

例如，你可以像这样检查一切是否正常:

```sql
SELECT
    database,
    table,
    is_leader,
    is_readonly,
    is_session_expired,
    future_parts,
    parts_to_check,
    columns_version,
    queue_size,
    inserts_in_queue,
    merges_in_queue,
    log_max_index,
    log_pointer,
    total_replicas,
    active_replicas
FROM system.replicas
WHERE
       is_readonly
    OR is_session_expired
    OR future_parts > 20
    OR parts_to_check > 10
    OR queue_size > 20
    OR inserts_in_queue > 10
    OR log_max_index - log_pointer > 10
    OR total_replicas < 2
    OR active_replicas < total_replicas
```

如果此`query`不返回任何内容，则表示一切正常。

## system.settings

包含有关当前正在使用的设置的信息。例如，用于执行用来你从 `system.settings` 表读查询。

列:

- `name`(`String`)-设置的名称。
- `value`(`String`)-设置的值。
- `changed`(`UInt8`) -设置是在配置中显式定义还是显式更改。

例：

```sql
SELECT *
FROM system.settings
WHERE changed
```

```log
┌─name───────────────────┬─value───────┬─changed─┐
│ max_threads            │ 8           │       1 │
│ use_uncompressed_cache │ 0           │       1 │
│ load_balancing         │ random      │       1 │
│ max_memory_usage       │ 10000000000 │       1 │
└────────────────────────┴─────────────┴─────────┘
```

## system.tables

包含服务器知道的每个表的元数据。`detached`(分离)的表未显示在中`system.tables`。

- `database` （`String`）—表所在的数据库的名称。
- `name` （`String`）-表名。
- `engine` （`String`）—表引擎名称（不带参数）。
- `is_temporary` （UInt8）-指示表是否为临时的标志。
- `data_path` （`String`）-文件系统中表数据的路径。
- `metadata_path` （`String`）-文件系统中表元数据的路径。
- `metadata_modification_time` （DateTime）-表元数据的最新修改时间。
- `dependencies_database` （Array（`String`））-数据库依赖项。
- `dependencies_table`（Array（`String`））-表依赖关系（基于当前表的`MaterializedView` 表）。
- `create_table_query` （`String`）-用于创建表的`query`。
- `engine_full` （`String`）-表引擎的参数。
- `partition_key` （`String`）-表中指定的分区键表达式。
- `sorting_key` （`String`）-表中指定的排序键表达式。
- `primary_key` （`String`）-表中指定的主键表达式。
- `sampling_key` （`String`）-表中指定的采样键表达式。

该`system.tables`表用于`SHOW TABLES``query`实现。

## system.zookeeper

如果没有配置ZooKeeper，则该表不存在。允许从配置中定义的 `ZooKeeper` 集群读取数据。查询必须在WHERE子句中有一个 "path" 条件。这是你想要获取数据的 `children` 在 `ZooKeeper` 中的路径。

该`query` `SELECT * FROM system.zookeeper WHERE path = '/clickhouse'` 输出`/clickhouse`节点上所有子级的数据。要输出所有根节点的数据，请写入 path ='/'。如果在“路径”中指定的路径不存在，一个异常将被抛出。

列：

- `name` （`String`）-节点的名称。
- `path` （`String`）-节点的路径。
- `value` （`String`）-节点值。
- `dataLength` （Int32）-值的大小。
- `numChildren` （Int32）-后代的数量。
- `czxid` （Int64）-创建节点的事务的 ID。
- `mzxid` （Int64）-上次更改节点的事务的 ID。
- `pzxid` （Int64）-上次删除或添加后代的事务的 ID。
- `ctime` （DateTime）-节点创建的时间。
- `mtime` （DateTime）-节点上一次修改的时间。
- `version` （Int32）-节点版本：更改节点的次数。
- `cversion` （Int32）-添加或删除的后代的数量。
- `aversion` （Int32）-ACL 的更改次数。
- `ephemeralOwner` （Int64）-对于临时节点，拥有该节点的会话的 ID。

例：

```sql
SELECT *
FROM system.zookeeper
WHERE path = '/clickhouse/tables/01-08/visits/replicas'
FORMAT Vertical
```

```log
Row 1:
──────
name:           example01-08-1.yandex.ru
value:
czxid:          932998691229
mzxid:          932998691229
ctime:          2015-03-27 16:49:51
mtime:          2015-03-27 16:49:51
version:        0
cversion:       47
aversion:       0
ephemeralOwner: 0
dataLength:     0
numChildren:    7
pzxid:          987021031383
path:           /clickhouse/tables/01-08/visits/replicas

Row 2:
──────
name:           example01-08-2.yandex.ru
value:
czxid:          933002738135
mzxid:          933002738135
ctime:          2015-03-27 16:57:01
mtime:          2015-03-27 16:57:01
version:        0
cversion:       37
aversion:       0
ephemeralOwner: 0
dataLength:     0
numChildren:    7
pzxid:          987021252247
path:           /clickhouse/tables/01-08/visits/replicas
```

## system.mutations

该表包含有关 MergeTree 表的[mutation](https://clickhouse.yandex/docs/en/query_language/alter/#alter-mutations)及其进度的信息。每个`mutation`命令由单行表示。该表包括以下列：

- **database**,**table** -数据库和表，施加 `mutation` 的名称。
- **mutation_id** : `mutation id`, 对于复制的表，这些 ID 对应于 ZooKeeper  中 `<table_path_in_zookeeper>/mutations/` 目录中的 znode 名称。对于未复制的表，ID 对应于表的数据目录中的文件名。
- **command**: mutation 命令`String`（`ALTER TABLE [db.]table`后面的`query`部分）。
- **create_time**： 当此突变命令被提交执行时。
- **block_numbers.partition_id**，**block_numbers.number** ： 嵌套列。对于复制表的突变，它为每个分区包含一条记录:由突变获得的分区ID和块号(在每个分区中，只有包含小于该分区中突变获得的块号的块的部分会发生突变)。在非复制表中，所有分区中的块号形成一个单独的序列。这意味着对于非复制表的突变，该列将包含一条记录，其中包含突变获得的单个块号。
- **parts_to_do**： 完成`mutation`需要`mutation`的`data part`的数量。
- **is_done**: `mutation`完成了吗？请注意，即使`parts_to_do = 0`由于长期运行 INSERT 可能会导致复制表的`mutation`尚未完成，这也会创建需要`mutation`的新`data part`。

如果对某些 `parts` 进行更改存在问题，则以下各列包含其他信息：

**Latest_failed_part**: 不能`mutation`的最新`part`的名称。

**last_fail_time**: 最近的`parts mutation`失败的时间。

**Latest_fail_reason**: 导致最近的`parts mutation`失败的异常消息。

## system.disks

包含有关服务器配置中定义的磁盘的信息。

列：

- `name`(String) -服务器配置中的磁盘名称。
- `path`(String) —文件系统中安装点的路径。
- `free_space`(`UInt64`) —磁盘上的可用空间（以字节为单位）。
- `total_space`(`UInt64`)-磁盘`volume`，以字节为单位。
- `keep_free_space`(`UInt64`)-应该在磁盘上保持空闲的磁盘空间量（以字节为单位）。在`keep_free_space_bytes`磁盘配置参数中定义。

## system.storage_policies

包含有关服务器配置中定义的`storage policy`和`volume`的信息。

列：

- `policy_name`(`String`)—`storage policy`的名称。
- `volume_name`(`String`)-在`storage policy`中定义的`volume`名称。
- `volume_priority`(`UInt64`)- `volume` 在 配置文件中定义的`order number`。
- `disks`(`Array(String)`)-磁盘名称，在`storage policy`中定义。
- `max_data_part_size`(`UInt64`)-可以存储在`volume`磁盘上的`data part`的最大大小（0-无限制）。
- `move_factor`(`Float64`)-空闲磁盘空间的比率。当比率超过配置参数的值时，ClickHouse开始按顺序将数据移动到下一个`volume`。

如果`storage policy`包含一个以上的`volume`，则每个`volume`的信息都存储在表的单独行中。
