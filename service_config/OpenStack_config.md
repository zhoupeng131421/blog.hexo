---
title: OpenStack based config
date: 2020-10-9
tags: [CentOS, OpenStack]
categories: 服务配置
---

# 各个节点基本配置
## Prepare
- close NetworkManager
``` shell
systemctl stop NetworkManager
systemctl disable NetworkManager
```
- clse firewall
```shell
systemctl stop firewarlld
systemctl disable firewalld
```
- set hostname: `hostnamectl set-hostname xx.xx.xx`
  - `vim /etc/hosts/`
    ```
    192.168.20.10 network.zhoupeng.com
    192.168.20.11 controller.zhoupeng.com
    192.168.20.12 block.zhoupeng.com
    192.168.20.13 computer.zhoupeng.com
    ```
- config static ip: `vim /etc/sysconfig/network-scripts/ifcfg-ensxxx`
``` shell
BOOTPROTO=dhcp  -->  BOOTPROTO=static
...                  IPADDR=192.168.xx.xx
...                  NATMASK=255.255.255.0
...                  ...
ONBOOT=no            ONBOOT=yes
```
## OpenStak package
- install yum-plugin-priorities, 防止高优先级软件被低优先级软件覆盖: `yum install -y yum-plugin-priorities`
- install epel 扩展 yum source: `yum install -y https://dl.fedoraproject.org/pub/epel/8/Everything/x86_64/Packages/e/epel-release-8-8.el8.noarch.rpm
`
- install OpenStack source: `yum install -y https://repos.fedorapeople.org/repos/openstack/openstack-victoria/rdo-release-victoria-0.el8.noarch.rpm`
- update: `yum upgrade`
- install openstack-selinux: `yum install -y openstack-selinux`

# Controller config
## software install
### install mariadb:
- `yum install -y mariadb mariadb-server` `pip3 install MYSQL-python`
  - CentOS8 的 MYSQL-python 安装过程中参照: https://blog.csdn.net/Jack_Roy/article/details/105255677
  - `cd /usr/xxx` and `cp configparser.py ConfigParser.py`
  - `yum install -y cmake gcc gcc-c++ mysql-devel`
  - `yum search python | grep devel` and install the correct version software
  - download https://downloads.mysql.com/archives/c-c/ SourceCode RedHat7xxx
  - `cmake mysql-connector-c-6.1.11-src/`
  - `cp /opt/mysql-connector-c/include/my_config.h /usr/include/mysql/`

### config mariadb
- `vim /etc/my.cnf.d/mariadb-server.cnf`:
  ```shell
  [mysqld]
  ...
  bind-address = 192.168.x.x
  default-storage-engine = innodb
  innodb_file_per_table
  collation-server = utf8_general_ci
  init-connect = 'SET NAMES utf8'
  character-set-server = utf8
  ```
- `systemctl start mariadb` `systemctl enable mariadb`
- init mysql db: `mysql_secure_installation`

### install Messing Server: for components communication
- 常见的消息代理软件: RabbitMQ Qpid ZeroMQ, OpenStack select `RabbitMQ`
- `yum install -y rabbitmq-server`
- `vim /etc/rabbitmq-env.cnf`
  - `NODENAME=rabbit@localhost`
- `systemctl enable rabbitmq-server` `system start rabbitmq-server`
- `rabbitmqctl change_password guest new_password` can change the account info

### install time sync server
> 其他所有节点都要用 controller 节点作为时间服务器来同步自己的时间. CentOS 不再支持 NTP, 改用 chrony.

- `sudo yum -y install chrony`
- `vim /etc/chrony.conf`
  ```shell

  ```

## config keystone server
### create authority server DB
- `mysql -u root -p`
  ``` shell
  # create DB
  CREATE DATABASE keystone;
  # create DB user who can completed control keystone database
  GRANT ALL PRIVILEGES ON keystone.* TO 'keystion'@'localhost' IDENTIFIED BY 'KEYSTONE_DBPASS';
  GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'KEYSTONE_DBPASS';
  ```
- create control token at initial config
  - `openssl rand -hex 10`, and store the token. f502fc2733bc04bb9fcb
### install openstack-keystone
- `yum install -y openstack-keystone` `pip3 install python-keystoneclient`
- `vim /etc/keystone/keystone.conf`
  ``` shell
  admin_token = f502fc2733bc04bb9fcb
  # [database]
  connection = mysql://keystone:KEYSTONE_DBPASS@controller.zhoupeng.com/keystone
  # [token]
  provider = keystone.token.providers.uuid.Provider
  driver = keystone.token.persistence.backends.sql.Token
  # [verbose] 未找到
  verbose = True
  ```
- 常见通用证书的秘钥，并限制相关文件的访问权限
    - `keystone-manage pki_setup --keystone-user keystone --keystone-group keystone`
  - `chown -R keystone:keystone /var/log/keystone`
  - `chown -R keystone:keystone /etc/keystone/ssl`
  - `chmod -R o-rwx /etc/keystone/ssl`
