# 句法

系统中有两种类型的解析器：完整的 SQL 解析器（递归下降解析器）和数据格式解析器（快速流解析器）。在除`INSERT`查询之外的所有情况下，仅使用完整的 SQL 解析器。该`INSERT`查询使用两个解析器：

INSERT INTO 吨 VALUES （1 ， '你好，世界' ）， （2 ， 'ABC' ）， （ 3 ， 'DEF' ）

该`INSERT INTO t VALUES`片段通过完整解析器解析，并将数据`(1, 'Hello, world'), (2, 'abc'), (3, 'def')`由快速流解析器解析。您还可以使用[input_format_values_interpret_expressions](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-input_format_values_interpret_expressions)设置打开数据的完整解析器。`input_format_values_interpret_expressions = 1`设为时，ClickHouse 首先尝试使用快速流解析器解析值。如果失败，ClickHouse 将尝试使用完整的解析器处理数据，将其视为 SQL [表达式](https://clickhouse.yandex/docs/en/query_language/syntax/#syntax-expressions)。

数据可以具有任何格式。收到查询时，服务器在 RAM 中计算的请求字节数不超过[max_query_size](https://clickhouse.yandex/docs/en/operations/settings/settings/#settings-max_query_size)个字节（默认情况下为 1 MB），其余部分将进行流解析。这意味着系统不会`INSERT`像 MySQL 那样遇到大查询问题。

`Values`在`INSERT`查询中使用格式时，似乎数据解析与`SELECT`查询中的表达式相同，但是事实并非如此。该`Values`格式是非常有限的。

接下来，我们将介绍完整的解析器。有关格式解析器的更多信息，请参见“ [格式”](https://clickhouse.yandex/docs/en/interfaces/formats/)部分。

## 空格[¶](https://clickhouse.yandex/docs/en/query_language/syntax/#spaces "永久链接")

在语法结构之间（包括查询的开始和结束），可以有任意数量的空格。空格符号包括空格，制表符，换行符，CR 和换页符。

## 注释[¶](https://clickhouse.yandex/docs/en/query_language/syntax/#comments "永久链接")

支持 SQL 风格和 C 风格的注释。SQL 样式的注释：从`--`到行尾。后面的空格`--`可以省略。C 风格的注释：从`/*`到`*/`。这些注释可以是多行。在此也不需要空格。

## 关键字[¶](https://clickhouse.yandex/docs/en/query_language/syntax/#syntax-keywords "永久链接")

关键字（例如`SELECT`）不区分大小写。与标准 SQL 相比，其他所有内容（列名，函数等）都是区分大小写的。

关键字不是保留的（它们只是在相应上下文中解析为关键字）。如果您使用与关键字相同的[标识符](https://clickhouse.yandex/docs/en/query_language/syntax/#syntax-identifiers)，请将其括在引号中。例如，`SELECT "FROM" FROM table_name`如果表`table_name`具有名称为的列，则该查询有效`"FROM"`。

## 标识符[¶](https://clickhouse.yandex/docs/en/query_language/syntax/#syntax-identifiers "永久链接")

标识符是：

- 群集，数据库，表，分区和列的名称。
- 职能。
- 数据类型。
- [表达式别名](https://clickhouse.yandex/docs/en/query_language/syntax/#syntax-expression_aliases)。

标识符可以带引号或不带引号。建议使用不带引号的标识符。

未加引号的标识符必须与正则表达式匹配，`^[a-zA-Z_][0-9a-zA-Z_]*$`并且不能等于[关键字](https://clickhouse.yandex/docs/en/query_language/syntax/#syntax-keywords)。例子：`x, _1, X_y__Z123_.`

如果要使用与关键字相同的标识符，或者要在标识符中使用其他符号，请使用双引号或反引号将其引起来，例如`"id"`，`` `id` ``。

## 文字[¶](https://clickhouse.yandex/docs/en/query_language/syntax/#literals "永久链接")

有：数字，字符串，复合和`NULL`文字。

### 数值[¶](https://clickhouse.yandex/docs/en/query_language/syntax/#numeric "永久链接")

一个数字文字试图被解析：

- 首先使用[strtoull](https://en.cppreference.com/w/cpp/string/byte/strtoul)函数作为 64 位带符号数字。
- 如果不成功，则使用[strtoll](https://en.cppreference.com/w/cpp/string/byte/strtol)函数作为 64 位无符号数字。
- 如果不成功，则使用[strtod](https://en.cppreference.com/w/cpp/string/byte/strtof)函数作为浮点数。
- 否则，将返回错误。

对应的值将具有该值适合的最小类型。例如，将 1 解析为`UInt8`，而将 256 解析为`UInt16`。有关更多信息，请参见[数据类型](https://clickhouse.yandex/docs/en/data_types/)。

例如：`1`，`18446744073709551615`，`0xDEADBEEF`，`01`，`0.1`，`1e100`，`-1e-100`，`inf`，`nan`。

### 字符串[¶](https://clickhouse.yandex/docs/en/query_language/syntax/#syntax-string-literal "永久链接")

仅支持单引号中的字符串文字。随附的字符可以反斜杠转义。下面的转义序列具有对应的特殊值：`\b`，`\f`，`\r`，`\n`，`\t`，`\0`，`\a`，`\v`，`\xHH`。在所有其他情况下，格式`\c`（其中`c`是任何字符）的转义序列都将转换为`c`。这意味着您可以使用序列`\'`和`\\`。该值将具有[String](https://clickhouse.yandex/docs/en/data_types/string/)类型。

您需要在字符串文字中转义的最小字符集：`'`和`\`。单引号可以与单引号，文字`'It\'s'`和`'It''s'`等号转义。

### 复合[¶](https://clickhouse.yandex/docs/en/query_language/syntax/#compound "永久链接")

数组`[1, 2, 3]`和元组支持构造：`(1, 'Hello, world!', 2)`实际上，它们不是文字，而是分别使用数组创建运算符和元组创建运算符的表达式。数组必须至少包含一项，并且一个元组必须至少包含两项。元组在查询`IN`子句中有特殊用途`SELECT`。可以通过查询获得元组，但不能将它们保存到数据库中（“ [内存”](https://clickhouse.yandex/docs/en/operations/table_engines/memory/)表除外）。

### 空[¶](https://clickhouse.yandex/docs/en/query_language/syntax/#null-literal "永久链接")

指示缺少该值。

为了存储`NULL`在表字段中，它必须为[Nullable](https://clickhouse.yandex/docs/en/data_types/nullable/)类型。

根据数据格式（输入或输出），`NULL`可能会有不同的表示形式。有关更多信息，请参见[数据格式](https://clickhouse.yandex/docs/en/interfaces/formats/#formats)文档。

加工有许多细微差别`NULL`。例如，如果比较操作的至少一个参数为`NULL`，则此操作的结果也将为`NULL`。乘法，加法和其他运算也是如此。有关更多信息，请阅读每个操作的文档。

在查询中，您可以`NULL`使用[IS NULL](https://clickhouse.yandex/docs/en/query_language/operators/#operator-is-null)和[IS NOT NULL](https://clickhouse.yandex/docs/en/query_language/operators/)运算符以及相关函数`isNull`和进行检查`isNotNull`。

## 功能[¶](https://clickhouse.yandex/docs/en/query_language/syntax/#functions "永久链接")

函数的编写方式类似于标识符，并在方括号中列出了参数列表（可能为空）。与标准 SQL 相比，括号是必需的，即使对于空的参数列表也是如此。范例：`now()`。有常规函数和汇总函数（请参见“汇总函数”一节）。一些聚合函数可以在方括号中包含两个参数列表。范例：`quantile (0.9) (x)`。这些聚合函数称为“参数”函数，第一个列表中的参数称为“参数”。不带参数的聚合函数的语法与常规函数的语法相同。

## 运算符[¶](https://clickhouse.yandex/docs/en/query_language/syntax/#operators "永久链接")

在查询解析期间，运算符将转换为相应的函数，同时要考虑其优先级和关联性。例如，将表达式`1 + 2 * 3 + 4`转换为`plus(plus(1, multiply(2, 3)), 4)`。

## 数据类型和数据库表引擎[¶](https://clickhouse.yandex/docs/en/query_language/syntax/#data-types-and-database-table-engines "永久链接")

`CREATE`查询中的数据类型和表引擎的编写方式与标识符或函数相同。换句话说，它们可能会或可能不会在方括号中包含参数列表。有关更多信息，请参见“数据类型”，“表引擎”和“创建”部分。

## 表达别名[¶](https://clickhouse.yandex/docs/en/query_language/syntax/#syntax-expression_aliases "永久链接")

别名是查询中表达式的用户定义名称。

expr AS 别名

- `AS`—用于定义别名的关键字。您可以在`SELECT`子句中为表名或列名定义别名，而无需使用`AS`关键字。

  例如，`SELECT table_name_alias.column_name FROM table_name table_name_alias`。

  在[CAST](https://clickhouse.yandex/docs/en/query_language/functions/type_conversion_functions/#type_conversion_function-cast)函数中，`AS`关键字具有另一种含义。请参阅功能说明。

- `expr` — ClickHouse 支持的任何表达式。

  例如，`SELECT column_name * 2 AS double FROM some_table`。

- `alias`—的名称`expr`。别名应符合[标识符](https://clickhouse.yandex/docs/en/query_language/syntax/#syntax-identifiers)语法。

  例如，`SELECT "table t".column_name FROM table_name AS "table t"`。

### 使用注意事项[¶](https://clickhouse.yandex/docs/en/query_language/syntax/#notes-on-usage "永久链接")

别名是查询或子查询的全局别名，您可以在查询的任何部分为任何表达式定义别名。例如，`SELECT (1 AS n) + 2, n`。

别名在子查询中和子查询之间不可见。例如，在执行查询时，`SELECT (SELECT sum(b.a) + num FROM b) - a.a AS num FROM a`ClickHouse 会生成异常`Unknown identifier: num`。

如果在`SELECT`子查询的子句中为结果列定义了别名，则这些列在外部查询中可见。例如，`SELECT n + m FROM (SELECT 1 AS n, 2 AS m)`。

请注意与列名或表名相同的别名。让我们考虑以下示例：

创建 表 t
（
a Int ，
b Int
）
ENGINE = TinyLog （）

SELECT
argMax （a ， b ），
sum （b ） AS b
从 t

从服务器（版本 18.14.17）接收到的异常：
代码：184。DB :: Exception：从 localhost：9000，127.0.0.1 接收。DB :: Exception：聚合函数 sum（b）在查询中的另一个聚合函数中找到。

在这个例子中，我们申报表`t`与列`b`。然后，在选择数据时，我们定义了`sum(b) AS b`别名。作为别名是全球性的，ClickHouse 取代的字面`b`在表达式`argMax(a, b)`与表达式`sum(b)`。这种替换导致了异常。

## 星号[¶](https://clickhouse.yandex/docs/en/query_language/syntax/#asterisk "永久链接")

在`SELECT`查询中，星号可以替换表达式。有关更多信息，请参见“选择”部分。

## 表达式[¶](https://clickhouse.yandex/docs/en/query_language/syntax/#syntax-expressions "永久链接")

表达式是函数，标识符，文字，运算符的应用，括号中的表达式，子查询或星号。它也可以包含一个别名。表达式列表是一个或多个用逗号分隔的表达式。反过来，函数和运算符可以将表达式作为参数。
