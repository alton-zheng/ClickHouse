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
```xml
Format	                      Input	Output
TabSeparated	                ✔	✔
TabSeparatedRaw	                ✗	✔
TabSeparatedWithNames	        ✔	✔
TabSeparatedWithNamesAndTypes	✔	✔
Template     	                ✔	✔
TemplateIgnoreSpaces	        ✔	✗
CSV                             ✔	✔
CSVWithNames	                ✔	✔
CustomSeparated	                ✔	✔
Values                      	✔	✔
Vertical                    	✗	✔
JSON                        	✗	✔
JSONCompact                     ✗	✔
JSONEachRow                     ✔	✔
TSKV	                        ✔	✔
Pretty	                        ✗	✔
PrettyCompact	                ✗	✔
PrettyCompactMonoBlock	        ✗	✔
PrettyNoEscapes             	✗	✔
PrettySpace                     ✗	✔
Protobuf                        ✔	✔
Parquet	                        ✔	✔
RowBinary                       ✔	✔
RowBinaryWithNamesAndTypes      ✔	✔
Native	                        ✔	✔
Null	                        ✗	✔
XML                             ✗	✔
CapnProto                       ✔	✗
```

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

