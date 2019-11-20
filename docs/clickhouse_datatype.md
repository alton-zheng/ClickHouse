# 数据类型

ClickHouse 支持的数据类型： 

---

## 整型

固定长度的 `Integer`： Int 和 UInt

- Int 范围：
  - `Int8` ：[-128 : 127]
  - `Int16` ：[-32768 : 32767]
  - `Int32` ：[-2147483648 : 2147483647]
  - `Int64` ：[-9223372036854775808 : 9223372036854775807]

- Uint 范围：
  - `UInt8` - [0 : 255]
  - `UInt16` - [0 : 65535]
  - `UInt32` - [0 : 4294967295]
  - `UInt64` - [0 : 18446744073709551615]
 
---
  
## Float

- `Float32` : `float`
- `Float64` : `double`

---

## Decimal

- 有符号的定点数: 
  - 可在加、减和乘法运算过程中保持精度。
  - 对于除法，最低有效数字会被丢弃（不舍入）。

- `Decimal(P, S)`
  - `P` : 精度。有效范围：`[1:38]`，决定可以有多少个十进制数字（包括分数）。
  - `S` : 规模。有效范围：`[0:P]`，决定数字的小数部分中包含的小数位数。
  
- `Decimal32(S)` : `( -1 * 10^(9 - S), 1 * 10^(9 - S) )`
- `Decimal64(S)` : `( -1 * 10^(18 - S), 1 * 10^(18 - S) )`
- `Decimal128(S)` : `( -1 * 10^(38 - S), 1 * 10^(38 - S) )`

---

## Bloolean

`ClickHouse` 没有单独的类型来存储布尔值。可以使用 `UInt8` 类型，取值限制为 `0` 或 `1`。

---

## String

- `String`: 
  - 可以任意长度。
  - 它可以包含任意的字节集，包含空字节。
  - 可以代替其他 数据库中的 `VARCHAR`、`BLOB`、`CLOB` 等类型。
  - `UTF-8` 编码

- `FixedString`
  - 固定长度 `N` 的字符串（`N` 必须是严格的正自然数） 
  - e.g. `<column_name> FixedString(N)`
  - 数据的长度恰好为N个字节时，`FixedString` 类型是高效的。 在其他情况下，这可能会降低效率。
   
---

## UUID

- 通用唯一标示符：(`UUID`):
  - `16-bytes`
  - e.g. `61f0c404-5cb3-11e7-907b-a6006ad3dba0`
  - 默认：`00000000-0000-0000-0000-000000000000`
  - 生成方式： `generateUUIDv4` 函数
 
--- 

## Date

- Date:
  - `2-bytes`
  - `unsigned`
  - 理论支持最大年份 `2106`, 实际支持： `2105`
  - `min`: `0000-00-00`
  - 不带时区信息
 
--- 

## DateTime

- `Unix timestamp`: 
  - 可带时区信息
  - DateTime([timezone])
  - 范围： `[1970-01-01 00:00:00, 2105-12-31 23:59:59]`
  
---

## Enum

- 'string' = integer: 
  - `ClickHouse` 仅存储数字， 但支持使用其名称进行读写操作。 
  - 范围：
    - `Enum(8)`: ·`[-128, 127]`
    - `Enum(16)`: `[-32768, 32767]`
  
- e.g.

```clickhouse
CREATE TABLE t_enum
(
    x Enum('hello' = 1, 'world' = 2)
)
ENGINE = TinyLog
```

---

## Array

- Array(T)
  - `T-type` 元素组成
  - `T` 可以是任意类型(数组也支持)
  - 构建方式： `[a,b]` 和 `array(T)`

---

## AggregateFunction

- AggregateFunction(name, types_of_arguments...)
  - `-State`: 中间状态 e.g. `uniqState` 
  - `-Merge`: 最终状态   e.g. `uniqMerge`
  - AggregateFunction — 参数化的数据类型。

- 参数
  - 聚合函数名
  - 如果函数具备多个参数列表，请在此处指定其他参数列表中的值
  - Types
  
- e.g.

```clickhouse
CREATE TABLE t
(
    column1 AggregateFunction(uniq, UInt64),
    column2 AggregateFunction(anyIf, String, UInt8),
    column3 AggregateFunction(quantiles(0.5, 0.9), UInt64)
) ENGINE = ...
```

- 以下查询结果总是一致。

```clickhouse
SELECT uniq(UserID) FROM table

SELECT uniqMerge(state) FROM (SELECT uniqState(UserID) AS state FROM table GROUP BY RegionID)
```

---

## Tuple

- Tuple(T1, T2, ...):
  - 元组，其中每个元素都有单独的类型。
  - 不能在表中存储元组（除了 `Memory Table`）
  - 可以用于临时列分组。 
  - 在查询中，IN 表达式和带特定参数的 lambda 函数可以来对临时列进行分组
  
- e.g.

```clickhouse
SELECT
    (1, 'a') AS x,
    toTypeName(x)
```

```log
┌─x───────┬─toTypeName(tuple(1, 'a'))─┐
│ (1,'a') │ Tuple(UInt8, String)      │
└─────────┴───────────────────────────┘

1 rows in set. Elapsed: 0.021 sec.
```

---

## Nullable

- `Nullable(TypeName)`:
  - e.g. `Nullable(Int8)` ： 一看就明白
  - 不能包含复合数据类型。 e.g. `Nullable(Array(Int8))`
  - 复合数据类型能包含 `Nullable`.
  - 默认值： Null. 在服务器配置另有说明例外。
  
## Nested

- Nested(Name1 Type1, Name2 Type2, ...)

- 直接上例题： 

```clickhouse
CREATE TABLE test.visits
(
    CounterID UInt32,
    StartDate Date,
    Sign Int8,
    IsNew UInt8,
    VisitID UInt64,
    UserID UInt64,
    ...
    Goals Nested
    (
        ID UInt32,
        Serial UInt32,
        EventTime DateTime,
        Price Int64,
        OrderID String,
        CurrencyID UInt32
    ),
    ...
) ENGINE = CollapsingMergeTree(StartDate, intHash32(UserID), (CounterID, StartDate, intHash32(UserID), VisitID), 8192, Sign)
```

- 查询： 

```clickhouse
SELECT
    Goals.ID,
    Goals.EventTime
FROM test.visits
WHERE CounterID = 101500 AND length(Goals.ID) < 5
LIMIT 10
```

## Special data Types

- Expression: 用于表示高阶函数中的lambda表达式。
- Set: 用于IN表达式的右半部分。
- Nothing:  不期望的值。配合其它类型用

e.g.

```clickhouse
SELECT  toTypeName （array （））

┌─toTypeName(array())─┐
│ Array(Nothing)      │
└─────────────────────┘
```

- `Interval`
  - 代表时间和日期间隔的数据类型族

- 支持： 
  - `SECOND`
  - `MINUTE`
  - `HOUR`
  - `DAY`
  - `WEEK`
  - `MONTH`
  - `QUARTER`
  - `YEAR`
 
- e.g. 
```clickhouse
SELECT now() as current_date_time, current_date_time + INTERVAL 4 DAY
```                                                                  
```clickhouse
┌───current_date_time─┬─plus(now(), toIntervalDay(4))─┐
│ 2019-10-23 10:58:45 │           2019-10-27 10:58:45 │
└─────────────────────┴───────────────────────────────┘
```

---

## Domains

- Domains
  - 特定实现的类型
  - 总是与某个现存的基础类型保持二进制兼容的同时添加一些额外的特性，以能够在维持磁盘数据不变的情况下使用这些额外的特性。
  - 不支持自定义`Domains`

- 使用：
  - 使用 `Domain` 类型作为表中列的类型
  - 对 `Domain` 类型的列进行读/写数据
  - 如果与 `Domain` 二进制兼容的基础类型可以作为索引，那么 `Domain` 类型也可以作为索引
  - 将 `Domain` 类型作为参数传递给函数使用
  - 其他

- 额外特性：
  - 在执行 `SHOW CREATE TABLE` 或 `DESCRIBE TABLE` 时，其对应的列总是展示为Domain类型的名称
  - 在 `INSERT INTO domain_table(domain_column) VALUES(...)` 中输入数据总是以更人性化的格式进行输入
  - 在 `SELECT domain_column FROM domain_table` 中数据总是以更人性化的格式输出
  - 在 `INSERT INTO domain_table FORMAT CSV ...` 中，实现外部源数据以更人性化的格式载入

- 限制：
  - 无法通过 `ALTER TABLE` 将基础类型的索引转换为Domain类型的索引。
  - 当从其他列或表插入数据时，无法将string类型的值隐式地转换为Domain类型的值。
  - 无法对存储为Domain类型的值添加约束。

---
### IPv4

- `IPv4` 是与 `UInt32` 类型保持二进制兼容的 `Domain` 类型 
  - 用于存储IPv4地址的值
  - 提供了更为紧凑的二进制存储
  - 支持识别可读性更加友好的输入输出格式。

- e.g. 

```clickhouse
CREATE TABLE hits (url String, from IPv4) ENGINE = MergeTree() ORDER BY from;
```                                                                          

- 输入输出可读性更好：
```clickhouse
INSERT INTO hits (url, from) VALUES ('https://wikipedia.org', '116.253.40.133')('https://clickhouse.yandex', '183.247.232.58')('https://clickhouse.yandex/docs/en/', '116.106.34.242');

SELECT * FROM hits; 
```

```log
┌─url────────────────────────────────┬───────────from─┐
│ https://clickhouse.yandex/docs/en/ │ 116.106.34.242 │
│ https://wikipedia.org              │ 116.253.40.133 │
│ https://clickhouse.yandex          │ 183.247.232.58 │
└────────────────────────────────────┴────────────────┘
```

- 二进制存储格式：
```clickhouse
SELECT toTypeName(from), hex(from) FROM hits LIMIT 1;
┌─toTypeName(from)─┬─hex(from)─┐
│ IPv4             │ B7F7E83A  │
└──────────────────┴───────────┘
```

- 不可隐式转换为除UInt32以外的其他类型类型。如果要将IPv4类型的值转换成字符串，你可以使用IPv4NumToString()显示的进行转换：

```clickhouse
SELECT toTypeName(s), IPv4NumToString(from) as s FROM hits LIMIT 1;
┌─toTypeName(IPv4NumToString(from))─┬─s──────────────┐
│ String                            │ 183.247.232.58 │
└───────────────────────────────────┴────────────────┘
```

- 或可以使用CAST将它转换为UInt32类型:

```clickhouse
SELECT toTypeName(i), CAST(from as UInt32) as i FROM hits LIMIT 1;
┌─toTypeName(CAST(from, 'UInt32'))─┬──────────i─┐
│ UInt32                           │ 3086477370 │
└──────────────────────────────────┴────────────┘
```

### IPv6

- `IPv6` 是与`FixedString(16)`类型保持二进制兼容的 `Domain` 类型 
  - 用于存储IPv6地址的值
  - 提供了更为紧凑的二进制存储
  - 支持识别可读性更加友好的输入输出格式。
  
**Basic Usage**: 

```clickhouse
CREATE TABLE hits (url String, from IPv6) ENGINE = MergeTree() ORDER BY url;

DESCRIBE TABLE hits;

┌─name─┬─type───┬─default_type─┬─default_expression─┬─comment─┬─codec_expression─┐
│ url  │ String │              │                    │         │                  │
│ from │ IPv6   │              │                    │         │                  │
└──────┴────────┴──────────────┴────────────────────┴─────────┴──────────────────┘
```

同时您也可以使用 `IPv6` 类型的列作为主键：

```clickhouse
CREATE TABLE hits (url String, from IPv6) ENGINE = MergeTree() ORDER BY from;
```

在写入与查询时，`IPv6` 类型能够识别可读性更加友好的输入输出格式：

```clickhouse
INSERT INTO hits (url, from) VALUES ('https://wikipedia.org', '2a02:aa08:e000:3100::2')('https://clickhouse.yandex', '2001:44c8:129:2632:33:0:252:2')('https://clickhouse.yandex/docs/en/', '2a02:e980:1e::1');

SELECT * FROM hits;
┌─url────────────────────────────────┬─from──────────────────────────┐
│ https://clickhouse.yandex          │ 2001:44c8:129:2632:33:0:252:2 │
│ https://clickhouse.yandex/docs/en/ │ 2a02:e980:1e::1               │
│ https://wikipedia.org              │ 2a02:aa08:e000:3100::2        │
└────────────────────────────────────┴───────────────────────────────┘
```

- 同时它提供更为紧凑的二进制存储格式：

```clickhouse
SELECT toTypeName(from), hex(from) FROM hits LIMIT 1;

┌─toTypeName(from)─┬─hex(from)────────────────────────┐
│ IPv6             │ 200144C8012926320033000002520002 │
└──────────────────┴──────────────────────────────────┘
```

- 不可隐式转换为除 `FixedString(16)` 以外的其他类型类型。如果要将 `IPv6` 类型的值转换成字符串，你可以使用 `IPv6NumToString()` 显示的进行转换：

```clickhouse
SELECT toTypeName(s), IPv6NumToString(from) as s FROM hits LIMIT 1;

┌─toTypeName(IPv6NumToString(from))─┬─s─────────────────────────────┐
│ String                            │ 2001:44c8:129:2632:33:0:252:2 │
└───────────────────────────────────┴───────────────────────────────┘
```

- 或使用CAST将其转换为 `FixedString(16)` ：

```clickhouse
SELECT toTypeName(i), CAST(from as FixedString(16)) as i FROM hits LIMIT 1;

┌─toTypeName(CAST(from, 'FixedString(16)'))─┬─i───────┐
│ FixedString(16)                           │  ���    │
└───────────────────────────────────────────┴─────────┘

```

