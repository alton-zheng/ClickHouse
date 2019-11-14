# User settings

`user.xml` 配置文件的 `users` 部分包含用户设置。

`users` 部分的结构:

```xml
<users>
    <!-- If user name was not specified, 'default' user is used. -->
    <user_name>
        <password></password>
        <!-- Or -->
        <password_sha256_hex></password_sha256_hex>

        <networks incl="networks" replace="replace">
        </networks>

        <profile>profile_name</profile>

        <quota>default</quota>

        <databases>
            <database_name>
                <table_name>
                    <filter>expression</filter>
                <table_name>
            </database_name>
        </databases>
    </user_name>
    <!-- Other users settings -->
</users>
```

## **user_name/password**

- 密码可以用明文或SHA256(十六进制格式)指定。
  - 若要以明文形式分配密码(不推荐)，请将其放在password元素中。
    - 例如, `<password>qwerty</password>` 。密码可以留空。
  - 要使用其 `SHA256` hash 分配密码，请将其放在 `password_sha256_hex` 元素中。
    - 例如, `<password_sha256_hex>65e84be33532fb784c48129675f9eff3a682b27168c0ea744b2cf58ee02337c5</password_sha256_hex>` 

如何从shell生成密码的例子:

`PASSWORD=$(base64 < /dev/urandom | head -c8); echo "$PASSWORD"; echo -n "$PASSWORD" | sha256sum | tr -d '-'`

结果的第一行是密码。第二行是对应的SHA256 hash。


## **user_name/networks**

用户可以从中连接到 `ClickHouse` 服务器的网络列表。

列表中的每个元素都可以有以下一种形式:

- `<ip>`: `ip address` 和 `network mask`
  - `213.180.204.3, 10.0.0.1/8, 10.0.0.1/255.255.255.0, 2a02:6b8::3, 2a02:6b8::3/64, 2a02:6b8::3/ffff:ffff:ffff:ffff::`

- `<host>`: 主机名
  - `server01.yandex.ru`
  - 要检查访问，需要执行 `DNS` 查询，并将所有返回的 `ip address` 与对等地址进行比较。

- `<host_regexp>` : 主机名的正则表达式。
  - `^server\d\d-\d\d-\d\.yandex\.ru$`
  - 要检查访问，需要对对等地址执行 [`DNS PTR`](https://en.wikipedia.org/wiki/Reverse_DNS_lookup) 查询，然后应用指定的 `regexp`。
  - 然后，对PTR查询的结果执行另一个DNS查询，并将所有接收到的地址与对等地址进行比较。
  - 我们强烈建议`regexp`以 `$` 结尾。

所有DNS请求的结果都会被缓存，直到服务器重新启动。

**Examples**

要为来自任何网络的用户开放访问，请指定:

```xml
<ip>::/0</ip>
```

**Warning**

从任何网络开放访问都是不安全的，除非你配置了防火墙或者服务器没有直接连接到互联网。

要仅从 `localhost` 开放访问，请指定:

```xml
<ip>::1</ip>
<ip>127.0.0.1</ip>
```

## **user_name/profile**
可以为用户分配设置 `profile`。设置 `profile` 在 `users.xml` 文件的单独部分中配置。有关更多信息，请参见 [settings_profiles](settings_profiles.md)。

## **user_name/quota**
`quotas`允许您跟踪或限制一段时间内的资源使用。`quotas`是在 `users.xml` 配置文件的`quotas`部分中配置的。

您可以为用户分配一个`quotas`集。有关`quotas`配置的详细描述，请参见[`quotas`](../quotas.md)。

## **user_name/databases**
这个部分，可以限制 `ClickHouse` 为当前用户的 `SELECT` 查询返回的行，从而实现基本的行级安全性。

**Example**

下面的配置强制用户 `user1` 只能看到作为 `SELECT` 查询结果的 `table1`表里的行数据，其中` id` 字段的值是 `1000`。

```xml
<user1>
    <databases>
        <database_name>
            <table1>
                <filter>id = 1000</filter>
            </table1>
        </database_name>
    </databases>
</user1>
```

- `filter` 可以是任何产生uint8类型值的表达式。
  - 它通常包含比较和逻辑运算符。
  - `database_name.table1` 表中的结果为0的结果被过滤掉，不为 `user1` 返回。
  - `filter` 与 `PREWHERE` 操作不兼容，禁用 `WHERE→PREWHERE` 优化。