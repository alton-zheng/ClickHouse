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