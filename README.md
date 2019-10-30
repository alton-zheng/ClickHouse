# ClickHouse

## 文档
- [官方文档](https://clickhouse.yandex/docs/en/)
- [最快开源 OLAP 引擎！ ClickHouse 在头条的技术演进](https://www.v2ex.com/t/580396)
- [《大数据实时分析领域的黑马ClickHouse》二次解读](https://blog.csdn.net/haitianxueyuan521/article/details/80983001)

- 实践案例：
- [使用ClickHouse对每秒6百万次请求进行HTTP分析](http://fashengba.com/post/http-analytics-for-6m-requests-per-second-using-clickhouse.html)

## 其它数据库
- [FusionDB](https://www.fusionlab.cn/zh-cn/fdb/index.html)

## 性能比对
- 新浪： 
  - 300亿数据
  - count  0.9s
  - 时间 group by count/100000000  limit 10  9s

## 优劣势
clickhouse
- \1. 不支持事务
- \2. 不支持update/Delete操作
- \3. 支持有限操作系统
- \4.  ClickHouse 比 Vertica InfiniDB MonetDB Infobright Hive MySQL MemSQL Greenplum 大部分场景要快，个别场景比某些差点
