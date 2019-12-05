# Access Rights(访问权限)

用户和访问权限在用户配置中设置。`users.xml`。

用户记录在该`users`部分中。这是`users.xml`文件的一部分：

```xml
<!-- Users and ACL. -->
<users>
    <!-- If the user name is not specified, the 'default' user is used. -->
    <default>
        <!-- Password could be specified in plaintext or in SHA256 (in hex format).

             If you want to specify password in plaintext (not recommended), place it in 'password' element.
             Example: <password>qwerty</password>.
             Password could be empty.

             If you want to specify SHA256, place it in 'password_sha256_hex' element.
             Example: <password_sha256_hex>65e84be33532fb784c48129675f9eff3a682b27168c0ea744b2cf58ee02337c5</password_sha256_hex>

             How to generate decent password:
             Execute: PASSWORD=$(base64 < /dev/urandom | head -c8); echo "$PASSWORD"; echo -n "$PASSWORD" | sha256sum | tr -d '-'
             In first line will be password and in second - corresponding SHA256.
        -->
        <password></password>

        <!-- A list of networks that access is allowed from.
            Each list item has one of the following forms:
            <ip> The IP address or subnet mask. For example: 198.51.100.0/24 or 2001:DB8::/32.
            <host> Host name. For example: example01. A DNS query is made for verification, and all addresses obtained are compared with the address of the customer.
            <host_regexp> Regular expression for host names. For example, ^example\d\d-\d\d-\d\.yandex\.ru$
                To check it, a DNS PTR request is made for the client's address and a regular expression is applied to the result.
                Then another DNS query is made for the result of the PTR query, and all received address are compared to the client address.
                We strongly recommend that the regex ends with \.yandex\.ru$.

            If you are installing ClickHouse yourself, specify here:
                <networks>
                        <ip>::/0</ip>
                </networks>
        -->
        <networks incl="networks" />

        <!-- Settings profile for the user. -->
        <profile>default</profile>

        <!-- Quota for the user. -->
        <quota>default</quota>
    </default>

    <!-- For requests from the Yandex.Metrica user interface via the API for data on specific counters. -->
    <web>
        <password></password>
        <networks incl="networks" />
        <profile>web</profile>
        <quota>default</quota>
        <allow_databases>
           <database>test</database>
        </allow_databases>
        <allow_dictionaries>
           <dictionary>test</dictionary>
        </allow_dictionaries>
    </web>
</users>
```

您可以看到两个用户的声明：`default`和`web`。我们分别添加了该`web`用户。

如果用户名未通过，则默认选择 `default` 用户。

如果服务器或集群的配置未指定`user`和 `password`(见 `Distributed engine`)，还可会使用 `default` 用户处理分布式查询.

用于在集群中组合的服务器之间交换信息的用户必须没有实质性的 `restrictions`(限制) 或 `quotas`——否则，分布式查询将失败。

密码以明文(不推荐)或 `SHA-256` 指定。此 `hash` 没加盐。在这方面，您不应该认为这些密码提供了防范潜在恶意攻击的安全性。相反，它们对于保护 `employees` 是必要的。

指定允许访问的网络列表。在此示例中，两个用户的网络列表是从 `/etc/metrika.xml` 包含 `networks` 替换的单独文件中加载的。这是其中的一部分：

```
<yandex>
    ...
    <networks>
        <ip>::/64</ip>
        <ip>203.0.113.0/24</ip>
        <ip>2001:DB8::/32</ip>
        ...
    </networks>
</yandex>
```

您可以直接在 `users.xml` 或 `users.d` 目录中的文件中定义此网络列表（有关更多信息，请参见“ [配置文件](configuration_files.md) ”部分）。

该配置包含注释，说明功能

在生产中使用时，仅指定`ip`元素（IP 地址及其掩码），因为使用`host`和`hoost_regexp`可能会导致额外的延迟。

接下来，指定用户设置配置文件（请参阅“ [设置配置文件](settings.md) ”部分。您可以指定默认配置文件`default'`。配置文件可以具有任何名称。您可以为不同的用户指定相同的配置文件。最重要的事情是您可以在设置配置文件为`readonly=1`，可确保具有只读访问权限，然后指定要使用的`quatas`（请参阅“ [`quatas`](https://clickhouse.yandex/docs/en/operations/quotas/#quotas) ”  部分）。您可以指定默认`quatas`：`default`。默认情况下，它在 config 中设置为仅计算资源使用情况，不包括`quatas`可以有任何名称，您可以为不同的用户指定相同的`quatas`-在这种情况下，将分别为每个用户计算资源使用量。

在可选`<allow_databases>`部分中，您还可以指定用户可以访问的数据库列表。
- 默认情况下，所有数据库都对用户可用。
- 您可以指定 `default` 数据库。
- 在这种情况下，默认情况下，用户将获得对数据库的访问权限。

在可选 `<allow_dictionaries>` 部分中，您还可以指定用户可以访问的词典列表。默认情况下，所有词典都对用户可用。

始终允许访问 `system` 数据库(因为此数据库用于处理查询)。

即使不允许访问单个数据库，用户也可以使用`SHOW`查询或系统表获取其中所有数据库和表的列表。

数据库访问与[readonly](settings.md)设置无关。

您不能授予对一个数据库的 `full access` 和一个数据库的`readonly` 权限。
