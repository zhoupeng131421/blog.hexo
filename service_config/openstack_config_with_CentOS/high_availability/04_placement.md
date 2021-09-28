---
title: OpenStack HA placement config
date: 2021-3-30
tags: [OpenStack, HA]
categories: 服务配置
---

# create database for placement:
```shell
mysql -u root -p
CREATE DATABASE placement;
GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'localhost' IDENTIFIED BY 'xxx';
GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%' IDENTIFIED BY 'xxx';
flush privileges;
```

# create placement endpoint:
- `openstack user create --domain default --password=xxx placement`
- `openstack role add --project service --user placement admin`
- `openstack service create --name placement --description "Placement API" placement`
- `openstack endpoint create --region RegionOne placement public http://10.10.10.69:8778`
- `openstack endpoint create --region RegionOne placement internal http://10.10.10.69:8778`
- `openstack endpoint create --region RegionOne placement admin http://10.10.10.69:8778`

# config placement:
```shell
openstack-config --set /etc/placement/placement.conf placement_database connection mysql+pymysql://placement:xxx@10.10.10.69/placement
openstack-config --set /etc/placement/placement.conf api auth_strategy keystone
openstack-config --set /etc/placement/placement.conf keystone_authtoken auth_url  http://10.10.10.51:5000/v3
openstack-config --set /etc/placement/placement.conf keystone_authtoken memcached_servers openstack-controller01:11211,openstack-controller02:11211,openstack-controller03:11211
openstack-config --set /etc/placement/placement.conf keystone_authtoken auth_type password
openstack-config --set /etc/placement/placement.conf keystone_authtoken project_domain_name Default
openstack-config --set /etc/placement/placement.conf keystone_authtoken user_domain_name Default
openstack-config --set /etc/placement/placement.conf keystone_authtoken project_name service
openstack-config --set /etc/placement/placement.conf keystone_authtoken username placement
openstack-config --set /etc/placement/placement.conf keystone_authtoken password xxx
```
- scp /etc/placement/placement.conf openstack-controller02:/etc/placement/
- scp /etc/placement/placement.conf openstack-controller03:/etc/placement/

- sync database:
    - `su -s /bin/sh -c "placement-manage db sync" placement`
    - verify: `mysql -uroot -p placement -e " show tables;"`

# config 00-placement-api.conf
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

# pcs source
- none operate