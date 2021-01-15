#### ClickHouse集群部署流程
1. Config.sh
   1. rsync defaultdata -> data
   2. cp $CLUSTER/config.xml -> /data0/jdolap/clickhouse/conf
   3. cp $CLUSTER/config{i}.xml -> /data{j}/jdolap/clickhouse/conf (例：/data0, /data2, /data4)
<br></br>
2. Engine.sh ： wget engine，替换 /data0/jdolap/clickhouse/lib/clickhouse
   <br></br>
3. Users.sh : cp $CLUSTER/users.xml -> /data0/jdolap/clickhouse/conf
   <br></br>
4. Metrika.sh :
    1. cp $CLUSTER/metrika{i}.xml -> /data0/jdolap/clickhouse/conf/metrika{i}.xml
    2. cp $CLUSTER/metrika{i}.xml -> /data{j}/jdolap/clickhouse/conf/
       <br></br>
5. Service.sh:
    1.  rsync defaultdata-> data
    2. cp $CLUSTER/clickhouse-server{i}.service -> /etc/systemd/system/
    3. systemctl enable clickhouse-service{i}.service
       <br></br>
6. Start.sh:
    systemctl start clickhouse-service{i}.service


#### 重要配置说明
1. metrika.xml

   例: 3shard3replica集群

   ip: 3

   分布：

   | | process0 | process1 | process2 |
       | --- | --- | --- | --- |
   | ip0 |  shard0副本00 |  shard2副本01 |  shard1副本02 | 
   | ip1 |  shard1副本00 |  shard0副本01 |  shard2副本02 |
   | ip2 |  shard2副本00 |  shard1副本01 |  shard0副本02 |

2. config.xml

    1. include metrika.xml
    2. include users.xml
    3. 设置 port
    4. 设置 disks：每个实例磁盘数=总磁盘数/实例个数

       实例0：/data0、/data1

            /data0存储元数据和默认磁盘, /data1是额外磁盘

       实例1：/data2、data3，

       实例3: /data4、data5

    5. 设置 disks volumes

       策略：JDOB

       一个卷俩磁盘： default 和 /data{ProcessIndex*SingleProcessDiskCount}