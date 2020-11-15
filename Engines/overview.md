# CH 中的表引擎
表引擎（即表的类型）决定了：
- 数据的存储方式和位置，写入以及读取位置
- 支持哪些查询，以及如何支持；
- 并发数据访问；
- 使用索引（如果存在）；
- 是否多线程执行请求；
- 数据复制的参数。

Engine Families

### Merge Tree
高负载任务-最健壮&功能最强大的表引擎们。

特性：快速的数据插入&后台数据处理；支持数据复制（带Replicated*的引擎）、分区、secondary data-skipping indexes。

家族：
- [MergeTree](MergeTree.md)
- [ReplicatedMergeTree](ReplicatedMergeTree.md)
- SummingMergeTree
- AggregatingMergeTree
- CollapsingMergeTree
- VersionedCollapsingMergeTree
- GraphiteMergeTree

### Log 引擎
最少功能的轻量级引擎。最有效的场景：快速写多个小表（不超过100W行）并整体查询。

家族：
- TinyLog
- StripeLog
- Log

### 集成引擎
用于与其他数据系统通讯：

家族：
- Kafka
- MySQL
- JDBC
- ODBC
- HDFS

### 特殊引擎
家族：
- Distributed
- MaterializedView
- Dictionary
- Merge
- File
- Null
- Set
- Join
- URL
- View
- Memory
- Buffer

### 虚拟列
在 engine 源码中定义的列，虚拟列是只读的，无法插入数据到虚拟列中；`SHOW CREATE TABLE` 和 `DESCRIBE TABLE` 不会显示虚拟列，必须显示使用列名查询。

如果创建了和虚拟列同名的列，虚拟列失效；不建议这么做。
