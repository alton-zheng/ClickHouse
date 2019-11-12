# Data Backup

尽管[复制](https://clickhouse.yandex/docs/en/operations/table_engines/replication/)提供了针对硬件故障的保护，但它不能防止人为错误：意外删除数据，删除错误的表或错误的群集上的表以及导致错误数据处理或数据损坏的软件错误。在许多情况下，此类错误会影响所有副本。ClickHouse 具有内置的防护措施，可以防止某些类型的错误-例如，默认情况下，[您不能仅使用包含超过 50 Gb 数据的类 MergeTree 引擎删除表](https://github.com/ClickHouse/ClickHouse/blob/v18.14.18-stable/dbms/programs/server/config.xml#L322-L330)。但是，这些保护措施并未涵盖所有可能的情况，可以规避。

为了有效减轻可能的人为错误，您应该仔细准备一项策略，**in advance**备份和还原数据。

`ClickHouse` 备份和还原, 没有通用方案，每个公司根据需求和数据量自行选择

- `Note`

数据备份和还原，对于不同的方案，都要经过严密的测试，在生产环境才能得心应手，游刃有余： 

## Duplicating Source Data Somewhere Else

通常，通过某种持久性队列（例如[Apache Kafka](https://kafka.apache.org/)）传递被摄入 ClickHouse 的数据。在这种情况下，可以配置其他订阅者集，这些订阅者集将在将同一数据流写入 ClickHouse 时读取该数据流，并将其存储在某处的冷存储器中。大多数公司已经有一些默认的推荐冷存储，它可以是对象存储或像[HDFS](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/HdfsDesign.html)这样的分布式文件系统。

## Filesystem Snapshots

一些本地文件系统提供快照功能（例如[ZFS](https://en.wikipedia.org/wiki/ZFS)），但是它们可能不是服务实时查询的最佳选择。一种可能的解决方案是使用这种文件系统创建其他副本，并将其从用于查询的[分布式](https://clickhouse.yandex/docs/en/operations/table_engines/distributed/)表中排除`SELECT`。此类副本上的快照将超出任何修改数据的查询。另外，这些副本可能具有特殊的硬件配置，每台服务器附加了更多磁盘，这将具有成本效益。

## clickhouse copier

[clickhouse-copier](utilities.md)是一种多功能工具:
- 重新分割 PB 级表的。
- 备份和还原数据（服务器和分布式架构）

对于较小的数据量，也可以使用简单`INSERT INTO ... SELECT ...` 通过网络写入到远程表。

## **Maipulations with Parts**

`ClickHouse` 允许使用`ALTER TABLE ... FREEZE PARTITION ...` `query` 创建表分区的本地副本。
这是通过使用到`/var/lib/clickhouse/shadow/`文件夹的硬链接来实现的，因此通常不会为旧数据占用额外的磁盘空间。
ClickHouse 服务器不会处理创建的文件副本，因此您可以将它们保留在此处：您将获得一个简单的备份，不需要任何其他外部系统，但是仍然容易出现硬件问题。
因此，最好将它们远程复制到另一个位置，然后删除本地副本。
分布式文件系统和对象存储仍然是一个不错的选择，但是具有足够大容量的普通附加文件服务器也可以工作（在这种情况下，传输将通过网络文件系统或[rsync](https://en.wikipedia.org/wiki/Rsync)进行)。

有关与分区操作有关的查询的更多信息，请参见[ALTER 文档](https://clickhouse.yandex/docs/en/query_language/alter/#alter_manipulations-with-partitions)。

第三方工具可用于自动执行此方法：[clickhouse-backup](https://github.com/AlexAkulov/clickhouse-backup)
