# Restrictions on query complexity

- 它们用于从用户接口提供更安全的执行。
  - 几乎所有的限制都只适用于 `SELECT`。
  - 对于分布式查询处理，限制分别应用于每个服务器。
  - `ClickHouse` 检查 `data part` 的限制，而不是每一行。意味着，可以使用`data part`的一部分数据

对 `maximum amount of something` 的限制可以取值为 `0`，表示 `unrestricted`。

大多数限制还具有 `overflow_mode` 设置，这意味着超出限制时该怎么做。它可以采用以下两个值之一：
  - `throw`
    - `throw` –引发异常（默认）。
  - `break`
    - `break` –停止执行查询并返回部分结果，就像源数据用完了一样。
  - 聚合限制（`group_by_overflow_mode`）也具有值`any`。
    - `any (only for group_by_overflow_mode)` –继续聚合进入集合的`keys`, 但是不添加新`key` 到集合。

## max_memory_usage

- 用于在单个服务器上运行查询的最大 RAM 量。
  - 在默认配置文件中，最大为 10 GB。
    - 该设置不考虑可用内存量或计算机上的总内存量。
    - 该限制适用于单个服务器内的单个查询。
    - 您可以`SHOW PROCESSLIST`用来查看每个查询的当前内存消耗。
    - 此外，将跟踪每个查询的峰值内存消耗并将其写入日志。
    
对于聚合函数 `min`，`max`，`any`，`anyLast`，`argMin`，`argMax` 的`String`和`Array`参数的状态，没有完全跟踪内存使用情况。

内存消耗也受参数`max_memory_usage_for_user`和`max_memory_usage_for_all_queries`限制。

## max_memory_usage_for_user

用于在单个服务器上运行用户查询的最大RAM数量。

默认值在[Settings.h](https://github.com/ClickHouse/ClickHouse/blob/master/dbms/src/Core/Settings.h#L288)中定义。默认情况下，内存不受限制（`max_memory_usage_for_user = 0`）。

## max_memory_usage_for_all_queries

用于在单个服务器上运行所有查询的最大 RAM 量。

默认值在[Settings.h](https://github.com/ClickHouse/ClickHouse/blob/master/dbms/src/Core/Settings.h#L289)中定义。默认情况下，内存不受限制（`max_memory_usage_for_all_queries = 0`）。

## max_rows_to_read

- 运行查询时可以从表中读取的最大行数(限制可以被打破)
  - 可以在每个块（而不是每一行）上检查以下限制。
  - 在多个线程中运行查询时，限制分别适用于每个线程。

## max_bytes_to_read

运行查询时可以从表中读取的最大字节数（未压缩的数据）。

## read_overflow_mode

- 当读取的数据量超过以下限制之一时该怎么办：
  - `throw`
  - `break`
  - 默认情况下 `throw`。

## max_rows_to_group_by

聚合收到 `unique key` 的最大数目。通过此设置，可以限制聚合时的内存消耗。

## group_by_overflow_mode

- 当聚合的 `unique key` 的数量超过限制时该怎么办：
  - `throw`        
  - `break`        
  - 默认情况下 `throw`。 
  - 使用`any`值可让您近似计算 `GROUP BY`。这种近似的质量取决于数据的统计性质。

## max_bytes_before_external_group_by

启用或禁用`GROUP BY`外部存储器中从句的执行。请参阅`外部存储器中的 GROUP BY`。

可能的值：

- 单个 `GROUP BY` 操作可以使用的最大RAM容量(以字节为单位)。
- `0`: `GROUP BY`禁用外部存储器。

默认值：0

## max_rows_to_sort

排序前的最大行数。这使您可以控制排序时的内存消耗。

## max_bytes_to_sort

排序前的最大字节数。

## sort_overflow_mode

- 如果排序前收到的行数超过限制之一：
  - `throw`        
  - `break`        
  - 默认情况下 `throw`。 

## max_result_rows

限制结果中的行数。运行部分分布式查询时，在远程服务器上检查子查询。

## max_result_bytes

限制结果中的字节数。与之前的设置相同。

## result_overflow_mode

如果结果量超过以下限制之一：
  - `throw`        
  - `break`        
  - 默认情况下 `throw`。
  - 使用`break`类似于使用 `LIMIT`。

## max_execution_time

最大查询执行时间，以秒为单位。目前，不检查排序阶段之一，也不考虑合并和最终聚合函数花费的时间。

## timeout_overflow_mode

- 如果查询运行时间超过`max_execution_time`：
  - `throw`        
  - `break`        
  - 默认情况下 `throw`。

## min_execution_speed

最小执行速度（每秒行数）。当 `timeout_before_checking_execution_speed` 到期时，检查每个数据块。如果执行速度较低，则会引发异常。

## min_execution_speed_bytes

每秒最小执行字节数。当`timeout_before_checking_execution_speed` 到期时，检查每个数据块。如果执行速度较低，则会引发异常。

## max_execution_speed

每秒最大执行行数。当 `timeout_before_checking_execution_speed` 到期时，检查每个数据块。如果执行速度很高，执行速度将降低。

## max_execution_speed_bytes

每秒最大执行字节数。当 `timeout_before_checking_execution_speed` 到期时，检查每个数据块。如果执行速度很高，执行速度将降低。

## timeout_before_checking_execution_speed

在以秒为单位的指定时间到期后，检查执行速度是否不太慢（不小于`min_execution_speed`）。

## max_columns_to_read

单个查询中可以从表中读取的最大列数。如果查询需要读取更多的列，则会引发异常。

## max_temporary_columns

运行查询时必须同时在 `RAM` 中保留的临时列的最大数量，包括常量列。如果临时列多于此，则会引发异常。

## max_temporary_non_const_columns

与 `max_temporary_columns` 相同，但不计算常量列。请注意，运行查询时经常会形成常数列，但它们需要大约零计算资源。

## max_subquery_depth

子查询的最大嵌套深度。如果子查询更深，则会引发异常。默认情况下为 `100`。

## max_pipeline_depth

最大 `pipeline deph`。对应于每个数据块在查询处理期间经历的转换次数。计入单个服务器的限制内。如果 `pipeline deph` 更大，则会引发异常。默认情况下为 `1000`。

## max_ast_depth

- 查询语法树的最大嵌套深度。
  - 如果超出，则抛出异常。
    - 此时，它不会在解析期间进行检查，而是在解析查询之后进行检查。在解析过程中可以创建任意嵌套深度的语法树，但是查询将失败。
  - 默认查询语法数嵌套深度 `1000`。

## max_ast_elements
查询语法树中元素的最大数目。如果超出，则抛出异常。与前面的设置相同，只有在解析查询之后才会检查它。默认`50000`。

## max_rows_in_set

- 从子查询创建的 `IN` 从句中的数据集的最大行数。

## max_bytes_in_set

- 由子查询创建的 `IN` 从句中的集合使用的最大字节数(未压缩数据)。

## set_overflow_mode

- 当数据量超过其中一个限制时该怎么做:
  - `throw`        
  - `break`        
  - 默认情况下 `throw`。

## max_rows_in_distinct

- 使用 `DISTINCT` 时不同行的最大数量。

## max_bytes_in_distinct

- 使用 `DISTINCT` 时哈希表使用的最大字节数。

## distinct_overflow_mode

- 当数据量超过其中一个限制时该怎么做:
  - `throw`        
  - `break`        
  - 默认情况下 `throw`。

## max_rows_to_transfer

使用`GLOBAL IN` 时可以传递到远程服务器或保存到临时表中的最大行数。

## max_bytes_to_transfer

- 使用 `GLOBAL IN` 时可以传递到远程服务器或保存在临时表中的最大字节数(未压缩数据)。

## transfer_overflow_mode

- 当超过其中一个限制时该怎么做:
  - `throw`        
  - `break`        
  - 默认情况下 `throw`。

## max_rows_in_join

- 限制连接表时使用的 `hash table` 中的行数。
  - 此设置适用于 `SELECT ... JOIN` 操作和 `join table engine`。
  - 如果一个查询包含多个连接, 将为每个中间结果检查该设置。
  - 要求：
    - 正整数
  - `0`: 代表行数不受限制
  - 默认值为：`0`

## max_bytes_in_join

- 与 `max_rows_in_join`雷同， 仅有2处有区别：
  - 它是限制大小，非行数
  - `0`: 代表内存不受限制 

## join_overflow_mode

- 当超过其中一个限制时该怎么做:
  - `throw`        
  - `break`        
  - 默认情况下 `throw`。
  
## max_partitions_per_insert_block

- 限制单个插入块中分区的最大数量。
  - 正整数。
  - `0` -无限数量的分区。
  - 默认值: `100`

细节

在插入数据时，ClickHouse计算插入块中的分区数。如果分区数大于max_partitions_per_insert_block，则ClickHouse抛出一个异常，其文本如下:

分区太多时，会导致严重的负面性能影响，包括服务器启动缓慢、`SELECT` 和 `INSERT` 缓慢。
  - 建议一个表的分区总数小于 `1000..10000`。
  - 请注意，分区并不是为了加速 `SELECT` (按键排序足以使范围查询更快)。
  - 分区用于数据操作(删除分区等)。