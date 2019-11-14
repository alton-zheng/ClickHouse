# ClickHouse 实用程序

- `clickhouse-local` : 允许在不停止 `ClickHouse` 服务器的情况下对数据运行 `SQL` 查询，类似于 `awk` 的做法。
- `clickhouse-copier` : 将数据从一个集群复制(重拆)到另一个集群。

# Clickhouse-copier

将数据从一个群集中的表复制到另一（或相同）群集中的表。

您可以在不同的服务器上运行多个 `clickhouse-copier` 实例来执行相同的任务。`ZooKeeper` 用于同步进程。

启动后，`clickhouse-copier`：

- 连接到 `ZooKeeper` 并接收：
  - 复制作业。
  - 复制作业的状态。

- 它执行工作: 
  - 每个正在运行的进程都选择源集群的`closest`分片，然后将数据复制到目标集群，并在必要时重新分片数据。

`clickhouse-copier`  跟踪 `ZooKeeper` 中的更改并实时更新状态。

为了减少网络流量，我们建议 `clickhouse-copier` 在源数据所在的同一服务器上运行。

## Running clickhouse-copier

该实用程序应手动运行：

```bash
$ clickhouse-copier copier --daemon --config zookeeper.xml --task-path /task/path --base-dir /path/to/dir
```

参数：

- `daemon`: `clickhouse-copier`以守护程序模式启动。
- `config`: `zookeeper.xml`带有用于连接 ZooKeeper 的参数的文件路径。
- `task-path`: ZooKeeper 节点的路径。该节点用于同步`clickhouse-copier`过程和存储任务。任务存储在中`$task-path/description`。
- `task-file` :带有任务配置的文件的可选路径，用于初始上传到 ZooKeeper。
- `task-upload-force`: `task-file`即使节点已经存在，也强制上载。
- `base-dir`:日志和辅助文件的路径。启动时，`clickhouse-copier` 在`$base-dir`中创建`clickhouse-copier_YYYYMMHHSS_<PID>`子目录。如果省略此参数，则在启动`clickhouse-copier`目录中创建目录。

## zookeeper.xml 的格式
```xml
<yandex>
    <logger>
        <level>trace</level>
        <size>100M</size>
        <count>3</count>
    </logger>

    <zookeeper>
        <node index="1">
            <host>127.0.0.1</host>
            <port>2181</port>
        </node>
    </zookeeper>
</yandex>
```

## Configuration of copying tasks

```xml
<yandex>
    <!-- Configuration of clusters as in an ordinary server config -->
    <remote_servers>
        <source_cluster>
            <shard>
                <internal_replication>false</internal_replication>
                    <replica>
                        <host>127.0.0.1</host>
                        <port>9000</port>
                    </replica>
            </shard>
            ...
        </source_cluster>

        <destination_cluster>
        ...
        </destination_cluster>
    </remote_servers>

    <!-- How many simultaneously active workers are possible. If you run more workers superfluous workers will sleep. -->
    <max_workers>2</max_workers>

    <!-- Setting used to fetch (pull) data from source cluster tables -->
    <settings_pull>
        <readonly>1</readonly>
    </settings_pull>

    <!-- Setting used to insert (push) data to destination cluster tables -->
    <settings_push>
        <readonly>0</readonly>
    </settings_push>

    <!-- Common setting for fetch (pull) and insert (push) operations. Also, copier process context uses it.
         They are overlaid by <settings_pull/> and <settings_push/> respectively. -->
    <settings>
        <connect_timeout>3</connect_timeout>
        <!-- Sync insert is set forcibly, leave it here just in case. -->
        <insert_distributed_sync>1</insert_distributed_sync>
    </settings>

    <!-- Copying tasks description.
         You could specify several table task in the same task description (in the same ZooKeeper node), they will be performed
         sequentially.
    -->
    <tables>
        <!-- A table task, copies one table. -->
        <table_hits>
            <!-- Source cluster name (from <remote_servers/> section) and tables in it that should be copied -->
            <cluster_pull>source_cluster</cluster_pull>
            <database_pull>test</database_pull>
            <table_pull>hits</table_pull>

            <!-- Destination cluster name and tables in which the data should be inserted -->
            <cluster_push>destination_cluster</cluster_push>
            <database_push>test</database_push>
            <table_push>hits2</table_push>

            <!-- Engine of destination tables.
                 If destination tables have not be created, workers create them using columns definition from source tables and engine
                 definition from here.

                 NOTE: If the first worker starts insert data and detects that destination partition is not empty then the partition will
                 be dropped and refilled, take it into account if you already have some data in destination tables. You could directly
                 specify partitions that should be copied in <enabled_partitions/>, they should be in quoted format like partition column of
                 system.parts table.
            -->
            <engine>
            ENGINE=ReplicatedMergeTree('/clickhouse/tables/{cluster}/{shard}/hits2', '{replica}')
            PARTITION BY toMonday(date)
            ORDER BY (CounterID, EventDate)
            </engine>

            <!-- Sharding key used to insert data to destination cluster -->
            <sharding_key>jumpConsistentHash(intHash64(UserID), 2)</sharding_key>

            <!-- Optional expression that filter data while pull them from source servers -->
            <where_condition>CounterID != 0</where_condition>

            <!-- This section specifies partitions that should be copied, other partition will be ignored.
                 Partition names should have the same format as
                 partition column of system.parts table (i.e. a quoted text).
                 Since partition key of source and destination cluster could be different,
                 these partition names specify destination partitions.

                 NOTE: In spite of this section is optional (if it is not specified, all partitions will be copied),
                 it is strictly recommended to specify them explicitly.
                 If you already have some ready paritions on destination cluster they
                 will be removed at the start of the copying since they will be interpeted
                 as unfinished data from the previous copying!!!
            -->
            <enabled_partitions>
                <partition>'2018-02-26'</partition>
                <partition>'2018-03-05'</partition>
                ...
            </enabled_partitions>
        </table_hits>

        <!-- Next table to copy. It is not copied until previous table is copying. -->
        </table_visits>
        ...
        </table_visits>
        ...
    </tables>
</yandex>
```

`clickhouse-copier` 跟踪 `/task/path/description` 更改并实时更新他们状态。
例如，如果更改 `max_workers`的值，则运行任务的进程数也会更改。

# Clickhouse-Local

- `Clickhouse-Local` 程序允许您对本地文件执行快速处理，而无需部署和配置 `ClickHouse` 服务器。
  - 接收 `represent table` 的数据并使用 `ClickHouse SQL` 方言查询它们。
  - `Clickhouse-Local` 使用与 `ClickHouse` 服务器相同的 `cpu core` 数，因此它支持大多数特性以及相同的格式和 `table engine`。
  - 默认情况下，`Clickhouse-Local` 不能访问同一主机上的数据，但它支持使用 `--config-file` 参数加载服务器配置。
  - 不建议将生产服务器配置加载到 `Clickhouse-Local`，因为在出现人为错误时数据可能会被损坏。

基本用法:

```bash
$ clickhouse-local --structure "table_structure" --input-format "format_of_incoming_data" -q "query"
```

- 参数:
  - `-S`, `--structure` :   输入数据的表结构。
  - `-if`, `--input-format`:  输入格式，默认 `TSV`。
  - `-f`, `--file` : 数据路径，默认为`stdin`。
  - `-q`, `--query` : 要执行的查询 `;` 结尾。
  - `-N`, `--table` : 用于存放输出数据表名，默认为 `table`。
  - `-of`, `--format`, `--output-format` : 输出格式, 默认`TSV`。
  - `--stacktrace` : 在异常情况下是否转储调试输出。
  - `-verbose` : 查询执行的更多细节。
  - `-s` : 禁用 `stderr` 日志。
  - `--config-file` : 配置文件的路径，格式与`ClickHouse`服务器相同，默认配置为空。
  - `--help` : `clickhouse-local`的参数帮助信息。


- 例如： 
```log
$ echo -e "1,2\n3,4" | clickhouse-local -S "a Int64, b Int64" -if "CSV" -q "SELECT * FROM table"
Read 2 rows, 32.00 B in 0.000 sec., 5182 rows/sec., 80.97 KiB/sec.
1   2
3   4

$ echo -e "1,2\n3,4" | clickhouse-local -q "CREATE TABLE table (a Int64, b Int64) ENGINE = File(CSV, stdin); SELECT a, b FROM table; DROP TABLE table"
Read 2 rows, 32.00 B in 0.000 sec., 4987 rows/sec., 77.93 KiB/sec.
1   2
3   4
```

- 现在为每个Unix用户输出内存用户:
```log
$ ps aux | tail -n +2 | awk '{ printf("%s\t%s\n", $1, $4) }' | clickhouse-local -S "user String, mem Float64" -q "SELECT user, round(sum(mem), 2) as memTotal FROM table GROUP BY user ORDER BY memTotal DESC FORMAT Pretty"
Read 186 rows, 4.15 KiB in 0.035 sec., 5302 rows/sec., 118.34 KiB/sec.
┏━━━━━━━━━━┳━━━━━━━━━━┓
┃ user     ┃ memTotal ┃
┡━━━━━━━━━━╇━━━━━━━━━━┩
│ bayonet  │    113.5 │
├──────────┼──────────┤
│ root     │      8.8 │
├──────────┼──────────┤
...
```


