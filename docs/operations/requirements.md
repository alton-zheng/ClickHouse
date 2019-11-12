# Requirements

## CPU

从预构建的 `deb` 软件包进行安装: 
- `x86_64` 架构并支持 SSE 4.2 指令的 `CPU`。

使用不支持 `SSE 4.2` 或 `AArch64` 或 `PowerPC64LE` 体系结构的处理器，用源代码在相关机器上进行重新编译安装。

`ClickHouse`:
- 并行数据处理
- 使用所有可用的硬件资源。
- `CPU` `core` 和 `clock rate` 越高，能更高效的工作

建议使用**`Turbo Boost`**和**`hyper-threading`**技术。在典型负载下，它可以显着提高性能。

## RAM

使用 `4GB` 以上 的 `RAM` 来执行复杂的查询。查询需要消耗内存资源，内存越大，越有利于缓存数据。

所需的 RAM 量取决于：

- 查询的复杂性。
- 查询中处理的数据量。

要计算所需的 RAM 容量，需要下列操作的临时数据大小估算：
- `GROUP BY`
- `DISTINCT`
- `JOIN`
- Other

- 可以使用外部存储器存储临时数据。`GROUP BY`。

## Swap File

在 `production environments` 中禁用交换文件。

## **Storage Subsystem**

需要 2GB 的可用磁盘空间安装 ClickHouse。

您的数据所需的存储量应单独计算。评估应包括：

- 数据量估算。

  您可以对数据进行采样，并从中获取一行的平均大小。然后，将该值乘以您计划存储的行数。

- 数据压缩系数。

  要估算数据压缩系数，请将数据样本存储 `ClickHouse` `Table Engine` 中，然后将数据的实际大小与存储的表的大小进行比较。例如，点击流数据通常压缩 `6-10` 倍。

- 估计的数据容量容量乘以副本数。

## Network

如果可以，使用 10G 或更高级别的网络。（官方建议， 万M 以上）

网络带宽对于处理具有大量中间数据的分布式查询至关重要。另外，网络速度会影响复制过程。

## Software

`ClickHouse` 是为 `Linux` 系列操作系统开发的。推荐的 `Linux` 发行版是 `Ubuntu`。该`tzdata`软件包应安装在系统中。

可以在其他操作系统系列中使用。请参阅文档 `Getting Started` 部分中的详细信息。

