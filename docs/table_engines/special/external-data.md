## External Data for Query Processing

---

### External Data

- `ClickHouse` 允许向服务器发送处理查询所需的数据( `External Data` )以及 `SELECT` 查询。
  - 这些数据放在一个临时表中（请参阅 "Temporary tables" 部分），可以在查询中使用（例如，在 IN 操作符中）。
  - 例如，如果您有一个包含重要用户标识符的文本文件，则可以将其与使用此列表过滤的查询一起上传到服务器。

**警告**: 

如果需要使用大量外部数据运行多个查询，请不要使用该特性。
最好提前把所需要的 `External Data` 以表的形式上传到数据库，再关联使用。

---

### 上传 `External Data`

- 下面两种方式上传：

  - `command-line` client(`non-interactive)
    - 在参数部分指定：
      - `--external --file=... [--name=...] [--format=...] [--types=...|--structure=...]`
      
        - require:
          - **--external** : 标记子句的开始。
          - **--file** : 带有表存储的文件的路径,或者，它指的是 STDIN. 只能从 stdin 中检索单个表
          
        - option:
          - **--name** : 表的名称，默认： `_data`.
          - **--format** : 文件中的数据格式。 默认： `TabSeparated`.
          
        - 以下参数必选其一:
          - **--types** : 
            - 以 `comma-separated` 的列类型列表。
            - 例如：`UInt64,String`
            - 列将被命名为 `_1`, `_2`,...
          - **--structure**: 
            - 表结构的格式  `UserID UInt64`, `URL String`.
            - 定义列的名字以及类型。
      
      - 在 "file" 中指定的文件将由 "format" 中指定的格式解析，使用在 "types" 或 "structure" 中指定的数据类型。
      - 该表将被上传到服务器，并在作为名称为 "name"临时表。
           
    - 对于传输的表的数量，可能有多个这样的部分。

栗子：
```bash
$ echo -ne "1\n2\n3\n" | clickhouse-client --query="SELECT count() FROM test.visits WHERE TraficSourceID IN _data" --external --file=- --types=Int8
849897
$ cat /etc/passwd | sed 's/:/\t/g' | clickhouse-client --query="SELECT shell, count() AS c FROM passwd GROUP BY shell ORDER BY c DESC" --external --file=- --name=passwd --structure='login String, unused String, uid UInt16, gid UInt16, comment String, home String, shell String'
/bin/sh 20
/bin/false      5
/bin/bash       4
/usr/sbin/nologin       1
/bin/sync       1
```    
     
  - `HTTP` interface
    - `External Data` 以 `multipart/form-data` 格式传递。
    - 每个表都作为单独的文件传输
    - 表名取自文件名
    - `query_string` 传递参数 `name_format`, `name_types` 和 `name_structure`
      - "name" 是这些参数对应的表名称
      - 参数的含义与使用命令行客户端时的含义相同。

**Example**：

```bash
$ cat /etc/passwd | sed 's/:/\t/g' > passwd.tsv

$ curl -F 'passwd=@passwd.tsv;' 'http://localhost:8123/?query=SELECT+shell,+count()+AS+c+FROM+passwd+GROUP+BY+shell+ORDER+BY+c+DESC&passwd_structure=login+String,+unused+String,+uid+UInt16,+gid+UInt16,+comment+String,+home+String,+shell+String'
/bin/sh 20
/bin/false      5
/bin/bash       4
/usr/sbin/nologin       1
/bin/sync       1
```

对于分布式查询，将临时表发送到所有远程服务器。