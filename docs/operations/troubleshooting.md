# 故障排除

## 安装

### 您无法使用 apt-get 从 ClickHouse 存储库中获取 Deb 程序包

- 检查防火墙设置。
- 如果由于某种原因无法访问存储库，请按照[入门](https://clickhouse.yandex/docs/en/getting_started/)文章中的说明下载软件包，然后使用`sudo dpkg -i <packages>`命令手动安装它们。您还需要`tzdata`包装。

## Connection to the Server

可能的问题：

- 服务器未运行。
- 意外或错误的配置参数。

### 服务器没有运行

**检查服务器是否为 running** 状态

命令：

```bash
$ sudo service clickhouse-server status
```

如果服务器未运行，请使用以下命令启动它：

```bash
$ sudo service clickhouse-server start
```

**检查日志**

默认情况下，`clickhouse-server` 主要日志在 `/var/log/clickhouse-server/clickhouse-server.log`

如果服务器成功启动，则应该看到以下字符串：

- `<Information> Application: starting up.` —服务器已启动。
- `<Information> Application: Ready for connections.` —服务器正在运行并且已准备好进行连接。

如果`clickhouse-server`启动失败并出现配置错误，则应该看到`<Error>`带有错误描述的字符串。例如：

```log
2019.01.11 15:23:25.549505 [ 45 ] {} <Error> ExternalDictionaries: Failed reloading 'event2id' external dictionary: Poco::Exception. Code: 1000, e.code() = 111, e.displayText() = Connection refused, e.what() = Connection refused
```

如果在文件末尾没有看到错误，请从字符开始浏览整个文件：

```log
<Information> Application: starting up.
```

如果尝试`clickhouse-server`在服务器上启动的第二个实例，则会看到以下日志：

```log
2019.01.11 15:25:11.151730 [ 1 ] {} <Information> : Starting ClickHouse 19.1.0 with revision 54413
2019.01.11 15:25:11.154578 [ 1 ] {} <Information> Application: starting up
2019.01.11 15:25:11.156361 [ 1 ] {} <Information> StatusFile: Status file ./status already exists - unclean restart. Contents:
PID: 8510
Started at: 2019-01-11 15:24:23
Revision: 54413

2019.01.11 15:25:11.156673 [ 1 ] {} <Error> Application: DB::Exception: Cannot lock file ./status. Another server instance in same directory is already running.
2019.01.11 15:25:11.156682 [ 1 ] {} <Information> Application: shutting down
2019.01.11 15:25:11.156686 [ 1 ] {} <Debug> Application: Uninitializing subsystem: Logging Subsystem
2019.01.11 15:25:11.156716 [ 2 ] {} <Information> BaseDaemon: Stop SignalListener thread
```

**请参阅 system.d 日志**

如果在 `clickhouse-server` 日志中找不到任何有用的信息或没有任何日志，则可以 `system.d` 使用以下命令查看日志：

```log
$ sudo journalctl -u clickhouse-server
```

**以交互方式启动 `Clickhouse-server`**

```bash
$ sudo -u clickhouse /usr/bin/clickhouse-server --config-file /etc/clickhouse-server/config.xml
```

此命令使用自动启动脚本的标准参数将服务器作为交互式应用程序启动。在此模式下`clickhouse-server`，将在控制台中打印所有事件消息。

### Configuration Parameters

`Check`：

- Docker 设置。

  如果您在 IPv6 网络中的 Docker 中运行 ClickHouse，请确保`network=host`已设置。

- 端点设置。

  检查 `listen_host` 和 `tcp_port` 设置。

  `ClickHouse` 服务器仅在默认情况下接受 `localhost` 连接。

- HTTP 协议设置。

  检查 HTTP API 的协议设置。

- 安全连接设置。

  `Check`：

  - `tcp_port_secure` 设置
  - `SSL sertificates` 设置

  连接时请使用适当的参数。例如，将`port_secure`参数 和 `clickhouse_client` 一起使用。

- 用户设置。

  您可能使用了错误的 `user name` 或 `password`。

## Query Processing

`ClickHouse` 无法处理查询，则会向客户端发送错误说明:

```log
$ curl 'http://localhost:8123/' --data-binary "SELECT a"
Code: 47, e.displayText() = DB::Exception: Unknown identifier: a. Note that there are no tables (FROM clause) in your query, context: required_names: 'a' source_tables: table_aliases: private_aliases: column_aliases: public_columns: 'a' masked_columns: array_join_columns: source_columns: , e.what() = DB::Exception
```

如设置了`clickhouse-client`与`stack-trace`参数，服务器会返回一个错误的堆栈跟踪说明。

您可能会看到一条有关断开连接的消息。
  - 在这种情况下，您可以重复查询。
  - 如果每次执行查询时连接中断，请检查服务器日志中是否有错误。

## Efficiency of Query Processing

如果您发现 ClickHouse 工作太慢，则需要分析服务器资源和网络上的负载以进行查询。

您可以使用 `clickhouse-benchmark` 实用工具来配置查询。它显示每秒处理的查询数，每秒处理的行数以及查询处理时间的百分位数。
