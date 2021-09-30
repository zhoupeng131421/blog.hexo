---
title: OpenStack HA basic config
date: 2021-3-29
tags: [OpenStack, HA]
categories: 服务配置
---

# 1. prepare
- 虚拟机准备：

|hostname|IP|servers|comment|
|----|----|----|----|
|openstack-controller01|10.10.10.51|chrony-server<br>DB-server<br>MessageQueue<br>Memcached<br>etcd<br>pacemaker<br>HAProxy|time sync server<br>database server<br>message queue<br>object cached server<br>key-value server<br>cluster resource manager<br>loadbalance server|
|openstack-controller02|10.10.10.52|chrony-client<br>DB-server<br>MessageQueue<br>Memcached<br>etcd<br>pacemaker<br>HAProxy|time sync server<br>database server<br>message queue<br>object cached server<br>key-value server<br>cluster resource manager<br>loadbalance server|
|openstack-controller03|10.10.10.53|chrony-client<br>DB-server<br>MessageQueue<br>Memcached<br>etcd<br>pacemaker<br>HAProxy|time sync server<br>database server<br>message queue<br>object cached server<br>key-value server<br>cluster resource manager<br>loadbalance server|
|openstack-compute01|10.10.10.54|chronyc-client||

- chrony: time sync server and client
- `mariadb-galera`用于`mariadb-server`的高可用同步,部署在controller节点.
- Pacemaker 承担集群资源管理者（CRM - Cluster Resource Manager）的角色，它是一款开源的高可用资源管理软件，适合各种大小集群.

## 1.1 ansible deploy to every node
- ssh-copy-id -i root@openstack-controller01
- ssh-copy-id -i root@openstack-controller02
- ssh-copy-id -i root@openstack-controller03
- ssh-copy-id -i root@openstack-compute01
- vim prepare-openstack-base.yml:
```yaml
---
- name: prepare controller nodes
  gather_facts: no
  hosts: all
  tasks:
    - name: Set time zone
      timezone:
        name: Asia/Shanghai
    - name: Install common packages
      yum:
        name: [net-tools, vim, git, wget, curl, chrony]
        state: present
        update_cache: yes
    - name: Install openstack client
      yum:
        name: [centos-release-openstack-train]
        state: present
        update_cache: yes
    - name: Update System
      yum:
        name: "*"
        state: latest
        update_cache: yes
    - name: Enable Chrony
      shell: |
        systemctl enable chronyd
        systemctl start chronyd
    - name: Disable Firewall
      shell: |
        systemctl stop firewalld
        systemctl disable firewalld
    - name: Disable SeLinux
      lineinfile:
        dest: /etc/selinux/config
        regexp: '^SELINUX='
        line: 'SELINUX=disabled'
    - name: optimize kernel parameter
      shell: |
        modprobe br_netfilter
        echo 'net.ipv4.ip_forward = 1' >>/etc/sysctl.conf
        echo 'net.bridge.bridge-nf-call-iptables=1' >>/etc/sysctl.conf
        echo 'net.bridge.bridge-nf-call-ip6tables=1'  >>/etc/sysctl.conf
        echo 'net.ipv4.ip_nonlocal_bind = 1' >>/etc/sysctl.conf
        sysctl -p
```
- vim hosts
```shell
[openstack-controller]
openstack-controller01
openstack-controller02
openstack-controller03

[openstack-compute]
openstack-compute01
openstack-compute02
openstack-compute03
```
- `ansible-playboot -i hosts prepare-openstack-base.yml`

## 1.2 ansible deploy to controller node
- vim prepare-openstack-controller.yml
```yaml
---
- name: prepare controller nodes
  gather_facts: no
  hosts: openstack-controller
  tasks:
    - name: Install openstack client
      yum:
        name: [python-openstackclient, openstack-selinux, openstack-utils]
        state: present
        update_cache: yes
    - name: Install Mariadb, rabbitmq, memcached, etcd, pacemaker, haproxy
      yum:
        name: [mariadb, mariadb-server, python2-PyMySQL, rabbitmq-server, memcached, python-memcached, etcd, pacemaker, pcs, corosync, fence-agents, resource-agents, haproxy]
        state: present
        update_cache: yes
    - name: Enable Servers
      shell: |
        systemctl enable mariadb.service rabbitmq-server memcached etcd pcsd haproxy
        systemctl start mariadb.service rabbitmq-server memcached etcd pcsd haproxy
```

- vim prepare-openstack-controller-services.yml:
```yaml
---
- name: controller keystone
  gather_facts: no
  hosts: openstack-controller
  tasks:
    - name: Install keystone, glance
      yum:
        name: [openstack-keystone, httpd, mod_wsgi, mod_ssl, openstack-glance, python2-glance, python2-glanceclient, openstack-placement-api, openstack-nova-api, openstack-nova-conductor, openstack-nova-novncproxy, openstack-nova-scheduler]
        state: present
        update_cache: yes
    - name: Enable servers
      shell: |
        systemctl enable httpd openstack-glance-api openstack-nova-api openstack-nova-scheduler openstack-nova-conductor openstack-nova-novncproxy
        systemctl start httpd openstack-glance-api openstack-nova-api openstack-nova-scheduler openstack-nova-conductor openstack-nova-novncproxy
    - name: backup keysthone config
      shell: |
        cp /etc/keystone/keystone.conf{,.bak}
        egrep -v '^$|^#' /etc/keystone/keystone.conf.bak >/etc/keystone/keystone.conf
    - name: backup glance config
      shell: |
        cp /etc/glance/glance-api.conf{,.bak}
        egrep -v '^$|^#' /etc/glance/glance-api.conf.bak >/etc/glance/glance-api.conf
    - name: backup placement-api config
      shell: |
        cp /etc/placement/placement.conf /etc/placement/placement.conf.bak
        grep -Ev '^$|#' /etc/placement/placement.conf.bak > /etc/placement/placement.conf
    - name: backup nova config
      shell: |
        cp -a /etc/nova/nova.conf{,.bak}
        grep -Ev '^$|#' /etc/nova/nova.conf.bak > /etc/nova/nova.conf
```

- ansible-playbook -i hosts prepare-openstack-controller-keystone.yml

# 2. config mariaDB
> Galera是一个MySQL的同步多主集群软件。能够实现同步复制，Active-active的多主拓扑结构，真正的multi-master，所有节点可以同时读写数据库，自动成员资格控制，失败节点从群集中删除，新节点加入数据自动复制，真正的并行复制，行级。

## 2.1 Install and config
- each controller node: `yum install mariadb-server-galera mariadb-galera-common galera xinetd rsync -y`
- init DB password on every controller node: mysql_secure_installation
```shell
Enter current password for root (enter for none):
Set root password? [Y/n] y
Remove anonymous users? [Y/n] y
Disallow root login remotely? [Y/n] n
Remove test database and access to it? [Y/n] y
Reload privilege tables now? [Y/n] y
```
- vim /etc/my.cnf.d/openstack.cnf
```shell
# bind-address   host ip
# wsrep_node_name host name
# wsrep_node_address host ip
[server]

[mysqld]
bind-address = 10.10.10.51
max_connections = 1000
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
log-error=/var/log/mariadb/mariadb.log
pid-file=/run/mariadb/mariadb.pid

[galera]
wsrep_on=ON
wsrep_provider=/usr/lib64/galera/libgalera_smm.so
wsrep_cluster_name="mariadb_galera_cluster"

wsrep_cluster_address="gcomm://openstack-controller01,openstack-controller02,openstack-controller03"
wsrep_node_name="openstack-controller01"
wsrep_node_address="10.10.10.51"

binlog_format=ROW
default_storage_engine=InnoDB
innodb_autoinc_lock_mode=2
wsrep_slave_threads=4
innodb_flush_log_at_trx_commit=2
innodb_buffer_pool_size=1024M
wsrep_sst_method=rsync
wsrep_sync_wait=1

[embedded]

[mariadb]

[mariadb-10.3]
```
- copy config file to other controller nodes:
    - scp -rp /etc/my.cnf.d/openstack.cnf openstack-controller02:/etc/my.cnf.d/openstack.cnf 
    - scp -rp /etc/my.cnf.d/openstack.cnf openstack-controller03:/etc/my.cnf.d/openstack.cnf
    - modify the wsrep_node_name, wsrep_node_address, bind-address

## 2.2 config cluster
- stop all DB server for controller nodes:
    - `systemctl stop mariadb`
- start mariaDB server at controller01 node:
    - /usr/libexec/mysqld --wsrep-new-cluster --user=root &
- other controller nodes join the cluster:
    - systemctl start mariadb.service
- reconfig mariadb at controller01 node:
    - 重启controller01节点；并在启动前删除contrller01节点之前的数据:
    - pkill -9 mysqld
    - rm -rf /var/lib/mysql/*
    - chown mysql:mysql /var/run/mariadb/mariadb.pid
    - reboot
    - systemctl status mariadb.service
- verify:
    - controller01:
        - mysql -uroot -p
        - create database cluster_test charset utf8mb4;
        - show databases;
    - controller02:
        - mysql -uroot -p
        - show databases;
    - controller03:
        - mysql -uroot -p
        - show databases;

## 2.3 set heartbeat check: `clustercheck`
- all controller node: `wget -P /extend/shell/ https://raw.githubusercontent.com/olafz/percona-clustercheck/master/clustercheck`
- vim /extend/shell/clustercheck:
```shell
MYSQL_USERNAME="clustercheck"
MYSQL_PASSWORD="123456"
MYSQL_HOST="localhost"
MYSQL_PORT="3306"
...
```
- chmod +x /extend/shell/clustercheck
- mv /usr/bin/clustercheck /usr/bin/clustercheck.bak
- cp /extend/shell/clustercheck /usr/bin
- create clusterchecker user in database:
    - mysql -uroot -p
    - GRANT PROCESS ON *.* TO 'clustercheck'@'localhost' IDENTIFIED BY '123456';
    - flush privileges;
- create heatbeat check file:
    - apply to all controller nodes
    - touch /etc/xinetd.d/galera-monitor
```shell
cat >/etc/xinetd.d/galera-monitor <<EOF
# default:on
# description: galera-monitor
service galera-monitor
{
port = 9200
disable = no
socket_type = stream
protocol = tcp
wait = no
user = root
group = root
groups = yes
server = /usr/bin/clustercheck
type = UNLISTED
per_source = UNLIMITED
log_on_success =
log_on_failure = HOST
flags = REUSE
}
EOF
```
- start heatbeat service
    - apply to all controller nodes
    - vim /etc/services
```shell
...
#wap-wsp        9200/tcp                # WAP connectionless session service
galera-monitor  9200/tcp                # galera-monitor
```
- systemctl daemon-reload
- systemctl enable xinetd
- systemctl start xinetd

- verify:
    - /usr/bin/clustercheck

## 2.4 troubleshooting
- 当突然停电，所有galera主机都非正常关机，来电后开机，会导致galera集群服务无法正常启动。以下为处理办法
- 第1步：开启galera集群的群主主机的mariadb服务。
- 第2步：开启galera集群的成员主机的mariadb服务。
- 异常处理：galera集群的群主主机和成员主机的mysql服务无法启动，如何处理？
- #解决方法一：
    - 第1步、删除garlera群主主机的/var/lib/mysql/grastate.dat状态文件
    - /bin/galera_new_cluster启动服务。启动正常。登录并查看wsrep状态。
    - 第2步：删除galera成员主机中的/var/lib/mysql/grastate.dat状态文件
    - systemctl restart mariadb重启服务。启动正常。登录并查看wsrep状态。
- #解决方法二：
    - 第1步、修改garlera群主主机的/var/lib/mysql/grastate.dat状态文件中的0为1
    - /bin/galera_new_cluster启动服务。启动正常。登录并查看wsrep状态。
    - 第2步：修改galera成员主机中的/var/lib/mysql/grastate.dat状态文件中的0为1
    - systemctl restart mariadb重启服务。启动正常。登录并查看wsrep状态。

# 3. config RabbitMQ
- Stop the rabbitmq-server service in the controller nodes except the controller01
- send .erlang.cookie to other controller nodes
    - scp /var/lib/rabbitmq/.erlang.cookie  openstack-controller02:/var/lib/rabbitmq/           
    - scp /var/lib/rabbitmq/.erlang.cookie  openstack-controller03:/var/lib/rabbitmq/
- chang the .erlang.cookie owner at other controller nodes
    - chown rabbitmq:rabbitmq /var/lib/rabbitmq/.erlang.cookie
- start the rabbitmq-server service in the other controller nodes
    - systemctl start rabbitmq-server
-  构建集群，controller02和03节点以ram节点的形式加入集群
    - rabbitmqctl stop_app
    - rabbitmqctl join_cluster \-\-ram rabbit@openstack-controller01
    - rabbitmqctl start_app
- verify at any controller node: rabbitmqctl cluster_status
- create rabbitmq manager account
    - rabbitmqctl add_user openstack xxx
    - rabbitmqctl set_user_tags openstack administrator
    - rabbitmqctl set_permissions -p "/" openstack ".*" ".*" ".*"
    - rabbitmqctl list_users
- set ha:
    - rabbitmqctl set_policy ha-all "^" '{"ha-mode":"all"}'
    - rabbitmqctl list_policies
- install web plugin:
    - apply all controller nodes
    - rabbitmq-plugins enable rabbitmq_management
    - netstat -lntup|grep 5672
    - verify at any controller node: http://openstack-controller01:15672

# 4. memcached and etcd
## 4.1 memcached
> Memcached是一款开源、高性能、分布式内存对象缓存系统，可应用各种需要缓存的场景，其主要目的是通过降低对Database的访问来加速web应用程序。
> Memcached一般的使用场景是：通过缓存数据库查询的结果，减少数据库访问次数，以提高动态Web应用的速度、提高可扩展性。
> 本质上，memcached是一个基于内存的key-value存储，用于存储数据库调用、API调用或页面引用结果的直接数据，如字符串、对象等小块任意数据。
> Memcached是无状态的，各控制节点独立部署，openstack各服务模块统一调用多个控制节点的memcached服务即可

- 在全部安装memcached服务的节点设置服务监听本地地址
    - `sed -i 's|127.0.0.1,::1|0.0.0.0|g' /etc/sysconfig/memcached`
    - systemctl restart memcached.service
    - systemctl status memcached.service

## 4.2 etcd
> OpenStack服务可以使用Etcd，这是一种分布式可靠的键值存储，用于分布式密钥锁定、存储配置、跟踪服务生存周期和其他场景；用于共享配置和服务发现特点是，安全，具有可选客户端证书身份验证的自动TLS；快速，基准测试10,000次/秒；可靠，使用Raft正确分发。

- apply to all controller nodes:
- cp -a /etc/etcd/etcd.conf{,.bak}
- [openstack-controller01 ~]#
```shell
cat << EOF | tee /etc/etcd/etcd.conf
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="http://10.10.10.51:2380"
ETCD_LISTEN_CLIENT_URLS="http://10.10.10.51:2379,http://127.0.0.1:2379"
ETCD_NAME="controller01"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.10.10.51:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://10.10.10.51:2379,http://127.0.0.1:2379"
ETCD_INITIAL_CLUSTER="controller01=http://10.10.10.51:2380,controller02=http://10.10.10.52:2380,controller03=http://10.10.10.53:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new
EOF
```
- [openstack-controller02 ~]#
```shell
cat << EOF | tee /etc/etcd/etcd.conf
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="http://10.10.10.52:2380"
ETCD_LISTEN_CLIENT_URLS="http://10.10.10.52:2379,http://127.0.0.1:2379"
ETCD_NAME="controller02"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.10.10.52:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://10.10.10.52:2379,http://127.0.0.1:2379"
ETCD_INITIAL_CLUSTER="controller01=http://10.10.10.51:2380,controller02=http://10.10.10.52:2380,controller03=http://10.10.10.53:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new
EOF
```
- [openstack-controller03 ~]#
```shell
cat << EOF | tee /etc/etcd/etcd.conf
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="http://10.10.10.53:2380"
ETCD_LISTEN_CLIENT_URLS="http://10.10.10.53:2379,http://127.0.0.1:2379"
ETCD_NAME="controller03"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.10.10.53:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://10.10.10.53:2379,http://127.0.0.1:2379"
ETCD_INITIAL_CLUSTER="controller01=http://10.10.10.51:2380,controller02=http://10.10.10.52:2380,controller03=http://10.10.10.53:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new
EOF
```
- all controller node:
    - systemctl restart etcd
    - systemctl status etcd
- verify:
    - etcdctl cluster-health
    - etcdctl member list

# 5. pacemaker config
|server|for use|
|-----|-----|
|pacemaker|资源管理器（CRM），负责启动与停止服务，位于 HA 集群架构中资源管理、资源代理层|
|corosync|消息层组件（Messaging Layer），管理成员关系、消息与仲裁，为高可用环境中提供通讯服务，位于高可用集群架构的底层，为各节点（node）之间提供心跳信息|
|resource-agents|资源代理，在节点上接收CRM的调度，对某一资源进行管理的工具，管理工具通常为脚本|
|pcs|命令行工具集|
|fence-agents|在一个节点不稳定或无答复时将其关闭，使其不会损坏集群的其它资源，其主要作用是消除脑裂|

## 5.1 base config
- modify cluster passwd for every controller node : echo xxx | passwd --stdin hacluster
- auth nodes at any controller node : `pcs cluster auth openstack-controller01 openstack-controller02 openstack-controller03 -u hacluster -p xxx`
- create and name cluster at any controller node: `pcs cluster setup --name openstack-cluster-01 openstack-controller01 openstack-controller02 openstack-controller03`
- cluster start:
    - pcs cluster start  --all
    - pcs cluster enable  --all
- commands:
    - pcs cluster status
    - cibadmin --query --scope nodes
    - pcs status corosync
    - corosync-cmapctl | grep members
    - pcs resource
- verify: https://openstack-controller01:2224

## 5.2 HA config
> config with any controller node

- 设置合适的输入处理历史记录及策略引擎生成的错误与警告，在trouble shooting故障排查时有用
```shell
pcs property set pe-warn-series-max=1000 \
pe-input-series-max=1000 \
pe-error-series-max=1000
```
- pacemaker基于时间驱动的方式进行状态处理，cluster-recheck-interval默认定义某些pacemaker操作发生的事件间隔为15min，建议设置为5min或3min
    - pcs property set cluster-recheck-interval=5
- corosync默认启用stonith，但stonith机制（通过ipmi或ssh关闭节点）并没有配置相应的stonith设备（通过crm_verify -L -V验证配置是否正确，没有输出即正确），此时pacemaker将拒绝启动任何资源；在生产环境可根据情况灵活调整，测试环境下可关闭
    - pcs property set stonith-enabled=false
- 默认当有半数以上节点在线时，集群认为自己拥有法定人数，是“合法”的，满足公式：total_nodes < 2 * active_nodes；
    - 以3个节点的集群计算，当故障2个节点时，集群状态不满足上述公式，此时集群即非法；当集群只有2个节点时，故障1个节点集群即非法，所谓的”双节点集群”就没有意义；
    - 在实际生产环境中，做2节点集群，无法仲裁时，可选择忽略；做3节点集群，可根据对集群节点的高可用阀值灵活设置
    - pcs property set no-quorum-policy=ignore
- v2的heartbeat为了支持多节点集群，提供了一种积分策略来控制各个资源在集群中各节点之间的切换策略；通过计算出各节点的的总分数，得分最高者将成为active状态来管理某个（或某组）资源；
    - 默认每一个资源的初始分数（取全局参数default-resource-stickiness，通过"pcs property list --all"查看）是0，同时每一个资源在每次失败之后减掉的分数（取全局参数default-resource-failure-stickiness）也是0，此时一个资源不论失败多少次，heartbeat都只是执行restart操作，不会进行节点切换；
    - 如果针对某一个资源设置初始分数”resource-stickiness“或"resource-failure-stickiness"，则取单独设置的资源分数；
    - 一般来说，resource-stickiness的值都是正数，resource-failure-stickiness的值都是负数；有一个特殊值是正无穷大（INFINITY）和负无穷大（-INFINITY），即"永远不切换"与"只要失败必须切换"，是用来满足极端规则的简单配置项；
    - 如果节点的分数为负，该节点在任何情况下都不会接管资源（冷备节点）；如果某节点的分数大于当前运行该资源的节点的分数，heartbeat会做出切换动作，现在运行该资源的节点将释 放资源，分数高出的节点将接管该资源
- `pcs property list` 只可查看修改后的属性值，参数”--all”可查看含默认值的全部属性值；
    - 也可查看/var/lib/pacemaker/cib/cib.xml文件，或”pcs cluster cib”，或“cibadmin --query --scope crm_config”查看属性设置，” cibadmin --query --scope resources”查看资源配置

## 5.3 config VIP
- `ocf`（standard属性）：资源代理（resource agent）的一种，另有systemd，lsb，service等；
- `heartbeat`：资源脚本的提供者（provider属性），ocf规范允许多个供应商提供同一资源代理，大多数ocf规范提供的资源代理都使用heartbeat作为provider；
- `IPaddr2`：资源代理的名称（type属性），IPaddr2便是资源的type；
- `cidr_netmask`: 子网掩码位数
- 通过定义资源属性（`standard:provider:type`），定位vip资源对应的ra脚本位置；
- centos系统中，符合ocf规范的ra脚本位于`/usr/lib/ocf/resource.d/`目录，目录下存放了全部的provider，每个provider目录下有多个type；
- `op`：表示Operations（运作方式 监控间隔= 30s）
- `pcs resource create vip ocf:heartbeat:IPaddr2 ip=10.10.10.69 cidr_netmask=24 op monitor interval=30s`
    - verify: https://10.10.10.69:2224
- web 界面通过 `Add Existing`, 输入任意节点ip 即可添加整个集群

# 6. Haproxy
## 6.1 basic config
- apply to every controller node
- mkdir /var/log/haproxy
- chmod a+w /var/log/haproxy
- vim /etc/rsyslog.conf:
```shell
$ModLoad imudp
$UDPServerRun 514
$ModLoad imtcp
$InputTCPServerRun 514
local0.=info    -/var/log/haproxy/haproxy-info.log
local0.=err     -/var/log/haproxy/haproxy-err.log
local0.notice;local0.!=err      -/var/log/haproxy/haproxy-notice.log
```
- systemctl restart rsyslog
- cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak
- cat /etc/haproxy/haproxy.cfg
```shell
global
  log      127.0.0.1     local0
  chroot   /var/lib/haproxy
  daemon
  group    haproxy
  user     haproxy
  maxconn  4000
  pidfile  /var/run/haproxy.pid
  stats    socket /var/lib/haproxy/stats

defaults
    mode                    http
    log                     global
    maxconn                 4000    #最大连接数
    option                  httplog
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout check           10s


# haproxy监控页
listen stats
  bind 0.0.0.0:1080
  mode http
  stats enable
  stats uri /
  stats realm OpenStack\ Haproxy
  stats auth admin:admin
  stats  refresh 30s
  stats  show-node
  stats  show-legends
  stats  hide-version

# horizon服务
 listen dashboard_cluster
  bind  10.10.10.69:80
  balance  source
  option  tcpka
  option  httpchk
  option  tcplog
  server openstack-controller01 10.10.10.51:80 check inter 2000 rise 2 fall 5
  server openstack-controller02 10.10.10.52:80 check inter 2000 rise 2 fall 5
  server openstack-controller03 10.10.10.53:80 check inter 2000 rise 2 fall 5

# mariadb服务；
#设置openstack-controller01节点为master，openstack-controller02/03节点为backup，一主多备的架构可规避数据不一致性；
#另外官方示例为检测9200（心跳）端口，测试在mariadb服务宕机的情况下，虽然”/usr/bin/clustercheck”脚本已探测不到服务，但受xinetd控制的9200端口依然正常，导致haproxy始终将请求转发到mariadb服务宕机的节点，暂时修改为监听3306端口
listen galera_cluster
  bind 10.10.10.69:3306
  balance  source
  mode    tcp
  server openstack-controller01 10.10.10.51:3306 check inter 2000 rise 2 fall 5
  server openstack-controller02 10.10.10.52:3306 backup check inter 2000 rise 2 fall 5
  server openstack-controller03 10.10.10.53:3306 backup check inter 2000 rise 2 fall 5

#为rabbirmq提供ha集群访问端口，供openstack各服务访问；
#如果openstack各服务直接连接rabbitmq集群，这里可不设置rabbitmq的负载均衡
 listen rabbitmq_cluster
   bind 10.10.10.69:5673
   mode tcp
   option tcpka
   balance roundrobin
   timeout client  3h
   timeout server  3h
   option  clitcpka
   server openstack-controller01 10.10.10.51:5672 check inter 10s rise 2 fall 5
   server openstack-controller02 10.10.10.52:5672 check inter 10s rise 2 fall 5
   server openstack-controller03 10.10.10.53:5672 check inter 10s rise 2 fall 5

# glance_api服务
 listen glance_api_cluster
  bind  10.10.10.69:9292
  balance  source
  option  tcpka
  option  httpchk
  option  tcplog
  timeout client 3h 
  timeout server 3h
  server openstack-controller01 10.10.10.51:9292 check inter 2000 rise 2 fall 5
  server openstack-controller02 10.10.10.52:9292 check inter 2000 rise 2 fall 5
  server openstack-controller03 10.10.10.53:9292 check inter 2000 rise 2 fall 5

# keystone_public _api服务
 listen keystone_public_cluster
  bind 10.10.10.69:5000
  balance  source
  option  tcpka
  option  httpchk
  option  tcplog
  server openstack-controller01 10.10.10.51:5000 check inter 2000 rise 2 fall 5
  server openstack-controller02 10.10.10.52:5000 check inter 2000 rise 2 fall 5
  server openstack-controller03 10.10.10.53:5000 check inter 2000 rise 2 fall 5

 listen nova_compute_api_cluster
  bind 10.10.10.69:8774
  balance  source
  option  tcpka
  option  httpchk
  option  tcplog
  server openstack-controller01 10.10.10.51:8774 check inter 2000 rise 2 fall 5
  server openstack-controller02 10.10.10.52:8774 check inter 2000 rise 2 fall 5
  server openstack-controller03 10.10.10.53:8774 check inter 2000 rise 2 fall 5

 listen nova_placement_cluster
  bind 10.10.10.69:8778
  balance  source
  option  tcpka
  option  tcplog
  server openstack-controller01 10.10.10.51:8778 check inter 2000 rise 2 fall 5
  server openstack-controller02 10.10.10.52:8778 check inter 2000 rise 2 fall 5
  server openstack-controller03 10.10.10.53:8778 check inter 2000 rise 2 fall 5

 listen nova_metadata_api_cluster
  bind 10.10.10.69:8775
  balance  source
  option  tcpka
  option  tcplog
  server openstack-controller01 10.10.10.51:8775 check inter 2000 rise 2 fall 5
  server openstack-controller02 10.10.10.52:8775 check inter 2000 rise 2 fall 5
  server openstack-controller03 10.10.10.53:8775 check inter 2000 rise 2 fall 5

 listen nova_vncproxy_cluster
  bind 10.10.10.69:6080
  balance  source
  option  tcpka
  option  tcplog
  server openstack-controller01 10.10.10.51:6080 check inter 2000 rise 2 fall 5
  server openstack-controller02 10.10.10.52:6080 check inter 2000 rise 2 fall 5
  server openstack-controller03 10.10.10.53:6080 check inter 2000 rise 2 fall 5

 listen neutron_api_cluster
  bind 10.10.10.69:9696
  balance  source
  option  tcpka
  option  httpchk
  option  tcplog
  server openstack-controller01 10.10.10.51:9696 check inter 2000 rise 2 fall 5
  server openstack-controller02 10.10.10.52:9696 check inter 2000 rise 2 fall 5
  server openstack-controller03 10.10.10.53:9696 check inter 2000 rise 2 fall 5

 listen cinder_api_cluster
  bind 10.10.10.69:8776
  balance  source
  option  tcpka
  option  httpchk
  option  tcplog
  server openstack-controller01 10.10.10.51:8776 check inter 2000 rise 2 fall 5
  server openstack-controller02 10.10.10.52:8776 check inter 2000 rise 2 fall 5
  server openstack-controller03 10.10.10.53:8776 check inter 2000 rise 2 fall 5

 listen cinder_volume
  bind 10.10.10.69:9292
  balance  source
  option  tcpka
  option  httpchk
  option  tcplog
  server openstack-controller01 10.10.10.51:9292 check inter 2000 rise 2 fall 5
  server openstack-controller02 10.10.10.52:9292 check inter 2000 rise 2 fall 5
  server openstack-controller03 10.10.10.53:9292 check inter 2000 rise 2 fall 5
```
- scp /etc/haproxy/haproxy.cfg openstack-controller02:/etc/haproxy/haproxy.cfg
- scp /etc/haproxy/haproxy.cfg openstack-controller03:/etc/haproxy/haproxy.cfg
- modify kernel parameter (maybe it has been done):
    - echo 'net.ipv4.ip_nonlocal_bind = 1' >>/etc/sysctl.conf
    - echo "net.ipv4.ip_forward = 1" >>/etc/sysctl.conf
    - sysctl -p
- systemctl restart haproxy
- systemctl status haproxy
- verify: http://10.10.10.69:1080 admin/admin

## 6.2 set pcs reseource
- lb-haproxy-clone (any controller node):
    - pcs resource create lb-haproxy systemd:haproxy clone
    - pcs resource
- 设置资源启动顺序：pcs constraint order start vip_public then lb-haproxy-clone kind=Optional
    - `cibadmin --query --scope constraints` 可以查看资源约束配置
- 将两种资源约束在一个节点上：
    - 官方建议设置vip运行在haproxy active的节点，通过绑定lb-haproxy-clone与vip服务，所以将两种资源约束在1个节点；约束后，从资源角度看，其余暂时没有获得vip的节点的haproxy会被pcs关闭 
    - 这里一定要进行资源绑定，否则每个节点都会启动haproxy，造成访问混乱
    - pcs constraint colocation add lb-haproxy-clone with vip_public
    - pcs resource