# 客户端

---

ClickHouse提供了两个网络接口（为了安全起见，都可以选择将两者包装在TLS(安全传输协议)中）：

- HTTP： 易于使用
- Native TCP:  开销小

在大多数情况下，建议使用适当的工具或库，而不是直接与这些工具或库进行交互。 Yandex的官方支持如下:
- Command-line client
- JDBC driver
- ODBC driver

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
  - 默认情况下，返回的数据是 `TabSeparated `格式的， 可配（Format)
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
$ echo 'SELECT 1' | curl 'http://user:password@localhost:8123/' -d @-

## 注意： 如果用户名没有指定，默认的用户是 default。如果密码没有指定，默认会使用空密码。 
```

### `HTTP` 压缩
`ClickHouse`支持`gzip`，`br`以及`deflate` 压缩方法: 

- 压缩请求 `Content-Encoding:compression_method`
- 压缩响应 `Accept-Encoding: compression_method`
- 要启用HTTP压缩，必须使用[`ClickHouse enable_http_compression`](./clickhouse_operations.md)设置
- 您可以在http_zlib_compression_level设置中为所有压缩方法配置数据压缩级别。
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

如果HTTP连接丢失，正在运行的请求不会自动停止。解析和数据格式化是在服务器端执行的，使用网络可能无效。可选的'query_id'参数可以作为查询ID（任何字符串）传递。有关更多信息，请参见“Settings, replace_running_query”部分。

可选的“quota_key”参数可以作为（Quotas）配额密钥（任何字符串）传递。有关更多信息，请参见“Quotas（配额）（运维部分有详细描述）”部分。

HTTP接口允许传递外部数据（external temporary tables）以进行查询。有关更多信息，请参见“External data for query processing”(运维部分有详细描述)部分。

### Response Buffering
您可以在服务器端启用响应缓冲。在`buffer_size`和`wait_end_of_query`URL参数提供了这个目的。
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













