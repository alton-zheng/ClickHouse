# Configuration Files

ClickHouse 支持多文件配置管理。主服务器配置文件是`/etc/clickhouse-server/config.xml`（服务器级别，在`session` 和 `query` 级别无法更改）。其他配置文件必须在`/etc/clickhouse-server/config.d`目录中。

- Note
  - 所有配置文件均应为 `XML` 格式。而且，它们通常应具有相同的根元素`<yandex>`。

- 主配置文件中指定的某些设置可以在其他配置文件中覆盖。`replace`或`remove`属性可以用于这些配置文件中的元素来指定。
  - 两者均未指定，它将以递归方式合并元素的内容，从而替换重复子元素的值。
  - 如果`replace`指定，则将整个元素替换为指定的元素。
  - 如果`remove`指定，则删除该元素。

该配置还可以定义`substitutions`。如果元素具有`incl`属性，则文件中的相应替换将用作值。
- 默认情况下，带替换文件的路径为`/etc/metrika.xml`。
- 可以在服务器配置的[include_from](settings.md)元素中进行更改。
- 替换值 `/yandex/substitution_name` 在此文件的元素中指定。如果指定的替代 `incl` 不存在，它将记录在日志中。
- 为防止 `ClickHouse` 记录缺少的替代项，请指定`optional="true"`属性（例如[宏的](https://clickhouse.yandex/docs/en/operations/server_settings/settings/)设置）。

也可以从 `ZooKeeper` 中进行替换。为此，请指定属性`from_zk = "/path/to/node"`。元素值替换为`/path/to/node` ZooKeeper 中节点的内容。您还可以将整个 `XML` 子树放在 `ZooKeeper` 节点上，并将其完全插入到 source 元素中。

该 `config.xml` 文件可以使用用户设置，配置文件和配额来指定单独的配置。此配置的相对路径在'users_config'元素中设置。默认情况下为`users.xml`。如果 `users_config` 被省略，则直接在 `config.xml` 中指定用户设置，配置文件和配额。

另外，`users_config` 可能会覆盖来自 `users_config.d` 目录(例如，users.d)和替换。例如，你可以有单独的配置文件为每个用户这样:

```bash
$ cat /etc/clickhouse-server/users.d/alice.xml
```

```
<yandex>
    <users>
      <alice>
          <profile>analytics</profile>
            <networks>
                  <ip>::/0</ip>
            </networks>
          <password_sha256_hex>...</password_sha256_hex>
          <quota>analytics</quota>
      </alice>
    </users>
</yandex>
```

对于每个配置文件，服务器在启动时也会生成`file-preprocessed.xml`文件。这些文件包含所有已完成的替换和覆盖，它们是用来提供信息的。如果配置文件中使用了`ZooKeeper`替换，但是在服务器启动时`ZooKeeper`不可用，服务器将从预处理文件加载配置。

服务器跟踪配置文件中的更改，以及执行替换和覆盖时使用的文件和ZooKeeper节点，并动态地为用户和集群重新加载设置。这意味着您可以修改集群、用户及其设置，而无需重新启动服务器。