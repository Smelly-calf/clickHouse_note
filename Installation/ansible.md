https://juejin.cn/post/6844903631066513421

一、 安装 Ansible:
```
sudo yum -y install ansible
```
默认安装路径: /etc/ansible

chmod -R 777 /etc/ansible

二、配置被管理主机ssh
```
ssh-keygen -t rsa -C "your@email.com"
ssh-copy-id root@agent_host_ip # 密码
```

配置hosts 添加被管理主机
```
vim /etc/ansible/hosts

[Client]
angent_host_ip_1
angent_host_ip_2
```

测试 ansible
```
ansible Client -m ping -uroot
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

三、ansible playbook

vim test.yaml
```
- hosts: all
  remote_user: root
  vars:
    httpd_port: 80

  tasks:
  - name: install httpd
    yum: name=httpd state=present
```
部署配置：

`ansible-playbook test.yaml`
