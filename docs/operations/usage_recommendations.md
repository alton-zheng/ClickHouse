# 使用建议(`Usage recommendations`)

## CPU 扩展调节器(`CPU Scaling Governor`)

始终使用`performance` `Scaling Governor`。该`on-demand` `Scaling Governor` 与不断的高需求差很多。

```bash
$ echo 'performance' | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
```

## CPU Limitations

处理器可能会过热。使用`dmesg`，看看  的 `Clock rate` 被由过热的`CPU`限制。也可以在数据中心级别在外部设置限制。您可以使用`turbostat`它来监视负载。

## RAM

- 对于少量数据（最多压缩约 `200 GB`），最好使用与数据量一样多的内存。
- 对于大量数据，以及在处理交互式（在线）查询时，应使用合理数量的 `RAM`（`128 GB` 或更多），以便热数据子集, 适合页面缓存。
- 即使对于每台服务器约 50 TB 的数据量，与 64 GB 相比，使用 128 GB 的 RAM 也可以显着提高查询性能。

不要禁用过量使用。该值`cat /proc/sys/vm/overcommit_memory`应为 0 或 1。运行

```bash
$ echo 0 | sudo tee /proc/sys/vm/overcommit_memory
```

## Huge Pages

始终禁用透明的大页面。它会干扰内存分配器，从而导致性能显着下降。

```bash
$ echo 'never' | sudo tee /sys/kernel/mm/transparent_hugepage/enabled
```

使用` perf top` 观看的内核内存管理花费的时间。永久的大页面也不需要分配。

## Storage Subsystem

- 尽量使用好的磁盘：
-> `SSD` -> `HDD`(`SATA HDD 7200 RPM`)
-> `Server`(`local hard drives`) -> 云服务器（少数具有附加磁盘架(`shelves`））

## RAID

使用 HDD 时，可以组合使用它们的 `RAID-10`，`RAID-5`，`RAID-6` 或 `RAID-50`。
对于 Linux，软件 RAID 更好（带有`mdadm`）。我们不建议使用 LVM。创建 `RAID-10` 时，选择`far`布局。如果预算允许，请选择 `RAID-10`。

如果您有四个以上的磁盘，请使用 `RAID-6（首选）` 或 `RAID-50`，而不是 `RAID-5`。使用 `RAID-5`，`RAID-6` 或 `RAID-50` 时，请增加 `stripe_cache_size`，因为默认值通常不是最佳选择。

```bash
$ echo 4096 | sudo tee /sys/block/md2/md/stripe_cache_size
```

从设备的数目和块的大小计算出准确的数，使用下式：`2 * num_devices * chunk_size_in_bytes / 4096`。

1024 KB 的块大小足以用于所有 `RAID` 配置。切勿将块大小设置得太小或太大。

您可以在 `SSD` 上使用 `RAID-0`。无论使用 `RAID`，始终使用复制来确保数据安全。

`long queue` 启用 `NCQ`。对于 `HDD`，选择 `CFQ` 调度程序，对于 `SSD`，选择 `noop`。不要减少“预读”设置。对于 HDD，启用写缓存。

## File System

- `Ext4` 是最可靠的选择。
  - 设置安装选项`noatime, nobarrier`。
  - `XFS` 也适用，但是尚未通过 ClickHouse 进行全面测试。
  - 大多数其他文件系统也应该可以正常工作。延迟分配的文件系统可以更好地工作。

## Linux 内核

不要使用过时的 Linux 内核。

## Network

如果使用的是 `IPv6`，请增加路由缓存的大小。3.2 之前的 Linux 内核在 `IPv6` 实现方面存在很多问题。

如果可能，请至少使用 10 GB 的网络。`万兆`

1 Gb 也可以使用，但是如果用数十兆字节的数据修补副本，或者使用大量中间数据处理分布式查询，情况将更糟。

## Zookeeper

ZooKeeper `fresh version` – 3.4.9 或更高版本。稳定的 Linux 发行版中的版本可能已过时。

您永远不要使用手动编写的脚本在不同的 ZooKeeper 群集之间传输数据，因为对于顺序节点，结果将是不正确的。出于相同的原因，切勿使用 "zkcopy" ：https://github.com/ksprojects/zkcopy/issues/15

如果要将现有的 ZooKeeper 群集分为两个，正确的方法是增加其副本的数量，然后将其重新配置为两个独立的群集。

不要在与 `ClickHouse` 相同的服务器上运行 ZooKeeper。因为 ZooKeeper 对延迟非常敏感，因此 `ClickHouse` 可能会利用所有可用的系统资源。

使用默认设置，`ZooKeeper` 是定时炸弹：

> 使用默认配置（请参阅自动清除）时，ZooKeeper 服务器不会从旧的快照和日志中删除文件，这是操作员的责任。

必须拆除这枚炸弹。

截至 2017 年 5 月 20 日，以下的 ZooKeeper（3.5.1）配置在 Yandex.Metrica 生产环境中使用：

zoo.cfg：

```cfg
# http://hadoop.apache.org/zookeeper/docs/current/zookeeperAdmin.html

# The number of milliseconds of each tick
tickTime=2000
# The number of ticks that the initial
# synchronization phase can take
initLimit=30000
# The number of ticks that can pass between
# sending a request and getting an acknowledgement
syncLimit=10

maxClientCnxns=2000

maxSessionTimeout=60000000
# the directory where the snapshot is stored.
dataDir=/opt/zookeeper/{{ cluster['name'] }}/data
# Place the dataLogDir to a separate physical disc for better performance
dataLogDir=/opt/zookeeper/{{ cluster['name'] }}/logs

autopurge.snapRetainCount=10
autopurge.purgeInterval=1


# To avoid seeks ZooKeeper allocates space in the transaction log file in
# blocks of preAllocSize kilobytes. The default block size is 64M. One reason
# for changing the size of the blocks is to reduce the block size if snapshots
# are taken more often. (Also, see snapCount).
preAllocSize=131072

# Clients can submit requests faster than ZooKeeper can process them,
# especially if there are a lot of clients. To prevent ZooKeeper from running
# out of memory due to queued requests, ZooKeeper will throttle clients so that
# there is no more than globalOutstandingLimit outstanding requests in the
# system. The default limit is 1,000.ZooKeeper logs transactions to a
# transaction log. After snapCount transactions are written to a log file a
# snapshot is started and a new transaction log file is started. The default
# snapCount is 10,000.
snapCount=3000000

# If this option is defined, requests will be will logged to a trace file named
# traceFile.year.month.day.
#traceFile=

# Leader accepts client connections. Default value is "yes". The leader machine
# coordinates updates. For higher update throughput at thes slight expense of
# read throughput the leader can be configured to not accept clients and focus
# on coordination.
leaderServes=yes

standaloneEnabled=false
dynamicConfigFile=/etc/zookeeper-{{ cluster['name'] }}/conf/zoo.cfg.dynamic
```

**Java version**:

```
Java(TM) SE Runtime Environment (build 1.8.0_25-b17)
Java HotSpot(TM) 64-Bit Server VM (build 25.25-b02, mixed mode)
```

**JVM parameters**:

```
NAME=zookeeper-{{ cluster['name'] }}
ZOOCFGDIR=/etc/$NAME/conf

# TODO this is really ugly
# How to find out, which jars are needed?
# seems, that log4j requires the log4j.properties file to be in the classpath
CLASSPATH="$ZOOCFGDIR:/usr/build/classes:/usr/build/lib/*.jar:/usr/share/zookeeper/zookeeper-3.5.1-metrika.jar:/usr/share/zookeeper/slf4j-log4j12-1.7.5.jar:/usr/share/zookeeper/slf4j-api-1.7.5.jar:/usr/share/zookeeper/servlet-api-2.5-20081211.jar:/usr/share/zookeeper/netty-3.7.0.Final.jar:/usr/share/zookeeper/log4j-1.2.16.jar:/usr/share/zookeeper/jline-2.11.jar:/usr/share/zookeeper/jetty-util-6.1.26.jar:/usr/share/zookeeper/jetty-6.1.26.jar:/usr/share/zookeeper/javacc.jar:/usr/share/zookeeper/jackson-mapper-asl-1.9.11.jar:/usr/share/zookeeper/jackson-core-asl-1.9.11.jar:/usr/share/zookeeper/commons-cli-1.2.jar:/usr/src/java/lib/*.jar:/usr/etc/zookeeper"

ZOOCFG="$ZOOCFGDIR/zoo.cfg"
ZOO_LOG_DIR=/var/log/$NAME
USER=zookeeper
GROUP=zookeeper
PIDDIR=/var/run/$NAME
PIDFILE=$PIDDIR/$NAME.pid
SCRIPTNAME=/etc/init.d/$NAME
JAVA=/usr/bin/java
ZOOMAIN="org.apache.zookeeper.server.quorum.QuorumPeerMain"
ZOO_LOG4J_PROP="INFO,ROLLINGFILE"
JMXLOCALONLY=false
JAVA_OPTS="-Xms{{ cluster.get('xms','128M') }} \
    -Xmx{{ cluster.get('xmx','1G') }} \
    -Xloggc:/var/log/$NAME/zookeeper-gc.log \
    -XX:+UseGCLogFileRotation \
    -XX:NumberOfGCLogFiles=16 \
    -XX:GCLogFileSize=16M \
    -verbose:gc \
    -XX:+PrintGCTimeStamps \
    -XX:+PrintGCDateStamps \
    -XX:+PrintGCDetails
    -XX:+PrintTenuringDistribution \
    -XX:+PrintGCApplicationStoppedTime \
    -XX:+PrintGCApplicationConcurrentTime \
    -XX:+PrintSafepointStatistics \
    -XX:+UseParNewGC \
    -XX:+UseConcMarkSweepGC \
-XX:+CMSParallelRemarkEnabled"
```

**Salt init**：

```
description "zookeeper-{{ cluster['name'] }} centralized coordination service"

start on runlevel [2345]
stop on runlevel [!2345]

respawn

limit nofile 8192 8192

pre-start script
    [ -r "/etc/zookeeper-{{ cluster['name'] }}/conf/environment" ] || exit 0
    . /etc/zookeeper-{{ cluster['name'] }}/conf/environment
    [ -d $ZOO_LOG_DIR ] || mkdir -p $ZOO_LOG_DIR
    chown $USER:$GROUP $ZOO_LOG_DIR
end script

script
    . /etc/zookeeper-{{ cluster['name'] }}/conf/environment
    [ -r /etc/default/zookeeper ] && . /etc/default/zookeeper
    if [ -z "$JMXDISABLE" ]; then
        JAVA_OPTS="$JAVA_OPTS -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.local.only=$JMXLOCALONLY"
    fi
    exec start-stop-daemon --start -c $USER --exec $JAVA --name zookeeper-{{ cluster['name'] }} \
        -- -cp $CLASSPATH $JAVA_OPTS -Dzookeeper.log.dir=${ZOO_LOG_DIR} \
        -Dzookeeper.root.logger=${ZOO_LOG4J_PROP} $ZOOMAIN $ZOOCFG
end script
```