分别介绍使用MacOS和Linuex编译clickhouse

https://clickhouse.com/docs/en/development/build-osx

# MacOS编译

参考：https://clickhouse.com/docs/en/development/build-osx

CLion设置:

Preferences配置 cmake 和 toolChains

![img](/Users/wangqian445/Library/Caches/com.jd.me/168b84f1-afb8-45c8-9ea1-761a042dab82.jpg)

![img](/Users/wangqian445/Library/Caches/com.jd.me/f80d4301-0599-4eb7-a752-cdb444fe89f3.jpg)



# Linux编译指定版本clickhouse

首先配置ssh
```
ssh-keygen -t rsa -C "something"
配置mac本地免密ssh：ssh-copy-id -i ~/.ssh/id_rsa.pub user@ip:port
```

开始编译ClickHouse
````
1.获取指定版本ck

https://github.com/ClickHouse/ClickHouse/releases

2.下载源码

```
git clone git@github.com:ClickHouse/ClickHouse.git
```

3. 设置git lfs?不确定是否起到决定性作用
```
// Skip smudge - We'll download binary files later in a faster batch
git lfs install --skip-smudge

// Do git clone here
git clone ...

// Fetch all the binary files in the new clone 
git lfs pull

// submodule成功后执行：Reinstate smudge
git lfs install --force
```

4.修改submodule

```shell
sed -i 's/https:\/\/github.com\//git@github.com:/g' .gitmodules
git config --global http.proxy '代理地址'
git config --global https.proxy '代理地址
```

5.编译特定版本ck
```
git checkout v22.5.2.53-stable #tag
git submodule sync
git submodule update --init
mkdir v22.5-build
cd v22.5-build
cmake ..
ninja
```

相关条目：
issue：https://github.com/ClickHouse/ClickHouse/issues/14036
安装git lfs,164我已经装了：https://gitee.com/help/articles/4235#article-header3
submodule操作：https://blog.csdn.net/guotianqing/article/details/82391665
smudge失败：https://stackoverflow.com/questions/46521122/smudge-error-error-downloading

---- 重新下载子模块，实际并没有成功 -----
删除子模块较复杂，步骤如下：
rm -rf 子模块目录 删除子模块目录及源码
vim .gitmodules 删除项目目录下.gitmodules文件中子模块相关条目
vim .git/config 删除配置项中子模块相关条目
rm .git/modules/* 删除模块下的子模块目录，每个子模块对应一个目录，注意只删除对应的子模块目录即可
git add .

添加子模块：
git submodule add git@github.com:unicode-org/icu.git contrib/icu
git submodule add git@github.com:ClickHouse/icudata.git contrib/icudata
````

删除代理
删除 ~/.gitconfig 文件的 http.proxy和https.proxy

使用docker编译 ClickHouse
```
>>>>启动docker基于ubuntu环境
docker run -it --name ckserver --net=host -v /code/clickhouse:/code/clickhouse  ubuntu:20.04 /bin/bash

# 安装clang-14
apt-get update
apt-get install git cmake ccache python3 ninja-build
apt install lsb-release wget software-properties-common
bash -c "$(wget -O - https://apt.llvm.org/llvm.sh)"

export CC=/usr/bin/clang-14
export CXX=/usr/bin/clang++-14

# 下载ClickHouse
git clone --recursive https://github.com/ClickHouse/ClickHouse.git
git submodule sync
git submodule update --init --recursive

# checkout到 v22.5.2.53-stable
git checkout v22.5.2.53-stable
mkdir build
cd build
cmake ..
ninja
exit

docker start ckserver
```

