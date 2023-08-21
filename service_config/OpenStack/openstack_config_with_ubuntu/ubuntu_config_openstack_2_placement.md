---
title: OpenStack 2 placement
date: 2020-10-21
tags: [placement, OpenStack]
categories: 服务配置
---

> this service install in the controller node before other services and after keystone

## prepare
- mysql
```shell
MariaDB [(none)]> CREATE DATABASE placement;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'localhost' IDENTIFIED BY 'PLACEMENT_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%' IDENTIFIED BY 'PLACEMENT_DBPASS';
```
- . admin-openrc
- openstack user create --domain default --password-prompt placement
- openstack role add --project service --user placement admin
- openstack service create --name placement --description "Placement API" placement
- openstack endpoint create --region RegionOne placement public http://controller:8778
- openstack endpoint create --region RegionOne placement internal http://controller:8778
- openstack endpoint create --region RegionOne placement admin http://controller:8778

## install and config
- apt install placement-api
- vim /etc/placement/placement.conf
```shell
[placement_database]
# ...
connection = mysql+pymysql://placement:PLACEMENT_DBPASS@controller/placement

[api]
# ...
auth_strategy = keystone

[keystone_authtoken]
# ...
auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = placement
password = placement_pass
```
- su -s /bin/sh -c "placement-manage db sync" placement
- service apache2 restart

## verify
- . admin-openrc
- placement-status upgrade check
- pip3 install osc-placement
	- My Ubuntu can not find the osc-placement package, so I need download the source code for osc-placement. then `tar -xvzf osc-xxx.tar.gz` `cd osc-xxx` `python3 setup.py install`
- openstack --os-placement-api-version 1.2 resource class list --sort-column name
- openstack --os-placement-api-version 1.6 trait list --sort-column name