# 服务器设定

## builtin_dictionaries_reload_interval [¶](https://clickhouse.yandex/docs/en/operations/server_settings/settings/#builtin-dictionaries-reload-interval "永久链接")

重新加载内置词典之前的时间间隔（以秒为单位）。

ClickHouse 每隔 x 秒重新加载内置词典。这样就可以在不重新启动服务器的情况下“即时”编辑字典。

默认值：3600

**例**

<builtin_dictionaries_reload_interval> 3600 </ builtin_dictionaries_reload_interval>

## 压缩[¶](https://clickhouse.yandex/docs/en/operations/server_settings/settings/#compression "永久链接")

数据压缩设置。

警告

如果您刚刚开始使用 ClickHouse，请不要使用它。

配置如下所示：

<compression> 
    <case> 
      <parameters /> 
    </ case>
    ...
</ compression>

您可以配置多个部分`<case>`。

封锁栏位`<case>`：

- `min_part_size` –表格部件的最小尺寸。
- `min_part_size_ratio` –表格部件的最小尺寸与表格整体尺寸之比。
- `method`–压缩方法。可接受的值：`lz4`或`zstd`（实验性）。

ClickHouse 检查`min_part_size`并`min_part_size_ratio`处理`case`符合这些条件的块。如果没有`<case>`匹配项，ClickHouse 将应用`lz4`压缩算法。

**例**

<compression incl = “ clickhouse_compression” \>
<case>
<min_part_size> 10000000000 </ min_part_size>
<min_part_size_ratio> 0.01 </ min_part_size_ratio>
<method> zstd </ method>
</ case>
</ compression>

## DEFAULT_DATABASE [¶](https://clickhouse.yandex/docs/en/operations/server_settings/settings/#default-database "永久链接")

默认数据库。

要获取数据库列表，请使用[SHOW DATABASES](https://clickhouse.yandex/docs/en/query_language/show/#show-databases)查询。

**例**

<default_database>默认</ default_database>

## DEFAULT_PROFILE [¶](https://clickhouse.yandex/docs/en/operations/server_settings/settings/#default-profile "永久链接")

默认设置配置文件。

设置配置文件位于参数中指定的文件中`user_config`。

**例**

<default_profile>默认</ default_profile>

## dictionaries_config [¶](https://clickhouse.yandex/docs/en/operations/server_settings/settings/#server_settings-dictionaries_config "永久链接")

外部词典的配置文件的路径。

路径：

- 指定绝对路径或相对于服务器配置文件的路径。
- 该路径可以包含通配符\*和？。

另请参阅“ [外部词典](https://clickhouse.yandex/docs/en/query_language/dicts/external_dicts/) ”。

**例**

<dictionaries_config> \* \_dictionary.xml </ dictionaries_config>

## dictionaries_lazy_load [¶](https://clickhouse.yandex/docs/en/operations/server_settings/settings/#server_settings-dictionaries_lazy_load "永久链接")

延迟加载字典。

如果为`true`，则每个字典都在首次使用时创建。如果字典创建失败，则正在使用字典的函数将引发异常。

如果为`false`，则在服务器启动时将创建所有词典，如果有错误，则服务器将关闭。

默认值为`true`。

**例**

<dictionaries_lazy_load>是</ dictionaries_lazy_load>

## format_schema_path [¶](https://clickhouse.yandex/docs/en/operations/server_settings/settings/#server_settings-format_schema_path "永久链接")

包含输入数据方案（例如[CapnProto](https://clickhouse.yandex/docs/en/interfaces/formats/#capnproto)格式的方案）的目录路径。

**例**

<！-包含用于各种输入格式的模式文件的目录。-\>
<format_schema_path> format_schemas / </ format_schema_path>

## 石墨[¶](https://clickhouse.yandex/docs/en/operations/server_settings/settings/#server_settings-graphite "永久链接")

发送数据到[Graphite](https://github.com/graphite-project)。

设定：

- 主机– Graphite 服务器。
- port – Graphite 服务器上的端口。
- interval –发送间隔，以秒为单位。
- 超时–发送数据的超时时间（以秒为单位）。
- root_path –密钥的前缀。
- 指标–从[system.metrics](https://clickhouse.yandex/docs/en/operations/system_tables/#system_tables-metrics)表发送数据。
- events –从[system.events](https://clickhouse.yandex/docs/en/operations/system_tables/#system_tables-events)表发送在该时间段内累积的增量数据。
- [events_cumulative](https://clickhouse.yandex/docs/en/operations/system_tables/#system_tables-events) –从[system.events](https://clickhouse.yandex/docs/en/operations/system_tables/#system_tables-events)表发送累积数据。
- 异步指标–从[system.asynchronous_metrics](https://clickhouse.yandex/docs/en/operations/system_tables/#system_tables-asynchronous_metrics)表发送数据。

您可以配置多个`<graphite>`子句。例如，您可以使用它以不同的时间间隔发送不同的数据。

**例**

<graphite> 
    <host> localhost </ host> 
    <port> 42000 </ port> 
    <timeout> 0.1 </ timeout> 
    <interval> 60 </ interval> 
    <root_path>一个 _分钟</ root_path> 
    <metrics> true </ metrics > 
    <events> true </ events> 
    <events_cumulative> false </ events_cumulative> 
    <asynchronous_metrics> true </ asynchronous_metrics> 
</ graphite>

## 石墨[辊¶](https://clickhouse.yandex/docs/en/operations/server_settings/settings/#server_settings-graphite_rollup "永久链接")

稀疏石墨数据的设置。

有关更多详细信息，请参见[GraphiteMergeTree](https://clickhouse.yandex/docs/en/operations/table_engines/graphitemergetree/)。

**例**

<graphite_rollup_example>
<default>
<function> max </ function>
<retention>
<age> 0 </ age>
<precision> 60 </ precision>
</ retention>
<retention>
<age> 3600 </ age>
<precision > 300 </ precision>
</ retention>
<retention>
<age> 86400 </ age>
<precision> 3600 </ precision>
</ retention>
</ default>
</ graphite_rollup_example>

## HTTP_PORT / HTTPS_PORT [¶](https://clickhouse.yandex/docs/en/operations/server_settings/settings/#http-port-https-port "永久链接")

通过 HTTP 连接到服务器的端口。

如果`https_port`指定，则必须配置[openSSL](https://clickhouse.yandex/docs/en/operations/server_settings/settings/#server_settings-openssl)。

如果`http_port`指定，则即使已设置 openSSL 配置也将被忽略。

**例**

<https> 0000 </ https>

## http_server_default_response [¶](https://clickhouse.yandex/docs/en/operations/server_settings/settings/#http-server-default-response "永久链接")

访问 ClickHouse HTTP 服务器时默认显示的页面。

**例**

`https://tabix.io/`访问时打开`http://localhost: http_port`。

<http_server_default_response>
<！\[CDATA \[<html ng-app =“ SMI2”> <head> <base href =“ http://ui.tabix.io/”> </ head> <body> <div ui-view =“” class =“ content-ui”> </ div> <script src =“ http://loader.tabix.io/master.js”> </ script> </ body> </ html>\]\]>
</ http_server_default_response>

## include_from [¶](https://clickhouse.yandex/docs/en/operations/server_settings/settings/#server_settings-include_from "永久链接")

带替换文件的路径。

有关更多信息，请参见“ [配置文件](https://clickhouse.yandex/docs/en/operations/configuration_files/#configuration_files) ”部分。

**例**

<include_from> /etc/metrica.xml </ include_from>

## interserver_http_port [¶](https://clickhouse.yandex/docs/en/operations/server_settings/settings/#interserver-http-port "永久链接")

在 ClickHouse 服务器之间交换数据的端口。

**例**

<interserver_http_port> 9009 </ interserver_http_port>

## interserver_http_host [¶](https://clickhouse.yandex/docs/en/operations/server_settings/settings/#interserver-http-host "永久链接")

其他服务器可以用来访问该服务器的主机名。

如果省略，则其定义与`hostname-f`命令相同。

对于脱离特定的网络接口很有用。

**例**

<interserver_http_host> example.yandex.ru </ interserver_http_host>

## interserver_http_credentials [¶](https://clickhouse.yandex/docs/en/operations/server_settings/settings/#server-settings-interserver_http_credentials "永久链接")

在使用 Replicated \*引擎进行[复制](https://clickhouse.yandex/docs/en/operations/table_engines/replication/)期间进行身份验证的用户名和密码。这些凭据仅用于副本之间的通信，与 ClickHouse 客户端的凭据无关。服务器正在检查这些凭据以连接副本，并在连接到其他副本时使用相同的凭据。因此，对于群集中的所有副本，应将这些凭据设置为相同。默认情况下，不使用身份验证。

本节包含以下参数：

- `user` \- 用户名。
- `password` —密码。

**例**

<interserver_http_credentials>
<user>管理员</ user>
<password> 222 </ password>
</ interserver_http_credentials>

## keep_alive_timeout [¶](https://clickhouse.yandex/docs/en/operations/server_settings/settings/#keep-alive-timeout "永久链接")

ClickHouse 在关闭连接之前等待传入请求的秒数。默认为 3 秒。

**例**

<keep_alive_timeout> 3 </ keep_alive_timeout>

## listen_host [¶](https://clickhouse.yandex/docs/en/operations/server_settings/settings/#server_settings-listen_host "永久链接")

对请求可以来自的主机的限制。如果要服务器回答所有这些问题，请指定`::`。

例子：

<listen_host> :: 1 </ listen_host>
<listen_host> 127.0.0.1 </ listen_host>

## 记录器[¶](https://clickhouse.yandex/docs/en/operations/server_settings/settings/#server_settings-logger "永久链接")

记录设置。

按键：

- 级别–日志记录级别。可接受的值：`trace`，`debug`，`information`，`warning`，`error`。
- log –日志文件。包含根据的所有条目`level`。
- errorlog –错误日志文件。
- 大小–文件大小。适用于`log`和`errorlog`。文件到达后`size`，ClickHouse 对其进行存档并重命名，并在其位置创建一个新的日志文件。
- count – ClickHouse 存储的已归档日志文件的数量。

**例**

<logger> 
    <level>跟踪</ level> 
    <log> /var/log/clickhouse-server/clickhouse-server.log </ log> 
    <errorlog> /var/log/clickhouse-server/clickhouse-server.err。日志</ errorlog> 
    <size> 1000M </ size> 
    <count> 10 </ count> 
</ logger>

还支持写入系统日志。配置示例：

<logger> 
    <use_syslog> 1 </ use_syslog> 
    <syslog> 
        <address> syslog.remote：10514 </ address> 
        <hostname> myhost.local </ hostname> 
        <facility> LOG_LOCAL6 </ facility> 
        <format> syslog </格式\> 
    </ syslog> 
</ logger>

按键：

- use_syslog —如果要写入系统日志，则为必需设置。
- address — syslogd 的主机\[：port\]。如果省略，则使用本地守护程序。
- 主机名-可选。从中发送日志的主机的名称。
- 设施- [系统日志设施关键词](https://en.wikipedia.org/wiki/Syslog#Facility)以大写字母与“LOG\_”前缀：（ ，`LOG_USER`，`LOG_DAEMON`，`LOG_LOCAL3`等）。默认值：`LOG_USER`如果`address`指定，`LOG_DAEMON otherwise.`
- 格式–邮件格式。可能的值：`bsd`和`syslog.`

## 宏[¶](https://clickhouse.yandex/docs/en/operations/server_settings/settings/#macros "永久链接")

复制表的参数替换。

如果不使用复制表，则可以省略。

有关更多信息，请参见“ [创建复制表](https://clickhouse.yandex/docs/en/operations/table_engines/replication/) ”部分。

**例**

<macros incl = “ macros” 可选= “” true“ />

## mark_cache_size [¶](https://clickhouse.yandex/docs/en/operations/server_settings/settings/#server-mark-cache-size "永久链接")

[MergeTree](https://clickhouse.yandex/docs/en/operations/table_engines/mergetree/)系列的表引擎使用的标记的高速缓存的近似大小（以字节为单位）。

服务器共享缓存，并根据需要分配内存。缓存大小必须至少为 5368709120。

警告

[mark_cache_min_lifetime](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-mark_cache_min_lifetime)设置可能会超出此参数。

**例**

<mark_cache_size> 5368709120 </ mark_cache_size>

## max_concurrent_queries [¶](https://clickhouse.yandex/docs/en/operations/server_settings/settings/#max-concurrent-queries "永久链接")

同时处理的最大请求数。

**例**

<max_concurrent_queries> 100 </ max_concurrent_queries>

## MAX_CONNECTIONS [¶](https://clickhouse.yandex/docs/en/operations/server_settings/settings/#max-connections "永久链接")

最大入站连接数。

**例**

<max_connections> 4096 </ max_connections>

## max_open_files [¶](https://clickhouse.yandex/docs/en/operations/server_settings/settings/#max-open-files "永久链接")

最大打开文件数。

默认情况下：`maximum`。

我们建议在 Mac OS X 中使用此选项，因为该`getrlimit()`函数返回的值不正确。

**例**

<max_open_files> 262144 </ max_open_files>

## max_table_size_to_drop [¶](https://clickhouse.yandex/docs/en/operations/server_settings/settings/#max-table-size-to-drop "永久链接")

删除表的限制。

如果[MergeTree](https://clickhouse.yandex/docs/en/operations/table_engines/mergetree/)表的大小超过`max_table_size_to_drop`（以字节为单位），则无法使用 DROP 查询将其删除。

如果仍然需要删除表而不重新启动 ClickHouse 服务器，请创建`<clickhouse-path>/flags/force_drop_table`文件并运行 DROP 查询。

默认值：50 GB。

值 0 表示您可以无限制地删除所有表。

**例**

<max_table_size_to_drop> 0 </ max_table_size_to_drop>

## merge_tree [¶](https://clickhouse.yandex/docs/en/operations/server_settings/settings/#server_settings-merge_tree "永久链接")

对[MergeTree 中的](https://clickhouse.yandex/docs/en/operations/table_engines/mergetree/)表进行[微调](https://clickhouse.yandex/docs/en/operations/table_engines/mergetree/)。

有关更多信息，请参见 MergeTreeSettings.h 头文件。

**例**

<merge_tree>
<max_suspicious_broken_parts> 5 </ max_suspicious_broken_parts>
</ merge_tree>

## OpenSSL 的[¶](https://clickhouse.yandex/docs/en/operations/server_settings/settings/#server_settings-openssl "永久链接")

SSL 客户端/服务器配置。

该`libpoco`库提供了对 SSL 的支持。该接口在文件[SSLManager.h](https://github.com/ClickHouse-Extras/poco/blob/master/NetSSL_OpenSSL/include/Poco/Net/SSLManager.h)中进行了描述

服务器/客户端设置的键：

- privateKeyFile –具有 PEM 证书的秘密密钥的文件的路径。该文件可能同时包含密钥和证书。
- certificateFile – PEM 格式的客户端/服务器证书文件的路径。如果`privateKeyFile`包含证书，则可以忽略它。
- caConfig –包含受信任的根证书的文件或目录的路径。
- VerificationMode –检查节点证书的方法。详细信息在[Context](https://github.com/ClickHouse-Extras/poco/blob/master/NetSSL_OpenSSL/include/Poco/Net/Context.h)类的描述中。可能的值：`none`，`relaxed`，`strict`，`once`。
- verifyDepth –验证链的最大长度。如果证书链长度超过设置的值，验证将失败。
- loadDefaultCAFile –指示将使用 OpenSSL 的内置 CA 证书。可接受的值：`true`，`false`。|
- cipherList –支持的 OpenSSL 加密。例如：`ALL:!ADH:!LOW:!EXP:!MD5:@STRENGTH`。
- cacheSessions –启用或禁用缓存会话。必须与结合使用`sessionIdContext`。可接受的值：`true`，`false`。
- sessionIdContext –服务器附加到每个生成的标识符的唯一随机字符集。字符串的长度不能超过`SSL_MAX_SSL_SESSION_ID_LENGTH`。始终建议使用此参数，因为如果服务器缓存会话和客户端请求缓存，它将帮助避免问题。默认值：`${application.name}`。
- sessionCacheSize –服务器缓存的最大会话数。预设值：1024 \* 20。0 –无限会话。
- sessionTimeout –在服务器上缓存会话的时间。
- extendedVerification –会话结束后自动扩展证书的验证。可接受的值：`true`，`false`。
- requireTLSv1 –需要 TLSv1 连接。可接受的值：`true`，`false`。
- requireTLSv1_1 –需要 TLSv1.1 连接。可接受的值：`true`，`false`。
- requireTLSv1 –需要 TLSv1.2 连接。可接受的值：`true`，`false`。
- fips –激活 OpenSSL FIPS 模式。如果库的 OpenSSL 版本支持 FIPS，则支持。
- privateKeyPassphraseHandler –类（PrivateKeyPassphraseHandler 子类），用于请求用于访问私钥的密码短语。例如：`<privateKeyPassphraseHandler>`，`<name>KeyFileHandler</name>`，`<options><password>test</password></options>`，`</privateKeyPassphraseHandler>`。
- invalidCertificateHandler –用于验证无效证书的类（CertificateHandler 的子类）。例如：`<invalidCertificateHandler> <name>ConsoleCertificateHandler</name> </invalidCertificateHandler>`。
- disableProtocols –不允许使用的协议。
- preferredServerCiphers –客户端上的首选服务器密码。

**设置示例：**

<openSSL> 
    <server> 
        <！-openssl req -subj“ / CN = localhost” -new -newkey rsa：2048 -days 365 -nodes -x509 -keyout /etc/clickhouse-server/server.key -out / etc /clickhouse-server/server.crt-> 
        <certificateFile> /etc/clickhouse-server/server.crt </ certificateFile> 
        <privateKeyFile> /etc/clickhouse-server/server.key </ privateKeyFile> 
        <！-openssl dhparam -out /etc/clickhouse-server/dhparam.pem 4096-> 
        <dhParamsFile> /etc/clickhouse-server/dhparam.pem </ dhParamsFile> 
        <verificationMode>无</ verificationMode> 
        <loadDefaultCAFile> true </ loadDefaultCAFile> 
        <cacheSessions> true </ cacheSessions>
        <disableProtocols> sslv2，sslv3 </ disableProtocols> 
        <preferServerCiphers> true </ preferServerCiphers> 
    </ server> 
    <client> 
        <loadDefaultCAFile> true </ loadDefaultCAFile> 
        <cacheSessions> true </ cacheSessions> 
        <disableProtocols> sslv2，slv > 
        <preferServerCiphers> true </ preferServerCiphers> 
        <！-用于自签名：<verificationMode>无</ verificationMode>-> 
        <invalidCertificateHandler> 
            <！-用于自签名：<name> AcceptCertificateHandler </ name >-> 
            <name> RejectCertificateHandler </ name>
        </ invalidCertificateHandler> 
    </ client> 
</ openSSL>

## part_log [¶](https://clickhouse.yandex/docs/en/operations/server_settings/settings/#server_settings-part-log "永久链接")

记录与[MergeTree](https://clickhouse.yandex/docs/en/operations/table_engines/mergetree/)关联的事件。例如，添加或合并数据。您可以使用日志来模拟合并算法并比较其特征。您可以可视化合并过程。

查询记录在[system.part_log](https://clickhouse.yandex/docs/en/operations/system_tables/#system_tables-part-log)表中，而不记录在单独的文件中。您可以在`table`参数中配置此表的名称（请参见下文）。

使用以下参数来配置日志记录：

- `database` –数据库名称。
- `table` –系统表的名称。
- `partition_by`–设置[自定义分区键](https://clickhouse.yandex/docs/en/operations/table_engines/custom_partitioning_key/)。
- `flush_interval_milliseconds` –将数据从内存中的缓冲区刷新到表的时间间隔。

**例**

<part_log>
<database>系统</ database>
<table> part_log </ table>
<partition_by> toMonday（event_date）</ partition_by>
<flush_interval_milliseconds> 7500 </ flush_interval_milliseconds>
</ part_log>

## 路径[¶](https://clickhouse.yandex/docs/en/operations/server_settings/settings/#server_settings-path "永久链接")

包含数据的目录的路径。

注意

斜杠是必需的。

**例**

<path> / var / lib / clickhouse / </ path>

## query_log [¶](https://clickhouse.yandex/docs/en/operations/server_settings/settings/#server_settings-query-log "永久链接")

用于记录通过[log_queries = 1](https://clickhouse.yandex/docs/en/operations/settings/settings/)设置接收的查询的设置。

查询记录在[system.query_log](https://clickhouse.yandex/docs/en/operations/system_tables/#system_tables-query-log)表中，而不记录在单独的文件中。您可以在`table`参数中更改表的名称（请参见下文）。

使用以下参数来配置日志记录：

- `database` –数据库名称。
- `table` –查询将登录的系统表的名称。
- `partition_by`–设置系统表的[自定义分区键](https://clickhouse.yandex/docs/en/operations/table_engines/custom_partitioning_key/)。
- `flush_interval_milliseconds` –将数据从内存中的缓冲区刷新到表的时间间隔。

如果该表不存在，ClickHouse 将创建它。如果在更新 ClickHouse 服务器时查询日志的结构发生了更改，则具有旧结构的表将重命名，并自动创建一个新表。

**例**

<query_log>
<database>系统</ database>
<table> query_log </ table>
<partition_by> toMonday（event_date）</ partition_by>
<flush_interval_milliseconds> 7500 </ flush_interval_milliseconds>
</ query_log>

## query_masking_rules [¶](https://clickhouse.yandex/docs/en/operations/server_settings/settings/#query-masking-rules "永久链接")

基于正则表达式规则，将它们存储在服务器日志之前被应用到查询以及所有的日志消息，`system.query_log`，`system.text_log`，`system.processes`表，并在日志发送给客户端。这样可以防止敏感数据从 SQL 查询（例如姓名/电子邮件/个人标识符/信用卡号等）泄漏到日志。

**例**

<query_masking_rules>
<rule>
<name>隐藏 SSN </ name>
<regexp>（^ | \ D）\ d {3}-\ d {2}-\ d {4}（\$ | \ D）</ regexp >
<replace> 000-00-0000 </ replace>
</ rule>
</ query_masking_rules>

配置字段：-- `name`规则名称（可选）-- `regexp`兼容 RE2 的正则表达式（强制性）-- `replace`敏感数据的替换字符串（默认为可选-六个星号）

屏蔽规则适用于整个查询（以防止敏感数据因格式错误/不可解析的查询而泄漏）。

`system.events`表具有计数器`QueryMaskingRulesMatch`，该计数器具有与查询屏蔽规则匹配的总数。

对于分布式查询，必须分别配置每个服务器，否则传递给其他节点的子查询将被存储而不会被屏蔽。

## remote_servers [¶](https://clickhouse.yandex/docs/en/operations/server_settings/settings/#server_settings_remote_servers "永久链接")

[分布式](https://clickhouse.yandex/docs/en/operations/table_engines/distributed/)表引擎和`cluster`表功能使用的群集的配置。

**例**

<remote_servers incl = “ clickhouse_remote_servers” />

有关`incl`属性的值，请参见“ [配置文件](https://clickhouse.yandex/docs/en/operations/configuration_files/#configuration_files) ”部分。

**也可以看看**

- [skip_unavailable_shards](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-skip_unavailable_shards)

## 时区[¶](https://clickhouse.yandex/docs/en/operations/server_settings/settings/#timezone "永久链接")

服务器的时区。

指定为 UTC 时区或地理位置（例如，非洲/阿比让）的 IANA 标识符。

当 DateTime 字段输出为文本格式（打印在屏幕或文件中），以及从字符串获取 DateTime 时，时区对于在 String 和 DateTime 格式之间进行转换是必需的。此外，如果在输入参数中未接收到时区，则在使用时间和日期的函数中会使用时区。

**例**

<timezone>欧洲/莫斯科</ timezone>

## TCP_PORT [¶](https://clickhouse.yandex/docs/en/operations/server_settings/settings/#server_settings-tcp_port "永久链接")

通过 TCP 协议与客户端进行通信的端口。

**例**

<tcp_port> 9000 </ tcp_port>

## tcp_port_secure [¶](https://clickhouse.yandex/docs/en/operations/server_settings/settings/#server_settings-tcp_port_secure "永久链接")

TCP 端口，用于与客户端进行安全通信。与[OpenSSL](https://clickhouse.yandex/docs/en/operations/server_settings/settings/#server_settings-openssl)设置一起使用。

**可能的值**

正整数。

**默认值**

<tcp_port_secure> 9440 </ tcp_port_secure>

## tmp_path [¶](https://clickhouse.yandex/docs/en/operations/server_settings/settings/#tmp-path "永久链接")

用于处理大型查询的临时数据的路径。

注意

斜杠是必需的。

**例**

<tmp_path> / var / lib / clickhouse / tmp / </ tmp_path>

## uncompressed_cache_size [¶](https://clickhouse.yandex/docs/en/operations/server_settings/settings/#server-settings-uncompressed_cache_size "永久链接")

表引擎从[MergeTree](https://clickhouse.yandex/docs/en/operations/table_engines/mergetree/)使用的未压缩数据的缓存大小（以字节为单位）。

服务器有一个共享缓存。内存是按需分配的。如果启用了选项[use_uncompressed_cache，](https://clickhouse.yandex/docs/en/operations/settings/settings/#setting-use_uncompressed_cache)则使用高速缓存。

在个别情况下，未压缩的缓存对于非常短的查询是有利的。

**例**

<uncompressed_cache_size> 8589934592 </ uncompressed_cache_size>

## user_files_path [¶](https://clickhouse.yandex/docs/en/operations/server_settings/settings/#server_settings-user_files_path "永久链接")

包含用户文件的目录。在表函数[file（）中使用](https://clickhouse.yandex/docs/en/query_language/table_functions/file/)。

**例**

<user_files_path> / var / lib / clickhouse / user_files / </ user_files_path>

## users_config [¶](https://clickhouse.yandex/docs/en/operations/server_settings/settings/#users-config "永久链接")

包含以下内容的文件的路径：

- 用户配置。
- 访问权限。
- 设置配置文件。
- 配额设置。

**例**

<users_config> users.xml </ users_config>

## 动物园管理员[¶](https://clickhouse.yandex/docs/en/operations/server_settings/settings/#server-settings_zookeeper "永久链接")

包含允许 ClickHouse 与[ZooKeeper](http://zookeeper.apache.org/)群集进行交互的设置。

使用复制表时，ClickHouse 使用 ZooKeeper 来存储副本的元数据。如果不使用复制表，则可以忽略此部分参数。

本节包含以下参数：

- `node`— ZooKeeper 端点。您可以设置多个端点。

  例如：

  `xml <node index="1"> <host>example_host</host> <port>2181</port> </node>`

  该`index`属性指定尝试连接到 ZooKeeper 集群时的节点顺序。

- `session_timeout` —客户端会话的最大超时（以毫秒为单位）。

- `root`-该[Z 序节点](http://zookeeper.apache.org/doc/r3.5.5/zookeeperOver.html#Nodes+and+ephemeral+nodes)被用作根由 ClickHouse 服务器使用 znodes。可选的。
- `identity`—用户和密码，ZooKeeper 可能需要这些用户和密码才能访问请求的 znode。可选的。

**配置示例**

<zookeeper> 
    <节点\> 
        <主机> example1 </主机\> 
        <端口> 2181 </端口\> 
    </节点\> 
    <节点\> 
        <主机>示例2 </主机\> 
        <端口> 2181 </端口\> 
    </节点\> 
    < session\_timeout\_ms> 30000 </ session\_timeout\_ms> 
    <operation\_timeout\_ms> 10000 </ operation\_timeout\_ms> 
    <！-可选。Chroot后缀。应该存在。-> 
    <root> / path / to / zookeeper / node </ root> 
    <！-可选。Zookeeper摘要ACL字符串。-> 
    <身份>用户：密码<

**也可以看看**

- [复写](https://clickhouse.yandex/docs/en/operations/table_engines/replication/)
- [ZooKeeper 程序员指南](http://zookeeper.apache.org/doc/current/zookeeperProgrammers.html)

## use_minimalistic_part_header_in_zookeeper [¶](https://clickhouse.yandex/docs/en/operations/server_settings/settings/#server-settings-use_minimalistic_part_header_in_zookeeper "永久链接")

ZooKeeper 中数据部分头的存储方法。

此设置仅适用于`MergeTree`家庭。可以指定：

- 在文件的[merge_tree](https://clickhouse.yandex/docs/en/operations/server_settings/settings/#server_settings-merge_tree)部分中全局`config.xml`。

  ClickHouse 对服务器上的所有表使用该设置。您可以随时更改设置。当设置更改时，现有表将更改其行为。

- 对于每个单独的表。

  创建表时，请指定相应的[引擎设置](https://clickhouse.yandex/docs/en/operations/table_engines/mergetree/#table_engine-mergetree-creating-a-table)。即使全局设置发生更改，具有此设置的现有表的行为也不会更改。

**可能的值**

- 0-功能已关闭。
- 1-功能已打开。

如果为`use_minimalistic_part_header_in_zookeeper = 1`，则[复制的](https://clickhouse.yandex/docs/en/operations/table_engines/replication/)表使用单个紧凑地存储数据部分的标题`znode`。如果表包含许多列，则此存储方法将大大减少 Zookeeper 中存储的数据量。

注意

应用后`use_minimalistic_part_header_in_zookeeper = 1`，您无法将 ClickHouse 服务器降级到不支持此设置的版本。在群集中的服务器上升级 ClickHouse 时要小心。不要一次升级所有服务器。在测试环境中或仅在群集的几台服务器上测试 ClickHouse 的新版本更为安全。

已经使用此设置存储的数据部件头无法恢复为其以前的（非紧凑）表示形式。

**默认值：** 0

## disable_internal_dns_cache [¶](https://clickhouse.yandex/docs/en/operations/server_settings/settings/#server-settings-disable_internal_dns_cache "永久链接")

禁用内部 DNS 缓存。建议用于在基础设施频繁更改的系统（例如 Kubernetes）中运行 ClickHouse。

**默认值：** 0

## dns_cache_update_period [¶](https://clickhouse.yandex/docs/en/operations/server_settings/settings/#server-settings-dns_cache_update_period "永久链接")

ClickHouse 内部 DNS 缓存中存储的 IP 地址的更新时间（以秒为单位）。更新是在单独的系统线程中异步执行的。

**默认值**：15。