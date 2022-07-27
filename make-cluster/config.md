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
    3. <listen_host>::</listen_host>
    4. storage_configuration：
        1. disks
            1. 一台物理机多个实例平分磁盘，例如：
                - 实例0：/data0、/data1 （ /data0存储元数据和数据, /data1只存储数据）
                - 实例1：/data2、data3
                - 实例3: /data4、data5
        2. storage_policy
           ```xml
           <jbod>
             <volumes>
                <hot_volume>
                 default
                </hot_volume>
                <cold_volume>
                 other_disks
                </cold_volume>
             </volumes>
           </jbod>
           ```
3. zookeeper和macro
   1. 一般写在metrika.xml文件
   2. 然后在config.xml需要include incl指定 metrika.xml 中的标签名称，例如
      ```xml
      <include_from>metrika1.xml</include_from>
      <remote_servers incl="clickhouse_remote_servers" />
      <macros incl="macros" optional="true" />
      <zookeeper incl="zookeeper-servers" optional="true" />
      ```