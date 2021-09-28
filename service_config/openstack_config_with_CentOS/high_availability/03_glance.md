---
title: OpenStack HA glance config
date: 2021-3-30
tags: [OpenStack, HA]
categories: 服务配置
---

# prepare
- create database for glance:
```shell
mysql -u root -p
CREATE DATABASE glance;
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'xxx';
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'xxx';
flush privileges;
```
- `source admin-openrc`
- `openstack project create --domain default --description "Service Project" service`
- `openstack user create --domain default --password xxx glance`
- `openstack role add --project service --user glance admin`
- `openstack service create --name glance --description "OpenStack Image" image`
- `openstack endpoint create --region RegionOne image public http://10.10.10.69:9292`
- `openstack endpoint create --region RegionOne image internal http://10.10.10.69:9292`
- `openstack endpoint create --region RegionOne image admin http://10.10.10.69:9292`

# config glance
- config glance.conf:
```shell
openstack-config --set /etc/glance/glance-api.conf DEFAULT bind_host 10.10.10.51
openstack-config --set /etc/glance/glance-api.conf database connection  mysql+pymysql://glance:xxx@10.10.10.69/glance
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
openstack-config --set /etc/glance/glance-api.conf keystone_authtoken password xxx
openstack-config --set /etc/glance/glance-api.conf paste_deploy flavor keystone
```
- `scp -rp /etc/glance/glance-api.conf openstack-controller02:/etc/glance/glance-api.conf`
- `scp -rp /etc/glance/glance-api.conf openstack-controller03:/etc/glance/glance-api.conf`
- [controller02~ ]# `openstack-config --set /etc/glance/glance-api.conf DEFAULT bind_host 10.10.10.52`
- [controller02~ ]# `openstack-config --set /etc/glance/glance-api.conf DEFAULT bind_host 10.10.10.53`

- create image storage folder at all controller nodes:
    -` mkdir /var/lib/glance/images/`
    - `chown glance:nobody /var/lib/glance/images`
- sync glance database: `su -s /bin/sh -c "glance-manage db_sync" glance`
    - verify: `mysql -uglance -p -e "use glance;show tables;"`

# verify image server
- `source admin-openrc`
- `wget -c http://download.cirros-cloud.net/0.5.1/cirros-0.5.1-x86_64-disk.img`
- `openstack image create --file ~/cirros-0.5.1-x86_64-disk.img --disk-format qcow2 --container-format bare --public cirros-qcow2`
- `openstack image list`

# add pcs source
- `pcs resource create openstack-glance-api systemd:openstack-glance-api clone interleave=true`
- `pcs resource`