# 使用 docker 安装 clickhouse
https://clickhouse.tech/docs/en/getting-started/install/

使用：https://hub.docker.com/r/yandex/clickhouse-server/

启动命令：

docker run 会拉取镜像无需单独拉取
1. `docker ps -a` 查看容器            
2. `docker rm --force some-clickhouse-server` 强制删除已启动的容器  
3. `mkdir -p $HOME/some_clickhouse_server` 创建挂载目录
4. 启动时挂载目录失败 `-v $HOME/some_clickhouse_server/conf:/etc/clickhouse-server`
   
   先 run 之后运行 
   `docker cp -a some-clickhouse-server:/etc/clickhouse-server $HOME/some_clickhouse_server/`
5. `docker run -d --name some-clickhouse-server --ulimit nofile=262144:262144 --volume=$HOME/some_clickhouse_server/clickhouse-server:/etc/clickhouse-server --volume=$HOME/some_clickhouse_server/data:/var/lib/clickhouse yandex/clickhouse-server`
   
   `docker inspect -f "{{.Mounts}}" some-clickhouse-server` 查看容器挂载目录
6. 修改配置：default SQL-driven 执行权限，config.xml -> access_control_path，users.xml -> access_management=1: enable access_management
7. `docker restart some-clickhouse-server` 重启容器服务
8. `docker exec -it some-clickhouse-server /bin/bash` 进入容器
9. `clickhouse-client` 进入交互式终端

8-9 可以使用单独的client连接服务器
```
$ docker run -it --rm --link some-clickhouse-server:clickhouse-server yandex/clickhouse-client --host clickhouse-server
```

之后可以跳转到 `Opertion/Privileges.md` 创建用户并修改权限啦！开始愉快的玩耍吧！

