# Configuration Files

ClickHouse 支持多文件配置管理。主服务器配置文件是`/etc/clickhouse-server/config.xml`（服务器级别，在`session` 和 `query` 级别无法更改）。其他配置文件必须在`/etc/clickhouse-server/config.d`目录中。

- Note
  - 所有配置文件均应为 `XML` 格式。而且，它们通常应具有相同的根元素`<yandex>`。

- 主配置文件中指定的某些设置可以在其他配置文件中覆盖。
- 可以为这些配置文件的元素指定 `replace` 或 `remove` 属性。
  - `replace`: 它将用指定的元素替换整个元素。
  - `remove`: 删除元素。
  - 如果两者都没有指定，则递归地组合元素的内容，替换重复的子元素的值。

- 配置还可以定义“substitutions”。
  - 如果元素具有 `incl` 属性，则将使用文件中相应的替换作为值。   
  - 默认情况下，带有替换的文件的路径是 `/etc/metrika.xml`。
  - 这可以在服务器配置中的[include_from](settings.md)元素中更改。
  - 替换值在这个文件中的 `/yandex/substitution_name` 元素中指定。
    - 如果 `incl` 中指定的替换不存在，则将其记录在日志中。
  - 为了防止 `ClickHouse` 记录丢失的替换，可以指定 `optional="true"` 属性


- `Substitutions` 也可以从 `zookeeper` 进行指定。
  - 为此，指定属性  `from_zk = "/path/to/node"` 。
  - 元素值被 `ZooKeeper` 中 `/path/to/node` 节点的内容替换。
  - 您还可以将整个 `XML` 子树放在 `ZooKeeper` 节点上，它将完全插入到源元素中。

- `config.xml` 文件可以使用 `user` 、`profile`  和 `quota` 设置指定配置。
  - 这个配置的相对路径是在 `'users_config'` 元素中设置的。
  - 默认情况下，它是 `users.xml` 。
  - 如果省略 `users_config` ，则直接在 `config.xml` 中指定 `user` 、`profile`  和 `quota` 
  
- 另外，`users_config` 可能会覆盖来自 `users_config.d` 目录(例如，users.d )和 `substitution` 的文件。 
  - 例如，你可以有单独的配置文件为每个用户这样:


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

- 对于每个配置文件，服务器在启动时也会生成 `file-preprocses.xml` 文件。
  - 这些文件包含所有已完成的替换和覆盖，它们是用来提供信息的。
  - 如果配置文件中使用了 `ZooKeeper` 替换，但是在服务器启动时 `ZooKeeper` 不可用，服务器将从预处理文件加载配置。

- 服务器跟踪 `config file` 中的更改，以及执行替换和覆盖时使用的文件和 `ZooKeeper` 节点，并动态地为用户和集群重新加载设置。
  - 这意味着您可以修改 `Cluster`、`user` 及其 设置，而无需重新启动服务器(热加载)。
