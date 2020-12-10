# Docker Clickhouse集群搭建

ck: 

https://blog.csdn.net/qq_42016966/article/details/107687181

zk: 

https://blog.csdn.net/myNameIssls/article/details/81561975

第三方开发的可视化界面： https://clickhouse.tech/docs/zh/interfaces/third-party/gui/


## 方案一 使用 docker 搭建

使用一台物理机搭建一个有两台 CK 的集群
  
Clickhouse 相关端口：http 端口：8123｜ tcp 端口：9000｜ mysql 端口：9004｜ 内部通讯端口：9009

1、操作 docker
```shell 
# 首先起一个单机的 CH
docker run -d \
   --name clickhouse-server \
   --ulimit nofile=262144:262144 \
   -p 9000:9000 \
   -p 8123:8123 \
   -p 9009:9009 \
   yandex/clickhouse-server

# 将上面的CK的配置文件复制到宿主机
docker cp clickhouse-server:/etc/clickhouse-server/ $HOME/ckcluster/

# 删除单机 CH server
docker rm --force clickhouse-server

# 起一个 clickhouse-server 将配置、日志、数据映射到宿主机
docker run --restart always -d \
      --name ckcluster-1 \
      --ulimit nofile=262144:262144 \
      --volume=$HOME/ckcluster/clickhouse-server/:/etc/clickhouse-server/ \
      --volume=$HOME/ckcluster/clickhouse/:/var/lib/clickhouse/ \
      --volume=$HOME/ckcluster/log/clickhouse-server/:/var/log/clickhouse-server/  \
      -p 9000:9000 \
      -p 8123:8123 \
      -p 9009:9009 \
      yandex/clickhouse-server


# 转到配置目录 复制一份配置文件
cd $HOME/ckcluster/ && cp -R clickhouse-server/ clickhouse-server2/

# 起第二个 CH 实例：名字、日志、数据、配置都在不同的目录，端口也加1
docker run --restart always -d \
    --name ckcluster_2 \
    --ulimit nofile=262144:262144 \
    --volume=$HOME/ckcluster/clickhouse-server2/:/etc/clickhouse-server/ \
    --volume=$HOME/ckcluster/clickhouse2/:/var/lib/clickhouse/ \
    --volume=$HOME/ckcluster/log/clickhouse-server2/:/var/log/clickhouse-server/  \
    -p 9001:9000 \
    -p 8124:8123 \
    -p 9010:9009 \
    yandex/clickhouse-server

# 先启一个 zookeeper
docker run --name zookeeper --restart always -d zookeeper
# 查看 WorkingDir
docker inspect zookeeper
# 停止 zookeeper
docker rm --force zookeeper

# 起一个 zookeeper，CH 的集群分布式依赖 zookeeper
docker run --restart always -d \
    --name zookeeper -p 2181:2181 \
    -v $HOME/ckcluster/zookeeper/conf/:/apache-zookeeper-3.6.2-bin/conf \
    -v $HOME/ckcluster/zookeeper/data/:/data/ \
    -v $HOME/ckcluster/zookeeper/datalog/:/datalog \
    -v $HOME/ckcluster/zookeeper/logs/:/logs \
    zookeeper
# 确认 ipv4 地址
docker inspect -f {{.NetworkSettings.IPAddress}} zookeeper

# 起一个 ZKUI，便于查看zookeeper中的情况
# zkui搭建：https://www.jianshu.com/p/db5d25a80e71
# 下载 zkui: 
git clone https://github.com/DeemOpen/zkui.git
# 传输到 docker，容器名:路径 
docker cp zkui zkui:/ 
# 进入容器后执行
cd zkui
mvn clean install
```
    
访问 `localhost:9090 ` 确认。

附：
- `docker inspect -f "{{.Mounts}}" some-clickhouse-server` 查看容器挂载目录
- `docker inspect some-clickhouse-server` 查看容器ipv4的地址
- `docker rm --force some-clickhouse-server ` 删除运行中的容器
- `docker ps [-a|--all]` 查看所有容器

2、修改 CH 配置

每个节点下修改配置，这里两个节点。

a. 修改 config.xml 
   
   interserver_http_host=本机IP/0.0.0.0, listen_host=本机IP/0.0.0.0
   
   添加两行
   
    
    <include_from>/etc/clickhouse-server/metrika.xml</include_from>     
    <timezone>Asia/Shanghai</timezone>
    
    
b. 创建 metrika.xml
```
<yandex>
    <!--ck集群节点-->
    <clickhouse_remote_servers>
<clickhouse_cluster_name>
<!--分片1-->
<shard>
    <internal_replication>true</internal_replication>
    <replica>
        <!--这里写节点1的IP4地址,docker inspect -f {{.NetworkSettings.IPAddress}} container-->
        <host>192.168.1.xx1</host> 
        <!--这里写节点1的tcp端口 -->
        <port>9000</port>
        <!--这里写节点1的账号-->
        <user>default</user>
        <!--这里写节点1的账号对应的密码, 默认无-->
        <password></password>
    </replica>
</shard>
<!--分片2-->
<shard>
    <internal_replication>true</internal_replication>
    <replica>
    	<!--这里写节点2的IP4地址-->
    	<!--这里的地址有点坑，它是docker容器的地址：查看命令：docker inspect 容器ID-->
        <host>192.168.1.xx2</host>
        <!--这里写节点2的tcp端口-->
        <port>9000</port>
        <!--这里写节点2的账号-->
        <user>default</user>
         <!--这里写节点2的账号对应的密码,默认无-->
        <password></password>
    </replica>
</shard>
</clickhouse_cluster_name>
    </clickhouse_remote_servers>


<!--zookeeper相关配置-->
<zookeeper-servers>
  <node index="1">
 <!--这里写Zookeeper的IP，如果用docker启动的： docker inspect 容器ID  查看docker容器的IPv4 -->
<host>192.168.1.1</host>
<!--这里写Zookeeper的端口-->
<port>2181</port>
  </node>
</zookeeper-servers>

<macros>
<layer>01</layer>
<shard>01</shard> <!--这个节点配置的分片号-->
<replica>01</replica> <!--当前分片为1，副本为1-->
</macros>

<networks>
<ip>::/0</ip>
</networks>

<!--压缩相关配置-->
<clickhouse_compression>
<case>
<min_part_size>10000000000</min_part_size>
<min_part_size_ratio>0.01</min_part_size_ratio>
<method>lz4</method> <!--压缩算法lz4压缩比zstd快, 更占磁盘-->
</case>
    </clickhouse_compression>
</yandex>
```
      
注意：需要更改配置文件，修改metrcs.xml里面 分片、副本名

3、 重启 

```
docker restart ckcluster1
docker restart ckcluster2
docker restart zookeeper
docker restart zkui
```

4、连接 clickhouse 集群
    ```
    # 方法1：从 server 端进入交互终端
    docker exec -it container /bin/bash;exit 
    > clickhouse-client 
    # 方法2：从 client 端进入交互终端
    docker run -it --rm --link ckcluster_1:ckcluster_1 yandex/clickhouse-client --host ckcluster_1
    ```
   
   终端输入：查看集群信息
   ```
   > select * from system.clusters;
   ```
   
   
## 方案二：docker-compose 自动化
 下载本项目 Packages 目录软件包一键安装： ckcluster-docker-compose.tar.xz
 
 解压后直接启动
 ```
$ tar axvf testck.tar.xz
$ cd test-ck-cluster
# 启动ck集群（包括zk服务）
$ docker-compose up -d
# 检查是否都已经启动
$ docker-compose ps
# 检查ck集群是否创建成功（这里预配置的是perftest_3shards_1replicas）
# docker exec [OPTIONS] CONTAINER COMMAND [ARG...]
$ docker exec testck_clickhouse01_1 clickhouse-client --query "select * from system.clusters"
```
终端模式：
```
docker exec -it testck_clickhouse01_1 clickhouse-client -h 127.0.0.1 /bin/bash
```

 跑官方的例子: https://clickhouse.tech/docs/en/getting-started/example-datasets/
   
## Q&A
1、docker container 容器无法停止：

   `docker rm --force 310aedd29373`:  Error response from daemon: removal of container 310aedd29373 is already in progress
    
   `docker logs -f clickhouse-server2` 查看日志：Error response from daemon: can not get logs from container which is dead or marked for removal
   
   解决：重启 docker (Docker for Mac Desktop 直接从 Docker 菜单栏 Quit，重启后删除废弃的容器即可)

2、docker-compose down/stop CONTAINER_ID Error: device is busy
 
```
Error response from daemon: Driver devicemapper failed to remove root filesystem 5b47acf5aa48c43d617747b007cff50f7ea042b66dab3a32eeb0c86b13efa41c: Device is Busy
``` 

尝试强制终止：`docker rm --force CONTAINER` 发现可以生效，重启正常了。

具体原因待再排查。
   