# 职能

至少\*有两种类型的函数-常规函数（它们仅称为“函数”）和聚合函数。这些是完全不同的概念。常规函数的工作方式就像分别应用于每行（对于每一行，函数的结果不依赖于其他行）。聚合函数从各个行中累积一组值（即，它们取决于整个行集）。

在本节中，我们讨论常规功能。有关集合函数，请参见“集合函数”一节。

_-“ arrayJoin”函数属于第三种函数；表函数也可以单独提及。_

## 强类型[¶](https://clickhouse.yandex/docs/en/query_language/functions/#strong-typing "永久链接")

与标准 SQL 相比，ClickHouse 具有强类型。换句话说，它不会在类型之间进行隐式转换。每个函数都适用于一组特定的类型。这意味着有时您需要使用类型转换功能。

## 通用子表达式消除[¶](https://clickhouse.yandex/docs/en/query_language/functions/#common-subexpression-elimination "永久链接")

查询中具有相同 AST（相同记录或相同语法分析结果）的所有表达式都被认为具有相同的值。此类表达式被串联并执行一次。同样的子查询也可以通过这种方式消除。

## 结果类型[¶](https://clickhouse.yandex/docs/en/query_language/functions/#types-of-results "永久链接")

所有函数都将返回一个返回值（不是多个值，也不是零值）。结果的类型通常仅由参数的类型定义，而不由值定义。tupleElement 函数（aN 运算符）和 toFixedString 函数是例外。

## 常量[¶](https://clickhouse.yandex/docs/en/query_language/functions/#constants "永久链接")

为简单起见，某些函数只能对某些参数使用常量。例如，LIKE 运算符的 right 参数必须为常数。几乎所有函数都为常量参数返回常量。生成随机数的函数是个例外。对于在不同时间运行的查询，“ now”函数返回不同的值，但结果被认为是常量，因为常量仅在单个查询中很重要。常量表达式也被视为常量（例如，可以从多个常量构造 LIKE 运算符的右半部分）。

可以对常量和非常量参数以不同的方式实现函数（执行不同的代码）。但是，常数和仅包含相同值的 true 列的结果应该彼此匹配。

## NULL 处理[¶](https://clickhouse.yandex/docs/en/query_language/functions/#null-processing "永久链接")

函数具有以下行为：

- 如果函数的至少一个参数为`NULL`，则函数的结果也为`NULL`。
- 在每个功能的说明中分别指定的特殊行为。在 ClickHouse 源代码中，这些函数具有`UseDefaultImplementationForNulls=false`。

## 常数[¶](https://clickhouse.yandex/docs/en/query_language/functions/#constancy "永久链接")

函数不能更改其参数的值–任何更改都将作为结果返回。因此，计算单独功能的结果不取决于在查询中写入功能的顺序。

## 错误处理[¶](https://clickhouse.yandex/docs/en/query_language/functions/#error-handling "永久链接")

如果数据无效，某些函数可能会引发异常。在这种情况下，查询被取消，并且错误文本返回给客户端。对于分布式处理，当其中一台服务器上发生异常时，其他服务器也会尝试中止查询。

## 参数表达式的求值[¶](https://clickhouse.yandex/docs/en/query_language/functions/#evaluation-of-argument-expressions "永久链接")

在几乎所有编程语言中，可能不会为某些运算符评估其中一个参数。这通常是运营商`&&`，`||`以及`?:`。但是在 ClickHouse 中，始终会评估函数（运算符）的参数。这是因为列的整个部分都被立即求值，而不是分别计算每行。

## 执行功能分布式查询处理[¶](https://clickhouse.yandex/docs/en/query_language/functions/#performing-functions-for-distributed-query-processing "永久链接")

对于分布式查询处理，在远程服务器上执行尽可能多的查询处理阶段，而其余阶段（合并中间结果和此后的所有操作）在请求者服务器上执行。

这意味着可以在不同的服务器上执行功能。例如，在查询中`SELECT f(sum(g(x))) FROM distributed_table GROUP BY h(y),`

- 如果 a `distributed_table`具有至少两个分片，则在远程服务器上执行功能'g'和'h'，而在请求者服务器上执行功能'f'。
- 如果 a `distributed_table`仅具有一个分片，则所有“ f”，“ g”和“ h”功能都在该分片的服务器上执行。

函数的结果通常不取决于在哪个服务器上执行。但是，有时这很重要。例如，与词典一起使用的函数使用它们在其上运行的服务器上存在的词典。另一个示例是`hostName`函数，该函数返回正在运行的服务器的名称，以便`GROUP BY`由服务器在`SELECT`查询中进行。

如果查询中的函数是在请求者服务器上执行的，但是您需要在远程服务器上执行，则可以将其包装为“任意”聚合函数或将其添加到中的键中`GROUP BY`。

# 算术函数

对于所有算术函数，如果存在结果类型，则将其计算为结果适合的最小数字类型。最小值根据位数，是否带符号以及是否浮动而同时获取。如果没有足够的位，则采用最高位类型。

例：

选择 toTypeName （0 ）， toTypeName （0 \+ 0 ）， toTypeName （0 \+ 0 \+ 0 ）， toTypeName （0 \+ 0 \+ 0 \+ 0 ）

─toTypeName（0）─┬─toTypeName（plus（0，0））─┬─toTypeName（plus（plus（0，0），0））─┬─toTypeName（plus（plus（plus（plus（0，0）） ，0），0））─┐
│UInt8│UInt16│UInt32│UInt64│
┴──────────────┴┴────────┴ ────────────────────┴──┴──────────────────── ────────────────┘

算术函数适用于 UInt8，UInt16，UInt32，UInt64，Int8，Int16，Int32，Int64，Float32 或 Float64 中的任何类型的对。

溢出的产生方式与 C ++中的产生方式相同。

## plus（a，b），a + b 运算符[¶](https://clickhouse.yandex/docs/en/query_language/functions/arithmetic_functions/#plus-a-b-a-b-operator "永久链接")

计算数字的总和。您还可以添加带有日期或日期和时间的整数。对于日期，添加整数意味着添加相应的天数。对于带时间的日期，意味着要加上相应的秒数。

## minus（a，b），a-b 运算符[¶](https://clickhouse.yandex/docs/en/query_language/functions/arithmetic_functions/#minus-a-b-a-b-operator "永久链接")

计算差异。结果总是签名的。

您还可以根据日期或带时间的日期计算整数。想法是一样的-见上文“加号”。

## 乘法（a，b），a \* b 运算符[¶](https://clickhouse.yandex/docs/en/query_language/functions/arithmetic_functions/#multiply-a-b-a-42-b-operator "永久链接")

计算数字的乘积。

## 除法（a，b），a / b 运算符[¶](https://clickhouse.yandex/docs/en/query_language/functions/arithmetic_functions/#divide-a-b-a-b-operator "永久链接")

计算数字的商。结果类型始终是浮点类型。它不是整数除法。对于整数除法，请使用“ intDiv”函数。除以零时，将得到“ inf”，“-inf”或“ nan”。

## intDiv（a，b）[¶](https://clickhouse.yandex/docs/en/query_language/functions/arithmetic_functions/#intdiv-a-b "永久链接")

计算数字的商。分成整数，四舍五入（按绝对值）。除以零或最小负数除以负一时，将引发异常。

## intDivOrZero（a，b）[¶](https://clickhouse.yandex/docs/en/query_language/functions/arithmetic_functions/#intdivorzero-a-b "永久链接")

与“ intDiv”的不同之处在于，除以零或最小负数除以负一时，它返回零。

## modulo（a，b），％b 运算符[¶](https://clickhouse.yandex/docs/en/query_language/functions/arithmetic_functions/#modulo-a-b-a-b-operator "永久链接")

计算除法后的余数。如果参数是浮点数，则通过删除小数部分将它们预先转换为整数。其余部分的含义与 C ++中的含义相同。截断除法用于负数。除以零或最小负数除以负一时，将引发异常。

## negate（a），-a 运算符[¶](https://clickhouse.yandex/docs/en/query_language/functions/arithmetic_functions/#negate-a-a-operator "永久链接")

计算带反号的数字。结果总是签名的。

## abs（a）[¶](https://clickhouse.yandex/docs/en/query_language/functions/arithmetic_functions/#arithm_func-abs "永久链接")

计算数字的绝对值（a）。也就是说，如果 a <0，则返回-a。对于无符号类型，它不执行任何操作。对于有符号整数类型，它将返回无符号数字。

## gcd（a，b）[¶](https://clickhouse.yandex/docs/en/query_language/functions/arithmetic_functions/#gcd-a-b "永久链接")

返回数字的最大公约数。除以零或最小负数除以负一时，将引发异常。

## lcm（a，b）[¶](https://clickhouse.yandex/docs/en/query_language/functions/arithmetic_functions/#lcm-a-b "永久链接")

返回数字的最小公倍数。除以零或最小负数除以负一时，将引发异常。

# 比较功能

比较函数始终返回 0 或 1（Uint8）。

可以比较以下类型：

- 数字
- 弦和固定弦
- 日期
- 与时间约会

在每个组中，但不在不同组之间。

例如，您不能将日期与字符串进行比较。您必须使用函数将字符串转换为日期，反之亦然。

字符串按字节进行比较。较短的字符串比以它开头且包含至少一个以上字符的所有字符串小。

注意。直到版本 1.1.54134 为止，已签名和未签名的数字都以与 C ++中相同的方式进行比较。换句话说，在类似 SELECT 9223372036854775807> -1 的情况下，您可能会得到错误的结果。在版本 1.1.54134 中，此行为已更改，现在在数学上是正确的。

## 等于 a = b 和 a == b 运算符[¶](https://clickhouse.yandex/docs/en/query_language/functions/comparison_functions/#function-equals "永久链接")

## 不等于，一个！运算符= b 和`<>`a [b¶](https://clickhouse.yandex/docs/en/query_language/functions/comparison_functions/#function-notequals "永久链接")

## 更少，`< operator`[¶](https://clickhouse.yandex/docs/en/query_language/functions/comparison_functions/#function-less "永久链接")

## 更大，`> operator`[¶](https://clickhouse.yandex/docs/en/query_language/functions/comparison_functions/#function-greater "永久链接")

## lessOrEquals，`<= operator`[¶](https://clickhouse.yandex/docs/en/query_language/functions/comparison_functions/#function-lessorequals "永久链接")

## GreaterOrEquals，`>= operator`

# 逻辑功能

逻辑函数接受任何数字类型，但是返回等于 0 或 1 的 UInt8 数字。

零作为参数被认为是“ false”，而任何非零值都被认为是“ true”。

## 和 AND 运算符[¶](https://clickhouse.yandex/docs/en/query_language/functions/logical_functions/#and-and-operator "永久链接")

## 或 OR 运算符[¶](https://clickhouse.yandex/docs/en/query_language/functions/logical_functions/#or-or-operator "永久链接")

## 不是，不是运算符[¶](https://clickhouse.yandex/docs/en/query_language/functions/logical_functions/#not-not-operator "永久链接")

## XOR [¶](https://clickhouse.yandex/docs/en/query_language/functions/logical_functions/#xor "永久链接")

# 类型转换功能

## 数值转换的常见问题[¶](https://clickhouse.yandex/docs/en/query_language/functions/type_conversion_functions/#numeric-conversion-issues "永久链接")

当您将一个值从一个数据类型转换为另一种数据类型时，您应该记住，在通常情况下，这是一种不安全的操作，可能会导致数据丢失。如果您尝试将值从较大的数据类型调整为较小的数据类型，或者在不同的数据类型之间转换值，则可能会发生数据丢失。

ClickHouse [与 C ++程序](https://en.cppreference.com/w/cpp/language/implicit_conversion)具有[相同的行为](https://en.cppreference.com/w/cpp/language/implicit_conversion)。

## toInt（8 | 16 | 32 | 64）[¶](https://clickhouse.yandex/docs/en/query_language/functions/type_conversion_functions/#toint-8-16-32-64 "永久链接")

将输入值转换为[Int](https://clickhouse.yandex/docs/en/data_types/int_uint/)数据类型。该功能系列包括：

- `toInt8(expr)`—结果为`Int8`数据类型。
- `toInt16(expr)`—结果为`Int16`数据类型。
- `toInt32(expr)`—结果为`Int32`数据类型。
- `toInt64(expr)`—结果为`Int64`数据类型。

**参量**

- `expr`—  返回一个数字或带有数字十进制表示形式的字符串的[表达式](https://clickhouse.yandex/docs/en/query_language/syntax/#syntax-expressions)。不支持数字的二进制，八进制和十六进制表示形式。前导零被去除。

**返回值**

在整数值`Int8`，`Int16`，`Int32`，或`Int64`数据类型。

函数使用[向零舍入](https://en.wikipedia.org/wiki/Rounding#Rounding_towards_zero)，这意味着它们会截断数字的小数位数。

[NaN 和 Inf](https://clickhouse.yandex/docs/en/data_types/float/#data_type-float-nan-inf)参数的函数行为未定义。使用函数时，请记住有关[数字转换的问题](https://clickhouse.yandex/docs/en/query_language/functions/type_conversion_functions/#numeric-conversion-issues)。

**例**

SELECT toInt64 （楠）， toInt32 （32 ）， toInt16 （'16' ）， toInt8 （8 。8 ）

to─────────toInt64（nan）─┬─toInt32（32）─┬─toInt16（'16'）─┬─toInt8（8.8）─┐
│-9223372036854775808│32│16│8│
└────────────────┴┴────────┴ ────┴────────────

## toInt（8 | 16 | 32 | 64）OrZero [¶](https://clickhouse.yandex/docs/en/query_language/functions/type_conversion_functions/#toint-8-16-32-64-orzero "永久链接")

## toInt（8 | 16 | 32 | 64）OrNull [¶](https://clickhouse.yandex/docs/en/query_language/functions/type_conversion_functions/#toint-8-16-32-64-ornull "永久链接")

## toUInt（8 | 16 | 32 | 64）[¶](https://clickhouse.yandex/docs/en/query_language/functions/type_conversion_functions/#touint-8-16-32-64 "永久链接")

将输入值转换为[UInt](https://clickhouse.yandex/docs/en/data_types/int_uint/)数据类型。该功能系列包括：

- `toUInt8(expr)`—结果为`UInt8`数据类型。
- `toUInt16(expr)`—结果为`UInt16`数据类型。
- `toUInt32(expr)`—结果为`UInt32`数据类型。
- `toUInt64(expr)`—结果为`UInt64`数据类型。

**参量**

- `expr`—  返回一个数字或带有数字十进制表示形式的字符串的[表达式](https://clickhouse.yandex/docs/en/query_language/syntax/#syntax-expressions)。不支持数字的二进制，八进制和十六进制表示形式。前导零被去除。

**返回值**

在整数值`UInt8`，`UInt16`，`UInt32`，或`UInt64`数据类型。

函数使用[向零舍入](https://en.wikipedia.org/wiki/Rounding#Rounding_towards_zero)，这意味着它们会截断数字的小数位数。

未定义负加和[NaN 和 Inf](https://clickhouse.yandex/docs/en/data_types/float/#data_type-float-nan-inf)参数的函数行为。例如`'-32'`，如果传递带有负数的字符串，则 ClickHouse 会引发异常。使用函数时，请记住有关[数字转换的问题](https://clickhouse.yandex/docs/en/query_language/functions/type_conversion_functions/#numeric-conversion-issues)。

**例**

SELECT toUInt64 （楠）， toUInt32 （\- 32 ）， toUInt16 （'16' ）， toUInt8 （8 。8 ）

───────toUInt64（nan）─┬─toUInt32（-32）─┬─toUInt16（'16'）─┬─toUInt8（8.8）─┐
│9223372036854775808│4294967264│16│8│
└────────────────┴────┴────────┴ ──────┴──────────

## toUInt（8 | 16 | 32 | 64）OrZero [¶](https://clickhouse.yandex/docs/en/query_language/functions/type_conversion_functions/#touint-8-16-32-64-orzero "永久链接")

## toUInt（8 | 16 | 32 | 64）OrNull [¶](https://clickhouse.yandex/docs/en/query_language/functions/type_conversion_functions/#touint-8-16-32-64-ornull "永久链接")

## toFloat（32 | 64）[¶](https://clickhouse.yandex/docs/en/query_language/functions/type_conversion_functions/#tofloat-32-64 "永久链接")

## toFloat（32 | 64）OrZero [¶](https://clickhouse.yandex/docs/en/query_language/functions/type_conversion_functions/#tofloat-32-64-orzero "永久链接")

## toFloat（32 | 64）OrNull [¶](https://clickhouse.yandex/docs/en/query_language/functions/type_conversion_functions/#tofloat-32-64-ornull "永久链接")

## TODATE [¶](https://clickhouse.yandex/docs/en/query_language/functions/type_conversion_functions/#todate "永久链接")

## toDateOrZero [¶](https://clickhouse.yandex/docs/en/query_language/functions/type_conversion_functions/#todateorzero "永久链接")

## toDateOrNull [¶](https://clickhouse.yandex/docs/en/query_language/functions/type_conversion_functions/#todateornull "永久链接")

## toDateTime [¶](https://clickhouse.yandex/docs/en/query_language/functions/type_conversion_functions/#todatetime "永久链接")

## toDateTimeOrZero [¶](https://clickhouse.yandex/docs/en/query_language/functions/type_conversion_functions/#todatetimeorzero "永久链接")

## toDateTimeOrNull [¶](https://clickhouse.yandex/docs/en/query_language/functions/type_conversion_functions/#todatetimeornull "永久链接")

## toDecimal（32 | 64 | 128）[¶](https://clickhouse.yandex/docs/en/query_language/functions/type_conversion_functions/#todecimal-32-64-128 "永久链接")

转换`value`为精度为的[十进制](https://clickhouse.yandex/docs/en/data_types/decimal/)数据类型`S`。该`value`可以是一个数字或一个字符串。在`S`（规模）参数指定小数位的数量。

- `toDecimal32(value, S)`
- `toDecimal64(value, S)`
- `toDecimal128(value, S)`

## toDecimal（32 | 64 | 128）OrNull [¶](https://clickhouse.yandex/docs/en/query_language/functions/type_conversion_functions/#todecimal-32-64-128-ornull "永久链接")

将输入字符串转换为[Nullable（Decimal（P，S））](https://clickhouse.yandex/docs/en/data_types/decimal/)数据类型值。该功能家族包括：

- `toDecimal32OrNull(expr, S)`—结果为`Nullable(Decimal32(S))`数据类型。
- `toDecimal64OrNull(expr, S)`—结果为`Nullable(Decimal64(S))`数据类型。
- `toDecimal128OrNull(expr, S)`—结果为`Nullable(Decimal128(S))`数据类型。

`toDecimal*()`如果您更喜欢`NULL`在输入值解析错误的情况下获取值而不是异常，则应使用这些函数代替函数。

**参量**

- `expr`— [Expression](https://clickhouse.yandex/docs/en/query_language/syntax/#syntax-expressions)，返回[String](https://clickhouse.yandex/docs/en/data_types/string/)数据类型的值。ClickHouse 需要十进制数字的文本表示形式。例如，`'1.111'`。
- `S` —小数位数，结果值的小数位数。

**返回值**

`Nullable(Decimal(P,S))`数据类型中的值。该值包含：

- `S`如果 ClickHouse 将输入字符串解释为数字，则带小数位的数字。
- `NULL`，如果 ClickHouse 无法将输入字符串解释为数字，或者输入数字包含的`S`位数不止小数。

**例子**

SELECT toDecimal32OrNull （的 toString （\- 1 。111 ）， 5 ） AS VAL ， toTypeName （VAL ）

┌──────val─┬─toTypeName（toDecimal32OrNull（toString（-1.111），5））─┐
│-1.11100│ 可为空（Decimal（9，5））│
└──────────┴────────────────────── ────────────┘

SELECT toDecimal32OrNull （的 toString （\- 1 。111 ）， 2 ） AS VAL ， toTypeName （VAL ）

┌──val─┬─toTypeName（toDecimal32OrNull（toString（-1.111），2））─┐
│ᴺᵁᴸᴸ│ 可为空（Decimal（9，2））│
──────┴────────────────────────── ──────────┘

## toDecimal（32 | 64 | 128）OrZero [¶](https://clickhouse.yandex/docs/en/query_language/functions/type_conversion_functions/#todecimal-32-64-128-orzero "永久链接")

将输入值转换为[Decimal（P，S）](https://clickhouse.yandex/docs/en/data_types/decimal/)数据类型。该功能家族包括：

- `toDecimal32OrZero( expr, S)`—结果为`Decimal32(S)`数据类型。
- `toDecimal64OrZero( expr, S)`—结果为`Decimal64(S)`数据类型。
- `toDecimal128OrZero( expr, S)`—结果为`Decimal128(S)`数据类型。

`toDecimal*()`如果您更喜欢`0`在输入值解析错误的情况下获取值而不是异常，则应使用这些函数代替函数。

**参量**

- `expr`— [Expression](https://clickhouse.yandex/docs/en/query_language/syntax/#syntax-expressions)，返回[String](https://clickhouse.yandex/docs/en/data_types/string/)数据类型的值。ClickHouse 需要十进制数字的文本表示形式。例如，`'1.111'`。
- `S` —小数位数，结果值的小数位数。

**返回值**

`Nullable(Decimal(P,S))`数据类型中的值。该值包含：

- `S`如果 ClickHouse 将输入字符串解释为数字，则带小数位的数字。
- 0（带`S`小数位），如果 ClickHouse 无法将输入字符串解释为数字，或者输入的数字包含多个`S`小数位。

**例**

SELECT toDecimal32OrZero （的 toString （\- 1 。111 ）， 5 ） AS VAL ， toTypeName （VAL ）

┌──────val─┬─toTypeName（toDecimal32OrZero（toString（-1.111），5））─┐
│-1.11100│ 小数（9，5）│
└──────────┴────────────────────── ────────────┘

SELECT toDecimal32OrZero （的 toString （\- 1 。111 ）， 2 ） AS VAL ， toTypeName （VAL ）

┌──val─┬─toTypeName（toDecimal32OrZero（toString（-1.111），2））─┐
│0.00│ 小数（9，2）│
──────┴────────────────────────── ──────────┘

## 的 toString [¶](https://clickhouse.yandex/docs/en/query_language/functions/type_conversion_functions/#tostring "永久链接")

用于在数字，字符串（但不是固定字符串），日期和带日期的日期之间进行转换的函数。所有这些函数都接受一个参数。

在转换为字符串或从字符串转换时，将使用与 TabSeparated 格式（以及几乎所有其他文本格式）相同的规则来格式化或解析该值。如果无法解析该字符串，则将引发异常并取消请求。

将日期转换为数字时（反之亦然），该日期对应于自 Unix 纪元开始以来的天数。将带时间的日期转换为数字时，反之亦然，带时间的日期对应于自 Unix 时代开始以来的秒数。

toDate / toDateTime 函数的日期和带时间的日期格式定义如下：

YYYY-MM-DD
YYYY-MM-DD hh：mm：ss

作为例外，如果将 UInt32，Int32，UInt64 或 Int64 数字类型转换为 Date，并且该数字大于或等于 65536，则该数字将解释为 Unix 时间戳（而不是天数），并且四舍五入到日期。这样可以支持写'toDate（unix_timestamp）'的常见情况，否则将是一个错误，并且需要编写更麻烦的'toDate（toDateTime（unix_timestamp））'。

日期和日期与时间之间的转换是自然的方式：通过添加空时间或减少时间。

数字类型之间的转换使用与 C ++中不同数字类型之间的分配相同的规则。

此外，DateTime 参数的 toString 函数可以采用另一个包含时区名称的 String 参数。示例：`Asia/Yekaterinburg`在这种情况下，时间将根据指定的时区进行格式化。

选择
现在（） AS now_local ，
的 toString （今（）， '亚洲/叶卡捷琳堡' ） AS now_yekat

───────────now_local─┬─now_yekat───────────┐
│2016-06-15 00:11:21│2016-06-15 02:11:21│
┘────────────────┴┴────────────┘

另请参见`toUnixTimestamp`功能。

## toFixedString（s，N）[¶](https://clickhouse.yandex/docs/en/query_language/functions/type_conversion_functions/#tofixedstring-s-n "永久链接")

将 String 类型的参数转换为 FixedString（N）类型（固定长度为 N 的字符串）。N 必须为常数。如果字符串的字节数少于 N，则将其与空字节一起传递到右侧。如果字符串的字节数大于 N，则将引发异常。

## toStringCutToZero（s）[¶](https://clickhouse.yandex/docs/en/query_language/functions/type_conversion_functions/#tostringcuttozero-s "永久链接")

接受 String 或 FixedString 参数。返回字符串，其内容在找到的第一个零字节处被截断。

例：

选择 toFixedString （'foo' ， 8 ） AS s ， toStringCutToZero （s ） AS s_cut

┌─s─────────────┬─s_cut─┐
│foo \ 0 \ 0 \ 0 \ 0 \ 0│foo│
└────────────┴

选择 toFixedString （'foo \ 0bar' ， 8 ） AS s ， toStringCutToZero （s ） AS s_cut

┌─s──────────┬─s_cut─┐
│foo \ 0bar \ 0│foo│
└──────────┴

## reinterpretAsUInt（8 | 16 | 32 | 64）[¶](https://clickhouse.yandex/docs/en/query_language/functions/type_conversion_functions/#reinterpretasuint-8-16-32-64 "永久链接")

## reinterpretAsInt（8 | 16 | 32 | 64）[¶](https://clickhouse.yandex/docs/en/query_language/functions/type_conversion_functions/#reinterpretasint-8-16-32-64 "永久链接")

## reinterpretAsFloat（32 | 64）[¶](https://clickhouse.yandex/docs/en/query_language/functions/type_conversion_functions/#reinterpretasfloat-32-64 "永久链接")

## reinterpretAsDate [¶](https://clickhouse.yandex/docs/en/query_language/functions/type_conversion_functions/#reinterpretasdate "永久链接")

## reinterpretAsDateTime [¶](https://clickhouse.yandex/docs/en/query_language/functions/type_conversion_functions/#reinterpretasdatetime "永久链接")

这些函数接受一个字符串，并将字符串开头的字节解释为主机顺序中的数字（小端）。如果字符串不够长，则函数将像使用必要数量的空字节填充字符串一样工作。如果字符串长于所需长度，则多余的字节将被忽略。日期被解释为自 Unix 纪元开始以来的天数，日期和时间被解释为自 Unix 纪元开始以来的秒数。

## reinterpretAsString [¶](https://clickhouse.yandex/docs/en/query_language/functions/type_conversion_functions/#type_conversion_functions-reinterpretAsString "永久链接")

此函数接受带时间的数字，日期或日期，并返回一个字符串，该字符串包含表示主机顺序（小尾数）的相应值的字节。从末尾丢弃空字节。例如，UInt32 类型值 255 是一个一字节长的字符串。

## reinterpretAsFixedString [¶](https://clickhouse.yandex/docs/en/query_language/functions/type_conversion_functions/#reinterpretasfixedstring "永久链接")

此函数接受带时间的数字，日期或日期，并返回一个 FixedString，其中包含表示主机顺序（小尾数）的相应值的字节。从末尾丢弃空字节。例如，UInt32 类型值为 255 的是固定字符串，长度为 1 个字节。

## CAST（x，t）[¶](https://clickhouse.yandex/docs/en/query_language/functions/type_conversion_functions/#type_conversion_function-cast "永久链接")

将“ x”转换为“ t”数据类型。还支持语法 CAST（x AS t）。

例：

选择
'2016 年 6 月 15 日 23:00:00' AS 时间戳，
CAST （时间戳 AS 的 DateTime ） AS 日期时间，
CAST （时间戳 AS 日期） AS 日期，
CAST （时间戳， '字符串' ） AS 字符串，
CAST （时间戳， “ FixedString（22）' ） AS fixed_string

─timestamp────────────┬ 日期 ──date─ 字符串 ──────── ────────── 固定字符串 ──────────────
│2016-06-15 23:00:00│2016-06-15 23:00:00│2016-06-15│2016-06-15 23:00:00│2016-06-15 23:00:00 \ 0 \ 0 \ 0│
────────────────┴────┴──────────┴ ──────┴──────────────────┴────────────────── ───────┘

转换为 FixedString（N）仅适用于 String 或 FixedString（N）类型的参数。

支持将类型转换为[Nullable](https://clickhouse.yandex/docs/en/data_types/nullable/)并返回。例：

从 t_null 选择 toTypeName （x ）

┌─toTypeName（x）─┐
│Int8│
│Int8│
└──────────────┘

SELECT toTypeName （CAST （x ， 'Nullable（UInt16）' ）） FROM t_null

to─toTypeName（CAST（x，'Nullable（UInt16）'））─┐
│ 可为空（UInt16）│
│ 可为空（UInt16）│
┘────────────────┘

## toInterval（年|季度|月|周|周|天|小时|分钟|第二）[¶](https://clickhouse.yandex/docs/en/query_language/functions/type_conversion_functions/#tointerval-year-quarter-month-week-day-hour-minute-second "永久链接")

将 Number 类型的参数转换为 Interval 类型（持续时间）。间隔类型实际上非常有用，您可以使用这种类型的数据直接通过 Date 或 DateTime 执行算术运算。同时，ClickHouse 提供了更方便的语法来声明 Interval 类型数据。例如：

WITH
toDate （'2019-01-01' ） AS date ，
INTERVAL 1 WEEK AS interval_week ，
toIntervalWeek （1 ） AS interval_to_week
SELECT
date \+ interval_week ，
date \+ interval_to_week

┬─ 加（日期，间隔周）─┬─ 加（日期，间隔周）─┐
│2019-01-08│2019-01-08│
──────────────────────┴────────────────── ─────────┘

## parseDateTimeBestEffort [¶](https://clickhouse.yandex/docs/en/query_language/functions/type_conversion_functions/#type_conversion_functions-parsedatetimebesteffort "永久链接")

将数字类型参数解析为 Date 或 DateTime 类型。与 toDate 和 toDateTime 不同，parseDateTimeBestEffort 可以处理更复杂的日期格式。有关更多信息，请参见链接：[复杂日期格式](https://xkcd.com/1179/)

## parseDateTimeBestEffortOrNull [¶](https://clickhouse.yandex/docs/en/query_language/functions/type_conversion_functions/#parsedatetimebesteffortornull "永久链接")

与[parseDateTimeBestEffort](https://clickhouse.yandex/docs/en/query_language/functions/type_conversion_functions/#type_conversion_functions-parsedatetimebesteffort)相同，[不同之](https://clickhouse.yandex/docs/en/query_language/functions/type_conversion_functions/#type_conversion_functions-parsedatetimebesteffort)处在于当遇到无法处理的日期格式时返回 null。

## parseDateTimeBestEffortOrZero [¶](https://clickhouse.yandex/docs/en/query_language/functions/type_conversion_functions/#parsedatetimebesteffortorzero "永久链接")

与[parseDateTimeBestEffort](https://clickhouse.yandex/docs/en/query_language/functions/type_conversion_functions/#type_conversion_functions-parsedatetimebesteffort)相同，除了遇到无法处理的日期格式时返回零日期或零日期时间。

# 处理日期和时间的功能

支持时区

在时区上有逻辑用途的所有用于日期和时间的函数都可以接受第二个可选的时区参数。例如：亚洲/叶卡捷琳堡。在这种情况下，他们使用指定的时区而不是本地（默认）时区。

SELECT
toDateTime （'2016-06-15 23:00:00' ） AS time ，
toDate （time ） AS date_local ，
toDate （time ， 'Asia / Yekaterinburg' ） AS date_yekat ，
toString （time ， 'US / Samoa' ） AS 萨摩亚

time────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
│2016-06-15 23:00:00│2016-06-15│2016-06-16│2016-06-15 09:00:00│
└────────────────┴┴──────┴──────── ────────────────

仅支持与 UTC 相差整小时的时区。

## toTimeZone [¶](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/#totimezone "永久链接")

将时间或日期和时间转换为指定的时区。

## toYear [¶](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/#toyear "永久链接")

将日期或带时间的日期转换为包含年份数字（AD）的 UInt16 数字。

## toQuarter [¶](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/#toquarter "永久链接")

将日期或带时间的日期转换为包含四分之一数字的 UInt8 数字。

## toMonth [¶](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/#tomonth "永久链接")

将日期或带时间的日期转换为包含月份数字（1-12）的 UInt8 数字。

## toDayOfYear [¶](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/#todayofyear "永久链接")

将日期或带时间的日期转换为包含一年中某一天的数字（1-366）的 UInt16 数字。

## toDayOfMonth [¶](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/#todayofmonth "永久链接")

将日期或带时间的日期转换为包含月中某一天的数字（1-31）的 UInt8 数字。

## toDayOfWeek [¶](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/#todayofweek "永久链接")

将日期或带时间的日期转换为包含星期几（星期一为 1，星期日为 7）的 UInt8 数字。

## toHour [¶](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/#tohour "永久链接")

将带时间的日期转换为包含 24 小时制（0-23）中小时数的 UInt8 数字。此功能假定如果时钟向前移动，则将其调动一小时并在凌晨 2 点发生；如果时钟向后移动，则将其调动一小时并于凌晨 3 点发生（这并不总是正确的-即使在莫斯科，时钟也是如此）在不同的时间两次更改）。

## toMinute [¶](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/#tominute "永久链接")

将带时间的日期转换为包含小时的分钟数（0-59）的 UInt8 数字。

## toSecond [¶](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/#tosecond "永久链接")

将带时间的日期转换为 UInt8 数字，其中包含分钟的秒数（0-59）。seconds 秒不计算在内。

## toUnixTimestamp [¶](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/#tounixtimestamp "永久链接")

将带时间的日期转换为 Unix 时间戳。

## toStartOfYear [¶](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/#tostartofyear "永久链接")

将日期或日期与时间四舍五入到一年的第一天。返回日期。

## toStartOfISOYear [¶](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/#tostartofisoyear "永久链接")

将日期或日期与时间四舍五入到 ISO 年的第一天。返回日期。

## toStartOfQuarter [¶](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/#tostartofquarter "永久链接")

将日期或带时间的日期四舍五入到该季度的第一天。该季度的第一天是 1 月 1 日，4 月 1 日，7 月 1 日或 10 月 1 日。返回日期。

## toStartOfMonth [¶](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/#tostartofmonth "永久链接")

将日期或带时间的日期四舍五入到该月的第一天。返回日期。

注意

解析错误日期的行为是特定于实现的。ClickHouse 可能会返回零日期，引发异常或“自然”溢出。

## toMonday [¶](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/#tomonday "永久链接")

将日期或带时间的日期四舍五入到最近的星期一。返回日期。

## toStartOfWeek（t \[，mode\]）[¶](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/#tostartofweek-t-mode "永久链接")

通过模式将日期或日期与时间四舍五入到最近的星期日或星期一。返回日期。模式参数的工作方式与 toWeek（）的模式参数完全相同。对于单参数语法，使用模式值 0。

## toStartOfDay [¶](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/#tostartofday "永久链接")

将时间与时间四舍五入到一天的开始。

## toStartOfHour [¶](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/#tostartofhour "永久链接")

将日期和时间四舍五入到小时的开始。

## toStartOfMinute [¶](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/#tostartofminute "永久链接")

将日期和时间四舍五入到分钟的开始。

## toStartOfFiveMinute [¶](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/#tostartoffiveminute "永久链接")

将时间与时间四舍五入到五分钟间隔的开始。

## toStartOfTenMinutes [¶](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/#tostartoftenminutes "永久链接")

将日期和时间四舍五入到十分钟间隔的开始。

## toStartOfFifteenMinutes [¶](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/#tostartoffifteenminutes "永久链接")

将日期和时间四舍五入到 15 分钟间隔的开始。

## toStartOfInterval（time_or_data，INTERVAL x unit \[，time_zone\]）[¶](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/#tostartofinterval-time-or-data-interval-x-unit-time-zone "永久链接")

这是名为的其他函数的概括`toStartOf*`。例如，  
`toStartOfInterval(t, INTERVAL 1 year)`返回与相同`toStartOfYear(t)`，与  
`toStartOfInterval(t, INTERVAL 1 month)`返回相同`toStartOfMonth(t)`，与  
`toStartOfInterval(t, INTERVAL 1 day)`返回相同`toStartOfDay(t)`，与  
`toStartOfInterval(t, INTERVAL 15 minute)`返回`toStartOfFifteenMinutes(t)`等。

## TOTIME [¶](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/#totime "永久链接")

将日期和时间转换为某个固定日期，同时保留时间。

## toRelativeYearNum [¶](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/#torelativeyearnum "永久链接")

从过去的某个固定点开始，将带有时间或日期的日期转换为年份。

## toRelativeQuarterNum [¶](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/#torelativequarternum "永久链接")

从过去的某个固定点开始，将带有时间或日期的日期转换为季度数。

## toRelativeMonthNum [¶](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/#torelativemonthnum "永久链接")

从过去的某个固定点开始，将带有时间或日期的日期转换为月份数。

## toRelativeWeekNum [¶](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/#torelativeweeknum "永久链接")

从过去的某个固定点开始，将带有时间或日期的日期转换为星期几。

## toRelativeDayNum [¶](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/#torelativedaynum "永久链接")

从过去的某个固定点开始，将带有时间或日期的日期转换为天数。

## toRelativeHourNum [¶](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/#torelativehournum "永久链接")

从过去的某个固定点开始，将带有时间或日期的日期转换为小时数。

## toRelativeMinuteNum [¶](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/#torelativeminutenum "永久链接")

从过去的某个固定点开始，将带有时间或日期的日期转换为分钟数。

## toRelativeSecondNum [¶](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/#torelativesecondnum "永久链接")

从过去的某个固定点开始，将带有时间或日期的日期转换为秒数。

## toISOYear [¶](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/#toisoyear "永久链接")

将日期或带时间的日期转换为包含 ISO 年份编号的 UInt16 编号。

## toISOWeek [¶](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/#toisoweek "永久链接")

将日期或带时间的日期转换为包含 ISO 周编号的 UInt8 数字。

## toWeek（date \[，mode\]）[¶](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/#toweek-date-mode "永久链接")

此函数返回日期或日期时间的星期数。toWeek（）的两个参数形式使您可以指定星期是从星期日还是星期一开始，以及返回值应在 0 到 53 还是从 1 到 53 的范围内。如果省略了 mode 参数，则默认 mode 为 0。`toISOWeek()`是等效于的兼容性函数`toWeek(date,3)`。下表描述了 mode 参数的工作方式。

模式

一周的第一天

范围

第一周是第一周…

0

星期日

0-53

今年的一个星期天

1 个

星期一

0-53

今年有四天或以上

2

星期日

1-53

今年的一个星期天

3

星期一

1-53

今年有四天或以上

4

星期日

0-53

今年有四天或以上

5

星期一

0-53

今年的星期一

6

星期日

1-53

今年有四天或以上

7

星期一

1-53

今年的星期一

8

星期日

1-53

包含 1 月 1 日

9

星期一

1-53

包含 1 月 1 日

对于含义为“今年有 4 天或更多天”的模式值，根据 ISO 8601：1988 对周进行编号：

- 如果包含 1 月 1 日的一周在新年中有 4 天或更多天，则为第 1 周。

- 否则，它是上一年的最后一周，下周是第 1 周。

对于含义为“包含 1 月 1 日”的模式值，包含 1 月 1 日的星期就是星期 1。包含一周的新一年中有多少天都没有关系，即使它只包含一天。

toWeek （日期， \[， 模式\] \[， 时区\]）

**参量**

- `date` –日期或日期时间。
- `mode` –可选参数，值的范围为\[0,9\]，默认值为 0。
- `Timezone` –可选参数，其行为类似于任何其他转换函数。

**例**

SELECT TODATE （'2016 年 12 月 27 日' ） AS 日期， toWeek （日期） AS week0 ， toWeek （日期，1 ） AS week1 ， toWeek （日期，9 ） AS week9 ;

─────── 日期 ─┬─ 星期 0─┬─ 星期 1─┬─ 星期 9─┐
│2016-12-27│52│52│1│
────────────┴──────┴────────┴────────

## toYearWeek（date \[，mode\]）[¶](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/#toyearweek-date-mode "永久链接")

返回日期的年和周。结果中的年份可能与该年份的第一周和最后一周的 date 参数中的年份不同。

模式参数的工作方式与 toWeek（）的模式参数完全相同。对于单参数语法，使用模式值 0。

`toISOYear()`是等效于的兼容性函数`intDiv(toYearWeek(date,3),100)`。

**例**

SELECT toDate （'2016-12-27' ） AS date ， toYearWeek （date ） AS yearWeek0 ， toYearWeek （date ，1 ） AS yearWeek1 ， toYearWeek （date ，9 ） AS yearWeek9 ;

┌─────── 日期 ─┬─yearWeek0─┬─yearWeek1─┬─yearWeek9─┐
│2016-12-27│201652│201652│201701│
────────────┴─────────┴──────┴

## 现在[¶](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/#now "永久链接")

接受零参数，并在请求执行的时刻之一返回当前时间。即使请求花费很长时间才能完成，此函数也会返回一个常量。

## 今天[¶](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/#today "永久链接")

接受零参数，并在请求执行的时刻之一返回当前日期。与“ toDate（now（））”相同。

## 昨天[¶](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/#yesterday "永久链接")

接受零参数，并在请求执行的时刻之一返回昨天的日期。与'today（）-1'相同。

## 时隙[¶](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/#timeslot "永久链接")

将时间舍入为半小时。此功能特定于 Yandex.Metrica，因为如果跟踪标签显示单个用户的连续网页浏览的时间差严格大于此数量，则半小时是将会话分成两个会话的最短时间。这意味着元组（标记 ID，用户 ID 和时隙）可用于搜索相应会话中包含的综合浏览量。

## toYYYYMM [¶](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/#toyyyymm "永久链接")

将日期或带时间的日期转换为包含年和月数字的 UInt32 数字（YYYY \* 100 + MM）。

## toYYYYMMDD [¶](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/#toyyyymmdd "永久链接")

将日期或带时间的日期转换为包含年和月数字的 UInt32 数字（YYYY _ 10000 + MM _ 100 + DD）。

## toYYYYMMDDhhmmss [¶](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/#toyyyymmddhhmmss "永久链接")

将日期或带时间的日期转换为包含年和月数字的 UInt64 数字（YYYY _ 10000000000 + MM _ 100000000 + DD _ 1000000 + hh _ 10000 + mm \* 100 + ss）。

## addYears，addMonths，addWeeks，addDays，addHours，addMinutes，addSeconds，addQuarters [¶](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/#addyears-addmonths-addweeks-adddays-addhours-addminutes-addseconds-addquarters "永久链接")

函数将 Date / DateTime 间隔添加到 Date / DateTime，然后返回 Date / DateTime。例如：

WITH
toDate （'2018-01-01' ） AS date ，
toDateTime （' 2018-01-01 00:00:00' ） AS date_time
SELECT
addYears （date ， 1 ） AS add_years_with_date ，
addYears （date_time ， 1 ） AS add_years_with_date_time

┌─add_years_with_date─┬──add_years_with_date_time─┐
│2019-01-01│2019-01-01 00:00:00│
┘────────────────┴┴────────┘

## 减去年份，减去月份，减去周，减去天，减去[小时](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/#subtractyears-subtractmonths-subtractweeks-subtractdays-subtracthours-subtractminutes-subtractseconds-subtractquarters "永久链接")，减去[分钟](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/#subtractyears-subtractmonths-subtractweeks-subtractdays-subtracthours-subtractminutes-subtractseconds-subtractquarters "永久链接")，减去[秒](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/#subtractyears-subtractmonths-subtractweeks-subtractdays-subtracthours-subtractminutes-subtractseconds-subtractquarters "永久链接")，减去[四分之一](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/#subtractyears-subtractmonths-subtractweeks-subtractdays-subtracthours-subtractminutes-subtractseconds-subtractquarters "永久链接")

函数将 Date / DateTime 间隔减去 Date / DateTime，然后返回 Date / DateTime。例如：

WITH
TODATE （'2019 年 1 月 1 日' ） AS 日期，
toDateTime （'2019 年 1 月 1 日 00:00:00' ） AS DATE_TIME
SELECT
subtractYears （日期， 1 ） AS subtract_years_with_date ，
subtractYears （DATE_TIME ， 1 ） AS subtract_years_with_date_time

┌─subtract_years_with_date─┬──subtract_years_with_date_time─┐
│2018 年 1 月 1 日 │2018 年 1 月 1 日 00:00:00│
────────────────────┴────────────────── ─────────┘

## dateDiff（'unit'，t1，t2，\[timezone\]）[¶](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/#datediff-unit-t1-t2-91-timezone-93 "永久链接")

返回以“单位”表示的两次之间的差，例如`'hours'`。't1'和't2'可以是 Date 或 DateTime，如果指定了'timezone'，则将其应用于两个参数。如果不是，则使用数据类型“ t1”和“ t2”的时区。如果该时区不同，则结果不确定。

支持的单位值：

单元

第二

分钟

小时

天

周

月

25 美分硬币

年

## timeSlots（StartTime，Duration，\[，Size\]）[¶](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/#timeslots-starttime-duration-91-size-93 "永久链接")

对于从“ StartTime”开始并持续“ Duration”秒的时间间隔，它将返回一个时间间隔数组，其中包括该时间间隔中的点，四舍五入到以秒为单位的“ Size”。“大小”是一个可选参数：常数 UInt32，默认设置为 1800。例如，`timeSlots(toDateTime('2012-01-01 12:20:00'), 600) = [toDateTime('2012-01-01 12:00:00'), toDateTime('2012-01-01 12:30:00')]`。对于在相应会话中搜索综合浏览量，这是必需的。

## formatDateTime（时间，格式\[，时区\]）[¶](https://clickhouse.yandex/docs/en/query_language/functions/date_time_functions/#formatdatetime-time-format-91-timezone-93 "永久链接")

函数根据给定的格式字符串格式化时间。注意：格式是一个常量表达式，例如，单个结果列不能有多种格式。

支持的格式修饰符：（“示例”列显示了 time 的格式化结果`2018-01-02 22:33:44`）

修饰符

描述

例

％C

年除以 100，并截断为整数（00-99）

20

％d

每月的某天，零填充（01-31）

02

％D

MM / DD / YY 的短日期，相当于％m /％d /％y

18/02/02

％e

该月的一天，空格填充（1-31）

2

％F

短 YYYY-MM-DD 日期，相当于％Y-％m-％d

2018-01-02

％H

小时以 24 小时制（00-23）

22

％一世

12 小时格式的小时（01-12）

10

％j

一年中的哪一天（001-366）

002

％m

以十进制数表示的月份（01-12）

01

％M

分钟（00-59）

33

％n

换行符（'\ n'）

％p

AM 或 PM 指定

下午

％R

24 小时 HH：MM 时间，相当于％H：％M

22:33

％S

秒（00-59）

44

％t

水平制表符（'\ t'）

％T

ISO 8601 时间格式（HH：MM：SS），等效于％H：％M：％S

22:33:44

％u

ISO 8601 工作日为数字，星期一为 1（1-7）

2

％V

ISO 8601 周编号（01-53）

01

％w

工作日为小数，星期日为 0（0-6）

2

％y

年，最后两位数（00-99）

18 岁

％Y

年

2018 年

%%

一个牌子

％


