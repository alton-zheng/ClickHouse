## `Table engines`

`table engines`（即表的类型）决定了：

- 数据的存储方式和位置，写到哪里以及从哪里读取数据
- 支持哪些`Query`以及如何支持。
- 并发数据访问。
- 索引的使用（如果存在）。
- 是否可以执行多线程请求。
- `Data replication`参数。

## Engine Families

### MergeTree

适用于高负载任务的最通用和功能最强大的`table engines`。这些引擎的共同特点是可以快速插入数据并进行后续的后台数据处理。 MergeTree 系列引擎支持`data replication`的引擎版本），`partition`和一些其他引擎不支持的其他功能。

该类型的引擎： 
- `MergeTree`
- `ReplacingMergeTree`
- `SummingMergeTree`
- `AggregatingMergeTree`
- `CollapsingMergeTree`
- `VersionedCollapsingMergeTree`
- `GraphiteMergeTree`

### `Log`

- 具有最小功能的`Lightweight engines`。
- 适合快速写入小表（最多约 `100万` 行）并在以后整体读取它们时，该类型的引擎是最有效的。
                                                                         
该类型的引擎：

- `TinyLog`
- `StripeLog`
- `Log`

### `Intergation engines`

用于与其他的数据存储与处理系统集成的引擎。 该类型的引擎：

- `Kafka`
- `MySQL`
- `ODBC`
- `JDBC`
- `HDFS`

### `Special engines`

该类型的引擎：

- `Distributed`
- `MaterializedView`
- `Dictionary`
- `Merge`
- `File`
- `Null`
- `Set`
- `Join`
- `URL`
- `View`
- `Memory`
- `Buffer`

## Virtual columns
- `Virtual column`: 
  - `table engines` 组成的一部分
    - 源代码定义
  - `read only`
  - 不在  `CREATE TABLE` 指定
  - 不会在下列场景结果中：
    - `SHOW CREATE TABLE`
    - `DESCRIBE TABLE`
  - 必须指定`virtual column`名称才能查询其中的数据
  - 若建表时指定的列名和虚拟列名冲突，虚拟列将不再提供查询功能。
  - 虚拟列一般以 `_` 开头