## File

文件表引擎以一种受支持的文件格式(`TabSeparated`、`Native`等)将数据保存在文件中。

- 使用例子:

  - 数据从 `ClickHouse` 导出到文件。
  - 将数据从一种格式转换为另一种格式。
  - 通过编辑磁盘上的文件更新 `ClickHouse` 中的数据。(仅支持更改 `File` 表引擎)
  
---

### Usage in ClickHouse Server

```
File(Format)
```

- Format: 
  - 指定一种可用的文件格式。
    - 需要满足 `SELECT` 和 `INSERT` 输入输出格式的要求，见 [Format](../../clickhouse_interfaces.md)

---

### Path Configuration
- 在服务器配置进行[设置](../../operations/server_configuration_parameters.md)
  - 其它地方，不允许为 `File` 指定文件系统路径。

---

### 文件保存路径(建表和写数)
- 当使用 `File(Format)` 创建表时，它将在该文件夹中创建空的子目录。当数据被写入该表时，放入到 （ `Path Configuration` 部分指定的目录）子目录中的 `data.Format` 文件

- 手动创建这个子文件夹和文件，然后将其附加到具有匹配名称的表信息上，这样就可以从该文件中查询数据。

**警告**: 

- 请小心使用此功能: 
  - 因为 ClickHouse不跟踪此类文件的外部更改。
  - 通过ClickHouse和ClickHouse外部同时进行写操作的结果是未定义的。

例子:

- 创建 `file_engine_table` 表:

```clickhouse
CREATE TABLE file_engine_table (name String, value UInt32) ENGINE=File(TabSeparated)
```

  - 默认情况下，ClickHouse将创建文件夹 `/var/lib/clickhouse/data/default/file_engine_table`(可在系统配置中进行设置)

- 手动创建 `/var/lib/clickhouse/data/default/file_engine_table/data.TabSeparated`

```log
$ cat data.TabSeparated
one 1
two 2
```

- 查询表数据：
```clickhouse
SELECT * FROM file_engine_table
```

```log
┌─name─┬─value─┐
│ one  │     1 │
│ two  │     2 │
└──────┴───────┘
```

---

### Usage in Clickhouse-local

- `clickhouse-local` `File Engine`: 
  - 除了格式之外，还接受文件路径
  - 默认输入/输出流可以使用数字或可读的名称(如 `0` 或`stdin` , `1` 或 `stdout` )来指定。
  
- **Example**:

```bash
$ echo -e "1,2\n3,4" | clickhouse-local -q "CREATE TABLE table (a Int64, b Int64) ENGINE = File(CSV, stdin); SELECT a, b FROM table; DROP TABLE table"
```

---

### Detail of implementation

- 多个`SELECT`查询可以并发执行，但是 `INSERT` 查询将彼此等待。

- 不支持:
  - `ALTER`
  - `SELECT ... SAMPLE`
  - 索引
  - 副本