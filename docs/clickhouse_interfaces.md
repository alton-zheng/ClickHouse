# 客户端

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
```xml
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

### TabSeparated [¶](https://clickhouse.yandex/docs/en/single/#tabseparated "Permanent link")

在 TabSeparated 格式中，数据按行写入。每行包含由制表符分隔的值。每个值后面都有一个制表符，但行中的最后一个值除外，后跟一个换行符。到处都有严格的 Unix 换行符。最后一行还必须在末尾包含换行符。值以文本格式编写，不带引号，并且转义了特殊字符。

该格式也可以在名称下使用`TSV`。

该`TabSeparated`格式便于使用自定义程序和脚本处理数据。默认情况下，它在 HTTP 界面和命令行客户端的批处理模式下使用。这种格式还允许在不同的 DBMS 之间传输数据。例如，您可以从 MySQL 获取转储并将其上传到 ClickHouse，反之亦然。

该`TabSeparated`格式支持输出总值（当使用 WITH TOTALS 时）和极值（当'extremes'设置为 1 时）。在这些情况下，总值和极值将在主数据之后输出。主要结果，总计值和极值用空行彼此分隔。例：

SELECT EventDate ， count （） AS c FROM test 。点击 GROUP BY EVENTDATE 与 TOTALS ORDER BY EVENTDATE FORMAT TabSeparated ``

2014-03-17 1406958
2014-03-18 1383658
2014-03-19 1405797
2014-03-20 1353623
2014-03-21 1245779
2014-03-22 1031592
2014-03-23 1046491

0000-00-00 8873898

2014-03-17 1031592
2014-03-23 1406958

#### 数据格式化[¶](https://clickhouse.yandex/docs/en/single/#data-formatting "Permanent link")

整数以十进制形式编写。数字开头可以包含一个额外的“ +”字符（解析时忽略，格式化时不记录）。非负数不能包含负号。读取时，允许将空字符串解析为零，或者（对于有符号类型）解析仅由减号组成的字符串为零。不适用于相应数据类型的数字可以解析为其他数字，而不会出现错误消息。

浮点数以十进制形式编写。点用作小数点分隔符。支持指数项，如'inf'，'+ inf'，'-inf'和'nan'。浮点数的输入可以以小数点开头或结尾。在格式化期间，浮点数可能会失去准确性。在解析过程中，并非严格要求读取最近的机器可表示数字。

日期以 YYYY-MM-DD 格式写入，并以相同的格式进行解析，但使用任何字符作为分隔符。带时间的日期以 YYYY-MM-DD hh：mm：ss 的格式写入，并以相同的格式进行解析，但使用任何字符作为分隔符。这一切都发生在客户端或服务器启动时的系统时区（取决于哪种格式的数据）。对于带有日期的日期，未指定夏令时。因此，如果转储在夏令时期间有时间，则转储不会明确匹配数据，并且解析将选择两次。在读取操作期间，可以使用自然溢出或不正确的消息将错误的日期和日期与时间解析为自然的日期或时间。

作为例外，如果时间戳由正好由 10 个十进制数字组成，则还支持 Unix 时间戳格式的带时间的日期解析。结果与时区无关。格式 YYYY-MM-DD hh：mm：ss 和 NNNNNNNNNN 自动区分。

字符串以反斜杠转义的特殊字符输出。下面的转义序列用于输出：`\b`，`\f`，`\r`，`\n`，`\t`，`\0`，`\'`，`\\`。解析还支持序列`\a`，`\v`和和`\xHH`（十六进制转义序列）以及任何`\c`序列，其中`c`任何字符（这些序列都转换为`c`）。因此，读取数据支持可以将换行符编写为`\n`或或换行符的格式`\`。例如，`Hello world`可以用以下任何一种形式来解析在单词之间用换行符代替空格的字符串：

您好\ nworld

你好\
世界

支持第二种变体，因为 MySQL 在编写制表符分隔的转储时会使用它。

以 TabSeparated 格式传递数据时，您需要转义的最少字符集：制表符，换行（LF）和反斜杠。

只有一小部分符号被转义。您可以轻松地发现一个字符串值，您的终端将在输出中破坏该字符串值。

数组被写为方括号中的逗号分隔值列表。数组中的数字项通常照常设置，但日期，带时间的日期和字符串以单引号括起来，并具有与上述相同的转义规则。

[NULL](https://clickhouse.yandex/docs/en/single/#../query_language/syntax/)的格式为`\N`。

### TabSeparatedRaw [¶](https://clickhouse.yandex/docs/en/single/#tabseparatedraw "Permanent link")

与`TabSeparated`格式不同的是，写入行时没有转义。此格式仅适用于输出查询结果，而不适用于解析（检索要插入表中的数据）。

该格式也可以在名称下使用`TSVRaw`。

### TabSeparatedWithNames [¶](https://clickhouse.yandex/docs/en/single/#tabseparatedwithnames "Permanent link")

与`TabSeparated`格式不同的是，列名写在第一行中。在解析期间，第一行将被完全忽略。您不能使用列名称来确定其位置或检查其正确性。（将来可能会添加对解析标题行的支持。）

该格式也可以在名称下使用`TSVWithNames`。

### TabSeparatedWithNamesAndTypes [¶](https://clickhouse.yandex/docs/en/single/#tabseparatedwithnamesandtypes "Permanent link")

与`TabSeparated`格式的不同之处在于，列名被写入第一行，而列类型被写入第二行。在解析期间，第一行和第二行被完全忽略。

该格式也可以在名称下使用`TSVWithNamesAndTypes`。

### 模板[¶](https://clickhouse.yandex/docs/en/single/#format-template "Permanent link")

此格式允许为带有指定转义规则的值的占位符指定自定义格式字符串。

它使用设置`format_template_resultset`，`format_template_row`，`format_template_rows_between_delimiter`和其它格式的一些设置（例如`output_format_json_quote_64bit_integers`使用时`JSON`逃逸，见下文）

设置`format_template_row`指定文件的路径，其中包含具有以下语法的行的格式字符串：

`delimiter_1${column_1:serializeAs_1}delimiter_2${column_2:serializeAs_2} ... delimiter_N`，

where `delimiter_i`是值之间的分隔符（`$`symbol 可以转义为`$$`）， `column_i`是要选择或插入其值的列的名称或索引（如果为空，则将跳过`serializeAs_i`该列），  是列值的转义规则。支持以下转义规则：

- `CSV`，`JSON`，`XML`（相同的名称的格式类似）
- `Escaped`（类似于`TSV`）
- `Quoted`（类似于`Values`）
- `Raw`（不转义，类似于`TSVRaw`）
- `None` （没有转义规则，请参阅进一步）

如果省略转义规则，`None`则将使用。`XML`并且`Raw`仅适用于输出。

因此，对于以下格式字符串：

`搜索短语：$ { SearchPhrase ：引用}，计数：$ { c ：转义的}，广告价格：$$$ {price：JSON}；`

的值`SearchPhrase`，`c`和`price`列，其被转义为`Quoted`，`Escaped`和`JSON`之间将被打印（供选择）或将被预期（嵌件）`Search phrase:`，`, count:`，`, ad price: $`和`;`分别分隔符。例如：

`Search phrase: 'bathroom interior design', count: 2166, ad price: $3;`

该`format_template_rows_between_delimiter`设置指定行之间的定界符，除最后一行（`\n`默认情况下）外，每行之后都会打印（或预期）定界符

设置`format_template_resultset`指定文件的路径，其中包含结果集的格式字符串。结果集的格式字符串与行的格式字符串具有相同的语法，并允许指定前缀，后缀和打印一些其他信息的方式。它包含以下占位符而不是列名：

- `data`是数据`format_template_row`格式为的行，以分隔`format_template_rows_between_delimiter`。该占位符必须是格式字符串中的第一个占位符。
- `totals`是具有`format_template_row`格式的总值的行（使用 WITH TOTALS 时）
- `min`是具有最小值`format_template_row`格式的行（当极值设置为 1 时）
- `max`是具有最大值`format_template_row`格式的行（当极值设置为 1 时）
- `rows`  是输出行的总数
- `rows_before_limit`是没有 LIMIT 的最少行数。仅当查询包含 LIMIT 时才输出。如果查询包含 GROUP BY，则 rows_before_limit_at_least 是没有 LIMIT 的确切行数。
- `time`  是请求执行时间，以秒为单位
- `rows_read`  是已读取的行数
- `bytes_read`  是已读取的字节数（未压缩）

占位符`data`，`totals`，`min`而`max`不能有逃避指定的规则（或`None`必须明确指定）。其余的占位符可以指定任何转义规则。如果`format_template_resultset`设置为空字符串，`${data}`则用作默认值。对于插入查询，格式允许跳过某些列或某些字段（如果带有前缀或后缀）（请参见示例）。

选择示例：

SELECT SearchPhrase ， 计数（） AS c FROM test 。命中 GROUP BY SearchPhrase ORDER BY c DESC LIMIT 5 格式 模板 设置
format_template_resultset = '/some/path/resultset.format' ， format_template_row = '/some/path/row.format' ， format_template_rows_between_delimiter = '\ n'

`/some/path/resultset.format`：

<！DOCTYPE HTML>

<html> <head> <title>搜索短语</ title> </ head>
 <身体>
  <table border =“ 1”> <caption>搜索词组</ caption>
    <tr> <th>搜索词组</ th> <th>计数</ th> </ tr>
    $ {data}
  </ table>
  <table border =“ 1”> <caption> Max </ caption>
    $ {max}
  </ table>
  <b>在$ {time：XML}秒内处理了$ {rows_read：XML}行</ b>
 </ body>
</ html>

`/some/path/row.format`：

<tr> <td> $ {0：XML} </ td> <td> $ {1：XML} </ td> </ tr>

结果：

<！DOCTYPE HTML>
< html \> < 头\> < 标题>搜索短语</ 标题\> </ 头\>
< 正文\>
< 表 边框= “ 1” \> < 标题>搜索短语</ 标题\>
< tr \> < th >搜索短语</ th \> < th > Count </ th > </ tr >
< tr \> < td \> </ td \> < td > 8267016 </ td \> </ tr \>
< tr \> < td >浴室室内设计</ td \> < td > 2166 </ td \> </ tr \>
< tr \> < td > yandex </ td \> < td > 1655 </ td \> </tr \>
< tr \> < td > 2014 春季时装</ td \> < td > 1549 </ td \> </ tr \>
< tr \> < td >自由形式照片</ td \> < td > 1480 </ td \> </ tr \>
</ table \>
< table border = “ 1” \> <字幕>最大值</ caption \>
< tr \> < td \> </ td \> < td > 8873898 </ td \> </ tr \>
</ table \>
< b >在 0.1569913 秒内处理了 3095973 行</ b \>
</ body \>
</ html >

插入示例：

一些头
页面浏览量：5，用户 ID：4324182021466249494，无用字段：你好，持续时间：146，签名：-1
页面浏览量：6，用户 ID：4324182021466249494，无用字段：世界，持续时间：185，签名：1
总行数：2

INSERT INTO UserActivity FORMAT 模板 设置
format_template_resultset = '/some/path/resultset.format' ， format_template_row = '/some/path/row.format'

`/some/path/resultset.format`：

某些标头\ n $ {数据} \ n总行数：$ {：CSV} \ n

`/some/path/row.format`：

页面浏览量：$ {PageViews：CSV}，用户ID：$ {UserID：CSV}，无用字段：$ {：CSV}，持续时间：$ {Duration：CSV}，符号：\$ {Sign：CSV}

`PageViews`，`UserID`，`Duration`和`Sign`内占位符表中的列的名称。`Useless field`行后`\nTotal rows:`和后缀后的值将被忽略。输入数据中的所有定界符必须严格等于指定格式字符串中的定界符。

### TemplateIgnoreSpaces [¶](https://clickhouse.yandex/docs/en/single/#templateignorespaces "Permanent link")

此格式仅适用于输入。与相似`Template`，但在输入流中的定界符和值之间跳过空格字符。但是，如果格式字符串包含空格字符，则将在输入流中使用这些字符。还允许指定空的占位符（`${}`或`${:None}`），以将某些分隔符拆分为单独的部分，以忽略它们之间的空格。此类占位符仅用于跳过空格字符。`JSON`如果列的值在所有行中都具有相同的顺序，则可以使用这种格式进行读取。例如，以下请求可用于从[JSON](https://clickhouse.yandex/docs/en/single/#json)格式的输出示例插入数据：

INSERT INTO TABLE_NAME FORMAT TemplateIgnoreSpaces 设置
format_template_resultset = '/some/path/resultset.format' ， format_template_row = '/some/path/row.format' ， format_template_rows_between_delimiter = ''

`/some/path/resultset.format`：

{$ {}“元” $ {}：$ {：JSON}，$ {}“数据” $ {}：$ {} \[$ {数据}\] $ {}，$ {}“总计” $ {}： $ {：JSON}，$ {}“极端” $ {}：$ {：JSON}，$ {}“行” $ {}：$ {：JSON}，$ {}“ rows_before_limit_at_least” $ {}：$ { ：JSON} \$ {}}

`/some/path/row.format`：

{$ {}“ SearchPhrase” $ {}：$ {} $ {phrase：JSON} $ {}，$ {}“ c” $ {}：$ {} $ {cnt：JSON} $ {}}

### TSKV [¶](https://clickhouse.yandex/docs/en/single/#tskv "Permanent link")

与 TabSeparated 相似，但是以 name = value 格式输出值。名称以与 TabSeparated 格式相同的方式转义，并且=符号也转义。

SearchPhrase = count（）= 8267016
SearchPhrase =浴室室内设计 count（）= 2166
SearchPhrase = yandex count（）= 1655
SearchPhrase = 2014 春季时尚 count（）= 1549
SearchPhrase =自由格式的图片数量（）= 1480
SearchPhrase = angelina jolie count（）= 1245
SearchPhrase = omsk count（）= 1112
SearchPhrase =狗品种的照片 count（）= 1091
SearchPhrase =窗帘设计 count（）= 1064
SearchPhrase = baku count（）= 1000

[NULL](https://clickhouse.yandex/docs/en/single/#../query_language/syntax/)的格式为`\N`。

选择 \* FROM t_null FORMAT TSKV

x = 1 y = \ N

当有许多小列时，此格式无效，并且通常没有理由使用它。不过，就效率而言，它并不比 JSONEachRow 差。

此格式同时支持数据输出和解析。对于解析，不同列的值支持任何顺序。可以省略某些值-将其视为等于其默认值是可以接受的。在这种情况下，零和空白行用作默认值。默认情况下不支持可以在表中指定的复杂值。

解析允许存在`tskv`没有等号或值的附加字段。该字段被忽略。

### CSV [¶](https://clickhouse.yandex/docs/en/single/#csv "Permanent link")

逗号分隔值格式（[RFC](https://tools.ietf.org/html/rfc4180)）。

格式化时，行用双引号引起来。字符串中的双引号作为一行中的两个双引号输出。没有其他用于转义字符的规则。日期和日期时间用双引号引起来。数字输出时不带引号。值由一个分隔符，这是分开的`,`默认。分隔符字符在设置[format_csv_delimiter 中](https://clickhouse.yandex/docs/en/single/#settings-format_csv_delimiter)定义。行使用 Unix 换行（LF）隔开。数组以 CSV 序列化的方式如下：首先将数组序列化为 TabSeparated 格式的字符串，然后将结果字符串以双引号输出到 CSV。CSV 格式的元组被序列化为单独的列（也就是说，它们在元组中的嵌套丢失了）。

\$ clickhouse-client --format_csv_delimiter = “ |” --query = “INSERT INTO test.csv FORMAT CSV” <data.csv

\*默认情况下，分隔符为`,`。有关更多信息，请参见[format_csv_delimiter](https://clickhouse.yandex/docs/en/single/#settings-format_csv_delimiter)设置。

解析时，可以使用或不使用引号来解析所有值。两个双人和单引号的支持。行也可以不带引号排列。在这种情况下，它们被解析到分隔符或换行（CR 或 LF）。违反 RFC 的，不带引号解析行的时候，在开头和结尾的空格和 tab 被忽略。对于换行符，都支持 Unix（LF），Windows（CR LF）和 Mac OS Classic（CR LF）类型。

如果   启用了[input_format_defaults_for_omitted_fields](https://clickhouse.yandex/docs/en/single/#session_settings-input_format_defaults_for_omitted_fields)，则[空无](https://clickhouse.yandex/docs/en/single/#session_settings-input_format_defaults_for_omitted_fields)引号的输入值将被相应列的默认值替换  。

`NULL`格式为`\N`或`NULL`或空的未加引号的字符串（请参阅设置[input_format_csv_unquoted_null_literal_as_null](https://clickhouse.yandex/docs/en/single/#settings-input_format_csv_unquoted_null_literal_as_null)和[input_format_defaults_for_omitted_fields](https://clickhouse.yandex/docs/en/single/#session_settings-input_format_defaults_for_omitted_fields)）。

CSV 格式与一样支持总计和极值的输出`TabSeparated`。

### CSVWithNames [¶](https://clickhouse.yandex/docs/en/single/#csvwithnames "Permanent link")

还打印标题行，类似于`TabSeparatedWithNames`。

### CustomSeparated [¶](https://clickhouse.yandex/docs/en/single/#format-customseparated "Permanent link")

类似[的模板](https://clickhouse.yandex/docs/en/single/#format-template)，但它打印或读取所有列，并使用从设置逃逸规则`format_custom_escaping_rule`从设置和分隔符`format_custom_field_delimiter`，`format_custom_row_before_delimiter`，`format_custom_row_after_delimiter`，`format_custom_row_between_delimiter`，`format_custom_result_before_delimiter`和`format_custom_result_after_delimiter`，而不是从格式字符串。还有一种`CustomSeparatedIgnoreSpaces`格式，类似于`TemplateIgnoreSpaces`。

### JSON [¶](https://clickhouse.yandex/docs/en/single/#json "Permanent link")

以 JSON 格式输出数据。除了数据表外，它还输出列名和类型，以及一些其他信息：输出行的总数，如果没有 LIMIT 则可以输出的行数。例：

SELECT SearchPhrase ， 计数（） AS c FROM test 。命中 GROUP BY SearchPhrase 与 TOTALS ORDER BY c DESC LIMIT 5 FORMAT JSON

{
“ meta” ：
\[
{
“ name” ： “ SearchPhrase” ，
“ type” ： “ String”
}，
{
“ name” ： “ c” ，
“ type” ： “ UInt64”
}
\]，

        “数据” ：
        \[
                {
                        “ SearchPhrase” ： “” ，
                        “ c” ： “ 8267016”
                }，
                {
                        “ SearchPhrase” ： “浴室室内设计” ，
                        “ c” ： “ 2166”
                }，
                {
                        “ SearchPhrase” ： “ yandex” ，
                        “ c” ： “ 1655”
                }，
                {
                        “ SearchPhrase” ： “ 2014春季时装” ，
                        “ c” ： “ 1549”
                }，
                {
                        “ SearchPhrase”： “自由格式的照片” ，
                        “ c” ： “ 1480”
                }
        \]，

        “总计” ：
        {
                “ SearchPhrase” ： “” ，
                “ c” ： “ 8873898”
        }，

        “ extremes” ：
        {
                “ min” ：
                {
                        “ SearchPhrase” ： “” ，
                        “ c” ： “ 1480”
                }，
                “ max” ：
                {
                        “ SearchPhrase” ： “” ，
                        “ c” ： “ 8267016”
                }
        }，

        “行” ： 5 ，

        “ rows\_before\_limit\_at\_least” ： 141137

}

JSON 与 JavaScript 兼容。为了确保这一点，还额外转义了一些字符：将斜杠`/`转义为`\/`;。替代的换行符`U+2028`和`U+2029`会破坏某些浏览器，因此会转义为`\uXXXX`。ASCII 控制字符被转义：退格，进纸，换行，回车，和水平制表与替换`\b`，`\f`，`\n`，`\r`，`\t`，以及在 00-1F 范围使用剩余的字节`\uXXXX`序列。无效的 UTF-8 序列更改为替换字符``，因此输出文本将包含有效的 UTF-8 序列。为了与 JavaScript 兼容，默认情况下，Int64 和 UInt64 整数用双引号引起来。要删除引号，可以设置配置参数[output_format_json_quote_64bit_integers 设置](https://clickhouse.yandex/docs/en/single/#session_settings-output_format_json_quote_64bit_integers)为 0。

`rows` –输出的总行数。

`rows_before_limit_at_least`没有 LIMIT 的最小行数。仅当查询包含 LIMIT 时才输出。如果查询包含 GROUP BY，则 rows_before_limit_at_least 是没有 LIMIT 的确切行数。

`totals` –总值（使用 WITH TOTALS 时）。

`extremes` –极值（当极值设置为 1 时）。

此格式仅适用于输出查询结果，而不适用于解析（检索要插入表中的数据）。

ClickHouse 支持[NULL](https://clickhouse.yandex/docs/en/single/#../query_language/syntax/)，其显示为`null`JSON 输出。

另请参见[JSONEachRow](https://clickhouse.yandex/docs/en/single/#jsoneachrow)格式。

### JSONCompact [¶](https://clickhouse.yandex/docs/en/single/#jsoncompact "Permanent link")

与 JSON 的区别仅在于数据行以数组而不是对象输出。

例：

{
“ meta” ：
\[
{
“ name” ： “ SearchPhrase” ，
“ type” ： “ String”
}，
{
“ name” ： “ c” ，
“ type” ： “ UInt64”
}
\]，

        “数据” ：
        \[
                \[ “” ， “ 8267016” \]，
                \[ “浴室室内设计” ， “ 2166” \]，
                \[ “ yandex” ， “ 1655” \]，
                \[ “ 2014年春季时尚趋势” ， “ 1549” \]，
                \[ “自由格式的照片” ， “ 1480” \]
        \]，

        “总计” ： \[ “” ，“ 8873898” \]，

        “ extremes” ：
        {
                “ min” ： \[ “” ，“ 1480” \]，
                “ max” ： \[ “” ，“ 8267016” \]
        }，

        “行” ： 5 ，

        “ rows\_before\_limit\_at\_least” ： 141137

}

此格式仅适用于输出查询结果，而不适用于解析（检索要插入表中的数据）。另请参阅`JSONEachRow`格式。

### JSONEachRow [¶](https://clickhouse.yandex/docs/en/single/#jsoneachrow "Permanent link")

使用这种格式时，ClickHouse 将行输出为分隔的，以换行符分隔的 JSON 对象，但整个数据都是无效的 JSON。

{ “ SearchPhrase” ：“窗帘设计” ，“ count（）” ：“ 1064” }
{ “ SearchPhrase” ：“ baku” ，“ count（）” ：“ 1000” }
{ “ SearchPhrase” ：“” ，“ count（ ）“ ：” 8267016“ }

插入数据时，应为每行提供一个单独的 JSON 对象。

#### 插入数据[¶](https://clickhouse.yandex/docs/en/single/#inserting-data "Permanent link")

INSERT INTO UserActivity FORMAT JSONEachRow { “浏览量” ：5 ， “用户 ID” ：“4324182021466249494” ，“ 持续时间” ：146 ，“注册” ：\- 1 } { “用户 ID” ：“4324182021466249494” ，“浏览量” ：6 ，“持续时间“ ：185 ，” Sign“ ：1 }

ClickHouse 允许：

- 对象中键-值对的任何顺序。
- 省略一些值。

ClickHouse 会忽略对象后的元素和逗号之间的空格。您可以在一行中传递所有对象。您不必用换行符将它们分开。

**遗漏值处理**

ClickHouse 用相应[数据类型](https://clickhouse.yandex/docs/en/single/#../data_types/)的默认值替换省略的值。

如果`DEFAULT expr`指定为，则 ClickHouse 根据[input_format_defaults_for_omitted_fields](https://clickhouse.yandex/docs/en/single/#session_settings-input_format_defaults_for_omitted_fields)设置使用不同的替换规则。

请考虑下表：

CREATE TABLE IF NOT EXISTS example_table
（
X UInt32 的，
一个 DEFAULT X \* 2
） ENGINE = 内存;

- 如果`input_format_defaults_for_omitted_fields = 0`，则对默认值`x`和`a`平等`0`（作为默认值`UInt32`的数据类型）。
- 如果为`input_format_defaults_for_omitted_fields = 1`，则默认值`x`等于`0`，但默认值`a`等于`x * 2`。

警告

与`insert_sample_with_metadata = 1`相比，与插入数据时，ClickHouse 消耗更多的计算资源`insert_sample_with_metadata = 0`。

#### 选择数据[¶](https://clickhouse.yandex/docs/en/single/#selecting-data "Permanent link")

以该`UserActivity`表为例：

ID──────────────UserID─┬─PageViews─┬─ 持续时间 ─┬─Sign─┐
│4324182021466249494│5│146│-1│
│4324182021466249494│6│185│1│
└──────────────────┴──────┴ ──┘

查询`SELECT * FROM UserActivity FORMAT JSONEachRow`返回：

{“ UserID”：“ 4324182021466249494”，“ PageViews”：5，“ Duration”：146，“ Sign”：-1}
{“ UserID”：“ 4324182021466249494”，“ PageViews”：6，“ Duration”：185，“ Sign”：1}

与[JSON](https://clickhouse.yandex/docs/en/single/#json)格式不同，不会替换无效的 UTF-8 序列。值的转义与相同`JSON`。

注意

字符串中可以输出任何字节集。`JSONEachRow`如果您确定可以将表中的数据格式化为 JSON 而不丢失任何信息，请使用格式。

#### 嵌套结构的使用[¶](https://clickhouse.yandex/docs/en/single/#jsoneachrow-nested "Permanent link")

如果您的表具有[嵌套的](https://clickhouse.yandex/docs/en/single/#../data_types/nested_data_structures/nested/)数据类型列，则可以插入具有相同结构的 JSON 数据。使用[input_format_import_nested_json](https://clickhouse.yandex/docs/en/single/#settings-input_format_import_nested_json)设置启用此功能。

例如，考虑下表：

创建 表 json_each_row_nested （n 嵌套 （s String ， i Int32 ） ） ENGINE = 内存

正如可以在看到`Nested`数据类型描述，ClickHouse 对待嵌套结构的每个组件作为一个单独的柱（`n.s`和`n.i`我们的表）。您可以通过以下方式插入数据：

INSERT INTO json_each_row_nested FORMAT JSONEachRow { “NS” ： \[ “ABC” ， “DEF” \]， “NI” ： \[ 1 ， 23 \] }

要将数据作为分层 JSON 对象插入，请设置[input_format_import_nested_json = 1](https://clickhouse.yandex/docs/en/single/#settings-input_format_import_nested_json)。

{
的“n” ： {
“S” ： \[ “ABC” ， “DEF” \]，
“I” ： \[ 1 ， 23 \]
}
}

没有此设置，ClickHouse 会引发异常。

SELECT 名称， 值 FROM 系统。设置 WHERE 名称 = 'input_format_import_nested_json'

─ 名称 ────────────────────┬─ 值 ─┐
│input_format_import_nested_json│0│
──────────────────┘

INSERT INTO json_each_row_nested FORMAT JSONEachRow { 的“n” ： { “S” ： \[ “ABC” ， “DEF” \]， “I” ： \[ 1 ， 23 \] }}

代码：117。DB :: Exception：解析 JSONEachRow 格式时发现未知字段：n ：（在第 1 行）

SET input_format_import_nested_json = 1 个
INSERT INTO json_each_row_nested FORMAT JSONEachRow { 的“n” ： { “S” ： \[ “ABC” ， “DEF” \]， “I” ： \[ 1 ， 23 \] }}
SELECT \* FROM json_each_row_nested

ns─ns──────────┬
│\['abc'，'def'\]│\[1,23\]│
└────────────┴

### 本机[¶](https://clickhouse.yandex/docs/en/single/#native "Permanent link")

最有效的格式。数据由块以二进制格式写入和读取。对于每个块，此块中的行数，列数，列名和类型以及列的部分依次记录。换句话说，这种格式是“ columnar” –不会将列转换为行。这是本机界面中用于服务器之间交互，使用命令行客户端以及 C ++客户端的格式。

您可以使用这种格式快速生成只能由 ClickHouse DBMS 读取的转储。自己使用这种格式没有任何意义。

### 空[¶](https://clickhouse.yandex/docs/en/single/#null "Permanent link")

什么也没有输出。但是，查询被处理，并且在使用命令行客户端时，数据被传输到客户端。这用于测试，包括生产力测试。显然，此格式仅适用于输出，不适用于解析。

### 漂亮[¶](https://clickhouse.yandex/docs/en/single/#pretty "Permanent link")

将数据输出为 Unicode 艺术表，也使用 ANSI 转义序列在终端中设置颜色。将绘制表格的完整网格，并且每行在终端中占据两行。每个结果块都作为单独的表输出。这是必需的，以便可以在不缓冲结果的情况下输出块（为了预先计算所有值的可见宽度，必须进行缓冲）。

[NULL](https://clickhouse.yandex/docs/en/single/#../query_language/syntax/)输出为`ᴺᵁᴸᴸ`。

示例（以[PrettyCompact](https://clickhouse.yandex/docs/en/single/#prettycompact)格式显示）：

选择 \* 从 t_null

┌─x─┬────y─┐
│1│ᴺᵁᴸᴸ│
└───┴──────┘

行不会以 Pretty \*格式转义。显示了[PrettyCompact](https://clickhouse.yandex/docs/en/single/#prettycompact)格式的[示例](https://clickhouse.yandex/docs/en/single/#prettycompact)：

SELECT '带\\的字符串引用\ '和\ t 字符' AS Escaping_test

┌─ 转义测试 ────────────────┐
│ 带有“引号”和字符的字符串 │
┘────────────────┘

为避免向终端转储过多数据，仅打印前 10,000 行。如果行数大于或等于 10,000，则显示消息“显示的前 10 000”。此格式仅适用于输出查询结果，而不适用于解析（检索要插入表中的数据）。

Pretty 格式支持输出总计值（使用 WITH TOTALS 时）和极值（当'extremes'设置为 1 时）。在这些情况下，总值和极值在主数据之后输出在单独的表中。示例（以[PrettyCompact](https://clickhouse.yandex/docs/en/single/#prettycompact)格式显示）：

SELECT EventDate ， count （） AS c FROM test 。点击 GROUP BY EVENTDATE 与 彩民 ORDER BY EVENTDATE FORMAT PrettyCompact

┌──EventDate─┬───────c─┐
│2014-03-17│1406958│
│2014 年 3 月 18 日 │1383658│
│2014-03-19│1405797│
│2014-03-20│1353623│
│2014-03-21│1245779│
│2014-03-22│1031592│
│2014-03-23│1046491│
──────────┴

总计：
┌──EventDate─┬───────c─┐
│ 注册 │8873898│
──────────┴

极端：
┌──EventDate─┬───────c─┐
│2014-03-17│1031592│
│2014-03-23│1406958│
──────────┴

### PrettyCompact [¶](https://clickhouse.yandex/docs/en/single/#prettycompact "Permanent link")

与[Pretty 的](https://clickhouse.yandex/docs/en/single/#pretty)不同之处在于，网格是在行之间绘制的，结果更加紧凑。默认情况下，此格式在交互式模式的命令行客户端中使用。

### PrettyCompactMonoBlock [¶](https://clickhouse.yandex/docs/en/single/#prettycompactmonoblock "Permanent link")

与[PrettyCompact 的](https://clickhouse.yandex/docs/en/single/#prettycompact)不同之处在于，最多缓冲 10,000 行，然后将其作为单个表而不是按块输出。

### PrettyNoEscapes [¶](https://clickhouse.yandex/docs/en/single/#prettynoescapes "Permanent link")

与 Pretty 的不同之处在于，未使用 ANSI 转义序列。这对于在浏览器中显示此格式以及使用“监视”命令行实用程序是必要的。

例：

\$ watch -n1 “ clickhouse-client --query ='SELECT 事件，值 FROM system.events FORMAT PrettyCompactNoEscapes'”

您可以使用 HTTP 接口在浏览器中显示。

#### PrettyCompactNoEscapes [¶](https://clickhouse.yandex/docs/en/single/#prettycompactnoescapes "Permanent link")

与之前的设置相同。

#### PrettySpaceNoEscapes [¶](https://clickhouse.yandex/docs/en/single/#prettyspacenoescapes "Permanent link")

与之前的设置相同。

### PrettySpace [¶](https://clickhouse.yandex/docs/en/single/#prettyspace "Permanent link")

与[PrettyCompact 的](https://clickhouse.yandex/docs/en/single/#prettycompact)不同之处[在于](https://clickhouse.yandex/docs/en/single/#prettycompact)，使用空白（空格字符）代替了网格。

### RowBinary [¶](https://clickhouse.yandex/docs/en/single/#rowbinary "Permanent link")

以二进制格式逐行格式化和解析数据。连续列出行和值，不带分隔符。由于此格式是基于行的，因此其效率比本机格式低。

整数使用定长的 Little Endian 表示形式。例如，UInt64 使用 8 个字节。DateTime 表示为 UInt32，其中包含 Unix 时间戳作为值。日期用一个 UInt16 对象表示，该对象包含自 1970-01-01 以来的天数作为值。字符串表示为 varint 长度（无符号[LEB128](https://en.wikipedia.org/wiki/LEB128)），后跟字符串的字节。FixedString 简单地表示为字节序列。

数组表示为 varint 长度（无符号[LEB128](https://en.wikipedia.org/wiki/LEB128)），后跟数组的连续元素。

为了支持[NULL](https://clickhouse.yandex/docs/en/single/#null-literal)，在每个[Nullable](https://clickhouse.yandex/docs/en/single/#../data_types/nullable/)值之前添加一个包含 1 或 0 的附加字节。如果为 1，则值为，`NULL`并且该字节被解释为单独的值。如果为 0，则字节后的值不是`NULL`。

### RowBinaryWithNamesAndTypes [¶](https://clickhouse.yandex/docs/en/single/#rowbinarywithnamesandtypes "Permanent link")

与[RowBinary](https://clickhouse.yandex/docs/en/single/#rowbinary)相似，但增加了标题：

- [LEB128](https://en.wikipedia.org/wiki/LEB128)编码的列数（N）
- N `String`s 指定列名
- N `String`s 个指定列类型

### 值[¶](https://clickhouse.yandex/docs/en/single/#data-format-values "Permanent link")

在方括号中打印每一行。行用逗号分隔。最后一行之后没有逗号。括号内的值也用逗号分隔。数字以不带引号的十进制格式输出。数组在方括号中输出。字符串，日期和带时间的日期以引号输出。转义规则和解析与[TabSeparated](https://clickhouse.yandex/docs/en/single/#tabseparated)格式相似。在格式化期间，不会插入多余的空格，但是在解析期间，会允许和跳过它们（数组值内部的空格除外，这是不允许的）。[NULL](https://clickhouse.yandex/docs/en/single/#../query_language/syntax/)表示为`NULL`。

以 Values 格式传递数据时需要转义的最小字符集：单引号和反斜杠。

这是在中使用的格式`INSERT INTO t VALUES ...`，但是您也可以使用它来格式化查询结果的格式。

另请参阅：[input_format_values_interpret_expressions](https://clickhouse.yandex/docs/en/single/#settings-input_format_values_interpret_expressions)和[input_format_values_deduce_templates_of_expressions](https://clickhouse.yandex/docs/en/single/#settings-input_format_values_deduce_templates_of_expressions)设置。

### 垂直[¶](https://clickhouse.yandex/docs/en/single/#vertical "Permanent link")

在指定的列名称的单独行上打印每个值。如果每行包含大量列，则此格式很方便仅打印一行或几行。

[NULL](https://clickhouse.yandex/docs/en/single/#../query_language/syntax/)输出为`ᴺᵁᴸᴸ`。

例：

SELECT \* FROM t_null 格式 垂直

第 1 行：
──────
x：1
y：ᴺᵁᴸᴸ

行不能以垂直格式进行转义：

SELECT '带有\\'的字符串用引号\ '和\ t 加上一些特殊的\ n 字符' AS 测试 格式 垂直

第 1 行：
──────
测试：带有“引号”并带有一些特殊字符的字符串
人物

此格式仅适用于输出查询结果，而不适用于解析（检索要插入表中的数据）。

### XML [¶](https://clickhouse.yandex/docs/en/single/#xml "Permanent link")

XML 格式仅适用于输出，不适用于解析。例：

<？xml version ='
1.0'encoding ='UTF-8'？> <结果\>
<元\>
<列\>
<列\>
<名称> SearchPhrase </ name>
<type>字符串</ type>
</ column>
<列\>
<名称> count（）</名称\>
<类型> UInt64 </类型\>
</列\>
</列\>
</ meta>
<数据\>
<行\>
<SearchPhrase> </ SearchPhrase>
<field> 8267016 < / field>
</ row>
<row>
<SearchPhrase>浴室室内设计</ SearchPhrase>
<field> 2166 </ field>
</ row>
<row>
<SearchPhrase> yandex </ SearchPhrase>
<field> 1655 </ field>
</ row>
<row>
<SearchPhrase> 2014 春季时装</ SearchPhrase>
<field> 1549 </ field>
</ row>
<row>
<SearchPhrase >自由照片</ SearchPhrase>
<字段> 1480 </场\>
</行\>
<行\>
<SearchPhrase>安吉丽娜</ SearchPhrase>
<字段> 1245 </场\>
</行\>
<行\>
<SearchPhrase>鄂木斯克</ SearchPhrase>
<field> 1112 </ field>
</ row>
<row>
<SearchPhrase>狗品种的照片</ SearchPhrase>
<field> 1091 </ field>
</ row>
<row>
<SearchPhrase>窗帘设计</ SearchPhrase>
<field> 1064 </ field>
</ row>
<行\>
<SearchPhrase>巴库</ SearchPhrase>
<字段> 1000 </场\>
</行\>
</数据\>
<行> 10 </行\>
<rows_before_limit_at_least> 141137 </ rows_before_limit_at_least>
</导致>

如果列名称的格式不可接受，则仅将“字段”用作元素名称。通常，XML 结构遵循 JSON 结构。就像 JSON 一样，无效的 UTF-8 序列会更改为替换字符``。因此输出文本将包含有效的 UTF-8 序列。

在字符串值中，字符`<`和`&`被转义为`<`和`&`。

数组输出为`<array><elem>Hello</elem><elem>World</elem>...</array>`，元组输出为`<tuple><elem>Hello</elem><elem>World</elem>...</tuple>`。

### CapnProto [¶](https://clickhouse.yandex/docs/en/single/#capnproto "Permanent link")

Cap'n Proto 是类似于协议缓冲区和 Thrift 的二进制消息格式，但与 JSON 或 MessagePack 不同。

Cap'n Proto 消息是严格键入的，并且不能自我描述，这意味着它们需要外部架构描述。该模式是即时应用的，并为每个查询缓存。

\$ cat capnproto_messages.bin | clickhouse-client --query “将 INERT 插入 test.hits 格式 CapnProto SETTINGS format_schema ='schema：Message'”

哪里`schema.capnp`是这样的：

struct Message {
SearchPhrase @ 0 ：Text ;
c @ 1 ：Uint64 ;
}

反序列化是有效的，通常不会增加系统负载。

另请参阅[格式架构](https://clickhouse.yandex/docs/en/single/#formatschema)。

### protobuf 的[¶](https://clickhouse.yandex/docs/en/single/#protobuf "Permanent link")

Protobuf-是[协议缓冲区](https://developers.google.com/protocol-buffers/)格式。

此格式需要外部格式架构。模式在查询之间进行缓存。ClickHouse 支持`proto2`和`proto3`语法。支持重复/可选/必填字段。

用法示例：

选择 \* 从 测试。表 FORMAT Protobuf SETTINGS format_schema = 'schemafile：MessageType'

猫 protobuf_messages.bin | clickhouse-client --query “将 INERT 插入 test.table 格式 Protobuf 设置 format_schema ='schemafile：MessageType'”

该文件`schemafile.proto`如下所示：

语法 =“ proto3” ;

message MessageType {
字符串 名称 = 1 ;
字符串 姓 = 2 ;
uint32 birthDate = 3 ;
重复的 字符串 phoneNumbers = 4 ;
};

若要查找表列和协议缓冲区的消息类型的字段之间的对应关系，ClickHouse 会对其名称进行比较。此比较不区分大小写，并且字符`_`（下划线）和`.`（点）被认为是相等的。如果列的类型和协议缓冲区的消息的字段不同，则将应用必要的转换。

支持嵌套消息。例如，对于`z`以下消息类型中的字段

消息 MessageType {
消息 XType {
消息 YType {
int32 z ;
};
重复 YType y ;
};
XType x ;
};

ClickHouse 试图找到一个名为列`x.y.z`（或`x_y_z`或`X.y_Z`等）。嵌套消息适合于输入或输出[嵌套数据结构](https://clickhouse.yandex/docs/en/single/#../data_types/nested_data_structures/nested/)。

在像这样的 protobuf 模式中定义的默认值

语法 =“ proto2” ;

message MessageType {
可选 int32 result_per_page = 3 \[默认= 10\]；
}

不适用；使用[表默认值](https://clickhouse.yandex/docs/en/single/#create-default-values)代替它们。

ClickHouse 以`length-delimited`格式输入和输出 protobuf 消息。这意味着在每条消息之前都应将其长度写为[varint](https://developers.google.com/protocol-buffers/docs/encoding#varints)。另请参阅[如何使用流行语言读取/写入以长度分隔的 protobuf 消息](https://cwiki.apache.org/confluence/display/GEODE/Delimiting+Protobuf+Messages)。

### 实木[复合](https://clickhouse.yandex/docs/en/single/#data-format-parquet "Permanent link")地板[¶](https://clickhouse.yandex/docs/en/single/#data-format-parquet "Permanent link")

[Apache Parquet](http://parquet.apache.org/)是 Hadoop 生态系统中广泛使用的一种列式存储格式。ClickHouse 支持对此格式的读取和写入操作。

#### 数据类型匹配[¶](https://clickhouse.yandex/docs/en/single/#data-types-matching "Permanent link")

下表显示支持的数据类型以及它们如何匹配 ClickHouse [数据类型](https://clickhouse.yandex/docs/en/single/#../data_types/)中`INSERT`和`SELECT`查询。

实木复合地板数据类型（`INSERT`）

ClickHouse 数据类型

实木复合地板数据类型（`SELECT`）

`UINT8`， `BOOL`

[UInt8](https://clickhouse.yandex/docs/en/single/#../data_types/int_uint/)

`UINT8`

`INT8`

[诠释 8](https://clickhouse.yandex/docs/en/single/#../data_types/int_uint/)

`INT8`

`UINT16`

[UInt16](https://clickhouse.yandex/docs/en/single/#../data_types/int_uint/)

`UINT16`

`INT16`

[16 位](https://clickhouse.yandex/docs/en/single/#../data_types/int_uint/)

`INT16`

`UINT32`

[UInt32](https://clickhouse.yandex/docs/en/single/#../data_types/int_uint/)

`UINT32`

`INT32`

[32 位](https://clickhouse.yandex/docs/en/single/#../data_types/int_uint/)

`INT32`

`UINT64`

[UInt64](https://clickhouse.yandex/docs/en/single/#../data_types/int_uint/)

`UINT64`

`INT64`

[整数 64](https://clickhouse.yandex/docs/en/single/#../data_types/int_uint/)

`INT64`

`FLOAT`， `HALF_FLOAT`

[浮点 32](https://clickhouse.yandex/docs/en/single/#../data_types/float/)

`FLOAT`

`DOUBLE`

[浮动 64](https://clickhouse.yandex/docs/en/single/#../data_types/float/)

`DOUBLE`

`DATE32`

[日期](https://clickhouse.yandex/docs/en/single/#../data_types/date/)

`UINT16`

`DATE64`， `TIMESTAMP`

[约会时间](https://clickhouse.yandex/docs/en/single/#../data_types/datetime/)

`UINT32`

`STRING`， `BINARY`

[串](https://clickhouse.yandex/docs/en/single/#../data_types/string/)

`STRING`

-

[固定字符串](https://clickhouse.yandex/docs/en/single/#../data_types/fixedstring/)

`STRING`

`DECIMAL`

[小数](https://clickhouse.yandex/docs/en/single/#../data_types/decimal/)

`DECIMAL`

ClickHouse 支持`Decimal`类型的可配置精度。该`INSERT`查询将 Parquet `DECIMAL`类型视为 ClickHouse `Decimal128`类型。

不支持的实木复合地板的数据类型：`DATE32`，`TIME32`，`FIXED_SIZE_BINARY`，`JSON`，`UUID`，`ENUM`。

ClickHouse 表列的数据类型可以与插入的 Parquet 数据的相应字段不同。在插入数据时，ClickHouse 根据上表解释数据类型，然后[将](https://clickhouse.yandex/docs/en/single/#type_conversion_function-cast)数据转换为为 ClickHouse 表列设置的数据类型。

#### 插入和选择数据[¶](https://clickhouse.yandex/docs/en/single/#inserting-and-selecting-data "Permanent link")

您可以通过以下命令将 Parquet 数据从文件插入 ClickHouse 表：

猫{文件名} | clickhouse-client --query = “ INSERT INTO {some_table} FORMAT Parquet”

您可以从 ClickHouse 表中选择数据，然后通过以下命令将其保存为 Parquet 格式的文件：

clickhouse \- 客户 --query = “SELECT \* FROM {some_table} FORMAT 镶木”> {some_file.pq}

要与 Hadoop 交换数据，可以使用[HDFS 表引擎](https://clickhouse.yandex/docs/en/single/#../operations/table_engines/hdfs/)。

### 格式模式[¶](https://clickhouse.yandex/docs/en/single/#formatschema "Permanent link")

包含格式架构的文件名由设置来设置`format_schema`。使用`Cap'n Proto`和格式之一时，需要设置此设置`Protobuf`。格式架构是文件名和此文件中消息类型名称的组合，以冒号分隔，例如`schemafile.proto:MessageType`。如果文件具有格式的标准扩展名（例如`.proto`for `Protobuf`），则可以将其省略，在这种情况下，格式架构看起来像`schemafile:MessageType`。

如果您以[交互方式](https://clickhouse.yandex/docs/en/single/#cli_usage)通过[客户端](https://clickhouse.yandex/docs/en/single/#../interfaces/cli/)输入或输出数据，则格式架构中指定的文件名可以包含绝对路径或相对于客户端当前目录的路径。如果以[批处理方式](https://clickhouse.yandex/docs/en/single/#cli_usage)使用客户端，则出于安全原因，架构的路径必须是相对的。

如果通过[HTTP 接口](https://clickhouse.yandex/docs/en/single/#../interfaces/http/)输入或输出数据，则格式模式中指定的文件名应位于   服务器配置中[format_schema_path](https://clickhouse.yandex/docs/en/single/#server_settings-format_schema_path)中指定的目录中。

### 跳过错误[¶](https://clickhouse.yandex/docs/en/single/#skippingerrors "Permanent link")

一些格式，如`CSV`，`TabSeparated`，`TSKV`，`JSONEachRow`，`Template`，`CustomSeparated`和`Protobuf`可以跳过损坏行是否发生了解析错误，并继续从下一行的开始解析。请参阅[input_format_allow_errors_num](https://clickhouse.yandex/docs/en/single/#settings-input_format_allow_errors_num)和  [input_format_allow_errors_ratio](https://clickhouse.yandex/docs/en/single/#settings-input_format_allow_errors_ratio)设置。限制：-在解析错误的情况下，错误会`JSONEachRow`跳过所有数据，直到出现新行（或 EOF）为止，因此必须使用行来分隔行`\n`以正确计数错误。- `Template`并`CustomSeparated`在最后一列之后使用定界符，并在行之间使用定界符来查找下一行的开头，因此，仅当其中至少一个不为空时，跳过错误才有效。

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












