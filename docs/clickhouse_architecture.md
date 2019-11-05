# ClickHouse 架构概述

- `ClickHouse` 是真正的`column-oriented`的`DBMS`。
  - 数据按列存储，并且在执行数组（向量或列的大块）期间存储。
  - 只要有可能，就将操作分配给数组，而不是单个值。
  - 这称为`vectorized query execution`，它有助于降低实际数据处理的成本。

> 这个想法并不是ClickHouse发明的。它的历史可以追溯到`APL`编程语言及其后代：`A +`，`J`，`K`，和`Q`。`Array programming`用于科学数据处理。这个想法在关系数据库中也不是什么新事物：例如，它在`Vectorwise`系统中使用。


加快查询处理速度2种方法：
- 运行时代码生成：
  - 动态地为每一类查询生成代码，消除了间接分派和动态分配。
  - 将许多操作融合在一起，从而充分利用 CPU `execution units`和`pipeline`。
- 向量化查询：
  - 它涉及临时向量，必须将这些临时向量写入高速缓存并读回
  - 更容易利用`CPU` 的 `SIMD` 功能
  
ClickHouse 将两种集合： 使用矢量化查询执行，并且对运行时代码生成的初始支持有限。
[研究论文](http://15721.courses.cs.cmu.edu/spring2016/papers/p5-sompolski.pdf)


## 列（Columns）

要表示内存中的列（实际上是列块），需使用  `IColumn`  接口。该接口提供了用于实现各种关系操作符的辅助方法。几乎所有的操作都是不可变的：这些操作不会更改原始列，但是会创建一个新的修改后的列。比如，`IColumn::filter`  方法接受过滤字节掩码，用于  `WHERE`  和  `HAVING`  关系操作符中。另外的例子：`IColumn::permute`  方法支持  `ORDER BY`  实现，`IColumn::cut`  方法支持  `LIMIT`  实现等等。

不同的  `IColumn`  实现（`ColumnUInt8`、`ColumnString`  等）负责不同的列内存布局。内存布局通常是一个连续的数组。对于数据类型为整型的列，只是一个连续的数组，比如  `std::vector`。对于  `String`  列和  `Array`  列，则由两个向量组成：其中一个向量连续存储所有的  `String`  或数组元素，另一个存储每一个  `String`  或  `Array`  的起始元素在第一个向量中的偏移。而  `ColumnConst`  则仅在内存中存储一个值，但是看起来像一个列。

## Field[¶](https://clickhouse.yandex/docs/zh/development/architecture/#field "Permanent link")

尽管如此，有时候也可能需要处理单个值。表示单个值，可以使用  `Field`。`Field`  是  `UInt64`、`Int64`、`Float64`、`String`  和  `Array`  组成的联合。`IColumn`  拥有  `operator[]`  方法来获取第  `n`  个值成为一个  `Field`，同时也拥有  `insert`  方法将一个  `Field`  追加到一个列的末尾。这些方法并不高效，因为它们需要处理表示单一值的临时  `Field`  对象，但是有更高效的方法比如  `insertFrom`  和  `insertRangeFrom`  等。

`Field`  中并没有足够的关于一个表（table）的特定数据类型的信息。比如，`UInt8`、`UInt16`、`UInt32`  和  `UInt64`  在  `Field`  中均表示为  `UInt64`。


## 抽象漏洞(`Leaky Abstractions`)

`IColumn` 具有用于数据的常见关系转换的方法，但这些方法并不能够满足所有需求。比如，`ColumnUInt64`  没有用于计算两列和的方法，`ColumnString`  没有用于进行子串搜索的方法。这些无法计算的例程在 `Icolumn` 之外实现。

列（`Columns`)上的各种函数可以通过使用  `Icolumn`  的方法来提取  `Field`  值，或根据特定的  `Icolumn`  实现的数据内存布局的知识，以一种通用但不高效的方式实现。为此，函数将会转换为特定的  `IColumn`  类型并直接处理内部表示。比如，`ColumnUInt64` 具有 `getData` 方法，该方法返回一个指向列的内部数组的引用，然后一个单独的例程可以直接读写或填充该数组。实际上，`leaky abstractions` 允许我们以更高效的方式来实现各种特定的例程。

## Data Type

`IDataType`  负责序列化和反序列化：读写二进制或文本形式的列或单个值构成的块。`IDataType`  直接与表的数据类型相对应。比如，有  `DataTypeUInt32`、`DataTypeDateTime`、`DataTypeString`  等数据类型。

`IDataType`  与  `IColumn`  之间的关联并不大。不同的数据类型在内存中能够用相同的  `IColumn`  实现来表示。比如，`DataTypeUInt32`  和  `DataTypeDateTime`  都是用  `ColumnUInt32`  或  `ColumnConstUInt32`  来表示的。另外，相同的数据类型也可以用不同的  `IColumn`  实现来表示。比如，`DataTypeUInt8`  既可以使用 `ColumnUInt8` 来表示，也可以使用过  `ColumnConstUInt8`  来表示。

`IDataType`  仅存储元数据。比如，`DataTypeUInt8` 不存储任何东西（除了`vptr`）; `DataTypeFixedString`  仅存储 `N`（固定长度字符串的串长度）。

`IDataType`  具有针对各种数据格式的辅助函数。比如如下一些辅助函数：序列化一个值并加上可能的引号；序列化一个值用于 JSON 格式；序列化一个值作为 XML 格式的一部分。辅助函数与数据格式并没有直接的对应。比如，两种不同的数据格式 `Pretty` 和 `TabSeparated` 均可以使用 `IDataType` 接口提供的 `serializeTextEscaped` 这一辅助函数。


## 块（Block）

`Block`  是表示内存中表的子集（chunk）的容器，是由三元组：`(IColumn, IDataType, 列名)`  构成的集合。在查询执行期间，数据是按  `Block`  进行处理的。如果我们有一个  `Block`，那么就有了数据（在  `IColumn`  对象中），有了数据的类型信息告诉我们如何处理该列，同时也有了列名（来自表的原始列名，或人为指定的用于临时计算结果的名字）。

当我们遍历一个块中的列进行某些函数计算时，会把另一个带结果数据的列加入到块中，但不会更改函数参数中的列，因为操作是不可变的。之后，不需要的列可以从块中删除，但不是修改。这对于消除公共子表达式非常方便。

`Block`  用于处理数据块。注意，对于相同类型的计算，列名和类型对不同的块保持相同，仅列数据不同。最好把块数据（`block data`）和块头（`block header`）分离开来，因为小块大小会因复制共享指针和列名而带来很高的临时字符串开销。

## 块流（Block Streams）

块流用于处理数据。我们可以使用块流从某个地方读取数据，执行数据转换，或将数据写到某个地方。`IBlockInputStream`  具有  `read`  方法，其能够在数据可用时获取下一个块。`IBlockOutputStream`  具有  `write`  方法，其能够将块写到某处。

块流负责：

1.  读或写一个表。表仅返回一个流用于读写块。
2.  完成数据格式化。比如，如果你打算将数据以  `Pretty` 格式输出到终端，你可以创建一个块输出流，将块写入该流中，然后进行格式化。
3.  执行数据转换。假设你现在有  `IBlockInputStream`  并且打算创建一个过滤流，那么你可以创建一个  `FilterBlockInputStream`  并用  `IBlockInputStream`  进行初始化。之后，当你从  `FilterBlockInputStream`  中拉取块时，会从你的流中提取一个块，对其进行过滤，然后将过滤后的块返回给你。查询执行流水线就是以这种方式表示的。

还有一些更复杂的转换。比如，当你从  `AggregatingBlockInputStream`  拉取数据时，会从数据源读取全部数据进行聚集，然后将聚集后的数据流返回给你。另一个例子：`UnionBlockInputStream`  的构造函数接受多个输入源和多个线程，其能够启动多线程从多个输入源并行读取数据。

> 块流使用“pull”方法来控制流：当你从第一个流中拉取块时，它会接着从嵌套的流中拉取所需的块，然后整个执行流水线开始工作。”pull“和“push”都不是最好的方案，因为控制流不是明确的，这限制了各种功能的实现，比如多个查询同步执行（多个流水线合并到一起）。这个限制可以通过协程或直接运行互相等待的线程来解决。如果控制流明确，那么我们会有更多的可能性：如果我们定位了数据从一个计算单元传递到那些外部的计算单元中其中一个计算单元的逻辑。阅读这篇[文章](http://journal.stuffwithstuff.com/2013/01/13/iteration-inside-and-out/)来获取更多的想法。

我们需要注意，查询执行流水线在每一步都会创建临时数据。我们要尽量使块的大小足够小，从而 CPU 缓存能够容纳下临时数据。在这个假设下，与其他计算相比，读写临时数据几乎是没有任何开销的。我们也可以考虑一种替代方案：将流水线中的多个操作融合在一起，使流水线尽可能短，并删除大量临时数据。这可能是一个优点，但同时也有缺点。比如，拆分流水线使得中间数据缓存、获取同时运行的类似查询的中间数据以及相似查询的流水线合并等功能很容易实现。

## 格式（Formats）[¶](https://clickhouse.yandex/docs/zh/development/architecture/#ge-shi-formats "Permanent link")

数据格式同块流一起实现。既有仅用于向客户端输出数据的”展示“格式，如  `IBlockOutputStream`  提供的  `Pretty`  格式，也有其它输入输出格式，比如  `TabSeparated`  或  `JSONEachRow`。

此外还有行流：`IRowInputStream`  和  `IRowOutputStream`。它们允许你按行 pull/push 数据，而不是按块。行流只需要简单地面向行格式实现。包装器  `BlockInputStreamFromRowInputStream`  和  `BlockOutputStreamFromRowOutputStream`  允许你将面向行的流转换为正常的面向块的流。

## I/O[¶](https://clickhouse.yandex/docs/zh/development/architecture/#i-o "Permanent link")

对于面向字节的输入输出，有  `ReadBuffer`  和  `WriteBuffer`  这两个抽象类。它们用来替代 C++ 的  `iostream`。不用担心：每个成熟的 C++ 项目都会有充分的理由使用某些东西来代替  `iostream`。

`ReadBuffer`  和  `WriteBuffer`  由一个连续的缓冲区和指向缓冲区中某个位置的一个指针组成。实现中，缓冲区可能拥有内存，也可能不拥有内存。有一个虚方法会使用随后的数据来填充缓冲区（针对  `ReadBuffer`）或刷新缓冲区（针对  `WriteBuffer`），该虚方法很少被调用。

`ReadBuffer`  和  `WriteBuffer`  的实现用于处理文件、文件描述符和网络套接字（socket），也用于实现压缩（`CompressedWriteBuffer`  在写入数据前需要先用一个  `WriteBuffer`  进行初始化并进行压缩）和其它用途。`ConcatReadBuffer`、`LimitReadBuffer`  和  `HashingWriteBuffer`  的用途正如其名字所描述的一样。

`ReadBuffer`  和  `WriteBuffer`  仅处理字节。为了实现格式化输入和输出（比如以十进制格式写一个数字），`ReadHelpers`  和  `WriteHelpers`  头文件中有一些辅助函数可用。

让我们来看一下，当你把一个结果集以  `JSON`  格式写到标准输出（stdout）时会发生什么。你已经准备好从  `IBlockInputStream`  获取结果集，然后创建  `WriteBufferFromFileDescriptor(STDOUT_FILENO)`  用于写字节到标准输出，创建  `JSONRowOutputStream`  并用  `WriteBuffer`  初始化，用于将行以  `JSON`  格式写到标准输出，你还可以在其上创建  `BlockOutputStreamFromRowOutputStream`，将其表示为  `IBlockOutputStream`。然后调用  `copyData`  将数据从  `IBlockInputStream`  传输到  `IBlockOutputStream`，一切工作正常。在内部，`JSONRowOutputStream`  会写入 JSON 分隔符，并以指向  `IColumn`  的引用和行数作为参数调用  `IDataType::serializeTextJSON`  函数。随后，`IDataType::serializeTextJSON`  将会调用  `WriteHelpers.h`  中的一个方法：比如，`writeText`  用于数值类型，`writeJSONString`  用于  `DataTypeString` 。

## 表（Tables）

表由  `IStorage`  接口表示。该接口的不同实现对应不同的表引擎。比如  `StorageMergeTree`、`StorageMemory`  等。这些类的实例就是表。

`IStorage`  中最重要的方法是  `read`  和  `write`，除此之外还有  `alter`、`rename`  和  `drop`  等方法。`read`  方法接受如下参数：需要从表中读取的列集，需要执行的  `AST`  查询，以及所需返回的流的数量。`read`  方法的返回值是一个或多个  `IBlockInputStream`  对象，以及在查询执行期间在一个表引擎内完成的关于数据处理阶段的信息。

在大多数情况下，`read`  方法仅负责从表中读取指定的列，而不会进行进一步的数据处理。进一步的数据处理均由查询解释器完成，不由  `IStorage`  负责。

但是也有值得注意的例外：

- AST 查询被传递给  `read`  方法，表引擎可以使用它来判断是否能够使用索引，从而从表中读取更少的数据。
- 有时候，表引擎能够将数据处理到一个特定阶段。比如，`StorageDistributed`  可以向远程服务器发送查询，要求它们将来自不同的远程服务器能够合并的数据处理到某个阶段，并返回预处理后的数据，然后查询解释器完成后续的数据处理。

表的  `read`  方法能够返回多个  `IBlockInputStream`  对象以允许并行处理数据。多个块输入流能够从一个表中并行读取。然后你可以通过不同的转换对这些流进行装饰（比如表达式求值或过滤），转换过程能够独立计算，并在其上创建一个  `UnionBlockInputStream`，以并行读取多个流。

另外也有  `TableFunction`。`TableFunction`  能够在查询的  `FROM`  字句中返回一个临时的  `IStorage`  以供使用。

要快速了解如何实现自己的表引擎，可以查看一些简单的表引擎，比如  `StorageMemory`  或  `StorageTinyLog`。

> 作为  `read`  方法的结果，`IStorage`  返回  `QueryProcessingStage` \- 关于 storage 里哪部分查询已经被计算的信息。当前我们仅有非常粗粒度的信息。Storage 无法告诉我们“对于这个范围的数据，我已经处理完了 WHERE 字句里的这部分表达式”。我们需要在这个地方继续努力。

## 解析器（Parsers）

查询由一个手写递归下降解析器解析。比如， `ParserSelectQuery`  只是针对查询的不同部分递归地调用下层解析器。解析器创建  `AST`。`AST`  由节点表示，节点是  `IAST`  的实例。

> 由于历史原因，未使用解析器生成器。

## 解释器（Interpreters）

解释器负责从  `AST`  创建查询执行流水线。既有一些简单的解释器，如  `InterpreterExistsQuery`  和  `InterpreterDropQuery`，也有更复杂的解释器，如  `InterpreterSelectQuery`。查询执行流水线由块输入或输出流组成。比如，`SELECT`  查询的解释结果是从  `FROM`  字句的结果集中读取数据的  `IBlockInputStream`；`INSERT`  查询的结果是写入需要插入的数据的  `IBlockOutputStream`；`SELECT INSERT`  查询的解释结果是  `IBlockInputStream`，它在第一次读取时返回一个空结果集，同时将数据从  `SELECT`  复制到  `INSERT`。

`InterpreterSelectQuery`  使用  `ExpressionAnalyzer`  和  `ExpressionActions`  机制来进行查询分析和转换。这是大多数基于规则的查询优化完成的地方。`ExpressionAnalyzer`  非常混乱，应该进行重写：不同的查询转换和优化应该被提取出来并划分成不同的类，从而允许模块化转换或查询。

## 函数（Functions）

函数既有普通函数，也有聚合函数。对于聚合函数，请看下一节。

普通函数不会改变行数 \- 它们的执行看起来就像是独立地处理每一行数据。实际上，函数不会作用于一个单独的行上，而是作用在以  `Block`  为单位的数据上，以实现向量查询执行。

还有一些杂项函数，比如  [blockSize](https://clickhouse.yandex/docs/zh/query_language/functions/other_functions/#function-blocksize)、[rowNumberInBlock](https://clickhouse.yandex/docs/zh/query_language/functions/other_functions/#function-rownumberinblock)，以及  [runningAccumulate](https://clickhouse.yandex/docs/zh/query_language/functions/other_functions/#function-runningaccumulate)，它们对块进行处理，并且不遵从行的独立性。

ClickHouse 具有强类型，因此隐式类型转换不会发生。如果函数不支持某个特定的类型组合，则会抛出异常。但函数可以通过重载以支持许多不同的类型组合。比如，`plus`  函数（用于实现  `+`  运算符）支持任意数字类型的组合：`UInt8` + `Float32`，`UInt16` + `Int8`  等。同时，一些可变参数的函数能够级接收任意数目的参数，比如  `concat`  函数。

实现函数可能有些不方便，因为函数的实现需要包含所有支持该操作的数据类型和  `IColumn`  类型。比如，`plus`  函数能够利用 C++ 模板针对不同的数字类型组合、常量以及非常量的左值和右值进行代码生成。

> 这是一个实现动态代码生成的好地方，从而能够避免模板代码膨胀。同样，运行时代码生成也使得实现融合函数成为可能，比如融合“乘-加”，或者在单层循环迭代中进行多重比较。

由于向量查询执行，函数不会“短路”。比如，如果你写  `WHERE f(x) AND g(y)`，两边都会进行计算，即使是对于  `f(x)`  为 0 的行（除非  `f(x)`  是零常量表达式）。但是如果  `f(x)`  的选择条件很高，并且计算  `f(x)`  比计算  `g(y)`  要划算得多，那么最好进行多遍计算：首先计算  `f(x)`，根据计算结果对列数据进行过滤，然后计算  `g(y)`，之后只需对较小数量的数据进行过滤。

## 聚合函数

聚合函数是状态函数。它们将传入的值激活到某个状态，并允许你从该状态获取结果。聚合函数使用  `IAggregateFunction`  接口进行管理。状态可以非常简单（`AggregateFunctionCount`  的状态只是一个单一的`UInt64`  值），也可以非常复杂（`AggregateFunctionUniqCombined`  的状态是由一个线性数组、一个散列表和一个  `HyperLogLog`  概率数据结构组合而成的）。

为了能够在执行一个基数很大的  `GROUP BY`  查询时处理多个聚合状态，需要在  `Arena`（一个内存池）或任何合适的内存块中分配状态。状态可以有一个非平凡的构造器和析构器：比如，复杂的聚合状态能够自己分配额外的内存。这需要注意状态的创建和销毁并恰当地传递状态的所有权，以跟踪谁将何时销毁状态。

聚合状态可以被序列化和反序列化，以在分布式查询执行期间通过网络传递或者在内存不够的时候将其写到硬盘。聚合状态甚至可以通过  `DataTypeAggregateFunction`  存储到一个表中，以允许数据的增量聚合。

> 聚合函数状态的序列化数据格式目前尚未版本化。如果只是临时存储聚合状态，这样是可以的。但是我们有  `AggregatingMergeTree`  表引擎用于增量聚合，并且人们已经在生产中使用它。这就是为什么在未来当我们更改任何聚合函数的序列化格式时需要增加向后兼容的支持。

## 服务器（Server）

服务器实现了多个不同的接口：

- 一个用于任何外部客户端的 HTTP 接口。
- 一个用于本机 ClickHouse 客户端以及在分布式查询执行中跨服务器通信的 TCP 接口。
- 一个用于传输数据以进行拷贝的接口。

在内部，它只是一个没有协程、纤程等的基础多线程服务器。服务器不是为处理高速率的简单查询设计的，而是为处理相对低速率的复杂查询设计的，每一个复杂查询能够对大量的数据进行处理分析。

服务器使用必要的查询执行需要的环境初始化  `Context`  类：可用数据库列表、用户和访问权限、设置、集群、进程列表和查询日志等。这些环境被解释器使用。

我们维护了服务器 TCP 协议的完全向后向前兼容性：旧客户端可以和新服务器通信，新客户端也可以和旧服务器通信。但是我们并不想永久维护它，我们将在大约一年后删除对旧版本的支持。

> 对于所有的外部应用，我们推荐使用 HTTP 接口，因为该接口很简单，容易使用。TCP 接口与内部数据结构的联系更加紧密：它使用内部格式传递数据块，并使用自定义帧来压缩数据。我们没有发布该协议的 C 库，因为它需要链接大部分的 ClickHouse 代码库，这是不切实际的。

## 分布式查询执行

集群设置中的服务器大多是独立的。你可以在一个集群中的一个或多个服务器上创建一个  `Distributed`  表。

- 
  - 不存储数据。
  - 它只为集群的多个节点上的所有本地表提供一个 `view`。

- 从 `Distributed`  表SELECT 时： 
  - 它会重写该查询，根据负载平衡设置来选择并将查询发送给服务器节点。
  - 请求远程服务器处理查询。
  - 接收并合并查询结果。
  - 分布式表会尝试将尽可能多的工作分配给远程服务器，并且不会通过网络发送太多的中间数据。

> 当  `IN`  或  `JOIN`  子句中包含子查询并且每个子查询都使用分布式表时，事情会变得更加复杂。我们有不同的策略来执行这些查询。

- 分布式查询执行没有全局查询计划。
  - 每个节点都有针对自己的工作部分的本地查询计划。
  - 仅有简单的一次性分布式查询执行：
    - 将查询发送给远程节点，然后合并结果。
    - 但是对于具有高基数的  `GROUP BY`  或具有大量临时数据的  `JOIN`  这样困难的查询的来说，这是不可行的：
      - 在这种情况下，我们需要在服务器之间“改组”数据，这需要额外的协调
      - `ClickHouse` 不支持这类查询执行，官方指明后续会改进。

## Merge Tree

- `MergeTree`  是一种 `storage engines`。
  - 按主键索引
  - `table`
    - 数据存储于按字典排序的 `parts`中。 
    - 列都存储在这些对应`parts`中 由压缩快(一般对应 `64KB` 到 `1MB`大小的未压缩数据，具体取决于平均值大小) 组成`column.bin`  文件
    - 列值按主键排序，一个块可存储多个列值。

- 主键
  - 组成
    - 列
    - 元组
  - 特性
    - 稀疏(`sparse`)
      - 索引某个范围内的数据
      - 一个单独的  `primary.idx`  文件具有每个第 N 行的主键值,其中 N 称为  `index_granularity`（通常，N = 8192）
      - 对于每一列，都有带有标记的  `column.mrk`  文件，该文件记录的是每个第 N 行在数据文件中的偏移量。
      - 每个标记是一个 pair：文件中的偏移量到压缩块的起始，以及解压缩块中的偏移量到数据的起始。
      - 压缩块根据标记对齐，并且解压缩块中的偏移量为 0。`primary.idx`  的数据始终驻留在内存，同时  `column.mrk`  的数据被缓存。

从  `MergeTree`  的一个分块中读取部分内容: 
  - 我们会查看  `primary.idx`  数据并查找可能包含所请求数据的范围；
  - 然后查看  `column.mrk`  并计算偏移量从而得知从哪里开始读取些范围的数据。
  
- 由于稀疏性，可能会读取额外的数据。
  - `ClickHouse` 不适用于高负载的简单点查询，因为对于每一个键，整个  `index_granularity`  范围的行的数据都需要读取，并且对于每一列需要解压缩整个压缩块。
  - 我们使索引稀疏，是因为每一个单一的服务器需要在索引没有明显内存消耗的情况下，维护数万亿行的数据。
  - 另外，由于主键是稀疏的，导致其不是唯一的：
    - 无法在 INSERT 时检查一个键在表中是否存在。你可以在一个表中使用同一个键创建多个行。

- `insert`
  - 当你向  `MergeTree`  中插入一堆数据时，数据按主键排序并形成一个新的`parts`。
  - 为了保证分块的数量相对较少，有后台线程定期选择一些分块并将它们合并成一个有序的分块，这就是  `MergeTree`  的名称来源。
  - 当然，合并会导致“写入放大”.
  - 合并之后，还会保留旧的分块一段时间，以便发生故障后更容易恢复，因此如果我们发现某些合并后的分块可能已损坏，我们可以将其替换为原分块。
  
- 所有的分块都是不可变的：它们仅会被创建和删除，不会被修改。 
  - 当运行  `SELECT` 请求时，`MergeTree`  会保存一个表的快照（ `parts` 集合）。

`MergeTree`  不是 `LSM` 树，因为它不包含`memtable`和`log`：
  - 插入的数据直接写入文件系统。
  - 这使得它仅适用于批量插入数据，而不适用于非常频繁地一行一行插入 - 大约每秒一次是没问题的，但是每秒一千次就会有问题。
  - 官方用这个方案，是为了简单起见，并经过大量的测试。

> `MergeTree table`只能有一个（`primary`）索引：没有任何 `secondary` 索引。在一个逻辑表下，允许有多个物理表示，比如，可以以多个物理顺序存储数据，或者同时表示预聚合数据和原始数据。

- 有些  `MergeTree`  引擎会在后台合并期间做一些额外工作，比如:
  -  `CollapsingMergeTree`  和  `AggregatingMergeTree`
    - 这可以视为对更新的特殊支持。
    - 请记住这些不是真正的更新，因为用户通常无法控制后台合并将会执行的时间。
    - 并且  `MergeTree`  中的数据几乎总是存储在多个分块中，而不是完全合并的形式。

## 复制（Replication）

- 基于`per-table`实现的。
  - 同一服务器可以同时存在 `replicated` 和 `non-replicated` 表
  - 定制化表副本数：
    - A 表2个副本， B表3个副本等

- 在`ReplicatedMergeTree`  `Storatge engine` 中实现的。
  - `ZooKeeper`  中的路径被指定为 `Storage engine` 的参数。
  - `ZooKeeper`  中所有具有相同路径的表互为副本：它们同步数据并保持一致性。只需创建或删除表，就可以实现动态添加或删除副本。

- 使用异步多主机方案
  - 你可以将数据插入到与 `ZooKeeper` 进行会话的任意副本中，并将数据复制到所有其它副本中。
  - 由于 `ClickHouse` 不支持 `UPDATE`，因此复制是无冲突的。
  - 由于没有对插入数据的仲裁机制，如果一个节点发生故障，刚刚插入的数据可能会丢失。

- 用于复制的元数据存储在 ZooKeeper 中。
  - 其中一个复制日志列出了要执行的操作。操作包括：
    - 获取分块
    - 合并分块
    - 删除分区等。
  - 每一个副本将复制日志复制到其队列中，然后执行队列中的操作。
    - 比如，在插入时，在复制日志中创建“获取分块”这一操作，然后每一个副本都会去下载该分块。
    - 所有副本之间会协调进行合并以获得相同字节的结果。
    - 所有的分块在所有的副本上以相同的方式合并。
    - 为实现该目的，其中一个副本被选为`leader`，该副本首先进行合并，并把“合并分块”操作写到日志中。

- 复制是物理的：
  - 只有压缩的分块会在节点之间传输，查询则不会。
  - 为了降低网络成本（避免网络放大），大多数情况下，会在每一个副本上独立地处理合并。
  - 只有在存在显著的合并延迟的情况下，才会通过网络发送大块的合并分块。

- 每一个副本将其状态作为分块和校验和组成的集合存储在 ZooKeeper 中。
  - 当本地文件系统中的状态与 ZooKeeper 中引用的状态不同时，该副本会通过从其它副本下载缺失和损坏的分块来恢复其一致性。
  - 当本地文件系统中出现一些意外或损坏的数据时，ClickHouse 不会将其删除，而是将其移动到一个单独的目录下并忘记它。

- ClickHouse 集群由独立的分片组成，每一个分片由多个副本组成。
  - 集群不是弹性的，因此在添加新的分片后，数据不会自动在分片之间重新平衡(集群负载不均衡)。
  - 我们应该实现一个表引擎，使得该引擎能够跨集群扩展数据，同时具有动态复制的`regions`，这些`regions`能够在集群之间自动拆分和平衡。


--- 

# 如何构建 ClickHouse 发布包

## 安装 Git 和 Pbuilder [¶](https://clickhouse.yandex/docs/en/development/build/#install-git-and-pbuilder "永久链接")

$ sudo apt-get更新
$ sudo apt-get install git pbuilder debhelper lsb-release fakeroot sudo debian-archive-keyring debian-keyring

## 结帐 ClickHouse 来源[¶](https://clickhouse.yandex/docs/en/development/build/#checkout-clickhouse-sources "永久链接")

$ git clone-递归--branch master https://github.com/ClickHouse/ClickHouse.git
$ cd ClickHouse

## 运行脚本版[¶](https://clickhouse.yandex/docs/en/development/build/#run-release-script "永久链接")

\$ ./版本

# 如何建立 ClickHouse 进行开发

以下教程基于 Ubuntu Linux 系统。经过适当的更改，它也应该可以在任何其他 Linux 发行版上运行。仅支持带有 SSE 4.2 的 x86_64。对 AArch64 的支持是试验性的。

要测试 SSE 4.2，请执行

\$ grep -q sse4_2 / proc / cpuinfo && 回显 “支持 SSE 4.2” || 回显 “不支持 SSE 4.2”

## 安装 Git 和 CMake 的[¶](https://clickhouse.yandex/docs/en/development/build/#install-git-and-cmake "永久链接")

\$ sudo apt-get install git cmake ninja-build

或使用 cmake3 代替旧系统上的 cmake。

## 安装 GCC [9¶](https://clickhouse.yandex/docs/en/development/build/#install-gcc-9 "永久链接")

有几种方法可以做到这一点。

### 从 PPA 软件包安装[¶](https://clickhouse.yandex/docs/en/development/build/#install-from-a-ppa-package "永久链接")

$ sudo apt-get install software-properties-common
$ sudo apt-add-repository ppa：ubuntu-toolchain-r / test
$ sudo apt-get更新
$ sudo apt-get install gcc-9 g ++-9

### 从源代码安装[¶](https://clickhouse.yandex/docs/en/development/build/#install-from-sources "永久链接")

查看[utils / ci / build-gcc-from-sources.sh](https://github.com/ClickHouse/ClickHouse/blob/master/utils/ci/build-gcc-from-sources.sh)

## 使用 GCC 9 进行构建[¶](https://clickhouse.yandex/docs/en/development/build/#use-gcc-9-for-builds "永久链接")

$ 出口 CC = gcc-9
$ export CXX = g ++-9

## 从包安装所需的库[¶](https://clickhouse.yandex/docs/en/development/build/#install-required-libraries-from-packages "永久链接")

\$ sudo apt-get install libicu-dev libreadline-dev gperf

## 结帐 ClickHouse 来源[¶](https://clickhouse.yandex/docs/en/development/build/#checkout-clickhouse-sources_1 "永久链接")

\$ git clone-递归 git@github.com：ClickHouse / ClickHouse.git

要么

$ git clone-递归https://github.com/ClickHouse/ClickHouse.git
$ cd ClickHouse

## 构建 ClickHouse [¶](https://clickhouse.yandex/docs/en/development/build/#build-clickhouse "永久链接")

$ mkdir构建
$ cd 构建
$ cmake ..
$忍者
\$ cd ..

要创建可执行文件，请运行`ninja clickhouse`。这将创建`dbms/programs/clickhouse`可执行文件，该可执行文件可以与`client`或`server`参数一起使用。

# 如何在 Mac OS X 上构建 ClickHouse

构建应可在 Mac OS X 10.12 上运行。

## 安装家酿[¶](https://clickhouse.yandex/docs/en/development/build_osx/#install-homebrew "永久链接")

$ / usr / bin / ruby -e “ $（ curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install ）”

## 安装所需的编译器，工具和库[¶](https://clickhouse.yandex/docs/en/development/build_osx/#install-required-compilers-tools-and-libraries "永久链接")

\$ brew install cmake 忍者 gcc icu4c openssl libtool gettext readline gperf

## 结帐 ClickHouse 来源[¶](https://clickhouse.yandex/docs/en/development/build_osx/#checkout-clickhouse-sources "永久链接")

\$ git clone-递归 git@github.com：yandex / ClickHouse.git

要么

\$ git clone-递归https://github.com/yandex/ClickHouse.git

\$ cd ClickHouse

对于最新的稳定版本，请切换到`stable`分支。

## 构建 ClickHouse [¶](https://clickhouse.yandex/docs/en/development/build_osx/#build-clickhouse "永久链接")

$ mkdir构建
$ cd 构建
$ cmake的.. -DCMAKE\_CXX\_COMPILER = `其中，g ++ - 8 ` -DCMAKE\_C\_COMPILER = `其中GCC-8 `
$忍者
\$ cd ..

## 注意事项[¶](https://clickhouse.yandex/docs/en/development/build_osx/#caveats "永久链接")

如果要运行 clickhouse-server，请确保增加系统的 maxfiles 变量。

注意

您将需要使用 sudo。

为此，创建以下文件：

/Library/LaunchDaemons/limit.maxfiles.plist：

<？xml version =“ 1.0” encoding =“ UTF-8”？>
<！DOCTYPE plist PUBLIC“-// Apple // DTD PLIST 1.0 // EN”
“ http://www.apple.com/DTDs/PropertyList -1.0.dtd“>
<plist version = ” 1.0“ \>
<dict>
<key>标签</ key>
<string> limit.maxfiles </ string>
<key> ProgramArguments </ key>
<array>
<string> launchctl </ string>
<string>限制</ string>
<string> maxfiles </ string>
<string> 524288 </ string>
<string> 524288 </ string>
</ array>
<key>RunAtLoad </ key>
<true />
<key> ServiceIPC </ key>
<false />
</ dict>
</ plist>

执行以下命令：

\$ sudo chown root：wheel /Library/LaunchDaemons/limit.maxfiles.plist

重启。

要检查它是否正常工作，可以使用`ulimit -n`命令。

# 如何在 Mac OS X 的 Linux 上构建 ClickHouse

这是当您拥有 Linux 机器并想使用它来构建`clickhouse`将在 OS X 上运行的二进制文件时的情况。这用于在 Linux 服务器上运行的连续集成检查。如果要直接在 Mac OS X 上构建 ClickHouse，请继续执行另一条指令：https://clickhouse.yandex/docs/en/development/build_osx/

Mac OS X 的交叉构建基于构建说明，请首先遵循它们。

# 安装 Clang-8

按照https://apt.llvm.org/中的说明进行Ubuntu或Debian安装。例如，用于Bionic的命令如下：

sudo echo “ deb \[trusted = yes\] http://apt.llvm.org/bionic/ llvm-toolchain-bionic-8 main” >\> /etc/apt/sources.list
须藤 apt-get install clang-8

# 安装交叉编译工具集

让我们记住`cctools`以\$ {CCTOOLS}  安装的路径

mkdir \$ { CCTOOLS }

git clone https://github.com/tpoechtrager/apple-libtapi.git
cd 苹果 libtapi
INSTALLPREFIX = \$ { CCTOOLS } ./build.sh
./install.sh
光盘 ..

git clone https://github.com/tpoechtrager/cctools-port.git
cd cctools-port / cctools
./configure --prefix = $ { CCTOOLS } --with-libtapi = $ { CCTOOLS } --target = x86_64-apple-darwin
进行安装

cd \$ { CCTOOLS }
wget https://github.com/phracker/MacOSX-SDKs/releases/download/10.14-beta4/MacOSX10.14.sdk.tar.xz
tar xJf MacOSX10.14.sdk.tar.xz

# 建立 ClickHouse

cd ClickHouse
mkdir build-osx
CC = clang-8 CXX = clang ++-8 cmake。-Bbuild-osx -DCMAKE_SYSTEM_NAME =达尔文\
-DCMAKE_AR：FILEPATH = $ { CCTOOLS } / bin / x86_64-apple-darwin-ar \ 
    -DCMAKE_RANLIB：FILEPATH = $ { CCTOOLS } / bin / x86_64-apple-darwin-ranlib \
\- DLINKER_NAME = $ { CCTOOLS } / bin / x86_64-apple-darwin-ld \ 
    -DSDK_PATH = $ { CCTOOLS } /MacOSX10.14.sdk
忍者-C build-osx

生成的二进制文件将具有 Mach-O 可执行格式，并且不能在 Linux 上运行。

# 如何编写 C ++代码

## 一般建议[¶](https://clickhouse.yandex/docs/en/development/style/#general-recommendations "永久链接")

**1.**以下是建议而非要求。

**2.**如果您正在编辑代码，则遵循现有代码的格式是有意义的。

**3.**需要代码样式以保持一致性。一致性使读取代码更容易，也使搜索代码更容易。

**4.**许多规则没有逻辑理由；它们是由既定惯例决定的。

## 格式化[¶](https://clickhouse.yandex/docs/en/development/style/#formatting "永久链接")

**1.**大多数格式化将由自动完成`clang-format`。

**2.**缩进为 4 个空格。配置您的开发环境，以便选项卡添加四个空格。

**3.**打开和关闭大括号必须放在单独的行上。

内联 void readBoolText （bool ＆ x ， ReadBuffer ＆ buf ）
{
char tmp = '0' ;
readChar （tmp ， buf ）;
x = tmp ！= '0' ;
}

**4.**如果整个功能主体为单个`statement`，则可以将其放在一行上。在花括号周围放置空格（除了行尾的空格）。

内联 size_t mask （） const { return buf_size （） \- 1 ; }
内联 size_t 位置（HashValue x ） const { return x ＆ mask （）; }

**5.**用于功能。不要在方括号内放置空格。

无效 重新插入（const Value ＆ x ）

memcpy （＆buf \[ place_value \]， ＆x ， sizeof （x ））;

**6.**在`if`，`for`，`while`和其他表达式，一个空间被插入到左括号前面（相对于函数调用）。

为 （为 size_t 我 = 0 ; 我 < 行; 我 \+ = 存储。index_granularity ）

**7.**添加围绕二元运算符（空格`+`，`-`，`*`，`/`，`%`，...）和三元运算符`?:`。

UINT16 年 = （小号\[ 0 \] \- '0' ） \* 1000 \+ （小号\[ 1 \] \- '0' ） \* 100 \+ （小号\[ 2 \] \- '0' ） \* 10 \+ （小号\[ 3 \] \- “0 ' ）;
UInt8 month = （s \[ 5 \] \- '0' ） \* 10 \+ （s \[ 6 \] \- '0' ）;
UInt8 天 = （s \[ 8 \] \- '0' ） \* 10 \+ （s \[ 9 \] \- '0' ）;

**8.**如果输入换行符，则将操作员放在新行上并增加其缩进量。

if （elapsed_ns ）
消息 << “（（
<< << rows_read_on_server \* 1000000000 / elapsed_ns << ” rows / s。，“
<< 字节读取\_on_server \* 1000.0 / elapsed_ns << ” MB / s。）“ ;

**9.**如果需要，可以在一行中使用空格对齐。

dst 。ClickLogID = 单击。LogID ;
dst 。ClickEventID = 单击。EventID ;
dst 。ClickGoodEvent = 单击。GoodEvent ;

**10.**请勿在运算符`.`，周围使用空格`->`。

如有必要，可以将操作员转到下一行。在这种情况下，其前面的偏移会增加。

**11.**不要使用空格元运算符（分隔`--`，`++`，`*`，`&`，...）从参数。

**12.**在逗号后但不要在空格前。对于`for`表达式内部的分号，使用相同的规则。

**13.**请勿使用空格分隔`[]`操作符。

**14.**在`template <...>`表达式中，在`template`和之间使用空格`<`。`<`前后没有空格`>`。

模板 < 类型名 TKey ， 类型名 TValue \>
struct AggregatedStatElement
{}

**15.**在类和结构中，分别在和上编写`public`，`private`和，并缩进其余代码。` protected``class/struct `

template < typename T \>
class MultiVersion
{
public ：
///使用对象的版本。shared_ptr 管理版本的生存期。
使用 Version = std :: shared_ptr < const T \> ;
...
}

**16.**如果`namespace`整个文件都使用相同的东西，并且没有其他重要意义，则不需要在内部偏移`namespace`。

**17.**如果一个块`if`，`for`，`while`，或其它表达由一个单一的`statement`，大括号是可选的。`statement`而是将其放在单独的行上。这条规则也适用于嵌套`if`，`for`，`while`，...

但是，如果内部`statement`包含大括号或`else`，则外部块应写在大括号中。

///完成写入。
用于 （auto ＆ stream ： stream ）
流。第二个-\> finalize （）;

**18.行尾**不应有空格。

**19.**源文件是 UTF-8 编码的。

**20.**非 ASCII 字符可用于字符串文字中。

<< “ ” << （定时器。经过（） / chunks_stats 。点击） << “微秒/打。” ;

**21**不要在一行中写多个表达式。

**22.将**函数内部的代码段分组，并用不超过一行的空行分隔它们。

**23.**用一两个空行分隔函数，类等。

**24.** `A const`（与值有关）必须在类型名称之前编写。

//正确的
const char \* pos
const std :: string ＆ s
//不正确的
char const \* pos

**25.**在声明指针或引用时，`*`和`&`符号应在两侧用空格隔开。

//正确的
const char \* pos
//不正确的
const char \* pos
const char \* pos

**26.**使用模板类型时，请为它们加上`using`关键字的别名（在最简单的情况下除外）。

换句话说，模板参数仅在`using`代码中指定，并且不会在代码中重复。

`using`  可以在本地声明，例如在函数内部。

//
使用 FileStreams = std :: map < std :: string ， std :: shared_ptr < Stream >\> ;
FileStreams 流；
//不正确的
std :: map < std :: string ， std :: shared_ptr < Stream >\> stream ;

**27.**不要在一个语句中声明多个不同类型的变量。

//不正确的
int x ， \* y ;

**28.**不要使用 C 样式转换。

//不正确的
std :: cerr << （int ）c << ; std :: endl ;
//正确的
std :: cerr << static_cast < int \> （c ） << std :: endl ;

**29.**在类和结构中，在每个可见性范围内分别对成员和函数进行分组。

**30.**对于小类和结构，没有必要将方法声明与实现分开。

对于任何类或结构中的小型方法，情况也是如此。

对于模板化的类和结构，请勿将方法声明与实现分开（因为否则必须在同一转换单元中定义它们）。

**31.**您可以将行换成 140 个字符，而不是 80 个字符。

**32.**如果不需要后缀，请始终使用前缀递增/递减运算符。

为 （名称:: 为 const_iterator 它 = COLUMN_NAMES 。开始（）; 它 ！= COLUMN_NAMES 。端（）; \+\+ 它）

## 注释[¶](https://clickhouse.yandex/docs/en/development/style/#comments "永久链接")

**1.**确保为代码的所有重要部分添加注释。

这个非常重要。编写注释可能会帮助您认识到代码不是必需的，或者它被设计为错误的。

/ \**可以使用的一部分内存。
*例如，如果 internal_buffer 为 1MB，并且只有 10 个字节从文件加载到缓冲区以供读取，则
_然后 working_buffer 的大小仅为 10 个字节
_（working_buffer.end（）将指向这 10 个字节之后的位置供阅读）。 \* /

**2.**评论可以根据需要进行详细说明。

**3.**将注释放在它们描述的代码之前。在极少数情况下，注释可以在代码后的同一行。

/ \*\*解析并执行查询。 \* /
void executeQuery （
ReadBuffer ＆ istr ， ///从何处读取查询（以及 INSERT 的数据，如果适用）
WriteBuffer ＆ ostr ， ///从何处写入结果
Context ＆ context ， /// DB，表，数据类型，引擎，函数，聚合函数...
BlockInputStreamPtr ＆ query_plan ， ///可以在此处编写有关如何执行查询的描述
QueryProcessingStage :: Enum stage = QueryProcessingStage :: Complete ///到哪个阶段处理 SELECT 查询
）

**4.**评论应仅以英文撰写。

**5.**如果要编写库，请在主头文件中包含解释它的详细注释。

**6.**不要添加不提供其他信息的注释。特别是，请勿留下这样的空注释：

/ \*
*过程名称：
*原始过程名称：
*作者：
*创建
日期：
*修改
日期：*修改作者：*原始文件名：
*用途：
*目的：
*名称：
*使用的类：
*常量：
*局部变量：
*参数：
*创建日期：
*目的： \* /

该示例是从资源[http://home.tamk.fi/~jaalto/course/coding-style/doc/unmaintainable-code/](http://home.tamk.fi/~jaalto/course/coding-style/doc/unmaintainable-code/)借来的。

**7.**不要在每个文件的开头写垃圾评论（作者，创建日期..）。

**8.**单行注释以三个斜杠开头：`///`多行注释以开头`/**`。这些评论被视为“文档”。

注意：您可以使用 Doxygen 从这些注释生成文档。但是通常不使用 Doxygen，因为在 IDE 中浏览代码更加方便。

**9.**多行注释的开头和结尾不得有空行（关闭多行注释的行除外）。

**10.**要注释掉代码，请使用基本注释，而不是“记录”注释。

**11.**提交之前，删除代码中注释掉的部分。

**12.**不要在注释或代码中使用亵渎行为。

**13.**请勿使用大写字母。不要使用过多的标点符号。

///失败了吗？？？

**14.**请勿使用注释进行分隔。

/// \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\* **\*\*\***

**15.**不要在评论中开始讨论。

///您为什么要这样做？

**16.**无需在块末尾写注释来描述其含义。

///为

## 名称[¶](https://clickhouse.yandex/docs/en/development/style/#names "永久链接")

**1.**在变量和类成员的名称中使用带下划线的小写字母。

size_t max_block_size ;

**2.**对于功能（方法）的名称，请使用以小写字母开头的 camelCase。

std :: 字符串 getName （） const 覆盖 { 返回 “内存” ； }

**3.**对于类（结构）的名称，请使用以大写字母开头的 CamelCase。除 I 以外的前缀不用于接口。

StorageMemory 类： 公共 IStorage

**4.** `using`命名方式与类相同，或以`_t`末尾命名。

**5.**模板类型参数的名称：在简单情况下，请使用`T`; `T`，`U`; `T1`，`T2`。

对于更复杂的情况，请遵循类名称的规则，或添加 prefix `T`。

模板 < 类型名 TKey ， 类型名 TValue \>
结构 AggregatedStatElement

**6.**模板常量参数的名称：遵循变量名称规则，或`N`在简单情况下使用。

模板 <不带 www 的 bool \> struct ExtractDomain

**7.**对于抽象类（接口），您可以添加`I`前缀。

IBlockInputStream 类

**8.**如果在本地使用变量，则可以使用简称。

在所有其他情况下，请使用描述含义的名称。

bool info_successfully_loaded = false ;

**9.**`define` s 的名称和全局常量使用带下划线的 ALL_CAPS。

＃定义 MAX_SRC_TABLE_NAMES_TO_STORE 1000

**10.**文件名应使用与其内容相同的样式。

如果文件包含单个类，请使用与类（CamelCase）相同的方式命名文件。

如果文件包含单个功能，请使用与功能相同的名称来命名文件（camelCase）。

**11.**如果名称包含缩写，则：

- 对于变量名，缩写应使用小写字母`mysql_connection`（而不是`mySQL_connection`）。
- 对于类和函数的名称，请使用大写字母缩写`MySQLConnection`（不是`MySqlConnection`）。

**12.**仅用于初始化类成员的构造函数参数的命名方式应与类成员相同，但末尾应使用下划线。

FileQueueProcessor （
const std :: string ＆ path* ，
const std :: string ＆ prefix* ，
std :: shared*ptr < FileHandler \> handler* ）
： 路径（path* ），
前缀（prefix* ），
处理程序（handler\_ ），
日志（＆Logger :: get （“ FileQueueProcessor” ））
{
}

如果在构造函数主体中未使用该参数，则可以省略下划线后缀。

**13.**局部变量和类成员的名称没有区别（不需要前缀）。

计时器 （不是 m_timer ）

**14.**对于中的常量`enum`，请使用带有大写字母的 CamelCase。ALL_CAPS 也可以接受。如果`enum`非本地，请使用`enum class`。

枚举 类 CompressionMethod
{
QuickLZ = 0 ，
LZ4 = 1 ，
};

**15.**所有名称必须为英文。不允许对俄语单词进行音译。

不是 Stroka

**16.**如果缩写是众所周知的，则可以接受（当您可以在 Wikipedia 或搜索引擎中轻松找到缩写的含义时）。

AST，SQL

不是\`NVDH\`（一些随机字母）

如果通常使用缩写形式，则不完整的单词是可以接受的。

如果注释中包含全名，也可以使用缩写。

**17.**具有 C ++源代码的文件名必须具有`.cpp`扩展名。头文件必须具有`.h`扩展名。

## 如何编写代码[¶](https://clickhouse.yandex/docs/en/development/style/#how-to-write-code "永久链接")

**1.**内存管理。

手动内存释放（`delete`）只能在库代码中使用。

在库代码中，`delete`运算符只能在析构函数中使用。

在应用程序代码中，内存必须由拥有它的对象释放。

例子：

- 最简单的方法是将一个对象放在堆栈上，或使其成为另一个类的成员。
- 对于大量小物体，请使用容器。
- 要自动重新分配驻留在堆中的少量对象，请使用`shared_ptr/unique_ptr`。

**2.**资源管理。

使用`RAII`并参见上文。

**3.**错误处理。

使用例外。在大多数情况下，您只需要抛出一个异常，并且不需要捕获它（因为`RAII`）。

在离线数据处理应用程序中，不捕获异常通常是可以接受的。

在处理用户请求的服务器中，通常足以在连接处理程序的顶层捕获异常。

在线程函数中，您应该捕获并保留所有异常，以将它们重新抛出到主线程之后`join`。

///如果还没有任何计算，
则如果 （！开始）
{
计算（）;
开始 = true ;
}
else ///如果计算已经在进行中，请等待结果
池。等待（）;

if （exception ）
异常-\> rethrow （）;

切勿隐藏未经处理的异常。永远不要盲目地将所有异常记录下来。

//
捕获 不正确（...） {}

如果您需要忽略某些异常，请仅对特定的异常进行处理，然后将其余的重新抛出。

捕捉 （const 的 DB :: 异常 ＆ ë ）
{
如果 （Ë 。代码（） == ErrorCode 的:: UNKNOWN_AGGREGATE_FUNCTION ）
返回 nullptr ;
否则
扔;
}

当使用带有响应代码或的函数时`errno`，请务必检查结果并在出现错误的情况下引发异常。

如果 （0 ！= close （fd ））
throwFromErrno （“无法关闭文件” \+ file_name ， ErrorCodes :: CANNOT_CLOSE_FILE ）;

`Do not use assert`。

**4.**异常类型。

无需在应用程序代码中使用复杂的异常层次结构。异常文本对于系统管理员而言应该是可以理解的。

**5.**析构函数抛出异常。

不建议这样做，但允许这样做。

使用以下选项：

- 创建一个函数（`done()`或`finalize()`）以预先完成可能导致异常的所有工作。如果调用了该函数，则以后的析构函数中不应有任何异常。
- 太复杂的任务（例如，通过网络发送消息）可以放在单独的方法中，类用户必须在销毁之前将其调用。
- 如果析构函数中有异常，最好记录它而不是隐藏它（如果记录器可用）。
- 在简单的应用程序中，可以依靠`std::terminate`（对于`noexcept`C ++ 11 中默认情况下）的异常进行处理。

**6.**匿名代码块。

您可以在单个函数内创建一个单独的代码块，以使某些变量成为局部变量，以便在退出该块时调用析构函数。

块 块 = 数据。在-\> 读（）;

{
std :: lock_guard < std :: 互斥锁\> lock （互斥锁）;
数据。准备 = true ;
数据。块 = 块;
}

ready_any 。设置（）;

**7.**多线程。

在离线数据处理程序中：

- 尝试在单个 CPU 内核上获得最佳性能。然后，可以根据需要并行化代码。

在服务器应用程序中：

- 使用线程池处理请求。在这一点上，我们没有需要用户空间上下文切换的任何任务。

Fork 不用于并行化。

**8.**同步线程。

通常有可能使不同的线程使用不同的存储单元（甚至更好：不同的缓存行），而不使用任何线程同步（除外`joinAll`）。

如果需要同步，在大多数情况下，在下使用互斥体就足够了`lock_guard`。

在其他情况下，请使用系统同步原语。不要使用繁忙的等待。

原子操作仅应在最简单的情况下使用。

除非这是您的主要专长，否则不要尝试实现无锁数据结构。

**9.**指针与引用。

在大多数情况下，请优先参考。

**10.**常量

使用常量引用，指向常量的指针`const_iterator`，和 const 方法。

考虑`const`将其作为默认值，并且`const`仅在必要时才使用非默认值。

当通过值传递变量时，`const`通常没有意义。

**11.**未签名。

`unsigned`必要时使用。

**12.**数值类型。

使用类型`UInt8`，`UInt16`，`UInt32`，`UInt64`，`Int8`，`Int16`，`Int32`，和`Int64`，以及`size_t`，`ssize_t`和`ptrdiff_t`。

不要使用这些类型的数字：`signed/unsigned long`，`long long`，`short`，`signed/unsigned char`，`char`。

**13.**传递参数。

通过引用传递复杂值（包括`std::string`）。

如果函数捕获了在堆中创建的对象的所有权，则将参数类型设置为`shared_ptr`或`unique_ptr`。

**14.**返回值。

在大多数情况下，只需使用即可`return`。不要写`[return std::move(res)]{.strike}`。

如果函数在堆上分配一个对象并返回它，请使用`shared_ptr`或`unique_ptr`。

在极少数情况下，您可能需要通过参数返回值。在这种情况下，该参数应为参考。

使用 AggregateFunctionPtr = std :: shared_ptr < IAggregateFunction \> ;

/ \*\*允许按名称创建聚合函数。 \* /
类 AggregateFunctionFactory
{
public ：
AggregateFunctionFactory （）;
AggregateFunctionPtr get （常量 字符串 和 名称， 常量数据 类型 和 参数类型） 常量;

**15.**名称空间。

无需`namespace`为应用程序代码使用单独的代码。

小型图书馆也不需要这个。

对于中型到大型库，请将所有内容放在中`namespace`。

在库`.h`文件中，您可以`namespace detail`用来隐藏应用程序代码不需要的实现细节。

在`.cpp`文件中，您可以使用`static`或匿名名称空间来隐藏符号。

另外，`namespace`可以将用作，`enum`以防止相应的名称落入外部名称`namespace`（但最好使用`enum class`）。

**16.**延迟初始化。

如果初始化需要参数，那么通常不应该编写默认的构造函数。

如果以后需要延迟初始化，则可以添加将创建无效对象的默认构造函数。或者，对于少量对象，可以使用`shared_ptr/unique_ptr`。

加载程序（DB :: 连接 \* connection* ， const std :: 字符串 和 查询， size_t max_block_size* ）;

///对于延迟的初始化
Loader （） {}

**17.**虚拟功能。

如果该类不适合多态使用，则无需使函数虚拟。这也适用于析构函数。

**18.**编码。

随处使用 UTF-8。使用`std::string`和`char *`。请勿使用`std::wstring`和`wchar_t`。

**19.**记录。

请参见代码中所有示例。

提交之前，请删除所有无意义的调试日志记录以及所有其他类型的调试输出。

即使在跟踪级别，也应避免登录周期。

日志在任何日志记录级别都必须可读。

在大多数情况下，仅应在应用程序代码中使用日志记录。

日志消息必须用英语书写。

日志最好应该是系统管理员可以理解的。

不要在日志中使用亵渎行为。

在日志中使用 UTF-8 编码。在极少数情况下，您可以在日志中使用非 ASCII 字符。

**20.**输入输出。

不要`iostreams`在对应用程序性能至关重要的内部循环中使用（永远不要使用`stringstream`）。

请改用`DB/IO`库。

**21.**日期和时间。

参见`DateLUT`图书馆。

**22.**包括。

始终使用`#pragma once`而不是包括防护。

**23.**使用。

`using namespace`未使用。您可以使用`using`特定的东西。但是将其本地化在类或函数内部。

**24.**`trailing return type`除非必要，否则请勿用于功能。

\[ 自动 ˚F （） \- ＆GT ; 无效;\] {。罢工}

**25.**变量的声明和初始化。

//正确的方式
std :: string s = “ Hello” ;
std :: string s { “ Hello” };

//错误的方式
auto s = std :: string { “ Hello” };

**26.**对于虚函数，请`virtual`在基类中编写，但`override`不要`virtual`在后代类中编写。

## C ++未使用的功能[¶](https://clickhouse.yandex/docs/en/development/style/#unused-features-of-c "永久链接")

**1.**不使用虚拟继承。

**2.**不使用 C ++ 03 的异常说明符。

## 平台[¶](https://clickhouse.yandex/docs/en/development/style/#platform "永久链接")

**1.**我们为特定平台编写代码。

但在其他条件相同的情况下，跨平台或可移植代码是首选。

**2.**语言：C ++ 17。

**3.**编译：`gcc`。目前（2017 年 12 月），该代码使用 7.2 版进行编译。（也可以使用进行编译`clang 4`。）

使用标准库（`libstdc++`或`libc++`）。

**4.**操作系统：Linux Ubuntu，不早于 Precise。

**5.**代码是针对 x86_64 CPU 体系结构编写的。

CPU 指令集是我们服务器中支持的最低设置。当前是 SSE 4.2。

**6.**使用`-Wall -Wextra -Werror`编译标志。

**7.**对所有库使用静态链接，但那些难以静态连接的库除外（请参阅`ldd`命令的输出）。

**8.**使用发行版设置开发和调试代码。

## 工具[¶](https://clickhouse.yandex/docs/en/development/style/#tools "永久链接")

**1.** KDevelop 是一个很好的 IDE。

**2.**对于调试，使用`gdb`，`valgrind`（`memcheck`）， ，`strace`，`-fsanitize=...`或`tcmalloc_minimal_debug`。

**3.**要进行概要分析，请使用`Linux Perf`，`valgrind`（`callgrind`）或`strace -cf`。

**4.**来源在 Git 中。

**5.**组装用途`CMake`。

**6.**使用`deb`软件包发布程序。

**7.**致力于掌握绝不能破坏构建。

尽管只有选定的修订才被认为是可行的。

**8.**即使代码只是部分准备就绪，也应尽可能频繁地提交。

为此使用分支。

如果`master`分支中的代码尚不可构建，请在之前将其从构建中排除`push`。您需要在几天内完成或删除它。

**9.**对于不重要的更改，请使用分支并将其发布在服务器上。

**10.**未使用的代码从存储库中删除。

## 库[¶](https://clickhouse.yandex/docs/en/development/style/#libraries "永久链接")

**1.**使用 C ++ 14 标准库（允许进行实验扩展）以及`boost`和`Poco`框架。

**2.**如有必要，您可以使用 OS 软件包中可用的任何知名库。

如果已经有一个好的解决方案，请使用它，即使这意味着您必须安装另一个库。

（但是要准备从代码中删除错误的库。）

**3.**如果软件包没有所需的软件包，版本过旧或编译类型错误，则可以安装软件包中没有的库。

**4.**如果库很小，并且没有自己的复杂构建系统，则将源文件放在`contrib`文件夹中。

**5.**始终优先使用已使用的库。

## 一般建议[¶](https://clickhouse.yandex/docs/en/development/style/#general-recommendations_1 "永久链接")

**1.**编写尽可能少的代码。

**2.**尝试最简单的解决方案。

**3.**在您知道代码将如何工作以及内部循环如何工作之前，请不要编写代码。

**4.**在最简单的情况下，请使用`using`而不是类或结构。

**5.**如有可能，请勿编写副本构造函数，赋值运算符，析构函数（如果一个类包含至少一个虚函数，则不是虚拟变量），移动构造函数或移动赋值运算符。换句话说，编译器生成的函数必须正确运行。您可以使用`default`。

**6.**鼓励简化代码。尽可能减少代码的大小。

## 其他建议[¶](https://clickhouse.yandex/docs/en/development/style/#additional-recommendations "永久链接")

**1.**明确指定`std::`以下类型`stddef.h`

不推荐。换句话说，我们建议您撰写`size_t`代替`std::size_t`，因为它的短。

可以添加`std::`。

**2.**明确指定`std::`标准 C 库中的函数

不推荐。换句话说，写`memcpy`而不是`std::memcpy`。

原因是存在类似的非标准功能，例如`memmem`。我们有时会使用这些功能。这些功能在中不存在`namespace std`。

如果你写`std::memcpy`的，而不是`memcpy`随处可见，再`memmem`不用`std::`会显得奇怪。

不过，`std::`如果您愿意，仍然可以使用。

**3.**当标准 C ++库中有可用的函数时，请使用 C 中的函数。

如果更有效，这是可以接受的。

例如，使用`memcpy`而不是`std::copy`复制大块内存。

**4.**多行函数参数。

允许使用以下任何包装样式：

功能（
T1 x1 ，
T2 x2 ）

函数（
为 size_t 左， 为 size_t 右，
常量 ＆ RangesInDataParts 范围，
为 size_t 极限）

函数（为 size_t 左， 为 size_t 右，
常量 ＆ RangesInDataParts 范围，
为 size_t 极限）

函数（为 size_t 左， 为 size_t 右，
常量 ＆ RangesInDataParts 范围，
为 size_t 极限）

函数（
为 size_t 左，
为 size_t 右，
常量 ＆ RangesInDataParts 范围，
为 size_t 极限）

# ClickHouse 测试

## 功能测试[¶](https://clickhouse.yandex/docs/en/development/tests/#functional-tests "永久链接")

功能测试是最简单易用的功能。ClickHouse 的大多数功能都可以通过功能测试进行测试，并且必须将它们用于可以以这种方式进行测试的 ClickHouse 代码中的每个更改。

每个功能测试都将一个或多个查询发送到正在运行的 ClickHouse 服务器，并将结果与参考进行比较。

测试位于`dbms/tests/queries`目录中。有两个子目录：`stateless`和`stateful`。无状态测试会在没有任何预加载测试数据的情况下运行查询-它们通常在测试本身内部动态地创建小型合成数据集。有状态测试需要从 Yandex.Metrica 中预加载测试数据，而公众无法使用。我们倾向于只使用`stateless`测试，而避免添加新`stateful`测试。

每个测试可以是以下两种类型之一：`.sql`和`.sh`。`.sql`test 是通过管道传递给的简单 SQL 脚本`clickhouse-client --multiquery --testmode`。`.sh`测试是一个脚本，它自己运行。

要运行所有测试，请使用`dbms/tests/clickhouse-test`工具。查找`--help`可能的选项列表。您可以简单地运行所有测试，也可以运行通过测试名称中的子字符串过滤的测试子集`./clickhouse-test substring`。

调用功能测试的最简单方法是复制`clickhouse-client`到`/usr/bin/`，`clickhouse-server`然后`./clickhouse-test`从其自己的目录运行。

要添加新测试，请在目录中创建一个`.sql`或`.sh`文件，`dbms/tests/queries/0_stateless`手动对其进行检查，然后通过`.reference`以下方式生成文件：`clickhouse-client -n --testmode < 00000_test.sql > 00000_test.reference`或`./00000_test.sh > ./00000_test.reference`。

测试仅应使用（创建，删除等）`test`假定已预先创建的数据库中的表；测试也可以使用临时表。

如果要在功能测试中使用分布式查询，则可以利用`remote`表函数和`127.0.0.{1..2}`服务器的地址来查询自身。或者，您可以在服务器配置文件（如）中使用预定义的测试群集`test_shard_localhost`。

有些测试标有`zookeeper`，`shard`或`long`在他们的名字。`zookeeper`适用于使用 ZooKeeper 的测试。`shard`用于需要服务器监听的测试`127.0.0.*`; `distributed`或`global`具有相同的含义。`long`适用于运行时间稍长一秒的测试。您可以禁用这些群体的使用测试`--no-zookeeper`，`--no-shard`并`--no-long`分别选择。

## 已知错误[¶](https://clickhouse.yandex/docs/en/development/tests/#known-bugs "永久链接")

如果我们知道一些可以通过功能测试轻松复制的错误，则将准备好的功能测试放在`dbms/tests/queries/bugs`目录中。这些测试将在`dbms/tests/queries/0_stateless`修正错误时转移到。

## 集成测试[¶](https://clickhouse.yandex/docs/en/development/tests/#integration-tests "永久链接")

集成测试允许在群集配置中测试 ClickHouse，并与其他服务器（如 MySQL，Postgres，MongoDB）进行 ClickHouse 交互。它们对于模拟网络拆分，数据包丢弃等非常有用。这些测试在 Docker 下运行，并使用各种软件创建多个容器。

查看`dbms/tests/integration/README.md`如何运行这些测试。

请注意，未测试 ClickHouse 与第三方驱动程序的集成。另外，我们目前还没有与 JDBC 和 ODBC 驱动程序进行集成测试。

## 单元测试[¶](https://clickhouse.yandex/docs/en/development/tests/#unit-tests "永久链接")

当您不想测试整个 ClickHouse 而不是单个隔离的库或类时，单元测试很有用。您可以使用`ENABLE_TESTS`CMake 选项启用或禁用测试构建。单元测试（和其他测试程序）位于`tests`代码的子目录中。要运行单元测试，请键入`ninja test`。有些测试使用`gtest`，但是有些只是在测试失败时返回非零退出代码的程序。

如果功能测试已经覆盖了代码，则不一定要进行单元测试（并且功能测试通常更易于使用）。

## 性能测试[¶](https://clickhouse.yandex/docs/en/development/tests/#performance-tests "永久链接")

通过性能测试，可以测量和比较 ClickHouse 在综合查询中某些孤立部分的性能。测试位于`dbms/tests/performance`。每个测试都由`.xml`带有测试用例描述的文件表示。使用`clickhouse performance-test`工具（嵌入在`clickhouse`二进制文件中）运行测试。请参阅参考资料`--help`。

每个测试都会在具有某些停止条件（例如“最大执行速度在三秒钟内不变”）的循环中运行一个或多个查询（可能带有参数组合），并测量一些有关查询性能的指标（例如“最大执行速度” ）。某些测试可能包含预加载的测试数据集中的前提条件。

如果要在某些情况下提高 ClickHouse 的性能，并且可以在简单查询中观察到改进，则强烈建议编写性能测试。它总是有意义的使用`perf top`过程中你的测试或其他 PERF 工具。

## 测试工具和脚本[¶](https://clickhouse.yandex/docs/en/development/tests/#test-tools-and-scripts "永久链接")

在某些程序`tests`的目录不准备测试，但测试工具。例如，对于`Lexer`有一个工具`dbms/src/Parsers/tests/lexer`，它只对 stdin 进行标记化，然后将彩色结果写入 stdout。您可以将这类工具用作代码示例以及进行探索和手动测试。

您还可以放置文件对`.sh`和`.reference`工具，以在某些预定义的输入上运行它-然后可以将脚本结果与`.reference`文件进行比较。这些测试不是自动化的。

## 计算其它测试[¶](https://clickhouse.yandex/docs/en/development/tests/#miscellanous-tests "永久链接")

在中有针对外部词典的测试，`dbms/tests/external_dictionaries`以及针对中的机器学习模型的测试`dbms/tests/external_models`。这些测试不会更新，必须转移到集成测试中。

仲裁插入有单独的测试。在单独的服务器此测试运行 ClickHouse 簇和仿真各种故障情况：网络分割，分组丢弃（ClickHouse 节点之间，ClickHouse 和动物园管理员之间，ClickHouse 服务器和客户机之间等）， `kill -9`，`kill -STOP`和`kill -CONT`，像[Jepsen 的](https://aphyr.com/tags/Jepsen)。然后测试检查是否写入了所有确认的插入，是否写入了所有拒绝的插入。

Quorum 测试是由 ClickHouse 开源之前由单独的团队编写的。该团队不再与 ClickHouse 合作。测试是偶然用 Java 编写的。由于这些原因，必须重写仲裁测试并将其移至集成测试。

## 手动测试[¶](https://clickhouse.yandex/docs/en/development/tests/#manual-testing "永久链接")

开发新功能时，也可以手动进行测试。您可以按照以下步骤操作：

生成 ClickHouse。在终端上运行 ClickHouse：将目录更改为`dbms/src/programs/clickhouse-server`并使用运行它`./clickhouse-server`。它将使用配置（`config.xml`，`users.xml`并在文件`config.d`和`users.d`默认目录）从当前目录。要连接到 ClickHouse 服务器，请运行`dbms/src/programs/clickhouse-client/clickhouse-client`。

请注意，所有 Clickhouse 工具（服务器，客户端等）都只是指向名为的单个二进制文件的符号链接`clickhouse`。您可以在找到此二进制文件`dbms/src/programs/clickhouse`。也可以使用`clickhouse tool`代替调用所有工具`clickhouse-tool`。

或者，您可以安装 ClickHouse 软件包：可以从 Yandex 存储库中稳定发行，也可以`./release`在 ClickHouse 源码根目录中自行构建软件包。然后使用启动服务器`sudo service clickhouse-server start`（或停止以停止服务器）。在查找日志`/etc/clickhouse-server/clickhouse-server.log`。

当您的系统上已经安装 ClickHouse 时，您可以构建一个新的`clickhouse`二进制文件并替换现有的二进制文件：

$ sudo服务clickhouse-server停止
$ sudo cp ./clickhouse / usr / bin /
\$ sudo 服务 clickhouse-server 启动

您也可以停止系统 clickhouse-server 并以相同的配置运行自己的服务器，但登录到终端：

$ sudo服务clickhouse-server停止
$ sudo -u clickhouse / usr / bin / clickhouse 服务器--config-file /etc/clickhouse-server/config.xml

gdb 的示例：

\$ sudo -u clickhouse gdb --args / usr / bin / clickhouse 服务器--config-file /etc/clickhouse-server/config.xml

如果系统 clickhouse-server 已经在运行，并且您不想停止它，则可以更改其中的端口号`config.xml`（或在`config.d`目录中的文件中覆盖它们），提供适当的数据路径并运行它。

`clickhouse`二进制文件几乎没有依赖性，可以在各种 Linux 发行版中使用。为了快速而肮脏地测试服务器上的更改，您可以`scp`将新构建的`clickhouse`二进制文件简单地存储到服务器上，然后按照上面的示例运行它。

## 测试环境[¶](https://clickhouse.yandex/docs/en/development/tests/#testing-environment "永久链接")

在发布稳定版本之前，我们将其部署在测试环境中。测试环境是一个集群，处理[Yandex.Metrica](https://metrica.yandex.com/)数据的 1/39 部分。我们与 Yandex.Metrica 团队共享我们的测试环境。ClickHouse 升级后，不会在现有数据之上停机。首先，我们看到数据已成功处理而没有滞后，复制继续进行，并且 Yandex.Metrica 团队看不到任何问题。可以通过以下方式进行第一次检查：

SELECT hostName （） AS h ， any （version （））， any （uptime （））， max （UTCEventTime ）， count （） FROM remote （'example01-01- {1..3} t' ， merge ， hits ） WHERE EVENTDATE \> = 今天（） \- 2 GROUP BY ħ ORDER BY ħ ;

在某些情况下，我们还会部署到 Yandex 的朋友团队的测试环境：市场，云等。此外，我们还有一些用于开发目的的硬件服务器。

## 负载测试[¶](https://clickhouse.yandex/docs/en/development/tests/#load-testing "永久链接")

部署到测试环境后，我们使用来自生产集群的查询运行负载测试。这是手动完成的。

确保已`query_log`在生产集群上启用。

收集一天或以上的查询日志：

\$ clickhouse-client --query = “从 system.query_log 中选择不同的查询，其中 event_date = today（）并查询类似'％ym：％'并且查询不像'％system.query_log％'并且 type = 2 AND is_initial_query” \> querys.tsv

这是一个复杂的例子。`type = 2`将过滤成功执行的查询。`query LIKE '%ym:%'`是要从 Yandex.Metrica 中选择相关查询。`is_initial_query`是仅选择由客户端而不是 ClickHouse 本身（作为分布式查询处理的一部分）启动的查询。

`scp`  将此日志记录到您的测试集群并按以下方式运行：

\$ clickhouse 基准-并发 16 <
query.tsv

（可能您还想指定一个`--user`）

然后将其放置一个晚上或周末，然后休息一下。

您应该检查`clickhouse-server`它不会崩溃，内存占用空间有限以及性能不会随着时间的推移而降低。

由于查询和环境的高度可变性，因此没有记录精确的查询执行时间，也没有进行比较。

## 构建测试[¶](https://clickhouse.yandex/docs/en/development/tests/#build-tests "永久链接")

构建测试允许检查构建是否在各种替代配置和某些外部系统上没有损坏。测试位于`ci`目录中。他们从 Docker，Vagrant 内部的源代码运行构建，有时`qemu-user-static`在 Docker 内部运行。这些测试正在开发中，并且测试运行不是自动化的。

动机：

通常，我们在 ClickHouse 版本的单个变体上发布并运行所有测试。但是，还有一些其他构建变体尚未经过全面测试。例子：

- 建立在 FreeBSD 上；
- 使用系统包中的库在 Debian 上构建；
- 使用库的共享链接进行构建；
- 建立在 AArch64 平台上；
- 在 PowerPc 平台上构建。

例如，使用系统软件包进行构建是一种不好的做法，因为我们不能保证系统将拥有确切的软件包版本。但这确实是 Debian 维护者所需要的。因此，我们至少必须支持这种构建版本。另一个例子：共享链接是常见的麻烦根源，但是对于某些发烧友来说是必需的。

尽管我们无法在所有版本的构建版本上运行所有测试，但我们至少要检查各种构建版本是否未损坏。为此，我们使用构建测试。

## 测试协议兼容性[¶](https://clickhouse.yandex/docs/en/development/tests/#testing-for-protocol-compatibility "永久链接")

扩展 ClickHouse 网络协议时，我们会手动测试旧的 clickhouse-client 与新的 clickhouse-server 一起工作，而新的 clickhouse-client 与旧的 clickhouse-server 一起工作（仅通过运行相应程序包中的二进制文件）。

## 帮助从编译器[¶](https://clickhouse.yandex/docs/en/development/tests/#help-from-the-compiler "永久链接")

ClickHouse 主代码（位于`dbms`目录中）与`-Wall -Wextra -Werror`以及一些其他启用的警告一起构建。尽管未为第三方库启用这些选项。

Clang 甚至有更多有用的警告-您可以使用它们来查找它们，`-Weverything`并选择一些内容进行默认构建。

对于生产版本，将使用 gcc（它仍然比 clang 产生效率更高的代码）。对于开发而言，使用 clang 通常更方便。您可以使用调试模式在自己的计算机上构建（以节省笔记本电脑的电量），但请注意，`-O3`由于更好的控制流和过程间分析，编译器能够生成更多警告。当使用 clang 进行构建时，`libc++`而不是使用 clang 进行构建；`libstdc++`当使用调试模式进行构建时，将使用的调试版本，`libc++`该版本允许在运行时捕获更多错误。

## 消毒剂[¶](https://clickhouse.yandex/docs/en/development/tests/#sanitizers "永久链接")

**地址消毒剂**。我们基于每次提交在 ASan 下运行功能和集成测试。

**Valgrind（Memcheck）**。我们在一夜之间在 Valgrind 下运行功能测试。这需要几个小时。当前库中存在一种已知的误报`re2`，请参阅[本文](https://research.swtch.com/sparse)。

**未定义的行为消毒剂。**我们基于每次提交在 ASan 下运行功能和集成测试。

**线程消毒剂**。我们根据每次提交在 TSan 下运行功能测试。我们仍然不按每次提交在 TSan 下运行集成测试。

**内存消毒剂**。目前，我们仍然不使用 MSan。

**调试分配器。**的 Debug 版本`jemalloc`用于调试版本。

## 起毛[¶](https://clickhouse.yandex/docs/en/development/tests/#fuzzing "永久链接")

我们使用简单的模糊测试来生成随机 SQL 查询并检查服务器是否不会死机。使用 Address sanitizer 执行模糊测试。您可以在中找到它`00746_sql_fuzzy.pl`。该测试应连续运行（隔夜或更长时间）。

截至 2018 年 12 月，我们仍未使用库代码的隔离模糊测试。

## 安全审计[¶](https://clickhouse.yandex/docs/en/development/tests/#security-audit "永久链接")

Yandex Cloud 部门的人员从安全角度对 ClickHouse 功能进行了一些基本概述。

## 静态分析仪[¶](https://clickhouse.yandex/docs/en/development/tests/#static-analyzers "永久链接")

我们`PVS-Studio`基于每次提交运行。我们已经评估`clang-tidy`，`Coverity`，`cppcheck`，`PVS-Studio`，`tscancode`。您将在`dbms/tests/instructions/`目录中找到使用说明。您也可以阅读[俄语文章](https://habr.com/company/yandex/blog/342018/)。

如果`CLion`用作 IDE，则可以利用一些`clang-tidy`开箱即用的功能。

## 强化[¶](https://clickhouse.yandex/docs/en/development/tests/#hardening "永久链接")

`FORTIFY_SOURCE`默认情况下使用。它几乎没有用，但在极少数情况下仍然有意义，我们不会禁用它。

## 代码样式[¶](https://clickhouse.yandex/docs/en/development/tests/#code-style "永久链接")

[这里](https://clickhouse.yandex/docs/en/development/style/)描述[了](https://clickhouse.yandex/docs/en/development/style/)代码样式规则。

要检查一些常见的样式冲突，可以使用`utils/check-style`脚本。

要强制使用正确的代码样式，可以使用`clang-format`。文件`.clang-format`位于源根目录。它主要与我们的实际代码样式相对应。但不建议将其应用于`clang-format`现有文件，因为它会使格式化变得更糟。您可以使用`clang-format-diff`在 clang 源存储库中找到的工具。

另外，您可以尝试使用`uncrustify`工具来重新格式化代码。配置位于`uncrustify.cfg`源根目录中。它没有经过测试`clang-format`。

`CLion`  有自己的代码格式化程序，必须针对我们的代码样式进行调整。

## Metrica B2B 测试[¶](https://clickhouse.yandex/docs/en/development/tests/#metrica-b2b-tests "永久链接")

每个 ClickHouse 版本都经过 Yandex Metrica 和 AppMetrica 引擎的测试。ClickHouse 的测试版本和稳定版本已部署在 VM 上，并与一小份 Metrica 引擎一起运行，该引擎正在处理输入数据的固定样本。然后将两个 Metrica 引擎实例的结果一起比较。

这些测试由独立的团队自动化。由于活动部件数量众多，大多数情况下，由于完全不相关的原因导致测试失败，这很难弄清。这些测试很可能对我们不利。然而，这些试验证明在约一或两次有用了几百。

## 测试覆盖率[¶](https://clickhouse.yandex/docs/en/development/tests/#test-coverage "永久链接")

截至 2018 年 7 月，我们不再跟踪测试范围。

## 自动化测试[¶](https://clickhouse.yandex/docs/en/development/tests/#test-automation "永久链接")

我们使用 Yandex 内部 CI 和名为“ Sandbox”的作业自动化系统进行测试。

构建作业和测试在每次提交的基础上在沙盒中运行。结果包和测试结果在 GitHub 上发布，可以通过直接链接下载。工件被永久存储。当您在 GitHub 上发送拉取请求时，我们将其标记为“可以测试”，并且我们的 CI 系统将为您构建 ClickHouse 程序包（发布，调试和地址清理器等）。

由于时间和计算能力的限制，我们不使用 Travis CI。我们不使用詹金斯。它曾经被使用过，现在我们很高兴我们没有使用詹金斯。

# 使用的第三方库

图书馆

执照

base64

[BSD 2 条款许可](https://github.com/aklomp/base64/blob/a27c565d1b6c676beaf297fe503c4518185666f7/LICENSE)

促进

[Boost 软件许可 1.0](https://github.com/ClickHouse-Extras/boost-extra/blob/6883b40449f378019aec792f9983ce3afc7ff16e/LICENSE_1_0.txt)

布罗特利

[麻省理工学院](https://github.com/google/brotli/blob/master/LICENSE)

Capnproto

[麻省理工学院](https://github.com/capnproto/capnproto/blob/master/LICENSE)

cctz

[Apache 许可 2.0](https://github.com/google/cctz/blob/4f9776a310f4952454636363def82c2bf6641d5f/LICENSE.txt)

双重转换

[BSD 3 条款许可](https://github.com/google/double-conversion/blob/cf2f0f3d547dc73b4612028a155b80536902ba02/LICENSE)

快速记忆

[麻省理工学院](https://github.com/yandex/ClickHouse/blob/master/libs/libmemcpy/impl/LICENSE)

谷歌测试

[BSD 3 条款许可](https://github.com/google/googletest/blob/master/LICENSE)

超扫描

[BSD 3 条款许可](https://github.com/intel/hyperscan/blob/master/LICENSE)

libbtrie

[BSD 2 条款许可](https://github.com/yandex/ClickHouse/blob/master/contrib/libbtrie/LICENSE)

libcxxabi

[BSD +麻省理工学院](https://github.com/yandex/ClickHouse/blob/master/libs/libglibc-compatibility/libcxxabi/LICENSE.TXT)

libdivide

[Zlib 许可证](https://github.com/yandex/ClickHouse/blob/master/contrib/libdivide/LICENSE.txt)

libgsasl

[LGPL v2.1](https://github.com/ClickHouse-Extras/libgsasl/blob/3b8948a4042e34fb00b4fb987535dc9e02e39040/LICENSE)

libhdfs3

[Apache 许可 2.0](https://github.com/ClickHouse-Extras/libhdfs3/blob/bd6505cbb0c130b0db695305b9a38546fa880e5a/LICENSE.txt)

libmetrohash

[Apache 许可 2.0](https://github.com/yandex/ClickHouse/blob/master/contrib/libmetrohash/LICENSE)

libpcg 随机

[Apache 许可 2.0](https://github.com/yandex/ClickHouse/blob/master/contrib/libpcg-random/LICENSE-APACHE.txt)

libressl

[OpenSSL 许可](https://github.com/ClickHouse-Extras/ssl/blob/master/COPYING)

librdkafka

[BSD 2 条款许可](https://github.com/edenhill/librdkafka/blob/363dcad5a23dc29381cc626620e68ae418b3af19/LICENSE)

libwidechar_width

[CC0 1.0 通用](https://github.com/yandex/ClickHouse/blob/master/libs/libwidechar_width/LICENSE)

llvm

[BSD 3 条款许可](https://github.com/ClickHouse-Extras/llvm/blob/163def217817c90fb982a6daf384744d8472b92b/llvm/LICENSE.TXT)

lz4

[BSD 2 条款许可](https://github.com/lz4/lz4/blob/c10863b98e1503af90616ae99725ecd120265dfb/LICENSE)

mariadb-connector-c

[LGPL v2.1](https://github.com/ClickHouse-Extras/mariadb-connector-c/blob/3.1/COPYING.LIB)

杂音

[公共区域](https://github.com/yandex/ClickHouse/blob/master/contrib/murmurhash/LICENSE)

pdqsort

[Zlib 许可证](https://github.com/yandex/ClickHouse/blob/master/contrib/pdqsort/license.txt)

Poco

[Boost 软件许可-版本 1.0](https://github.com/ClickHouse-Extras/poco/blob/fe5505e56c27b6ecb0dcbc40c49dc2caf4e9637f/LICENSE)

原虫

[BSD 3 条款许可](https://github.com/ClickHouse-Extras/protobuf/blob/12735370922a35f03999afff478e1c6d7aa917a4/LICENSE)

re2

[BSD 3 条款许可](https://github.com/google/re2/blob/7cf8b88e8f70f97fd4926b56aa87e7f53b2717e0/LICENSE)

UnixODBC

[LGPL v2.1](https://github.com/ClickHouse-Extras/UnixODBC/tree/b0ad30f7f6289c12b76f04bfb9d466374bb32168)

lib

[Zlib 许可证](https://github.com/ClickHouse-Extras/zlib-ng/blob/develop/LICENSE.md)

标准

[BSD 3 条款许可](https://github.com/facebook/zstd/blob/dev/LICENSE)


