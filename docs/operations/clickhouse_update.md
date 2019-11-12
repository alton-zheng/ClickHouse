# ClickHouse Update

如果 ClickHouse 是从 deb 软件包安装的，请在服务器上执行以下命令：

```
$ sudo apt-get update
$ sudo apt-get install clickhouse-client clickhouse-server
$ sudo service clickhouse-server restart
```

如果您使用建议的 deb 软件包以外的其他方式安装 ClickHouse，请使用适当的更新方法。

ClickHouse 不支持分布式更新。
- 该操作应在每个单独的服务器上连续执行。
- 不要同时更新群集上的所有服务器，否则群集将在一段时间内不可用。
