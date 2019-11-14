# Server settings

## builtin_dictionaries_reload_interval

```
<builtin_dictionaries_reload_interval>3600</builtin_dictionaries_reload_interval>
```

- 重新加载 `内置` 词典之前的时间间隔（以秒为单位）。
  - 默认值：3600
  - 可以在不重新启动服务器的情况下编辑字典。

## compression

```
<compression incl="clickhouse_compression">
    <case>
        <min_part_size>10000000000</min_part_size>
        <min_part_size_ratio>0.01</min_part_size_ratio>
        <method>zstd</method>
    </case>
    ...
</compression>
```

- 数据压缩设置
  - 可以配置多个`<case>`
    - `min_part_size` –表 `part` 的最小尺寸， `byte` 为单位
    - `min_part_size_ratio` –表 `part` 的最小`size`与表整体`size`之比。
    - `method`–压缩方法。可接受的值：`lz4`或`zstd`（实验性）
    - 检查 `min_part_size` 和 `min_part_size_ratio`，并处理匹配这些条件的`case` 项。如果没有一个 `<case>` 匹配，默认 `lz4` 算法

## DEFAULT_DATABASE

```xml
<default_database>default</default_database>
```

## DEFAULT_PROFILE

默认的 `user_config`。

```xml
<default_profile>default</default_profile>
```

## dictionaries_config

```xml
<dictionaries_config>*_dictionary.xml</dictionaries_config>
```

- 外部词典的配置文件的路径
  - 可用绝对路径或相对路径
  - 可用通配符 `*` 或 `?`

## dictionaries_lazy_load

```xml
<dictionaries_lazy_load>true</dictionaries_lazy_load>
```

- 延迟加载字典。
  - 如果为`true`，则每个字典都在首次使用时创建。如果字典创建失败，则正在使用字典的函数将引发异常。
  - 如果为`false`，则在服务器启动时将创建所有词典，如果有错误，则服务器将关闭。
  - 默认值为`true`。

## format_schema_path

包含输入数据方案（例如[CapnProto](https://clickhouse.yandex/docs/en/interfaces/formats/#capnproto)格式的方案）的目录路径。

```xml
<!-- Directory containing schema files for various input formats. -->
  <format_schema_path>format_schemas/</format_schema_path>
```

## graphite

```xml
<graphite>
    <host>localhost</host>
    <port>42000</port>
    <timeout>0.1</timeout>
    <interval>60</interval>
    <root_path>one_min</root_path>
    <metrics>true</metrics>
    <events>true</events>
    <events_cumulative>false</events_cumulative>
    <asynchronous_metrics>true</asynchronous_metrics>
</graphite>
```

- 发送数据到[Graphite](https://github.com/graphite-project)。
  - `host`– `Graphite` 服务器主机名称或ip。
  - `port` – `Graphite` 服务器上的端口。
  - `interval` –发送间隔，以秒为单位。
  - `timeout`–发送数据的超时时间（以秒为单位）。
  - `root_path` –密钥的前缀
  - `metrics`–从[system.metrics](system_tables.md)表发送数据。
  - `events` –从[system.events](system_tables.md)表发送在该时间段内累积的增量数据。
  - `events_cumulative` –从[system.events](system_tables.md)表发送累积数据。
  - `asynchronous_metrics`–从[system.asynchronous_metrics](system_tables.md)表发送数据。
  - 您可以配置多个`<graphite>`子句。例如，可以使用它以不同的时间间隔发送不同的数据。  


## graphite_rollup

稀疏石墨数据的设置。

有关更多详细信息，请参见[GraphiteMergeTree](https://clickhouse.yandex/docs/en/operations/table_engines/graphitemergetree/)。

**例**

```xml
<graphite_rollup_example>
    <default>
        <function>max</function>
        <retention>
            <age>0</age>
            <precision>60</precision>
        </retention>
        <retention>
            <age>3600</age>
            <precision>300</precision>
        </retention>
        <retention>
            <age>86400</age>
            <precision>3600</precision>
        </retention>
    </default>
</graphite_rollup_example>
```

## OpenSSL

```xml
<openSSL>
    <server>
        <!-- openssl req -subj "/CN=localhost" -new -newkey rsa:2048 -days 365 -nodes -x509 -keyout /etc/clickhouse-server/server.key -out /etc/clickhouse-server/server.crt -->
        <certificateFile>/etc/clickhouse-server/server.crt</certificateFile>
        <privateKeyFile>/etc/clickhouse-server/server.key</privateKeyFile>
        <!-- openssl dhparam -out /etc/clickhouse-server/dhparam.pem 4096 -->
        <dhParamsFile>/etc/clickhouse-server/dhparam.pem</dhParamsFile>
        <verificationMode>none</verificationMode>
        <loadDefaultCAFile>true</loadDefaultCAFile>
        <cacheSessions>true</cacheSessions>
        <disableProtocols>sslv2,sslv3</disableProtocols>
        <preferServerCiphers>true</preferServerCiphers>
    </server>
    <client>
        <loadDefaultCAFile>true</loadDefaultCAFile>
        <cacheSessions>true</cacheSessions>
        <disableProtocols>sslv2,sslv3</disableProtocols>
        <preferServerCiphers>true</preferServerCiphers>
        <!-- Use for self-signed: <verificationMode>none</verificationMode> -->
        <invalidCertificateHandler>
            <!-- Use for self-signed: <name>AcceptCertificateHandler</name> -->
            <name>RejectCertificateHandler</name>
        </invalidCertificateHandler>
    </client>
</openSSL>
```

SSL 客户端/服务器配置。


该`libpoco`库提供了对 `SSL` 的支持。该接口在文件[SSLManager.h](https://github.com/ClickHouse-Extras/poco/blob/master/NetSSL_OpenSSL/include/Poco/Net/SSLManager.h)中进行了描述

服务器/客户端设置的 keys：

- `privateKeyFile` –具有 `PEM` 证书的秘密密钥的文件的路径。该文件可能同时包含密钥和证书。
- `certificateFile` – `PEM` 格式的客户端/服务器证书文件的路径。如果`privateKeyFile`包含证书，则可以忽略它。
- `caConfig` –包含受信任的根证书的文件或目录的路径。
- `VerificationMode` –检查节点证书的方法。详细信息在[Context](https://github.com/ClickHouse-Extras/poco/blob/master/NetSSL_OpenSSL/include/Poco/Net/Context.h)类的描述中。可能的值：`none`，`relaxed`，`strict`，`once`。
- `verifyDepth` –验证链的最大长度。如果证书链长度超过设置的值，验证将失败。
- `loadDefaultCAFile` –指示将使用 OpenSSL 的内置 CA 证书。可接受的值：`true`，`false`。|
- `cipherList` –支持的 OpenSSL 加密。例如：`ALL:!ADH:!LOW:!EXP:!MD5:@STRENGTH`。
- `cacheSessions` –启用或禁用缓存会话。必须与结合使用`sessionIdContext`。可接受的值：`true`，`false`。
- `sessionIdContext` –服务器附加到每个生成的标识符的唯一随机字符集。字符串的长度不能超过`SSL_MAX_SSL_SESSION_ID_LENGTH`。始终建议使用此参数，因为如果服务器缓存会话和客户端请求缓存，它将帮助避免问题。默认值：`${application.name}`。
- `sessionCacheSize` –服务器缓存的最大会话数。预设值：1024 \* 20。0 –无限会话。
- `sessionTimeout` –在服务器上缓存会话的时间。
- `extendedVerification` –会话结束后自动扩展证书的验证。可接受的值：`true`，`false`。
- `requireTLSv1` –需要 TLSv1 连接。可接受的值：`true`，`false`。
- `requireTLSv1_1` –需要 TLSv1.1 连接。可接受的值：`true`，`false`。
- `requireTLSv1` –需要 TLSv1.2 连接。可接受的值：`true`，`false`。
- `fips` –激活 OpenSSL FIPS 模式。如果库的 OpenSSL 版本支持 FIPS，则支持。
- `privateKeyPassphraseHandler` –类（PrivateKeyPassphraseHandler 子类），用于请求用于访问私钥的密码短语。例如：`<privateKeyPassphraseHandler>`，`<name>KeyFileHandler</name>`，`<options><password>test</password></options>`，`</privateKeyPassphraseHandler>`。
- `invalidCertificateHandler` –用于验证无效证书的类（CertificateHandler 的子类）。例如：`<invalidCertificateHandler> <name>ConsoleCertificateHandler</name> </invalidCertificateHandler>`。
- `disableProtocols` –不允许使用的协议。
- `preferredServerCiphers` –客户端上的首选服务器密码。

**设置示例：**

## HTTP_PORT / HTTPS_PORT

```xml
<https>0000</https>
```

通过 HTTP 连接到服务器的端口。

如果`https_port`指定，则必须配置 `openSSL`, 见上文
如果`http_port`指定，则即使已设置 `openSSL` 配置也将被忽略。

## http_server_default_response

```xml
<http_server_default_response>
  <![CDATA[<html ng-app="SMI2"><head><base href="http://ui.tabix.io/"></head><body><div ui-view="" class="content-ui"></div><script src="http://loader.tabix.io/master.js"></script></body></html>]]>
</http_server_default_response>
```

## include_from

```xml
<include_from>/etc/metrica.xml</include_from>
```

- 带替换文件的路径。

有关更多信息，请参见 [配置文件](configuration_files.md) 部分。

## interserver_http_port

```xml
<interserver_http_port>9009</interserver_http_port>
```

在 ClickHouse 服务器之间交换数据的端口。

## interserver_http_host

```xml
<interserver_http_host>example.yandex.ru</interserver_http_host>
```

其他服务器可用于访问此服务器的主机名。

如果省略，它的定义方式与 `hostname-f` 命令相同。

用于脱离特定的网络接口。

## interserver_http_credentials

```xml
<interserver_http_credentials>
    <user>admin</user>
    <password>222</password>
</interserver_http_credentials>
```

在使用 `Replicated*` 引擎进行[复制](../table_engines/merge-tree-family/data-replication.md)期间进行身份验证的用户名和密码。这些凭据仅用于副本之间的通信，与 ClickHouse 客户端的凭据无关。服务器正在检查这些凭据以连接副本，并在连接到其他副本时使用相同的凭据。因此，对于群集中的所有副本，应将这些凭据设置为相同。默认情况下，不使用身份验证。

本节包含以下参数：

在使用复制*引擎进行复制期间用于身份验证的用户名和密码。这些凭据仅用于副本之间的通信，与ClickHouse客户机的凭据无关。服务器正在检查这些连接副本的凭据，并在连接其他副本时使用相同的凭据。因此，应该为集群中的所有副本设置相同的凭据。默认情况下，不使用身份验证。

本节包含以下参数:

- `user` : 用户名。
- `password` : 密码。

## keep_alive_timeout

- ClickHouse 连接空闲保留的时间，超过此设置时间，将关闭连接
  - 默认为 3 秒。

```xml
<keep_alive_timeout>3</keep_alive_timeout>
```

## listen_host

```xml
<listen_host>::1</listen_host>
<listen_host>127.0.0.1</listen_host>
```

对请求可以来自的主机的限制。如果要服务器响应所有主机的请求，请指定`::`。

## Logger

```xml
<logger>
    <level>trace</level>
    <log>/var/log/clickhouse-server/clickhouse-server.log</log>
    <errorlog>/var/log/clickhouse-server/clickhouse-server.err.log</errorlog>
    <size>1000M</size>
    <count>10</count>
</logger>
```

- `Logger` 设置。
  - `level`: `logger` 级别。(`trace`，`debug`，`information`，`warning`，`error`)
  - `log`: 日志文件路径，包含所有 `logger` 级别的日志
  - `errorlog`:  日志文件路径，包含所有 `logger` 级别的日志
  - `size` : 文件大小。适用于`log`和`errorlog`。文件到达后`size`，ClickHouse 对其进行存档并重命名，并在其位置创建一个新的日志文件。
  - `count`:  存储已归档日志文件的数量。


支持写入系统日志： 
```xml
<logger>
    <use_syslog>1</use_syslog>
    <syslog>
        <address>syslog.remote:10514</address>
        <hostname>myhost.local</hostname>
        <facility>LOG_LOCAL6</facility>
        <format>syslog</format>
    </syslog>
</logger>
```


- `keys list`：
  - `use_syslog` ： 如果要写入系统日志，则为必需设置。
  - `address` ： `syslogd` 的 `host[:port]`。如果省略，则使用本地守护程序。
  - `hostname` : 可选。从中发送日志的主机的名称。
  - `facility` :  [系统日志设施关键词](https://en.wikipedia.org/wiki/Syslog#Facility)以大写字母 `LOG_` 前缀：（ `LOG_USER`，`LOG_DAEMON`，`LOG_LOCAL3`等）。如指定了 `address` 参数，默认值：`LOG_USER`，否则默认为 `LOG_DAEMON`
  - `format` : 信息格式。可能的值：`bsd` 和`syslog.`

## macros

```xml
<macros incl="macros" optional="true" />
```

`replicated tables` 的参数替换。

如果不使用`replicated table`，则可以省略。

有关更多信息，请参见 [创建复制表](../table_engines/merge-tree-family/data-replication.md) 部分。

## mark_cache_size

```xml
<mark_cache_size>5368709120</mark_cache_size>
```
`MergeTree` 系列的表引擎使用的标记的高速缓存的近似大小（以字节为单位）。

服务器共享缓存，并根据需要分配内存。缓存大小必须至少为 5368709120。

警告

`mark_cache_min_lifetime` 设置可能会超出此参数。

## max_concurrent_queries

```xml
<max_concurrent_queries>100</max_concurrent_queries>
```
同时处理的最大请求数。

## MAX_CONNECTIONS

```xml
<max_connections>4096</max_connections>
```

最大连接数。

## max_open_files

```xml
<max_open_files>262144</max_open_files>
```

- 最大打开文件数。
  - 默认情况下：`maximum`。

我们建议在 `Mac OS X` 中使用此选项，因为该`getrlimit()`函数返回的值不正确。

## max_table_size_to_drop

```xml
<max_table_size_to_drop>0</max_table_size_to_drop>
```

- 删除表的限制
  - 如果 `MergeTree` 表的大小超过`max_table_size_to_drop`（以字节为单位），则无法使用 `DROP` 指令将其删除。
    - 如果仍然需要删除表而不重新启动 ClickHouse 服务器，请创建`<clickhouse-path>/flags/force_drop_table`文件并运行 DROP 查询。

默认值：`50 GB`。

`0` 表示您可以无限制地删除所有表。

## merge_tree

```xml
<merge_tree>
    <max_suspicious_broken_parts>5</max_suspicious_broken_parts>
</merge_tree>
```

对 `MergeTree`表进行微调。

有关更多信息，请参见 `MergeTreeSettings.h` head 文件。

## part_log

```xml
<part_log>
    <database>system</database>
    <table>part_log</table>
    <partition_by>toMonday(event_date)</partition_by>
    <flush_interval_milliseconds>7500</flush_interval_milliseconds>
</part_log>
```

- 记录有关 `MergeTree` 的事件。
  - 例如，添加或合并数据。 
  - 可以使用日志来模拟合并算法并比较其特征。
  - 可视化合并过程。
  - 记录在 `system.part_log` 表中，而不记录在单独的文件中。
  - 您可以在 `table` 参数中配置此表的名称（请参见下文）。

使用以下参数来配置日志记录：

- `database` –数据库名称。
- `table` –系统表的名称。
- `partition_by`–设置 `customer partitioning key`。
- `flush_interval_milliseconds` –将数据从内存中的缓冲区刷新到表的时间间隔。

## path

```xml
<path>/var/lib/clickhouse/</path>
```

- 包含数据的目录的路径。
  - 斜杠是必需的。

## query_log

```xml
<query_log>
    <database>system</database>
    <table>query_log</table>
    <partition_by>toMonday(event_date)</partition_by>
    <flush_interval_milliseconds>7500</flush_interval_milliseconds>
</query_log>
```

- 记录设置了`log_queries=1`(非系统配置项) 查询的日志配置
  - 查询记录在[system.query_log](system_tables.md)表中，而不记录在单独的文件中。
    - 如果该表不存在，`ClickHouse` 将创建它。
    - 如果在更新 `ClickHouse` 服务器时查询日志的结构发生了更改，则具有旧结构的表将重命名，并自动创建一个新表。
  - 您可以在`table`参数中更改表的名称（请参见下文）。

使用以下参数来配置日志记录：

- `database` –数据库名称。
- `table` –查询将登录的系统表的名称。
- `partition_by`–设置系统表的 `customer partitioning key`
- `flush_interval_milliseconds` –将数据从内存中的缓冲区刷新到表的时间间隔。



## query_masking_rules(查询屏蔽规则)

```xml
<query_masking_rules>
    <rule>
        <name>hide SSN</name>
        <regexp>(^|\D)\d{3}-\d{2}-\d{4}($|\D)</regexp>
        <replace>000-00-0000</replace>
    </rule>
</query_masking_rules>
```

基于`regexps` 的规则，它将应用于查询以及在将所有日志消息存储到服务器日志系统之前的所有日志消息。`system.query_log`，`system.text_log`，`system.processes`表，并在日志中发送给客户端。这可以防止敏感数据从SQL查询(如姓名/电子邮件/个人标识符/信用卡号码等)泄露到日志。

- 配置字段：
  - `name`: 规则名称（可选）
  - `regexp`: 兼容 RE2 的正则表达式（强制性）
  - `replace`: 敏感数据的替换字符串（默认为可选-`******`）

- 屏蔽规则贯穿于整个查询
  - 以防止敏感数据因格式错误/不可解析的查询而泄漏。
  - `system.events`表具有计数器 `QueryMaskingRulesMatch`，该计数器具有与查询屏蔽规则匹配的总数。
  - 对于分布式查询，必须分别配置每个服务器，否则传递给其他节点的子查询将被存储而不会被屏蔽。

## remote_servers

```xml
<remote_servers incl="clickhouse_remote_servers" />
```
`Distributed` 表引擎和`cluster`表功能使用的群集的配置。

有关`incl`属性的值，请参见“ [配置文件](https://clickhouse.yandex/docs/en/operations/configuration_files/#configuration_files) ”部分。


## timezone
```xml
<timezone>Europe/Moscow</timezone>
```

服务器的时区。

指定为UTC时区或地理位置(例如，`Africa/Abidjan`)的IANA标识符。

- 当将DateTime字段输出为文本格式(打印在屏幕上或文件中)时，以及从字符串获取DateTime时，时区对于字符串和DateTime格式之间的转换是必要的。
- 此外，如果在输入参数中没有接收到时区，则在处理时间和日期的函数中使用时区。


## TCP_PORT

```xml
<tcp_port>9000</tcp_port>
```

通过 TCP 协议与客户端进行通信的端口。

## tcp_port_secure

```xml
<tcp_port_secure>9440</tcp_port_secure>
```

- TCP 端口，用于与客户端进行安全通信。与 `OpenSSL` 设置一起使用。
  - 正整数
  - 默认端口 `9440`

## tmp_path

```xml
<tmp_path>/var/lib/clickhouse/tmp/</tmp_path>
```

- 用于处理大型查询的临时数据的路径。
  - 斜杠是必需的。

## uncompressed_cache_size

```xml
<uncompressed_cache_size>8589934592</uncompressed_cache_size>
```

对于`MergeTree`中的`table engine`使用的未压缩数据，缓存大小(以字节为单位)。

服务器有一个共享缓存。内存按需分配。如果启用了 `use_uncompressed_cache` 选项，则使用该缓存。

未压缩的缓存对于个别情况下的非常短的查询是有利的。

## user_files_path

```xml
<user_files_path>/var/lib/clickhouse/user_files/</user_files_path>
```

包含用户文件的目录。在表函数 `file()` 中使用

## users_config

```xml
<users_config>users.xml</users_config>
```

包含以下内容的文件的路径：

- 用户配置。
- 访问权限。
- 设置配置文件。
- 配额设置。

## zookeeper

```xml
<zookeeper>
    <node>
        <host>example1</host>
        <port>2181</port>
    </node>
    <node>
        <host>example2</host>
        <port>2181</port>
    </node>
    <session_timeout_ms>30000</session_timeout_ms>
    <operation_timeout_ms>10000</operation_timeout_ms>
    <!-- Optional. Chroot suffix. Should exist. -->
    <root>/path/to/zookeeper/node</root>
    <!-- Optional. Zookeeper digest ACL string. -->
    <identity>user:password</identity>
</zookeeper>
```

- 包含允许 `ClickHouse` 与 `ZooKeeper` 群集进行交互的设置。
  - 使用复制表时，`ClickHouse` 使用 `ZooKeeper` 来存储副本的元数据。如果不使用复制表，则可以忽略此部分参数。

包含以下参数：

  - `node`— `ZooKeeper` 端点。您可以设置多个端点。
    - `xml <node index="1"> <host>example_host</host> <port>2181</port> </node>`
    - 该`index`属性指定尝试连接到 ZooKeeper 集群时的节点顺序。
    
  - `session_timeout` 
    - 客户端会话的最大超时（以毫秒为单位）。
  
  - `root`
    - 用于作为 `ClickHouse` 服务器使用的 `znodes`的根的`znode`。
    - 可选。
    
  - `identity`
    - 用户和密码，ZooKeeper 可能需要这些用户和密码才能访问请求的 znode。
    - 可选的。

## use_minimalistic_part_header_in_zookeeper

ZooKeeper 中数据 `part` `headers` 的存储方法。

此设置仅适用于`MergeTree Family`。可以指定：

- 在文件的 `merge_tree` 部分中 全局 `config.xml`。
  - ClickHouse使用服务器上所有表的设置。您可以随时更改设置。当设置更改时，现有表将更改其行为。

- 对于每个单独的表。
  - 创建表时，指定相应的引擎设置。
  - 即使全局设置更改，具有此设置的现有表的行为也不会更改。

- value: 
  - `0` : `turn off`
  - `1` : `turn on`
  - 默认值： `0`

如果 `use_minimalistic_part_header_in_zookeeper = 1`，则 `replicated tables` 使用单个 `znode` 紧凑地存储 `data part` 的`head`。如果表中包含许多列，则此存储方法将显著减少 `Zookeeper` 中存储的数据量。

- 注意
  - 应用后`use_minimalistic_part_header_in_zookeeper = 1`，您无法将 ClickHouse 服务器降级到不支持此设置的版本。
  - 已经使用此设置存储的数据部件头无法恢复为其以前的（非紧凑）表示形式.

## disable_internal_dns_cache

禁用内部DNS缓存。推荐用于在Kubernetes等基础架构频繁变化的`ClickHouse`

**默认值：** 0

## dns_cache_update_period

更新存储在 `ClickHouse` 内部`DNS`缓存中的IP地址的周期(以秒为单位)。更新是在一个单独的系统线程中异步执行的。

**默认值**：15。