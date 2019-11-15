# Settings

## distributed_product_mode

更改 `分布式子查询` 行为。

当查询包含分布式表的 `product` 时，即当分布式表的查询包含分布式表的 `non-GLOBAL` 子查询时，ClickHouse 将应用此设置。

限制条件：

- 仅适用于 `IN` 和 `JOIN` 子查询。
- 仅当 `FROM` 部分使用包含多个 `shard` 的分布式表时。
- 如果子查询涉及一个包含多个 `shard` 的分布式表。
- 不为 `table-value` 用 `remote` 函数。

可能的值：

- `deny`: 默认值。禁止使用这些类型的子查询（返回 `Double-distributed in/JOIN subqueries is denied` 异常）。
- `local`: 将子查询中的数据库和表替换为目标服务器(`shard`)的本地数据库和表，保留正常的`IN`/`JOIN`.
- `global`: 将`IN`/ `JOIN`查询替换为`GLOBAL IN`/`GLOBAL JOIN.`
- `allow` : 允许使用这些类型的子查询。

## enable_optimize_predicate_expression

- 打开`SELECT`查询中的谓词下推。
  - 谓词下推可能会大大减少分布式查询的网络流量。
    - `0` : 禁用。
    - `1` : 启用。

默认值：`1`

**用法**

考虑以下查询：

1.  `SELECT count() FROM test_table WHERE date = '2018-10-10'`
2.  `SELECT count() FROM (SELECT * FROM test_table) WHERE date = '2018-10-10'`

如果为`enable_optimize_predicate_expression = 1`，`2` 和 `1` 执行计划相同，效率一致

## fallback_to_stale_replicas_for_distributed_queries

- 如果更新的数据不可用，则将查询强制到过期的`replicas`。[Replication](../../table_engines/merge-tree-family/data-replication.md)。
  - 从表的过期`replicas`中选择最相关的`replicas`。
  - 当从指向 `replicated table` 的分布式表中执行 `SELECT` 时使用。

默认情况下，1(启用)。

## force_index_by_date

如果无法按日期使用索引，则禁用查询执行。

与 MergeTree 系列中的表一起使用。

- `0`: 禁用
- `1`: 启用

官文没有说明默认值，以后遇到再补充

## force_primary_key

如果无法通过 `PK` 建立索引，则禁用查询执行。

与 MergeTree 系列中的表一起使用。

- `0`: 禁用
- `1`: 启用

官文没有说明默认值，以后遇到再补充


## format_schema

当您使用需要架构定义的格式（例如[Cap'n Proto](https://capnproto.org/)或[Protobuf）](https://developers.google.com/protocol-buffers/)时，此参数很有用。
该值取决于格式。

## fsync_metadata

写入 `.sql` 文件时启用或禁用[fsync](http://pubs.opengroup.org/onlinepubs/9699919799/functions/fsync.html)`。默认启用。

如果服务器具有数百万个不断创建和销毁的微小表块，则禁用它是有意义的。

## enable_http_compression

在对 `HTTP` 请求的响应中启用或禁用数据压缩。

有关更多信息，请阅读[HTTP 接口描述](../../clickhouse_interfaces.md)
- `0`: 禁用。(默认)
- `1`: 启用。

## http_zlib_compression_level

如果 `enable_http_compression = 1` 则设置对 HTTP 请求的响应中的数据压缩级别。

可能的值：[`1`,`9`] 闭合区间

默认值：`3`

## http_native_compression_disable_checksumming_on_decompress

从客户端解压缩 HTTP POST 数据时启用或禁用校验和验证。仅用于 ClickHouse `native compression` 格式（不适用于`gzip`或`deflate`）。

有关更多信息，请阅读[HTTP 接口描述](../../clickhouse_interfaces.md)

- `0`: 禁用。
- `1`: 启用。

默认值：0

## send_progress_in_http_headers

`X-ClickHouse-Progress` 在`clickhouse-server` 响应中启用或禁用 HTTP 响应标头。

有关更多信息，请阅读[HTTP 接口描述](../../clickhouse_interfaces.md)。

- 0-禁用。
- 1-启用。

默认值：0

## input_format_allow_errors_num

设置从文本格式（`CSV`，`TSV` 等）读取时可接受的最大错误数。

默认值为 0。

始终与配对 `input_format_allow_errors_ratio`。

如果在读取行时发生错误，但错误计数器仍小于`input_format_allow_errors_num`，则 ClickHouse 会忽略该行并继续进行下一行。

如果同时超过`input_format_allow_errors_num`和`input_format_allow_errors_ratio`，则 ClickHouse 会引发异常。

## input_format_allow_errors_ratio

设置从文本格式（`CSV`，`TSV` 等）读取时允许的最大错误百分比。错误百分比设置为 0 到 1 之间的浮点数。

默认值为 0。

始终与配对`input_format_allow_errors_num`。

如果在读取行时发生错误，但错误计数器仍小于`input_format_allow_errors_ratio`，则 ClickHouse 会忽略该行并继续进行下一行。

如果同时超过`input_format_allow_errors_num`和`input_format_allow_errors_ratio`，则 ClickHouse 会引发异常。

## input_format_values_interpret_expressions

如果快速流解析器无法解析数据，则启用或禁用完整的 SQL 解析器。
此设置仅用于数据插入时的 [`value`](../../clickhouse_interfaces.md)格式。有关语法分析的更多信息，请参见[`Syntax`](https://clickhouse.yandex/docs/en/query_language/syntax/)部分。

可能的值：

- 0-禁用。

  在这种情况下，您必须提供格式化的数据。请参阅[格式](https://clickhouse.yandex/docs/en/interfaces/formats/)部分。

- 1-启用。

  在这种情况下，可以将 SQL 表达式用作值，但是这种方式的数据插入速度要慢得多。如果仅插入格式化的数据，则 ClickHouse 的行为就像设置值为 0。

默认值：1。

**使用例**

插入具有不同设置的[DateTime](https://clickhouse.yandex/docs/en/data_types/datetime/)类型值。

```bash
SET input_format_values_interpret_expressions = 0;
INSERT INTO datetime_t VALUES (now())
```

```
Exception on client:
Code: 27. DB::Exception: Cannot parse input: expected ) before: now()): (at row 1)
```

```bash
SET input_format_values_interpret_expressions = 1;
INSERT INTO datetime_t VALUES (now())
```

```log
ok.
```

最后一个查询等效于以下内容：

```log
SET input_format_values_interpret_expressions = 0;
INSERT INTO datetime_t SELECT now()

ok.
```

## input_format_values_deduce_templates_of_expressions

为 [`value`](../../clickhouse_interfaces.md)格式的 SQL 表达式 启用或禁用模板推导。
- `Values` 如果连续行中的表达式具有相同的结构，则可以更快地解析和解释表达式。
- ClickHouse 将尝试推导表达式的模板，使用该模板解析以下行，并对成功解析的行进行评估。对于以下查询：

```sql
INSERT INTO test VALUES (lower('Hello')), (lower('world')), (lower('INSERT')), (upper('Values')), ...
```

- (1) if `input_format_values_interpret_expressions=1`和`format_values_deduce_templates_of_expressions=0`表达式将为每一行分别解释（对于大量行，这非常慢）
- (2) 如果第一行，第二行和第三行中的`input_format_values_interpret_expressions=0`和`format_values_deduce_templates_of_expressions=1`表达式将使用模板进行解析`lower(String)`并一起解释，则表达式第四行将与另一个模板（`upper(String)`）进行解析
- (3) 如 `input_format_values_interpret_expressions=1`和`format_values_deduce_templates_of_expressions=1`-与(2)情况相同，但如果无法推断出模板，则还允许回退到(1)。

此功能是实验性的，默认情况下处于禁用状态。

## input_format_values_accurate_types_of_literals

- 此设置仅在`input_format_values_deduce_templates_of_expressions = 1` 时使用。某些列的表达式可能具有相同的结构，但包含不同类型的数字文字，例如
  - 启用此设置时，ClickHouse将检查文字的实际类型，并使用相应类型的表达式模板。
  - 在某些情况下，它可能会显著降低值中的表达式求值的速度。
  - 当禁用时，ClickHouse可能会对某些文字使用更通用的类型(例如`Float64`或`Int64`而不是`UInt64`)，但它可能会导致溢出和精度问题。
  - 默认启用。
  
```log
(..., abs(0), ...),             -- UInt64 literal
(..., abs(3.141592654), ...),   -- Float64 literal
(..., abs(-1), ...),            -- Int64 literal
```

## input_format_defaults_for_omitted_fields

执行插入查询时，将省略的输入列值替换为相应列的默认值。此选项仅适用于`JSONEachRow`、`CSV`和`TabSeparated`格式。 [格式](../../clickhouse_interfaces.md)

**Note**

启用此选项后，将扩展表元数据从服务器发送到客户机。它会消耗服务器上的额外计算资源，并会降低性能。

可能的值:

0 -禁用。
1 -启用。(默认)

## input_format_tsv_empty_as_default

启用时，将`TSV` 中的空输入字段替换为默认值。对于复杂的默认表达式 `input_format_defaults_for_omitted_fields` 也必须启用。

默认情况下禁用。

## input_format_null_as_default

如果输入数据包含NULL，但是对应列的数据类型不可为空(对于文本输入格式)，则启用或禁用默认值。

## input_format_skip_unknown_fields

启用或禁用跳过额外数据插入。

在写入数据时，如果输入数据包含目标表中不存在的列，则ClickHouse抛出异常。如果启用了此配置项，ClickHouse不会插入额外的数据，也不会抛出异常。

- 支持格式:
  - `JSONEachRow`
  - `CSVWithNames`
  - `TabSeparatedWithNames`
  - `TSKV`

- `0` : 禁用。 (默认)
- `1` : 启用。

## input_format_import_nested_json
启用或禁用使用嵌套对象插入JSON数据。

- 支持格式:
  - `JSONEachRow`

- `0` -禁用。(默认)
- `1` -启用。

## input_format_with_names_use_header

- 启用或禁用在插入数据时检查列顺序。
  - 为了提高插入性能，如果您确定输入数据的列顺序与目标表中的相同，我们建议禁用此检查。

- 支持格式:
  - `CSVWithNames`
  - `TabSeparatedWithNames`
  
0 -禁用。
1 -启用。(默认)

## date_time_input_format

- 允许选择日期和时间的文本表示的解析器。
  - 此设置不适用于日期和时间函数。
  - `best_effort` : 支持扩展解析。
    - `ClickHouse` 可以解析基本的`YYYY-MM-DD HH:MM:SS`格式和所有 [ISO 8601](https://en.wikipedia.org/wiki/ISO_8601) 日期和时间格式。例如,`'2018-06-08T01:02:03.000Z'`。
  - `'basic'` : 使用 `basic` 解析器。
    - `ClickHouse` 只能解析基本的 `YYYY-MM-DD HH:MM:SS` 格式。例如，`'2019-08-20 10:18:56'`。
    - 默认


## join_default_strictness

设置连接子句的默认严格性。

- `value`: 
  - `ALL`: 如果 `right table` 有几个匹配的行，ClickHouse将从匹配的行创建一个笛卡尔积。这是标准SQL中的正常连接行为。
    - 默认
  - `ANY`: 如果 `right table` 有几个匹配的行，那么只有找到的第一个行被连接。如果 `right table` 只有一个匹配的行，`ANY` 和 `ALL` 的结果是相同的
  - `ASOF`: 用于连接具有不确定匹配的序列。
  - `Empty string`: 如果查询中没有指定ALL或ANY, ClickHouse将抛出异常。

## join_any_take_last_row

改变 `ANY` 严格连接操作的行为。

注意

此设置仅适用于具有 `join engine tables` 的 `JOIN` 连接。

- `0` : 如果`right table` 有多个匹配行，则只联接找到的第一个匹配行。 默认 `0`
- `1` : 如果`right table` 有多个匹配的行，那么只有最后找到的一个被连接。

## join_use_nulls

设置 `JOIN` 行为的类型。合并表时，可能出现空单元格。ClickHouse会根据这个设置以不同的方式填充它们。

可能的值:

- `0`: 空单元格由相应字段类型的默认值填充。
- `1`: JOIN的行为与标准SQL中的行为相同。相应字段的类型转换为空，空单元格用NULL填充。
- 默认值: `0`


## join_any_take_last_row

更改 `ANY JOIN` 的行为。
  - `0`: 禁用, 任何联接都将接受为键找到的第一行。
  - `1`: 启用(默认)，如果同一键有多行，则任何联接都接受最后匹配的行。该设置仅在 `join table engine` 中使用。

## max_block_size

- 数据由`block`(列`part`的集合)处理。单个块的内部处理周期足够高效，但是每个块都有显著的开销。
  - `max_block_size` 设置建议从表中加载多大的块(以行数为单位)。
  - 块的大小不应该太小，这样每个块上的开销仍然可以看到，但是不应该太大，这样在快速处理第一个块之后完成的带有LIMIT的查询就不会太大。
  - 目标是避免在多个线程中提取大量列时消耗太多内存，并至少保留一些缓存位置。

默认值: `65,536`

- `max_block_size` 大小的块并不总是从表中加载。
- 如果需要检索的数据明显减少，则处理更小的块。

## preferred_block_size_bytes
- 用于与`max_block_size` 相同的目的，以字节为单位。
  - 块大小不能超过max_block_size行。
  - 默认:`1,000,000` byte
  - 它只在从`MergeTree engine` 读取数据时起作用。

## merge_tree_uniform_read_distribution
- 当从 `MergeTree*` 表中读取数据时，采用多线程。
  - 该设置打开/关闭工作线程上的统一分配的读取任务。
  - 均匀分布算法的目标是使 `SELECT` 查询中所有线程的执行时间近似相等。
    - `0` ： 禁用
    - `1` ： 生效（默认值）

## merge_tree_min_rows_for_concurrent_read

- 对象：`MergeTree* engine` 表       
  - 正整数。                         
  - 默认值:`163,840`
  - 超过则并发读   

## merge_tree_min_bytes_for_concurrent_read

- 对象：`MergeTree* engine` 表
  - 正整数。
  - 默认值:`240*1024*1024`，
  - 超过则并发读

## merge_tree_min_rows_for_seek

- 对象：`MergeTree* engine` 表
  - 正整数。
  - 默认值:`0`
  - 文件中两个 `data block` 之间的距离行数小于它则不会遍历该文件，而是按顺序读取数据

## merge_tree_min_bytes_for_seek

- 对象：`MergeTree* engine` 表
  - 正整数。
  - 默认值:`0`
  - 文件中两个 `data block` 之间的数据大小小于它将顺序读取包含这两个块的文件范围，从而避免额外的查找。

## merge_tree_coarse_index_granularity

- 颗粒子范围
  - 对象：`MergeTree* engine` 表
  - 正偶数。
  - 默认值:`8`
  - 搜索时将所需要的`key`,所在范围划分为颗粒子大小的子区域，然后递归地查找。 


## merge_tree_max_rows_to_use_cache

- 一个查询内最大缓存的未压缩`block`的行数
  - 该设置保护缓存不受读取大量数据的查询的破坏。
  - `uncompressed_cache_size` 服务器设置定义未压缩块的缓存大小。
  - 对象：`MergeTree* engine` 表
  - 正整数。
  - 默认值: `128*8192`
  - 超过它，不会使用缓存


## merge_tree_max_bytes_to_use_cache

- 一个查询内可使用缓存的最大的未压缩`block` 的 byte大小
  - 该设置保护缓存不受读取大量数据的查询的破坏。
  - `uncompressed_cache_size` 服务器设置定义未压缩块的缓存大小。
  - 对象：`MergeTree* engine` 表
  - 正整数。
  - 默认值: `1920*1024*1024`
  - 超过它不会使用缓存。
  
  
## min_bytes_to_use_direct_io

- 使用直接 `I/O` 访问存储磁盘所需的最小数据量。
  - `0` : 默认， 禁止直接`I/O`。
  - 正整数。

## log_query

设置查询日志。具体影响：  [服务器设置](../server_configuration_parameters.md)

```log
log_query = 1
```

## max_insert_block_size

- 最大`INSERT` 块尺寸。
  - 此设置仅适用于服务器形成块的情况。
  - 例如，对于通过HTTP接口进行的插入，服务器解析数据格式并形成指定大小的块。
  - 使用 `clickhouse-client` 时，客户机解析数据本身，此设置项不会影响插入块的大小。
  - 使用`INSERT SELECT`时，此设置也不生效（应该说这种场景下，服务器不会检查此配置，因为先`select` 时，块大小就满足此条件，没必要多此一举），因为数据是使用SELECT之后形成的相同 `block` 插入的。
  - 默认值:`1,048,576`
    - 默认值略大于 `max_block_size`。
    - 这是因为某些表引擎(`*MergeTree`)为每个插入的块在磁盘上形成一个数据`part`，这是一个相当大的实体。
    - 类似地，`*MergeTree`表在插入期间对数据进行排序，足够大的块大小允许在RAM中对更多数据进行排序。


## max_replica_delay_for_distributed_queries

- 为 `distributed queries` 禁用滞后`replicas`。
  - 设置时间为秒。如果`replicas`的滞后超过设定值，则不使用该`replicas`。
  - 默认值: `300`。
  - 当从指向复制表的分布式表中执行 `SELECT` 时使用。

## max_threads

- 查询处理线程的最大数量(本服务器)
  - 不包括从远程服务器检索数据的线程(请参阅 `max_distributed_connections` 参数)。
  - 此参数适用于并行执行查询处理 `pipeline` 的相同阶段的线程。
    - 例如，在从表中读取数据时，如果可以使用函数计算表达式，使用WHERE进行筛选，并使用至少'max_threads'线程数并行地预聚合GROUP BY，则使用'max_threads'。
  - `default`: 物理CPU内核的数量。
  - 在以下情况下，可以设置 `max_thread` 较低的数量：
    - 由于 `LIMIT` 条件，能快速完成的请求
      - 如 每个 `block` 都能使得此次请求完成的情况下， 因 `max_threads` = `8`，每次都会检索 `8` 个`block`，其实只要检索一个`block` 即可。
      - 如果一次在服务器上运行的SELECT查询少于一个，则将此参数设置为略小于实际处理器核数的值。此句还不是很明白，原文如下： "If less than one SELECT query is normally run on a server at a time, set this parameter to a value slightly less than the actual number of processor cores."
  - `max_threads` 值越小，消耗的内存越少。

## max_compress_block_size

- 压缩写入表之前未压缩数据块的最大大小。
  - 默认值为1,048,576 (1 MiB)。(官方建议不作修改)
  - 如果减小大小，压缩率将显著降低，压缩和解压缩速度将由于缓存局部性而略微增加，内存消耗将减少。

不要将 `block`(由`byte`组成的`memory chunk`)与查询处理块(表中的行集合)混淆。

## min_compress_block_size

如果未压缩数据小于'max_compress_block_size'，则块的实际大小不小于此值，也不小于一个标记的数据量。



我们正在编写一个。当写入8192行时，数据的总大小为32 KB。因为min_compress_block_size = 65,536，每两个标记形成一个压缩块。

我们正在编写一个字符串类型的URL列(每个值的平均大小为60个字节)。当写入8192行时，数据的平均大小将略小于500 KB。由于这是超过65,536，一个压缩块将形成的每一个标志。在这种情况下，当从磁盘读取单个标记范围内的数据时，不会解压缩额外的数据。

通常没有任何理由改变这个设置。

- `block` 的实际大小最小值（未压缩数据）
  - 为了减少处理时延迟而定义
  - 对象：`MergeTree* engine` 表
  - 正整数。
  - 默认值: `65,536`，`64KB`

假设在创建表时将 'index_granularity' 设置为 `8192`：
  - 写一个 `UInt32-type` 的列(每个值 `4 bytes`), 写入 `8192` 行， 数据大小为 `32 KB`. 那么此压缩块由两个 `mark` 形成，数据大小为 `64KB` (未压缩)
  - 写一个 `String type` 的 `URL` 列(每个列平均大小为 `60 bytes`), 写入 `8192` 行， 数据大小为 `500 KB`， 由于大小超过`64KB`, 将会为每个 `mark` 形成 `block`。 

## mark_cache_min_lifetime

如果超过了 `mark_cache_size`(服务器设置里) 设置的值，那么只删除大于mark_cache_min_lifetime秒的记录。如果服务器内存很小，需要降低它来节约内存的使用。

默认值: `10,000 s`

## max_query_size

- 可以被带到 `RAM` 中使用SQL解析器进行解析的查询的最大 `part`。
  - `INSERT` 查询还包含由单独的流解析器(使用 `O(1) RAM` )处理这类请求，它不在此限制中。

默认值: `256KB` 

## interactive_delay

检查请求执行是否已取消并发送进度的间隔(以微秒为单位)。

默认值: `100,000` (`10次/s`)

## `connect_timeout`, `receive_timeout`, `send_timeout`
用于与客户端通信的 `socket` 上的超时(以秒为单位)。

默认值: `10`,`300`,`300`

## `poll_interval`

在指定的秒数内锁定等待循环

默认值: `10`

## max_distributed_connections

用于分布式处理单个查询到单个分布式表的与远程服务器并发的最大连接数。我们建议设置的值不少于集群中的服务器数量。

默认值:`1024`

以下参数仅用于创建分布式表(以及启动服务器)时，因此没有理由在运行时更改它们。

## distributed_connections_pool_size

与 `remote server` 同时连接的最大数量，用于对单个分布式表的所有查询进行分布式处理。我们建议设置的值不少于集群中的服务器数量。

默认值: `1024`

## connect_timeout_with_failover_ms

- 如果在集群定义中使用 `shard` 和 `replica` 部分，则连接到 `distributed table engine` 的远程服务器的超时(以毫秒为单位)。
- 如果不成功，将尝试连接多个`replicas`。

默认值: `50`

## connections_with_failover_max_tries

- 分布式表引擎对每个`replicas`的最大连接尝试次数。

默认值:`3`

## extremes

是否计算极值(查询结果列中的最小值和最大值)。接受0或1。默认情况下，0(禁用)。有关更多信息，请参见 [extreme values] 一节。

---

## use_uncompressed_cache

- 是否使用未压缩块的缓存。
  - `0`: 禁用（默认）
  - `1`: 启用
  - 在处理大量短查询时，使用未压缩的缓存(仅用于 `MergeTree` 系列中的表)可以显著降低延迟并增加吞吐量。
  - 为频繁发送短请求的用户启用此设置。
  - 还要注意 `uncompressed_cache_size` 配置参数(仅在配置文件中设置)——未压缩的缓存块的大小。默认是 `8GB`。
  - 未压缩的缓存将根据需要填充，而最少使用的数据将被自动删除。
  - 对于读取至少相当大的数据量(100万行或更多)的查询，将自动禁用未压缩的缓存，以便为真正小的查询节省空间。
  - 这意味着您可以始终将 `use_uncompressed_cache` 设置设置为 `1`

---

## replace_running_query

- 在使用HTTP接口时，可以传递`query_id`参数。
  - 这是用作查询标识符的任何字符串。
  - 如果来自同一用户的具有相同 `query_id` 的查询此时已经存在，则其行为取决于 `replace_running_query` 参数。
    - `0`: 抛异常(如果一个具有相同 `query_id` 的查询已经在运行，则不允许该查询运行)。 
    - `1`: 取消旧, 开始新

## stream_flush_interval_ms

在超时或线程生成 `max_insert_block_size` 行的情况下，对带有流的表有效。
  - 默认值是`7500` ms。
  - 值越小，数据被刷新到表中的频率就越高。
  - 设置过低的值会导致性能低下。

## load_balance

- 指定用于分布式查询处理的 `replicas` 选择算法。

- 支持以下选择 `replicas` 的算法:
  - `Random (by default)`
  - `Nearest hostname`
  - `In order`
  - `First or random`

---

###  Random (by default)

```bash
load_balancing = random
```
  
- 查询以最少的错误发送到`replicas`，如果有多个错误，则发送到其中任何一个。
  - `Disadvantages`: 
    - 没有考虑服务器最接近的值;
    - 如果`replicas`有不同的数据，您也将得到不同的数据。

--- 

### Nearest Hostname

```bash
load_balancing = nearest_hostname
```

每5分钟，误差的数量除以2。因此，用指数平滑法计算了最近一次的误差。
  - 如果有一个`replicas`有最小数量的错误(即最近在其他`replicas`上发生的错误)，查询就会发送给它。
  - 如果有多个`replicas`相同的最小数量的错误,查询发送到`replicas`的主机名是最类似于配置文件的服务器的主机名(在相同的位置,不同的字符数的最小长度两个主机名)。
  - 例如，`example01-01-1`和`example01-01-2.yandex.ru`在一个位置不同，而`example01-01-1`和`example01-02-2`在两个位置不同。
    - 这个方法看起来很原始，但是它不需要关于网络拓扑的外部数据，也不需要比较IP地址，这对于我们的IPv6地址来说很复杂。
    - 因此，如果有相同的`replicas`，则最好使用名称最接近的`replicas`。
    - 我们还可以假设，当将查询发送到同一服务器时，在没有故障的情况下，分布式查询也将发送到相同的服务器。
    - 因此，即使在`replicas`上放置了不同的数据，查询也会返回几乎相同的结果。

--- 

### In Order

```
load_balancing = in_order
```

具有相同错误数量的`replicas`将按照在配置中指定的顺序访问。当您确切地知道哪个`replicas`更可取时，此方法是合适的。

---

### First or Random
```bash
load_balance = first_or_random
```

该算法选择集合中的第一个 `replicas`，如果第一个 `replicas` 不可用，则选择一个随机 `replicas`。
它在 `corss-replication` 拓扑设置中有效，但在其他配置中无效。

`first_or_random` 算法解决了 `in_order` 算法的问题。

在 `in_order` 中，如果一个`replicas`宕机，那么下一个`replicas`将承受双倍的负载，而其余的`replicas`将处理通常的通信量。

当使用 `first_or_random` 算法时，负载均匀地分布在仍然可用的 `replicas` 中。

---

## prefer_localhost_replica

- 在处理分布式查询时，使用本地主机`replicas`启用/禁用优先级。
  - `1` : 本地 `replicas` 优先 （默认）
  - `0` : `ClickHouse` 使用 `load_balance` 设置指定的平衡策略。
  - 如果使用 `max_parallel_replicas` ，则禁用此设置。

---

## totals_mode

当 `HAVING` 存在 和 `max_rows_to_group_by`, `group_by_overflow_mode = 'any'` 存在。参见 `WITH TOTALS modifier` 部分。

---

## totals_auto_threshold

`totals_mode = 'auto'` 门槛 。参见 `WITH TOTALS modifier` 部分

---

## max_parallel_replicas

执行查询时每个 `shard` 的最大 `replicas` 数。
为了一致性(获得相同数据分割的不同 `part`)，此选项仅在设置采样键时有效。
不控制复制延迟。

## complile

允许编译查询。默认情况下，0(禁用)。

- 编译仅用于 `query-processing` 管道的 `part`: 用于 `GROUP BY` 的第一阶段。
  - 如果编译了管道的这一部分，由于部署了 `short cycles` 和 `inlining aggregation function` 调用，查询可能运行得更快。
  - 对于具有多个简单聚合函数的查询，可以看到最大的性能改进(在极少数情况下可以提高4倍)。
  - 通常，性能收益是微不足道的。
  - 在非常罕见的情况下，它可能会降低查询的执行速度。
  
---

## min_count_to_compile

- 在运行编译之前可能使用已编译代码块的次数。
  - 默认情况下: `3`
  - 对于测试，可以将该值设置为`0`:
    - 编译以同步方式运行，查询将等待编译过程结束后再继续执行。
    - 对于所有其他情况，使用从`1`开始的值。
    - 编译通常需要`5-10s`。
    - 如果值为`1`或更多，则在单独的线程中异步进行编译。
    - 结果将在准备就绪时立即使用，包括当前正在运行的查询。
  - 对于查询中使用的聚合函数和GROUP BY子句中的键类型的每个不同组合，都需要编译后的代码。
  - 编译的结果以 `.so` 文件的形式保存在构建目录中。编译结果的数量没有限制，因为它们不占用太多空间。
  - 旧的结果将在服务器重新启动后使用，除非服务器升级—在这种情况下，旧的结果将被删除。

---
## output_format_json_quote_64bit_integers

如果该值为真，那么在使用`JSON*` `Int64`和 `UInt64` 格式(为了与大多数 `JavaScript` 实现兼容)时，整数出现在引号中;否则，将输出不带引号的整数。

---

## format_csv_delimiter

在 `CSV` 数据中解释为分隔符的字符。默认:  `,`

## input_format_csv_unquoted_null_literal_as_null

对于 `CSV` 输入格式，启用或禁用将未引用的 `NULL` 解析为文字(即 `\N` 的同义词)。

## insert_quorum

- 使 `quorum` write。
  - `insert_quorum` < `2`， `quorum` write 功能 禁用
  - `insert_quorum` >= 2，`quorum` write 功能 启用
  -  `inset_quorum` = `0` (默认)

**Quorum writes**

只有当在 `insert_quorum_timeout` 期间成功地将数据写入 `replicas` 的 `insert_quorum` 时，`INSERT` 才会成功。

如果由于某种原因，写操作成功的 `replicas` 数量没有达到 `insert_quorum`，则认为写操作失败, 将从已经写入数据的所有`replicas`中删除插入的 `block`。

`quorum` 中的所有`replicas`都是一致的，即，它们包含以前所有 `INSERT` 查询的数据。`INSERT` 序列被线性化。

在读取从 `insert_quorum` 写入的数据时，可以使用 `select_sequential_consistency` 设置。

---

**ClickHouse generates an exception**

- 如果查询时可用`replicas`的数量小于 `insert_quorum` 。
  - 当前一个 `block` 尚未插入到`replicas`的`insert_quorum` 中时，尝试写入数据。如果用户试图在使用 `insert_quorum` 完成前一个插入之前执行插入，可能会出现这种情况。

**See also the following parameters**
`insert_quorum_timeout`
`select_sequential_consistency`

## insert_quorum_timeout

仲裁写入超时(以秒为单位)。
如果超时已过，但还没有发生写操作，ClickHouse将生成一个异常，客户端必须重复 `INSERT` 查询，将相同的块写入相同的或任何其他`replicas`。

默认值: `60s`

**请参阅以下参数:**
  -  `insert_quorum`
  -  `select_sequential_consistency`

---
## `select_sequential_consistency`

- 启用或禁用选择查询的顺序一致性:
  - `0` -禁用。(默认)
  - `1` -启用。

**Usage**

- 当启用顺序一致性时，ClickHouse允许客户端仅对那些包含用 `insert_quorum` 执行的所有先前 `INSERT` 查询的数据的 `replicas` 执行`SELECT` 查询。
  - 如果客户端引用一个部分`replicas`，ClickHouse将生成一个异常。SELECT查询将不包括尚未写入 `replicas` 仲裁的数据。

---
## max_network_bytes

在执行查询时，限制通过网络接收或传输的数据量(以字节为单位)。此设置适用于每个查询。

- value: 
  - 正整数。
  - `0` : `data volume` 控制被禁用。(默认)
  
--- 
## max_network_bandwidth

以每秒字节为单位限制网络上的数据交换速度。此设置适用于每个查询。

- `value`: 
  - 正整数。
  - `0` : 带宽控制被禁用。(默认)

--- 
## max_network_bandwidth_for_user

以每秒字节为单位限制网络上的数据交换速度。此设置适用于单个用户执行的所有并发运行的查询。

- value:                   
  - 正整数。                   
  - `0` : -数据速度的控制被禁用。(默认) 

---
## max_network_bandwidth_for_all_users
限制在网络上每秒以字节为单位交换数据的速度。此设置适用于服务器上所有并发运行的查询。

- value:                   
  - 正整数。                   
  - `0` : -数据速度的控制被禁用。(默认) 

---
## allow_experimental_cross_to_join_conversion

启用或禁用:
逗号语法的`JOIN`查询重写为 `JOIN` `ON/USING` 语法。如果设置值为0，则
如果 `WHERE` 允许，将 `CROSS JOIN` 转换为 `INNER JOIN`

- value: 
  - `0` : 禁用。不支持使用逗号的语法处理查询，并抛出异常。
  - `1` : 启用。


## count_distinct_implementation

指定应该使用哪个 `uniq*` 函数来执行 `COUNT(DISTINCT…)` 构造。

- value: 
  - `uniq`
  - `uniqCombined`
  - `uniqCombined64`
  - `uniqHLL12`
  - `uniqExact` (`default`)

## skip_unavailable_shards

- 启用或禁用跳过不可用的 `shard`

- 如果 `shard` 的所有 `replicas` 都不可用，则认为 `shard` 不可用。在下列情况下无法使用 `replicas`:

  - 因为任何原因，`ClickHouse` 都不能连接到 `replica` 

    - 当连接到一个`replicas`时，`ClickHouse` 会执行几次尝试。如果所有这些尝试都失败，则认为`replicas`不可用。

  - 无法通过DNS解析`replicas`。

    - 如果`replicas`的主机名不能通过DNS解析，则说明以下情况:

      - `replicas`的主机没有DNS记录。它可能发生在动态DNS系统中，例如 `Kubernetes` ，在停机期间节点可能无法解析，这不是错误。

      - 配置错误。`ClickHouse` 配置文件包含错误的主机名。

- value:                
  - `1` : 启用。
    - 如果 `shard` 不可用，ClickHouse将根据部分数据返回结果，并且不会报告节点可用性问题。
  - `0`： 禁用（默认）
    - 如果碎片不可用，`ClickHouse` 将抛出异常。

## optimize_throw_if_noop

启用或禁用在优化查询未执行合并时引发异常

默认情况下，即使不执行任何操作，优化也会成功返回。
此设置允许您区分这些情况并在异常消息中获得原因。

1 -启用抛出异常。
0 -禁止抛出异常。(默认)

---

## distributed_replica_error_half_life

控制分布式表的错误（`replicas` 不可用）被归零的速度。 累计 `5` 个错误后， 在 `distributed_replica_error_half_life`*3 内恢复正常。

- `type`:`seconds`
  - 默认值:`60s`
  
## distributed_replica_error_cap

每个 `replicas` 的错误计数都以这个值为上限，从而防止单个 `replicas` 积累到很大的程度。

- `type`:`unsigned int`
  - 默认值:`1000`
  
## os_thread_priority

为执行查询的线程设置优先级([nice](https://en.wikipedia.org/wiki/Nice_(Unix)))。

当选择下一个线程在每个可用的CPU内核上运行时，OS调度器会考虑这个优先级。

警告

要使用此设置，需要设置 `CAP_SYS_NICE` 功能。`clickhouse-server` 包在安装过程中设置它。一些虚拟环境不允许您设置 `CAP_SYS_NICE` 功能。
在本例中，`clickhouse-server` 在开始时显示关于它的消息。

- value: 
  - `[-20,19]`
  
较低的值意味着较高的优先级。优先级低的线程比优先级高的线程执行得更频繁。 对于长时间运行的非交互式查询，高值是更好的选择，因为它允许它们在短时间的交互式查询到来时快速放弃资源。

默认值: `0`


