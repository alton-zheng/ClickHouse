## GraphiteMergeTree

这个引擎是为 `thin` 和 `aggregation`  /`average`(`rollup`) [`Graphite`](https://graphite.readthedocs.io/en/latest/index.html) 数据而设计的。

它可能对那些想要使用ClickHouse作为石墨数据存储的开发人员有帮助。

- 不需要 `rollup`，可以使用任何 `table engine` 来存储石墨数据。 
- 需要 `rollup`，可以使用 `GraphiteMergeTree`。
  - 该引擎减少了存储容量，提高了从石墨查询的效率。

引擎从 `MergeTree` 继承属性。

### Creating a Table

```sql
CREATE TABLE [IF NOT EXISTS] [db.]table_name [ON CLUSTER cluster]
(
    Path String,
    Time DateTime,
    Value <Numeric_type>,
    Version <Numeric_type>
    ...
) ENGINE = GraphiteMergeTree(config_section)
[PARTITION BY expr]
[ORDER BY expr]
[SAMPLE BY expr]
[SETTINGS name=value, ...]
```

- 用于石墨数据的表格应该包含以下数据列:
  - 指标名称(石墨传感器)。数据类型: `String`
  - 度量指标的时间。数据类型: `DateTime`
  - 指标的值。数据类型:任何数字
  - 指标的版本。数据类型:任何数字
    - 如果版本相同，ClickHouse将保存最高版本或最后写入的行。其他行在数据部分合并期间被删除。

这些列的名称应该在rollup配置中设置

#### **GraphiteMergeTree参数**

- `config_section`: —配置文件中节的名称，其中是rollup set的规则。


### `Rollup` configuration

rollup的设置由服务器配置中的 `graphite_rollup` 参数定义。参数的名称可以是任意的。您可以创建多个配置并将它们用于不同的表。

`Rollup configuration` 结构:

```log
required-columns
patterns
```

#### Required Columns
- `path_column_name` : 存储指标名称(石墨传感器)的列的名称。默认值: `Path`
- `time_column_name`—存储指标时间的列的名称。默认值:`Time`
- `value_column_name`—在 `time_column_name` 中设置的时间存储指标值的列的名称。默认值:`Value`
- `version_column_name`—存储度量的版本的列的名称。默认值: `Timestamp`.

#### Patterns

- `patterns` 部分的结构:

```
pattern
    regexp
    function
pattern
    regexp
    age + precision
    ...
pattern
    regexp
    function
    age + precision
    ...
pattern
    ...
default
    function
    age + precision
    ...
```

**Attention**

- `Pattern` 必须严格排序:

  - 没有 `function` 或 `retention` 的模式
  - `function`和`retention`的模式
  - `default` 的模式
  
- 处理行时，ClickHouse检查模式部分中的规则。
  - 每个 `pattern` (包括默认模式)部分都可以包含用于聚合、保留参数或两者的 `function` 参数
  - 如果指标名称与 `regexp` 匹配，则应用 `pattern` 部分(或多个部分)中的规则;
  - 否则，将使用 `default` 部分中的规则。

`pattern` 和 `default` 部分的字段:

- `regexp` : 指标名称的模式。
- `age`: 数据的最小年龄(以秒为单位)。
- `precision`: 如何精确地定义数据的年代(以秒为单位)。应该是86400(一天的秒数)的除数。
- `function`: 聚合函数的名称，应用于年龄在[`age`, `age + precision`]范围内的数据。


```log
<graphite_rollup>
    <version_column_name>Version</version_column_name>
    <pattern>
        <regexp>click_cost</regexp>
        <function>any</function>
        <retention>
            <age>0</age>
            <precision>5</precision>
        </retention>
        <retention>
            <age>86400</age>
            <precision>60</precision>
        </retention>
    </pattern>
    <default>
        <function>max</function>
        <retention>
            <age>0</age>
            <precision>60</precision>
        </retention>
        <retention>
            <age>3600</age>
            <precision>300</precision>
        </retention>
        <retention>
            <age>86400</age>
            <precision>3600</precision>
        </retention>
    </default>
</graphite_rollup>
```