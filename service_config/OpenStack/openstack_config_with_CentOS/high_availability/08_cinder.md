---
title: OpenStack HA cinder config
date: 2021-9-14
tags: [OpenStack, HA]
categories: 服务配置
---

> Cinder的核心功能是对卷的管理，允许对卷、卷的类型、卷的快照、卷备份进行处理。它为后端不同的存储设备提供给了统一的接口，不同的块设备服务厂商在Cinder中实现其驱动，可以被Openstack整合管理，nova与cinder的工作原理类似。支持多种 back-end（后端）存储方式，包括 LVM，NFS，Ceph 和其他诸如 EMC、IBM 等商业存储产品和方案。

# Intro
- Cinder-api 是 cinder 服务的 endpoint，提供 rest 接口，负责处理 client 请求，并将 RPC 请求发送至 cinder-scheduler 组件。
- Cinder-scheduler 负责 cinder 请求调度，其核心部分就是 scheduler_driver, 作为 scheduler manager 的 driver，负责 cinder-volume 具体的调度处理，发送 cinder RPC 请求到选择的 cinder-volume。
- Cinder-volume 负责具体的 volume 请求处理，由不同后端存储提供 volume 存储空间。目前各大存储厂商已经积极地将存储产品的 driver 贡献到 cinder 社区

# prepare
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
# cinder controller node config
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
# PCS resource
- 在任意控制节点操作；添加资源cinder-api与cinder-scheduler
- cinder-api与cinder-scheduler以active/active模式运行；
- openstack-nova-volume以active/passive模式运行
```shell
pcs resource create openstack-cinder-api systemd:openstack-cinder-api clone interleave=true
pcs resource create openstack-cinder-scheduler systemd:openstack-cinder-scheduler clone interleave=true
```
- verify: pcs resource

# cinder storage node config
- 使用ceph作为后端的存储进行使用
- 在采用ceph或其他商业/非商业后端存储时，建议将`cinder-volume`服务部署在控制节点，通过pacemaker将服务运行在active/passive模式。
## cinder-volume 部署在控制节点
- all controller node:
    - #openstack-config --set /etc/cinder/cinder.conf  DEFAULT enabled_backends lvm
    - openstack-config --set /etc/cinder/cinder.conf  DEFAULT enabled_backends ceph
## cinder-volume单独部署：
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
## others
```shell
systemctl restart openstack-cinder-volume.service target.service
systemctl enable openstack-cinder-volume.service target.service
systemctl status openstack-cinder-volume.service target.service
```
- verify: openstack volume service list
- PCS resource: vim /etc/haproxy/haproxy.conf
```shell
 listen cinder_volume
  bind 10.10.10.69:8776
  balance  source
  option  tcpka
  option  httpchk
  option  tcplog
  server openstack-controller01 10.10.10.51:8776 check inter 2000 rise 2 fall 5
  server openstack-controller02 10.10.10.52:8776 check inter 2000 rise 2 fall 5
  server openstack-controller03 10.10.10.53:8776 check inter 2000 rise 2 fall 5
```
- scp haproxy.conf controller02:/etc/haproxy/haproxy.conf
- scp haproxy.conf controller03:/etc/haproxy/haproxy.conf