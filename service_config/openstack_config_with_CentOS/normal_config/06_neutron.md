---
title: OpenStack 6 neutron
date: 2020-10-21
tags: [OpenStack, neutron]
categories: 服务配置
---

# controller node
> Neutron server, ML2 plug-in, and DHCP agent, Layer-2 agent, Layer-3 agent.

## prepare
- mysql -u root -p
```shell
CREATE DATABASE neutron;
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY 'NEUTRON_DBPASS';
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'NEUTRON_DBPASS';
```
- . admin-openrc
- openstack user create --domain default --password-prompt neutron
- openstack role add --project service --user neutron admin
- openstack service create --name neutron --description "OpenStack Networking" network
- openstack endpoint create --region RegionOne network public http://controller:9696
- openstack endpoint create --region RegionOne network internal http://controller:9696
- openstack endpoint create --region RegionOne network admin http://controller:9696

## install & config
- yum install openstack-neutron openstack-neutron-ml2 openstack-neutron-linuxbridge ebtables
- vim /etc/neutron/neutron.conf
```shell
[database]
# ...
connection = mysql+pymysql://neutron:NEUTRON_DBPASS@controller/neutron

[DEFAULT]
# ...
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = true

[DEFAULT]
# ...
transport_url = rabbit://openstack:RABBIT_PASS@controller

[DEFAULT]
# ...
auth_strategy = keystone

[keystone_authtoken]
# ...
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = neutron
password = xxx

[DEFAULT]
# ...
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true

[nova]
# ...
auth_url = http://controller:5000
auth_type = password
project_domain_name = Default
user_domain_name = Default
region_name = RegionOne
project_name = service
username = nova
password = xxx

[oslo_concurrency]
# ...
lock_path = /var/lib/neutron/tmp
```
- vim /etc/neutron/plugins/ml2/ml2_conf.ini
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
- vim /etc/neutron/plugins/ml2/linuxbridge_agent.ini
```shell
[linux_bridge]
# tunnel network interface(ensxxx)
physical_interface_mappings = provider:PROVIDER_INTERFACE_NAME

[vxlan]
enable_vxlan = true
# manager network ipaddr
local_ip = OVERLAY_INTERFACE_IP_ADDRESS
l2_population = true

[securitygroup]
# ...
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```
- modprobe br_netfilter
- sysctl net.bridge.bridge-nf-call-iptables=1
- sysctl net.bridge.bridge-nf-call-ip6tables=1
- vim /etc/neutron/l3_agent.ini
```shell
[DEFAULT]
# ...
interface_driver = linuxbridge
```
- vim /etc/neutron/dhcp_agent.ini
```shell
[DEFAULT]
# ...
interface_driver = linuxbridge
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = true
```
- vim /etc/neutron/metadata_agent.ini
```shell
[DEFAULT]
# ...
nova_metadata_host = controller
# shared secret
metadata_proxy_shared_secret = METADATA_SECRET
```
- vim /etc/nova/nova.conf
```shell
[neutron]
# ...
url = http://controller:9696
auth_url = http://controller:5000
auth_type = password
project_domain_name = Default
user_domain_name = Default
region_name = RegionOne
project_name = service
username = neutron
password = xxx
service_metadata_proxy = true
metadata_proxy_shared_secret = METADATA_SECRET
```
- ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
- su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
- systemctl restart openstack-nova-api.service
- systemctl enable neutron-server.service neutron-linuxbridge-agent.service neutron-dhcp-agent.service neutron-metadata-agent.service
- systemctl start neutron-server.service neutron-linuxbridge-agent.service neutron-dhcp-agent.service neutron-metadata-agent.service
- systemctl enable neutron-l3-agent.service
- systemctl start neutron-l3-agent.service

# compute node
> ML2 plug-in, Layer-2 agent.

## Ubuntu OS
- apt install neutron-linuxbridge-agent
- vim /etc/neutron/neutron.conf
```shell
[database]
# ...

[DEFAULT]
# ...
transport_url = rabbit://openstack:RABBIT_PASS@controller

[DEFAULT]
# ...
auth_strategy = keystone

[keystone_authtoken]
# ...
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = neutron
password = xxx

[oslo_concurrency]
# ...
lock_path = /var/lib/neutron/tmp
```
- vim  /etc/neutron/plugins/ml2/linuxbridge_agent.ini
```shell
[linux_bridge]
# tunnel network interface(ensxxx)
physical_interface_mappings = provider:PROVIDER_INTERFACE_NAME

[vxlan]
enable_vxlan = true
# manager network ipaddr
local_ip = OVERLAY_INTERFACE_IP_ADDRESS
l2_population = true

[securitygroup]
# ...
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```
- sysctl net.bridge.bridge-nf-call-iptables=1
- sysctl net.bridge.bridge-nf-call-ip6tables=1
- vim /etc/nova/nova.conf
```shell
[neutron]
# ...
url = http://controller:9696
auth_url = http://controller:5000
auth_type = password
project_domain_name = Default
user_domain_name = Default
region_name = RegionOne
project_name = service
username = neutron
password = xxx
```
- service nova-compute restart
- service neutron-linuxbridge-agent restart
## CentOS OS
- yum install openstack-neutron-linuxbridge ebtables ipset
- vim /etc/neutron/neutron.conf
```shell
[database]
#connection = xxx

[DEFAULT]
# ...
transport_url = rabbit://openstack:RABBIT_PASS@controller

[DEFAULT]
# ...
auth_strategy = keystone

[keystone_authtoken]
# ...
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = neutron
password = xxx

[oslo_concurrency]
# ...
lock_path = /var/lib/neutron/tmp
```
- vim /etc/neutron/plugins/ml2/linuxbridge_agent.ini
```shell
[linux_bridge]
#tunnel network eth
physical_interface_mappings = provider:ensxxx

[vxlan]
enable_vxlan = true
# manager network ipaddr
local_ip = OVERLAY_INTERFACE_IP_ADDRESS
l2_population = true

[securitygroup]
# ...
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```
- modprobe br_netfilter
- sysctl net.bridge.bridge-nf-call-iptables=1
- sysctl net.bridge.bridge-nf-call-ip6tables=1
- vim /etc/nova/nova.conf
```shell
[neutron]
# ...
url = http://controller:9696
auth_url = http://controller:5000
auth_type = password
project_domain_name = Default
user_domain_name = Default
region_name = RegionOne
project_name = service
username = neutron
password = xxx
```
- systemctl restart openstack-nova-compute.service
- systemctl enable neutron-linuxbridge-agent.service
- systemctl start neutron-linuxbridge-agent.service

# verify
- controller node: `neutron agent-list`
