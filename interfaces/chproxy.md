# chproxy 
仓库地址：https://github.com/Vertamedia/chproxy

ClickHouse数据库的http代理和负载均衡器，功能：

代理： 
- 可以根据用户输入将请求代理到多个不同CH集群
- 可以将输入用户映射到集群的用户，防止暴露CH集群真实用户名和密码；可以将多个不同的输入用户映射到单个CH用户。
- 可以通过HTTP和HTTPS请求CH
- 可以通过IP/IP掩码 列表限制HTTP和HTTPS请求
- 可以通过IP/IP掩码列表限制用户的访问权限
- 限制每个用户的查询超时时间，超时或取消的查询将通过KILL QUERY 强制终止；
- 限制每个用户的请求速率；
- 限制每个用户的请求并发数；
- 为每个输入用户和每个集群用户独立设置所有权限；

负载：
- 延迟执行请求，直到符合每个用户的限制；
- 为每个用户的响应设置缓存；
- 响应缓存具有内置的保护机制，可防止thundering herd（雷电群）问题（也称为 dogpile effect(狗桩效应)）；
- 使用 least load（最少负载）+ round robin（轮询）技术在副本和节点之间平均分配请求；

监控：
- 监控节点运行状况，并防止请求发送到运行状态不佳的节点上；
- 通过 Let's Encrypt 支持自动 HTTPS 证书发行和更新；
- 通过 HTTP 或 HTTPS 代理请求到每个已配置的集群；
- 代理请求到CH之前添加 remote/local地址 和 in/out用户名到request header:User-Agent，可以在 system.query_log.http_user_agent 查询到请求头的User_Agent信息
- 以 prometheus text format 公开各种有用指标；
- 向chproxy进程发送 SIGHUP 信号更新集群配置-不用重启集群；
- 易于管理和运行-只需将配置文件路径传递给 chproxy binary；
- 易于配置。

chproxy产生的原因：
- max_execution_time 对分布式表不起作用的bug
- max_concurrent_queries 只能作用于单节点

因此 chproxy 维护了两个不同的http代理，分别用户 INSERT 和 SELECT。见下面使用一节。

## 使用 
- 通常 INSERT 是从「位于少数子网」中的应用程序服务器发送的，必须限制其他子网的 INSERT；
- 将 INSERT 分散到多个可用分片中，然后直接路由到分片表而不是分布式表（有可能都路由到单一节点），减小单节点负担；
- SELECT：不同业务根据在线用户数和报告类型生成的负载可能不同，因此必须限制不同业务的负载；   
- 可以在多个不同的服务器上启动相同的代理，达到可伸缩和高可用的目的。

`chproxy` Config:
```
server:
  http:
      listen_addr: ":9090"
      allowed_networks: ["10.10.1.0/24","10.10.2.0/24"]
  https:
    listen_addr: ":443"
    autocert:
      cache_dir: "certs_dir"

users:
  - name: "insert"
    allowed_networks: ["10.10.1.0/24"]
    to_cluster: "stats-raw"
    to_user: "default"

  - name: "report"
    allowed_networks: ["10.10.2.0/24"]
    to_cluster: "stats-aggregate"
    to_user: "readonly"
    max_concurrent_queries: 6
    max_execution_time: 1m

  - name: "web"
    password: "****"
    to_cluster: "stats-raw"
    to_user: "web"
    max_concurrent_queries: 2
    max_execution_time: 30s
    requests_per_minute: 10
    deny_http: true
    allow_cors: true
    max_queue_size: 40
    max_queue_time: 25s
    cache: "shortterm"

clusters:
  - name: "stats-aggregate"
    nodes: [
      "10.10.20.1:8123",
      "10.10.20.2:8123"
    ]
    users:
    - name: "readonly"
      password: "****"

  - name: "stats-raw"
    nodes: [
     "10.10.10.1:8123",
     "10.10.10.2:8123",
     "10.10.10.3:8123",
     "10.10.10.4:8123"
    ]
    users:
      - name: "default"

      - name: "web"
        password: "****"

caches:
  - name: "shortterm"
    dir: "/path/to/cache/dir"
    max_size: 150Mb
    expire: 130s
```
配置说明：
1. server:
    - Chproxy 可接受 HTTP 和 HTTPS 协议的请求，HTTPS 必须使用自定义证书或者自动"Let's Encrypt"颁发的证书
    - allowed_networks用于限制访问chproxy的IP列表或IP掩码列表，可应用于 `http`,`https`,`metrics`,`user`,`clustetr-user` 
    
        假设通过 ClickHouse-grafana 或者 tabix 构建图表或任何其他网站访问 CH，在不信任的网站传输未加密的密码/数据是个坏主意。在这种场景下必须使用HTTPS访问集群。
2. users: 
    - 配置输入用户`name`和集群user`to_user`的映射， 可以限制单个应用的`max_concurrent_queries`。假设两个应用读取CH，CH最大并发查询数4，分摊到一个应用上就是2，即输入用户的`max_concurrent_queries`=集群最大并发查询数/应用数
    - 证书：请求chproxy必须带携带在此处被授权的证书，可通过BasicAuth或用户名/密码字符串传递证书。     
3. clusters:
    - cluster name/nodes : chproxy 可以配置多个集群，每个集群必须有 name 和 nodes/replicas 列表；
    - balance: 每个集群的请求会被均衡到集群中各个副本和节点，采用 round-robin 和 least-load 方法来负载均衡，即轮询选择最小负载的节点；
    - node priority: 一段时间请求不成功的节点会自动降低优先级，新的请求会被发送到「最小负载的副本中」「最小负载的健康节点」；
    - check healthy(故障剔除): chproxy会定期检查节点可用性，执行节点维护：不可用节点自动从集群中排除，再次可用时会再加到集群中，无需从ClickHouse配置中删除该节点。
    - kill query: 超过 `max_execution_time` 的查询会被 chproxy 自动 kill，使用 `default` 用户 kill queries，可在`kill_query_user`重写用户。
    - chproxy 配置中没有指定 `users`，默认使用无限制的default用户。
4. caching:
    - cache response: chproxy 可以配置缓存响应，可创建多个不同配置的 cache-configs
    - cache name: 为用户分配缓存名称来启用响应缓存，多个用户可共享同一缓存
    - enable/disable cache: 目前只有 SELECT 响应可使用缓存，请求参数`no_cache=1`显式禁用缓存；
    - cache_namespace：请求参数`cache_namespace`可以指定相同query使用不同的响应缓存；
    - 缓存刷新：缓存命名空间顶部可以构建即时缓存刷新，只需切换到新的命名空间即可。
5. security:
    - chproxy 代理到ClickHouse节点之前会从请求中移除所有参数(除了 users 参数和几个必须参数外)，防止不安全地修改ClickHouse配置；
    - 配置 limits, allowed_networks, password 时要小心，默认情况下，chproxy 会检测「明显的子网配置错误（"0.0.0.0/0"）」或者「使用未加密的HTTP发送密码错误」；
    - 禁用安全检查：`hack_me_please: true`，用于校验配置过程中使用。
    
## chproxy 安装
官方文档：https://github.com/Vertamedia/chproxy#how-to-install

集群：https://www.jianshu.com/p/9498fedcfee7   

#### 1. 下载二进制文件
https://github.com/Vertamedia/chproxy/releases
    
选择最新版本

下载并解压到 `～/soft` 目录下：

`tar xvaf chproxy-linux-amd64-v1.14.0.tar.gz`

新建配置文件： `~/chporxy/config.xml`
配置思路：
- 划分 INSERT 集群: 两个节点 ["localhost:8123", "localhost:8124"]
- 划分 SELECT 集群: 一个节点 ["localhost:8125"]

```
server:
  http:
      listen_addr: ":9090"
      allowed_networks: ["172.0.0.0/8"]

users:
  - name: "insert"
    to_cluster: "stats-raw"
    to_user: "default"

  - name: "report"
    to_cluster: "stats-aggregate"
    to_user: "readonly"
    max_concurrent_queries: 6
    max_execution_time: 1m

clusters:
  - name: "stats-raw"
    nodes: [
        "localhost:8123",
        "localhost:8124"
    ]
    users:
      - name: "default"

  - name: "stats-aggregate"
    nodes: [
      "localhost:8125"
    ]
    users:
      - name: "readonly"
        password: "readonly"

caches:
  - name: "shortterm"
    dir: "~/chproxy/cache/shortterm"
    max_size: 150Mb
    expire: 130s
```
 
也可以是shard副本列表:
```
clusters:
    - name: "shard1"
      replicas:
        - name: "replica1"
          nodes: ["localhost:8123"]
        - name: "replica2"
          nodes: ["localhost:8124"]
      users:
        -name: "default"
```
一个在线 sha1 加密工具：https://www.googlespeed.cn/sha/
创建 readonly 用户 with DOUBLE_SHA1_HASH：
```
echo -ne "CREATE USER readonly ON CLUSTER perftest_3shards_1replicas IDENTIFIED WITH DOUBLE_SHA1_HASH BY '9a27718297218c3757c365d357d13f49d0fa3065' " | curl 'http://localhost:8123/' --data-binary @-

# 使用 CHProxy 访问
echo 'SELECT 1' | curl 'http://localhost:9090/?user=report' --data-binary @-
```

#### 2. 启动
vim ~/chproxy/restart.sh
```
#!/bin/bash
cd $(dirname)
ps -ef | grep chproxy | head -2 | tail -1 | awk '{print $2}' | xargs kill -9
sudo -u chproxy nohup ./chproxy -config=./config/config.yml >> ./logs/chproxy.out 2>&1 &
```
创建用户和目录：
```
sudo useradd chproxy
mkdir -p ~/chproxy/logs
mkdir -p ~/chproxy/config
mkdir -p ~/chproxy/cache/shortterm
mkdir -p ~/chproxy/cache/longterm

cp ~/soft/chproxy ~/chproxy/
cp ~/chproxy/config.yml ~/chproxy/config/
cp ~/chproxy/restart.sh ~/chproxy/

sudo chown -R chproxy:chproxy ~/chproxy
```

启动: `sh ~/chproxy/restart.sh`
使用代理访问CH集群：
```
echo 'SELECT 1' | curl 'http://localhost:9092/?user=report&password=' --data-binary @-
```

#### 3. clickhouse-grafana
https://grafana.com/grafana/plugins/vertamedia-clickhouse-datasource

https://github.com/Vertamedia/clickhouse-grafana

安装
```
wget https://dl.grafana.com/oss/release/grafana-7.3.1-1.x86_64.rpm
sudo yum install grafana-7.3.1-1.x86_64.rpm
```
插件：
```
 grafana-cli plugins install vertamedia-clickhouse-datasource
```
访问：`http://localhost:3000/. `

Q&A：
1. Method Not Allowed

    原因：没有启动 chproxy

2. http connections are not allowed from 127.0.0.1

    原因：使用 localhost 访问的正确配置：`allowed_networks=[127.0.0.0/24]`

#### todo:
CHProxy 支持用户指定 shard 写本地表

