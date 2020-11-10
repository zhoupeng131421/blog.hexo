---
title: OpenStack 5 nova
date: 2020-10-21
tags: [OpenStack, nova]
categories: 服务配置
---

# controller node
## prepare
- mysql -u root -p
```shell
CREATE DATABASE nova_api;
CREATE DATABASE nova;
CREATE DATABASE nova_cell0;

GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' IDENTIFIED BY 'NOVA_DBPASS';
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY 'NOVA_DBPASS';

GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY 'NOVA_DBPASS';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'NOVA_DBPASS';

GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' IDENTIFIED BY 'NOVA_DBPASS';
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' IDENTIFIED BY 'NOVA_DBPASS';
```
- . admin-openrc
- openstack user create --domain default --password-prompt nova
- openstack role add --project service --user nova admin
- openstack service create --name nova --description "OpenStack Compute" compute
- openstack endpoint create --region RegionOne compute public http://controller:8774/v2.1
- openstack endpoint create --region RegionOne compute internal http://controller:8774/v2.1
- openstack endpoint create --region RegionOne compute internal http://controller:8774/v2.1

## install & config
- yum install openstack-nova-api openstack-nova-conductor openstack-nova-novncproxy openstack-nova-scheduler
- vim /etc/nova/nova.conf
```shell
[DEFAULT]
# ...
enabled_apis = osapi_compute,metadata

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
password = xxx

[DEFAULT]
# ...
my_ip = 10.0.0.11

[DEFAULT]
# ...
use_neutron = true
firewall_driver = nova.virt.firewall.NoopFirewallDriver

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

[placement]
# ...
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = xxx
```
- su -s /bin/sh -c "nova-manage api_db sync" nova
- su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
- su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
- su -s /bin/sh -c "nova-manage db sync" nova
- su -s /bin/sh -c "nova-manage cell_v2 list_cells" nova
- systemctl enable openstack-nova-api.service openstack-nova-scheduler.service openstack-nova-conductor.service openstack-nova-novncproxy.service
- systemctl start openstack-nova-api.service openstack-nova-scheduler.service openstack-nova-conductor.service openstack-nova-novncproxy.service

# compute node
## install & config
### Ubuntu OS
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
password = xxx

[DEFAULT]
# ...
# manager network
my_ip = MANAGEMENT_INTERFACE_IP_ADDRESS

[DEFAULT]
# ...
use_neutron = true
firewall_driver = nova.virt.firewall.NoopFirewallDriver

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
password = xxx
```
- egrep -c '(vmx|svm)' /proc/cpuinfo
    - if return 0, then `vim /etc/nova/nova-compte.conf`: `[libvirt:]: virt_type = qemu`
- service nova-compute restart

### CentOS OS
- yum install openstack-nova-compute
- vim /etc/nova/nova.conf
```shell
[DEFAULT]
# ...
enabled_apis = osapi_compute,metadata

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
password = xxx

[DEFAULT]
# manager network ipaddr
my_ip = MANAGEMENT_INTERFACE_IP_ADDRESS

[DEFAULT]
# ...
use_neutron = true
firewall_driver = nova.virt.firewall.NoopFirewallDriver

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
password = xxx
```
- egrep -c '(vmx|svm)' /proc/cpuinfo
    - if return 0, then `vim /etc/nova/nova.conf`: `[libvirt]: virt_type = qemu`
- systemctl enable libvirtd.service openstack-nova-compute.service
- systemctl start libvirtd.service openstack-nova-compute.service

### controller operation
- controller: `. admin-openrc`
- controller: `openstack compute service list --service nova-compute`
- controller: discover`su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova`
- controller: auto discover: `vim /etc/nova/nova.conf`: `[schedular]: discovery_hosts_in_cells_interval = 300`

# verify
- . admin-openrc
- openstack compute service list
- openstack catalog list
- openstack image list
- nova-status upgrade check
    - bug: Forbidden: Forbidden (HTTP 403)
    - add config: `vim /etc/httpd/conf.d/00-placement-api.conf`:
    ``` shell
    <Directory /usr/bin>
    <IfVersion >= 2.4>
        Require all granted
    </IfVersion>
    <IfVersion < 2.4>
        Order allow,deny
        Allow from all
    </IfVersion>
    </Directory>
    ```
    - systemctl restart httpd