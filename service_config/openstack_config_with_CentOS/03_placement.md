---
title: OpenStack 3 placement
date: 2020-10-21
tags: [OpenStack, placement]
categories: 服务配置
---

## prepase
- mysql -u root -p
```shell
CREATE DATABASE placement;
GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'localhost' IDENTIFIED BY 'PLACEMENT_DBPASS';
GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%' IDENTIFIED BY 'PLACEMENT_DBPASS';
```
- . admin-openrc
- openstack user create --domain default --password-prompt placement
- openstack role add --project service --user placement admin
- openstack service create --name placement --description "Placement API" placement
- openstack endpoint create --region RegionOne placement public http://controller:8778
- openstack endpoint create --region RegionOne placement internal http://controller:8778
- openstack endpoint create --region RegionOne placement admin http://controller:8778

## install
- yum install openstack-placement-api
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
password = xxx
```
- su -s /bin/sh -c "placement-manage db sync" placement
- systemctl restart httpd

## verify
- . admin-openrc
- placement-status upgrade check
- Run some commands against the placement API:
    - download pip-xxx.tar.gz
    - `cp pip.tar.gz /usr/local`
    - `tar -xvzf pip.tar.gz`
    - `cd pip`
    - `python setup.py install`
    - update with proxy: `pip install --proxy="http://x.x.x.x:x" --upgrade pip`
    - `pip install --proxy="http://x.x.x.x:x" osc-placement`
    - list available resource classes:
    ```shell
    openstack --os-placement-api-version 1.2 resource class list --sort-column name
    openstack --os-placement-api-version 1.6 trait list --sort-column name
    ```