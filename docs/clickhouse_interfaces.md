# Client

---

ClickHouse提供了两个网络接口（为了安全起见，都可以选择将两者包装在TLS(安全传输协议)中）：

- HTTP： 易于使用
- Native TCP:  开销小

在大多数情况下，建议使用适当的工具或库，而不是直接与这些工具或库进行交互。 Yandex的官方支持如下:
- command-line client
- JDBC driver
- ODBC driver
- C++ client library

还有许多第三方库可供使用ClickHouse: 
- Client libraries
- Integrations
- Visual interfaces


接下来针对上面的内容，进行阐述：

---

## HTTP 客户端
- 跨平台，跨编程语言使用ClickHouse.
- `HTTP` 接口比 `TCP` 原生接口更为局限， 但有更好的兼容性。 
- ClickHouse 默认端口 `8123` 来监控 `HTTP` 请求, 端口可配。 

```bash
curl 'http://localhost:8123/'
ok.
```

- 可以通过下面几种方式发送`HTTP` 请求：
  - `query` 参数,即 `GET` 请求
  - `POST`
  - 混合形式： 参数 + `POST`

状态码：
  - `200`： 请求成功
  - `500`:  请求失败

- 注意点：
  - 使用`GET` 请求时， `readonly` 会被设置。
  - `INSERT` 必须通过 `POST` 请求。
  - `URL`的大小会限制在 `16KB`.
  - `wget` 命令不推荐使用，因它在 `HTTP 1.1` 协议下使用 `keep-alive` 和 `Transfer-Encoding:chunked` 头部设置时，不能很好的功能。
  - `curl` 命令在`url` 中填写`sql`语句时，如果遇到空格需要转义。
  - 默认情况下，返回的数据是 `TabSeparated `格式的， 可配（`Format`)
  - `compress=1` ，服务会返回压缩的数据
  - `decompress=1` ，服务会解压通过 POST 方法发送的数据
  - 默认数据库为 `default` (已注册)

- 下面举个栗子：

```bash
## 使用类似 INSERT 的查询来插入数据：
echo 'INSERT INTO t VALUES (1),(2),(3)' | curl 'http://localhost:8123/' --data-binary @-

## 数据可以从查询中单独发送
echo '(4),(5),(6)' | curl 'http://localhost:8123/?query=INSERT%20INTO%20t%20VALUES' --data-binary @-

## 可以指定任何数据格式。值的格式和写入表 t 的值的格式相同：
$ echo '(7),(8),(9)' | curl 'http://localhost:8123/?query=INSERT%20INTO%20t%20FORMAT%20Values' --data-binary @-

## 若要插入 tab 分割的数据，需要指定对应的格式：
$ echo -ne '10\n11\n12\n' | curl 'http://localhost:8123/?query=INSERT%20INTO%20t%20FORMAT%20TabSeparated' --data-binary @-

## 查询表
$ curl 'http://localhost:8123/?query=SELECT%20a%20FROM%20t'

## 删除表
$ echo 'DROP TABLE t' | curl 'http://localhost:8123/' --data-binary @-
```

- 用户密码可通过下面两种方式设置
```bash
## 通过 HTTP Basic Authentication。示例：
$ echo 'SELECT 1' | curl 'http://user:password@localhost:8123/' -d @-
## 通过 URL 参数 中的 'user' 和 'password'。示例：
$ echo 'SELECT 1' | curl 'http://localhost:8123/?user=user&password=password' -d @-

## 注意： 如果用户名没有指定，默认的用户是 default。如果密码没有指定，默认会使用空密码。 
```

### `HTTP` 压缩
`ClickHouse`支持`gzip`，`br`以及`deflate` 压缩方法: 

- 压缩请求 `Content-Encoding:compression_method`
- 压缩响应 `Accept-Encoding: compression_method`
- 要启用HTTP压缩，必须使用[`ClickHouse enable_http_compression`](./clickhouse_operations.md)设置
- 您可以在`http_zlib_compression_level`设置中为所有压缩方法配置数据压缩级别。
- 在传输大量数据或创建立即压缩的转储时，可以使用它来减少网络流量。
- 默认情况下，某些`HTTP`客户端可能会从服务器解压缩数据（使用`gzip`和`deflate`），即使正确使用压缩设置，也可能会获得解压缩的数据。

通过压缩发送数据

```bash
#发送数据到服务器:
$ curl -vsS "http://localhost:8123/?enable_http_compression=1" -d 'SELECT number FROM system.numbers LIMIT 10' -H 'Accept-Encoding: gzip'

#发送数据到客户端:
$ echo "SELECT 1" | gzip -c | curl -sS --data-binary @- -H 'Content-Encoding: gzip' 'http://localhost:8123/'
```

你能使用`database`  url 参数指定默认`database`:
```bash
$ echo 'SELECT number FROM numbers LIMIT 10' | curl 'http://localhost:8123/?database=system' --data-binary @-
```

你可以在HTTP 协议中使用 `ClickHouse session`. 

你需要添加 `session_id` GET 参数到请求体中。
- 你能使用任何字符串作为`session_id`
- `session` 在闲置情况下，存活60s, 可以通过配置服务 `default_session_timeout` 参数进行修改或者在请求体中增加`session_timeout`GET参数来指定。
- `session_check=1` 检查会话状态。
- 一次只能在一个`session` 中执行一次查询操作

您可以收到有关查询在进度X-ClickHouse-Progress响应头。为此，启用send_progress_in_http_headers(在运维知识进行阐述)。标头序列的示例：

```
X-ClickHouse-Progress: {"read_rows":"2752512","read_bytes":"240570816","total_rows_to_read":"8880128"}
X-ClickHouse-Progress: {"read_rows":"5439488","read_bytes":"482285394","total_rows_to_read":"8880128"}
X-ClickHouse-Progress: {"read_rows":"8783786","read_bytes":"819092887","total_rows_to_read":"8880128"}
```

可能的头字段：
- `read_rows` — 读取的行数
- `read_bytes` — 读取的数据量（以字节为单位）。
- `total_rows_to_read` — 要读取的总行数。
- `written_rows` — 写入的行数。
- `written_bytes` — 以字节为单位的数据量。

如果HTTP连接丢失，正在运行的请求不会自动停止。解析和数据格式化是在服务器端执行的，使用网络可能无效。可选的`query_id`参数可以作为查询ID（任何字符串）传递。有关更多信息，请参见“Settings, replace_running_query”部分。

可选的“quota_key”参数可以作为（Quotas）配额密钥（任何字符串）传递。有关更多信息，请参见“Quotas（配额）（运维部分有详细描述）”部分。

HTTP接口允许传递外部数据（external temporary tables）以进行查询。有关更多信息，请参见“External data for query processing”(运维部分有详细描述)部分。

### Response Buffering
您可以在服务器端启用响应缓冲。在`buffer_size`和`wait_end_of_query` URL参数提供了这个目的。
 - `buffer_size`确定结果中要缓冲在服务器内存中的字节数。如果结果主体大于此阈值，则将缓冲区写入HTTP通道，并将剩余数据直接发送到HTTP通道。
 - 为了确保缓冲整个响应，请设置`wait_end_of_query=1`。在这种情况下，未存储在内存中的数据将被缓冲在临时服务器文件中。

例：

```bash
$ curl -sS 'http://localhost:8123/?max_result_bytes=4000000&buffer_size=3000000&wait_end_of_query=1' -d 'SELECT toUInt8(number) FROM system.numbers LIMIT 9000000 FORMAT RowBinary'
```

使用缓冲以避免在将`Response`代码和`HTTP Headers`发送到客户端之后发生查询处理错误的情况。在这种情况下，错误消息会写在响应正文的末尾，而在客户端，只能在解析阶段检测到错误。

### Queries with Parameters
您可以使用参数创建查询，并通过相应的HTTP请求参数传递参数值。有关更多信息，请参见上文。

Example:
¶
```bash
$ curl -sS "<address>?param_id=2&param_phrase=test" -d "SELECT * FROM table WHERE int_column = {id:UInt8} and string_column = {phrase:String}"
```


## Command-line Client

要从命令行工作，可以使用`clickhouse-client`：

```bash
$ clickhouse-client
ClickHouse client version 0.0.26176.
Connecting to localhost:9000.
Connected to ClickHouse server version 0.0.26176.

:)
```

客户端支持命令行选项和配置文件。有关更多信息，请参见下文[`Configuring`]。

### Usage

客户端可以在交互式和非交互式（`batch`）模式下使用。要使用批处理模式，请指定`query`参数，或将数据发送到`stdin`（它验证`stdin`不是`terminal`），或两者兼而有之。与 `HTTP` 接口类似，当使用`query`参数并将数据发送到`stdin`时，请求是`query`参数，换行符和`stdin`中数据的串联。这对于大型 INSERT `queries` 很方便。

使用客户端插入数据的示例：

```bash
$ echo -ne "1, 'some text', '2016-08-14 00:00:00'\n2, 'some more text', '2016-08-14 00:00:01'" | clickhouse-client --database=test --query="INSERT INTO test FORMAT CSV";

$ cat <<_EOF | clickhouse-client --database=test --query="INSERT INTO test FORMAT CSV";
3, 'some text', '2016-08-14 00:00:00'
4, 'some more text', '2016-08-14 00:00:01'
_EOF

$ cat file.csv | clickhouse-client --database=test --query="INSERT INTO test FORMAT CSV";
```

- non-interactive(`batch mode`)
  - 通过`FORMAT` 设置数据格式，默认为 `TabSeparated`。 
  - `--query`： 执行一个语句。
  - `--multiquery` 通过`script` 执行除了 `INSERT` 批量语句。
  - 查询结果不需要另外指定分隔符（连续输出）。
  - 启动`clickhouse-client`, 需要几十毫秒时间。
  
- `interactive mode`
  - 通过`FORMAT` 设置数据格式，默认为 `PrettyCompact`。 也可以通过`\G`在查询末尾使用命令行中的`--format`或`--vertical`参数或使用客户端配置文件来指定格式。
  - 未指定`multiline`, 语句末尾不需要 `;` 要输入多行， 末尾 加`\`
  - 指定`multiline`, 语句末尾加 `;` 号并按`Enter`。如果在输入行的末尾省略了`;`，则将要求您输入下一行。
  - 可以指定`\G`代替分号或在分号之后。这表示垂直格式。以这种格式，每个值都打印在单独的行上，这对于宽表很方便。添加了此不常用功能仅为了与 `MySQL CLI` 兼容。
  - 命令行基于`readline`（和`history`或`libedit`，或者`without a library`，具体情况取决于`build`).换句话说，它使用熟悉的键盘快捷键并保留历史记录。历史记录被写入`~/.clickhouse-client-history`。
  - 退出`client`，请按 `Ctrl+D`（或 `Ctrl+C`)，或输入以下命令之一:  [`exit`, `quit`, `logout`, `exit;`, `quit;`, `logout;`, `q`, `Q`, `:q`]
  - 您可以通过按`Ctrl+C`取消长查询。但是，您仍然需要稍等一会儿，服务器才能中止请求。在某些阶段无法取消查询。如果您不等待，然后再次按`Ctrl+C`，则客户端将退出。
  - 处理请求时，客户端显示：
    - 进度，每秒更新不超过 10 次（默认情况下）。对于快速查询，进度可能没有时间显示。
    - 解析后的格式化查询，用于调试很合适。
    - 指定格式的结果。
    - result: 行数，消耗时间以及平均速度。
  - 命令行客户端允许传递`external data`（`external temporary tables`）以进行查询。有关更多信息，请参见`external temporary tables`部分。

####  Queries with Parameters

您可以使用参数创建查询，并从客户端应用程序将值传递给它们。这样可以避免在客户端使用特定的动态值格式化查询。例如：

```bash
$ clickhouse-client --param_parName="[1, 2]"  -q "SELECT * FROM table WHERE a = {parName:Array(UInt16)}"
```

##### **Query Syntax**

像往常一样设置查询格式，然后将要从应用程序参数传递给查询的值放在大括号中，格式如下：

```
{<name>:<data type>}
```
- `name`—占位符标识符。在控制台客户端中，应在应用程序参数中将其用作`--param_<name> = value`。
- `data type`—  应用程序参数值的[Data type](../docs/clickhouse_datatype.md)。例如，像这样的数据结构`(integer, ('string', integer))`可以具有`Tuple(UInt8, Tuple(String, UInt8))`数据类型（您也可以使用其他`integer` 类型。

##### Example

```bash
$ clickhouse-client --param_tuple_in_tuple="(10, ('dt', 10))" -q "SELECT * FROM table WHERE val = {tuple_in_tuple:Tuple(UInt8, Tuple(String, UInt8))}"
```

### Configuring

您可以使用以下方法将参数传递给`clickhouse-client`（所有参数都有默认值）：

- 从命令行

命令行选项将覆盖配置文件中的默认值和设置。

- 配置文件

配置文件中的设置将覆盖默认值。

#### Command Line Options

- `--host, -h`-服务器名称，默认为`localhost`。您可以使用名称或 `IPv4` 或 `IPv6` 地址。
- `--port`–要连接的端口。默认值：`9000`。请注意，`HTTP` 接口和本机接口使用不同的端口。
- `--user, -u`–用户名。默认值：默认。
- `--password`\- 密码。默认值：空字符串。
- `--query, -q` –使用非交互模式时要处理的查询。
- `--database, -d`–选择当前的默认数据库。默认值：服务器设置中的当前数据库（默认为`default`）。
- `--multiline, -m` –如果指定，则允许多行查询（不要在 Enter 上发送查询）。
- `--multiquery, -n`–如果指定，则允许处理多个用逗号分隔的查询。仅适用于非交互模式。
- `--format, -f` –使用指定的默认格式输出结果。
- `--vertical, -E`–如果指定，则默认使用`vertical`格式输出结果。这与`--format = Vertical`相同。以这种格式，每个值都打印在单独的行上，这在显示宽表时很有用。
- `--time, -t` –如果指定，则在非交互模式下将查询执行时间打印为`stderr`.
- `--stacktrace` –如果指定，则在发生异常时打印`stack trace`。
- `--config-file` –配置文件的名称。
- `--secure` –如果指定，将通过安全连接连接到服务器。
- `--param_<name>`— 具体参数值,根据配置和业务指定即可。

#### Configurations Files

`clickhouse-client`  使用以下第一个现有文件：

- 在`-config-file`参数中定义。
- `./clickhouse-client.xml`
- `~/.clickhouse-client/config.xml`
- `/etc/clickhouse-client/config.xml`

配置文件示例：
```xml
<config>
    <user>username</user>
    <password>password</password>
    <secure>False</secure>
</config>
```

## Native Interface(TCP)

`Native`协议用于`Command-line Client`，分布式查询处理期间的服务器间以及其他`C++`程序. 目前官方还没有正式的文档对此进行说明，但是可以从 `ClickHouse` 源代码（从[此处](https://github.com/ClickHouse/ClickHouse/tree/master/dbms/src/Client)开始）和/或通过拦截和分析 TCP 流量进行反向工程来分析。


## Formats for Input and Output Data

ClickHouse 可以`accept`和`return`各种格式的数据。输入支持的格式可用于解析提供给`INSERT`的数据，`SELECT`从文件支持的表（例如 `File`，`URL` 或 `HDFS`）执行或读取外部字典。支持输出的格式可用于安排`SELECT`结果，并在文件支持的表中执行`INSERT`。

支持的格式为：

```
Format	                      Input	Output
TabSeparated	                ✔	✔
TabSeparatedRaw	                ✗	✔
TabSeparatedWithNames	        ✔	✔
TabSeparatedWithNamesAndTypes	✔	✔
Template     	                ✔	✔
TemplateIgnoreSpaces	        ✔	✗
CSV                             ✔	✔
CSVWithNames	                ✔	✔
CustomSeparated	                ✔	✔
Values                      	✔	✔
Vertical                    	✗	✔
JSON                        	✗	✔
JSONCompact                     ✗	✔
JSONEachRow                     ✔	✔
TSKV	                        ✔	✔
Pretty	                        ✗	✔
PrettyCompact	                ✗	✔
PrettyCompactMonoBlock	        ✗	✔
PrettyNoEscapes             	✗	✔
PrettySpace                     ✗	✔
Protobuf                        ✔	✔
Parquet	                        ✔	✔
RowBinary                       ✔	✔
RowBinaryWithNamesAndTypes      ✔	✔
Native	                        ✔	✔
Null	                        ✗	✔
XML                             ✗	✔
CapnProto                       ✔	✗
```


您可以使用 ClickHouse 设置来控制某些格式处理参数。有关更多信息，请阅读【运维】部分。

## TabSeparated

在 TabSeparated 格式中，数据按行写入。每行包含由制表符分隔的值。除了行中的最后一个值（后面紧跟换行符）之外，每个值都跟随一个制表符。 在任何地方都可以使用严格的 Unix 命令行。最后一行还必须在最后包含换行符。值以文本格式编写，不包含引号，并且要转义特殊字符。

这种格式也可以用  `TSV`  来表示。

TabSeparated 格式非常方便用于自定义程序或脚本处理数据。HTTP 客户端接口默认会用这种格式，命令行客户端批量模式下也会用这种格式。这种格式允许在不同数据库之间传输数据。例如，从 MYSQL 中导出数据然后导入到 ClickHouse 中，反之亦然。

TabSeparated 格式支持输出数据总值（当使用 WITH TOTALS） 以及极值（当 'extremes' 设置是 1）。这种情况下，总值和极值  输出在主数据的后面。主要的数据，总值，极值会以一个空行隔开，例如：

```sql
SELECT EventDate, count() AS c FROM test.hits GROUP BY EventDate WITH TOTALS ORDER BY EventDate FORMAT TabSeparated``
```

```log
2014-03-17      1406958
2014-03-18      1383658
2014-03-19      1405797
2014-03-20      1353623
2014-03-21      1245779
2014-03-22      1031592
2014-03-23      1046491

0000-00-00      8873898

2014-03-17      1031592
2014-03-23      1406958
```

### 数据解析格式

整数以十进制形式写入。数字在开头可以包含额外的  `+`  字符（解析时忽略，格式化时不记录）。非负数不能包含负号。 读取时，允许将空字符串解析为零，或者（对于带符号的类型）将仅包含负号的字符串解析为零。 不符合相应数据类型的数字可能会被解析为不同的数字，而不会显示错误消息。

浮点数以十进制形式写入。点号用作小数点分隔符。支持指数等符号，如'inf'，'+ inf'，'-inf'和'nan'。 浮点数的输入可以以小数点开始或结束。 格式化的时候，浮点数的精确度可能会丢失。 解析的时候，没有严格需要去读取与机器可以表示的最接近的数值。

日期会以 YYYY-MM-DD 格式写入和解析，但会以任何字符作为分隔符。 带时间的日期会以 YYYY-MM-DD hh:mm:ss 格式写入和解析，但会以任何字符作为分隔符。 这一切都发生在客户端或服务器启动时的系统时区（取决于哪一种格式的数据）。对于具有时间的日期，夏时制时间未指定。 因此，如果转储在夏令时中有时间，则转储不会明确地匹配数据，解析将选择两者之一。 在读取操作期间，不正确的日期和具有时间的日期可以使用自然溢出或空日期和时间进行分析，而不会出现错误消息。

有个例外情况，Unix 时间戳格式（10 个十进制数字）也支持使用时间解析日期。结果不是时区相关的。格式 YYYY-MM-DD hh:mm:ss 和 NNNNNNNNNN 会自动区分。

字符串以反斜线转义的特殊字符输出。 以下转义序列用于输出：`\b`，`\f`，`\r`，`\n`，`\t`，`\0`，`\'`，`\\`。 解析还支持`\a`，`\v`和`\xHH`（十六进制转义字符）和任何`\c`字符，其中`c`是任何字符（这些序列被转换为`c`）。 因此，读取数据支持可以将换行符写为`\n`或`\`的格式，或者换行。例如，字符串  `Hello world`  在单词之间换行而不是空格可以解析为以下任何形式：

```
Hello\nworld

Hello\
world
```

第二种形式是支持的，因为 MySQL 读取 tab-separated 格式数据集的时候也会使用它。

在 TabSeparated 格式中传递数据时需要转义的最小字符集为：Tab，换行符（LF）和反斜杠。

只有一小组符号会被转义。你可以轻易地找到一个字符串值，但这不会正常在你的终端显示。

数组写在方括号内的逗号分隔值列表中。 通常情况下，数组中的数字项目会被拼凑，但日期，带时间的日期以及字符串将使用与上面相同的转义规则用单引号引起来。

`NULL` 将输出为  `\N`。

## TabSeparatedRaw

与  `TabSeparated`  格式不一样的是，行数据是不会被转义的。 该格式仅适用于输出查询结果，但不适用于解析输入（将数据插入到表中）。

这种格式也可以使用名称  `TSVRaw`  来表示。

## TabSeparatedWithNames

与  `TabSeparated`  格式不一样的是，第一行会显示列的名称。 在解析过程中，第一行完全被忽略。您不能使用列名来确定其位置或检查其正确性。 （未来可能会加入解析头行的功能）

这种格式也可以使用名称  `TSVWithNames`  来表示。

## TabSeparatedWithNamesAndTypes

与  `TabSeparated`  格式不一样的是，第一行会显示列的名称，第二行会显示列的类型。 在解析过程中，第一行和第二行完全被忽略。

这种格式也可以使用名称  `TSVWithNamesAndTypes`  来表示。

## TSKV

与  `TabSeparated`  格式类似，但它输出的是  `name=value`  的格式。名称会和  `TabSeparated`  格式一样被转义，`=`  字符也会被转义。

```log
SearchPhrase=   count()=8267016
SearchPhrase=bathroom interior design    count()=2166
SearchPhrase=yandex     count()=1655
SearchPhrase=2014 spring fashion    count()=1549
SearchPhrase=freeform photos       count()=1480
SearchPhrase=angelina jolie    count()=1245
SearchPhrase=omsk       count()=1112
SearchPhrase=photos of dog breeds    count()=1091
SearchPhrase=curtain designs        count()=1064
SearchPhrase=baku       count()=1000
```

`NULL`  输出为  `\N`。

```sql
SELECT * FROM t_null FORMAT TSKV
```

```log
x=1 y=\N
```

当有大量的小列时，这种格式是低效的，通常没有理由使用它。它被用于 Yandex 公司的一些部门。

数据的输出和解析都支持这种格式。对于解析，任何顺序都支持不同列的值。可以省略某些值，用  `-`  表示， 它们被视为等于它们的默认值。在这种情况下，零和空行被用作默认值。作为默认值，不支持表中指定的复杂值。

对于不带等号或值，可以用附加字段  `tskv`  来表示，这种在解析上是被允许的。这样的话该字段被忽略。

## CSV

按逗号分隔的数据格式([RFC](https://tools.ietf.org/html/rfc4180))。

格式化的时候，行是用双引号括起来的。字符串中的双引号会以两个双引号输出，除此之外没有其他规则来做字符转义了。日期和时间也会以双引号包括。数字的输出不带引号。值由一个单独的字符隔开，这个字符默认是  `,`。行使用 Unix 换行符（LF）分隔。 数组序列化成 CSV 规则如下：首先将数组序列化为 TabSeparated 格式的字符串，然后将结果字符串用双引号包括输出到 CSV。CSV 格式的元组被序列化为单独的列（即它们在元组中的嵌套关系会丢失）。

```bash
clickhouse-client --format_csv_delimiter="|" --query="INSERT INTO test.csv FORMAT CSV" < data.csv
```

默认情况下间隔符是  `,` ，在  `format_csv_delimiter`  中可以了解更多间隔符配置。

解析的时候，可以使用或不使用引号来解析所有值。支持双引号和单引号。行也可以不用引号排列。 在这种情况下，它们被解析为逗号或换行符（CR 或 LF）。在解析不带引号的行时，若违反 RFC 规则，会忽略前导和尾随的空格和制表符。 对于换行，全部支持 Unix（LF），Windows（CR LF）和 Mac OS Classic（CR LF）。

`NULL`  将输出为  `\N`。

CSV 格式是和 TabSeparated 一样的方式输出总数和极值。

## CSVWithNames

会输出带头部行，和  `TabSeparatedWithNames`  一样。

## JSON

以 JSON 格式输出数据。除了数据表之外，它还输出列名称和类型以及一些附加信息：输出行的总数以及在没有 LIMIT 时可以输出的行数。 例：

```sql
SELECT SearchPhrase, count() AS c FROM test.hits GROUP BY SearchPhrase WITH TOTALS ORDER BY c DESC LIMIT 5 FORMAT JSON
```

```log
{
        "meta":
        [
                {
                        "name": "SearchPhrase",
                        "type": "String"
                },
                {
                        "name": "c",
                        "type": "UInt64"
                }
        ],

        "data":
        [
                {
                        "SearchPhrase": "",
                        "c": "8267016"
                },
                {
                        "SearchPhrase": "bathroom interior design",
                        "c": "2166"
                },
                {
                        "SearchPhrase": "yandex",
                        "c": "1655"
                },
                {
                        "SearchPhrase": "spring 2014 fashion",
                        "c": "1549"
                },
                {
                        "SearchPhrase": "freeform photos",
                        "c": "1480"
                }
        ],

        "totals":
        {
                "SearchPhrase": "",
                "c": "8873898"
        },

        "extremes":
        {
                "min":
                {
                        "SearchPhrase": "",
                        "c": "1480"
                },
                "max":
                {
                        "SearchPhrase": "",
                        "c": "8267016"
                }
        },

        "rows": 5,

        "rows_before_limit_at_least": 141137
}
```

JSON 与 JavaScript 兼容。为了确保这一点，一些字符被另外转义：斜线`/`被转义为`\/`; 替代的换行符  `U+2028`  和  `U+2029`  会打断一些浏览器解析，它们会被转义为  `\uXXXX`。 ASCII 控制字符被转义：退格，换页，换行，回车和水平制表符被替换为`\b`，`\f`，`\n`，`\r`，`\t`  作为使用`\uXXXX`序列的 00-1F 范围内的剩余字节。 无效的 UTF-8 序列更改为替换字符 ，因此输出文本将包含有效的 UTF-8 序列。 为了与 JavaScript 兼容，默认情况下，Int64 和 UInt64 整数用双引号引起来。要除去引号，可以将配置参数 output_format_json_quote_64bit_integers 设置为 0。

`rows` – 结果输出的行数。

`rows_before_limit_at_least`  去掉 LIMIT 过滤后的最小行总数。 只会在查询包含 LIMIT 条件时输出。 若查询包含 GROUP BY，rows_before_limit_at_least 就是去掉 LIMIT 后过滤后的准确行数。

`totals` – 总值 （当使用 TOTALS 条件时）。

`extremes` – 极值 （当 extremes 设置为 1 时）。

该格式仅适用于输出查询结果，但不适用于解析输入（将数据插入到表中）。

ClickHouse 支持  `NULL` 在 JSON 格式中以  `null`  输出来表示.

参考 JSONEachRow 格式。

## JSONCompact

与 JSON 格式不同的是它以数组的方式输出结果，而不是以结构体。

示例：

```json
{
        "meta":
        [
                {
                        "name": "SearchPhrase",
                        "type": "String"
                },
                {
                        "name": "c",
                        "type": "UInt64"
                }
        ],

        "data":
        [
                ["", "8267016"],
                ["bathroom interior design", "2166"],
                ["yandex", "1655"],
                ["fashion trends spring 2014", "1549"],
                ["freeform photo", "1480"]
        ],

        "totals": ["","8873898"],

        "extremes":
        {
                "min": ["","1480"],
                "max": ["","8267016"]
        },

        "rows": 5,

        "rows_before_limit_at_least": 141137
}
```

这种格式仅仅适用于输出结果集，而不适用于解析（将数据插入到表中）。 参考  `JSONEachRow`  格式。

## JSONEachRow

将数据结果每一行以 JSON 结构体输出（换行分割 JSON 结构体）。但整个数据都是无效的JSON。

```json
{"SearchPhrase":"curtain designs","count()":"1064"}
{"SearchPhrase":"baku","count()":"1000"}
{"SearchPhrase":"","count":"8267016"}
```

插入数据时，应为每行提供一个单独的JSON对象。

---

### 插入数据

```sql
INSERT INTO UserActivity FORMAT JSONEachRow {"PageViews":5, "UserID":"4324182021466249494", "Duration":146,"Sign":-1} {"UserID":"4324182021466249494","PageViews":6,"Duration":185,"Sign":1}
```

- ClickHouse允许：
  - 对象中`K-V`对的任何顺序。
  - 省略一些值。

ClickHouse会忽略对象后的元素和逗号之间的空格。您可以在一行中传递所有对象。您不必用换行符将它们分开。

**遗漏值处理**

ClickHouse用相应数据类型的默认值替换省略的值。

如 `DEFAULT expr` 被指定，则`ClickHouse` 根据 `input_format_defaults_for_omitted_fields` 设置使用不同的替换规则。

请考虑下表：
```sql
CREATE TABLE IF NOT EXISTS example_table
(
    x UInt32,
    a DEFAULT x * 2
) ENGINE = Memory;
```

`insert_sample_with_metadata` = 0，则 x和a的默认值等于0(作为`UInt32`数据类型的默认值)。
`insert_sample_with_metadata` = 1，则默认值x等于0，但默认值a等于 x * 2。

请谨慎使用此选项。启用它会对 `ClickHouse` 服务器性能产生负面影响。

**Select Data**

以 `UserActivity` 表为例：
```log
───UserID─┬─PageViews─┬─Duration─┬─Sign─┐
│ 4324182021466249494 │         5 │      146 │   -1 │
│ 4324182021466249494 │         6 │      185 │    1 │
└─────────────────────┴───────────┴──────────┴──────┘
```

查询 `SELECT * FROM UserActivity FORMAT JSONEachRow` 返回：

```log
{"UserID":"4324182021466249494","PageViews":5,"Duration":146,"Sign":-1}
{"UserID":"4324182021466249494","PageViews":6,"Duration":185,"Sign":1}
```

与JSON格式不同，不会替换无效的 `UTF-8` 序列。值的转义与`JSON` 相同。

字符串中可以输出任何字节集。 如果您确定可以将表中的数据格式化为JSON而不丢失任何信息，请使用 `JSONEachRow` 格式。

## Native

最高性能的格式。 据通过二进制格式的块进行写入和读取。对于每个块，该块中的行数，列数，列名称和类型以及列的部分将被相继记录。 换句话说，这种格式是 “列式”的, 它不会将列转换为行。 这是用于在服务器之间进行交互的本地界面中使用的格式，用于使用命令行客户端和 C++ 客户端。

您可以使用此格式快速生成只能由 `ClickHouse DBMS` 读取的格式。但自己处理这种格式是没有意义的。

## Null

没有输出。但是，查询已处理完毕，并且在使用命令行客户端时，数据将传输到客户端。这仅用于测试，包括生产力测试。 显然，这种格式只适用于输出，不适用于解析。

## Pretty

将数据以表格形式输出，也可以使用 ANSI 转义字符在终端中设置颜色。 它会绘制一个完整的表格，每行数据在终端中占用两行。 每一个结果块都会以单独的表格输出。这是很有必要的，以便结果块不用缓冲结果输出（缓冲在可以预见结果集宽度的时候是很有必要的）。

`NULL` 输出为  `ᴺᵁᴸᴸ`。

```sql
SELECT * FROM t_null
```

```log
┌─x─┬────y─┐
│ 1 │ ᴺᵁᴸᴸ │
└───┴──────┘
```

行不会以 `Pretty*` 格式转义。显示了 `PrettyCompact` 格式的示例：
```log
SELECT 'String with \'quotes\' and \t character' AS Escaping_test
┌─Escaping_test────────────────────────┐
│ String with 'quotes' and   character │
└──────────────────────────────────────┘
```

为避免将太多数据传输到终端，只打印前 10,000 行。 如果行数大于或等于 10,000，则会显示消息“Showed first 10 000”。 该格式仅适用于输出查询结果，但不适用于解析输入（将数据插入到表中）。

Pretty 格式支持输出总值（当使用 WITH TOTALS 时）和极值（当  `extremes`  设置为 1 时）。 在这些情况下，总数值和极值在主数据之后以单独的表格形式输出。 示例（以 PrettyCompact 格式显示）：

```sql
SELECT EventDate, count() AS c FROM test.hits GROUP BY EventDate WITH TOTALS ORDER BY EventDate FORMAT PrettyCompact
```
```log
┌──EventDate─┬───────c─┐
│ 2014-03-17 │ 1406958 │
│ 2014-03-18 │ 1383658 │
│ 2014-03-19 │ 1405797 │
│ 2014-03-20 │ 1353623 │
│ 2014-03-21 │ 1245779 │
│ 2014-03-22 │ 1031592 │
│ 2014-03-23 │ 1046491 │
└────────────┴─────────┘

Totals:
┌──EventDate─┬───────c─┐
│ 0000-00-00 │ 8873898 │
└────────────┴─────────┘

Extremes:
┌──EventDate─┬───────c─┐
│ 2014-03-17 │ 1031592 │
│ 2014-03-23 │ 1406958 │
└────────────┴─────────┘
```

## PrettyCompact

与  `Pretty`  格式不一样的是，`PrettyCompact`  去掉了行之间的表格分割线，这样使得结果更加紧凑。这种格式会在交互命令行客户端下默认使用。

## PrettyCompactMonoBlock

与  `PrettyCompact`  格式不一样的是，它支持 10,000 行数据缓冲，然后输出在一个表格中，不会按照块来区分

## PrettyNoEscapes

与  `Pretty`  格式不一样的是，它不使用 ANSI 字符转义， 这在浏览器显示数据以及在使用  `watch`  命令行工具是有必要的。

示例：

```log
watch -n1 "clickhouse-client --query='SELECT event, value FROM system.events FORMAT PrettyCompactNoEscapes'"
```

您可以使用 HTTP 接口来获取数据，显示在浏览器中。

### PrettyCompactNoEscapes

用法类似上述。

### PrettySpaceNoEscapes

用法类似上述。

## PrettySpace

与  `PrettyCompact`(#prettycompact) 格式不一样的是，它使用空格来代替网格来显示数据。

## RowBinary

以二进制格式逐行格式化和解析数据。行和值连续列出，没有分隔符。 这种格式比 Native 格式效率低，因为它是基于行的。

整数使用固定长度的小端表示法。 例如，UInt64 使用 8 个字节。 DateTime 被表示为 UInt32 类型的 Unix 时间戳值。 Date 被表示为 UInt16 对象，它的值为 1970-01-01 以来的天数。 字符串表示为 varint 长度（无符号  [LEB128](https://en.wikipedia.org/wiki/LEB128)），后跟字符串的字节数。 FixedString 被简单地表示为一个字节序列。

数组表示为 varint 长度（无符号  [LEB128](https://en.wikipedia.org/wiki/LEB128)），后跟有序的数组元素。

对于  `NULL`  的支持， 一个为 1 或 0 的字节会加在每个  `Nullable`  值前面。如果为 1, 那么该值就是  `NULL`。 如果为 0，则不为  `NULL`。

## RowBinaryWithNamesAndTypes

与`RowBinary`相似，但增加了标题：`LEB128-encoded` 的列数 (N) N  `String` 指定列名称* N 个 字符串String 指定列类型

## Values

在括号中打印每一行。行由逗号分隔。最后一行之后没有逗号。括号内的值也用逗号分隔。数字以十进制格式输出，不含引号。 数组以方括号输出。带有时间的字符串，日期和时间用引号包围输出。转义字符的解析规则与  `TabSeparated`  格式类似。 在格式化过程中，不插入额外的空格，但在解析过程中，空格是被允许并跳过的（除了数组值之外的空格，这是不允许的）。`NULL`  为  `NULL`。

以 Values 格式传递数据时需要转义的最小字符集是：单引号和反斜线。

这是  `INSERT INTO t VALUES ...`  中可以使用的格式，但您也可以将其用于查询结果。

## Vertical

使用指定的列名在单独的行上打印每个值。如果每行都包含大量列，则此格式便于打印一行或几行。

`NULL` 输出为  `ᴺᵁᴸᴸ`。

示例:

```log
SELECT * FROM t_null FORMAT Vertical
Row 1:
──────
x: 1
y: ᴺᵁᴸᴸ
```

行不转义在 `Vertical` 格式:
```log
SELECT 'string with \'quotes\' and \t with some special \n characters' AS test FORMAT Vertical
Row 1:
──────
test: string with 'quotes' and   with some special
 characters
```

该格式仅适用于输出查询结果，但不适用于解析输入（将数据插入到表中）。

## XML

该格式仅适用于输出查询结果，但不适用于解析输入，示例：

```xml
<?xml version='1.0' encoding='UTF-8' ?>
<result>
        <meta>
                <columns>
                        <column>
                                <name>SearchPhrase</name>
                                <type>String</type>
                        </column>
                        <column>
                                <name>count()</name>
                                <type>UInt64</type>
                        </column>
                </columns>
        </meta>
        <data>
                <row>
                        <SearchPhrase></SearchPhrase>
                        <field>8267016</field>
                </row>
                <row>
                        <SearchPhrase>bathroom interior design</SearchPhrase>
                        <field>2166</field>
                </row>
                <row>
                        <SearchPhrase>yandex</SearchPhrase>
                        <field>1655</field>
                </row>
                <row>
                        <SearchPhrase>2014 spring fashion</SearchPhrase>
                        <field>1549</field>
                </row>
                <row>
                        <SearchPhrase>freeform photos</SearchPhrase>
                        <field>1480</field>
                </row>
                <row>
                        <SearchPhrase>angelina jolie</SearchPhrase>
                        <field>1245</field>
                </row>
                <row>
                        <SearchPhrase>omsk</SearchPhrase>
                        <field>1112</field>
                </row>
                <row>
                        <SearchPhrase>photos of dog breeds</SearchPhrase>
                        <field>1091</field>
                </row>
                <row>
                        <SearchPhrase>curtain designs</SearchPhrase>
                        <field>1064</field>
                </row>
                <row>
                        <SearchPhrase>baku</SearchPhrase>
                        <field>1000</field>
                </row>
        </data>
        <rows>10</rows>
        <rows_before_limit_at_least>141137</rows_before_limit_at_least>
</result>
```

如果列名称没有可接受的格式，则仅使用  `field`  作为元素名称。 通常，XML 结构遵循 JSON 结构。 就像 JSON 一样，将无效的 UTF-8 字符都作替换，以便输出文本将包含有效的 UTF-8 字符序列。

在字符串值中，字符  `<`  和  `＆`  被转义为  `<`  和  `＆`。

数组输出为  `<array> <elem> Hello </ elem> <elem> World </ elem> ... </ array>`，元组输出为  `<tuple> <elem> Hello </ elem> <elem> World </ ELEM> ... </tuple>` 。

## CapnProto

Cap'n Proto 是一种二进制消息格式，类似 Protocol Buffers 和 Thriftis，但与 JSON 或 MessagePack 格式不一样。

Cap'n Proto 消息格式是严格类型的，而不是自我描述，这意味着它们不需要外部的描述。这种格式可以实时地应用，并针对每个查询进行缓存。

```bash
cat capnproto_messages.bin | clickhouse-client --query "INSERT INTO test.hits FORMAT CapnProto SETTINGS format_schema='schema:Message'"
```

其中  `schema.capnp`  描述如下：

```log
struct Message {
  SearchPhrase @0 :Text;
  c @1 :Uint64;
}
```

Cap'n Proto 反序列化是很高效的，通常不会增加系统的负载。

### protobuf

Protobuf-是[协议缓冲区](https://developers.google.com/protocol-buffers/)格式。

此格式需要外部格式架构。模式在查询之间进行缓存。ClickHouse 支持 `proto2` 和 `proto3` 语法。支持重复/可选/必填字段。

用法示例：

```sql
SELECT * FROM test.table FORMAT Protobuf SETTINGS format_schema = 'schemafile:MessageType'
```

```log
cat protobuf_messages.bin | clickhouse-client --query "INSERT INTO test.table FORMAT Protobuf SETTINGS format_schema='schemafile:MessageType'"
```

该文件`schemafile.proto`如下所示：

```log
syntax = "proto3";

message MessageType {
  string name = 1;
  string surname = 2;
  uint32 birthDate = 3;
  repeated string phoneNumbers = 4;
};
```

若要查找表列和协议缓冲区的消息类型的字段之间的对应关系，ClickHouse 会对其名称进行比较。此比较不区分大小写，并且字符`_`（下划线）和`.`（点）被认为是相等的。如果列的类型和协议缓冲区的消息的字段不同，则将应用必要的转换。

支持嵌套消息。例如，对于`z`以下消息类型中的字段

```log
message MessageType {
  message XType {
    message YType {
      int32 z;
    };
    repeated YType y;
  };
  XType x;
};
```

ClickHouse 试图找到一个名为列`x.y.z`（或`x_y_z`或`X.y_Z`等）。嵌套消息适合于输入或输出`嵌套数据结构`。

在像这样的 protobuf 模式中定义的默认值

```pb
syntax = "proto2";

message MessageType {
  optional int32 result_per_page = 3 [default = 10];
}
```

不适用；使用 `表默认值` 代替它们。

ClickHouse 以`length-delimited`格式输入和输出 protobuf 消息。这意味着在每条消息之前都应将其长度写为[varint](https://developers.google.com/protocol-buffers/docs/encoding#varints)。另请参阅[如何使用流行语言读取/写入以长度分隔的 protobuf 消息](https://cwiki.apache.org/confluence/display/GEODE/Delimiting+Protobuf+Messages)。

### Format Schema

包含 `Format Schema` 的文件名由设置 `format_schema` 设置。当使用 `Cap'n Proto` 和`Protobuf` 格式之一时，需要设置此设置。`Format Schema`是该文件中的文件名和消息类型的名称的组合，以 `:` 分隔，例如 `schemafile.proto:MessageType` 。如果文件具有格式的标准扩展名(例如，Protobuf的`.proto`)，则可以省略它，在本例中，`Format Schema`类似于`schemafile:MessageType`。

如果以 `interactive mode` 通过 `Client` 输入或输出数据，则 `Format Schema` 中指定的文件名可以包含绝对路径或相对于 `Client` 上的当前目录的路径。如果在批处理模式下使用`Client`，则由于安全原因，到模式的路径必须是相对的。

如果您通过 `HTTP interface` 输入或输出数据，那么在 `Format Schema` 中指定的文件名应该位于服务器配置 `format_schema_path` 中指定的目录中


## JDBC Driver

- **[Official driver](https://github.com/ClickHouse/clickhouse-jdbc)**
- Third-party drivers：
  - [ClickHouse-Native-JDBC](https://github.com/housepower/ClickHouse-Native-JDBC)
  - [clickhouse4j](https://github.com/blynkkk/clickhouse4j)
  
## ODBC Driver

- [Official driver](https://github.com/ClickHouse/clickhouse-odbc)

## C++ Client Library

请参阅[clickhouse-cpp](https://github.com/ClickHouse/clickhouse-cpp)

## Client Libraries from Third-party Developers

免责声明

Yandex 的并**没有**维护下面列出的库，也没有做过广泛的测试，以确保其质量。

- Python
  - [infi.clickhouse_orm](https://github.com/Infinidat/infi.clickhouse_orm)
  - [clickhouse-driver](https://github.com/mymarilyn/clickhouse-driver)
  - [clickhouse-client](https://github.com/yurial/clickhouse-client)
  - [aiochclient](https://github.com/maximdanilchenko/aiochclient)
- PHP
  - [SeasClick](https://github.com/SeasX/SeasClick)
  - [phpClickHouse](https://github.com/smi2/phpClickHouse)
  - [clickhouse-php-client](https://github.com/8bitov/clickhouse-php-client)
  - [clickhouse-client](https://github.com/bozerkins/clickhouse-client)
  - [PhpClickHouseClient](https://github.com/SevaCode/PhpClickHouseClient)
- Go
  - [clickhouse](https://github.com/kshvakov/clickhouse/)
  - [go-clickhouse](https://github.com/roistat/go-clickhouse)
  - [mailrugo-clickhouse](https://github.com/mailru/go-clickhouse)
  - [golang-clickhouse](https://github.com/leprosus/golang-clickhouse)
- NodeJs
  - [clickhouse (NodeJs)](https://github.com/TimonKK/clickhouse)
  - [node-clickhouse](https://github.com/apla/node-clickhouse)
- Perl
  - [perl-DBD-ClickHouse](https://github.com/elcamlost/perl-DBD-ClickHouse)
  - [HTTP-ClickHouse](https://metacpan.org/release/HTTP-ClickHouse)
  - [AnyEvent-ClickHouse](https://metacpan.org/release/AnyEvent-ClickHouse)
- Ruby
  - [clickhouse (Ruby)](https://github.com/archan937/clickhouse)
- R
  - [clickhouse-r](https://github.com/hannesmuehleisen/clickhouse-r)
  - [RClickhouse](https://github.com/IMSMWU/RClickhouse)
- Java
  - [clickhouse-client-java](https://github.com/VirtusAI/clickhouse-client-java)
  - [clickhouse-client](https://github.com/Ecwid/clickhouse-client)
- Scala
  - [clickhouse-scala-client](https://github.com/crobox/clickhouse-scala-client)
- Kotlin
  - [AORM](https://github.com/TanVD/AORM)
- C#
  - [ClickHouse.Ado](https://github.com/killwort/ClickHouse-Net)
  - [ClickHouse.Net](https://github.com/ilyabreev/ClickHouse.Net)
- Elixir
  - [clickhousex](https://github.com/appodeal/clickhousex/)
- Nim
  - [nim-clickhouse](https://github.com/leonardoce/nim-clickhouse)
  
官方没有相关维护除C++ 以外的库， 需要开发人员自己写库

## Integration Libraries from Third-party Developers

免责声明

- Yandex 的并**没有**维持下面列出的工具和库，并没有做任何广泛的测试，以确保其质量。
- 官方没有维护相关库， 需要开发人员自己写库

- 基础架构 Products[¶](https://clickhouse.yandex/docs/en/interfaces/third-party/integrations/#infrastructure-products "Permanent link")
  ***
  - Relational database management systems
    - [MySQL](https://www.mysql.com/)
      - [ProxySQL](https://github.com/sysown/proxysql/wiki/ClickHouse-Support)
      - [clickhouse-mysql-data-reader](https://github.com/Altinity/clickhouse-mysql-data-reader)
      - [horgh-replicator](https://github.com/larsnovikov/horgh-replicator)
    - [PostgreSQL](https://www.postgresql.org/)
      - [clickhousedb_fdw](https://github.com/Percona-Lab/clickhousedb_fdw)
      - [infi.clickhouse_fdw](https://github.com/Infinidat/infi.clickhouse_fdw) (uses [infi.clickhouse_orm](https://github.com/Infinidat/infi.clickhouse_orm))
      - [pg2ch](https://github.com/mkabilov/pg2ch)
      - [clickhouse_fdw](https://github.com/adjust/clickhouse_fdw)
    - [MSSQL](https://en.wikipedia.org/wiki/Microsoft_SQL_Server)
      - [ClickHouseMigrator](https://github.com/zlzforever/ClickHouseMigrator)
  - Message queues
    - [Kafka](https://kafka.apache.org/)
      - [clickhouse_sinker](https://github.com/housepower/clickhouse_sinker) (uses [Go client](https://github.com/kshvakov/clickhouse/))
  - Object storages
    - [S3](https://en.wikipedia.org/wiki/Amazon_S3)
      - [clickhouse-backup](https://github.com/AlexAkulov/clickhouse-backup)
  - Container orchestration
    - [Kubernetes](https://kubernetes.io/)
      - [clickhouse-operator](https://github.com/Altinity/clickhouse-operator)
  - Configuration management
    - [puppet](https://puppet.com/)
      - [innogames/clickhouse](https://forge.puppet.com/innogames/clickhouse)
      - [mfedotov/clickhouse](https://forge.puppet.com/mfedotov/clickhouse)
  - Monitoring
    - [Graphite](https://graphiteapp.org/)
      - [graphouse](https://github.com/yandex/graphouse)
      - [carbon-clickhouse](https://github.com/lomik/carbon-clickhouse)
    - [Grafana](https://grafana.com/)
      - [clickhouse-grafana](https://github.com/Vertamedia/clickhouse-grafana)
    - [Prometheus](https://prometheus.io/)
      - [clickhouse_exporter](https://github.com/f1yegor/clickhouse_exporter)
      - [PromHouse](https://github.com/Percona-Lab/PromHouse)
      - [clickhouse_exporter](https://github.com/hot-wifi/clickhouse_exporter) (uses [Go client](https://github.com/kshvakov/clickhouse/))
    - [Nagios](https://www.nagios.org/)
      - [check_clickhouse](https://github.com/exogroup/check_clickhouse/)
    - [Zabbix](https://www.zabbix.com/)
      - [clickhouse-zabbix-template](https://github.com/Altinity/clickhouse-zabbix-template)
    - [Sematext](https://sematext.com/)
      - [clickhouse integration](https://github.com/sematext/sematext-agent-integrations/tree/master/clickhouse)
  - Logging
    - [rsyslog](https://www.rsyslog.com/)
      - [omclickhouse](https://www.rsyslog.com/doc/master/configuration/modules/omclickhouse.html)
    - [fluentd](https://www.fluentd.org/)
      - [loghouse](https://github.com/flant/loghouse) (for [Kubernetes](https://kubernetes.io/))
    - [logagent](https://www.sematext.com/logagent)
      - [logagent output-plugin-clickhouse](https://sematext.com/docs/logagent/output-plugin-clickhouse/)
  - Geo
    - [MaxMind](https://dev.maxmind.com/geoip/)
      - [clickhouse-maxmind-geoip](https://github.com/AlexeyKupershtokh/clickhouse-maxmind-geoip)
  Programming Language Ecosystems[¶](https://clickhouse.yandex/docs/en/interfaces/third-party/integrations/#programming-language-ecosystems "Permanent link")
  ***
  - Python
    - [SQLAlchemy](https://www.sqlalchemy.org/)
      - [sqlalchemy-clickhouse](https://github.com/cloudflare/sqlalchemy-clickhouse) (uses [infi.clickhouse_orm](https://github.com/Infinidat/infi.clickhouse_orm))
    - [pandas](https://pandas.pydata.org/)
      - [pandahouse](https://github.com/kszucs/pandahouse)
  - R
    - [dplyr](https://db.rstudio.com/dplyr/)
      - [RClickhouse](https://github.com/IMSMWU/RClickhouse) (uses [clickhouse-cpp](https://github.com/artpaul/clickhouse-cpp))
  - Java
    - [Hadoop](http://hadoop.apache.org/)
      - [clickhouse-hdfs-loader](https://github.com/jaykelin/clickhouse-hdfs-loader) (uses [JDBC](https://clickhouse.yandex/docs/en/query_language/table_functions/jdbc/))
  - Scala
    - [Akka](https://akka.io/)
      - [clickhouse-scala-client](https://github.com/crobox/clickhouse-scala-client)
  - C#
    - [ADO.NET](https://docs.microsoft.com/en-us/dotnet/framework/data/adonet/ado-net-overview)
      - [ClickHouse.Ado](https://github.com/killwort/ClickHouse-Net)
      - [ClickHouse.Net](https://github.com/ilyabreev/ClickHouse.Net)
      - [ClickHouse.Net.Migrations](https://github.com/ilyabreev/ClickHouse.Net.Migrations)
  - Elixir
    - [Ecto](https://github.com/elixir-ecto/ecto)
      - [clickhouse_ecto](https://github.com/appodeal/clickhouse_ecto)


## Visual Interfaces from Third-party Developers

### Open-Source

#### Tabix

[Tabix](https://github.com/tabixio/tabix)项目中 ClickHouse 的 Web 界面。

特征：

- 直接从浏览器使用 ClickHouse，而无需安装其他软件。
- 查询编辑器带有语法突出显示。
- 自动完成命令。
- 图形化查询执行工具。
- 配色方案选项。

[Tabix 文档](https://tabix.io/doc/)。

#### HouseOps

[HouseOps](https://github.com/HouseOps/HouseOps)是 OSX，Linux 和 Windows 的 UI / IDE。

特征：

- 具有语法突出显示的查询生成器。在表格或 JSON 视图中查看响应。
- 将查询结果导出为 CSV 或 JSON。
- 带有说明的过程列表。写模式。能够停止（`KILL`）一个进程。
- 数据库图。显示所有表及其列以及其他信息。
- 快速查看列大小。
- 服务器配置。

计划开发以下功能：

- 数据库管理。
- 用户管理。
- 实时数据分析。
- 集群监控。
- 集群管理。
- 监视复制表和 Kafka 表。

#### LightHouse

[LightHouse](https://github.com/VKCOM/lighthouse)是[ClickHouse](https://github.com/VKCOM/lighthouse)的轻量级 Web 界面。

特征：

- 带有过滤和元数据的表列表。
- 具有过滤和排序功能的表格预览。
- 只读查询执行。

#### Redash

[Redash](https://github.com/getredash/redash)是用于数据可视化的平台。

Redash 支持包括 ClickHouse 在内的多个数据源，可以将来自不同数据源的查询结果合并到一个最终数据集中。

特征：

- 强大的查询编辑器。
- 数据库资源管理器。
- 可视化工具，使您可以用不同的形式表示数据。

#### DBeaver

[DBeaver](https://dbeaver.io/)具有 ClickHouse 支持的通用桌面数据库客户端。

特征：

- 使用语法高亮和自动完成功能进行查询开发。
- 带有过滤器和元数据搜索的表列表。
- 表数据预览。
- 全文搜索。

#### clickhouse-CLI

[clickhouse-cli](https://github.com/hatarist/clickhouse-cli)是使用 Python 3 编写的 ClickHouse 的替代命令行客户端。

特点：-自动完成。-查询和数据输出的语法突出显示。-寻呼机支持数据输出。-自定义类似 PostgreSQL 的命令。

#### clickhouse-flamegraph

[clickhouse-flamegraph](https://github.com/Slach/clickhouse-flamegraph)是一种专用工具，可以可视化`system.trace_log`as [flamegraph](http://www.brendangregg.com/flamegraphs.html)。

### Commercial(商业)

#### DataGrip

[DataGrip](https://www.jetbrains.com/datagrip/)是 JetBrains 的数据库 IDE，具有对 ClickHouse 的专门支持。它还嵌入到其他基于 IntelliJ 的工具中：`PyCharm`，`IntelliJ IDEA`，`GoLand`，`PhpStorm` 等。

特征：

- 快速完成代码。
- ClickHouse 语法突出显示。
- 支持特定于 ClickHouse 的功能，例如嵌套列，表引擎。
- 数据编辑器。
- 重构。
- 搜索和导航。

## Proxy Servers from Third-party Developers

### chproxy

[chproxy](https://github.com/Vertamedia/chproxy)是 `ClickHouse` 数据库的 `http proxy` 和`balancer`。

特征：

- 每用户路由和响应缓存。
- 灵活的限制。
- SSL 证书自动更新。

在 Go 中实现。

### KittenHouse

[KittenHouse](https://github.com/VKCOM/kittenhouse)被设计为 `ClickHouse` 和应用程序服务器之间的本地代理，以防在应用程序端无法或不方便地缓冲 `INSERT` 数据。

特征：

- 内存和磁盘上的数据缓冲。
- `Per-table`路由。
- `Load-balancing` 和 `health checking`

在 Go 中实现。

### ClickHouse-Bulk

[ClickHouse-Bulk](https://github.com/nikepan/clickhouse-bulk)是一个简单的 ClickHouse 插入收集器。

特征：

- 分组请求并按阈值或间隔发送。
- 多个远程服务器。
- 基本身份验证。

在 Go 中实现。












