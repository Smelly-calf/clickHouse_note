本节主要搭建keeper集群，然后把copier的keeper.xml配置改成用自己搭建的keeper集群，为了复现社区的一个issue。

issue：https://github.com/ClickHouse/ClickHouse/issues/39282

keeper源码走读：https://mp.weixin.qq.com/s/qO4IcZb3X1mIeYXv1QLv_A

步骤：

一、基于tag 22.5.2.53 编译clickhouse：编译过程见 [编译ClickHouse指定版本](../source-code/day1-build.md)

二、准备三台ssd机器，用于部署clickhouse和keeper

三、部署：https://clickhouse.com/docs/en/guides/sre/keeper/clickhouse-keeper/

1. 启动keeper

   keeper配置

   ```xml
   <keeper_server>
       <tcp_port>9181</tcp_port>
       <server_id>3</server_id>
       <log_storage_path>/{chroot}/clickhouse/coordination/log</log_storage_path>
       <snapshot_storage_path>/{chroot}/clickhouse/coordination/snapshots</snapshot_storage_path>
   
       <coordination_settings>
           <operation_timeout_ms>10000</operation_timeout_ms>
           <session_timeout_ms>30000</session_timeout_ms>
           <raft_logs_level>warning</raft_logs_level>
       </coordination_settings>
   
       <raft_configuration>
           <server>
               <id>1</id>
               <hostname>chnode1</hostname>
               <port>9444</port>
           </server>
           <server>
               <id>2</id>
               <hostname>chnode2</hostname>
               <port>9444</port>
           </server>
           <server>
               <id>3</id>
               <hostname>chnode3</hostname>
               <port>9444</port>
           </server>
       </raft_configuration>
   </keeper_server>
   
   <!-- enable zookeeper use keeper -->
   <zookeeper>
           <node>
               <host>chnode1</host>
               <port>9181</port>
           </node>
           <node>
               <host>chnode2</host>
               <port>9181</port>
           </node>
           <node>
               <host>chnode3</host>
               <port>9181</port>
           </node>
   </zookeeper>
   ```

2. 所有节点cliickhouse启动后验证keeper

   ```
   echo ruok | nc localhost 9181; echo
   ```

3. 集群模式

   config.xml 中引用 metrika.xml
   ```xml
    <include_from>{chroot}/clickhouse/conf/metrika.xml</include_from>
    <macros incl="macros" optional="true" />
   ```

   metrika.xml：
   ```xml
   <yandex>
   <remote_servers>
       <dest_cluster>
           <shard>
               <replica>
                   <host>chnode1</host>
                   <port>9000</port>
                   <user>default</user>
               </replica>
           </shard>
           <shard>
               <replica>
                   <host>chnode2</host>
                   <port>9000</port>
                   <user>default</user>
               </replica>
           </shard>
         </dest_cluster>
     </remote_servers>
     <macros>
       <shard_dict>1</shard_dict>
       <replica_dict>1</replica_dict>
      </macros>
     </yandex>
   ```

重启
   ```
   chown -R clickhouse:clickhouse ~/clickhouse
   systemctl restart clickhouse.service
   ```

验证clusters
   ```
   show clusters;
   ```

四、copier迁移数据：source_cluster ->  dest_cluster

keepeer.xml
```
<clickhouse>
    <logger>
        <level>trace</level>
        <size>100M</size>
        <count>3</count>
    </logger>

    <zookeeper>
        <node index="1">
            <host>keeperip</host> 
            <port>9181</port>
        </node>
    </zookeeper>
</clickhouse>
```

tasks.xml见 [copier使用](copier使用.md)

启动copier

```
./clickhouse copier --daemon --config-file clickhouse-server/copier/keeper.xml --task-path /transfer/test01 --base-dir clickhouse-server/log

重启时先清空keeper数据，可以使用 zkCli 操作 keeper
```

验证数据量
```
select count() from calf.test_move_d;
```

