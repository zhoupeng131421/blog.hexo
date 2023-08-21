---
title: OpenStack 4 nova
date: 2020-10-21
tags: [OpenStack, nova]
categories: 服务配置
---

# controller node
## prepare
- mysql
```shell
MariaDB [(none)]> CREATE DATABASE nova_api;
MariaDB [(none)]> CREATE DATABASE nova;
MariaDB [(none)]> CREATE DATABASE nova_cell0;

MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' IDENTIFIED BY 'NOVA_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY 'NOVA_DBPASS';

MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY 'NOVA_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'NOVA_DBPASS';

MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' IDENTIFIED BY 'NOVA_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' IDENTIFIED BY 'NOVA_DBPASS';
```
- source admin-openrc
- openstack user create --domain default --password-prompt nova
- openstack role add --project service --user nova admin
- openstack service create --name nova --description "OpenStack Compute" compute
- openstack endpoint create --region RegionOne compute public http://controller:8774/v2.1
- openstack endpoint create --region RegionOne compute internal http://controller:8774/v2.1
- openstack endpoint create --region RegionOne compute admin http://controller:8774/v2.1

## install and config
- apt install nova-api nova-conductor nova-consoleauth nova-novncproxy nova-scheduler
- vim /etc/nova/nova.conf
```shell
[api_database]
# ...
connection = mysql+pymysql://nova:NOVA_DBPASS@controller/nova_api

[database]
# ...
connection = mysql+pymysql://nova:NOVA_DBPASS@controller/nova

[DEFAULT]
# ...
transport_url = rabbit://openstack:RABBIT_PASS@controller

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
username = nova
password = nova_pass

[DEFAULT]
# ...
my_ip = 192.168.1.41

[DEFAULT]
# ...
use_neutron = true
firewall_driver = nova.virt.firewall.NoopFirewallDriver

[neutron]
# reference Networking service install config

[vnc]
enabled = true
# ...
server_listen = $my_ip
server_proxyclient_address = $my_ip

[glance]
# ...
api_servers = http://controller:9292

[oslo_concurrency]
# ...
lock_path = /var/lib/nova/tmp

[DEFAULT]
# Due to a packaging bug, remove the log_dir option from the [DEFAULT] section

[placement]
# this is another related server
# ...
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = placement_pass
```
- su -s /bin/sh -c "nova-manage api_db sync" nova
- su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
- su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
- su -s /bin/sh -c "nova-manage db sync" nova
- verify: `su -s /bin/sh -c "nova-manage cell_v2 list_cells" nova`
- service nova-api restart
- service nova-consoleauth restart
- service nova-scheduler restart
- service nova-conductor restart
- service nova-novncproxy restart

# computer node
## install and config
- apt install nova-compute
- vim /etc/nova/nova.conf
```shell
[DEFAULT]
# ...
transport_url = rabbit://openstack:RABBIT_PASS@controller

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
username = nova
password = nova_pass

[DEFAULT]
# ...
my_ip = 192.168.1.43
use_neutron = true
firewall_driver = nova.virt.firewall.NoopFirewallDriver

[neutron]
# refer to the networking service install

[vnc]
# ...
enabled = true
server_listen = 0.0.0.0
server_proxyclient_address = $my_ip
novncproxy_base_url = http://controller:6080/vnc_auto.html

[glance]
# ...
api_servers = http://controller:9292

[oslo_concurrency]
# ...
lock_path = /var/lib/nova/tmp

[placement]
# ...
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = placement_pass
```
- egrep -c '(vmx|svm)' /proc/cpuinfo
	- if return 0, `vim /etc/nova/nova-compute.conf` `[libvirt].virt_type = qemu`
- service nova-compute restart

## Add the computer node in controller
- . admin-openrc
- openstack compute service list --service nova-compute
- su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
	- we can config `[scheduler].discover_hosts_in_cells_interval = 300` in /etc/nova/nova.conf to auto discover hosts.

# verify
- `. admin-openrc` in controller node
- openstack compute service list
- openstack catalog list
- openstack image list
- nova-status upgrade check