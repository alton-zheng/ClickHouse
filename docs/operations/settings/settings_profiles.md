# Settings profiles

`settings profile` 是按相同名称分组的设置的集合。每个 `ClickHouse` 用户都有一个 `profile`。要应用 `profile` 中的所有设置，请设置 `profile` 设置。

例：

安装`web` `profile`。

```
SET profile = 'web'
```

`settings profiles` 在用户配置文件中声明。通常在`users.xml`中: 

```xml
<!-- Settings profiles -->
<profiles>
    <!-- Default settings -->
    <default>
        <!-- The maximum number of threads when running a single query. -->
        <max_threads>8</max_threads>
    </default>

    <!-- Settings for quries from the user interface -->
    <web>
        <max_rows_to_read>1000000000</max_rows_to_read>
        <max_bytes_to_read>100000000000</max_bytes_to_read>

        <max_rows_to_group_by>1000000</max_rows_to_group_by>
        <group_by_overflow_mode>any</group_by_overflow_mode>

        <max_rows_to_sort>1000000</max_rows_to_sort>
        <max_bytes_to_sort>1000000000</max_bytes_to_sort>

        <max_result_rows>100000</max_result_rows>
        <max_result_bytes>100000000</max_result_bytes>
        <result_overflow_mode>break</result_overflow_mode>

        <max_execution_time>600</max_execution_time>
        <min_execution_speed>1000000</min_execution_speed>
        <timeout_before_checking_execution_speed>15</timeout_before_checking_execution_speed>

        <max_columns_to_read>25</max_columns_to_read>
        <max_temporary_columns>100</max_temporary_columns>
        <max_temporary_non_const_columns>50</max_temporary_non_const_columns>

        <max_subquery_depth>2</max_subquery_depth>
        <max_pipeline_depth>25</max_pipeline_depth>
        <max_ast_depth>50</max_ast_depth>
        <max_ast_elements>100</max_ast_elements>

        <readonly>1</readonly>
    </web>
</profiles>
```

- 该示例指定了两个 `profile`：`default`和`web`。
  - 该 `default` 配置文件有一个特殊目的：它必须始终存在，并在启动服务器时应用。
  - 换句话说，`default` 配置文件包含默认设置。
  - 该`web`配置文件是常规配置文件，可以使用`SET`查询或 `HTTP` 查询中的 URL 参数进行设置。

设置配置文件可以相互继承。要使用继承，请`profile`在配置文件中列出的其他设置之前指示该设置。