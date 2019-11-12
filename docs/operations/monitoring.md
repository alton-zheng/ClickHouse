# Monitoring

可以监视：

- `hardware resources` 的使用情况。
- `ClickHouse` 服务器指标。

## Resource Utilization

ClickHouse 本身不会监视 `hardware resources` 的状态。

强烈建议为以下内容设置监视：

- 处理器上的 `Load` 和 `temperature`。

可以使用[`dmesg`](https://en.wikipedia.org/wiki/Dmesg)，[`turbostat`](https://www.linux.org/docs/man8/turbostat.html)或其他工具。

- 利用 `storage system`，`RAM` 和 `network`。

## ClickHouse Server metrics

ClickHouse 服务器具有用于自我状态监视的嵌入式工具。

要跟踪服务器事件，请使用服务器日志。请参阅配置文件的[`记录器`](server_configuration_parameters.md)部分。

ClickHouse 收集：

- 服务器如何使用计算资源的不同指标。
- 有关查询处理的常见统计信息。

您可以在下列[system_tables.md]中找到指标信息：
- `system.metrics`
- `system.events`
- `system.asynchronous_metrics`

- 您可以将 ClickHouse 配置为将指标导出到[Graphite](https://github.com/graphite-project):
  - 请参见 ClickHouse 服务器配置文件中的[Graphite 部分](server_configuration_parameters.md)。
  - 应遵循其官方[指南](https://graphite.readthedocs.io/en/latest/install.html)设置 `Graphite`, 才能正常导出。

- 可以通过 HTTP API 监视服务器可用性。将`HTTP GET`请求发送到`/`。如果服务器可用，它将以响应`200 OK`。

监视集群配置中的`Server`，应设置[max_replica_delay_for_distributed_queries](settings.md)参数并使用 HTTP 资源`/replicas-delay`。
- 如果副本可用且不延迟在其他副本之后，则`/replicas-delay`返回一个请求`200 OK`。
- 如果副本被延迟，它将返回有关间隔的信息。
