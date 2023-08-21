---
title: OpenStack 5 network
date: 2020-10-21
tags: [OpenStack, neutron]
categories: 服务配置
---

# controller node
- mysql
```shell
MariaDB [(none)]> CREATE DATABASE neutron;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY 'NEUTRON_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'NEUTRON_DBPASS';
MariaDB [(none)]> exit
```
- . admin-openrc
- openstack user create --domain default --password-prompt neutron
- openstack role add --project service --user neutron admin
- openstack service create --name neutron --description "OpenStack Networking" network
- openstack endpoint create --region RegionOne network public http://controller:9696
- openstack endpoint create --region RegionOne network internal http://controller:9696
- openstack endpoint create --region RegionOne network admin http://controller:9696

# 官方指导：
## controller node
- apt install neutron-server neutron-plugin-ml2 neutron-linuxbridge-agent neutron-l3-agent neutron-dhcp-agent  neutron-metadata-agent
- /etc/neutron/neutron.conf
- /etc/neutron/plugins/ml2/ml2_conf.ini
- /etc/neutron/plugins/ml2/linuxbridge_agent.ini
- /etc/neutron/l3_agent.ini
- /etc/neutron/dhcp_agent.ini
- /etc/neutron/metadata_agent.ini
- /etc/nova/nova.conf
## compute node
- apt install neutron-linuxbridge-agent
- /etc/neutron/neutron.conf
- /etc/neutron/plugins/ml2/linuxbridge_agent.ini
- /etc/nova/nova.conf

# 改进：
> controller: neutron-server neutron-linuxbridge-agent
> network: neutron-server neutron-plugin-ml2 neutron-linuxbridge-agent neutron-l3-agent neutron-dhcp-agent  neutron-metadata-agent
> compute: neutron-server neutron-linuxbridge-agent

## controller node
- vim `/etc/neutron/neutron.conf`
```shell
[database]
# ...
connection = mysql+pymysql://neutron:NEUTRON_DBPASS@controller/neutron

[DEFAULT]
# ...
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = true
transport_url = rabbit://openstack:RABBIT_PASS@controller
auth_strategy = keystone

[keystone_authtoken]
# ...
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = neutron_pass

[DEFAULT]
# ...
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true

[nova]
# ...
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = nova
password = nova_pass

[oslo_concurrency]
# ...
lock_path = /var/lib/neutron/tmp
```
- vim  `/etc/neutron/plugins/ml2/ml2_conf.ini`
```shell
[ml2]
# ...
type_drivers = flat,vlan,vxlan
tenant_network_types = vxlan
mechanism_drivers = linuxbridge,l2population
extension_drivers = port_security

[ml2_type_flat]
# ...
flat_networks = provider

[ml2_type_vxlan]
# ...
vni_ranges = 1:1000

[securitygroup]
# ...
enable_ipset = true
```
- vim `/etc/nova/nova.conf`
```shell
[neutron]
# ...
url = http://controller:9696
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = NEUTRON_PASS
service_metadata_proxy = true
metadata_proxy_shared_secret = METADATA_SECRET
```
- su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
- service nova-api restart
- service neutron-server restart

## network node
- apt install neutron-server neutron-plugin-ml2 neutron-linuxbridge-agent neutron-l3-agent neutron-dhcp-agent neutron-metadata-agent
- vim `/etc/neutron/neutron.conf`
```shell
[database]
# connect = xxx

[DEFAULT]
# ...
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = true
transport_url = rabbit://openstack:RABBIT_PASS@controller
auth_strategy = keystone

[keystone_authtoken]
# ...
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = neutron_pass

[oslo_concurrency]
# ...
lock_path = /var/lib/neutron/tmp
```

- vim `/etc/neutron/plugins/ml2/ml2_conf.ini`
```shell
[ml2]
# ...
type_drivers = flat,vlan,vxlan
tenant_network_types = vxlan
mechanism_drivers = linuxbridge,l2population
extension_drivers = port_security

[ml2_type_flat]
# ...
flat_networks = provider

[ml2_type_vxlan]
# ...
vni_ranges = 1:1000

[securitygroup]
# ...
enable_ipset = true
```

- vim `/etc/neutron/plugins/ml2/linuxbridge_agent.ini`
```shell
[linux_bridge]
physical_interface_mappings = provider:ens224

[vxlan]
enable_vxlan = true
local_ip = 192.168.1.40
l2_population = true

[securitygroup]
# ...
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```
- sysctl net.bridge.bridge-nf-call-iptables=1
- sysctl net.bridge.bridge-nf-call-ip6tables=1

- vim `/etc/neutron/l3_agent.ini`
```shell
[DEFAULT]
# ...
interface_driver = linuxbridge
```

- vim `/etc/neutron/dhcp_agent.ini`
```shell
[DEFAULT]
# ...
interface_driver = linuxbridge
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = true
```

- vim `etc/neutron/metadata_agent.ini`
```shell
[DEFAULT]
# ...
nova_metadata_host = controller
metadata_proxy_shared_secret = METADATA_SECRET
```

- 

## compute node
- apt install neutron-linuxbridge-agent
- vim `/etc/neutron/neutron.conf`
```shell
[database]
# connect = xxx

[DEFAULT]
# ...
transport_url = rabbit://openstack:RABBIT_PASS@controller
auth_strategy = keystone

[keystone_authtoken]
# ...
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = neutron_pass

[oslo_concurrency]
# ...
lock_path = /var/lib/neutron/tmp
```

- vim `/etc/neutron/plugins/ml2/linuxbridge_agent.ini`
```shell
[linux_bridge]
physical_interface_mappings = provider:ens224

[vxlan]
enable_vxlan = true
local_ip = 192.168.1.43
l2_population = true

[securitygroup]
# ...
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```
- sysctl net.bridge.bridge-nf-call-iptables=1
- sysctl net.bridge.bridge-nf-call-ip6tables=1
- /etc/init.d/nova-compute restart
- /etc/inig.d/neutron-linuxbridge-agent restart

- vim `/etc/nova/nova.conf`
```shell
[neutron]
# ...
url = http://controller:9696
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = neutron_pass
service_metadata_proxy = true
metadata_proxy_shared_secret = METADATA_SECRET
```

## verify
- controller node: `neutron agent-list`


controller:
yum install openstack-neutron openstack-neutron-ml2 python-neutronclient
/etc/neutron/neutron.conf
/etc/neutron/plugins/ml2/ml_conf.ini
/etc/nova/nova.conf
network:
yum install openstack-neutron openstack-neutron-ml2 openstack-neutron-openvswitch
/etc/neutron/neutron.conf
/etc/neutron/plugins/ml2/ml_conf.ini
/etc/neutron/l3_agent.ini
/etc/neutron/dhcp_agent.ini
/etc/neutron/metadata_agent.ini
compute:
yum install openstack-neutron-ml2 openstack-neutron-openvswitch
/etc/neutron/neutron.conf
/etc/neutron/plugins/ml2/ml2_conf.ini