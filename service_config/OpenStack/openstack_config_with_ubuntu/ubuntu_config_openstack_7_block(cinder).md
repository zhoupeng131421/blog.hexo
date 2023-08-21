---
title: OpenStack 6 block
date: 2020-10-21
tags: [OpenStack, cinder]
categories: 服务配置
---

# controller node
- mysql -u root -p
```shell
MariaDB [(none)]> CREATE DATABASE cinder;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'localhost' IDENTIFIED BY 'CINDER_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' IDENTIFIED BY 'CINDER_DBPASS';
MariaDB [(none)]> exit
```
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

- apt install cinder-api cinder-scheduler
- vim /etc/cinder/cinder.conf
```shell
[database]
# ...
connection = mysql+pymysql://cinder:CINDER_DBPASS@controller/cinder

[DEFAULT]
# ...
transport_url = rabbit://openstack:RABBIT_PASS@controller
auth_strategy = keystone
my_ip = 192.168.1.41

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
password = cinder_pass

[oslo_concurrency]
# ...
lock_path = /var/lib/cinder/tmp
```
- su -s /bin/sh -c "cinder-manage db sync" cinder

- vim /etc/nova/nova.conf
```shell
[cinder]
os_region_name = RegionOne
```

- service nova-api restart
- service cinder-scheduler restart
- service apache2 restart


# storage node
- apt install lvm2 thin-provisioning-tools
- pvcreate /dev/sdb
- vgcreate cinder-volumes /dev/sdb
- vim /etc/lvm/lvm.conf
```shell
devices{
...
filter = [ "a/sdb/", "r/.*/"]
# if the system use lvm, we need add "a/ada/" --> filter = [ "a/sda/", "a/sdb/", "r/.*/"]
# if the compute node use the lvm, we also to config the filter --> filter = [ "a/sda/", "r/.*/"]
...
}
```
- apt install cinder-volume
- vim `etc/cinder/cinder.conf`
```shell
[database]
# ...
connection = mysql+pymysql://cinder:CINDER_DBPASS@controller/cinder

[DEFAULT]
# ...
transport_url = rabbit://openstack:RABBIT_PASS@controller
auth_strategy = keystone
my_ip = 192.168.1.42
enabled_backends = lvm
glance_api_servers = http://controller:9292

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
password = cinder_pass

[lvm]
# ...
volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
volume_group = cinder-volumes
target_protocol = iscsi
target_helper = tgtadm

[oslo_concurrency]
# ...
lock_path = /var/lib/cinder/tmp
```

- service tgt restart
- service cinder-volume restart