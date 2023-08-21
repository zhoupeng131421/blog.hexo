---
title: OpenStack 1 keystone
date: 2020-10-21
tags: [keystone, OpenStack]
categories: 服务配置
---

# Based config

## hostname
- hostnamectl set-hostname controller
- vim /etc/hosts:
```shell
192.168.1.40 network
192.168.1.41 controller
192.168.1.42 block
192.168.1.43 computer
```

## network
- vim /etc/netplan/xxxx.yaml
```shell
    ens192:
      dhcp4: yes
    ens224:
      dhcp4: false
      addresses: [192.168.10.10/24]
```
- disable ufw: `service ufw stop` `ufw disable`
## network time protocol
- setup timezone:
	- `tzselect` -> Asia -> China -> Beijing
	- cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
- apt install chrony
- vim /etc/chrony/chrony.conf
	```shell
	server 192.168.1.41 iburst
	allow 192.168.1.0/24
	```
- service chrony restart
- chronyc sources

## Openstack package
- add-apt-repository cloud-archive:stein
- apt update && apt dist-upgrade
- apt install python3-openstackclient

## sql database
- apt install mariadb-server python-pymysql
- vim /etc/mysql/mariadb.conf.d/99-openstack.cnf
```shell
[mysqld]
bind-address = 192.168.1.41

default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
```
- service mysql restart
- mysql_secure_installation

## message queue server
- apt install rabbitmq-server
- rabbitmqctl add_user openstack RABBIT_PASS
- rabbitmqctl set_permissions openstack ".*" ".*" ".*"

## memcached
- apt install memcached python-memcache
- vim /etc/memcached.conf
```shell
-l 192.168.1.41
```
- service memcached restart

# keystone
## prepare
- mysql
```shell
MariaDB [(none)]> CREATE DATABASE keystone;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'KEYSTONE_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'KEYSTONE_DBPASS';
MariaDB [(none)]> exit;
```

## install
- apt install keystone
- vim /etc/keystone/keystone.conf
```shell
[database]
# ...
connection = mysql+pymysql://keystone:KEYSTONE_DBPASS@controller/keystone
# ...
[token]
# ...
provider = fernet
```
- su -s /bin/sh -c "keystone-manage db_sync" keystone
- keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
- keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
- keystone-manage bootstrap --bootstrap-password ADMIN_PASS --bootstrap-admin-url http://controller:5000/v3/ --bootstrap-internal-url http://controller:5000/v3/ --bootstrap-public-url http://controller:5000/v3/ --bootstrap-region-id RegionOne

## config Apache2 server
- vim /etc/apache2/apache2.conf
```shell
ServerName controller
```
- service apache2 restart
- config env
```shell
export OS_USERNAME=admin
export OS_PASSWORD=ADMIN_PASS
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
```

## Create a domain, projects, users, and roles
- openstack domain create --description "An Example Domain" example
	- there will have error: `[openstack]"The request you have made requires authentication. (HTTP 401)l"`
	- /bin/sh -c "keystone-manage db_sync"
	- keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
	- keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
	- keystone-manage bootstrap --bootstrap-password ADMIN_PASS --bootstrap-admin-url http://controller:5000/v3/ --bootstrap-internal-url http://controller:5000/v3/ --bootstrap-public-url http://controller:5000/v3/ --bootstrap-region-id RegionOne
	- I think this is a bug, we need reconfig thouse parameters
- openstack project create --domain default --description "Service Project" service
- openstack project create --domain default --description "Demo Project" myproject
- openstack user create --domain default --password-prompt myuser
- openstack role create myrole
- openstack role add --project myproject --user myuser myrole
### verify operation
- unset OS_AUTH_URL OS_PASSWORD
- openstack --os-auth-url http://controller:5000/v3 --os-project-domain-name Default --os-user-domain-name Default --os-project-name admin --os-username admin token issue
- openstack --os-auth-url http://controller:5000/v3 --os-project-domain-name Default --os-user-domain-name Default --os-project-name myproject --os-username myuser token issue
### Use script config openstack client environment
- admin project:
	- vim admin-openrc
	```shell
	export OS_PROJECT_DOMAIN_NAME=Default
	export OS_USER_DOMAIN_NAME=Default
	export OS_PROJECT_NAME=admin
	export OS_USERNAME=admin
	export OS_PASSWORD=ADMIN_PASS
	export OS_AUTH_URL=http://controller:5000/v3
	export OS_IDENTITY_API_VERSION=3
	export OS_IMAGE_API_VERSION=2
	```
	- . admin-openrc
	- openstack token issue
- demo porject:
	- vim demo-openrc
	```shell
	export OS_PROJECT_DOMAIN_NAME=Default
	export OS_USER_DOMAIN_NAME=Default
	export OS_PROJECT_NAME=myproject
	export OS_USERNAME=myuser
	export OS_PASSWORD=demo_pass
	export OS_AUTH_URL=http://controller:5000/v3
	export OS_IDENTITY_API_VERSION=3
	export OS_IMAGE_API_VERSION=2
	```
	- . demo-openrc
	- openstack token issue


