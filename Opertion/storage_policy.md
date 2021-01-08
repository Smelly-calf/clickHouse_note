## 一、Create table on cluster

1、解决 circular replicated 循环副本问题：

每个server 配置两个shard的副本时，create database on cluster 会报错：exactly two instance on the same server…
原因：
由于同一个server配置了两个shard，无法区分数据放在哪个shard，解决方式如下：
The trick here is to put every shard into a separate database!

解决方式是为每个 shard 指定不同的 database，不同shard数据放在不同的database 下.
```
<shard>
<internal_replication>true</internal_replication>
<replica>
<default_database>testcluster_shard_1</default_database>
<host>cluster_node_1</host>
</replica>
<replica>
<default_database>testcluster_shard_1</default_database>
<host>cluster_node_2</host>
</replica>
</shard>
```

不同节点上的元数据如下
1. Schemas of the 1st Node
    * testcluster_shard_1
    * testcluster_shard_3
2. Schemas of the 2nd Node
    * testcluster_shard_2
    * testcluster_shard_1
3. Schemas of the 3rd Node
    * testcluster_shard_3
    * testcluster_shard_2

## 二、Create with mergeTree storage policy 问题

https://altinity.com/blog/2019/11/29/amplifying-clickhouse-capacity-with-multi-volume-storage-part-2

multi-volume

clickhouse 组织磁盘的多卷机制，可以通过配置将磁盘排列成卷，使用多层存储极大提升Clickhouse server容量。
未使用 storage policies 时，表MergeTree的数据只存储在一个default磁盘：如 /var/lib/clickhouse/

从 system.tables 查询表所在路径:
`SELECT name, data_paths FROM system.tables WHERE name = 'sample1'`

创建表sample 并插入数据
`CREATE TABLE sample1 (id UInt64) Engine=MergeTree ORDER BY id`

查看 table 路径:
`docker exec testck_clickhouse01_1 clickhouse-client -h 127.0.0.1 --user default --query "SELECT name, data_paths FROM system.tables WHERE name = 'sample1' format TabSeparatedWithNames"`
￼
查看表 parts：
`docker exec testck_clickhouse01_1 clickhouse-client -h 127.0.0.1 --user default --query "select table,disk_name,path from system.parts where table='sample1' format TabSeparatedWithNames"`
￼
存储策略表 system.storage_policies：
`SELECT policy_name, volume_name, disks FROM system.storage_policies`

#### 一、单磁盘策略
storage policy 配置文件： /etc/clickhouse-server/config.d/storage.xml

<policies>
      <ebs_gp2_1_only> <!-- name for new storage policy -->
        <volumes>  
          <ebs_gp2_1_volume> <!-- name of volume -->
            <!-- 
                we have only one disk in that volume  
                and we reference here the name of disk
                as configured above in <disks> section
            -->
            <disk>ebs_gp2_1</disk>
          </ebs_gp2_1_volume>
        </volumes>
      </ebs_gp2_1_only>
    </policies>

以上配置了一个名为 ebs_gp2_1_only 的新的 storage policy， 包含 policy_name，volume_name，disk 的定义，向后兼容，之前的table 使用 default policy。

SETTINGS storage_policy = 'ebs_gp2_1_only' 指定表存储使用 ebs_gp2_1_only 定义的磁盘。
使用 INSERT …SELECT 不受来源表和目标表不同磁盘存储的限制。


#### 二、JBOD：single-tired volume serval disk 单层卷多磁盘
```
<ebs_gp2_jbod> > <!-- name for new storage policy -->
<volumes>
<ebs_gp2_jbod_volume> <!-- name of volume -->
<!--
   the order of listing disks inside
   volume defines round-robin sequence
   -->
<disk>ebs_gp2_1</disk>
<disk>ebs_gp2_2</disk>
</ebs_gp2_jbod_volume>
</volumes>
</ebs_gp2_jbod>
```
数据以轮询的方式分布在不同磁盘：每次insert在下一个 disk 创建一个part，该 part 的一半在一个 disk，其余部分在另外的 disk。

优势：
1. 扩展存储-通过附加磁盘，比迁移到 RAID(磁盘阵列) 更简单；
2. 多线程并行多个磁盘读写，提高读写速度。
3. table loading 速度，由于每个磁盘 parts 数量较少，加载速度更快。

容错：一个 volumn 配置多个 disk 可能会增加数据丢失的风险，总是使用副本保证容错性。

Clickhouse 会进行 Background merges 后台合并，会将位于不同磁盘的parts合并并放在该卷的其中一个磁盘（也采用轮询 round-robin 的方式），使用 OPTIMIZE table ‘table_name’ 强制合并后查询 parts 信息，应该由多个变成一个。
由于 Background merges 的存在，因此JBOD 不能保证数据均匀分配在不同磁盘，也不能保证 I/O 吞吐量一定优于单个磁盘，要想保证数据分布均匀和I/O 吞吐量改用 RAID。

———————————————————以上都是单层卷存储，接下来是分层存储的配置，不同优先级的 volumn —————————

#### 三、Multi-tired storage: volumes with different priorities

回到我们最初要解决的问题：我们有新插入的数据即热点数据，定期访问，需要快速且昂贵的存储。同时有除批量查询之后很少访问的冷数据，性能不是主要考虑因素，使用慢的便宜的存储即可。

1、问题：区分热点数据和冷数据存储，对于热点数据实现快速I/O
storage policy 要解决的两个问题：
1. Insert 新的数据时part的位置；
2. 何时移动至slow存储

2、Background merges: part size

MergeTree engine 会在 merge 的时候根据 part size 移动数据在不同的volumns之间。
MergeTree engine会在后台不断进行 Background merges，随着时间推移，会合并新插入的数据和 small parts 到更大的parts，几次merge之后会出现 big parts，在 MergeTree engine 中 Part size与Part age 是紧密联系的，通常 the bigger the part，the older it is。

因此分别配置 superfast 热点存储和普通存储，在热点存储中设置 max_data_part_size_bytes 决定何时迁移至慢磁盘。
另外volume 的顺序很重要，新part 首先会尝试放在配置的第一个 volume，其次是第二个，因此应该将热点存储放在第一位。
```
<ebs_hot_and_cold>
<volumes>
<hot_volume>
<disk>ebs_gp2_2</disk>
<!--
   that volume allowed to store only parts which
   size is less or equal 200Mb
-->
<max_data_part_size_bytes>200000000</max_data_part_size_bytes>
</hot_volume>
<cold_volume>
<disk>ebs_sc1_1</disk>
<!--
  that volume will be used only when the first
  has no space of if part size doesn't satisfy
  the max_data_part_size_bytes requirement of the
  first volume, i.e. if part size is greater
  than 200Mb
-->
</cold_volume>
</volumes>
</ebs_hot_and_cold>
```

查询热卷->冷卷设置的parts大小:
```
SELECT disk_name, formatReadableSize(bytes_on_disk) AS size FROM system.parts WHERE (table = 'sample4') AND active;
```
New part 的 size 是一个估算值，insert 时候part size使用未压缩的part size 估算，merge 的时候使用合并后的压缩大小之和+10%来估算，因此有可能较大的 part 在 hot volumn，而较小的 part 在 cold volumn。

使用 alter 语句 manually move parts
```
ALTER TABLE sample4 MOVE PART 'all_570_570_0' TO VOLUME 'cold_volume'
ALTER TABLE sample4 MOVE PART 'all_570_570_0' TO DISK 'ebs_gp2_1'
ALTER TABLE sample4 MOVE PARTITION tuple() TO VOLUME 'cold_volume'
```

3、Background moves: move_factor

除了 background merges 外，background moves 也会移动 part，当 volumn free space 小于 10% （由参数 move_factor 控制，默认值 0.1）会进行 background moves， move_factor=0 表示屏蔽 background moves。
我们可以通过system.part_log 查看移动过程，需要配置part_log.xml 使 system.part_log 生效。

在 storage.xml 中配置 move_factor
```
<ebs_hot_and_cold_movefactor99>
<volumes>
<hot_volume>
<disk>ebs_gp2_2</disk>
<max_data_part_size_bytes>200000000</max_data_part_size_bytes>
</hot_volume>
<cold_volume>
<disk>ebs_sc1_1</disk>
</cold_volume>
</volumes>
<move_factor>0.9999</move_factor>
</ebs_hot_and_cold_movefactor99>
```
配0.9999是为了更快的在 system.part_log 看到移动过程。

```
/etc/clickhouse-server/config.d/part_log.xml
<yandex>
<part_log>
<database>system</database>
<table>part_log</table>
<flush_interval_milliseconds>7500</flush_interval_milliseconds>
</part_log>
</yandex>
```

创建表并load some data
```
CREATE TABLE sample5 (id UInt64) Engine=MergeTree ORDER BY id SETTINGS storage_policy = 'ebs_hot_and_cold_movefactor99';

INSERT INTO sample5 SELECT rand() FROM numbers(100);
```
验证 move 过程：
`SELECT event_type, path_on_disk FROM system.part_log`


终极方案：结合 JBOD 和 分层存储
```
<policies>
<three_tier>
<volumes>
<hot_volume>
<disk>superfast_ssd1</disk>
<disk>superfast_ssd2</disk>
<max_data_part_size_bytes>200000000</max_data_part_size_bytes>
</hot_volume>
<jbod_volume>
<disk>normal_hdd1</disk>
<disk>normal_hdd2</disk>
<disk>normal_hdd3</disk>
<max_data_part_size_bytes>80000000000</max_data_part_size_bytes>
</jbod_volume>
<archive_volume>
<disk>slow_and_huge</disk>
</archive_volume>
</volumes>
</three_tier>
</policies>
```


查看数据 parts：
```
CREATE TABLE sample1 (id UInt64) Engine=MergeTree ORDER BY id;

INSERT INTO sample1 SELECT * FROM numbers(1000000)

-- 查看表路径
SELECT name, data_paths FROM system.tables WHERE name = 'sample1'

-- 查看表 parts 信息
SELECT name, disk_name, path FROM system.parts WHERE (table = 'sample1') AND active
```

