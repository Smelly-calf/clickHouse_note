# ClickHouse 性能

https://clickhouse.tech/docs/en/introduction/performance/

## 单个查询吞吐量
单个查询吞吐量=每秒返回的行数/MB。

如果数据存放在 Page Cache，一个不是太复杂的查询在现在的硬件基础上可以达到<em><b>每台服务器 2-10GB/s </b></em>的处理速度；

如果不在 page cache，取决于磁盘子系统读取数据的能力和数据压缩率。磁盘读取能力 400MB/s 和数据压缩率 3 代表 <em><b>处理速度 1.2GB/s</b></em>.

吞吐量=处理速度/每行列的总大小，假设列的总大小 10 个字节，1.2GB/s 的处理速度 对应 (1200MB/10B)/s ~ <em><b>120 Millon rows/s 的吞吐量</b></em>.

分布式处理的处理速度几乎是线性增长，前提是聚合或排序的结果行数不太大。

## 短查询的延迟
如果数据在 page cache，使用主键的少量列少量行的查询大约 50 ms 延迟。

否则根据公式：<em><b>latency:寻址时间(10ms) \* 查询的 columns 数 * 数据 parts 数</b></em>

## 处理大量短查询时的吞吐量
ClickHouse 每秒可处理几百个查询。建议<em><b>单台服务器每秒最多 100 次查询</b></em>.

## 插入数据的性能
建议以 <em>至少1000 行的数据包</em> 插入，或者 <em><b>每秒 insert 不超过 1 次</b></em>。

MergeTree table 的插入速度是 50～200MB/s，假设一行是 1Kb，那么每秒可插入 50,000~200,000 行。

行小则插入性能越高：

Banner System data->每秒 insert 500,000 rows， 

Graphite data->每秒可 insert 1,000,000 rows。
