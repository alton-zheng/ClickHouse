## Distributed

- **`Distributed engine` 本身不存储数据**, 但可以在多个服务器上进行分布式查询。
  - 读是自动并行的。
  - 读取时，远程服务器表的索引（如果有的话）会被使用。 
  - `Distributed engine` 参数：
    - 服务器配置文件中的集群名;
    - 远程数据库名;
    - 远程表名;
    - `data shard` 键（可选）。 
 
示例：

```Distributed(logs, default, hits[, sharding_key])```

- 将会从位于“logs”集群中 default.hits 表所有服务器上读取数据。 
  - 远程服务器不仅用于读取数据，还会对尽可能数据做部分处理。 
    - 例如，对于使用 GROUP BY 的查询，数据首先在远程服务器聚合，之后返回聚合函数的中间状态给查询请求的服务器。
    - 再在请求的服务器上进一步汇总数据。

数据库名参数除了用数据库名之外，也可用返回字符串的常量表达式。例如：`currentDatabase()`

logs – 服务器配置文件中的集群名称。

集群示例配置如下：

```xml
<remote_servers>
    <logs>
        <shard>
            <!-- Optional. Shard weight when writing data. Default: 1. -->
            <weight>1</weight>
            <!-- Optional. Whether to write data to just one of the replicas. Default: false (write data to all replicas). -->
            <internal_replication>false</internal_replication>
            <replica>
                <host>example01-01-1</host>
                <port>9000</port>
            </replica>
            <replica>
                <host>example01-01-2</host>
                <port>9000</port>
            </replica>
        </shard>
        <shard>
            <weight>2</weight>
            <internal_replication>false</internal_replication>
            <replica>
                <host>example01-02-1</host>
                <port>9000</port>
            </replica>
            <replica>
                <host>example01-02-2</host>
                <secure>1</secure>
                <port>9440</port>
            </replica>
        </shard>
    </logs>
</remote_servers>
```

- `logs` 集群：
  - 两个 `shard` 组成
    - 每个 `shard` 包含两个 `replicas`。
    - `shard` 是指包含数据不同部分的服务器（要读取所有数据，必须访问所有 `shard`）。 
    - `replicas` 是存储备份数据的服务器（要读取所有数据，访问任一 `replicas` 上的数据即可）。

集群名称不能包含点号。

- 每个服务器需要指定  `host`，`port`，和可选的  `user`，`password`，`secure`，`compression`  的参数：

  - `host` : 
    - 远程服务器地址。
    - 可以域名、IPv4 或 IPv6。
    - 如果指定域名，则服务在启动时发起一个 DNS 请求，并且请求结果会在服务器运行期间一直被记录。
      - 如果 DNS 请求失败，则服务不会启动。
      - 如果你修改了 DNS 记录，则需要重启服务。
      
  - `port` :
    - 消息传递的 TCP 端口（「tcp_port」配置通常设为 9000）。
    - 不要跟 http_port 混淆。
    
  - `user` : 用于连接远程服务器的用户名。
    - 默认值：`default`
      - 该用户必须有权限访问该远程服务器。
      - 访问权限配置在 users.xml 文件中。
      - 更多信息，请查看[Access right](../../operations/access_rights.md)部分。
      
  - `password` : 
    - 用于连接远程服务器的密码。
    - 默认值：`""`(空串)
    
  - `secure` : 
    - 是否使用 `ssl` 进行连接，设为 `true` 时，通常也应该设置 `port` = `9440`。
    - 服务器也要监听  `9440` 并有正确的证书。
    
  - `compression` : 是否使用数据压缩。默认值：`true`

- 配置了 `replicas`，读取操作会从每个 `shard` 里选择一个可用的 `replicas`。
  - 可配置负载平衡算法（挑选 `replicas` 的方式）请参阅[load_balancing](../../operations/settings/settings.md)设置。
  - 如果跟服务器的连接不可用，则在尝试短超时的重连。
    - 如果重连失败，则选择下一个 `replicas`，依此类推。
    - 如果跟所有 `replicas` 的连接尝试都失败，则尝试用相同的方式再重复几次。
    - 该机制有利于系统可用性，但不保证完全容错：如有远程服务器能够接受连接，但无法正常工作或状况不佳。

- 你可以配置一个（这种情况下，查询操作更应该称为远程查询，而不是分布式查询）或任意多个 `shard`
  - 在每个 `shard` 中，可以配置一个或任意多个 `replicas`
  - 不同 `shard` 可配置不同数量的 `replicas`

- 可以在配置中配置任意数量的集群。
  - 使用 `'system.clusters'` 表, 查看集群。

- 通过分布式引擎可以像使用本地服务器一样使用集群。
  - 集群不是自动扩展的：你必须编写集群配置到服务器配置文件中（最好，给所有集群的服务器写上完整配置）。

- 不支持用分布式表查询别的分布式表（除非该表只有一个`shard`）。或者说，要用分布表查查询 `"final"` 的数据表。

- 分布式引擎需要将集群信息写入配置文件。
  - 配置文件中的集群信息会即时更新，无需重启服务器。
  - 如果你每次是要向不确定的一组 `shard` 和 `replicas` 发送查询，则不适合创建分布式表 - 而应该使用“远程”表函数。 请参阅[table function]()部分。

---

### 集群写数方法：
- 自已指定要将哪些数据写入哪些服务器，并直接在每个`shard`上执行写入
  - 换句话说，在分布式表上 `SELECT`，在数据表上 `INSERT`
  - 这是最灵活的解决方案
    - 你可以使用任何 `replicas` 方案，对于复杂业务特性的需求，这可能是非常重要的
    - 这也是最佳解决方案，因为数据可以完全独立地写入不同的 `replicas`

- 在分布式表上执行 `INSERT`。
  - 在这种情况下，分布式表会跨服务器分发插入数据。
  - 为了写入分布式表，必须要配置`sharding_key`（最后一个参数）。
  - 当然，如果只有一个 `shard`，则写操作在没有 `sharding_key` 的情况下也能工作，因为这种情况下 `sharding_key` 没有意义。

---

### `shard` 权重( `weight` )

- 每个 `shard` 都可以在配置文件中定义权重.
  - 默认情况下，权重等于 1.
  - 数据依据 `shard` 权重按比例分发到 `shard` 上.
    - 例如，如果有两个 `shard`，第一个 `shard` 的权重是 `9`，而第二个 `shard` 的权重是 `10`.
      - 发送 `9/19` 的行到第一个 `shard`. 
      - 发送 `10/19` 行到第二个 `shard`.

---

### `internal_replication` （ `shard` 的 参数）

- `true` : 
  - 写操作只选一个正常的 `replicas` 写入数据。
    - 如果写的是 `replicas table` ( `*ReplicaMergeTree` )，请使用此方案。
    - `*ReplicaMergeTree` 自身负责备份操作。
    - 不需要分布式表来对数据进行备份。

- `false` (默认)： 
  - 写操作会将数据写入所有 `replicas`。
    - 分布式表本身来复制数据。
    - 因为不会检查 `replicas` 的一致性，并且随着时间的推移，副本数据可能会有些不一样。
    - 没 `*ReplicaMergeTree` 容错性好。

---

### 数据行如何分配 `shard`

- 分配规则：
  - 首先计算 `shard` 表达式
  - 然后将这个计算结果除以所有 `shard` 的权重总和得到余数。
  - 该行会发送到余数在 [`'prev_weight'`, `'prev_weights + weight'`) 区间对应的 `shard` 上
    - `'prev_weights'` 是该`shard`前面的所有`shard`的权重和
    - `'weight'` 是该 `shard` 的权重。
    - 例如，如果有两个 `shard`，第一个 `shard` 权重为 9，而第二个 `shard` 权重为 10
      - 余数在 `[0, 9)` 中的行发给第一个 `shard`
      - 余数在 `[9,19)` 中的行发给第二个 `shard`

- `shard` 表达式： 
  - 常量和表列组成的任何返回整数表达式。
    - 例如：
      - 使用表达式 'rand()' 来随机分配数据
      - 使用 'UserID' 来按用户 ID 的余数分布（相同用户的数据将分配到单个`shard`上，这可降低带有用户信息的 IN 和 JOIN 的语句运行的复杂度）。
    - 如果该列数据分布不够均匀，可以将其包装在散列函数中：`intHash64(UserID)`

- 余数分配规则局限性：
  - 它适用于中型和大型数据（数十台服务器）的场景，但不适用于巨量数据（数百台或更多服务器）的场景。
    - 巨量数据的场景，应根据业务特性需求考虑 `shard` 方案，而不是直接用分布式表的多`shard`。

---

### 分布式 `SELECT`

- 被发送到所有`shard`，并且无论数据在 `shard` 中如何分布（即使数据完全随机分布）都可正常工作。

- 添加新 `shard` 时，不必将旧数据传输到该`shard`。
  - 你可以给新 `shard` 分配比例大点的权重重然后写新数据 
  - 数据可能会稍分布不均，但查询会正确高效地运行。

---

### `Shard` 方案

下面的情况，你需要关注 `shard` 方案：

- 使用需要特定 `key` 连接数据（ `IN` 或 `JOIN` ）的查询。
  - 如果数据是用该键进行 `shard` ，则应使用本地 `IN` 或 `JOIN` 而不是 `GLOBAL IN` 或 `GLOBAL JOIN`，这样效率更高。
  
- 使用大量服务器（上百或更多），但有大量小查询（个别客户端的查询, `website`, 广告商或合作伙伴）。
  - 为了使小查询不影响整个集群，让单个客户端的数据处于单个 `shard` 上是有意义的。
  - 或者，正如我们在 `Yandex.Metrica` 中所做的那样，你可以配置两级`shard` ：
    - 将整个集群划分为`"layers"`，一个层可以包含多个`shard`。
    - 单个 `client` 的数据位于 `single layer` 上，根据需要将 `shard` 添加到 `layer` 中，`layer` 中的数据随机分布。
    - 然后给每 `layer` 创建分布式表，再创建一个全局的分布式表用于全局的查询。

---

### Insert Data

- 数据是异步写入的。
  - 对于分布式表的 `INSERT`，数据块只写本地文件系统。
  - 之后会尽快地在后台发送到远程服务器。
  - 你可以通过查看表目录中的文件列表（等待发送的数据）来检查数据是否成功发送：`/var/lib/clickhouse/data/database/table/`
  - 如果在 INSERT 到分布式表时服务器节点丢失或重启（如，设备故障），则插入的数据可能会丢失。
  - 如果在表目录中检测到损坏的数据 `shard` ，则会将其转移到 `broken/` 子目录，并不再使用。

---

### max_parallel_replicas

启用 `max_parallel_replicas` 选项后，会在分表的所有副本上并行查询处理。

更多信息，请参阅 [max_parallel_replicas](../../operations/settings/settings.md) 设置部分。

