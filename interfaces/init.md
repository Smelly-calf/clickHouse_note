# ClickHouse 连接
## 1. Command-line client
从官网下载 https://repo.clickhouse.tech/tgz/stable/ 同时下载 clickhouse-client-20.6.6.7.tgz，clickhouse-common-static-20.6.6.7.tgz

启动：
```
# 用http协议获取所有ck实例列表（账号、密码、集群域名请按实际替换，下同）
curl -sSL "http://$user:$pwd@httpDomain:$httpPort" -d "SELECT host_address, port FROM system.clusters WHERE cluster != 'system_cluster'"
 
# 下载并安装
mkdir -p ~/.local/bin && wget -O- http://storage.jd.local/jdolap-deploy/clickhouse-20.6.6.7.tar.xz | tar -x --xz -C ~/.local/bin -f - && echo "export PATH=\$PATH:\$HOME/.local/bin" >> ~/.bashrc && source ~/.bashrc
 
# 随机选一个ck实例，用clickhouse-client连接（IP、端口、账号、密码按实际替换）
clickhouse client -m -h $ip --port 9600 --user $user --password $pwd 
# 添加参数 --send_logs_level=trace 可以在客户端看到详细日志，一般情况下可以不加，性能调优时可以加上
```

进入命令行客户端: 可以在命令行添加options/配置文件指定参数。

关于 clickhouse-client Code: 210. DB::NetException: Connection refused (localhost:9000) 问题：
- config.xml 配置 `<listen_host>::</listen_host>`

两种模式：
- 交互和 Batch mode 两种模式
- Batch mode 模式
    - 类似于HTTP请求，包含 "query" 参数 + 换行符 + 标准输入 "stdin"，这对于大型INSERT查询很方便
    - 默认数据格式是 `TabSeparated`, 使用 `FORMAT` 指定query格式;
    - 默认单个查询，`--multiquery` 执行除了 INSERT 之外的多查询,
- 交互模式
    - 指定 `multiline` 可多行输入，换行之前输入反斜杠 `\` , 使用分号结束查询
    - 结尾加 `\G` 符号表示 Vertical 格式输出: 每个value占一行，对于宽表很方便，为了与MySQL CLI兼容
    - 命令行基于 `readline`, 历史记录保存在 `~/.clickhouse-client-history`
    - 默认格式为`PrettyCompact`,使用`--formt|--vertical`参数，或使用客户端配置文件
    - 退出命令行：Ctrl+D (or Ctrl+C) 或 “exit”, “quit”, “logout”, “exit;”, “quit;”, “logout;”, “q”, “Q”, “:q”
    - 展示：1. 处理进度(更新频率最多10次/秒)；2.格式化后的查询语句; 3.特定格式输出结果; 4.结果行数、消耗时间、平均处理速度
    
创建参数
- `--param_parName=""` 定义参数，并参数值通过 `{<name>:<data type>}` 传递到query语句中
- `clickhouse-client --param_tuple_in_tuple="(10, ('dt', 10))" -q "SELECT * FROM table WHERE val = {tuple_in_tuple:Tuple(UInt8, Tuple(String, UInt8))}"`

#### Configuring
使用配置的顺序:
- 命令行参数｜命令行`--config-file`指定的配置文件
- 默认配置文件中的值

#### Command Line Options 
- `--host`, -h -– The server name`, ‘localhost’ by default. You can use either the name or the IPv4 or IPv6 address.
- `--port` – The port to connect to. Default value: 9000. Note that the HTTP interface and the native interface use different ports.
- `--user`, -u – The username. Default value: default.
- `--password` – The password. Default value: empty string.
- `--query`, -q – The query to process when using non-interactive mode.
- `--database`, -d – Select the current default database. Default value: the current database from the server settings (‘default’ by default).
- `--multiline`, -m – If specified`, allow multiline queries (do not send the query on Enter).
- `--multiquery`, -n – If specified`, allow processing multiple queries separated by semicolons.
- `--format`, -f – Use the specified default format to output the result.
- `--vertical`, -E – If specified`, use the Vertical format by default to output the result. This is the same as ‘–format=Vertical’. In this format`, each value is printed on a separate line`, which is helpful when displaying wide tables.
- `--time`, -t – If specified`, print the query execution time to ‘stderr’ in non-interactive mode.
- `--stacktrace – If specified`, also print the stack trace if an exception occurs.
- `--config-file` – The name of the configuration file.
- `--secure – If specified`, will connect to server over secure connection.
- `--history_file` — Path to a file containing command history.
- `--param_<name>` — Value for a query with parameters.

#### Configuration Files：
clickhouse-client uses the first existing file of the following:

Defined in the --config-file parameter.
- ./clickhouse-client.xml
- ~/.clickhouse-client/config.xml
- /etc/clickhouse-client/config.xml

Example of a config file:
```
<config>
    <user>username</user>
    <password>password</password>
    <secure>False</secure>
</config>    
```

## 2. HTTP 接口 , 文档性的且易于直接使用
- 任何语言任何平台都可以通过 HTTP 访问 ClickHouse
- ClickHouse默认的HTTP监听端口：<em>8123</em>
- `curl 'http://localhost:8123/'` 返回 OK, 见配置`<http_server_default_response>`
- 健康检查接口：` curl 'http://localhost:8123/ping' `

#### 使用 HTTP 请求 CH Server
- URL 大小不超过16KB
- 默认format:`TabSeparated`，指定Format: URL参数`default_format`或Header:`X-ClickHouse-Format`
- GET 请求：URL参数query发送查询语句，GET请求是 readonly, 修改数据只能用POST. `curl 'http://localhost:8123/?query=SELECT%201'`
- POST 请求：query使用URL参数或者 body `echo 'SELECT 1' | curl 'http://localhost:8123/' --data-binary @-`
    - --data-binary key=value
    
        指定请求中的数据为纯二进制数据
        value如果是@file_name，则保留文件中的回车符和换行符，不做任何转换
- 压缩：压缩程序`clickhouse-compressor`
    - 开启CH compression压缩：` http_native_compression_disable_checksumming_on_decompress`
        - URL参数 `compress=1`，Server端压缩返回的数据
        - URL参数 `decompress=1`，Server端解压缩POST请求中被压缩的数据
    - 开启 HTTP 压缩: `enable_http_compression`
        - Header:`Content-Encoding: compression_method` 发送压缩后的POST数据；
        - `Accept-Encoding: compression_method` 返回压缩后的数据；
        - CH支持的压缩格式:gzip,deflate
        - 压缩等级：`http_zlib_compression_level`
    - Example
        ```
      #Sending data to the server:
      $ curl -vsS "http://localhost:8123/?enable_http_compression=1" -d 'SELECT number FROM system.numbers LIMIT 10' -H 'Accept-Encoding: gzip'
      
      #Sending data to the client:
      $ echo "SELECT 1" | gzip -c | curl -sS --data-binary @- -H 'Content-Encoding: gzip' 'http://localhost:8123/'
      ```
    - 一些HTTP客户端默认会解压缩来自服务端的数据
- database：URL中指定database或Header:`X-ClickHouse-Database` 指定 database
- user/password:
    1. `$ echo 'SELECT 1' | curl 'http://user:password@localhost:8123/' -d @-`
    2. `echo 'SELECT 1' | curl 'http://localhost:8123/?user=user&password=password' -d @-`
    3. `echo 'SELECT 1' | curl -H 'X-ClickHouse-User: user' -H 'X-ClickHouse-Key: password' 'http://localhost:8123/' -d @-`
- profile: `curl http://localhost:8123/?profile=web&max_rows_to_read=1000000000&query=SELECT+1`
- 其他参数：参见 SET 小节
   - 本机 TCP 接口, 更少的开销
        - Command-line CLient、服务器之间、C++程序的接口
        - 暂无正式规范，可从源码进行反向工程或拦截分析TCP流量获取
- session：参数：`session_id`，可以是任何字符串
    - session_timeout: 默认一个Session保持60s终止，在服务端配置`default_session_timeout`修改，或GET参数`session_timeout`显式指定  
    - session_check: 参数 `session_check=1` 开启session状态检查
    - 一个query同时只能在一个session被执行
- 查询进度：参数：`X-ClickHouse-Progress`，接收查询进度信息，需要在服务端开启 `send_progress_in_http_headers`
- query_id: 使用`query_id`作为查询ID传递，避免HTTP查询丢失；The section “Settings, replace_running_query”.
- quota_key: quota密钥；The section “Quotas”.
- 外部数据/临时表: HTTP接口允许传递外部数据/外部临时表；The section “External data for query processing”.

#### 响应缓冲
可以在服务端对响应进行缓冲
- `buffer_size`: 服务器内存 缓存字节数
- `wait_end_of_query=1`: 开启wait_end_of_query后，超过缓存的部分会被缓存到服务器临时文件中
使用缓冲可避免处理query过程中的错误，错误会写在响应正文的末尾，客户端只有在解析时检测到错误。
 
#### 预定义 HTTP 接口
可以在服务端预定义 HTTP 接口。

在`http_handlers`小节中进行配置:
```
<http_handlers>
    <rule>
        <url>/predefined_query</url>
        <methods>POST,GET</methods>
        <handler>
            <type>predefined_query_handler</type>
            <query>SELECT * FROM system.metrics LIMIT 5 FORMAT Template SETTINGS format_template_resultset = 'prometheus_template_output_format_resultset', format_template_row = 'prometheus_template_output_format_row', format_template_rows_between_delimiter = '\n'</query>
        </handler>
    </rule>
    <rule>...</rule>
    <rule>...</rule>
</http_handlers>
```
使用 Predefined HTTP Interface 可以很容易和第三方工具如 Prometheus exporter 结合

请求 URL: `curl -v 'http://localhost:8123/predefined_query'`

`rule` can configure `method`, `headers`, `url`, `handler`

有三种类型的 Handler：
- `predefined_query_handler`：在handler配置中指定 `<query></query>`
- `dynamic_query_handler`: 在handler配置指定query参数`<query_param_name>`，在url中使用相应的参数动态输入查询语句
- `static`: 返回 `status`,`content_type`,`response_content`
    - `response_content` 可以是一项配置、绝对路径的文件、或者相对路径的文件

## MySQL 接口
CH支持MySQL有线协议，可通过配置文件中的`<mysql_port>`启用。
```
<mysql_port>9004</mysql_port>
```
使用命令行工具`mysql`连接：
```
mysql --protocol tcp -u default -P 9004
```
使用double SHA1 密码兼容 MySQL 客户端连接，其他密码可能无法通过验证。
   
## 第三方接口
- [chproxy](chproxy.md)
- [Grafana](Grafana.md)
 
      
