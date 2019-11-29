# Compression in ClickHouse

ClickHouse 支持  `LZ4`（默认）, `ZSTD`, `Multiple`, `Delta`, `T64`, `DoubleDelta`, `Gorilla` 压缩算法

2018 年前， ClickHouse 仅支持两种 `LZ4`（默认）, `ZSTD`

[Compression in ClickHouse](https://www.altinity.com/blog/2017/11/21/compression-in-clickhouse),  原文在*`2017/11/21`* 发表

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
  - 可达到的最大压缩比约为250
  
---  

#### 格式

- LZ4 压缩块由序列组成
  - 序列由一组 `iteral`（未压缩字节） 组成， 后跟匹配副本。
  
- 序列： 
  - `[token]literals(offset,matchlength)[token](offset,matchlength)...`
  - 每个序列都以 `token` 开头。
    - `token` 占一个 `byte`, 分隔为 `2` 个 `4-bits` `field`.
     - 每个 `field` 范围 `[0, 15]`.
     - 第一个 `field` 在 `token` 的 4个 `high-bits`, 它存储 `literal` 的长度。
       - 如 `field` 值 为 0, 则没有 `literal`
       - 额外 `byte` [`0`, `255`]
       - 当 `byte` 值等于 `255`， 将输出另一个 `byte`。
  - `token` 后面可以有任意数量的 `byte`


到目前为止的内容，举例解析：        

- 例1: `48` 的 `literal` 长度将表示为:
  - `15`: 4个 `high-bits` 的值
  - `33`: (= `48-15`) 剩余长度

- 例2:`280` 的 `literal` 长度将表示为:
  - `15`: 4个 `high-bits` 的值
  - `255`: `byte` 已填充，因为 `280-15 >= 255`
  - `10`:(= `280 - 15 - 255`) 剩余长度

- 例3:长度为`15`的 `literal` 将表示为:
  - `15`: 4个 `high-bits` 的值
  - `0`:(=`15-15`) 
  


- `literal`： 紧随 `token` 和可选长度 `byte`
  - 它们的数量与之前 `decode` 的数量(`iterals` 长度)一样多。有可能没有 `literal`。
  
- 匹配副本： `literal` 后
  - `offset`: 
    - 偏移量
    - 2 `byte` 的值
      - `little endian`
      - 第一个 `byte` 是 `low` 位字节
      - 第二个 `byte` 是 `high` 位字节
    - 表示匹配项位置
    - max: `65535`
    - min: `1` 
  - `matchlength`: 
    - 匹配长度
    - `minmatch` 长度为4。 
    - 需要使用`token` 第二个 `field`, 4个 `low-bits`
      - `[0, 15]`
        - `0`： 代表4个 `byte`
        - `15`: 代表 `19+` `byte`, 额外的 byte, `[0, 255]`
        - 额外的 `byte` 被添加到total中
        
        
解码匹配长度到达当前序列的末端。下一个字节将是另一个序列的开始。但是在移动到下一个序列之前，需要使用解码的匹配位置和长度。解码器将匹配长度字节从匹配位置复制到当前位置。

在某些情况下，匹配长度大于偏移量。因此，match_pos + matchlength > current_pos，这意味着以后要复制的字节还没有被解码。这称为“重叠匹配”，必须特别小心处理。一个常见的情况是偏移量为1，这意味着最后一个字节重复匹配长度的次数。

#### 结束 Block 定义

- 终止一个块需要特定的规则。
  - 最后一个序列只包含 `literal`。
  - 最后5个 `byte` 的输入总是 `literal`, 因此，最后一个序列至少包含5个 `byte`
    - 特殊:
        - 如果输入小于 5 `byte`，则只有一个序列，它以 `literal` 的形式包含整个输入。
        - 空输入可以用一个零字节表示，可以解释为没有文字和匹配的最终标记。
    
  - 最后一个匹配必须在 `Block` 结束前至少12个 `byte` 开始。最后一个匹配项是倒数第二个匹配项的一部分。后面是最后一个序列，它只包含 `literal`。
    - 注意，因此，一个 <13 `byte` 的独立块不能被压缩，因为匹配项必须复制“某些东西”，所以它至少需要一个先前的 `byte`。
    - 当一个 block 可以引用来自另一个 block 的数据时，它可以立即使用匹配而不是 literal 开始，因此一个12 `byte` 的块可以被压缩。
    
- 当一个块不符合这些结束条件时，一个符合的解码器被允许以不正确为由拒绝该块。

- 这些规则的存在是为了确保一致的译码器能够被设计成速度快、能发出推测性的指令，同时不会读取或写入超过所提供的I/O缓冲区。

### Example

```log
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
