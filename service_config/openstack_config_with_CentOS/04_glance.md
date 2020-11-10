---
title: OpenStack 4 glance
date: 2020-10-21
tags: [OpenStack, glance]
categories: 服务配置
---

## prepare
- mysql -u root -p
```shell
CREATE DATABASE glance;
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' 
IDENTIFIED BY 'GLANCE_DBPASS';
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'GLANCE_DBPASS';
```
- . admin-openrc
- openstack user create --domain default --password-prompt glance
- openstack role add --project service --user glance admin
- openstack service create --name glance --description "OpenStack Image" image
- openstack endpoint create --region RegionOne image public http://controller:9292
- openstack endpoint create --region RegionOne image internal http://controller:9292
- openstack endpoint create --region RegionOne image admin http://controller:9292

## install & config
- yum install openstack-glance
- vim /etc/glance/glance-api.conf
```shell
[database]
# ...
connection = mysql+pymysql://glance:GLANCE_DBPASS@controller/glance

[keystone_authtoken]
# ...
www_authenticate_uri  = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = xxx

[paste_deploy]
# ...
flavor = keystone

[glance_store]
# ...
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/
```
- vim /etc/glance/glance-registry.conf
```shell
[database]
# ...
connection = mysql+pymysql://glance:GLANCE_DBPASS@controller/glance

[keystone_authtoken]
# ...
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = xxx

[paste_deploy]
# ...
flavor = keystone
```
- su -s /bin/sh -c "glance-manage db_sync" glance
- systemctl enable openstack-glance-api.service   openstack-glance-registry.service
- systemctl start openstack-glance-api.service   openstack-glance-registry.service

## verify
- . admin-openrc
- wget http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img
- openstack image create "cirros" --file cirros-0.4.0-x86_64-disk.img --disk-format qcow2 --container-format bare --public
- openstack image list