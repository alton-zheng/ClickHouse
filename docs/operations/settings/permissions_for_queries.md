# Permissions for queries

ClickHouse 中的查询可以分为几种类型：

1.  读：`SELECT`，`SHOW`，`DESCRIBE`，`EXISTS`。
2.  写：`INSERT`，`OPTIMIZE`。
3.  更改设置：`SET`，`USE`。
4.  [DDL](https://en.wikipedia.org/wiki/Data_definition_language)查询：`CREATE`，`ALTER`，`RENAME`，`ATTACH`，`DETACH`，`DROP` `TRUNCATE`。
5.  `KILL QUERY`。

以下设置通过查询类型来调节用户权限：

- `readonly` : 限制对所有类型的查询（DDL 查询除外）的权限。
- `allow_ddl`: 限制 DDL 查询的权限。

`KILL QUERY`  可以使用任何设置执行。

## readonly

限制读取数据，写入数据和更改设置查询的权限。

可能的值：

- `0`: 允许所有查询。
- `1`: 仅允许读取数据查询。
- `2`: 允许读取数据和更改设置查询。

设置`readonly = 1` 后，用户无法在当前会话中更改`readonly`和`allow_ddl`设置。

在 `HTTP interface`使用 `GET` 方法时，`readonly = 1` 将自动设置。若要修改数据，请使用`POST`方法。

设置 `readonly = 1` 禁止用户更改所有设置。有一种方法可以禁止用户仅更改特定设置，有关详细信息，请参阅[constraints_on_settings](constraints_on_settings.md)。

默认值：0

## allow_ddl

允许或拒绝[DDL](https://en.wikipedia.org/wiki/Data_definition_language)查询。

可能的值：

- `0`: 不允许 `DDL` 查询。
- `1`: 允许 `DDL` 查询。

如果 `allow_ddl = 0` 当前会话无法执行 `SET allow_ddl = 1`。

默认值：1