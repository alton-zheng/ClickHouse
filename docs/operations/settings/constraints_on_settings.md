# Constraints on Settings

设置约束可以在`user.xml`配置文件的 `users` 部分中定义，并禁止用户通过`SET`查询更改某些设置。约束定义如下：

```xml
<profiles>
  <user_name>
    <constraints>
      <setting_name_1>
        <min>lower_boundary</min>
      </setting_name_1>
      <setting_name_2>
        <max>upper_boundary</max>
      </setting_name_2>
      <setting_name_3>
        <min>lower_boundary</min>
        <max>upper_boundary</max>
      </setting_name_3>
      <setting_name_4>
        <readonly/>
      </setting_name_4>
    </constraints>
  </user_name>
</profiles>
```

如果用户试图违反约束，则会引发异常，并且设置实际上不会更改。目前支持三种类型的约束：`min`，`max`，`readonly`。的`min`和`max`约束指定上限和下限为数值设置，并且可以组合使用。该`readonly`约束指定用户根本无法更改相应的设置。
如果用户试图违反约束，则抛出异常，并且设置实际上没有更改。支持三种类型的约束:`min`、`max`、`readonly`。`min` 和 `max` 约束指定数值设置的上限和下限，可以组合使用。`readonly` 约束指定用户根本不能更改相应的设置。

```xml
<profiles>
  <default>
    <max_memory_usage>10000000000</max_memory_usage>
    <force_index_by_date>0</force_index_by_date>
    ...
    <constraints>
      <max_memory_usage>
        <min>5000000000</min>
        <max>20000000000</max>
      </max_memory_usage>
      <force_index_by_date>
        <readonly/>
      </force_index_by_date>
    </constraints>
  </default>
</profiles>
```

以下查询均引发异常：

```bash
SET max_memory_usage=20000000001;
SET max_memory_usage=4999999999;
SET force_index_by_date=1;
```

```
Code: 452, e.displayText() = DB::Exception: Setting max_memory_usage should not be greater than 20000000000.
Code: 452, e.displayText() = DB::Exception: Setting max_memory_usage should not be less than 5000000000.
Code: 452, e.displayText() = DB::Exception: Setting force_index_by_date should not be changed.
```

**注：**:
- `default` 配置文件有一个特殊的处理:
  - 为`default` 配置文件定义的所有约束都成为默认约束，因此它们限制所有用户，直到它们显式地覆盖这些用户。