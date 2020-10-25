# clickHouse_note

# Clickhouse 笔记
https://clickhouse.tech/docs/en/
## Clickhouse 是什么？
通俗来讲，Clickhouse 是一个面向列的数据库管理系统，用于 OLAP：查询的在线分析处理。

对于面向行的数据库管理系统来说，一行物理存储在一起。

对于面向列的数据库管理系统来说，一列物理存储在一起，不同列数据分开存储。

根据不同场景设计不同数据系统，考虑的因素：
- 什么查询：what queries are made, how often, and in what proportion;
- 读取和更新关系
- 读取多少数据：how much data is read for each type of query - rows, columns, and bytes;
- 读取和更新的比例：the relationship between reading and updating data;
- 事物、隔离机制：whether transactions are used, and how isolated they are;
- 查询的延迟和吞吐量的要求：requirements for latency and throughput for each type of query, and so on.
- 数据复制和逻辑完整性要求：requirements for data replication and logical integrity;


OLAP 场景的关键特性：
- 读取和更新的关系
    - 读请求：绝大多数请求是为读取而访问；
    - 批量更新而非单行：Data is updated in fairly large batches (> 1000 rows), not by single rows; or it is not updated at all.
    - 数据只添加基本不更新
- 面向列
    - 表很宽：很多列；
    - 列值相当小：数字或短字符串（例如每个 URL 60字节）
    - 查询一小部分列：数据读取时从数据库中提取很多行，但一小部分列；
- 查询
    - 查询频率相对较小：每台服务器每秒几百次（hundreds of queries/second-几百 QPS）查询甚至更少。
    - 简单查询允许 50ms 的延迟；
    - 高吞吐量：处理单个查询需要高吞吐量（每秒每台服务器数十亿行）；
    - 弱数据一致性要求
    - 每个查询是<em><b>只有一张大表</b></em>和多个小表的关联；
    - 查询结果远小于源数据，查询结果在单个服务器的 RAM 中就可以存下。

面向列的数据库更适用于 OLAP场景（at least 100 times faster in processing most queries）：
- I/O：
    - 只读取想要的列，读取 100 列中的 5 列可以减少 20 倍的 I/O；
    - 以数据包的形式读取，易于压缩，显著减少 I/O 数量；
    - 由于减少了 I/O，更多数据可存在于系统缓存
    
- CPU
  
  执行单个查询需要大量行，有助于对整个向量而不是单独的行调度操作，且有助于实现无需调度代价的查询引擎。否则查询解释器处理大量的行不可避免会使 CPU 停顿。
  
  按列存储并在可能时按列处理是有意义的。
  
  - 实现向量引擎：针对矢量而不是单个值编写所有操作。这意味着不需要频繁调用操作，并且调度代价可忽略不计。操作代码包含一个优化的内部循环。
  - 代码生成：为查询生成的代码中包含所有间接调用。

    在“常规”数据库中不会执行此操作，因为在运行简单查询时这没有意义。但是，也有例外：<em>MemSQL使用代码生成来减少处理SQL查询时的延迟。 （比较：分析型DBMS需要优化吞吐量而不是延迟。）</em>
    
    <em><font color="red">请注意，为了提高CPU效率，查询语言必须是声明性的（SQL或MDX），或者至少是向量（J，K）。该查询应仅包含隐式循环，以便进行优化。</em>