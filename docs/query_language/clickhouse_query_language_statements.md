
## SELECT 查询语法[¶](https://clickhouse.yandex/docs/en/single/#select-queries-syntax "Permanent link")

`SELECT`  执行数据检索。

\[ WITH expr_list | （子查询）\]
SELECT \[ DISTINCT \] expr_list
\[ FROM \[ db 。\] 表 | （子查询） | table_function \] \[ 最终\]
\[ 样本 sample_coeff \]
\[ ARRAY JOIN ...\]
\[ 全局\] \[ 任意| ALL \] \[ INNER | 左| 右| 充分| CROSS \] \[ 外部\] JOIN （子查询）| 表 使用 columns_list
\[ PREWHERE expr \]
\[ WHERE expr \]
\[ GROUP BY expr_list \] \[ WITH TOTALS \]
\[ HAVING expr \]
\[ ORDER BY expr_list \]
\[ LIMIT \[ n ， \] m \]
\[ UNION ALL ...\]
\[INTO OUTFILE 文件名\]
\[ 格式 格式\]
\[ 限制 \[ offset_value ， \] n BY 列\]

所有子句都是可选的，除了 SELECT 之后紧随其后的必需表达式列表之外。下面的子句以与查询执行传送器几乎相同的顺序描述。

如果查询中省略了`DISTINCT`，`GROUP BY`和`ORDER BY`条款和`IN`和`JOIN`子查询，该查询将被完全处理的料流，使用 O（1）的 RAM 量。否则，如果没有指定适当的限制的查询可能会消耗大量的 RAM： ，`max_memory_usage`，`max_rows_to_group_by`，`max_rows_to_sort`，`max_rows_in_distinct`，`max_bytes_in_distinct`，`max_rows_in_set`，`max_bytes_in_set`，`max_rows_in_join`，`max_bytes_in_join`，。`max_bytes_before_external_sort` `max_bytes_before_external_group_by`有关更多信息，请参见“设置”部分。可以使用外部排序（将临时表保存到磁盘）和外部聚合。`The system does not have "merge join"`。

#### WITH 子句[¶](https://clickhouse.yandex/docs/en/single/#with-clause "Permanent link")

本节提供对公用表表达式（[CTE](https://en.wikipedia.org/wiki/Hierarchical_and_recursive_queries_in_SQL)）的支持，但有一些限制：1.不支持递归查询 2.在 WITH 节中使用子查询时，其结果应为仅具有一行的标量。3.表达式的结果不适用于子查询 WITH 子句表达式的结果可以在 SELECT 子句中使用。

示例 1：使用常量表达式作为“变量”

与 ``2019-08-01 15:23:00'' 作为 ts_upper_bound
SELECT \*
FROM hits 在
哪里
EventDate = toDate （ts_upper_bound ） 和
EventTime <= ts_upper_bound

示例 2：从 SELECT 子句列列表中逐出 sum（bytes）表达式结果

WITH sum （bytes ） as s
SELECT
formatReadableSize （s ），
表
FROM system 。部分
GROUP BY 表
ORDER BY 小号

示例 3：使用标量子查询的结果

/ _这个例子将返回最巨大的表的 TOP 10 _ /
WITH
（
SELECT 总和（字节）
FROM 系统。份
WHERE 活性
） AS total_disk_usage
SELECT
（总和（字节） / total_disk_usage ） \* 100 AS table_disk_usage ，
表
FROM 系统。零件
GROUP BY 表
ORDER BY table_disk_usage DESC
LIMIT 10

示例 4：在子查询中重新使用表达式作为当前限制子查询中表达式使用的一种解决方法，您可以复制它。

WITH \[ 'hello' \] AS hello
SELECT
hello ， \*
FROM
（
WITH \[ 'hello' \] AS hello
SELECT hello
）

┌─hello───────┬─hello─────┐
│\['hello'\]│\['hello'\]│
└─────────┴

#### FROM 子句[¶](https://clickhouse.yandex/docs/en/single/#select-from "Permanent link")

如果忽略 FROM 子句，将从`system.one`表中读取数据。该`system.one`表仅包含一行（该表实现与其他 DBMS 中的 DUAL 表相同的目的）。

该`FROM`子句指定从中读取数据的源：

- 表
- 子查询
- [表功能](https://clickhouse.yandex/docs/en/single/#table_functions/)

`ARRAY JOIN`并且`JOIN`可能还包括常规（请参见下文）。

代替表，`SELECT`可以在括号中指定子查询。与标准 SQL 相比，不需要在子查询之后指定同义词。为了兼容性，可以`AS name`在子查询之后编写，但是指定的名称未在任何地方使用。

要执行查询，将从适当的表中提取查询中列出的所有列。外部查询不需要的任何列都从子查询中抛出。如果查询未列出任何列（例如`SELECT count() FROM t`），则无论如何都会从表中提取一些列（首选最小的列），以计算行数。

该`FINAL`修改可以在使用`SELECT`从发动机的选择查询[MergeTree](https://clickhouse.yandex/docs/en/single/#../operations/table_engines/mergetree/)家庭。指定时`FINAL`，数据将完全“合并”。请记住，`FINAL`除了查询中指定的列之外，使用线索还可以读取与主键相关的列。此外，查询将在单个线程中执行，并且数据将在查询执行期间合并。这意味着，使用时`FINAL`，查询将被缓慢处理。在大多数情况下，请避免使用`FINAL`。该`FINAL`修饰符可应用于在后台合并中执行数据转换的 MergeTree 家族的所有引擎（GraphiteMergeTree 除外）。

#### 样本子句[¶](https://clickhouse.yandex/docs/en/single/#select-sample-clause "Permanent link")

该`SAMPLE`子句允许进行近似查询处理。

启用数据采样后，不会对所有数据执行查询，而是仅对特定部分数据（采样）执行查询。例如，如果您需要计算所有访问的统计信息，那么只需对所有访问的 1/10 部分执行查询，然后将结果乘以 10 就足够了。

在以下情况下，近似查询处理可能会很有用：

- 如果您有严格的时序要求（例如<100ms），但是您无法证明需要额外的硬件资源来满足这些要求。
- 如果原始数据不准确，则近似值不会明显降低质量。
- 业务需求以近似结果为目标（出于成本效益，或为了向高级用户销售确切结果）。

注意

仅在表创建期间指定了采样表达式的情况下，才可以对[MergeTree](https://clickhouse.yandex/docs/en/single/#../operations/table_engines/mergetree/)系列中的表使用采样（请参见[MergeTree engine](https://clickhouse.yandex/docs/en/single/#table_engine-mergetree-creating-a-table)）。

数据采样的功能如下：

- 数据采样是确定性机制。同一`SELECT .. SAMPLE`查询的结果始终相同。
- 采样对于不同的表始终如一地工作。对于具有单个采样键的表，具有相同系数的采样始终选择可能数据的相同子集。例如，一个用户 ID 样本从不同的表中获取具有所有可能的用户 ID 的相同子集的行。这意味着您可以在[IN](https://clickhouse.yandex/docs/en/single/#select-in-operators)子句的子查询中使用样本。另外，您可以使用[JOIN](https://clickhouse.yandex/docs/en/single/#select-join)子句加入示例。
- 采样允许从磁盘读取较少的数据。请注意，您必须正确指定采样键。有关更多信息，请参见[创建 MergeTree 表](https://clickhouse.yandex/docs/en/single/#table_engine-mergetree-creating-a-table)。

对于`SAMPLE`子句，支持以下语法：

样本子句语法

描述

`SAMPLE k`

这`k`是从 0 到 1 的数字。  
查询是对`k`部分数据执行的。例如，`SAMPLE 0.1`对 10％的数据运行查询。[阅读更多](https://clickhouse.yandex/docs/en/single/#select-sample-k)

`SAMPLE n`

这`n`是一个足够大的整数。  
查询至少`n`在行的样本上执行（但不多于此数）。例如，`SAMPLE 10000000`对至少 10,000,000 行运行查询。[阅读更多](https://clickhouse.yandex/docs/en/single/#select-sample-n)

`SAMPLE k OFFSET m`

这里`k`和`m`是从 0 到 1 的数字  
该查询上的样品中执行`k`所述数据的级分。用于样本的数据被`m`分数抵消。[阅读更多](https://clickhouse.yandex/docs/en/single/#select-sample-offset)

##### 样本[K¶](https://clickhouse.yandex/docs/en/single/#select-sample-k "Permanent link")

这`k`是从 0 到 1 的数字（支持小数和十进制表示法）。例如，`SAMPLE 1/2`或`SAMPLE 0.5`。

在`SAMPLE k`子句中，样本是从`k`一部分数据中获取的。示例如下所示：

SELECT
标题，
计数（） \* 10 AS 浏览量
从 hits_distributed
样品 0 。1
WHERE
CounterID = 34
GROUP BY 标题
ORDER BY PageViews DESC LIMIT 1000

在此示例中，查询是对来自 0.1（10％）数据的样本执行的。聚合函数的值不会自动校正，因此要获得近似结果，请将该值`count()`手动乘以 10。

##### 样本[N¶](https://clickhouse.yandex/docs/en/single/#select-sample-n "Permanent link")

这`n`是一个足够大的整数。例如，`SAMPLE 10000000`。

在这种情况下，查询至少`n`在行的样本上执行（但不多于此）。例如，`SAMPLE 10000000`对至少 10,000,000 行运行查询。

由于数据读取的最小单位是一个颗粒（其大小由设置来`index_granularity`设置），因此设置一个比颗粒大得多的样本是有意义的。

使用该`SAMPLE n`子句时，您不知道处理了哪些相对百分比的数据。因此，您不知道聚合函数应乘以的系数。使用`_sample_factor`虚拟列获取近似结果。

该`_sample_factor`列包含动态计算的相对系数。使用指定的采样键[创建](https://clickhouse.yandex/docs/en/single/#table_engine-mergetree-creating-a-table)表时，将自动[创建](https://clickhouse.yandex/docs/en/single/#table_engine-mergetree-creating-a-table)此列。该`_sample_factor`列的用法示例如下所示。

让我们考虑一下表`visits`，该表包含有关站点访问的统计信息。第一个示例显示如何计算页面浏览量：

SELECT sum （PageViews \* \_sample_factor ）
FROM 访问次数
SAMPLE 10000000

下一个示例显示了如何计算访问总数：

SELECT 总和（\_sample_factor ）
FROM 访问
样本 千万

下面的示例显示了如何计算平均会话持续时间。请注意，您不需要使用相对系数来计算平均值。

SELECT avg （持续时间）
从 访问
样本 10000000

##### 样本 K 偏移[M¶](https://clickhouse.yandex/docs/en/single/#select-sample-offset "Permanent link")

这里`k`和`m`从 0 到 1 的实例是编号如下所示。

**例子 1**

样品 1 / 10

在此示例中，样本是所有数据的 1/10：

`[++------------------]`

**例子 2**

样品 1 / 10 OFFSET 1 / 2

在此，从数据的后半部分抽取 10％的样本。

`[----------++--------]`

#### 数组联接子句[¶](https://clickhouse.yandex/docs/en/single/#select-array-join-clause "Permanent link")

允许执行`JOIN`数组或嵌套数据结构。目的类似于[arrayJoin](https://clickhouse.yandex/docs/en/single/#functions_arrayjoin)函数，但其功能更广泛。

SELECT < expr_list \>
FROM < left_subquery \>
\[ LEFT \] ARRAY JOIN < 数组\>
\[ WHERE | PREWHERE < expr \> \]
...

您只能`ARRAY JOIN`在查询中指定一个子句。

运行时优化查询执行顺序`ARRAY JOIN`。尽管`ARRAY JOIN`必须始终在该`WHERE/PREWHERE`子句之前指定它，但是可以在该子句之前`WHERE/PREWHERE`（如果需要此子句中的结果）或在完成它之后（以减少计算量）执行该操作。处理顺序由查询优化器控制。

支持的类型`ARRAY JOIN`如下：

- `ARRAY JOIN`-在这种情况下，的结果中不包含空数组`JOIN`。
- `LEFT ARRAY JOIN`-的结果`JOIN`包含具有空数组的行。空数组的值设置为数组元素类型的默认值（通常为 0，空字符串或 NULL）。

下面的示例演示`ARRAY JOIN`and `LEFT ARRAY JOIN`子句的用法。让我们创建一个具有[Array](https://clickhouse.yandex/docs/en/single/#../data_types/array/) type 列的表并将值插入其中：

创建 表 arrays_test
（
s String ，
arr Array （UInt8 ）
） ENGINE = 内存;

INSERT INTO arrays_test
VALUES （'你好' ， \[ 1 ，2 \]）， （'世界' ， \[ 3 ，4 ，5 \]）， （'再见' ， \[\]）;

ar─s───────────┬┬arr─────┐
│ 您好 │\[1,2\]│
│ 世界 │\[3,4,5\]│
│ 再见 │\[\]│
───────────┴

下面的示例使用以下`ARRAY JOIN`子句：

SELECT s ， arr
来自 arrays_test
ARRAY JOIN arr ;

─s───────┬─arr─┐
│ 你好 │1│
│ 你好 │2│
│ 世界 │3│
│ 世界 │4│
│ 世界 │5│
└───────┴─────┘

下一个示例使用以下`LEFT ARRAY JOIN`子句：

SELECT s ， arr
从 arrays_test
左 数组 JOIN arr ;

┌─s───────────┬─arr─┐
│ 你好 │1│
│ 你好 │2│
│ 世界 │3│
│ 世界 │4│
│ 世界 │5│
│ 再见 │0│
└───────────┴

##### 使用别名[¶](https://clickhouse.yandex/docs/en/single/#using-aliases "Permanent link")

可以在`ARRAY JOIN`子句中为数组指定别名。在这种情况下，可以使用该别名访问数组项，但是可以使用原始名称访问数组本身。例：

SELECT s ， arr ， 一个
FROM arrays_test
ARRAY JOIN arr AS a ;

┌─s───────┬─arr─────┬─a─┐
│ 您好 │\[1,2\]│1│
│ 您好 │\[1,2\]│2│
│ 世界 │\[3,4,5\]│3│
│ 世界 │\[3,4,5\]│4│
│ 世界 │\[3,4,5\]│5│
───────┴───────┴

使用别名，您可以`ARRAY JOIN`对外部数组执行操作。例如：

SELECT 小号， arr_external
FROM arrays_test
ARRAY JOIN \[ 1 ， 2 ， 3 \] AS arr_external ;

┌─s───────────┬─arr_external─┐
│ 你好 │1│
│ 你好 │2│
│ 你好 │3│
│ 世界 │1│
│ 世界 │2│
│ 世界 │3│
│ 再见 │1│
│ 再见 │2│
│ 再见 │3│
───────────┴

`ARRAY JOIN`子句中可以用逗号分隔多个数组。在这种情况下，`JOIN`将同时执行它们（直接和，而不是笛卡尔积）。请注意，所有数组的大小必须相同。例：

SELECT 小号， ARR ， 一个， NUM ， 映射
FROM arrays_test
ARRAY JOIN 的常用 3 AS 一个， arrayEnumerate （ARR ） AS NUM ， arrayMap （X \- \> X \+ 1 ， ARR ） AS 映射;

─s───────┬─arr─────┬─a─┬─num─┬─ 映射 ─┐
│ 你好 │\[1,2\]│1│1│2│
│ 你好 │\[1,2\]│2│2│3│
│ 世界 │\[3,4,5\]│3│1│4│
│ 世界 │\[3,4,5\]│4│2│5│
│ 世界 │\[3,4,5\]│5│3│6│
└───────┴────────┴────┴──────────────┘

下面的示例使用[arrayEnumerate](https://clickhouse.yandex/docs/en/single/#array_functions-arrayenumerate)函数：

SELECT s ， arr ， a ， num ， arrayEnumerate （arr ）
来自 arrays_test
ARRAY JOIN arr AS a ， arrayEnumerate （arr ） AS num ;

┌─s───────┬─arr─────┬─a─┬─num─┬─arrayEnumerate（arr）──
│ 您好 │\[1,2\]│1│1│\[1,2\]│
│ 你好 │\[1,2\]│2│2│\[1,2\]│
│ 世界 │\[3,4,5\]│3│1│\[1,2,3\]│
│ 世界 │\[3,4,5\]│4│2│\[1,2,3\]│
│ 世界 │\[3,4,5\]│5│3│\[1,2,3\]│
└───────┴────────┴────┴────┴──────────────── ┘

##### 具有嵌套数据结构的数组联接[¶](https://clickhouse.yandex/docs/en/single/#array-join-with-nested-data-structure "Permanent link")

`ARRAY`JOIN''也适用于[嵌套数据结构](https://clickhouse.yandex/docs/en/single/#../data_types/nested_data_structures/nested/)。例：

CREATE TABLE nested_test
（
小号 字符串，
巢 嵌套（
X UINT8 ，
ÿ UInt32 的）
） ENGINE = 内存;

INSERT INTO nested_test
VALUES （'你好' ， \[ 1 ，2 \]， \[ 10 ，20 \]）， （'世界' ， \[ 3 ，4 ，5 \]， \[ 30 ，40 ，50 \]）， （'再见' ， \[ \]， \[\]）;

n─s─────────┬─nest.x──┬─nest.y───────┐
│ 您好 │\[1,2,2││\[10,20\]│
│ 世界 │\[3,4,5\]│\[30,40,50\]│
│ 再见 │\[\]│\[\]│
─────────┴───────┴┴──────────┘

SELECT 小号， ' 鸟巢。X `，` 鸟巢。ÿ `
FROM nested_test
ARRAY JOIN 窝;

┌─s─────┬─nest.x─┬─nest.y─┐
│ 你好 │1│10│
│ 你好 │2│20│
│ 世界 │3│30│
│ 世界 │4│40│
│ 世界 │5│50│
└───────┴──────┴

在中指定嵌套数据结构的名称时`ARRAY JOIN`，其含义`ARRAY JOIN`与其组成的所有数组元素的含义相同。示例如下：

SELECT 小号， ' 鸟巢。X `，` 鸟巢。Ÿ `FROM nested_test ARRAY JOIN` 窝。X `，` 鸟巢。ÿ ` ;

┌─s─────┬─nest.x─┬─nest.y─┐
│ 你好 │1│10│
│ 你好 │2│20│
│ 世界 │3│30│
│ 世界 │4│40│
│ 世界 │5│50│
└───────┴──────┴

这种变化也很有意义：

SELECT 小号， ' 鸟巢。X `，` 鸟巢。Ÿ `FROM nested_test ARRAY JOIN` 窝。X ` ;

┌─s───────┬─nest.x─┬─nest.y───────┐
│ 你好 │1│\[10,20\]│
│ 你好 │2│\[10,20\]│
│ 世界 │3│\[30,40,50\]│
│ 世界 │4│\[30,40,50\]│
│ 世界 │5│\[30,40,50\]│
───────┴──────┴

别名可用于嵌套数据结构，以便选择`JOIN`结果或源数组。例：

SELECT 小号， `ñ 。X` ， `Ñ 。Ÿ` ， `鸟巢。X` ， `鸟巢。ÿ`
FROM nested_test
ARRAY JOIN 巢 AS Ñ ;

─s───────┬─nx─┬─ny─┬─nest.x──┬─nest.y───────┐
│ 你好 │1│10│\[1,2\]│\[10,20\]│
│ 你好 │2│20│\[1,2\]│\[10,20\]│
│ 世界 │3│30│\[3,4,5\]│\[30,40,50\]│
│ 世界 │4│40│\[3,4,5\]│\[30,40,50\]│
│ 世界 │5│50│\[3,4,5\]│\[30,40,50\]│
└───────┴──────┴─────┴────────┴────────┘

使用[arrayEnumerate](https://clickhouse.yandex/docs/en/single/#array_functions-arrayenumerate)函数的示例：

SELECT 小号， `ñ 。X` ， `Ñ 。Ÿ` ， `鸟巢。X` ， `鸟巢。ÿ` ， NUM
FROM nested_test
ARRAY JOIN 巢 AS Ñ ， arrayEnumerate （`巢。X` ） AS NUM ;

─s───────┬─nx─┬─ny─┬─nest.x──┬─nest.y─────────num─┐
│ 你好 │1│10│\[1,2\]│\[10,20\]│1│
│ 你好 │2│20│\[1,2\]│\[10,20\]│2│
│ 世界 │3│30│\[3,4,5\]│\[30,40,50\]│1│
│ 世界 │4│40│\[3,4,5\]│\[30,40,50\]│2│
│ 世界 │5│50│\[3,4,5\]│\[30,40,50\]│3│
───────┴──────┴─────┴────────┴────────┴

#### JOIN 子句[¶](https://clickhouse.yandex/docs/en/single/#select-join "Permanent link")

以正常的[SQL JOIN 方式](<https://en.wikipedia.org/wiki/Join_(SQL)>)联接数据。

注意

与[ARRAY JOIN](https://clickhouse.yandex/docs/en/single/#select-array-join-clause)无关。

选择 < expr_list \>
从 < left_subquery \>
\[ GLOBAL \] \[ ANY | ALL \] \[ INNER | 左| 右| 满| CROSS \] \[ OUTER \] JOIN < 右子查询\>
（ON < expr_list \> ）| （使用 < column_list \> ） ...

可以代替`<left_subquery>`和指定表名`<right_subquery>`。这与`SELECT * FROM table`子查询等效，除了在特殊情况下，该表具有[Join](https://clickhouse.yandex/docs/en/single/#../operations/table_engines/join/)引擎（为连接准备的数组）时。

##### 所支持的类型`JOIN`[¶](https://clickhouse.yandex/docs/en/single/#select-join-types "Permanent link")

- `INNER JOIN`（或`JOIN`）
- `LEFT JOIN`（或`LEFT OUTER JOIN`）
- `RIGHT JOIN`（或`RIGHT OUTER JOIN`）
- `FULL JOIN`（或`FULL OUTER JOIN`）
- `CROSS JOIN`（或`,`）

请参阅标准[SQL JOIN](<https://en.wikipedia.org/wiki/Join_(SQL)>)描述。

##### 多个[JOIN¶](https://clickhouse.yandex/docs/en/single/#multiple-join "Permanent link")

执行查询，ClickHouse 将多表联接重写为两表联接的顺序。例如，如果有四个要联接的表，则 ClickHouse 会联接第一个和第二个表，然后将结果与第三个表联接，最后一步，它将联接第四个表。

如果查询包含该`WHERE`子句，则 ClickHouse 会尝试通过中间联接从该子句下推筛选器。如果无法将过滤器应用于每个中间联接，则在完成所有联接之后，ClickHouse 会应用过滤器。

我们建议使用`JOIN ON`or `JOIN USING`语法来创建查询。例如：

选择 \* FROM t1 加入 t2 ON t1 。a = t2 。一个 JOIN T3 ON T1 。a = t3 。一种

您可以在`FROM`子句中使用逗号分隔的表列表。这仅适用于[allow_experimental_cross_to_join_conversion = 1](https://clickhouse.yandex/docs/en/single/#settings-allow_experimental_cross_to_join_conversion)设置。例如：

SELECT \* FROM T1 ， T2 ， T3 WHERE T1 。a = t2 。a AND t1 。a = t3 。一种

不要混合使用这些语法。

ClickHouse 不直接支持带逗号的语法，因此我们不建议使用它们。该算法尝试根据`CROSS JOIN`and `INNER JOIN`子句重写查询，然后继续进行查询处理。重写查询时，ClickHouse 会尝试优化性能和内存消耗。默认情况下，ClickHouse 将逗号视为`INNER JOIN`子句，并在算法无法保证返回所需数据时将其转换`INNER JOIN`为子句。` CROSS JOIN``INNER JOIN `

##### 严格性[¶](https://clickhouse.yandex/docs/en/single/#select-join-strictness "Permanent link")

- `ALL`—如果右表具有多个匹配的行，则 ClickHouse  从匹配的行创建[笛卡尔乘积](https://en.wikipedia.org/wiki/Cartesian_product)。这是`JOIN`SQL 中的标准行为。
- `ANY`—如果右表具有多个匹配的行，则仅连接找到的第一个行。如果右表只有一个匹配行，则使用`ANY`和`ALL`关键字的查询结果是相同的。
- `ASOF`—用于连接不完全匹配的序列。`ASOF JOIN`使用方法如下所述。

**ASOF JOIN 用法**

`ASOF JOIN`  当您需要连接不完全匹配的记录时非常有用。

的表`ASOF JOIN`必须具有有序序列列。这列不能单独在一个表中，而应该是数据类型之一：`UInt32`，`UInt64`，`Float32`，`Float64`，`Date`，和`DateTime`。

您可以使用以下类型的语法：

- `ASOF JOIN ... ON`

  `sql SELECT expressions_list FROM table_1 ASOF LEFT JOIN table_2 ON equi_cond AND closest_match_cond`

  您可以使用任意多个相等条件，也可以仅使用一个最接近的匹配条件。例如，`SELECT count() FROM A ASOF LEFT JOIN B ON A.a == B.b AND B.t <= A.t`。仅`table_2.some_col <= table_1.some_col`和`table_1.some_col >= table2.some_col`条件类型可用。您不能应用其他条件，例如`>`或`!=`。

- `ASOF JOIN ... USING`

  `sql SELECT expressions_list FROM table_1 ASOF JOIN table_2 USING (equi_column1, ... equi_columnN, asof_column)`

  `ASOF JOIN`采用`equi_columnX`加盟平等和`asof_column`用于连接上与最接近的匹配`table_1.asof_column >= table2.asof_column`条件。该`asof_column`列必须是`USING`子句中的最后一个。

例如，考虑以下表格：

     table\_1 table\_2

事件 ev_time | user_id 事件| ev_time | 用户身份
\-\-\-\-\-\-\-\-\-\- | \-\-\-\-\-\-\-\-\- | \-\-\-\-\-\-\-\-\-\- \-\-\-\-\-\-\-\-\-\- | \-\-\-\-\-\-\-\- \-| \-\-\-\-\-\-\-\-\-\-  
 ……
event_1_1 | 12:00 | 42 event_2_1 | 11:59 | 42
... event_2_2 | 12:30 | 42
event_1_2 | 13:00 | 42 event_2_3 | 13:00 | 42
……

`ASOF JOIN`可以从中获取用户事件的时间戳，`table_1`并从中找到`table_2`最接近（等于或小于）事件时间戳的事件`table_1`。在此，该`user_id`列可用于相等时的联接，而该`ev_time`列可用于最接近的匹配的联接。在我们的示例中，`event_1_1`可以与结合使用`event_2_1`，`event_1_2`可以与结合`event_2_3`，但是`event_2_2`不能结合。

注意

`ASOF`[Join](https://clickhouse.yandex/docs/en/single/#../operations/table_engines/join/)表引擎**不**支持 join 。

要设置默认的严格性值，请使用会话配置参数[join_default_strictness](https://clickhouse.yandex/docs/en/single/#settings-join_default_strictness)。

##### 全局联接[¶](https://clickhouse.yandex/docs/en/single/#global-join "Permanent link")

使用 normal 时`JOIN`，查询将发送到远程服务器。在每个子查询上运行子查询以创建正确的表，并使用该表执行联接。换句话说，右表是在每个服务器上分别形成的。

使用时`GLOBAL ... JOIN`，请求者服务器首先运行一个子查询以计算正确的表。该临时表被传递到每个远程服务器，并使用传输的临时数据在它们上运行查询。

使用时要小心`GLOBAL`。有关更多信息，请参见“ [分布式子查询](https://clickhouse.yandex/docs/en/single/#select-distributed-subqueries) ”部分。

##### 使用建议[¶](https://clickhouse.yandex/docs/en/single/#usage-recommendations "Permanent link")

运行时`JOIN`，相对于查询的其他阶段没有优化的执行顺序。连接（在右表中进行搜索）在筛选`WHERE`之前和聚合之前运行。为了显式设置处理顺序，我们建议运行带有`JOIN`子查询的子查询。

例：

SELECT
CounterID ，
命中，
访问
FROM
（
SELECT
CounterID ，
计数（） AS 命中
FROM 测试。打
GROUP BY CounterID
） 任何 LEFT JOIN
（
SELECT
CounterID ，
总和（注册） AS 访问
FROM 测试。访问
GROUP BY CounterID
） 使用 CounterID
ORDER BY 命中 数据中心
极限 10

┌─CounterID─┬─── 点击数 ─┬─ 访问 ─┐
│1143050│523264│13665│
│731962│475698│102716│
│722545│337212│108187│
│722889│252197│10547│
│2237260│196036│9522│
│23057320│147211│7689│
│722818│90109│17847│
│48221│85379│4652│
│19762435│77807│7026│
│722884│77492│11056│
─────────┴

子查询不允许你设置的名称或将其用于从特定的子查询引用的列。在中指定的列`USING`在两个子查询中必须具有相同的名称，而其他列的名称必须不同。您可以使用别名来更改子查询列的名称（示例中使用别名`hits`和`visits`）。

该`USING`子句指定一个或多个要连接的列，以建立这些列的相等性。设置的列列表不带括号。不支持更复杂的连接条件。

右表（子查询结果）位于 RAM 中。如果没有足够的内存，则无法运行`JOIN`。

每次使用相同`JOIN`的查询运行时，由于未缓存结果，因此再次运行子查询。为避免这种情况，请使用特殊的[Join](https://clickhouse.yandex/docs/en/single/#../operations/table_engines/join/)表引擎，该引擎是一个准备好进行连接的数组，始终位于 RAM 中。

在某些情况下，使用`IN`代替会更有效`JOIN`。在各种类型的`JOIN`，最有效的是`ANY LEFT JOIN`，那么`ANY INNER JOIN`。效率最低的是`ALL LEFT JOIN`和`ALL INNER JOIN`。

如果您需要一个`JOIN`用于连接维度表的表（这些表是包含维度属性的相对较小的表，例如广告活动的名称），则`JOIN`由于每个查询都会重新访问正确的表，因此 a  可能不太方便。在这种情况下，您应该使用“外部词典”功能代替`JOIN`。有关更多信息，请参阅“ [外部词典](https://clickhouse.yandex/docs/en/single/#dicts/external_dicts/) ”部分。

**内存限制**

ClickHouse 使用[哈希](https://en.wikipedia.org/wiki/Hash_join)联接算法。ClickHouse 接受`<right_subquery>`并在 RAM 中为其创建哈希表。如果需要限制联接操作的内存消耗，请使用以下设置：

- [max_rows_in_join](https://clickhouse.yandex/docs/en/single/#settings-max_rows_in_join) —限制哈希表中的行数。
- [max_bytes_in_join](https://clickhouse.yandex/docs/en/single/#settings-max_bytes_in_join) —限制哈希表的大小。

当达到这些限制中的任何一个时，ClickHouse 就会按照[join_overflow_mode](https://clickhouse.yandex/docs/en/single/#settings-join_overflow_mode)设置的指示进行操作。

##### 处理空单元格或空单元格[¶](https://clickhouse.yandex/docs/en/single/#processing-of-empty-or-null-cells "Permanent link")

在联接表时，可能会出现空单元格。设置[join_use_nulls](https://clickhouse.yandex/docs/en/single/#settings-join_use_nulls)定义[ClickHouse](https://clickhouse.yandex/docs/en/single/#settings-join_use_nulls)如何填充这些单元格。

如果`JOIN`键是[可空](https://clickhouse.yandex/docs/en/single/#../data_types/nullable/)字段，则其中至少一个键的值为[NULL](https://clickhouse.yandex/docs/en/single/#null-literal)的行不合并。

##### 语法限制[¶](https://clickhouse.yandex/docs/en/single/#syntax-limitations "Permanent link")

对于`JOIN`单个`SELECT`查询中的多个子句：

- `*`仅在连接表（而不是子查询）的情况下才能使用通过所有列。
- 该`PREWHERE`子句不可用。

对于`ON`，`WHERE`和`GROUP BY`条款：

- 不能在`ON`，`WHERE`和`GROUP BY`子句中使用任意表达式，但是可以在`SELECT`子句中定义一个表达式，然后通过别名在这些子句中使用它。

#### WHERE 子句[¶](https://clickhouse.yandex/docs/en/single/#where-clause "Permanent link")

如果有 WHERE 子句，则它必须包含 UInt8 类型的表达式。这通常是带有比较和逻辑运算符的表达式。此表达式将在所有其他转换之前用于过滤数据。

如果数据库表引擎支持索引，则会根据使用索引的能力来评估表达式。

#### PREWHERE 子句[¶](https://clickhouse.yandex/docs/en/single/#prewhere-clause "Permanent link")

该子句与 WHERE 子句具有相同的含义。区别在于从表中读取数据的方式不同。使用 PREWHERE 时，首先仅读取执行 PREWHERE 所需的列。然后，读取运行查询所需的其他列，但仅读取 PREWHERE 表达式为 true 的那些块。

如果查询中的少数列使用了过滤条件，但提供了强大的数据过滤功能，则使用 PREWHERE 是有意义的。这减少了要读取的数据量。

例如，对于提取大量列但仅对少数列进行过滤的查询，编写 PREWHERE 很有用。

只有该`*MergeTree`系列的表才支持 PREWHERE 。

查询可以同时指定 PREWHERE 和 WHERE。在这种情况下，PREWHERE 在 WHERE 之前。

如果将“ optimize_move_to_prewhere”设置设置为 1 且省略了 PREWHERE，则系统将使用启发式方法自动将表达式的一部分从 WHERE 移至 PREWHERE。

#### GROUP BY 子句[¶](https://clickhouse.yandex/docs/en/single/#select-group-by-clause "Permanent link")

这是面向列的 DBMS 的最重要部分之一。

如果有 GROUP BY 子句，则它必须包含一个表达式列表。每个表达式在这里将被称为“键”。SELECT，HAVING 和 ORDER BY 子句中的所有表达式都必须根据键或聚合函数来计算。换句话说，从表中选择的每一列都必须在键或聚合函数中使用。

如果查询仅在聚合函数内包含表列，则可以省略 GROUP BY 子句，并假定通过空键集进行聚合。

例：

SELECT
计数（），
中位数（FetchTiming \> 60 ？ 60 ： FetchTiming ），
计数（） \- 和（刷新）
FROM 命中

但是，与标准 SQL 相比，如果表不包含任何行（根本不存在任何行，或者使用 WHERE 进行过滤后都没有任何行），则返回空结果，而不返回结果从包含聚合函数初始值的行之一开始。

与 MySQL（并符合标准 SQL）相反，您无法获得不在键或聚合函数中的某些列的某些值（常量表达式除外）。要解决此问题，您可以使用“ any”聚合函数（获取第一个遇到的值）或“ min / max”。

例：

SELECT
domainWithoutWWW （URL ） AS 域，
count （），
任何（Title ） AS 标题 -获取每个域的第一个出现的页眉。
FROM hits
GROUP BY 网 域

对于遇到的每个不同的键值，GROUP BY 都会计算一组聚合函数值。

数组列不支持 GROUP BY。

常量不能指定为聚合函数的参数。示例：sum（1）。取而代之的是，您可以摆脱常量。范例：`count()`。

##### NULL 处理[¶](https://clickhouse.yandex/docs/en/single/#null-processing "Permanent link")

对于分组，ClickHouse 将[NULL](https://clickhouse.yandex/docs/en/single/#syntax/)解释为一个值，然后将`NULL=NULL`。

这是一个示例，说明这意味着什么。

假设您有此表：

┌─x─┬────y─┐
│1│2│
│2│ᴺᵁᴸᴸ│
│3│2│
│3│3│
│3│ᴺᵁᴸᴸ│
└───┴──────┘

查询`SELECT sum(x), y FROM t_null_big GROUP BY y`结果：

┌─sum（x）─┬────y─┐
│4│2│
│3│3│
│5│ᴺᵁᴸᴸ│
└────────┴──────

您可以看到，`GROUP BY`对于`y = NULL`sum 来说`x`，好像`NULL`是这个值。

如果您将多个键传递给`GROUP BY`，结果将为您提供选择的所有组合，就好像`NULL`是特定值一样。

##### WITH TOTALS 修饰符[¶](https://clickhouse.yandex/docs/en/single/#with-totals-modifier "Permanent link")

如果指定了 WITH TOTALS 修饰符，则将计算另一行。该行将具有包含默认值（零或空行）的键列，以及具有在所有行中计算得出的值（“总”值）的聚合函数列。

此额外的行以 JSON *，TabSeparated *和 Pretty \*格式输出，与其他行分开。在其他格式下，不输出该行。

在 JSON *格式中，此行作为单独的“总计”字段输出。在 TabSeparated *格式中，该行位于主要结果之后，其后是一个空行（在其他数据之后）。在 Pretty \*格式中，该行在主要结果之后输出为单独的表。

`WITH TOTALS`存在 HAVING 时可以以不同的方式运行。行为取决于“ totals_mode”设置。默认情况下，`totals_mode = 'before_having'`。在这种情况下，将对所有行（包括未通过 HAVING 和“ max_rows_to_group_by”传递的行）计算“总计”。

其他选择仅包括在“总计”中经过 HAVING 的行，并且其行为与设置`max_rows_to_group_by`和有所不同`group_by_overflow_mode = 'any'`。

`after_having_exclusive`–不要包含未通过的行`max_rows_to_group_by`。换句话说，“总计”将具有少于或等于`max_rows_to_group_by`省略的行数。

`after_having_inclusive`–在“总计”中包括所有未通过“ max_rows_to_group_by”的行。换句话说，“总计”将具有多于或等于`max_rows_to_group_by`省略的行数。

`after_having_auto`–计算通过 HAVING 传递的行数。如果超过一定数量（默认为 50％），请在“总计”中包含所有未通过“ max_rows_to_group_by”的行。否则，不要包括它们。

`totals_auto_threshold`–默认情况下为 0.5。的系数`after_having_auto`。

如果`max_rows_to_group_by`并`group_by_overflow_mode = 'any'`没有使用，所有的变化`after_having`都是一样的，你可以使用其中的任何（例如，`after_having_auto`）。

您可以在子查询中使用 WITH TOTALS，包括 JOIN 子句中的子查询（在这种情况下，将合计各个总计值）。

##### GROUP BY 在外部存储器[¶](https://clickhouse.yandex/docs/en/single/#select-group-by-in-external-memory "Permanent link")

您可以启用将临时数据转储到磁盘以限制期间的内存使用`GROUP BY`。所述[max_bytes_before_external_group_by](https://clickhouse.yandex/docs/en/single/#settings-max_bytes_before_external_group_by)设置确定用于转储阈 RAM 消耗`GROUP BY`临时数据到文件系统。如果设置为 0（默认值），则将其禁用。

使用时`max_bytes_before_external_group_by`，建议您将其设置为`max_memory_usage`大约两倍。这是必需的，因为要进行聚合有两个阶段：读取日期和形成中间数据（1）和合并中间数据（2）。将数据转储到文件系统只能在阶段 1 中进行。如果没有转储临时数据，则阶段 2 可能需要与阶段 1 相同的内存。

例如，如果将[max_memory_usage](https://clickhouse.yandex/docs/en/single/#settings_max_memory_usage)设置为 10000000000，并且要使用外部聚合，则将其设置`max_bytes_before_external_group_by`为 10000000000，将 max_memory_usage 设置为 20000000000 是有意义的。触发外部聚合时（如果至少有一个临时数据转储），则最大消耗的 RAM 的数量仅比的多`max_bytes_before_external_group_by`。

通过分布式查询处理，可以在远程服务器上执行外部聚合。为了使请求者服务器仅使用少量的 RAM，请将其设置`distributed_aggregation_memory_efficient`为 1。

合并刷新到磁盘的数据时，以及`distributed_aggregation_memory_efficient`启用设置后合并来自远程服务器的结果时，最多消耗`1/256 * the_number_of_threads`RAM 总量。

启用外部聚合后，如果`max_bytes_before_external_group_by`数据少于（即未刷新数据），则查询的运行速度与不使用外部聚合一样快。如果刷新了任何临时数据，则运行时间将延长几倍（大约三倍）。

如果您有一个`ORDER BY`带有`LIMIT`after 的`GROUP BY`，则已用 RAM 的数量取决于中的数据量`LIMIT`，而不是整个表中的数据量。但是，如果`ORDER BY`没有`LIMIT`，请不要忘记启用外部排序（`max_bytes_before_external_sort`）。

#### LIMIT BY 子句[¶](https://clickhouse.yandex/docs/en/single/#limit-by-clause "Permanent link")

带有`LIMIT n BY expressions`子句的查询`n`为的每个不同值选择第一行`expressions`。键`LIMIT BY`可以包含任意数量的[表达式](https://clickhouse.yandex/docs/en/single/#syntax-expressions)。

ClickHouse 支持以下语法：

- `LIMIT [offset_value, ]n BY expressions`
- `LIMIT n OFFSET offset_value BY expressions`

在查询处理期间，ClickHouse 选择按排序键排序的数据。排序键是使用[ORDER BY](https://clickhouse.yandex/docs/en/single/#select-order-by)子句显式设置的，也可以作为表引擎的属性隐式设置。然后，ClickHouse 应用`LIMIT n BY expressions`并`n`为的每个不同组合返回第一行`expressions`。如果`OFFSET`指定，则对于属于的不同组合的每个数据块`expressions`，ClickHouse `offset_value`将从块的开头跳过行数，并返回最大的`n`行数。如果`offset_value`大于数据块中的行数，则 ClickHouse 将从数据块中返回零行。

`LIMIT BY`与无关`LIMIT`。它们都可以在同一查询中使用。

**例子**

样表：

创建 表 limit_by （id Int ， val Int ） ENGINE = 内存;
INSERT INTO limit_by 值（1 ， 10 ）， （1 ， 11 ）， （1 ， 12 ）， （2 ， 20 ）， （2 ， 21 ）;

查询：

SELECT \* FROM limit_by ORDER BY id ， val LIMIT 2 BY id

┌─id─┬─val─┐
│1│10│
│1│11│
│2│20│
│2│21│
└──────────

SELECT \* FROM limit_by ORDER BY ID ， VAL LIMIT 1 ， 2 BY ID

┌─id─┬─val─┐
│1│11│
│1│12│
│2│21│
└──────────

该`SELECT * FROM limit_by ORDER BY id, val LIMIT 2 OFFSET 1 BY id`查询返回相同的结果。

以下查询返回每`domain, device_type`对的前 5 个引荐来源网址，总共最多包含 100 行（`LIMIT n BY + LIMIT`）。

选择
domainWithoutWWW （URL ） AS 域，
domainWithoutWWW （REFERRER_URL ） AS 引荐，
DEVICE_TYPE ，
计数（） CNT
FROM 命中
GROUP BY 域， 引荐， DEVICE_TYPE
ORDER BY CNT DESC
LIMIT 5 BY 域， DEVICE_TYPE
LIMIT 100

#### 有子句[¶](https://clickhouse.yandex/docs/en/single/#having-clause "Permanent link")

允许过滤在 GROUP BY 之后收到的结果，类似于 WHERE 子句。WHERE 和 HAVING 的不同之处在于 WHERE 在聚合（GROUP BY）之前执行，而 HAVING 在聚合之后执行。如果未执行聚合，则不能使用 HAVING。

#### ORDER BY 子句[¶](https://clickhouse.yandex/docs/en/single/#select-order-by "Permanent link")

ORDER BY 子句包含一个表达式列表，可以为每个表达式分配 DESC 或 ASC（排序方向）。如果未指定方向，则采用 ASC。ASC 升序排序，DESC 降序排序。排序方向适用于单个表达式，而不适用于整个列表。例：`ORDER BY Visits DESC, SearchPhrase`

对于按字符串值排序，可以指定排序规则（比较）。示例：`ORDER BY SearchPhrase COLLATE 'tr'`-假设字符串是 UTF-8 编码的，则使用土耳其字母按升序对关键字进行排序，不区分大小写。可以为 ORDER BY 中的每个表达式分别指定或不指定 COLLATE。如果指定了 ASC 或 DESC，则在其后指定 COLLATE。使用 COLLATE 时，排序始终不区分大小写。

我们仅建议使用 COLLATE 进行少量行的最终排序，因为使用 COLLATE 进行排序的效率不如按字节进行常规排序。

排序表达式列表中具有相同值的行以任意顺序输出，该顺序也可以是不确定的（每次都不同）。如果省略 ORDER BY 子句，则行的顺序也未定义，也可能不确定。

`NaN`和`NULL`排序顺序：

- 随着改性剂`NULLS FIRST`-首先`NULL`，然后`NaN`，然后其他的值。
- 使用修饰符`NULLS LAST`-首先是值`NaN`，然后是`NULL`。
- 默认值—与`NULLS LAST`修饰符相同。

例：

对于表

┌─x─┬────y─┐
│1│ᴺᵁᴸᴸ│
│2│2│
│1│ 楠 │
│2│2│
│3│4│
│5│6│
│6│ 楠 │
│7│ᴺᵁᴸᴸ│
│6│7│
│8│9│
└───┴──────┘

运行查询`SELECT * FROM t_null_nan ORDER BY y NULLS FIRST`以获取：

┌─x─┬────y─┐
│1│ᴺᵁᴸᴸ│
│7│ᴺᵁᴸᴸ│
│1│ 楠 │
│6│ 楠 │
│2│2│
│2│2│
│3│4│
│5│6│
│6│7│
│8│9│
└───┴──────┘

对浮点数进行排序时，NaN 与其他值分开。不管排序顺序如何，NaN 都会排在最后。换句话说，对于升序排序，它们的放置就好像它们比所有其他数字都大，而对于降序排序，它们的放置就好像它们比其余数字都小。

如果除 ORDER BY 外还指定了足够小的 LIMIT，则使用较少的 RAM。否则，所花费的内存量与要排序的数据量成正比。对于分布式查询处理，如果省略 GROUP BY，则在远程服务器上部分完成排序，然后在请求者服务器上合并结果。这意味着对于分布式排序，要排序的数据量可能大于单个服务器上的内存量。

如果没有足够的 RAM，则可以在外部存储器中进行排序（在磁盘上创建临时文件）。`max_bytes_before_external_sort`为此使用设置。如果将其设置为 0（默认值），则禁用外部排序。如果启用，则当要排序的数据量达到指定的字节数时，将对收集的数据进行排序并转储到临时文件中。读取所有数据后，将合并所有排序的文件并输出结果。文件被写入配置中的/ var / lib / clickhouse / tmp /目录（默认情况下，但是您可以使用'tmp_path'参数更改此设置）。

运行查询可能比'max_bytes_before_external_sort'使用更多的内存。因此，此设置的值必须明显小于“ max_memory_usage”。例如，如果您的服务器具有 128 GB 的 RAM，并且您需要运行单个查询，请将“ max_memory_usage”设置为 100 GB，将“ max_bytes_before_external_sort”设置为 80 GB。

外部排序的工作效率远不及 RAM 排序。

#### SELECT 子句[¶](https://clickhouse.yandex/docs/en/single/#select-select "Permanent link")

在`SELECT`完成上面列出的所有子句的计算之后，将分析该子句中指定的[表达式](https://clickhouse.yandex/docs/en/single/#syntax-expressions)。更具体地说，如果存在任何聚合函数，则分析聚合函数上方的表达式。聚合函数及其下的所有内容都是在聚合（`GROUP BY`）期间计算得出的。这些表达式的工作就像将它们应用于结果中的单独行一样。

如果要获取结果中的所有列，请使用星号（`*`）符号。例如，`SELECT * FROM ...`。

要通过[re2](<https://en.wikipedia.org/wiki/RE2_(software)>)正则表达式匹配结果中的某些列，可以使用该`COLUMNS`表达式。

栏（'regexp' ）

例如，考虑表：

CREATE TABLE 默认值。col_names （aa Int8 ， ab Int8 ， bc Int8 ） ENGINE = TinyLog

以下查询从`a`名称中包含符号的所有列中选择数据。

SELECT COLUMNS （'A' ） FROM col_names

aa─aa─┬─ab─┐
│1│1│
└────┴────┘

您可以`COLUMNS`在查询中使用多个表达式，也可以对其应用函数。

例如：

从 col_names 中选择 SELECT COLUMNS （'a' ）， COLUMNS （'c' ）， toTypeName （COLUMNS （'c' ））

┌─aa─┬─ab─┬─BC─b─toTypeName（bc）─┐
│1│1│1│Int8│
────┴────┴────┴────────┘

使用函数时要小心，因为`COLUMN`表达式返回的列数是可变的，并且，如果函数不支持此数目的参数，则 ClickHouse 会引发异常。

例如：

SELECT COLUMNS （'A' ） \+ COLUMNS （'C' ） FROM col_names

从服务器（版本 19.14.1）收到异常：
代码：42。DB :: Exception：从 localhost：9000 接收。DB :: Exception：函数 plus 的参数数量不匹配：传递 3，应为 2。

在这个例子中，`COLUMNS('a')`返回两列`aa`，`ab`以及`COLUMNS('c')`返回`bc`列。该`+`运营商不能申请到 3 个参数，所以 ClickHouse 投用关于它的消息的异常。

与`COLUMNS`表达式匹配的列可以具有不同的类型。如果`COLUMNS`不匹配任何列，并且是中的单个表达式`SELECT`，则 ClickHouse 会引发异常。

#### DISTINCT 条款[¶](https://clickhouse.yandex/docs/en/single/#select-distinct "Permanent link")

如果指定了 DISTINCT，则结果中所有完全匹配的行集中只有一行。结果将与在没有聚合功能的情况下在 SELECT 中指定的所有字段中指定 GROUP BY 相同。但是与 GROUP BY 有一些区别：

- DISTINCT 可以与 GROUP BY 一起应用。
- 当省略 ORDER BY 并定义 LIMIT 时，查询将在读取了所需数量的不同行后立即停止运行。
- 数据块在处理时输出，而无需等待整个查询完成运行。

如果 SELECT 至少有一个数组列，则不支持 DISTINCT。

`DISTINCT`与[NULL 一起使用](https://clickhouse.yandex/docs/en/single/#syntax/)，就好像`NULL`是一个特定值，而和一样`NULL=NULL`。换句话说，在`DISTINCT`结果中，不同的组合`NULL`仅发生一次。

ClickHouse 支持在一个查询中将`DISTINCT`and `ORDER BY`子句用于不同的列。该`DISTINCT`子句在该`ORDER BY`子句之前执行。

表格示例：

┌─a─┬─b─┐
│2│1│
│1│2│
│3│3│
│2│4│
└───┴───┘

通过`SELECT DISTINCT a FROM t1 ORDER BY b ASC`查询选择数据时，我们得到以下结果：

┌─a─┐
│2│
│1│
│3│
└───┘

如果更改排序方向`SELECT DISTINCT a FROM t1 ORDER BY b DESC`，则会得到以下结果：

┌─a─┐
│3│
│1│
│2│
└───┘

行在`2, 4`排序之前被剪切。

在对查询进行编程时，请考虑这种实现的特殊性。

#### LIMIT 条款[¶](https://clickhouse.yandex/docs/en/single/#limit-clause "Permanent link")

`LIMIT m`允许您`m`从结果中选择第一行。

`LIMIT n, m`允许您`m`在跳过第一`n`行之后从结果中选择第一行。该`LIMIT m OFFSET n`语法也支持。

`n`并且`m`必须是非负整数。

如果没有`ORDER BY`子句对结果进行显式排序，则结果可能是任意的和不确定的。

#### UNION ALL 子句[¶](https://clickhouse.yandex/docs/en/single/#union-all-clause "Permanent link")

您可以使用 UNION ALL 组合任意数量的查询。例：

SELECT CounterID ， 1 个 AS 表， toInt64 （count （）） AS c
FROM test 。命中
GROUP BY CounterID

UNION ALL

SELECT CounterID ， 2 个 AS 表， 求和（Sign ） AS c
FROM 测试。访问
GROUP BY CounterID
HAVING c \> 0

仅支持 UNION ALL。不支持常规的 UNION（UNION DISTINCT）。如果需要 UNION DISTINCT，则可以从包含 UNION ALL 的子查询中编写 SELECT DISTINCT。

属于 UNION ALL 的查询可以同时运行，并且它们的结果可以混合在一起。

结果的结构（列的数量和类型）必须与查询匹配。但是列名称可以不同。在这种情况下，最终结果的列名称将从第一个查询中获取。对联合执行类型转换。例如，如果合并的两个查询具有相同的字段，`Nullable`且`Nullable`具有兼容类型的非和类型，则结果将`UNION ALL`具有`Nullable`类型字段。

属于 UNION ALL 的查询不能放在方括号中。ORDER BY 和 LIMIT 用于单独的查询，而不是最终结果。如果需要对最终结果进行转换，则可以将带有 UNION ALL 的所有查询放在 FROM 子句的子查询中。

#### INTO OUTFILE 子句[¶](https://clickhouse.yandex/docs/en/single/#into-outfile-clause "Permanent link")

添加`INTO OUTFILE filename`子句（其中 filename 是字符串文字），以将查询输出重定向到指定文件。与 MySQL 相反，该文件是在客户端创建的。如果文件名相同的文件已经存在，查询将失败。此功能在命令行客户端和 clickhouse-local 中可用（通过 HTTP 接口发送的查询将失败）。

默认输出格式为 TabSeparated（与命令行客户端批处理模式相同）。

#### 格式条款[¶](https://clickhouse.yandex/docs/en/single/#format-clause "Permanent link")

指定“ FORMAT 格式”以获取任何指定格式的数据。您可以将其用于方便或创建转储。有关更多信息，请参见“格式”部分。如果省略 FORMAT 子句，则使用默认格式，该格式取决于访问数据库的设置和接口。对于批处理模式下的 HTTP 接口和命令行客户端，默认格式为 TabSeparated。对于交互模式下的命令行客户端，默认格式为 PrettyCompact（具有吸引人的紧凑表）。

使用命令行客户端时，数据以内部有效格式传递到客户端。客户端独立解释查询的 FORMAT 子句并格式化数据本身（从而减轻了网络和服务器的负担）。

#### 运算符[¶](https://clickhouse.yandex/docs/en/single/#select-in-operators "Permanent link")

在`IN`，`NOT IN`，`GLOBAL IN`，和`GLOBAL NOT IN`运营商分别覆盖，因为它们的功能是相当丰富的。

运算符的左侧是单列或元组。

例子：

SELECT 用户 ID IN （123 ， 456 ） FROM ...
SELECT （CounterID ， 用户 ID ） IN （（34 ， 123 ）， （101500 ， 456 ）） FROM ...

如果左侧是索引中的单个列，而右侧是一组常量，则系统将使用索引来处理查询。

不要明确列出太多的值（即数百万）。如果数据集很大，请将其放在临时表中（例如，请参阅“用于查询处理的外部数据”部分），然后使用子查询。

运算符的右侧可以是一组常量表达式，一组具有常量表达式的元组（如上面的示例所示），也可以是括号内的数据库表或 SELECT 子查询的名称。

如果运算符的右侧是表的名称（例如`UserID IN users`），则它等效于 subquery `UserID IN (SELECT * FROM users)`。处理与查询一起发送的外部数据时，请使用此选项。例如，查询可以与加载到“用户”临时表的一组用户 ID 一起发送，应对其进行过滤。

如果运算符的右侧是具有 Set 引擎（始终在 RAM 中的已准备数据集）的表名，则不会为每个查询再次创建该数据集。

子查询可以指定多个列来过滤元组。例：

SELECT （CounterID ， UserID ） IN （SELECT CounterID ， UserID FROM ...） FROM ...

IN 运算符左右两列应具有相同的类型。

IN 运算符和子查询可以出现在查询的任何部分，包括聚合函数和 lambda 函数。例：

SELECT
EVENTDATE ，
平均（用户 ID IN
（
SELECT 用户 ID
FROM 测试。命中
WHERE EVENTDATE = TODATE （'2014 年 3 月 17 日' ）
）） AS 比
FROM 测试。命中
GROUP BY EventDate
ORDER BY EventDate ASC

┌──EventDate─┬──── 比率 ─┐
│2014-03-17│1│
│2014 年 3 月 18 日 │0.807696│
│2014 年 3 月 19 日 │0.755406│
│2014 年 3 月 20 日 │0.723218│
│2014-03-21│0.697021│
│2014 年 3 月 22 日 │0.647851│
│2014-03-23│0.648416│
└──────────┴

对于 3 月 17 日之后的每一天，请计算 3 月 17 日访问该网站的用户的浏览量所占的百分比。IN 子句中的子查询始终仅在一台服务器上运行一次。没有相关的子查询。

##### NULL 处理[¶](https://clickhouse.yandex/docs/en/single/#null-processing_1 "Permanent link")

在请求处理期间，IN 运算符假定使用[NULL](https://clickhouse.yandex/docs/en/single/#syntax/)进行运算的结果始终等于`0`，而不管`NULL`运算符的右侧还是左侧。`NULL`值不包含在任何数据集中，彼此不对应，无法进行比较。

这是`t_null`表格的示例：

┌─x─┬────y─┐
│1│ᴺᵁᴸᴸ│
│2│3│
└───┴──────┘

运行查询`SELECT x FROM t_null WHERE y IN (NULL,3)`将为您提供以下结果：

┌─x─┐
│2│
└───┘

您可以看到`y = NULL`查询结果中排除了该行。这是因为 ClickHouse 无法确定是否`NULL`将其包含在`(NULL,3)`集合中，并`0`作为操作的结果返回，并将`SELECT`此行从最终输出中排除。

从 t_null 选择 y IN （NULL ， 3 ）

in─in（y，元组（NULL，3））─┐
│0│
│1│
└──────────────┘

##### 分布式子查询[¶](https://clickhouse.yandex/docs/en/single/#select-distributed-subqueries "Permanent link")

带有子查询的 IN-有两个选项（类似于 JOIN）：normal `IN`/ `JOIN`和`GLOBAL IN`/ `GLOBAL JOIN`。它们在分布式查询处理中的运行方式不同。

注意

请记住，以下描述的算法可能会根据[设置](https://clickhouse.yandex/docs/en/single/#../operations/settings/settings/) `distributed_product_mode`设置而有所不同。

使用常规 IN 时，查询将发送到远程服务器，并且每个服务器都运行`IN`or `JOIN`子句中的子查询。

使用`GLOBAL IN`/时`GLOBAL JOINs`，首先为`GLOBAL IN`/  运行所有子查询`GLOBAL JOINs`，然后将结果收集在临时表中。然后，将临时表发送到每个远程服务器，并在其中使用此临时数据运行查询。

对于非分布式查询，请使用常规`IN`/ `JOIN`。

在`IN`/ `JOIN`子句中使用子查询进行分布式查询处理时要小心。

让我们看一些例子。假设集群中的每个服务器都有一个正常的**local_table**。每个服务器还具有一个**Distributed_table**表，该表具有**Distributed**类型，用于查看集群中的所有服务器。

对于对**分布式表**的查询，查询将被发送到所有远程服务器，并使用**local_table**在它们上运行。

例如，查询

从 distributed_table 中选择 uniq （UserID ）

将被发送到所有远程服务器

从 local_table 中选择 uniq （UserID ）

并对其并行运行，直到达到可以合并中间结果的阶段。然后，中间结果将被返回到请求者服务器并在其上合并，最终结果将被发送到客户端。

现在，让我们检查一个带有 IN 的查询：

SELECT uniq 的（用户 ID ） FROM distributed_table WHERE CounterID = 101500 和 用户 ID IN （SELECT 用户 ID FROM local_table WHERE CounterID = 34 ）

- 计算两个网站的受众的交集。

该查询将作为以下内容发送到所有远程服务器

从 local_table WHERE CounterID = 101500 中选择 uniq （UserID ） 和 UserID IN （从 local_table WHERE CounterID = 34 中选择用户 ID ）

换句话说，IN 子句中设置的数据将仅在每个服务器本地存储的数据之间独立地在每个服务器上收集。

如果您已经为这种情况做好了准备，并且已将数据分布在群集服务器上，从而单个 UserID 的数据完全驻留在单个服务器上，则此操作将正确，最佳地工作。在这种情况下，所有必要的数据将在每台服务器上本地可用。否则，结果将不准确。我们将查询的这种变化称为“本地 IN”。

若要更正当数据在群集服务器中随机分布时查询的工作方式，可以在子查询中指定**分布式表**。查询如下所示：

SELECT uniq 的（用户 ID ） FROM distributed_table WHERE CounterID = 101500 和 用户 ID IN （SELECT 用户 ID FROM distributed_table WHERE CounterID = 34 ）

该查询将作为以下内容发送到所有远程服务器

从 local*table WHERE 计数器 ID = 101500 和用户 ID IN 中选择 uniq （UserID ） （从分布式*表 WHERE CounterID = 34 选择用户 ID ）

子查询将开始在每个远程服务器上运行。由于子查询使用分布式表，因此每个远程服务器上的子查询将重新发送给每个远程服务器，如下所示：

从 local_table 中选择用户 ID WHERE 计数器 ID = 34

例如，如果您有一个由 100 个服务器组成的集群，那么执行整个查询将需要 10,000 个基本请求，这通常被认为是不可接受的。

在这种情况下，应始终使用 GLOBAL IN 而不是 IN。让我们看看它如何用于查询

SELECT uniq （UserID ） 从 分布式表 WHERE CounterID = 101500 和 用户 ID GLOBAL IN （SELECT UserID FROM 分布式表 WHERE CounterID = 34 ）

请求者服务器将运行子查询

从分布式表中选择用户 ID WHERE 计数器 ID = 34

并将结果放入 RAM 中的临时表中。然后，该请求将作为

SELECT uniq 的（用户 ID ） FROM local_table WHERE CounterID = 101500 和 用户 ID GLOBAL IN \_data1

临时表`_data1`将随查询一起发送到每个远程服务器（临时表的名称是实现定义的）。

这比使用普通 IN 更为理想。但是，请记住以下几点：

1.  创建临时表时，数据不是唯一的。若要减少通过网络传输的数据量，请在子查询中指定 DISTINCT。（对于普通的 IN，您不需要这样做。）
2.  临时表将被发送到所有远程服务器。传输不考虑网络拓扑。例如，如果 10 个远程服务器位于相对于请求者服务器而言非常遥远的数据中心中，则数据将通过该通道发送 10 次到远程数据中心。使用 GLOBAL IN 时，尽量避免使用大数据集。
3.  将数据传输到远程服务器时，对网络带宽的限制是不可配置的。您可能会使网络超载。
4.  尝试在服务器之间分配数据，这样就不必定期使用 GLOBAL IN。
5.  如果您需要经常使用 GLOBAL IN，请计划 ClickHouse 群集的位置，以使单个副本组驻留在一个以上的数据中心中，并且它们之间具有快速的网络，以便可以在单个数据中完全处理查询。中央。

`GLOBAL IN`万一该本地表仅在请求者服务器上可用并且您要在远程服务器上使用来自该表的数据，则在子句中指定本地表也很有意义。

#### 极值[¶](https://clickhouse.yandex/docs/en/single/#extreme-values "Permanent link")

除了结果之外，您还可以获取结果列的最小值和最大值。为此，将**极限**设置设置为 1。将为数字类型，日期和带时间的日期计算最小值和最大值。对于其他列，将输出默认值。

计算出另外两行–分别为最小值和最大值。这些额外的两行以`JSON*`，`TabSeparated*`和`Pretty*` [格式](https://clickhouse.yandex/docs/en/single/#../interfaces/formats/)输出，与其他行分开。它们不会以其他格式输出。

在`JSON*`格式中，极限值在单独的“极限”字段中输出。在`TabSeparated*`格式中，该行位于主要结果之后，如果存在则在“总计”之后。它之前是一个空行（在其他数据之后）。对于`Pretty*`格式，该行将在主要结果之后（`totals`如果存在）之后作为单独的表输出。

将为之前`LIMIT`和之后的行计算极值`LIMIT BY`。但是，使用时`LIMIT offset, size`，之前的行`offset`包含在中`extremes`。在流请求中，结果还可能包含通过的少量行`LIMIT`。

#### 注释[¶](https://clickhouse.yandex/docs/en/single/#notes "Permanent link")

在`GROUP BY`和`ORDER BY`条款不支持位置参数。这与 MySQL 矛盾，但符合标准 SQL。例如，`GROUP BY 1, 2`将被解释为按常量分组（即，将所有行聚合为一个）。

您可以`AS`在查询的任何部分使用同义词（别名）。

您可以在查询的任何部分而不是表达式中添加星号。分析查询后，星号会展开到所有表列的列表（不包括`MATERIALIZED`和`ALIAS`列）。仅在少数情况下使用星号是合理的：

- 创建表转储时。
- 对于仅包含几列的表，例如系统表。
- 用于获取有关表中哪些列的信息。在这种情况下，请设置`LIMIT 1`。但是最好使用`DESC TABLE`查询。
- 当对少量色谱柱进行强过滤时，使用`PREWHERE`。
- 在子查询中（因为外部查询不需要的列从子查询中排除）。

在所有其他情况下，我们不建议使用星号，因为它只会给您带来柱状 DBMS 的缺点，而不是优点。换句话说，不建议使用星号。

### 插入[¶](https://clickhouse.yandex/docs/en/single/#insert "Permanent link")

添加数据。

基本查询格式：

INSERT INTO \[ 分贝。\] 表 \[（C1 ， C2 ， C3 ）\] VALUES （V11 ， V12 ， V13 ）， （V21 ， V22 ， V23 ）， ...

该查询可以指定要插入的列的列表`[(c1, c2, c3)]`。在这种情况下，其余列将填充：

- 根据`DEFAULT`表定义中指定的表达式计算的值。
- 如果`DEFAULT`未定义表达式，则为零和空字符串。

如果[strict_insert_defaults = 1](https://clickhouse.yandex/docs/en/single/#../operations/settings/settings/)，则`DEFAULT`必须在查询中列出未定义的列。

数据可以通过 ClickHouse 支持的任何[格式](https://clickhouse.yandex/docs/en/single/#formats)传递到 INSERT 。必须在查询中明确指定格式：

INSERT INTO \[ 分贝。\] 表 \[（C1 ， C2 ， C3 ）\] FORMAT FORMAT_NAME data_set

例如，以下查询格式与 INSERT ... VALUES 的基本版本相同：

INSERT INTO \[ DB \] 表 \[（C1 ， C2 ， C3 ）\] FORMAT 值 （V11 ， V12 ， V13 ）， （V21 ， V22 ， V23 ）， ...

ClickHouse 会删除所有空格和数据前的换行符（如果有的话）。形成查询时，建议将数据放在查询运算符之后的新行中（如果数据以空格开头，这很重要）。

例：

INSERT INTO 牛逼 FORMAT TabSeparated
11 你好， 世界！
22 Qwerty

您可以使用命令行客户端或 HTTP 接口将数据与查询分开插入。有关更多信息，请参见“ [接口](https://clickhouse.yandex/docs/en/single/#interfaces) ”  部分。

#### 约束[¶](https://clickhouse.yandex/docs/en/single/#constraints_1 "Permanent link")

如果 table 具有[约束条件](https://clickhouse.yandex/docs/en/single/#constraints)，则将针对插入数据的每一行检查其表达式。如果不满足这些约束中的任何一个，服务器将引发一个包含约束名称和表达式的异常，查询将被停止。

#### 插入[¶](https://clickhouse.yandex/docs/en/single/#insert_query_insert-select "Permanent link")的结果`SELECT`

INSERT INTO \[ 分贝。\] 表 \[（C1 ， C2 ， C3 ）\] SELECT ...

列根据其在 SELECT 子句中的位置进行映射。但是，它们在 SELECT 表达式和 INSERT 表中的名称可能不同。如有必要，执行类型转换。

的不同之处值的数据格式没有允许值设定为表述如`now()`，`1 + 2`，等。“值”格式允许有限地使用表达式，但是不建议这样做，因为在这种情况下，将低效的代码用于执行它们。

不支持修改数据部分其他查询：`UPDATE`，`DELETE`，`REPLACE`，`MERGE`，`UPSERT`，`INSERT UPDATE`。但是，您可以使用删除旧数据`ALTER TABLE ... DROP PARTITION`。

`FORMAT`如果`SELECT`子句包含表函数[input（），](https://clickhouse.yandex/docs/en/single/#table_functions/input/)则必须在查询末尾指定子句。

#### 性能注意事项[¶](https://clickhouse.yandex/docs/en/single/#performance-considerations "Permanent link")

`INSERT`按主键对输入数据进行排序，然后按月份将其划分为多个分区。如果您插入混合月份的数据，则可能会大大降低`INSERT`查询的性能。为避免这种情况：

- 批量添加数据，例如一次添加 100,000 行。
- 按月分组数据，然后将其上传到 ClickHouse。

在以下情况下，性能不会降低：

- 数据是实时添加的。
- 您上传的数据通常是按时间排序的。

## 创建查询[¶](https://clickhouse.yandex/docs/en/single/#create-queries "Permanent link")

### CREATE DATABASE [¶](https://clickhouse.yandex/docs/en/single/#query_language-create-database "Permanent link")

创建数据库。

CREATE DATABASE \[ IF NOT EXISTS \] DB_NAME \[ ON CLUSTER 簇\] \[ ENGINE = 发动机（...）\]

#### 条款[¶](https://clickhouse.yandex/docs/en/single/#clauses "Permanent link")

- `IF NOT EXISTS`

  如果`db_name`数据库已经存在，则 ClickHouse 不会创建新数据库，并且：

  - 如果指定了子句，则不会引发异常。
  - 如果未指定子句，则引发异常。

- `ON CLUSTER`

  ClickHouse `db_name`在指定群集的所有服务器上创建数据库。

- `ENGINE`

  - [的 MySQL](https://clickhouse.yandex/docs/en/single/#../database_engines/mysql/)

    允许您从远程 MySQL 服务器检索数据。

  默认情况下，ClickHouse 使用其自己的[数据库引擎](https://clickhouse.yandex/docs/en/single/#../database_engines/)。

### 创建表[¶](https://clickhouse.yandex/docs/en/single/#create-table-query "Permanent link")

该`CREATE TABLE`查询可以有几种形式。

CREATE TABLE \[ IF NOT EXISTS \] \[ 分贝。\] 表名 \[ ON CLUSTER 簇\]
（
NAME1 \[ TYPE1 \] \[ DEFAULT | MATERIALIZED | ALIAS expr1 的\] \[ compression_codec \] \[ TTL 表达式 1 \]，
NAME2 \[ TYPE2 \] \[ DEFAULT | MATERIALIZED | ALIAS 表达式 2 \] \[ compression_codec \] \[ TTL expr2 \]，
...
） 引擎 = 引擎

如果未设置'db'，则在'db'数据库或当前数据库中创建一个名为'name'的表，并在方括号和'engine'引擎中指定结构。该表的结构是列说明的列表。如果引擎支持索引，则将它们指示为表引擎的参数。

列描述是`name type`最简单的情况。范例：`RegionID UInt32`。也可以为默认值定义表达式（请参见下文）。

CREATE TABLE \[ IF NOT EXISTS \] \[ 分贝。\] 表名 AS \[ DB2 。\] NAME2 \[ ENGINE = 发动机\]

创建具有与另一个表相同结构的表。您可以为表指定其他引擎。如果未指定引擎，则将使用与该`db2.name2`表相同的引擎。

CREATE TABLE \[ IF NOT EXISTS \] \[ 分贝。\] 表名 AS table_function （）

用[表函数](https://clickhouse.yandex/docs/en/single/#table_functions/)返回的结构和数据创建[表](https://clickhouse.yandex/docs/en/single/#table_functions/)。

CREATE TABLE \[ IF NOT EXISTS \] \[ DB \] table_name 的 ENGINE = 引擎 AS SELECT ...

`SELECT`使用“引擎”引擎创建具有查询结果之类的结构的表，并用 SELECT 中的数据填充该表。

在所有情况下，如果`IF NOT EXISTS`指定，则如果表已存在，查询将不会返回错误。在这种情况下，查询将不会执行任何操作。

`ENGINE`查询中的子句之后可以有其他子句。请参阅[表引擎](https://clickhouse.yandex/docs/en/single/#table_engines)的描述中有关如何创建表的详细文档。

#### 默认值[¶](https://clickhouse.yandex/docs/en/single/#create-default-values "Permanent link")

列描述可以为一个默认值指定一个表达式，以下列方式之一：`DEFAULT expr`，`MATERIALIZED expr`，`ALIAS expr`。范例：`URLDomain String DEFAULT domain(URL)`。

如果未定义默认值的表达式，则数字的默认值将设置为零，字符串的空字符串，数组的空数组以及`0000-00-00`日期或`0000-00-00 00:00:00`带时间的日期的默认值将设置为零。不支持 NULL。

如果定义了默认表达式，则列类型是可选的。如果没有显式定义的类型，则使用默认的表达式类型。示例：`EventDate DEFAULT toDate(EventTime)`–“日期”类型将用于“事件日期”列。

如果显式定义了数据类型和默认表达式，则将使用类型转换函数将此表达式转换为指定的类型。示例：与的`Hits UInt32 DEFAULT 0`含义相同`Hits UInt32 DEFAULT toUInt32(0)`。

默认表达式可以定义为表常量和列中的任意表达式。创建和更改表结构时，它将检查表达式是否不包含循环。对于 INSERT，它将检查表达式是否可解析-可以计算出它们的所有列均已传递。

`DEFAULT expr`

正常的默认值。如果 INSERT 查询未指定相应的列，则将通过计算相应的表达式来填充它。

`MATERIALIZED expr`

物化表达。不能为 INSERT 指定这样的列，因为它总是被计算的。对于没有列列表的 INSERT，不考虑这些列。此外，在 SELECT 查询中使用星号时，不会替换此列。这是为了保持不变，`SELECT *`可以使用 INSERT  将获得的转储插入到表中，而无需指定列列表。

`ALIAS expr`

代名词。这样的列根本不存储在表中。它的值不能插入表中，并且在 SELECT 查询中使用星号时，不能替换它的值。如果别名在查询解析期间扩展，则可以在 SELECT 中使用。

使用 ALTER 查询添加新列时，不会写入这些列的旧数据。相反，当读取没有新列值的旧数据时，默认情况下会即时计算表达式。但是，如果运行表达式需要查询中未指定的不同列，则将另外读取这些列，但仅针对需要它的数据块读取。

如果将新列添加到表中，但以后又更改其默认表达式，则用于旧数据的值将更改（对于未在磁盘上存储值的数据）。请注意，运行后台合并时，合并部分之一中缺少的列数据将写入合并部分。

无法为嵌套数据结构中的元素设置默认值。

#### 约束[¶](https://clickhouse.yandex/docs/en/single/#constraints "Permanent link")

连同列描述一起可以定义约束：

CREATE TABLE \[ IF NOT EXISTS \] \[ DB \] table_name 的 \[ ON CLUSTER 集群\]
（
名 1 \[ TYPE1 \] \[ DEFAULT | MATERIALIZED | ALIAS 表达式 1 \] \[ compression_codec \] \[ TTL 表达式 1 \]，
...
约束 constraint_name_1 CHECK boolean_expr_1 ，
...
） 引擎 = 引擎

`boolean_expr_1`可以由任何布尔表达式表示。如果为表定义了约束，则将针对`INSERT`查询中的每一行检查每个约束。如果不满足任何约束条件，服务器将引发约束条件名称和检查表达式的异常。

添加大量约束可能会对大`INSERT`查询的性能产生负面影响。

#### TTL 表达式[¶](https://clickhouse.yandex/docs/en/single/#ttl-expression "Permanent link")

定义值的存储时间。只能为 MergeTree 系列表指定。有关详细说明，请参见[TTL 用于列和表](https://clickhouse.yandex/docs/en/single/#table_engine-mergetree-ttl)。

#### 列压缩编解码器[¶](https://clickhouse.yandex/docs/en/single/#column-compression-codecs "Permanent link")

默认情况下，ClickHouse 将在[服务器设置中](https://clickhouse.yandex/docs/en/single/#compression)定义的压缩方法应用于列。您还可以为`CREATE TABLE`查询中的每个单独的列定义压缩方法。

创建 表 codec_example
（
dt 日期 编解码器（ZSTD ），
ts 日期时间 编解码器（LZ4HC ），
浮点值 Float32 编解码器（NONE ），
双值 Float64 编解码器（LZ4HC （9 ））
值 Float32 编解码器（Delta ， ZSTD ）
）
ENGINE = < 引擎\>
。 。

如果指定了编解码器，则默认编解码器不适用。编解码器可以组合在管道中，例如`CODEC(Delta, ZSTD)`。要为您的项目选择最佳编解码器组合，请通过类似于 Altinity [New Encodings 以提高 ClickHouse 效率](https://www.altinity.com/blog/2019/7/new-encodings-to-improve-clickhouse)文章中所述的基准。

警告

您无法使用外部实用工具对 ClickHouse 数据库文件进行解压缩`lz4`。而是使用特殊的[clickhouse-compressor](https://github.com/yandex/ClickHouse/tree/master/dbms/programs/compressor)实用程序。

以下表引擎支持压缩：

- [合并树](https://clickhouse.yandex/docs/en/single/#../operations/table_engines/mergetree/)家族
- [日志](https://clickhouse.yandex/docs/en/single/#../operations/table_engines/log_family/)家族
- [组](https://clickhouse.yandex/docs/en/single/#../operations/table_engines/set/)
- [加入](https://clickhouse.yandex/docs/en/single/#../operations/table_engines/join/)

ClickHouse 支持通用编解码器和专用编解码器。

##### 专用编解码器[¶](https://clickhouse.yandex/docs/en/single/#create-query-specialized-codecs "Permanent link")

这些编解码器旨在通过使用数据的特定功能来使压缩更有效。其中一些编解码器不会自行压缩数据。取而代之的是，他们为通用编解码器准备了数据，与没有这种准备的情况相比，压缩压缩后的数据压缩效果更好。

专用编解码器：

- `Delta(delta_bytes)`—压缩方法，其中原始值被两个相邻值的差代替，但第一个值保持不变。最多`delta_bytes`用于存储增量值，`delta_bytes`原始值的最大大小也是如此。可能的`delta_bytes`值：1、2、4、8。默认值`delta_bytes`是`sizeof(type)`等于 1、2、4 或 8。在所有其他情况下，其值为 1。
- `DoubleDelta`—计算增量的增量并将其以紧凑的二进制形式写入。对于具有恒定步幅的单调序列（例如时间序列数据），可以实现最佳压缩率。可以与任何固定宽度类型一起使用。实现大猩猩 TSDB 中使用的算法，并将其扩展为支持 64 位类型。为 32 字节增量使用 1 个额外的位：5 位前缀而不是 4 位前缀。有关更多信息，请参见在[Gorilla 中](http://www.vldb.org/pvldb/vol8/p1816-teller.pdf)压缩时间戳[：快速，可扩展的内存中时间序列数据库](http://www.vldb.org/pvldb/vol8/p1816-teller.pdf)。
- `Gorilla`—计算当前值与先前值之间的 XOR，并以紧凑的二进制形式写入。存储一系列缓慢变化的浮点值时非常有效，因为当相邻值二进制相等时，可获得最佳压缩率。实现大猩猩 TSDB 中使用的算法，并将其扩展为支持 64 位类型。有关更多信息，请参见在[Gorilla 中](http://www.vldb.org/pvldb/vol8/p1816-teller.pdf)压缩值[：快速，可扩展的内存中时间序列数据库](http://www.vldb.org/pvldb/vol8/p1816-teller.pdf)。
- `T64`-即作物值的未使用的高位整数数据类型（包括压缩方法`Enum`，`Date`和`DateTime`）。在其算法的每个步骤中，编解码器均采用 64 个值的块，将它们放入 64x64 位矩阵，对其进行转置，裁剪未使用的值位，然后将其余值作为序列返回。未使用的位是在使用压缩的整个数据部分中最大值和最小值之间没有区别的位。

` DoubleDelta``Gorilla `Gorilla TSDB 中使用了编解码器和编解码器作为其压缩算法的组成部分。大猩猩方法在时间戳随时间顺序缓慢变化的情况下有效。时间戳由`DoubleDelta`编解码器有效压缩，而值由`Gorilla`编解码器有效压缩。例如，要获取有效存储的表，可以按以下配置创建它：

创建 表 codec_example
（
时间戳记 DateTime CODEC （DoubleDelta ），
slow_values Float32 CODEC （Gorilla ）
）
ENGINE = MergeTree （）

##### 通用编解码器[¶](https://clickhouse.yandex/docs/en/single/#create-query-common-purpose-codecs "Permanent link")

编解码器：

- `NONE` —无压缩。
- `LZ4`—  默认使用无损[数据压缩算法](https://github.com/lz4/lz4)。应用 LZ4 快速压缩。
- `LZ4HC[(level)]`—具有可配置级别的 LZ4 HC（高压缩）算法。默认级别：9.设置`level <= 0`将应用默认级别。可能的级别：\[1，12\]。推荐电平范围：\[4，9\]。
- `ZSTD[(level)]`—  可配置的[ZSTD 压缩算法](https://en.wikipedia.org/wiki/Zstandard)`level`。可能的级别：\[1，22\]。默认值：1。

高压缩级别对于非对称场景非常有用，例如一次压缩，反复解压缩。更高的级别意味着更好的压缩和更高的 CPU 使用率。

### 临时表[¶](https://clickhouse.yandex/docs/en/single/#temporary-tables "Permanent link")

ClickHouse 支持具有以下特征的临时表：

- 会话结束时，包括连接断开，临时表都会消失。
- 临时表仅使用内存引擎。
- 无法为临时表指定数据库。它是在数据库外部创建的。
- 如果一个临时表与另一个表具有相同的名称，并且查询指定表名而不指定 DB，则将使用该临时表。
- 对于分布式查询处理，查询中使用的临时表将传递到远程服务器。

要创建临时表，请使用以下语法：

CREATE TEMPORARY TABLE \[ IF NOT EXISTS \] table_name 的 \[ ON CLUSTER 集群\]
（
名 1 \[ TYPE1 \] \[ DEFAULT | MATERIALIZED | ALIAS 表达式 1 \]，
NAME2 \[ TYPE2 \] \[ DEFAULT | MATERIALIZED | ALIAS 表达式 2 \]，
...
）

在大多数情况下，临时表不是手动创建的，而是在使用外部数据进行查询或分布式时创建的`(GLOBAL) IN`。有关更多信息，请参见适当的部分。

### 分布式 DDL 查询（ON CLUSTER 子句）[¶](https://clickhouse.yandex/docs/en/single/#distributed-ddl-queries-on-cluster-clause "Permanent link")

在`CREATE`，`DROP`，`ALTER`，和`RENAME`查询的支持集群上的分布式执行。例如，下面的查询创建`all_hits` `Distributed`在每个主机上表`cluster`：

CREATE TABLE IF NOT EXISTS all_hits ON CLUSTER 簇 （p 日期， 我 的 Int32 ） ENGINE = 分布式（簇， 默认， 点击）

为了正确运行这些查询，每个主机必须具有相同的群集定义（为简化同步配置，您可以使用 ZooKeeper 中的替代项）。他们还必须连接到 ZooKeeper 服务器。该查询的本地版本最终将在群集中的每个主机上实现，即使某些主机当前不可用。确保在单个主机内执行查询的顺序。 `ALTER`复制表尚不支持查询。

### CREATE VIEW [¶](https://clickhouse.yandex/docs/en/single/#create-view "Permanent link")

CREATE \[ MATERIALIZED \] VIEW \[ IF NOT EXISTS \] \[ DB \] 表名 \[ TO \[ DB \] 名称\] \[ ENGINE = 引擎\] \[ 填充\] AS SELECT ...

创建一个视图。有两种类型的视图：普通视图和 MATERIALIZED 视图。

普通视图不存储任何数据，而只是从另一个表执行读取。换句话说，普通视图仅是一个保存的查询。从视图读取时，此保存的查询在 FROM 子句中用作子查询。

例如，假设您已经创建了一个视图：

CREATE VIEW 视图 AS SELECT ...

并写了一个查询：

从视图中选择 a ， b ， c

此查询与使用子查询完全等效：

从（SELECT ...）选择 a ， b ， c

物化视图存储由相应的 SELECT 查询转换的数据。

创建实例化视图时，必须指定 ENGINE –用于存储数据的表引擎。

物化视图的安排如下：将数据插入 SELECT 中指定的表中时，此 SELECT 查询将转换部分插入的数据，并将结果插入视图中。

如果指定 POPULATE，则在创建现有表数据时会将其插入视图中，就像创建一个一样`CREATE TABLE ... AS SELECT ...`。否则，查询仅包含创建视图后插入表中的数据。我们不建议使用 POPULATE，因为在视图创建过程中插入表中的数据不会被插入其中。

甲`SELECT`查询可以包含`DISTINCT`，`GROUP BY`，`ORDER BY`，`LIMIT`...注意相应的转换是插入的数据的每个块独立地执行。例如，如果`GROUP BY`设置为 if ，则数据将在插入期间聚合，但仅在单个插入数据包中聚合。数据将不会进一步汇总。例外是使用独立执行数据聚合的 ENGINE 时，例如`SummingMergeTree`。

`ALTER`对物化视图的查询执行尚未完全开发，因此可能会带来不便。如果实例化视图使用 construction `TO [db.]name`，则可以`DETACH`运行该视图，然后运行`ALTER`目标表，然后运行`ATTACH`先前的分离（`DETACH`）视图。

视图看起来与普通表相同。例如，它们在`SHOW TABLES`查询结果中列出。

没有用于删除视图的单独查询。要删除视图，请使用`DROP TABLE`。

### 改变[¶](https://clickhouse.yandex/docs/en/single/#query_language_queries_alter "Permanent link")

该`ALTER`查询仅支持`*MergeTree`表，以及`Merge`和`Distributed`。该查询具有多种变体。

#### 列手法[¶](https://clickhouse.yandex/docs/en/single/#column-manipulations "Permanent link")

更改表结构。

ALTER TABLE \[ db \]。名称 \[ ON CLUSTER 群集\] 添加| 删除| CLEAR | 评论| 修改 栏 ...

在查询中，指定一个或多个逗号分隔操作的列表。每个动作都是对列的操作。

支持以下操作：

- [ADD COLUMN](https://clickhouse.yandex/docs/en/single/#alter_add-column) —将新列添加到表中。
- [DROP COLUMN-](https://clickhouse.yandex/docs/en/single/#alter_drop-column)删除列。
- [CLEAR COLUMN-](https://clickhouse.yandex/docs/en/single/#alter_clear-column)重置列值。
- [COMMENT COLUMN](https://clickhouse.yandex/docs/en/single/#alter_comment-column) —在该列中添加文本注释。
- [MODIFY COLUMN-](https://clickhouse.yandex/docs/en/single/#alter_modify-column)更改列的类型和/或默认表达式。

这些动作将在下面详细描述。

##### 添加列[¶](https://clickhouse.yandex/docs/en/single/#alter_add-column "Permanent link")

ADD COLUMN \[ IF NOT EXISTS \] 名称 \[ 类型\] \[ default_expr \] \[ AFTER name_after \]

增加了一个新的列表中与指定的`name`，`type`和`default_expr`（见节[默认表达式](https://clickhouse.yandex/docs/en/single/#create-default-values)）。

如果`IF NOT EXISTS`包含该子句，则该列已存在时查询将不会返回错误。如果指定`AFTER name_after`（另一列的名称），则该列将添加到表列列表中指定的列之后。否则，该列将添加到表的末尾。请注意，无法将列添加到表的开头。对于操作链，`name_after`可以是在先前操作之一中添加的列的名称。

添加列只会更改表结构，而无需对数据执行任何操作。之后的数据不会出现在磁盘上`ALTER`。如果从表中读取数据时缺少列的数据，则将其填充为默认值（如果有，则执行默认表达式，或者使用零或空字符串）。合并数据部分后，该列将出现在磁盘上（请参阅[MergeTree](https://clickhouse.yandex/docs/en/single/#../operations/table_engines/mergetree/)）。

这种方法使我们可以`ALTER`立即完成查询，而无需增加旧数据量。

例：

ALTER TABLE 访问 ADD COLUMN 浏览器 字符串 AFTER user_id

##### 删除列[¶](https://clickhouse.yandex/docs/en/single/#alter_drop-column "Permanent link")

DROP COLUMN \[ IF EXISTS \] 名

删除名称为的列`name`。如果`IF EXISTS`指定了子句，则该列不存在时查询将不会返回错误。

从文件系统中删除数据。由于这会删除整个文件，因此查询几乎立即完成。

例：

ALTER TABLE 访问 DROP COLUMN 浏览器

##### 清除列[¶](https://clickhouse.yandex/docs/en/single/#alter_clear-column "Permanent link")

CLEAR COLUMN \[ IF EXISTS \] 名 IN PARTITION 的 partition_name

重置指定分区的列中的所有数据。在[如何指定分区表达式](https://clickhouse.yandex/docs/en/single/#alter-how-to-specify-part-expr)一节中阅读有关设置分区名称的更多信息。

如果`IF EXISTS`指定了子句，则该列不存在时查询将不会返回错误。

例：

ALTER TABLE 访问 CLEAR COLUMN 浏览器 在 PARTITION 元组（）

##### 评论栏[¶](https://clickhouse.yandex/docs/en/single/#alter_comment-column "Permanent link")

COMMENT COLUMN \[ IF EXISTS \] 名 '注释'

在该列中添加评论。如果`IF EXISTS`指定了子句，则该列不存在时查询将不会返回错误。

每列可以有一个注释。如果该列已存在注释，则新注释将覆盖先前的注释。

注释存储在[DESCRIBE TABLE](https://clickhouse.yandex/docs/en/single/#misc-describe-table)查询`comment_expression`返回的列中。

例：

ALTER TABLE 访问 COMMENT COLUMN 浏览器 “该表显示了用于访问该站点的浏览器。”

##### 修改栏[¶](https://clickhouse.yandex/docs/en/single/#alter_modify-column "Permanent link")

MODIFY COLUMN \[ IF EXISTS \] 名称 \[ 类型\] \[ default_expr \]

该查询将`name`列的类型更改为`type`和/或将默认表达式更改为`default_expr`。如果`IF EXISTS`指定了子句，则该列不存在时查询将不会返回错误。

更改类型时，将转换值，就像将[toType](https://clickhouse.yandex/docs/en/single/#functions/type_conversion_functions/)函数应用于它们一样。如果仅更改默认表达式，则查询不会做任何复杂的事情，并且几乎立即完成。

例：

ALTER TABLE 访问 MODIFY COLUMN 浏览器 Array （String ）

更改列类型是唯一的复杂操作–它将更改带有数据的文件的内容。对于大表，这可能需要很长时间。

有几个处理阶段：

- 准备具有已修改数据的临时（新）文件。
- 重命名旧文件。
- 将临时（新）文件重命名为旧名称。
- 删除旧文件。

只有第一阶段需要时间。如果在此阶段出现故障，则不会更改数据。如果在连续的一个阶段中发生故障，则可以手动恢复数据。例外是如果旧文件已从文件系统中删除，但新文件的数据未写入磁盘并丢失。

`ALTER`复制更改列的查询。指令被保存在 ZooKeeper 中，然后每个副本将它们应用。所有`ALTER`查询均以相同顺序运行。该查询等待其他副本上完成适当的操作。但是，更改复制表中列的查询可能会中断，并且所有操作将异步执行。

##### 变更查询限制[¶](https://clickhouse.yandex/docs/en/single/#alter-query-limitations "Permanent link")

该`ALTER`查询使您可以在嵌套数据结构中创建和删除单独的元素（列），但不能在整个嵌套数据结构中创建和删除它们。要添加嵌套的数据结构，您可以添加名称类似于`name.nested_name`和类型的列`Array(T)`。嵌套数据结构等效于名称在点前具有相同前缀的多个数组列。

不支持删除主键或采样键（`ENGINE`表达式中使用的列）中的列。仅当此更改不会导致数据被修改（例如，允许您将值添加到 Enum 或将类型从更改`DateTime`为`UInt32`）时，才可以更改主键中包含的列的类型。

如果`ALTER`查询不足以进行所需的表更改，则可以创建一个新表，使用[INSERT SELECT](https://clickhouse.yandex/docs/en/single/#insert_query_insert-select)查询将数据复制到该表，然后使用[RENAME](https://clickhouse.yandex/docs/en/single/#misc_operations-rename)查询切换表并删除旧表。您可以使用[clickhouse-copier](https://clickhouse.yandex/docs/en/single/#../operations/utils/clickhouse-copier/)作为`INSERT SELECT`查询的替代方法。

该`ALTER`查询将阻止该表的所有读取和写入。换句话说，如果查询`SELECT`时正在运行一个 long `ALTER`，`ALTER`查询将等待它完成。同时，对同一表的所有新查询将在`ALTER`运行时等待。

对于本身不存储数据的表（例如`Merge`和`Distributed`），`ALTER`只需更改表结构，而不更改从属表的结构。例如，当为`Distributed`表运行 ALTER 时，还需要`ALTER`在所有远程服务器上为表运行。

#### 使用键表达式进行操作[¶](https://clickhouse.yandex/docs/en/single/#manipulations-with-key-expressions "Permanent link")

支持以下命令：

MODIFY ORDER BY new_expression

它仅适用于[`MergeTree`](https://clickhouse.yandex/docs/en/single/#../operations/table_engines/mergetree/)族中的表（包括  [复制的](https://clickhouse.yandex/docs/en/single/#../operations/table_engines/replication/)表）。该命令将表的[排序键](https://clickhouse.yandex/docs/en/single/#../operations/table_engines/mergetree/)更改   为`new_expression`（一个表达式或一个表达式元组）。主键保持不变。

在仅更改元数据的意义上，该命令是轻量级的。要保留排序键表达式对数据部分行进行排序的属性，您不能将包含现有列的表达式添加到排序键（仅`ADD COLUMN`在同一`ALTER`查询中由命令添加的列）。

#### 带有数据跳过索引的操作[¶](https://clickhouse.yandex/docs/en/single/#manipulations-with-data-skipping-indices "Permanent link")

它仅适用于[`*MergeTree`](https://clickhouse.yandex/docs/en/single/#../operations/table_engines/mergetree/)族中的表（包括  [复制的](https://clickhouse.yandex/docs/en/single/#../operations/table_engines/replication/)表）。提供以下操作：

- `ALTER TABLE [db].name ADD INDEX name expression TYPE type GRANULARITY value AFTER name [AFTER name2]` -将索引描述添加到表元数据。

- `ALTER TABLE [db].name DROP INDEX name` -从表元数据中删除索引描述，并从磁盘中删除索引文件。

从某种意义上说，这些命令是轻量级的，它们仅更改元数据或删除文件。同样，它们被复制（通过 ZooKeeper 同步索引元数据）。

#### 有约束手法[¶](https://clickhouse.yandex/docs/en/single/#manipulations-with-constraints "Permanent link")

查看更多关于[约束的信息](https://clickhouse.yandex/docs/en/single/#constraints)

可以使用以下语法添加或删除约束：

ALTER TABLE \[ db \]。命名 ADD 约束 constraint_name 命令 CHECK 表达;
ALTER TABLE \[ db \]。名称 DROP CONSTRAINT 约束名称;

查询将在表中添加或删除有关约束的元数据，因此将立即对其进行处理。

如果添加了约束检查，*将不会*对现有数据*执行*约束检查。

复制表上的所有更改都将广播到 ZooKeeper，因此将应用于其他副本。

#### 使用分区和零件进行操作[¶](https://clickhouse.yandex/docs/en/single/#alter_manipulations-with-partitions "Permanent link")

可以对[分区](https://clickhouse.yandex/docs/en/single/#../operations/table_engines/custom_partitioning_key/)执行以下操作：

- [DETACH PARTITION](https://clickhouse.yandex/docs/en/single/#alter_detach-partition) –将分区移动到`detached`目录并忘记它。
- [DROP PARTITION](https://clickhouse.yandex/docs/en/single/#alter_drop-partition) –删除分区。
- [ATTACH PART | PARTITION](https://clickhouse.yandex/docs/en/single/#alter_attach-partition) –将`detached`目录中的一部分或分区添加到表中。
- [REPLACE PARTITION-](https://clickhouse.yandex/docs/en/single/#alter_replace-partition)将数据分区从一个表复制到另一个表。
- [清除分区中的列](https://clickhouse.yandex/docs/en/single/#alter_clear-column-partition) -重置分区中指定列的值。
- [CLEAR INDEX IN PARTITION-](https://clickhouse.yandex/docs/en/single/#alter_clear-index-partition)重置[分区中](https://clickhouse.yandex/docs/en/single/#alter_clear-index-partition)的指定二级索引。
- [FREEZE PARTITION](https://clickhouse.yandex/docs/en/single/#alter_freeze-partition) –创建分区的备份。
- [FETCH PARTITION](https://clickhouse.yandex/docs/en/single/#alter_fetch-partition) –从另一台服务器下载分区。

##### DETACH PARTITION [¶](https://clickhouse.yandex/docs/en/single/#alter_detach-partition "Permanent link")

ALTER TABLE 表名 DETACH PARTITION partition_expr

将指定分区的所有数据移动到`detached`目录中。服务器会忘记分离的数据分区，就好像它不存在一样。在您进行[ATTACH](https://clickhouse.yandex/docs/en/single/#alter_attach-partition)查询之前，服务器将不知道此数据。

例：

ALTER TABLE 访问 DETACH PARTITION 201901

在[如何指定分区表达式](https://clickhouse.yandex/docs/en/single/#alter-how-to-specify-part-expr)一节中阅读有关设置[分区表达式的信息](https://clickhouse.yandex/docs/en/single/#alter-how-to-specify-part-expr)。

执行查询后，您可以对`detached`目录中的数据执行任何所需的操作-将其从文件系统中删除，或直接保留。

此查询已复制–将数据移动到`detached`所有副本上的目录中。请注意，您只能在领导者副本上执行此查询。要确定副本是否为领导者，请`SELECT`对[system.replicas](https://clickhouse.yandex/docs/en/single/#system_tables-replicas)表执行查询。或者，更容易`DETACH`对所有副本进行查询-除领导者副本之外，所有副本都会引发异常。

##### 删除分区[¶](https://clickhouse.yandex/docs/en/single/#alter_drop-partition "Permanent link")

ALTER TABLE table_name DROP PARTITION partition_expr

从表中删除指定的分区。此查询将分区标记为非活动状态，并大约在 10 分钟内完全删除数据。

在[如何指定分区表达式](https://clickhouse.yandex/docs/en/single/#alter-how-to-specify-part-expr)一节中阅读有关设置[分区表达式的信息](https://clickhouse.yandex/docs/en/single/#alter-how-to-specify-part-expr)。

查询被复制–删除所有副本上的数据。

##### 删除分区| [PART¶](https://clickhouse.yandex/docs/en/single/#alter_drop-detached "Permanent link")

ALTER TABLE 表名 DROP DETACHED PARTITION | PART partition_expr

从中删除指定分区的指定部分或所有部分`detached`。在“ [如何指定](https://clickhouse.yandex/docs/en/single/#alter-how-to-specify-part-expr)分区表达式”一节中阅读有关设置分区表达式的更多信息。

##### 附件分区|部分[¶](https://clickhouse.yandex/docs/en/single/#alter_attach-partition "Permanent link")

ALTER TABLE table_name 附加 分区| PART partition_expr

从`detached`目录将数据添加到表中。可以为整个分区或单独的部分添加数据。例子：

ALTER TABLE 访问 ATTACH PARTITION 201901 ;
ALTER TABLE 访问 ATTACH PART 201901 \_2_2_0 ;

在“ [如何指定](https://clickhouse.yandex/docs/en/single/#alter-how-to-specify-part-expr)分区表达式”一节中阅读有关设置分区表达式的更多信息。

此查询被复制。每个副本检查`detached`目录中是否有数据。如果数据在此目录中，则查询将检查完整性，并验证其是否与启动查询的服务器上的数据匹配。如果一切正确，则查询会将数据添加到副本。如果不是，它将从查询请求者副本或已添加数据的另一个副本下载数据。

因此，您可以将数据`detached`放在一个副本上的目录中，并使用`ALTER ... ATTACH`查询将其添加到所有副本上的表中。

##### REPLACE PARTITION [¶](https://clickhouse.yandex/docs/en/single/#alter_replace-partition "Permanent link")

ALTER TABLE table2 替换 分区 partition_expr from table1

此查询将数据分区从复制`table1`到`table2`。请注意，不会从中删除数据`table1`。

为了使查询成功运行，必须满足以下条件：

- 两个表必须具有相同的结构。
- 两个表必须具有相同的分区键。

##### 分区中的清除列[¶](https://clickhouse.yandex/docs/en/single/#alter_clear-column-partition "Permanent link")

ALTER TABLE table_name CLEAR COLUMN column_name IN PARTITION partition_expr

重置分区中指定列中的所有值。如果`DEFAULT`在创建表时确定了该子句，则此查询会将列值设置为指定的默认值。

例：

ALTER TABLE 访问 CLEAR COLUMN 小时 在 PARTITION 201902

##### 冻结分区[¶](https://clickhouse.yandex/docs/en/single/#alter_freeze-partition "Permanent link")

ALTER TABLE table_name FREEZE \[ PARTITION partition_expr \]

该查询创建指定分区的本地备份。如果`PARTITION`省略该子句，则查询将立即创建所有分区的备份。

注意

在不停止服务器的情况下执行整个备份过程。

需要注意的是旧风格的表，你可以指定分区名称的前缀（例如，“2019”） -那么该查询创建所有相应分区的备份。在[如何指定分区表达式](https://clickhouse.yandex/docs/en/single/#alter-how-to-specify-part-expr)一节中阅读有关设置[分区表达式的信息](https://clickhouse.yandex/docs/en/single/#alter-how-to-specify-part-expr)。

在执行时，对于数据快照，查询将创建到表数据的硬链接。硬链接位于目录中`/var/lib/clickhouse/shadow/N/...`，其中：

- `/var/lib/clickhouse/`  是在配置中指定的有效 ClickHouse 目录。
- `N`  是备份的增量编号。

在备份内部创建与内部相同的目录结构`/var/lib/clickhouse/`。该查询对所有文件执行“ chmod”，禁止写入文件。

创建备份后，可以将数据从复制`/var/lib/clickhouse/shadow/`到远程服务器，然后从本地服务器删除。请注意，该`ALTER t FREEZE PARTITION`查询不会被复制。它仅在本地服务器上创建本地备份。

该查询几乎立即创建了备份（但是首先它会等待对相应表的当前查询完成运行）。

`ALTER TABLE t FREEZE PARTITION`仅复制数据，而不复制表元数据。要备份表元数据，请复制文件`/var/lib/clickhouse/metadata/database/table.sql`

要从备份还原数据，请执行以下操作：

1.  如果表不存在，则创建它。要查看查询，请使用.sql 文件（将`ATTACH`其替换为`CREATE`）。
2.  将数据从`data/database/table/`备份内部的目录复制到该`/var/lib/clickhouse/data/database/table/detached/`目录。
3.  运行`ALTER TABLE t ATTACH PARTITION`查询以将数据添加到表中。

从备份还原不需要停止服务器。

有关备份和还原数据的更多信息，请参见“ [数据备份”](https://clickhouse.yandex/docs/en/single/#../operations/backup/)部分。

##### 分区中的清除索引[¶](https://clickhouse.yandex/docs/en/single/#alter_clear-index-partition "Permanent link")

ALTER TABLE table_name 清除 索引 index_name 在 分区 partition_expr

该查询的工作方式与相似`CLEAR COLUMN`，但它会重置索引而不是列数据。

##### FETCH PARTITION [¶](https://clickhouse.yandex/docs/en/single/#alter_fetch-partition "Permanent link")

ALTER TABLE table_name FETCH PARTITION partition_expr 来自 'path-in-zookeeper'

从另一台服务器下载分区。该查询仅适用于复制的表。

该查询执行以下操作：

1.  从指定的分片下载分区。在“ zookeeper 路径”中，您必须指定 ZooKeeper 中的分片路径。
2.  然后查询将下载的数据放入表的`detached`目录`table_name`。使用[ATTACH PARTITION | PART](https://clickhouse.yandex/docs/en/single/#alter_attach-partition)查询将数据添加到表中。

例如：

ALTER TABLE 用户 FETCH PARTITION 201902 FROM '/ clickhouse /表/ 01-01 /访问' ;
ALTER TABLE 用户的 ATTACH PARTITION 201902 ;

注意：

- 该`ALTER ... FETCH PARTITION`查询不会被复制。它将分区`detached`仅放置在本地服务器上的目录中。
- 该`ALTER TABLE ... ATTACH`查询被复制。它将数据添加到所有副本。数据从`detached`目录添加到其中一个副本，并从相邻副本添加到其他副本。

在下载之前，系统会检查分区是否存在并且表结构是否匹配。从正常副本中自动选择最合适的副本。

尽管查询被调用`ALTER TABLE`，但它不会更改表结构，也不会立即更改表中可用的数据。

##### MOVE PARTITION | PART [¶](https://clickhouse.yandex/docs/en/single/#alter_move-partition "Permanent link")

##### 如何设置分区表达式[¶](https://clickhouse.yandex/docs/en/single/#alter-how-to-specify-part-expr "Permanent link")

您可以通过`ALTER ... PARTITION`不同的方式在查询中指定分区表达式：

- 作为表`partition`列中的值`system.parts`。例如，`ALTER TABLE visits DETACH PARTITION 201901`。
- 作为来自表列的表达式。支持常量和常量表达式。例如，`ALTER TABLE visits DETACH PARTITION toYYYYMM(toDate('2019-01-25'))`。
- 使用分区 ID。分区 ID 是分区的字符串标识符（如果可能的话，是可读的），用作文件系统和 ZooKeeper 中分区的名称。必须在`PARTITION ID`子句中用单引号指定分区 ID 。例如，`ALTER TABLE visits DETACH PARTITION ID '201901'`。
- 在“ [ALTER ATTACH PART”](https://clickhouse.yandex/docs/en/single/#alter_attach-partition)和“ [DROP DETACHED PART”](https://clickhouse.yandex/docs/en/single/#alter_drop-detached)查询中，要指定零件的名称，请使用字符串文字和[system.detached_parts](https://clickhouse.yandex/docs/en/single/#system_tables-detached_parts)表`name`列中的值。例如，。`ALTER TABLE visits ATTACH PART '201901_1_1_0'`

指定分区时引号的用法取决于分区表达式的类型。例如，对于`String`类型，您必须在引号（`'`）中指定其名称。对于`Date`和`Int*`类型，不需要引号。

对于旧式表，可以将分区指定为数字`201901`或字符串`'201901'`。新样式表的语法在类型上更为严格（类似于 VALUES 输入格式的解析器）。

以上所有规则对于[OPTIMIZE](https://clickhouse.yandex/docs/en/single/#misc_operations-optimize)查询也适用。如果在优化未分区表时仅需要指定分区，请设置表达式`PARTITION tuple()`。例如：

优化 表格 table_not_partitioned PARTITION 元组（） FINAL ;

这些`ALTER ... PARTITION`查询的示例在测试[`00502_custom_partitioning_local`](https://github.com/ClickHouse/ClickHouse/blob/master/dbms/tests/queries/0_stateless/00502_custom_partitioning_local.sql)和中进行了演示[`00502_custom_partitioning_replicated_zookeeper`](https://github.com/ClickHouse/ClickHouse/blob/master/dbms/tests/queries/0_stateless/00502_custom_partitioning_replicated_zookeeper.sql)。

#### ALTER 查询的同步[¶](https://clickhouse.yandex/docs/en/single/#synchronicity-of-alter-queries "Permanent link")

对于不可复制的表，所有`ALTER`查询都是同步执行的。对于可复制表，查询仅向中添加有关适当操作的指令`ZooKeeper`，并且这些操作本身将尽快执行。但是，查询可以等待所有副本上完成这些操作。

对于`ALTER ... ATTACH|DETACH|DROP`查询，您可以使用该`replication_alter_partitions_sync`设置来设置等待。可能的值：`0`–不要等待；`1`–仅等待自己执行（默认）；`2`–等待所有。

#### 变异[¶](https://clickhouse.yandex/docs/en/single/#alter-mutations "Permanent link")

变异是一种 ALTER 查询变体，它允许更改或删除表中的行。与用于点数据更改的标准查询`UPDATE`和`DELETE`查询相比，突变用于更改表中许多行的繁重操作。支持`MergeTree`表引擎系列，包括具有复制支持的引擎。

现有表可以按原样进行突变（无需进行转换），但是将第一个突变应用于表后，其元数据格式将与以前的服务器版本不兼容，并且无法恢复到以前的版本。

当前可用的命令：

ALTER TABLE \[ db 。\] 表 DELETE WHERE filter_expr

该`filter_expr`类型必须为`UInt8`。查询将删除该表中该表达式采用非零值的行。

ALTER TABLE \[ db 。\] 表 UPDATE column1 = expr1 \[， ...\]在 哪里 filter_expr

该`filter_expr`类型必须为`UInt8`。该查询将指定列的值更新为行中对应表达式的值，对于这些表达式，其`filter_expr`取非零值。使用`CAST`运算符将值强制转换为列类型。不支持更新用于计算主键或分区键的列。

ALTER TABLE \[ db 。\] 表 MATERIALIZE INDEX 名称 IN PARTITION partition_name

该查询将重建`name`分区中的二级索引`partition_name`。

一个查询可以包含多个用逗号分隔的命令。

对于\* MergeTree 表，通过重写整个数据部分来执行变异。没有原子性-零件准备就绪后会立即替换它们，并且`SELECT`在突变期间开始执行的查询将看到已经被突变的零件中的数据以及尚未被突变的零件中的数据。

突变完全按照其创建顺序排序，并按该顺序应用于各个部分。突变也通过 INSERT 进行部分排序-在提交突变之前插入表中的数据将被突变，而在插入突变之后插入的数据将不被突变。请注意，突变不会以任何方式阻止 INSERT。

添加突变条目后，突变查询立即返回（如果将复制表添加到 ZooKeeper，对于非复制表则添加到文件系统）。变异本身使用系统配置文件设置异步执行。要跟踪突变的进度，可以使用该[`system.mutations`](https://clickhouse.yandex/docs/en/single/#system_tables-mutations)表。即使 ClickHouse 服务器重新启动，成功提交的变异也将继续执行。一旦提交了突变，就无法回滚，但是如果由于某种原因突变被卡住，则可以通过[`KILL MUTATION`](https://clickhouse.yandex/docs/en/single/#kill-mutation)查询将其取消。

完成变异的条目不会立即删除（保留条目的数量由`finished_mutations_to_keep`存储引擎参数确定）。较旧的突变条目将被删除。

## 系统查询[¶](https://clickhouse.yandex/docs/en/single/#query_language-system "Permanent link")

- [重新加载字典](https://clickhouse.yandex/docs/en/single/#query_language-system-reload-dictionaries)
- [重新加载字典](https://clickhouse.yandex/docs/en/single/#query_language-system-reload-dictionary)
- [删除 DNS 缓存](https://clickhouse.yandex/docs/en/single/#query_language-system-drop-dns-cache)
- [下降标记缓存](https://clickhouse.yandex/docs/en/single/#query_language-system-drop-marks-cache)
- [冲洗日志](https://clickhouse.yandex/docs/en/single/#query_language-system-flush_logs)
- [重新加载配置](https://clickhouse.yandex/docs/en/single/#query_language-system-reload-config)
- [关掉](https://clickhouse.yandex/docs/en/single/#query_language-system-shutdown)
- [杀](https://clickhouse.yandex/docs/en/single/#query_language-system-kill)
- [停止分布式发送](https://clickhouse.yandex/docs/en/single/#query_language-system-stop-distributed-sends)
- [冲洗分配](https://clickhouse.yandex/docs/en/single/#query_language-system-flush-distributed)
- [开始分布式发送](https://clickhouse.yandex/docs/en/single/#query_language-system-start-distributed-sends)
- [停止合并](https://clickhouse.yandex/docs/en/single/#query_language-system-stop-merges)
- [开始合并](https://clickhouse.yandex/docs/en/single/#query_language-system-start-merges)

### 重新加载字典[¶](https://clickhouse.yandex/docs/en/single/#query_language-system-reload-dictionaries "Permanent link")

重新加载之前已成功加载的所有词典。默认情况下，字典是延迟加载的（请参阅[dictionaries_lazy_load](https://clickhouse.yandex/docs/en/single/#server_settings-dictionaries_lazy_load)），因此，它们不是在启动时自动加载的，而是通过 dictGet 函数或 ENGINE = Dictionary 从表中的 SELECT 首次访问时进行初始化的。该`SYSTEM RELOAD DICTIONARIES`查询将重新加载，例如字典（加载）。`Ok.`无论字典更新的结果如何，总是返回。

### RELOAD 字典 dictionary_name [¶](https://clickhouse.yandex/docs/en/single/#query_language-system-reload-dictionary "Permanent link")

完全重新加载字典`dictionary_name`，而不管字典的状态如何（LOADED / NOT_LOADED / FAILED）。`Ok.`无论更新字典的结果如何，总是返回。字典的状态可以通过查询`system.dictionaries`表来检查。

SELECT 名称， 状态 FROM 系统。词典;

### 删除 DNS 缓存[¶](https://clickhouse.yandex/docs/en/single/#query_language-system-drop-dns-cache "Permanent link")

重置 ClickHouse 的内部 DNS 缓存。有时（对于旧的 ClickHouse 版本），在更改基础结构（更改其他 ClickHouse 服务器或词典使用的服务器的 IP 地址）时，必须使用此命令。

有关更方便的（自动）缓存管理，请参阅 disable_internal_dns_cache，dns_cache_update_period 参数。

### 删除标记缓存[¶](https://clickhouse.yandex/docs/en/single/#query_language-system-drop-marks-cache "Permanent link")

重置标记缓存。用于 ClickHouse 的开发和性能测试。

### 冲洗日志[¶](https://clickhouse.yandex/docs/en/single/#query_language-system-flush_logs "Permanent link")

将日志消息的缓冲区刷新到系统表（例如 system.query_log）。调试时不等待 7.5 秒。

### RELOAD CONFIG [¶](https://clickhouse.yandex/docs/en/single/#query_language-system-reload-config "Permanent link")

重新加载 ClickHouse 配置。当配置存储在 ZooKeeeper 中时使用。

### 关机[¶](https://clickhouse.yandex/docs/en/single/#query_language-system-shutdown "Permanent link")

通常会关闭 ClickHouse（例如`service clickhouse-server stop`/ `kill {$pid_clickhouse-server}`）

### KILL [¶](https://clickhouse.yandex/docs/en/single/#query_language-system-kill "Permanent link")

中止 ClickHouse 流程（如`kill -9 {$ pid_clickhouse-server}`）

### 管理分布式表[¶](https://clickhouse.yandex/docs/en/single/#query_language-system-distributed "Permanent link")

ClickHouse 可以管理[分布式](https://clickhouse.yandex/docs/en/single/#../operations/table_engines/distributed/)表。当用户将数据插入这些表时，ClickHouse 首先创建应发送到群集节点的数据队列，然后异步发送。您可以使用[STOP DISTRIBUTED SENDS](https://clickhouse.yandex/docs/en/single/#query_language-system-stop-distributed-sends)，[FLUSH DISTRIBUTED](https://clickhouse.yandex/docs/en/single/#query_language-system-flush-distributed)和[START DISTRIBUTED SENDS](https://clickhouse.yandex/docs/en/single/#query_language-system-start-distributed-sends)查询来管理队列处理。您还可以使用该`insert_distributed_sync`设置同步插入分布式数据。

### 停止分布式 SENDS [¶](https://clickhouse.yandex/docs/en/query_language/system/#query_language-system-stop-distributed-sends "永久链接")

将数据插入分布式表时，禁用后台数据分发。

SYSTEM STOP 分布式 SENDS \[ 分贝。\] < distributed_table_name >

### 冲洗分布式[¶](https://clickhouse.yandex/docs/en/query_language/system/#query_language-system-flush-distributed "永久链接")

强制 ClickHouse 将数据同步发送到群集节点。如果任何节点不可用，ClickHouse 会引发异常并停止查询执行。您可以重试查询，直到查询成功为止，这将在所有节点重新联机后发生。

SYSTEM FLUSH DISTRIBUTED \[ db 。\] < distributed_table_name >

### 开始分布式发送[¶](https://clickhouse.yandex/docs/en/query_language/system/#query_language-system-start-distributed-sends "永久链接")

将数据插入分布式表中时启用后台数据分发。

SYSTEM START 分布式 SENDS \[ 分贝。\] < distributed_table_name >

### 停止合并[¶](https://clickhouse.yandex/docs/en/query_language/system/#query_language-system-stop-merges "永久链接")

提供了停止 MergeTree 系列中表的后台合并的可能性：

SYSTEM STOP MERGES \[\[ 分贝。\] merge_tree_family_table_name \]

!!! note 注意：`DETACH / ATTACH`表将启动该表的后台合并，即使之前已停止所有 MergeTree 表的合并。

### 开始合并[¶](https://clickhouse.yandex/docs/en/query_language/system/#query_language-system-start-merges "永久链接")

提供了为 MergeTree 系列中的表启动后台合并的可能性：

系统 启动 合并 \[\[ db 。\] merge_tree_family_table_name \]

## SHOW CREATE TABLE [¶](https://clickhouse.yandex/docs/en/query_language/show/#show-create-table "永久链接")

显示 创建 \[ 临时\] 表 \[ db 。\] 表 \[ INTO OUTFILE 文件名\] \[ 格式 格式\]

返回一个单一`String`类型的“ statement”列，其中包含一个单一值– `CREATE`用于创建指定表的查询。

## SHOW DATABASES [¶](https://clickhouse.yandex/docs/en/query_language/show/#show-databases "永久链接")

SHOW DATABASES \[ INTO OUTFILE 文件名\] \[ FORMAT 格式\]

打印所有数据库的列表。此查询与相同`SELECT name FROM system.databases [INTO OUTFILE filename] [FORMAT format]`。

## 显示过程列表[¶](https://clickhouse.yandex/docs/en/query_language/show/#show-processlist "永久链接")

SHOW PROCESSLIST \[ INTO OUTFILE 文件名\] \[ FORMAT 格式\]

输出[system.processes](https://clickhouse.yandex/docs/en/operations/system_tables/#system_tables-processes)表的内容，该表包含当前正在处理的查询列表（查询除外）`SHOW PROCESSLIST`。

该`SELECT * FROM system.processes`查询返回有关所有当前查询的数据。

提示（在控制台中执行）：

\$ watch -n1 “ clickhouse-client --query ='SHOW PROCESSLIST'”

## 显示表[¶](https://clickhouse.yandex/docs/en/query_language/show/#show-tables "永久链接")

显示表列表。

SHOW \[ TEMPORARY \] TABLES \[ FROM < 分贝\> \] \[ LIKE '<图案>' \] \[ LIMIT < Ñ \> \] \[ INTO OUTFILE < 文件名\> \] \[ FORMAT < 格式\> \]

如果`FROM`未指定该子句，则查询从当前数据库返回表列表。

您可以通过`SHOW TABLES`以下方式获得与查询相同的结果：

从系统中选择名称 。表 WHERE database = < db \> \[ AND 名称 LIKE < 模式\> \] \[ LIMIT < N \> \] \[ INTO OUTFILE < 文件名\> \] \[ FORMAT < 格式\> \]

**例**

以下查询从`system`数据库的表列表中选择前两行，其名称包含`co`。

SHOW TABLES FROM 系统 LIKE '％CO％' LIMIT 2

┌─name───────────────────────────┐
│aggregate_function_combinators│
│ 归类 │
└──────────── ────────────────
