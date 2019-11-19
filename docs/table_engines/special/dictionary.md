## Dictionary

`Dictionary`  引擎将字典数据展示为一个 ClickHouse 的表。

例如: `products`  字典：

```xml
<dictionaries>
<dictionary>
        <name>products</name>
        <source>
            <odbc>
                <table>products</table>
                <connection_string>DSN=some-db-server</connection_string>
            </odbc>
        </source>
        <lifetime>
            <min>300</min>
            <max>360</max>
        </lifetime>
        <layout>
            <flat/>
        </layout>
        <structure>
            <id>
                <name>product_id</name>
            </id>
            <attribute>
                <name>title</name>
                <type>String</type>
                <null_value></null_value>
            </attribute>
        </structure>
</dictionary>
</dictionaries>
```

查询字典中的数据：

````clickhouse
SELECT
    name,
    type,
    key,
    attribute.names,
    attribute.types,
    bytes_allocated,
    element_count,
    source
FROM system.dictionaries
WHERE name = 'products'
````

```log
┌─name─────┬─type─┬─key────┬─attribute.names─┬─attribute.types─┬─bytes_allocated─┬─element_count─┬─source──────────┐
│ products │ Flat │ UInt64 │ ['title']       │ ['String']      │        23065376 │        175032 │ ODBC: .products │
└──────────┴──────┴────────┴─────────────────┴─────────────────┴─────────────────┴───────────────┴─────────────────┘
```


你可以使用  [dictGet*](https://clickhouse.yandex/docs/en/query_language/functions/ext_dict_functions/#ext_dict_functions)  函数来获取这种格式的字典数据。

**知悉**:

- 以下情况，这种视图并没有什么帮助：
  - 获取原始数据
  - 使用 `JOIN` 操作的时候
  
对于这些情况，你可以使用 `Dictionary` 引擎，它可以将字典数据展示在表中。

语法：

```clickhouse
CREATE TABLE %table_name% (%fields%) engine = Dictionary(%dictionary_name%)`
```

示例：

```clickhouse
create table products (product_id UInt64, title String) Engine = Dictionary(products);
```

```log
ok
```


`SELECT`: 

```clickhouse
select * from products limit 1;
```

```log
┌────product_id─┬─title───────────┐
│        152689 │ Some item       │
└───────────────┴─────────────────┘
```