---
title: OpenStack HA config with Ubuntu
date: 2021-9-16
tags: [OpenStack, HA, Ubuntu]
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
## Ubuntu prepare
- set root password: sudo passwd
- permit root account via ssh: /etc/ssh/sshd_config: PermitRootLogin yes
- vim /etc/resolv.conf: nameserver 10.10.10.1
- ssh-copy-id -i root@openstack-controller01
- ssh-copy-id -i root@openstack-controller02
- ssh-copy-id -i root@openstack-controller03
- ssh-copy-id -i root@openstack-compute01
- ssh-copy-id -i root@openstack-compute01

- apt install -y net-tools vim git wget curl
- apt install -y pacemaker pcs corosync fence-agents resource-agents haproxy
- apt install -y chrony
    - controller: allow 10.10.10.0/24
    - other nodes: server openstack-controller01 iburst
```shell
modprobe br_netfilter
echo 'net.ipv4.ip_forward = 1' >>/etc/sysctl.conf
echo 'net.bridge.bridge-nf-call-iptables=1' >>/etc/sysctl.conf
echo 'net.bridge.bridge-nf-call-ip6tables=1'  >>/etc/sysctl.conf
echo 'net.ipv4.ip_nonlocal_bind = 1' >>/etc/sysctl.conf
sysctl -p
```
- add-apt-repository cloud-archive:wallaby
- apt install -y python3-openstackclient
- apt install -y mariadb-server python3-pymysql rabbitmq-server memcached python3-memcache etcd
## mariadb ha:
> Galera是一个MySQL的同步多主集群软件。能够实现同步复制，Active-active的多主拓扑结构，真正的multi-master，所有节点可以同时读写数据库，自动成员资格控制，失败节点从群集中删除，新节点加入数据自动复制，真正的并行复制，行级。

### Install and config
- (all controller nodes)
    - apt install -y mariadb-server python3-pymysql mariadb-client galera-3 galera-arbitrator-3 libmariadb3 mariadb-backup
        - because the mariadb vesion is 10.3.31, so uses the galera-3
        - ref: https://mariadb.com/kb/en/getting-started-with-mariadb-galera-cluster/
    - mysql_secure_installation
- vim /etc/mysql/conf.d/openstack.cnf
```shell
#bind-address   主机ip
#wsrep_node_name 主机名
#wsrep_node_address 主机ip
[server]

[mysqld]
bind-address = 10.10.10.51
max_connections = 1000
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
log-error=/var/log/mysql/mariadb.log
pid-file=/run/mysql/mysql.pid


[galera]
wsrep_on=ON
wsrep_provider=/usr/lib/libgalera_smm.so
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
- scp -rp /etc/mysql/cnf.d/openstack.cnf controller02:/etc/mysql/cnf.d/openstack.cnf 
- scp -rp /etc/mysql/cnf.d/openstack.cnf controller03:/etc/mysql/cnf.d/openstack.cnf
modify the bind-address wsrep_node_name and wsrep_node_address according to real info

### config cluster
- stop all DB server for controller nodes:
    - `systemctl stop mariadb`
- start mariaDB server at controller01 node:
    - galera_new_cluster
- restart mariaDB server at other nodes:
    - systemctl start mariadb
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
- all controller node:
    - apt install -y xinetd
    - `wget -P /extend/shell/ https://raw.githubusercontent.com/olafz/percona-clustercheck/master/clustercheck`
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

## rabbitmq config
- - Stop the rabbitmq-server service in the controller nodes except the controller01
- send .erlang.cookie to other controller nodes
    - scp /var/lib/rabbitmq/.erlang.cookie  openstack-controller02:/var/lib/rabbitmq/
    - scp /var/lib/rabbitmq/.erlang.cookie  openstack-controller03:/var/lib/rabbitmq/
- chang the .erlang.cookie owner at other controller nodes
    - chown rabbitmq:rabbitmq /var/lib/rabbitmq/.erlang.cookie
- start the rabbitmq-server service in the other controller nodes
    - systemctl start rabbitmq-server
- 构建集群，controller02和03节点以ram节点的形式加入集群
    - rabbitmqctl stop_app
    - rabbitmqctl join_cluster \-\-ram rabbit@openstack-controller01
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
    - `sed -i 's|127.0.0.1,::1|0.0.0.0|g' /etc/memcached.conf`
    - systemctl restart memcached.service
    - systemctl status memcached.service
### ETCD
- apply to all controller nodes:
- cp -a /etc/default/etcd{,.bak}
- [openstack-controller01 ~]#
```shell
cat << EOF | tee /etc/default/etcd
ETCD_DATA_DIR="/var/lib/etcd"
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
cat << EOF | tee /etc/default/etcd
ETCD_DATA_DIR="/var/lib/etcd"
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
cat << EOF | tee /etc/default/etcd
ETCD_DATA_DIR="/var/lib/etcd"
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
### Verify the cluster configuration
- Before we make any changes, it’s a good idea to check the validity of the configuration.
```shell
[root@controller1 ~]# crm_verify -L -V
   error: unpack_resources: Resource start-up disabled since no STONITH resources have been defined
   error: unpack_resources: Either configure some or disable STONITH with the stonith-enabled option
   error: unpack_resources: NOTE: Clusters with shared data need STONITH to ensure data integrity
Errors found during check: config not valid
As you can see, the tool has found some errors.
```
- In order to guarantee the safety of your data, [5] fencing (also called STONITH) is enabled by default. However, it also knows when no STONITH configuration has been supplied and reports this as a problem (since the cluster will not be able to make progress if a situation requiring node fencing arises).
- We will disable this feature for now and configure it later. To disable STONITH, set the stonith-enabled cluster option to false on both the controller nodes:
```shell
[root@controller1 ~]# pcs property set stonith-enabled=false
[root@controller1 ~]# crm_verify -L
```
- *WARNING*:
    - The use of stonith-enabled=false is completely inappropriate for a production cluster. It tells the cluster to simply pretend that the nodes which fails are safely in powered off state. Some vendors will refuse to support clusters that have STONITH disabled.

### further config
- any controller node:
    - some node was already running (probably just installing packages creates some default cluster), so destroyed everything (`pcs cluster destroy`), started pcsd again and then repeated the steps.
    - pcs host auth openstack-controller01 openstack-controller02 openstack-controller03 -u hacluster -p zp@131421
    - pcs cluster setup openstack-cluster-01 --start openstack-controller01 openstack-controller02 openstack-controller03
- pcs cluster start --all
- pcs cluster enable --all
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
- 设置资源启动顺序：pcs constraint order start vip_public then lb-haproxy-clone kind=Optional
    - `cibadmin --query --scope constraints` 可以查看资源约束配置
- 将两种资源约束在一个节点上：
    - 官方建议设置vip运行在haproxy active的节点，通过绑定lb-haproxy-clone与vip服务，所以将两种资源约束在1个节点；约束后，从资源角度看，其余暂时没有获得vip的节点的haproxy会被pcs关闭 
    - 这里一定要进行资源绑定，否则每个节点都会启动haproxy，造成访问混乱
    - pcs constraint colocation add lb-haproxy-clone with vip_public
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
keystone-manage bootstrap --bootstrap-password xxx \
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
