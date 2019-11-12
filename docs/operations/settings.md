# 设定值

可以通过多种方式进行以下所有设置。设置是分层配置的，因此每个后续层都重新定义了先前的设置。

按优先级配置设置的方式：

- `users.xml`服务器配置文件中的设置。

  设置元素`<profiles>`。

- 会话设置。

  发送`SET setting=value`从以交互方式 ClickHouse 控制台客户端。同样，您可以在 HTTP 协议中使用 ClickHouse 会话。为此，您需要指定`session_id`HTTP 参数。

- 查询设置。

  - 在非交互模式下启动 ClickHouse 控制台客户端时，请设置启动参数`--setting=value`。
  - 使用 HTTP API 时，请传递 CGI 参数（`URL?setting_1=value&setting_2=value...`）。

本节未涵盖只能在服务器配置文件中进行的设置。

# 查询权限

ClickHouse 中的查询可以分为几种类型：

1.  读取数据查询：`SELECT`，`SHOW`，`DESCRIBE`，`EXISTS`。
2.  编写数据查询：`INSERT`，`OPTIMIZE`。
3.  更改设置查询：`SET`，`USE`。
4.  [DDL](https://en.wikipedia.org/wiki/Data_definition_language)查询：`CREATE`，`ALTER`，`RENAME`，`ATTACH`，`DETACH`，`DROP` `TRUNCATE`。
5.  `KILL QUERY`。

以下设置通过查询类型来调节用户权限：

- [只读](https://clickhouse.yandex/docs/en/operations/settings/permissions_for_queries/#settings_readonly) -限制对所有类型的查询（DDL 查询除外）的权限。
- [allow_ddl-](https://clickhouse.yandex/docs/en/operations/settings/permissions_for_queries/#settings_allow_ddl)限制 DDL 查询的权限。

`KILL QUERY`  可以使用任何设置执行。

## 只读[¶](https://clickhouse.yandex/docs/en/operations/settings/permissions_for_queries/#settings_readonly "永久链接")

限制读取数据，写入数据和更改设置查询的权限。

请参阅[上面](https://clickhouse.yandex/docs/en/operations/settings/permissions_for_queries/#permissions_for_queries)的查询如何将查询分为几种类型。

可能的值：

- 0-允许所有查询。
- 1-仅允许读取数据查询。
- 2-允许读取数据和更改设置查询。

设置后`readonly = 1`，用户无法在当前会话中更改`readonly`和`allow_ddl`设置。

`GET`在[HTTP 界面中](https://clickhouse.yandex/docs/en/interfaces/http/)使用方法时，`readonly = 1`将自动设置。若要修改数据，请使用`POST`方法。

设置`readonly = 1`禁止用户更改所有设置。有一种方法可以禁止用户仅更改特定设置，有关详细信息，请参阅[设置约束](https://clickhouse.yandex/docs/en/operations/settings/constraints_on_settings/)。

默认值：0

## allow_ddl [¶](https://clickhouse.yandex/docs/en/operations/settings/permissions_for_queries/#settings_allow_ddl "永久链接")

允许或拒绝[DDL](https://en.wikipedia.org/wiki/Data_definition_language)查询。

请参阅[上面](https://clickhouse.yandex/docs/en/operations/settings/permissions_for_queries/#permissions_for_queries)的查询如何将查询分为几种类型。

可能的值：

- 0-不允许 DDL 查询。
- 1-允许 DDL 查询。

`SET allow_ddl = 1`如果`allow_ddl = 0`当前会话无法执行。

默认值：1

# 对查询复杂度的限制

对查询复杂度的限制是设置的一部分。使用它们是为了从用户界面提供更安全的执行。几乎所有限制仅适用于`SELECT`。对于分布式查询处理，限制分别应用于每个服务器。

ClickHouse 会检查数据部分的限制，而不是每一行的限制。这意味着您可以超出数据部分大小的限制值。

对“最大数量”的限制可以取值为 0，表示“不受限制”。大多数限制还具有“ overflow_mode”设置，这意味着超出限制时该怎么做。它可以采用以下两个值之一：`throw`或`break`。聚合限制（group_by_overflow_mode）也具有值`any`。

`throw` –引发异常（默认）。

`break` –停止执行查询并返回部分结果，就像源数据用完了一样。

`any (only for group_by_overflow_mode)` –继续聚合进入集合的密钥，但不要向集合中添加新密钥。

## max_memory_usage [¶](https://clickhouse.yandex/docs/en/operations/settings/query_complexity/#settings_max_memory_usage "永久链接")

用于在单个服务器上运行查询的最大 RAM 量。

在默认配置文件中，最大为 10 GB。

该设置不考虑可用内存量或计算机上的总内存量。该限制适用于单个服务器内的单个查询。您可以`SHOW PROCESSLIST`用来查看每个查询的当前内存消耗。此外，将跟踪每个查询的峰值内存消耗并将其写入日志。

不监视某些聚合功能状态的内存使用情况。

内存使用率未完全跟踪的集合函数状态`min`，`max`，`any`，`anyLast`，`argMin`，`argMax`从`String`和`Array`参数。

内存消耗也受参数`max_memory_usage_for_user`和限制`max_memory_usage_for_all_queries`。

## max_memory_usage_for_user [¶](https://clickhouse.yandex/docs/en/operations/settings/query_complexity/#max-memory-usage-for-user "永久链接")

用于在单个服务器上运行用户查询的最大 RAM 量。

默认值在[Settings.h](https://github.com/ClickHouse/ClickHouse/blob/master/dbms/src/Core/Settings.h#L288)中定义。默认情况下，金额不受限制（`max_memory_usage_for_user = 0`）。

另请参见[max_memory_usage](https://clickhouse.yandex/docs/en/operations/settings/query_complexity/#settings_max_memory_usage)的描述。

## max_memory_usage_for_all_queries [¶](https://clickhouse.yandex/docs/en/operations/settings/query_complexity/#max-memory-usage-for-all-queries "永久链接")

用于在单个服务器上运行所有查询的最大 RAM 量。

默认值在[Settings.h](https://github.com/ClickHouse/ClickHouse/blob/master/dbms/src/Core/Settings.h#L289)中定义。默认情况下，金额不受限制（`max_memory_usage_for_all_queries = 0`）。

另请参见[max_memory_usage](https://clickhouse.yandex/docs/en/operations/settings/query_complexity/#settings_max_memory_usage)的描述。

## max_rows_to_read [¶](https://clickhouse.yandex/docs/en/operations/settings/query_complexity/#max-rows-to-read "永久链接")

可以在每个块（而不是每一行）上检查以下限制。也就是说，限制可以稍微打破。在多个线程中运行查询时，以下限制分别适用于每个线程。

运行查询时可以从表中读取的最大行数。

## max_bytes_to_read [¶](https://clickhouse.yandex/docs/en/operations/settings/query_complexity/#max-bytes-to-read "永久链接")

运行查询时可以从表中读取的最大字节数（未压缩的数据）。

## read_overflow_mode [¶](https://clickhouse.yandex/docs/en/operations/settings/query_complexity/#read-overflow-mode "永久链接")

当读取的数据量超过以下限制之一时该怎么办：“抛出”或“中断”。默认情况下，抛出。

## max_rows_to_group_by [¶](https://clickhouse.yandex/docs/en/operations/settings/query_complexity/#settings-max_rows_to_group_by "永久链接")

从聚合收到的唯一密钥的最大数目。通过此设置，可以限制聚合时的内存消耗。

## group_by_overflow_mode [¶](https://clickhouse.yandex/docs/en/operations/settings/query_complexity/#group-by-overflow-mode "永久链接")

当聚合的唯一键的数量超过限制时该怎么办：“ throw”，“ break”或“ any”。默认情况下，抛出。使用“ any”值可让您近似计算 GROUP BY。这种近似的质量取决于数据的统计性质。

## max_bytes_before_external_group_by [¶](https://clickhouse.yandex/docs/en/operations/settings/query_complexity/#settings-max_bytes_before_external_group_by "永久链接")

启用或禁用`GROUP BY`外部存储器中子句的执行。请参阅[外部存储器中的 GROUP BY](https://clickhouse.yandex/docs/en/query_language/select/#select-group-by-in-external-memory)。

可能的值：

- 单个[GROUP BY](https://clickhouse.yandex/docs/en/query_language/select/#select-group-by-clause)操作可以使用的最大 RAM 量（以字节为单位）。
- 0- `GROUP BY`禁用外部存储器。

默认值：0

## max_rows_to_sort [¶](https://clickhouse.yandex/docs/en/operations/settings/query_complexity/#max-rows-to-sort "永久链接")

排序前的最大行数。这使您可以限制排序时的内存消耗。

## max_bytes_to_sort [¶](https://clickhouse.yandex/docs/en/operations/settings/query_complexity/#max-bytes-to-sort "永久链接")

排序前的最大字节数。

## sort_overflow_mode [¶](https://clickhouse.yandex/docs/en/operations/settings/query_complexity/#sort-overflow-mode "永久链接")

如果排序前收到的行数超过限制之一：“ throw”或“ break”，该怎么办。默认情况下，抛出。

## max_result_rows [¶](https://clickhouse.yandex/docs/en/operations/settings/query_complexity/#max-result-rows "永久链接")

限制结果中的行数。运行部分分布式查询时，还检查子查询以及在远程服务器上。

## max_result_bytes [¶](https://clickhouse.yandex/docs/en/operations/settings/query_complexity/#max-result-bytes "永久链接")

限制结果中的字节数。与之前的设置相同。

## result_overflow_mode [¶](https://clickhouse.yandex/docs/en/operations/settings/query_complexity/#result-overflow-mode "永久链接")

如果结果量超过以下限制之一：“ throw”或“ break”，该怎么办。默认情况下，抛出。使用“ break”类似于使用 LIMIT。

## 的 max_execution_time [¶](https://clickhouse.yandex/docs/en/operations/settings/query_complexity/#max-execution-time "永久链接")

最大查询执行时间，以秒为单位。目前，不检查排序阶段之一，也不合并和最终确定聚合函数。

## timeout_overflow_mode [¶](https://clickhouse.yandex/docs/en/operations/settings/query_complexity/#timeout-overflow-mode "永久链接")

如果查询运行时间超过'max_execution_time'：'throw'或'break'，该怎么办。默认情况下，抛出。

## min_execution_speed [¶](https://clickhouse.yandex/docs/en/operations/settings/query_complexity/#min-execution-speed "永久链接")

最小执行速度（每秒行数）。当“ timeout_before_checking_execution_speed”到期时，检查每个数据块。如果执行速度较低，则会引发异常。

## min_execution_speed_bytes [¶](https://clickhouse.yandex/docs/en/operations/settings/query_complexity/#min-execution-speed-bytes "永久链接")

每秒最小执行字节数。当“ timeout_before_checking_execution_speed”到期时，检查每个数据块。如果执行速度较低，则会引发异常。

## max_execution_speed [¶](https://clickhouse.yandex/docs/en/operations/settings/query_complexity/#max-execution-speed "永久链接")

每秒最大执行行数。当“ timeout_before_checking_execution_speed”到期时，检查每个数据块。如果执行速度很高，执行速度将降低。

## max_execution_speed_bytes [¶](https://clickhouse.yandex/docs/en/operations/settings/query_complexity/#max-execution-speed-bytes "永久链接")

每秒最大执行字节数。当“ timeout_before_checking_execution_speed”到期时，检查每个数据块。如果执行速度很高，执行速度将降低。

## timeout_before_checking_execution_speed [¶](https://clickhouse.yandex/docs/en/operations/settings/query_complexity/#timeout-before-checking-execution-speed "永久链接")

在以秒为单位的指定时间到期后，检查执行速度是否不太慢（不小于'min_execution_speed'）。

## max_columns_to_read [¶](https://clickhouse.yandex/docs/en/operations/settings/query_complexity/#max-columns-to-read "永久链接")

单个查询中可以从表中读取的最大列数。如果查询需要读取更多的列，则会引发异常。

## max_temporary_columns [¶](https://clickhouse.yandex/docs/en/operations/settings/query_complexity/#max-temporary-columns "永久链接")

运行查询时必须同时在 RAM 中保留的临时列的最大数量，包括常量列。如果临时列多于此，则会引发异常。

## max_temporary_non_const_columns [¶](https://clickhouse.yandex/docs/en/operations/settings/query_complexity/#max-temporary-non-const-columns "永久链接")

与“ max_temporary_columns”相同，但不计算常量列。请注意，运行查询时经常会形成常数列，但它们需要大约零计算资源。

## max_subquery_depth [¶](https://clickhouse.yandex/docs/en/operations/settings/query_complexity/#max-subquery-depth "永久链接")

子查询的最大嵌套深度。如果子查询更深，则会引发异常。默认情况下为 100。

## max_pipeline_depth [¶](https://clickhouse.yandex/docs/en/operations/settings/query_complexity/#max-pipeline-depth "永久链接")

最大管道深度。对应于每个数据块在查询处理期间经历的转换次数。计入单个服务器的限制内。如果管道深度更大，则会引发异常。默认情况下为 1000。

## max_ast_depth [¶](https://clickhouse.yandex/docs/en/operations/settings/query_complexity/#max-ast-depth "永久链接")

查询语法树的最大嵌套深度。如果突破，则将引发异常。此时，它不解析期间检查，但只解析查询之后。也就是说，在解析过程中可能会创建太深的语法树，但查询将失败。默认情况下为 1000。

## max_ast_elements [¶](https://clickhouse.yandex/docs/en/operations/settings/query_complexity/#max-ast-elements "永久链接")

查询语法树中的最大元素数。如果突破，则将引发异常。与以前的设置相同，仅在解析查询后才对其进行检查。默认情况下为 50,000。

## max_rows_in_set [¶](https://clickhouse.yandex/docs/en/operations/settings/query_complexity/#max-rows-in-set "永久链接")

从子查询创建的 IN 子句中的数据集的最大行数。

## max_bytes_in_set [¶](https://clickhouse.yandex/docs/en/operations/settings/query_complexity/#max-bytes-in-set "永久链接")

从子查询创建的 IN 子句中的集合使用的最大字节数（未压缩的数据）。

## set_overflow_mode [¶](https://clickhouse.yandex/docs/en/operations/settings/query_complexity/#set-overflow-mode "永久链接")

当数据量超过以下限制之一时该怎么办：“抛出”或“中断”。默认情况下，抛出。

## max_rows_in_distinct [¶](https://clickhouse.yandex/docs/en/operations/settings/query_complexity/#max-rows-in-distinct "永久链接")

使用 DISTINCT 时的最大不同行数。

## max_bytes_in_distinct [¶](https://clickhouse.yandex/docs/en/operations/settings/query_complexity/#max-bytes-in-distinct "永久链接")

使用 DISTINCT 时，哈希表使用的最大字节数。

## distinct_overflow_mode [¶](https://clickhouse.yandex/docs/en/operations/settings/query_complexity/#distinct-overflow-mode "永久链接")

当数据量超过以下限制之一时该怎么办：“抛出”或“中断”。默认情况下，抛出。

## max_rows_to_transfer [¶](https://clickhouse.yandex/docs/en/operations/settings/query_complexity/#max-rows-to-transfer "永久链接")

使用 GLOBAL IN 时可以传递到远程服务器或保存在临时表中的最大行数。

## max_bytes_to_transfer [¶](https://clickhouse.yandex/docs/en/operations/settings/query_complexity/#max-bytes-to-transfer "永久链接")

使用 GLOBAL IN 时，可以传递到远程服务器或保存在临时表中的最大字节数（未压缩的数据）。

## transfer_overflow_mode [¶](https://clickhouse.yandex/docs/en/operations/settings/query_complexity/#transfer-overflow-mode "永久链接")

当数据量超过以下限制之一时该怎么办：“抛出”或“中断”。默认情况下，抛出。

## max_rows_in_join [¶](https://clickhouse.yandex/docs/en/operations/settings/query_complexity/#settings-max_rows_in_join "永久链接")

限制联接表时使用的哈希表中的行数。

此设置适用于[SELECT ... JOIN](https://clickhouse.yandex/docs/en/query_language/select/#select-join)操作和[Join](https://clickhouse.yandex/docs/en/operations/table_engines/join/)表引擎。

如果查询包含多个联接，ClickHouse 将为每个中间结果检查此设置。

达到限制后，ClickHouse 可以继续执行其他操作。使用[join_overflow_mode](https://clickhouse.yandex/docs/en/operations/settings/query_complexity/#settings-join_overflow_mode)设置选择操作。

可能的值：

- 正整数。
- 0-无限行。

默认值：0

## max_bytes_in_join [¶](https://clickhouse.yandex/docs/en/operations/settings/query_complexity/#settings-max_bytes_in_join "永久链接")

限制联接表时使用的哈希表的大小（以字节为单位）。

此设置适用于[SELECT ... JOIN](https://clickhouse.yandex/docs/en/query_language/select/#select-join)操作和[Join 表引擎](https://clickhouse.yandex/docs/en/operations/table_engines/join/)。

如果查询包含联接，ClickHouse 将为每个中间结果检查此设置。

达到限制后，ClickHouse 可以继续执行其他操作。使用[join_overflow_mode](https://clickhouse.yandex/docs/en/operations/settings/query_complexity/#settings-join_overflow_mode)设置选择操作。

可能的值：

- 正整数。
- 0-禁用内存控制。

默认值：0

## join_overflow_mode [¶](https://clickhouse.yandex/docs/en/operations/settings/query_complexity/#settings-join_overflow_mode "永久链接")

定义达到以下任何一个连接限制时 ClickHouse 会执行的操作：

- [max_bytes_in_join](https://clickhouse.yandex/docs/en/operations/settings/query_complexity/#settings-max_bytes_in_join)
- [max_rows_in_join](https://clickhouse.yandex/docs/en/operations/settings/query_complexity/#settings-max_rows_in_join)

可能的值：

- `THROW` — ClickHouse 引发异常并中断操作。
- `BREAK` — ClickHouse 会中断操作，并且不会引发异常。

默认值：`THROW`。

**也可以看看**

- [JOIN 子句](https://clickhouse.yandex/docs/en/query_language/select/#select-join)
- [联接表引擎](https://clickhouse.yandex/docs/en/operations/table_engines/join/)

## max_partitions_per_insert_block [¶](https://clickhouse.yandex/docs/en/operations/settings/query_complexity/#max-partitions-per-insert-block "永久链接")

限制单个插入块中的最大分区数。

- 正整数。
- 0-无限数量的分区。

默认值：100。

**细节**

插入数据时，ClickHouse 计算插入的块中的分区数。如果分区数大于`max_partitions_per_insert_block`，则 ClickHouse 会引发带有以下文本的异常：

> “单个 INSERT 块的分区太多（大于“ + toString（max_parts）+”）。限制由'max_partitions_per_insert_block'设置控制。大量分区是一种常见的误解。这将导致严重的负面性能影响，包括缓慢的服务器启动，缓慢的 INSERT 查询和缓慢的 SELECT 查询。表的分区总数建议低于 1000..10000。请注意，分区的目的不是加快 SELECT 查询的速度（ORDER BY 键足以进行范围查询分区）用于数据处理（DROP 分区等）。”
>
> # 设定值
>
> ## distributed_product_mode [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#distributed-product-mode "永久链接")
>
> 更改[分布式子查询](https://clickhouse.yandex/docs/en/query_language/select/)的行为。
>
> 当查询包含分布式表的乘积时，即当分布式表的查询包含分布式表的非 GLOBAL 子查询时，ClickHouse 将应用此设置。
>
> 限制条件：
>
> - 仅适用于 IN 和 JOIN 子查询。
> - 仅当 FROM 部分使用包含多个分片的分布式表时。
> - 如果子查询涉及一个包含多个分片的分布式表。
> - 不用于表值[远程](https://clickhouse.yandex/docs/en/query_language/table_functions/remote/)功能。
>
> 可能的值：
>
> - `deny`\- 默认值。禁止使用这些类型的子查询（返回“ Double-distributed in / JOIN 子查询被拒绝”异常）。
> - `local`—将子查询中的数据库和表替换为目标服务器（碎片）的本地查询，而保留普通的`IN`/`JOIN.`
> - `global`—将`IN`/ `JOIN`查询替换为`GLOBAL IN`/`GLOBAL JOIN.`
> - `allow` \- 允许使用这些类型的子查询的。
>
> ## enable_optimize_predicate_expression [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#enable-optimize-predicate-expression "永久链接")
>
> 打开`SELECT`查询中的谓词下推。
>
> 谓词下推可能会大大减少分布式查询的网络流量。
>
> 可能的值：
>
> - 0-禁用。
> - 1-启用。
>
> 默认值：1。
>
> **用法**
>
> 考虑以下查询：
>
> 1.  `SELECT count() FROM test_table WHERE date = '2018-10-10'`
> 2.  `SELECT count() FROM (SELECT * FROM test_table) WHERE date = '2018-10-10'`
>
> 如果为`enable_optimize_predicate_expression = 1`，则这些查询的执行时间相等，因为 ClickHouse `WHERE`在处理子查询时适用于该子查询。
>
> 如果为`enable_optimize_predicate_expression = 0`，则第二个查询的执行时间会更长，因为该`WHERE`子句适用于子查询完成后的所有数据。
>
> ## fallback_to_stale_replicas_for_distributed_queries [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-fallback_to_stale_replicas_for_distributed_queries "永久链接")
>
> 如果没有更新的数据，则强制查询到过期的副本。请参见“ [复制](https://clickhouse.yandex/docs/en/operations/table_engines/replication/) ”。
>
> ClickHouse 从表的过时副本中选择最相关的。
>
> `SELECT`从指向复制表的分布式表执行时使用。
>
> 默认情况下，为 1（启用）。
>
> ## force_index_by_date [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-force_index_by_date "永久链接")
>
> 如果无法按日期使用索引，则禁用查询执行。
>
> 与 MergeTree 系列中的表一起使用。
>
> 如果为`force_index_by_date=1`，则 ClickHouse 将检查查询是否具有可用于限制数据范围的日期键条件。如果没有合适的条件，它将引发异常。但是，它不会检查条件是否实际上减少了要读取的数据量。例如，`Date != ' 2000-01-01 '`即使条件与表中的所有数据匹配（例如，运行查询需要完全扫描），该条件也是可以接受的。有关 MergeTree 表中数据范围的更多信息，请参见“ [MergeTree](https://clickhouse.yandex/docs/en/operations/table_engines/mergetree/) ”。
>
> ## force_primary_key [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#force-primary-key "永久链接")
>
> 如果无法通过主键建立索引，则禁用查询执行。
>
> 与 MergeTree 系列中的表一起使用。
>
> 如果为`force_primary_key=1`，则 ClickHouse 会检查查询是否具有可用于限制数据范围的主键条件。如果没有合适的条件，它将引发异常。但是，它不会检查条件是否实际上减少了要读取的数据量。有关 MergeTree 表中数据范围的更多信息，请参见“ [MergeTree](https://clickhouse.yandex/docs/en/operations/table_engines/mergetree/) ”。
>
> ## format_schema [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#format-schema "永久链接")
>
> 当您使用需要架构定义的格式（例如[Cap'n Proto](https://capnproto.org/)或[Protobuf）](https://developers.google.com/protocol-buffers/)时，此参数很有用。该值取决于格式。
>
> ## fsync_metadata [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#fsync-metadata "永久链接")
>
> 写入文件时启用或禁用[fsync](http://pubs.opengroup.org/onlinepubs/9699919799/functions/fsync.html)`.sql`。默认启用。
>
> 如果服务器具有数百万个不断创建和销毁的微小表块，则禁用它是有意义的。
>
> ## enable_http_compression [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-enable_http_compression "永久链接")
>
> 在对 HTTP 请求的响应中启用或禁用数据压缩。
>
> 有关更多信息，请阅读[HTTP 接口描述](https://clickhouse.yandex/docs/en/interfaces/http/)。
>
> 可能的值：
>
> - 0-禁用。
> - 1-启用。
>
> 默认值：0
>
> ## http_zlib_compression_level [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-http_zlib_compression_level "永久链接")
>
> 如果[enable_http_compression = 1，](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-enable_http_compression)则设置对 HTTP 请求的响应中的数据压缩级别。
>
> 可能的值：1 到 9 之间的数字。
>
> 默认值：3。
>
> ## http_native_compression_disable_checksumming_on_decompress [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-http_native_compression_disable_checksumming_on_decompress "永久链接")
>
> 从客户端解压缩 HTTP POST 数据时启用或禁用校验和验证。仅用于 ClickHouse 本机压缩格式（不适用于`gzip`或`deflate`）。
>
> 有关更多信息，请阅读[HTTP 接口描述](https://clickhouse.yandex/docs/en/interfaces/http/)。
>
> 可能的值：
>
> - 0-禁用。
> - 1-启用。
>
> 默认值：0
>
> ## send_progress_in_http_headers [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-send_progress_in_http_headers "永久链接")
>
> `X-ClickHouse-Progress`在`clickhouse-server`响应中启用或禁用 HTTP 响应标头。
>
> 有关更多信息，请阅读[HTTP 接口描述](https://clickhouse.yandex/docs/en/interfaces/http/)。
>
> 可能的值：
>
> - 0-禁用。
> - 1-启用。
>
> 默认值：0
>
> ## input_format_allow_errors_num [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-input_format_allow_errors_num "永久链接")
>
> 设置从文本格式（CSV，TSV 等）读取时可接受的最大错误数。
>
> 默认值为 0。
>
> 始终与配对`input_format_allow_errors_ratio`。
>
> 如果在读取行时发生错误，但错误计数器仍小于`input_format_allow_errors_num`，则 ClickHouse 会忽略该行并继续进行下一行。
>
> 如果同时超过`input_format_allow_errors_num`和`input_format_allow_errors_ratio`，则 ClickHouse 会引发异常。
>
> ## input_format_allow_errors_ratio [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-input_format_allow_errors_ratio "永久链接")
>
> 设置从文本格式（CSV，TSV 等）读取时允许的最大错误百分比。错误百分比设置为 0 到 1 之间的浮点数。
>
> 默认值为 0。
>
> 始终与配对`input_format_allow_errors_num`。
>
> 如果在读取行时发生错误，但错误计数器仍小于`input_format_allow_errors_ratio`，则 ClickHouse 会忽略该行并继续进行下一行。
>
> 如果同时超过`input_format_allow_errors_num`和`input_format_allow_errors_ratio`，则 ClickHouse 会引发异常。
>
> ## input_format_values_interpret_expressions [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-input_format_values_interpret_expressions "永久链接")
>
> 如果快速流解析器无法解析数据，则启用或禁用完整的 SQL 解析器。此设置仅用于数据插入时的“ [值”](https://clickhouse.yandex/docs/en/interfaces/formats/#data-format-values)格式。有关语法分析的更多信息，请参见“ [语法”](https://clickhouse.yandex/docs/en/query_language/syntax/)部分。
>
> 可能的值：
>
> - 0-禁用。
>
>   在这种情况下，您必须提供格式化的数据。请参阅[格式](https://clickhouse.yandex/docs/en/interfaces/formats/)部分。
>
> - 1-启用。
>
>   在这种情况下，可以将 SQL 表达式用作值，但是这种方式的数据插入速度要慢得多。如果仅插入格式化的数据，则 ClickHouse 的行为就像设置值为 0。
>
> 默认值：1。
>
> **使用例**
>
> 插入具有不同设置的[DateTime](https://clickhouse.yandex/docs/en/data_types/datetime/)类型值。
>
> SET input_format_values_interpret_expressions = 0 ;
> INSERT INTO datetime_t VALUES （now （））
>
> 客户端异常：
> 代码：27。DB :: Exception：无法解析输入：预期）之前：now（））：（在第 1 行）
>
> SET input_format_values_interpret_expressions = 1 ;
> INSERT INTO datetime_t VALUES （now （））
>
> 好。
>
> 最后一个查询等效于以下内容：
>
> SET input_format_values_interpret_expressions = 0 ;
> INSERT INTO datetime_t SELECT 现在（）
>
> 好。
>
> ## input_format_values_deduce_templates_of_expressions [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-input_format_values_deduce_templates_of_expressions "永久链接")
>
> 为“ [值”](https://clickhouse.yandex/docs/en/interfaces/formats/#data-format-values)格式的 SQL 表达式启用或禁用模板推导。`Values`如果连续行中的表达式具有相同的结构，则可以更快地解析和解释表达式。ClickHouse 将尝试推导表达式的模板，使用该模板解析以下行，并对成功解析的行进行评估。对于以下查询：
>
> INSERT INTO 测试 VALUES （低（'你好' ））， （下（'世界' ））， （下（'插入' ））， （上（'值' ））， ...
>
> - if `input_format_values_interpret_expressions=1`和`format_values_deduce_templates_of_expressions=0`表达式将为每一行分别解释（对于大量行，这非常慢）
> - 如果第一行，第二行和第三行中的`input_format_values_interpret_expressions=0`和`format_values_deduce_templates_of_expressions=1`表达式将使用模板进行解析`lower(String)`并一起解释，则表达式第四行将与另一个模板（`upper(String)`）进行解析
> - if `input_format_values_interpret_expressions=1`和`format_values_deduce_templates_of_expressions=1`-与前面的情况相同，但如果无法推断出模板，则还允许回退到分别解释表达式。
>
> 此功能是实验性的，默认情况下处于禁用状态。
>
> ## input_format_values_accurate_types_of_literals [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-input_format_values_accurate_types_of_literals "永久链接")
>
> 仅当时使用此设置`input_format_values_deduce_templates_of_expressions = 1`。可能发生的情况是，某些列的表达式具有相同的结构，但包含不同类型的数字文字，例如
>
> （...， ABS （0 ）， ...）， \- UINT64 字面
> （...， ABS （3 。141592654 ）， ...）， \- Float64 字面
> （...， ABS （\- 1 ） ， ...）， -Int64 文字
>
> 启用此设置后，ClickHouse 将检查文字的实际类型，并使用相应类型的表达式模板。在某些情况下，它可能显着减慢小鼠中的表达评价`Values`。禁用后，ClickHouse 可能对某些文字使用更通用的类型（例如`Float64`或`Int64`代替`UInt64`for `42`），但是它可能导致溢出和精度问题。默认启用。
>
> ## input_format_defaults_for_omitted_fields [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#session_settings-input_format_defaults_for_omitted_fields "永久链接")
>
> 执行`INSERT`查询时，将省略的输入列值替换为各个列的默认值。此选项仅适用于[JSONEachRow](https://clickhouse.yandex/docs/en/interfaces/formats/#jsoneachrow)，[CSV](https://clickhouse.yandex/docs/en/interfaces/formats/#csv)和[TabSeparated](https://clickhouse.yandex/docs/en/interfaces/formats/#tabseparated)格式。
>
> 注意
>
> 启用此选项后，扩展表元数据将从服务器发送到客户端。它消耗了服务器上的其他计算资源，并可能降低性能。
>
> 可能的值：
>
> - 0-禁用。
> - 1-启用。
>
> 默认值：1。
>
> ## input_format_tsv_empty_as_default [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-input_format_tsv_empty_as_default "永久链接")
>
> 启用后，将 TSV 中的空白输入字段替换为默认值。对于复杂的默认表达式，也`input_format_defaults_for_omitted_fields`必须启用。
>
> 默认禁用。
>
> ## input_format_null_as_default [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-input_format_null_as_default "永久链接")
>
> 如果输入数据包含`NULL`，则启用或禁用默认值，如果输入数据包含，则对应列的数据类型不包含`Nullable(T)`（对于文本输入格式）。
>
> ## input_format_skip_unknown_fields [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-input_format_skip_unknown_fields "永久链接")
>
> 启用或禁用跳过多余数据的插入。
>
> 写入数据时，如果输入数据包含目标表中不存在的列，则 ClickHouse 会引发异常。如果启用了跳过，则 ClickHouse 不会插入额外的数据，也不会引发异常。
>
> 支持的格式：
>
> - [JSONEachRow](https://clickhouse.yandex/docs/en/interfaces/formats/#jsoneachrow)
> - [CSVWithNames](https://clickhouse.yandex/docs/en/interfaces/formats/#csvwithnames)
> - [TabSeparatedWithNames](https://clickhouse.yandex/docs/en/interfaces/formats/#tabseparatedwithnames)
> - [TSKV](https://clickhouse.yandex/docs/en/interfaces/formats/#tskv)
>
> 可能的值：
>
> - 0-禁用。
> - 1-启用。
>
> 默认值：0
>
> ## input_format_import_nested_json [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-input_format_import_nested_json "永久链接")
>
> 启用或禁用带有嵌套对象的 JSON 数据插入。
>
> 支持的格式：
>
> - [JSONEachRow](https://clickhouse.yandex/docs/en/interfaces/formats/#jsoneachrow)
>
> 可能的值：
>
> - 0-禁用。
> - 1-启用。
>
> 默认值：0
>
> **也可以看看**
>
> - [嵌套结构](https://clickhouse.yandex/docs/en/interfaces/formats/#jsoneachrow-nested)与`JSONEachRow`格式的[用法](https://clickhouse.yandex/docs/en/interfaces/formats/#jsoneachrow-nested)。
>
> ## input_format_with_names_use_header [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-input_format_with_names_use_header "永久链接")
>
> 启用或禁用在插入数据时检查列顺序。
>
> 为了提高插入性能，如果您确定输入数据的列顺序与目标表中的顺序相同，建议禁用此检查。
>
> 支持的格式：
>
> - [CSVWithNames](https://clickhouse.yandex/docs/en/interfaces/formats/#csvwithnames)
> - [TabSeparatedWithNames](https://clickhouse.yandex/docs/en/interfaces/formats/#tabseparatedwithnames)
>
> 可能的值：
>
> - 0-禁用。
> - 1-启用。
>
> 默认值：1。
>
> ## date_time_input_format [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-date_time_input_format "永久链接")
>
> 允许选择日期和时间的文本表示解析器。
>
> 该设置不适用于[日期和时间功能](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/)。
>
> 可能的值：
>
> - `'best_effort'` —启用扩展解析。
>
>   ClickHouse 可以解析基本`YYYY-MM-DD HH:MM:SS`格式以及所有[ISO 8601](https://en.wikipedia.org/wiki/ISO_8601)日期和时间格式。例如，`'2018-06-08T01:02:03.000Z'`。
>
> - `'basic'` —使用基本解析器。
>
>   ClickHouse 只能解析基本`YYYY-MM-DD HH:MM:SS`格式。例如，`'2019-08-20 10:18:56'`。
>
> 默认值：`'basic'`。
>
> **也可以看看**
>
> - [DateTime 数据类型。](https://clickhouse.yandex/docs/en/data_types/datetime/)
> - [处理日期和时间的功能。](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/)
>
> ## join_default_strictness [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-join_default_strictness "永久链接")
>
> 设置[JOIN 子句的](https://clickhouse.yandex/docs/en/query_language/select/#select-join)默认严格性。
>
> 可能的值：
>
> - `ALL`—如果右表具有多个匹配的行，则 ClickHouse  从匹配的行创建[笛卡尔乘积](https://en.wikipedia.org/wiki/Cartesian_product)。这是`JOIN`标准 SQL  的正常行为。
> - `ANY`—如果右表具有多个匹配的行，则仅连接找到的第一个行。如果右表只有一个匹配行，则`ANY`和的结果`ALL`相同。
> - `ASOF` —用于加入不确定匹配的序列。
> - `Empty string`-如果`ALL`还是`ANY`未在查询中指定的，ClickHouse 抛出异常。
>
> 默认值：`ALL`。
>
> ## join_any_take_last_row [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-join_any_take_last_row "永久链接")
>
> `ANY`严格更改联接操作的行为。
>
> 注意
>
> 此设置仅适用于`JOIN`使用[Join](https://clickhouse.yandex/docs/en/operations/table_engines/join/)引擎表的操作。
>
> 可能的值：
>
> - 0-如果右表具有多个匹配行，则仅找到的第一个联接。
> - 1-如果右表具有多个匹配行，则仅连接找到的最后一个。
>
> 默认值：0
>
> **也可以看看**
>
> - [JOIN 子句](https://clickhouse.yandex/docs/en/query_language/select/#select-join)
> - [联接表引擎](https://clickhouse.yandex/docs/en/operations/table_engines/join/)
> - [join_default_strictness](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-join_default_strictness)
>
> ## join_use_nulls [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-join_use_nulls "永久链接")
>
> 设置[JOIN](https://clickhouse.yandex/docs/en/query_language/select/)行为的类型。合并表格时，可能会出现空单元格。ClickHouse 根据此设置以不同的方式填充它们。
>
> 可能的值：
>
> - 0-空单元格用相应字段类型的默认值填充。
> - 1- `JOIN`行为与标准 SQL 相同。相应字段的类型将转换为[Nullable](https://clickhouse.yandex/docs/en/data_types/nullable/#data_type-nullable)，并且空单元格将填充[NULL](https://clickhouse.yandex/docs/en/query_language/syntax/)。
>
> 默认值：0
>
> ## join_any_take_last_row [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-join_any_take_last_row "永久链接")
>
> 更改的行为`ANY JOIN`。禁用后，`ANY JOIN`将找到找到的第一行密钥。启用后，`ANY JOIN`如果同一键有多个行，则采用最后匹配的行。该设置仅在[Join table engine 中使用](https://clickhouse.yandex/docs/en/operations/table_engines/join/)。
>
> 可能的值：
>
> - 0-禁用。
> - 1-启用。
>
> 默认值：1。
>
> ## max_block_size [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#max-block-size "永久链接")
>
> 在 ClickHouse 中，数据由块（列部分的集合）处理。单个块的内部处理周期足够有效，但是每个块都有明显的开销。该`max_block_size`设置建议从表中加载什么大小的块（以行数为单位）。块的大小不应太小，以使每个块上的开销仍然很明显，但又不要太大，以便在处理第一个块后快速完成带有 LIMIT 的查询。目的是避免在多个线程中提取大量列时占用过多内存，并至少保留一些缓存局部性。
>
> 默认值：65,536
>
> `max_block_size`并非总是从表中加载大小为的块。如果很明显需要检索较少的数据，则处理较小的块。
>
> ## preferred_block_size_bytes [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#preferred-block-size-bytes "永久链接")
>
> 与的用途相同`max_block_size`，但是它通过将建议的块大小调整为适合块中的行数来设置字节大小。但是，块大小不能超过`max_block_size`行。默认值：1,000,000。仅在从 MergeTree 引擎读取时有效。
>
> ## merge_tree_uniform_read_distribution [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#setting-merge_tree_uniform_read_distribution "永久链接")
>
> 从[MergeTree \*](https://clickhouse.yandex/docs/en/operations/table_engines/mergetree/)表中读取时，ClickHouse 使用多个线程。此设置打开/关闭工作线程上读取任务的均匀分布。均匀分布的算法旨在使`SELECT`查询中所有线程的执行时间大致相等。
>
> 可能的值：
>
> - 0-不使用统一读取分布。
> - 1-使用统一读取分配。
>
> 默认值：1。
>
> ## merge_tree_min_rows_for_concurrent_read [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#setting-merge_tree_min_rows_for_concurrent_read "永久链接")
>
> 如果要从[MergeTree \*](https://clickhouse.yandex/docs/en/operations/table_engines/mergetree/)表的文件中读取的行数超过了，`merge_tree_min_rows_for_concurrent_read`那么 ClickHouse 会尝试在多个线程上从该文件执行并发读取。
>
> 可能的值：
>
> - 任何正整数。
>
> 默认值：163840
>
> ## merge_tree_min_bytes_for_concurrent_read [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#setting-merge_tree_min_bytes_for_concurrent_read "永久链接")
>
> 如果要从[MergeTree \* -engine](https://clickhouse.yandex/docs/en/operations/table_engines/mergetree/)表的一个文件读取的字节数超过了，`merge_tree_min_bytes_for_concurrent_read`则 ClickHouse 会尝试在多个线程上从该文件执行并发读取。
>
> 可能的值：
>
> - 任何正整数。
>
> 默认值：240×1024×1024。
>
> ## merge_tree_min_rows_for_seek [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#setting-merge_tree_min_rows_for_seek "永久链接")
>
> 如果要在一个文件中读取的两个数据块之间的距离小于`merge_tree_min_rows_for_seek`行，则 ClickHouse 不会搜索文件，而是顺序读取数据。
>
> 可能的值：
>
> - 任何正整数。
>
> 默认值：0
>
> ## merge_tree_min_bytes_for_seek [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#setting-merge_tree_min_bytes_for_seek "永久链接")
>
> 如果要在一个文件中读取的两个数据块之间的距离小于`merge_tree_min_bytes_for_seek`行，则 ClickHouse 不会搜索文件，而是顺序读取数据。
>
> 可能的值：
>
> - 任何正整数。
>
> 默认值：0
>
> ## merge_tree_coarse_index_granularity [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#setting-merge_tree_coarse_index_granularity "永久链接")
>
> 搜索数据时，ClickHouse 检查索引文件中的数据标记。如果 ClickHouse 发现所需键在某个范围内，则会将该范围划分为多个`merge_tree_coarse_index_granularity`子范围，然后在该范围内递归搜索所需键。
>
> 可能的值：
>
> - 任何正偶数整数。
>
> 默认值：8。
>
> ## merge_tree_max_rows_to_use_cache [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#setting-merge_tree_max_rows_to_use_cache "永久链接")
>
> 如果 ClickHouse `merge_tree_max_rows_to_use_cache`在一个查询中读取的行数不止，则不使用未压缩块的缓存。该[uncompressed_cache_size](https://clickhouse.yandex/docs/en/operations/server_settings/settings/#server-settings-uncompressed_cache_size)服务器设置定义未压缩块的高速缓存的大小。
>
> 未压缩块的高速缓存存储为查询提取的数据。ClickHouse 使用此缓存来加快对重复的小型查询的响应。此设置通过读取大量数据的查询来保护高速缓存免受垃圾破坏。
>
> 可能的值：
>
> - 任何正整数。
>
> 默认值：128×8192。
>
> ## merge_tree_max_bytes_to_use_cache [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#setting-merge_tree_max_bytes_to_use_cache "永久链接")
>
> 如果 ClickHouse `merge_tree_max_bytes_to_use_cache`在一个查询中读取的字节数多，则不会使用未压缩块的缓存。该[uncompressed_cache_size](https://clickhouse.yandex/docs/en/operations/server_settings/settings/#server-settings-uncompressed_cache_size)服务器设置定义未压缩块的高速缓存的大小。
>
> 未压缩块的高速缓存存储为查询提取的数据。ClickHouse 使用此缓存来加快对重复的小型查询的响应。此设置通过读取大量数据的查询来保护高速缓存免受垃圾破坏。
>
> 可能的值：
>
> - 任何正整数。
>
> 默认值：1920×1024×1024。
>
> ## min_bytes_to_use_direct_io [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-min_bytes_to_use_direct_io "永久链接")
>
> 使用直接 I / O 访问存储磁盘所需的最小数据量。
>
> 从表格读取数据时，ClickHouse 使用此设置。如果要读取的所有数据的总存储量超过`min_bytes_to_use_direct_io`字节，则 ClickHouse 会使用该`O_DIRECT`选项从存储磁盘读取数据。
>
> **可能的值**
>
> - 0-直接 I / O 被禁用。
> - 正整数。
>
> **默认值**：0
>
> ## log_queries [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-log-queries "永久链接")
>
> 设置查询日志。
>
> 以此设置发送到 ClickHouse 的查询将根据[query_log](https://clickhouse.yandex/docs/en/operations/server_settings/settings/#server_settings-query-log)服务器配置参数中的规则记录。
>
> **范例**：
>
> log_queries = 1
>
> ## max_insert_block_size [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-max_insert_block_size "永久链接")
>
> 插入表中要形成的块的大小。此设置仅在服务器构成块的情况下适用。例如，对于通过 HTTP 接口的 INSERT，服务器解析数据格式并形成指定大小的块。但是，当使用 clickhouse-client 时，客户端会解析数据本身，并且服务器上的“ max_insert_block_size”设置不会影响插入的块的大小。使用 INSERT SELECT 时，该设置也没有目的，因为数据是使用 SELECT 之后形成的相同块插入的。
>
> 默认值：1,048,576
>
> 默认值比略大`max_block_size`。其原因是因为某些表引擎（`*MergeTree`）在磁盘上为每个插入的块形成了一个数据部分，这是一个相当大的实体。类似地，`*MergeTree`表在插入期间对数据进行排序，并且足够大的块大小允许对 RAM 中的更多数据进行排序。
>
> ## max_replica_delay_for_distributed_queries [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-max_replica_delay_for_distributed_queries "永久链接")
>
> 对分布式查询禁用滞后副本。请参见“ [复制](https://clickhouse.yandex/docs/en/operations/table_engines/replication/) ”。
>
> 以秒为单位设置时间。如果副本滞后于设置值，则不使用该副本。
>
> 默认值：300。
>
> `SELECT`从指向复制表的分布式表执行时使用。
>
> ## max_threads 的[¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-max_threads "永久链接")
>
> 查询处理线程的最大数量，不包括用于从远程服务器检索数据的线程（请参见“ max_distributed_connections”参数）。
>
> 此参数适用于并行执行查询处理管道相同阶段的线程。例如，当从表中读取数据时，如果可以使用函数求值，使用 WHERE 进行过滤并使用至少“ max_threads”个线程并行地为 GROUP BY 进行预聚合，则可以使用“ max_threads”。
>
> 默认值：物理 CPU 内核数。
>
> 如果通常一次在服务器上运行少于一个 SELECT 查询，则将此参数设置为稍小于处理器核心实际数量的值。
>
> 对于由于 LIMIT 而快速完成的查询，可以设置较低的“ max_threads”。例如，如果每个块中都有必要的条目数，并且 max_threads = 8，则将检索 8 个块，尽管仅读取一个块就足够了。
>
> `max_threads`值越小，消耗的内存越少。
>
> ## max_compress_block_size [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#max-compress-block-size "永久链接")
>
> 在压缩以写入表之前，未压缩数据块的最大大小。默认情况下为 1,048,576（1 MiB）。如果减小大小，则由于高速缓存局部性，压缩率将显着降低，压缩和解压缩速度会略有提高，并且内存消耗也会减少。通常没有任何理由更改此设置。
>
> 不要将压缩块（由字节组成的内存块）与查询处理块（表中的一组行）混淆。
>
> ## min_compress_block_size [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#min-compress-block-size "永久链接")
>
> 对于[MergeTree](https://clickhouse.yandex/docs/en/operations/table_engines/mergetree/)表。为了减少处理查询时的延迟，如果块的大小至少为'min_compress_block_size'，则在写入下一个标记时将压缩该块。默认值为 65,536。
>
> 如果未压缩的数据小于“ max_compress_block_size”，则块的实际大小不小于此值且不小于一个标记的数据量。
>
> 让我们看一个例子。假设在创建表期间将“ index_granularity”设置为 8192。
>
> 我们正在编写一个 UInt32 类型的列（每个值 4 个字节）。当写入 8192 行时，总计将为 32 KB 数据。由于 min_compress_block_size = 65,536，因此每两个标记将形成一个压缩块。
>
> 我们正在编写一个 String 类型的 URL 列（每个值的平均大小为 60 个字节）。当写入 8192 行时，平均值将略小于 500 KB 数据。由于大于 65,536，将为每个标记形成一个压缩块。在这种情况下，从磁盘读取单个标记范围内的数据时，不会解压缩多余的数据。
>
> 通常没有任何理由更改此设置。
>
> ## mark_cache_min_lifetime [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-mark_cache_min_lifetime "永久链接")
>
> 如果超出了[mark_cache_size](https://clickhouse.yandex/docs/en/operations/server_settings/settings/#server-mark-cache-size)设置的值，则仅删除早于 mark_cache_min_lifetime 秒的记录。如果主机的 RAM 量较少，则可以降低此参数。
>
> 默认值：10000 秒。
>
> ## max_query_size [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-max_query_size "永久链接")
>
> 可以带到 RAM 以使用 SQL 解析器进行解析的查询的最大部分。INSERT 查询还包含由单独的流解析器（消耗 O（1）RAM）处理的 INSERT 数据，该数据未包含在此限制中。
>
> 默认值：256 KiB。
>
> ## interactive_delay [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#interactive-delay "永久链接")
>
> 用于检查请求执行是否已被取消，发送所述进度在微秒的时间间隔。
>
> 默认值：100,000（检查取消和发送进度十倍每秒）。
>
> ## connect_timeout，receive_timeout，send_timeout [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#connect-timeout-receive-timeout-send-timeout "永久链接")
>
> 用于与客户端通信的套接字上的超时（以秒为单位）。
>
> 预设值：10、300、300。
>
> ## POLL_INTERVAL [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#poll-interval "永久链接")
>
> 将等待循环锁定指定的秒数。
>
> 默认值：10
>
> ## max_distributed_connections [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#max-distributed-connections "永久链接")
>
> 与远程服务器的并发连接的最大数量，用于将单个查询分布式处理到单个 Distributed 表。我们建议设置一个不小于集群中服务器数量的值。
>
> 默认值：1024
>
> 以下参数仅在创建分布式表（以及启动服务器）时使用，因此没有理由在运行时更改它们。
>
> ## distributed_connections_pool_size [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#distributed-connections-pool-size "永久链接")
>
> 与远程服务器的并发连接的最大数量，用于对单个 Distributed 表进行所有查询的分布式处理。我们建议设置一个不小于集群中服务器数量的值。
>
> 默认值：1024
>
> ## connect_timeout_with_failover_ms [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#connect-timeout-with-failover-ms "永久链接")
>
> 如果群集定义中使用了“ shard”和“ replica”部分，则连接到分布式表引擎的远程服务器的超时时间（以毫秒为单位）。如果不成功，则尝试进行几次尝试以连接到各种副本。
>
> 默认值：50。
>
> ## connections_with_failover_max_tries [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#connections-with-failover-max-tries "永久链接")
>
> 分布式表引擎与每个副本的最大连接尝试次数。
>
> 默认值：3。
>
> ## 极端[¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#extremes "永久链接")
>
> 是否计算极值（查询结果列中的最小值和最大值）。接受 0 或 1。默认情况下，0（禁用）。有关更多信息，请参见“极限值”部分。
>
> ## use_uncompressed_cache [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#setting-use_uncompressed_cache "永久链接")
>
> 是否使用未压缩块的缓存。接受 0 或 1。默认情况下，0（禁用）。当使用大量短查询时，使用未压缩的缓存（仅适用于 MergeTree 系列中的表）可以显着减少延迟并提高吞吐量。为频繁发送简短请求的用户启用此设置。还请注意[uncompressed_cache_size](https://clickhouse.yandex/docs/en/operations/server_settings/settings/#server-settings-uncompressed_cache_size)配置参数（仅在配置文件中设置）–未压缩的缓存块的大小。默认情况下为 8 GiB。未压缩的缓存将根据需要填充，并且使用最少的数据将被自动删除。
>
> 对于读取至少一些数据量（一百万行或更多）的查询，未压缩的缓存将自动禁用，以节省真正小的查询的空间。这意味着您可以始终将“ use_uncompressed_cache”设置设为 1。
>
> ## replace_running_query [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#replace-running-query "永久链接")
>
> 使用 HTTP 接口时，可以传递'query_id'参数。这是用作查询标识符的任何字符串。如果此时已经存在来自具有相同“ query_id”的相同用户的查询，则行为取决于“ replace_running_query”参数。
>
> `0` （默认）–引发异常（如果已经在运行具有相同“ query_id”的查询，则不允许查询运行）。
>
> `1` –取消旧查询，然后开始运行新查询。
>
> Yandex.Metrica 使用将此参数设置为 1 来实施针对细分条件的建议。输入下一个字符后，如果旧查询尚未完成，则应将其取消。
>
> ## stream_flush_interval_ms [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#stream-flush-interval-ms "永久链接")
>
> 作品与在超时的情况下的流表，或者当一个线程生成[max_insert_block_size](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-max_insert_block_size)行。
>
> 默认值为 7500。
>
> 值越小，数据刷新到表的频率越高。将该值设置得太低会导致性能下降。
>
> ## load_balancing [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-load_balancing "永久链接")
>
> 指定用于分布式查询处理的副本选择算法。
>
> ClickHouse 支持以下选择副本的算法：
>
> - [随机](https://clickhouse.yandex/docs/en/operations/settings/settings/#load_balancing-random)（默认）
> - [最近的主机名](https://clickhouse.yandex/docs/en/operations/settings/settings/#load_balancing-nearest_hostname)
> - [为了](https://clickhouse.yandex/docs/en/operations/settings/settings/#load_balancing-in_order)
> - [首先还是随机](https://clickhouse.yandex/docs/en/operations/settings/settings/#load_balancing-first_or_random)
>
> ### 随机（默认）[¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#load_balancing-random "永久链接")
>
> load_balancing = 随机
>
> 计算每个副本的错误数量。查询将以最少的错误发送到副本，如果存在多个错误，则发送到其中任何一个。缺点：不考虑服务器的邻近性；如果副本具有不同的数据，则您还将获得不同的数据。
>
> ### 最近的主机名[¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#load_balancing-nearest_hostname "永久链接")
>
> load_balancing = 最近的主机名
>
> 计算每个副本的错误数量。每隔 5 分钟，错误数量将被 2 整除。因此，最近一次使用指数平滑计算了错误数量。如果有一个副本的错误数量最少（即，最近在其他副本上发生的错误），则将查询发送给它。如果有多个副本且错误的最小数量相同，则查询将以与配置文件中的服务器主机名最相似的主机名发送到副本（相同位置的不同字符数，最多两个主机名的最小长度）。
>
> 例如，example01-01-1 和 example01-01-2.yandex.ru 在一个位置上是不同的，而 example01-01-1 和 example01-02-2 在两个位置上是不同的。这种方法看似原始，但它不需要有关网络拓扑的外部数据，也不需要比较 IP 地址，这对于我们的 IPv6 地址而言很复杂。
>
> 因此，如果存在等效的副本，则首选名称最接近的副本。我们还可以假设在将查询发送到同一服务器时，在没有故障的情况下，分布式查询也将到达相同的服务器。因此，即使将不同的数据放在副本上，查询也将返回几乎相同的结果。
>
> ### 按顺序[¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#load_balancing-in_order "永久链接")
>
> load_balancing = 顺序
>
> 具有相同错误数量的副本将以与配置中指定的顺序相同的顺序进行访问。当您确切知道哪个副本更可取时，此方法适用。
>
> ### 第一或随机[¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#load_balancing-first_or_random "永久链接")
>
> load_balancing = first_or_random
>
> 此算法选择集合中的第一个副本，如果第一个副本不可用，则选择一个随机副本。它在交叉复制拓扑设置中有效，但在其他配置中无效。
>
> 该`first_or_random`算法解决了算法的问题`in_order`。使用`in_order`，如果一个副本掉线，则下一个副本将加倍负载，而其余副本则处理通常的流量。使用该`first_or_random`算法时，负载在仍然可用的副本之间平均分配。
>
> ## prefer_localhost_replica [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-prefer_localhost_replica "永久链接")
>
> 处理分布式查询时，最好使用 localhost 副本启用/禁用。
>
> 可能的值：
>
> - 1-ClickHouse 始终向本地副本发送查询（如果存在）。
> - 0-ClickHouse 使用[load_balancing](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-load_balancing)设置指定的平衡策略。
>
> 默认值：1。
>
> 警告
>
> 如果使用[max_parallel_replicas，请](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-max_parallel_replicas)禁用此设置。
>
> ## totals_mode [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#totals-mode "永久链接")
>
> 存在 HAVING 时以及存在 max_rows_to_group_by 和 group_by_overflow_mode ='any'时如何计算 TOTALS。请参见“共计修饰符”部分。
>
> ## totals_auto_threshold [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#totals-auto-threshold "永久链接")
>
> 的门槛`totals_mode = 'auto'`。请参见“共计修饰符”部分。
>
> ## max_parallel_replicas [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-max_parallel_replicas "永久链接")
>
> 执行查询时，每个分片的最大副本数。为了保持一致性（以获取同一数据拆分的不同部分），此选项仅在设置采样键时有效。复制延迟不受控制。
>
> ## 编译[¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#compile "永久链接")
>
> 启用查询编译。默认情况下，0（禁用）。
>
> 编译仅用于查询处理管道的一部分：用于聚合的第一阶段（GROUP BY）。如果管道的这一部分已编译，则由于部署周期短和内联聚合函数调用，查询可能会运行得更快。对于具有多个简单聚合函数的查询，可以看到最大的性能改进（在极少数情况下，速度提高了四倍）。通常，性能提升微不足道。在极少数情况下，它可能会减慢查询的执行速度。
>
> ## min_count_to_compile [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#min-count-to-compile "永久链接")
>
> 运行编译之前可能使用已编译代码块的次数。默认情况下为 3。对于测试，该值可以设置为 0：编译同步运行，并且查询在继续执行之前等待编译过程的结束。对于所有其他情况，请使用以 1 开头的值。编译通常需要大约 5-10 秒。如果该值为 1 或更大，则在单独的线程中异步进行编译。结果准备就绪后将立即使用，包括当前正在运行的查询。
>
> 对于查询中使用的聚合函数和 GROUP BY 子句中的键类型的每种不同组合，都需要编译后的代码。编译结果以.so 文件的形式保存在生成目录中。编译结果的数量没有限制，因为它们不占用太多空间。重新启动服务器后，将使用旧结果，除非升级服务器-在这种情况下，将删除旧结果。
>
> ## output_format_json_quote_64bit_integers [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#session_settings-output_format_json_quote_64bit_integers "永久链接")
>
> 如果该值为 true，则在使用 JSON \* Int64 和 UInt64 格式时（以与大多数 JavaScript 实现兼容），引号中会出现整数。否则，将输出不带引号的整数。
>
> ## format_csv_delimiter [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-format_csv_delimiter "永久链接")
>
> 字符被解释为 CSV 数据中的定界符。默认情况下，定界符为`,`。
>
> ## input_format_csv_unquoted_null_literal_as_null [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-input_format_csv_unquoted_null_literal_as_null "永久链接")
>
> 对于 CSV 输入格式，启用或禁用未引用解析`NULL`为原义字词（为的同义词`\N`）。
>
> ## insert_quorum [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-insert_quorum "永久链接")
>
> 启用仲裁写入。
>
> - 如果为`insert_quorum < 2`，则仲裁写入被禁用。
> - 如果为`insert_quorum >= 2`，则启用仲裁写入。
>
> 默认值：0
>
> **法定人数**
>
> `INSERT`仅当 ClickHouse `insert_quorum`在期间成功将数据正确写入副本的时，成功`insert_quorum_timeout`。如果由于任何原因而成功写入的副本数量未达到`insert_quorum`，则认为写入失败，并且 ClickHouse 将从已写入数据的所有副本中删除插入的块。
>
> 仲裁中的所有副本都是一致的，即它们包含所有先前`INSERT`查询的数据。该`INSERT`序列是线性化。
>
> 读取从写入的数据时`insert_quorum`，可以使用[select_sequential_consistency](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-select_sequential_consistency)选项。
>
> **ClickHouse 生成异常**
>
> - 如果查询时的可用副本数小于`insert_quorum`。
> - 当尚未将前一个块插入`insert_quorum`副本的时尝试写入数据。如果用户尝试`INSERT`在完成前一个操作之前执行一个操作，则可能会发生这种情况`insert_quorum`。
>
> **另请参阅以下参数：**
>
> - [insert_quorum_timeout](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-insert_quorum_timeout)
> - [select_sequential_consistency](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-select_sequential_consistency)
>
> ## insert_quorum_timeout [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-insert_quorum_timeout "永久链接")
>
> 仲裁写入超时（以秒为单位）。如果超时已过去并且尚未发生任何写入，则 ClickHouse 将生成异常，并且客户端必须重复查询才能将同一块写入相同或任何其他副本。
>
> 默认值：60 秒。
>
> **另请参阅以下参数：**
>
> - [insert_quorum](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-insert_quorum)
> - [select_sequential_consistency](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-select_sequential_consistency)
>
> ## select_sequential_consistency [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-select_sequential_consistency "永久链接")
>
> 启用或禁用`SELECT`查询的顺序一致性：
>
> 可能的值：
>
> - 0-禁用。
> - 1-启用。
>
> 默认值：0
>
> **用法**
>
> 启用顺序一致性后，ClickHouse 允许客户端`SELECT`仅对那些包含`INSERT`使用执行的所有先前查询中的数据的副本执行查询`insert_quorum`。如果客户端引用部分副本，则 ClickHouse 将生成一个异常。SELECT 查询将不包括尚未写入副本仲裁的数据。
>
> **也可以看看**
>
> - [insert_quorum](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-insert_quorum)
> - [insert_quorum_timeout](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-insert_quorum_timeout)
>
> ## max_network_bytes [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-max_network_bytes "永久链接")
>
> 限制执行查询时通过网络接收或传输的数据量（以字节为单位）。此设置适用于每个单独的查询。
>
> 可能的值：
>
> - 正整数。
> - 0-数据音量控制已禁用。
>
> 默认值：0
>
> ## max_network_bandwidth [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-max_network_bandwidth "永久链接")
>
> 限制通过每秒字节数在网络上进行数据交换的速度。此设置适用于每个查询。
>
> 可能的值：
>
> - 正整数。
> - 0-禁用带宽控制。
>
> 默认值：0
>
> ## max_network_bandwidth_for_user [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-max_network_bandwidth_for_user "永久链接")
>
> 限制通过每秒字节数在网络上进行数据交换的速度。此设置适用于单个用户执行的所有同时运行的查询。
>
> 可能的值：
>
> - 正整数。
> - 0-禁用数据速度控制。
>
> 默认值：0
>
> ## max_network_bandwidth_for_all_users [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-max_network_bandwidth_for_all_users "永久链接")
>
> 限制每秒通过字节在网络上交换数据的速度。此设置适用于服务器上所有同时运行的查询。
>
> 可能的值：
>
> - 正整数。
> - 0-禁用数据速度控制。
>
> 默认值：0
>
> ## allow_experimental_cross_to_join_conversion [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-allow_experimental_cross_to_join_conversion "永久链接")
>
> 启用或禁用：
>
> 1.  重写查询以将联接从语法加逗号到`JOIN ON/USING`语法。如果设置值为 0，则 ClickHouse 不会使用使用逗号的语法来处理查询，并且会引发异常。
> 2.  转换`CROSS JOIN`到`INNER JOIN`，如果`WHERE`条件允许的话。
>
> 可能的值：
>
> - 0-禁用。
> - 1-启用。
>
> 默认值：1。
>
> **也可以看看**
>
> - [多个联接](https://clickhouse.yandex/docs/en/query_language/select/#select-join)
>
> ## count_distinct_implementation [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-count_distinct_implementation "永久链接")
>
> 指定`uniq*`应使用哪些函数来执行[COUNT（DISTINCT ...）](https://clickhouse.yandex/docs/en/query_language/agg_functions/reference/#agg_function-count)构造。
>
> 可能的值：
>
> - [优衣库](https://clickhouse.yandex/docs/en/query_language/agg_functions/reference/#agg_function-uniq)
> - [uniqCombined](https://clickhouse.yandex/docs/en/query_language/agg_functions/reference/#agg_function-uniqcombined)
> - [uniqCombined64](https://clickhouse.yandex/docs/en/query_language/agg_functions/reference/#agg_function-uniqcombined64)
> - [uniqHLL12](https://clickhouse.yandex/docs/en/query_language/agg_functions/reference/#agg_function-uniqhll12)
> - [uniqExact](https://clickhouse.yandex/docs/en/query_language/agg_functions/reference/#agg_function-uniqexact)
>
> 默认值：`uniqExact`。
>
> ## skip_unavailable_shards [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-skip_unavailable_shards "永久链接")
>
> 启用或禁用无提示跳过：
>
> - 节点，如果其名称无法通过 DNS 解析。
>
>   禁用跳过功能后，ClickHouse 要求[群集配置](https://clickhouse.yandex/docs/en/operations/server_settings/settings/#server_settings_remote_servers)中的所有节点都可以通过 DNS 解析。否则，在尝试对群集执行查询时，ClickHouse 会引发异常。
>
>   如果启用了跳过，ClickHouse 会将未解析的节点视为不可用，并在每次连接尝试时尝试解析它们。由于用户可以指定错误的节点名称，而 ClickHouse 不会报告该错误，因此这种行为会带来群集配置错误的风险。但是，这在具有动态 DNS 的系统（例如[Kubernetes）中很有用](https://kubernetes.io/)，因为在停机期间节点可能无法解析，这也不是错误。
>
> - 分片（如果没有可用的分片副本）。
>
>   禁用跳过后，ClickHouse 会引发异常。
>
>   启用跳过后，ClickHouse 将返回部分答案，并且不会报告有关节点可用性的问题。
>
> 可能的值：
>
> - 1-启用跳过。
> - 0-禁用跳过。
>
> 默认值：0
>
> ## optimize_throw_if_noop [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#setting-optimize_throw_if_noop "永久链接")
>
> 如果[OPTIMIZE](https://clickhouse.yandex/docs/en/query_language/misc/#misc_operations-optimize)查询未执行合并，则启用或禁用引发异常。
>
> 默认情况下，`OPTIMIZE`即使没有执行任何操作，也会成功返回。通过此设置，您可以区分这些情况，并在异常消息中获取原因。
>
> 可能的值：
>
> - 1-启用引发异常。
> - 0-禁用引发异常。
>
> 默认值：0
>
> ## distributed_replica_error_half_life [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-distributed_replica_error_half_life "永久链接")
>
> - 类型：秒
> - 默认值：60 秒
>
> 控制将分布式表的错误快速归零的方式。假设当前某个副本在一段时间内不可用并且累积了 5 个错误，并且 distributed_replica_error_half_life 设置为 1 秒，则自上次错误以来，该副本将在 3 秒内恢复为正常。
>
> **也可以看看**
>
> - [表引擎分布式](https://clickhouse.yandex/docs/en/operations/table_engines/distributed/)
> - [`distributed_replica_error_cap`](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-distributed_replica_error_cap)
>
> ## distributed_replica_error_cap [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-distributed_replica_error_cap "永久链接")
>
> - 类型：unsigned int
> - 默认值：1000
>
> 每个副本的错误计数都以该值为上限，以防止单个副本累积到很多错误。
>
> **也可以看看**
>
> - [表引擎分布式](https://clickhouse.yandex/docs/en/operations/table_engines/distributed/)
> - [`distributed_replica_error_half_life`](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-distributed_replica_error_half_life)
>
> ## os_thread_priority [¶](https://clickhouse.yandex/docs/en/operations/settings/settings/#setting-os_thread_priority "永久链接")
>
> 为执行查询的线程设置优先级（[nice](<https://en.wikipedia.org/wiki/Nice_(Unix)>)）。当选择在每个可用 CPU 内核上运行的下一个线程时，OS 调度程序会考虑此优先级。
>
> 警告
>
> 要使用此设置，您需要设置`CAP_SYS_NICE`功能。该`clickhouse-server`软件包将在安装过程中进行设置。某些虚拟环境不允许您设置`CAP_SYS_NICE`功能。在这种情况下，请`clickhouse-server`在开始时显示有关此消息。
>
> 可能的值：
>
> - 您可以在范围内设置值`[-20, 19]`。
>
> 较低的值表示较高的优先级。具有低`nice`优先级值的线程比具有高优先级值的线程执行得更频繁。高值对于长时间运行的非交互式查询是更可取的，因为它允许它们在到达时快速放弃资源，而转向短交互式查询。
>
> 默认值：0

# 设定设定档

设置配置文件是一组以相同名称分组的设置。每个 ClickHouse 用户都有一个个人资料。要在配置文件中应用所有设置，请设置`profile`设置。

例：

安装`web`配置文件。

SET 个人资料 = “网络”

设置配置文件在用户配置文件中声明。通常是这样`users.xml`。

例：

<！-设置配置文件-\>
<配置文件\>
<！-默认设置-\>
<默认\>
<！-运行单个查询时的最大线程数。-\>
<max_threads> 8 </ max_threads>
</ default>

    <！ -设置为从用户接口quries - >
    <网络\>
        <max\_rows\_to_read>十亿</ max\_rows\_to_read>
        <max\_bytes\_to_read>千亿</ max\_bytes\_to_read>

        <max\_rows\_to\_group\_by> 1000000 </ max\_rows\_to\_group\_by>
        <group\_by\_overflow_mode>任意</ group\_by\_overflow_mode>

        <max\_rows\_to_sort> 1000000 </ max\_rows\_to_sort>
        <max\_bytes\_to_sort> 1000000000 </ max\_bytes\_to_sort>

        <max\_result\_rows> 100000 </ max\_result\_rows>
        <max\_result\_bytes>亿</ max\_result\_bytes>
        <result\_overflow\_mode>断裂</ result\_overflow\_mode>

        <max\_execution\_time> 600 </ max\_execution\_time>
        <min\_execution\_speed> 1000000 </ min\_execution\_speed>
        <timeout\_before\_checking\_execution\_speed> 15 </ timeout\_before\_checking\_execution\_speed>

        <max\_columns\_to_read> 25 </ max\_columns\_to_read>
        <max\_temporary\_columns> 100 </ max\_temporary\_columns>
        <max\_temporary\_non\_const\_columns> 50 </ max\_temporary\_non\_const\_columns>

        <max\_subquery\_depth> 2 </ max\_subquery\_depth>
        <max\_pipeline\_depth> 25 </ max\_pipeline\_depth>
        <max\_ast\_depth> 50 </ max\_ast\_depth>
        <max\_ast\_elements> 100 </ max\_ast\_elements>

        <readonly> 1 </ readonly>
    </ web>

</ profiles>

该示例指定了两个配置文件：`default`和`web`。该`default`配置文件有一个特殊目的：它必须始终存在，并在启动服务器时应用。换句话说，`default`配置文件包含默认设置。该`web`配置文件是常规配置文件，可以使用`SET`查询或 HTTP 查询中的 URL 参数进行设置。

设置配置文件可以相互继承。要使用继承，请`profile`在配置文件中列出的其他设置之前指示该设置。

# 设置限制

设置约束可以`users`在`user.xml`配置文件的部分中定义，并禁止用户通过`SET`查询更改某些设置。约束定义如下：

<profiles> 
  <user_name> 
    <constraints> 
      <setting\_name\_1> 
        <min> lower_boundary </ min> 
      </ setting\_name\_1> 
      <setting\_name\_2> 
        <max> upper_boundary </ max> 
      </ setting\_name\_2> 
      <setting\_name\_3> 
        <min> lower_boundary </ min> 
        <max> upper_boundary </ max> 
      </ setting\_name\_3> 
      <setting\_name\_4> 
        <readonly /> 
      </ setting\_name\_4> 
    </ constraints> 
  </ user_name> 
</ profiles>

如果用户试图违反约束，则会引发异常，并且设置实际上不会更改。目前支持三种类型的约束：`min`，`max`，`readonly`。的`min`和`max`约束指定上限和下限为数值设置，并且可以组合使用。该`readonly`约束指定用户根本无法更改相应的设置。

**示例：**让`users.xml`包括行：

<profiles> 
  <默认\> 
    <max\_memory\_usage> 10000000000 </ max\_memory\_usage> 
    <force\_index\_by_date> 0 </ force\_index\_by_date>
    ...
    <constraints> 
      <max\_memory\_usage> 
        <min> 5000000000 </ min> 
        <max> 20000000000 </ max> 
      </ max\_memory\_usage> 
      <force\_index\_by_date> 
        <readonly /> 
      </ force\_index\_by_date> 
    </ constraints> 
  </ default> 
</ profiles>

以下查询均引发异常：

SET max_memory_usage = 20000000001 ;
SET max_memory_usage = 4999999999 ;
SET force_index_by_date = 1 ;

代码：452，e.displayText（）= DB :: Exception：设置 max_memory_usage 不应大于 20000000000。
代码：452，e.displayText（）= DB :: Exception：设置 max_memory_usage 不应小于 5000000000。
代码：452，e.displayText（）= DB :: Exception：不应更改设置 force_index_by_date。

**注：**该`default`配置文件有一个特殊的处理：对定义的所有约束`default`轮廓成为默认的约束，所以他们限制所有用户，直到他们为这些用户明确覆盖。

# 用户设置

配置文件的`users`部分`user.xml`包含用户设置。

本`users`节的结构：

<users> 
    <！-如果未指定用户名，则使用“默认”用户。-\> 
    <用户名\> 
        <密码\> </ password> 
        <！-或-\> 
        <password\_sha256\_hex> </ password\_sha256\_hex>

        <networks  incl = “ networks”  replace = “替换” \>
        </ networks>

        <profile> profile_name </ profile>

        <quota>默认</ quota>

        <数据库\>
            <数据库名称\>
                <表名称\>
                    <过滤器>表达式</过滤器\>
                <表名称\>
            </数据库名称\>
        </数据库\>
    </
    用户名称

\> <！-其他用户设置-\> </用户>

### 用户名/密码[¶](https://clickhouse.yandex/docs/en/operations/settings/settings_users/#user-name-password "永久链接")

密码可以以纯文本或 SHA256（十六进制格式）指定。

- 要以纯文本形式分配密码（**不建议使用**），请将其放在一个`password`元素中。

  例如，`<password>qwerty</password>`。密码可以留空。

- 要使用其 SHA256 哈希值分配密码，请将其放在一个`password_sha256_hex`元素中。

  例如，`<password_sha256_hex>65e84be33532fb784c48129675f9eff3a682b27168c0ea744b2cf58ee02337c5</password_sha256_hex>`。

  如何从 shell 生成密码的示例：

  `PASSWORD=$(base64 < /dev/urandom | head -c8); echo "$PASSWORD"; echo -n "$PASSWORD" | sha256sum | tr -d '-'`

  结果的第一行是密码。第二行是对应的 SHA256 哈希。

### USER_NAME /网络[¶](https://clickhouse.yandex/docs/en/operations/settings/settings_users/#user-name-networks "永久链接")

用户可以从中连接到 ClickHouse 服务器的网络列表。

列表的每个元素可以具有以下形式之一：

- `<ip>` — IP 地址或网络掩码。

  例如：`213.180.204.3`，`10.0.0.1/8`，`10.0.0.1/255.255.255.0`，`2a02:6b8::3`，`2a02:6b8::3/64`，`2a02:6b8::3/ffff:ffff:ffff:ffff::`。

- `<host>` \- 主机名。

  范例：`server01.yandex.ru`。

  为了检查访问，将执行 DNS 查询，并将所有返回的 IP 地址与对等地址进行比较。

- `<host_regexp>` —主机名的正则表达式。

  例， `^server\d\d-\d\d-\d\.yandex\.ru$`

  为了检查访问，对同一个地址执行[DNS PTR 查询](https://en.wikipedia.org/wiki/Reverse_DNS_lookup)，然后应用指定的正则表达式。然后，对 PTR 查询的结果执行另一个 DNS 查询，并将所有接收到的地址与对等地址进行比较。我们强烈建议 regexp 以\$结尾。

DNS 请求的所有结果都将缓存，直到服务器重新启动。

**例子**

要为来自任何网络的用户打开访问权限，请指定：

<ip> :: / 0 </ ip>

警告

除非您已正确配置防火墙或服务器未直接连接到 Internet，否则从任何网络打开访问都是不安全的。

要仅从本地主机打开访问，请指定：

<ip> :: 1 </ ip>
<ip> 127.0.0.1 </ ip>

### user_name / [profile¶](https://clickhouse.yandex/docs/en/operations/settings/settings_users/#user-name-profile "永久链接")

您可以为用户分配设置配置文件。设置配置文件在`users.xml`文件的单独部分中配置。有关更多信息，请参阅[设置配置文件](https://clickhouse.yandex/docs/en/operations/settings/settings_profiles/)。

### user_name / [quota¶](https://clickhouse.yandex/docs/en/operations/settings/settings_users/#user-name-quota "永久链接")

配额使您可以跟踪或限制一段时间内的资源使用情况。`quotas`在`users.xml`配置文件的部分中配置配额。

您可以为用户分配配额。有关配额配置的详细说明，请参见[配额](https://clickhouse.yandex/docs/en/operations/quotas/#quotas)。

### 用户名/数据库[¶](https://clickhouse.yandex/docs/en/operations/settings/settings_users/#user-name-databases "永久链接")

在本部分中，您可以限制 ClickHouse 返回的行以供`SELECT`当前用户进行的查询，从而实现基本的行级安全性。

**例**

以下配置强制用户`user1`只能看到`table1`作为`SELECT`查询结果的行，其中`id`字段的值为 1000。

<user1> 
    <databases> 
        <database_name> 
            <table1> 
                <filter> id = 1000 </ filter> 
            </ table1> 
        </ database_name> 
    </ databases> 
</ user1>

所述`filter`可导致任何表达式[UINT8](https://clickhouse.yandex/docs/en/data_types/int_uint/)型值。它通常包含比较和逻辑运算符。`database_name.table1`该用户未返回过滤结果为 0 的行。过滤与`PREWHERE`操作不兼容，并禁用`WHERE→PREWHERE`优化。
