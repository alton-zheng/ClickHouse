# ClickHouse 实用程序

- [clickhouse-local-](https://clickhouse.yandex/docs/en/operations/utils/clickhouse-local/)允许在不停止 ClickHouse 服务器的情况下对数据运行 SQL 查询，类似于`awk`此操作。
- [clickhouse-](https://clickhouse.yandex/docs/en/operations/utils/clickhouse-copier/) copier-单击数据[复制器](https://clickhouse.yandex/docs/en/operations/utils/clickhouse-copier/) -将数据从一个群集复制（和重新分片）到另一个群集。

# Clickhouse 复印机

将数据从一个群集中的表复制到另一（或相同）群集中的表。

您可以`clickhouse-copier`在不同的服务器上运行多个实例以执行相同的工作。ZooKeeper 用于同步进程。

启动后，`clickhouse-copier`：

- 连接到 ZooKeeper 并接收：

  - 复制作业。
  - 复印作业的状态。

- 它执行工作。

  每个正在运行的进程都选择源集群的“最近”分片，然后将数据复制到目标集群，并在必要时重新分片数据。

`clickhouse-copier`  跟踪 ZooKeeper 中的更改并将其动态应用。

为了减少网络流量，我们建议`clickhouse-copier`在源数据所在的同一服务器上运行。

## 运行 clickhouse，复印机[¶](https://clickhouse.yandex/docs/en/operations/utils/clickhouse-copier/#running-clickhouse-copier "永久链接")

该实用程序应手动运行：

\$ clickhouse-copier 复印机--daemon --config zookeeper.xml --task-path / task / path --base-dir / path / to / dir

参数：

- `daemon`— `clickhouse-copier`以守护程序模式启动。
- `config`— `zookeeper.xml`带有用于连接 ZooKeeper 的参数的文件路径。
- `task-path`— ZooKeeper 节点的路径。该节点用于同步`clickhouse-copier`过程和存储任务。任务存储在中`$task-path/description`。
- `task-file` —带有任务配置的文件的可选路径，用于初始上传到 ZooKeeper。
- `task-upload-force`— `task-file`即使节点已经存在，也强制上载。
- `base-dir`—日志和辅助文件的路径。启动时，在中`clickhouse-copier`创建`clickhouse-copier_YYYYMMHHSS_<PID>`子目录`$base-dir`。如果省略此参数，则在启动目录中创建目录`clickhouse-copier`。

## zookeeper.xml 的格式[¶](https://clickhouse.yandex/docs/en/operations/utils/clickhouse-copier/#format-of-zookeeper-xml "永久链接")

<yandex> 
    <logger> 
        <level>跟踪</ level> 
        <size> 100M </ size> 
        <count> 3 </ count> 
    </ logger>

    <zookeeper>
        <node  index = “ 1” \>
            <host> 127.0.0.1 </ host>
            <port> 2181 </ port>
        </ node>
    </ zookeeper>

</ yandex>

## 复制任务配置[¶](https://clickhouse.yandex/docs/en/operations/utils/clickhouse-copier/#configuration-of-copying-tasks "永久链接")

<yandex> 
    <！-与普通服务器配置中一样配置群集-\> 
    <remote_servers> 
        <source_cluster> 
            <shard> 
                <internal_replication> false </ internal_replication> 
                    <replica> 
                        <host> 127.0.0.1 </ host> 
                        < port> 9000 </ port> 
                    </ replica> 
            </ shard>
            ...
        </ source_cluster>

        <目标群集>
        ...
        </ destination_cluster>
    </ remote_servers>

    <！-可以同时活跃多少工人。如果您运行更多的工人，多余的工人将入睡。-\>
    <max_workers> 2 </ max_workers>

    <！-用于从源群集表中获取（拉出）数据的
    设置
        -\> <settings_pull> <readonly> 1 </ readonly>
    </ settings_pull>

    <！-用于将数据插入（推入）目标集群表的
    设置
        -\> <settings_push> <readonly> 0 </ readonly>
    </ settings_push>

    <！-获取（拉）和插入（推）操作的常用设置。同样，复印机过程上下文也使用它。

它们分别由<settings_pull />和<settings_push />覆盖。->
<设置\>
<连接超时> 3 </连接超时\>
<！-强制设置了同步插入，以防万一。-\>
<插入插入的同步> 1 </插入插入的同步\>
</设置>

    <！-复制任务说明。

您可以在同一任务描述中（在同一 ZooKeeper 节点中）指定多个表任务，它们将按
顺序
执行。 -\>
<表\>
<！-表任务，复制一个表。-\>
<table_hits>
<！-源群集名称（来自<remote_servers />部分）和应复制的表->
<cluster_pull> source_cluster </ cluster_pull>
<database_pull> test </ database_pull>
<table_pull >点击</ table_pull>

            <！-目标群集名称和应在其中插入数据的表-\>
            <cluster_push> destination_cluster </ cluster_push>
            <database_push>测试</ database_push>
            <table_push> hits2 </ table_push>

            <！-目标表的引擎。

如果尚未创建目标表，则工作人员将使用源表中的列定义和
此处的引擎定义来创建它们。

注意：如果第一个工作程序开始插入数据并检测到目标分区不为空，则该分区将
被删除并重新填充，如果目标表中已经有一些数据，请考虑到
该分区。您可以直接 指定应在<enabled_partitions />中复制的分区，它们应采用带引号的格式，例如
system.parts 表的
分区列。 -\>
<引擎>
ENGINE = ReplicatedMergeTree（'/ clickhouse / tables / {cluster} / {shard} / hits2'，'{replica}'）
截止时间：星期一（日期）
ORDER BY（CounterID，EventDate）
</ engine>

            <！-用于将数据插入到目标群集的
            分片键-\> <sharding_key> jumpConsistentHash（intHash64（UserID），2）</ sharding_key>

            <！-可选表达式，用于在从源服务器提取数据时过滤数据-\>
            <where_condition> CounterID！= 0 </ where_condition>

            <！-本节指定应复制的分区，其他分区将被忽略。

分区名称应
与 system.parts 表的分区列
具有相同的格式（即带引号的文本）。 由于源群集和目标群集的分区键可能不同，因此
这些分区名称指定了目标分区。

注意：尽管本节是可选的（如果未指定，则将复制所有分区），
但强烈建议您明确指定它们。
如果目标群集上已经有一些可用的分区，则它们
将在复制开始时被删除，因为它们将被
作为上一次复制中未完成的数据插入！
-\>
<enabled_partitions>
<partition> '2018-02-26' </ partition>
<partition> '2018-03-05' </ partition>
...
</ enabled_partitions>
</ table_hits>

        <！-下一张要复制的表格。在复制上一个表之前，不会复制它。-\>
        </ table_visits>
        ...
        </ table_visits>
        ...
    </ tables>

</ yandex>

`clickhouse-copier`跟踪更改`/task/path/description`并实时应用它们。例如，如果更改的值`max_workers`，则运行任务的进程数也会更改。

# Clickhouse-Local

该`clickhouse-local`程序使您可以对本地文件执行快速处理，而不必部署和配置 ClickHouse 服务器。

接受代表表格的数据，并使用[ClickHouse SQL 方言](https://clickhouse.yandex/docs/en/query_language/)查询它们。

`clickhouse-local`  使用与 ClickHouse 服务器相同的核心，因此它支持大多数功能以及相同的格式和表引擎集。

默认情况下`clickhouse-local`，不能访问同一主机上的数据，但是它支持使用`--config-file`参数加载服务器配置。

警告

不建议将生产服务器配置加载到其中，`clickhouse-local`因为如果发生人为错误，数据可能会损坏。

## 用法[¶](https://clickhouse.yandex/docs/en/operations/utils/clickhouse-local/#usage "永久链接")

基本用法：

\$ clickhouse-local-结构“ table_structure” -输入格式“ format_of_incoming_data” -q “查询”

参数：

- `-S`，`--structure`—输入数据的表结构。
- `-if`，`--input-format`— `TSV`默认情况下，输入格式。
- `-f`，`--file`— `stdin`默认情况下，数据路径。
- `-q` `--query`—以`;`as 分隔符执行的查询。
- `-N`，`--table`— `table`默认情况下，将表名放在输出数据的位置。
- `-of`，`--format`，`--output-format`-输出格式，`TSV`默认情况下。
- `--stacktrace` —是否在异常情况下转储调试输出。
- `--verbose` -有关查询执行的更多详细信息。
- `-s`—禁用`stderr`日志记录。
- `--config-file` —与 ClickHouse 服务器格式相同的配置文件路径，默认情况下配置为空。
- `--help`—的参数参考`clickhouse-local`。

此外，每个 ClickHouse 配置变量都有一些参数，这些参数更常用而不是`--config-file`。

## 例子[¶](https://clickhouse.yandex/docs/en/operations/utils/clickhouse-local/#examples "永久链接")

\$ echo -e “ 1,2 \ n3,4” | clickhouse-local -S “ a Int64，b Int64” -if “ CSV” -q “ SELECT \* FROM table”
读取 2 行，32 .00 B in 0 .000 秒，5182 行/秒，80 .97 KiB /秒。
1 2
3 4

上一个示例与以下示例相同：

\$ echo -e “ 1,2 \ n3,4” | clickhouse 本地-q “CREATE TABLE 表（一个的 Int64，B 的 Int64）ENGINE =文件（CSV，标准输入）;选择一个，B，从表; DROP TABLE 表”
阅读 2 行，32 在 0.00 乙 0 0.000 秒。 ，4987 行/秒，77 .93 KiB /秒。
1 2
3 4

现在让我们为每个 Unix 用户输出内存用户：

$ ps aux | 尾-n +2 | awk '{printf（“％s \ t％s \ n”，$ 1，\$ 4）}' | | clickhouse-local -S “用户字符串，mem Float64” -q “选择用户，将 round（sum（mem），2）作为 memTotal 从表 GROUP BY 用户 ORDER BY memTotal DESC FORMAT 漂亮”

读取 186 行，0.035 秒中的 4.15 KiB，5302 行/秒，118.34 KiB /秒。
━━━━━━━━━┳━━━━━━━━━━┓
┃ 用户 ┃memTotal┃
━━━━━━━━━╇━━━━━━━━━━┩
│ 卡口 │113.5│
├────────┼
│ 根 │8.8│
├────────┼
...