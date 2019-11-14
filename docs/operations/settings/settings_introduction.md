# Settings

有多种方法可以实现下面描述的所有设置。设置是在`layers`配置的，因此每个后续层重新定义以前的设置。

- 配置设置的方法，按优先级排序:
  -  服务器配置文件 `users.xml` 中的设置。
    - 在元素 `<profiles>` 中设置。
  - `session` 设置。
    - 在交互模式下，从 `ClickHouse` 控制台客户端发送 `SET setting=value`。
    - 类似地，您可以在HTTP协议中使用`ClickHouse session`。
    - 为此，需要指定`session_id` HTTP参数。
  - `Query` 设置。
    - 在 `non-interactive` 模式下启动 `ClickHouse` 控制台客户端时，设置启动参数 `--setting=value`。
    - 当使用 `HTTP API` 时，传递CGI(公共网关接口)参数(`URL?setting_1=value&setting_2=value...`)。
  

- [permissions_for_queries](permissions_for_queries.md)
- [query_complexity](query_complexity.md)
- [settings](settings.md)
- [settings_profiles](settings_profiles.md)
- [constraints_on_settings](constraints_on_settings.md)
- [settings_users](settings_users.md)