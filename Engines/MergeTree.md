# MergeTree 表引擎
MergeTree家族的引擎们设计的主旨是将大量数据快速插入表中。数据被快速写入table每个part，后台按照一定规则合并这些part，这种方法比连续地重写数据存储有效得多。

主要特性：
- 通过primary key(未指定用order key)排序存储的数据；
    
    创建小的稀疏索引来快速定位数据
- 可选 partitioning key 实现分区
    
    使用分区键划分数据，分区表在某些特定操作下比普通表更有效。
- 数据采样
    
    如有必要，可以在表中设置数据采样方法。
    
    
#### 建标语句
```
CREATE TABLE [IF NOT EXISTS] table_name [ON CLUSTER cluster_name]
(
    col1 type1 [DEFAULT|METERALIZED|ALIAS expr1] [TTL expr1],
    col2 type2 [DEFAULT|METERALIZED|ALIAS expr2] [TTL expr2],
    ...
    INDEX index_name1 expr1 TYPE type1(...) GRANULARITY value1,
    INDEX index_name2 expr2 TYPE type2(...) GRANULARITY value2
) ENGINE = MergeTree()
ORDER BY expr
[PARTITION BY expr]
[PRIMARY KEY expr]
[SAMPLE BY expr]
[TTL expr [DELETE|TO DISK 'xxx'|TO VOLUME 'xxx'], ...]
[SETTINGS name=value, ...]
```
说明：
- ENGINE=MergeTree() 没有参数
- ORDER BY 必须指定，默认主键，排序键
- SAMPLE BY : 指定 sample key 则主键必须包含采样表达式，例如 `SAMPLE BY intHash32(UserID) ORDER BY (CounterID, EventDate, intHash32(UserID))`
- TTL: 必须有 date 或 datetime 列，例如 `TTL date + INTERVAL 1 DAY`
- SETTINGS: 控制MergeTree行为的参数：`index_granularity`索引间最大行数(默认8192)，`index_granularity_bytes`索引间最大字节数(默认10Mb)，