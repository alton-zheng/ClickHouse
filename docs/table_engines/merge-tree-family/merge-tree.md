## MergeTree

- `MergeTree` 系列
  - `ClickHouse` 最强大的 `table engines`。
  - 巨量数据批量写入，按规则合并。
  - 表数据按`primary key` 排序(小稀疏索引)， 利于快速检索数据
  - 支持分区，加快检索速度。
  - 支持数据副本(`ReplicatedMergeTree`)
  - 可以给表数据指定采样方法

---

### Creating a Table

```
CREATE TABLE [IF NOT EXISTS] [db.]table_name [ON CLUSTER cluster]
(
    name1 [type1] [DEFAULT|MATERIALIZED|ALIAS expr1] [TTL expr1],
    name2 [type2] [DEFAULT|MATERIALIZED|ALIAS expr2] [TTL expr2],
    ...
    INDEX index_name1 expr1 TYPE type1(...) GRANULARITY value1,
    INDEX index_name2 expr2 TYPE type2(...) GRANULARITY value2
) ENGINE = MergeTree()
[PARTITION BY expr]
[ORDER BY expr]
[PRIMARY KEY expr]
[SAMPLE BY expr]
[TTL expr]
[SETTINGS name=value, ...]
```

**Query Clauses**

- `ENGINE`: 引擎名和参数。 `ENGINE = MergeTree()`. `MergeTree`  引擎没有参数。

- `PARTITION BY`: 分区键
  - `Month`:  `toYYYYMM(date_column)`
    - `data_column`： Date 类型的列

- `ORDER BY` : 表的`sort key`。
  - 列元组或任意的表达式。
    - `ORDER BY (CounterID, EventDate)`

- `PRIMARY KEY`: `primary key`
  - 默认情况下 `primary key` 跟`sort key`（由  `ORDER BY`  子句指定）相同。 
  - 因此，大部分情况下不需要再专门指定一个  `PRIMARY KEY`  子句。

- `SAMPLE BY`
  - 抽样， `primary key` 中必须包含这个表达式。
    - 例如： `SAMPLE BY intHash32(UserID) ORDER BY (CounterID, EventDate, intHash32(UserID))` 。

- `SETTINGS`: 影响  `MergeTree`  性能的额外参数：
  - `index_granularity`:索引粒度。
    - 即索引中相邻『标记』间的数据行数。
    - 默认值，8192 。
    - 该列表中所有可用的参数可以从这里查看  [MergeTreeSettings.h](https://github.com/ClickHouse/ClickHouse/blob/master/dbms/src/Storages/MergeTree/MergeTreeSettings.h) 。
  - `index_granularity_bytes`: 数据粒度以 `byte` 为单位的最大大小。
    - 默认值：`10Mb`。
    - 要仅按行数限制颗粒大小，请设置0（不建议）。请参阅`Data Storage`。
  - `enable_mixed_granularity_parts`: 启用或禁用使用该 `index_granularity_bytes` 设置控制颗粒尺寸。
    - 在版本19.11之前，只有 `index_granularity` 粒度限制的设置。
    - `index_granularity_bytes` 从具有大行（数十和数百MB）的表中选择数据时，此设置可提高 `ClickHouse` 性能。因此，如果您的表具有较大的行，则可以打开表的设置以提高 `SELECT` 查询效率。
  - `use_minimalistic_part_header_in_zookeeper`: 数据`parts`头在 `ZooKeeper` 中的存储方式。
    - `use_minimalistic_part_header_in_zookeeper=1` ，·ZooKeeper· 会存储更少的数据。
    - 更多信息参考『服务配置参数』,请看运维相关知识 。
  - `min_merge_bytes_to_use_direct_io` ： 使用 `direct I/O` 来操作`disk`的合并操作时要求的最小数据量。
    - 合并数据`parts`时，`ClickHouse` 会计算要被合并的所有数据的总存储空间。
    - 如果大小超过了  `min_merge_bytes_to_use_direct_io`  设置的字节数，则 `ClickHouse` 将使用直接 I/O 接口（`O_DIRECT`  选项）对`disk`读写。
    - 如果设置  `min_merge_bytes_to_use_direct_io = 0` ，则会禁用 `direct I/O`。
    - 默认值：`10 * 1024 * 1024 * 1024`  bytes(`10G`)。
  - `merge_with_ttl_timeout`: 重复与TTL合并之前的最小延迟（以秒为单位）。
    - 默认值：86400（1天）。
  - `write_final_mark`: 启用或禁用在数据部分的末尾写入最终索引标记。
    - 默认值：1  (不要关闭此标记)

**示例配置**

```
ENGINE MergeTree() PARTITION BY toYYYYMM(EventDate) ORDER BY (CounterID, EventDate, intHash32(UserID)) SAMPLE BY intHash32(UserID) SETTINGS index_granularity=8192
```

- 同时我们设置了一个按用户 ID 哈希的抽样表达式。
- 这让你可以有该表中每个  `CounterID`  和  `EventDate`  下面的数据的伪随机分布。
- 如果你在查询时指定了  `SAMPLE` 子句。
- `ClickHouse` 会返回对于用户子集的一个均匀的伪随机数据采样。
- `index_granularity`  可省略，默认值为 `8192` 。

---

### Data Storage

- 表由按 `primary key` 排序的数据  *`parts`*  组成
- 当数据被插入到表中时，会分成数据`parts`并按 `primary key` 的字典序排序
- 合并相同 `partition` 数据到 `parts`
  - 不会合并来自不同 `partition` 的数据`parts`
  - 相同`PK`的数据可能在不同的`parts`
- ClickHouse 会为每个数据 `part` 创建一个索引文件(.idx)，索引文件包含每个索引行（『标记』）的 `primary key` 值。
  - 索引行号定义为  `n * index_granularity` 。
  - 最大的  `n`  等于总行数除以  `index_granularity`  的值的整数部分。
  - 对于每列，跟 `primary key` 相同的索引行处也会写入『标记』。这些『标记』让你可以直接找到数据所在的列。
- 颗粒的大小受表引擎的 `index_granularity` 和 `index_granularity_bytes` 设置限制。
  - 颗粒中的行数在此 `[1, index_granularity]` 范围内，具体取决于行的大小。
  - `index_granularity_bytes` 如果单行的大小大于设置的值，则颗粒的大小可能会超过。在这种情况下，颗粒的大小等于行的大小。

---

###  Primary keys and indexes in queries

我们以  `(CounterID, Date)`  以 `primary key` 。排序好的索引的图示会是下面这样：

```
Whole data:     [-------------------------------------------------------------------------]
CounterID:      [aaaaaaaaaaaaaaaaaabbbbcdeeeeeeeeeeeeefgggggggghhhhhhhhhiiiiiiiiikllllllll]
Date:           [1111111222222233331233211111222222333211111112122222223111112223311122333]
Marks:           |      |      |      |      |      |      |      |      |      |      |
                a,1    a,2    a,3    b,3    e,2    e,3    g,1    h,2    i,1    i,3    l,3
Marks numbers:   0      1      2      3      4      5      6      7      8      9      10
```

如果指定查询如下：

- `CounterID in ('a', 'h')`，服务器会读取标记号在  `[0, 3)`  和  `[6, 8)`  区间中的数据。
- `CounterID in ('a', 'h') AND Date = 3`，服务器会读取标记号在  `[1, 3)`  和  `[7, 8)`  区间中的数据。
- `Date = 3`，服务器会读取标记号在  `[1, 10]`  区间中的数据。

使用 `index` 比全表`Scan` 更高效。


- 稀疏索引
  - 引起额外的数据读取。
    - 当读取 `primary key` 单个 `range` 范围的数据时，每个 `data block` 中最多会多读  `index_granularity * 2`  行额外的数据。
    - 大部分情况下，当  `index_granularity = 8192`  时，`ClickHouse` 的性能并不会降级。
    - 能操作有巨量行的表。
      - 因为这些索引是常驻内存（RAM）的。

- `ClickHouse` 不要求 `primary key` 惟一。所以，你可以插入多条具有相同 `primary key` 的行。

####  Selecting the Primary Key

 `primary key` 中列的数量并没有明确的限制。依据数据结构，你应该让 `primary key` 包含多些或少些列。这样可以：

- 改善索引的性能。

  如果当前 `primary key` 是  `(a, b)` ，然后加入另一个  `c`  列，满足下面条件时，则可以改善性能：
  - 有带有  `c`  列条件的查询。 
  - 表经常有数倍于 `index_granularity` 设置值的  `(a, b)`  都是相同的值。

- 改善数据压缩。

  - `ClickHouse` 以 `primary key` 排序`parts`数据
    - 数据的一致性越高，压缩越好。

- `CollapsingMergeTree` 和  `SummingMergeTree` 引擎里，数据合并时，会有额外的处理逻辑。

  在这种情况下，指定一个跟 `primary key` 不同的  *sort by*  也是有意义的。

长的 `primary key` 会对插入性能和内存消耗有负面影响，但 `primary key` 中额外的列并不影响  `SELECT`  查询的性能。

#### Choosing a Primary Key that Differs from the Sorting Key

与`sort key`不同的`primary key`。 
  - `primary key` 表达式元组必须是`sort key`表达式元组的一个前缀。
  - 当使用下列引擎时，这样的设置非常有价值：
     - `SummingMergeTree`
     -`AggregatingMergeTree`
     - `Sort key` 经常频繁更新
     - `primary key` 中仅预留少量列保证高效范围扫描

- `sort key`的修改是轻量级
  - 因为新列同时被加入到表和`sort key`后时，已存在的数据`parts`并不需要修改。
  - 由于旧的`sort key`是新`sort key`的前缀，并且刚刚添加的列中没有数据，因此在表修改时的数据对于新旧的`sort key`来说都是有序的。

#### Use of indexes and Partitions in Queries

对于  `SELECT`  查询，是否可以使用索引:
  
- 如果  `WHERE/PREWHERE`  子句有下面这些表达式（作为谓词链接一子项或整个）则可以使用索引：
    - 基于 `primary key` 或分区键的列或表达式:
      - 部分的等式或比较运算表达式；
      - 固定前缀的  `IN`  或  `LIKE`  表达式；

    - 基于 `primary key` 或分区键的列:
      - 函数；

    - 基于 `primary key` 或分区键的表达式:
      - 逻辑表达式。

当引擎配置如下时：

```
ENGINE MergeTree() PARTITION BY toYYYYMM(EventDate) ORDER BY (CounterID, EventDate) SETTINGS index_granularity=8192
```

这种情况下，这些查询：

```sql
SELECT count() FROM table WHERE EventDate = toDate(now()) AND CounterID = 34
SELECT count() FROM table WHERE EventDate = toDate(now()) AND (CounterID = 34 OR CounterID = 42)
SELECT count() FROM table WHERE ((EventDate >= toDate('2014-01-01') AND EventDate <= toDate('2014-01-31')) OR EventDate = toDate('2014-05-01')) AND CounterID IN (101500, 731962, 160656) AND (CounterID = 101500 OR EventDate != toDate('2014-05-01'))
```

ClickHouse 会依据 `primary key` 索引剪掉不符合的数据，依据按月分区的分区键剪掉那些不包含符合数据的分区。

上文的查询显示，即使索引用于复杂表达式。因为读表操作是组织好的，所以，使用索引不会比完整扫描慢。

下面这个例子中，不会使用索引。

```sql
SELECT count() FROM table WHERE CounterID = 34 OR URL LIKE '%upyachka%'
```

要检查 ClickHouse 执行一个查询时能否使用索引，可设置  `force_index_by_date` 和  `force_primary_key`, 运维部分。

按月分区的分区键是只能读取包含适当范围日期的数据块。这种情况下，数据块会包含很多天（最多整月）的数据。
- 在块中，数据按 `primary key` 排序， `primary key` 第一列可能不包含日期。
- 因此，仅使用日期而没有带 `primary key` 前缀条件的查询将会导致读取超过这个日期范围。

#### Use of Index for Partially-Monotonic Primary Keys
例如，考虑一个月中的几天。它们形成一个月的[`monotonic sequence`](https://en.wikipedia.org/wiki/Monotonic_function)，但在更长的时间内不单调。这是部分单调的序列。如果用户使用部分单调的主键创建表，则ClickHouse将照常创建稀疏索引。当用户从这种表中选择数据时，`ClickHouse` 将分析查询条件。如果用户希望在索引的两个标记之间获取数据并且这两个标记均在一个月之内，则ClickHouse可以在这种特殊情况下使用索引，因为它可以计算查询参数与索引标记之间的距离。

如果查询参数范围内主键的值不表示`monotonic sequence`，则 `ClickHouse` 无法使用索引。在这种情况下，`ClickHouse` 使用完整扫描方法。

`ClickHouse` 不仅在每月的某几天序列中使用此逻辑，而且还在表示部分 `monotonic sequence` 的任何主键中使用此逻辑。

#### Data Skipping Indexes (Experimental)

需要设置  `allow_experimental_data_skipping_indices`  为 1 才能使用此索引。（执行  `SET allow_experimental_data_skipping_indices = 1`）。

此索引在  `CREATE`  语句的列部分里定义。

```
INDEX index_name expr TYPE type(...) GRANULARITY granularity_value
```

`*MergeTree` 系列的表都能指定跳数索引。

这些索引是由数据块按粒度分割后的每部分在指定表达式上汇总信息  `granularity_value`  组成（粒度大小用`table engines`里  `index_granularity`  的指定）。 

这些汇总信息有助于用 `where` 语句跳过大片不满足的数据，从而减少 `SELECT` 查询从`disk`读取的数据量

**Example**

```sql
CREATE TABLE table_name
(
    u64 UInt64,
    i32 Int32,
    s String,
    ...
    INDEX a (u64 * i32, s) TYPE minmax GRANULARITY 3,
    INDEX b (u64 * length(s)) TYPE set(1000) GRANULARITY 4
) ENGINE = MergeTree()
...
```

上例中的索引能让 ClickHouse 执行下面这些查询时减少读取数据量。

```sql
SELECT count() FROM table WHERE s < 'z'
SELECT count() FROM table WHERE u64 * i32 == 10 AND u64 * length(s) >= 1234
```

**Available Types of indices**

- `minmax`  存储指定表达式的极值（如果表达式是  `tuple` ，则存储  `tuple`  中每个元素的极值），这些信息用于跳过数据块，类似 `primary key` 。

- `set(max_rows)`  存储指定表达式的惟一值（不超过  `max_rows`  个，`max_rows=0`  则表示『无限制』）。这些信息可用于检查  `WHERE`  表达式是否满足某个数据块。

- `ngrambf_v1(n, size_of_bloom_filter_in_bytes, number_of_hash_functions, random_seed)`  
  - 存储包含数据块中所有 n 元短语的  [`Bloom filter`](https://en.wikipedia.org/wiki/Bloom_filter) 。
  - 只可用在字符串上。 可用于优化  `equals` ， `like`  和  `in`  表达式的性能。 
    
    - `n` : 短语长度。 
    - `size_of_bloom_filter_in_bytes`: `Bloom filter`大小，单位字节。（因为压缩得好，可以指定比较大的值，如 `256` 或 `512`）。
    -  `number_of_hash_functions` : `Bloom filter` 中使用的 hash 函数的个数。
    -  `random_seed` : hash 函数的随机种子。

- `tokenbf_v1(size_of_bloom_filter_in_bytes, number_of_hash_functions, random_seed)`  
  - 跟  `ngrambf_v1`  类似，它只存储被非字母数据字符分割的`parts`。

- `bloom_filter([false_positive])`
  - 为指定的列存储 `Bloom filter`。
  - 可选 `false_positive` 参数是从过滤器接收到错误肯定响应的可能性。可能的值：`（0，1）`。默认值：`0.025`。
  
  - 支持的数据类型：
    - `Int*`
    - `UInt*`
    - `Float*`
    - `Enum`
    - `Date`
    - `DateTime`
    - `String`
    - `FixedString`
    - `Array`
    - `LowCardinality`
    - `Nullable`
  
  以下函数可以使用它：`equals`，`notEquals`，`in`，`notIn`和`has`。
  
```sql
INDEX sample_index (u64 * length(s)) TYPE minmax GRANULARITY 4
INDEX sample_index2 (u64 * length(str), i32 + f64 * 100, date, str) TYPE set(100) GRANULARITY 4
INDEX sample_index3 (lower(str), str) TYPE ngrambf_v1(3, 256, 2, 0) GRANULARITY 4
```

**Functions Support**

`WHERE`子句中的条件包含使用列操作的函数的调用。如果列是索引的一部分，则 `ClickHouse` 在执行功能时会尝试使用此索引。`ClickHouse` 支持使用索引的功能的不同子集。

该set索引可以与所有函数一起使用。下表显示了其他索引的功能子集。

![support_function_for_data_skip_index.png](../images/support_function_for_data_skip_index.png)

常量参数小于`ngram`大小的函数不能使用`ngrambf_v1`查询优化。

`Bloom Filter`可以有 `false positive` 匹配，所以 当函数预期为 `false` 时，`ngrambf_v1`，`tokenbf_v1` 和 `bloom_filter` 索引不能用于优化查询，例如：

- `Can be optimized`：
 - s LIKE '%test%'
 - NOT s NOT LIKE '%test%'
 - s = 1
 - NOT s != 1
 - startsWith(s, 'test')

- `Can't be optimized`：
  - NOT s LIKE '%test%'
  - s NOT LIKE '%test%'
  - NOT s = 1
  - s != 1
  - NOT startsWith(s, 'test')

---

### Concurrent Data Access

应对表的并发访问，我们使用多版本机制。换言之，当同时读和更新表时，数据从当前查询到的一组`parts`中读取。没有冗长的的锁。插入不会阻碍读取。

对表的读操作是自动并行的。

---

### TTL for Columns and Tables

确定`value`的`lifetime`。

可以为整个表和每个单独的列设置 `TTL` 从句。如果同时设置了两种 `TTL`，则 `ClickHouse` 将使用更早时间的`TTL`配置。

该表必须在 `Date` 或 `DateTime` 数据类型的列上定义数据的生存期。如：

```
TTL time_column
TTL time_column + interval
```

需要使用 `time interval` 操作，定义 `interval`:

```
TTL date_time + INTERVAL 1 MONTH
TTL date_time + INTERVAL 15 HOUR
```

*Column TTL**

当列中的值过期时，`ClickHouse` 会将其替换为列数据类型的默认值。如果数据部分中的所有列值均已过期，则 ClickHouse 将从文件系统中的数据部分删除此列。

该`TTL`子句不能用于`key column`。

例子：

用 TTL 创建表

```sql
CREATE TABLE example_table 
(
    d DateTime,
    a Int TTL d + INTERVAL 1 MONTH,
    b Int TTL d + INTERVAL 1 MONTH,
    c String
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(d)
ORDER BY d;
```

将 TTL 添加到现有表的列中

```sql
ALTER TABLE example_table
    MODIFY COLUMN
    c String TTL d + INTERVAL 1 DAY;
```

更改列的 TTL

```sql
ALTER TABLE example_table
    MODIFY COLUMN
    c String TTL d + INTERVAL 1 MONTH;
```

**Table TTL**

当 `table` 中的数据过期时，`ClickHouse` 会删除所有对应的行。

例子：

用 TTL 创建表

```sql
CREATE TABLE example_table 
(
    d DateTime,
    a Int
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(d)
ORDER BY d
TTL d + INTERVAL 1 MONTH;
```

改变表的 TTL

```sql
CREATE TABLE example_table 
(
    d DateTime,
    a Int
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(d)
ORDER BY d
TTL d + INTERVAL 1 MONTH;
```

**Removing Data**

- 当 `ClickHouse` 合并数据部分时，将删除 TTL 过期的数据。

- 数据已过期时，它将执行 `off-shedule` 合并。
  - 要控制此类合并的频率，可以设置 `merge_with_ttl_timeout`(运维部分)。
  - 如果该值太低，它将执行许多 `off-schedule` 合并，这可能会消耗大量资源。

如果`SELECT`在合并之间执行查询，则可能会获得过期的数据。为了避免这种情况，请在之前使用[OPTIMIZE](https://clickhouse.yandex/docs/en/query_language/misc/#misc_operations-optimize)查询`SELECT`。

---

## Using multiple block devices for data storage

---

### **General**

`MergeTree` 系列的表能够将其数据存储在多个块设备上.
- 例如，当某个表的数据隐式拆分为 `hot` 和 `cold` 时，这可能会很有用。
  - 定期请求最新数据，但只需要少量空间。
  - 相反，很少要求提供详尽的历史数据。如果有多个`disk`可用，则 `hot` 数据可能位于快速`disk`（NVMe SSD 甚至是内存）中，而 `cold` 数据可能位于相对较慢的`disk`（HDD）上。

- `part` 是 `MergeTree` 表的最小可移动单位。
  - 在后台的`disk`之间（根据用户设置)
  - 通过 `ALTER` 查询来移动部件。 

---

### **Terms**

-`Disk`: 挂载到文件系统的块设备。
- `Default disk`-包含在 `<path>` 标签中指定的路径的`disk`config.xml`。
- `Volume`: 一组相等的有序`disk`（类似于[JBOD](https://en.wikipedia.org/wiki/Non-RAID_drive_architectures)）。
- `Storage policy`-许多 `Volume` 以及在它们之间移动数据的规则。

可以在系统表, [system.storage_policies](https://clickhouse.yandex/docs/en/operations/system_tables/#system_tables-storage_policies)和[system.disks](https://clickhouse.yandex/docs/en/operations/system_tables/#system_tables-disks)找到提供给所描述实体的名称。可以将 `storage policy` 名称用作 `MergeTree` 系列表的参数。

---

### Configuration

`Disk`, `Volume` 和 `Storage policy` 应在 `main` 文件中`config.xml`或`config.d`目录中不同文件的标记`<storage_configuration>`内声明。配置文件中的此部分具有以下结构：

```xml
<disks>
    <fast_disk> <!-- disk name -->
        <path>/mnt/fast_ssd/clickhouse</path>
    </fast_disk>
    <disk1>
        <path>/mnt/hdd1/clickhouse</path>
        <keep_free_space_bytes>10485760</keep_free_space_bytes>_
    </disk1>
    <disk2>
        <path>/mnt/hdd2/clickhouse</path>
        <keep_free_space_bytes>10485760</keep_free_space_bytes>_
    </disk2>

    ...
</disks>
```

- `disk`名称作为标记名称给出。e.g. `fast_disk` 为 `disk`的名称
- `path`: 服务器将用来存储数据（`data`和`shadow`文件夹）的路径应以 '/' 结尾。
- `keep_free_space_bytes` —要保留的可用`disk`空间量。

`disk`定义的顺序并不重要。

 `storage policy`配置：

```xml
<policies>
    <hdd_in_order> <!-- policy name -->
        <volumes>
            <single> <!-- volume name -->
                <disk>disk1</disk>
                <disk>disk2</disk>
            </single>
        </volumes>
    </hdd_in_order>

    <moving_from_ssd_to_hdd>
        <volumes>
            <hot>
                <disk>fast_ssd</disk>
                <max_data_part_size_bytes>1073741824</max_data_part_size_bytes>
            </hot>
            <cold>
                <disk>disk1</disk>
            </cold>            
        </volumes>
        <move_factor>0.2</move_factor>
    </moving_from_ssd_to_hdd>
</policies>
```

- `volume` 和 `storage policy` 名称以标签名称形式给出。
- `disk` : `volume` 中的`disk`。
- `max_data_part_size_bytes` —可以存储在任何 `volume` 的 `disk` 上的部件的最大大小。
- `move_factor` —当可用空间量小于该因子时，数据将自动开始在下一个`volume`（如果有）上移动（默认值为 `0.1`）。

在给定的示例中，该`hdd_in_order`策略实现了[循环](https://en.wikipedia.org/wiki/Round-robin_scheduling)方法。由于该策略仅定义一个`volume`（`single`），因此数据以循环顺序存储在其所有`disk`上。如果有多个类似的`disk`安装到系统，则此策略将非常有用。如果有不同的`disk`，则`moving_from_ssd_to_hdd`可以使用该策略。该`volume`hot`由一个 SSD `disk`（`fast_ssd`）组成，该`volume`上可以存储的部件的最大大小为 1GB。所有大小大于 1GB 的部件将直接存储在`cold`包含 HDD `disk`的`volume`上`disk1`。同样，一旦`disk`的`fast_ssd`容量超过 80％，数据将`disk1`通过后台进程传输。

 `storage policy`中`volume`枚举的顺序很重要。一旦一个`volume`被过度填充，数据将移至下一个。`disk`枚举的顺序也很重要，因为数据是依次存储在`disk`上的。

创建表时，可以将配置的 `storage policy` 之一应用于表：

```sql
CREATE TABLE table_with_non_default_policy (
    EventDate Date,
    OrderID UInt64,
    BannerID UInt64,
    SearchPhrase String
) ENGINE = MergeTree
ORDER BY (OrderID, BannerID)
PARTITION BY toYYYYMM(EventDate)
SETTINGS storage_policy = 'moving_from_ssd_to_hdd'
```

该`default` `storage policy`意味着使用只有一个`volume`，其中仅由一个在给定的`disk`<path>`。创建表后，将无法更改其 `storage policy`。

---

### Details

对于 `MergeTree` 表，数据以不同的方式进入`disk`：

- 作为插入（`INSERT` 查询）的结果。
- 在后台合并和`mutation` (见 `Sql Reference/ALTER/Mutations` 部分) 过程中。[官方地址](https://clickhouse.yandex/docs/en/query_language/alter/#alter-mutations)。
- 从另一个副本下载时。
- 由于 `partition freezing` 而导致(见 `sql reference/ALTER/FREEZE PARTITION` ) [][ALTER TABLE ... FREEZE PARTITION](https://clickhouse.yandex/docs/en/query_language/alter/#alter_freeze-partition)。

在所有这些情况下，除了`mutation`和`partition frozen`外，根据给定的 `storage policy`，一部分存储在`volume`和`disk`上：

1.  选择具有足够`disk`空间来存储部件（`unreserved_space > current_part_size`）并允许存储给定大小（`max_data_part_size_bytes > current_part_size`）的部件的第一个`volume`（按定义顺序）。
2.  在该`volume`中，选择该`disk`之后的`disk`，该`disk`用于存储先前的数据块，并且其可用空间大于部件大小（`unreserved_space - keep_free_space_bytes > current_part_size`）。

在幕后，`mutations` 和 `partition frozen` 利用[`hard links`](https://en.wikipedia.org/wiki/Hard_link)。不支持不同`disk`之间的`hard links`，因此在这种情况下，生成的 `parts` 与初始`disk`存储在同一`disk`上。

在后台，`part` 被`volume`之间的自由空间（的量的基础上移动`move_factor`参数）根据`volume`在配置文件中声明的顺序。数据永远不会从最后一个传输到第一个。可以使用系统表[system.part_log](https://clickhouse.yandex/docs/en/operations/system_tables/#system_tables-part-log)（字段`type = MOVE_PART`）和[system.parts](https://clickhouse.yandex/docs/en/operations/system_tables/#system_tables-parts)（字段`path`和`disk`）来监视后台移动。同样，可以在服务器日志中找到详细信息。

用户可以使用查询[ALTER TABLE ... MOVE PART | PARTITION ... TO VOLUME | DISK ...](https://clickhouse.yandex/docs/en/query_language/alter/#alter_move-partition)来强制将一`parts`或`partition`从一个`volume`移动到另一个`volume`，所有对后台操作的限制都已考虑在内。该查询将自行启动移动，而无需等待后台操作完成。如果没有足够的可用空间或不满足任何要求的条件，用户将收到错误消息。

移动数据不会干扰数据复制。因此，可以为不同副本上的同一表指定不同的 `storage policy`。

后台合并和`mutation`完成后，仅在一定时间（`old_parts_lifetime`）后才删除旧`parts`。在此期间，它们不会移至其他`volume`或`disk`。因此，在最终删除这些`part`之前，仍要考虑它们以评估占用的`disk`空间。