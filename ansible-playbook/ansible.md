https://juejin.cn/post/6844903631066513421

一、 安装 Ansible:

安装 python
```
# 安装python
wget https://www.python.org/ftp/python/2.7.10/Python-2.7.10.tgz
tar xvf Python-2.7.10.tgz
# 指定安装路径
cd Python-2.7.10
./configure --prefix=/usr/local/python2
# 编译安装
make
make install
# 创建软连接
\rm -f /usr/bin/python
ln -s /usr/local/python2/bin/python /usr/bin/python
# 安装pip: 需要setuptools和pip两个包：
wget https://pypi.python.org/packages/11/b6/abcb525026a4be042b486df43905d6893fb04f05aac21c32c638e939e447/pip-9.0.1.tar.gz#md5=35f01da33009719497f01a4ba69d63c9
wget https://soft.itbulu.com/tools/setuptools-0.6c11.tar.gz
# 先安装setuptools
tar -xvf setuptools-0.6c11.tar.gz
cd setuptools-0.6c11
python setup.py build 
python setup.py install
# 安装pip
tar xvf pip-9.0.1.tar.gz
cd pip-9.0.1
python setup.py install
# 软连：
ln -s /usr/local/python2/bin/pip /usr/bin/pip
```

使用 pip 安装 ansible
```
pip download ansible==2.9.15
# 安装依赖：whl 文件用 pip install 安装，tar.gz解压后进入目录 python setup.py install 
# 安装顺序参考：https://blog.51cto.com/75368/2141015
tar zxvf pycparser-2.18.tar.gz
pip  install  asn1crypto-0.23.0-py2.py3-none-any.whl
pip  install  ipaddress-1.0.18-py2-none-any.whl
pip  install  enum34-1.1.6-py2-none-any.whl
pip  install  idna-2.6-py2.py3-none-any.whl
pip  install  cffi-1.11.2-cp27-cp27mu-manylinux1_x86_64.whl
pip  install  six-1.11.0-py2.py3-none-any.whl
tar zxvf  MarkupSafe-1.0.tar.gz
pip  install  PyNaCl-1.2.0-cp27-cp27mu-manylinux1_x86_64.whl
pip  install  cryptography-2.1.4-cp27-cp27mu-manylinux1_x86_64.whl
pip  install  bcrypt-3.1.4-cp27-cp27mu-manylinux1_x86_64.whl
pip  install  pyasn1-0.4.2-py2.py3-none-any.whl
tar -zxvf  pycrypto-2.6.1.tar.gz
pip  install  setuptools-38.2.3-py2.py3-none-any.whl
tar zxvf PyYAML-3.12.tar.gz
pip  install  Jinja2-2.8.1-py2.py3-none-any.whl 
pip  install  paramiko-2.4.0-py2.py3-none-any.whl

# 最后安装 ansible:
tar zxvf ansible-2.2.1.0.tar.gz
cd ansible-2.2.1.0/
python setup.py install
cd examples/
cp ansible.cfg hosts /etc/ansible/

# 配置环境变量(pip uninstall ansible 会提示安装路径，咳咳)
ln -s /usr/local/python2/bin/ansible /usr/bin/ansible
ln -s /usr/local/python2/bin/ansible-playbook /usr/bin/ansible-playbook
# 配置权限
sudo chmod -R 777 /usr/local/python2/lib/python2.7/site-packages/pip-20.3.3.dist-info
```
完成!

附：使用 yum 安装 ansible（没有成功过）:
```
# No module named yum
which yum
vim /bin/yum
修改 #!/usr/bin/python -> #!/usr/bin/python2.7
# yum安装ansible
sudo yum -y install ansible
```

配置文件路径: /etc/ansible

chmod -R 777 /etc/ansible

二、配置被管理主机ssh
```
ssh-keygen -t rsa -C "your@email.com"
ssh-copy-id -i ~/.ssh/id_rsa.pub root@agent_host_ip 
```

配置hosts 添加被管理主机
```
vim ipfile 

[Client]
angent_host_ip_1
angent_host_ip_2
```

测试 ansible
```
ansible -i ipfile Client -m ping -uroot
```
成功日志：
```
angent_host_ip_1 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "ping": "pong"
}
angent_host_ip_2 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "ping": "pong"
- hosts: all
}
```

三、ansible-playbook 部署集群

vim ansible.sh
```
#!/bin/bash
CLUSTER=ZYX_CK_TS_WQ
CONFDIR="/data4/home/wangqian/chproject/jdolap-deploy"

IPFILE=$1

cat > test.yaml << EOF
---
- hosts: all
  gather_facts: no
  become: true
  remote_user: root

  tasks:
  - name: copy ClickHouse config
    copy:
        src: $CONFDIR/jdolap-clickhouse/$CLUSTER
        dest: /data0/jdolap/clickhouse/$CLUSTER
        owner: root
        group: root
        mode: '0755'
  - name: start clickhouse service
    shell: /bin/bash /data0/jdolap-clickhouse/Sre/make.sh config.sh $CLUSTER
EOF

        cat test.yaml
        ansible-playbook test.yaml -i ${IPFILE} -f 10
        # \rm -rf test.yaml
```
———————————ipfile————————
```
[Server]
host1
host2
host3
```

———————————命令
执行：
```
chmod a+x ansible.sh
./ansible.sh /data4/home/wangqian/ipfile
```

vim main.go
```
package main
import (
        "os/exec"
        "fmt"
        "log"
)

func main(){
        out, err:=exec.Command("./ansible.sh", "/data4/home/wangqian/ipfile").Output()
        if err!=nil{
                log.Fatalf("cmd.Run err: %v",err)
        }
        fmt.Println(string(out))
}
```

go run main.go

