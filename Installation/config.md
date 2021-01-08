ClickHouse集群配置说明

1. 生成 metrika.xml shard/replica
   
    例: 3shard3replica
    
    ip: 3
    
    metrika.xml: 
        总副本=3*3=9
        每节点副本=9/3=3
        每shard3副本分布在3个节点
   
2. 生成 users.xml 
3. 生成 config.xml 
    
    1. include metrika.xml
    2. include users.xml
    3. 设置 port:
         每节点3实例port分布=实例id(从0开始)*100+port
    4. 设置 disks:
         每节点3实例disk分布=(实例id+1)*每实例diskCount~(实例id+2)*每实例diskCount
       
         例：节点共12个磁盘3实例，除去磁盘0(存储代码)，每实例diskCount=3
       
        实例0的3个disk
       ```
       <disks>
           <disk1>
                <path>/data1/jdolap/clickhouse/data/</path>
           </disk1>
           <disk2>
                <path>/data2/jdolap/clickhouse/data/</path>
           </disk2>
           <disk3>
                <path>/data3/jdolap/clickhouse/data/</path>
           </disk3>
       </disks>
       ```
    5. 设置 disks volumes
        
        目前存储策略是 JDOB 单卷多磁盘存储策略。