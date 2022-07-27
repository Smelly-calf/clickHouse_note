1. 首先下载clickhouse可执行文件，启动copier时需要，例如下载到 clickhouse-server/clickhouse

2. 确定源集群和目标集群：

   例如：源集群：source_cluster，目标集群：dest_cluster

3. 准备数据

源集群执行

   ```sql
   -- 创建账户
   CREATE USER IF NOT EXISTS calf ON CLUSTER source_cluster IDENTIFIED WITH double_sha1_password BY 'passwd';
   
   -- 建库
   CREATE DATABASE IF NOT EXISTS "calf" ON CLUSTER "source_cluster";
   
   -- 授权
   GRANT ON CLUSTER source_cluster SHOW, SELECT, INSERT, ALTER, CREATE DATABASE, CREATE TABLE, CREATE VIEW, CREATE DICTIONARY, DROP, TRUNCATE, OPTIMIZE, SYSTEM MERGES, SYSTEM TTL MERGES, SYSTEM FETCHES, SYSTEM MOVES, SYSTEM SENDS, SYSTEM REPLICATION QUEUES, SYSTEM SYNC REPLICA, SYSTEM RESTART REPLICA, SYSTEM FLUSH DISTRIBUTED, dictGet ON "calf".*, SYSTEM RELOAD DICTIONARY ON *.* TO "calf";
   
   -- 创建复制表
   CREATE TABLE IF NOT EXISTS calf.test_move_local on CLUSTER source_cluster (
     tenant_id UInt32,
     alert_id String,
     timestamp DateTime Codec(Delta, LZ4),
     alert_data String,
     acked UInt8 DEFAULT 0,
     ack_time DateTime DEFAULT toDateTime(0),
     ack_user LowCardinality(String) DEFAULT ''
   ) ENGINE = ReplicatedMergeTree(
     '/source_cluster/clickhouse/tables/calf/test_move_local/shard-{shard}', '{replica}') PARTITION BY toYYYYMM(timestamp) ORDER BY (tenant_id, timestamp, alert_id) SETTINGS index_granularity = 8192;
     
   -- 创建分布式表
   CREATE TABLE IF NOT EXISTS calf.test_move_d ON CLUSTER source_cluster AS calf.test_move_local ENGINE = Distributed('source_cluster', 'calf', 'test_move_local', rand());
   
   -- 分布式表插入10000条随机生成的数
   INSERT INTO
     calf.test_move_d (tenant_id, alert_id, timestamp, alert_data)
   SELECT
     toUInt32(rand(1) % 1000 + 1) AS tenant_id,
     randomPrintableASCII(64) as alert_id,
     toDateTime('2020-01-01 00:00:00') + rand(2) % (3600 * 24 * 30) * 12 as timestamp,
     randomPrintableASCII(1024) as alert_data
   FROM
     numbers(10000);
     
    -- 验证数据量
    select count() from calf.test_move_d;
   ```



目标集群执行：

   ```sql
   -- 创建账户
   CREATE USER IF NOT EXISTS calf ON CLUSTER dest_cluster IDENTIFIED WITH double_sha1_password BY 'passwd';
   
   -- 建库
   CREATE DATABASE IF NOT EXISTS "calf" ON CLUSTER "dest_cluster";
    
   -- 授权
   GRANT ON CLUSTER dest_cluster SHOW, SELECT, INSERT, ALTER, CREATE DATABASE, CREATE TABLE, CREATE VIEW, CREATE DICTIONARY, DROP, TRUNCATE, OPTIMIZE, SYSTEM MERGES, SYSTEM TTL MERGES, SYSTEM FETCHES, SYSTEM MOVES, SYSTEM SENDS, SYSTEM REPLICATION QUEUES, SYSTEM SYNC REPLICA, SYSTEM RESTART REPLICA, SYSTEM FLUSH DISTRIBUTED, dictGet ON "calf".*, SYSTEM RELOAD DICTIONARY ON *.* TO "calf";
   
   -- 创建复制表
   CREATE TABLE IF NOT EXISTS calf.test_move_local on CLUSTER dest_cluster (
     tenant_id UInt32,
     alert_id String,
     timestamp DateTime Codec(Delta, LZ4),
     alert_data String,
     acked UInt8 DEFAULT 0,
     ack_time DateTime DEFAULT toDateTime(0),
     ack_user LowCardinality(String) DEFAULT ''
   ) ENGINE = ReplicatedMergeTree(
     '/clickhouse/tables/calf/test_move_local/shard-{shard}', '{replica}') PARTITION BY toYYYYMM(timestamp) ORDER BY (tenant_id, timestamp, alert_id) SETTINGS index_granularity = 8192;
     
   -- 创建分布式表
   CREATE TABLE IF NOT EXISTS calf.test_move_d ON CLUSTER dest_cluster AS calf.test_move_local ENGINE = Distributed('dest_cluster', 'calf', 'test_move_local', rand());
   ```


copier配置：

https://clickhouse.com/docs/en/operations/utilities/clickhouse-copier/

cd clickhouse-server/

vim keeper.xml

```
<clickhouse>
    <logger>
        <level>trace</level>
        <size>100M</size>
        <count>3</count>
    </logger>

    <zookeeper>
        <node index="1">
            <host>zkip</host> 
            <port>2181</port>
        </node>
    </zookeeper>
</clickhouse>
```

tasks.xml
```
<clickhouse>
    <!-- Configuration of clusters as in an ordinary server config -->
    <remote_servers>
        <source_cluster>
            <shard>
                <internal_replication>true</internal_replication>
                <!-- <weight>true</weight> -->
                <replica>
                    <host>ip</host>
                    <port>9000</port>
                    <user>calf</user>
                    <password>passwd</password>
                </replica>
                ... 
            </shard>
            ... 
        </source_cluster>

        <dest_cluster>
            <shard>
                <internal_replication>true</internal_replication>
                <!-- <weight>true</weight> -->
                <replica>
                    <host>ip</host>
                    <port>9000</port>
                    <user>calf</user>
                    <password>passwd</password>
                </replica>
                ... 
            </shard>
            ... 
        </dest_cluster>
    </remote_servers>

    <!-- How many simultaneously active workers are possible. If you run more workers superfluous workers will sleep. -->
    <max_workers>2</max_workers>

    <!-- Setting used to fetch (pull) data from source cluster tables -->
    <settings_pull>
        <readonly>1</readonly>
    </settings_pull>

    <!-- Setting used to insert (push) data to destination cluster tables -->
    <settings_push>
        <readonly>0</readonly>
    </settings_push>

    <!-- Common setting for fetch (pull) and insert (push) operations. Also, copier process context uses it.
         They are overlaid by <settings_pull/> and <settings_push/> respectively. -->
    <settings>
        <connect_timeout>3</connect_timeout>
        <!-- Sync insert is set forcibly, leave it here just in case. -->
        <insert_distributed_sync>1</insert_distributed_sync>
    </settings>

    <!-- Copying tasks description.
         You could specify several table task in the same task description (in the same ZooKeeper node), they will be performed
         sequentially.
    -->
    <tables>
        <!-- A table task, copies one table. -->
        <table_hits>
            <!-- Source cluster name (from <remote_servers/> section) and tables in it that should be copied -->
            <cluster_pull>source_cluster</cluster_pull>
            <database_pull>calf</database_pull>
            <table_pull>test_move_local</table_pull>

            <!-- Destination cluster name and tables in which the data should be inserted -->
            <cluster_push>dest_cluster</cluster_push>
            <database_push>calf</database_push>
            <table_push>test_move_local</table_push>

            <!-- Engine of destination tables.
                 If destination tables have not be created, workers create them using columns definition from source tables and engine
                 definition from here.

                 NOTE: If the first worker starts insert data and detects that destination partition is not empty then the partition will
                 be dropped and refilled, take it into account if you already have some data in destination tables. You could directly
                 specify partitions that should be copied in <enabled_partitions/>, they should be in quoted format like partition column of
                 system.parts table.
            -->
            <engine>
            ENGINE = ReplicatedMergeTree('/clickhouse/tables/calf/test_move_local/shard-{shard}', '{replica}')
PARTITION BY toYYYYMM(timestamp)
ORDER BY (tenant_id, timestamp, alert_id)
SETTINGS index_granularity = 8192
            </engine>

            <!-- Sharding key used to insert data to destination cluster -->
            <sharding_key>jumpConsistentHash(intHash64(tenant_id), 2)</sharding_key>

            <!-- This section specifies partitions that should be copied, other partition will be ignored.
                 Partition names should have the same format as
                 partition column of system.parts table (i.e. a quoted text).
                 Since partition key of source and destination cluster could be different,
                 these partition names specify destination partitions.

                 NOTE: In spite of this section is optional (if it is not specified, all partitions will be copied),
                 it is strictly recommended to specify them explicitly.
                 If you already have some ready partitions on destination cluster they
                 will be removed at the start of the copying since they will be interpeted
                 as unfinished data from the previous copying!!!
            -->
        </table_hits>
    </tables>
</clickhouse>
```

上传tasks.xml
 ```
slog zkip
cd zookeeper/pkg/bin

./zkCli.sh -server zkip:2181 create /transfer ""
./zkCli.sh -server zkip:2181 create /transfer/test01 ""
./zkCli.sh -server zkip:2181 create "/transfer/test01/description" "`cat tasks.xml`"
./zkCli.sh -server zkip:2181 get /transfer/test01/description
 ```

启动copier
```
cd ~/clickhouse-server
./clickhouse copier --daemon --config-file clickhouse-server/copier/keeper.xml --task-path /transfer/test01 --base-dir clickhouse-server/log
```



