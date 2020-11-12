# MergeTree 表引擎
MergeTree家族的引擎们设计的主旨是将大量数据快速插入表中。数据被快速写入table每个part，后台按照一定规则合并这些part，这种方法比连续地重写数据存储有效得多。

主要特性：
- 通过primary key(未指定用order key)排序存储的数据；
    
    创建小的稀疏索引来快速定位数据
- 可选 partitioning key 实现分区
    
    使用分区键划分数据，分区表在某些特定操作下比普通表更有效。
- 数据采样
    
    如有必要，可以在表中设置数据采样方法。
    
    
#### 建表
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
- SETTINGS: 控制MergeTree行为的参数：
    - `index_granularity`索引间最大行数(默认8192)
    - `index_granularity_bytes`索引间最大字节数(默认10Mb)
    - `min_merge_bytes_to_use_direct_io` merge操作可使用直接IO(O_DIRECT操作)访问存储磁盘的最小数据量，默认 10Gb 
    - `write_final_mark` 允许在part最后一个字节之后写入索引标记, 默认：1，不要关闭
    - `merge_max_block_size` 合并的最大块的行数，默认：8192（跟`index_granularity`默认值相同）
    - `min_bytes_for_wide_part`, `min_rows_for_wide_part`，wide格式的最小part字节数/行数，默认none
   
   示例:
   
   `ENGINE MergeTree() PARTITION BY toYYYYMM(EventDate) ORDER BY (CounterID, EventDate, intHash32(UserID)) SAMPLE BY intHash32(UserID) SETTINGS index_granularity=8192`
    
    说明：1. 分区：使用`toYYYYMM`达到按月分区的目的，2. 采样：对 UserID 做 hash 作为采样，相当于对 CounterID 和 EventDate 作了伪随机，CH 会 平均伪随机 的返回 User 子集的样本。 3. `index_granularity`默认就是8192，此处可省略。
    
    
#### 数据存储
一个表由 <em>按主键排序好</em> 的 <em>data parts</em> 组成。

- data part：插入数据时会创建独立的 data part，每个 data part 按主键以字典形式排序。以 `(CounterID, Date)` 主键为例，part中的数据先按照 `CounterID` 排序，`CounterId`相同的按`Date`排序。

- 分区和part：不同 partition 的数据会被划分到不同的 data part（data part就是数据块），后台merge不会将不同分区的数据合并到同一个part，因此merge机制不能保证同一主键的数据一定在一个part中。
    
- 存储格式：data part 有`Wide`和`Compact`两种存储格式。`Wide`格式每一列存储在文件系统一个单独文件，`Compact`所有列存储在同一文件中。`Compact`格式适用于小的频繁的插入场景。

    存储格式由`min_bytes_for_wide_part`和`min_rows_for_wide_part`控制，如果 data part 比预设值小就是`Compact`格式，否则就是`Wide`格式，没有设置就是 `Wide`格式。
- Marks: 每个part逻辑上划分为颗粒，每个颗粒是CH读取数据时的最小不可分割数据集，即一次性读取一整个颗粒的数据。

    每个颗粒的第一行会创建基于主键值的标记（mark），每个part会创建一个索引文件存储 marks，如下面的`mr_test`表的 `202011`分区下有一个`primary.idx`文件。
    
    `index_granularity`和`index_granularity_bytes`设置表引擎的索引粒度，但是一行不会被拆分到两个part，即行大小超过索引粒度字节数限制，该行会整体存放到当前part；因此在这种情况下 颗粒的大小=行数。
 
 ```
  Whole data:     [---------------------------------------------]
  CounterID:      [aaaaaaaaaaaaaaaaaabbbbcdeeeeeeeeeeeeefgggggggghhhhhhhhhiiiiiiiiikllllllll]
  Date:           [1111111222222233331233211111222222333211111112122222223111112223311122333]
  Marks:           |      |      |      |      |      |      |      |      |      |      |
                  a,1    a,2    a,3    b,3    e,2    e,3    g,1    h,2    i,1    i,3    l,3
  Marks numbers:   0      1      2      3      4      5      6      7      8      9      10
```
- `CounterID in ('a','h')`-服务器会读取标记 [0,3) 和 [6,8) 范围的数据；`CounterID in ('a','h') and Date=3` - 读取标记 [1,3) 和 [7,8) 范围的数据；`Date=3` - 读取标记[1,10]范围的数据
- 稀疏索引允许读取多余的行，当读取单个范围的主键时，每个数据块中最多可以读取`index_granularity * 2`个额外的行。
- 稀疏索引可以处理大量的行，因为大多数这种索引都跟计算机的 RAM 是匹配的。
- CH中不要求唯一，主键可以重复。

一个数据存储测试：
```
$ docker exec -it testck_clickhouse01_1 clickhouse-client -m
>
CREATE TABLE mr_test ON CLUSTER 'perftest_3shards_1replicas'
(
    CounterID int,
    Date Datetime
) ENGINE = MergeTree()
ORDER BY (CounterID,Date)
PARTITION BY toYYYYMM(Date)
SETTINGS index_granularity=8192;
> 
INSERT INTO mr_test VALUES(1,'2020-11-01 00:00:00'),(2,'2020-11-02 00:00:00'),
                          (3,'2020-11-03 00:00:00'),(4,'2020-11-04 00:00:00'),
                          (5,'2020-11-05 00:00:00'),(6,'2020-11-06 00:00:00'),
                          (7,'2020-11-07 00:00:00'),(8,'2020-11-08 00:00:00'), 
                          (9,'2020-11-09 00:00:00'),(10,'2020-11-10 00:00:00'),
                          (11,'2020-11-11 00:00:00'); 
# 插入不同月份的数据
> 
INSERT INTO mr_test VALUES(12,'2020-12-01 00:00:00'),(13,'2020-12-02 00:00:00'),
                          (14,'2020-12-03 00:00:00'),(15,'2020-12-04 00:00:00'),
                          (16,'2020-12-05 00:00:00'),(17,'2020-12-06 00:00:00'),
                          (18,'2020-12-07 00:00:00'),(19,'2020-12-08 00:00:00'), 
                          (20,'2020-12-09 00:00:00'),(21,'2020-12-10 00:00:00'),
                          (22,'2020-12-11 00:00:00'); 
```
看看数据存储 `/var/lib/clickhouse下：/data/default/mr_test`
![data_part](data_part.png) 

不同分区（此处是月份）的数据分为了两个 data part 存储。

分区里面的数据：
![a_data_part_data](partition_files.png)

- primary.idx：该 part 所有颗粒第一行的主键值标记
- count.txt：该 part 行数

#### 主键的选择
CH 主键没有列数的要求，需要根据具体的数据结构决定主键列的选择：
- 主键（a,b）添加一列 c 可以提高以下查询性能：查询条件包含`c`；相同（a,b）的数据特别多，加 c 可以跳过相当长的数据。
- CH 使用主键排序数据，提高数据一致性从而提升压缩性能
- 主键过长会影响插入性能，但不会影响 SELECT 性能
- 如果使用 `ORDER BY tuple()` 创建 MergeTree table，会按插入顺序排序；向保留插入顺序需要设置`max_insert_threads = 1`

    