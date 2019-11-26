# 介绍

## 什么是ClickHouse? 
ClickHouse是一个用于联机分析(OLAP)的列式数据库管理系统(DBMS)。
---
常见的列式数据库有： Vertica、 Paraccel (Actian Matrix，Amazon Redshift)、 Sybase IQ、 Exasol、 Infobright、 InfiniDB、 MonetDB (VectorWise， Actian Vector)、 LucidDB、 SAP HANA、 Google Dremel、 Google PowerDrill、 Druid、 kdb+。

## OLAP 场景的关键特征
- 大多数是读请求
- 数据总是以相当大的批(> 1000 rows)进行写入
- 不修改已添加的数据
- 每次查询都从数据库中读取大量的行，但是同时又仅需要少量的列
- 宽表，即每个表包含着大量的列
- 较少的查询(通常每台服务器每秒数百个查询或更少)
- 对于简单查询，允许延迟大约50毫秒
- 列中的数据相对较小： 数字和短字符串(例如，每个URL 60个字节)
- 处理单个查询时需要高吞吐量（每个服务器每秒高达数十亿行）
- 事务不是必须的
- 对数据一致性要求低
- 每一个查询除了一个大表外都很小
- 查询结果明显小于源数据，换句话说，数据被过滤或聚合后能够被盛放在单台服务器的内存中

## 列式数据库更适合 OLAP 场景的原因：

列式数据库更适合于OLAP场景(对于大多数查询而言，处理速度至少提高了100倍)，下面详细解释了原因(通过图片更有利于直观理解)：
- 行式

![row_oriented](../images/row_oriented.gif)

- 列式

![column_oriented](../images/column_oriented.gif)

从以上两图就可以直观的感受到列式的优势，下面进行简单的解释：

### Input/output
- 针对分析类查询，通常只需要读取表的一小部分列。在列式数据库中你可以只读取你需要的数据。例如，如果只需要读取100列中的5列，这将帮助你最少减少20倍的I/O消耗。
- 由于数据总是打包成批量读取的，所以压缩是非常容易的。同时数据按列分别存储这也更容易压缩。这进一步降低了I/O的体积。
- 由于I/O的降低，这将帮助更多的数据被系统缓存。

例如，查询“统计每个广告平台的记录数量”需要读取“广告平台ID”这一列，它在未压缩的情况下需要1个字节进行存储。如果大部分流量不是来自广告平台，那么这一列至少可以以十倍的压缩率被压缩。当采用快速压缩算法，它的解压速度最少在十亿字节(未压缩数据)每秒。换句话说，这个查询可以在单个服务器上以每秒大约几十亿行的速度进行处理。这实际上是当前实现的速度。 

```bash
$ clickhouse-client
ClickHouse client version 0.0.52053.
Connecting to localhost:9000.
Connected to ClickHouse server version 0.0.52053.

:) SELECT CounterID, count() FROM hits GROUP BY CounterID ORDER BY count() DESC LIMIT 20

SELECT
    CounterID,
    count()
FROM hits
GROUP BY CounterID
ORDER BY count() DESC
LIMIT 20

┌─CounterID─┬──count()─┐
│    114208 │ 56057344 │
│    115080 │ 51619590 │
│      3228 │ 44658301 │
│     38230 │ 42045932 │
│    145263 │ 42042158 │
|    and..    and..    |
└───────────┴──────────┘

20 rows in set. Elapsed: 0.153 sec. Processed 1.00 billion rows, 4.00 GB (6.53 billion rows/s., 26.10 GB/s.)

:)
```

## ClickHouse的独特功能：

- 真正的列式数据管理系统
  - ClickHouse 仅存储数据本身，它不单单是一个数据库，还是一个数据库管理系统。允许在运行时创建表和数据库，加载数据，运行查询，而无需重新配置或重启服务。

- 数据压缩
  - ClickHouse 针对数据进行了压缩。

- 数据磁盘存储
  - ClickHouse被设计用于工作在传统磁盘上的系统，它提供每GB更低的存储成本，但如果有可以使用SSD和内存，它也会合理的利用这些资源。

- 多Core 并行处理
  - ClickHouse会使用服务器上一切可用的资源，从而以最自然的方式并行处理大型查询。
  
- 多服务器分布式处理
  - 在ClickHouse中，数据可以保存在不同的shard上，每一个shard都由一组用于容错的replica组成，查询可以并行地在所有shard上进行处理。这些对用户来说是透明的
  
- 支持SQL
  - 支持的查询包括 `GROUP BY`，`ORDER BY`，`IN`，`JOIN` 以及非相关子查询。 不支持窗口函数和相关子查询。
  
- 向量引擎
  - `ClickHouse` 为了高效的使用CPU，数据不仅仅按列存储，同时还按向量(列的一部分)进行处理，这样可以更加高效地使用CPU。

- 实时写数
  - `ClickHouse` 支持在表中定义主键，数据总是以增量特定值或范围的查找。数据总是以增量有序的方式存储在 `MergeTree` 中。不仅高效而且不会存在加锁的行为。
  
- 索引
  - 按照主键对数据进行排序，这将帮助ClickHouse在几十毫秒以内完成对数据特定值或范围的查找。
  
- 适合在线查询
  - 在线查询意味着在没有对数据做任何预处理的情况下以极低的延迟处理查询并将结果加载到用户的页面中。·
  
- 支持近似查询
  - ClickHouse 提供几种在允许牺牲数据精度的情况下对查询进行加速的方法：
    - 用于近似计算的各类聚合函数，如：distinct values, medians, quantiles
    - 基于数据的部分样本进行近似查询。这时，仅会从磁盘检索少部分比例的数据。
    - 不使用全部的聚合条件，通过随机选择有限个数据聚合条件进行聚合。这在数据聚合条件满足某些分布条件下，在提供相当准确的聚合结果的同时降低了计算资源的使用。
    
- 支持数据复制和数据完整性
  - ClickHouse 使用异步多主复制技术。 当数据被写入任何一个可用副本后，系统会在后台将数据分发给其他副本，以保证系统在不同副本上保持相同的数据。在大多数情况下ClickHouse能在故障后自动恢复，在一些少数的复杂情况下需要手动恢复。
  
## ClickHouse 可以考虑缺点的功能
- 没有完整的事务支持。（可以说没有）
- 缺少高频率，低延迟的修改或删除已存在数据的能力。仅能用于批量删除或修改数据，但这符合 [GDPR](https://gdpr-info.eu/)。
- 稀疏索引使得ClickHouse不适合通过其键检索单行的点查询。


# 指南

## 系统要求
如果从官方仓库安装，需要确保您使用的是x86_64处理器构架的Linux并且支持SSE 4.2指令集

检查是否支持SSE 4.2：
```bash
grep -q sse4_2 /proc/cpuinfo && echo "SSE 4.2 supported" || echo "SSE 4.2 not supported"
```

推荐使用Ubuntu或者Debian。终端必须使用UTF-8编码。


# 客户端

---

ClickHouse提供了两个网络接口（为了安全起见，都可以选择将两者包装在TLS(安全传输协议)中）：

- HTTP： 易于使用
- Native TCP:  开销小

在大多数情况下，建议使用适当的工具或库，而不是直接与这些工具或库进行交互。 Yandex的官方支持如下:
- command-line client
- JDBC driver
- ODBC driver
- C++ client library

还有许多第三方库可供使用ClickHouse: 
- Client libraries
- Integrations
- Visual interfaces


接下来针对上面的内容，进行阐述：

## HTTP 客户端
- 跨平台，跨编程语言使用ClickHouse.
- `HTTP` 接口比 `TCP` 原生接口更为局限， 但有更好的兼容性。 
- ClickHouse 默认端口 `8123` 来监控 `HTTP` 请求, 端口可配。 

```bash
curl 'http://localhost:8123/'
ok.
```

- 可以通过下面几种方式发送`HTTP` 请求：
  - `query` 参数,即 `GET` 请求
  - `POST`
  - 混合形式： 参数 + `POST`

状态码：
  - `200`： 请求成功
  - `500`:  请求失败
  
- 注意点：
  - 使用`GET` 请求时， `readonly` 会被设置。
  - `INSERT` 必须通过 `POST` 请求。
  - `URL`的大小会限制在 `16KB`.
  - `wget` 命令不推荐使用，因它在 `HTTP 1.1` 协议下使用 `keep-alive` 和 `Transfer-Encoding:chunked` 头部设置时，不能很好的功能。
  - `curl` 命令在`url` 中填写`sql`语句时，如果遇到空格需要转义。
  - 默认情况下，返回的数据是 `TabSeparated `格式的， 可配（`Format`)
  - `compress=1` ，服务会返回压缩的数据
  - `decompress=1` ，服务会解压通过 POST 方法发送的数据
  - 默认数据库为 `default` (已注册)
  - 使用缓冲以避免在将`Response`代码和`HTTP Headers`发送到客户端之后发生查询处理错误的情况。在这种情况下，错误消息会写在响应正文的末尾，而在客户端，只能在解析阶段检测到错误。
     
- 用户密码可通过下面两种方式设置
```bash
## 通过 HTTP Basic Authentication。示例：
$ echo 'SELECT 1' | curl 'http://user:password@localhost:8123/' -d @-
## 通过 URL 参数 中的 'user' 和 'password'。示例：
$ echo 'SELECT 1' | curl 'http://localhost:8123/?user=user&password=password' -d @-

## 注意： 如果用户名没有指定，默认的用户是 default。如果密码没有指定，默认会使用空密码。 
```

### `HTTP` 压缩
`ClickHouse`支持`gzip`，`br`以及`deflate` 压缩方法: 

- 压缩请求 `Content-Encoding:compression_method`
- 压缩响应 `Accept-Encoding: compression_method`
- 要启用HTTP压缩，必须使用[`ClickHouse enable_http_compression`](./clickhouse_operations.md)设置
- 您可以在`http_zlib_compression_level`设置中为所有压缩方法配置数据压缩级别。
- 在传输大量数据或创建立即压缩的转储时，可以使用它来减少网络流量。
- 默认情况下，某些`HTTP`客户端可能会从服务器解压缩数据（使用`gzip`和`deflate`），即使正确使用压缩设置，也可能会获得解压缩的数据。

## Command-line Client

- 要从命令行工作，可以使用`clickhouse-client` 命令

- non-interactive(`batch mode`)
  - 通过`FORMAT` 设置数据格式，默认为 `TabSeparated`。 
  - `--query`： 执行一个语句。
  - `--multiquery` 通过`script` 执行除了 `INSERT` 批量语句。
  - 查询结果不需要另外指定分隔符（连续输出）。
  - 启动`clickhouse-client`, 需要几十毫秒时间。
  
- `interactive mode`
  - 通过`FORMAT` 设置数据格式，默认为 `PrettyCompact`。 也可以通过`\G`在查询末尾使用命令行中的`--format`或`--vertical`参数或使用客户端配置文件来指定格式。
  - 未指定`multiline`, 语句末尾不需要 `;` 要输入多行， 末尾 加`\`
  - 指定`multiline`, 语句末尾加 `;` 号并按`Enter`。如果在输入行的末尾省略了`;`，则将要求您输入下一行。
  - 可以指定`\G`代替分号或在分号之后。这表示垂直格式。以这种格式，每个值都打印在单独的行上，这对于宽表很方便。添加了此不常用功能仅为了与 `MySQL CLI` 兼容。
  - 命令行基于`readline`（和`history`或`libedit`，或者`without a library`，具体情况取决于`build`).换句话说，它使用熟悉的键盘快捷键并保留历史记录。历史记录被写入`~/.clickhouse-client-history`。
  - 退出`client`，请按 `Ctrl+D`（或 `Ctrl+C`)，或输入以下命令之一:  [`exit`, `quit`, `logout`, `exit;`, `quit;`, `logout;`, `q`, `Q`, `:q`]
  - 您可以通过按`Ctrl+C`取消长查询。但是，您仍然需要稍等一会儿，服务器才能中止请求。在某些阶段无法取消查询。如果您不等待，然后再次按`Ctrl+C`，则客户端将退出。
  - 处理请求时，客户端显示：
    - 进度，每秒更新不超过 10 次（默认情况下）。对于快速查询，进度可能没有时间显示。
    - 解析后的格式化查询，用于调试很合适。
    - 指定格式的结果。
    - result: 行数，消耗时间以及平均速度。
  - 命令行客户端允许传递`external data`（`external temporary tables`）以进行查询。有关更多信息，请参见`external temporary tables`部分。

## Native Interface(TCP)

`Native`协议用于`Command-line Client`，分布式查询处理期间的服务器间以及其他`C++`程序. 目前官方还没有正式的文档对此进行说明，但是可以从 `ClickHouse` 源代码（从[此处](https://github.com/ClickHouse/ClickHouse/tree/master/dbms/src/Client)开始）和/或通过拦截和分析 TCP 流量进行反向工程来分析。

## Formats for Input and Output Data

ClickHouse 可以`accept`和`return`各种格式的数据。输入支持的格式可用于解析提供给`INSERT`的数据，`SELECT`从文件支持的表（例如 `File`，`URL` 或 `HDFS`）执行或读取外部字典。支持输出的格式可用于安排`SELECT`结果，并在文件支持的表中执行`INSERT`。

支持的格式为：

|  Format	                       | Input	  | Output |
|  ------                          | -----    | ------ |
|  TabSeparated                    |   R      |   R    |
|  TabSeparatedRaw	               |   W      |   R    |
|  TabSeparatedWithNames	       |   R      |   R    |
|  TabSeparatedWithNamesAndTypes   |   R      |   R    |
|  Template     	               |   R      |   R    |
|  TemplateIgnoreSpaces	           |   R      |   W    |
|  CSV                             |   R      |   R    |
|  CSVWithNames	                   |   R      |   R    |
|  CustomSeparated	               |   R      |   R    |
|  Values                      	   |   R      |   R    |
|  Vertical                    	   |   W      |   R    |
|  JSON                        	   |   W      |   R    |
|  JSONCompact                     |   W      |   R    |
|  JSONEachRow                     |   R      |   R    |
|  TSKV	                           |   R      |   R    |
|  Pretty	                       |   W      |   R    |
|  PrettyCompact	               |   W      |   R    |
|  PrettyCompactMonoBlock	       |   W      |   R    |
|  PrettyNoEscapes             	   |   W      |   R    |
|  PrettySpace                     |   W      |   R    |
|  Protobuf                        |   R      |   R    |
|  Parquet	                       |   R      |   R    |
|  RowBinary                       |   R      |   R    |
|  RowBinaryWithNamesAndTypes      |   R      |   R    |
|  Native	                       |   R      |   R    |
|  Null	                           |   W      |   R    |
|  XML                             |   W      |   R    |
|  CapnProto                       |   R      |   W    |

注： 后期再细看

## JDBC Driver

- **[Official driver](https://github.com/ClickHouse/clickhouse-jdbc)**
- Third-party drivers：
  - [ClickHouse-Native-JDBC](https://github.com/housepower/ClickHouse-Native-JDBC)
  - [clickhouse4j](https://github.com/blynkkk/clickhouse4j)
  
## ODBC Driver

- [Official driver](https://github.com/ClickHouse/clickhouse-odbc)

## C++ Client Library

请参阅[clickhouse-cpp](https://github.com/ClickHouse/clickhouse-cpp)

官方没有除C++ 客户端以外的框架支持，比如其它语言的客户端库， 集成库， 可视化库。 需要开发人员自行编写或使用非官方第三方库


## Database Engines

数据库引擎允许您使用表。

默认情况下，ClickHouse 使用其native 数据库引擎，该引擎提供可配置的table engines和SQL。

您还可以使用以下数据库引擎：

- `MySQL`

- `Lazy`

## 架构

- `Table Storage and Query` 

![table query](../images/Clickhouse-Storage:Query.jpg)

- `Server`

![server](../images/ClickHouse-server.jpg)

- `Cluster`

![cluster](../images/ClickHouse-cluster.jpg)


# Compression in ClickHouse

[Compression in ClickHouse](https://www.altinity.com/blog/2017/11/21/compression-in-clickhouse),  原文在*`2017/11/21`* 发表

`ClickHouse` 支持2种压缩方法（算法）， 默认 `L24`, 可以用相关配置指定：
  - `LZ4`
  - `ZSTD`

[基于`Mysql`压缩算法测试报告](https://www.percona.com/blog/2016/04/13/evaluating-database-compression-methods-update/)

结论： 简而言之，LZ4在速度上会更快，但是压缩率较低，ZSTD正好相反。尽管ZSTD比LZ4慢，但是相比传统的压缩方式Zlib，无论是在压缩效率还是速度上，都可以作为Zlib的替代品。

- 基于 `ClickHouse` 的测试：
  - table `lineorder` [**结构和数据**](https://www.altinity.com/blog/2017/6/16/clickhouse-in-a-general-analytical-workload-based-on-star-schema-benchmark)
  - 未压缩数据集 `680G`
  - 压缩数据能力测试结果：
    - `LZ4` `184G/680G`（`27%`)
    - `ZSTD` `135G/680G`（`20%`)
    - 压缩比 `LZ4/ZSTD`: `3.7/5.0`
  - 性能测试：
    - `SQL`:
      - `SELECT toYear(LO_ORDERDATE) AS yod, sum(LO_REVENUE) FROM lineorder GROUP BY yod;` 没有 `where`,`order by` 等操作。 
    - 冷数据：
      - 两者区别不大，时间主要消耗在`I/O`,远大于解压缩的时间
    - 热数据(已缓存)：
      - 热数据请求下，`LZ4`会更快，此时`IO`代价小，数据解压缩成为性能瓶颈。消耗时间上两者比率： `9/16`, 快了差不多2倍
  - 结论：
    - 大范围扫描的查询，瓶颈是`I/O`， 首选 `ZSTD`。
    - 当`I/O`快时，`LZ4`为首选
    - 具体选择什么，看具体场景和需求。 

- `ClickHouse` 压缩算法配置：
```xml
    <compression incl="clickhouse_compression">
            <case>
                    <method>zstd</method>
            </case>
    </compression>
```


## `LZ4` 算法
- `LZ4`
  - lz4 开源后， 安卓，`Apple`,`python`,`clickhouse` 都有支持 [code](https://github.com/lz4/lz4)
  - `Format`
    - `[token]literals(offset,match_length)[token](offset,match length)...`
      - `Literals`指没有重复、首次出现的字节流，即不可压缩的部分
      - `Match`指重复项，可以压缩的部分
      - `Token`记录literal长度，match长度。作为解压时候memcpy的参数
      
- 官方发布压缩算法测试结果：

|  Compressor             | Ratio   | Compression | Decompression |
|  ----------             | -----   | ----------- | ------------- |
|  memcpy                 |  1.000  | 13700 MB/s  |  13700 MB/s   |
|**LZ4 default (v1.9.0)** |**2.101**| **780 MB/s**| **4970 MB/s** |
|  LZO 2.09               |  2.108  |   670 MB/s  |    860 MB/s   |
|  QuickLZ 1.5.0          |  2.238  |   575 MB/s  |    780 MB/s   |
|  Snappy 1.1.4           |  2.091  |   565 MB/s  |   1950 MB/s   |
| [Zstandard] 1.4.0 -1    |  2.883  |   515 MB/s  |   1380 MB/s   |
|  LZF v3.6               |  2.073  |   415 MB/s  |    910 MB/s   |
| [zlib] deflate 1.2.11 -1|  2.730  |   100 MB/s  |    415 MB/s   |
|**LZ4 HC -9 (v1.9.0)**   |**2.721**|    41 MB/s  | **4900 MB/s** |
| [zlib] deflate 1.2.11 -6|  3.099  |    36 MB/s  |    445 MB/s   |

```
input：abcde_bcdefgh_abcdefghxxxxxxx
output：abcde_(5,4)fgh_(14,5)fghxxxxxxx
```

## `ZSTD` 算法
- `ZSTD`
  - 在2016年由`facebook` 开源的无损压缩算法。 [code](https://github.com/facebook/zstd)
  - 压缩率和压缩/解压缩性能都很突出
  - 支持训练方式生成字典。
  - 开源后， `Linux` 内核/`HTTP` 协议/`hadoop 3`/ `Hbase 2`/`spark2.3+`/`kafka 2.10` 都加入对`ZSTD`的支持 
  
- 性能
  - 对大数据量的文本压缩场景，zstd是综合考虑压缩率和压缩性能最优的选择，其次是lz4。
  - 对小数据量的压缩场景，如果能使用zstd的字典方式，压缩效果更为突出。
  - 字典特定于数据的越多，它的效率就越高（没有通用字典）
  - 

- 官方发布压缩算法测试结果：

| Compressor name         | Ratio | Compression| Decompress.|
| ---------------         | ------| -----------| ---------- |
| **zstd 1.4.4 -1**       | 2.884 |   520 MB/s |  1600 MB/s |
| zlib 1.2.11 -1          | 2.743 |   110 MB/s |   440 MB/s |
| brotli 1.0.7 -0         | 2.701 |   430 MB/s |   470 MB/s |
| quicklz 1.5.0 -1        | 2.238 |   600 MB/s |   800 MB/s |
| lzo1x 2.09 -1           | 2.106 |   680 MB/s |   950 MB/s |
| lz4 1.8.3               | 2.101 |   800 MB/s |  4220 MB/s |
| snappy 1.1.4            | 2.073 |   580 MB/s |  2020 MB/s |
| lzf 3.6 -1              | 2.077 |   440 MB/s |   930 MB/s |

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

## MergeTree

- `MergeTree` 系列
  - `ClickHouse` 最强大的 `table engines`。
  - 巨量数据批量写入，按规则合并。
  - 表数据按`primary key` 排序(小稀疏索引)， 利于快速检索数据
  - 支持分区，加快检索速度。
  - 支持数据副本(`ReplicatedMergeTree`)
  - 可以给表数据指定采样方法

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

要检查 ClickHouse 执行一个查询时能否使用索引，可设置  `force_index_by_date` 和  `force_primary_key`, 见运维部分。

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

### Concurrent Data Access

应对表的并发访问，我们使用多版本机制。换言之，当同时读和更新表时，数据从当前查询到的一组`parts`中读取。没有冗长的的锁。插入不会阻碍读取。

对表的读操作是自动并行的。

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

## Using multiple block devices for data storage

### **General**

`MergeTree` 系列的表能够将其数据存储在多个块设备上.
- 例如，当某个表的数据隐式拆分为 `hot` 和 `cold` 时，这可能会很有用。
  - 定期请求最新数据，但只需要少量空间。
  - 相反，很少要求提供详尽的历史数据。如果有多个`disk`可用，则 `hot` 数据可能位于快速`disk`（NVMe SSD 甚至是内存）中，而 `cold` 数据可能位于相对较慢的`disk`（HDD）上。

- `part` 是 `MergeTree` 表的最小可移动单位。
  - 在后台的`disk`之间（根据用户设置)
  - 通过 `ALTER` 查询来移动部件。 

### **Terms**

-`Disk`: 挂载到文件系统的块设备。
- `Default disk`-包含在 `<path>` 标签中指定的路径的`disk`config.xml`。
- `Volume`: 一组相等的有序`disk`（类似于[JBOD](https://en.wikipedia.org/wiki/Non-RAID_drive_architectures)）。
- `Storage policy`-许多 `Volume` 以及在它们之间移动数据的规则。

可以在系统表, [system.storage_policies](https://clickhouse.yandex/docs/en/operations/system_tables/#system_tables-storage_policies)和[system.disks](https://clickhouse.yandex/docs/en/operations/system_tables/#system_tables-disks)找到提供给所描述实体的名称。可以将 `storage policy` 名称用作 `MergeTree` 系列表的参数。

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


## Log Engine Family

`Log Engine` 是为了需要写入许多小数据量（少于`100万`行）的表的场景而开发的。

这系列的 `Engines` 有：

- [StripeLog](stripe-log.md)
- [Log](log.md)
- [TinyLog](tiny-log.md)

### **Common properties**

`Engines`：

- 将数据存储在`Disk`上。
- 写入时将数据`Append`到文件末尾，逐行写入。
- 支持并发数据访问锁。
  - 在 `INSERT` 查询期间，表是 `locked` 状态.
    - `locked` 状态， `read` 和 `write` 处于 `waiting` 状态。
    - `unlock` 状态， 支持并发非 `write` 操作。

- 不支持 `mutations` 操作。

- 不支持索引。
  - 范围查询效率不高。

- 不要自动写数据。定时写入的操作不建议使用，配置表（变更少，数据量小），每次都需要全量查询的表，可以用此系列 `Engines` 
  - 如果有操作中断了写操作（例如，服务器异常关闭），则可能会获得带有损坏数据的表。

### **Differences**
- `TinyLog Engine` : 
  - 是该系列中最简单的引擎，功能最差，效率最低。
  - 不支持并发 `read` 数据。
  - 所有系列 `Engines` 性能最差

- `Log Engine` 和 `StripeLog Engine`: 
  - 支持并发 `read` 操作
  - 读取数据时，`ClickHouse` 使用多个线程。
    - 每个线程处理一个单独的数据块。
  - `Log Engine` 单列一个单独的文件。
    - 提供了更高的效率。在 `Log Engine Family` 中，效率最高。
  - `StripeLog Engine` 将所有数据存储在一个文件中。
    - 在操作系统中使用较少的 `descriptors`
    
    
# Future

## 2019 年第三季度

- 字典的 `DDL`
- 与类似 `S3` 的对象存储库集成
- 多个存储热/冷数据，支持 `JBOD`

## 2019 年第四季度

- 加入不受可用内存的限制
- 资源池可在用户之间更精确地分配集群容量
- 细粒度的授权
- 与外部认证服务集成

