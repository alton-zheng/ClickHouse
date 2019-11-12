## HDFS[¶](https://clickhouse.yandex/docs/zh/single/#table_engines-hdfs "Permanent link")

This engine provides integration with [Apache Hadoop](https://en.wikipedia.org/wiki/Apache_Hadoop) ecosystem by allowing to manage data on [HDFS](https://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-hdfs/HdfsDesign.htmll)via ClickHouse. This engine is similar to the [File](https://clickhouse.yandex/docs/zh/single/#file/) and [URL](https://clickhouse.yandex/docs/zh/single/#url/) engines, but provides Hadoop-specific features.

### Usage[¶](https://clickhouse.yandex/docs/zh/single/#usage "Permanent link")

ENGINE = HDFS(URI, format)

The `URI` parameter is the whole file URI in HDFS. The `format` parameter specifies one of the available file formats. To perform `SELECT` queries, the format must be supported for input, and to perform `INSERT` queries -- for output. The available formats are listed in the [Formats](https://clickhouse.yandex/docs/zh/single/#formats) section. The path part of `URI` may contain globs. In this case the table would be readonly.

**Example:**

**1.** Set up the `hdfs_engine_table` table:

CREATE TABLE hdfs_engine_table (name String, value UInt32) ENGINE=HDFS('hdfs://hdfs1:9000/other_storage', 'TSV')

**2.** Fill file:

INSERT INTO hdfs_engine_table VALUES ('one', 1), ('two', 2), ('three', 3)

**3.** Query the data:

SELECT \* FROM hdfs_engine_table LIMIT 2

┌─name─┬─value─┐
│ one │ 1 │
│ two │ 2 │
└──────┴───────┘

### Implementation Details[¶](https://clickhouse.yandex/docs/zh/single/#implementation-details "Permanent link")

- Reads and writes can be parallel
- Not supported:
  - `ALTER` and `SELECT...SAMPLE` operations.
  - Indexes.
  - Replication.

**Globs in path**

Multiple path components can have globs. For being processed file should exists and matches to the whole path pattern. Listing of files determines during `SELECT` (not at `CREATE` moment).

- `*` — Substitutes any number of any characters except `/` including empty string.
- `?` — Substitutes any single character.
- `{some_string,another_string,yet_another_one}` — Substitutes any of strings `'some_string', 'another_string', 'yet_another_one'`.
- `{N..M}` — Substitutes any number in range from N to M including both borders.

Constructions with `{}` are similar to the [remote](https://clickhouse.yandex/docs/zh/single/#../../query_language/table_functions/remote/) table function.

**Example**

1.  Suppose we have several files in TSV format with the following URIs on HDFS:

2.  'hdfs://hdfs1:9000/some_dir/some_file_1'

3.  'hdfs://hdfs1:9000/some_dir/some_file_2'
4.  'hdfs://hdfs1:9000/some_dir/some_file_3'
5.  'hdfs://hdfs1:9000/another_dir/some_file_1'
6.  'hdfs://hdfs1:9000/another_dir/some_file_2'
7.  'hdfs://hdfs1:9000/another_dir/some_file_3'

8.  There are several ways to make a table consisting of all six files:

CREATE TABLE table_with_range (name String, value UInt32) ENGINE = HDFS('hdfs://hdfs1:9000/{some,another}\_dir/some_file\_{1..3}', 'TSV')

Another way:

CREATE TABLE table_with_question*mark (name String, value UInt32) ENGINE = HDFS('hdfs://hdfs1:9000/{some,another}\_dir/some_file*?', 'TSV')

Table consists of all the files in both directories (all files should satisfy format and schema described in query):

CREATE TABLE table_with_asterisk (name String, value UInt32) ENGINE = HDFS('hdfs://hdfs1:9000/{some,another}\_dir/\*', 'TSV')

Warning

If the listing of files contains number ranges with leading zeros, use the construction with braces for each digit separately or use `?`.

**Example**

Create table with files named `file000`, `file001`, ... , `file999`:

CREARE TABLE big_table (name String, value UInt32) ENGINE = HDFS('hdfs://hdfs1:9000/big_dir/file{0..9}{0..9}{0..9}', 'CSV')