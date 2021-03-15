---
title: OpenStack 9 block_storage(cinder)
date: 2020-10-21
tags: [OpenStack, cinder]
categories: 服务配置
---

# controller node
## prepare
- mysql -u root -p
```shell
CREATE DATABASE cinder;
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'localhost' IDENTIFIED BY 'CINDER_DBPASS';
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' IDENTIFIED BY 'CINDER_DBPASS';
```
- . admin-openrc
- openstack user create --domain default --password-prompt cinder
- openstack role add --project service --user cinder admin
- openstack service create --name cinderv2 --description "OpenStack Block Storage" volumev2
- openstack service create --name cinderv3 --description "OpenStack Block Storage" volumev3
- openstack endpoint create --region RegionOne volumev2 public http://controller:8776/v2/%\(project_id\)s
- openstack endpoint create --region RegionOne volumev2 internal http://controller:8776/v2/%\(project_id\)s
- openstack endpoint create --region RegionOne volumev2 admin http://controller:8776/v2/%\(project_id\)s
- openstack endpoint create --region RegionOne volumev3 public http://controller:8776/v3/%\(project_id\)s
- openstack endpoint create --region RegionOne volumev3 internal http://controller:8776/v3/%\(project_id\)s
- openstack endpoint create --region RegionOne volumev3 admin http://controller:8776/v3/%\(project_id\)s

## install & config
- yum install openstack-cinder
- vim /etc/cinder/cinder.conf
```shell
[database]
# ...
connection = mysql+pymysql://cinder:CINDER_DBPASS@controller/cinder

[DEFAULT]
# ...
transport_url = rabbit://openstack:RABBIT_PASS@controller

[DEFAULT]
# ...
auth_strategy = keystone

[keystone_authtoken]
# ...
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = cinder
password = xxx

[DEFAULT]
# ...
# manager network ipaddr
my_ip = x.x.x.x

[oslo_concurrency]
# ...
lock_path = /var/lib/cinder/tmp
```
- su -s /bin/sh -c "cinder-manage db sync" cinder
- vim /etc/nova/nova.conf: `[cinder]: os_region_name = RegionOne`
- systemctl restart openstack-nova-api.service
- systemctl enable openstack-cinder-api.service openstack-cinder-scheduler.service
- systemctl start openstack-cinder-api.service openstack-cinder-scheduler.service

# block node
## prepare
- yum install lvm2 device-mapper-persistent-data
- systemctl enable lvm2-lvmetad.service
- systemctl start lvm2-lvmetad.service
- pvcreate /dev/sdb
- pvcreate /dev/sdc
- vgcreate cinder-volumes /dev/sdb /dev/sdc

- vim /etc/lvm/lvm.conf
```shell
devices{
...
filter = [ "a/sdb/", "a/sdc/","r/.*/"]
# if the system use lvm, we need add "a/ada/" --> filter = [ "a/sda/", "a/sdb/", "r/.*/"]
# if the compute node use the lvm, we also to config the filter --> filter = [ "a/sda/", "r/.*/"]
...
}
```
## install & config
- yum install openstack-cinder targetcli python-keystone
- vim /etc/cinder/cinder.conf
```shell
[database]
# ...
connection = mysql+pymysql://cinder:CINDER_DBPASS@controller/cinder

[DEFAULT]
# ...
transport_url = rabbit://openstack:RABBIT_PASS@controller

[DEFAULT]
# ...
auth_strategy = keystone

[keystone_authtoken]
# ...
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = cinder
password = xxx

[DEFAULT]
# ...
# maanger network ipaddr
my_ip = MANAGEMENT_INTERFACE_IP_ADDRESS

[lvm]
volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
volume_group = cinder-volumes
target_protocol = iscsi
target_helper = lioadm

[DEFAULT]
# ...
enabled_backends = lvm

[DEFAULT]
# ...
glance_api_servers = http://controller:9292

[oslo_concurrency]
# ...
lock_path = /var/lib/cinder/tmp
```
- systemctl enable openstack-cinder-volume.service target.service
- systemctl start openstack-cinder-volume.service target.service

# verify
- controller node: `. admin-openrc` `openstack volume service list`