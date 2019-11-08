## Log Engine Family

`Log Engine` 是为了需要写入许多小数据量（少于`100万`行）的表的场景而开发的。

这系列的 `Engines` 有：

- [StripeLog](stripe-log.md)
- [Log](log.md)
- [TinyLog](tiny-log.md)

### **Common properties**

`Engines`：

- 将数据存储在`Disk`上。
- 写入时将数据`Append`到文件末尾，逐行写入。
- 支持并发数据访问锁。
  - 在 `INSERT` 查询期间，表是 `locked` 状态.
    - `locked` 状态， `read` 和 `write` 处于 `waiting` 状态。
    - `unlock` 状态， 支持并发非 `write` 操作。

- 不支持 `mutations` 操作。

- 不支持索引。
  - 范围查询效率不高。

- 不要自动写数据。定时写入的操作不建议使用，配置表（变更少，数据量小），每次都需要全量查询的表，可以用此系列 `Engines` 
  - 如果有操作中断了写操作（例如，服务器异常关闭），则可能会获得带有损坏数据的表。

### **Differences**
- `TinyLog Engine` : 
  - 是该系列中最简单的引擎，功能最差，效率最低。
  - 不支持并发 `read` 数据。
  - 所有系列 `Engines` 性能最差

- `Log Engine` 和 `StripeLog Engine`: 
  - 支持并发 `read` 操作
  - 读取数据时，`ClickHouse` 使用多个线程。
    - 每个线程处理一个单独的数据块。
  - `Log Engine` 单列一个单独的文件。
    - 提供了更高的效率。在 `Log Engine Family` 中，效率最高。
  - `StripeLog Engine` 将所有数据存储在一个文件中。
    - 在操作系统中使用较少的 `descriptors`