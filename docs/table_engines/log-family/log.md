## Log

`Log Engine` 与 `TinyLog Engine` 的区别: 

- 列文件中存在一个小的 `marks` 文件。
  - 写在每个 `data block` 上
  - 包含 `offset`，这些 `offset` 指示从何处开始读取文件以跳过指定的行数。
  - 这使得可以在多个线程中读取表数据。
  - 适用 `temporary data`，`write once` 以及 `test` 或演习为目的的场景。