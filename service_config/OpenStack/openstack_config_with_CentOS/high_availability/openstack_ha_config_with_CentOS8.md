---
title: OpenStack HA config with CentOS8
date: 2021-9-16
tags: [OpenStack, HA]
categories: 服务配置
---
[toc]

# 一. structure
- hosts:

|HostName|IP|Services|
|---|---|---|
|openstack-controller01|10.10.10.51|
|openstack-controller02|10.10.10.52|
|openstack-controller03|10.10.10.53|
- user and password

|service|web|username|password|
|---|---|---|---|
|host||root|zp@131421|
|mysql||root|zp@131421|
|mariadb clustercheck||clustercheck|123456|
|rabbitmq|http://10.10.10.51:15672|openstack|zp@131421|
|pacemaker|https://10.10.10.51:2224|hacluster|zp@131421|
|pacemaker HA|https://10.10.10.69:2224|hacluster|zp@131421|
|haproxy|http://10.10.10.69:1080|admin|admin|
|keystone||admin|w1zlcjmK|
|keystone||demo|123456|
|glance||glance|56KtIfqc|
|placement||placement|58PGTpgC|
|nova||nova|7Pbjqyyf|
|neutron||neutron|6ZKOyWXt|
|horazion|||GzDgYC2m|
|cinder|||PGZoNz7H|
## CentOS8 prepare
- set root password: sudo passwd
- permit root account via ssh: /etc/ssh/sshd_config: PermitRootLogin yes
- vim /etc/resolv.conf: nameserver 10.10.10.1
- ssh-copy-id -i root@openstack-controller01
- ssh-copy-id -i root@openstack-controller02
- ssh-copy-id -i root@openstack-controller03
- ssh-copy-id -i root@openstack-compute01
- ssh-copy-id -i root@openstack-compute01

- yum install -y net-tools vim git wget curl

# 二. controller node basic config
## ansible deploy basic apps
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
        name: [centos-release-openstack-victoria]
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
```
- NTP (all nodes)
    - yum install chrony
    - openstack-controller01, vim /etc/chrony.conf: allow 10.10.10.0/24
    - all nodes except openstack-controller01, vim /etc/chrony.conf: server openstack-controller01 iburst
    - systemctl enable chronyd.service
    - systemctl start chronyd.service
    - verify: chronyc sources
- firewall and seLinux:
    - systemctl stop firewalld
    - systemctl disable firewalld
    - sed -i 's/enforcing/disabled/' /etc/selinux/config
    - setenforce 0
- openstack package:
    - (all nodes) yum install centos-release-openstack-victoria
    - yum install yum-utils -y 
    - (controller node)yum config-manager --set-enabled ha
    - yum config-manager --set-enabled powertools
    - rpm -Uvh https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
    - yum clean all
    - yum makecache
    - #yum upgrade -y
    - yum install python3-openstackclient -y
    - openstack-utils:
        - mkdir -p /opt/tools
        - yum install crudini -y
        - wget -P /opt/tools https://cbs.centos.org/kojifiles/packages/openstack-utils/2017.1/1.el7/noarch/openstack-utils-2017.1-1.el7.noarch.rpm
        - rpm -ivh /opt/tools/openstack-utils-2017.1-1.el7.noarch.rpm
    - openstack services: yum install -y mariadb-server python2-PyMySQL rabbitmq-server memcached python3-memcached etcd
    - (all nodes) haproxy and pacemarker: yum install -y pacemaker pcs corosync fence-agents resource-agents haproxy
        - https://www.geekpills.com/operating-system/linux/pacemake-high-availability-cluster-on-centos-rhel-8
## mariadb ha:
> Galera是一个MySQL的同步多主集群软件。能够实现同步复制，Active-active的多主拓扑结构，真正的multi-master，所有节点可以同时读写数据库，自动成员资格控制，失败节点从群集中删除，新节点加入数据自动复制，真正的并行复制，行级。

### Install and config
- (all controller nodes)
    - yum install mariadb mariadb-server python2-PyMySQL -y
    - yum install mariadb-server-galera mariadb-galera-common galera xinetd rsync -y
```shell
yum remove mariadb mariadb-client mariadb-server python2-PyMySQL mariadb-server-galera mariadb-galera-common galera rsync -y
rm -rf /etc/my.sql.d/ /var/lib/mysql/
yum install -y mariadb mariadb-server python2-PyMySQL mariadb-server-galera mariadb-galera-common galera rsync
```
        - systemctl restart mariadb.service
        - systemctl enable mariadb.service
    - init the database:
        - mysql_secure_installation
- vim /etc/my.cnf.d/openstack.cnf
```shell
#bind-address   主机ip
#wsrep_node_name 主机名
#wsrep_node_address 主机ip

[root@controller01 ~]# cat /etc/my.cnf.d/openstack.cnf
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
[embedded]

[mariadb]

[mariadb-10.3]
```
- scp -rp /etc/my.cnf.d/openstack.cnf controller02:/etc/my.cnf.d/openstack.cnf 
- scp -rp /etc/my.cnf.d/openstack.cnf controller03:/etc/my.cnf.d/openstack.cnf
modify the bind-address wsrep_node_name and wsrep_node_address according to real info

### config cluster
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
    - systemctl start mariadb.service
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
        - drop database cluster_test;

### set heartbeat check: clustercheck
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

### troubleshooting
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
- error:
```shell
2021-09-26 14:50:50 0 [ERROR] mysqld: Can't create/write to file '/run/mariadb/mariadb.pid' (Errcode: 13 "Permission denied")
2021-09-26 14:50:50 0 [ERROR] Can't start server: can't create PID file: Permission denied
```
    - 删除 /run/mariadb/mariadb.pid
- error:
```shell
2021-09-26 14:50:11 0 [Warning] WSREP: error while trying to listen 'tcp://0.0.0.0:4567?socket.non_blocking=1', asio error 'bind: Address already in use'
2021-09-26 14:50:11 0 [ERROR] WSREP: failed to open gcomm backend connection: 98: error while trying to listen 'tcp://0.0.0.0:4567?socket.non_blocking=1', asio error 'bind: Address already in use': 98 (Address already in use)
         at gcomm/src/asio_tcp.cpp:listen():931
2021-09-26 14:50:11 0 [ERROR] WSREP: gcs/src/gcs_core.cpp:gcs_core_open():209: Failed to open backend connection: -98 (Address already in use)
2021-09-26 14:50:11 0 [ERROR] WSREP: gcs/src/gcs.cpp:gcs_open():1475: Failed to open channel 'mariadb_galera_cluster' at 'gcomm://openstack-controller01,openstack-controller02,openstack-controller03': -98 (Address already in use)
2021-09-26 14:50:11 0 [ERROR] WSREP: gcs connect failed: Address already in use
2021-09-26 14:50:11 0 [ERROR] WSREP: wsrep::connect(gcomm://openstack-controller01,openstack-controller02,openstack-controller03) failed: 7
2021-09-26 14:50:11 0 [ERROR] Aborting
```
    - ps -ef | grep sql
    - kill -9 xxxx

## rabbitmq config
- - Stop the rabbitmq-server service in the controller nodes except the controller01
- send .erlang.cookie to other controller nodes
    - scp /var/lib/rabbitmq/.erlang.cookie  openstack-controller02:/var/lib/rabbitmq/
    - scp /var/lib/rabbitmq/.erlang.cookie  openstack-controller03:/var/lib/rabbitmq/
- chang the .erlang.cookie owner at other controller nodes
    - chown rabbitmq:rabbitmq /var/lib/rabbitmq/.erlang.cookie
- start the rabbitmq-server service in the other controller nodes
    - systemctl start rabbitmq-server
-  构建集群，controller02和03节点以ram节点的形式加入集群
    - rabbitmqctl stop_app
    - rabbitmqctl join_cluster --ram rabbit@openstack-controller01
    - rabbitmqctl start_app
- verify at any controller node: rabbitmqctl cluster_status
- create rabbitmq manager account
    - rabbitmqctl add_user openstack zp@131421
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

## memcached and ETCD
### memcached
- 在全部安装memcached服务的节点设置服务监听本地地址
    - `sed -i 's|127.0.0.1,::1|0.0.0.0|g' /etc/sysconfig/memcached`
    - systemctl restart memcached.service
    - systemctl status memcached.service
### ETCD
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
ETCD_INITIAL_CLUSTER_STATE="new"
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
ETCD_INITIAL_CLUSTER_STATE="new"
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
ETCD_INITIAL_CLUSTER_STATE="new"
EOF
```
- modify etcd config at all node, vim /usr/lib/systemd/system/etcd.service:
```shell
[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
EnvironmentFile=-/etc/etcd/etcd.conf
User=etcd
# set GOMAXPROCS to number of processors
ExecStart=/bin/bash -c "GOMAXPROCS=$(nproc) /usr/bin/etcd \
--name=\"${ETCD_NAME}\" \
--data-dir=\"${ETCD_DATA_DIR}\" \
--listen-peer-urls=\"${ETCD_LISTEN_PEER_URLS}\" \
--listen-client-urls=\"${ETCD_LISTEN_CLIENT_URLS}\" \
--initial-advertise-peer-urls=\"${ETCD_INITIAL_ADVERTISE_PEER_URLS}\" \
--advertise-client-urls=\"${ETCD_ADVERTISE_CLIENT_URLS}\" \
--initial-cluster=\"${ETCD_INITIAL_CLUSTER}\"  \
--initial-cluster-token=\"${ETCD_INITIAL_CLUSTER_TOKEN}\" \
--initial-cluster-state=\"${ETCD_INITIAL_CLUSTER_STATE}\""
Restart=on-failure
LimitNOFILE=65536
```
- all controller node:
    - systemctl daemon-reload
    - systemctl restart etcd
    - systemctl status etcd
- verify:
    - etcdctl cluster-health
    - etcdctl member list

## pacemaker config
### basic config
- modify cluster passwd for every controller node : echo xxx | passwd --stdin hacluster
- systemctl enable pcsd
- systemctl start pcsd
- any controller node:
  - pcs host auth openstack-controller01 openstack-controller02 openstack-controller03 -u hacluster -p zp@131421
  - pcs cluster setup openstack-cluster-01 --start openstack-controller01 openstack-controller02 openstack-controller03
- pcs cluster start  --all
- pcs cluster enable  --all
- cluster status: pcs cluster status
- corosync status: pcs status corosync
- node info: corosync-cmapctl | grep members
- resource info: pcs resource
- verify: https://openstack-controller01:2224
### pacemaker ha
- env: any controller node
```shell
pcs property set pe-warn-series-max=1000 pe-input-series-max=1000 pe-error-series-max=1000
pcs property set cluster-recheck-interval=5
pcs property set stonith-enabled=false
pcs property set no-quorum-policy=ignore
pcs property list
```
### pacemaker vip
- [root@controller01 ~]# pcs resource create vip ocf:heartbeat:IPaddr2 ip=10.10.10.69 cidr_netmask=24 op monitor interval=30s
- web 界面通过 `Add Existing`, 输入任意节点ip 即可添加整个集群

## Haproxy
### basic config
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

### set pcs reseource
- lb-haproxy-clone (any controller node):
    - pcs resource create lb-haproxy systemd:haproxy clone
    - pcs resource
- 设置资源启动顺序：pcs constraint order start vip then lb-haproxy-clone kind=Optional
    - `cibadmin --query --scope constraints` 可以查看资源约束配置
- 将两种资源约束在一个节点上：
    - 官方建议设置vip运行在haproxy active的节点，通过绑定lb-haproxy-clone与vip服务，所以将两种资源约束在1个节点；约束后，从资源角度看，其余暂时没有获得vip的节点的haproxy会被pcs关闭 
    - 这里一定要进行资源绑定，否则每个节点都会启动haproxy，造成访问混乱
    - pcs constraint colocation add lb-haproxy-clone with vip
    - pcs resource

# 三. Keystone
## prepare
- mysql -uroot -pxxx:
```shell
CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'w1zlcjmK';
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'w1zlcjmK';
flush privileges;
```
- yum install openstack-keystone httpd python3-mod_wsgi mod_ssl -y
## config
- cp /etc/keystone/keystone.conf{,.bak}
- egrep -v '^$|^#' /etc/keystone/keystone.conf.bak >/etc/keystone/keystone.conf
- config:
```shell
openstack-config --set /etc/keystone/keystone.conf cache backend oslo_cache.memcache_pool
openstack-config --set /etc/keystone/keystone.conf cache enabled true
openstack-config --set /etc/keystone/keystone.conf cache memcache_servers openstack-controller01:11211,openstack-controller02:11211,openstack-controller03:11211
openstack-config --set /etc/keystone/keystone.conf database connection mysql+pymysql://keystone:w1zlcjmK@10.10.10.69/keystone
openstack-config --set /etc/keystone/keystone.conf token provider fernet
```
- scp -rp /etc/keystone/keystone.conf openstack-controller02:/etc/keystone/keystone.conf
- scp -rp /etc/keystone/keystone.conf openstack-controller03:/etc/keystone/keystone.conf
- sync keystone database:
    - infill database: `su -s /bin/sh -c "keystone-manage db_sync" keystone`
    - verify:`mysql -uroot -p  keystone  -e "show  tables";`
- init Fernet secret key:
    - `keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone`
    - `keystone-manage credential_setup --keystone-user keystone --keystone-group keystone`
    - `scp -rp /etc/keystone/fernet-keys /etc/keystone/credential-keys openstack-controller02:/etc/keystone/`
    - `scp -rp /etc/keystone/fernet-keys /etc/keystone/credential-keys openstack-controller03:/etc/keystone/`
    - [controller02 | 03~ ]# `chown -R keystone:keystone /etc/keystone/credential-keys/`
    - [controller02 | 03~ ]# `chown -R keystone:keystone /etc/keystone/fernet-keys/ `
## init
- bootstrap: 
```shell
keystone-manage bootstrap --bootstrap-password w1zlcjmK \
    --bootstrap-admin-url http://10.10.10.69:5000/v3/ \
    --bootstrap-internal-url http://10.10.10.69:5000/v3/ \
    --bootstrap-public-url http://10.10.10.69:5000/v3/ \
    --bootstrap-region-id RegionOne
```
## further config
- cp /etc/httpd/conf/httpd.conf{,.bak}
- sed -i "s/#ServerName www.fzLab.siemens.net:80/ServerName ${HOSTNAME}/" /etc/httpd/conf/httpd.conf
- 不同的节点替换不同的ip地址
    - sed -i "s/Listen\ 80/Listen\ 10.10.10.51:80/g" /etc/httpd/conf/httpd.conf
    - sed -i "s/Listen\ 80/Listen\ 10.10.10.52:80/g" /etc/httpd/conf/httpd.conf
    - sed -i "s/Listen\ 80/Listen\ 10.10.10.53:80/g" /etc/httpd/conf/httpd.conf
- all controller node:
```shell
ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/
##controller01
sed -i "s/Listen\ 5000/Listen\ 10.10.10.51:5000/g" /etc/httpd/conf.d/wsgi-keystone.conf
sed -i "s#*:5000#10.10.10.51:5000#g" /etc/httpd/conf.d/wsgi-keystone.conf
##controller02
sed -i "s/Listen\ 5000/Listen\ 10.10.10.52:5000/g" /etc/httpd/conf.d/wsgi-keystone.conf
sed -i "s#*:5000#10.10.10.52:5000#g" /etc/httpd/conf.d/wsgi-keystone.conf
##controller03
sed -i "s/Listen\ 5000/Listen\ 10.10.10.53:5000/g" /etc/httpd/conf.d/wsgi-keystone.conf
sed -i "s#*:5000#10.10.10.53:5000#g" /etc/httpd/conf.d/wsgi-keystone.conf

systemctl restart httpd.service
systemctl enable httpd.service
systemctl status httpd.service
```
- config user script:
```shell
cat >> ~/admin-openrc << EOF
#admin-openrc
export OS_USERNAME=admin
export OS_PASSWORD=w1zlcjmK
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://10.10.10.69:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
EOF
```
## create domain project user role
- demo project:
```shell
openstack domain create --description "An Example Domain" example
openstack project create --domain default --description "demo Project" demo
openstack user create --domain default --password 123456 demo
openstack role create user
openstack role list
openstack role add --project demo --user  demo user
```
- demo-openrc:
```shell
cat >> ~/demo-openrc << EOF
#demo-openrc
export OS_USERNAME=demo
export OS_PASSWORD=123456
export OS_PROJECT_NAME=
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://10.10.10.69:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
EOF
```
- source  ~/demo-openrc
- scp -rp ~/demo-openrc openstack-controller02:~/
- scp -rp ~/demo-openrc openstack-controller03:~/
- verify: openstack token issue
- verify keystone:
    - admin user request the authentication token:
```shell
source admin-openrc
openstack --os-auth-url http://10.10.10.69:5000/v3 \
  --os-project-domain-name Default --os-user-domain-name Default \
  --os-project-name admin --os-username admin token issue
```
    - demo user request the authentication token:
```shell
source demo-openrc
openstack --os-auth-url http://10.10.10.69:5000/v3 \
  --os-project-domain-name Default --os-user-domain-name Default \
  --os-project-name demo --os-username demo token issue
```

## set pcs source
- pcs resource create openstack-keystone systemd:httpd clone interleave=true
- pcs resource

# 四. glance
## prepare
- create database for glance:
```shell
mysql -u root -p
CREATE DATABASE glance;
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY '56KtIfqc';
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY '56KtIfqc';
flush privileges;
```
- `source admin-openrc`
- `openstack project create --domain default --description "Service Project" service`
- `openstack user create --domain default --password 56KtIfqc glance`
- `openstack role add --project service --user glance admin`
- `openstack service create --name glance --description "OpenStack Image" image`
- `openstack endpoint create --region RegionOne image public http://10.10.10.69:9292`
- `openstack endpoint create --region RegionOne image internal http://10.10.10.69:9292`
- `openstack endpoint create --region RegionOne image admin http://10.10.10.69:9292`
## config glance
- yum install openstack-glance python3-glance python3-glanceclient -y
- cp /etc/glance/glance-api.conf{,.bak}
- egrep -v '^$|^#' /etc/glance/glance-api.conf.bak >/etc/glance/glance-api.conf
- config glance.conf:
```shell
openstack-config --set /etc/glance/glance-api.conf DEFAULT bind_host 10.10.10.51
openstack-config --set /etc/glance/glance-api.conf database connection  mysql+pymysql://glance:56KtIfqc@10.10.10.69/glance
openstack-config --set /etc/glance/glance-api.conf glance_store stores file,http
openstack-config --set /etc/glance/glance-api.conf glance_store default_store file
openstack-config --set /etc/glance/glance-api.conf glance_store filesystem_store_datadir /var/lib/glance/images/
openstack-config --set /etc/glance/glance-api.conf keystone_authtoken www_authenticate_uri   http://10.10.10.69:5000
openstack-config --set /etc/glance/glance-api.conf keystone_authtoken auth_url  http://10.10.10.69:5000
openstack-config --set /etc/glance/glance-api.conf keystone_authtoken memcached_servers openstack-controller01:11211,openstack-controller02:11211,openstack-controller03:11211
openstack-config --set /etc/glance/glance-api.conf keystone_authtoken auth_type password
openstack-config --set /etc/glance/glance-api.conf keystone_authtoken project_domain_name Default
openstack-config --set /etc/glance/glance-api.conf keystone_authtoken user_domain_name Default
openstack-config --set /etc/glance/glance-api.conf keystone_authtoken project_name  service
openstack-config --set /etc/glance/glance-api.conf keystone_authtoken username glance
openstack-config --set /etc/glance/glance-api.conf keystone_authtoken password 56KtIfqc
openstack-config --set /etc/glance/glance-api.conf paste_deploy flavor keystone
```
- `scp -rp /etc/glance/glance-api.conf openstack-controller02:/etc/glance/glance-api.conf`
- `scp -rp /etc/glance/glance-api.conf openstack-controller03:/etc/glance/glance-api.conf`
- [controller02~ ]# `openstack-config --set /etc/glance/glance-api.conf DEFAULT bind_host 10.10.10.52`
- [controller02~ ]# `openstack-config --set /etc/glance/glance-api.conf DEFAULT bind_host 10.10.10.53`
- [all]# 
```shell
systemctl enable openstack-glance-api.service
systemctl restart openstack-glance-api.service
systemctl status openstack-glance-api.service
```
- create image storage folder at all controller nodes:
    -` mkdir /var/lib/glance/images/`
    - `chown glance:nobody /var/lib/glance/images`
- sync glance database: `su -s /bin/sh -c "glance-manage db_sync" glance`
    - verify: `mysql -uglance -p -e "use glance;show tables;"`

## verify image server
- `source admin-openrc`
- `wget -c http://download.cirros-cloud.net/0.5.1/cirros-0.5.1-x86_64-disk.img`
- `openstack image create --file ~/cirros-0.5.1-x86_64-disk.img --disk-format qcow2 --container-format bare --public cirros-qcow2`
- `openstack image list`

## add pcs source
- `pcs resource create openstack-glance-api systemd:openstack-glance-api clone interleave=true`
- `pcs resource`


# 五. placement
## prepare
- mysql -uroot -p
```shell
CREATE DATABASE placement;
GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'localhost' IDENTIFIED BY '58PGTpgC';
GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%' IDENTIFIED BY '58PGTpgC';
flush privileges;
```
- create placement endpoint:
```shell
openstack user create --domain default --password=58PGTpgC placement
openstack role add --project service --user placement admin
openstack service create --name placement --description "Placement API" placement
openstack endpoint create --region RegionOne placement public http://10.10.10.69:8778
openstack endpoint create --region RegionOne placement internal http://10.10.10.69:8778
openstack endpoint create --region RegionOne placement admin http://10.10.10.69:8778
```
## config placement:
- yum install openstack-placement-api -y
- cp /etc/placement/placement.conf /etc/placement/placement.conf.bak
- grep -Ev '^$|#' /etc/placement/placement.conf.bak > /etc/placement/placement.conf
```shell
openstack-config --set /etc/placement/placement.conf placement_database connection mysql+pymysql://placement:58PGTpgC@10.10.10.69/placement
openstack-config --set /etc/placement/placement.conf api auth_strategy keystone
openstack-config --set /etc/placement/placement.conf keystone_authtoken auth_url  http://10.10.10.69:5000/v3
openstack-config --set /etc/placement/placement.conf keystone_authtoken memcached_servers openstack-controller01:11211,openstack-controller02:11211,openstack-controller03:11211
openstack-config --set /etc/placement/placement.conf keystone_authtoken auth_type password
openstack-config --set /etc/placement/placement.conf keystone_authtoken project_domain_name Default
openstack-config --set /etc/placement/placement.conf keystone_authtoken user_domain_name Default
openstack-config --set /etc/placement/placement.conf keystone_authtoken project_name service
openstack-config --set /etc/placement/placement.conf keystone_authtoken username placement
openstack-config --set /etc/placement/placement.conf keystone_authtoken password 58PGTpgC
```
- scp /etc/placement/placement.conf openstack-controller02:/etc/placement/
- scp /etc/placement/placement.conf openstack-controller03:/etc/placement/

- sync database:
    - `su -s /bin/sh -c "placement-manage db sync" placement`
    - verify: `mysql -uroot -p placement -e " show tables;"`

## config 00-placement-api.conf
- apply to every controller node
- controller01:
    - `cp /etc/httpd/conf.d/00-placement-api.conf{,.bak}`
    - `sed -i "s/Listen\ 8778/Listen\ 10.10.10.51:8778/g" /etc/httpd/conf.d/00-placement-api.conf`
    - `sed -i "s/*:8778/10.10.10.51:8778/g" /etc/httpd/conf.d/00-placement-api.conf`
- controller02:
    - `cp /etc/httpd/conf.d/00-placement-api.conf{,.bak}`
    - `sed -i "s/Listen\ 8778/Listen\ 10.10.10.52:8778/g" /etc/httpd/conf.d/00-placement-api.conf`
    - `sed -i "s/*:8778/10.10.10.52:8778/g" /etc/httpd/conf.d/00-placement-api.conf`
- controller03:
    - `cp /etc/httpd/conf.d/00-placement-api.conf{,.bak}`
    - `sed -i "s/Listen\ 8778/Listen\ 10.10.10.53:8778/g" /etc/httpd/conf.d/00-placement-api.conf`
    - `sed -i "s/*:8778/10.10.10.53:8778/g" /etc/httpd/conf.d/00-placement-api.conf`
- vim /etc/httpd/conf.d/00-placement-api.conf
```
...
  #SSLCertificateKeyFile
  #SSLCertificateKeyFile ...
  <Directory /usr/bin>
   <IfVersion >= 2.4>
      Require all granted
   </IfVersion>
   <IfVersion < 2.4>
      Order allow,deny
      Allow from all
   </IfVersion>
  </Directory>
...
```
- systemctl restart httpd.service
- netstat -lntup|grep 8778
- lsof -i:8778
- check placement state: `placement-status upgrade check`

# 六. nova
## prepare
- `mysql -uroot -p`
```shell
CREATE DATABASE nova_api;
CREATE DATABASE nova;
CREATE DATABASE nova_cell0;

GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' IDENTIFIED BY '7Pbjqyyf';
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY '7Pbjqyyf';

GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY '7Pbjqyyf';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY '7Pbjqyyf';

GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' IDENTIFIED BY '7Pbjqyyf';
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' IDENTIFIED BY '7Pbjqyyf';
flush privileges;
```

- create endpoint:
```shell
source admin-openrc
openstack user create --domain default --password 7Pbjqyyf nova
openstack role add --project service --user nova admin
openstack service create --name nova --description "OpenStack Compute" compute
openstack endpoint create --region RegionOne compute public http://10.10.10.69:8774/v2.1
openstack endpoint create --region RegionOne compute internal http://10.10.10.69:8774/v2.1
openstack endpoint create --region RegionOne compute admin http://10.10.10.69:8774/v2.1
```
## config nova-api
- yum install openstack-nova-api openstack-nova-conductor openstack-nova-novncproxy openstack-nova-scheduler -y
- cp -a /etc/nova/nova.conf{,.bak}
- grep -Ev '^$|#' /etc/nova/nova.conf.bak > /etc/nova/nova.conf
- [controller01~ ]#:
```shell
openstack-config --set /etc/nova/nova.conf DEFAULT enabled_apis  osapi_compute,metadata
openstack-config --set /etc/nova/nova.conf DEFAULT my_ip  10.10.10.51
openstack-config --set /etc/nova/nova.conf DEFAULT use_neutron  true
openstack-config --set /etc/nova/nova.conf DEFAULT firewall_driver  nova.virt.firewall.NoopFirewallDriver
openstack-config --set /etc/nova/nova.conf DEFAULT transport_url rabbit://openstack:zp@131421@openstack-controller01:5672,openstack:zp@131421@openstack-controller02:5672,openstack:zp@131421@openstack-controller03:5672
openstack-config --set /etc/nova/nova.conf DEFAULT osapi_compute_listen_port 8774
openstack-config --set /etc/nova/nova.conf DEFAULT metadata_listen_port 8775
openstack-config --set /etc/nova/nova.conf DEFAULT metadata_listen '$my_ip'
openstack-config --set /etc/nova/nova.conf DEFAULT osapi_compute_listen '$my_ip'
openstack-config --set /etc/nova/nova.conf api auth_strategy  keystone
openstack-config --set /etc/nova/nova.conf api_database  connection  mysql+pymysql://nova:7Pbjqyyf@10.10.10.69/nova_api
openstack-config --set /etc/nova/nova.conf cache backend oslo_cache.memcache_pool
openstack-config --set /etc/nova/nova.conf cache enabled True
openstack-config --set /etc/nova/nova.conf cache memcache_servers openstack-controller01:11211,openstack-controller02:11211,openstack-controller03:11211
openstack-config --set /etc/nova/nova.conf database connection  mysql+pymysql://nova:7Pbjqyyf@10.10.10.69/nova
openstack-config --set /etc/nova/nova.conf keystone_authtoken www_authenticate_uri  http://10.10.10.69:5000/v3
openstack-config --set /etc/nova/nova.conf keystone_authtoken auth_url  http://10.10.10.69:5000/v3
openstack-config --set /etc/nova/nova.conf keystone_authtoken memcached_servers  openstack-controller01:11211,openstack-controller02:11211,openstack-controller03:11211
openstack-config --set /etc/nova/nova.conf keystone_authtoken auth_type  password
openstack-config --set /etc/nova/nova.conf keystone_authtoken project_domain_name  Default
openstack-config --set /etc/nova/nova.conf keystone_authtoken user_domain_name  Default
openstack-config --set /etc/nova/nova.conf keystone_authtoken project_name  service
openstack-config --set /etc/nova/nova.conf keystone_authtoken username  nova
openstack-config --set /etc/nova/nova.conf keystone_authtoken password  7Pbjqyyf
openstack-config --set /etc/nova/nova.conf vnc enabled  true
openstack-config --set /etc/nova/nova.conf vnc server_listen  '$my_ip'
openstack-config --set /etc/nova/nova.conf vnc server_proxyclient_address  '$my_ip'
openstack-config --set /etc/nova/nova.conf vnc novncproxy_host '$my_ip'
openstack-config --set /etc/nova/nova.conf vnc novncproxy_port  6080
openstack-config --set /etc/nova/nova.conf glance  api_servers  http://10.10.10.69:9292
openstack-config --set /etc/nova/nova.conf oslo_concurrency lock_path  /var/lib/nova/tmp
openstack-config --set /etc/nova/nova.conf placement region_name  RegionOne
openstack-config --set /etc/nova/nova.conf placement project_domain_name  Default
openstack-config --set /etc/nova/nova.conf placement project_name  service
openstack-config --set /etc/nova/nova.conf placement auth_type  password
openstack-config --set /etc/nova/nova.conf placement user_domain_name  Default
openstack-config --set /etc/nova/nova.conf placement auth_url  http://10.10.10.69:5000/v3
openstack-config --set /etc/nova/nova.conf placement username  placement
openstack-config --set /etc/nova/nova.conf placement password  58PGTpgC
```
- note:
    - 注意几个密码：MQ_PASS, PLACEMENT_PASS
    - 前端采用haproxy时，服务连接rabbitmq会出现连接超时重连的情况，可通过各服务与rabbitmq的日志查看；
    - transport_url=rabbit://openstack:zp@131421@10.10.10.69:5672
    - rabbitmq本身具备集群机制，官方文档建议直接连接rabbitmq集群；但采用此方式时服务启动有时会报错，原因不明；如果没有此现象，建议连接rabbitmq直接对接集群而非通过前端haproxy的vip+端口
        - openstack-config --set /etc/nova/nova.conf DEFAULT transport_url rabbit://openstack:xxx@openstack-controller01:5672,openstack:xxx@openstack-controller02:5672,openstack:xxx@openstack-controller03:5672
- `scp -rp /etc/nova/nova.conf openstack-controller02:/etc/nova/`
- `scp -rp /etc/nova/nova.conf openstack-controller03:/etc/nova/`
- [controller02~ ]# `sed -i "s#10.10.10.51#10.10.10.52#g" /etc/nova/nova.conf`
- [controller03~ ]# `sed -i "s#10.10.10.51#10.10.10.53#g" /etc/nova/nova.conf`
- sync database:
    - `su -s /bin/sh -c "nova-manage api_db sync" nova`
    - `su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova`
    - `su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova`
    - `su -s /bin/sh -c "nova-manage db sync" nova`
    - verify:
        - `su -s /bin/sh -c "nova-manage cell_v2 list_cells" nova`
        - `mysql -h openstack-controller01 -u nova -p -e "use nova_api;show tables;"`
        - `mysql -h openstack-controller01 -u nova -p -e "use nova;show tables;"`
        - `mysql -h openstack-controller01 -u nova -p -e "use nova_cell0;show tables;"`

- systemctl restart openstack-nova-api.service 
- systemctl restart openstack-nova-scheduler.service 
- systemctl restart openstack-nova-conductor.service 
- systemctl restart openstack-nova-novncproxy.service
- systemctl enable openstack-nova-api.service 
- systemctl enable openstack-nova-scheduler.service 
- systemctl enable openstack-nova-conductor.service 
- systemctl enable openstack-nova-novncproxy.service
- systemctl status openstack-nova-api.service 
- systemctl status openstack-nova-scheduler.service 
- systemctl status openstack-nova-conductor.service 
- systemctl status openstack-nova-novncproxy.service
- verify: 
    - `netstat -tunlp | egrep '8774|8775|8778|6080'`
    - `curl http://10.10.10.69:8774`
    - `openstack compute service list`
    - `openstack catalog list`
    - `nova-status upgrade check`
## pcs source
- `pcs resource create openstack-nova-api systemd:openstack-nova-api clone interleave=true`
- `pcs resource create openstack-nova-scheduler systemd:openstack-nova-scheduler clone interleave=true`
- `pcs resource create openstack-nova-conductor systemd:openstack-nova-conductor clone interleave=true`
- `pcs resource create openstack-nova-novncproxy systemd:openstack-nova-novncproxy clone interleave=true`
- note: openstack-nova-api，openstack-nova-conductor与openstack-nova-novncproxy 等无状态服务以active/active模式运行；openstack-nova-scheduler等服务以active/passive模式运行
## compute config
- yum install -y openstack-nova-compute openstack-utils
- backup conf file:
    - cp /etc/nova/nova.conf{,.bak}
    - grep -Ev '^$|#' /etc/nova/nova.conf.bak > /etc/nova/nova.conf
- get hardware info: `egrep -c '(vmx|svm)' /proc/cpuinfo`
- config nove:
```shell
openstack-config --set  /etc/nova/nova.conf DEFAULT enabled_apis  osapi_compute,metadata
openstack-config --set  /etc/nova/nova.conf DEFAULT transport_url  rabbit://openstack:zp@131421@10.10.10.69
openstack-config --set  /etc/nova/nova.conf DEFAULT my_ip 10.10.10.54
openstack-config --set  /etc/nova/nova.conf DEFAULT use_neutron  true
openstack-config --set  /etc/nova/nova.conf DEFAULT firewall_driver  nova.virt.firewall.NoopFirewallDriver
openstack-config --set  /etc/nova/nova.conf api auth_strategy  keystone
openstack-config --set /etc/nova/nova.conf  keystone_authtoken www_authenticate_url  http://10.10.10.69:5000
openstack-config --set  /etc/nova/nova.conf keystone_authtoken auth_url  http://10.10.10.69:5000
openstack-config --set  /etc/nova/nova.conf keystone_authtoken memcached_servers  openstack-controller01:11211,openstack-controller02:11211,openstack-controller03:11211
openstack-config --set  /etc/nova/nova.conf keystone_authtoken auth_type  password
openstack-config --set  /etc/nova/nova.conf keystone_authtoken project_domain_name  Default
openstack-config --set  /etc/nova/nova.conf keystone_authtoken user_domain_name  Default
openstack-config --set  /etc/nova/nova.conf keystone_authtoken project_name  service
openstack-config --set  /etc/nova/nova.conf keystone_authtoken username  nova
openstack-config --set  /etc/nova/nova.conf keystone_authtoken password  7Pbjqyyf
openstack-config --set /etc/nova/nova.conf libvirt virt_type  kvm
openstack-config --set  /etc/nova/nova.conf vnc enabled  true
openstack-config --set  /etc/nova/nova.conf vnc server_listen  0.0.0.0
openstack-config --set  /etc/nova/nova.conf vnc server_proxyclient_address  '$my_ip'
openstack-config --set  /etc/nova/nova.conf vnc novncproxy_base_url http://10.10.10.69:6080/vnc_auto.html
openstack-config --set  /etc/nova/nova.conf glance api_servers  http://10.10.10.69:9292
openstack-config --set  /etc/nova/nova.conf oslo_concurrency lock_path  /var/lib/nova/tmp
openstack-config --set  /etc/nova/nova.conf placement region_name  RegionOne
openstack-config --set  /etc/nova/nova.conf placement project_domain_name  Default
openstack-config --set  /etc/nova/nova.conf placement project_name  service
openstack-config --set  /etc/nova/nova.conf placement auth_type  password
openstack-config --set  /etc/nova/nova.conf placement user_domain_name  Default
openstack-config --set  /etc/nova/nova.conf placement auth_url  http://10.10.10.69:5000/v3
openstack-config --set  /etc/nova/nova.conf placement username  placement
openstack-config --set  /etc/nova/nova.conf placement password  58PGTpgC

```
- systemctl restart libvirtd.service openstack-nova-compute.service
- systemctl enable libvirtd.service openstack-nova-compute.service
- systemctl status libvirtd.service openstack-nova-compute.service
## add compute node:
- `openstack compute service list --service nove-compute`
- found compute node: `su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova`
- auto found:
    - `openstack-config --set  /etc/nova/nova.conf scheduler discover_hosts_in_cells_interval 600`
    - `systemctl restart openstack-nova-api.service`    
- verify: 
    - `openstack compute service list`
    - `openstack catalog list`
    - `openstack image list`
    - `nova-status upgrade check`

# 七. neutron
## prepare
- mysql -u root -pxxx
```shell
CREATE DATABASE neutron;
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY '6ZKOyWXt';
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY '6ZKOyWXt';
flush privileges;
```
- create endpoint
```shell
source admin-openrc
openstack user create --domain default --password 6ZKOyWXt neutron
openstack role add --project service --user neutron admin
openstack service create --name neutron --description "OpenStack Networking" network
openstack endpoint create --region RegionOne network public http://10.10.10.69:9696
openstack endpoint create --region RegionOne network internal http://10.10.10.69:9696
openstack endpoint create --region RegionOne network admin http://10.10.10.69:9696
```
## controller node config
- Services:
    - openstack-neutron：neutron-server的包
    - openstack-neutron-ml2：ML2 plugin的包
    - openstack-neutron-linuxbridge：linux bridge network provider相关的包
    - ebtables：防火墙相关的包
    - conntrack-tools： 该模块可以对iptables进行状态数据包检查
- 全部 controller 结点：
    - yum install openstack-neutron openstack-neutron-ml2 openstack-neutron-linuxbridge ebtables conntrack-tools -y
    - 备份配置文件： /etc/neutron/neutron.conf
        - cp -a /etc/neutron/neutron.conf{,.bak}
        - grep -Ev '^$|#' /etc/neutron/neutron.conf.bak > /etc/neutron/neutron.conf
### neutron server config
- openstack-config at controller01(10.10.10.51):
``` shell
    openstack-config --set  /etc/neutron/neutron.conf DEFAULT bind_host 10.10.10.51
    openstack-config --set  /etc/neutron/neutron.conf DEFAULT core_plugin ml2
    openstack-config --set  /etc/neutron/neutron.conf DEFAULT service_plugins router
    openstack-config --set  /etc/neutron/neutron.conf DEFAULT allow_overlapping_ips true
    #直接连接rabbitmq集群
    openstack-config --set /etc/neutron/neutron.conf DEFAULT transport_url rabbit://openstack:zp@131421@openstack-controller01:5672,openstack:zp@131421@openstack-controller02:5672,openstack:zp@131421@openstack-controller03:5672
    openstack-config --set  /etc/neutron/neutron.conf DEFAULT auth_strategy  keystone
    openstack-config --set  /etc/neutron/neutron.conf DEFAULT notify_nova_on_port_status_changes  true
    openstack-config --set  /etc/neutron/neutron.conf DEFAULT notify_nova_on_port_data_changes  true
    #启用l3 ha功能
    #openstack-config --set  /etc/neutron/neutron.conf DEFAULT l3_ha True
    #最多在几个l3 agent上创建ha router
    #openstack-config --set  /etc/neutron/neutron.conf DEFAULT max_l3_agents_per_router 3
    #可创建ha router的最少正常运行的l3 agnet数量
    #openstack-config --set  /etc/neutron/neutron.conf DEFAULT min_l3_agents_per_router 2
    #dhcp高可用，在3个网络节点各生成1个dhcp服务器
    #openstack-config --set  /etc/neutron/neutron.conf DEFAULT dhcp_agents_per_network 3
    openstack-config --set  /etc/neutron/neutron.conf DEFAULT l3_ha True
    openstack-config --set  /etc/neutron/neutron.conf DEFAULT max_l3_agents_per_router 2
    openstack-config --set  /etc/neutron/neutron.conf DEFAULT min_l3_agents_per_router 1
    openstack-config --set  /etc/neutron/neutron.conf DEFAULT dhcp_agents_per_network 2

    openstack-config --set  /etc/neutron/neutron.conf database connection  mysql+pymysql://neutron:6ZKOyWXt@10.10.10.69/neutron

    openstack-config --set  /etc/neutron/neutron.conf keystone_authtoken www_authenticate_uri  http://10.10.10.69:5000
    openstack-config --set  /etc/neutron/neutron.conf keystone_authtoken auth_url  http://10.10.10.69:5000
    openstack-config --set  /etc/neutron/neutron.conf keystone_authtoken memcached_servers  openstack-controller01:11211,openstack-controller02:11211,openstack-controller03:11211
    openstack-config --set  /etc/neutron/neutron.conf keystone_authtoken auth_type  password
    openstack-config --set  /etc/neutron/neutron.conf keystone_authtoken project_domain_name  default
    openstack-config --set  /etc/neutron/neutron.conf keystone_authtoken user_domain_name  default
    openstack-config --set  /etc/neutron/neutron.conf keystone_authtoken project_name  service
    openstack-config --set  /etc/neutron/neutron.conf keystone_authtoken username  neutron
    openstack-config --set  /etc/neutron/neutron.conf keystone_authtoken password  6ZKOyWXt

    openstack-config --set  /etc/neutron/neutron.conf nova  auth_url http://10.10.10.69:5000
    openstack-config --set  /etc/neutron/neutron.conf nova  auth_type password
    openstack-config --set  /etc/neutron/neutron.conf nova  project_domain_name default
    openstack-config --set  /etc/neutron/neutron.conf nova  user_domain_name default
    openstack-config --set  /etc/neutron/neutron.conf nova  region_name RegionOne
    openstack-config --set  /etc/neutron/neutron.conf nova  project_name service
    openstack-config --set  /etc/neutron/neutron.conf nova  username nova
    openstack-config --set  /etc/neutron/neutron.conf nova  password 7Pbjqyyf

    openstack-config --set  /etc/neutron/neutron.conf oslo_concurrency lock_path  /var/lib/neutron/tmp
```
- scp -rp /etc/neutron/neutron.conf openstack-controller02:/etc/neutron/
- scp -rp /etc/neutron/neutron.conf openstack-controller03:/etc/neutron/
- openstack-contraller02# sed -i "s#10.10.10.51#10.10.10.52#g" /etc/neutron/neutron.conf
- openstack-contraller03# sed -i "s#10.10.10.51#10.10.10.53#g" /etc/neutron/neutron.conf
### ml2 config
- openstack-controller01:
    - cp -a /etc/neutron/plugins/ml2/ml2_conf.ini{,.bak}
    - grep -Ev '^$|#' /etc/neutron/plugins/ml2/ml2_conf.ini.bak > /etc/neutron/plugins/ml2/ml2_conf.ini
    - command:
```shell
openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 type_drivers  flat,vlan,vxlan
openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 tenant_network_types vxlan
openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 mechanism_drivers  linuxbridge,l2population
openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 extension_drivers  port_security
openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2_type_flat flat_networks  provider
openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2_type_vxlan vni_ranges 1:1000
openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini securitygroup enable_ipset  true
```
- scp -rp /etc/neutron/plugins/ml2/ml2_conf.ini openstack-controller02:/etc/neutron/plugins/ml2/ml2_conf.ini
- scp -rp /etc/neutron/plugins/ml2/ml2_conf.ini openstack-controller03:/etc/neutron/plugins/ml2/ml2_conf.ini
### nova config
- 修改配置文件/etc/nova/nova.conf
- 在全部控制节点上配置nova服务与网络节点服务进行交互
```shell
openstack-config --set  /etc/nova/nova.conf neutron url  http://10.10.10.69:9696
openstack-config --set  /etc/nova/nova.conf neutron auth_url  http://10.10.10.69:5000
openstack-config --set  /etc/nova/nova.conf neutron auth_type  password
openstack-config --set  /etc/nova/nova.conf neutron project_domain_name  default
openstack-config --set  /etc/nova/nova.conf neutron user_domain_name  default
openstack-config --set  /etc/nova/nova.conf neutron region_name  RegionOne
openstack-config --set  /etc/nova/nova.conf neutron project_name  service
openstack-config --set  /etc/nova/nova.conf neutron username  neutron
openstack-config --set  /etc/nova/nova.conf neutron password  6ZKOyWXt
openstack-config --set  /etc/nova/nova.conf neutron service_metadata_proxy  true
openstack-config --set  /etc/nova/nova.conf neutron metadata_proxy_shared_secret  6ZKOyWXt
```
### sync nova database
- [root@controller01 ~]# su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
- verify: mysql -h openstack-controller03 -u neutron -p6ZKOyWXt -e "use neutron;show tables;"
### create soft linker for ML2
- all controller node:
    - ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
### restart nova and neutron server
- all controller node:
```shell
systemctl enable neutron-server.service
systemctl restart neutron-server.service
systemctl status neutron-server.service
systemctl restart openstack-nova-api.service
systemctl status openstack-nova-api.service
```
## compute node config
- 由于这里部署为neutron server与neutron agent分离，所以采取这样的部署方式，常规的控制节点部署所有neutron的应用包括server和agent；
- 计算节点部署neutron agent、linuxbridge和nova配置即可；也可以单独准备网络节点进行neutron agent的部署；
### neutron service config
- yum install openstack-neutron openstack-neutron-ml2 openstack-neutron-linuxbridge ebtables -y
- 备份配置文件/etc/nova/nova.conf
    - cp -a /etc/neutron/neutron.conf{,.bak}
    - grep -Ev '^$|#' /etc/neutron/neutron.conf.bak > /etc/neutron/neutron.conf
- neutron config:
```shell
openstack-config --set  /etc/neutron/neutron.conf DEFAULT bind_host 10.10.10.54
openstack-config --set /etc/neutron/neutron.conf DEFAULT transport_url rabbit://openstack:zp@131421@openstack-controller01:5672,openstack:zp@131421@openstack-controller02:5672,openstack:zp@131421@openstack-controller03:5672
openstack-config --set  /etc/neutron/neutron.conf DEFAULT auth_strategy keystone 
#配置RPC的超时时间，默认为60s,可能导致超时异常.设置为180s
openstack-config --set  /etc/neutron/neutron.conf DEFAULT rpc_response_timeout 180

openstack-config --set  /etc/neutron/neutron.conf keystone_authtoken www_authenticate_uri http://10.10.10.69:5000
openstack-config --set  /etc/neutron/neutron.conf keystone_authtoken auth_url http://10.10.10.69:5000
openstack-config --set  /etc/neutron/neutron.conf keystone_authtoken memcached_servers openstack-controller01:11211,openstack-controller02:11211,openstack-controller03:11211
openstack-config --set  /etc/neutron/neutron.conf keystone_authtoken auth_type password
openstack-config --set  /etc/neutron/neutron.conf keystone_authtoken project_domain_name default
openstack-config --set  /etc/neutron/neutron.conf keystone_authtoken user_domain_name default
openstack-config --set  /etc/neutron/neutron.conf keystone_authtoken project_name service
openstack-config --set  /etc/neutron/neutron.conf keystone_authtoken username neutron
openstack-config --set  /etc/neutron/neutron.conf keystone_authtoken password 6ZKOyWXt

openstack-config --set  /etc/neutron/neutron.conf oslo_concurrency lock_path /var/lib/neutron/tmp
```
- scp -rp /etc/neutron/neutron.conf openstack-compute02:/etc/neutron/
- [computer02 #] sed -i "s#10.10.10.54#10.10.10.55#g" /etc/neutron/neutron.conf
### nova config
- all compute nodes
```shell
openstack-config --set  /etc/nova/nova.conf neutron url http://10.10.10.69:9696
openstack-config --set  /etc/nova/nova.conf neutron auth_url http://10.10.10.69:5000
openstack-config --set  /etc/nova/nova.conf neutron auth_type password
openstack-config --set  /etc/nova/nova.conf neutron project_domain_name default
openstack-config --set  /etc/nova/nova.conf neutron user_domain_name default
openstack-config --set  /etc/nova/nova.conf neutron region_name RegionOne
openstack-config --set  /etc/nova/nova.conf neutron project_name service
openstack-config --set  /etc/nova/nova.conf neutron username neutron
openstack-config --set  /etc/nova/nova.conf neutron password 6ZKOyWXt
```
### plugin and agent config
#### ML2
- all compute nodes
- cp -a /etc/neutron/plugins/ml2/ml2_conf.ini{,.bak}
- grep -Ev '^$|#' /etc/neutron/plugins/ml2/ml2_conf.ini.bak > /etc/neutron/plugins/ml2/ml2_conf.ini
```shell
openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 type_drivers  flat,vlan,vxlan
openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 tenant_network_types vxlan
openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 mechanism_drivers  linuxbridge,l2population
openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 extension_drivers  port_security
openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2_type_flat flat_networks  provider
openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2_type_vxlan vni_ranges 1:1000
openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini securitygroup enable_ipset  true
```
- scp -rp /etc/neutron/plugins/ml2/ml2_conf.ini openstack-compute02:/etc/neutron/plugins/ml2/ml2_conf.ini
#### Linux bridge agent config
- all compute nodes
- cp -a /etc/neutron/plugins/ml2/linuxbridge_agent.ini{,.bak}
- grep -Ev '^$|#' /etc/neutron/plugins/ml2/linuxbridge_agent.ini.bak >/etc/neutron/plugins/ml2/linuxbridge_agent.ini
```shell
#环境无法提供四张网卡；建议生产环境上将每种网络分开配置
#provider网络对应规划的ens192
openstack-config --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini linux_bridge physical_interface_mappings  provider:ens192

openstack-config --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini vxlan enable_vxlan true

#tunnel租户网络（vxlan）vtep端点，这里对应规划的ens192地址，根据节点做相应修改
openstack-config --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini vxlan local_ip 10.10.10.54

openstack-config --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini vxlan l2_population true
openstack-config --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini securitygroup enable_security_group  true
openstack-config --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini securitygroup firewall_driver  neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```
- scp -rp /etc/neutron/plugins/ml2/linuxbridge_agent.ini  openstack-compute02:/etc/neutron/plugins/ml2/
- [compute02 #] sed -i "s#10.10.10.54#10.10.10.55#g" /etc/neutron/plugins/ml2/linuxbridge_agent.ini 
#### L3 agent config
- all compute nodes
- cp -a /etc/neutron/l3_agent.ini{,.bak}
- grep -Ev '^$|#' /etc/neutron/l3_agent.ini.bak > /etc/neutron/l3_agent.ini
- openstack-config --set /etc/neutron/l3_agent.ini DEFAULT interface_driver linuxbridge
#### DHCP agent config
- all compute nodes
- cp -a /etc/neutron/dhcp_agent.ini{,.bak}
- grep -Ev '^$|#' /etc/neutron/dhcp_agent.ini.bak > /etc/neutron/dhcp_agent.ini
```shell
openstack-config --set  /etc/neutron/dhcp_agent.ini DEFAULT interface_driver linuxbridge
openstack-config --set  /etc/neutron/dhcp_agent.ini DEFAULT dhcp_driver neutron.agent.linux.dhcp.Dnsmasq
openstack-config --set  /etc/neutron/dhcp_agent.ini DEFAULT enable_isolated_metadata true
```
#### Metadata agent config
- 元数据代理提供配置信息，例如实例的凭据
- metadata_proxy_shared_secret 的密码与控制节点上/etc/nova/nova.conf文件中密码一致；
- cp -a /etc/neutron/metadata_agent.ini{,.bak}
- grep -Ev '^$|#' /etc/neutron/metadata_agent.ini.bak > /etc/neutron/metadata_agent.ini
```shell
openstack-config --set /etc/neutron/metadata_agent.ini DEFAULT nova_metadata_host 10.10.10.69
openstack-config --set /etc/neutron/metadata_agent.ini DEFAULT metadata_proxy_shared_secret 6ZKOyWXt
openstack-config --set /etc/neutron/metadata_agent.ini cache memcache_servers openstack-controller01:11211,openstack-controller02:11211,openstack-controller03:11211
```
- scp -rp /etc/neutron/metadata_agent.ini openstack-compute02:/etc/neutron/
#### other config
- all compute nodes
```shell
echo 'net.ipv4.ip_nonlocal_bind = 1' >>/etc/sysctl.conf
echo 'net.bridge.bridge-nf-call-iptables=1' >>/etc/sysctl.conf
echo 'net.bridge.bridge-nf-call-ip6tables=1'  >>/etc/sysctl.conf

#启用网络桥接器支持，需要加载 br_netfilter 内核模块；否则会提示没有目录
modprobe br_netfilter
sysctl -p
```
- systemctl restart openstack-nova-compute.service
```shell
systemctl enable neutron-linuxbridge-agent neutron-dhcp-agent neutron-metadata-agent neutron-l3-agent
systemctl restart neutron-linuxbridge-agent neutron-dhcp-agent neutron-metadata-agent neutron-l3-agent
systemctl status neutron-linuxbridge-agent neutron-dhcp-agent neutron-metadata-agent neutron-l3-agent
```
## verify
- [root@controller01]# openstack extension list --network
- [root@controller01]# openstack network agent list
## PCS resource
- 只需要添加neutron-server，其他的neutron-agent服务：neutron-linuxbridge-agent，neutron-l3-agent，neutron-dhcp-agent与neutron-metadata-agent 不需要添加了；因为部署在了计算节点上
- any controller node:
```shell
#pcs resource create neutron-linuxbridge-agent systemd:neutron-linuxbridge-agent clone interleave=true
#pcs resource create neutron-l3-agent systemd:neutron-l3-agent clone interleave=true
#pcs resource create neutron-dhcp-agent systemd:neutron-dhcp-agent clone interleave=true
#pcs resource create neutron-metadata-agent systemd:neutron-metadata-agent clone interleave=true
pcs resource create neutron-server systemd:neutron-server clone interleave=true
```
- verify: pcs resource

# 八. horazion
- OpenStack仪表板Dashboard服务的项目名称是Horizon，它所需的唯一服务是身份服务keystone，开发语言是python的web框架Django。
- 仪表盘使得通过OpenStack API与OpenStack计算云控制器进行基于web的交互成为可能。
- Horizon 允许自定义仪表板的商标；并提供了一套内核类和可重复使用的模板及工具。
## config
- all controller node
- yum install openstack-dashboard memcached python3-memcached -y
### local_settings
- cp -a /etc/openstack-dashboard/local_settings{,.bak}
- grep -Ev '^$|#' /etc/openstack-dashboard/local_settings.bak >/etc/openstack-dashboard/local_settings
- vim /etc/openstack-dashboard/local_settings
```shell
#指定在网络服务器中配置仪表板的访问位置;默认值: "/"
WEBROOT = '/dashboard/'
#配置仪表盘在controller节点上使用OpenStack服务
OPENSTACK_HOST = "10.10.10.69"

#允许主机访问仪表板,接受所有主机,不安全不应在生产中使用
ALLOWED_HOSTS = ['*', 'localhost']
#ALLOWED_HOSTS = ['one.example.com', 'two.example.com']

#配置memcached会话存储服务
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
CACHES = {
    'default': {
         'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
         'LOCATION': 'openstack-controller01:11211,openstack-controller02:11211,openstack-controller03:11211',
    }
}

#启用身份API版本3
OPENSTACK_KEYSTONE_URL = "http://%s:5000/v3" % OPENSTACK_HOST

#启用对域的支持
OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True

#配置API版本
OPENSTACK_API_VERSIONS = {
    "identity": 3,
    "image": 2,
    "volume": 3,
}

#配置Default为通过仪表板创建的用户的默认域
OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "Default"

#配置user为通过仪表板创建的用户的默认角色
OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"

#如果选择网络选项1，请禁用对第3层网络服务的支持,如果选择网络选项2,则可以打开
OPENSTACK_NEUTRON_NETWORK = {
    #自动分配的网络
    'enable_auto_allocated_network': False,
    #Neutron分布式虚拟路由器（DVR）
    'enable_distributed_router': False,
    #FIP拓扑检查
    'enable_fip_topology_check': False,
    #高可用路由器模式
    'enable_ha_router': True,
    #下面三个已过时,不用过多了解,官方文档配置中是关闭的
    'enable_lb': False,
    'enable_firewall': False,
    'enable_vpn': False,
    #ipv6网络
    'enable_ipv6': True,
    #Neutron配额功能
    'enable_quotas': True,
    #rbac政策
    'enable_rbac_policy': True,
    #路由器的菜单和浮动IP功能,Neutron部署中有三层功能的支持;可以打开
    'enable_router': True,
    #默认的DNS名称服务器
    'default_dns_nameservers': [],
    #网络支持的提供者类型,在创建网络时，该列表中的网络类型可供选择
    'supported_provider_types': ['*'],
    #使用与提供网络ID范围,仅涉及到VLAN，GRE，和VXLAN网络类型
    'segmentation_id_range': {},
    #使用与提供网络类型
    'extra_provider_types': {},
    #支持的vnic类型,用于与端口绑定扩展
    'supported_vnic_types': ['*'],
    #物理网络
    'physical_networks': [],
}

#配置时区为亚洲上海
TIME_ZONE = "Asia/Shanghai"
....
```
- scp -rp /etc/openstack-dashboard/local_settings  openstack-controller02:/etc/openstack-dashboard/
- scp -rp /etc/openstack-dashboard/local_settings  openstack-controller03:/etc/openstack-dashboard/
### openstack-dashboard config
- all controller nodes
- cp /etc/httpd/conf.d/openstack-dashboard.conf{,.bak}
- ln -s /etc/openstack-dashboard /usr/share/openstack-dashboard/openstack_dashboard/conf
    - 建立策略文件（policy.json）的软链接，否则登录到dashboard将出现权限错误和显示混乱
- sed -i '3a WSGIApplicationGroup\ %{GLOBAL}' /etc/httpd/conf.d/openstack-dashboard.conf
    - 赋权，在第3行后新增 WSGIApplicationGroup %{GLOBAL}
- scp -rp /etc/httpd/conf.d/openstack-dashboard.conf  openstack-controller02:/etc/httpd/conf.d/
- scp -rp /etc/httpd/conf.d/openstack-dashboard.conf  openstack-controller03:/etc/httpd/conf.d/
```shell
systemctl restart httpd.service memcached.service
systemctl enable httpd.service memcached.service
systemctl status httpd.service memcached.service
```
## verify
- http://10.10.10.69/dashboard
    - domain: default
    - user: admin
    - pass: w1zlcjmK

# 九. cinder
## Intro
- Cinder的核心功能是对卷的管理，允许对卷、卷的类型、卷的快照、卷备份进行处理。它为后端不同的存储设备提供给了统一的接口，不同的块设备服务厂商在Cinder中实现其驱动，可以被Openstack整合管理，nova与cinder的工作原理类似。支持多种 back-end（后端）存储方式，包括 LVM，NFS，Ceph 和其他诸如 EMC、IBM 等商业存储产品和方案。
- Cinder-api 是 cinder 服务的 endpoint，提供 rest 接口，负责处理 client 请求，并将 RPC 请求发送至 cinder-scheduler 组件。
- Cinder-scheduler 负责 cinder 请求调度，其核心部分就是 scheduler_driver, 作为 scheduler manager 的 driver，负责 cinder-volume 具体的调度处理，发送 cinder RPC 请求到选择的 cinder-volume。
- Cinder-volume 负责具体的 volume 请求处理，由不同后端存储提供 volume 存储空间。目前各大存储厂商已经积极地将存储产品的 driver 贡献到 cinder 社区
## prepare
- any controller node:
- data:
```shell
mysql -u root -pxxx

create database cinder;
grant all privileges on cinder.* to 'cinder'@'%' identified by 'PGZoNz7H';
grant all privileges on cinder.* to 'cinder'@'localhost' identified by 'PGZoNz7H';
flush privileges;
```
- openstack:
```shell
source admin-openrc
openstack user create --domain default --password PGZoNz7H cinder
openstack role add --project service --user cinder admin
openstack service create --name cinderv2 --description "OpenStack Block Storage" volumev2
openstack service create --name cinderv3 --description "OpenStack Block Storage" volumev3
#v2
openstack endpoint create --region RegionOne volumev2 public http://10.10.10.69:8776/v2/%\(project_id\)s
openstack endpoint create --region RegionOne volumev2 internal http://10.10.10.69:8776/v2/%\(project_id\)s
openstack endpoint create --region RegionOne volumev2 admin http://10.10.10.69:8776/v2/%\(project_id\)s
#v3
openstack endpoint create --region RegionOne volumev3 public http://10.10.10.69:8776/v3/%\(project_id\)s
openstack endpoint create --region RegionOne volumev3 internal http://10.10.10.69:8776/v3/%\(project_id\)s
openstack endpoint create --region RegionOne volumev3 admin http://10.10.10.69:8776/v3/%\(project_id\)s
```
## cinder-api cinder-scheduler config
- all controller nodes
- yum install openstack-cinder -y
- cp -a /etc/cinder/cinder.conf{,.bak}
- grep -Ev '^$|#' /etc/cinder/cinder.conf.bak > /etc/cinder/cinder.conf
```shell
openstack-config --set /etc/cinder/cinder.conf DEFAULT my_ip 10.10.10.51
openstack-config --set /etc/cinder/cinder.conf DEFAULT auth_strategy keystone
openstack-config --set /etc/cinder/cinder.conf DEFAULT glance_api_servers http://10.10.10.69:9292
openstack-config --set /etc/cinder/cinder.conf DEFAULT osapi_volume_listen '$my_ip'
openstack-config --set /etc/cinder/cinder.conf DEFAULT osapi_volume_listen_port 8776
openstack-config --set /etc/cinder/cinder.conf DEFAULT log_dir /var/log/cinder
#直接连接rabbitmq集群
openstack-config --set /etc/cinder/cinder.conf DEFAULT transport_url rabbit://openstack:zp@131421@openstack-controller01:5672,openstack:zp@131421@openstack-controller02:5672,openstack:zp@131421@openstack-controller03:5672
openstack-config --set /etc/cinder/cinder.conf  database connection mysql+pymysql://cinder:PGZoNz7H@10.10.10.69/cinder
openstack-config --set /etc/cinder/cinder.conf  keystone_authtoken  www_authenticate_url  http://10.10.10.69:5000
openstack-config --set /etc/cinder/cinder.conf  keystone_authtoken  auth_url  http://10.10.10.69:5000
openstack-config --set /etc/cinder/cinder.conf  keystone_authtoken  memcached_servers  openstack-controller01:11211,openstack-controller02:11211,openstack-controller03:11211
openstack-config --set /etc/cinder/cinder.conf  keystone_authtoken  auth_type  password
openstack-config --set /etc/cinder/cinder.conf  keystone_authtoken  project_domain_name  default
openstack-config --set /etc/cinder/cinder.conf  keystone_authtoken  user_domain_name  default
openstack-config --set /etc/cinder/cinder.conf  keystone_authtoken  project_name  service
openstack-config --set /etc/cinder/cinder.conf  keystone_authtoken  username  cinder
openstack-config --set /etc/cinder/cinder.conf  keystone_authtoken  password PGZoNz7H
openstack-config --set /etc/cinder/cinder.conf  oslo_concurrency  lock_path  /var/lib/cinder/tmp
```
- scp -rp /etc/cinder/cinder.conf openstack-controller02:/etc/cinder/
- scp -rp /etc/cinder/cinder.conf openstack-controller03:/etc/cinder/
- [controller 02]# sed -i "s#10.10.10.51#10.10.10.52#g" /etc/cinder/cinder.conf
- [controller 03]# sed -i "s#10.10.10.51#10.10.10.53#g" /etc/cinder/cinder.conf
- all controller node:
    - openstack-config --set /etc/nova/nova.conf cinder os_region_name RegionOne
- any controller node:
    - 同步cinder数据库: su -s /bin/sh -c "cinder-manage db sync" cinder
    - verify: mysql -ucinder -pPGZoNz7H -e "use cinder;show tables;"
- all controller node:
```shell
systemctl restart openstack-nova-api.service
systemctl enable openstack-cinder-api.service openstack-cinder-scheduler.service
systemctl restart openstack-cinder-api.service openstack-cinder-scheduler.service
systemctl status openstack-cinder-api.service openstack-cinder-scheduler.service
```
- vefify
    - `openstack volume service list` or
    - cinder service-list
## PCS resource
- 在任意控制节点操作；添加资源cinder-api与cinder-scheduler
- cinder-api与cinder-scheduler以active/active模式运行；
- openstack-nova-volume以active/passive模式运行
```shell
pcs resource create openstack-cinder-api systemd:openstack-cinder-api clone interleave=true
pcs resource create openstack-cinder-scheduler systemd:openstack-cinder-scheduler clone interleave=true
```
- verify: pcs resource
## cind-volume config
- 使用ceph作为后端的存储进行使用
- 在采用ceph或其他商业/非商业后端存储时，建议将`cinder-volume`服务部署在控制节点，通过pacemaker将服务运行在active/passive模式。
- all controller node:
    - openstack-config --set /etc/cinder/cinder.conf  DEFAULT enabled_backends ceph
### cind-volume 单独部署时：
- yum install openstack-cinder targetcli python3-keystone -y
- cp -a /etc/cinder/cinder.conf{,.bak}
- grep -Ev '#|^$' /etc/cinder/cinder.conf.bak>/etc/cinder/cinder.conf
```shell
openstack-config --set /etc/cinder/cinder.conf DEFAULT transport_url rabbit://openstack:zp@131421@openstack-controller01:5672,openstack:zp@131421@openstack-controller02:5672,openstack:zp@131421@openstack-controller03:5672
openstack-config --set /etc/cinder/cinder.conf  DEFAULT auth_strategy keystone
openstack-config --set /etc/cinder/cinder.conf  DEFAULT my_ip 10.10.10.xx
openstack-config --set /etc/cinder/cinder.conf  DEFAULT glance_api_servers http://10.10.10.69:9292
openstack-config --set /etc/cinder/cinder.conf  DEFAULT enabled_backends ceph
openstack-config --set /etc/cinder/cinder.conf  database connection mysql+pymysql://cinder:PGZoNz7H@10.10.10.69/cinder
openstack-config --set /etc/cinder/cinder.conf  keystone_authtoken www_authenticate_uri http://10.10.10.69:5000
openstack-config --set /etc/cinder/cinder.conf  keystone_authtoken auth_url http://10.10.10.69:5000
openstack-config --set /etc/cinder/cinder.conf  keystone_authtoken memcached_servers  openstack-controller01:11211,openstack-controller02:11211,openstack-controller03:11211
openstack-config --set /etc/cinder/cinder.conf  keystone_authtoken auth_type password
openstack-config --set /etc/cinder/cinder.conf  keystone_authtoken project_domain_name default
openstack-config --set /etc/cinder/cinder.conf  keystone_authtoken user_domain_name default
openstack-config --set /etc/cinder/cinder.conf  keystone_authtoken project_name service
openstack-config --set /etc/cinder/cinder.conf  keystone_authtoken username cinder
openstack-config --set /etc/cinder/cinder.conf  keystone_authtoken password PGZoNz7H
openstack-config --set /etc/cinder/cinder.conf  oslo_concurrency lock_path /var/lib/cinder/tmp
```
### others
```shell
systemctl restart openstack-cinder-volume.service target.service
systemctl enable openstack-cinder-volume.service target.service
systemctl status openstack-cinder-volume.service target.service
```
- verify: openstack volume service list
- PCS resource config at any controller node: vim /etc/haproxy/haproxy.conf
```shell
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
- scp haproxy.conf controller02:/etc/haproxy/haproxy.conf
- scp haproxy.conf controller03:/etc/haproxy/haproxy.conf

# 十. OpenStack integrate with Ceph
## Intro
- 数据存储类型
    - 临时性存储：主要由本地文件系统提供，并主要用于nova虚拟机的本地系统与临时数据盘，以及存储glance上传的系统镜像；
    - 永久性存储：主要由cinder提供的块存储与swift提供的对象存储构成，以cinder提供的块存储应用最为广泛，块存储通常以云盘的形式挂载到虚拟机中使用。
- Openstack中需要进行数据存储的三大项目主要是nova项目（虚拟机镜像文件），glance项目（共用模版镜像）与cinder项目（块存储）。
- ![](openstack_ha_config_with_CentOS8/openstack-with-ceph.png)
    - ceph与openstack集成主要用到ceph的rbd服务，ceph底层为rados存储集群，ceph通过librados库实现对底层rados的访问；
    - openstack各项目客户端调用librbd，再由librbd调用librados访问底层rados；实际使用中，nova需要使用libvirtdriver驱动以通过libvirt与qemu调用librbd；cinder与glance可直接调用librbd；
    - 写入ceph集群的数据被条带切分成多个object，object通过hash函数映射到pg（构成pg容器池pool），然后pg通过几圈crush算法近似均匀地映射到物理存储设备osd（osd是基于文件系统的物理存储设备，如xfs，ext4等）。
## Openstack cluster config
- openstack cluster 作为 ceph 的客户端。
- 所有节点安装ceph，目的是在openstack cluster可以是用ceph命令
    - yum install curl python3 podman -y
    - curl --silent --remote-name --location https://github.com/ceph/ceph/raw/octopus/src/cephadm/cephadm
    - chmod +x cephadm
    - ./cephadm add-repo --release octopus
    - ./cephadm install
- glance服务所在节点安装python3-rbd
    - glance-api服务运行在3个控制节点，因此三台控制节点都必须安装
    - yum install python3-rbd -y

- cinder-volume(controller node)与nova-compute(compute node)服务所在节点安装ceph-common
    - cephadm install ceph-common
- copy ceph keyring and config to openstack nodes
```shell
scp -rp /etc/ceph/ceph.client.admin.keyring openstack-controller01:/etc/ceph/
scp -rp /etc/ceph/ceph.client.admin.keyring openstack-controller02:/etc/ceph/
scp -rp /etc/ceph/ceph.client.admin.keyring openstack-controller03:/etc/ceph/
scp -rp /etc/ceph/ceph.client.admin.keyring openstack-compute01:/etc/ceph/
scp -rp /etc/ceph/ceph.client.admin.keyring openstack-compute02:/etc/ceph/
scp -rp /etc/ceph/ceph.conf openstack-controller01:/etc/ceph/ceph.conf
scp -rp /etc/ceph/ceph.conf openstack-controller02:/etc/ceph/ceph.conf
scp -rp /etc/ceph/ceph.conf openstack-controller03:/etc/ceph/ceph.conf
scp -rp /etc/ceph/ceph.conf openstack-compute01:/etc/ceph/ceph.conf
scp -rp /etc/ceph/ceph.conf openstack-compute02:/etc/ceph/ceph.conf
```
## ceph cluster config
### requires
- *`Glance`*作为openstack中镜像服务，支持多种适配器，支持将镜像存放到本地文件系统，http服务器，ceph分布式文件系统，glusterfs和sleepdog等开源的分布式文件系统上。目前glance采用的是本地filesystem的方式存储，存放在默认的路径/var/lib/glance/images下，当把本地的文件系统修改为分布式的文件系统ceph之后，原本在系统中镜像将无法使用，所以建议当前的镜像删除，部署好ceph之后，再统一上传至ceph中存储。
- *`Nova`*负责虚拟机的生命周期管理，包括创建，删除，重建，开机，关机，重启，快照等，作为openstack的核心，nova负责IaaS中计算重要的职责，其中nova的存储格外重要，默认情况下，nova将instance的数据存放在/var/lib/nova/instances/%UUID目录下，使用本地的存储空间。使用这种方式带来的好处是:简单，易实现，速度快，故障域在一个可控制的范围内。然而，缺点也非常明显：compute出故障，上面的虚拟机down机时间长，没法快速恢复，此外，一些特性如热迁移live-migration,虚拟机容灾nova evacuate等高级特性，将无法使用，对于后期的云平台建设，有明显的缺陷。对接 Ceph 主要是希望将实例的系统磁盘文件储存到 Ceph 集群中。与其说是对接 Nova，更准确来说是对接 QEMU-KVM/libvirt，因为 librbd 早已原生集成到其中。
- *`Cinder`*为 OpenStack 提供卷服务，支持非常广泛的后端存储类型。对接 Ceph 后，Cinder 创建的 Volume 本质就是 Ceph RBD 的块设备，当 Volume 被虚拟机挂载后，Libvirt 会以 rbd 协议的方式使用这些 Disk 设备。除了 cinder-volume 之后，Cinder 的 Backup 服务也可以对接 Ceph，将备份的 Image 以对象或块设备的形式上传到 Ceph 集群。
### solution
- 使用ceph的rbd接口，需要通过libvirt，所以需要在客户端机器上安装libvirt和qemu，关于ceph和openstack结合的结构如下，同时，在openstack中，需要用到存储的地方有三个:
    1. glance的镜像，默认的本地存储，路径在/var/lib/glance/images目录下，
    2. nova虚拟机存储，默认本地，路径位于/var/lib/nova/instances目录下，
    3. cinder存储，默认采用LVM的存储方式。
### create pool
- 为 Glance、Nova、Cinder 创建专用的RBD Pools池
- 在ceph-mon-01管理节点上操作；命名为：openstack-volumes，openstack-vms，openstack-images
```shell
ceph osd pool create openstack-images 32 32
ceph osd pool create openstack-volumes 32 32
ceph osd pool create openstack-vms 32 32

rbd pool init openstack-volumes
rbd pool init openstack-images
rbd pool init openstack-vms

ceph osd pool stats
```
- remove pool:
```shell
#cephadm中需要进入容器
podman exec -it ceph-3309274e-1532-11ec-9225-5480284ad7d1-mon.ceph-mon-01 bash
#在ceph.conf配置文件中添加如下内容
[mon]
mon allow pool delete = true
#退出容器
exit

#重启ceph-mon服务
systemctl restart ceph-3309274e-1532-11ec-9225-5480284ad7d1-mon.ceph-mon-01.service
##执行删除pool命令
ceph osd pool delete volumes volumes --yes-i-really-really-mean-it
```
### ceph auth to openstack
- create account for Glance and Cinder
    - [root@cephnode01 ~]# ceph auth get-or-create client.glance mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=openstack-images'
        - out: [client.glance]    key = AQBZXUFhiPqAHBAAlvfM0vp3KN/MH7QKWpTY6A==

    - ceph auth get-or-create client.cinder mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=openstack-volumes, allow rwx pool=openstack-vms, allow rx pool=openstack-images'
        - out: [client.cinder]    key = AQCFXUFhsL9WExAA09N4gAb5UUS4r1eH+PwTwQ==
- push the client.glance to controller nodes
    - ceph auth get-or-create client.glance | ssh root@openstack-controller01 tee /etc/ceph/ceph.client.glance.keyring
    - ceph auth get-or-create client.glance | ssh root@openstack-controller02 tee /etc/ceph/ceph.client.glance.keyring
    - ceph auth get-or-create client.glance | ssh root@openstack-controller03 tee /etc/ceph/ceph.client.glance.keyring
    - ssh root@openstack-controller01 chown glance:glance /etc/ceph/ceph.client.glance.keyring
    - ssh root@openstack-controller02 chown glance:glance /etc/ceph/ceph.client.glance.keyring
    - ssh root@openstack-controller03 chown glance:glance /etc/ceph/ceph.client.glance.keyring
- push the client.cinder to comtroller nodes and compute nodes
    - ceph auth get-or-create client.cinder | ssh root@openstack-controller01 tee /etc/ceph/ceph.client.cinder.keyring
    - ceph auth get-or-create client.cinder | ssh root@openstack-controller02 tee /etc/ceph/ceph.client.cinder.keyring
    - ceph auth get-or-create client.cinder | ssh root@openstack-controller03 tee /etc/ceph/ceph.client.cinder.keyring
    - ssh root@openstack-controller01 chown cinder:cinder /etc/ceph/ceph.client.cinder.keyring
    - ssh root@openstack-controller02 chown cinder:cinder /etc/ceph/ceph.client.cinder.keyring
    - ssh root@openstack-controller03 chown cinder:cinder /etc/ceph/ceph.client.cinder.keyring

### added to Libvirt
- nova-compute 所在节点需要将client.cinder用户的秘钥文件存储到libvirt中；当基于ceph后端的cinder卷被attach到虚拟机实例时，libvirt需要用到该秘钥以访问ceph集群；
- 在ceph管理节点向计算（存储）节点推送client.cinder秘钥文件，生成的文件是临时性的，将秘钥添加到libvirt后可删除
    - ceph auth get-key client.cinder | ssh root@openstack-compute01 tee /etc/ceph/client.cinder.key
    - ceph auth get-key client.cinder | ssh root@openstack-compute02 tee /etc/ceph/client.cinder.key
#### compute nodes config:
- 生成随机 UUID,作为Libvirt秘钥的唯一标识,全部计算节点可共用此uuid;只需要生成一次，所有的cinder-volume、nova-compute都是用同一个UUID,请保持一致;
- uuidgen --> 8dac896c-59b4-45a7-8cad-016d561992b0
- save uuid to xml file:
```shell 
cat >> /etc/ceph/secret.xml <<EOF
<secret ephemeral='no' private='no'>
  <uuid>8dac896c-59b4-45a7-8cad-016d561992b0</uuid>
  <usage type='ceph'>
    <name>client.cinder secret</name>
  </usage>
</secret>
EOF
```
- scp -rp /etc/ceph/secret.xml openstack-compute02:/etc/ceph/
- 定义Libvirt秘钥;全部计算节点执行
    - [compute01 ~]# virsh secret-define --file /etc/ceph/secret.xml
    - [compute02 ~]# virsh secret-define --file /etc/ceph/secret.xml
- 设置秘钥的值为client.cinder用户的key，Libvirt;凭此key就能以Cinder的用户访问Ceph集群
    - [compute01 ~]# virsh secret-set-value --secret 8dac896c-59b4-45a7-8cad-016d561992b0 --base64 $(cat /etc/ceph/client.cinder.key)
    - [compute02 ~]# virsh secret-set-value --secret 8dac896c-59b4-45a7-8cad-016d561992b0 --base64 $(cat /etc/ceph/client.cinder.key)
- verify:
    - virsh secret-list
## glance integrate with ceph
- Glance 为 OpenStack 提供镜像及其元数据注册服务，Glance 支持对接多种后端存储。与 Ceph 完成对接后，Glance 上传的 Image 会作为块设备储存在 Ceph 集群中。新版本的 Glance 也开始支持 enabled_backends 了，可以同时对接多个存储提供商。
- 写时复制技术(copy-on-write)：内核只为新生成的子进程创建虚拟空间结构，它们复制于父进程的虚拟空间结构，但是不为这些段分配物理内存，它们共享父进程的物理空间，当父子进程中有更改相应的段的行为发生时，再为子进程相应的段分配物理空间。写时复制技术大大降低了进程对资源的浪费。
### config glance-api.conf
- all controller nodes
- cp /etc/glance/glance-api.conf{,.bak2}
```shell
#删除glance-api如下的默认配置
[glance_store]
Local File  
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/
```
```shell
#启用映像的写时复制
openstack-config --set /etc/glance/glance-api.conf DEFAULT show_image_direct_url True
#变更默认使用的本地文件存储为ceph rbd存储
openstack-config --set /etc/glance/glance-api.conf glance_store stores rbd
openstack-config --set /etc/glance/glance-api.conf glance_store default_store rbd
openstack-config --set /etc/glance/glance-api.conf glance_store rbd_store_pool openstack-images
openstack-config --set /etc/glance/glance-api.conf glance_store rbd_store_user glance
openstack-config --set /etc/glance/glance-api.conf glance_store rbd_store_ceph_conf /etc/ceph/ceph.conf
openstack-config --set /etc/glance/glance-api.conf glance_store rbd_store_chunk_size 8
```
- systemctl restart openstack-glance-api.service
- lsof -i:9292
### test
- `wget -c http://download.cirros-cloud.net/0.5.1/cirros-0.5.1-x86_64-disk.img`
- qemu-img info cirros-0.5.1-x86_64-disk.img
- qemu-img convert -f qcow2 -O raw cirros-0.5.1-x86_64-disk.img  cirros-0.5.1-x86_64-disk.raw
- ls
- qemu-img info cirros-0.5.1-x86_64-disk.raw
- openstack image create --container-format bare --disk-format raw --file cirros-0.5.1-x86_64-disk.raw --protected --public cirros_raw
    - 注意：如果在创建 image 时候出现 AttributeError: 'NoneType' object has no attribute 'Rados' 错误，则需要安装 python-rados
    - 此时可能会报错误：
        - BadRequestException: 400: Client Error for url: http://10.10.10.69:9292/v2/images, 400 Bad Request: Failed validating &#x27;type&#x27; in schema[&#x27;properties&#x27;][&#x27;protected&#x27;]:: On instance[&#x27;protected&#x27;]:: &#x27;True&#x27;: {&#x27;description&#x27;: &#x27;If true, image will not be deletable.&#x27;,: Provided object does not match schema &#x27;image&#x27;: &#x27;True&#x27; is not of type &#x27;boolean&#x27;: &#x27;type&#x27;: &#x27;boolean&#x27;}
        - 命令行中删除 --protected 即可。可能是api接口bug，不能设置portected属性。
## cinder integrate with ceph
### cinder config (apply to cinder-volume.service nodes)
- cp /etc/cinder/cinder.conf{,.bak2}
```shell
#后端使用ceph存储已经在部署cinder服务时进行配置
openstack-config --set /etc/cinder/cinder.conf DEFAULT enabled_backends ceph
openstack-config --set /etc/cinder/cinder.conf DEFAULT glance_api_version 2

openstack-config --set /etc/cinder/cinder.conf ceph volume_driver cinder.volume.drivers.rbd.RBDDriver
openstack-config --set /etc/cinder/cinder.conf ceph rbd_pool openstack-volumes
openstack-config --set /etc/cinder/cinder.conf ceph rbd_ceph_conf /etc/ceph/ceph.conf
openstack-config --set /etc/cinder/cinder.conf ceph rbd_flatten_volume_from_snapshot false
openstack-config --set /etc/cinder/cinder.conf ceph rbd_max_clone_depth 5
openstack-config --set /etc/cinder/cinder.conf ceph rbd_store_chunk_size 4
openstack-config --set /etc/cinder/cinder.conf ceph rados_connect_timeout -1
openstack-config --set /etc/cinder/cinder.conf ceph rbd_user cinder
#注意替换cinder用户访问ceph集群使用的Secret UUID
openstack-config --set /etc/cinder/cinder.conf ceph rbd_secret_uuid 8dac896c-59b4-45a7-8cad-016d561992b0
openstack-config --set /etc/cinder/cinder.conf ceph volume_backend_name ceph
```
- systemctl restart openstack-cinder-volume.service
- systemctl status openstack-cinder-volume.service
- openstack volume service list
```shell
+------------------+-----------------------------+------+---------+-------+----------------------------+
| Binary           | Host                        | Zone | Status  | State | Updated At                 |
+------------------+-----------------------------+------+---------+-------+----------------------------+
| cinder-scheduler | openstack-controller01      | nova | enabled | up    | 2021-09-27T01:33:36.000000 |
| cinder-scheduler | openstack-controller03      | nova | enabled | up    | 2021-09-27T01:33:35.000000 |
| cinder-scheduler | openstack-controller02      | nova | enabled | up    | 2021-09-27T01:33:29.000000 |
| cinder-volume    | openstack-controller01@ceph | nova | enabled | up    | 2021-09-27T01:33:34.000000 |
| cinder-volume    | openstack-controller02@ceph | nova | enabled | up    | 2021-09-27T01:33:29.000000 |
| cinder-volume    | openstack-controller03@ceph | nova | enabled | up    | 2021-09-27T01:33:35.000000 |
+------------------+-----------------------------+------+---------+-------+----------------------------+
```
### test
#### a. create blank volume
- cinder type-create ceph
- cinder type-key ceph set volume_backend_name=ceph
- cinder extra-specs-list
- `openstack volume create --type ceph --size 1 ceph-volume`
- openstack volume list
#### b. create volume depends the image
- openstack image list
- openstack volume create --image xxx(image-id) --type ceph --size 1 cirros_raw_image
- verify
    - openstack volume list
    - rbd ls openstack-volumes
    - rbd info openstack-volumes/volume-xxx
    - rados ls -p openstack-volumes
- 为镜像创建的卷生成快照测试
    - openstack volume snapshot create --volume cirros_raw_image cirros_raw_image_snap01
```shell
+-------------+--------------------------------------+
| Field       | Value                                |
+-------------+--------------------------------------+
| created_at  | 2021-09-27T01:41:33.180200           |
| description | None                                 |
| id          | a6a51b94-1b67-4d5c-bcaf-807a027e0a7a |
| name        | cirros_raw_image_snap01              |
| properties  |                                      |
| size        | 1                                    |
| status      | creating                             |
| updated_at  | None                                 |
| volume_id   | 6fc28c1f-9c35-4a5d-8e20-a8dccc59c45b |
+-------------+--------------------------------------+
```
- openstack volume snapshot list
    - rbd snap ls volumes/volume-6fc28c1f-9c35-4a5d-8e20-a8dccc59c45b
## nove virtual machine storage with ceph
- Nova是OpenStack中的计算服务。 Nova存储与默认的运行虚拟机相关联的虚拟磁盘镜像，在/var/lib/nova/instances/%UUID目录下。Ceph是可以直接与Nova集成的存储后端之一。
- Nova 为 OpenStack 提供计算服务，对接 Ceph 主要是希望将实例的系统磁盘文件储存到 Ceph 集群中。与其说是对接 Nova，更准确来说是对接QEMU-KVM/libvirt，因为 librbd 早已原生集成到其中。
- 如果需要从ceph rbd中启动虚拟机，必须将ceph配置为nova的临时后端；推荐在计算节点的配置文件中启用rbd cache功能；为了便于故障排查，配置admin socket参数，这样每个使用ceph rbd的虚拟机都有1个socket将有利于虚拟机性能分析与故障解决；相关配置只涉及全部计算节点ceph.conf文件的[client]与[client.cinder]字段.
### compute node config
- 创建ceph.conf文件中指定的socker与log相关的目录，并更改属主
    - mkdir -p /var/run/ceph/guests/ /var/log/qemu/
    - chown qemu:libvirt /var/run/ceph/guests/ /var/log/qemu/
- vim /etc/ceph/ceph.conf
```shell
[client]
rbd cache = true
rbd cache writethrough until flush = true
admin socket = /var/run/ceph/guests/$cluster-$type.$id.$pid.$cctid.asok
log file = /var/log/qemu/qemu-guest-$pid.log
rbd concurrent management ops = 20
```

- cp /etc/nova/nova.conf{,.bak2}
```shell
#有时候碰到硬盘太大,比如需要创建80G的虚拟机,则会创建失败,需要修改nova.conf里面的vif超时参数
openstack-config --set /etc/nova/nova.conf DEFAULT vif_plugging_timeout 180
openstack-config --set /etc/nova/nova.conf DEFAULT vif_plugging_is_fatal False

#支持虚拟机硬件加速;前面已添加
openstack-config --set /etc/nova/nova.conf libvirt virt_type kvm
openstack-config --set /etc/nova/nova.conf libvirt images_type rbd
openstack-config --set /etc/nova/nova.conf libvirt images_rbd_pool openstack-vms
openstack-config --set /etc/nova/nova.conf libvirt images_rbd_ceph_conf /etc/ceph/ceph.conf
openstack-config --set /etc/nova/nova.conf libvirt rbd_user cinder
openstack-config --set /etc/nova/nova.conf libvirt rbd_secret_uuid 8dac896c-59b4-45a7-8cad-016d561992b0

openstack-config --set /etc/nova/nova.conf libvirt disk_cachemodes \"network=writeback\"
openstack-config --set /etc/nova/nova.conf libvirt live_migration_flag \"VIR_MIGRATE_UNDEFINE_SOURCE,VIR_MIGRATE_PEER2PEER,VIR_MIGRATE_LIVE,VIR_MIGRATE_PERSIST_DEST,VIR_MIGRATE_TUNNELLED\"

#禁用文件注入
openstack-config --set /etc/nova/nova.conf libvirt inject_password false
openstack-config --set /etc/nova/nova.conf libvirt inject_key false
openstack-config --set /etc/nova/nova.conf libvirt inject_partition -2 
#虚拟机临时root磁盘discard功能;unmap参数在scsi接口类型磁盘释放后可立即释放空间
openstack-config --set /etc/nova/nova.conf libvirt hw_disk_discard unmap
```
- systemctl restart libvirtd.service openstack-nova-compute.service
- systemctl status libvirtd.service openstack-nova-compute.service
- vim /etc/libvirt/libvirtd.conf
```shell
#compute01
[root@compute01 ~]# egrep -vn "^$|^#" /etc/libvirt/libvirtd.conf 
20:listen_tls = 0
34:listen_tcp = 1
52:tcp_port = "16509"
#取消注释,并修改监听端口
65:listen_addr = "10.10.10.54"
#取消注释,同时取消认证
167:auth_tcp = "none"

#compute02
[root@compute02 ~]# egrep -vn "^$|^#" /etc/libvirt/libvirtd.conf 
20:listen_tls = 0
34:listen_tcp = 1
52:tcp_port = "16509"
65:listen_addr = "10.10.10.55"
167:auth_tcp = "none"
```
- vim /etc/sysconfig/libvirtd: LIBVIRTD_ARGS="--listen"
- 免密互相登录：
    - yum install sshpass -y
    - usermod  -s /bin/bash nova
    - passwd nova
    - su nova
    - ssh-keygen -t rsa -f ~/.ssh/id_rsa -P ''
    - ssh-copy-id -i openstack-compute0x
- systemctl mask libvirtd.socket libvirtd-ro.socket libvirtd-admin.socket libvirtd-tls.socket libvirtd-tcp.socket
- service libvirtd restart
- systemctl restart openstack-nova-compute.service
    - netstat -lantp|grep libvirtd
