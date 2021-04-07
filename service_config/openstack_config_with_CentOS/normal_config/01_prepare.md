---
title: OpenStack 1 prepare
date: 2020-10-21
tags: [OpenStack]
categories: 服务配置
---

## prepare
- apply to every node
- `export http_porxy=http://x.x.x.x:xx`
- `yum update`
- `yum install net-tools`
- `yum install vim`
- disabled firewalld: `systemctl stop firewalld`, `systemctl disable firewalld`
- disabled selinux:
```shell
getenforce
setenforce 0
vim /etc/selinux/config
    SELINUX=disabled
```
- set time zone: `cp -f /usr/share/zoneinfo/Asia/Shanghai /etc/localtime`

## network
### IP
- `vim /etc/sysconfig/network-scrips/ifcfg-ensxxx.`
```shelll
BOOTPROTO=static
IPADDR=192.168.10.10
NETMASK=255.255.255.0
GATEWAY=192.168.10.1
ONBOOT=yes
```
### hostname
- `hostnamectl set-hostname controller`
- `vim /etc/hosts`
```shell
192.168.1.40 network
192.168.1.41 controller
192.168.1.42 block
192.168.1.43 compute1
192.168.1.45 object
```
### NTP(network time protocol)
- controller node:
    - `yum install chrony`
    - `vim /etc/chrony.conf`: `server controller iburst` `allow 192.168.1.0/24`
    - `systemctl enable chronyd.service`
    - `systemctl start chronyd.service`
- other nodes:
    - `yum install chrony`
    - `vim /etc/chrony.conf`: `server controller iburst`
    - `systemctl enable chronyd.service`
    - `systemctl start chronyd.service`
- verify
    - `chronyc sources`

## install openstack client
- apply every node
- `yum install centos-release-openstack-stein`
- `yum upgrade`
- controller node:
    - `yum install python-openstackclient`
    - `yum install openstack-selinux`

## SQL database
- the database typically runs on the controller node
- `yum install mariadb mariadb-server python2-PyMySQL`
- `vim /etc/my.cnf.d/openstack.cnf`
```shell
[mysqld]
bind-address = 192.168.1.xx

default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
```
- `systemctl enable mariadb.service`
- `systemctl start mariadb.service`
- `mysql_secure_installation`

## Message queue
- The message queue service typically runs on the controller node
- `yum install rabbitmq-server`
- `systemctl enable rabbitmq-server.service`
- `systemctl start rabbitmq-server.service`
- add openstack user: `rabbitmqctl add_user openstack RABBIT_PASS`
- access for openstack user: `rabbitmqctl set_permissions openstack ".*" ".*" ".*"`

## memcached
- The memcached service typically runs on the controller node.
- `yum install memcached python-memcached`
- vim /etc/sysconfig/memcached: `OPTIONS="-l 127.0.0.1,::1,controller"`
- `systemctl enable memcached.service`
- `systemctl start memcached.service`

## etcd
- The etcd service runs on the controller node.
- `yum install etcd`
- `vim /etc/etcd/etcd.conf`
```shell
#[Member]
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="http://10.0.0.11:2380"
ETCD_LISTEN_CLIENT_URLS="http://10.0.0.11:2379"
ETCD_NAME="controller"
#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.0.0.11:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://10.0.0.11:2379"
ETCD_INITIAL_CLUSTER="controller=http://10.0.0.11:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-01"
ETCD_INITIAL_CLUSTER_STATE="new"
```
- `systemctl enable etcd`
- `systemctl start etcd`