---
title: OpenStack 2 keystone
date: 2020-10-21
tags: [OpenStack, keystone]
categories: 服务配置
---

## prepare
- `mysql -u root -p`
```shell
CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'KEYSTONE_DBPASS';
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'KEYSTONE_DBPASS';
exit
```

## install & config
- yum install openstack-keystone httpd mod_wsgi
- /etc/keystone/keystone.conf
```shell
[database]
# ...
connection = mysql+pymysql://keystone:KEYSTONE_DBPASS@controller/keystone

[token]
# ...
provider = fernet
```
- populate the keystone service database: `su -s /bin/sh -c "keystone-manage db_sync" keystone`
- keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
- keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
- keystone-manage bootstrap --bootstrap-password ADMIN_PASS   --bootstrap-admin-url http://controller:5000/v3/   --bootstrap-internal-url http://controller:5000/v3/   --bootstrap-public-url http://controller:5000/v3/   --bootstrap-region-id RegionOne
- `vim /etc/httpd/conf/httpd.conf`: `ServerName controller`
- `ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/`
- systemctl enable httpd.service
- systemctl start httpd.service
- export env:
```shell
export OS_USERNAME=admin
export OS_PASSWORD=ADMIN_PASS
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
```

## create domain, project, users, roles
- unset http_proxy
- openstack domain create --description "An Example Domain" example
- openstack project create --domain default --description "Service Project" service
- openstack project create --domain default --description "Demo Project" myproject
- openstack user create --domain default --password-prompt myuser
- openstack role create myrole
- openstack role add --project myproject --user myuser myrole

## verify
- unset OS_AUTH_URL OS_PASSWORD
- openstack --os-auth-url http://controller:5000/v3   --os-project-domain-name Default --os-user-domain-name Default   --os-project-name admin --os-username admin token issue
- openstack --os-auth-url http://controller:5000/v3   --os-project-domain-name Default --os-user-domain-name Default   --os-project-name myproject --os-username myuser token issue

## create env scripts
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
- vim demo-openrc
```shell
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=myproject
export OS_USERNAME=myuser
export OS_PASSWORD=123456
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
```