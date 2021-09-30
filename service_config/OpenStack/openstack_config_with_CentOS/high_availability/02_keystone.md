---
title: OpenStack HA keystone config
date: 2021-3-30
tags: [OpenStack, HA]
categories: 服务配置
---

# prepare
- vim prepare-openstack-controller-keystone.yml:
```shell
---
- name: controller keystone
  gather_facts: no
  hosts: openstack-controller
  tasks:
    - name: Install keystone
      yum:
        name: [openstack-keystone, httpd, mod_wsgi, mod_ssl]
        state: present
        update_cache: yes
    - name: backup keysthone config
      shell: |
        cp /etc/keystone/keystone.conf{,.bak}
        egrep -v '^$|^#' /etc/keystone/keystone.conf.bak >/etc/keystone/keystone.conf
```
- ansible-playbook -i hosts prepare-openstack-controller-keystone.yml
- [controller01~ ]# mysql -u root -p
```shell
CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'xxx';
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'xxx';
flush privileges;
exit
```

# keystone config
- vim /etc/keystone/keystone.conf
```shell
[cache]
backend = oslo_cache.memcache_pool
enabled = true
memcache_servers = openstack-controller01:11211,openstack-controller02:11211,openstack-controller03:11211

[database]
connection = mysql+pymysql://keystone:xxx@10.10.10.69/keystone

[token]
provider = fernet
```
- scp -rp /etc/keystone/keystone.conf openstack-controller02:/etc/keystone/keystone.conf
- scp -rp /etc/keystone/keystone.conf openstack-controller03:/etc/keystone/keystone.conf

- sync keystone database:
    - `su -s /bin/sh -c "keystone-manage db_sync" keystone`
    - verify:`mysql -uroot -p  keystone  -e "show  tables";`
- init Fernet secret key:
    - `keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone`
    - `keystone-manage credential_setup --keystone-user keystone --keystone-group keystone`
    - `scp -rp /etc/keystone/fernet-keys /etc/keystone/credential-keys openstack-controller02:/etc/keystone/`
    - `scp -rp /etc/keystone/fernet-keys /etc/keystone/credential-keys openstack-controller03:/etc/keystone/`
    - [controller02 | 03~ ]# `chown -R keystone:keystone /etc/keystone/credential-keys/`
    - [controller02 | 03~ ]# `chown -R keystone:keystone /etc/keystone/fernet-keys/ `
- bootstrap: 
```shell
keystone-manage bootstrap --bootstrap-password xxx \
    --bootstrap-admin-url http://10.10.10.69:5000/v3/ \
    --bootstrap-internal-url http://10.10.10.69:5000/v3/ \
    --bootstrap-public-url http://10.10.10.69:5000/v3/ \
    --bootstrap-region-id RegionOne
```

# http server
- apply to every controller node
- httpd.conf:
    - `cp /etc/httpd/conf/httpd.conf{,.bak}`
    - `sed -i "s/#ServerName www.example.com:80/ServerName ${HOSTNAME}/" /etc/httpd/conf/httpd.conf`
```shell
##controller01
sed -i "s/Listen\ 80/Listen\ 10.10.10.51:80/g" /etc/httpd/conf/httpd.conf
##controller02
sed -i "s/Listen\ 80/Listen\ 10.10.10.52:80/g" /etc/httpd/conf/httpd.conf
##controller03
sed -i "s/Listen\ 80/Listen\ 10.10.10.53:80/g" /etc/httpd/conf/httpd.conf
```
- config wsgi-keystone.conf
    - ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/
```shell
#不同的节点替换不同的ip地址
##controller01
sed -i "s/Listen\ 5000/Listen\ 10.10.10.51:5000/g" /etc/httpd/conf.d/wsgi-keystone.conf
sed -i "s#*:5000#10.10.10.51:5000#g" /etc/httpd/conf.d/wsgi-keystone.conf
##controller02
sed -i "s/Listen\ 5000/Listen\ 10.10.10.52:5000/g" /etc/httpd/conf.d/wsgi-keystone.conf
sed -i "s#*:5000#10.10.10.52:5000#g" /etc/httpd/conf.d/wsgi-keystone.conf
##controller03
sed -i "s/Listen\ 5000/Listen\ 10.10.10.53:5000/g" /etc/httpd/conf.d/wsgi-keystone.conf
sed -i "s#*:5000#10.10.10.53:5000#g" /etc/httpd/conf.d/wsgi-keystone.conf
```
- systemctl restart httpd.service
- systemctl enable httpd.service
- systemctl status httpd.service

# user parameter script
- admin-openrc:
```shell
cat >> ~/admin-openrc << EOF
#admin-openrc
export OS_USERNAME=admin
export OS_PASSWORD=w1zlcjmK
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://10.10.10.69:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
EOF
```
- `source  ~/admin-openrc`
- `scp -rp ~/admin-openrc openstack-controller02:~/`
- `scp -rp ~/admin-openrc openstack-controller03:~/`
- verify:
    - `openstack domain list`
    - `openstack token issue`

# create domain project user role
- openstack domain create --description "An Example Domain" example
- openstack project create --domain default --description "demo Project" demo
- openstack user create --domain default --password 123456 demo
- openstack role create user
- openstack role list
- openstack role add --project demo --user  demo user
- demo-openrc:
```shell
cat >> ~/demo-openrc << EOF
#demo-openrc
export OS_USERNAME=demo
export OS_PASSWORD=123456
export OS_PROJECT_NAME=
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://10.10.10.69:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
EOF
```
- source  ~/demo-openrc
- scp -rp ~/demo-openrc openstack-controller02:~/
- scp -rp ~/demo-openrc openstack-controller03:~/
- verify: openstack token issue
- verify keystone:
    - admin user request the authentication token:
```shell
source admin-openrc
openstack --os-auth-url http://10.10.10.69:5000/v3 \
  --os-project-domain-name Default --os-user-domain-name Default \
  --os-project-name admin --os-username admin token issue
```
    - demo user request the authentication token:
```shell
source demo-openrc
openstack --os-auth-url http://10.10.10.69:5000/v3 \
  --os-project-domain-name Default --os-user-domain-name Default \
  --os-project-name demo --os-username demo token issue
```

# set pcs source
- pcs resource create openstack-keystone systemd:httpd clone interleave=true
- pcs resource