# Quotas

`Quotas`(限额) 允许您在一段时间内限制资源的使用，或者简单地跟踪资源的使用。`Quotas` 是在用户配置中设置的。这通常是'users.xml'。

该系统还具有限制单个查询复杂性的功能。参见 `Restrictions on query complexity` 一节)。

- 与查询复杂度限制相比，`Quotas`:
  - 对一组可以在一段时间内运行的查询设置限制，而不是限制单个查询。
  - 说明用于分布式查询处理的所有远程服务器上的资源。

让我们看看 `users.xml` 文件定义限额的部分。

```xml
<!-- Quotas -->
<quotas>
    <!-- Quota name. -->
    <default>
        <!-- Restrictions for a time period. You can set many intervals with different restrictions. -->
        <interval>
            <!-- Length of the interval. -->
            <duration>3600</duration>

            <!-- Unlimited. Just collect data for the specified time interval. -->
            <queries>0</queries>
            <errors>0</errors>
            <result_rows>0</result_rows>
            <read_rows>0</read_rows>
            <execution_time>0</execution_time>
        </interval>
    </default>
</quotas>
```


默认情况下，`Quotas` 只跟踪每小时的资源消耗，而不限制使用。每个间隔计算的资源消耗将在每个请求之后输出到服务器日志。

```xml
<statbox>
    <!-- Restrictions for a time period. You can set many intervals with different restrictions. -->
    <interval>
        <!-- Length of the interval. -->
        <duration>3600</duration>

        <queries>1000</queries>
        <errors>100</errors>
        <result_rows>1000000000</result_rows>
        <read_rows>100000000000</read_rows>
        <execution_time>900</execution_time>
    </interval>

    <interval>
        <duration>86400</duration>

        <queries>10000</queries>
        <errors>1000</errors>
        <result_rows>5000000000</result_rows>
        <read_rows>500000000000</read_rows>
        <execution_time>7200</execution_time>
    </interval>
</statbox>
```

对于 `statbox` `Quotas` ，每小时和每24小时(`86,400 s`)设置限制。时间间隔从实现定义的固定时间点开始计算。换句话说，24小时的间隔不一定在午夜开始。

当间隔结束时，所有收集到的值都被清除。接下来的一个小时，`Quotas` 计算开始。

以下是可以限制的数量:

`queries` –请求总数。

`errors` –引发异常的查询数。

`result_rows` –作为结果给出的总行数。

`read_rows` –在所有远程服务器上，从表中读取的用于运行查询的源行总数。

`execution_time` –查询的总执行时间，以秒为单位（有效时间）。

如果超过了至少一个时间间隔，就会抛出一个异常，其中包含一个文本，说明超过了哪个时间间隔，以及新的时间间隔何时开始(何时可以再次发送查询)。

`Quotas` 可以使用 `Quotas` 密钥”功能来独立报告多个密钥的资源。下面是一个例子:

```
<!-- For the global reports designer. -->
<web_global>
    <!-- keyed – The quota_key "key" is passed in the query parameter,
            and the quota is tracked separately for each key value.
        For example, you can pass a Yandex.Metrica username as the key,
            so the quota will be counted separately for each username.
        Using keys makes sense only if quota_key is transmitted by the program, not by a user.

        You can also write <keyed_by_ip /> so the IP address is used as the quota key.
        (But keep in mind that users can change the IPv6 address fairly easily.)
    -->
    <keyed />
```

`Quotas` 是在配置的“users”部分分配给用户的。请参阅 “Access rights” 一节。

对于 `distributed query` 处理，累积的数量存储在请求服务器上。因此，如果用户转到另一个服务器，那里的 `quotas` 将“start over”(重新开始)。

当服务器重新启动时，将重置 `Quotas`。