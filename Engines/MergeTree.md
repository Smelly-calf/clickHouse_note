# MergeTree 表引擎

2020.11.13

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

- 颗粒: 每个part逻辑上划分为颗粒，每个颗粒是CH读取数据时的最小不可分割数据集，即一次性读取一整个颗粒的数据。

    每个颗粒的第一行会创建基于主键值的标记（mark），每个part会创建一个索引文件存储 marks，如下面的`mr_test`表的 `202011`分区下有一个`primary.idx`文件。
    
    `index_granularity`和`index_granularity_bytes`设置表引擎的索引粒度，但是一行不会被拆分到两个part，即行大小超过索引粒度字节数限制，该行会整体存放到当前part；因此在这种情况下 颗粒的大小=行数。

一个数据存储测试：
```
$ docker run -d --name oneck -p 9009:9000 -v $HOME/oneck/etc:/etc/clickhouse-server -v $HOME/oneck/data:/var/lib/clickhouse yandex/clickhouse-server:20.6.10
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
![data_part](illstration/data_part.png) 

不同分区（此处是月份）的数据分为了两个 data part 存储。

分区下包含的文件：
![a_data_part_data](illstration/partition_files.png)

主要文件说明：
- primary.idx //mark值和主键元组的映射
- count.txt //part行数
- minmax_Date.idx // 分区键最大最小值
- xx.bin //列向量
- xx.mrk2 //mark值和列的映射 
 
#### 主键和索引
CH <em>根据主键来建立索引 marks</em>，索引粒度由 `index_granularity`和`index_granularity_bytes`设置，表示两个索引之间的行数/字节数。

如下图是主键为 (CounterID, Date) 索引粒度为 7 的索引标记图：
 ```
  Whole data:     [---------------------------------------------]
  CounterID:      [aaaaaaaaaaaaaaaaaabbbbcdeeeeeeeeeeeeefgggggggghhhhhhhhhiiiiiiiiikllllllll]
  Date:           [1111111222222233331233211111222222333211111112122222223111112223311122333]
  Marks:           |      |      |      |      |      |      |      |      |      |      |
                  a,1    a,2    a,3    b,3    e,2    e,3    g,1    h,2    i,1    i,3    l,3
  Marks numbers:   0      1      2      3      4      5      6      7      8      9      10
```

- 当查询条件是`CounterID in ('a','h')`-服务器会读取标记 [0,3) 和 [6,8) 范围的数据；`CounterID in ('a','h') and Date=3` - 读取标记 [1,3) 和 [7,8) 范围的数据；`Date=3` - 读取标记[1,10]范围的数据
- 稀疏索引允许读取多余的行，当读取单个范围的主键时，每个数据块中最多可以读取`index_granularity * 2`的额外的行。
- 稀疏索引可以处理大量的行，因为大多数这种索引都跟计算机的 RAM 是匹配的。
- CH中不要求主键唯一，即主键可以重复。

当查询条件是 `Date=3` 时可以命中 marks:[1,10], 原因：因为 CH 会为 MergeTree 类型的每一列存储索引信息.

CH命中索引标记的场景：WHERE/PREWHERE的每个子句(非AND连接的子句)都必须包含 分区键/主键的 以下表达式：
1. 等值或不等或范围表达式
2. 固定前缀的 LIKE/IN 表达式
3. 以上表达式的逻辑关系表达式
4. 查询参数范围的主键值的索引区间是一个单调区间

   对部分单调命中索引标记的理解：有两个条件：
   1. 查询参数范围的主键值在同一个分区；
   2. 查询参数范围的主键值的索引区间是一个单调区间；此时CH可计算出参数距离索引之间的距离从而定位到索引，然后使用索引进行查询。
   
分析CH查询时是否命中索引：开启`force_index_by_date`和`force_primary_key`，以及查询计划 trace log 工具。

#### 主键的选择
CH 主键没有列数的要求，需要根据具体的数据结构决定主键列的选择，原则：
- 提高查询性能：如主键(a,b)添加一列c可以提高查询条件包含c的查询性能
- 利用主键排序提高数据一致性从而提升压缩性能
- 主键不能过长：主键过长不会影响SELECT性能，但会影响插入性能
- 如果使用 `ORDER BY tuple()` 创建 MergeTree table，会按插入顺序排序；要保留插入顺序需要设置`max_insert_threads = 1`

主键和 sorting key：
- 主键可以与sorting键不同但必须以 sorting 建开头；
- 当表维度很多时应当使用 sorting key 包含所有维度，而主键只保留部分维度列，使用主键定位数据范围排序键排序。

#### Skipping Indexes
使用列表达式创建索引，提高查询性能：
```
INDEX index_name expr TYPE type(...) GRANULARITY granularity_value
```
说明：
- expr: 列表达式
- granularity_value: 索引的颗粒大小
- 索引 type:
    - primary key 
    
      支持的函数子集：equals, notEquals, like, notLike, startsWith, in, notIn, less, greater, lessOrEquals, greaterOrEquals, empty, notEmpty
      
      不支持的函数子集：endsWith, multiSearchAny, hasToken 
    - minmax
    
      索引存储表达式的最值
      
      不支持的函数：endsWith, multiSearchAny, hasToken ，同 primary key
    - set(max_rows)
    
      存储表达式的唯一值，不超过 max_rows 行
      
      支持所有函数
    - ngrambf_v1(n, size_of_bloom_filter_in_bytes, num_of_hash_functions, random_seed)
    
      存储一个布隆过滤器，该布隆过滤器包含数据块中所有ngrams。
      
      只对 strings 起作用，优化 equals,like,in 表达式。
      - n //ngram大小
      - size_of_bloom_filter_in_bytes //布隆过滤器字节大小
      - num_of_hash_functions //hash函数个数
      - random_seed //hash种子
      
      支持 endsWith, multiSearchAny
      
      不支持 less, greater, lessOrEquals, greaterOrEquals, empty, notEmpty, hasToken
      
      特殊：函数参数小于 ngram 大小的常量，不会使用 ngrambf_v1 优化查询。
    - tokenbf_v1(size_of_bloom_filter_in_bytes, num_of_hash_functions, random_seed)
    
      和 ngrambf_v1相同，但存储的是token而不是ngram，token是非字母数字字符分隔的序列
      
      不支持 multiSearchAny, less, greater, lessOrEquals, greaterOrEquals, empty, notEmpty
      
      支持 hasToken
    - bloom_filter([false_positive])
    
      可选参数false_positive表示从布隆过滤器接收误报的可能性，可能的值在(0,1)区间，默认值是 0.025.
      
      支持的数据类型：Int*, UInt*, Float*, Enum, Date, DateTime, String, FixedString, Array, LowCardinality, Nullable.
      
      起作用的函数：equals, notEquals, in, notIn, has.

      特殊：布隆过滤器有假阳性校验，对于 bloom_filter, ngrambf_v1, tokenbf_v1 索引类型来说，当表达式结果预期为假不会使用索引优化查询。
      
      例如：预期为假的表达式：`NOT s LIKE '%test%'`
    
例：
```
docker exec -it testck_clickhouse01_1 clickhouse-client -m --send_logs_level=trace
CREATE TABLE table_name
(
    u64 UInt64,
    i32 Int32,
    s String,
    INDEX a (u64 * i32, s) TYPE minmax GRANULARITY 3,
    INDEX b (u64 * length(s)) TYPE set(1000) GRANULARITY 4
) ENGINE = MergeTree()
ORDER BY tuple();

INSERT INTO table_name SELECT rand(1)%10000 AS u64, rand(1)%100 AS i32, toString(rand(1)%10000) AS s from numbers(10000);
```
会使用索引的查询，从中看出使用索引中包含的一列也会命中索引，因为 CH 会为 MergeTree 类型的每一列存储索引信息：
```
SELECT count() FROM table_name WHERE s < 'z'; 
SELECT count() FROM table_name WHERE u64 * i32 == 10 AND u64 * length(s) >= 1234;
```

todo TTL for Columns and Tables

#### 小结
MergeTree 系列引擎是CH功能最强大最健壮的表引擎，
##### 特性：
- Primary key 排序存储数据，稀疏索引快速定位数据位置
- partition 分区
- sample 采样: 主键必须包含sample key

##### 存储：
- part
    - primary.idx // 该part所有index
    - bin // 数据
    - 列名.mrk // 每一列的标注
    - minmax_partition.idx // 最小最大分区键索引 

##### 索引类型：
todo

##### 今日讨论：
CK数据块划分跟粒度是否有关？




    
