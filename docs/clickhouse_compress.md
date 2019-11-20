# Compression in ClickHouse

[Compression in ClickHouse](https://www.altinity.com/blog/2017/11/21/compression-in-clickhouse),  原文在*`2017/11/21`* 发表

`ClickHouse` 支持2种压缩方法（算法）， 默认 `L24`, 可以用相关配置指定：
  - `LZ4`
  - `ZSTD`

[基于`Mysql`压缩算法测试报告](https://www.percona.com/blog/2016/04/13/evaluating-database-compression-methods-update/)

结论： 简而言之，LZ4在速度上会更快，但是压缩率较低，ZSTD正好相反。尽管ZSTD比LZ4慢，但是相比传统的压缩方式Zlib，无论是在压缩效率还是速度上，都可以作为Zlib的替代品。

- 基于 `ClickHouse` 的测试：
  - table `lineorder` [**结构和数据**](https://www.altinity.com/blog/2017/6/16/clickhouse-in-a-general-analytical-workload-based-on-star-schema-benchmark)
  - 未压缩数据集 `680G`
  - 压缩数据能力测试结果：
    - `LZ4` `184G/680G`（`27%`)
    - `ZSTD` `135G/680G`（`20%`)
    - 压缩比 `LZ4/ZSTD`: `3.7/5.0`
  - 性能测试：
    - `SQL`:
      - `SELECT toYear(LO_ORDERDATE) AS yod, sum(LO_REVENUE) FROM lineorder GROUP BY yod;` 没有 `where`,`order by` 等操作。 
    - 冷数据：
      - 两者区别不大，时间主要消耗在`I/O`,远大于解压缩的时间
    - 热数据(已缓存)：
      - 热数据请求下，`LZ4`会更快，此时`IO`代价小，数据解压缩成为性能瓶颈。消耗时间上两者比率： `9/16`, 快了差不多2倍
  - 结论：
    - 大范围扫描的查询，瓶颈是`I/O`， 首选 `ZSTD`。
    - 当`I/O`快时，`LZ4`为首选
    - 具体选择什么，看具体场景和需求。 

- `ClickHouse` 压缩算法配置：
```xml
    <compression incl="clickhouse_compression">
            <case>
                    <method>zstd</method>
            </case>
    </compression>
```


## `LZ4` 算法

---

### 简介

- LZ4是无损压缩算法
  - `lz4` 开源， 安卓，`Apple`,`python`, 都有支持 [code](https://github.com/lz4/lz4)
    - `clickhouse`（原引用,没做任何修改。见 `ClickHouse` 源码 [lz4.c](https://github.com/lz4/lz4/blob/master/lib/lz4.c) 文件, 此文件可从ClickHouse [源代码](https://github.com/ClickHouse/ClickHouse/tree/master/contrib/) 跳转
  - 可提供每核 `> 500 MB/s` 的压缩速度，并可与 `multi-core` CPU一起扩展。
  - 它具有极快的解码器，每个内核的速度为`N GB/s`(`N` 在 4~5之间)，通常在 `multi-core` 系统上达到 `RAM` 速度限制。
  - 可以动态调整速度，选择一个“加速”因子，以权衡压缩比以获得更快的速度。
  - 另一方面，还提供了高压缩导数 `LZ4_HC`，以 `CPU` 时间为代价来提高压缩率。所有版本均具有相同的减压速度。
  
  - `Format`
    - `[token]literals(offset,match_length)[token](offset,match length)...`
      - `Literals`指没有重复、首次出现的字节流，即不可压缩的部分
      - `Match`指重复项，可以压缩的部分
      - `Token`记录literal长度，match长度。作为解压时候memcpy的参数

- LZ4 有两种 `Block` 格式

---

### 原始 Block

---

#### 描述

- LZ4是一个 `LZ77` 类型的压缩器，具有固定的、面向字节的编码。
  - 没有熵编码器 `back-end` 或 `framing layer`(下面介绍)。
    - `熵`(`entropy`)编码器, 无序，不丢失任何信息的编码, 这里不展开，有兴趣的，可以找下资料。 
  - 简单和快速。
  - 不可压缩的块数据会扩展 `0.4%`
  - 它有助于以后的优化、紧凑性和特性。
    - 扩展性好，可优化空间大。
  
---  

#### 格式

- LZ4 压缩块由序列组成
  - 序列由一组 `iteral`（未压缩字节） 组成， 后跟匹配副本。
  
- 序列： 
  - `[token]literals(offset,match_length)[token](offset,match length)...`
  - 每个序列都以 `token` 开头。
    - `token` 是一个 byte 值， 分隔为 `2` 个 `4-bits` `field`.
     - 每个 `field` 范围 `[0, 15]`.
     - 第一个 `field` 在 `token` 的 4个 `high-bits`, 它存储 `literal` 的 `length`。
       - 如 `field` 值 为 0, 则没有 `literal`
       - `byte` 表示一个从 `0` 到 `255` 的值。
       - 当 `byte` 值等于 `255`， 将输出另一个 `byte`。
  - `token` 后面可以有任意数量的 `byte`


到目前为止的内容，举例解析：        

- 例1: `48` 的 `literal` 长度将表示为:
  - `15`: 4个 `high-bits` 的值
  - `33`: (= `48-15`) 剩余长度

- 例2:`280` 的 `literal` 长度将表示为:
  - `15`: 4个 `high-bits` 的值
  - `255`: `byte` 以填充完，因为 `280-15 >= 255`
  - `10`:(= `280 - 15 - 255`) 剩余长度

- 例3:长度为`15`的 `literal` 将表示为:
  - `15`: 4个 `high-bits` 的值
  - `0`:(=`15-15`) 

紧随令牌和可选长度字节之后的是字面值本身。它们的数量与之前解码的数量(文字的长度)一样多。有可能没有文字。

文字之后是匹配复制操作。

从偏移量开始。这是一个2字节的值，采用小端字节格式(第一个字节是“低”字节，第二个字节是“高”字节)。

偏移量表示要从中复制的匹配项的位置。1表示“当前位置- 1字节”。最大偏移量为65535，不能编码65536。注意，0是一个无效的值，没有被使用。

然后我们需要提取匹配长度。为此，我们使用第二个令牌字段，即低4位。显然，值的范围是0到15。但是在这里，0意味着复制操作将是最小的。一个匹配的最小长度minmatch是4。因此，0的值表示4个字节，15的值表示19+字节。与文字长度类似，在达到最大可能的值(15)时，我们输出额外的字节，一次输出一个字节，其值从0到255不等。它们被添加到total中以提供最终的比赛长度。255的值表示还有一个字节需要读取和添加，这种方式输出的可选字节数没有限制。(这指的是可达到的最大压缩比约为250)。

解码匹配长度到达当前序列的末端。下一个字节将是另一个序列的开始。但是在移动到下一个序列之前，需要使用解码的匹配位置和长度。解码器将匹配长度字节从匹配位置复制到当前位置。

在某些情况下，匹配长度大于偏移量。因此，match_pos + matchlength > current_pos，这意味着以后要复制的字节还没有被解码。这称为“重叠匹配”，必须特别小心处理。一个常见的情况是偏移量为1，这意味着最后一个字节重复匹配长度的次数。


### Example

```log
input：abcde_bcdefgh_abcdefghxxxxxxx
output：abcde_(5,4)fgh_(14,5)fghxxxxxxx
```

### 压缩算法解析
     
- 官方发布压缩算法测试结果：

|  Compressor             | Ratio   | Compression | Decompression |
|  ----------             | -----   | ----------- | ------------- |
|  memcpy                 |  1.000  | 13700 MB/s  |  13700 MB/s   |
|**LZ4 default (v1.9.0)** |**2.101**| **780 MB/s**| **4970 MB/s** |
|  LZO 2.09               |  2.108  |   670 MB/s  |    860 MB/s   |
|  QuickLZ 1.5.0          |  2.238  |   575 MB/s  |    780 MB/s   |
|  Snappy 1.1.4           |  2.091  |   565 MB/s  |   1950 MB/s   |
| [Zstandard] 1.4.0 -1    |  2.883  |   515 MB/s  |   1380 MB/s   |
|  LZF v3.6               |  2.073  |   415 MB/s  |    910 MB/s   |
| [zlib] deflate 1.2.11 -1|  2.730  |   100 MB/s  |    415 MB/s   |
|**LZ4 HC -9 (v1.9.0)**   |**2.721**|    41 MB/s  | **4900 MB/s** |
| [zlib] deflate 1.2.11 -6|  3.099  |    36 MB/s  |    445 MB/s   |



## `ZSTD` 算法
- `ZSTD`
  - 在2016年由`facebook` 开源的无损压缩算法。 [code](https://github.com/facebook/zstd)
  - 压缩率和压缩/解压缩性能都很突出
  - 支持训练方式生成字典。
  - 开源后， `Linux` 内核/`HTTP` 协议/`hadoop 3`/ `Hbase 2`/`spark2.3+`/`kafka 2.10` 都加入对`ZSTD`的支持 

- 官方发布压缩算法测试结果：

| Compressor name         | Ratio | Compression| Decompress.|
| ---------------         | ------| -----------| ---------- |
| **zstd 1.4.4 -1**       | 2.884 |   520 MB/s |  1600 MB/s |
| zlib 1.2.11 -1          | 2.743 |   110 MB/s |   440 MB/s |
| brotli 1.0.7 -0         | 2.701 |   430 MB/s |   470 MB/s |
| quicklz 1.5.0 -1        | 2.238 |   600 MB/s |   800 MB/s |
| lzo1x 2.09 -1           | 2.106 |   680 MB/s |   950 MB/s |
| lz4 1.8.3               | 2.101 |   800 MB/s |  4220 MB/s |
| snappy 1.1.4            | 2.073 |   580 MB/s |  2020 MB/s |
| lzf 3.6 -1              | 2.077 |   440 MB/s |   930 MB/s |
