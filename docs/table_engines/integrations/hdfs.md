## HDFS

这个引擎提供了与[Apache Hadoop](https://en.wikipedia.org/wiki/Apache_Hadoop)生态系统的集成，允许通过ClickHouse来管理hdfs上的数据。

这个引擎类似于 `File` 和 `URL`引擎(见 `Special` )，但是提供了hadoop独有的特性。

---

### Usage
```
ENGINE = HDFS(URI, format)
```

- `URI` 参数是HDFS中的整个文件URI。
  - `URI` 的路径部分可能包含globs。
    - 在这种情况下，表将是只读的。
    
- `format` 参数指定一种可用的文件格式。
  - 要执行 `SELECT` 查询，必须支持用于输入的`format`，并执行 `INSERT` 查询——用于输出。
  - [`format`](../../clickhouse_interfaces.md) 部分列出了可用的格式。
  

**栗:**

1. 创建 `hdfs_engine_table` 表:
```clickhouse
CREATE TABLE hdfs_engine_table (name String, value UInt32) ENGINE=HDFS('hdfs://hdfs1:9000/other_storage', 'TSV')
```

2. 填写文件:
```clickhouse
INSERT INTO hdfs_engine_table VALUES ('one', 1), ('two', 2), ('three', 3)
```

3.查询数据:
```clickhouse
SELECT * FROM hdfs_engine_table LIMIT 2
```

```log
┌─name─┬─value─┐
│ one  │     1 │
│ two  │     2 │
└──────┴───────┘
```

---

### Implementation Details

- 读写可以是并行的
- 不支持:
  - `Alter` 和 `SELECT...SAMPLE` 操作。
  - 索引
  - 备份
  
**Globs in path**

- 多个路径组件可以有全局特性。
  - 正在处理的文件应该存在并与整个路径模式匹配。
  - 文件列表在 `SELECT` 期间确定(而不是在 `CREATE` 时)。
    - `*` : 替换除 `/` 包括空字符串之外的任意数量的任何字符。
    - `?`: 替换任何单个字符。
    - `{some_string,another_string,yet_another_one}` :
      - 替换任何字符串 `'some_string'`,`'another_string'`,`'yet_another_one'`
    - `{N..M}` : 替换从N到M范围的任何数字(闭区间)
    
带有{}的结构类似于远程表函数(见查询语法知识)。

**栗子**：

假设我们有几个 `TSV` 格式的文件，在HDFS上有以下 `URI`:
  - 'hdfs://hdfs1:9000/some_dir/some_file_1'
  - 'hdfs://hdfs1:9000/some_dir/some_file_2'
  - 'hdfs://hdfs1:9000/some_dir/some_file_3'
  - 'hdfs://hdfs1:9000/another_dir/some_file_1'
  - 'hdfs://hdfs1:9000/another_dir/some_file_2'
  - 'hdfs://hdfs1:9000/another_dir/some_file_3'

有几种方法使一个表由所有六个文件:

```clickhouse
CREATE TABLE table_with_range (name String, value UInt32) ENGINE = HDFS('hdfs://hdfs1:9000/{some,another}_dir/some_file_{1..3}', 'TSV')
```

另一种方法: 

```clickhouse
CREATE TABLE table_with_question_mark (name String, value UInt32) ENGINE = HDFS('hdfs://hdfs1:9000/{some,another}_dir/some_file_?', 'TSV')
```

表中包含两个目录中的所有文件(所有文件应满足查询中描述的格式和模式):
```clickhouse
CREATE TABLE table_with_asterisk (name String, value UInt32) ENGINE = HDFS('hdfs://hdfs1:9000/{some,another}_dir/*', 'TSV')
```

---

**警告**: 
如果文件清单中包含带前导零的数字范围，则对每个数字分别使用带大括号的构造或使用`?`.

**栗子**：
创建一个带有`file000`、`file001`、…`file999`的表:
```clickhouse
CREARE TABLE big_table (name String, value UInt32) ENGINE = HDFS('hdfs://hdfs1:9000/big_dir/file{0..9}{0..9}{0..9}', 'CSV')
```