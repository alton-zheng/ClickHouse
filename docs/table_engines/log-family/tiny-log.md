## TinyLog

- `write-one` 数据，然后根据需要多次读取。
  - 例如，您可以将 `TinyLog-type` 表用于小批量处理的中间数据。
  - 请注意，将数据存储在大量小表中效率很低。

`Query` 在单个 `Stream` 中执行。
  - 适合许多小表且 `write-once`的场景，打开的文件数少（单表数据在一个文件（`data.bin`）中）