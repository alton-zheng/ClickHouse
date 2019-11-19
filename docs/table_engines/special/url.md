## URL(URL, Format)

用于管理远程 `HTTP/HTTPS` 服务器上的数据。该引擎类似  [File](file.md)  引擎。

简单来说： 就是用 SQL 方式，向远程服务器，发送 `GET`, `POST` 请求。

---

### 在 ClickHouse 服务器中使用引擎

- `Format`:   见 [Format](../../clickhouse_interfaces.md)。

- `URL`  必须符合统一资源定位符的结构。
  - 指定的 `URL` 必须指向一个 `HTTP` 或 `HTTPS` 服务器。
  - 对于服务端响应，不需要任何额外的 `HTTP` 头标记。

- `INSERT`  和  `SELECT`  查询会分别转换为  `POST`  和  `GET`  请求。 
  - 对于  `POST`  请求的处理，远程服务器必须支持  [分块传输编码](https://en.wikipedia.org/wiki/Chunked_transfer_encoding)。

**示例：**

- 建 `url_engine_table` 表： 

```clickhouse
CREATE TABLE url_engine_table (word String, value UInt64)
ENGINE=URL('http://127.0.0.1:12345/', CSV)
```

- 用标准的 Python 3 工具库创建一个基本的 HTTP 服务并启动它：

```python
from http.server import BaseHTTPRequestHandler, HTTPServer

class CSVHTTPServer(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.send_header('Content-type', 'text/csv')
        self.end_headers()

        self.wfile.write(bytes('Hello,1\nWorld,2\n', "utf-8"))

if __name__ == "__main__":
    server_address = ('127.0.0.1', 12345)
    HTTPServer(server_address, CSVHTTPServer).serve_forever()
```
```log
python3 server.py
```

**3.**  查询请求:

```clickhouse
SELECT * FROM url_engine_table
```

```log
┌─word──┬─value─┐
│ Hello │     1 │
│ World │     2 │
└───────┴───────┘
```

### 实现细节： 

- 并发读写
- 不支持：
  - `ALTER`  和  `SELECT...SAMPLE`  操作
  - 索引(`indexes`)
  - 副本（`Replication`）