---
title: OpenStack 3 glance
date: 2020-10-21
tags: [OpenStack, glance]
categories: 服务配置
---

> glance is a image server, on the controller node.

## prepare
### database
- mysql
```shell
MariaDB [(none)]> CREATE DATABASE glance;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'GLANCE_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'GLANCE_DBPASS';
MariaDB [(none)]> exit;
```
### Create the service credentials
- .admin-openrc
- openstack user create --domain default --password-prompt glance
- openstack role add --project service --user glance admin
- openstack service create --name glance --description "OpenStack Image" image
- openstack endpoint create --region RegionOne image public http://controller:9292
- openstack endpoint create --region RegionOne image internal http://controller:9292
- openstack endpoint create --region RegionOne image admin http://controller:9292

## install and configure
- apt install glance
- vim /etc/glance/glance-api.conf
```shell
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
password = glance_pass
# ...
[paste_deploy]
# ...
flavor = keystone
# ...
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
# ...
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
password = glance_pass
# ...
[paste_deploy]
# ...
flavor = keystone
```
- su -s /bin/sh -c "glance-manage db_sync" glance
- service glance-registry restart
    - Bug: there is not have the service glance-registry at /etc/init.d/
    - cp /etc/init.d/glance-api /etc/init.d/glance-registry
    - vim /etc/init.d/glance-registry
        - replace all 'api' to 'registry'
- service glance-api restart

## verify
- source admin-openrc
- wget http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img
- openstack image create "cirros" --file cirros-0.4.0-x86_64-disk.img --disk-format qcow2 --container-format bare --public
- openstack image list